# 50 interview Q&A from my BRAHMO submission

How to use: read the question, answer aloud in about 60–90 seconds, then compare with the model answer. These answers are written in first person but are **drafts**: rewrite them in your own words, and drop any point you cannot explain. Never claim experience you don't have. Legal points tagged [V] were checked only against secondary sources; say so if asked.

---

## A. Big picture (1–6)

**1. Summarise your solution in 60 seconds.**
A clinician-facing layer for Indian doctors that does three things: turns brand names into ingredients, runs rule-based medication-safety checks, and answers reference questions from versioned, cited Indian sources. An LLM is only used to word answers from already-retrieved, cited evidence. It never sits in a safety check, never calculates a patient's dose, and never talks to patients. The doctor always approves, and everything is audited.

**2. What problem were they really asking you to solve?**
Not "build a chatbot", but "decide how to make a clinical AI trustworthy at national scale". The grading was on decisions: what to keep, change or add, with a rejected alternative and an accepted cost for each.

**3. What did you keep from their design, and why?**
Structured-first routing (identity, price and regulatory status come from data, not generation), FDC decomposition before any check, supersession is not conflict, the evidence-sufficiency gate, and 100% human review for safety-rail content. I kept them because each protects a fixed law, and I defended them as my own choices.

**4. What did you change?**
I removed the general-model-knowledge tier for clinical content, stopped tenant overlays from overriding safety results, made "no hit" depend on a coverage list, gated checks on whether the Scribe's entities were confirmed, limited cumulative exposure to totals as written, and changed replay to use stored outputs.

**5. What did you add that they hadn't thought of?**
A governance layer (AI register, change control, post-market monitoring, incident path), a two-channel alert design, a correction inbox that doesn't turn corrections into truth, a source-rights register, and a coverage matrix separating availability from validated coverage.

**6. Walk me through the doctor path end to end.**
Input (voice/text) → Scribe entities and a patient-context snapshot → normaliser (clarify if ambiguous) → drug master → safety rail and/or reference Q&A → results with states and citations → doctor reviews, edits, approves → save/export → audit log.

---

## B. Safety design (7–18)

**7. Why can't an LLM be in the safety-check path?**
Checks must be repeatable, explainable and testable. A model can be wrong without warning. So checks are pure rules over versioned data. That is a fixed law. The model's inputs are still controlled, which is why unconfirmed entities give `UNVERIFIED_INPUT`.

**8. List the eight check states and why they exist.**
HIT, CHECKED_NO_HIT, PARTIAL_COVERAGE, UNVERIFIED_INPUT, NOT_CHECKED, DATA_EXPIRED, SOURCE_CONFLICT, SERVICE_UNAVAILABLE. They exist so missing data never looks like "safe".

**9. What is the most important safety insight in your answer?**
"No rule found" is not "safe". With a curated rule set, a pair that isn't in the set was simply not checked. So `CHECKED_NO_HIT` is allowed only when both ingredients are inside the reviewed batch's declared coverage. Otherwise the result is `PARTIAL_COVERAGE`.

**10. Give an example of PARTIAL_COVERAGE.**
Four ingredients, one outside the reviewed set. The interaction check shows the results for covered pairs and says the one ingredient was not checked. It is never rolled up into "no issues".

**11. How do you stop Scribe errors becoming safety errors?**
Scribe is model-derived. Every entity has `entity_status` (unconfirmed until the doctor confirms). Checks on unconfirmed input return `UNVERIFIED_INPUT`. A clean result is downgraded when any line was excluded, while a real `HIT` is never hidden.

**12. How do you handle brand ambiguity?**
Never auto-resolve. Fuzzy matching only suggests. Ambiguous or unmatched text goes to a queue with a reason code and the doctor sees a clarification prompt. A held-out mapping key must show zero silent wrong mappings.

**13. Why decompose combination (FDC) products first?**
Indian prescribing is combination-heavy, and the same ingredient can hide in two brands. All logic runs at ingredient level so duplicates and interactions are found.

**14. How do you treat cumulative exposure given the "no dose calculation" fence?**
I show the arithmetic total of strengths as written. I do not compare it against label maximums, use weight, or do paediatric logic. The ceiling comparison stays off until legal and clinical approval, because it is close to patient-specific dose calculation.

**15. How is regulatory status modelled?**
As effective-dated events (notification id, dates, source, scope, supersession, stays), never a `banned` boolean. Status is the latest event as of a date. In my demo project a combination goes prohibited, then stayed, then prohibited again.

