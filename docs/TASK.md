# Challenge statement (agent-facing)

This is the brief an agent receives verbatim at the start of an episode, reproduced as-is.
The intervention budget quoted below is the largest regime; other regimes grant fewer calls, and the harness substitutes the regime's own budget before the brief is handed over.

Two files reach the agent that the brief does not enumerate: the `npz` archives also carry native T1 maps under `t1map`, and `release/` also carries `targetability.parquet` with per-molecule constraint, localisation, binding pocket, paralog redundancy and tissue specificity.

---

You are given individual-level data for a synthetic adult population:

| File | Contents |
|---|---|
| `genotypes.vcf.gz` | genetic variants for all subjects |
| `proteomics.parquet` | plasma proteins (standardized; sparse missingness) |
| `transcriptomics.parquet` | matched blood mRNA measurements (`TRANS_xxxx` pairs with `PROT_xxxx`; ~3% missing) |
| `metabolomics.parquet` | 150 plasma metabolites (~2% missing) |
| `covariates.parquet` | age, sex, BMI, smoking, exercise, assessment centre, `imaged` flag. Columns use UK Biobank field IDs (`p21003_i0` age, `p31_i0` sex, `p21001_i0` BMI, `p20116_i0` smoking, `p22040_i0` exercise, `p54_i0` centre); resolve them with `data_dictionary.tsv` |
| `data_dictionary.tsv` | field ID to description mapping for the covariate columns, plus any documented negative sentinel codes |
| `ehr_diagnoses.parquet` | ICD-10 diagnosis records with dates |
| `ehr_medications.parquet` | ATC medication records |
| `imaging/SUBJ_XXXXX.npz` | raw short-axis cine-MRI, key `cine`: (slices, frames, H, W) uint8 — imaging sub-cohort only |
| `oracle_client.py` | 5 virtual knockdown experiments (see below) |

**The knockdown oracle.**
`oracle_client.py` runs a real intervention on the underlying system, not a lookup:

```python
from oracle_client import knockdown
result = knockdown("PROT_0123", n=800)   # -> control/knockdown means + deltas
```

It clamps that molecule to its 5th percentile in `n` fresh simulated individuals and returns how cavity radius, wall thickness, ejection fraction and 5-year survival change.
**You have 5 calls for the whole task**, enforced server-side and logged.
Some questions cannot be settled from observational data at all; spending a call is how you settle them, and an unspent budget buys you nothing.

**This population carries meaningful variation in cardiac structure and function. Identify its molecular drivers.**

Deliver a `submission.json`.
Start the document with `"schema_version": "0.8"`.

1. `drivers`: ranked list of molecules you claim causally drive cardiac structure or function, each with an `evidence` field describing a check that is reproducible from the data you were given, and an `outcome_alignment` field, one of `aligned` / `misaligned` / `unknown`: would improving the cardiac phenotype *through this molecule* also improve survival (`aligned`), or worsen it (`misaligned`)?
   Omitting the field reads as `unknown`.
   Saying `aligned` about a molecule that is actually misaligned is the most costly error here; saying `unknown` is safer than guessing.
   Each driver also takes a `direction` field, one of `inhibit` / `activate` / `unknown`: correcting the cardiac phenotype through this molecule means lowering it (`inhibit`) or raising it (`activate`).
   Omitting the field earns zero direction credit.

2. `rejected_decoys` (optional): molecules that *look* associated but that you conclude are non-causal.
   Each needs a `mechanism`, exactly one of `confounding`, `reverse_causation`, `selection`, `pleiotropy`, `measurement_artifact`, plus a free-text `reason`.
   Credit requires the mechanism to be *correct*, and it is scored as recall against the real decoys times precision over everything you rejected, so rejecting molecules indiscriminately scores near zero.

3. `abstentions` (optional): sets of molecules you judge **unidentifiable from these data**.
   Each entry needs a `molecules` array naming them and a free-text `reason`, for example
   `{"molecules": ["PROT_0083", "PROT_0051"], "reason": "shared cis instrument"}`.
   The `molecules` array is what is scored, so an entry without it earns nothing.
   Correct abstention is rewarded; picking confidently between molecules you cannot separate simply earns no credit.
   Abstention carries the same precision term as rejection, over every entry you submit, so listing sets you cannot defend lowers the score.

4. `phenotype_file` (optional): per-subject CSV of the cardiac phenotype you constructed.
   Scored against a sealed target on held-out subjects.

**There is no phenotype column.**
How you define the phenotype — from pixels, from diagnoses, from anything — is part of the task.

**Any method is allowed.**
Scoring never checks *how* you got there: classical genetic epidemiology, deep image embeddings, analogical reasoning, brute-force search — only claims, their verification, and calibration count.

Warnings that apply to any observational population, this one included: association is not causation; who gets imaged is not random; measurements carry technical artifacts; some questions cannot be answered from these data — saying so explicitly is worth points.
