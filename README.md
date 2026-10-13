# Customer Support Agent: Multi-Agent ADK Demo

A working customer support agent built with the [Google Agent Development Kit (ADK)](https://adk.dev/) and deployable to Vertex AI Agent Runtime. A coordinator routes each question to a specialist, then rewrites the result into a clear, user-friendly answer.

This repository is designed to be read and understood. Routing, tool-level access control, memory, evaluation, and deployment are kept small enough to follow in one sitting.

## What It Does

```text
                    ┌─────────────────────────────────────────┐
User ──────────────►│ root_agent (coordinator)                │
                    │ gemini-3.1-pro-preview                  │
                    │                                         │
                    │ routes ──┬─► BillingSpecialist           │
                    │           │    └─ lookup_invoice         │
                    │           ├─► ShippingSpecialist         │
                    │           │    ├─ get_order_status       │
                    │           │    └─ deny_billing_tools     │
                    │           └─► RefundsSpecialist           │
                    │                                         │
                    │ Own tools: get_weather, get_current_time│
                    │ PreloadMemoryTool (past conversations)  │
                    └─────────────────────────────────────────┘
                                      │
                                      ▼
                         One synthesized, warm answer
```

For example, ask **“Where is my order ORD-8871?”** The coordinator calls `ShippingSpecialist`, which calls `get_order_status`. The coordinator then turns the raw facts into a friendly two-to-four sentence reply and explains the next step.

## Five Ideas Worth Learning

### 1. `AgentTool`, Not `sub_agents`

The specialists are attached as `AgentTool(agent=...)`, not as `sub_agents`.

With `sub_agents`, ADK transfers control to the specialist, which can reply to the user directly. The coordinator then has no opportunity to synthesize that answer. With `AgentTool`, the specialist's answer returns as a tool result, so the coordinator can rewrite it in one consistent voice.

See the `# ponytail:` comment on `root_agent` in `app/agent.py`.

### 2. Two Models, on Purpose

| Role | Model | Why |
| --- | --- | --- |
| Coordinator | `gemini-3.1-pro-preview` | Routing and synthesis need judgment. |
| Specialists | `gemini-3.7-flash` | Narrow tasks can use a faster, lower-cost model. |

The models are configured near the top of `app/agent.py` as `COORDINATOR_MODEL` and `WORKER_MODEL`.

### 3. Access Control in Code, Not Just in the Prompt

`ShippingSpecialist` must never access billing data. Two layers enforce this restriction:

```python
BILLING_TOOLS = [lookup_invoice]
BILLING_TOOL_NAMES = frozenset(t.__name__ for t in BILLING_TOOLS)

def deny_billing_tools(tool, args, tool_context) -> dict | None:
    if tool.name in BILLING_TOOL_NAMES:
        return {"error": f"Access denied: {tool.name} is a billing tool ..."}
    return None  # None means allow the call
```

- **Toolset:** The billing tool is not included in the shipping agent's tool list.
- **Callback:** `before_tool_callback=deny_billing_tools` blocks the call at runtime as an additional guard. Returning a dictionary short-circuits the tool call; returning `None` allows it.

Why use both? Prompt instructions can be overridden by prompt injection. A callback enforces the restriction in code. The tests in `tests/unit/test_tool_policy.py` cover this policy.

Add a new billing tool to `BILLING_TOOLS` and each guarded agent inherits the restriction without requiring changes to other files.

### 4. Memory Across Sessions

- `PreloadMemoryTool()` injects a `PAST_CONVERSATIONS` block into the request of the agent it is attached to. The coordinator has its own copy; memory does not flow down from another agent.
- `after_agent_callback=persist_turn_to_memory` writes the completed turn, including the user message, tool calls, and final answer, to memory.
- The backend is selected in `app/app_utils/services.py`: Vertex AI Memory Bank when `GOOGLE_CLOUD_AGENT_ENGINE_ID` is set, and in-memory storage otherwise. A local run can exercise the same code path without cloud setup.

### 5. One Agent, One Identity

The coordinator and all three specialists deploy as **one agent with one service identity**. The specialists are objects in the same process, not separate deployments.

| Layer | Mechanism | Controls |
| --- | --- | --- |
| Inside the process | `deny_billing_tools` | Which specialist can call which tool. |
| Cloud boundary | Agent Identity / IAM | What the entire agent can access in Google Cloud. |

## Project Structure

```text
customer-support-agent/
├── app/
│   ├── agent.py                    # Agents, tools, policy, and memory
│   ├── fast_api_app.py             # FastAPI server (ADK routes + A2A)
│   └── app_utils/
│       ├── services.py              # Session, artifact, and memory service wiring
│       ├── a2a.py                   # Agent2Agent protocol endpoints
│       └── reasoning_engine_adapter.py
├── tests/
│   ├── unit/
│   │   └── test_tool_policy.py      # Billing-access policy tests
│   ├── integration/                 # Agent stream and server end-to-end tests
│   └── eval/                        # Datasets and LLM-as-judge configuration
├── deployment/terraform/            # Infrastructure (single-project layout)
├── GEMINI.md                        # AI-assisted development guide
└── pyproject.toml
```

## Requirements

- **[uv](https://docs.astral.sh/uv/getting-started/installation/):** Python package and project manager. Add packages with `uv add <package>`; avoid plain `pip` for project dependencies.
- **Google Agents CLI:** Install with `uv tool install google-agents-cli`.
- **[Google Cloud SDK](https://cloud.google.com/sdk/docs/install):** Required for Google Cloud authentication and deployment.
- **Python 3.11–3.13**

## Quick Start

```bash
# 1. Set up the CLI and its skills
uvx google-agents-cli setup

# 2. Install project dependencies
agents-cli install

# 3. Configure your environment
cp .env.example .env
# Edit .env and set GOOGLE_CLOUD_PROJECT to your Google Cloud project ID.
gcloud auth application-default login

# 4. Start the local playground
agents-cli playground
```

Prefer Google AI Studio over Vertex AI? In `.env`, comment out the three `GOOGLE_*` lines and set `GEMINI_API_KEY=...` instead.

Try these prompts in the playground:

| Ask | Expected route |
| --- | --- |
| `Can you tell me the status of invoice INV-1042?` | `BillingSpecialist` |
| `Where is my order ORD-8871?` | `ShippingSpecialist` |
| `How long does a refund take?` | `RefundsSpecialist` |
| `What's the weather in SF?` | Coordinator answers directly |

The invoice and order tools return **fixture data** (see the `# ponytail:` comments). Replace them with your real billing and fulfilment API clients when integrating external systems.

## Commands

| Command | What it does |
| --- | --- |
| `agents-cli install` | Install dependencies with uv. |
| `agents-cli playground` | Start the local development server and web UI. |
| `agents-cli lint` | Run Ruff, ty, and codespell. |
| `uv run pytest tests/unit tests/integration` | Run unit and integration tests. |
| `agents-cli eval` | Generate, grade, and analyze evaluations. |
| `agents-cli deploy` | Deploy to Vertex AI Agent Runtime. |
| `agents-cli publish gemini-enterprise` | Register the deployed agent. |

ADK's own CLI is also available: `uv run adk ...`.

## Evaluation

The `tests/eval/` directory contains evaluation datasets and grading configuration.

- `datasets/customer-support-dataset.json`: one case for each routing path.
- `eval_config.yaml`: metrics to run.
- `response_quality.py`: a local LLM-as-judge that scores accuracy, relevance, and clarity from 1–5, returning a structured `{score, explanation}` verdict.
- `agent_turn_count`: an inline metric demonstrating how little code a custom metric can require.

Run the evaluation command:

```bash
agents-cli eval
# See `agents-cli eval --help` for available subcommands.
```

Results are written to `artifacts/grade_results/` as HTML and JSON; this directory is gitignored.

## Deployment

Set the Google Cloud project and deploy the agent:

```bash
gcloud config set project <your-project-id>
agents-cli deploy
```

To provision infrastructure with Terraform, first copy the variable template:

```bash
cp deployment/terraform/single-project/vars/env.tfvars.example \
   deployment/terraform/single-project/vars/env.tfvars

# Edit project_id, then run:
agents-cli infra cicd
```

The `env.tfvars` and `.env` files are gitignored so local configuration and credentials stay out of version control.

Telemetry is built in: traces go to Cloud Trace, logs to Cloud Logging, and completions to BigQuery.

## A2A (Agent2Agent)

The FastAPI app serves an [A2A protocol](https://a2a-protocol.org/) agent card and JSON-RPC endpoint alongside the ADK routes, allowing other agents to call this one. You can test it with the [A2A Inspector](https://github.com/a2aproject/a2a-inspector).

## Where to Change What

| You want to… | Edit |
| --- | --- |
| Add a specialist | `_worker(...)` and one `AgentTool` in `root_agent.tools`. |
| Add a tool | A plain function with a docstring; the docstring is the tool specification. |
| Change routing rules | `COORDINATOR_INSTRUCTION`. |
| Restrict a tool | Add it to `BILLING_TOOLS` or write a new `before_tool_callback`. |
| Swap models | `COORDINATOR_MODEL` / `WORKER_MODEL`. |
| Change the memory backend | `app/app_utils/services.py`. |

Scaffolded with `agents-cli` 1.4.2. Source headers use the Apache 2.0 license.
