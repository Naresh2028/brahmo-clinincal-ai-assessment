# Shipping a small LLM feature on your own: a guide (no code, resources only)

Goal for one day: a small, **deployed** feature that answers a question from a few documents with citations, and says "cannot confirm" when the documents do not cover it, plus a tiny evaluation set. Be honest in the interview: this is a **prototype you built and deployed**, not a production product with real users.

## 1. Decide the tiny scope (30 min)
- Pick 8-10 short fictional "guideline" snippets (you write them; mark as fictional). Give each an id, a title, and a "population" (adult/child).
- Feature: `/ask?q=...` returns {answer, citations[], status: "answered" | "cannot_confirm"}.
- Rule from your assessment: no consequential claim without a citation; otherwise abstain.

## 2. Learn just enough (2-3 hours total)
Resources (check that links and versions are current; I cannot verify them live):
- **Anthropic docs** (docs.claude.com): Messages API quickstart, prompt engineering overview, tool use / structured JSON output, and the section on evaluating prompts.
- **Anthropic Cookbook** (github.com/anthropics/anthropic-cookbook): look for retrieval/RAG and evaluation examples.
- **Anthropic engineering post "Building effective agents"**: when to keep things simple.
- **FastAPI tutorial** (fastapi.tiangolo.com): request bodies, Pydantic models, dependencies.
- **Pydantic docs**: validating the model's JSON output.
- **pgvector README** (github.com/pgvector/pgvector) and PostgreSQL full-text search docs: what "vector search" and "hybrid search" mean. For a one-day prototype, simple keyword search over 10 snippets is fine; just be able to explain how vectors would be added.
- **Eugene Yan, "Patterns for Building LLM-based Systems & Products"** and **Hamel Husain's writing on LLM evals**: how to test LLM features.
- **OWASP Top 10 for LLM Applications**: prompt injection and data leakage (your assessment already covers this).

## 3. Build in this order (one afternoon)
1. Retrieval first, no LLM: given a question, return the best 1-3 snippets by keyword match. Write 10 test questions and check retrieval by hand.
2. Add the LLM only to **compose text from the retrieved snippets**, with a system prompt that says: use only the provided snippets; cite ids; if not covered, say so.
3. **Validate the output in code**: parse JSON, check every cited id exists in the retrieved set, reject answers with no citation (return "cannot_confirm"). This mirrors "claims without citation ids are stripped".
4. Treat the retrieved text and the question as data, not instructions (test a snippet that says "ignore previous instructions").
5. Log each request (question, snippet ids, prompt version, model name, output) to a file or table.

## 4. Evaluate (1 hour)
Build 20 cases in a JSON file: 10 answerable, 5 not covered by your snippets (must abstain), 3 wrong-population (child vs adult; must not use the wrong snippet), 2 injection attempts. Compute: correct answers / answerable, abstained correctly / unanswerable, and **confidently wrong** = answered but unsupported or wrong. Keep the list and the numbers. This is your "evaluation harness that caught failures" story. Record one real failure it caught and how you fixed it.

## 5. Deploy (1-2 hours)
- Put the code in a GitHub repo with a README, a `.env.example`, and `pytest` running in GitHub Actions.
- Deploy to a free-tier host (Render, Fly.io, Railway or similar; check current free options). Keep the API key in environment variables, never in the repo.
- Add a spending limit in your LLM provider's console and rate limiting to your endpoint.

## 6. What to say in the interview
- What you built, in two sentences, and that the data is fictional.
- Where the LLM was wrong or incomplete in your testing, and what code check you added.
- The trade-offs: cost per request, latency (streaming helps), abstention vs helpfulness.
- What you would do next in production: real sources and rights, clinician-reviewed evaluation set, monitoring, human review, access control.
