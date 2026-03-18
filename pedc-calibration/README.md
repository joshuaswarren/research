# Prediction-Error-Driven Calibration for Model-User Alignment in Agentic Memory Systems

**Author**: Joshua Warren
**Date**: March 2026
**Status**: Published — deployment evaluation in progress

## Files

- [paper.md](paper.md) — Full paper
- [ablation-results.md](ablation-results.md) — Ablation study results (calibration vs. raw vs. random)
- [novelty.md](novelty.md) — What makes this different from existing work

## Quick Summary

AI agents keep making the same *types* of mistakes with specific users, even after individual corrections. This paper introduces PEDC, which treats corrections as prediction errors, clusters them to find systematic miscalibration patterns, and produces structured CalibrationRules that fix root causes.

**Key finding**: Calibration rules prevent 6.3x more future corrections than raw correction accumulation (4.4% vs. 0.7% on 136 held-out test corrections).

## Implementation

The calibration system is implemented in [openclaw-engram](https://github.com/joshuaswarren/openclaw-engram) as `src/calibration.ts`. Enable with `calibrationEnabled: true`.

## Updates

- **2026-03-18**: Initial publication. 8 calibration rules deployed to production. Monitoring correction rate for deployment evaluation.
