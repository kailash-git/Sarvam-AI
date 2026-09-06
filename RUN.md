# RUN.md &mdash; setup, run, evaluate, reset

## 1. Requirements

- Python 3.11+ (tested on 3.13)
- Node is **not** required (the frontend is a single static HTML file served by the API)
- All core logic is stdlib-only. Only the HTTP layer needs packages.

## 2. Setup

```bash
cd D:/Sarvam_AI
python -m venv .venv
# Windows PowerShell:  .venv\Scripts\Activate.ps1
# Git Bash:            source .venv/Scripts/activate
pip install -r requirements.txt
```

(You can also run everything except `serve` with no dependencies at all.)

## 3. Initialise the database

```bash
python manage.py reset
```

This deletes `database/kivi.db`, re-applies `database/migrations/*.sql`, and loads
`database/seed/seed.json`. Expected output (counts): `memory=5, alias=12, observation=18,
evidence=18, request=0, decision=0, sound_pattern=1` and seed memories:

```
Kivi=0.8405(active)  Sarvam=0.7448(active)  MCP=0.7122(active)
Sivi=0.7122(active)  Devi=0.3787(proposed)
```

`sound_pattern=1` is the one seeded **accent rule** (`w->v (medial)`, representing prior
onboarding). The accent profile is otherwise learned from corrections at runtime.

`migrate` and `seed` are also available separately.

## 4. Run the app

```bash
python manage.py serve
```

Open <http://127.0.0.1:8000> &mdash; four tabs:

1. **Try it** &mdash; type an utterance (or click **Record** to capture your microphone),
   press **Process**, see the three transcript levels and the decision trace. Use
   **confirm / reject** on any trace row to teach the memory, then watch it re-process.
2. **Memory** &mdash; browse memories; click a row for aliases, evidence, contexts,
   observations; teach a new memory. The top panel shows the **accent profile**
   (the user's learned ASR sound-substitutions); it grows as you confirm/reject
   mishearings. Also exposed at `GET /api/phonetic-profile`.
3. **Evaluation** &mdash; **Load results** shows the committed metrics + full report.
4. **Reset** &mdash; one click restores the seed (the reset/repeat workflow).

### One-shot from the CLI (no server)

```bash
python manage.py process "Open the kiwi service."
python manage.py process "I ate a kiwi today."
```

## 5. Evaluation

```bash
python manage.py eval
```

Resets the DB before every case, runs `evaluation/dataset/cases.jsonl` (28 cases:
useful interventions, deliberate non-interventions, and 5 `personal_phonetics` cases),
and writes:

- `evaluation/results/summary.json` &mdash; headline metrics, latency, model usage,
  by-category, and a `personal_phonetics` block (what each accent case demonstrates)
- `evaluation/results/report.md` &mdash; human-readable report incl. a
  **Personal (accent-adaptive) phonetics** section and a **Failures** section
- `evaluation/results/cases/<id>.json` &mdash; per-case: input, expected/actual output,
  expected/actual decision, scores, reason, latency
- `evaluation/results/failures/<id>.json` &mdash; one file per failing case (none currently)

These files are committed so a reviewer sees the numbers without running anything.
The run is **deterministic** (echo ASR + rule-based formatter + fixed `config.json`) &mdash;
re-running produces identical `summary.json`. After the run the DB is left clean and seeded.

## 6. Tests

```bash
python -m unittest discover -s tests -v
```

Covers phonetics, fuzzy bounds, the five pipeline behaviours (software REPLACE, fruit KEEP,
ambiguity DEFER, weak-memory no-auto-replace, unknown-word KEEP), the read-only-processing
invariant, the learning pipeline (scoped rejection does not touch global confidence;
scoped rejection blocks one context only; correction creates a memory that then replaces),
and **personal phonetics** (rule derivation is position-tagged and directional; a learned
accent rule nominates a never-seen mishearing; correction history lifts a borderline match;
the accent profile never overrides a deliberate KEEP and does not grow during processing).

## 7. Configuration

All thresholds and weights live in `config.json`:

| Key | Meaning |
|---|---|
| `thresholds.replace` / `.defer` | combined-score bands for REPLACE / DEFER |
| `thresholds.ctx_neg` | negative-context score that triggers a hard KEEP (deliberate non-intervention) |
| `thresholds.ctx_pos` / `.ctx_low` | context-gate shaping |
| `thresholds.phonetic_fuzzy_floor` | a phonetic-key match also needs this much fuzzy similarity to become a candidate |
| `thresholds.ambiguity_margin` | max combined-score gap between two memories to call it ambiguous -> DEFER |
| `confidence.*` | logistic parameters + `authoritative_floor` (an explicit user correction is confident by definition) |
| `personal_phonetics.min_observations` | how many times a learned accent rule must be seen before it fires in retrieval |
| `personal_phonetics.match_floor` / `.match_score_cap` | similarity floor for an accent-nominated candidate, and the ceiling on its surface score (kept below an exact hit) |
| `personal_phonetics.combined_boost` | max multiplicative lift on the combined score from "this user's ASR has mangled this memory before" |
| `personal_phonetics.tiebreak_min_count` | correction-count gap at which history breaks an otherwise-ambiguous match |

Values are provisional; `evaluation/results/report.md` is where they are validated.

## 8. Audio / recording notes

- The **Record** button uses the browser `MediaRecorder` API and posts a `webm/opus` blob
  to `POST /api/process-audio`.
- Live/recorded audio needs a **real** ASR provider. The default `asr_provider` is `echo`
  (typed text only); set `"asr_provider": "whisper"` in `config.json` and install
  `faster-whisper` + `ffmpeg` for real transcription, or `"fixture"` for stored transcripts.
  With `echo`, `/api/process-audio` returns HTTP 409 with an explanatory message.
- The **evaluation harness always uses `echo`** for determinism; live recordings are never
  part of the eval set.

## 9. Reset / repeat

```bash
python manage.py reset          # CLI
# or the "Reset to seed" button on the Reset tab
# or  POST /api/admin/reset
```

Idempotent, always recovers, and is what `eval` runs before each case.
