# Your AI Governance course → BRAHMO assessment map

Source: your course notes (IAPP AIGP-style). Lecture numbers refer to your file. I read lectures 54, 62, 63, 66, 71, 94, 99, 100, 101 and 112 closely. I did not read the rest in detail. Paraphrased, not copied. Use these as *inspiration for your own wording*.

## A. Roles: who is who (lectures 43, 90, 94)

| Role in the course | In BRAHMO | Consequence for your design |
|---|---|---|
| Provider/developer | BRAHMO (builds the AI layer) | Owns documentation, testing evidence, instructions for use, change control, quality management |
| Deployer | Clinic / hospital tenant | Must use within intended scope, train doctors, monitor, escalate; cannot hand responsibility to the vendor |
| User | The doctor | Must be able to understand, challenge and override outputs |

Use in: **S2 (multi-tenancy row)**, **S8 (handover: instructions for use, tenant duties)**.
[K] In the EU Act a deployer that changes a system's intended purpose can become a provider. Check the Act text before using this. It supports "tenant overlay must not change safety scope" (D9).

## B. Course principle → where it goes in your submission

| Course principle (lecture) | Idea in plain words | Put it in | Concrete mechanism to write |
|---|---|---|---|
| Meaningful human oversight (94, 100) | Human can understand, challenge, override; works under workload; has review triggers, escalation and pause powers; trained reviewers | S5, S8 | Doctor approval event; one-tap override; pause switch per capability; reviewer checklist; tested with 60–100 patients/day in mind (3–7 min) |
| Transparency as usable information (100) | Tell deployers purpose, limits, what NOT to use it for; not "explain the algorithm" | S8 | Instructions for use + intended-use / prohibited-use statement + coverage manifest |
| Notice that AI is involved (100, Art. 50 idea) | Users must know content is AI-generated | S4, S5 | Label AI-composed text; label non-India or general-knowledge content |
| Documentation as a living control, decision records (63, 99) | Record why decisions were made, approvals, exceptions; update on every significant change | S2 (the register *is* decision records) | Each S2 row = a decision record; tie doc update to release |
| Record-keeping to reconstruct events, with limited sensitive data exposure (99) | Logs show inputs, versions, approvals; design logs so PHI exposure is controlled | S2 audit row (D10) | Hash-chained log; PHI separated/encrypted |
| Data provenance & lineage (66) | Source, permissions, transformations, dataset versions; for retrieval systems: author and last-updated date; review triggers for new source/schema change | S3 | Source register with rights per use; corpus release IDs; triggers for re-review |
| Model cards & conformity documentation (71) | Purpose, limits, failure modes, monitoring thresholds, prohibited uses, approvals | S8, governance register | One "capability card" per supported capability (e.g., FDC check, reference Q&A) |
| Monitoring beyond accuracy, drift, thresholds, owners (62) | Watch override rates, escalations, complaints, unsafe output spikes, subgroup performance, latency; drift includes vendor updates, prompt changes, new retrieval content | S7 | Production sampling; metrics + thresholds that trigger review; named owner who can pause |
| Change control as mini-deployment (62) | Data, prompt, model and vendor changes tested before users see them | S2 (D4, D7), S7 | Mapped regression per change type (already in their plan; you add thresholds + rollback) |
| Incident plan beyond security breaches (62) | Harmful outputs and unsafe recommendations count as incidents; contain, communicate, root-cause, update tests | S8 | Incident taxonomy incl. "wrong clinical output"; runbook; link to DPDP 72-hour breach duty |
| Risk scoring: probability × severity (53) | Prioritise risks by likelihood and harm | S6 | Severity column already required; define scale once (Critical/High/Med) |
| Impact assessment: who is harmed, how, what mitigations (59, 99) | Assessment before scaling; document residual risk | S4, S8 | Short impact assessment per capability and population (pediatric, pregnant, elderly) |
| NIST 7 trustworthy characteristics & 4 functions (54) | Valid/reliable, safe, secure/resilient, accountable/transparent, explainable, privacy-enhanced, fair; Govern/Map/Measure/Manage | S5, S7 | Use as headings in the governance table; "fair" → performance slices by language, specialty, population |
| Failsafe when uncertain (54) | System should not confidently push harm when outside limits | S5 | Maps directly to their 8-state contract and abstention |
| Prompt injection / exfiltration resilience (54) | Retrieval sources and prompts need protection | S5 | Their Law 9, add tests as release gates |
| Quality management system (100) | Repeatable process for risk, documentation, testing, change control, corrective actions | S7, S8 | QA owner + reviewers run the loop; corrective action log |
| AI literacy (91) | Staff/users trained to use responsibly | S8 | Doctor onboarding module; reviewer training record |
| Enforcement readiness: AI inventory + risk classification first (101) | Know what AI systems exist and their risk class; evidence ready before asked | S2/S6 | AI register (models, prompts, rule sets, corpus releases) with owner and risk class |
| ISO 42001 PDCA loop (112) | Plan–Do–Check–Act, internal audits, management review | S7 | Quarterly governance review as the "Check/Act" step (optional later certification) |
| Stop-and-escalate for prohibited uses (101) | Do not mitigate prohibited use with a disclaimer | S4 | Fences are hard refusals, not disclaimers |

## C. Cautions

1. **Exam heuristics ≠ law.** The notes say things like "assume high risk" for exam scenarios. Don't present that as BRAHMO's legal classification. Frame the EU Act as a *design reference*.
2. **Fines (35M EUR / 7%) apply to prohibited practices only (lecture 101).** Don't cite them for BRAHMO.
3. **Your notes skip deployment lessons 72–87** and contain no India or medical-specific content. India and medical rules must come from the primary sources in my earlier draft (DPDP Rules, CDSCO draft guidance, Telemedicine guidelines, MeitY guidelines).
4. Parts of your file are AI-generated chat residue (e.g., "I can continue with Section 5…"). Ignore them.

## D. Three lines you could adapt for S9 (only if true)

- Used AI to draft and cross-check; verified legal claims against primary sources.
- Example of a real check: [pick one you actually verified, e.g., EU AI Act high-risk dates after the Omnibus].
- Remaining uncertainty: [e.g., whether CDSCO's software guidance classifies this tool as a medical device].
