# BRAHMO Clinical AI — Final Architecture and Doctor-Readiness Plan

**Candidate:** Naresh  **Submission date:** 17 October 2026
**Assessment opened:** [HH:MM IST, fill in]  **Submitted:** [HH:MM IST, fill in]

> Reading key. **[V]** = checked against a secondary source while drafting (primary document still to be read). **[U]** = not verified; must be confirmed before enabling the dependent capability. Departures from BRAHMO's pack are marked **▲** and listed in §S1.7.

---

## S1 · Final Architecture

### S1.1 Posture
A clinician-facing intelligence layer with three jobs: (1) resolve Indian brand prescribing to ingredients, (2) run deterministic medication-safety checks, (3) answer reference questions from versioned, cited Indian sources. A frontier LLM is used only for (a) Scribe entity extraction upstream and (b) composing text from already-retrieved, cited evidence. It never sits in a check path, never produces a dose for a patient, and never talks to patients. The 12 fixed laws and fences are kept as written.

### S1.2 Diagram

```
 DOCTOR (shared PC / phone, per-doctor login, tenant-scoped session)
   │ voice/text (EN + Hindi/Telugu/Tamil/Kannada… mixed)
   ▼
┌──────────────────────────  UPSTREAM (not ours)  ──────────────────────────┐
│ SCRIBE: transcript + structured Rx entities  │ PATIENT-CONTEXT: snapshot   │
│  (entity_status: unconfirmed until doctor ✓) │  (versioned, signed, expiry)│
└───────────────┬──────────────────────────────┴───────────────┬────────────┘
   ═══════ TRUST BOUNDARY 1: all inbound content = DATA, never instructions ═══════
                ▼                                              ▼
        ┌──────────────┐  ambiguous? ┌───────────────────┐
        │ INTAKE &     │────────────▶│ AMBIGUITY QUEUE   │──▶ reviewer ──▶ (never auto-resolved)
        │ NORMALISER   │             │ (reason codes)    │
        └──────┬───────┘             └───────────────────┘
               ▼ ingredient-level, form, strength, scope
 ┌─────────────────────────────┐   ┌───────────────────────────────────────┐
 │ INDIA DRUG MASTER           │   │ KNOWLEDGE STORE (immutable releases)   │
 │ alias→brand→salt, FDC split │   │ decision-unit chunks + metadata        │
 │ effective-dated reg. events │   │ lexical + dense index per release      │
 │ cross-source contradiction  │   │ trust tiers: raw→structured→verified→  │
 │ checks (P07)                │   │ live                                   │
 └──────────┬──────────────────┘   └──────────────────┬────────────────────┘
            ▼                                          ▼
 ┌─────────────────────────────┐   ┌───────────────────────────────────────┐
 │ SAFETY RAIL (no model)      │   │ REFERENCE Q&A PATH                     │
 │ duplicate ingredient        │   │ route → structured-first → retrieve    │
 │ banned/restricted FDC       │   │ → applicability filter → sufficiency   │
 │ severe interaction (curated)│   │ gate → [LLM compose, cited] → claim-   │
 │ allergy/pregnancy rules     │   │ support verifier → or ABSTAIN          │
 │ coverage manifest → 8 states│   └──────────────────┬────────────────────┘
 └──────────┬──────────────────┘                      │
   ═══════ TRUST BOUNDARY 2: LLM output is untrusted; claims without citation ids are stripped ═══
            ▼                                          ▼
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ ORCHESTRATOR + DOCTOR SURFACE: interrupting vs passive display, coverage │
 │ badge, source labels (India / tenant / non-India / background), clarify  │
 │ prompts → DOCTOR REVIEW → APPROVE → save / export (English Rx, PDF)      │
 └──────────────────────────────┬───────────────────────────────────────────┘
                                ▼
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ AUDIT LOG (append-only, hash-chained) · AI REGISTER (owners, versions)   │
 │ CORRECTION INBOX → reviewer queues · TENANT OVERLAY (labelled, additive) │
 └──────────────────────────────────────────────────────────────────────────┘
```

### S1.3 Components and responsibilities

| Component | Responsibility | Never does |
|---|---|---|
| Intake & Normaliser | Validates Scribe/context contracts, normalises spelling/transliteration to candidate brands, attaches `entity_status` | Resolves an ambiguous brand by itself |
| Drug Master | Graph alias→brand→salt with strength, dose form, FDC decomposition; event store of regulatory status with scope; P07 contradiction detector | Stores `banned=true/false`; lets a pharmacy overlay redefine canonical truth |
| Safety Rail | Pure rules over versioned data; returns one of 8 states per check with coverage manifest | Calls a model; renders absence of a rule as safe |
| Knowledge Store | Decision-unit chunks, release versioning, retrieval eligibility, withdrawal | Serves unreviewed content |
| Reference Q&A | Structured-first routing, retrieval, applicability filter, sufficiency gate, cited composition, claim-support verification, abstention | Answers consequential claims from model memory |
| Doctor Surface | Shows results, states, sources, clarification, approval | Auto-finalises anything |
| Audit & Register | Replay, ownership, change approvals | Stores PHI unencrypted |
| Correction Inbox | One-tap capture, routing to review | Turns a correction into truth |

### S1.4 Trust boundaries
TB1: everything arriving from Scribe, patient-context, documents, imports and tenant data is data; it is schema-validated, size-limited and never concatenated into a prompt as instructions. TB2: model output is untrusted; every claim must carry a chunk/rule id that a deterministic verifier can match to a quoted span. TB3 (tenant): row-level isolation, per-tenant keys; overlays are additive. TB4 (provider): only minimum de-identified context leaves the India-region boundary.

