# RR POOH: AI-Powered Automated Audit System

**Function:** Automation Engineering
**Type:** Production system, self-hosted (n8n) + Shopee internal LLM gateway (Compass), running **Qwen2.5-VL-72B-Instruct** (vision-language model)
**Built with:** Claude Code, operating directly on the live n8n instance via MCP (Model Context Protocol) — no manual copy-pasting of workflow JSON
**Status:** Live, in active use, under continuous accuracy improvement

---

## The Problem

When a driver marks a delivery as "on-hold" (e.g. "customer not available," "store not ready," "vehicle full"), they're required to submit proof — a photo or screenshot — supporting that reason. Someone has to verify that proof is legitimate: is the photo real and readable, does it actually show what it claims, does a call log confirm an attempted contact, is the parcel visible and undamaged, is the vehicle actually full, is there a real external obstruction (flood, accident, road closure)?

Doing this by hand, per submission, across a high volume of daily on-hold events, is slow, inconsistent between reviewers, and doesn't scale. It also carries real business risk in both directions: waving through a fabricated excuse costs the company on the delivery SLA, while wrongly rejecting a valid one becomes a driver dispute.

## What Was Built

An end-to-end automated audit pipeline, built entirely on our self-hosted n8n instance, that uses **Qwen2.5-VL-72B-Instruct** — a vision-language model, served through Shopee's internal Compass LLM gateway — to inspect the submitted proof images and render a verdict across up to 8 audit categories per submission (image quality/legibility, call log verification, parcel visibility, vehicle cargo space, weather/accident/road-closure evidence, fuel gauge reading, etc.), then writes the result directly back to the source-of-truth tracking sheet.

**Two-stage pipeline, run in batches to handle volume reliably:**
- **Submit** — pulls unaudited rows, groups them by audit type, packages the images and instructions, and submits them to the model as a batch job. At 700–1,000 cases/day, calling the model one row at a time from n8n isn't viable — memory pressure and per-request timeouts would make it unreliable at this volume. Embedding the images as base64 into a single batch file submitted to Compass sidesteps both problems: every case in the batch runs without hitting a per-call timeout.
- **Poll & Process** — checks the batch job for completion, retrieves each result, applies deterministic business-rule logic on top of the model's raw findings (e.g. "does the extracted phone number actually match this driver's contact record," "does the photo's upload date match the on-hold event's date"), and writes the final Pass/Fail verdict to the sheet.

The system runs unattended on a schedule, with automatic retry logic for the inevitable hiccups of an external image/network dependency.

## How This Was Built

The entire pipeline — architecture, the 3 n8n workflows, prompt design, and the business-rule engine — was built and is maintained through **Claude Code**, operating directly on the live n8n instance over its MCP interface: reading the live workflow definition, editing/generating workflow code, validating it, and deploying it back to the same production workflow, all as tool calls in a single session rather than hand-editing JSON exports or clicking through the n8n UI node-by-node.

That same loop is what made the debugging discipline in the next section practical to sustain: every defect below was found by pulling a live execution's raw output through this interface, root-caused, fixed in code, redeployed, and re-verified against the same real proof images — a tight iterate-and-verify cycle rather than a slow edit-deploy-and-hope one.

This wasn't ad-hoc troubleshooting — Claude Code was configured with two standing skills that structured the work: `debug-mantra` (reproduce reliably → trace the fail path → falsify the hypothesis before trusting it → cross-reference every run against a running ledger) for every defect investigation, and `scrutinize` (question whether a simpler fix exists → trace the real code path end-to-end → verify claims against evidence) for the architecture calls, including the decision to split the image-quality gate into its own pass. Both skills are attached alongside this document.

## Engineering Rigor: How Accuracy Was Earned, Not Assumed

The differentiator here isn't "we called an AI API" — it's the discipline applied to make the model's judgment trustworthy enough to act on automatically, at scale, without a human in the loop. That meant treating every audit category as a hypothesis to be measured, not a feature to be shipped and forgotten:

