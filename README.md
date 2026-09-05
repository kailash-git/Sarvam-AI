# Kivi · Phonetic Memory

Word-level memory that moves a transcript from **formatted** to **memory-aware** —
the smallest system that makes Kivi feel as though it has met this person before.

```
ASR output      ask aditya to review the sarvam kiwi service
Formatted       Ask Aditya to review the Sarvam Kiwi service.
Memory-aware    Ask Aaditya to review the Sarvam Kivi service.
```

`Aditya → Aaditya` is a person's preferred spelling that speech recognition
cannot know. `Kiwi → Kivi` is a product name the language model has no reason to
prefer. Kivi learns both from ordinary use and applies them the next time the
person speaks — and, just as importantly, leaves the fruit alone in *"I ate a
kiwi for breakfast."*

**Start here:** [`RUN.md`](RUN.md) — the primary review method and exact commands.
Evaluation output is committed at [`eval/results/report.md`](eval/results/report.md).

---

## 1. The product

### What it means for Kivi to "remember a person's words"

Kivi keeps a small set of **memory entries**. Each entry is one term the person
uses that a general model gets wrong: a name, a company, a product, a piece of
jargon — together with the exact surface form they expect to see.

An entry is not just a string. It carries:

| field | why it exists |
|---|---|
| `canonical` | the surface form to produce (`Aaditya`, `Kivi`, `PyTorch`) |
| `category` | `person` / `org` / `product` / `term` / `other` — a light prior on how eagerly to act |
| `phonetic_primary` / `_secondary` | Double Metaphone of the canonical, for indexed "sounds-like" lookup |
| `aliases` | concrete forms it has been *heard* as (`kiwi`, `pie torch`, `servam`) |
| `status` | `candidate` → `active` → `suppressed` (see lifecycle) |
| `confidence` | drives whether an otherwise-matching entry is strong enough to act on |
| `source`, `note`, timestamps | provenance for the inspector |

### Which observations change future behaviour

Kivi learns from **explicit signals only**. This is a deliberate boundary — the
brief asks for the smallest system that works, not the largest one describable.

| observation | what it is | effect |
|---|---|---|
| **correction** | the person edited a formatted transcript | the `formatted → edited` diff yields `(wrong form → right form)` pairs; each creates or reinforces a **candidate** |
| **dictionary** | the person explicitly added a word | trusted immediately → **active** |
| **rejection** | the person undid a rewrite Kivi made | negative evidence; enough of them **suppress** the entry |
| **acceptance** | the person kept a rewrite | small positive reinforcement |

Not used: free typing elsewhere, usage telemetry, guessing entities from
context. Those belong to Kivi's larger episodic/semantic memory, not to this
edge.

### When Kivi should do nothing

A rewrite is only safe when the evidence is strong *and* nothing about the
sentence argues against it. Kivi deliberately does nothing when:

| situation | reason tag |
|---|---|
| only one unconfirmed correction so far | `not_active` |
| the entry was rejected repeatedly | `suppressed` |
| the text already matches the canonical form | `already_canonical` |
| the span is an everyday word in an ordinary sentence (`service`, `join`, `mark`) | `common_word_guard` |
| the sentence is about food and the span is a common-noun homophone (`kiwi` the fruit) | `food_context_guard` |
| two entries match the span equally well and context can't break the tie (`Jon` vs `John`) | `ambiguous_conflict` |
| the metaphone codes rhyme but the spellings are far apart (`Colin` vs `Cologne`) | `weak_surface_match` |
| the combined score is under the apply threshold | `below_threshold` |
| entry confidence is under the minimum to act | `low_confidence` |

Every non-action is recorded with its tag, so the demo and the evaluation can
always answer *why*.

---

## 2. Architecture

```
          Web UI  ─┐                     ┌─ services/learning.py    observation → memory
          CLI     ─┼─►  FastAPI  ──────► ├─ services/retrieval.py   phonetic n-gram lookup + scoring
                   │    app/api/routes   ├─ services/pipeline.py    decide → apply → trace
                   │                     ├─ services/phonetics.py   Double Metaphone + rapidfuzz
                   │                     └─ services/llm.py          mock (default) | live Anthropic
                   └────────────────────────────────► SQLite (embedded, durable, WAL)
```

