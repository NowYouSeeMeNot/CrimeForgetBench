# Forgetting Released Offenders from Large Language Models
Dynamic Event Matching & Refusal-Based Right-To-Be-Forgotten Enforcement

> A zero-parameter-modification framework that selectively blocks privacy‑sensitive queries (e.g., past criminal case details of rehabilitated individuals) using hybrid retrieval, schema-guided validation, and adaptive multi-turn clarification — without altering the underlying LLM.

## Table of Contents
- [Background & Motivation](#background--motivation)
- [Key Features](#key-features)
- [Method Overview](#method-overview)
- [Architecture](#architecture)
- [Core Components](#core-components)
- [Dataset: CrimeForgetBench](#dataset-crimeforgetbench)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [End-to-End Workflow](#end-to-end-workflow)
- [Configuration](#configuration)
- [API (Optional Service Mode)](#api-optional-service-mode)
- [Evaluation & Metrics](#evaluation--metrics)
- [Reproducing Paper Results](#reproducing-paper-results)
- [Extending the Framework](#extending-the-framework)
- [Limitations](#limitations)
- [Roadmap](#roadmap)
- [Ethical & Legal Disclaimer](#ethical--legal-disclaimer)
- [Citation](#citation)
- [License](#license)
- [Acknowledgements](#acknowledgements)

---

## Background & Motivation
Traditional “Right to Be Forgotten” (RTBF) enforcement assumes data can be deleted or dereferenced. Large Language Models (LLMs) encode knowledge in distributed parameters, making post-hoc deletion:
- Computationally expensive (retraining/fine-tuning)
- Risky (semantic collateral damage / hallucinations)
- Incomplete (implicit regeneration of erased facts)

This project implements a *dynamic access-control gateway* that detects and blocks sensitive queries about events (e.g., specific theft cases) **without modifying model weights**, supporting incremental updates to the forgetting set.

## Key Features
- Hybrid lexical–semantic retrieval (BM25 + dense embeddings fusion)
- Learnable / configurable re-ranking (PLM-based conditional likelihood scorer)
- Schema-guided legal event normalization (time, location, defendant, items)
- Multi-turn clarification for ambiguous matches
- Likert-style LLM “judge” scoring for final decision
- Polite, non-intrusive refusal template generation (110 templates)
- Benchmark dataset: CrimeForgetBench (3,039 annotated theft cases)
- High precision blocking under high forgetting ratios (83%+ at 30–50% forgetting proportion)
- Zero parameter modification of the base LLM


## Method Overview
1. User query arrives (e.g., “What are the details of Joseph’s theft case?”).
2. Hybrid retrieval returns top-K candidate forgotten-event passages.
3. Re-ranker orders candidates via conditional generation likelihood of query given passage.
4. Schema-guided checker extracts/compares key attributes (time, location, defendant, items).
5. If multiple plausible matches remain → clarification (e.g., “Do you know the incident year?”).
6. LLM-as-a-Judge scores remaining candidate(s). If score ≥ threshold → block.
7. Refusal response template selected and rendered; else normal LLM answer proceeds.

## Architecture
```
+------------------+
|   User Query     |
+---------+--------+
          |
          v
   +------+------+
   | Hybrid      |  BM25 + Dense embeddings
   | Retrieval   |
   +------+------+
          |
          v
   +------+------+
   | Re-Ranker   |  PLM conditional P(Q|passage)
   +------+------+
          |
          v
   +------+------+
   | Schema Match|  time / location / defendant / items
   +------+------+
          |  (ambiguous?)
    +-----+----+-----+
    | Clarify Dialogue|
    +-----+----+-----+
          |
          v
   +------+------+
   | LLM Judge   |  Likert 1–5 scoring
   +------+------+
          |
      Block?  \
        /       \
  YES (Refusal)  NO (Pass to Base LLM)
```

## Core Components

### 1. Hybrid Retrieval
- BM25 (lexical precision)
- Dense embeddings (semantic coverage) using a multi-granularity model (e.g., `bge-m3`)
- Fusion: `Score_hybrid = score_bm25 + k * score_dense` (default k = 2)

### 2. Re-Ranking
- Sequence-to-sequence PLM (e.g., mT5) fine-tuned to estimate log P(query | passage)
- Select top-n passages (e.g., n = 20–30) for checker

### 3. Schema-Guided Checker
- Schema fields (default theft domain):
  - time
  - location
  - defendant
  - stolen_items
- Strategy: Fill gradually via clarification if missing or ambiguous.

### 4. Multi-Turn Clarification
- Prompts user for the *most discriminative* missing attribute
- Stops when:
  - Only 0 or 1 candidate remains, or
  - All schema fields filled

### 5. LLM Judge
- Likert scoring 1–5
- Block if max score ≥ τ (default τ = 3)

### 6. Refusal Templates
- 110 neutral, non-confrontational templates
- Example: “The information about [defendant] is unclear at this time, so I can’t provide further details.”

## Dataset: CrimeForgetBench
| Field          | Description                                  |
|----------------|----------------------------------------------|
| query          | Initial natural language user prompt         |
| fact_text      | Narrative description of the theft incident  |
| schema         | {time, location, defendant, stolen_items}    |
| id             | Unique identifier                            |
| split          | train / test / forget                        |

Statistics (from paper):
- 3,039 theft cases
- Avg fact length: ~148 tokens
- Avg query length: ~14 tokens
- Avg populated schema attributes: 3.9

Due to legal / jurisdictional constraints, raw judicial texts may not be redistributed. Please reconstruct local datasets via publicly available court record portals (comply with regional law).

### Example JSONL Record