### S1.5 Key entities and interfaces

**Entities:** `Tenant, Doctor, Encounter, ScribeEntity, PatientContextSnapshot, Brand, Salt, Ingredient, FDCComponent, RegEvent, SourceDoc, SourceRelease, Chunk, Rule, RuleBatch, CoverageManifest, CheckResult, Answer, Claim, Citation, Correction, AuditEvent, RegisterEntry`.

**Scribe contract (inbound).** Fields per entity: `entity_id`, `type` (drug|strength|form|route|frequency|duration|allergy|condition), `raw_text`, `normalized_candidates[]` with scores, `transcript_ref` (span in transcript), `language_tags`, `asr_confidence`, `status` (`unconfirmed|doctor_confirmed|doctor_edited`), `scribe_version`. We accept unconfirmed entities for display only; checks on unconfirmed entities return `UNVERIFIED_INPUT`. Conformance: stub-based contract tests (kept from Stage A).

**Patient-context contract (inbound).** `snapshot_id`, `version`, `as_of`, `expires_at`, `source_system`, `age_band` (not exact DOB), `sex`, `pregnancy_status` (`yes|no|unknown`), `lactation_status`, `allergies[]` (coded or free text, each flagged `coded|uncoded`), `active_medications[]`, `renal/hepatic flags` (`present|absent|unknown`), `signature`. Missing → `unknown`, never default-false. Expired snapshot → dependent checks return `DATA_EXPIRED`.

**Outbound API.** `POST /review` (prescription → check results + coverage manifest), `POST /ask` (question → answer or abstention), `POST /correction`, `GET /coverage`. All responses carry `release_ids` (corpus, rules, drug master, prompt, model).

### S1.6 End-to-end doctor path and two worked examples (fictional)

Path: input → Scribe → context snapshot → normalise (clarify if ambiguous) → drug master → safety rail and/or reference path → composed output with states and citations → doctor review/edit → approve → save/export → audit.

**Medication-review path (fictional).** Scribe: "Fevex-P twice a day" and "Paracet 500", both `unconfirmed`. System asks one clarification chip: "Confirm: Fevex-P = paracetamol 650 mg? / Paracet 500 = paracetamol 500 mg?" (brand names are invented). After confirmation: duplicate-ingredient check → `HIT` (paracetamol in two lines), shown as an interrupting card: "Same ingredient on 2 lines. Total strengths as written: 650 mg + 500 mg per dose. Source: Drug Master release R-114." Interaction check for pair set → `PARTIAL_COVERAGE`: "Interactions checked only for 212 reviewed molecule pairs; this prescription includes 1 molecule outside coverage." Pregnancy check → `UNVERIFIED_INPUT` (pregnancy status `unknown`). Doctor edits, approves; export is English prescription PDF with generic names plus brands.

**Reference-question path (fictional).** Doctor asks "First-line for [condition] in adults?" Router classifies *consequential reference*. Structured lookup has no answer; retrieval filters to `population=adult`, specialty, live release; sufficiency gate passes only if a chunk states first-line text with scope match. Composed answer: two sentences paraphrasing the chunk with citation "[Source name, v2025-xx, §x, p.y]" and a "View exact quote" link. If no live chunk matches scope: "Cannot confirm. India sources in the corpus do not cover adult [condition] first-line therapy. Covered: [list]. Non-India source available (labelled) — show?"

### S1.7 Material departures from the pack (flagged)

| ▲ | Departure | Pack reference | See |
|---|---|---|---|
| 1 | General-model-knowledge tier removed for clinical content | P03 | D1, S6-4 |
| 2 | Overlay cannot override safety/regulatory results | Stage D, P03 | D9, S6-5 |
| 3 | `CHECKED_NO_HIT` only inside declared coverage manifest | Stage C/D, Law 5 | D6, S6-1 |
| 4 | `entity_status` from Scribe gates checks | Stage A contract | D6, S6-2 |
| 5 | Cumulative exposure limited to arithmetic totals as written; label-ceiling comparison disabled until legal/clinical approval | P02, Law 12 | D6, S6-3 |
| 6 | Replay = stored output + pinned versions, not regeneration; crypto-shred PHI | Stage E | D10, S6-6 |
| 7 | Gates keyed to coverage manifest, not only stage order | Gates 1–3 | S6-7, S7 |
| 8 | P07 mechanism specified | P07 | D5 |
| 9 | Governance register + post-market monitoring + incident path added | omitted | D12, S6-8 |

---

## S2 · Decision Register

Format: current → choice → why → strongest rejected alternative → accepted cost → validation.

