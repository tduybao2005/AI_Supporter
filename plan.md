# IMPLEMENTATION PLAN — Edge AI Clinical Assistant for Vietnamese Hospitals

> Version 3.1 (improved) | Qwen3-14B + QLoRA + 2-tier RAG | On-premise | Multi-instance, single model artifact
> Original Vietnamese plan preserved in `plan_v3_original_vi.md`.

---

## 0. What this product is — and is not

**Is:** A *clinician-in-the-loop drafting assistant*. It listens to / reads a consultation, retrieves relevant guidelines and the hospital's own data, and produces a **draft** structured report (SOAP + suggestions). The doctor edits, approves, and signs. The doctor is always the decision-maker and the legal author of the record.

**Is not:** An autonomous diagnostic or prescribing system. It never finalizes a record, never sends anything to a patient, and never acts without a clinician's signature.

**Core principles**
- Clinician-in-the-loop: the AI drafts, the doctor decides and is accountable.
- Fully on-premise: patient data never leaves the hospital.
- One fine-tuned model artifact for every hospital; only the RAG corpus changes per site.
- **Describe, never prescribe**: the system surfaces evidence and patterns; it does not recommend a specific treatment as "the right one."
- Fail safe: when retrieval is insufficient, the system abstains ("not enough data") rather than guessing.

---

## 1. Architecture overview

```
Doctor speaks / types the consultation
        ↓
Local ASR (PhoWhisper / Whisper-v3) + medical-lexicon biasing → Text
        ↓
PII scrubbing (transcript) → de-identified text
        ↓
RAG pipeline (automatic):
  Tier 1 (public, shared): MoH treatment guidelines + National Drug Formulary
  Tier 2 (per-hospital):   hospital formulary (in-stock drugs)
                         + prescribing-statistics table (ICD-10 → drug → %)
                         + patient record from HIS (role-based access)
        ↓
LLM Qwen3-14B (fine-tuned) → draft report (JSON, schema v3.0)
        ↓
Safety layer: deterministic checks (formulary, dose range, interactions, abstention)
        ↓
Doctor reviews + edits + signs → immutable audit log
```

> **Terminology fix:** This is **multi-instance, single-artifact**, not "multi-tenant." Each hospital runs its own isolated on-prem instance with its own data; they happen to share the same model file. No data is ever co-mingled across hospitals.

---

## Phase 0 — Foundations (BEFORE any code)

These gate everything. Do not start data work until 0.1–0.3 are answered.

### 0.1 Regulatory & legal position (ground it, don't assert it)
- [ ] Get a written legal opinion on **Software as a Medical Device (SaMD)** classification under **Nghị định 98/2021/NĐ-CP** — confirm whether a "drafting assistant with mandatory clinician sign-off" falls outside or inside medical-device registration, and on what conditions.
- [ ] Map obligations under **Nghị định 13/2023/NĐ-CP (PDPD)** for *sensitive personal data* (health): lawful basis, consent, data-subject rights, the requirement to file an Impact Assessment dossier.
- [ ] Map obligations under **Luật Khám bệnh, chữa bệnh 2023** for electronic medical records and who may author them.
- [ ] **Liability model:** define in writing who is responsible if a draft contributes to harm. Default position: the signing clinician is the author; the tool is decision-support. Get this reviewed, not assumed.
- [ ] Decide data-retention and audit-log policy that satisfies both clinical-record law and PDPD.

### 0.2 Evaluation foundation (the single most important deliverable)
- [ ] Build a **held-out gold eval set**: 200–500 real-style cases, answer keys written and double-reviewed by clinicians. Never used for training. Versioned.
- [ ] Define **acceptance criteria as concrete numbers** (see §1.5 for the rubric) *before* building anything.
- [ ] Define what a *failure* looks like and which failures are blocking vs. acceptable.

### 0.3 PII / de-identification pipeline
- [ ] Build automatic PII scrubbing for both transcripts and training data (names, phone, address, ID numbers — including names spoken mid-sentence in ASR output).
- [ ] Validate the scrubber's recall on a labeled sample; de-identification of free-text speech is hard and must be measured, not assumed.

### 0.4 Pilot hospital
- [ ] Secure at least one pilot hospital with a signed agreement covering HIS access, clinician time for eval/annotation, and a defined go-live gate.
- [ ] Identify the hospital's HIS vendor early — it dictates the integration effort (see §3).

---

## Phase 1 — Data & Dataset

