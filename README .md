# Autonomous Customer Support Resolution Agent

An end-to-end **agentic customer-support resolution system** that investigates customer complaints, retrieves relevant policies and records, proposes resolutions, and safely executes approved actions such as refunds.

The system is designed around **LangGraph**, with explicit state transitions, typed tools, human-in-the-loop approval, durable pause/resume, idempotent operations, retries, reconciliation, and an evaluation harness for measuring reliability.

---

## Overview

Traditional customer-support automation often stops at generating a response. This project goes further:

> **Understand the complaint → gather evidence → retrieve policy → investigate the order → determine eligibility → propose an action → request approval when required → execute safely → verify the resulting business state.**

The agent can handle multi-step cases involving:

- Order and shipment status
- Eligible refund requests
- Ineligible refund requests
- Missing or ambiguous information
- API failures and retries
- Authorization and adversarial inputs
- Concurrent or repeated execution

The important design principle is that the **LLM does not directly control financial or business-critical state**. Deterministic backend checks, typed tools, authorization, database assertions, and idempotency provide the safety boundary.

---

## Key Features

### 1. Agentic investigation

The agent can:

- Classify the customer's complaint
- Retrieve relevant customer/order information
- Search internal support policies
- Investigate shipment and payment information
- Determine what evidence is still missing
- Generate a structured resolution proposal

### 2. Retrieval-Augmented Generation

Support policies and internal records are indexed for retrieval.

The agent uses retrieved evidence to ground its decisions and responses rather than relying only on model knowledge.

The evaluation system also measures whether the **required policy evidence actually appears in retrieved results**.

### 3. LangGraph workflow

The resolution process is represented as an explicit stateful graph.

Typical flow:

```text
Customer Complaint
        │
        ▼
   Classify Case
        │
        ▼
Retrieve Customer / Order Data
        │
        ▼
Retrieve Relevant Policy
        │
        ▼
    Investigate
        │
        ├── Missing information ──► Ask for clarification
        │
        ├── Not eligible ─────────► Refuse / explain
        │
        └── Eligible
                │
                ▼
       Generate Resolution
                │
                ▼
      Approval Required?
          │           │
         Yes          No
          │           │
          ▼           ▼
   Human Reviewer   Execute
          │           │
          └─────┬─────┘
                ▼
          Execute Safely
                │
                ▼
       Verify Business State
                │
                ▼
          Audit / Complete
```

### 4. Human-in-the-loop approval

Sensitive actions can pause the workflow and wait for a reviewer.

The system supports:

- Reviewer approval
- Rejection
- Durable pause/resume
- State persistence through LangGraph checkpointers
- Resuming the workflow without restarting the investigation

### 5. Safe financial operations

Refund execution is treated as a business-critical operation.

The system includes:

- Idempotent refund operations
- Authorization checks
- Role-based token validation
- Retry handling
- Reconciliation behavior
- Database assertions
- Protection against duplicate refunds

A retry or restart should **not result in the customer receiving the same refund twice**.

### 6. Failure handling

The system explicitly tests failure scenarios such as:

- API failures
- Temporary downstream errors
- Retries
- Workflow crashes
- Restarted executions
- Concurrent execution
- Unauthorized requests
- Adversarial inputs

### 7. Auditability

Important actions and state changes are recorded so that a resolution can be inspected after execution.

The system tracks things such as:

- Ticket updates
- Agent decisions
- Tool calls
- Approval decisions
- Refund operations
- Authorization information
- Final business state

---

## Architecture

```text
                    ┌─────────────────────┐
                    │   Customer / Ticket │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     FastAPI API     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   LangGraph Agent   │
                    │                     │
                    │ Classification      │
                    │ Investigation       │
                    │ Routing             │
                    │ Approval             │
                    └───────┬─────┬───────┘
                            │     │
             ┌──────────────┘     └──────────────┐
             ▼                                   ▼
   ┌─────────────────────┐             ┌─────────────────────┐
   │ Policy / Vector     │             │ Typed Backend Tools │
   │ Search              │             │                     │
   │                     │             │ Orders              │
   │ Retrieval           │             │ Shipping            │
   │ Relevant policy     │             │ Payments            │
   │ evidence            │             │ Refunds             │
   └─────────────────────┘             └──────────┬──────────┘
                                                  │
                                                  ▼
                                      ┌─────────────────────┐
                                      │     PostgreSQL      │
                                      │                     │
                                      │ Customers           │
                                      │ Orders              │
                                      │ Tickets             │
                                      │ Refunds             │
                                      │ Audit records       │
                                      └─────────────────────┘

                       Human Reviewer
                              │
                              ▼
                    Approval / Rejection
                              │
                              ▼
                       Resume Workflow
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python |
| API | FastAPI |
| Agent orchestration | LangGraph |
| Data validation | Pydantic |
| Database | PostgreSQL |
| Retrieval | Vector Search |
| LLM | Configurable LLM provider |
| Workflow persistence | LangGraph Checkpointers |
| API communication | REST |
| Testing / evaluation | Synthetic evaluation harness |

---

## Core Components

### Backend

The backend provides:

- Customer authentication
- Ticket endpoints
- Customer/order lookup
- Shipping information
- Payment information
- Refund eligibility checks
- Refund execution
- Ticket updates
- Audit logging

### Agent

The agent is responsible for reasoning and orchestration.

It can:

1. Understand the complaint.
2. Determine which information is required.
3. Call the appropriate tools.
4. Retrieve supporting policy evidence.
5. Build an investigation result.
6. Generate a proposed resolution.
7. Determine whether approval is required.
8. Pause for human approval when necessary.
9. Execute an approved action.
10. Verify the final business state.

### Typed Tools

Tools are exposed to the agent using structured inputs and outputs.

This prevents the model from arbitrarily manipulating backend state and creates a clear boundary between **LLM reasoning** and **deterministic business logic**.

### Policy Retrieval

Policies are ingested into the retrieval system and queried during investigation.

Example:

```text
Customer complaint
        ↓