Same core behind both interfaces. `mock` mode makes the rewrite a deterministic
span replacement; `live` mode calls the model with the same glossary but the
pipeline only accepts edits it independently sanctioned.

### The pipeline, per utterance

1. **Extract candidates** (`retrieval.py`). Walk every 1–3-token span of the
   formatted text. Compare each span only against memory forms of the *same
   token length*, using the phonetic similarity metric. Keep the tightest,
   strongest match per region; attach the runners-up as `rivals`.
2. **Score.** `combined = 0.62·phonetic + 0.28·confidence + category_prior`,
   with small adjustments for a distinct alias hit, an everyday-word span, and a
   multi-word match.
3. **Decide** (`pipeline.py`). One `Decision` per candidate, `apply` or `skip`,
   each with a reason tag from the closed vocabulary above. Guards run in a fixed
   order: status → food-context → common-word → ambiguity (with context
   resolution) → weak-surface → confidence → threshold.
4. **Apply.** Build span-accurate replacements, case-matched to the original
   token. Hand the compact glossary + sanctioned replacements to `llm.py`.
5. **Trace.** Persist the utterance (all three transcript levels), every
   decision (applied and not), the replacements, latency, and LLM-call count.

### Phonetic matching (`phonetics.py`)

- **Keys:** Double Metaphone primary + secondary, plus Soundex as a looser
  bucket. Indexed in `memory_entries` / `aliases`.
- **Similarity** blends: primary-code Indel ratio, secondary-code ratio, and a
  letter-level score that is the max of raw Indel ratio, Jaro-Winkler, and an
  Indel ratio over **v/w/k/c-folded skeletons** (Metaphone drops the semivowel
  in "kiwi", so "kiwi"/"Kivi" needs the fold). A large raw edit distance vetoes
  a code-space-only rhyme.
- **`surface_similarity`** is a stricter letters-only measure (no Jaro-Winkler
  prefix bonus) used by the `weak_surface_match` guard.

### Data model

`memory_entries`, `aliases`, `observations` (immutable audit trail),
`interventions` (one row per candidate considered, applied or not), `utterances`
(the three transcript levels + JSON decision trace), `schema_migrations`. Full
DDL: [`app/db/migrations/0001_initial.sql`](app/db/migrations/0001_initial.sql).
Forward-only migration runner: [`app/db/migrate.py`](app/db/migrate.py).

### Lifecycle

```
              correction ×1                correction ×KIVI_PROMOTE_AFTER
   (nothing) ───────────────►  candidate ───────────────────────────────►  active
                                   │                                         │
                                   │           dictionary add (any time)     │
                                   └─────────────────────────────────────────┘
                                                                             │
                                            rejection ×KIVI_SUPPRESS_AFTER   ▼
                                   active ───────────────────────────►  suppressed
```

`suppressed` entries stay visible in the inspector but are never applied. Entries
can also be activated / suppressed / deleted by hand from the UI or API.

---

## 3. What the abstraction makes possible

Because an entry separates *canonical form*, *category*, *phonetic key*, *learned
aliases*, and *confidence*, the system can distinguish cases that a
find-and-replace list cannot:

- **Same sound, different intent.** "the Sarvam **Kivi** service" rewrites;
  "I ate a **kiwi**" does not — the food-context guard reads the sentence, not
  just the word.
- **Learned mis-hearings vs canonical-only guesses.** A span that matches a
  *learned alias* (`servam → Sarvam`) is trusted more than one that only rhymes
  with a canonical never heard that way — which is how `Colin` avoids eating
  `Cologne`.
- **Evidence strength.** One correction is a candidate and does nothing; two
  make it act; an explicit dictionary add acts immediately.
- **Genuine ambiguity.** `Jon` and `John` both in memory, "Jhon" spoken → do
  nothing and say so — unless "Jon" also appears elsewhere in the sentence, in
  which case that resolves it.
- **Reversibility.** Two rejections suppress an entry; the demo shows this live.

Where it should deliberately do nothing: everyday words, weak evidence,
suppressed or deleted entries, already-correct text, and unresolved ambiguity —
all covered as first-class cases in the evaluation, not afterthoughts.

---

## 4. Evaluation