### 1.1 RAG Tier 1 (doable now, no hospital needed)
- [ ] Download MoH treatment-guideline PDFs (moh.gov.vn).
- [ ] Download the National Drug Formulary (dav.gov.vn).
- [ ] Convert PDF → clean Markdown with Marker/Nougat.
- [ ] **Table-aware chunking** — never split by character count; that shreds dose tables. Keep each dose table intact as one chunk with its heading.
- [ ] Load into Qdrant locally; measure retrieval **recall@k** on a sample question set (target in §1.5).
- [ ] Record source metadata on every chunk (document name, edition, date) — required for citations and for the "knowledge refresh" workflow.

### 1.2 RAG Tier 2 (needs pilot hospital)

**Step A — Extract from HIS**
- [ ] Identify the HIS software (VietHIS, Medisoft, FPT.eHospital, HIS-VN, etc.).
- [ ] Request a periodic export: prescription date, ICD-10 code, list of drugs prescribed. **No** patient identifiers.
- [ ] Agree the export mechanism explicitly (API vs. DB view vs. manual CSV) — this is the integration cost driver.

**Step B — Automatic processing pipeline (Python)**
```
HIS export (CSV/Excel)
        ↓ remove identifier columns (PII)
        ↓ normalize ICD-10 codes
        ↓ group by ICD-10 → count frequency per drug
        ↓ drop drugs no longer in hospital stock
        ↓ COMPUTE GUARDRAIL FLAGS (see §1.4) — e.g. flag antibiotic-for-viral-URI patterns
        ↓ clean statistics table
        ↓ load into Qdrant (monthly refresh)
```

**Step C — Hospital formulary**
- [ ] Receive the in-stock drug list from the hospital pharmacist (usually Excel).
- [ ] Script reads + loads into Qdrant. This is the source of truth for the safety layer's formulary check.

### 1.3 Fine-tune dataset (JSONL)

**Pair structure**
```json
{
  "instruction": "You are a clinical drafting assistant...",
  "input": "[Transcript] + [RAG docs] + [Prescribing-statistics table] + [Schema]",
  "output": "{ report JSON per schema v3.0 }"
}
```

**Output schema (v3.0)** — unchanged from v3, retained:
```json
{
  "soap": { "S": "", "O": "", "A": "", "P": "" },
  "trieuchung": [], "tiensu": [], "diung": [],
  "chandoan_goiy": [], "icd10_goiy": [], "cls_goiy": [],
  "thuoc_dangdung": [],
  "thuoc_goiy": [{ "thuoc": "", "ly_do": "", "nguon": "" }],
  "canhbao": [], "thieu_thong_tin": [],
  "nguon_thamchieu": [{ "noidung": "", "nguon": "" }]
}
```

> **Source rules — only three allowed source types:**
> - A Tier-1 RAG document (MoH guideline, Formulary)
> - "Prescribing statistics [hospital name]" (Tier 2) — **always labeled as descriptive frequency, never as a recommendation**
> - "General medical literature" (Qwen3 background knowledge)
> Every `thuoc_goiy` and every claim in `soap.A`/`P` must carry a source. No source → it goes in `thieu_thong_tin`, not in the suggestion.

**Synthetic-data quality control (new — this was a hidden risk)**

Generating training data with a teacher LLM distills the teacher's *clinical errors* into your model. Mitigations:
- [ ] Hand-write 10–20 gold pairs first; use them as the style/quality bar.
- [ ] Use a strong teacher model, but **clinician review rate scales with risk**: 100% review of any pair containing a drug/dose; ≥20% random review of the rest.
- [ ] Reject any synthetic pair whose drug suggestions fail the §1.5 safety layer.
- [ ] Track inter-rater agreement among reviewers; if it's low, the rubric is ambiguous — fix it.

**Data volume plan**

| Stage | Pairs | Method |
|-------|-------|--------|
| Gold standard | 10–20 | Author + clinician, hand-written |
| First trial | 300–500 | LLM-generated, clinician spot-check per rule above |
| Scale | 2,000–5,000 | Only after eval set hits §1.5 thresholds |

**Specialty distribution**

| Group | Share |
|-------|-------|
| Cardiology (CAD, AFib, HTN) | 30% |
| Respiratory (pneumonia, COPD, asthma) | 20% |
| Gastroenterology | 15% |
| Endocrinology (DM, thyroid) | 15% |
| Neurology (stroke, headache) | 10% |
| Basic surgery | 5% |
| General Vietnamese (anti-forgetting) | 5–15% |

