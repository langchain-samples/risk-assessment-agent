# Risk Assessment & Governance Agent

A conversational AI agent that helps organizations assess risk and compliance requirements for AI initiatives. Describe what you want to build, and the agent walks you through a structured assessment covering the **EU AI Act**, **AICM** (AI Compliance Management), and **AIEU** (AI Ethics in the EU).

## What It Does

The agent acts as a GRC (Governance, Risk, Compliance) advisor. When you tell it about an AI system you're planning to build, it:

1. **Asks clarifying questions** — understands your system's purpose, data, users, deployment context, and timeline
2. **Dispatches subagents** to gather information:
   - **Regulatory Research Agent** — searches the internet for current AI Act, AICM, and AIEU requirements relevant to your system
   - **GRC Database Analyst Agent** — queries the organization's internal GRC database for existing controls, risks, audit findings, and compliance gaps
   - **Risk Assessor Agent** — synthesizes everything into a structured risk assessment
3. **Delivers a comprehensive assessment** including:
   - AI Act risk classification (Unacceptable / High / Limited / Minimal)
   - Specific regulatory obligations with article references
   - Control gap analysis against existing organizational controls
   - Prioritized recommendations (must-have / should-have / nice-to-have)

## Architecture

```
┌──────────────────────────────────────────────┐
│           Orchestrator Agent                 │
│   (create_agent + system prompt)             │
│                                              │
│   Subagents called as tools:                 │
│   ┌──────────────┐  ┌───────────────────┐    │
│   │  Regulatory   │  │  GRC Database     │    │
│   │  Research     │  │  Analyst          │    │
│   │  (Tavily)     │  │  (SQLite)         │    │
│   └──────────────┘  └───────────────────┘    │
│   ┌──────────────────────────────────────┐   │
│   │  Risk Assessor                       │   │
│   │  (synthesizes findings)              │   │
│   └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
         │                        │
    LangSmith               LangGraph Studio
    (tracing)               (langgraph.json)
```

| Component | Demo | Production Swap |
|-----------|------|-----------------|
| LLM | Gemini (API key), OpenAI, or Vertex AI | Azure OpenAI, Bedrock, etc. |
| Database | SQLite (`risk_governance.db`) | BigQuery, Snowflake, or any SQLAlchemy DB |
| Search | Tavily | Stardog knowledge graph, internal KB |
| Tracing | LangSmith | LangSmith |

## Prerequisites

