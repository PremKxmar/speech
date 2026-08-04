# KAVACH — Code-Switch Behaviour Graphs for Voice Biometric Security

**Target venue:** SPELLL-2026, 5th International Conference on Speech and Language
Technologies for Low-Resource Languages, Amrita Vishwa Vidyapeetham, Coimbatore,
17–19 December 2026. Theme: *Knowledge Graphs for low-resource languages using LLM
and Multimodal data.*

**Repository:** `PremKxmar/speech.git`

This document explains what the system is, why each design decision was made, and
what has actually been measured versus what is still assumed. Read
[HANDOFF.md](HANDOFF.md) first if you are picking the work up mid-stream.

---

## 1. The idea in one paragraph

A bilingual Tamil–English speaker does not choose between languages at random.
They have habits: kinship terms in Tamil, technology in English, numbers one way
when counting money and the other way when giving a phone number. Those habits are
stable within a person and different between people. **KAVACH models a speaker's
language-choice habits as a graph over semantic concept classes — the Code-Switch
Behaviour Graph (CSBG) — and uses it as a soft biometric factor alongside a
conventional voiceprint.**

The security argument is what makes this worth publishing. A voice cloner can
reproduce *how someone sounds* from thirty seconds of audio, and an attacker who
scrapes social media can learn *what someone knows*. Neither of those gives them
*how that person distributes languages across meaning*. That is the gap the CSBG
is aimed at, and attack A4 below is the experiment that tests it.

---

## 2. Architecture

```
audio ──► ASR (faster-whisper)  ──► word-level LID ──► semantic tagging ──► tokens
  │                                  (rules + LLM)      (21 concept classes)
  │                                                              │
  ├──► ECAPA-TDNN embedding ──────────────────────┐              ▼
  │                                               │        CSBG (per speaker)
  ├──► signal-integrity tests ────────────┐       │              │
  │    (splice / duplicate artefacts)     │       │              ▼
  │                                       │       │      log-likelihood ratio
  └──► challenge freshness ───────┐       │       │       vs. background model
                                  │       │       │              │
                                  ▼       ▼       ▼              ▼
                              ┌─────────────────────────────────────┐
                              │  GATES          WEIGHTED BRANCHES   │
                              │  liveness       speaker_embedding   │
                              │  integrity      csbg                │
                              │                 knowledge           │
                              └─────────────────────────────────────┘
                                            │
                                     ACCEPT / REJECT / BORDERLINE
```

### Repository layout

```
backend/kavach/
  csbg/            THE RESEARCH CORE
    ontology.py      21 semantic classes; which are elicitable; which are low-signal
    tokens.py        Token, UtteranceTokens — the unit everything downstream reads
    graph.py         CSBG construction, CMI, I-index, density
    scoring.py       log-likelihood ratio vs. UBM, cohort z-normalisation
    metrics.py       graph-level statistics
  lid/             word-level language ID + semantic tagging (rules, then LLM)
  attacks/         A1–A5 attack construction, splice/replay detection, IAPMR
  eval/
    metrics.py       EER, minDCF, DET, bootstrap CIs, Wilson intervals
    ablation.py      dev/test split, fitted weights, fitted veto, ablations
  api/             FastAPI layer serving the React UI
    schemas.py       Pydantic contract mirroring the frontend's types.ts
    converters.py    the single place domain ↔ wire conversion happens
    store.py         SQLite records + filesystem audio
    pipeline.py      the runtime: enrolment, challenge, verification
    attacks.py       the Attack Lab
    app.py           19 routes + audio serving
  integrity.py     signal-integrity gate (splice + duplicate detection)
  fusion.py        branch combination, gates, vetoes
  audio.py  asr.py  embedding.py  matcher.py  skg.py  challenge.py  simulation.py
kavach/            the React + Vite + TypeScript UI (user-supplied)
tests/             434 tests
```

---

## 3. Design decisions, and why

These are the decisions a reviewer will ask about. Each one is implemented and
tested; the file that owns it is named.

### 3.1 Unavailable is not zero — `fusion.py`

A branch that could not be measured is dropped and the remaining weights are
renormalised. Scoring it 0.0 would silently convert a missing measurement into
strong evidence of an impostor. The response records which branches actually
contributed.

### 3.2 Liveness is a gate, not a weight — `fusion.py`

An expired or reused challenge is not "some evidence against" — it is
disqualifying, because it is exactly what a replay looks like. Weighted fusion
would let a very strong voice match outvote it.

### 3.3 A weighted average cannot express a veto, and the threat model needs one — `fusion.py`