**New input type — teach the model to read the statistics table (descriptively):**
```
[Hospital prescribing statistics]:
  R05 (Cough): Drug D 87%, Drug E 45%, Drug G 30%
  Avg treatment duration: D+E ~5 days
```
→ Model must write: *"In practice this hospital frequently prescribes D (87%)."*
→ Model must **never** write: *"You should prescribe D."* (only the doctor decides)
→ When the statistics conflict with a Tier-1 guideline, the model must **surface the conflict**, not pick a side (see §1.4).

### 1.4 Anti-anchoring guardrails for Tier-2 statistics (new — critical)

The prescribing-statistics feature is the most novel idea and the most dangerous. Frequency is not endorsement, and real-world VN prescribing includes well-known bad habits (e.g. antibiotics for viral URIs). Guardrails:
- [ ] **Always frame as description**, with the explicit count/percentage and the word "frequency" — never bare drug names.
- [ ] **Guideline-conflict flag:** at pipeline build time, mark any (ICD-10 → drug) pattern that contradicts a Tier-1 guideline (e.g. antibiotic for a viral code). The model must show the flag, not hide it.
- [ ] **Never let statistics alone justify a suggestion.** A `thuoc_goiy` entry sourced only from Tier-2 statistics is downgraded to an FYI note, not a suggestion.
- [ ] **Stewardship metric:** track whether the tool's drafts increase or decrease inappropriate-antibiotic suggestions vs. baseline. This is both an ethics safeguard and a sales argument.

---

## 1.5 Evaluation rubric & acceptance criteria (new — the missing core)

You cannot say "passes threshold" without defining the threshold. Each output field is scored differently.

| Output element | Metric | How measured | Suggested go-live threshold |
|----------------|--------|--------------|------------------------------|
| ICD-10 suggestions | Top-3 accuracy | exact code match vs. gold | ≥ 85% |
| Symptom / history / allergy extraction | Recall & precision | span match vs. gold | recall ≥ 90%, precision ≥ 85% |
| Drug suggestions appropriateness | Blinded clinician 1–5 rating | ≥2 clinicians rate; report mean + % "unsafe" | mean ≥ 4.0, **unsafe = 0%** |
| SOAP free-text quality | Clinician 1–5 rubric (completeness, faithfulness, no fabrication) | blinded rating | mean ≥ 4.0 |
| **Hallucination / fabrication** | % of outputs with any unsupported clinical claim | clinician audit | **< 2%, hard gate** |
| Abstention correctness | % of "insufficient data" cases correctly abstained | vs. gold | ≥ 90% |
| Safety-layer catch rate | % of injected unsafe drug/dose cases blocked | red-team test set | 100% on dose-out-of-range & non-formulary |
| RAG retrieval | recall@5 | sample question set | ≥ 90% |
| Citation validity | % of cited sources that actually support the claim | audit | ≥ 95% |

