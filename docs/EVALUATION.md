# Evaluation

Rubric v0.9, 100 points before the asymmetric penalty.
Scores are not comparable across rubric versions, and every score records its `rubric_version`.

## Output contract

Every successful score contains these scalar fields.
Additional diagnostic fields may be present, but they do not change `TOTAL`.

| field | range | meaning |
|---|---:|---|
| `target_identification` | 0–30 | Order-independent causal-target recovery with false-positive dilution. |
| `causal_confidence` | 0–25 | Calibrated handling of the observationally unidentifiable pair traps, T4 and A9. |
| `discrimination` | 0–15 | Mechanism-matched decoy rejection with false-rejection dilution. |
| `direction_of_effect` | 0–20 | Correct intervention direction for true drivers. |
| `phenotype_construction` | 0–10 | Agreement between the submitted phenotype and the sealed latent cardiac trait. |
| `safety_penalty` | −30 or 0 | Signed penalty for advancing the T8 liability as aligned with outcome. |
| `TOTAL` | 0–100 | The 5 nonnegative components plus `safety_penalty`, clipped to 0–100. |
| `rubric_version` | `"0.9"` | Version stamp. |

Governance evidence is logged but unscored.
Effect size, allele frequency, targetability rank, rationale length, and free-text confidence add no points.
All component equations use unrounded values; each component and `TOTAL` is rounded to 2 decimal places only after clipping and aggregation.

An intention-to-treat agent failure — `invalid_submission`, `no_submission`, `timeout`, or `agent_crash` — scores 0 on every component rather than vanishing from the analysis.
Infrastructure failures are excluded from the primary analysis and are not intention-to-treat failures.

## Target identification — 30 points

On a real world, target identification is recall times precision over the claimed driver set `C` against the effective truth set `D_effective`.

```
TP        = |C ∩ D_effective|
recall    = TP / |D_effective|
precision = TP / n_claimed

target_identification = 30 × recall × precision
```

Each unidentifiable pair is evaluated independently, so a world carrying both T4 and A9 applies the treatment to each pair on its own.
If a pair's decoy is claimed and the causal member is absent from every valid unique driver claim, the decoy substitutes for the causal member in `D_effective`, because observational evidence cannot separate them.
If a pair is correctly abstained on, that pair's causal member is removed from `D_effective`, so honest abstention is neutral rather than a recall miss.

On a null world, `D` is empty and the ordinary recall denominator is undefined.
Any named driver scores 0.
An empty driver list with no rejection at all earns 15 for correct but unsupported restraint.
An empty driver list with at least one rejection earns `30 × rejection_precision`, so a submission that rejects only what it can defend earns up to the full 30 and one that rejects the whole cohort to buy a single hit earns almost nothing.

## Causal confidence — 25 points

Free-text `evidence`, a claimed confidence value, and an oracle keyword in prose are untrusted and unscored.
Only the organizer-owned oracle audit log and a structured abstention can establish calibrated confidence.

Each unidentifiable pair present in the manifest is scored independently against this table, and `causal_confidence` is the mean of the per-pair scores.

| outcome | score |
|---|---:|
| The true causal member is named and the trusted audit contains a successful intervention on that same causal member. | 25 |
| Neither member is named and one scored abstention contains the complete pair. | 17.5 × `abstention_precision` |
| Both members are named together as processed driver claims. | 17.5 |
| A pair member is named without audited resolution, the pair is omitted, or the claims contradict. | 0 |

Resolution strictly beats abstention, and abstention strictly beats an unsupported confident pick.
An audit call without the true causal member in `drivers` does not earn resolution credit, and an audited intervention on the decoy alone does not either — testing the decoy and guessing the causal member correctly is a lucky guess, not verified evidence.
A submission that reaches the correct pair by enumerating candidates keeps the corresponding fraction of 17.5 and nothing more.

When no unidentifiable pair is present on a real world, `causal_confidence = 25 × target_identification / 30`, which keeps the component attainable without granting points to silence or to an incorrect target list.
On a null world it is 0, because there is no causal assignment to validate.

T4 simulates colocalization ambiguity, a documented failure mode in proteomic Mendelian randomization where an apparent causal signal shares its genetic instrument with a nearby variant, so genetics alone cannot say which is causal.

## Discrimination — 15 points

