# 🤖 Multi-Agent Research & Report Generation

An end-to-end **autonomous research report generation system** powered by a multi-agent AI architecture built on **LangGraph**. Submit a topic, and a team of AI agents collaborates — searching the web, conducting interviews, analyzing sources, and producing a publication-ready report in DOCX/PDF format.

![Python](https://img.shields.io/badge/Python-3.11+-blue?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.120-009688?logo=fastapi&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-0.6.8-orange)
![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker&logoColor=white)
![Azure](https://img.shields.io/badge/Azure%20Container%20Apps-Deployed-0078D4?logo=microsoftazure&logoColor=white)
![License](https://img.shields.io/badge/License-Proprietary-red)

---

## 📌 Table of Contents

- [Overview](#-overview)
- [System Architecture](#-system-architecture)
- [Agent Details](#-agent-details)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Getting Started](#-getting-started)
- [Configuration](#%EF%B8%8F-configuration)
- [API Endpoints](#-api-endpoints)
- [Deployment](#-deployment)
- [Use Cases](#-use-cases)
- [Future Enhancements](#-future-enhancements)

---

## 📌 Overview

This project transforms a high-level research query into a structured, well-written report by orchestrating multiple specialized AI agents. The agents work together through a **LangGraph StateGraph** — dynamically creating analyst personas, conducting multi-turn interviews with web-sourced context, and compiling the results into professional documents.

### Key Capabilities

- **End-to-end automation:** `Topic → Persona Creation → Research → Analysis → Report`
- **Multi-agent collaboration** with parallel interview execution
- **Human-in-the-loop** feedback before research begins
- **Real-time web search** via Tavily API
- **Multi-LLM support:** OpenAI, Google Gemini, Groq (DeepSeek)
- **Dual output formats:** DOCX and PDF
- **Production-ready:** Dockerized with Jenkins CI/CD → Azure Container Apps

---

## 🧠 System Architecture

The system follows an **agentic workflow** orchestrated by LangGraph, where each agent has a well-defined responsibility:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        FastAPI Web Layer                                │
│   ┌──────────┐    ┌──────────────┐      ┌──────────────────────────┐    │
│   │  Web UI  │───▶│ Report Routes │───▶│    ReportService         │    │
│   └──────────┘    └──────────────┘      └────────────┬─────────────┘    │
└────────────────────────────────────────────────────┼─────────────────── ┘
                                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   LangGraph Orchestrator (StateGraph)                   │
│                                                                         │
│   ┌──────────────────┐          ┌──────────────────────┐                │
│   │ 1. Create Analyst│────────▶|  2. Human Feedback   │                │
│   │    Agent         │          │    (Interrupt Point) │                │
│   └──────────────────┘          └─────────┬────────────┘                │
│                                    ▼ (approve)   ▲ (refine)             │
│                              ┌─────────────────────────┐                │
│                              │  Fan-Out: N Interviews  │                │
│                              │  (one per analyst)      │                |
|                              ───────────────────────────                |
|                                          ▼                               |
│   ┌──────────────────────────────────────────────────────────────┐      │
│   │              Interview Sub-Graph (per analyst)               │      │
│   │                                                              │      │
│   │  ❓ Ask Question ──▶ 🌐 Search Web ──▶ 🧠 Generate Answer │      │
│   │                                              │               │      │
│   │                      💾 Save Interview ◀─────┘              │      │
│   │                              │                               │      │
│   │                      📝 Write Section                        │     │
│   └───────────────────────────────┬───────────────────────────────┘     │
│                                   │ (fan-in: all sections)              │
│                    ┌──────────────┼──────────────┐                      │
│                    ▼              ▼              ▼                      │
│            ┌────────────┐ ┌────────────┐ ┌────────────┐                 │
│            │ Write Report│ │ Write Intro │Write Concl.│                 │
│            └─────┬──────┘ └─────┬──────┘ └─────┬──────┘                 │
│                  └──────────────┼──────────────┘                        │
│                                 ▼                                       │
│                       ┌──────────────────┐                              │
│                       │ Finalize Report  │                              │
│                       └────────┬─────────┘                              │
└────────────────────────────────┼────────────────────────────────────────┘
                                 ▼
                    ┌────────────────────────┐
                    │  📁 DOCX    📁 PDF     │
                    │  generated_report/     │
                    └────────────────────────┘
```

### Data Flow Summary

| Phase | State Keys | Description |
|-------|-----------|-------------|
| **Init** | `topic`, `max_analysts` | User submits a research topic |
| **Persona Creation** | `analysts: List[Analyst]` | LLM generates N analyst personas with distinct focuses |
| **Human Review** | `human_analyst_feedback` | Graph pauses — user can refine or approve analysts |
| **Parallel Interviews** | `messages`, `context`, `interview`, `sections` | Each analyst runs a full interview sub-graph concurrently |
| **Report Assembly** | `content`, `introduction`, `conclusion` | Three parallel nodes process all sections simultaneously |
| **Finalization** | `final_report` | Merges all parts, saves as DOCX and PDF |

---

## 🤖 Agent Details

### 1. Analyst Creator Agent
- **Node:** `create_analyst`
- **File:** `research_and_analyst/workflows/report_generator_workflow.py`
- **Role:** Dynamically generates AI analyst personas based on the research topic. Each analyst has a unique name, role, affiliation, and research focus. Uses LLM structured output (`Perspectives` model).

### 2. Interview Sub-Graph (Compound Agent)
- **File:** `research_and_analyst/workflows/interview_workflow.py`
- Runs **in parallel** for each analyst (fan-out via `Send()`). Contains 5 sequential nodes:

| Node | Method | Role |
|------|--------|------|
| `ask_question` | `_generate_question()` | **Analyst persona** asks probing, topic-specific questions |
| `search_web` | `_search_web()` | **Search Agent** — converts question to Tavily web search, retrieves documents |
| `generate_answer` | `_generate_answer()` | **Analysis Agent** — expert answers using retrieved context with source citations |
| `save_interview` | `_save_interview()` | Persists the full interview transcript |
| `write_section` | `_write_section()` | **Synthesis Agent** — converts interview insights into a structured report section |

### 3. Report Writer Agent
- **Node:** `write_report`
- **Role:** Compiles all analyst sections into a unified narrative with structured insights.

### 4. Introduction & Conclusion Writers
- **Nodes:** `write_introduction`, `write_conclusion`
- **Role:** Generate contextual opening and closing sections using all gathered research.

### 5. Report Finalizer
- **Node:** `finalize_report`
- **Role:** Assembles introduction + content + conclusion + sources into the final document.

### 6. Human Feedback (Validation Agent)
- **Node:** `human_feedback` (interrupt point)
- **Role:** Human-in-the-loop validation — review generated analysts and provide steering feedback before interviews begin.

### 7. Orchestrator
- **Implementation:** LangGraph `StateGraph` with `MemorySaver` checkpointing
- **Role:** Manages agent sequencing, conditional edges, fan-out/fan-in parallelism, and state persistence.

---

## 🔧 Tech Stack

| Category | Technology |
|----------|-----------|
| **Orchestration** | LangGraph (StateGraph, conditional edges, fan-out/fan-in) |
| **LLM Providers** | OpenAI (`gpt-4o-mini`), Google (`gemini-2.0-flash`), Groq (`deepseek-r1-distill-llama-70b`) |
| **Web Search** | Tavily Search API |
| **Web Framework** | FastAPI + Jinja2 Templates |
| **Authentication** | SQLAlchemy + SQLite + bcrypt (passlib) |
| **Prompt Templating** | Jinja2 |
| **Document Generation** | python-docx (DOCX), ReportLab (PDF) |
| **Structured Logging** | structlog (JSON) |
| **Containerization** | Docker (multi-stage build) |
| **CI/CD** | Jenkins Pipeline |
| **Cloud Deployment** | Azure Container Apps + Azure Container Registry |
| **Language** | Python 3.11+ |

---

## 📂 Project Structure

```
multi-agent-report-generation/
├── main.py                          # Entry point (placeholder)
├── pyproject.toml                   # Project metadata & dependencies
├── requirements.txt                 # Pinned pip dependencies
├── Dockerfile                       # Multi-stage production Docker image
├── Dockerfile.jenkins               # Jenkins agent Docker image
├── Jenkinsfile                      # CI/CD pipeline definition
├── build-and-push-docker-image.sh   # ACR image build script
├── setup-app-infrastructure.sh      # Azure infrastructure provisioning
├── azure-deploy-jenkins.sh          # Jenkins deployment helper
│
├── research_and_analyst/            # Core application package
│   ├── __init__.py
│   ├── api/                         # FastAPI web layer
│   │   ├── main.py                  #   App factory, middleware, routes
│   │   ├── routes/
│   │   │   └── report_routes.py     #   Auth + report generation endpoints
│   │   ├── services/
│   │   │   └── report_service.py    #   Business logic (graph execution)
│   │   ├── models/                  #   API request/response models
│   │   └── templates/               #   Jinja2 HTML templates
│   │
│   ├── workflows/                   # LangGraph agent workflows
│   │   ├── report_generator_workflow.py  # Main orchestration graph
│   │   └── interview_workflow.py         # Interview sub-graph
│   │
│   ├── schemas/
│   │   └── models.py                # Pydantic models & state definitions
│   │
│   ├── prompt_lib/
│   │   └── prompt_locator.py        # All Jinja2 prompt templates
│   │
│   ├── config/
│   │   └── configuration.yaml       # LLM & embedding model config
│   │
│   ├── utils/
│   │   ├── config_loader.py         # YAML config reader
│   │   └── model_loader.py          # LLM & embedding model factory
│   │
│   ├── database/
│   │   └── db_config.py             # SQLAlchemy + SQLite setup
│   │
│   ├── logger/
│   │   └── custom_logger.py         # Structured JSON logging (structlog)
│   │
│   └── exception/
│       └── custom_exception.py      # Custom exception hierarchy
│
├── static/                          # Frontend assets
│   ├── css/styles.css
│   └── js/app.js
│
├── generated_report/                # Output directory (auto-created)
│   └── <Topic>_<Timestamp>/         # Per-report subfolder
│       ├── report.docx
│       └── report.pdf
│
└── logs/                            # JSON structured log files
```

---

## 🚀 Getting Started

### Prerequisites

- Python 3.11+
- API keys for at least one LLM provider + Tavily

### 1. Clone the Repository

```bash
git clone https://github.com/BharathKA13/multi-agent-report-generation.git
cd multi-agent-report-generation
```

### 2. Create Virtual Environment

```bash
python -m venv venv
# Windows
venv\Scripts\activate
# Linux/macOS
source venv/bin/activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Set Environment Variables

Create a `.env` file in the project root:

```env
# Choose one: openai | google | groq
LLM_PROVIDER=google

# API Keys (provide keys for your chosen provider)
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=AIza...
GROQ_API_KEY=gsk_...

# Required for web search
TAVILY_API_KEY=tvly-...
```

### 5. Run the Application

```bash
# Web UI (FastAPI)
uvicorn research_and_analyst.api.main:app --host 0.0.0.0 --port 8000 --reload

# CLI mode (standalone)
python research_and_analyst/workflows/report_generator_workflow.py
```

### 6. Access the UI

Open [http://localhost:8000](http://localhost:8000) in your browser. Sign up, log in, and submit a research topic from the dashboard.

---

## ⚙️ Configuration

### LLM Provider Selection

Set `LLM_PROVIDER` env var to switch between providers. The models are configured in `research_and_analyst/config/configuration.yaml`:

```yaml
llm:
  groq:
    provider: "groq"
    model_name: "deepseek-r1-distill-llama-70b"
    temperature: 0
    max_output_tokens: 2048

  google:
    provider: "google"
    model_name: "gemini-2.0-flash"
    temperature: 0
    max_output_tokens: 2048

  openai:
    provider: "openai"
    model_name: "gpt-4o-mini"
    temperature: 0
```

---

## 🌐 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Health check for container orchestration |
| `GET` | `/` | Login page |
| `POST` | `/login` | Authenticate user |
| `GET` | `/signup` | Registration page |
| `POST` | `/signup` | Create new user |
| `GET` | `/dashboard` | Research report dashboard (auth required) |
| `POST` | `/generate_report` | Start report generation pipeline |
| `POST` | `/submit_feedback` | Submit human feedback on analysts |
| `GET` | `/download/{filename}` | Download generated DOCX/PDF |

---

## 🐳 Deployment

### Docker

```bash
# Build the image
docker build -t research-report-app .

# Run the container
docker run -d -p 8000:8000 \
  -e LLM_PROVIDER=google \
  -e GOOGLE_API_KEY=your-key \
  -e TAVILY_API_KEY=your-key \
  research-report-app
```

### Azure Container Apps (via Jenkins)

The project includes a full CI/CD pipeline:

1. **`build-and-push-docker-image.sh`** — Builds and pushes to Azure Container Registry
2. **`setup-app-infrastructure.sh`** — Provisions Azure resource group, ACR, and Container App environment
3. **`Jenkinsfile`** — Automated pipeline: checkout → install → test → deploy → verify

```bash
# Provision infrastructure
./setup-app-infrastructure.sh

# Build and push Docker image
./build-and-push-docker-image.sh

# Jenkins handles the rest automatically on push to main
```

---

## 📈 Use Cases

- **Automated literature reviews** — Generate comprehensive overviews of any research domain
- **Technical & market research** — Rapid analysis of emerging technologies or market trends
- **Policy & domain analysis** — Multi-perspective reports on complex topics
- **Internal knowledge synthesis** — Consolidate distributed knowledge into structured documents
- **Decision-support documentation** — Evidence-based reports with cited sources

---

## ✅ Key Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Modularity** | Each agent is an independent graph node, easily replaceable or extensible |
| **Parallelism** | Fan-out interviews run concurrently; report assembly runs in parallel |
| **Human-in-the-loop** | Graph interrupt allows human steering before expensive research begins |
| **Traceability** | Structured JSON logging (structlog) + interview transcripts preserved in state |
| **Multi-provider** | Switch LLM providers via a single env var — no code changes |
| **Production-ready** | Docker multi-stage build, health checks, Jenkins CI/CD, Azure deployment |

---

## 🚀 Future Enhancements

- [ ] Multi-turn feedback loops (refine report iteratively)
- [ ] Citation tracking with automatic bibliography generation
- [ ] Vector database integration for long-term memory across reports
- [ ] Streaming UI with real-time agent status updates
- [ ] Support for document upload (PDF/URL) as additional research sources
- [ ] Role-based access control and report sharing

---

