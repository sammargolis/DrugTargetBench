# Training

Empty. Nothing in this directory is released yet.

Reserved for the reinforcement-learning and expert-iteration configs that turn the generator into a practice environment rather than a sealed one-shot evaluation.
The generator supports unlimited world generation with dense per-behaviour feedback, which is what makes training against it possible without exhausting a fixed panel.

Planned contents:

| path | purpose |
|---|---|
| `tier1_bandit/` | single-turn action-selection configs |
| `tier2_rollout/` | multi-turn rollout against a live oracle, one world per sandbox |
| `serving/` | serving configs for the open-weights arms |
| `grpo/` | GRPO configs and the reward definition |
| `expert_iteration/` | expert-iteration loop configs |

These land with the environment release, not before.
