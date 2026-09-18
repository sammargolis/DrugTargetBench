# Testing

Reserved for the evaluation-side artifacts needed to reproduce a published table rather than to run a new sweep.

The intended contents:

| path | purpose |
|---|---|
| `panel/` | the frozen panel manifest and its provenance record |
| `golden/` | the golden rubric test matrix, evaluated independently of the scorer |
| `reproduce/` | scripts that rebuild the published score table from the episode logs |
| `integrity/` | truth isolation, score determinism, and oracle parity checks |

Empty in this release.
Reproduction material is published with the dataset links.
