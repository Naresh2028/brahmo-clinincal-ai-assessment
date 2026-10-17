# BRAHMO draft: S2, S3, S4, S6 (working draft for Naresh to edit, verify and own)

> Status: DRAFT. Every legal/source claim is tagged [V] verified from a secondary source this session, or [U] unverified: check the primary document before submitting. Rewrite in your own words and defend each choice yourself.

---

## S2 · Decision Register

Format: current position/omission → choice → why → strongest rejected alternative → accepted cost → validation.

| # | Area | Current → My choice | Why | Rejected alternative | Accepted cost | Validation |
|---|---|---|---|---|---|---|
| D1 | Grounding sources (P03) | Ladder retained but **tier 4 (general model knowledge) removed for all clinical content**; allowed only for non-clinical help (e.g. explain a term) and labeled. Tier 2 (tenant protocols) shown as a labeled *overlay*, never overriding tier 1 for safety/regulatory claims. | A "background" class is where the classifier leaks; one misclassified dose question breaks Law 2. | Keep P03 as is with a classifier guard. | More "cannot confirm" answers; lower perceived helpfulness early. | Classifier red-team set: consequential queries disguised as background; blocker = any unlabeled/ungrounded consequential claim (S7). |
| D2 | Chunking (P05, "bulk token-chunking banned") | Retain the ban. Chunk = **decision unit**: one recommendation + its population, setting, dose-form, exceptions, footnotes, table row with header and unit carried inline, plus parent-section path. Tables chunked per row with column headers repeated. Cross-page links stored as explicit edges. | Token windows split "adult dose" from "not in pregnancy". | Fixed-size 512-token overlapping windows (cheap, scalable). | Manual/assisted structuring per document; slower corpus growth. | Held-out scope test: every chunk must carry population+form+units; citation-support check (S3.5). |
| D3 | Retrieval | Structured-first (P01 retained) then **hybrid lexical + dense with metadata filters (population, form, specialty, source version) applied before ranking**, followed by a deterministic evidence-sufficiency gate (P06 retained, extended with a scope-match check). | Scope filtering prevents a pediatric passage answering an adult question; relevance alone is not applicability. | Dense-only retrieval with a reranker. | Metadata curation burden; some recall loss when metadata missing (system abstains). | Retrieval recall@k *and* applicability precision measured separately on the clinician-keyed set. |
| D4 | Embedding / index | Self-hosted multilingual embedding model pinned by version + lexical index; index built per **published corpus release** (immutable, tagged). Embedding model change = full re-eval. | Law 8/10: replayability and no silent upgrades; Indian-language and transliterated queries. | Hosted embedding API (simpler, better quality). | Ops burden; possibly lower quality than frontier embeddings. | Replay test: a past query on a past release returns identical chunk ids. |
| D5 | Drug-data strategy (P01, P02, P07, P08) | Retain graph (alias→brand→salt, FDC decomposition, event-sourced regulation). Add: **P07 mechanism**: automated contradiction checks (same brand with differing composition across sources, strength implausible for form, ingredient missing vs sibling brands, banned status vs live market listings) that route to the ambiguity queue; no source row becomes canonical without ≥2 independent agreeing sources or human review. | A poisoned canonical row must be catchable (P07). | Trust a single licensed brand database as canonical. | Review queue load; slower onboarding of new SKUs. | Seeded-poison test set: inject N corrupted rows; require 100% quarantined, 0 served. |
| D6 | Safety-check boundary (P02 + fence) | Checks are pure rules over versioned data (Law 4). Scribe output is LLM-derived, so **every check input carries `entity_status` (confirmed by doctor / unconfirmed)**; unconfirmed input yields `UNVERIFIED_INPUT`. `CHECKED_NO_HIT` only when both drugs fall inside the rule set's declared coverage manifest; otherwise `PARTIAL_COVERAGE`. Cumulative exposure = arithmetic sum of prescribed strengths vs a cited **adult label maximum**; no weight-based or pediatric logic. | Curated critical set (Stage D) makes "no rule found" ≠ "safe". Summing across FDCs sits near the dose-calculation fence, so the boundary is stated explicitly. | Treat absence of a rule as no-hit. | Many PARTIAL_COVERAGE results at first; doctors see "not fully checked". | All 8 states reachable; held-out pairs outside coverage must never return CHECKED_NO_HIT. |
| D7 | Model/provider posture | Frontier LLM only for retrieval-grounded composition and Scribe; **provider-agnostic gateway**, pinned model version, no training on customer data, zero/limited retention contract, India-region or contractual cross-border terms. Output schema validated; claims without a citation id are stripped. | Law 10; DPDP cross-border is permitted unless restricted [V]; vendor lock-in and deprecation risk. | Fine-tune a custom medical model. | Dependence on vendor; regression run per model change. | Pinned-version eval; golden-thread rerun on every change. |
| D8 | Data residency | Store and process PHI in India-region infrastructure by default; LLM calls send **minimum necessary, de-identified context** (no name/phone/ID). | DPDP Rules permit transfer except to restricted countries [V]; sector rules and hospital contracts may be stricter [U]. | Single global region. | Fewer provider options; latency. | PHI-in-logs scan; egress allowlist test; payload sampling. |
| D9 | Multi-tenancy | Tenant-scoped row-level isolation + per-tenant keys; overlays are additive and labeled; **overlay cannot suppress or downgrade a regulatory or safety-rail result**. | A hospital protocol must not turn a banned-FDC hit into silence. | Tenant overlay outranks national default universally (current Stage D). | Hospitals cannot fully customize safety. | Cross-tenant negative suite; overlay-vs-rail conflict tests. |
| D10 | Audit | Append-only, hash-chained event log of inputs, retrieved chunk ids, rule versions, prompt/model ids, **stored generated output** (replay = stored artifact + pinned versions, not regeneration). PHI separated and encrypted; erasure handled by crypto-shredding the PHI payload while retaining the hashed skeleton. | Law 8 vs LLM nondeterminism; DPDP erasure vs audit retention. | Re-run the model to "replay". | Storage; legal review of retention. | Byte-consistent replay test; erasure drill. |
| D11 | Review model (P05) | Retain 100% human review for safety-rail content, but add **coverage manifest**: supported scope = only reviewed rule batches; backlog shrinks scope, never ships unreviewed (retained from §4 human lane). | Matches D6. | Sampling-based review. | Narrow safety coverage until reviewed. | Release gate: manifest equals approved-batch list. |

