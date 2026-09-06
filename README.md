# Kivi &middot; Personal Phonetic Memory (prototype)

> Kivi's dictionary is **not** a list of replacements. It is an evidence-backed memory of
> how *one particular person* uses words. A phonetic/fuzzy match only **nominates a
> candidate** &mdash; a deterministic, explainable engine then decides **REPLACE / KEEP / DEFER**.

This is a working end-to-end prototype of the system described in the project roadmap. It
implements the full pipeline, including ASR and formatting (which the brief says need not be
built), so a reviewer can experience every stage.

```
AUDIO / TEXT
   -> ASR (LEVEL 1: raw transcript)
   -> FORMATTER (LEVEL 2: generic punctuation / casing)
   -> SPAN BUILDER (align formatted tokens <-> ASR tokens)
   -> CANDIDATE RETRIEVAL (exact / normalized / fuzzy / phonetic)   <- candidates only
   -> CONTEXT + EVIDENCE ANALYSIS (positive / negative context, confidence, contradiction)
   -> DECISION ENGINE (multiplicative score + hard gates)  -> REPLACE / KEEP / DEFER
   -> SPAN REWRITER (LEVEL 3: memory-aware transcript)
   -> DECISION TRACE (why every word changed or was deliberately left alone)
```

## What makes it more than fuzzy matching

| Idea | Where it lives |
|---|---|
| A phonetic hit is a **nomination**, not a verdict | `kivi/memory/retrieval.py` |
| **Personal, accent-adaptive phonetics** &mdash; Kivi learns *how this particular person's ASR mangles sounds* from their own corrections, and uses that to nominate mishearings a generic algorithm misses and to break ambiguities toward the word they've had to fix before | `kivi/memory/phonetic_profile.py` |
| **Confidence** (is this real?) and **applicability** (does it apply here?) are separate axes | `kivi/memory/confidence.py` + `kivi/context.py` |
| **Context-scoped negative evidence** &mdash; rejecting "I ate a kiwi" never weakens the `Kivi` memory | `kivi/memory/learning.py` (`negative_context` evidence, excluded from confidence) |
| **Processing is read-only** with respect to memory; only user observations mutate it | `kivi/pipeline.py` vs `kivi/memory/learning.py` |
| **DEFER** is a real outcome for ambiguity / weak memories / contradictions | `kivi/decision.py` |
| **Deliberate non-intervention** is a first-class, tested, measured behaviour | eval category `contextual_contradiction`, metric `non_intervention_accuracy`, `memory_stability_pass` |
| Every decision has scores + evidence + a human sentence | `kivi/explain.py`, Decision Trace tab |
| Thresholds are named constants, treated as hypotheses validated by evaluation | `config.json`, `evaluation/results/report.md` |

## Flagship behaviour

| Input | LEVEL 2 (formatted) | LEVEL 3 (memory-aware) | Why |
|---|---|---|---|
| `Open the kiwi service.` | `Open the kiwi service.` | `Open the **Kivi** service.` | active memory + software context |
| `I ate a kiwi today.` | `I ate a kiwi today.` | `I ate a kiwi today.` *(unchanged)* | memory matches, but the food context is a do-not-apply context |
| `ask civi` | `Ask civi.` | `Ask civi.` *(DEFER)* | `civi` fits both `Kivi` and colleague `Sivi`; too close to choose |
| `we should ship devy this sprint` | ... | *(DEFER)* | `Devi` is only a `proposed` memory &mdash; never auto-replaces |
| `deploy the Zephyr module` | ... | *(KEEP)* | no memory; no hallucinated one is created |
| `tell lehan about the sync` *(after correcting `lehaan`/`lehin` &rarr; `Rehan`)* | `Tell lehan about the sync.` | `Tell **Rehan** about the sync.` | generic phonetics + fuzzy both miss `l`&harr;`r`; the learned **accent rule** `l&rarr;r (onset)` nominates it |
| `ask civi` *(after 4&times; correcting `Kivi` mishearings)* | `Ask civi.` | `Ask **Kivi**.` | `civi` still fits `Kivi` and `Sivi`, but the user's ASR has mangled `Kivi` 4&times; and `Sivi` 0&times; &mdash; history breaks the tie (cf. the bare `ask civi` above, which still DEFERs) |

