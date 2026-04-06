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
| LLM | Gemini (API key) or OpenAI | Vertex AI, Azure OpenAI, Bedrock |
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
| `GOOGLE_API_KEY` | Gemini (default LLM) | https://aistudio.google.com/apikey |
| `OPENAI_API_KEY` | OpenAI (alternative LLM) | https://platform.openai.com/api-keys |
| `TAVILY_API_KEY` | Tavily search | https://tavily.com |
| `LANGSMITH_API_KEY` | LangSmith tracing | https://smith.langchain.com |

Set `MODEL_PROVIDER=gemini` (default) or `MODEL_PROVIDER=openai` in `.env` to choose the LLM.

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

## LangSmith Tracing

All traces are automatically sent to LangSmith when `LANGSMITH_TRACING=true` is set. Traces include the full orchestrator + subagent hierarchy — you can see each subagent's tool calls (SQL queries, web searches) nested under the parent trace.

View traces at: https://smith.langchain.com (under the `risk-assessment-agent` project)

## Dependencies

```
langchain>=1.0,<2.0
langchain-core>=1.0,<2.0
langchain-google-genai>=2.1.0
langchain-openai>=0.3.0
langchain-community>=0.3.0,<0.4.0
langchain-tavily
langgraph>=1.0,<2.0
langsmith>=0.3.0
sqlalchemy>=2.0.0
python-dotenv>=1.0.0
rich>=13.0.0
```

## File Structure

```
risk-assessment-agent/
├── agent.py              # Orchestrator agent with subagent tools, exports `agent` for Studio
├── tools.py              # Subagent definitions (regulatory, GRC analyst, risk assessor) + tool wrappers
├── chat.py               # Interactive terminal chat loop (rich UI)
├── seed_db.py            # Creates synthetic GRC SQLite database
├── risk_governance.db    # SQLite database (generated by seed_db.py)
├── langgraph.json        # LangGraph Studio / CLI configuration
├── .env.example          # API key template
├── .env                  # Your API keys (not committed)
├── pyproject.toml        # Project dependencies
└── README.md
```
