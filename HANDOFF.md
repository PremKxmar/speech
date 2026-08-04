# HANDOFF

State of the work as of the last commit, and what to do next. Read
[PROJECT.md](PROJECT.md) for the design reasoning; this file is only about where
things stand and what is left.

---

## Current state

- **434 tests passing**, offline, in about 60 seconds.
- Working tree clean, everything pushed to `PremKxmar/speech.git`.
- Backend runs and serves the UI. Frontend has never been rendered against it.
- ECAPA-TDNN installed and verified. Whisper installed, checkpoint not downloaded.
- **No human speech has been recorded.** Every number in the repo is synthetic.

```bash
git clone https://github.com/PremKxmar/speech.git
cd speech
pip install -r requirements-core.txt
pytest                    # expect 434 passed
```

---

## The one thing that blocks publication

**There is no corpus.** The code is publishable-grade; the paper is not producible,
because no experiment has been run on human speakers. `simulation.py` generates
speakers who differ *by construction* — that a model separates them proves the
implementation is correct, not that Tamil–English speakers actually have stable,
distinguishable code-switching habits. That is the hypothesis the paper claims, and
it is untested.

Nothing else on this list matters as much. Everything below is preparation for the
day the recordings exist.

---

## Next tasks, in priority order

### 1. Corpus layer — `backend/kavach/corpus.py` (does not exist yet)

A manifest format and loader so real recordings drop in where `simulation.py`
currently sits. Needs:

- A manifest schema (speaker id, session, utterance, elicitation prompt, consent
  reference, device, environment).
- A loader producing the same `list[UtteranceTokens]` shape that `build_trials`
  already consumes, so `eval/ablation.py` needs no changes.
- A recording protocol document: how many speakers, how many sessions each, what
  prompts elicit which semantic classes, what the consent form must say.
  `csbg/ontology.py::ELICITABLE_CLASSES` is the constraint that drives the prompt
  design.

Design note: sessions must be separable, because §5.3 shows same-session versus
cross-session materially changes what the integrity detector can see — and because
enrolment stability needs a within-speaker across-session measurement.

### 2. Figures — `backend/kavach/eval/figures.py` (does not exist yet)

matplotlib, no seaborn, publication style. The user's standing constraint applies
to figures as well as UI: **no neon, no AI-looking gradients.** Greyscale-safe,
serif labels, no chartjunk.

- DET curve, one line per configuration
- Ablation bar chart with bootstrap CIs
- CSBG heatmap (21 classes × language choice) for two contrasting speakers
- Enrolment stability curve (`ablation.stability_curve` already computes it)
- **IAPMR vs. within-speaker consistency scatter** — this is the §5.4 figure and the
  most interesting one in the paper

### 3. Experiment runner — `backend/kavach/experiments.py` (does not exist yet)

One command that produces every number and figure in the paper, with seeds and a
manifest recorded. `python -m kavach.experiments --out paper/results/`. Emits
`results.json`, `figures/*.pdf`, `tables/*.tex`. The paper then reads from
`results.json` so no number is ever hand-typed into LaTeX.

### 4. Scoring-level ablations — `eval/ablation.py`

`ablate_policy` handles policy-level ablations. These need a fresh `build_trials`
and are not implemented in `run_ablation` yet:

- class set: with and without `LOW_SIGNAL_CLASSES`
- transition stream on/off
- cohort z-normalisation on/off

Also: `stability_curve` exists but `run_ablation` never calls it — it has to be
invoked separately.

### 5. Smaller items

- **Per-speaker IAPMR through the API.** `AttackSuite.per_speaker_breakdown` exists;
  no route exposes it.
- **Run the frontend.** `npm install && npm run dev` against a live backend, then
  check every page and `npm run lint` (`tsc --noEmit`). The contract test guards
  field *names*, but as `tests/test_api.py` puts it: *"TypeScript cannot check a
  JSON payload at runtime, so nothing in the frontend build catches a renamed field
  — the failure appears as a blank panel in a demo."* No page has been rendered
  against the real backend.
- **Whisper checkpoint.** `large-v3` is ~3 GB and has never been downloaded. Decide
  between `large-v3` (better Tamil) and `small` (usable on this laptop) and record
  which was used — `whisper_language` also matters: forcing `ta` generally produces
  better Tamil but can suppress English segments, which is exactly wrong for
  code-mixed input. Evaluate both and report which you used.

---

## Traps already hit — do not re-derive these

1. **Do not set a threshold by reasoning about a score scale.** It has been wrong
   twice here (`CSBG_VETO_FLOOR`, `INTEGRITY_FLOOR`), both times because the
   constant lived in a module that could not observe what the other module emitted.
   Measure the distribution first. Both constants now have tests that re-derive them
   from real detector output.

2. **Do not let a grid search pick a floor that sits on a discrete evidence level.**
   `score < floor` means a floor of exactly 0.20 catches none of the probes scoring
   exactly 0.20, while looking optimal. See `integrity.INTEGRITY_FLOOR`.

3. **Do not report an unattainable operating point as a rate.** `format_rate()`
   returns `"n/a"` for NaN. If you add a metric, follow it.

4. **Do not let the test suite inherit its environment.** It used to pass in 48 s
   only because the models were not installed. `KAVACH_OFFLINE` and
   `tests/conftest.py` make that explicit; keep new tests inside it, and mark
   anything needing a real checkpoint `@pytest.mark.models`.

5. **Do not substring-search a serialised structure for a secret.**
   `test_public_dict_hides_the_answer` used `assert answer not in str(public)`, and
   `public_dict()` carries Unix timestamps while one SKG fixture fact is the room
   number `"214"`. A float timestamp is ten digits, so it contains any given
   three-digit run every few hundred seconds — the suite failed roughly once in
   several full runs, on a collision with the clock. The assertion shape is wrong
   in both directions: too weak (an answer split across fields, or case-folded,
   slips through) and too strong (any numeric field can collide). Check the fields
   that could actually carry it.

6. **`data/` is git-ignored in its entirety** because it holds voiceprints next to
   hometowns and family names. `Store.delete_speaker` must erase audio, tokens,
   facts, graphs and history — a deletion request that leaves recordings behind has
   not been honoured.

7. **Never report a number from `simulation.py` as an experimental result.** Its own
   docstring says so.

---

## Standing instructions from the user

- Push to `PremKxmar/speech.git` at every step.
- The UI lives in `kavach/` and is user-supplied. Extend it; do not replace it.
- **No AI-looking design** — no neon colours, no gradient-heavy dashboards. This
  applies to figures too.
- The deadline is the user's concern, not the assistant's. Build the project.

---

## Open questions for the user

1. **How many speakers can realistically be recorded, and over how many sessions?**
   This determines whether the evaluation is a pilot (n≈10, report descriptively) or
   a study (n≈30+, report EER with intervals). `eval/ablation.py` already discards
   cross-boundary trials, so the usable trial count falls faster than the speaker
   count suggests.

2. **Is there an ANTHROPIC_API_KEY available for the semantic tagging pass?**
   Without it the LID pipeline falls back to rules plus a low-confidence guess for
   unresolved Latin tokens, which `PipelineStats.is_corpus_grade` correctly refuses
   to certify. The tagging quality bounds everything downstream.

3. **Is a voice cloner in scope?** A3–A5 acoustic scores are currently drawn from a
   documented distribution. Without a real cloner those rows stay simulated and
   `paper_ready()` keeps refusing them. XTTS-v2 is the obvious candidate and is
   already named in the requirements.
