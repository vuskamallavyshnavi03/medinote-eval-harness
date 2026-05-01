# HEALOSBENCH — Eval Harness for Structured Clinical Extraction

> **Take-home assessment** · target ~8–12 focused hours · synthetic data only

You're shipping an LLM-powered feature that turns a clinical transcript into structured JSON: chief complaint, vitals, medications, diagnoses, and follow-up plan. Once it's in production, you can't just "vibe-check" the prompt — you need a **repeatable evaluation harness** that tells you, with numbers, whether prompt v7 is better than prompt v6, on which fields, and where it fails.

Your job is to build that harness end-to-end: dataset loader, runner, evaluator, dashboard.

---

## Table of Contents

1. [What's Provided](#whats-provided)
2. [Stack](#stack)
3. [What You're Building](#what-youre-building)
4. [Hard Requirements](#hard-requirements)
5. [Stretch Goals](#stretch-goals)
6. [Constraints](#constraints)
7. [How to Run](#how-to-run)
8. [What We're Looking For](#what-were-looking-for)
9. [Submission](#submission)

---

## What's Provided

In `data/`:

| File | Description |
| --- | --- |
| `transcripts/*.txt` | 50 synthetic doctor–patient transcripts (~150–800 tokens each). Real-feeling but fully synthetic; no PHI. |
| `gold/*.json` | For each transcript, the ground-truth structured extraction a human annotator produced. |
| `schema.json` | The JSON Schema all extractions must conform to. |

The schema covers:

- `chief_complaint` *(string)*
- `vitals` *(object: `bp`, `hr`, `temp_f`, `spo2` — any may be `null`)*
- `medications` *(array of `{ name, dose, frequency, route }`)*
- `diagnoses` *(array of `{ description, icd10? }`)*
- `plan` *(array of strings)*
- `follow_up` *(object: `interval_days` int or null, `reason` string or null)*

> ⚠️ You **may not** modify the gold files or the schema. You **may** extend the transcript set with additional cases.

---

## Stack

The monorepo is already wired up:

- **Workspaces**: bun workspaces + Turborepo
- **`apps/web`** — Next.js 16 client-only dashboard
- **`apps/server`** — Hono on `:8787`, runs evals and stores results
- **`packages/db`** — Postgres + Drizzle ORM for storing runs
- **`packages/env`** — typed environment loading (zod)
- **`packages/auth`** — better-auth (not required for the eval task; ignore unless useful)
- **`packages/config`**, **`packages/ui`** — shared TS config and UI primitives

You will also create (or extend):

- **`packages/shared`** — shared types between server and web (schema types, run/result DTOs).
- **`packages/llm`** — a thin wrapper around the Anthropic SDK, with prompt strategies, tool use, retry-with-feedback, and prompt caching.

You'll need an Anthropic API key in `apps/server/.env` as `ANTHROPIC_API_KEY`. Use **Haiku 4.5** (`claude-haiku-4-5-20251001`) for cost; the eval is designed to be useful at Haiku quality.

---

## What You're Building

### 1. The extractor

> `packages/llm` + `apps/server/src/services/extract.service.ts`

- Takes a transcript and a **prompt strategy** (`zero_shot`, `few_shot`, `cot`) and returns extracted JSON.
- Use **Anthropic tool use** (or a strict JSON output mode) to force schema-conformant output. Free-form `JSON.parse` of model text is **not** acceptable.
- **Retry loop**: if the output fails JSON Schema validation, send the validation errors back to the model and let it self-correct. Cap at 3 attempts. Log every attempt.
- **Prompt caching**: the system prompt + few-shot examples must be cache-controlled so repeated runs don't pay for the same tokens. Verify via the SDK's `cache_read_input_tokens` field and surface this in the run summary.
- All three strategies live in the same codebase as swappable modules so adding a fourth is a 30-line change.

### 2. The evaluator

> `apps/server/src/services/evaluate.service.ts`

For each `(transcript, prediction, gold)` triple, compute **per-field scores using the metric appropriate to the field**:

| Field | Metric |
| --- | --- |
| `chief_complaint` | Fuzzy string match (normalize case/punctuation; token-set ratio or similar). Score ∈ [0, 1]. |
| `vitals.*` | Exact match per sub-field, with a tolerance for numeric fields (e.g. `temp_f` ±0.2 °F). Per-field 0/1, then averaged. |
| `medications` | Set-based **precision / recall / F1**. Two meds match if `name` is a fuzzy match **and** `dose` + `frequency` agree after normalization (e.g. `BID` == `twice daily`, `10 mg` == `10mg`). |
| `diagnoses` | Set-based F1 by `description` fuzzy match; bonus credit if predicted `icd10` matches gold. |
| `plan` | Set-based F1 on plan items, fuzzy-matched. |
| `follow_up` | Exact match on `interval_days`, fuzzy on `reason`. |

You must also detect and report:

- **Schema-invalid outputs** that escaped the retry loop (should be rare; track the rate).
- **Hallucinated fields** — values present in prediction but with no textual support in the transcript. Implement a simple grounding check: the predicted value (or a normalized form of it) must appear as a substring or close fuzzy match in the transcript. Flag and count these.

Per run, store: per-case scores, per-field aggregates, hallucination count, schema-failure count, total tokens (input/output/cache-read/cache-write), wall time, total cost in USD.

### 3. The runner

> `apps/server/src/services/runner.service.ts`

- `POST /api/v1/runs` with `{ strategy, model, dataset_filter? }` starts a run.
- Runs are concurrent (up to 5 cases in-flight) but respect Anthropic rate limits — implement a token-bucket or simple semaphore-with-backoff. **Don't** just `Promise.all` 50 cases.
- Stream progress to the dashboard via **SSE** as cases complete.
- Runs are **resumable**: if the server crashes mid-run, restarting and hitting `POST /api/v1/runs/:id/resume` continues from the last completed case (no double-charging).
- **Idempotency**: posting the same `{ strategy, model, transcript_id }` twice without `force=true` should return the cached result, not re-call the LLM.

### 4. The dashboard

> `apps/web`

- **Runs list** — every run, with strategy, model, aggregate F1, cost, duration, status.
- **Run detail** — table of all 50 cases with per-case scores; click into a case to see:
  - The transcript (highlighted where prediction values are grounded).
  - The gold JSON and the predicted JSON, side-by-side, with a **field-level diff**.
  - The full LLM trace: every attempt in the retry loop, each request and response, cache stats.
- **Compare view** — pick two runs and see per-field score deltas with a clear "which strategy wins on which field" breakdown. **This is the most important screen — make it good.**

### 5. Reproducibility

- A single command runs a full 50-case eval from the CLI without the dashboard, and prints a summary table to stdout. Used in CI / for sharing results:

  ```bash
  bun run eval -- --strategy=cot --model=claude-haiku-4-5-20251001
  ```

- Every run pins the prompt content via a **content hash** so "prompt v6" is unambiguous. Changing any character in the prompt produces a new hash.

---

## Hard Requirements

1. **Tool use / structured output, not regex on model text.** If you `JSON.parse` raw model output without a schema-enforcing path, you fail this requirement.
2. **Retry-with-error-feedback** loop, capped at 3, all attempts logged.
3. **Prompt caching** working and verified — show `cache_read_input_tokens` increasing across runs in the dashboard.
4. **Concurrency control** — no naïve `Promise.all`. Document (in `NOTES.md`) what your strategy does when Anthropic returns a 429.
5. **Resumable runs** — kill the server mid-run, restart, resume. This must actually work and you must include a test for it.
6. **Per-field metrics matched to field type** — exact, numeric-tolerant, fuzzy, set-F1 — used appropriately. A single "exact-match-everything" implementation fails this requirement.
7. **Hallucination detection** with a documented method, even if simple.
8. **Compare view** that surfaces real signal — not just two columns of numbers, but per-field deltas with a winner.
9. **At least 8 tests**, including: schema-validation retry path, fuzzy med matching, set-F1 correctness on a tiny synthetic case, hallucination detector positive + negative, resumability, idempotency, rate-limit backoff (mock the SDK), prompt-hash stability.
10. **No leaking the API key** to the browser. The web app talks only to Hono; only Hono talks to Anthropic.

---

## Stretch Goals

*Only if you have time — these are not required to pass.*

- **Prompt diff view** that shows what changed between two prompt versions and which cases regressed.
- **Active-learning hint**: surface the 5 cases with the highest disagreement between strategies — these are the cases most worth annotating better.
- **Cost guardrail**: refuse to start a run whose projected cost exceeds a configurable cap (estimate from token counts before sending).
- **Second model** (e.g. Sonnet 4.6) so the compare view also handles cross-model comparisons.

---

## Constraints

- **Synthetic data only.** Don't bring in real medical data, and don't put real patient info in test fixtures.
- **Budget**: a full 50-case Haiku run on all three strategies should cost **under $1**. If your harness can't hit that, your caching or prompt design needs work.
- **Time**: aim for **8–12 focused hours**. A polished 35-case version beats a buggy 50-case one.

---

## How to Run

```bash
# 1. Install
bun install

# 2. Configure
echo "ANTHROPIC_API_KEY=sk-ant-..." > apps/server/.env

# 3. Database (Postgres)
bun run db:push

# 4. Dev (web + server)
bun run dev

# 5. In another shell — CLI eval
bun run eval -- --strategy=zero_shot
```

You'll need a Postgres instance running locally. Set `DATABASE_URL` in `apps/server/.env` (e.g. `postgres://postgres:postgres@localhost:5432/healosbench`).

---

## What We're Looking For

- **Eval methodology taste.** The right metric for the right field. Honest reporting of failure modes (schema invalid, hallucinated, undergrounded). A compare view that would actually help you decide which prompt to ship.
- **Prompt engineering judgement.** Three strategies that are *meaningfully* different, not three flavors of the same prompt. A short writeup in `NOTES.md` of what you saw and why one wins on which fields.
- **LLM plumbing fluency.** Tool use, caching, retries, concurrency, idempotency — the things that separate a toy from a system you'd run in CI.
- **Test signal.** Tests target the things that actually break: rate limits, validation failures, resumes, fuzzy matchers.
- **A short `NOTES.md`** with: results table for the three strategies, what surprised you, what you'd build next, what you cut.

### What we're **not** looking for

- A pretty UI. Tailwind defaults are fine.
- Multi-user auth, multi-tenant, deployment.
- Hand-tuned prompts overfit to these 50 cases — we may swap the eval set.

---

## Submission

1. Push to a private repo and grant access, **or** zip the working tree (excluding `node_modules`).
2. Include `NOTES.md` at the repo root.
3. Include the output of one full 3-strategy CLI run (a `results/` folder or a paste in `NOTES.md`).
4. Make sure `bun install && bun run eval -- --strategy=zero_shot` works from a clean clone.

Good luck — and have fun.