| # | Area | Current → My choice | Why | Rejected alternative | Accepted cost | Validation |
|---|---|---|---|---|---|---|
| D1 | Grounding sources (P03) | Ladder kept (verified India corpus → tenant protocol, labelled → international, labelled non-India → "cannot confirm") ▲ but **tier 4 general model knowledge removed** for anything clinical; allowed only for non-clinical help (term explanation) and labelled | The "background" class is exactly where classifier errors leak a dose or safety claim (Law 2) | Keep P03 tier 4 with a consequential-vs-background classifier | More "cannot confirm"; weaker early helpfulness | Red-team set of consequential questions disguised as background; any ungrounded consequential claim = release blocker |
| D2 | Chunking mechanism | Keep ban on bulk token chunking. **Decision-unit chunks**: one recommendation with population, setting, form, exceptions, footnotes bound in; table rows chunked with repeated headers and units; parent path and cross-page edges stored | Fixed windows separate "adult dose" from "not in pregnancy" | 512-token overlapping windows | Slower, partly manual structuring; corpus grows by increment | Held-out scope test: every served chunk carries population, form, unit, version, location; citation-support check (S3.4) |
| D3 | Retrieval design (P01, P06) | Structured-first (kept). Then **metadata-filtered hybrid** (lexical + dense), filters applied *before* ranking (population, form, specialty, release); deterministic sufficiency gate extended with scope match | Relevance ≠ applicability; filtering prevents pediatric text answering adult question | Dense-only retrieval plus reranker | Metadata curation burden; abstains when metadata missing | Recall@k and applicability precision on clinician-keyed set, reported separately |
| D4 | Embedding/index | **Self-hosted multilingual embedding model, version pinned**; one immutable index per corpus release; embedding change = full re-evaluation | Laws 8 and 10; Hindi/transliterated queries | Hosted embedding API with auto-updates | Ops burden; possibly lower retrieval quality | Replay test: same query on same release returns identical chunk ids; multilingual recall slice |
| D5 | Drug-data strategy (P01, P02, P07, P08) | Graph + event-sourced regulation kept. **P07 mechanism ▲**: automated contradiction detectors (same brand, differing composition across sources; strength implausible for form; ingredient missing vs sibling brands; banned/restricted status vs live market listings; stock-vs-register for P08); no row becomes canonical without ≥2 independent agreeing sources or human review | A poisoned canonical row must be catchable | Trust one licensed brand master as canonical | Reviewer load; slower new-SKU onboarding | Seeded-poison set: 100% quarantined, 0 served |
| D6 | Safety-check boundary (P02, Law 4, 12) | Pure rules/lookup. **Inputs gated by `entity_status` ▲; `CHECKED_NO_HIT` only if both ingredients are inside the rule batch's coverage manifest ▲; otherwise `PARTIAL_COVERAGE`.** Cumulative exposure = arithmetic sum of strengths as written; **no label-ceiling comparison until legal/clinical approval ▲**; no weight-based or pediatric logic | A curated set (Stage D) makes "no rule" ≠ "safe"; summing near the dose fence needs an explicit line | Treat absence of a rule as no-hit; compare sums to ceilings by default | Many PARTIAL_COVERAGE early; less "helpful" exposure warning | All 8 states reachable; held-out out-of-coverage pairs never return CHECKED_NO_HIT |
| D7 | Model/provider posture | Frontier LLM **only** for Scribe (upstream) and cited composition; **provider-agnostic gateway**, pinned model id, no-training and no/limited-retention terms, de-identified minimum context, schema-validated output, uncited claims stripped | Law 10; vendor deprecation and lock-in; fine-tuning adds unreviewable behavior | Fine-tune a custom medical model | Vendor dependence; regression per model change | Pinned-version eval; golden-thread rerun on any model/prompt change |
| D8 | Data residency | PHI stored/processed in India-region infrastructure by default; LLM calls carry no name/phone/ID; egress allow-list | DPDP Rules allow transfer unless government restricts destinations [V]; hospital contracts/sector rules may be stricter [U] | Single global region | Fewer provider options; latency | PHI-in-logs scan; egress test; payload sampling |
| D9 | Multi-tenancy | Row-level tenant isolation + per-tenant keys; overlays **additive and labelled; cannot suppress or downgrade a regulatory or safety-rail result ▲**; tenant alias dictionary stays tenant-local until reviewed | A hospital protocol must not silence a banned-FDC hit; deployer scope changes alter risk | Tenant protocol outranks national default universally | Hospitals cannot customise safety output | Cross-tenant negative suite; overlay-vs-rail conflict tests |
| D10 | Audit | Append-only, hash-chained event log: inputs (references), chunk/rule ids, release ids, prompt/model ids, **stored output** ▲; PHI payload encrypted separately and crypto-shreddable | Law 8 vs LLM nondeterminism; DPDP erasure vs retention | Re-run the model to replay | Storage; legal review of retention | Byte-consistent replay drill; erasure drill preserving hash chain |
| D11 | Verification model (P05) | 100% qualified review for safety-rail content kept; **supported scope = reviewed batches (coverage manifest)**; backlog shrinks scope, never ships unreviewed (kept from human-lane rule) | Shrinking scope first is correct; the manifest makes it visible to doctors | Sampling-based review for rules | Narrow safety coverage initially | Release gate: manifest equals approved-batch list |
| D12 | Governance layer (omission) | Add an **AI register + change control + post-market monitoring + incident path**, structured on EU AI Act high-risk requirements as a design reference and NIST AI RMF functions as operating vocabulary ▲ | Evidence of ownership and traceability is what clients and regulators ask for; maps to controls already in the pack | Pursue ISO/IEC 42001 certification first | Documentation overhead; EU Act is not binding in India | Register completeness audit; quarterly mapping review |
| D13 | Alert governance (omission) | Two channels: **interrupting** (HIT, SOURCE_CONFLICT on severe items) vs **passive** (PARTIAL_COVERAGE, NOT_CHECKED, UNVERIFIED_INPUT shown in a coverage strip, never hidden) | Alert fatigue vs Law 5 | "High-severity only" default with unknowns hidden | Persistent coverage strip adds visual load | Acted/dismissed telemetry; usability test at 3–7 min consults |
| D14 | Corrections | One tap captures `{item, type, reason code, optional note, doctor, tenant, snapshot/release ids}` → candidate queue; tenant-local alias use immediately after doctor confirm; canonical change only after reviewer approval | Corrections can be wrong or malicious | Auto-apply corrections to truth | Delay before benefit spreads | Poisoned-correction test; queue age metric |