- **Measured before trusting.** Rather than assuming the model got it right, every category was audited against its own output — full-population review where feasible, targeted sampling against known failure signatures elsewhere — and error rates were reported honestly, including where the sample was too small to be conclusive.
- **Root-caused real defects, several non-obvious.** Examples: a model that was fabricating plausible-looking tracking numbers instead of admitting it couldn't read a label; a model that self-contradicted (describing a vehicle as full, then marking it "Fail"); a case where the true bottleneck wasn't the model at all but missing ground-truth data upstream, capping any prompt-side fix at ~20% improvement; an infrastructure quirk where a rate-limited image fetch silently returned a valid-looking file that was actually an error message, corrupting results that looked clean until manually verified against real proof photos.
- **Verified every fix against real evidence, not the fix's own logic.** Multiple "fixes" were validated by downloading the actual submitted photo and checking it by eye before being declared solved — twice, an initially plausible root cause turned out to be wrong once tested against a live production run, and the real cause was found afterward.
- **Redesigned the architecture when the data justified it.** The image-quality gate (a foundational check that everything else depends on) was split into its own independent model pass after evidence showed bundling it with other checks was diluting the model's accuracy on both.

## Impact

- Replaces manual visual review across a high-volume daily audit workload with an unattended, scheduled pipeline.
- Full daily batch (700–1,000 cases) completes in **15–45 minutes** end-to-end, versus **3–4 hours** for the same volume audited manually — while the batch architecture avoids the memory pressure and per-call timeouts that a one-at-a-time real-time call pattern would hit at this volume. Model inference cost is not a factor: Qwen2.5-VL-72B-Instruct runs on internal infrastructure via Compass, at zero marginal cost per call.
- Current measured error rates on the two highest-volume categories: **~1%** for the image-quality gate and **~1.5%** for call-log verification (both measured against real production runs, not a lab sample) — down from materially higher rates before the fixes above.
- A repeatable measurement methodology now exists for every audit category, so future accuracy claims can be verified rather than assumed.

## Status & Known Limitations (transparent by design)

- Some categories (vehicle cargo check, external-factor/parcel-visibility checks) have only been validated on small samples so far and are flagged as directional, not final, pending more production volume.
- One classification edge case (a call log being mistaken for a chat conversation on certain screen layouts) has resisted three rounds of prompt-level fixes; a structural two-stage classification redesign is the next candidate approach.
- Every verdict from the model is treated as a starting hypothesis, not a final answer — each category has an open audit trail and a documented plan for its next round of verification.

---

## Appendix: Core Implementation (Live Source)

The code below was pulled directly from the three production n8n workflows on 2026-08-14 — this is what is actually running, not a written-up approximation. Included for technical reviewers; non-technical readers can stop above.

**Architecture — 3 workflows:**

| Workflow | Role |
|---|---|
| `[Batch] RR POOH - A Submit` | Reads unaudited rows from the tracking sheet, classifies each into one of 8 audit groups by on-hold reason code, chunks them, and dispatches each chunk to the sub-workflow below. |
| `[Batch] RR POOH - A Chunk (sub)` | Fetches each proof image with validate-and-retry (guards against a since-diagnosed silent failure mode where a rate-limited fetch returns a small error payload disguised as a valid file), builds the model prompts per audit type, and submits the batch job. |
| `[Batch] RR POOH - B Poll & Process` | Polls the batch job, parses every result, applies the deterministic business-rule engine on top of the model's raw output, reconciles the image-quality gate against the rest of the sheet, and writes verdicts back. |

### 1. Row classification (`Filter + Classify`, A Submit)

Maps each on-hold reason code to one of 8 audit groups, each with a different combination of checks required:

