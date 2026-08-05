# HANDOFF

State of the work as of the last commit, and what to do next. Read
[PROJECT.md](PROJECT.md) for the design reasoning; this file is only about where
things stand and what is left.

---

## Current state

- **600 tests passing**, offline, in about 36 seconds.
- Working tree clean, everything pushed to `PremKxmar/speech.git`.
- Backend runs and serves the UI. **The frontend has now been run against it**,
  all eight pages render, and `tsc --noEmit` is clean. Three bugs that only
  appear against a real backend were found and fixed — see §5.6–5.8 of
  PROJECT.md.
- The corpus layer, the figures, and the experiment runner all exist. One
  command produces every table and figure.
- **No human speech has been recorded.** Every number in the repo is synthetic.

```bash
git clone https://github.com/PremKxmar/speech.git
cd speech
pip install -r requirements-core.txt
pytest                    # expect 600 passed
```

**Python 3.10+ is required, and it is not optional.** The code uses
`dataclass(slots=True)`; on 3.9 every test file fails at collection with
`TypeError: dataclass() got an unexpected keyword argument 'slots'`. The
machine this was last built on had only the system 3.9, so:

```bash
uv venv --python 3.11 .venv          # or any 3.10+ interpreter
uv pip install --python .venv/bin/python -r requirements-core.txt
.venv/bin/python -m pytest
```

Note `pyproject.toml` sets `addopts = "-q"`, so passing `-q` again suppresses
the pass/fail summary line. Run plain `pytest` to see the count.

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

## What was built last session

All five items on the previous list are done.

| Item | Where |
|---|---|
| Corpus layer | `backend/kavach/corpus.py`, `tests/test_corpus.py` (61 tests) |
| Recording protocol | [RECORDING_PROTOCOL.md](RECORDING_PROTOCOL.md) |
| Figures | `backend/kavach/eval/figures.py`, `tests/test_figures.py` (41 tests) |
| Experiment runner | `backend/kavach/experiments.py`, `tests/test_experiments.py` (31 tests) |
| Scoring ablations + stability wiring | `eval/ablation.py::ablate_scoring`, `run_ablation(enrolment=...)` |
| Per-speaker IAPMR | `GET /api/attacks/per-speaker`, rendered in the Attack Lab |

One command now produces the whole evaluation:

```bash
python -m kavach.experiments --out paper/results/ --simulate-branches
# -> results.json, report.md, tables/*.tex, figures/*.pdf
```

Everything it emits is stamped unreportable, in the JSON, in the README it
writes, and as a banner inside every `.tex` file. Removing that banner requires
a corpus, not an edit.

---

## Next tasks, in priority order

### 1. Record the corpus

**In progress.** Four speakers have returned complete sets. Collection is
running on the scripted route: `participant_scripts/SPEAKER_A.md` …
`SPEAKER_J.md`, one language profile per speaker, matrix in
`participant_scripts/README.md`. Known mapping so far: `jai` read script A.

`ingest.py` turns returned folders into a manifest. Read the "What the first
four returns taught us" section of [RECORDING_PROTOCOL.md](RECORDING_PROTOCOL.md)
before ingesting anything — two of four sent WhatsApp voice notes, which makes
the codec a speaker attribute, and every numeral transcribed as a digit until
the `suppress_numerals` fix.

**A scripted corpus cannot support §5.1** and `Provenance.SCRIPTED` enforces
that. What it does support: word-level LID on known text, whether the pipeline
recovers a planted profile end to end, and the acoustic/integrity branches,
which do not care that the words were authored. Run the spontaneous protocol in
parallel with anyone who will sit for it — five spontaneous speakers plus ten
scripted is a better paper than either alone.

The go/no-go question is unchanged and now answerable sooner: **do genuine and
impostor CSBG scores separate at all on real speakers?** With one session each,
`--within-session` scores the pilot and stamps the run unreportable, which is
the right trade for a smoke test. §12 of `KAVACH_Project_Idea.md` is the
fallback paper if the answer is no, and it is a respectable one.

### 2. Wire the real branches into the experiment runner — DONE

`eval/branches.py` supplies both, behind `--real-branches`: ECAPA-TDNN cosine
against an enrolled template, and the answer matcher against the claimed
speaker's enrolled answer to the same prompt. Off by default because it needs
the corpus audio present and downloads the speechbrain checkpoint on first use.
`--real-branches` and `--simulate-branches` together raise.

Three things there are load-bearing and easy to undo by accident:

- **Unmeasurable trials score `nan`, never `0.0`.** On a [0, 1] branch scale
  `0.0` is maximal evidence *against* the claim, so a missing recording scored
  as `0.0` looks like a confident impostor detection and improves the EER.
- **Cosine maps to [0, 1] affinely, not by clamping negatives.** The affine map
  is strictly monotone so it moves no trial past another; clamping would
  collapse the impostor tail the veto threshold is fitted on.