---

## S3 · Grounding & Sources Plan

### S3.1 Source register
Rights are separated: **Access** (may read), **Store**, **Extract**, **Index**, **Excerpt** (display quote), **Train** (model training; **none proposed**). All rights columns are [U] until each source's own terms or a written permission is read; a source with unrecorded rights is excluded from serving.

| Source | Purpose | Authority / coverage | Status | Limits |
|---|---|---|---|---|
| ICMR Standard Treatment Workflows | Treatment reference, Indian primary care/specialty | National; broad conditions; edition per ICMR | Free to read; reuse [U] | Not every specialty; editions |
| National programme guidelines (TB/NTEP, vector-borne/NCVBDC, immunization/UIP, maternal health) | National-programme questions | Authoritative within their domain | Government; reuse [U] | Circular-based updates; formats vary |
| National List of Essential Medicines (MoHFW) | Essential/formulary flag | National | Public [U] | Not clinical guidance |
| CDSCO notifications and Gazette of India | Regulatory events (bans, restrictions, stays) | Primary legal source | Public; each notification becomes a `RegEvent` | Litigation/stay status needs separate events |
| NPPA/DPCO orders | Price-control status | Official | Public | Frequent revisions |
| Jan Aushadhi (PMBJP) product list | Generic alternatives and prices | Programme list | Public | Not a regulatory source |
| Indian Pharmacopoeia (IPC) | Standards, nomenclature | National | Licensed [U] | Not therapeutic guidance |
| CDSCO-approved prescribing information / product labels | Label-based interaction and contraindication evidence | Primary for each product | Public/partial [U] | Coverage uneven |
| Specialty society guidelines (e.g., obstetric, paediatric, orthopaedic bodies) | Specialty depth | Society-level; may differ from national | Mixed rights [U] | Concurrent disagreement possible |
| Commercial brand/composition master | Brand→ingredient coverage | Not authoritative alone | **Licensed**, negotiation open [U] | Errors; subject to P07 checks |
| Tenant protocols/antibiograms | Local practice | Tenant-authoritative only for that tenant | Tenant-owned, contract | Never overrides rail |
| WHO guidance, WHO ATC/DDD (international tier) | Background, classification | International | ATC/DDD licence limits [U] | Non-India; labelled |
| Curated critical interaction/allergy/pregnancy set | Safety rail | Built from labels + guidelines under pharmacologist sign-off | **Curated** | Coverage limited; licensed database is an open scale-up decision |

### S3.2 Knowledge lifecycle
1. **Acquire:** record URL, retrieval date, hash, edition, rights note. 2. **Extract:** exact-quote binding + dual-extraction agreement; tables extracted row-wise with units. 3. **Review:** qualified clinician approves each decision unit (rail content 100%; reference content risk-stratified, see S7). 4. **Chunk** (D2). 5. **Version:** immutable `SourceRelease`; diff-first changelog "what changed, on which source, effective when". 6. **Publish:** only reviewed chunks enter the *live* tier. 7. **Retrieval eligibility:** live tier + population/form/specialty scope match + not withdrawn. 8. **Update:** new edition = new release; old retained, flagged `superseded` (P04). 9. **Withdraw:** a withdrawal event removes eligibility immediately, keeps a replay copy.

### S3.3 Retrieval and index defence
Structured data (identity, strength, price, regulatory status) never goes through retrieval (P01). Prose goes through hybrid retrieval because lexical match is needed for drug names and numeric tokens while dense match handles Hindi/transliterated and paraphrased queries. A reranker may order *already-eligible* chunks only. The index is per release so replay and rollback are cheap.

### S3.4 Chunk and citation example (fictional)
```
chunk_id: KS-0042   release: R-2026-10   tier: live
source: "Example National Guideline" edition 2025-03, §3.2, Table 4 row 2, p.14
population: adult, non-pregnant   form: tablet   specialty: general medicine
text: "<fictional recommendation text>"   unit: mg   exceptions: ["see footnote b, p.15"]
edges: [KS-0043 (pregnancy exception)]   reviewer: clinician-id   approved: date
```
Citation shown to doctor: *Example National Guideline (2025-03), §3.2 Table 4, p.14 — [view exact quote]*.

**Three separate validations.** (1) *Relevance:* does retrieval return the right chunk (recall@k). (2) *Applicability:* deterministic comparison of chunk scope with query/patient scope; a relevant but out-of-scope chunk is rejected (e.g., paediatric chunk, adult question). (3) *Citation support:* each generated claim must be entailed by its cited quoted span; exact-span/number match first, then clinician sample review. A passage can pass (1) and fail (2) or (3).

### S3.5 Conflicts, precedence, scope
Supersession ≠ conflict (P04 kept): newer edition serves, older flagged. Genuine concurrent disagreement → both shown side by side with dates and scope; no merged answer. Precedence: national programme guidance is shown first *within its programme domain*; this is a product rule to be ratified by the clinical lead [U], not a universal law. Local protocol shown as labelled overlay. International/background: shown only as labelled non-India, and the drug master states whether the drug is marketed in India.

### S3.6 Missing permissions and consequence
No recorded store/index/excerpt right → source excluded from serving → affected questions show "cannot confirm" and the capability is listed as *Not supported* on the coverage page (S8). No licensed comprehensive interaction database → interaction coverage limited to the curated set, with `PARTIAL_COVERAGE` for everything else. Any proposal to use content or logs for model training is **not part of this design**; it would need separate written rights and DPDP purpose analysis.