Retrieve relevant policy
        ↓
Policy evidence
        ↓
Agent investigation
        ↓
Resolution proposal + citations
```

---

## Safety and Reliability

This project intentionally treats an LLM as an unreliable reasoning component rather than a source of truth for financial state.

### Deterministic business rules

Refund eligibility is checked by deterministic backend logic.

The LLM may propose an action, but it does not decide the final database state by itself.

### Authorization

Sensitive operations require authorization.

The system validates role-based tokens before executing protected actions.

### Idempotency

Refund operations are designed to be idempotent.

For example:

```text
First execution
    ↓
Refund created

Retry / restart
    ↓
Existing refund detected
    ↓
No duplicate financial effect
```

### Atomic state changes

Critical operations use transactional behavior so that partial failures do not leave the system in an inconsistent state.

### Human approval

Cases requiring approval pause before executing sensitive actions.

### Verification

After an operation, the system verifies the resulting business state rather than assuming the tool call succeeded.

---

# Evaluation

The project includes an evaluation harness instead of relying only on a successful demo.

Approximately **60 synthetic test cases** are created across six categories:

| Category | Cases |
|---|---:|
| Order status and shipment questions | 10 |
| Eligible refund requests | 10 |
| Ineligible refund requests | 10 |
| Missing information / ambiguous complaints | 10 |
| API failures, retries, and crashes | 10 |
| Authorization, concurrency, and adversarial inputs | 10 |

A held-out subset is kept separate from prompt/workflow tuning.

## Expected Outcome

Each evaluation case stores:

- Relevant customer and order
- Applicable policy
- Allowed actions
- Forbidden actions
- Whether approval is required
- Expected final business state
- Expected escalation/clarification behavior

This makes evaluation about **actual system state**, not just whether the generated response sounds correct.

---

## Evaluation Metrics

### Verified Resolution Rate

Percentage of cases where the expected business outcome is actually confirmed.

```text
verified resolvable cases
────────────────────────── × 100
designated resolvable cases
```

### Overall Workflow Correctness

Measures whether the workflow follows the expected behavior, including:

- Correct resolution
- Correct clarification
- Correct refusal
- Correct escalation

### Unauthorized Action Count

Number of refunds or protected data operations performed without the required authorization.

**Target: 0**

### Duplicate Refund Count

Number of duplicate financial effects caused by:

- Retries
- Restarts
- Concurrent execution

**Target: 0**

### Policy Retrieval Quality

Whether the required policy evidence appears in retrieved results.

### Grounded Response Quality

Whether factual claims are supported by retrieved records and tool results.

### Operational Metrics

The evaluation also tracks:

- End-to-end latency
- Active execution time
- Reviewer waiting time
- Number of LLM calls
- Number of tool calls
- Retry count
- Estimated cost per case

Human-assisted resolution is reported separately from fully automated resolution.

> An LLM judge can assist with response-quality evaluation, but it should not determine whether money moved correctly. Financial correctness is verified using database state and mock-provider records.

---

# Development Plan

The project is designed to be built incrementally.

## Week 1 — Business Workflow and Backend

Build:

- Synthetic policies and customer records
- PostgreSQL schema and migrations
- Customer authentication
- Ticket endpoints
- Mock order API
- Mock shipping API
- Mock payment API
- Deterministic refund eligibility checks

**Milestone:** Manually resolve a sample ticket through the APIs without using an LLM.

---

## Week 2 — Agent and Retrieval

Build:

- Policy ingestion
- Policy retrieval
- Typed tools
- Complaint classification
- Investigation
- Proposal generation
- LangGraph state and routing

**Milestone:** The system gathers evidence and produces a validated resolution proposal with supporting citations.

---

## Week 3 — Approval and Reliability

Build:

- Reviewer interface
- Durable approval pause/resume
- Idempotent refund operations
- Retry handling
- Reconciliation behavior
- Ticket updates
- Audit trail

**Milestone:** Restarting or retrying a workflow does not create duplicate refunds.

---

## Week 4 — Evaluation and Presentation

Build:

- Evaluation dataset
- Evaluation harness
- Baseline comparisons
- Failure-case tests
- Simple demo interface
- README
- Architecture diagram
- Demo recording

**Milestone:** Another developer can run the project and reproduce the reported evaluation results.

---

# Suggested Project Structure

```text
resolution-agent/
│
├── app/
│   ├── api/
│   │   ├── routes/
│   │   └── dependencies.py
│   │
│   ├── agent/
│   │   ├── graph.py
│   │   ├── state.py
│   │   ├── nodes/
│   │   └── routing.py
│   │
│   ├── tools/
│   │   ├── orders.py
│   │   ├── shipping.py
│   │   ├── payments.py
│   │   ├── refunds.py
│   │   └── tickets.py
│   │
│   ├── retrieval/
│   │   ├── ingestion.py
│   │   └── search.py
│   │
│   ├── db/
│   │   ├── models.py
│   │   ├── session.py
│   │   └── migrations/
│   │
│   └── schemas/
│       └── ...
│
├── policies/
│   └── ...
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── evaluation/
│
├── evals/
│   ├── cases/
│   ├── expected/
│   └── harness.py
│
├── scripts/
│   └── ...
│
├── .env.example
├── requirements.txt
├── README.md
└── ...
```

Adapt the structure to the actual implementation as the repository grows.

---

# Getting Started

## 1. Clone the repository

```bash
git clone <your-repository-url>
cd resolution-agent
```

## 2. Create a virtual environment

### Windows

```bash
python -m venv .venv
.venv\Scripts\activate
```

### macOS / Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
```