**16. What happens when a service or data is stale or down?**
Visible states: `DATA_EXPIRED` for stale rules, `SERVICE_UNAVAILABLE` for a down rail, and a degraded banner if the LLM is down while structured results still work. There is no silent fallback to a cached "safe".

**17. How do you prevent alert fatigue without hiding unknowns?**
Two channels. Interrupting: HIT and severe conflicts. Passive: partial coverage, not checked and unverified input, shown in a permanent coverage strip. I'd validate it with acted-vs-dismissed telemetry and usability tests at 3–7 minute consultations.

**18. What does the doctor do with an open HIT?**
They can approve with a one-tap acknowledgement and a reason code. It's logged but doesn't block, because the doctor decides. Open clarifications, though, do block approval.

---

## C. Sources, RAG and retrieval (19–29)

**19. Why not just use a frontier LLM with good prompts?**
Consequential claims need verifiable provenance. A general model can sound right and be wrong. I use it for wording only, after retrieval, and strip any claim without a citation id.

**20. Why did you remove the general-knowledge tier?**
The "background" class is where a classifier mistake leaks a dose or safety claim. Accepted cost: more "cannot confirm" answers early.

**21. What is a decision-unit chunk?**
One recommendation with its population, setting, form, exceptions and footnotes bound together. Table rows keep headers and units. Fixed token windows can separate "adult dose" from "not in pregnancy".

**22. Describe your retrieval design.**
Structured-first for identity, price and status. For prose: metadata filters first (population, form, specialty, release), then hybrid lexical plus dense search, then an optional reranker on eligible chunks only, then a deterministic sufficiency gate.

**23. Why hybrid search?**
Lexical matching is needed for drug names and numbers; dense retrieval handles paraphrase and Hindi or transliterated queries.

**24. Why self-hosted, version-pinned embeddings?**
Replayability and no silent upgrades. A hosted model that updates itself would change retrieval without a re-evaluation. Cost: operating burden and possibly lower quality.

**25. Relevance, applicability and citation support: what's the difference?**
Relevance: is it the right passage. Applicability: does its scope match the question (adult vs child). Citation support: does the cited text actually entail the claim. A chunk can be relevant but inapplicable or unsupportive, so each is tested separately.

**26. How do you handle conflicts between sources?**
A newer edition supersedes the old one (old kept, flagged). Real concurrent disagreement is shown side by side with dates and scope, never merged. National-programme priority within its domain is a product rule to be ratified by the clinical lead.

**27. Which sources did you recommend?**
ICMR Standard Treatment Workflows, national programme guidelines, NLEM, CDSCO notifications and the Gazette, NPPA orders, the Jan Aushadhi list, the Indian Pharmacopoeia, product labels, specialty society guidelines, a licensed brand master, tenant protocols, and WHO as a labelled international tier.

**28. Access vs permission: why does it matter?**
Being able to read a source doesn't mean you may store, extract, index or quote it. I recorded rights per use. A source with unrecorded rights is excluded, and the affected capability shows "not supported". I propose no training use.

**29. What if you have no licensed interaction database?**
Coverage is limited to the clinician-reviewed curated set, and everything else shows `PARTIAL_COVERAGE`. Licensing is an open scale-up decision, not something to hide.

---

## D. India, law and governance (30–38)

**30. What law applies to patient data?**
The DPDP Act 2023 and Rules 2025. The Rules were notified 13 Nov 2025; consent managers take effect after 12 months and core duties after 18 months (13 May 2027) [V, secondary source]. I'd confirm against the Gazette.

**31. How does DPDP shape your design?**
Parental consent for children's data, breach notification (Board without delay, detailed report within 72 hours), de-identified minimum data to model providers, India-region storage by default, and erasure handled by crypto-shredding.

**32. Is the tool a medical device in India?**
Open question. CDSCO issued a draft medical-device-software guidance on 21 Oct 2025 with classes A–D [V, secondary]; finalisation unverified. I keep functions to reference, retrieval and rule-based checks with the doctor deciding, and require a classification memo from regulatory counsel before enabling each capability.

**33. What about telemedicine and professional rules?**
The Telemedicine Practice Guidelines 2020 apply to registered practitioners [V]. The tool never prescribes or messages patients. The NMC 2023 conduct regulations were reported put in abeyance; their current status and the generic-name rules are unverified, so prescription export is flagged for legal review.