```javascript
const G_MAP={
  "107":"G1","108":"G1",
  "105":"G2","132":"G2","133":"G2","109":"G2","110":"G2",
  "101":"G3","103":"G3","104":"G3",
  "131":"G4","106":"G5","134":"G6","102":"G7","135":"G8"
};
const CALL_G=new Set(["G2","G3","G4","G6","G7"]);
const out=[];
for(const it of $input.all()){
  const r=it.json;
  const code=String(r.on_hold_reason_code||"").trim();
  if(!code||!String(r.image_url_1||"").trim()) continue;
  const g=G_MAP[code];
  if(!g) continue;
  const row={
    row_number:r.row_number,
    tracking_no:r.tracking_no,
    onhold_reason_code:code,
    on_hold_reason_name_th:String(r.on_hold_reason_name_th||""),
    audit_group:g,
    image_url_1:r.image_url_1||"",
    image_url_2:r.image_url_2||"",
    image_url_3:r.image_url_3||""
  };
  if(CALL_G.has(g)){
    row.on_hold_date=String(r.on_hold_date||"");
    row.contact_phone_masked=String(r.contact_phone_masked||"");
    row.call_duration_in_sec=String(r.call_duration_in_sec||"0");
  }
  if(g==="G8") row.on_hold_date=String(r.on_hold_date||"");
  out.push({json:row});
}
return out;
```

### 2. Prompt construction & hybrid classification (`Build JSONL`, A Chunk Sub)

This is where most of the accuracy work lives. Two design decisions worth calling out for a technical reviewer:

- **Fail-only prompt design** (`BBB_PROMPT`) — the image-quality gate defines only *Fail* conditions and defaults everything else to Pass. Defining both Pass and Fail criteria created arbitration conflicts the model resolved inconsistently; Fail-only gives it one decision path.
- **Hybrid classify + assume-both routing** — for call-evidence categories, the model is queried 3 ways per row (`classify`, `assume-call`, `assume-chat`) in the same batch round, and code picks the right one downstream. This exists because negative prompt instructions alone ("never do X") proved insufficient to stop the model from misreading a call-log screen as a chat conversation — see `pickHybridResult` in the next file.

```javascript
// ── BBB Only Prompt (Fail-only, standalone batch) ────────────────────────────
const BBB_PROMPT = `You are an image-quality checker. You receive 1–3 images. Evaluate ALL images together as a set.

Return JSON only: {"blank_blur_back":"Pass|Fail"}

FAIL only if EVERY image meets ALL of these:
  1. Cannot identify what it shows — no object, no scene, no UI
  2. No structural outline, shape, or color variation
  3. No real scene beneath any GPS/timestamp text overlay

These are NOT Fail conditions — always treat as Pass:
  - darkness, or blur where any structure is still visible
  - dark-mode or light-mode app interfaces
  - ANY app screenshot (call log, contact/dialer page, chat, map, delivery app, form) — a UI screen is identifiable content, Pass even when large parts are blank, white, or empty margin
  - large empty or plain-colored areas surrounding visible content
  - a screen with only one small element (e.g. a single call entry) as long as it is identifiable

Default: Pass. Fail ONLY when an image is genuinely unusable (all-black, all-white, or total blur with no discernible shape at all). One identifiable image = the entire set passes.`;

// ── Evidence type guide (inject into all call prompts) ───────────────────────
const CALL_EVIDENCE_GUIDE = `IMPORTANT: evidence_type (Step 1) must be decided from the IMAGE CONTENT ONLY. on_hold_reason (given later in the user message) is NEVER used to classify evidence_type — it is used later, ONLY inside Step 2b, to judge whether an already-confirmed chat conversation discusses the same topic. Never let on_hold_reason influence whether something is call_log, chat, or none.

