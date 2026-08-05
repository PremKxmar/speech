# KAVACH

**K**nowledge-graph **A**nchored **V**oice **A**uthentication for **C**ode-switched,
**H**ybrid-language speakers.

A speaker-verification system for Tamil–English code-mixed speech whose research
contribution is the **Code-Switch Behaviour Graph (CSBG)**: a per-speaker graph
over 21 semantic concept classes recording *which language this speaker chooses
for which subject*, scored as a log-likelihood ratio against a universal
background model.

The claim the whole system is built to test is one sentence:

> a code-switching profile is hard to steal even when the voice and the secret
> have both already been stolen.

## Where to start

| Document | What it is for |
|---|---|
| **[HANDOFF.md](HANDOFF.md)** | Current state, what is left, and the traps already hit. Read this first if you are picking the work up. |
| **[PROJECT.md](PROJECT.md)** | What the system is, every design decision and why, what has been measured versus assumed, and the full session history. |
| `KAVACH_Project_Idea.md` | The research proposal: novelty argument, experiment design, findings write-up. |

**One thing to know before reading any number in this repository:** no human
speakers have been recorded. Everything currently runs on `simulation.py`, whose
speakers differ by construction. See PROJECT.md §4.

---

## Layout

```
backend/kavach/         the system
    csbg/               ontology, graph construction, LLR scoring, metrics
    lid/                word-level language ID and semantic tagging
    attacks/            the A1–A5 threat model and its detectors
    eval/               EER, minDCF, DET curves, ablations, figures
    api/                FastAPI layer serving the frontend
    corpus.py           recorded-speech manifest, loader, elicitation protocol
    experiments.py      one command producing every table and figure
    audio.py asr.py embedding.py matcher.py skg.py challenge.py fusion.py
kavach/                 the frontend (Vite + React + TypeScript)
tests/                  600 tests, none of which need a GPU
```

Recording a corpus? Read **[RECORDING_PROTOCOL.md](RECORDING_PROTOCOL.md)**
before contacting a participant — consent cannot be applied retroactively.

---

## Running it

### Backend

```bash
pip install -r requirements-core.txt      # seconds; no torch
uvicorn kavach.api.app:app --reload --port 8000 --app-dir backend
```

That starts in **degraded mode**: without `speechbrain` and `faster-whisper`
there is no acoustic branch and no transcript, so those branches report
themselves *unmeasured* rather than scoring zero. `GET /api/health` lists what
loaded and what did not. The corpus, the graph explorer and the attack lab all
work in this mode.

For the full system:

```bash
pip install -r requirements.txt           # torch, speechbrain, whisper, LaBSE
export ANTHROPIC_API_KEY=...              # semantic tagging + challenge generation
```

Configuration is `backend/kavach/config.py`, overridable by environment
variable with a `KAVACH_` prefix or a `.env` file. Every threshold in there is
a **reasoned starting point, not a fitted value** — `eval/` fits them on a dev
split, and the fitted numbers are what a paper reports.

### Frontend

```bash
cd kavach
cp .env.example .env.local                # VITE_USE_MOCK=false
npm install
npm run dev                               # http://localhost:3000
```

With `VITE_USE_MOCK=true` it runs entirely on `src/api/mock.ts` and needs no
backend. That is a design preview, not the system: the mock's numbers are
hand-written, and its `expectedAnswerEntity` field carries the real challenge
answer, which the backend deliberately never sends.

### Tests

```bash
pytest                                    # 600 passed, ~36s
pytest -m models                          # only the tests needing real checkpoints
```

**Python 3.10+ is required.** On 3.9 every test file fails at collection
(`dataclass(slots=True)`). If the system interpreter is older:

```bash
uv venv --python 3.11 .venv
uv pip install --python .venv/bin/python -r requirements-core.txt
.venv/bin/python -m pytest
```

### The offline evaluation

```bash
python -m kavach.experiments --out paper/results/
```

Writes `results.json` (the paper reads from here — no number is hand-typed into
LaTeX), `report.md`, `tables/*.tex` and `figures/*.pdf`, alongside the git
commit and every seed. Without `--manifest` it runs on `simulation.py`, and
**every output is stamped unreportable** — in the JSON, in the README it
writes, and as a banner inside each `.tex` file.

---

## Things that are easy to get wrong here

**The demo numbers are not results.** The Attack Lab's acoustic scores are
modelled, not measured — no voice cloner ships with this — so every run carries
`simulated: true` and `AttackTable.paper_ready()` refuses the row. The
Evaluation page scores whatever logins happened to be clicked through, with no
trial design and no dev/test split. Both say so in their own module docstrings.
The reportable numbers come from `eval/` run offline.

**`/api/challenge` does not return the answer.** The frontend's TypeScript says
it does. Filling that field in would let whoever is authenticating read the
expected answer out of the network tab, which does not weaken the knowledge
branch so much as delete it. `Settings.demo_reveal_answers` exists for offline
demos, defaults to `False`, and is reported in `/api/health` so a demo build
announces itself.

**An unmeasured branch is not a failed branch.** A missing model scoring `0.0`
is indistinguishable from an impostor scoring `0.0`, and the first would look
like a working defence in exactly the table this project exists to produce.
`fusion.fuse` renormalises over the branches that ran.

**`data/` holds voiceprints next to hometowns, schools and family names.** It is
git-ignored, it is not encrypted, and a deletion request must remove the audio
and the knowledge graph, not just the speaker row (`Store.delete_speaker`).

**The mock has no failure modes, so run the UI against a *degraded* backend.**
Three frontend bugs survived until the first real run because `mock.ts` always
returns a populated, well-formed payload — one of them crashed every page in
the application when no model was loaded. See PROJECT.md §5.8.
