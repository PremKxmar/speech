## Verification results

Split: dev: 2 speakers / 20 trials | test: 2 speakers / 20 trials | cohort: 40 trials (z-norm only)
Branches measured: speaker_embedding, csbg

```
weights    : speaker_embedding=0.742, csbg=0.258
             (logistic regression on dev)
threshold  : 0.8186 (dev EER operating point)
veto       : none
```

| Configuration | EER % [95% CI] | minDCF | FAR@1%FRR | FRR@1%FAR | n gen/imp |
|---|---|---|---|---|---|
| ECAPA alone | 0.00 [0.00-0.00] | 0.0000 | 0.00 | 0.00 | 10/10  [!] |
| CSBG alone | 30.00 [10.00-52.62] | 0.7000 | n/a | 70.00 | 10/10 (4 vetoed)  [!] |
| + CSBG only | 0.00 [0.00-0.00] | 0.0000 | 0.00 | 0.00 | 10/10  [!] |

`[!]` marks a configuration with fewer than 30 trials on a side; its interval is too wide to compare against another row.

minDCF parameters: p_target=0.05, c_miss=1.0, c_fa=1.0.

### Ablations

**Scope is not decoration.** Each Δ is against the un-ablated baseline *of its own scope*, and rows in different scopes are not comparable with each other. Scoring ablations are measured on the branch they change, because a branch that another one dominates shows +0.00 for every switch and the zero says nothing about the switch -- see `ablate_scoring`.

| Removed | Scope | EER % | Δ EER | Note |
|---|---|---|---|---|
| equal branch weights | full system | 10.00 | +10.00 | What the logistic-regression fit was worth. |
| low-signal classes included | csbg | 40.00 | +10.00 | FUNCTION_WORD, NAMED_ENTITY and OTHER put back. Excluding them is a hypothesis stated in ontology.LOW_SIGNAL_CLASSES; a positive delta confirms it on this data, a negative one retires it. |
| transition stream removed | csbg | 40.00 | +10.00 | P(lang given class and prev_lang) dropped, its weight redistributed. Transition cells are ~4x sparser than lexical ones, so this is the row that says whether sequence information survives short probes. |
| graph metrics removed | csbg | 30.00 | +0.00 | CMI and I-index dropped. They are already ramped down on short probes by csbg.scoring, so a near-zero delta here is expected and is a check on that ramp rather than a finding. |
| lexical stream removed | csbg | 40.00 | +10.00 | Transitions and metrics alone. The complement of the row above: together they say how much of the CSBG is carried by which language a class takes versus how the speaker moves between them. |
| cohort z-norm removed | csbg | 20.00 | -10.00 | Raw LLRs are not comparable across claimed speakers: an unusual speaker produces large-magnitude scores for everyone. Expect this row to read ~0 on a single branch and still matter in fusion. EER is rank-based, so a per-speaker rescale only moves it by *reordering* speakers against each other -- which needs the population to vary in score scale in the first place. Fusion is not rank-based: it adds the branch to others on a shared [0, 1] scale, where the shift is exactly what makes the sum comparable. A near-zero delta here alongside a large one in the configuration table is the expected shape, not a contradiction. |

### CSBG stability against enrolment budget

Read left to right: how much speech a defender needs. Read as an attacker's eavesdropping budget: how much they need to steal it.

| Utterances | ~seconds | EER % [95% CI] |
|---|---|---|
| 2 | 52 | 25.00 [12.50-37.50] |
| 5 | 130 | 18.75 [10.42-31.25] |

### Caveats

- CSBG scores are cohort z-normalised. Statistics are fitted from impostor trials whose *probe* is a dev speaker, which covers test models (2/2 test speakers) without using a test probe -- see `cohort_fitting_trials`. Report un-normalised results too: z-norm usually helps materially, and hiding that is hiding a design decision.
- Branches not measured on this run: knowledge. Rows that would have used them are omitted rather than scored, so this table compares fewer systems than the paper's -- it is not a partial version of it.
- No veto floor bought any FAR reduction inside the 2% FRR budget on dev. Report that the veto was fitted and discarded -- that is a result about the branch, not an omission.

## Corpus coverage

4 speakers | 3304 tokens (2729 language choices)

A class needs about 4 observations before a speaker's own evidence outweighs the backoff prior (SmoothingConfig.class_alpha).

| Class | Tokens | Speakers with own evidence |
|---|---|---|
| NUMBER | 164 | 4/4 |
| TIME_DATE | 221 | 4/4 |
| KINSHIP | 78 | 4/4 |
| FOOD | 127 | 4/4 |
| PLACE_LOCAL | 104 | 4/4 |
| PLACE_GLOBAL | 9 | 1/4 |
| TECH_DIGITAL | 18 | 2/4 |
| EDU_WORK | 87 | 4/4 |
| MONEY_COMMERCE | 61 | 4/4 |
| EMOTION_STATE | 39 | 4/4 |
| BODY_HEALTH | 23 | 3/4 |
| TRANSPORT | 23 | 3/4 |
| RELIGION_FESTIVAL | 25 | 4/4 |
| MEDIA_ENTERTAIN | 69 | 4/4 |
| DISCOURSE_MARKER | 100 | 4/4 |
| POLITENESS | 0 | 0/4 |
| QUANTITY_MEASURE | 205 | 4/4 |
| ACTION_VERB | 416 | 4/4 |
| FUNCTION_WORD | 735 | 4/4 |
| NAMED_ENTITY | 3 | 0/4 |
| OTHER | 222 | 4/4 |

| Prompt | Utterances | Mean tokens | Target hit rate |
|---|---|---|---|
| p01_family | 4 | 54.2 | 75% |
| p02_food | 4 | 54.0 | 75% |
| p03_commute | 4 | 72.8 | 100% |
| p04_money | 4 | 74.0 | 100% |
| p05_phone | 4 | 77.8 | 100% |
| p06_study | 4 | 77.2 | 100% |
| p07_festival | 4 | 65.5 | 100% |
| p08_travel | 4 | 56.2 | 100% |
| p09_film | 4 | 73.8 | 100% |
| p10_numbers | 4 | 38.8 | 100% |
| p11_health | 4 | 52.8 | 100% |
| p12_market | 4 | 63.0 | 100% |
| p13_hometown | 4 | 60.5 | 100% |
| p14_control_name | 4 | 5.5 | 100% |