**Rules**
- The **hallucination gate and the "0% unsafe drug" gate are blocking** — failing either means no go-live, regardless of other scores.
- Evaluation is run on the held-out set (never trained on) at every model version and at every new hospital (with that hospital's RAG corpus).
- Quarterly re-evaluation in production (§4).

---

## Phase 1.5 — Safety layer (expanded design)

This is **not** "50 lines of Python." It is a deterministic gate between the model and the doctor, backed by a structured knowledge base.

**Components**
- [ ] **Formulary check** — every drug in the output exists in this hospital's in-stock list; otherwise flag.
- [ ] **Dose-range check** — requires a structured drug DB with dose ranges; flag out-of-range. Phase 1: ranges for the top ~200 drugs by volume. Note explicitly where renal/hepatic/age/weight adjustment is *not yet* automated.
- [ ] **Drug–drug interaction check** — integrate a real DDI knowledge base; do not hand-roll. Flag major interactions against `thuoc_dangdung`.
- [ ] **Allergy cross-check** — cross-reference suggestions against the patient's recorded allergies (`diung`).
- [ ] **Abstention enforcement** — if RAG retrieval confidence is below threshold, force "insufficient data" rather than letting the model answer.
- [ ] **Disclaimer + immutable audit log** on every output (who, what, when, model version, RAG corpus version).
- [ ] **Scope of guarantee, stated honestly:** the safety layer is a backstop, not a guarantee of clinical correctness. Document what it does and does not check.

> Deploy only when both blocking gates in §1.5 pass on the eval set.

---

## Phase 2 — Fine-tune the model (one-time)

### Local setup (no GPU)
- [ ] Python 3.10+
- [ ] Build + validate `dataset.jsonl` (`python3 -m json.tool`, `jq empty` over all lines).
- [ ] Confirm the chat template matches Qwen3's expected format. **Qwen3 is a hybrid reasoning model** — decide explicitly whether you fine-tune in thinking or non-thinking mode and keep the template consistent between train and serve, or behavior will drift.
- [ ] Run 50–100 steps on free Colab to confirm loss decreases.

### Real training (cloud GPU — RunPod/Vast.ai)
- [ ] Rent an RTX 4090 (~$0.5/hr) — note: a 14B QLoRA over a few thousand examples and several epochs can run longer than 3–5h; budget realistically and watch utilization.
- [ ] Download Qwen3-14B-Instruct from HuggingFace (~28GB) directly on the cloud server.
- [ ] Install `unsloth`, `transformers`, `peft`, `trl`, `bitsandbytes`, `datasets`.
- [ ] Upload `dataset.jsonl`; enable WandB; watch the loss curve.
- [ ] Run QLoRA. Smooth loss → finish, export LoRA adapter. Anomalous loss → stop, inspect data.

### Packaging (on cloud, before releasing the server)
- [ ] Merge LoRA adapter into base Qwen3-14B.
- [ ] **Measure eval-set accuracy at fp16 / 8-bit / 4-bit** — quantization can degrade clinical accuracy; pick the smallest quant that still passes §1.5 gates.
- [ ] Produce the serving artifact that **matches your serving engine** (see the corrected stack below) — not necessarily GGUF.
- [ ] Download the chosen artifact; archive the adapter + base hash + dataset version for reproducibility.

---

## Phase 3 — Per-hospital deployment

### Hardware
- [ ] Internal LAN workstation: RTX 3090/4090 (24GB VRAM), 32GB+ RAM. **Budget ~$2,000–3,000 per site** — this is a real, recurring per-hospital cost, not $0.

### Serving — corrected stack (this was an error in v3)
Pick **one** consistent path:
- **Production / concurrency:** vLLM serving an **AWQ or GPTQ** (or FP8) quantized model. vLLM gives real multi-user throughput.
- **Dev / single-user / simplest ops:** Ollama or llama.cpp serving a **GGUF** model.
- ❌ Do **not** plan to serve GGUF under vLLM — its GGUF support is experimental and slow and defeats the reason to use vLLM.
- [ ] Load the artifact, run a smoke-test inference, then run the eval set.

### RAG setup for this site
- [ ] Install Qdrant locally.
- [ ] Load Tier 1 (shared MoH guidelines + Formulary).
- [ ] Run the HIS pipeline → load Tier 2 (statistics table + formulary).
- [ ] Connect read access to patient records from the HIS, role-based per doctor/department.

> **Reality check on HIS integration (was badly understated):** every Vietnamese HIS is different and many have no usable API — integration is often direct DB access or scheduled exports. **Budget weeks of engineering per new HIS vendor**, plus security review for live patient-record access. "Onboard a new hospital ~$0" is false; the honest number is "days only if the HIS vendor is one we've already integrated, otherwise weeks."

### Pre-go-live gate
- [ ] Run the (reduced) eval set with *this hospital's* guidelines + statistics.
- [ ] Pass §1.5 acceptance criteria (both blocking gates) → go-live.
- [ ] Fail → fix RAG/prompt; do not retrain.

### Integration
- [ ] **ASR robustness (expanded):** test PhoWhisper vs Whisper-v3 on the hospital's real audio; add **medical-lexicon biasing** for drug names/doses; run an ASR error eval focused on drug-name and dosage accuracy, because these errors propagate straight to patient safety.
- [ ] Wire the Python backend: transcript → PII scrub → RAG → LLM → safety layer → JSON.
- [ ] Open the internal web UI for doctors.

---

## Phase 4 — Operations & updates

### Knowledge updates (no retraining)
```
New MoH guideline   → remove old, load new into Qdrant Tier 1 (~15 min)
New stock drug      → update Tier-2 formulary
Prescribing stats   → pipeline runs monthly, auto-updates
```

### Behavior updates (requires LoRA retraining)
```
Systematic report-format errors / missing sections / off-tone
→ retrain LoRA (~hours, low GPU cost)
→ then RE-RUN the full eval set AND re-validate at EVERY deployed site
  before rolling the new artifact out (new — version governance)
```

> **What to tell hospitals (honest framing):**
> ✅ "No retraining needed to update *knowledge*."
> ❌ "Never need to retrain, ever."
> ➕ "A new model version is re-validated against the eval set and at your site before it reaches you."

### Monitoring
- [ ] Quarterly eval-set metrics in production.
- [ ] Track hallucination rate and the antibiotic-stewardship metric over time.
- [ ] Triage clinician-reported errors: knowledge error → fix RAG; behavior error → retrain.
- [ ] Maintain the audit log for the legally required retention period.

---

## Technical stack (corrected)

| Component | Choice | Notes |
|-----------|--------|-------|
| Base LLM | Qwen3-14B-Instruct | Apache 2.0, strong Vietnamese; mind hybrid thinking mode in fine-tuning |
| Fine-tune | QLoRA + Unsloth | Cloud, low cost |
| ASR | PhoWhisper / Whisper-v3 | + medical-lexicon biasing; eval on drug/dose accuracy |
| Vector DB | Qdrant (prod) / ChromaDB (dev) | local |
| **Serving (prod)** | **vLLM + AWQ/GPTQ/FP8** | concurrency — **not GGUF** |
| **Serving (dev)** | **Ollama / llama.cpp + GGUF** | single-user / simplest ops |
| Quantization | choose per serving engine; verify on eval set | smallest quant that passes §1.5 gates |
| Drug knowledge base | structured formulary + dose ranges + DDI source | powers the safety layer |
| Monitoring | WandB (train) + custom eval harness | |
| HIS pipeline | Python + pandas | + guideline-conflict flagging |

---

## Honest cost & effort model (replaces the "~$0" estimates)

| Item | Realistic cost |
|------|----------------|
| One-time QLoRA training run | low (tens of USD incl. retries), not the bottleneck |
| Small Colab test | $0 |
| Vector DB, serving software | $0 (open source) |
| Knowledge updates | ~$0 in compute (RAG edit) |
| **Per-hospital workstation** | **~$2,000–3,000 (recurring per site)** |
| **HIS integration (new vendor)** | **weeks of engineering + security review** |
| HIS integration (known vendor) | days |
| **Clinician time** | **the single largest real cost** — gold data, eval-set answer keys, blinded ratings, per-site validation |
| Per-site re-validation on model update | clinician + engineer time each release |
| Legal/regulatory opinion | one-time professional fee |

> The cheap part is the GPU. The expensive parts are clinician time, HIS integration, and getting evaluation/regulation right. Plan budgets accordingly.

---

## Indicative timeline (new)

Rough sequencing, not commitments — adjust to pilot reality.

| Milestone | Depends on |
|-----------|-----------|
| M1: Legal position + eval set defined + PII pipeline | Phase 0 |
| M2: Tier-1 RAG working, recall@5 ≥ 90% | M1 (parallel, no hospital needed) |
| M3: Gold dataset (10–20) + first trial set (300–500), clinician-reviewed | M1 |
| M4: First fine-tune passes eval-set gates | M2, M3 |
| M5: Safety layer + Tier-2 pipeline (pilot HIS) | M4 + pilot agreement |
| M6: Pilot go-live (passes both blocking gates on pilot's corpus) | M5 |
| M7: Scale dataset, second hospital onboarded | M6 |

Critical-path risks: pilot HIS access (M5/M6) and clinician availability for eval/annotation (M1–M6).

---

## Risk register (new)

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Synthetic data distills teacher's clinical errors | Wrong model behavior | Risk-scaled clinician review; safety-layer filtering of training pairs |
| Statistics RAG normalizes bad prescribing | Patient harm, ethics, liability | §1.4 anti-anchoring guardrails + stewardship metric |
| ASR mis-hears drug names/doses | Patient harm | Lexicon biasing + drug/dose-focused ASR eval; doctor confirms |
| HIS integration far harder than expected | Schedule + cost blowout | Identify vendor in Phase 0; honest per-vendor budget |
| SaMD classification triggers device regulation | Legal block | Phase 0 legal opinion before build |
| PII leaks via free-text transcript | PDPD violation | Measured de-id recall; on-prem only |
| Quantization degrades clinical accuracy | Silent quality loss | Eval at fp16/8/4-bit; gate on §1.5 |
| Model update breaks a deployed site | Regression in production | Per-site re-validation before rollout (§4) |
| vLLM+GGUF mismatch (original plan) | Broken/slow serving | Corrected stack above |

---

## Do-now steps (no GPU, no hospital needed)

1. Get the **legal/regulatory opinion** started (Phase 0.1) — longest lead time.
2. Draft the **evaluation rubric & acceptance numbers** (§1.5) and the annotation guideline.
3. Read 1–2 MoH guidelines to understand the domain.
4. Hand-write **10 gold JSONL pairs**; validate with `python3 -m json.tool`.
5. Have a clinician **review those 10 pairs** before any bulk LLM generation.
6. Build **Tier-1 RAG** and measure recall@5.
7. Begin **pilot-hospital negotiation**, confirming the **HIS vendor** up front.