---

## S3 · Grounding & Sources Plan (research notes)

Rights column: access ≠ permission. Each source needs separate answers for store / extract / index / display excerpt / train. All rights unverified [U] until read in the source's own terms.

| Source | Purpose | Status | Key limitation |
|---|---|---|---|
| ICMR Standard Treatment Workflows | Indian treatment references | Free to read; reuse terms [U] | Editions; not all specialties |
| National programme guidelines (NTEP/TB, NCVBDC, NHM/immunization, maternal health) | National-programme answers | Government publications; reuse terms [U] | Format variety; updates by circular |
| National List of Essential Medicines | Essential/formulary status | Government; [U] | Not clinical guidance |
| CDSCO notifications / Gazette of India | Regulatory events (bans, restrictions) | Official; feed into effective-dated event store | Stays/litigation need separate events |
| NPPA / DPCO price orders | Price control | Official | Frequent revisions |
| Indian Pharmacopoeia | Drug standards | Licensed/purchase [U] | |
| Jan Aushadhi product list | Generic alternatives | Public list | Not a regulatory source |
| Commercial brand databases | Brand/composition master | Licensed, rights negotiated [U] | Not to be sole canonical truth |
| International (WHO, others) | Tier-3, labeled non-India | Terms vary [U] | Not India-validated |

Reported (secondary sources): the Government banned 156 FDCs under section 26A in 2024 and the ban was challenged in the Delhi High Court [U: confirm gazette notification numbers, dates, scope and any stays from the primary notification]. This is the exact case the event-sourced store must model (scope, effective date, stay, supersession).

**Lifecycle:** acquire (record URL, hash, licence) → extract (exact-quote binding + dual extraction agreement) → clinician review → decision-unit chunking → immutable release version → publish only if reviewed → retrieve only from live release → update creates new release; old retained flagged → withdrawal event removes retrieval eligibility but keeps replay copy.

**Chunk example (fictional):**
`chunk_id: X-0042 | source: "Example Guideline" v2025-03 p.14 §3.2 table 4 row 2 | population: adult, non-pregnant | form: tablet | claim: "<fictional dose text>" | unit: mg | exceptions: [see p.15 footnote b] | edges: [X-0043]`

**Validation separate from relevance:** (1) retrieval relevance = does the chunk match the query; (2) applicability = deterministic check chunk scope vs patient/query scope; (3) citation support = each generated claim must be entailed by its cited quote (exact-span check + human sample). A passage can be relevant but inapplicable (pediatric) or unsupportive.

**Conflicts:** supersession ≠ conflict (P04 retained); concurrent disagreement shown side by side with dates; national programme precedence only within its domain, as a product rule to be confirmed by clinical reviewer [U].

**Missing permissions consequence:** any source without recorded store/index/excerpt rights is excluded from serving; affected capability shows "cannot confirm" and is listed on the coverage page.

---

## S4 · Rules of Behavior for India

