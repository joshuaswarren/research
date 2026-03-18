# What Makes This Novel

## The Distinction: Correction Accumulation vs. Miscalibration Diagnosis

### What existing systems do: Correction Accumulation

Every system that "learns from mistakes" works the same way: when the user corrects the agent, store the correction. Next time, read the corrections before acting. This is **accumulation** — a growing list of "don't do X" rules.

Examples:
- Context Studios: "Every time a human corrects an agent, the correction gets appended to a file with a timestamp, and the agent reads this file before every run"
- Letta/MemGPT: Self-editing memory where the agent updates its own state after corrections
- Engram's existing corrections: 1,002 correction memories stored as individual facts

The problem with accumulation: the list grows forever, rules are disconnected from each other, and the system never understands WHY it keeps making similar mistakes. Having 1,002 individual corrections doesn't help if the same TYPE of mistake keeps happening in new contexts.

### What our system does: Miscalibration Diagnosis

Instead of accumulating individual corrections, we analyze the CORPUS of corrections to identify PATTERNS of systematic miscalibration between the model and the user.

The key question isn't "what did the user correct?" — it's "WHERE is the model's mental model of the user systematically wrong?"

The output isn't "don't do X" rules. It's calibration adjustments:
- **Condition**: When does this miscalibration trigger?
- **Model tendency**: What does the model tend to assume or do wrong?
- **User expectation**: What does the user actually want instead?
- **Calibration**: How should the model adjust?

### Concrete example from Josh's data

**Accumulation approach** (what exists today):
```
- Correction 1: "The task was about validating whether to archive, not expanding scope"
- Correction 2: "There are 10 target accounts, not 2"
- Correction 3: "The redesign was already implemented"
- Correction 4: "Don't expand scope beyond what was asked"
```
These sit as 4 disconnected facts. The model might avoid these exact mistakes but will make the same TYPE of mistake in a new context.

**Miscalibration diagnosis** (what our system produces):
```
CalibrationRule:
  condition: "When the user describes a task or when scope boundaries are ambiguous"
  modelTendency: "The model tends to expand scope and start implementing solutions directly"
  userExpectation: "The user wants narrow task definitions and explicit confirmation before expanding"
  calibration: "Before implementing changes directly, confirm scope. Do not assume broader scope than stated."
```

This ONE rule addresses ALL FOUR corrections — and any future correction of the same type. It's not "don't do X" — it's "you systematically misjudge this user's scope expectations."

## Why This Is Novel (vs. Prior Art)

| System | Stores corrections? | Clusters corrections? | Identifies root cause? | Produces calibration rules? |
|--------|:---:|:---:|:---:|:---:|
| Correction file (Context Studios) | Yes | No | No | No |
| MemGPT/Letta self-edit | Yes | No | No | No |
| Engram corrections | Yes | No | No | No |
| RLHF/DPO | (implicitly) | No | (via gradient) | (via weights) |
| **This system** | Yes | **Yes** | **Yes** | **Yes** |

The novel contribution is the middle two columns: clustering corrections to find patterns, and identifying the root cause of systematic miscalibration.

## The Biological Analogy

This maps to **cerebellar calibration** in neuroscience:

1. The cerebellum receives prediction errors (motor commands that didn't produce expected results)
2. It doesn't store individual errors — it updates a FORWARD MODEL of how the body works
3. Over time, the forward model becomes calibrated to the specific body's characteristics
4. Result: smoother, more accurate movements without conscious correction

Our system does the same thing for model-user interaction:
1. Receive prediction errors (user corrections)
2. Don't just store them — update a CALIBRATION MODEL of how this user thinks
3. Over time, the calibration model becomes specific to this user's expectations
4. Result: fewer corrections, better alignment, without retraining the model

## The Tesla FSD Parallel

Tesla's FSD reportedly runs in shadow mode even when disengaged. When there's a surprising difference between what the human driver did and what FSD would have done, that divergence is analyzed for training.

Our system does the same thing:
- The model makes predictions about what the user wants (via recall)
- When the user corrects (divergence from prediction), that's the signal
- Chains of similar divergences are analyzed during "sleep time"
- The result is a calibration adjustment, not a datapoint

## The Evaluation

We can measure this with a temporal holdout test:
1. Calibrate on early corrections (training set)
2. Test: would later corrections have been preventable? (LLM-as-judge)
3. Metric: % of future corrections that calibration rules would have prevented

This evaluation methodology is also novel — no existing benchmark measures "correction preventability."