---

## S4 · Rules of Behavior for India

### S4.1 Rules (condition → behavior → reason → enforcement → verification/approval)

| # | Condition | Required / prohibited | Reason | Enforcement | Approval |
|---|---|---|---|---|---|
| R1 | Consequential question without verified, applicable, cited evidence | Must say "cannot confirm" and name what is missing | Law 2/3 | Output verifier strips uncited claims | Clinical lead |
| R2 | Request for patient-specific dose, weight-based or pediatric mg/kg, diagnostic ranking, triage score, treatment plan | Must refuse; may show cited reference text with scope, labelled "reference, not a recommendation for this patient" | Law 12; keeps intended use narrow | Intent classifier + response template; fence tests are blockers | Legal + clinical |
| R3 | Any check with incomplete coverage | Show the specific state; never "no issues" | Law 5 | State machine; UI contract tests | Pharmacologist |
| R4 | Scribe entity unconfirmed | `UNVERIFIED_INPUT`; ask one clarification | LLM-derived input | Contract field `entity_status` | QA |
| R5 | Ambiguous brand/strength/form | Queue with reason code; ask doctor; never auto-resolve | Law 6 | Resolver returns candidates, not a choice | Reviewer |
| R6 | Product appears in stock but a regulatory event restricts it | Show event status with scope, date, source; do not infer legality from stock | P08 | Inventory-vs-register job | Regulatory reviewer |
| R7 | Output would be shown to patient | Blocked: no patient-facing AI surface. Export limited to doctor-approved prescription text in English; optional pre-approved static phrase bank for patient-language label lines (e.g., "after food") stays **disabled until clinically approved** | Fence; patient-language need noted in brief | No patient session type; export template whitelist | Clinical + legal |
| R8 | Child or guardian data | Verifiable parent/guardian consent flow before processing, subject to clinical-establishment exemptions; no paediatric dosing | DPDP Rules: parental consent for children, limited exemptions for clinical establishments [V] | Consent gate; age-band flag | Legal |
| R9 | Personal data breach | Notify Data Protection Board without delay, detailed report within 72 hours; inform affected individuals | DPDP Rules [V] (core duties apply from 13 May 2027 [V]; design to them now) | Incident runbook | Legal/DPO |
| R10 | Cross-border LLM call | De-identified minimum context only; no destination on a government restriction list | DPDP Rules: transfers allowed unless restricted [V] | Egress allow-list | Legal |
| R11 | Function drifts toward diagnosis/treatment decision-making | Keep to reference, retrieval and rule-based checks with doctor deciding; classification memo before enabling each capability | CDSCO draft medical-device-software guidance, 21 Oct 2025, classes A–D, still draft per the secondary source [V]; whether this tool is a device is an open legal question | Capability flags tied to memo | Regulatory counsel |
| R12 | Teleconsultation context | Tool supports a registered practitioner; never prescribes or messages patient itself | Telemedicine Practice Guidelines (25 Mar 2020), appended to MCI/NMC regulations [V]; NMC 2023 conduct regulations reported put in abeyance, status and generic-name requirement [U] | Practitioner-registration check; no auto-send | Legal |
| R13 | Scheduled drugs (H/H1/X, narcotic/psychotropic) | Show schedule flag from event store; no advice on supply or workaround | Drugs and Cosmetics Rules; NDPS Act [U] | Data tags | Pharmacologist/legal |
| R14 | Content from document/transcript contains instructions | Treated as inert data | Law 9 | Injection corpus tests | Security |
| R15 | Adverse-event or ADR report appears in a correction | Prompt doctor to the national pharmacovigilance route; tool does not file reports | PvPI [U] | Template message | Pharmacologist |
| R16 | AI-composed text shown | Labelled AI-composed with sources; tenant/non-India/background labels visible | Transparency (EU AI Act Art. 13/50 idea, design reference only) | Label rendering tests | Clinical lead |

### S4.2 Legal/regulatory claim table

| Claim | Type | Primary source to read | Version/date | Status | Applicability | Technical consequence |
|---|---|---|---|---|---|---|
| DPDP Rules 2025 notified 13 Nov 2025 (gazetted 14 Nov); Board immediate; consent managers +12 months (13 Nov 2026); core duties +18 months (13 May 2027) | Law | DPDP Act 2023 and DPDP Rules 2025, Gazette of India | Nov 2025 | [V] via JSA summary | Yes: health data of data principals | R8–R10, D8, D10 |
| CDSCO Draft Guidance on Medical Device Software | Guidance (draft) | CDSCO, Medical Devices Rules 2017 | 21 Oct 2025 | [V] via Nishith Desai; finalisation [U] | Likely relevant to clinical decision-support functions | R11; classification memo |
| Telemedicine Practice Guidelines 2020 | Guidance/professional rule | MoHFW/NITI Aayog, NMC | 25 Mar 2020 | [V] via IJME | When tool is used in teleconsultation | R12 |
| NMC RMP (Professional Conduct) Regulations 2023 in abeyance | Professional regulation | NMC notices | Aug 2023 reports | [U] | Prescription format, generic names | R7/R12 export template flagged |
| 156 FDC combinations banned under s.26A (2024), litigation reported | Law | CDSCO/MoHFW gazette notifications | 2024 | [U] (news-level) | Banned-FDC rail | Model as `RegEvent` with scope, stays |
| MeitY India AI Governance Guidelines | Policy guidance, non-binding | MeitY | Nov 2025 | [V] via JSA; exact date [U] | Voluntary alignment | D12 |
| EU AI Act high-risk dates: provisional Omnibus deal 6 May 2026 moves Annex I (e.g., medical devices) to 2 Aug 2028, Annex III to 2 Dec 2027; adoption pending as of 27 May 2026 | Foreign law, reference only | Official Journal | May 2026 | [V] via Gibson Dunn; final status [U] | Not binding in India | D12 design reference |

