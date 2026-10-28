# Distilling harness-induced behaviors into smaller coding models

**Status:** training research design, not a trained model or benchmark result.

## Question
Can a local student learn the decision behaviors supplied by a stronger runtime harness—when to inspect files, recover from tool errors, check dependencies and verify completion—so that inference requires fewer expensive advisor calls? Harness-Zero (https://arxiv.org/abs/2609.24974) reports harness-to-weight transfer in several agent domains; this does not demonstrate the same outcome for coding tasks. Open-SWE-Traces (https://arxiv.org/abs/2606.16038) is relevant trajectory research.

## Data contract
Collect state→action examples only from trajectories with independent task or step verification. Mark `good_action`, `corrected_action`, `rejected_action`, or `unknown`; never treat an agent's self-report as ground truth. Preserve task, repository revision, tool schema, result pairing, intervention cause, verifier outcome and model identity. Exclude private data, credentials, benchmark gold patches and teacher outputs whose terms prohibit reuse. Split by entire repository and issue family to avoid near-duplicate leakage.

## Training method
Start with one narrow capability such as schema-error recovery or verifier timing. Build small, auditable instruction/tool-turn pairs and hard negatives. Train a reproducible LoRA/SFT adapter on an appropriately licensed open model; record base checkpoint hash, tokenizer, adapter configuration, optimizer, seed, data manifest and GPU hours. Freeze the adapter before final evaluation. Do not assume fine-tuning on one harness protocol transfers to another tool format.

## Factorial evaluation
Compare four cells: base model/base harness, base model/improved harness, distilled model/base harness, distilled model/improved harness. Use identical unseen repositories, tool surface, engine, token limits and independent verifiers. This separates weight gains from harness gains and detects interactions. Report verified task completion, false completions, tool errors, token/cache costs, wall time and training cost. A cheaper but less reliable student must be evaluated on cost **per solved task**, not cost per attempt.

## Economics and stopping
Break-even tasks = (data generation + training + failed experiment cost) / (baseline cost per task − student cost per task), only when the denominator is positive and quality is comparable. Stop if the student loses held-out capability, needs a larger rescue harness, or only memorizes task patterns. Treat no gain as a useful result.
