# ShopAssist — LLM Cost Optimization

## Overview
ShopAssist is a Tier-1 customer support copilot for e-commerce ticket resolution. For incoming tickets, the copilot reads the customer thread, loads order metadata, chooses a resolution action (`REFUND`, `REPLACE`, `SEND_ARTICLE`, or `ESCALATE_TIER2`), and cites the governing knowledge-base (KB) policy article.

This project optimizes token consumption and billing points across both single-call and multi-stage pipeline architectures while strictly preserving ticket-resolution correctness and required output schemas.

| Track | Code | Spec |
| :--- | :--- | :--- |
| Q1 — single call | [question1/agent.py](question1/agent.py) | [questions/question-1.md](questions/question-1.md) |
| Q2 — pipeline | [question2/agent.py](question2/agent.py) | [questions/question-2.md](questions/question-2.md) |

---

## Setup and Execution

```bash
pip install -r requirements.txt
python question1/run.py
python question2/run.py
pytest -q
```

> **Note on Test Execution:** Running the full test suite via `pytest -q` will encounter failures on the two baseline cost assertions (`test_baseline_cost_is_high` in Q1 and Q2). These assertions explicitly expect the original, high baseline cost floor and therefore conflict with the optimized cost targets. All ticket-resolution correctness tests continue to pass.

### Billing Model (`shared/billing.py`)
- **Uncached Input:** 3.0 points / token
- **Cached Input:** 0.3 points / token (identical prefix ≥ 1,024 tokens)
- **Output:** 15.0 points / token
- **Formula:** `points = int(uncached_input * 3 + cached_input * 0.3 + output * 15)`

---

## Question 1 — Single-Agent Optimization
Question 1 (`question1/agent.py`) uses a single LLM call per ticket to classify and resolve support requests.

### Key Changes
- **Removed Verbose Reasoning Instructions:** Eliminated trigger phrases such as `"Think step by step"` and `"Write full reasoning"` that caused the LLM simulator to append $\ge 480$ tokens of synthetic padding notes.
- **Preserved Existing Ticket-Handling Logic:** Kept prompt structure, context assembly, and decision rules intact.
- **Preserved Markers and Output Schema:** Maintained mandatory `ORDER_SNAPSHOT:` / `END_SNAPSHOT` delimiters and the required JSON output schema (`action`, `kb_article`, `customer_reply`).

### Results
- **Baseline Cost:** 35,568 points across four public tickets (~8,892 points / ticket)
- **Optimized Cost:** 8,367 points across four public tickets (~2,092 points / ticket)
- **Target Constraint:** $\le$ 3,000 points / ticket
- **Cost Reduction:** Approximately **76.5%**

---

## Question 2 — Multi-Stage Pipeline Optimization
Question 2 (`question2/agent.py`) implements a 4-stage customer-support pipeline: **triage $\rightarrow$ retrieve $\rightarrow$ compose $\rightarrow$ audit**.

### Key Changes
- **Deterministic Triage & Retrieval:** Replaced redundant, expensive LLM-based triage and retrieval stages with lightweight deterministic operations. Retrieval uses deterministic KB title-index access through `title_index()`, rather than semantic or relevance-based retrieval.
- **Removed Prompt Bloat & Synthetic Filler:** Eliminated 120,000 characters of synthetic help-center filler in triage, 80,000 characters in audit, and duplicate triage interpolation in the compose prompt.
- **Consolidated Generative LLM Execution:** Retained a single clean LLM call in `_compose` to produce the final resolution.
- **Structural Audit:** Replaced the redundant second LLM pass in `_audit` with direct programmatic validation via `parse_resolution`, validating JSON structure and required keys rather than independent policy compliance.

### Results
- **Baseline Cost:** 293,331 points across four public tickets (~73,333 points / ticket)
- **Optimized Cost:** 4,782 points across four public tickets (~1,195.5 points / ticket)
- **Target Constraint:** $\le$ 2,500 points / ticket
- **Cost Reduction:** Approximately **98.37%**

---

## Validation
- **Question 1 Public Tickets:** `test_baseline_resolves_public_tickets` passes (correct `action` and `kb_article` across `T-1001` to `T-1004`).
- **Question 1 Holdout Test:** `test_holdout_subscription` passes on holdout ticket `T-1091`.
- **Question 2 Public Tickets:** `test_pipeline_resolves_public_tickets` passes across all public tickets (`T-1001` to `T-1004`). Public-ticket correctness was verified, but Q2 holdout behavior has not yet been independently verified.
- **Baseline Cost Assertions:** The two baseline cost checks (`test_baseline_cost_is_high` in Q1 asserting `points > 15,000` and in Q2 asserting `points > 70,000`) conflict with the optimized cost targets ($\le 3,000$ and $\le 2,500$ points/ticket) and therefore fail after optimization. These assertions were designed to confirm that the initial unoptimized system was expensive, whereas meeting the target cost constraints requires driving points well below those baseline thresholds.

---

## Engineering Trade-offs
- **Deterministic Title Indexing vs. Semantic Retrieval:** Replacing LLM-based retrieval with deterministic title-index access via `title_index()` significantly reduces token consumption and latency. However, accessing a static title list is not equivalent to true semantic or relevance-based retrieval over a dynamic knowledge catalog. In a production deployment with hundreds of evolving articles, an indexed semantic retrieval system (such as vector search or hybrid BM25 + dense embeddings) should be used.
- **Structural JSON Validation vs. Independent Policy Audit:** The audit stage currently validates JSON structure and key presence rather than independent policy compliance. While this prevents malformed outputs at zero token cost, it does not evaluate policy adherence. A production-grade architecture could introduce deterministic policy-rule validation or reserved LLM auditing for high-risk or ambiguous customer scenarios.