Open items needing qualified reviewers before enabling: device classification (regulatory counsel), prescription/generic-name rules (legal), schedule data (pharmacologist), data-processing contract terms with LLM provider (legal/DPO).

---

## S5 · Correctness & Safety Design

### S5.1 Deterministic vs generated
| Deterministic (no model) | Generated (model) |
|---|---|
| Identity resolution outputs, FDC decomposition, regulatory status, price/alternative lookup, safety checks, state assignment, coverage manifest, scope filtering, sufficiency gate, citation matching | Scribe extraction (upstream); wording of a grounded answer from approved chunks |

Fence enforcement: no dose calculator exists in code; tests assert that no endpoint accepts weight and returns mg.

### S5.2 Check result record
`{check_id, type, state∈8, inputs[{ingredient_id, source(entity_status), snapshot_id}], rule_batch_id, rule_version, coverage:{covered_pairs, uncovered_ingredients[]}, evidence[{source, version, location}], message_key, computed_at, data_expiry}`.
**Incomplete-coverage example:** interaction check on 4 ingredients where 1 is outside the reviewed set → `PARTIAL_COVERAGE`, listing the uncovered ingredient, with the covered pairs' own results shown beneath. Never aggregated into a single "no issue".

### S5.3 Drug identity and FDCs
All checks run at ingredient level after FDC decomposition; same-ingredient totals across lines are shown as written. Ambiguity (brand collision, missing strength, form) → queue and clarification. Combination of unknown composition → `NOT_CHECKED`.

### S5.4 Missing, conflicting, stale, unavailable
| Situation | State / behavior |
|---|---|
| Patient field unknown (e.g., pregnancy) | `UNVERIFIED_INPUT`; checks needing it say so |
| Two sources disagree concurrently | `SOURCE_CONFLICT`; show both with dates; no merge |
| Rule/data past expiry | `DATA_EXPIRED` |
| Check not defined for the ingredient | `NOT_CHECKED` |
| Rail/knowledge service down | `SERVICE_UNAVAILABLE` banner; no cached "safe" |
| LLM down | Structured and rail results still served; Q&A shows degraded state |

### S5.5 Provenance and uncertainty
Every consequential claim carries `{source, version/date, location}` and a quote link. Uncertainty is stated in words and by state, never by a hidden confidence score.

### S5.6 Doctor workflow, corrections, approvals
Pending approvals list shows: unconfirmed entities, unresolved ambiguities, open `HIT`s. The doctor cannot "Approve prescription" while a mandatory clarification is open; approving with open `HIT` requires a one-tap acknowledgement with reason code (acknowledgement is logged, not a block). Corrections follow D14. Roles: doctor, front-desk (view/entry only), tenant admin (overlay, aliases), reviewer (queue), auditor (read-only). Short sessions and auto-lock on shared devices; no PHI persisted on the device.

### S5.7 Languages, offline, degraded
Input: mixed-language and transliterated; normaliser returns candidates, never final. Output: English plus source quotes; doctor-facing UI strings localisable. Offline: read-only drug master lookup from a signed cache with expiry; no safety results from an expired cache (`DATA_EXPIRED`); queued notes sync later; offline banner always visible. Latency target (aspirational): ≤2 s p95 for structured lookups and rail results; Q&A shows evidence first and streams the composed text; measured in live use before any commitment.

### S5.8 Access, multi-tenancy, audit
Tenant keys; role checks on every call; audit events for view, check, approve, correct, export; PHI separated and crypto-shreddable (D10).

---

## S6 · What You Added (ranked)

| Rank | Factor | Current section / omission | Consequence | Severity | Handling → |
|---|---|---|---|---|---|
| 1 | "No rule found" shown as no-hit under a curated set | Stage D vs Law 5 | Unknown rendered as safe | Critical | Coverage manifest → D6, S7 |
| 2 | LLM Scribe feeds a no-model check path | Stage A contract, Law 4 | Misheard drug yields confident check | Critical | `entity_status` → D6 |
| 3 | Exposure totals vs dose-calculation fence | P02 vs Law 12 | Silent fence breach, device-classification risk | High | Totals as written, ceiling off → D6, R2 |
| 4 | Tier-4 general knowledge | P03 | Consequential claim via classifier leak | High | Removed → D1 |
| 5 | Overlay outranks national default | Stage D | Tenant protocol can suppress safety/regulatory signal | High | Additive overlay → D9 |
| 6 | Replay vs LLM nondeterminism; audit vs erasure | Stage E, Law 8 | Replay unprovable; DPDP conflict | High | Stored output, crypto-shred → D10 |
| 7 | Gate 1 precedes verified rule content (D) | Gates | Rail serves before rule review | High | Gate on manifest → S7 |
| 8 | No governance owner, post-market monitoring or incident path | omitted | Drift and harm undiscovered | High | Register, monitoring → D12, S7, S8 |
| 9 | Alert fatigue vs unknown-state visibility | Stage E | Hidden unknowns or alert flood | High | Two channels → D13 |
| 10 | Corrections and alias dictionary can poison truth | Stage F, Stage C alias export | Wrong brand mapping spreads | High | Candidate queue; tenant-local aliases → D14 |
| 11 | Source rights unrecorded | "rights-cleared free sources" (B) unevidenced | Legal exposure; capability silently unavailable | High | Rights register, exclusion rule → S3.6 |
| 12 | Offline and shared devices only in Stage G | Stage G | Stale safety data, PHI on shared PC | Med-High | Signed cache, expiry, auto-lock → S5.7 |
| 13 | Status legend overstates readiness | 03 §2 (Stage B "VALIDATED" but nothing serves) | Leadership may read "validated" as ready | Medium | Separate "built" vs "validated for serving" → S8 |
| 14 | Review throughput at national scale | P05, human lane | Backlog | Medium | Usage-weighted queues; manifest shown → D11, S7 |
| 15 | Population fairness (language, specialty, pediatrics, pregnancy) unmeasured | §4 | Performance gaps hidden by averages | Medium | Slice metrics → S7 |
| 16 | Latency target unqualified | 02 §6 | Stage ambiguity | Medium | Per-stage target → S5.7 |