| Condition | Required / prohibited | Reason | Enforcement | Approval needed |
|---|---|---|---|---|
| Doctor asks a consequential question with no verified, applicable, cited evidence | Must say it cannot confirm and name what is missing | Law 2 | Output validator strips uncited claims | Clinical lead |
| Request for patient-specific dose, weight-based dose, diagnosis ranking, triage, treatment plan | Must refuse; may show cited reference text with scope | Fence (Law 12); also limits device classification risk | Intent classifier + output template; fence tests are blockers | Legal + clinical |
| Any check without full coverage | Show state PARTIAL_COVERAGE/NOT_CHECKED, never "safe" | Law 5 | State machine; UI tests | Clinical pharmacologist |
| Prescription entities unconfirmed by doctor | Show UNVERIFIED_INPUT; no check result relied upon | Scribe is model-derived | Contract field `entity_status` | Engineering + QA |
| Child patient data | Guardian consent flow before processing; no pediatric dosing | DPDP Rules require verifiable parental consent for children, with limited exemptions for clinical establishments [V via JSA summary]; confirm applicability | Consent gate; age flag | Legal |
| Personal data breach | Notify Board without delay, detailed report within 72 hours; inform individuals | DPDP Rules [V via JSA summary] | Incident runbook | Legal/DPO |
| Software claims diagnosis/treatment function | Treat as potential medical device software; keep intended use to reference, retrieval and rule-based checks, doctor decides | CDSCO draft guidance on medical device software dated 21 Oct 2025, classes A–D, still a draft per a 31 Oct 2025 secondary source [V/U: check if finalised since] | Feature-classification memo gates release | Qualified regulatory counsel |
| Stock listed but product banned/restricted | Show regulatory event status with scope and dates | P08; stock ≠ legality | Inventory vs event cross-check | Regulatory reviewer |
| Telemedicine context | Product supports registered practitioner only; does not itself prescribe | Telemedicine Practice Guidelines 2020, added as Appendix 5 to the 2002 conduct regulation [V via IJME letter]; NMC 2023 conduct regulations reported put in abeyance [U: confirm current status and generic-name requirement from NMC/gazette] | Practitioner-registration check; no auto-send | Legal |
| Schedule H/H1/X, NDPS drugs | Show schedule flag from event store; no prescription generation or advice on supply | Drugs and Cosmetics Rules schedules [U: confirm current text] | Data tags | Pharmacologist |
| Document/transcript contains instructions | Treat as inert data | Law 9 | Injection corpus tests | Security |

Dates to record in final submission: DPDP Rules notified 13 Nov 2025 (gazette 14 Nov); Board immediately; consent managers after 12 months (13 Nov 2026); core duties after 18 months (13 May 2027) [V via JSA summary; confirm against the Gazette]. Note that today (17 Oct 2026) core duties are not yet in force, but design to them.

---

## S6 · What You Added (ranked)

| Rank | Factor | Current section/omission | Consequence | Severity | Handling (→ S2) |
|---|---|---|---|---|---|
| 1 | "No rule found" rendered as no-hit under a curated critical set | Stage D + Law 5 interplay | Unknown shown as safe: catastrophic | Critical | D6 coverage manifest |
| 2 | LLM Scribe feeds a no-LLM check path | Contract A, Law 4 | Misheard drug yields confident check | Critical | D6 `entity_status` + doctor confirmation |
| 3 | Cumulative exposure vs dose-calculation fence | P02 | Silent breach of Law 12 | High | D6 boundary |
| 4 | Tier-4 general knowledge in ladder | P03 | Consequential claim via classifier leak | High | D1 |
| 5 | Overlay outranking national default | Stage D | Tenant can suppress safety/regulatory signal | High | D9 |
| 6 | Replay vs LLM nondeterminism; audit vs erasure | Stage E, Law 8 | Replay claim unprovable; DPDP conflict | High | D10 |
| 7 | Gate order: Gate 1 (after C) precedes verified rules (D) | §3 gates | Rail serving before rule content reviewed | Medium-High | Gate on coverage manifest, not stage |
| 8 | Alert-fatigue defaults vs unknown-state display | Stage E | Hidden unknowns or alert flood | Medium-High | Two-channel display: interrupting vs passive |
| 9 | Status legend inconsistency (Stage B "VALIDATED" but serving nothing) | 03 §2 | Overstated readiness | Medium | Separate "built" from "validated for serving" in S8 |
| 10 | Offline/shared devices | Stage G only | Stale data, PHI left on shared PC | Medium-High | Cache with expiry→`DATA_EXPIRED`; session lock; no PHI at rest on shared device |
| 11 | Mixed-language and transliterated input | Stage G | Misresolved brand | Medium | Ambiguity queue; language-specific eval slices |
| 12 | Who accepts corrections as truth | One-tap loop (F) | Poisoned corrections | Medium | Correction = candidate, queued for reviewer |
| 13 | Review throughput at national scale | P05, human lane | Backlog; shrinking scope | Medium | Priority by usage weight; coverage manifest visible to doctors |

---

Items to do yourself: S1 diagram, S5, S7, S8, S9; read primary sources for every [U]; record opening/submission times; keep total ≤15 pages.
