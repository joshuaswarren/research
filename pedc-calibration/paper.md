# Prediction-Error-Driven Calibration for Model-User Alignment in Agentic Memory Systems

**Joshua Warren**
March 2026

---

## Abstract

Long-term memory systems for AI agents store user corrections as individual facts, yet agents persistently repeat the same *types* of mistakes across sessions. We introduce Prediction-Error-Driven Calibration (PEDC), a mechanism that treats user corrections as prediction errors, clusters them to identify systematic miscalibration between model and user, and synthesizes structured CalibrationRules that address root causes rather than individual symptoms. We evaluate PEDC on 1,002 production corrections from a deployed agentic system (OpenClaw/Engram) using a temporal holdout methodology. In ablation, PEDC-generated calibration rules prevent 6.3x more future corrections than raw correction accumulation (4.4% vs. 0.7% on 136 held-out corrections). Analysis of the correction corpus reveals that 25% of corrections (254/1,002) are systematic and calibration-addressable, while 75% are one-off factual errors outside the scope of any calibration system. We discuss the analogy to cerebellar forward model calibration, compare against existing approaches including sleep-time consolidation and self-editing memory, and propose a novel evaluation metric: correction preventability rate.

## 1. Introduction

Agentic AI systems that interact with specific users over extended periods face a persistent challenge: they keep making the same types of mistakes. Not the same exact mistake — correction accumulation handles that — but the same *category* of error. An agent that expands task scope without asking will continue to do so in new contexts, even after being corrected dozens of times on specific instances, because each correction is stored as an isolated fact rather than analyzed as evidence of a systematic pattern.

This problem is distinct from the well-studied challenge of memory retrieval. Current systems (MemGPT/Letta [Packer et al., 2023], LightMem [2025], Nemori [2025]) excel at storing and retrieving information. Sleep-time consolidation approaches [Li et al., ICLR 2026; Sleep-Time Compute, 2025] reorganize and compress stored memories during offline periods. Self-editing memory [Packer et al., 2023] allows agents to update their own state. However, none of these systems analyze *where the model's predictions about the user are systematically wrong*.

We propose reframing user corrections as **prediction errors** — signals that the model's internal model of the user diverged from reality. A single prediction error is noise. A pattern of similar prediction errors across sessions reveals a systematic miscalibration that can be diagnosed and corrected.

Our contributions are:
1. **Problem formulation**: We distinguish between correction accumulation (storing individual corrections) and miscalibration diagnosis (identifying systematic patterns across corrections and producing structured calibration rules).
2. **PEDC system**: A pipeline that reads correction corpora, uses an LLM to cluster corrections by misunderstanding type, and synthesizes CalibrationRules with structured condition/tendency/expectation/calibration fields.
3. **Evaluation methodology**: A temporal holdout approach with LLM-as-judge scoring, plus an ablation comparing calibration rules against raw correction accumulation and random baselines.
4. **Production evaluation**: Results on 1,002 corrections from a deployed system serving a real user over 41 days.

## 2. Related Work

### 2.1 Agentic Memory Systems

MemGPT [Packer et al., 2023] introduced self-editing memory where an LLM manages its own memory hierarchy. Letta extends this with stateful agents that update memory after interactions. LightMem [ICLR 2026] introduces offline "sleep" periods for memory reorganization, deduplication, and abstraction. Nemori [2025] adds memory lifecycle management with creation, updating, retrieval, and deletion. MemoryOS [EMNLP 2025] provides operating-system-inspired memory management. These systems treat corrections as data to store and retrieve. None analyze corrections as evidence of systematic miscalibration.

### 2.2 Sleep-Time Consolidation

"Language Models Need Sleep" [Li et al., ICLR 2026] proposes a sleep paradigm where models transfer short-term in-context knowledge to long-term parameters via reinforcement learning-based knowledge seeding. Sleep-Time Compute [2025] uses idle time for pre-computation. LightMem's offline reorganization operates similarly. These approaches consolidate the *memory store* — reorganizing, compressing, and strengthening stored information. PEDC consolidates the *model-user relationship* — diagnosing where the model's understanding of the user is systematically wrong.

### 2.3 Learning from Corrections

Existing approaches to learning from corrections fall into three categories:

**Correction accumulation**: Appending corrections to a file or memory store that the agent reads before acting. This scales linearly with corrections and does not generalize across contexts. The approach described by Context Studios ("every time a human corrects an agent, the correction gets appended to a file") exemplifies this pattern.

**Self-editing memory**: The agent modifies its own memory state after a correction (Letta/MemGPT). Individual facts are updated, but no pattern analysis occurs across corrections.

**RLHF/DPO**: Reinforcement learning from human feedback adjusts model parameters based on preference signals. This operates at the model level, not the user level, and requires retraining.

PEDC differs from all three by analyzing corrections as a corpus, clustering by misunderstanding type, and producing structured rules that address root causes.

### 2.4 Neuroscience Inspiration