## Personal (accent-adaptive) phonetics

A generic phonetic key (Soundex / Metaphone / the coarse key in `kivi/phonetics.py`)
encodes how English sounds *in general* &mdash; it is identical for every user. But a
*personal* phonetic memory is about **one particular person**: a given speaker +
microphone + ASR model mangles the same sounds the same way, repeatedly.

Every correction the user makes is aligned (`misheard` &harr; `chosen`) and coalesced into
small, position-tagged substitution rules (`w&rarr;v (medial)`, `l&rarr;r (onset)`, `&empty;&rarr;n (coda)`).
Those rules accumulate in one inspectable table (`sound_pattern`) &mdash; the user's
**accent profile** &mdash; and feed retrieval two ways:

1. **Candidate creation** &mdash; apply the profile to an unmatched span; if the result
   lands on a known memory, nominate it (`match_method = personal`). This catches
   cross-class confusions (`l`/`r`, `n`/`l`, dropped codas) that share no generic
   phonetic key and fall below the fuzzy floor.
2. **Disambiguation prior** &mdash; a memory this user's ASR has demonstrably mangled
   before gets a small, capped score boost, and can break a near-tie between two
   candidates (`ask civi` &rarr; `Kivi`, not `Sivi`).

Safety is structural: the boost is applied **after** the hard gates, so it can never
override a deliberate KEEP (negative context) or auto-apply a not-yet-active memory.
The profile is written **only** by the learning path; processing never grows it
(asserted in the eval's idempotency check and in `tests/test_basic.py`).
Thresholds live in `config.json` under `personal_phonetics`.

## Tech

Python 3.11+, **stdlib only** for all core logic (phonetics, fuzzy match, decision engine,
learning). FastAPI + uvicorn only for the HTTP layer. **SQLite** single file
(`database/kivi.db`) with plain numbered SQL migrations. Vanilla-JS single-page frontend.
No LLM in the core path &mdash; `model_calls` and `est_cost_usd` are `0` by design.

## Layout

```
manage.py                 migrate / seed / reset / serve / eval / process
config.json                all thresholds & weights (provisional, validated by eval)
backend/kivi/              core library (normalization, phonetics, retrieval, context,
                           decision, rewrite, explain, pipeline, memory/*, asr/*, formatter/*, db/*)
                           memory/phonetic_profile.py = accent-adaptive phonetics
backend/api/main.py        FastAPI endpoints (incl. GET /api/phonetic-profile)
database/migrations/       0001_init.sql, 0002_personal_phonetics.sql
database/seed/seed.json    5 seed memories + 1 seed accent rule
evaluation/dataset/        cases.jsonl  (28 cases: useful interventions, deliberate
                           non-interventions, and 5 personal_phonetics cases)
evaluation/run.py          harness
evaluation/results/        COMMITTED: summary.json, report.md, cases/*.json, failures/*.json
frontend/index.html        Try it / Memory / Evaluation / Reset  (+ in-browser recording)
tests/test_basic.py        unittest suite
```

## Current results (`evaluation/results/summary.json`)

- 28 / 28 cases pass; action accuracy 100%; exact output match 100%
- intervention precision / recall / F1 = 100%
- **non-intervention accuracy 100%**, over-intervention count **0**
- memory-stability pass **6/6** (rejections & contradictions never damage a valid memory)
- idempotency pass **1/1** (re-processing is byte-identical and grows no memory or accent-profile rows)
- **personal-phonetics 5/5** &mdash; accent rule creates a candidate generic phonetics
  misses; correction history lifts a borderline match and breaks an ambiguity; and it
  never overrides a deliberate KEEP (`evaluation/results/summary.json` &rarr; `personal_phonetics`)
- latency p50 ~1 ms, p95 ~2 ms; model calls 0; cost $0

The evaluation dataset is small and co-designed with the system &mdash; its value is the
**methodology and the harness**, which records per-case scores and writes a
`failures/<id>.json` for any miss (see `evaluation/results/report.md`).

See **RUN.md** to set up, run, evaluate, and reset.