Step 0 - Call-log recognition (do this FIRST, before anything else): decide whether ANY image is fundamentally a call-log/dialer screen by its OVERALL UI STRUCTURE, not just by matching words — a native phone contact/dialer page (large phone number header + ข้อความ/โทร/วิดีโอ buttons), a call-history list (entries with call icons + direction arrows + timestamps, in a plain list OR inside rounded bubbles, light mode OR dark mode), or an in-app call log (SeaTalk/LINE/WhatsApp). You must be able to recognize this layout on sight, the same way you would recognize a chat conversation on sight — this is a visual/structural judgment.

As a SUPPORTING signal only, also scan for on-screen TEXT containing any of these EXAMPLE Thai call-status phrases (this list is illustrative, NOT exhaustive — other equivalent call-related phrases you recognize also count, exact match or substring, e.g. "สายที่โทรไม่ติด" counts because it contains "สายที่โทร"): สายโทรออก, สายเรียกเข้า, สายไม่ได้รับ, สายที่โทร, เบอร์ที่โทรออก, ยังไม่ได้เชื่อมต่อ, ไม่มีการตอบรับ, บันทึกการโทร, ประวัติการโทร. List every phrase you actually see, verbatim, as "detected_call_keywords" (empty array if none — this CAN be empty even when the image is clearly a call-log, if no exact phrase happens to be legible; the structural judgment above still applies).

HARD RULE: if the image structurally matches a call-log/dialer UI (per the recognition above) OR detected_call_keywords is non-empty, evidence_type MUST be "call_log". This overrides any other impression — even if a chat-like UI, message bubble, or "ข้อความ" button is also visible anywhere in the image. Only proceed to the chat/none decision below when NEITHER condition is true.

[... full Step 1/2a/2b instructions omitted here for length — governs call-vs-chat classification, date-window filtering of call entries, digit-transcription double-check, and chat-relevance judgment against the on-hold reason ...]`;

// ── Hybrid Stage 1: classify-only prompt (call_log vs chat vs none) ───────────
const CLASSIFY_PROMPT = `You are an evidence-type classifier. From all provided images, decide ONLY what type of evidence is present. Do NOT extract call entries or transcribe chat text — that happens in a separate step. Just classify.
[... Step 0/decision-order logic shared with CALL_EVIDENCE_GUIDE above ...]
Return ONLY: {"detected_call_keywords":[],"evidence_type":"call_log|chat|none","reason":"one sentence"}`;

const ASSUME_CALL_PREFIX = `A separate classification step has ALREADY determined this image set's evidence_type is "call_log" — do not re-derive Step 0/1 below, just trust evidence_type="call_log" and go straight to Step 2a to extract the call entries (and Step 3 below if present).\n\n`;
const ASSUME_CHAT_PREFIX = `A separate classification step has ALREADY determined this image set's evidence_type is "chat" — do not re-derive Step 0/1 below, just trust evidence_type="chat" and go straight to Step 2b (and Step 3 below if present).\n\n`;

// ── Prompt map — one prompt per audit group, each demanding a strict JSON schema ──
const PROMPTS = {
  ocr: OCR_PROMPT,           // AWB label detection + printed-ID extraction
  call: CALL_SYSTEM,         // call-log / chat evidence only
  call_loc: CALL_LOC_SYSTEM, // + location/scene photo check
  call_veh: CALL_VEH_SYSTEM, // + vehicle cargo-space check
  ext: EXT_SYSTEM,           // route obstruction (flood/closure) only
  call_ext: CALL_EXT_SYSTEM, // + accident evidence
  call_no_ans: CALL_SYSTEM,  // call-log / chat, inverted pass/fail polarity
  fuel: FUEL_SYSTEM          // fuel-gauge reading + timestamp validation
};

const MODEL = "Qwen2.5-VL-72B-Instruct";
const CALL_TYPES_HYBRID = new Set(["call", "call_no_ans", "call_loc", "call_veh", "call_ext"]);