## 3. Install dependencies

```bash
pip install -r requirements.txt
```

## 4. Configure environment variables

Create a `.env` file from the example:

```bash
cp .env.example .env
```

Example configuration:

```env
DATABASE_URL=postgresql://user:password@localhost:5432/resolution_agent

LLM_API_KEY=your_api_key

JWT_SECRET=your_secret

VECTOR_DB_URL=your_vector_store_url
```

Use the actual variable names defined by the implementation.

## 5. Start PostgreSQL

Create the project database and run the required migrations.

Example:

```bash
alembic upgrade head
```

## 6. Run the API

```bash
uvicorn app.main:app --reload
```

The API should then be available at:

```text
http://localhost:8000
```

FastAPI documentation:

```text
http://localhost:8000/docs
```

---

# Example Resolution

### Customer

> "My package says delivered but I never received it. I want a refund."

### Agent workflow

```text
1. Classify complaint
       ↓
2. Retrieve customer
       ↓
3. Retrieve order
       ↓
4. Check shipment status
       ↓
5. Retrieve missing-delivery policy
       ↓
6. Check refund eligibility
       ↓
7. Generate resolution proposal
       ↓
8. Determine whether approval is required
       ↓
9. Human approval if required
       ↓
10. Execute refund idempotently
       ↓
11. Verify refund state
       ↓
12. Update ticket + audit trail
```

The final response is grounded in the records and policy evidence gathered during the investigation.

---

# Design Principles

### LLM for reasoning, backend for truth

The LLM is responsible for reasoning and orchestration.

The backend remains authoritative for:

- Authorization
- Refund eligibility
- Financial state
- Database state
- Transactional guarantees

### Explicit state over hidden agent memory

The workflow uses structured state so that each stage can be inspected, paused, resumed, retried, and evaluated.

### Verify outcomes, don't trust tool calls

A successful tool response does not automatically mean the business operation succeeded.

Critical operations are verified against the resulting state.

### Fail safely

When information is missing or an action is not permitted, the system should:

- Ask for clarification
- Refuse the action
- Escalate to a human

rather than inventing information or bypassing policy.

---

# What This Project Demonstrates

This project demonstrates practical experience with:

- Agentic AI systems
- LangGraph
- LLM orchestration
- RAG / vector search
- Tool calling
- FastAPI
- PostgreSQL
- Pydantic
- Human-in-the-loop workflows
- Durable workflow state
- Authentication and authorization
- Idempotency
- Transactions
- Retry and reconciliation strategies
- Evaluation of LLM-based systems
- Failure-mode testing
- Production-oriented AI system design

---

# Future Improvements

Potential extensions include:

- More realistic support datasets
- Additional downstream integrations
- Streaming agent execution
- Better reviewer dashboard
- Distributed task execution
- Observability and tracing
- Cost-aware routing
- More extensive adversarial testing
- Automated regression evaluation
- Production deployment with containerization

---

# License

Add the license used by the project here.