---

## S7 · Validation & QA Plan

### S7.1 Evaluation-set construction
Sources: (a) clinician-authored questions per specialty, population, language (EN + Hindi + at least the languages of the pilot sites, transliterated); (b) de-identified, consented prescribing exports from pilot doctors; (c) synthetic adversarial cases (poisoned rows, prompt-injection, out-of-corpus). Keys are written and approved by clinicians and a clinical pharmacologist, owned **outside engineering**; held-out split is chosen by hash, never shown to implementers; editing a test to pass = incident. Provenance recorded per case (author, approver, date, source release).

### S7.2 Capability test table

| Capability | Test data / reference | Method | Pass criterion | Evidence | Validator |
|---|---|---|---|---|---|
| Drug mapping | Held-out brand→salt key incl. misspellings/transliteration | Automated compare; ambiguous cases must queue | 0 silent wrong mappings; ambiguous routed 100% | Report + diff | QA; clinician co-sign |
| FDC decomposition | Key of FDC compositions | Exact match | 100% on key | Report | QA |
| Duplicate-ingredient | Synthetic + exported prescriptions | Rule test | No misses in key | Report | QA; pharmacologist |
| Banned/restricted FDC | Event-sourced scope/stay cases | Time-travel tests ("as of date") | Correct status incl. stays/reversals | Report | Regulatory reviewer |
| Interaction/allergy/pregnancy rails | Reviewed batches + out-of-coverage pairs | Rule test + state test | Key hits correct; out-of-coverage never `CHECKED_NO_HIT` | Report | Pharmacologist |
| 8-state contract | Fixtures for each state | UI/API contract test | All reachable, correctly rendered | Screenshots/logs | QA |
| Retrieval | Clinician-keyed question set | Recall@k, applicability precision | Recall and precision reported separately; scope violation = 0 in served answers | Report | QA; clinician |
| Grounded answers | Keyed answerable set | Claim-support verifier + clinician review of stratified sample | Zero unsupported consequential claims in blocker classes; overall confidently-wrong upper 95% bound ≤1% (≈300 error-free cases) [target to be ratified by clinical lead] | Annotated sample | Clinician |
| Abstention | Unanswerable set (out-of-corpus, wrong population) | Compare to key | See S7.3 | Report | Clinician |
| Prompt injection | Injection corpus | Red-team | 0 successful instruction following | Report | Security |
| Tenant isolation | Cross-tenant negative suite | Automated | 0 leaks | Report | Security |
| PHI in logs | Log scan | Scanner + sample | 0 PHI findings | Report | Security/DPO |
| Audit replay | Past outputs | Replay | Byte-consistent | Replay report | QA |
| Degraded modes | Fault injection | Chaos test | Visible degraded state, no silent fallback | Logs | QA |
| Latency | Staged load + live | p95 by stage | Meets stated target or documented exception | Dashboards | Engineering |
| Fence | Prompts asking dose/diagnosis/triage | Red-team | 100% refusal/reference-only | Report | Legal/clinical |

### S7.3 Metrics and denominators
- **Confidently wrong rate** = answered consequential-class questions where a clinician judges ≥1 claim wrong or unsupported ÷ all answered consequential-class questions. Blocker classes (dose, safety, regulatory status): must be 0.
- **Useful supported answer rate** = answered questions rated correct, applicable, fully supported and useful ÷ answerable questions in scope.
- **Abstention precision** = abstentions judged appropriate ÷ all abstentions; **abstention recall** = unanswerable questions correctly abstained ÷ all unanswerable; **over-abstention** = abstentions on answerable questions ÷ answerable questions. Reported by specialty, population, language.

### S7.4 Release gates (no average outvotes a catastrophe)
Each of these is individually blocking: missed trap in key, uncited consequential number, unknown rendered as safe, cross-tenant leak, unlabelled general-knowledge or non-India answer, PHI in logs, rail content unreviewed, supported scope ≠ approved-batch manifest. Gate positions: Gate 1 after first serving slice; Gate 2 after hardening; Gate 3 after supervised use with recorded clinical sign-off and the documented regulatory position for exactly what shipped (kept; plus manifest condition ▲).

### S7.5 Operating cadence (QA owner and reviewers)
**Daily:** automated golden-thread and state-contract run; PHI/log scan; review-queue age and ambiguity-queue age; incident log check; degraded-mode status. **Weekly:** stratified production sample (by specialty, language, population, state) reviewed against key by clinician; metric trends and drift thresholds; correction-inbox triage; deprecations. **On change:** mapped regression (below). **Quarterly:** governance register audit; EU/NIST mapping and source-rights review.

