## Verification results

Split: dev: 3 speakers / 45 trials | test: 4 speakers / 80 trials | cohort: 120 trials (z-norm only)
Branches measured: speaker_embedding, csbg

```
weights    : speaker_embedding=0.646, csbg=0.354
             (logistic regression on dev)
threshold  : 0.8186 (dev EER operating point)
veto       : none
```

| Configuration | EER % [95% CI] | minDCF | FAR@1%FRR | FRR@1%FAR | n gen/imp |
|---|---|---|---|---|---|
| ECAPA alone | 0.00 [0.00-0.00] | 0.0000 | 0.00 | 0.00 | 20/60  [!] |
| CSBG alone | 28.33 [18.33-40.00] | 0.9500 | n/a | 95.00 | 20/60 (16 vetoed)  [!] |
| + CSBG only | 0.00 [0.00-0.00] | 0.0000 | 0.00 | 0.00 | 20/60  [!] |

`[!]` marks a configuration with fewer than 30 trials on a side; its interval is too wide to compare against another row.

minDCF parameters: p_target=0.05, c_miss=1.0, c_fa=1.0.

### Ablations

**Scope is not decoration.** Each Δ is against the un-ablated baseline *of its own scope*, and rows in different scopes are not comparable with each other. Scoring ablations are measured on the branch they change, because a branch that another one dominates shows +0.00 for every switch and the zero says nothing about the switch -- see `ablate_scoring`.

| Removed | Scope | EER % | Δ EER | Note |
|---|---|---|---|---|
| equal branch weights | full system | 3.33 | +3.33 | What the logistic-regression fit was worth. |
| low-signal classes included | csbg | 25.00 | -3.33 | FUNCTION_WORD, NAMED_ENTITY and OTHER put back. Excluding them is a hypothesis stated in ontology.LOW_SIGNAL_CLASSES; a positive delta confirms it on this data, a negative one retires it. |
| transition stream removed | csbg | 28.33 | +0.00 | P(lang given class and prev_lang) dropped, its weight redistributed. Transition cells are ~4x sparser than lexical ones, so this is the row that says whether sequence information survives short probes. |
| graph metrics removed | csbg | 28.33 | +0.00 | CMI and I-index dropped. They are already ramped down on short probes by csbg.scoring, so a near-zero delta here is expected and is a check on that ramp rather than a finding. |
| lexical stream removed | csbg | 30.00 | +1.67 | Transitions and metrics alone. The complement of the row above: together they say how much of the CSBG is carried by which language a class takes versus how the speaker moves between them. |
| cohort z-norm removed | csbg | 26.67 | -1.67 | Raw LLRs are not comparable across claimed speakers: an unusual speaker produces large-magnitude scores for everyone. Expect this row to read ~0 on a single branch and still matter in fusion. EER is rank-based, so a per-speaker rescale only moves it by *reordering* speakers against each other -- which needs the population to vary in score scale in the first place. Fusion is not rank-based: it adds the branch to others on a shared [0, 1] scale, where the shift is exactly what makes the sum comparable. A near-zero delta here alongside a large one in the configuration table is the expected shape, not a contradiction. |

### CSBG stability against enrolment budget

Read left to right: how much speech a defender needs. Read as an attacker's eavesdropping budget: how much they need to steal it.

| Utterances | ~seconds | EER % [95% CI] |
|---|---|---|
| 2 | 50 | 25.00 [20.24-32.14] |
| 5 | 125 | 28.57 [18.59-32.74] |

### Caveats

- CSBG scores are cohort z-normalised. Statistics are fitted from impostor trials whose *probe* is a dev speaker, which covers test models (4/4 test speakers) without using a test probe -- see `cohort_fitting_trials`. Report un-normalised results too: z-norm usually helps materially, and hiding that is hiding a design decision.
- Branches not measured on this run: knowledge. Rows that would have used them are omitted rather than scored, so this table compares fewer systems than the paper's -- it is not a partial version of it.
- No veto floor bought any FAR reduction inside the 2% FRR budget on dev. Report that the veto was fitted and discarded -- that is a result about the branch, not an omission.

## Corpus coverage

7 speakers | 6495 tokens (5442 language choices)

A class needs about 4 observations before a speaker's own evidence outweighs the backoff prior (SmoothingConfig.class_alpha).

| Class | Tokens | Speakers with own evidence |
|---|---|---|
| NUMBER | 352 | 7/7 |
| TIME_DATE | 434 | 7/7 |
| KINSHIP | 141 | 7/7 |
| FOOD | 238 | 7/7 |
| PLACE_LOCAL | 173 | 7/7 |
| PLACE_GLOBAL | 13 | 1/7 |
| TECH_DIGITAL | 36 | 5/7 |
| EDU_WORK | 150 | 7/7 |
| MONEY_COMMERCE | 97 | 7/7 |
| EMOTION_STATE | 76 | 7/7 |
| BODY_HEALTH | 50 | 6/7 |
| TRANSPORT | 67 | 6/7 |
| RELIGION_FESTIVAL | 43 | 7/7 |
| MEDIA_ENTERTAIN | 112 | 7/7 |
| DISCOURSE_MARKER | 133 | 7/7 |
| POLITENESS | 0 | 0/7 |
| QUANTITY_MEASURE | 348 | 7/7 |
| ACTION_VERB | 687 | 7/7 |
| FUNCTION_WORD | 1919 | 7/7 |
| NAMED_ENTITY | 3 | 0/7 |
| OTHER | 370 | 7/7 |

| Prompt | Utterances | Mean tokens | Target hit rate |
|---|---|---|---|
| p01_family | 7 | 69.9 | 86% |
| p02_food | 7 | 66.1 | 86% |
| p03_commute | 7 | 77.3 | 100% |
| p04_money | 7 | 79.0 | 100% |
| p05_phone | 7 | 78.3 | 100% |
| p06_study | 7 | 81.4 | 100% |
| p07_festival | 7 | 76.3 | 100% |
| p08_travel | 7 | 68.6 | 100% |
| p09_film | 7 | 77.7 | 100% |
| p10_numbers | 7 | 39.0 | 100% |
| p11_health | 7 | 66.4 | 100% |
| p12_market | 7 | 72.0 | 100% |
| p13_hometown | 7 | 70.1 | 100% |
| p14_control_name | 7 | 5.7 | 100% |