With weights (0.40, 0.30, 0.30) and threshold 0.55, a branch weighted 0.30 can move
the fused score by at most 0.30. An attacker who fools the acoustic branch (0.85)
and knows the answer (0.90) is already at 0.61 from those two alone — so **no** CSBG
score, not even zero, can pull the total below threshold. That is attack A4, the
one the CSBG exists to stop, and under plain weighted fusion the CSBG is
structurally incapable of stopping it.

The fix follows from what the score *is*: a log-likelihood ratio, so a strongly
negative value is affirmative evidence *against* identity, not weak support for it.
Averaging that away is a category error. `FusionPolicy.veto_thresholds` lets a
branch reject outright below a floor.

### 3.4 Signal integrity is a second gate — `integrity.py`

A spliced file carries a *perfect* voiceprint, because it is made of the victim's
real voice. Averaging a "who is this" score with a "was this edited" score means a
good enough voice buys tolerance for an edit, which is backwards. So integrity is
disqualifying, like liveness.

It is emitted for **every** configuration including the ECAPA-only baseline,
because edit-artefact tests need no challenge, no knowledge graph and no CSBG. The
consequence is that the A1 and A2 rows go flat across all four columns, and that
flatness is the finding: those attacks are stopped by signal evidence, identically
well with or without the research contribution.

### 3.5 Leave-one-out background model — `csbg/scoring.py`, `eval/ablation.py`

The UBM must exclude the claimed speaker. Including them makes the likelihood ratio
compare a speaker against a population containing themselves.

### 3.6 Dev/test split by speaker, never by trial — `eval/ablation.py`

`split_by_speaker` partitions speakers and **discards** trials that straddle the
boundary — roughly half the impostor trials at `dev_fraction=0.4`. Fewer trials is
cheaper than a leak.

### 3.7 Degraded mode is a first-class state — `api/pipeline.py`

Missing model → the branch reports `available=False` and `/api/health` names what is
missing and what each absence costs. An authenticate call in this mode returns
REJECT whose explanation contains "system failure" — a user told "your voice did not
match" when no model ran has been told something false.

### 3.8 The knowledge factor must not be in the response — `api/converters.py`

`types.ts` declares `Challenge.expectedAnswerEntity` and the UI mock fills it with
the real answer, which anyone authenticating could read out of the network tab. The
field stays so TypeScript validates, returns `""` by default, and is gated behind
`Settings.demo_reveal_answers` (default False) which `/api/health` reports so a demo
build announces itself.

---

## 4. What has been measured, and what has not

**This is the section to read before writing any number into a paper.**

| Component | Status |
|---|---|
| CSBG construction, LLR scoring, cohort z-norm | Real, tested |
| EER / minDCF / DET / bootstrap CIs | Real, tested |
| Fitted fusion weights, threshold, veto floor | Real, fitted on a dev split |
| Splice + duplicate detection | Real, calibrated on synthetic audio |
| ECAPA-TDNN embeddings | Model installed and verified (192-d, loads in 0.2 s) |
| **Speech corpus** | **Does not exist. Everything runs on `simulation.py`.** |
| **Voice cloning (A3–A5 acoustic scores)** | **Modelled from a documented distribution, not measured.** |
| LLM semantic tagging | Falls back to rules; not corpus-grade without an API key |

`AttackTable.paper_ready()` refuses every current row, by design.

**The single blocking gap for publication is the corpus.** No number in this
repository is an experimental result about human speakers, because no human
speakers have been recorded. `simulation.py` says so in its own docstring:
synthetic speakers differ *by construction*, so a model separating them proves the
code is correct, not that the hypothesis is true.

---

## 5. Findings worth writing up

Three of these came out of building the system and are documented in
`KAVACH_Project_Idea.md` §4.5.2 and §5.1.2.

### 5.1 A threshold is meaningless until you measure what the score does

**The CSBG veto floor.** Set to 0.15 by reasoning about the score scale. Connecting
the scorer to fusion for the first time showed it could never fire: the logistic
squash compresses hard past an LLR of −2, so a maximally-contradicting impostor
scores 0.29, not near zero. The floor sat *below the impostor mode*. It hid because
the fusion tests use hand-written branch scores and the scorer tests never look at
fusion — the constant lived in the module that could not check it. Now 0.35, with
`TestVetoCalibration` asserting it against what the real scorer emits.

**The integrity floor, independently.** Set to 0.55 by the same kind of reasoning.
Measured over 80 clean and 80 spliced recordings:

| floor | FRR | detection |
|---|---|---|
| 0.25 | 0.00% | 82.5% |
| 0.55 | 5.00% | 82.5% |

Identical detection, and one genuine speaker in twenty locked out by a gate no
other branch can overturn. The evidence is bimodal, so raising the floor across the
empty middle buys nothing and only reaches into the clean tail.