- **Coverage is counted and blocks reporting.** A branch that scored nothing and
  a branch that scored everything and found no signal produce the same fusion
  table.

**The knowledge branch measures nothing on the current pilot, by construction.**
It needs the claimed speaker to have answered the probe's prompt at enrolment.
A cross-session protocol always gives it that; a within-session split never
does, because a prompt held out as a probe is by definition absent from that
speaker's enrolment. It reports zero coverage and blocks the run rather than
emitting nan noise. Second sessions fix it — nothing in the code will.

### 3. Annotate the corpus

**The Whisper question is closed** — decided on real audio, reasoning in
`Settings.whisper_model`. `large-v3` for annotation (`small` emits fragments of
unrelated languages at code-switch boundaries), auto-detect for language (it
keeps English in Latin script, which is what `lid.rules` reads for free), and
`suppress_numerals` now genuinely suppresses numerals.

`annotate.py` does the pass. Stage 1 (ASR) is built and runs. **Stage 2 needs
`ANTHROPIC_API_KEY` and nothing downstream works without it** — `lid.rules`
decides no semantic class, so a rules-only pass puts every token in
`SemanticClass.OTHER`, the CSBG has one class containing everything, and every
speaker's graph is identical. This is the single blocking dependency for a
first number.

**WER against the read-speech scripts is not computable, and that is a finding
rather than a gap.** The scripts romanise Tamil because participants read them
aloud; Whisper writes Tamil in Tamil script. No Tamil word can align, so the
WER is pinned above 100% (measured: 281%, 221%, 201%) and describes an
orthography mismatch. Restricting to Latin tokens does *not* rescue it —
romanised Tamil is Latin, so the filter strips nothing from the reference and
all the Tamil from the hypothesis; that scored 96% and looked like a result.
`annotate.transcripts_are_comparable` refuses both and `asr_wer` stays None.

The Track-1 contribution therefore needs **hand-labelled word-level LID on a
subset**, and `kavach.goldset` is the tooling for it:

    python -m kavach.goldset export --manifest data/corpus_v1/manifest.json \
        --out data/goldset_v1.tsv --utterances 20
    # a Tamil-English bilingual fills in the lang and class columns
    python -m kavach.goldset score --manifest data/corpus_v1/manifest.json \
        --gold data/goldset_v1.tsv --report paper/results/lid_track1.md

The export samples round-robin across speakers (a uniform sample from four
speakers lands most tokens on one voice, and LID accuracy is per-speaker here)
and writes its whole instruction sheet into the file header.

**Labels are blank by default and `--prefill` is opt-in.** Prefilling turns
labelling into agreeing: a corrector who sees "TA" agrees far more often than a
labeller starting from nothing, so measured accuracy drifts toward 100%
whatever the tagger does. The file records which mode produced it and
`GoldScore` carries the caveat into the report, so a prefilled set can never be
reported as blind.

The scorer reports the **transliteration slice** separately —
`transliterated_total` / `transliteration_recall`, Tamil-script tokens the human
labelled EN. Aggregate accuracy hides them because they are a small fraction
overall and not a small fraction of NUMBER and TIME_DATE.

### 3a. Tools built while waiting for the corpus

Three commands that did not exist before and are needed to read what comes out
of the pipeline.

**`python -m kavach.inspect_corpus --manifest ...`** — read the corpus with your
eyes. `--summary` (default) is the go/no-go read: per-speaker CMI, Tamil share,
switch counts, and the language-choice-per-class table, which is the CSBG
printed. It states the conclusion rather than leaving it to be computed — a
Tamil-share spread under 0.10 across speakers gets "that is very little to
separate on". `--text` for transcripts, `--tags` for token tables, `--translit`
for every Tamil-script token labelled English.

Use `--translit` to *check* the transliteration recovery rather than trust it.
A wrong entry there is an English label on a word the speaker really did say in
Tamil, which is the same bias pointing the other way, and nothing downstream
can tell the two apart.

**`python -m kavach.goldset`** — see §3 above.

**`python -m kavach.experiments --real-branches --goldset ...`** — see §2 and §3.

### 3b. Tagging survives a rate limit now — but know how it behaves

A corpus pass drives one request per utterance with no pacing, against a free
tier that allows ten-odd a minute. Retries are automatic (429/5xx, exponential
backoff with jitter, `Retry-After` honoured and capped at 120s) and printed, so
a stall is visible rather than looking like a hang.

If a run still dies, **it has already saved what it tagged** (checkpoint every
ten utterances, plus a save before the exception propagates). Resume with
`--stage tag --resume`, never `--force`: `--force` re-sends the utterances that
succeeded, against the same rate limit that stopped the run. `--resume` also
does the right thing over rules-only tokens left by a no-key smoke test, which
plain `--stage tag` skips because they already have tokens.