The prediction error framing draws loosely from cerebellar motor calibration. The cerebellum adjusts a forward model of the body based on prediction errors during motor execution [Wolpert & Ghahramani, 2000; Popa & Ebner, 2018]. Individual motor errors do not get stored; instead, the forward model is updated to reduce systematic bias. PEDC applies an analogous principle at the level of model-user interaction: individual corrections are not the lesson — the systematic bias revealed by a pattern of corrections is the lesson. We note that the mechanisms differ fundamentally (synaptic weight adjustment vs. LLM text analysis) and do not claim mechanistic correspondence.

## 3. Method: Prediction-Error-Driven Calibration

### 3.1 Overview

PEDC operates as an offline consolidation process within an agentic memory system. It reads the agent's stored corrections, sends them to an LLM for pattern analysis, and produces CalibrationRules that are injected into the agent's recall context during future interactions.

### 3.2 CalibrationRule Structure

Each CalibrationRule contains four fields:

```
CalibrationRule {
  condition:       "When this type of situation arises"
  modelTendency:   "The model tends to assume or do X"
  userExpectation: "But this user actually wants Y"
  calibration:     "Instead, the model should do Z"
}
```

This structure captures not just *what* to do differently, but *why* — the model's tendency and the user's expectation provide context that helps the model generalize to novel situations.

### 3.3 Consolidation Pipeline

**Step 1: Correction corpus ingestion.** All stored corrections are read from the memory system. Each correction is a text string describing what the user corrected (e.g., "The user corrected that the staging server is on proxmox2, not proxmox4").

**Step 2: LLM-assisted clustering and diagnosis.** Corrections are sent to an LLM in batches (40 corrections per batch in our implementation). The LLM prompt instructs it to:
- Group corrections by the *type of misunderstanding* they reveal
- For each cluster, identify the model's systematic tendency and the user's actual expectation
- Produce a CalibrationRule for each identified pattern

**Step 3: Rule deduplication and merge.** Rules are deduplicated by SHA-256 hash of their condition + calibration fields. Existing rules that match new rules are reinforced (confidence increased, evidence count incremented) rather than duplicated.

**Step 4: Recall injection.** During recall, CalibrationRules are formatted as a "Model Calibration" section and injected early in the recall context, before conversation history or retrieved facts. This ensures the model sees its calibration adjustments before processing the user's request.

### 3.4 Implementation

PEDC is implemented as a single TypeScript module (`calibration.ts`, 280 lines) within the Engram memory plugin for OpenClaw. It uses the FallbackLlmClient for LLM access and stores CalibrationRules in a JSON index at `state/calibration/calibration-index.json`.

## 4. Experimental Setup

### 4.1 Data

We evaluate on production correction data from a deployed OpenClaw/Engram instance serving a single user over 41 days (February 5 – March 17, 2026). The corpus contains 1,002 corrections across diverse domains including software development, infrastructure management, project planning, and personal communication preferences.

### 4.2 Correction Type Analysis

We categorize corrections into four types:

| Type | Count | % | Calibration-addressable |
|------|-------|---|:-:|
| Factual/specific bugs | 748 | 74.7% | No |
| Tool usage patterns | 115 | 11.5% | Yes |
| Scope boundary issues | 81 | 8.1% | Yes |
| Behavioral/preference mismatches | 58 | 5.8% | Yes |
| **Total addressable** | **254** | **25.3%** | **Yes** |

74.7% of corrections describe one-off factual errors (wrong URL, wrong file path, wrong API response). These are outside the scope of any calibration system — they require domain-specific knowledge, not behavioral adjustment. The remaining 25.3% are systematic: the model repeatedly makes the same *type* of mistake due to a consistent misunderstanding of the user's expectations.

### 4.3 Temporal Holdout

We split at February 20, 2026:
- **Training set**: 866 corrections (before Feb 20)
- **Test set**: 136 corrections (Feb 20 onward)

This ensures no temporal leakage: calibration rules are generated exclusively from early corrections and evaluated on later ones.

### 4.4 Ablation Conditions

We compare three conditions:

1. **Calibration rules** (PEDC): 9 rules synthesized from the 866 training corrections via LLM clustering
2. **Raw corrections** (accumulation baseline): The 20 most recent training corrections shown verbatim
3. **Random corrections** (diversity baseline): 9 randomly selected training corrections

### 4.5 Evaluation Metric: Correction Preventability

For each test correction, an LLM judge evaluates: "Given this guidance [calibration rules / raw corrections / random corrections], would this specific correction have been preventable?"

The judge uses temperature 0.0 and is instructed to be strict — only marking corrections as preventable when there is a clear, direct match between the guidance and the error being corrected.

## 5. Results

### 5.1 Calibration Rules Produced

PEDC generated 9 calibration rules from 866 training corrections. Selected examples:

**Rule 1 (Scope boundary):**
- Condition: "When the user describes a task or scope boundaries are ambiguous"
- Model tendency: "Expands scope and starts implementing without asking"
- Calibration: "Only initiate actions explicitly requested. If in doubt, ask."

**Rule 2 (Communication boundary):**
- Condition: "When sending messages about work/finance topics"
- Model tendency: "May not account for time-sensitive boundaries"
- Calibration: "Do NOT message Joshua about work/finance on weekends."

