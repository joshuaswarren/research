# Research

Papers on agentic memory, model-user calibration, and AI systems.

Working in public — these are living documents that get updated as results come in.

## Papers

### Prediction-Error-Driven Calibration (PEDC)
**Status**: Published (March 2026) — deployment evaluation in progress

[Read the paper](pedc-calibration/paper.md)

A system that diagnoses systematic miscalibration between an AI agent and a specific user by analyzing patterns in user corrections. Instead of storing corrections as individual facts, PEDC clusters them to identify root causes and produces structured CalibrationRules. Evaluated on 1,002 production corrections from OpenClaw/Engram.

**Key result**: Calibration rules prevent 6.3x more future corrections than raw correction accumulation.

**Implementation**: [openclaw-engram](https://github.com/joshuaswarren/openclaw-engram) — `src/calibration.ts`

---

*Joshua Warren — [joshuawarren.com](https://joshuawarren.com)*