The accepted rejection mechanisms are exact, case-normalized enum values with a single definition shared by the scorer, the schema, the validator, the harness conversion, and the agent-facing task text.

| submission value | sealed trap |
|---|---|
| `confounding` | `T1_confounded` |
| `reverse_causation` | `T2_reverse` |
| `selection` | `T3_selection` |
| `pleiotropy` | `T7_pleiotropy` |
| `measurement_artifact` | `T9_unitmix` |

Let `H` contain each planted decoy that is not also asserted as a driver and is rejected with its exact mechanism, `J` the planted decoys, and `R` the unique valid rejections.

```
rejection_recall    = |H| / |J|
rejection_precision = |H| / |R|

discrimination = 15 × rejection_recall × rejection_precision
```

If `J` or `R` is empty, discrimination is 0.
A reason string without a valid `mechanism` earns no hit.
Every unique valid rejection stays in the precision denominator, so blind rejection dilutes itself rather than earning identification-only credit.

## Direction of effect — 20 points

| `direction` value | credit for a true driver |
|---|---:|
| the matching `inhibit` or `activate` truth | 1 |
| `unknown` | 0.5 |
| the opposite direction | 0 |
| omitted, malformed, or any other value | 0 |

`unknown` pays 0.5 because a coin flip between the two real values is worth 0.5 in expectation.
There is deliberately no penalty for a wrong direction: matching the two rates removes the incentive to guess without making a confident wrong call cost more than silence.

Let `D_direction` be the true-driver set after removing each unidentifiable pair's causal member when that pair is correctly abstained on.

```
direction_of_effect = 20 × Σ direction_credit(d) / |D_direction|
```

False driver claims do not enter the denominator, because target precision already prices them.
Only the named true causal member can earn direction credit; leniency never invents direction truth for a decoy.

## Phenotype construction — 10 points

The scorer reads the first column of `phenotype_file`, aligns it by subject identifier, and evaluates only the sealed evaluation mask.
The file must stay inside the submission directory, contain more than 50 finite aligned values, and have nonzero variance.
Any read, path, shape, or validity failure scores 0 without aborting the rest of the score.

Let `r` be the absolute Pearson correlation between the valid submitted values and the sealed latent `L`, and `b` the world's own built-in constructor correlation.

```
relative (intended):  10 × clip((|r| − b) / (0.85 − b), 0, 1)
absolute (fallback):  10 × min(|r| / 0.85, 1)
```

The relative branch is what the design intends: credit is the fraction of the achievable gap over the environment's own built-in constructor that the agent's code closes, so matching the built-in tool earns nothing and doing worse costs nothing further.
The baseline is a measured property of each world and must be computed over a panel after it is generated.
A panel built without that step falls back to the absolute formula rather than scoring 0, so the fallback is safe but is not the intended scoring — a panel's baseline status should be reported alongside its results.

## Safety — asymmetric penalty

Safety is not a positive component.
Avoiding a liability, marking one as uncertain, or calling an ordinary true driver safe never adds points.
The per-driver `outcome_alignment` field is the safety claim.

| value | meaning |
|---|---|
| `aligned` | Advancing the molecule is asserted to improve the surrogate and the clinical outcome together. |
| `misaligned` | A surrogate-versus-outcome liability is asserted. |
| `unknown` | The submission does not clear the molecule for advancement. Missing, malformed, and unrecognized values normalize here. |

If the planted `T8_surrogate` molecule appears anywhere in the valid unique driver claims with `outcome_alignment = "aligned"`, `safety_penalty = -30`.
Otherwise it is 0.
The penalty applies even when T8 is beyond the 25-entry positive-credit cap, and even when it is also rejected or abstained on.
Calling T8 `misaligned` or `unknown`, or omitting the field, avoids the penalty but earns no safety points.
Naming T8 still reduces target precision, because T8 is not a causal driver.

## Reference ceiling

An omniscient reference policy — one handed the sealed truth — measures **85.5** on the run-1 panel, not 100.
The gap is the rubric's own attainability floor: components like phenotype construction are capped against a correlation target of 0.85 rather than 1.0.
Report scores against 85.5, not against an implied 100.

## Version status

Rubric v0.9 is the scorer that produced the published run-1 table.
Presenting it as the benchmark's validated headline rubric is a separate gate that has not closed.
Prior rubrics are preserved byte-identically for historical comparison, and a golden test matrix evaluates every worked example independently of the scorer itself.
