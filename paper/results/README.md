# Experiment run 2026-08-06T18:31:58+00:00

**These numbers are NOT reportable.**

- commit `e7472b7e7c13` **(working tree dirty)**
- python 3.11.15 on macOS-26.6-arm64-arm-64bit
- corpus `kavach_corpus_v2` (SCRIPTED), 7 speakers, 98 utterances
- seed 0, dev fraction 0.4, 500 bootstrap resamples
- word-level LID accuracy: **unmeasured** (no `--goldset`)
- acoustic (ECAPA): scored 2142 of 2142 trials (100.0%)
- knowledge (answer matcher): scored 0 of 2142 trials (0.0%)

## Why these numbers may not be reported

1. provenance is SCRIPTED: speakers read authored text, so a CSBG recovers the profile the script assigned; the acoustic and integrity branches are unaffected, but no §5.1 claim about natural code-switching follows from it
2. 1 of 98 utterances are excluded and contribute to no graph (1x Whisper translated it into English rather than transcribing the Tamil; three decoding attempts (auto x2, forced ta) all returned zero Tamil, so its tokens are language choices the speaker never made) -- report this count and these reasons alongside any result
3. 7 speakers have one session, so no cross-session trial exists for them and §5.3 cannot be computed on this corpus
4. the knowledge (answer matcher) branch scored no trials at all (knowledge (answer matcher): 0/2142 trials scored (0.0%); unmeasured: claimed speaker never answered this prompt at enrolment x2142), so every fusion row that names it was in fact fused without it
5. enrolment and probe speech were split within a session, which measures how well the estimator memorises one recording sitting

## Files

| Path | What it is |
|---|---|
| `results.json` | Every number. **The paper reads from here.** |
| `report.md` | The same run as prose tables |
| `tables/*.tex` | `tabular` bodies for `\input{}` |
| `figures/*.pdf` | Vector figures; `.png` beside each |

Nothing here is hand-edited. Re-run `python -m kavach.experiments` to regenerate.