**Rule 3 (Tool usage):**
- Condition: "When using the message tool"
- Model tendency: "Uses outdated parameter names"
- Calibration: "Always use 'target' parameter for channel ID. Not 'to' or 'channelId'."

These 9 rules compress 254 calibration-addressable corrections into actionable guidance — a 28:1 compression ratio.

### 5.2 Ablation Results

| Condition | Preventable | Rate | vs. Raw |
|-----------|-------------|------|---------|
| **Calibration rules** (9 rules) | 6/136 | **4.4%** | **6.3x** |
| Random corrections (9 random) | 4/136 | 2.9% | 4.1x |
| Raw corrections (20 most recent) | 1/136 | 0.7% | 1.0x |

PEDC-generated calibration rules prevent 6.3x more future corrections than raw correction accumulation. The ordering (calibration > random > raw) holds consistently. Random corrections outperform raw because diverse samples cover more error types than the 20 most recent, which tend to cluster around a single incident.

### 5.3 Qualitative Assessment

The 9 produced rules were reviewed by the user (first author). All 9 were judged to accurately capture real patterns of miscalibration. Each rule corresponds to a type of mistake the user had noticed recurring across sessions but had not previously been able to address through individual corrections.

## 6. Discussion

### 6.1 Why 4.4% Is the Right Number to Report

The 4.4% preventability rate applies to *all* test corrections, including the 75% that are factual bugs outside the scope of any calibration system. On the calibration-addressable subset (~34 of 136 test corrections), 6 were prevented — approximately 18%. We report the conservative 4.4% figure because the addressable/non-addressable categorization was done post-hoc and could introduce bias.

### 6.2 Accumulation vs. Diagnosis

The 6.3x improvement over raw correction accumulation validates the core thesis: analyzing correction *patterns* produces more transferable knowledge than accumulating individual corrections. A list of 1,002 "don't do X" rules cannot generalize to new contexts. Nine calibration rules describing *why* the model tends to be wrong can.

### 6.3 Limitations

**Single user.** All data comes from one user of one system. We cannot claim generalizability across users, domains, or agent architectures.

**LLM-as-judge circularity.** An LLM generates the calibration rules; a different LLM judges whether they would prevent corrections. Human evaluation would be stronger.

**No deployment evaluation.** We evaluate preventability retrospectively, not prospectively. Deploying calibration rules and measuring actual correction rate reduction over time is the critical next step.

**Rule conflicts.** The current system has no mechanism for resolving conflicts between calibration rules or retiring rules when user expectations change.

### 6.4 Comparison with Standard Benchmarks

We also evaluated on standard memory benchmarks (LongMemEval, MemoryArena, AMA-Bench). Calibration does not affect recall accuracy — these benchmarks test whether information can be retrieved, not whether the model understands the user. Benchmark results were flat (±0.4%), confirming that PEDC addresses an orthogonal dimension of agent capability.

## 7. Future Work

**Production deployment**: Deploy calibration rules to the production Engram instance and measure correction rate reduction over 2-4 weeks.

**Multi-user analysis**: With correction data from multiple users, separate model-level miscalibration (the model tends to expand scope with *everyone*) from user-specific miscalibration (the model expands scope only with *this* user). Model-level findings could be reported to model providers.

**Continuous calibration**: Run PEDC as a weekly consolidation job, producing updated rules as new corrections accumulate and retiring rules that no longer apply.

**Rule conflict resolution**: Develop mechanisms for detecting and resolving conflicts between calibration rules, and for deprioritizing rules when user behavior changes.

## 8. Conclusion

We introduced Prediction-Error-Driven Calibration (PEDC), a system that diagnoses systematic miscalibration between an AI agent and a specific user by analyzing patterns in user corrections. PEDC produces structured CalibrationRules that address root causes rather than individual symptoms, achieving 6.3x improvement over raw correction accumulation in a temporal holdout evaluation on 1,002 production corrections. The key distinction from prior work is the shift from correction accumulation to miscalibration diagnosis — from "remember what went wrong" to "understand why you keep being wrong."

## References

- Li, Y. et al. (2026). Language Models Need Sleep: Learning to Self Modify and Consolidate Memories. ICLR 2026.
- LightMem (2025). Lightweight and Efficient Memory-Augmented Generation. arXiv:2510.18866.
- Nemori (2025). arXiv:2508.03341.
- Packer, C. et al. (2023). MemGPT: Towards LLMs as Operating Systems. arXiv:2310.08560.
- PersonaLens (2025). arXiv:2506.09902.
- Popa, L.S. & Ebner, T.J. (2018). Cerebellum, Predictions and Errors. Frontiers in Cellular Neuroscience.
- SimpleMem (2026). arXiv:2601.02553.
- Sleep-Time Compute (2025). Beyond Inference Scaling at Test-time. arXiv:2504.13171.
- Wolpert, D.M. & Ghahramani, Z. (2000). Computational principles of movement neuroscience. Nature Neuroscience.
- MemoryOS (2025). EMNLP 2025.
- PersonalLLM (2025). ICLR 2025.
- Mem0 (2025). arXiv:2504.19413.