**The generalisable point:** a threshold expressed in units of a score is only
meaningful once you have measured what that score actually does. Two correct
modules with a constant passed between them is exactly where this hides.

### 5.2 A grid search will walk onto a knife edge

`calibrate_floor` sometimes returns exactly 0.200 — and the detectors emit fixed
evidence weights for categorical findings (0.9 for inserted digital silence, 0.8 for
a hard cut away from a pause), so those probes score exactly 0.10 and 0.20. The
comparison is `score < floor`, so a floor of exactly 0.20 catches *none* of the
orphan-click cases while looking optimal to the search. The floor must sit strictly
inside the gap between the highest categorical level (0.20) and the lowest clean
score (0.274).

### 5.3 Polishing a splice makes it easier to detect

| attacker | detection | d' |
|---|---|---|
| naive splice, cross-session | 82.5% | 2.60 |
| naive splice, same session | 63.7% | 1.66 |
| **careful** splice, cross-session | **88.8%** | **4.97** |

`SpliceConfig.careful()` inserts a room-tone pause so the join sounds natural — and
a pause at the join is precisely what the background test examines. The naive hard
cut has no pause, so nothing but the click detector ever looks at it. **The
attacker's best move against this defence is the crude one.** The number to quote is
the middle row: an attacker who records all their raw material in one sitting leaves
no background step, and a third of their attempts get through.

### 5.4 The A5 tension — the most interesting thing in the attack table

A5 is the style-adaptive attacker, who estimates the victim's CSBG from speech they
could overhear. The A5-observed curve measures *within-speaker consistency*, which
means: **the more reliably someone code-switches, the better the biometric works and
the more cheaply it is stolen.** The speech a defender needs to enrol a usable graph
is the speech an attacker needs to steal one. Written up in §5.1.2 with a suggested
scatter plot.

A5 runs are labelled ORACLE rather than OBSERVED when the eavesdropping budget
exceeds the victim's recorded speech, because an unbinding budget is the upper bound
wearing the observed row's name.

### 5.5 An unattainable operating point is not a rate

`FAR@1%FRR` was printing `100.00` for vetoed configurations — a "no such operating
point" sentinel rendered as a measured rate, sitting next to a baseline's `46.70`.
Anyone reading that table would conclude fusion is catastrophic at a point fusion
cannot occupy. Now `math.nan` → `format_rate()` → `"n/a"`, with the veto count beside
it.

Related: `_rates` anchored its threshold sweep at `min(all) − 1.0`, and with `−inf`
present that is `−inf`, and `−inf >= −inf` is True — so every vetoed trial was
accepted at the bottom of the sweep and the FRR floor vanished. The first fix
(substituting a low finite stand-in) was also wrong, because *any* finite value is
acceptable at a low enough threshold. The correct fix anchors from the lowest
*finite* score and passes `−inf` straight through.

---

## 6. Session history

### Session 1 — proposal and literature
Wrote `KAVACH_Project_Idea.md`. A literature search forced a revision of the novelty
claims (commit `13dc4fd`): code-switching as a *soft biometric under adversarial
conditions* is the defensible framing, not code-switching detection generally.

### Session 2 — the research core
`csbg/` (ontology, tokens, graph, scoring, metrics) and word-level LID with semantic
tagging. Commits `b3fcfe5`, `62b31f7`.

### Session 3 — audio, knowledge, decisions
`audio.py`, `asr.py`, `embedding.py`, `skg.py`, `matcher.py`, `challenge.py`,
`fusion.py`. Commit `f511d1f`.

### Session 4 — the attack suite
A1–A5, `attacks/`, and the veto path in fusion. This is where the CSBG veto floor
was found to be uncalibrated. Commits `e6b273c`, `50e478e`.

### Session 5 — the API
`api/` — all 19 endpoints transcribed from the UI's `client.ts` rather than the
reverse, since the UI is a given. Found and closed the `expectedAnswerEntity` leak.
Commit `ce34c77`, 380 tests.

### Session 6 — the offline evaluation
`eval/ablation.py`: all-pairs trials, split by speaker, weights → threshold → veto
fitted on dev only, bootstrap intervals, ablations, stability, fairness slices.
Found the two reporting bugs in §5.5. Commit `3a68d8d`, 413 tests.

Representative output on a 24-speaker **simulated** corpus:

