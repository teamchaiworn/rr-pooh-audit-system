# RR POOH: AI-Powered Automated Audit System

**In short:** Used **Claude Code (Enterprise)** to build and maintain a production audit system on our self-hosted n8n instance — end to end, solo, via Claude Code's MCP connection to n8n (not just occasional code snippets). It now runs unattended, auditing 700–1,000 delivery-proof submissions/day in 15–45 minutes (vs. 3–4 hours by hand), with measured error rates of ~1% and ~1.5% on the two highest-volume checks. Full breakdown, source code, and the debugging skills used are below for anyone who wants the details.

- **Function:** Automation Engineering
- **Type:** Production system, self-hosted (n8n) + Shopee internal LLM gateway (Compass), running **Qwen2.5-VL-72B-Instruct** (vision-language model)
- **Built with:** Claude Code, operating directly on the live n8n instance via MCP (Model Context Protocol) — no manual copy-pasting of workflow JSON
- **Status:** Live, in active use, under continuous accuracy improvement

---

## The Problem

Drivers marking a delivery "on-hold" must submit photo proof (e.g. a call log, a full cargo bay, a flooded road) — and someone has to verify each one is real and matches the claim. Doing this by hand across a high daily volume is slow, inconsistent between reviewers, and carries real business risk both ways: waving through a fake excuse costs the delivery SLA, wrongly rejecting a real one becomes a driver dispute.

## What Was Built

An end-to-end automated audit pipeline, built entirely on our self-hosted n8n instance, that uses **Qwen2.5-VL-72B-Instruct** — a vision-language model, served through Shopee's internal Compass LLM gateway — to inspect the submitted proof images and render a verdict across up to 8 audit categories per submission (image quality/legibility, call log verification, parcel visibility, vehicle cargo space, weather/accident/road-closure evidence, fuel gauge reading, etc.), then writes the result directly back to the source-of-truth tracking sheet.

**Two-stage pipeline, run in batches to handle volume reliably:**
- **Submit** — pulls unaudited rows, groups them by audit type, packages the images and instructions, and submits them to the model as a batch job. At 700–1,000 cases/day, calling the model one row at a time from n8n isn't viable — memory pressure and per-request timeouts would make it unreliable at this volume. Embedding the images as base64 into a single batch file submitted to Compass sidesteps both problems: every case in the batch runs without hitting a per-call timeout.
- **Poll & Process** — checks the batch job for completion, retrieves each result, applies deterministic business-rule logic on top of the model's raw findings (e.g. "does the extracted phone number actually match this driver's contact record," "does the photo's upload date match the on-hold event's date"), and writes the final Pass/Fail verdict to the sheet.

The system runs unattended on a schedule, with automatic retry logic for the inevitable hiccups of an external image/network dependency.

## How This Was Built

The entire pipeline — architecture, the 3 n8n workflows, prompt design, and the business-rule engine — was built and is maintained through **Claude Code**, operating directly on the live n8n instance over its MCP interface: reading the live workflow definition, editing/generating workflow code, validating it, and deploying it back to the same production workflow, all as tool calls in a single session rather than hand-editing JSON exports or clicking through the n8n UI node-by-node.

That same loop is what made the debugging discipline behind this system practical to sustain: every defect was found by pulling a live execution's raw output through this interface, root-caused, fixed in code, redeployed, and re-verified against the same real proof images — a tight iterate-and-verify cycle rather than a slow edit-deploy-and-hope one.

This wasn't ad-hoc troubleshooting — Claude Code was configured with two standing skills that structured the work: `debug-mantra` (reproduce reliably → trace the fail path → falsify the hypothesis before trusting it → cross-reference every run against a running ledger) for every defect investigation, and `scrutinize` (question whether a simpler fix exists → trace the real code path end-to-end → verify claims against evidence) for the architecture calls, including the decision to split the image-quality gate into its own pass. Both skills are attached alongside this document.

---

## Appendix: Core Implementation (Live Source)

The full workflow JSON (`workflows/`) is the actual production source. Below is a condensed tour for technical reviewers; non-technical readers can stop above.

**Architecture — 3 workflows:**

| Workflow | Role |
|---|---|
| `[Batch] RR POOH - A Submit` | Reads unaudited rows from the tracking sheet, classifies each into one of 8 audit groups by on-hold reason code, chunks them, and dispatches each chunk to the sub-workflow below. |
| `[Batch] RR POOH - A Chunk (sub)` | Fetches each proof image with validate-and-retry (guards against a since-diagnosed silent failure mode where a rate-limited fetch returns a small error payload disguised as a valid file), builds the model prompts per audit type, and submits the batch job. |
| `[Batch] RR POOH - B Poll & Process` | Polls the batch job, parses every result, applies the deterministic business-rule engine on top of the model's raw output, reconciles the image-quality gate against the rest of the sheet, and writes verdicts back. |

**1. Row classification** (`Filter + Classify`, A Submit) — maps each on-hold reason code to one of 8 audit groups, each requiring a different combination of checks:

```javascript
const G_MAP={
  "107":"G1","108":"G1",
  "105":"G2","132":"G2","133":"G2","109":"G2","110":"G2",
  "101":"G3","103":"G3","104":"G3",
  "131":"G4","106":"G5","134":"G6","102":"G7","135":"G8"
};
// G1=OCR, G2=call, G3=call+location, G4=call+vehicle, G5=external-obstruction only,
// G6=call+external, G7=call (inverted pass/fail), G8=fuel-gauge
```

**2. Prompt construction & hybrid classification** (`Build JSONL`, A Chunk Sub) — two design decisions worth calling out:

- **Fail-only prompt design.** The image-quality gate defines only *Fail* conditions and defaults everything else to Pass. Defining both Pass and Fail criteria created arbitration conflicts the model resolved inconsistently; Fail-only gives it one decision path.
- **Hybrid classify + assume-both routing.** For call-evidence categories, the model is queried 3 ways per row (`classify`, `assume-call`, `assume-chat`) in the same batch round, and code picks the right one downstream. This exists because negative prompt instructions alone ("never do X") proved insufficient to stop the model from misreading a call-log screen as a chat conversation.

**3. Deterministic business-rule engine** (`Process Results`, B Poll & Process) — the code-side layer that turns raw model output into an auditable verdict. It never trusts the model's read of a tracking number at face value: OCR results go through 3-tier matching (exact suffix → ambiguous-character normalization for lookalikes like `O`/`0`, `S`/`5`, `I`/`1`/`L` → edit-distance fallback) against the driver's actual contact record and the on-hold event's own date, and per-reason-code rules apply logic the model itself never sees — e.g. reason code `102` inverts the pass/fail polarity of "did the driver answer," while `103` is deliberately lenient. Full function bodies (`evalCallLog`, `pickHybridResult`, `computeOCR`, the reconciliation pass) are in `workflows/B_Poll_Process.json`.

**Reconciliation gate** (runs after every poll, order-independent and self-healing): a separate pass re-reads the sheet's `bbb` column and force-overwrites all 6 audit columns to `N/A` on any row where the image-quality gate failed — decoupling "is this photo usable at all" from every downstream judgment, so a bad photo can never accidentally produce a false Pass on, say, the call-log check.