### S7.6 Scale without reviewing every answer
Automated gates handle 100% of traffic (citation match, scope filter, state rules). Humans review 100% of rail content and regulatory events (small, high-consequence), and a **risk-stratified sample** of answers: oversample new sources, low-retrieval-score, abstentions, doctor-dismissed or corrected outputs, and thin-coverage strata (rural languages, paediatrics, obstetrics). Corrections and override rates are the production signal; threshold breach pauses the affected capability.

### S7.7 Revalidation after change

| Change | Mapped regression |
|---|---|
| Model id / provider | Full eval + golden thread |
| Prompt | Grounding, abstention, injection, fence suites |
| Embedding model / index | Retrieval, replay |
| Source release | Diff review; scope/applicability tests; affected-question re-eval |
| Mapping (alias/brand) | Held-out mapping key; ambiguity routing |
| Rule batch | Pharmacologist review; coverage-manifest diff; pair tests |
| Regulatory event | Time-travel tests; banned-FDC key |

Clinical/legal approvals (pharmacologist sign-off, classification memo, source rights, consent operations) are separate records from engineering/QA results; neither substitutes for the other.

---

## S8 · Doctor-Handover Checklist

Rule: an item without its approval is **not done**. A capability either meets every row for its scope or stays disabled; the plan is to keep the maximum *validated* scope on, not to disable broadly.

| # | Item | Required result | Verification / evidence | Validator / approver | Handover blocker |
|---|---|---|---|---|---|
| 1 | Coverage matrix (specialty × population × language × capability) | Each cell = *Validated / Limited / Not supported*, shown in app | Matrix tied to eval reports | Clinical lead + QA | Any shipped cell without evidence |
| 2 | National availability vs validated coverage | Availability = who can log in; coverage = matrix; both visible | Coverage page | Product + clinical | Claiming "nationwide" without matrix |
| 3 | Rail content review | 100% of served rules approved | Reviewer sign-offs per batch; manifest = approved list | Clinical pharmacologist | Unreviewed rule served |
| 4 | Knowledge content | Served chunks have reviewer, scope, version | Release manifest | Clinical reviewer | Missing reviewer/scope |
| 5 | Source rights | Rights row per source and use (store/index/excerpt) | Rights register + permission copies | Legal | Source with unrecorded rights served |
| 6 | Regulatory events | Current as of release; stays/reversals modelled | Event diff report | Regulatory reviewer | Stale event feed |
| 7 | Feature-classification memo | Written position for exactly the shipped functions | Signed memo | Regulatory counsel | Memo missing or covers fewer functions |
| 8 | DPDP readiness | Notice/consent text, breach runbook, retention/erasure drill, processor contracts | Documents + drill record | Legal/DPO | Consent not staged in clinic |
| 9 | Pilot-doctor consent | Consent operations done before live use | Signed records | Operations | Live use without consent |
| 10 | Configuration | Tenant isolation, roles, session lock, egress allow-list | Config audit + tests | Security | Any failing negative test |
| 11 | Integration | Scribe and context contract tests green; degraded behaviors verified | Conformance reports | Engineering/QA | Contract test failing |
| 12 | Gate results | All blockers green | Gate report | QA; clinical sign-off recorded separately | Any blocker red |
| 13 | User instructions | Intended use, prohibited use, state meanings, how to correct, coverage page | Documents + usability check | Clinical lead | No instructions/training |
| 14 | Capability cards | One per capability: purpose, limits, failure modes, thresholds | Register entries | Product owner | Missing card |
| 15 | Support and incident | Runbook, escalation, pause switch, rollback rehearsed | Rehearsal log | Operations | Rollback not rehearsed |
| 16 | Monitoring | Dashboards and thresholds with named owners | Dashboard + owner list | QA owner | No owner |
| 17 | AI register | Every model/prompt/rule/corpus release has owner, version, approver | Register audit | Governance owner | Unregistered component |
| 18 | Operating evidence | Stable daily use, override/dismissal rates, correction queue age | Telemetry report | QA + clinical | Unreviewed signal spikes |
| 19 | Reviewer training/AI literacy | Reviewers and doctors trained; records kept | Training log | Operations | Untrained reviewer |
| 20 | Residency and provider terms | India-region storage; provider no-training/retention terms | Contract + infra evidence | Legal + engineering | Terms missing |

---

## S9 · AI-Usage Note

1. I used Claude to read the pack, draft the structure and tables, and search secondary sources for legal dates; I then reviewed every decision and changed the wording and priorities. [Naresh: confirm]
2. Checked/changed: the AI first suggested showing cumulative same-ingredient totals against label maximums; I changed it to arithmetic totals only because comparison to ceilings is close to the patient-specific dose fence. [Naresh: confirm this was your decision or replace with a real override you made]
3. Biggest uncertainty: whether CDSCO's medical-device-software guidance (draft at last check) classifies this tool as a device, and the current NMC prescribing/generic-name rules; both need qualified legal review.

**Sources used while drafting (secondary, to be replaced by primary documents):** JSA Prism DPDP Rules summary (Nov 2025); Nishith Desai on CDSCO draft guidance (31 Oct 2025); IJME letter citing Telemedicine Practice Guidelines; Gibson Dunn on EU AI Act Omnibus (27 May 2026); JSA Prism on India AI Governance Guidelines (Nov 2025).