**34. "A disclaimer says the doctor decides": is that enough legally?**
No. Doctor supervision and disclaimers don't settle device classification or accountability. That's why the fences limit functions and why counsel must sign off.

**35. Why did you add a governance layer?**
Clients and regulators ask for evidence of ownership and traceability. The pack already had most controls but no AI register, owners, post-market monitoring or incident path. I structured it on the EU AI Act's high-risk requirements as a design reference, with NIST AI RMF as operating vocabulary.

**36. Is the EU AI Act binding for BRAHMO?**
Not in India, and I don't claim compliance. A provisional Digital Omnibus deal (6 May 2026) would move medical-device high-risk dates to 2 Aug 2028, but adoption was pending in my source [V, secondary]. I used it as a reference.

**37. What does India have for AI governance?**
MeitY's AI Governance Guidelines (Nov 2025): seven principles, no new AI law, sector regulators leading [V, secondary]. Non-binding. So my claim is "no binding medical-AI-specific regime that I could verify", not "no rules".

**38. Why not ISO/IEC 42001 first?**
Certification is overhead before the product is validated. I kept it as an optional later step. Accepted cost: less external proof now.

---

## E. Validation and QA (39–45)

**39. How do you build the evaluation set?**
Clinician-authored questions by specialty, population and language; de-identified, consented prescribing exports; adversarial synthetic cases. Keys are approved by clinicians and a pharmacologist and owned outside engineering. The held-out split is by hash and never shown to implementers.

**40. Define "confidently wrong".**
Answered consequential-class questions where a clinician finds at least one wrong or unsupported claim, divided by all answered consequential-class questions. In blocker classes (dose, safety, regulatory status) it must be zero.

**41. How can you set a pass level?**
Zero in blocker classes, and an overall upper 95% bound of 1% (about 300 error-free cases). That target is my proposal for the clinical lead to ratify.

**42. How do you measure abstentions?**
Precision (appropriate abstentions ÷ all abstentions), recall (unanswerable questions correctly abstained ÷ all unanswerable) and over-abstention (abstained on answerable ÷ answerable).

**43. What blocks a release?**
Any one of: a missed trap, an uncited consequential number, an unknown shown as safe, a cross-tenant leak, an unlabelled general-knowledge answer, PHI in logs, unreviewed rail content, or supported scope not matching the approved manifest. No average outvotes a catastrophe.

**44. How do one QA person and reviewers cover all-India scale?**
Automation covers 100% of traffic (citation match, scope filter, state rules). Humans review 100% of rail content and regulatory events, plus a risk-stratified sample of answers (new sources, abstentions, corrected outputs, thin strata like rural languages and paediatrics). Override and correction rates trigger a pause.

**45. What changes trigger revalidation?**
Model, prompt, embedding, source release, mapping, rule batch or regulatory event. Each maps to a specific regression suite, with clinical and legal approvals recorded separately from engineering results.

---

## F. Operations, security, and delivery (46–50)

**46. How do you isolate tenants?**
Row-level isolation and per-tenant keys. Overlays (hospital protocols, alias dictionaries) are additive, labelled and tenant-local. Overlay cannot suppress or downgrade a regulatory or safety result. A cross-tenant negative suite is a release blocker.

**47. How does audit and replay work?**
An append-only, hash-chained log of input references, chunk and rule ids, release ids, prompt and model ids, and the stored output. Replay means the stored artifact with pinned versions, not re-running the model. PHI is encrypted separately and can be crypto-shredded without breaking the chain.

**48. What does offline and shared-device use look like?**
Read-only drug master from a signed cache with expiry; no safety results from an expired cache; visible offline banner; per-doctor login, auto-lock, no PHI stored on the device.

**49. How did you use AI, and where was it wrong?**
[Use your true story.] Example: AI first proposed comparing summed doses with label maximums; I changed it because that sits too close to the dose-calculation fence. I also checked legal dates against sources instead of trusting the draft.

**50. What are you least sure about?**
Whether CDSCO's software guidance classifies the tool as a device, and the current NMC prescribing rules. Both need qualified legal review. I also haven't validated my numeric targets with a clinician, and the latency target is aspirational until measured in live use.

---

### Quick drills
- Say the 8 states from memory.
- Explain "no rule found ≠ safe" in 20 seconds.
- Explain relevance vs applicability vs citation support with one example.
- Name two things you would change in your own design.
