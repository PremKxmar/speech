# Experiment run 2026-08-05T23:29:53+00:00

**These numbers are NOT reportable.**

- commit `0663a31b65af`
- python 3.11.15 on macOS-26.6-arm64-arm-64bit
- corpus `corpus_v1` (SCRIPTED), 4 speakers, 56 utterances
- seed 0, dev fraction 0.4, 500 bootstrap resamples
- word-level LID accuracy: **unmeasured** (no `--goldset`)
- acoustic (ECAPA): scored 720 of 720 trials (100.0%)
- knowledge (answer matcher): scored 0 of 720 trials (0.0%)

## Why these numbers may not be reported

1. provenance is SCRIPTED: speakers read authored text, so a CSBG recovers the profile the script assigned; the acoustic and integrity branches are unaffected, but no §5.1 claim about natural code-switching follows from it
2. 4 speakers have one session, so no cross-session trial exists for them and §5.3 cannot be computed on this corpus
3. the knowledge (answer matcher) branch scored no trials at all (knowledge (answer matcher): 0/720 trials scored (0.0%); unmeasured: claimed speaker never answered this prompt at enrolment x720), so every fusion row that names it was in fact fused without it
4. enrolment and probe speech were split within a session, which measures how well the estimator memorises one recording sitting

## Files

| Path | What it is |
|---|---|
| `results.json` | Every number. **The paper reads from here.** |
| `report.md` | The same run as prose tables |
| `tables/*.tex` | `tabular` bodies for `\input{}` |
| `figures/*.pdf` | Vector figures; `.png` beside each |

Nothing here is hand-edited. Re-run `python -m kavach.experiments` to regenerate.