`eval/dataset/cases.yaml` — 34 cases, each self-contained: the evidence to teach,
the utterance under test, the exact required output, and (for restraint cases)
the reason tag that must fire. Cases are declared by class —
`should_intervene` / `should_not_intervene` / `ambiguous` — and span: person
spellings, brands, multi-word jargon, casing, homophones, everyday-word
collisions, proper-noun collisions, weak evidence, suppression, deletion,
ambiguity with and without a context tie-breaker, auto-learned aliases, and an
empty memory.

`python -m eval.run` runs every case in an **isolated database** (reset → seed →
one pipeline call), then writes:

- [`eval/results/results.json`](eval/results/results.json) — inputs, expected,
  actual, memory state, every decision, outcome, latency, per case.
- [`eval/results/report.md`](eval/results/report.md) — headline metrics;
  **useful interventions** and **unnecessary/incorrect interventions** as
  separate ledgers (as the brief asks); a **known-limitations** section;
  a per-case table; failure detail.

Committed run (`KIVI_LLM_MODE=mock`, deterministic): **32 / 32 scored cases pass**,
intervention precision / recall / do-nothing accuracy **1.00 / 1.00 / 1.00**,
p50 / p95 latency ≈ **5 / 11 ms**, **0** LLM calls, DB ≈ **80 KB** after the whole
suite. Two cases are tagged `known_limitation` and are **excluded from the
headline metrics** and listed separately so they cannot flatter the numbers.

Metrics are honest about mode: `mock` issues zero model calls and near-zero
latency by construction; `--live` issues one call per utterance and the report
records the real count. Cost in `mock` is zero; in `live` it is one short
constrained completion per utterance.

### Known limitations (tracked in the suite)

1. **The food-context guard is sentence-level.** An eating clause anywhere in a
   sentence suppresses a legitimate later product mention
   (*"I ate lunch then joined the kivi review"*). Splitting clauses needs
   sentence understanding beyond a word-level memory.
2. **No morphology.** *"kiwis"* is matched to *"Kivi"* and rewritten, dropping
   the plural. A phonetic memory has no inflection model.

Other boundaries: English-centric phonetics (Metaphone/Soundex); the everyday-word
and food-cue lists are small and hand-curated; single-user (no per-user
partitioning); `mock` rewrite is exact span replacement, so it does not
demonstrate morphological smoothing that a `live` model would do.

---

## 5. Repository layout

```
app/
  main.py                 FastAPI app (runs migrations on startup)
  config.py               env-driven settings, all with defaults
  api/routes.py           HTTP surface (thin adapters over services/)
  db/
    migrations/0001_initial.sql
    migrate.py             forward-only migration runner
    models.py              connection management, WAL, row helpers
    seed.py                reproducible seed data (taught via the real API)
  services/
    phonetics.py           keys + similarity + surface similarity
    learning.py            observation → candidate/active/suppressed
    retrieval.py           span extraction, matching, scoring, overlap resolution
    pipeline.py            decision layer + apply + trace  (reason vocabulary)
    llm.py                 mock (deterministic) + live Anthropic rewrite
    common_words.py        everyday-word + food-context guard lists
    repo.py                data access
    admin.py               reset
  web/index.html           single-page demo (no build step)
cli/kivi.py                same core on the command line + `walkthrough`
eval/
  dataset/cases.yaml       34 cases
  run.py                   isolated-per-case runner + metrics + report
  results/                 committed results.json + report.md
tests/                     phonetics, learning lifecycle, pipeline guards, API
RUN.md                     primary review method + exact commands
```

---

## 6. AI use

- Built with **Claude Code (Claude Sonnet 5)** as a pair-programmer: it wrote the
  bulk of the code, the evaluation dataset, and this documentation from my design
  brief, and iterated on the phonetic scoring and the decision guards against the
  test suite. Every design decision recorded here — the evidence model, the
  status lifecycle, the closed reason vocabulary, the mock-first LLM strategy,
  the guard ordering, the two acknowledged limitations — was reviewed and
  directed by me.
- **Runtime AI:** the memory-aware rewrite step. Default `mock` uses no model
  (deterministic span replacement). `KIVI_LLM_MODE=live` calls the Anthropic
  Messages API (`claude-sonnet-5` by default) with a constrained prompt and a
  strict acceptance filter; env var and `.env.example` are documented in
  `RUN.md`. No credentials are committed.
- No hidden benchmark, private corpus, or pretrained phonetic model is used —
  Metaphone / Soundex / edit distance only.