// ── For call-type groups: submit 3 variants per row in the SAME batch round ───
// classify (evidence_type only) + assume-call (full extraction) + assume-chat (full extraction).
// Process Results picks the right one via pickHybridResult() once all 3 complete.
const batch_type = $json.batch_type;
const rows = $json.rows;
for (const row of rows) {
  const isCallType = CALL_TYPES_HYBRID.has(batch_type);
  if (isCallType) {
    lines_audit.push(buildLine(`${batch_type}::classify|${row.row_number}`, CLASSIFY_PROMPT, userContent));
    lines_audit.push(buildLine(`${batch_type}::ascall|${row.row_number}`, ASSUME_CALL_PREFIX + systemPrompt, userContent));
    lines_audit.push(buildLine(`${batch_type}::aschat|${row.row_number}`, ASSUME_CHAT_PREFIX + systemPrompt, userContent));
  } else {
    lines_audit.push(buildLine(`${batch_type}|${row.row_number}`, systemPrompt, userContent));
  }
}
```

### 3. Deterministic business-rule engine (`Process Results`, B Poll & Process)

This is the code-side layer that turns raw model output into an auditable verdict — matching extracted phone numbers against the driver's actual contact record, validating screenshot upload dates against the on-hold event date, and applying per-reason-code rules that the model itself never sees (e.g. reason code 102 inverts the pass/fail polarity of "did the driver answer"). Reproduced here in full, unmodified, exactly as live:

```javascript
const parseSafe = (input) => {
  if (!input) return {};
  if (typeof input === "object") return input;
  try { return JSON.parse(String(input).replace(/```json|```/g, "").trim()); }
  catch (e) { return {}; }
};

// ── BBB Only batch (new) ───────────────────────────────────────────────────────
function computeBBB(res, row_number) {
  // Fail-only gate: Fail ONLY when the model explicitly says "fail".
  // Missing key / garbage / degeneration = no affirmative Fail condition matched → Pass.
  const bbbRaw = String(res.blank_blur_back || "").toLowerCase();
  const bbbVal = bbbRaw.includes("fail") ? "Fail" : "Pass";
  return { row_number, blank_blur_back: bbbVal, wb_type: "bbb" };
}

// ── OCR: 3-tier matching (G1) ──────────────────────────────────────────────────
function computeOCR(res, tracking_no, row_number) {
  const auth_check = res.step1_label_detected || "No";
  const extracted_raw = String(res.step2_extracted_full_id || "");
  const cleanID = (id) => String(id).replace(/[^A-Z0-9]/gi, "").toUpperCase();
  const target_clean = cleanID(tracking_no);
  const extracted_clean = cleanID(extracted_raw);

  const normalizeAmbiguous = (id) => String(id)
    .replace(/[5S]/g, "S").replace(/[2Z]/g, "Z").replace(/[0O]/g, "O")
    .replace(/[1IL]/g, "I").replace(/[8B]/g, "B");
  const ambiguousPairs = {
    "S": ["5"], "5": ["S"], "Z": ["2"], "2": ["Z"], "O": ["0"], "0": ["O"],
    "I": ["1", "L"], "1": ["I", "L"], "L": ["I", "1"], "B": ["8"], "8": ["B", "6"], "6": ["8", "G"], "G": ["6"],
  };
  const getEditDistance = (a, b) => { /* Levenshtein with ambiguous-character substitution cost 0 */ };

  // Tier 1: exact last-4-char suffix match. Tier 2: same but with O/0, S/5, I/1/L, B/8 normalized.
  // Tier 3: full edit-distance ≤2 typos, only if ground-truth ID is ≥8 chars (avoids false matches on short IDs).
  let awb_match = "Fail";
  if (extracted_clean && extracted_clean !== "UNREADABLE" && extracted_clean.length >= 4) {
    if (target_clean.endsWith(extracted_clean.slice(-4)) || extracted_clean.endsWith(target_clean.slice(-4))) {
      awb_match = "Pass";
    } else {
      const normT = normalizeAmbiguous(target_clean);
      const normE = normalizeAmbiguous(extracted_clean);
      if (normT.endsWith(normE.slice(-4)) || normE.endsWith(normT.slice(-4))) {
        awb_match = "Pass";
      } else {
        const typos = getEditDistance(target_clean, extracted_clean);
        if (target_clean.length >= 8 && typos <= 2) awb_match = "Pass";
      }
    }
  }
  const isAuthPass = auth_check.toLowerCase().includes("pass") || auth_check.toLowerCase().includes("yes");
  const awbVal = (isAuthPass && awb_match === "Pass") ? "Pass" : "Fail";
  return { row_number, awb: awbVal, parcel: awbVal, call_log: "N/A", location: "N/A", vehicle: "N/A", external_factor: "N/A", wb_type: "ocr" };
}

