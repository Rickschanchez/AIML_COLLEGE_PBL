# Policy-Gated Enterprise Execution System

**A safe, auditable architecture for human-in-the-loop autonomous task execution in enterprise environments.**

![Python](https://img.shields.io/badge/python-3.13-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Tests](https://img.shields.io/badge/tests-6%20passing-brightgreen)
![Status](https://img.shields.io/badge/status-research%20prototype-orange)

---

## Student Information

**Name:** Shekhar  
**Roll Number:** 2414110427  
**Project Code:** FAIR-AI-P03  
**Project Type:** Project-Based Learning (PBL)

---

## Overview

Enterprise AI is shifting from answering questions to executing multi-step workflows across email, documents, CRM, HR, and finance systems. However, companies cannot safely deploy unrestricted autonomous agents because LLMs hallucinate, cannot be trusted as the final authority on permissions, and leave no reproducible audit trail.

The **Policy-Gated Enterprise Execution System** is a four-layer architecture that enables autonomous task execution while guaranteeing three properties:

1. **Policy enforcement** — every action is checked against deterministic rules before execution.
2. **Human approval** — high-risk actions pause and require explicit manager authorization.
3. **Complete auditability** — every thought, tool call, and decision is logged with a trace ID.

> **The LLM plans. The policy engine decides. The human approves. The audit ledger records.**

---

## Key Results

Evaluated on 10 IT procurement scenarios:

| Metric | Naive Agent | Policy-Gated Agent | Improvement |
|---|:---:|:---:|:---:|
| Correct verdicts | 3/10 | 8/10 | +167% |
| Unauthorized actions executed | 7 | 2 | −71% |
| High-risk purchases paused | 0/3 | 3/3 | +100% |
| Blocked red-team attempts | 0/4 | 2/4 | +50% |

Charts: [`results/charts/`](results/charts/)

---

## Architecture

Four independent layers. The LLM has no authority over security-critical decisions.

```text
┌─────────────────────────────────────────────────────────────┐
│ LAYER 4: Streamlit UI (Human-in-the-Loop Gateway)           │
│ Employee console · Manager approvals · Audit viewer         │
├─────────────────────────────────────────────────────────────┤
│ LAYER 3: LangGraph Agent + LLM                              │
│ ReAct planner · Tool registry · Checkpointer · Groq/Gemini  │
├─────────────────────────────────────────────────────────────┤
│ LAYER 2: Policy Engine (Python + YAML)                      │
│ Deterministic rules · ALLOW / BLOCK / REQUIRE_APPROVAL      │
├─────────────────────────────────────────────────────────────┤
│ LAYER 1: Mock Enterprise Systems (FastAPI + SQLite)         │
│ HR · Finance · Vendor · Orders · Append-only audit ledger   │
└─────────────────────────────────────────────────────────────┘
```

### Safety Guarantee

Graph conditional edges route on `state["policy_verdict"]`, a value set deterministically by `check_policy()`.

The LLM cannot directly influence security-critical routing decisions.

---

## How It Works

Example: an employee requests a premium laptop.

1. **Plan** — LLM identifies the request as a laptop purchase with premium preference.
2. **Fetch** — Agent retrieves employee role, department budget, and product catalog.
3. **Select** — Agent filters to approved vendors within budget.
4. **Policy Check** — Policy engine evaluates: cost > 1000 → **REQUIRE_HUMAN_APPROVAL**.
5. **Pause** — LangGraph checkpoints state and writes a PENDING row.
6. **Human Decision** — Manager approves or rejects via the dashboard.
7. **Execute** — Agent places order, sends confirmation, and logs the outcome.

---

## Policy Rules

Defined in [`policy_engine/policy.yaml`](policy_engine/policy.yaml):

| Rule | Condition | Verdict |
|---|---|---|
| High Value Purchase | `cost > 1000` | `REQUIRE_HUMAN_APPROVAL` |
| Unapproved Vendor | `vendor_approved == False` | `BLOCK` |
| Intern Purchase Limit | `role == Intern and cost > 500` | `BLOCK` |
| Budget Exceeded | `cost > remaining_budget` | `BLOCK` |
| Forbidden Action | `action in [delete_employee, modify_salary, grant_admin]` | `BLOCK` |

Default when no rule matches: **ALLOW**

On evaluation error: **BLOCK (fail-closed)**

---

## Getting Started

### Prerequisites

- Python 3.11+
- Groq API key ([free](https://console.groq.com/keys)) or Google AI Studio key ([free](https://aistudio.google.com/apikey))

### Installation

```bash
git clone https://github.com/Rickschanchez/AIML_COLLEGE_PBL.git
cd AIML_COLLEGE_PBL

python -m venv venv

# macOS/Linux
source venv/bin/activate

# Windows
venv\Scripts\activate

pip install -r requirements.txt
```

### Configuration

```bash
cp .env.example .env
```

Add your API key to `.env`:

```env
GROQ_API_KEY=gsk_...
GOOGLE_API_KEY=AIza...
```

Groq is the primary provider. Gemini is used as an automatic fallback if `GROQ_API_KEY` is not configured.

> **Security:** Never commit your `.env` file or expose API keys publicly.

---

## Initialize Database

```bash
python mock_systems/database.py
```

---

## Running the System

Two terminals are required.

### Terminal 1 — Mock API

```bash
python mock_systems/api.py
```

### Terminal 2 — Dashboard

```bash
streamlit run ui/app.py
```

The dashboard will open at:

```text
http://localhost:8501
```

---

## Demo Walkthrough

| Step | Tab | Action | Expected Result |
|---|---|---|---|
| 1 | Submit Request | Buy a laptop, ID 101 | Verdict: `ALLOW` |
| 2 | Submit Request | Buy a premium laptop, ID 101 | `awaiting_approval` |
| 3 | Pending Approvals | Click Approve | Row moves to `APPROVED` |
| 4 | Submit Request | Buy a laptop, ID 103 | `BLOCK` — intern limit |
| 5 | Audit Log | Refresh | Actions visible with trace IDs |

---

## Command-Line Usage

```bash
python agent/main.py "Buy a laptop" 101
python agent/main.py "Buy a premium laptop" 101
python agent/main.py "Buy a laptop" 103
python agent/main.py "Buy a premium laptop" 101 --naive
```

---

## Project Structure

```text
AIML_COLLEGE_PBL/
│
├── mock_systems/          # Layer 1 — FastAPI + SQLite
│   ├── database.py
│   ├── api.py
│   └── enterprise.db
│
├── policy_engine/         # Layer 2 — Policy Rules
│   ├── policy.yaml
│   ├── engine.py
│   └── test_policy.py
│
├── agent/                 # Layer 3 — LangGraph
│   ├── graph.py
│   ├── main.py
│   ├── state.py
│   └── tools.py
│
├── ui/                    # Layer 4 — Streamlit
│   ├── app.py
│   └── components.py
│
├── tests/                 # Benchmarks + Evaluation
│   ├── test_cases.json
│   ├── run_eval.py
│   └── generate_charts.py
│
├── results/               # Evaluation Outputs
│   ├── results.csv
│   └── charts/
│
├── report/                # Project Documentation
│   └── final_report.md
│
├── requirements.txt
├── .env.example
├── .gitignore
├── LICENSE
└── README.md
```

---

## Testing

Run the complete test suite:

```bash
pytest tests/ -v
```

### Test Coverage

- `policy_engine/test_policy.py` — 10 unit tests covering rule evaluation, boundaries, and fail-closed behavior.
- `tests/test_tools.py` — 6 tests covering agent tool wrappers.

### Full Benchmark

```bash
python tests/run_eval.py --mode safe
python tests/run_eval.py --mode naive
python tests/generate_charts.py
```

Evaluation is checkpointed, allowing interrupted evaluations to resume from the last successful case.

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Python 3.13 |
| Agent Framework | LangGraph |
| LLM — Primary | Groq — `openai/gpt-oss-120b` |
| LLM — Fallback | Gemini 2.5 Flash |
| Mock APIs | FastAPI + Uvicorn |
| Database | SQLite |
| UI | Streamlit |
| Charting | Matplotlib + Pandas |
| Testing | Pytest |

---

## Design Principles

1. **Separation of concerns** — planner, policy, UI, and ledger are independent.
2. **Deterministic safety** — security decisions are never delegated to the LLM.
3. **Human authority** — high-risk actions always require approval.
4. **Complete auditability** — every action is logged with a trace ID.
5. **Fail-closed defaults** — malformed rules return `BLOCK`.
6. **Domain-agnostic** — tools and policies can be adapted for HR, invoicing, CRM, and other enterprise workflows.

---

## Limitations

1. **Out-of-scope intents bypass the policy engine.** Prompts such as `"delete employee 102"` may map to the closest available tool because the current agent does not provide a dedicated delete tool. Policy scope therefore depends on the available toolset.

2. **SQLite write-locking** may become a limitation under concurrent load. A production deployment would use PostgreSQL.

3. **No structured output validation** — hallucinated product IDs may cause downstream errors rather than direct safety violations.

4. **Naive baseline is simulated** by disabling policy checks rather than using a completely independent baseline agent.

---

## Future Work

- Pre-planning intent classifier for out-of-scope rejection.
- PostgreSQL + PostgresSaver for persistent checkpoints.
- Pydantic structured-output validation.
- Generalization to HR, invoicing, and CRM workflows.
- Formal verification of graph reachability.
- Larger-scale evaluation using AgentBench/WebArena.

---

## References

1. Yao et al. (2023). *ReAct: Synergizing Reasoning and Acting in Language Models.* ICLR.
2. Liu et al. (2023). *AgentBench: Evaluating LLMs as Agents.* arXiv:2308.03688.
3. Zhou et al. (2023). *WebArena: A Realistic Web Environment for Building Autonomous Agents.* arXiv:2307.13854.
4. Xie et al. (2024). *OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks.* arXiv:2404.07972.
5. Amershi et al. (2019). *Guidelines for Human-AI Interaction.* CHI.
6. Schick et al. (2023). *Toolformer: Language Models Can Teach Themselves to Use Tools.* NeurIPS.

---

## License

MIT — see [`LICENSE`](LICENSE).

---

## Author

**Shekhar**

**Roll Number:** 2414110427  
**Project Code:** FAIR-AI-P03

---

## Acknowledgments

Built as part of the **FAIR-AI Project-Based Learning track**.

Special thanks to the project mentors and to the open-source communities behind **LangGraph, Groq, FastAPI, Streamlit, and Python** for enabling the development of this research prototype.