```
Split: dev 10 speakers / 800 trials | test 14 speakers / 1568 trials
weights   : speaker_embedding=0.369, csbg=0.309, knowledge=0.322  (logistic regression on dev)
threshold : 0.5919 (dev EER operating point)
veto      : 0.450 on csbg -- removes 2.36% FAR, costs 1.25% FRR

| Configuration | EER % [95% CI]      | minDCF | FAR@1%FRR | FRR@1%FAR | n gen/imp             |
| ECAPA alone   | 21.98 [19.71-30.36] | 0.9286 | 46.70     | 85.71     | 112/1456              |
| CSBG alone    | 28.37 [23.21-32.49] | 0.9464 | n/a       | 86.61     | 112/1456 (373 vetoed) |
| + Knowledge   |  5.49 [3.57-6.56]   | 0.3901 | 5.49      | 28.57     | 112/1456              |
| + CSBG only   | 22.39 [18.75-26.79] | 0.9011 | n/a       | 83.04     | 112/1456 (373 vetoed) |
| Full fusion   |  3.71 [2.88-6.25]   | 0.3647 | n/a       | 21.43     | 112/1456 (373 vetoed) |
```

**Not a result.** Simulated speakers differ by construction.

### Session 7 — models, integrity, and an honest test environment (this session)

1. **Installed the heavy models.** speechbrain 1.1.0, faster-whisper 1.2.1,
   librosa 0.11.0. ECAPA-TDNN verified: 192-d embeddings, 0.2 s load from cache.
   The acoustic branch is now real rather than modelled.

2. **Built the signal-integrity gate** (`integrity.py`), wired it into
   `pipeline.verify()` before the models run, into `fusion.fuse()` as a second gate,
   and into the Attack Lab where A1 and A2 are now built as **real waveforms** from
   the victim's stored recordings and scored by the real detector. That column needs
   no simulation caveat. The Attack Lab also computes its own ablation every run —
   the same trials scored with the gate off — because a flat row of zeros does not
   say *why* it is flat.

3. **Calibrated the integrity floor empirically** (§5.1, §5.2, §5.3) and wrote
   `tests/test_integrity.py`, 21 tests whose entire purpose is the seam between the
   detectors and the constant that thresholds them.

4. **Found that the test suite was passing by accident.** It ran in ~48 s and never
   touched the network *because speechbrain and faster-whisper were not installed*.
   Nothing in the tests asked for that. Installing the models turned it into a run
   that tried to download a 3 GB Whisper checkpoint and never finished. Added
   `Settings.offline` and `tests/conftest.py` so degraded mode is declared rather
   than inherited, with `@pytest.mark.models` as the opt-in for real checkpoints.

5. **Fixed a phantom-field bug in the frontend contract test.** The TypeScript
   interface parser did not strip `//` comments, so a comment reading "gates, not
   weighted factors: they carry weight 0" injected a field named `factors`. Prose is
   not syntax.

6. **Fixed a latent flake in the challenge-leak test.** It asserted
   `expected_answer not in str(public_dict())`, and the dict carries Unix
   timestamps while an SKG fixture fact is the room number `"214"` — so the suite
   failed roughly once in several full runs on a collision with the clock. See
   HANDOFF.md trap 5.

Test count: **434 passing**, offline, in about 60 seconds, verified across three
consecutive full runs.

---

## 7. Running it

```bash
# Backend — core only, no heavy models, runs in degraded mode
pip install -r requirements-core.txt
pytest                                   # 434 tests, offline, ~60s

# Backend — everything
pip install -r requirements.txt
pytest -m models                         # the tests that load real checkpoints

uvicorn kavach.api.app:app --reload --port 8000 --app-dir backend

# Frontend
cd kavach
cp .env.example .env.local               # VITE_USE_MOCK=false
npm install
npm run dev                              # http://localhost:3000
```

`/api/health` reports which branches can run and what each absence costs.

Useful environment variables (all prefixed `KAVACH_`):

| Variable | Effect |
|---|---|
| `KAVACH_OFFLINE=1` | Load no model checkpoints; exercise degraded mode deliberately |
| `KAVACH_WHISPER_MODEL=small` | Smaller ASR checkpoint (default `large-v3`, ~3 GB) |
| `KAVACH_DEMO_REVEAL_ANSWERS=1` | Populate `expectedAnswerEntity`. **Demos only.** |
| `ANTHROPIC_API_KEY` | Enables LLM tagging and challenge generation |

---

## 8. Reading order for someone new

1. `KAVACH_Project_Idea.md` — the research proposal, the novelty argument, §4.5.2
   and §5.1.2 for the findings above.
2. `backend/kavach/csbg/ontology.py` — the 21 classes and why each is or is not
   elicitable. Everything else reads these.
3. `backend/kavach/csbg/scoring.py` — the likelihood ratio, which is the method.
4. `backend/kavach/fusion.py` — the module docstring is the argument for the veto.
5. `backend/kavach/integrity.py` — the argument for the second gate, and the
   calibration story.
6. `backend/kavach/eval/ablation.py` — how the paper's table is produced.

Module docstrings carry the reasoning throughout. Where a number was chosen rather
than measured, the docstring says so.