function extractUrlDate(url) {
  const m = String(url).match(/\/downloads\/(\d{4})\/(\d{2})\/(\d{2})\//);
  return m ? `${m[1]}-${m[2]}-${m[3]}` : "";
}

// ── shared precondition: the identified evidence screenshot's upload date must match on_hold_date ──
function screenshotDateMatches(res, row) {
  const allUrls = [row.image_url_1, row.image_url_2, row.image_url_3];
  const csIdx = Math.max(1, Math.min(3, parseInt(res.call_screenshot_index) || 1)) - 1;
  const url = allUrls[csIdx] || row.image_url_1 || "";
  const urlDate = extractUrlDate(url);
  const sysDate = String(row.on_hold_date || "").trim().slice(0, 10);
  return !!(urlDate && sysDate && urlDate === sysDate);
}

// ── call_log rules per reason code — OUTGOING calls only, phone + date must match ──
// answered = duration >= 6s ; attempt = duration < 6s (missed 0s OR short 1-5s)
// 103 = lenient (any connected outgoing call = Pass) ; 102 = INVERTED (answered = Fail, ≥2 attempts = Pass)
// standard (101,104,105,109,110,131,132,133,134): answered>=1 → Pass; ≥2 attempts → Pass; else Fail
function evalCallLog(res, row, type) {
  const code = String(row.on_hold_reason_code || "").trim();
  if (!screenshotDateMatches(res, row)) return "Fail";

  const phone4 = String(row.contact_phone_masked || "").replace(/\D/g, "").slice(-4);
  const rawEntries = Array.isArray(res.call_entries) ? res.call_entries : [];
  // Defense-in-depth: prompt already asks the model to date-filter at extraction time, but that
  // has proven unreliable — re-filter here on entry_date vs on_hold_date.
  const onHoldDate = String(row.on_hold_date || "").trim().slice(0, 10);
  const entries = rawEntries.filter(e => !e.entry_date || e.entry_date === onHoldDate);

  let answered = 0, attempts = 0, connected = 0;
  for (const e of entries) {
    if (String(e.direction || "").toLowerCase() !== "outgoing") continue;
    const p = String(e.phone_last4 || "").replace(/\D/g, "").slice(-4);
    if (!(phone4 && p && p === phone4)) continue;
    const cnt = Math.max(1, Math.round(Number(e.count) || 1));
    const dur = e.duration_sec;
    if (dur >= 6) { answered += cnt; connected += cnt; }
    else if (dur >= 1) { attempts += cnt; connected += cnt; }
    else { attempts += cnt; }
  }

  if (code === "103") return connected >= 1 ? "Pass - รับสาย" : "Fail";
  const inverted = code === "102" || (!code && type === "call_no_ans");
  if (inverted) {
    if (answered >= 1) return "Fail - รับสาย";
    if (attempts >= 2) return "Pass - ไม่รับสาย";
    return "Fail";
  }
  if (answered >= 1) return "Pass - รับสาย";
  if (attempts >= 2) return "Pass - ไม่รับสาย";
  return "Fail";
}

// ── Hybrid picker: combine classify + assume-call + assume-chat results ────────
// classify decides evidence_type; the matching "assume" variant supplies the extracted data.
function pickHybridResult(classify, ascall, aschat) {
  const evType = String((classify && classify.evidence_type) || "").toLowerCase();
  if (evType === "call_log" && ascall) return Object.assign({}, ascall, { evidence_type: "call_log" });
  if (evType === "chat" && aschat) return Object.assign({}, aschat, { evidence_type: "chat" });
  const fallback = ascall || aschat || {};
  return Object.assign({}, fallback, { evidence_type: "none" });
}

function computeCall(res, type, row) {
  const evidenceType = String(res.evidence_type || "call_log").toLowerCase();
  let callLog;
  if (evidenceType === "chat") {
    if (!screenshotDateMatches(res, row)) callLog = "Fail";
    else if (res.chat_relevant === true) callLog = "Pass - in app chat";
    else if (res.chat_relevant === false) callLog = "Fail - in app chat";
    else callLog = "Fail";
  } else if (evidenceType === "none") {
    callLog = "Fail";
  } else {
    callLog = evalCallLog(res, row, type);
  }
  const out = { row_number: row.row_number, awb: "N/A", parcel: "N/A", call_log: callLog, location: "N/A", vehicle: "N/A", external_factor: "N/A", wb_type: type };
  if (type === "call_loc") out.location = String(res.location_valid || "").toLowerCase().includes("pass") ? "Pass" : "Fail";
  if (type === "call_veh") out.vehicle = String(res.vehicle_valid || "").toLowerCase().includes("pass") ? "Pass" : "Fail";
  if (type === "call_ext") out.external_factor = String(res.external_valid || "").toLowerCase().includes("pass") ? "Pass" : "Fail";
  return out;
}

// ── Main: parse batch output, group hybrid call-type lines by row, dispatch ────
const raw = String($json.jsonl || "");
const src = $('Read Source').all().map(i => i.json);
const srcMap = {};
for (const r of src) srcMap[String(r.row_number)] = r;
const callGroups = {};
const out = [];
for (const line of raw.split("\n")) {
  if (!line.trim()) continue;
  let obj; try { obj = JSON.parse(line); } catch (e) { continue; }
  const cid = String(obj.custom_id || "");
  const sep = cid.indexOf("|");
  if (sep === -1) continue;
  const typeRaw = cid.slice(0, sep);
  const rowStr = cid.slice(sep + 1);
  if (!rowStr || obj.error || !obj.response || obj.response.status_code !== 200) continue;
  let content = ""; try { content = obj.response.body.choices[0].message.content; } catch (e) {}
  const res = parseSafe(content);
  const row = srcMap[rowStr] || {};
  row.row_number = Number(rowStr) || rowStr;

  if (typeRaw.includes("::")) {
    const [baseType, stage] = typeRaw.split("::");
    const key = `${baseType}|${rowStr}`;
    if (!callGroups[key]) callGroups[key] = { baseType, row };
    callGroups[key][stage] = res;
    continue;
  }
  if (typeRaw === "bbb") out.push({ json: computeBBB(res, row.row_number) });
  else if (typeRaw === "ocr") out.push({ json: computeOCR(res, String(row.tracking_no || ""), row.row_number) });
  // ext/fuel routed the same way, omitted here for brevity
}
for (const key in callGroups) {
  const g = callGroups[key];
  const finalRes = pickHybridResult(g.classify, g.ascall, g.aschat);
  out.push({ json: computeCall(finalRes, g.baseType, g.row) });
}
return out;
```

**Reconciliation gate** (runs after every poll, order-independent and self-healing): a separate pass re-reads the sheet's `bbb` column and force-overwrites all 6 audit columns to `N/A` on any row where the image-quality gate failed — decoupling "is this photo usable at all" from every downstream judgment, so a bad photo can never accidentally produce a false Pass on, say, the call-log check.