- Python 3.11+
- [uv](https://docs.astral.sh/uv/) (`curl -LsSf https://astral.sh/uv/install.sh | sh`)
- [LangGraph CLI](https://langchain-ai.github.io/langgraph/cloud/reference/cli/) (`pipx install langgraph-cli`)
- API keys (see below)

## Setup

```bash
# Clone / navigate to project
cd risk-assessment-agent

# Install dependencies
uv sync

# Configure environment variables
cp .env.example .env
# Edit .env with your API keys
```

### Required API Keys

| Key | Service | Get it at |
|-----|---------|-----------|
| `TAVILY_API_KEY` | Tavily search | https://tavily.com |
| `LANGSMITH_API_KEY` | LangSmith tracing | https://smith.langchain.com |

Plus one of the LLM provider keys below.

### Choosing an LLM Provider

Set `MODEL_PROVIDER` in `.env` to one of: `gemini` (default), `openai`, or `vertex`.

#### Option A: Gemini (API key)

The simplest option — uses Google's Gemini API directly.

```bash
# .env
GOOGLE_API_KEY=your_key_here
MODEL_PROVIDER=gemini
```

Get a key at https://aistudio.google.com/apikey

#### Option B: OpenAI

```bash
# .env
OPENAI_API_KEY=your_key_here
MODEL_PROVIDER=openai
```

Get a key at https://platform.openai.com/api-keys

#### Option C: Vertex AI

Uses Google Cloud's Vertex AI with a service account — the enterprise option for production deployments.

1. **Create a GCP service account** with the `Vertex AI User` role
2. **Download the service account JSON key** and save it as `vertexCred.json` in the project root
3. **Configure `.env`**:

```bash
# .env
GOOGLE_APPLICATION_CREDENTIALS=./vertexCred.json
MODEL_PROVIDER=vertex
```

> `vertexCred.json` is in `.gitignore` and will not be committed. Never check credentials into source control.

### Seed the Database

```bash
uv run python seed_db.py
```

This creates `risk_governance.db` with synthetic GRC data: 10 regulatory frameworks (including AI Act, AICM, AIEU), 12 policies, 35 controls, 22 risks, 28 mitigations, 18 audit findings, and 48 compliance mappings.

## Running

### Option 1: LangGraph Studio (recommended for demos)

```bash
langgraph dev
```

This starts the LangGraph development server and opens LangGraph Studio in your browser. The agent appears as **"Risk Assessment Agent"** in the Studio UI.

- API: http://127.0.0.1:2024
- Studio UI: opens automatically (or visit https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024)
- API Docs: http://127.0.0.1:2024/docs

> **Note**: If `langgraph dev` can't find project dependencies, install them into the LangGraph CLI environment:
> ```bash
> pipx runpip langgraph-cli install -e .
> ```

### Option 2: Interactive Chat (terminal)

```bash
uv run python chat.py
```

Rich-formatted terminal chat with conversation memory. Type `quit` or `exit` to end.

### Option 3: LangGraph API (programmatic)

```bash
# Start the server
langgraph dev --no-browser

# Create a thread
curl -s http://127.0.0.1:2024/threads \
  -H "Content-Type: application/json" -d '{}'

# Send a message (replace THREAD_ID)
curl -s "http://127.0.0.1:2024/threads/THREAD_ID/runs/wait" \
  -H "Content-Type: application/json" \
  -d '{
    "assistant_id": "Risk Assessment Agent",
    "input": {
      "messages": [{"role": "user", "content": "I want to build an AI credit scoring system for EU lending decisions."}]
    }
  }'
```

## Sample Conversation

```
You: I want to build an AI-powered customer credit scoring system that will be
     used to make lending decisions for customers in the EU.

Agent: That's a significant initiative! To give you the most accurate assessment,
       could you tell me more about it?
       - What specific types of data will the system use?
       - Will it make fully automated decisions or assist loan officers?
       - What is your target deployment timeline?

You: It uses historical loan data, income, and credit bureau data. It will assist
     loan officers but also auto-reject below a threshold. Deploy in 6 months.

Agent: [Dispatches regulatory research, GRC database analysis, and risk assessment subagents]

       ## AI Credit Scoring System: Risk Assessment

       ### AI Act Risk Classification: HIGH-RISK
       Classified under EU AI Act Annex III, point 5(b) — AI systems used to
       evaluate creditworthiness of natural persons...

       ### Key Regulatory Obligations
       - Art. 9: Risk Management System (lifecycle)
       - Art. 10: Data Governance (bias detection)
       - Art. 14: Human Oversight (override capability)
       - Art. 43: Conformity Assessment (before deployment)
       ...

       ### Critical Gaps
       - No conformity assessment process exists
       - No AI system registration process for EU database
       - Human override not available for auto-rejection decisions
       ...
```

## Pre-populating LangSmith Traces

To generate realistic trace data for demos or evaluation, run the simulation script:

```bash
uv run python populate_traces.py
```

This uses [openevals multi-turn simulation](https://docs.langchain.com/langsmith/multi-turn-simulation) to run 8 scenarios with simulated user personas (VP of Engineering, HR Director, CISO, etc.) that interact naturally with the agent over multiple turns. Each run generates slightly different conversations since the simulated users respond dynamically to the agent's questions.

| Scenario | Persona | Turns | What it covers |
|----------|---------|-------|----------------|
| AI Credit Scoring System | VP of Engineering | 4 | High-risk financial AI |
| AI Hiring Screening Tool | HR Director | 4 | High-risk employment AI |
| Customer Service Chatbot | Product Manager | 4 | Limited-risk AI |
| Fraud Detection System | Chief Risk Officer | 4 | High-risk with human-in-the-loop |
| AI Medical Triage Assistant | Healthtech CTO | 5 | High-risk healthcare AI |
| Compliance Status Review | Compliance Officer | 3 | GRC database queries |
| AI Content Moderation | Trust & Safety Director | 4 | High-risk content moderation |
| Open Risk Register Review | CISO | 3 | Risk register queries |

> **Note**: The simulated users require an OpenAI API key (`OPENAI_API_KEY` in `.env`) since they use `gpt-4o-mini`. The agent itself can use any configured provider.

## Offline Evaluations

The project includes an evaluation suite that tests the agent across 4 models using a synthetic dataset uploaded to LangSmith.

### 1. Create the Dataset

```bash
uv run python eval_dataset.py
```

This creates a LangSmith dataset called **"Risk Assessment Agent Evaluations"** with 8 synthetic examples covering:

| Example | Expected Risk Level | Key Tools |
|---------|-------------------|-----------|
| AI Credit Scoring System | High | All 3 subagents |
| AI Hiring Screening Tool | High | All 3 subagents |
| Customer Service Chatbot | Limited | All 3 subagents |
| AI Fraud Detection System | High | All 3 subagents |
| AI Medical Triage Assistant | High | All 3 subagents |
| AI Content Moderation System | High | All 3 subagents |
| Compliance Status Review | N/A | grc_database_analysis |
| AI Predictive Maintenance | Minimal | All 3 subagents |

Each example includes expected outputs: risk classification, expected tool calls, regulatory frameworks to cite, and assessment sections to include.

### 2. Run Evaluations

```bash
uv run python offline_evals.py
```

This runs 4 experiments — one per model — against the dataset:

| Model | Provider |
|-------|----------|
| `gpt-4.1-mini` | OpenAI |
| `gpt-4.1` | OpenAI |
| `claude-sonnet-4-20250514` | Anthropic |
| `gemini-2.5-flash` | Google |

Each experiment runs all 8 dataset examples through the agent and scores them with 4 evaluators:

| Evaluator | Type | What It Measures |
|-----------|------|-----------------|
| `subagent_delegation_quality` | LLM Judge (trajectory) | Did the agent call the right subagents for the query? |
| `risk_classification_accuracy` | LLM Judge (single step) | Is the EU AI Act risk classification correct? |
| `regulatory_framework_coverage` | Custom Code | Are expected frameworks (AI Act, AICM, AIEU) cited? |
| `assessment_structure_completeness` | Custom Code | Does the response include all required sections? |

Results are visible in the LangSmith **Experiments** tab with model, prompt, and tool metadata populated for comparison.

> **Note**: Running all 4 experiments requires API keys for OpenAI (`OPENAI_API_KEY`), Anthropic (`ANTHROPIC_API_KEY`), and Google (`GOOGLE_API_KEY`). The LLM judge evaluators also use OpenAI (`gpt-4.1`). Comment out entries in `MODEL_CONFIGS` in `offline_evals.py` to skip models you don't have keys for.

## LangSmith Tracing

All traces are automatically sent to LangSmith when `LANGSMITH_TRACING=true` is set. Traces include the full orchestrator + subagent hierarchy — you can see each subagent's tool calls (SQL queries, web searches) nested under the parent trace.

View traces at: https://smith.langchain.com (under the `risk-assessment-agent` project)

## Dependencies

```
langchain>=1.0,<2.0
langchain-core>=1.0,<2.0
langchain-anthropic>=0.3.0
langchain-google-genai>=2.1.0
langchain-google-vertexai>=3.2.0
langchain-openai>=0.3.0
langchain-community>=0.3.0,<0.4.0
langchain-tavily
langgraph>=1.0,<2.0
langsmith>=0.3.0
sqlalchemy>=2.0.0
python-dotenv>=1.0.0
rich>=13.0.0
openevals>=0.1.0
```

## File Structure

```
risk-assessment-agent/
├── agent.py              # Orchestrator agent with subagent tools, exports `agent` for Studio
├── tools.py              # Subagent definitions (regulatory, GRC analyst, risk assessor) + tool wrappers
├── chat.py               # Interactive terminal chat loop (rich UI)
├── seed_db.py            # Creates synthetic GRC SQLite database
├── populate_traces.py    # Multi-turn simulation script to pre-populate LangSmith traces
├── eval_dataset.py       # Synthetic evaluation dataset — creates & uploads to LangSmith
├── offline_evals.py      # Offline evaluations — 4 evaluators × 4 models
├── risk_governance.db    # SQLite database (generated by seed_db.py)
├── langgraph.json        # LangGraph Studio / CLI configuration
├── .env.example          # API key template
├── .env                  # Your API keys (not committed)
├── pyproject.toml        # Project dependencies
└── README.md
```