### 4. Smaller items

All three of the previous items here are done:

- **Graph Explorer label overlap** — fixed. Concentric now spaces by label
  width and ranks the language nodes into the middle explicitly (its default
  ranking is node degree, which put them there most of the time anyway and
  silently reordered whenever the edge threshold moved).
- **The Speaker Knowledge Graph view** — implemented as `SKGViz.tsx`. It
  renders only what the wire format carries; inferring each node's semantic
  class would mean duplicating `FACT_TYPES` from `skg.py` in TypeScript, where
  it drifts the first time a fact type is added.
- **`Settings.demo_reveal_answers`** — tested both directions. The useful test
  is a sweep: with the flag off, a distinctive answer string must not appear in
  the body of *any* route that touches a speaker, which catches a future route
  that serialises a challenge wholesale. `/api/speakers/{id}/skg` is asserted to
  still return it — that route is the enrolment editor, and it doubles as the
  positive control proving the sweep can see.

Still open:

- The **Export PNG** button exports PNG, not SVG. cytoscape renders to canvas
  and SVG needs the `cytoscape-svg` extension; adding it is the fix if a vector
  figure is wanted.

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

8. **A normaliser fitted on dev has no statistics for test speakers.**
   `fit_cohort_normaliser(split.dev)` covered *zero* of the eleven test
   speakers, because `split_by_speaker` puts a trial in dev only when both its
   speakers are dev speakers. Every test lookup fell back to (mean 0, std 1),
   so cohort z-norm was reported on the test table and did not happen. Fixed by
   keeping the discarded cross-boundary trials on `Split.cohort` and fitting
   from those whose *probe* is a dev speaker — see `cohort_fitting_trials` for
   why the other half of that cohort is still excluded. **The general trap: a
   lookup with a silent default cannot tell you it missed.**

9. **An ablation measured on the wrong scope reads +0.00 and the zero lies.**
   Every CSBG scoring ablation showed no effect on the fused system, because
   the knowledge branch separates far better and pins the EER. The rows only
   mean something measured on the branch they change. `AblationRow.scope`
   records which system each row is an EER *of*, and the markdown prints it.

10. **A seeded RNG must not mint identifiers.** `run_attack` built its run id
    from the same seeded `random.Random` that makes its scores reproducible, so
    attacking a second speaker with the same attack type produced a duplicate
    primary key and a 500 — on the one workflow the Attack Lab exists for.
    Reproducible scores are the property worth having; a reproducible id is a
    collision.

11. **Escape LaTeX in one pass, not with chained `str.replace`.** Escaping `\`
    first yields `\textbackslash{}`, and a later `{` rule then escapes the
    braces of that replacement. Reordering only moves the collision, because
    `~` and `^` expand to braces too. See `experiments._TEX_ESCAPES`.

12. **The mock hides the degraded path.** Three separate frontend bugs were
    invisible in `VITE_USE_MOCK=true` because the mock always returns a
    populated, well-formed payload: an empty `models` array crashed the whole
    app, a hardcoded `spk_001` made the Graph Explorer draw nothing, and the
    Overview printed invented figures. **Run the UI against a degraded backend
    before demoing it**, not just against a healthy one.

---

## Standing instructions from the user

- Push to `PremKxmar/speech.git` at every step.
- The UI lives in `kavach/` and is user-supplied. Extend it; do not replace it.
- **No AI-looking design** — no neon colours, no gradient-heavy dashboards. This
  applies to figures too.
- The deadline is the user's concern, not the assistant's. Build the project.

---

## Answered, and what followed from each

1. **How many speakers?** ~25–30, over multiple sessions. The corpus layer and
   `RECORDING_PROTOCOL.md` are written to that number: ≥25 for usable EER
   intervals, ≥2 sessions per speaker weeks apart for §5.3, and
   `Corpus.reportability()` names every single-session speaker because the
   second sitting is the thing that cannot be recovered later.

2. **ANTHROPIC_API_KEY?** Not available yet. So `Corpus.reportability()`
   counts `n_guessed_tokens` and refuses a rules-only annotation, using the
   same rule `PipelineStats.is_corpus_grade` already applies rather than
   inventing a second standard.

3. **Voice cloner?** Not in scope yet. A3–A5 acoustic scores stay drawn from a
   documented distribution, `paper_ready()` keeps refusing those rows, and the
   experiment runner adds its own blocker whenever stand-in branches are used.

## Still open

1. **Which Whisper checkpoint** — see task 3 above. This one needs a decision
   before annotation starts, because it changes every token downstream.

2. **Is a resource-paper track available at SPELLL-2026?** §5.4 treats the
   corpus as the insurance policy; whether it can be submitted as its own
   contribution changes how much rides on the CSBG result.
