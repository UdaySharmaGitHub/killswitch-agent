# AI Agent Kill Switch — Complete Project Blueprint

> **A production-grade safety mechanism for autonomous AI Agents that enables real-time control, emergency shutdown, granular capability management, and full observability.**

---

## Table of Contents

1. [Why This Project Matters](#1-why-this-project-matters)
2. [What is a Kill Switch in AI Agents](#2-what-is-a-kill-switch-in-ai-agents)
3. [System Architecture](#3-system-architecture)
4. [Tech Stack — Detailed Justification](#4-tech-stack--detailed-justification)
5. [Project Structure](#5-project-structure)
6. [Core Components — Full Implementation](#6-core-components--full-implementation)
   - 6.1 [Agent Core (LangGraph ReAct Agent)](#61-agent-core-langgraph-react-agent)
   - 6.2 [Agent Tools](#62-agent-tools)
   - 6.3 [Kill Switch Guard](#63-kill-switch-guard)
   - 6.4 [Kill Switch State Manager (Redis)](#64-kill-switch-state-manager-redis)
   - 6.5 [Audit Logger](#65-audit-logger)
   - 6.6 [Watchdog Service](#66-watchdog-service)
   - 6.7 [Control Plane API (FastAPI)](#67-control-plane-api-fastapi)
   - 6.8 [Frontend Dashboard (React + Tailwind)](#68-frontend-dashboard-react--tailwind)
   - 6.9 [Prometheus Metrics Exporter](#69-prometheus-metrics-exporter)
   - 6.10 [Grafana + Loki + Prometheus Configuration](#610-grafana--loki--prometheus-configuration)
   - 6.11 [LLM Tracing with Langfuse](#611-llm-tracing-with-langfuse)
   - 6.12 [Docker Compose (Full Stack)](#612-docker-compose-full-stack)
7. [Workflow Diagrams](#7-workflow-diagrams)
8. [Environment Variables](#8-environment-variables)
9. [How to Run the Project](#9-how-to-run-the-project)
10. [Testing Strategy](#10-testing-strategy)
11. [Sprint Plan — 4 Week Timeline](#11-sprint-plan--4-week-timeline)

---

## 1. Why This Project Matters

### The Problem

AI Agents are becoming increasingly autonomous. They can browse the web, write code, execute shell commands, make API calls, modify databases, and interact with external services — all without human intervention. This autonomy creates a critical problem: **what happens when an agent goes wrong?**

Real-world incidents that demonstrate the urgency:

- **Runaway agents** — An autonomous agent enters an infinite loop, calling paid APIs thousands of times and running up a $10,000 bill in minutes.
- **Hallucination-driven actions** — An agent hallucinates a SQL query and drops a production table.
- **Prompt injection** — A malicious input tricks the agent into exfiltrating sensitive data via API calls.
- **Cascading failures** — An agent spawns sub-agents, each spawning more, overwhelming your infrastructure.
- **Unintended side effects** — An agent "helpfully" deletes old files that turn out to be critical backups.

### Why Kill Switches Are Critical Now

1. **Regulatory Compliance** — The EU AI Act (2024) and NIST AI Risk Management Framework both require "human override" mechanisms for autonomous AI systems. A kill switch is the baseline implementation.

2. **Enterprise Adoption** — Companies deploying AI agents in production (customer support, DevOps automation, data pipelines) need the ability to instantly halt rogue agents without taking down their entire system.

3. **Agentic AI is Scaling** — Frameworks like LangGraph, AutoGen, and CrewAI are making it trivial to build multi-step autonomous agents. The tooling for controlling them has not kept pace.

4. **Trust and Safety** — Users will not trust an AI agent they cannot stop. A visible, reliable kill switch is a fundamental trust signal.

### What You Will Learn

By building this project, you will understand:

- How autonomous AI agents execute tasks (ReAct loop, tool calling)
- How to intercept and control every action an agent takes
- How to build distributed safety mechanisms using Redis
- How to implement three-layer kill switches (hard, soft, capability)
- How to build real-time observability into AI systems
- How to design operator dashboards for AI agent management
- How to containerise and deploy a full AI safety stack

---

## 2. What is a Kill Switch in AI Agents

A kill switch is a **safety mechanism** that allows a human operator to immediately halt, degrade, or roll back the actions of an autonomous AI agent. It operates at multiple levels:

### Kill Switch Levels

```
┌─────────────────────────────────────────────────────────────────┐
│  LEVEL 1 — HARD KILL (Emergency Stop)                          │
│  ► Immediately terminates ALL agent processes                   │
│  ► Revokes all active sessions                                  │
│  ► Kills running containers / pods                              │
│  ► Used when: agent is actively causing damage                  │
├─────────────────────────────────────────────────────────────────┤
│  LEVEL 2 — SOFT KILL (Graceful Shutdown)                       │
│  ► Stops accepting new tasks                                    │
│  ► Drains in-flight tasks to completion                         │
│  ► Checkpoints agent state for resumption                       │
│  ► Used when: scheduled maintenance or non-urgent stop          │
├─────────────────────────────────────────────────────────────────┤
│  LEVEL 3 — CAPABILITY KILL (Feature Flags)                     │
│  ► Disables specific tools (e.g. block DB writes)               │
│  ► Agent continues running with reduced permissions             │
│  ► Granular control over what the agent can and cannot do       │
│  ► Used when: isolating a specific risky behaviour              │
└─────────────────────────────────────────────────────────────────┘
```

### Kill Switch vs Guardrails

| Aspect | Kill Switch | Guardrails |
|--------|-------------|------------|
| Trigger | Human operator (manual) | Automated rules (programmatic) |
| Timing | Reactive — after a problem is detected | Preventive — before the action happens |
| Scope | Global or per-capability | Per-action or per-rule |
| Purpose | Emergency stop | Input/output validation |
| Example | "Stop the agent NOW" | "Block SQL DELETE statements" |

**Both are necessary.** This project focuses on the kill switch, but we include guardrails in the tool execution layer as a complementary defence.

---

## 3. System Architecture

### High-Level Architecture

```
                         ┌─────────────────────────┐
                         │    OPERATOR / USER       │
                         │  (React Dashboard)       │
                         └───────────┬─────────────┘
                                     │ HTTP
                                     ▼
                         ┌─────────────────────────┐
                         │    CONTROL PLANE API     │
                         │      (FastAPI)           │
                         │                          │
                         │  POST /kill              │
                         │  POST /revive            │
                         │  POST /disable/{tool}    │
                         │  GET  /status            │
                         │  GET  /audit-log         │
                         │  GET  /metrics           │
                         │  WebSocket /ws/events    │
                         └───────────┬─────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                 │
                    ▼                ▼                 ▼
          ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
          │    REDIS     │  │  POSTGRESQL  │  │   WATCHDOG   │
          │              │  │              │  │   SERVICE    │
          │ Kill flags   │  │ Audit log    │  │              │
          │ Capabilities │  │ Task history │  │ Heartbeat    │
          │ Heartbeat    │  │ Agent state  │  │ monitor      │
          │ Pub/Sub      │  │              │  │ Auto-kill    │
          └──────┬──────┘  └──────────────┘  └──────────────┘
                 │
                 │  polled before EVERY tool call
                 ▼
          ┌─────────────────────────────────────────────────┐
          │              AI AGENT (LangGraph)                │
          │                                                  │
          │   ┌──────────┐     ┌───────────────────────┐    │
          │   │  LLM     │────►│  KILL SWITCH GUARD     │    │
          │   │  (GPT-4o)│     │  (intercepts all       │    │
          │   └──────────┘     │   tool calls)          │    │
          │                    └───────────┬───────────┘    │
          │                                │                 │
          │         ┌──────────┬──────────┬──────────┐      │
          │         ▼          ▼          ▼          ▼      │
          │   ┌──────────┐┌────────┐┌─────────┐┌────────┐  │
          │   │  File    ││  DB    ││  API    ││  Code  │  │
          │   │  Manager ││ Writer ││ Caller  ││Executor│  │
          │   └──────────┘└────────┘└─────────┘└────────┘  │
          └─────────────────────────────────────────────────┘
                 │
                 │  metrics + logs + traces
                 ▼
          ┌─────────────────────────────────────────────────┐
          │           OBSERVABILITY STACK                    │
          │                                                  │
          │   Prometheus ──► Grafana (metrics dashboards)   │
          │   Loki      ──► Grafana (log aggregation)       │
          │   Langfuse        (LLM reasoning traces)        │
          └─────────────────────────────────────────────────┘
```

### Data Flow — Normal Execution

```
1. User sends task ──► Control Plane API
2. API creates task record in PostgreSQL
3. Agent receives task, starts ReAct loop
4. Agent decides to call a tool (e.g. db_writer)
5. Kill Switch Guard checks Redis:
   ├── Global kill active?     → YES → Raise HardKillException
   ├── Tool disabled?          → YES → Raise CapabilityDisabledException
   └── All clear?              → YES → Proceed with tool call
6. Tool executes, result returned to agent
7. Metrics emitted to Prometheus
8. Logs emitted to Loki
9. LLM trace recorded in Langfuse
10. Agent continues to next step (loop back to 4)
```

### Data Flow — Kill Switch Activation

```
1. Operator hits "KILL" on dashboard
2. Dashboard calls POST /kill on Control Plane API
3. API sets kill flag in Redis
4. API publishes "kill" event on Redis Pub/Sub
5. API logs event in PostgreSQL audit table
6. Agent's next tool call hits Kill Switch Guard
7. Guard reads Redis → kill flag is active
8. Guard raises HardKillException
9. Agent loop terminates
10. Agent state is checkpointed (LangGraph)
11. Dashboard receives kill confirmation via WebSocket
12. Grafana dashboard shows kill event on timeline
```

---

## 4. Tech Stack — Detailed Justification

### 4.1 Agent Framework: LangGraph

| Aspect | Detail |
|--------|--------|
| **What it is** | A framework by LangChain for building stateful, multi-step AI agents as graphs |
| **Why we chose it** | Native support for **interrupts** (pause agent mid-execution), **checkpointing** (save and resume agent state), and **human-in-the-loop** patterns — all essential for kill switch implementation |
| **Alternatives considered** | AutoGen (Microsoft) — good for multi-agent, but weaker interrupt support; CrewAI — higher-level abstraction, less control over execution flow |
| **Cost** | Free and open-source |
| **Key feature for this project** | `interrupt_before` and `interrupt_after` hooks on graph nodes let us inject kill switch checks natively |

### 4.2 LLM: OpenAI GPT-4o (or Anthropic Claude)

| Aspect | Detail |
|--------|--------|
| **What it is** | Large Language Model that powers the agent's reasoning |
| **Why we chose it** | Best-in-class tool calling support (structured JSON output for tool invocations), fast, reliable |
| **Alternatives** | Claude 3.5 Sonnet (excellent reasoning, better safety alignment), Llama 3 (free but weaker tool calling) |
| **Cost** | Pay-per-token (GPT-4o: ~$2.50/1M input tokens, ~$10/1M output tokens) |
| **Key feature** | Structured tool calling with JSON schema — agent reliably picks and parameterises tools |

### 4.3 Kill Switch State Store: Redis

| Aspect | Detail |
|--------|--------|
| **What it is** | In-memory key-value store with pub/sub support |
| **Why we chose it** | Sub-millisecond reads (critical — the guard checks Redis on EVERY tool call, so latency must be near-zero), built-in TTL for heartbeat expiry, Pub/Sub for real-time kill event broadcasting |
| **Alternatives considered** | PostgreSQL — too slow for per-action polling (5-20ms vs 0.1ms); etcd — more complex to operate, designed for config not runtime flags; Environment variables — no way to change at runtime without restart |
| **Cost** | Free and open-source |
| **Key feature** | `SETEX` command — set a key with automatic expiry (used for heartbeat: if agent doesn't refresh the key within 30s, it auto-expires and watchdog triggers) |

### 4.4 Audit Log & Task History: PostgreSQL

| Aspect | Detail |
|--------|--------|
| **What it is** | Relational database for structured, durable data |
| **Why we chose it** | ACID compliance ensures audit log entries cannot be lost or corrupted; SQL queries make it easy to search audit history ("show me all kill events in the last 24 hours"); append-only table design prevents tampering |
| **Alternatives considered** | MongoDB — no ACID by default, harder to enforce append-only; SQLite — no concurrent access, not suitable for multi-service architecture |
| **Cost** | Free and open-source |
| **Key feature** | Append-only audit table with immutable records — critical for compliance and post-incident analysis |

### 4.5 Control Plane API: FastAPI

| Aspect | Detail |
|--------|--------|
| **What it is** | Modern, high-performance Python web framework |
| **Why we chose it** | Native async support (non-blocking API calls), automatic OpenAPI docs (operators can test endpoints via Swagger UI), built-in WebSocket support (real-time dashboard updates), Python-native (same language as the agent — shared models and utilities) |
| **Alternatives considered** | Flask — no native async, no WebSocket support; Express.js — different language from agent, adds complexity; Django — too heavy for a microservice API |
| **Cost** | Free and open-source |
| **Key feature** | WebSocket endpoints for real-time push of kill switch events to the frontend dashboard |

### 4.6 Frontend Dashboard: React + Tailwind CSS + Recharts

| Aspect | Detail |
|--------|--------|
| **What it is** | Modern web application for operator control |
| **Why we chose React** | Component-based architecture ideal for a dashboard with many independent widgets (status indicators, charts, logs); massive ecosystem; easy state management with hooks |
| **Why Tailwind CSS** | Utility-first CSS — rapid UI development without writing custom CSS files; consistent design system; small production bundle |
| **Why Recharts** | React-native charting library — renders metrics charts (tool call rate, error rate) directly in the dashboard without embedding Grafana iframes |
| **Alternatives considered** | Streamlit — faster to prototype but limited customisation and no real WebSocket support; Vue — equally capable but React has broader adoption; Angular — too heavy for a dashboard |
| **Cost** | All free and open-source |
| **Key feature** | WebSocket integration for real-time log streaming and kill switch status updates |

### 4.7 Metrics Collection: Prometheus

| Aspect | Detail |
|--------|--------|
| **What it is** | Time-series database that scrapes metrics from your services |
| **Why we chose it** | Industry standard for infrastructure monitoring; pull-based model (Prometheus scrapes your `/metrics` endpoint — your agent doesn't need to push data); powerful query language (PromQL) for aggregations |
| **Role in this project** | Collects numerical metrics: kill switch activations, tool call counts, error rates, agent uptime, latency percentiles |
| **Cost** | Free and open-source (Apache 2.0 license) |
| **Key feature** | Alerting rules — automatically fire an alert if (for example) the agent's error rate exceeds 10% in a 5-minute window |

### 4.8 Log Aggregation: Grafana Loki

| Aspect | Detail |
|--------|--------|
| **What it is** | Log aggregation system designed to work with Grafana |
| **Why we chose it** | Integrates natively with Grafana (same dashboard for metrics AND logs); lightweight compared to ELK stack (Elasticsearch + Logstash + Kibana); label-based indexing is efficient for log querying |
| **Alternatives considered** | ELK Stack — powerful but extremely resource-heavy (Elasticsearch alone needs 2-4GB RAM minimum); Fluentd — a log forwarder, not a storage/query system |
| **Role in this project** | Stores and queries all agent logs: tool call logs, kill switch events, guard decisions, error traces |
| **Cost** | Free and open-source |

### 4.9 Metrics & Log Visualisation: Grafana

| Aspect | Detail |
|--------|--------|
| **What it is** | Dashboard and visualisation platform |
| **Why we chose it** | Unified view of metrics (from Prometheus) AND logs (from Loki) in a single interface; rich visualisation options (time series, heatmaps, tables, alerts); template variables for dynamic dashboards |
| **Role in this project** | Operator-facing dashboards showing agent health, kill switch history, tool usage patterns, error trends |
| **Cost** | Free and open-source (self-hosted) |

### 4.10 LLM Tracing: Langfuse

| Aspect | Detail |
|--------|--------|
| **What it is** | Open-source observability platform specifically for LLM applications |
| **Why we chose it** | Captures the full LLM reasoning chain (prompt → response → tool choice → tool result → next prompt); cost tracking per agent run (token counts, estimated $); latency breakdown per LLM call |
| **Alternatives considered** | LangSmith — excellent but proprietary and locked to LangChain ecosystem; Arize Phoenix — good but less mature; OpenTelemetry — too generic, doesn't understand LLM-specific traces |
| **Role in this project** | Understanding WHY the agent made certain decisions before the kill switch was triggered — essential for post-incident analysis |
| **Cost** | Free self-hosted; free cloud tier (50k observations/month) |

### 4.11 Containerisation: Docker + Docker Compose

| Aspect | Detail |
|--------|--------|
| **What it is** | Container runtime and multi-container orchestration |
| **Why we chose it** | Single `docker compose up` command starts the entire stack (agent, API, Redis, PostgreSQL, Prometheus, Grafana, Loki, dashboard); reproducible environments; easy to share and demo |
| **Cost** | Free and open-source |

### Full Stack Summary

```
┌──────────────────────────────────────────────────────────────┐
│                        TECH STACK                            │
├──────────────────────────────────────────────────────────────┤
│  Agent Intelligence    │  OpenAI GPT-4o / Claude 3.5        │
│  Agent Framework       │  LangGraph (Python)                │
│  Kill Switch State     │  Redis 7+                          │
│  Persistent Storage    │  PostgreSQL 16                     │
│  Control Plane API     │  FastAPI (Python 3.11+)            │
│  Frontend Dashboard    │  React 18 + Tailwind CSS + Recharts│
│  Metrics               │  Prometheus                        │
│  Logs                  │  Grafana Loki                      │
│  Dashboards            │  Grafana                           │
│  LLM Traces            │  Langfuse                          │
│  Containerisation      │  Docker + Docker Compose           │
│  Language              │  Python 3.11+ (backend)            │
│                        │  TypeScript (frontend)             │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Project Structure

```
killswitch-agent/
│
├── README.md                          # This file
├── .env.example                       # Environment variable template
├── .gitignore
├── docker-compose.yml                 # Full stack orchestration
├── Makefile                           # Convenience commands
│
├── agent/                             # AI Agent (Python)
│   ├── __init__.py
│   ├── main.py                        # Entry point — starts the agent
│   ├── config.py                      # Configuration loader
│   ├── core/
│   │   ├── __init__.py
│   │   ├── graph.py                   # LangGraph agent definition
│   │   ├── state.py                   # Agent state schema
│   │   └── prompts.py                 # System prompts
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── base.py                    # Base tool class with guard integration
│   │   ├── file_manager.py            # File read/write/delete tool
│   │   ├── db_writer.py               # SQL execution tool
│   │   ├── api_caller.py              # HTTP request tool
│   │   └── code_executor.py           # Python/shell execution tool
│   ├── guard/
│   │   ├── __init__.py
│   │   ├── kill_switch_guard.py       # Core guard — intercepts all tool calls
│   │   └── guardrails.py             # Input/output validation rules
│   ├── observability/
│   │   ├── __init__.py
│   │   ├── metrics.py                 # Prometheus metrics definitions
│   │   ├── logger.py                  # Structured logging (JSON, Loki-compatible)
│   │   └── tracer.py                  # Langfuse LLM tracing
│   ├── Dockerfile
│   └── requirements.txt
│
├── kill_switch/                       # Kill Switch Core Library (Python)
│   ├── __init__.py
│   ├── state_manager.py               # Redis-backed kill switch state
│   ├── audit.py                       # PostgreSQL audit logger
│   ├── watchdog.py                    # Heartbeat monitor service
│   ├── events.py                      # Redis Pub/Sub event system
│   └── models.py                      # Shared Pydantic models
│
├── control_plane/                     # Control Plane API (FastAPI)
│   ├── __init__.py
│   ├── main.py                        # FastAPI app entry point
│   ├── config.py                      # Configuration
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── kill_switch.py             # Kill/revive/disable endpoints
│   │   ├── agent.py                   # Agent status and task management
│   │   ├── audit.py                   # Audit log query endpoints
│   │   └── websocket.py              # Real-time event streaming
│   ├── middleware/
│   │   ├── __init__.py
│   │   └── auth.py                    # API key / JWT authentication
│   ├── Dockerfile
│   └── requirements.txt
│
├── dashboard/                         # Frontend Dashboard (React)
│   ├── package.json
│   ├── tsconfig.json
│   ├── tailwind.config.js
│   ├── vite.config.ts
│   ├── Dockerfile
│   ├── public/
│   │   └── index.html
│   └── src/
│       ├── App.tsx                    # Main app with routing
│       ├── main.tsx                   # Entry point
│       ├── index.css                  # Tailwind imports
│       ├── api/
│       │   └── client.ts              # API client (axios + WebSocket)
│       ├── components/
│       │   ├── KillSwitchPanel.tsx     # Big red button + status
│       │   ├── CapabilityToggles.tsx   # Per-tool enable/disable
│       │   ├── AuditLog.tsx           # Real-time audit log feed
│       │   ├── AgentStatus.tsx        # Agent health indicator
│       │   ├── MetricsChart.tsx       # Tool call rate, error rate charts
│       │   ├── LiveLogStream.tsx      # WebSocket-powered log stream
│       │   └── Navbar.tsx             # Navigation bar
│       ├── hooks/
│       │   ├── useWebSocket.ts        # WebSocket connection hook
│       │   └── useKillSwitch.ts       # Kill switch state hook
│       ├── pages/
│       │   ├── Dashboard.tsx          # Main dashboard page
│       │   ├── AuditPage.tsx         # Full audit log with search
│       │   └── SettingsPage.tsx       # Configuration page
│       └── types/
│           └── index.ts               # TypeScript type definitions
│
├── observability/                     # Observability Configuration
│   ├── prometheus/
│   │   └── prometheus.yml             # Prometheus scrape config
│   ├── grafana/
│   │   ├── provisioning/
│   │   │   ├── datasources/
│   │   │   │   └── datasources.yml    # Auto-provision Prometheus + Loki
│   │   │   └── dashboards/
│   │   │       ├── dashboard.yml      # Dashboard provisioning config
│   │   │       └── agent-dashboard.json # Pre-built Grafana dashboard
│   │   └── grafana.ini                # Grafana configuration
│   └── loki/
│       └── loki-config.yml            # Loki storage and retention config
│
├── database/                          # Database Migrations
│   ├── init.sql                       # Initial schema (audit log, tasks)
│   └── seed.sql                       # Optional seed data for development
│
└── tests/                             # Test Suite
    ├── __init__.py
    ├── conftest.py                    # Shared fixtures (Redis mock, DB mock)
    ├── test_kill_switch_guard.py      # Guard unit tests
    ├── test_state_manager.py          # Redis state manager tests
    ├── test_hard_kill.py              # Integration: hard kill stops agent
    ├── test_soft_kill.py              # Integration: soft kill drains tasks
    ├── test_capability_kill.py        # Integration: disable specific tools
    ├── test_watchdog.py               # Watchdog auto-kill tests
    ├── test_audit_log.py              # Audit immutability tests
    └── test_api.py                    # Control plane API endpoint tests
```

---

## 6. Core Components — Full Implementation

### 6.1 Agent Core (LangGraph ReAct Agent)

**File: `agent/core/state.py`**

This defines the state schema that flows through the agent graph. Every node in the graph reads from and writes to this state.

```python
"""Agent state schema for the LangGraph ReAct agent."""

from typing import Annotated, TypedDict, Sequence
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages


class AgentState(TypedDict):
    """
    The state object that flows through every node in the agent graph.
    
    Attributes:
        messages: Conversation history (system + user + assistant + tool messages).
                  Uses LangGraph's add_messages reducer to append, not overwrite.
        task_id: Unique identifier for the current task.
        status: Current agent status — "running", "paused", "killed", "completed".
        current_tool: Name of the tool currently being executed (for capability checks).
        checkpoint_id: ID of the last saved checkpoint (for resume after kill).
    """
    messages: Annotated[Sequence[BaseMessage], add_messages]
    task_id: str
    status: str
    current_tool: str | None
    checkpoint_id: str | None
```

**File: `agent/core/prompts.py`**

```python
"""System prompts for the AI agent."""

SYSTEM_PROMPT = """You are OperatorBot, an autonomous task execution agent.

You have access to the following tools:
- file_manager: Read, write, and delete files on the filesystem.
- db_writer: Execute SQL queries (SELECT, INSERT, UPDATE, DELETE) on PostgreSQL.
- api_caller: Make HTTP requests (GET, POST, PUT, DELETE) to external APIs.
- code_executor: Execute Python or shell scripts in a sandboxed environment.

RULES:
1. Always explain your reasoning BEFORE calling a tool.
2. Use the minimum number of tool calls necessary.
3. If a tool call fails, analyse the error and try a different approach.
4. Never execute destructive operations (DELETE, DROP, rm -rf) without explicit user confirmation.
5. Report your final result clearly when the task is complete.

You will be stopped if a kill switch is activated. This is normal — cooperate with the shutdown."""
```

**File: `agent/core/graph.py`**

This is the heart of the agent — the LangGraph graph definition with kill switch integration.

```python
"""LangGraph agent definition with kill switch integration."""

import os
from typing import Literal
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.prebuilt import ToolNode

from agent.core.state import AgentState
from agent.core.prompts import SYSTEM_PROMPT
from agent.tools import get_all_tools
from agent.guard.kill_switch_guard import KillSwitchGuard
from agent.observability.metrics import (
    agent_tool_calls_total,
    agent_llm_calls_total,
    agent_errors_total,
)
from agent.observability.logger import get_logger
from agent.observability.tracer import get_langfuse_handler

logger = get_logger(__name__)
guard = KillSwitchGuard()


def create_agent_graph():
    """
    Build and compile the LangGraph agent.
    
    The graph has three nodes:
    1. 'reasoner' — LLM decides the next action
    2. 'guard_check' — Kill Switch Guard verifies the action is allowed
    3. 'tool_executor' — Executes the approved tool call
    
    Flow:
        reasoner → should_continue? → guard_check → tool_executor → reasoner (loop)
                                   ↘ END (if task complete or killed)
    """
    
    # --- LLM Setup ---
    llm = ChatOpenAI(
        model=os.getenv("OPENAI_MODEL", "gpt-4o"),
        temperature=0,
        api_key=os.getenv("OPENAI_API_KEY"),
    )
    
    tools = get_all_tools()
    llm_with_tools = llm.bind_tools(tools)
    langfuse_handler = get_langfuse_handler()
    
    # --- Node: Reasoner ---
    def reasoner_node(state: AgentState) -> dict:
        """LLM reasons about the next step and decides which tool to call."""
        logger.info("Agent entering reasoner node", extra={"task_id": state["task_id"]})
        agent_llm_calls_total.inc()
        
        messages = [SystemMessage(content=SYSTEM_PROMPT)] + list(state["messages"])
        
        response = llm_with_tools.invoke(
            messages,
            config={"callbacks": [langfuse_handler]} if langfuse_handler else {},
        )
        
        return {"messages": [response]}
    
    # --- Node: Guard Check ---
    def guard_check_node(state: AgentState) -> dict:
        """
        Kill Switch Guard intercepts the tool call BEFORE execution.
        
        This is the critical safety node. It checks:
        1. Is the global kill switch active?
        2. Is the specific tool capability disabled?
        3. Does the tool call pass guardrail validation?
        
        If any check fails, the agent is halted.
        """
        last_message = state["messages"][-1]
        
        if not hasattr(last_message, "tool_calls") or not last_message.tool_calls:
            return {"status": "completed"}
        
        for tool_call in last_message.tool_calls:
            tool_name = tool_call["name"]
            tool_args = tool_call["args"]
            
            logger.info(
                f"Guard checking tool call: {tool_name}",
                extra={
                    "task_id": state["task_id"],
                    "tool_name": tool_name,
                    "tool_args": tool_args,
                },
            )
            
            # This raises an exception if the kill switch is active
            guard.check(tool_name, tool_args)
            
            agent_tool_calls_total.labels(tool=tool_name, status="approved").inc()
        
        return {"current_tool": tool_name}
    
    # --- Node: Tool Executor ---
    tool_node = ToolNode(tools)
    
    # --- Routing ---
    def should_continue(state: AgentState) -> Literal["guard_check", "end"]:
        """Decide whether to continue the loop or end."""
        last_message = state["messages"][-1]
        
        if state.get("status") == "killed":
            logger.warning("Agent was killed — ending execution")
            return "end"
        
        if hasattr(last_message, "tool_calls") and last_message.tool_calls:
            return "guard_check"
        
        return "end"
    
    # --- Build Graph ---
    graph = StateGraph(AgentState)
    
    graph.add_node("reasoner", reasoner_node)
    graph.add_node("guard_check", guard_check_node)
    graph.add_node("tool_executor", tool_node)
    
    graph.set_entry_point("reasoner")
    
    graph.add_conditional_edges(
        "reasoner",
        should_continue,
        {"guard_check": "guard_check", "end": END},
    )
    graph.add_edge("guard_check", "tool_executor")
    graph.add_edge("tool_executor", "reasoner")
    
    # --- Compile with Checkpointing ---
    checkpointer = PostgresSaver.from_conn_string(os.getenv("DATABASE_URL"))
    
    compiled = graph.compile(
        checkpointer=checkpointer,
        interrupt_before=["tool_executor"],  # LangGraph native interrupt point
    )
    
    return compiled
```

**File: `agent/main.py`**

```python
"""Agent entry point — starts the autonomous agent loop."""

import os
import asyncio
import uuid
from dotenv import load_dotenv
from langchain_core.messages import HumanMessage

from agent.core.graph import create_agent_graph
from agent.observability.logger import get_logger
from agent.observability.metrics import start_metrics_server
from kill_switch.state_manager import KillSwitchStateManager
from kill_switch.events import KillSwitchEventListener

load_dotenv()
logger = get_logger(__name__)


async def run_agent(task: str):
    """
    Run the agent on a single task.
    
    Args:
        task: The natural language task for the agent to execute.
    """
    task_id = str(uuid.uuid4())
    logger.info(f"Starting agent for task: {task}", extra={"task_id": task_id})
    
    graph = create_agent_graph()
    state_manager = KillSwitchStateManager()
    
    config = {"configurable": {"thread_id": task_id}}
    
    initial_state = {
        "messages": [HumanMessage(content=task)],
        "task_id": task_id,
        "status": "running",
        "current_tool": None,
        "checkpoint_id": None,
    }
    
    try:
        # Start heartbeat
        state_manager.start_heartbeat(task_id)
        
        # Run the agent graph
        result = await graph.ainvoke(initial_state, config=config)
        
        logger.info(
            f"Agent completed task: {task_id}",
            extra={"task_id": task_id, "status": "completed"},
        )
        return result
        
    except Exception as e:
        logger.error(
            f"Agent terminated: {str(e)}",
            extra={"task_id": task_id, "error": str(e)},
        )
        raise
    finally:
        state_manager.stop_heartbeat()


async def main():
    """Main entry point — start metrics server and wait for tasks."""
    start_metrics_server(port=int(os.getenv("METRICS_PORT", "8001")))
    
    logger.info("OperatorBot agent started and waiting for tasks")
    
    # In production, this would poll a task queue (RabbitMQ, Redis, etc.)
    # For demo purposes, run a single task:
    demo_task = os.getenv("DEMO_TASK", "List all files in the /data directory")
    await run_agent(demo_task)


if __name__ == "__main__":
    asyncio.run(main())
```

---

### 6.2 Agent Tools

**File: `agent/tools/base.py`**

```python
"""Base tool class with kill switch guard integration."""

from langchain_core.tools import tool
from agent.guard.kill_switch_guard import KillSwitchGuard

guard = KillSwitchGuard()


def guarded_tool(func):
    """
    Decorator that wraps a LangChain tool with kill switch guard checks.
    Every tool call passes through this decorator before execution.
    """
    original = func
    
    def wrapper(*args, **kwargs):
        # Guard checks happen in the guard_check_node,
        # but this is a secondary safety net at the tool level
        guard.check(original.name, kwargs)
        return original.invoke(*args, **kwargs)
    
    wrapper.__name__ = original.name
    wrapper.__doc__ = original.description
    return wrapper
```

**File: `agent/tools/file_manager.py`**

```python
"""File management tool — read, write, delete files."""

import os
from langchain_core.tools import tool


@tool
def file_manager(action: str, path: str, content: str = "") -> str:
    """
    Manage files on the filesystem.
    
    Args:
        action: One of 'read', 'write', 'delete', 'list'.
        path: File or directory path.
        content: Content to write (only for 'write' action).
    
    Returns:
        Result of the file operation.
    """
    ALLOWED_BASE = os.getenv("AGENT_WORKSPACE", "/tmp/agent_workspace")
    
    # Security: prevent path traversal
    full_path = os.path.abspath(os.path.join(ALLOWED_BASE, path))
    if not full_path.startswith(ALLOWED_BASE):
        return "ERROR: Path traversal detected. Access denied."
    
    try:
        if action == "read":
            with open(full_path, "r") as f:
                return f.read()
        
        elif action == "write":
            os.makedirs(os.path.dirname(full_path), exist_ok=True)
            with open(full_path, "w") as f:
                f.write(content)
            return f"File written: {path}"
        
        elif action == "delete":
            os.remove(full_path)
            return f"File deleted: {path}"
        
        elif action == "list":
            entries = os.listdir(full_path)
            return "\n".join(entries) if entries else "(empty directory)"
        
        else:
            return f"ERROR: Unknown action '{action}'. Use read/write/delete/list."
    
    except Exception as e:
        return f"ERROR: {str(e)}"
```

**File: `agent/tools/db_writer.py`**

```python
"""Database tool — execute SQL queries on PostgreSQL."""

import os
import psycopg2
from langchain_core.tools import tool


@tool
def db_writer(query: str, params: list = None) -> str:
    """
    Execute a SQL query on the PostgreSQL database.
    
    Args:
        query: SQL query to execute (SELECT, INSERT, UPDATE, DELETE).
        params: Optional list of query parameters for parameterised queries.
    
    Returns:
        Query results for SELECT, or affected row count for mutations.
    
    WARNING: DELETE and DROP operations should only be used when explicitly
    requested by the user. The kill switch guard may block these operations.
    """
    conn = None
    try:
        conn = psycopg2.connect(os.getenv("AGENT_DATABASE_URL"))
        cursor = conn.cursor()
        
        cursor.execute(query, params or [])
        
        if query.strip().upper().startswith("SELECT"):
            columns = [desc[0] for desc in cursor.description]
            rows = cursor.fetchall()
            result = [dict(zip(columns, row)) for row in rows]
            return str(result)
        else:
            conn.commit()
            return f"Query executed. Rows affected: {cursor.rowcount}"
    
    except Exception as e:
        if conn:
            conn.rollback()
        return f"ERROR: {str(e)}"
    
    finally:
        if conn:
            conn.close()
```

**File: `agent/tools/api_caller.py`**

```python
"""HTTP API calling tool — make requests to external services."""

import requests
from langchain_core.tools import tool


@tool
def api_caller(
    method: str,
    url: str,
    headers: dict = None,
    body: dict = None,
    timeout: int = 30,
) -> str:
    """
    Make an HTTP request to an external API.
    
    Args:
        method: HTTP method — GET, POST, PUT, DELETE.
        url: Full URL to call.
        headers: Optional request headers (dict).
        body: Optional request body for POST/PUT (dict).
        timeout: Request timeout in seconds (default: 30).
    
    Returns:
        Response status code and body.
    """
    BLOCKED_DOMAINS = ["localhost", "127.0.0.1", "0.0.0.0", "169.254.169.254"]
    
    # Security: block internal network access
    from urllib.parse import urlparse
    parsed = urlparse(url)
    if parsed.hostname in BLOCKED_DOMAINS:
        return "ERROR: Access to internal network addresses is blocked."
    
    try:
        response = requests.request(
            method=method.upper(),
            url=url,
            headers=headers or {},
            json=body,
            timeout=timeout,
        )
        
        return (
            f"Status: {response.status_code}\n"
            f"Headers: {dict(response.headers)}\n"
            f"Body: {response.text[:2000]}"  # Truncate large responses
        )
    
    except requests.exceptions.Timeout:
        return f"ERROR: Request timed out after {timeout}s"
    except Exception as e:
        return f"ERROR: {str(e)}"
```

**File: `agent/tools/code_executor.py`**

```python
"""Code execution tool — run Python/shell scripts in a sandboxed environment."""

import subprocess
import tempfile
import os
from langchain_core.tools import tool


@tool
def code_executor(code: str, language: str = "python") -> str:
    """
    Execute code in a sandboxed environment.
    
    Args:
        code: The code to execute.
        language: 'python' or 'bash'.
    
    Returns:
        stdout and stderr output from the execution.
    """
    TIMEOUT = int(os.getenv("CODE_EXECUTION_TIMEOUT", "30"))
    
    if language not in ("python", "bash"):
        return "ERROR: Only 'python' and 'bash' are supported."
    
    ext = ".py" if language == "python" else ".sh"
    cmd = ["python3"] if language == "python" else ["bash"]
    
    try:
        with tempfile.NamedTemporaryFile(
            mode="w", suffix=ext, delete=False, dir="/tmp"
        ) as f:
            f.write(code)
            f.flush()
            script_path = f.name
        
        result = subprocess.run(
            cmd + [script_path],
            capture_output=True,
            text=True,
            timeout=TIMEOUT,
            cwd="/tmp/agent_workspace",
            env={
                "PATH": "/usr/bin:/bin",
                "HOME": "/tmp",
            },
        )
        
        output = ""
        if result.stdout:
            output += f"STDOUT:\n{result.stdout}\n"
        if result.stderr:
            output += f"STDERR:\n{result.stderr}\n"
        output += f"Exit code: {result.returncode}"
        
        return output or "No output."
    
    except subprocess.TimeoutExpired:
        return f"ERROR: Execution timed out after {TIMEOUT}s"
    except Exception as e:
        return f"ERROR: {str(e)}"
    finally:
        if os.path.exists(script_path):
            os.unlink(script_path)
```

**File: `agent/tools/__init__.py`**

```python
"""Register all agent tools."""

from agent.tools.file_manager import file_manager
from agent.tools.db_writer import db_writer
from agent.tools.api_caller import api_caller
from agent.tools.code_executor import code_executor


def get_all_tools():
    """Return all tools available to the agent."""
    return [file_manager, db_writer, api_caller, code_executor]


TOOL_NAMES = ["file_manager", "db_writer", "api_caller", "code_executor"]
```

---

### 6.3 Kill Switch Guard

**File: `agent/guard/kill_switch_guard.py`**

This is the most critical file in the project — it sits between the agent's decision and every tool execution.

```python
"""
Kill Switch Guard — the central safety checkpoint.

Every tool call passes through this guard BEFORE execution.
The guard checks three things:
1. Is the global kill switch active? (Hard Kill)
2. Is the specific tool disabled? (Capability Kill)
3. Does the tool call pass guardrail validation? (Input Validation)

If ANY check fails, an exception is raised and the agent is halted.
"""

import time
from kill_switch.state_manager import KillSwitchStateManager
from kill_switch.audit import AuditLogger
from agent.guard.guardrails import validate_tool_call
from agent.observability.metrics import (
    agent_tool_calls_total,
    kill_switch_activations_total,
)
from agent.observability.logger import get_logger

logger = get_logger(__name__)


class HardKillException(Exception):
    """Raised when the global kill switch is active."""
    pass


class CapabilityDisabledException(Exception):
    """Raised when a specific tool/capability is disabled."""
    pass


class GuardrailViolation(Exception):
    """Raised when a tool call violates input guardrails."""
    pass


class KillSwitchGuard:
    """
    The Kill Switch Guard intercepts every tool call and decides
    whether to allow or block it.
    
    Usage:
        guard = KillSwitchGuard()
        guard.check("db_writer", {"query": "SELECT * FROM users"})
    """
    
    def __init__(self):
        self.state_manager = KillSwitchStateManager()
        self.audit = AuditLogger()
    
    def check(self, tool_name: str, tool_args: dict) -> bool:
        """
        Run all kill switch and guardrail checks.
        
        Args:
            tool_name: Name of the tool being called.
            tool_args: Arguments passed to the tool.
        
        Returns:
            True if the tool call is allowed.
        
        Raises:
            HardKillException: If global kill switch is active.
            CapabilityDisabledException: If the tool is disabled.
            GuardrailViolation: If the tool call violates a guardrail.
        """
        check_start = time.time()
        
        # --- CHECK 1: Global Kill Switch ---
        if self.state_manager.is_killed():
            logger.critical(
                "HARD KILL — agent stopped",
                extra={"tool_name": tool_name, "tool_args": tool_args},
            )
            kill_switch_activations_total.labels(level="hard").inc()
            
            self.audit.log(
                action="hard_kill_blocked",
                tool_name=tool_name,
                tool_args=tool_args,
                reason="Global kill switch is active",
            )
            
            raise HardKillException(
                "Global kill switch is ACTIVE. All agent operations are halted. "
                "Contact the operator to revive the agent."
            )
        
        # --- CHECK 2: Capability Kill ---
        if self.state_manager.is_capability_disabled(tool_name):
            logger.warning(
                f"CAPABILITY KILL — {tool_name} is disabled",
                extra={"tool_name": tool_name, "tool_args": tool_args},
            )
            kill_switch_activations_total.labels(level="capability").inc()
            
            self.audit.log(
                action="capability_kill_blocked",
                tool_name=tool_name,
                tool_args=tool_args,
                reason=f"Tool '{tool_name}' is currently disabled by operator",
            )
            
            raise CapabilityDisabledException(
                f"Tool '{tool_name}' has been disabled by the operator. "
                f"The agent cannot use this tool until it is re-enabled."
            )
        
        # --- CHECK 3: Guardrail Validation ---
        violation = validate_tool_call(tool_name, tool_args)
        if violation:
            logger.warning(
                f"GUARDRAIL VIOLATION — {violation}",
                extra={"tool_name": tool_name, "tool_args": tool_args},
            )
            agent_tool_calls_total.labels(tool=tool_name, status="guardrail_blocked").inc()
            
            self.audit.log(
                action="guardrail_blocked",
                tool_name=tool_name,
                tool_args=tool_args,
                reason=violation,
            )
            
            raise GuardrailViolation(violation)
        
        # --- ALL CHECKS PASSED ---
        check_duration = time.time() - check_start
        logger.info(
            f"Guard APPROVED: {tool_name} (check took {check_duration:.3f}s)",
            extra={"tool_name": tool_name, "check_duration_ms": check_duration * 1000},
        )
        
        self.audit.log(
            action="tool_approved",
            tool_name=tool_name,
            tool_args=tool_args,
            reason="All checks passed",
        )
        
        return True
```

**File: `agent/guard/guardrails.py`**

```python
"""
Input guardrails — automated rules that block dangerous tool calls
BEFORE they reach the kill switch level.

These are complementary to the kill switch: the kill switch is a human-triggered
emergency stop; guardrails are automated safety nets.
"""

import re


# Dangerous SQL patterns that should never be executed without explicit approval
DANGEROUS_SQL_PATTERNS = [
    r"\bDROP\s+TABLE\b",
    r"\bDROP\s+DATABASE\b",
    r"\bTRUNCATE\b",
    r"\bDELETE\s+FROM\s+\w+\s*$",  # DELETE without WHERE clause
    r"\bALTER\s+TABLE\b.*\bDROP\b",
]

# Blocked URL patterns for the API caller
BLOCKED_URL_PATTERNS = [
    r"https?://169\.254\.169\.254",   # AWS metadata endpoint
    r"https?://metadata\.google",     # GCP metadata
    r"https?://localhost",
    r"https?://127\.0\.0\.1",
    r"https?://0\.0\.0\.0",
]

# Dangerous shell commands
DANGEROUS_SHELL_PATTERNS = [
    r"\brm\s+-rf\s+/",
    r"\bmkfs\b",
    r"\bdd\s+if=",
    r"\b:(){ :|:& };:",  # Fork bomb
    r"\bchmod\s+-R\s+777\s+/",
    r"\bcurl\b.*\|\s*bash",  # Piping curl to bash
]


def validate_tool_call(tool_name: str, tool_args: dict) -> str | None:
    """
    Validate a tool call against guardrail rules.
    
    Args:
        tool_name: Name of the tool.
        tool_args: Arguments passed to the tool.
    
    Returns:
        None if the call is safe; a violation description string if blocked.
    """
    
    if tool_name == "db_writer":
        query = tool_args.get("query", "")
        for pattern in DANGEROUS_SQL_PATTERNS:
            if re.search(pattern, query, re.IGNORECASE):
                return (
                    f"Guardrail violation: SQL query matches dangerous pattern "
                    f"'{pattern}'. Query: '{query[:100]}...'"
                )
    
    elif tool_name == "api_caller":
        url = tool_args.get("url", "")
        for pattern in BLOCKED_URL_PATTERNS:
            if re.search(pattern, url, re.IGNORECASE):
                return (
                    f"Guardrail violation: URL matches blocked pattern "
                    f"'{pattern}'. URL: '{url}'"
                )
    
    elif tool_name == "code_executor":
        code = tool_args.get("code", "")
        for pattern in DANGEROUS_SHELL_PATTERNS:
            if re.search(pattern, code, re.IGNORECASE):
                return (
                    f"Guardrail violation: Code matches dangerous pattern "
                    f"'{pattern}'."
                )
    
    elif tool_name == "file_manager":
        action = tool_args.get("action", "")
        path = tool_args.get("path", "")
        
        # Block deletion of root or parent directories
        if action == "delete" and path in ("/", ".", "..", "~"):
            return f"Guardrail violation: Cannot delete path '{path}'."
    
    return None  # All checks passed
```

---

### 6.4 Kill Switch State Manager (Redis)

**File: `kill_switch/state_manager.py`**

```python
"""
Redis-backed Kill Switch State Manager.

This is the source of truth for the kill switch state. All services
(agent, control plane, watchdog) read from and write to Redis.

Why Redis?
- Sub-millisecond reads (the guard checks Redis on EVERY tool call)
- Built-in TTL for heartbeat auto-expiry
- Pub/Sub for real-time event broadcasting
- Atomic operations prevent race conditions
"""

import os
import time
import threading
import redis
from agent.observability.logger import get_logger

logger = get_logger(__name__)


class KillSwitchStateManager:
    """Manages kill switch state in Redis."""
    
    # Redis key namespace
    GLOBAL_KILL_KEY = "killswitch:global"
    CAPABILITY_HASH_KEY = "killswitch:capabilities"
    HEARTBEAT_KEY_PREFIX = "killswitch:heartbeat:"
    SOFT_KILL_KEY = "killswitch:soft_kill"
    KILL_REASON_KEY = "killswitch:reason"
    KILL_ACTOR_KEY = "killswitch:actor"
    KILL_TIMESTAMP_KEY = "killswitch:timestamp"
    
    def __init__(self):
        self.redis = redis.Redis(
            host=os.getenv("REDIS_HOST", "localhost"),
            port=int(os.getenv("REDIS_PORT", "6379")),
            db=int(os.getenv("REDIS_DB", "0")),
            decode_responses=True,
            socket_connect_timeout=5,
            retry_on_timeout=True,
        )
        self._heartbeat_thread = None
        self._heartbeat_running = False
    
    # ==========================================
    # HARD KILL — Global Emergency Stop
    # ==========================================
    
    def activate_hard_kill(self, actor: str, reason: str = ""):
        """
        Activate the global kill switch.
        All agent operations will be blocked immediately.
        """
        pipe = self.redis.pipeline()
        pipe.set(self.GLOBAL_KILL_KEY, "1")
        pipe.set(self.KILL_REASON_KEY, reason)
        pipe.set(self.KILL_ACTOR_KEY, actor)
        pipe.set(self.KILL_TIMESTAMP_KEY, str(time.time()))
        pipe.publish("killswitch:events", f"hard_kill|{actor}|{reason}")
        pipe.execute()
        
        logger.critical(
            f"HARD KILL ACTIVATED by {actor}: {reason}",
            extra={"actor": actor, "reason": reason},
        )
    
    def deactivate_hard_kill(self, actor: str, reason: str = ""):
        """Deactivate the global kill switch (revive the agent)."""
        pipe = self.redis.pipeline()
        pipe.delete(self.GLOBAL_KILL_KEY)
        pipe.delete(self.KILL_REASON_KEY)
        pipe.delete(self.KILL_ACTOR_KEY)
        pipe.delete(self.KILL_TIMESTAMP_KEY)
        pipe.publish("killswitch:events", f"revive|{actor}|{reason}")
        pipe.execute()
        
        logger.info(
            f"Kill switch DEACTIVATED by {actor}: {reason}",
            extra={"actor": actor, "reason": reason},
        )
    
    def is_killed(self) -> bool:
        """Check if the global kill switch is active."""
        return self.redis.get(self.GLOBAL_KILL_KEY) == "1"
    
    # ==========================================
    # SOFT KILL — Graceful Shutdown
    # ==========================================
    
    def activate_soft_kill(self, actor: str, reason: str = ""):
        """
        Activate soft kill — agent finishes current task but accepts no new ones.
        """
        self.redis.set(self.SOFT_KILL_KEY, "1")
        self.redis.publish("killswitch:events", f"soft_kill|{actor}|{reason}")
        logger.warning(f"SOFT KILL activated by {actor}: {reason}")
    
    def deactivate_soft_kill(self, actor: str):
        """Deactivate soft kill."""
        self.redis.delete(self.SOFT_KILL_KEY)
        logger.info(f"Soft kill deactivated by {actor}")
    
    def is_soft_killed(self) -> bool:
        """Check if soft kill is active."""
        return self.redis.get(self.SOFT_KILL_KEY) == "1"
    
    # ==========================================
    # CAPABILITY KILL — Disable Specific Tools
    # ==========================================
    
    def disable_capability(self, tool_name: str, actor: str, reason: str = ""):
        """Disable a specific tool capability."""
        self.redis.hset(self.CAPABILITY_HASH_KEY, tool_name, "disabled")
        self.redis.publish(
            "killswitch:events",
            f"capability_disabled|{actor}|{tool_name}|{reason}",
        )
        logger.warning(
            f"Capability '{tool_name}' DISABLED by {actor}: {reason}",
            extra={"tool_name": tool_name, "actor": actor},
        )
    
    def enable_capability(self, tool_name: str, actor: str):
        """Re-enable a specific tool capability."""
        self.redis.hset(self.CAPABILITY_HASH_KEY, tool_name, "enabled")
        self.redis.publish(
            "killswitch:events",
            f"capability_enabled|{actor}|{tool_name}",
        )
        logger.info(f"Capability '{tool_name}' ENABLED by {actor}")
    
    def is_capability_disabled(self, tool_name: str) -> bool:
        """Check if a specific tool is disabled."""
        return self.redis.hget(self.CAPABILITY_HASH_KEY, tool_name) == "disabled"
    
    def get_all_capabilities(self) -> dict:
        """Get the status of all tool capabilities."""
        return self.redis.hgetall(self.CAPABILITY_HASH_KEY)
    
    # ==========================================
    # HEARTBEAT — Agent Liveness Check
    # ==========================================
    
    def send_heartbeat(self, agent_id: str, ttl: int = 30):
        """
        Send a heartbeat signal.
        The key auto-expires after `ttl` seconds.
        If the agent dies, the key expires, and the watchdog detects it.
        """
        key = f"{self.HEARTBEAT_KEY_PREFIX}{agent_id}"
        self.redis.setex(key, ttl, "alive")
    
    def is_agent_alive(self, agent_id: str) -> bool:
        """Check if the agent's heartbeat is still active."""
        key = f"{self.HEARTBEAT_KEY_PREFIX}{agent_id}"
        return self.redis.exists(key) == 1
    
    def start_heartbeat(self, agent_id: str, interval: int = 10, ttl: int = 30):
        """Start a background thread that sends heartbeats periodically."""
        self._heartbeat_running = True
        
        def _heartbeat_loop():
            while self._heartbeat_running:
                self.send_heartbeat(agent_id, ttl)
                time.sleep(interval)
        
        self._heartbeat_thread = threading.Thread(
            target=_heartbeat_loop, daemon=True
        )
        self._heartbeat_thread.start()
        logger.info(f"Heartbeat started for agent {agent_id}")
    
    def stop_heartbeat(self):
        """Stop the heartbeat thread."""
        self._heartbeat_running = False
        if self._heartbeat_thread:
            self._heartbeat_thread.join(timeout=5)
    
    # ==========================================
    # STATUS — Aggregated State
    # ==========================================
    
    def get_full_status(self) -> dict:
        """Get the complete kill switch status as a single dict."""
        return {
            "global_kill_active": self.is_killed(),
            "soft_kill_active": self.is_soft_killed(),
            "capabilities": self.get_all_capabilities(),
            "kill_reason": self.redis.get(self.KILL_REASON_KEY),
            "kill_actor": self.redis.get(self.KILL_ACTOR_KEY),
            "kill_timestamp": self.redis.get(self.KILL_TIMESTAMP_KEY),
        }
```

---

### 6.5 Audit Logger

**File: `kill_switch/audit.py`**

The audit log is append-only and immutable — critical for post-incident analysis and regulatory compliance.

```python
"""
Append-only Audit Logger backed by PostgreSQL.

Every kill switch event, tool call approval, and guard decision
is recorded here. Records are immutable — no UPDATE or DELETE
operations are permitted on the audit table.

This is essential for:
- Post-incident forensics ("what happened before the kill switch?")
- Compliance (EU AI Act requires human oversight records)
- Debugging agent behaviour
"""

import os
import time
import json
import uuid
import psycopg2
from psycopg2.extras import RealDictCursor
from agent.observability.logger import get_logger

logger = get_logger(__name__)


class AuditLogger:
    """Append-only audit log backed by PostgreSQL."""
    
    def __init__(self):
        self.conn_string = os.getenv("DATABASE_URL")
    
    def _get_connection(self):
        return psycopg2.connect(self.conn_string)
    
    def log(
        self,
        action: str,
        tool_name: str = None,
        tool_args: dict = None,
        reason: str = "",
        actor: str = "system",
    ):
        """
        Write an immutable audit record.
        
        Args:
            action: Event type (e.g. 'hard_kill_blocked', 'tool_approved',
                    'capability_disabled', 'watchdog_kill')
            tool_name: Name of the tool involved (if applicable).
            tool_args: Arguments passed to the tool (if applicable).
            reason: Human-readable reason for the event.
            actor: Who triggered the event ('system', 'operator', agent_id).
        """
        record = {
            "id": str(uuid.uuid4()),
            "timestamp": time.time(),
            "action": action,
            "tool_name": tool_name,
            "tool_args": json.dumps(tool_args) if tool_args else None,
            "reason": reason,
            "actor": actor,
        }
        
        try:
            conn = self._get_connection()
            cursor = conn.cursor()
            
            cursor.execute(
                """
                INSERT INTO audit_log (id, timestamp, action, tool_name, tool_args, reason, actor)
                VALUES (%(id)s, to_timestamp(%(timestamp)s), %(action)s, %(tool_name)s,
                        %(tool_args)s, %(reason)s, %(actor)s)
                """,
                record,
            )
            conn.commit()
            conn.close()
            
            logger.info(
                f"Audit: {action} | {tool_name} | {reason}",
                extra=record,
            )
        
        except Exception as e:
            # Audit logging failure is a critical error — it must never fail silently
            logger.critical(f"AUDIT LOG FAILURE: {str(e)}", extra=record)
            raise
    
    def query(
        self,
        action: str = None,
        tool_name: str = None,
        since: float = None,
        limit: int = 100,
    ) -> list[dict]:
        """
        Query audit log records.
        
        Args:
            action: Filter by action type.
            tool_name: Filter by tool name.
            since: Only records after this Unix timestamp.
            limit: Maximum number of records to return.
        
        Returns:
            List of audit records as dicts.
        """
        conn = self._get_connection()
        cursor = conn.cursor(cursor_factory=RealDictCursor)
        
        query = "SELECT * FROM audit_log WHERE 1=1"
        params = []
        
        if action:
            query += " AND action = %s"
            params.append(action)
        if tool_name:
            query += " AND tool_name = %s"
            params.append(tool_name)
        if since:
            query += " AND timestamp >= to_timestamp(%s)"
            params.append(since)
        
        query += " ORDER BY timestamp DESC LIMIT %s"
        params.append(limit)
        
        cursor.execute(query, params)
        results = cursor.fetchall()
        conn.close()
        
        return [dict(r) for r in results]
```

**File: `kill_switch/models.py`**

```python
"""Shared Pydantic models used across all services."""

from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum


class KillLevel(str, Enum):
    HARD = "hard"
    SOFT = "soft"
    CAPABILITY = "capability"


class KillSwitchStatus(BaseModel):
    global_kill_active: bool
    soft_kill_active: bool
    capabilities: dict[str, str]  # tool_name -> "enabled" | "disabled"
    kill_reason: Optional[str] = None
    kill_actor: Optional[str] = None
    kill_timestamp: Optional[str] = None


class KillRequest(BaseModel):
    reason: str = Field(..., description="Why the kill switch is being activated")
    actor: str = Field(..., description="Who is activating the kill switch")


class CapabilityRequest(BaseModel):
    tool_name: str
    action: str = Field(..., pattern="^(enable|disable)$")
    reason: str = ""
    actor: str


class AuditRecord(BaseModel):
    id: str
    timestamp: float
    action: str
    tool_name: Optional[str]
    tool_args: Optional[str]
    reason: str
    actor: str


class AgentTask(BaseModel):
    task_id: str
    task: str
    status: str = "pending"
```

---

### 6.6 Watchdog Service

**File: `kill_switch/watchdog.py`**

The watchdog is a **dead man's switch** — if the agent stops sending heartbeats, the watchdog assumes something is wrong and auto-kills.

```python
"""
Watchdog Service — Dead Man's Switch.

Runs as a SEPARATE process from the agent. Monitors the agent's
heartbeat in Redis. If the agent fails to send a heartbeat within
the timeout window, the watchdog:
1. Activates the hard kill switch
2. Logs the event to the audit log
3. Publishes an alert event

Why a separate process?
If the agent crashes or enters an infinite loop, it can't kill itself.
The watchdog is the external safety net.
"""

import os
import time
import threading
from kill_switch.state_manager import KillSwitchStateManager
from kill_switch.audit import AuditLogger
from agent.observability.logger import get_logger
from agent.observability.metrics import watchdog_alerts_total

logger = get_logger(__name__)


class WatchdogService:
    """
    Monitors agent heartbeat and auto-kills if the agent goes silent.
    """
    
    def __init__(
        self,
        agent_id: str,
        check_interval: int = 10,  # Check every 10 seconds
        grace_period: int = 60,    # Allow 60 seconds without heartbeat before kill
    ):
        self.agent_id = agent_id
        self.check_interval = check_interval
        self.grace_period = grace_period
        self.state_manager = KillSwitchStateManager()
        self.audit = AuditLogger()
        self._running = False
        self._consecutive_failures = 0
        self._max_failures = grace_period // check_interval
    
    def start(self):
        """Start the watchdog monitoring loop."""
        self._running = True
        logger.info(
            f"Watchdog started for agent {self.agent_id} "
            f"(check every {self.check_interval}s, "
            f"grace period {self.grace_period}s)"
        )
        
        thread = threading.Thread(target=self._monitor_loop, daemon=True)
        thread.start()
        return thread
    
    def stop(self):
        """Stop the watchdog."""
        self._running = False
        logger.info("Watchdog stopped")
    
    def _monitor_loop(self):
        """Main monitoring loop — runs in a background thread."""
        while self._running:
            try:
                is_alive = self.state_manager.is_agent_alive(self.agent_id)
                
                if is_alive:
                    self._consecutive_failures = 0
                    logger.debug(f"Agent {self.agent_id} heartbeat: OK")
                else:
                    self._consecutive_failures += 1
                    logger.warning(
                        f"Agent {self.agent_id} heartbeat MISSED "
                        f"({self._consecutive_failures}/{self._max_failures})"
                    )
                    
                    if self._consecutive_failures >= self._max_failures:
                        self._trigger_auto_kill()
                
            except Exception as e:
                logger.error(f"Watchdog error: {str(e)}")
            
            time.sleep(self.check_interval)
    
    def _trigger_auto_kill(self):
        """Trigger an automatic hard kill due to lost heartbeat."""
        logger.critical(
            f"WATCHDOG AUTO-KILL: Agent {self.agent_id} "
            f"heartbeat lost for {self.grace_period}s"
        )
        
        self.state_manager.activate_hard_kill(
            actor="watchdog",
            reason=f"Agent heartbeat lost for {self.grace_period} seconds",
        )
        
        self.audit.log(
            action="watchdog_auto_kill",
            reason=f"Heartbeat timeout ({self.grace_period}s)",
            actor="watchdog",
        )
        
        watchdog_alerts_total.inc()
        
        # Reset counter to prevent repeated kills
        self._consecutive_failures = 0


# Entry point for running the watchdog as a standalone service
if __name__ == "__main__":
    agent_id = os.getenv("AGENT_ID", "operator-bot-1")
    watchdog = WatchdogService(agent_id=agent_id)
    
    thread = watchdog.start()
    
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        watchdog.stop()
```

**File: `kill_switch/events.py`**

```python
"""
Redis Pub/Sub Event System for real-time kill switch notifications.

The Control Plane API subscribes to these events and forwards them
to the frontend dashboard via WebSocket.
"""

import os
import threading
import redis
from agent.observability.logger import get_logger

logger = get_logger(__name__)


KILL_SWITCH_CHANNEL = "killswitch:events"


class KillSwitchEventListener:
    """Listens to kill switch events via Redis Pub/Sub."""
    
    def __init__(self, callback=None):
        self.redis = redis.Redis(
            host=os.getenv("REDIS_HOST", "localhost"),
            port=int(os.getenv("REDIS_PORT", "6379")),
            decode_responses=True,
        )
        self.pubsub = self.redis.pubsub()
        self.callback = callback or self._default_handler
        self._running = False
    
    def start(self):
        """Start listening for events in a background thread."""
        self.pubsub.subscribe(KILL_SWITCH_CHANNEL)
        self._running = True
        
        thread = threading.Thread(target=self._listen_loop, daemon=True)
        thread.start()
        logger.info("Kill switch event listener started")
        return thread
    
    def stop(self):
        self._running = False
        self.pubsub.unsubscribe()
    
    def _listen_loop(self):
        while self._running:
            message = self.pubsub.get_message(timeout=1)
            if message and message["type"] == "message":
                event_data = message["data"]
                logger.info(f"Kill switch event: {event_data}")
                self.callback(event_data)
    
    @staticmethod
    def _default_handler(event_data: str):
        logger.info(f"Received kill switch event: {event_data}")
```

---

### 6.7 Control Plane API (FastAPI)

**File: `control_plane/main.py`**

```python
"""
Control Plane API — FastAPI application.

This is the HTTP interface operators use to manage the kill switch.
It also serves WebSocket connections for real-time dashboard updates.
"""

import os
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager

from control_plane.routers import kill_switch, agent, audit, websocket
from kill_switch.events import KillSwitchEventListener
from agent.observability.logger import get_logger

logger = get_logger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown events."""
    # Start event listener
    listener = KillSwitchEventListener(callback=websocket.broadcast_event)
    listener.start()
    logger.info("Control Plane API started")
    
    yield
    
    listener.stop()
    logger.info("Control Plane API stopped")


app = FastAPI(
    title="Kill Switch Control Plane",
    description="API for managing the AI Agent Kill Switch",
    version="1.0.0",
    lifespan=lifespan,
)

# CORS — allow the React dashboard to connect
app.add_middleware(
    CORSMiddleware,
    allow_origins=[os.getenv("DASHBOARD_URL", "http://localhost:5173")],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Mount routers
app.include_router(kill_switch.router, prefix="/api/kill-switch", tags=["Kill Switch"])
app.include_router(agent.router, prefix="/api/agent", tags=["Agent"])
app.include_router(audit.router, prefix="/api/audit", tags=["Audit"])
app.include_router(websocket.router, prefix="/ws", tags=["WebSocket"])


@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

**File: `control_plane/routers/kill_switch.py`**

```python
"""Kill Switch API endpoints — the operator's control interface."""

from fastapi import APIRouter, HTTPException, Depends
from kill_switch.state_manager import KillSwitchStateManager
from kill_switch.audit import AuditLogger
from kill_switch.models import KillRequest, CapabilityRequest, KillSwitchStatus
from control_plane.middleware.auth import verify_operator

router = APIRouter()
state_manager = KillSwitchStateManager()
audit_logger = AuditLogger()


@router.get("/status", response_model=KillSwitchStatus)
async def get_status():
    """Get the current kill switch status (all levels)."""
    return state_manager.get_full_status()


@router.post("/kill")
async def activate_hard_kill(
    request: KillRequest,
    _=Depends(verify_operator),
):
    """
    HARD KILL — Immediately halt all agent operations.
    
    This is the emergency stop button. Once activated:
    - All tool calls are blocked immediately
    - The agent cannot execute any actions
    - Only a POST to /revive can restart the agent
    """
    state_manager.activate_hard_kill(
        actor=request.actor,
        reason=request.reason,
    )
    
    audit_logger.log(
        action="hard_kill_activated",
        reason=request.reason,
        actor=request.actor,
    )
    
    return {"status": "killed", "message": "Agent has been hard-killed."}


@router.post("/revive")
async def revive_agent(
    request: KillRequest,
    _=Depends(verify_operator),
):
    """Deactivate the kill switch and allow the agent to resume."""
    state_manager.deactivate_hard_kill(
        actor=request.actor,
        reason=request.reason,
    )
    
    audit_logger.log(
        action="kill_switch_deactivated",
        reason=request.reason,
        actor=request.actor,
    )
    
    return {"status": "active", "message": "Agent has been revived."}


@router.post("/soft-kill")
async def activate_soft_kill(
    request: KillRequest,
    _=Depends(verify_operator),
):
    """
    SOFT KILL — Graceful shutdown.
    Agent finishes current task but accepts no new ones.
    """
    state_manager.activate_soft_kill(
        actor=request.actor,
        reason=request.reason,
    )
    return {"status": "soft_killed", "message": "Agent will shut down after current task."}


@router.post("/capability")
async def toggle_capability(
    request: CapabilityRequest,
    _=Depends(verify_operator),
):
    """
    CAPABILITY KILL — Enable or disable a specific tool.
    
    This allows fine-grained control: disable DB writes while
    keeping file operations active, for example.
    """
    if request.action == "disable":
        state_manager.disable_capability(
            tool_name=request.tool_name,
            actor=request.actor,
            reason=request.reason,
        )
    else:
        state_manager.enable_capability(
            tool_name=request.tool_name,
            actor=request.actor,
        )
    
    audit_logger.log(
        action=f"capability_{request.action}d",
        tool_name=request.tool_name,
        reason=request.reason,
        actor=request.actor,
    )
    
    return {
        "tool": request.tool_name,
        "status": request.action + "d",
    }
```

**File: `control_plane/routers/websocket.py`**

```python
"""WebSocket endpoint for real-time kill switch event streaming."""

import asyncio
from fastapi import APIRouter, WebSocket, WebSocketDisconnect
from agent.observability.logger import get_logger

logger = get_logger(__name__)
router = APIRouter()

# Connected WebSocket clients
connected_clients: list[WebSocket] = []


@router.websocket("/events")
async def websocket_endpoint(websocket: WebSocket):
    """
    WebSocket endpoint that streams kill switch events in real-time.
    The React dashboard connects here to get live updates.
    """
    await websocket.accept()
    connected_clients.append(websocket)
    logger.info(f"WebSocket client connected. Total: {len(connected_clients)}")
    
    try:
        while True:
            # Keep connection alive; actual events are pushed via broadcast_event()
            await websocket.receive_text()
    except WebSocketDisconnect:
        connected_clients.remove(websocket)
        logger.info(f"WebSocket client disconnected. Total: {len(connected_clients)}")


async def broadcast_event(event_data: str):
    """Broadcast a kill switch event to all connected WebSocket clients."""
    disconnected = []
    
    for client in connected_clients:
        try:
            await client.send_text(event_data)
        except Exception:
            disconnected.append(client)
    
    for client in disconnected:
        connected_clients.remove(client)
```

**File: `control_plane/routers/audit.py`**

```python
"""Audit log query endpoints."""

from fastapi import APIRouter, Query
from kill_switch.audit import AuditLogger
from typing import Optional

router = APIRouter()
audit_logger = AuditLogger()


@router.get("/logs")
async def get_audit_logs(
    action: Optional[str] = Query(None, description="Filter by action type"),
    tool_name: Optional[str] = Query(None, description="Filter by tool name"),
    since: Optional[float] = Query(None, description="Unix timestamp — records after this time"),
    limit: int = Query(100, le=1000, description="Max records to return"),
):
    """Query the immutable audit log."""
    records = audit_logger.query(
        action=action,
        tool_name=tool_name,
        since=since,
        limit=limit,
    )
    return {"records": records, "count": len(records)}
```

**File: `control_plane/middleware/auth.py`**

```python
"""API authentication middleware."""

import os
from fastapi import HTTPException, Security
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)


async def verify_operator(api_key: str = Security(api_key_header)):
    """
    Verify the operator's API key.
    
    In production, replace this with JWT/OAuth2 authentication.
    For this project, a simple API key is sufficient.
    """
    expected_key = os.getenv("OPERATOR_API_KEY")
    
    if not expected_key:
        raise HTTPException(status_code=500, detail="OPERATOR_API_KEY not configured")
    
    if api_key != expected_key:
        raise HTTPException(status_code=403, detail="Invalid API key")
    
    return True
```

---

### 6.8 Frontend Dashboard (React + Tailwind)

The dashboard is the operator's primary interface for managing the kill switch.

**File: `dashboard/package.json`**

```json
{
  "name": "killswitch-dashboard",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "react-router-dom": "^6.23.0",
    "axios": "^1.7.0",
    "recharts": "^2.12.0",
    "lucide-react": "^0.378.0",
    "date-fns": "^3.6.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.38",
    "tailwindcss": "^3.4.3",
    "typescript": "^5.4.5",
    "vite": "^5.2.0"
  }
}
```

**File: `dashboard/src/types/index.ts`**

```typescript
// Kill Switch Types

export interface KillSwitchStatus {
  global_kill_active: boolean;
  soft_kill_active: boolean;
  capabilities: Record<string, "enabled" | "disabled">;
  kill_reason: string | null;
  kill_actor: string | null;
  kill_timestamp: string | null;
}

export interface AuditRecord {
  id: string;
  timestamp: number;
  action: string;
  tool_name: string | null;
  tool_args: string | null;
  reason: string;
  actor: string;
}

export interface MetricDataPoint {
  time: string;
  tool_calls: number;
  errors: number;
  blocked: number;
}
```

**File: `dashboard/src/api/client.ts`**

```typescript
/**
 * API client for the Kill Switch Control Plane.
 * Handles REST calls and WebSocket connection.
 */

import axios from "axios";
import type { KillSwitchStatus, AuditRecord } from "../types";

const API_BASE = import.meta.env.VITE_API_URL || "http://localhost:8000";
const WS_BASE = import.meta.env.VITE_WS_URL || "ws://localhost:8000";
const API_KEY = import.meta.env.VITE_API_KEY || "dev-operator-key";

const api = axios.create({
  baseURL: `${API_BASE}/api`,
  headers: { "X-API-Key": API_KEY },
});

// --- Kill Switch ---

export async function getStatus(): Promise<KillSwitchStatus> {
  const { data } = await api.get("/kill-switch/status");
  return data;
}

export async function activateHardKill(reason: string, actor: string) {
  return api.post("/kill-switch/kill", { reason, actor });
}

export async function reviveAgent(reason: string, actor: string) {
  return api.post("/kill-switch/revive", { reason, actor });
}

export async function activateSoftKill(reason: string, actor: string) {
  return api.post("/kill-switch/soft-kill", { reason, actor });
}

export async function toggleCapability(
  toolName: string,
  action: "enable" | "disable",
  actor: string,
  reason: string = ""
) {
  return api.post("/kill-switch/capability", {
    tool_name: toolName,
    action,
    actor,
    reason,
  });
}

// --- Audit ---

export async function getAuditLogs(
  limit: number = 100
): Promise<AuditRecord[]> {
  const { data } = await api.get("/audit/logs", { params: { limit } });
  return data.records;
}

// --- WebSocket ---

export function connectWebSocket(
  onMessage: (event: string) => void
): WebSocket {
  const ws = new WebSocket(`${WS_BASE}/ws/events`);

  ws.onopen = () => console.log("WebSocket connected");
  ws.onmessage = (event) => onMessage(event.data);
  ws.onerror = (error) => console.error("WebSocket error:", error);
  ws.onclose = () => {
    console.log("WebSocket disconnected. Reconnecting in 3s...");
    setTimeout(() => connectWebSocket(onMessage), 3000);
  };

  return ws;
}
```

**File: `dashboard/src/components/KillSwitchPanel.tsx`**

```tsx
/**
 * KillSwitchPanel — The Big Red Button.
 *
 * Displays the current kill switch status and provides
 * one-click activation/deactivation for the operator.
 */

import React, { useState } from "react";
import { ShieldAlert, ShieldCheck, AlertTriangle } from "lucide-react";
import type { KillSwitchStatus } from "../types";
import { activateHardKill, reviveAgent, activateSoftKill } from "../api/client";

interface Props {
  status: KillSwitchStatus;
  onStatusChange: () => void;
}

export default function KillSwitchPanel({ status, onStatusChange }: Props) {
  const [reason, setReason] = useState("");
  const [loading, setLoading] = useState(false);

  const handleHardKill = async () => {
    if (!reason.trim()) {
      alert("Please provide a reason for the kill switch activation.");
      return;
    }
    setLoading(true);
    try {
      await activateHardKill(reason, "operator");
      onStatusChange();
      setReason("");
    } finally {
      setLoading(false);
    }
  };

  const handleRevive = async () => {
    setLoading(true);
    try {
      await reviveAgent("Operator revived agent", "operator");
      onStatusChange();
    } finally {
      setLoading(false);
    }
  };

  const handleSoftKill = async () => {
    setLoading(true);
    try {
      await activateSoftKill(reason || "Graceful shutdown", "operator");
      onStatusChange();
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="bg-white rounded-xl shadow-lg p-6 border-2 border-gray-100">
      {/* Status Indicator */}
      <div className="flex items-center justify-between mb-6">
        <h2 className="text-xl font-bold text-gray-800">Kill Switch Control</h2>
        <div
          className={`flex items-center gap-2 px-4 py-2 rounded-full text-sm font-medium ${
            status.global_kill_active
              ? "bg-red-100 text-red-700"
              : status.soft_kill_active
              ? "bg-yellow-100 text-yellow-700"
              : "bg-green-100 text-green-700"
          }`}
        >
          {status.global_kill_active ? (
            <><ShieldAlert size={16} /> KILLED</>
          ) : status.soft_kill_active ? (
            <><AlertTriangle size={16} /> SOFT KILL</>
          ) : (
            <><ShieldCheck size={16} /> ACTIVE</>
          )}
        </div>
      </div>

      {/* Kill Reason (shown when active) */}
      {status.global_kill_active && status.kill_reason && (
        <div className="mb-4 p-3 bg-red-50 rounded-lg text-sm text-red-700">
          <strong>Reason:</strong> {status.kill_reason}
          <br />
          <strong>Killed by:</strong> {status.kill_actor} at{" "}
          {new Date(parseFloat(status.kill_timestamp!) * 1000).toLocaleString()}
        </div>
      )}

      {/* Reason Input */}
      <input
        type="text"
        placeholder="Reason for action (required for kill)..."
        value={reason}
        onChange={(e) => setReason(e.target.value)}
        className="w-full px-4 py-2 border border-gray-300 rounded-lg mb-4 focus:ring-2 focus:ring-red-500 focus:border-transparent"
      />

      {/* Action Buttons */}
      <div className="flex gap-3">
        {!status.global_kill_active ? (
          <>
            <button
              onClick={handleHardKill}
              disabled={loading}
              className="flex-1 bg-red-600 hover:bg-red-700 text-white font-bold py-3 px-6 rounded-lg transition-colors disabled:opacity-50 shadow-md"
            >
              HARD KILL
            </button>
            <button
              onClick={handleSoftKill}
              disabled={loading}
              className="flex-1 bg-yellow-500 hover:bg-yellow-600 text-white font-bold py-3 px-6 rounded-lg transition-colors disabled:opacity-50"
            >
              SOFT KILL
            </button>
          </>
        ) : (
          <button
            onClick={handleRevive}
            disabled={loading}
            className="flex-1 bg-green-600 hover:bg-green-700 text-white font-bold py-3 px-6 rounded-lg transition-colors disabled:opacity-50 shadow-md"
          >
            REVIVE AGENT
          </button>
        )}
      </div>
    </div>
  );
}
```

**File: `dashboard/src/components/CapabilityToggles.tsx`**

```tsx
/**
 * CapabilityToggles — Per-tool enable/disable controls.
 *
 * Shows a toggle switch for each tool. When a tool is disabled,
 * the agent can still run but cannot use that specific capability.
 */

import React from "react";
import { FileText, Database, Globe, Terminal } from "lucide-react";
import type { KillSwitchStatus } from "../types";
import { toggleCapability } from "../api/client";

const TOOL_CONFIG = [
  { name: "file_manager", label: "File Manager", icon: FileText, color: "blue" },
  { name: "db_writer", label: "DB Writer", icon: Database, color: "purple" },
  { name: "api_caller", label: "API Caller", icon: Globe, color: "green" },
  { name: "code_executor", label: "Code Executor", icon: Terminal, color: "orange" },
];

interface Props {
  status: KillSwitchStatus;
  onStatusChange: () => void;
}

export default function CapabilityToggles({ status, onStatusChange }: Props) {
  const handleToggle = async (toolName: string, currentlyDisabled: boolean) => {
    const action = currentlyDisabled ? "enable" : "disable";
    await toggleCapability(toolName, action, "operator", `Operator ${action}d ${toolName}`);
    onStatusChange();
  };

  return (
    <div className="bg-white rounded-xl shadow-lg p-6 border-2 border-gray-100">
      <h2 className="text-xl font-bold text-gray-800 mb-4">Tool Capabilities</h2>
      <div className="space-y-3">
        {TOOL_CONFIG.map(({ name, label, icon: Icon, color }) => {
          const isDisabled = status.capabilities[name] === "disabled";
          return (
            <div
              key={name}
              className={`flex items-center justify-between p-3 rounded-lg border ${
                isDisabled ? "bg-red-50 border-red-200" : "bg-gray-50 border-gray-200"
              }`}
            >
              <div className="flex items-center gap-3">
                <Icon size={20} className={`text-${color}-500`} />
                <span className="font-medium text-gray-700">{label}</span>
              </div>
              <button
                onClick={() => handleToggle(name, isDisabled)}
                className={`relative inline-flex h-6 w-11 items-center rounded-full transition-colors ${
                  isDisabled ? "bg-red-400" : "bg-green-500"
                }`}
              >
                <span
                  className={`inline-block h-4 w-4 transform rounded-full bg-white transition-transform ${
                    isDisabled ? "translate-x-1" : "translate-x-6"
                  }`}
                />
              </button>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

**File: `dashboard/src/components/AuditLog.tsx`**

```tsx
/**
 * AuditLog — Real-time feed of all kill switch events.
 *
 * Shows a scrollable list of audit records, colour-coded by severity.
 * New records from WebSocket are prepended at the top.
 */

import React from "react";
import { formatDistanceToNow } from "date-fns";
import type { AuditRecord } from "../types";

const ACTION_STYLES: Record<string, string> = {
  hard_kill_activated: "bg-red-100 text-red-800",
  hard_kill_blocked: "bg-red-100 text-red-800",
  capability_disabled: "bg-yellow-100 text-yellow-800",
  capability_enabled: "bg-green-100 text-green-800",
  tool_approved: "bg-blue-100 text-blue-800",
  guardrail_blocked: "bg-orange-100 text-orange-800",
  watchdog_auto_kill: "bg-red-100 text-red-800",
  kill_switch_deactivated: "bg-green-100 text-green-800",
};

interface Props {
  records: AuditRecord[];
}

export default function AuditLog({ records }: Props) {
  return (
    <div className="bg-white rounded-xl shadow-lg p-6 border-2 border-gray-100">
      <h2 className="text-xl font-bold text-gray-800 mb-4">Audit Log</h2>
      <div className="space-y-2 max-h-96 overflow-y-auto">
        {records.length === 0 ? (
          <p className="text-gray-400 text-center py-8">No events recorded yet.</p>
        ) : (
          records.map((record) => (
            <div
              key={record.id}
              className="flex items-start gap-3 p-3 rounded-lg bg-gray-50 hover:bg-gray-100 transition-colors"
            >
              <span
                className={`px-2 py-1 text-xs font-medium rounded ${
                  ACTION_STYLES[record.action] || "bg-gray-100 text-gray-600"
                }`}
              >
                {record.action}
              </span>
              <div className="flex-1 min-w-0">
                <p className="text-sm text-gray-700 truncate">
                  {record.reason || "No reason provided"}
                </p>
                <p className="text-xs text-gray-400 mt-1">
                  {record.tool_name && <span className="font-mono">{record.tool_name}</span>}
                  {record.tool_name && " | "}
                  {record.actor} |{" "}
                  {formatDistanceToNow(new Date(record.timestamp * 1000), { addSuffix: true })}
                </p>
              </div>
            </div>
          ))
        )}
      </div>
    </div>
  );
}
```

**File: `dashboard/src/components/MetricsChart.tsx`**

```tsx
/**
 * MetricsChart — Real-time tool call rate and error rate.
 *
 * Uses Recharts to render a live-updating line chart.
 * Data is fetched from the control plane API (which proxies Prometheus).
 */

import React from "react";
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
  ResponsiveContainer,
} from "recharts";
import type { MetricDataPoint } from "../types";

interface Props {
  data: MetricDataPoint[];
}

export default function MetricsChart({ data }: Props) {
  return (
    <div className="bg-white rounded-xl shadow-lg p-6 border-2 border-gray-100">
      <h2 className="text-xl font-bold text-gray-800 mb-4">Agent Activity</h2>
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" stroke="#f0f0f0" />
          <XAxis dataKey="time" tick={{ fontSize: 12 }} />
          <YAxis tick={{ fontSize: 12 }} />
          <Tooltip />
          <Legend />
          <Line
            type="monotone"
            dataKey="tool_calls"
            stroke="#3b82f6"
            strokeWidth={2}
            name="Tool Calls"
            dot={false}
          />
          <Line
            type="monotone"
            dataKey="errors"
            stroke="#ef4444"
            strokeWidth={2}
            name="Errors"
            dot={false}
          />
          <Line
            type="monotone"
            dataKey="blocked"
            stroke="#f59e0b"
            strokeWidth={2}
            name="Blocked"
            dot={false}
          />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

**File: `dashboard/src/pages/Dashboard.tsx`**

```tsx
/**
 * Dashboard — Main operator dashboard page.
 *
 * Assembles all components into a single operator view:
 * - Kill Switch control panel (top)
 * - Capability toggles (left)
 * - Metrics chart (right)
 * - Audit log (bottom, full width)
 */

import React, { useState, useEffect, useCallback } from "react";
import KillSwitchPanel from "../components/KillSwitchPanel";
import CapabilityToggles from "../components/CapabilityToggles";
import AuditLog from "../components/AuditLog";
import MetricsChart from "../components/MetricsChart";
import { getStatus, getAuditLogs, connectWebSocket } from "../api/client";
import type { KillSwitchStatus, AuditRecord, MetricDataPoint } from "../types";

export default function Dashboard() {
  const [status, setStatus] = useState<KillSwitchStatus>({
    global_kill_active: false,
    soft_kill_active: false,
    capabilities: {},
    kill_reason: null,
    kill_actor: null,
    kill_timestamp: null,
  });
  const [auditRecords, setAuditRecords] = useState<AuditRecord[]>([]);
  const [metrics, setMetrics] = useState<MetricDataPoint[]>([]);

  const refreshStatus = useCallback(async () => {
    try {
      const newStatus = await getStatus();
      setStatus(newStatus);
    } catch (error) {
      console.error("Failed to fetch status:", error);
    }
  }, []);

  const refreshAuditLog = useCallback(async () => {
    try {
      const records = await getAuditLogs(50);
      setAuditRecords(records);
    } catch (error) {
      console.error("Failed to fetch audit logs:", error);
    }
  }, []);

  useEffect(() => {
    // Initial fetch
    refreshStatus();
    refreshAuditLog();

    // Poll status every 5 seconds
    const statusInterval = setInterval(refreshStatus, 5000);
    const auditInterval = setInterval(refreshAuditLog, 10000);

    // WebSocket for real-time events
    const ws = connectWebSocket((event) => {
      console.log("Kill switch event:", event);
      refreshStatus();
      refreshAuditLog();
    });

    return () => {
      clearInterval(statusInterval);
      clearInterval(auditInterval);
      ws.close();
    };
  }, [refreshStatus, refreshAuditLog]);

  return (
    <div className="min-h-screen bg-gray-50 p-6">
      <header className="mb-8">
        <h1 className="text-3xl font-bold text-gray-900">
          AI Agent Kill Switch Dashboard
        </h1>
        <p className="text-gray-500 mt-1">
          Monitor and control the autonomous agent in real-time
        </p>
      </header>

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
        <KillSwitchPanel status={status} onStatusChange={refreshStatus} />
        <CapabilityToggles status={status} onStatusChange={refreshStatus} />
      </div>

      <div className="mb-6">
        <MetricsChart data={metrics} />
      </div>

      <AuditLog records={auditRecords} />
    </div>
  );
}
```

**File: `dashboard/src/App.tsx`**

```tsx
import React from "react";
import { BrowserRouter, Routes, Route } from "react-router-dom";
import Dashboard from "./pages/Dashboard";

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Dashboard />} />
      </Routes>
    </BrowserRouter>
  );
}
```

**File: `dashboard/src/main.tsx`**

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import "./index.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

**File: `dashboard/src/index.css`**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---

### 6.9 Prometheus Metrics Exporter

**File: `agent/observability/metrics.py`**

```python
"""
Prometheus metrics definitions for the AI agent.

These metrics are exposed on a /metrics HTTP endpoint that
Prometheus scrapes at a configurable interval.
"""

from prometheus_client import (
    Counter,
    Histogram,
    Gauge,
    start_http_server,
)

# --- Tool Call Metrics ---
agent_tool_calls_total = Counter(
    "agent_tool_calls_total",
    "Total number of tool calls made by the agent",
    labelnames=["tool", "status"],  # status: approved, guardrail_blocked, kill_blocked
)

# --- LLM Metrics ---
agent_llm_calls_total = Counter(
    "agent_llm_calls_total",
    "Total number of LLM inference calls",
)

agent_llm_latency_seconds = Histogram(
    "agent_llm_latency_seconds",
    "LLM inference latency in seconds",
    buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

# --- Kill Switch Metrics ---
kill_switch_activations_total = Counter(
    "kill_switch_activations_total",
    "Total number of kill switch activations",
    labelnames=["level"],  # level: hard, soft, capability
)

kill_switch_status = Gauge(
    "kill_switch_status",
    "Current kill switch status (1 = active, 0 = inactive)",
    labelnames=["level"],
)

# --- Error Metrics ---
agent_errors_total = Counter(
    "agent_errors_total",
    "Total number of agent errors",
    labelnames=["error_type"],
)

# --- Watchdog Metrics ---
watchdog_alerts_total = Counter(
    "watchdog_alerts_total",
    "Total number of watchdog auto-kill alerts",
)

# --- Guard Check Metrics ---
guard_check_latency_seconds = Histogram(
    "guard_check_latency_seconds",
    "Latency of kill switch guard checks in seconds",
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1],
)


def start_metrics_server(port: int = 8001):
    """Start the Prometheus metrics HTTP server."""
    start_http_server(port)
    print(f"Prometheus metrics server started on port {port}")
```

**File: `agent/observability/logger.py`**

```python
"""
Structured JSON logger — compatible with Grafana Loki.

All log entries are JSON objects, which Loki can parse and index.
This enables powerful log queries like:
  {service="agent"} | json | action="hard_kill_blocked"
"""

import logging
import json
import sys
from datetime import datetime, timezone


class JsonFormatter(logging.Formatter):
    """Format log records as JSON for Loki ingestion."""
    
    def format(self, record):
        log_entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "service": "killswitch-agent",
            "module": record.module,
            "message": record.getMessage(),
        }
        
        # Merge extra fields (task_id, tool_name, etc.)
        if hasattr(record, "__dict__"):
            for key, value in record.__dict__.items():
                if key not in (
                    "name", "msg", "args", "created", "filename",
                    "funcName", "levelname", "levelno", "lineno",
                    "module", "msecs", "pathname", "process",
                    "processName", "relativeCreated", "stack_info",
                    "thread", "threadName", "exc_info", "exc_text",
                    "message", "taskName",
                ):
                    log_entry[key] = value
        
        return json.dumps(log_entry)


def get_logger(name: str) -> logging.Logger:
    """Get a JSON-formatted logger."""
    logger = logging.getLogger(name)
    
    if not logger.handlers:
        handler = logging.StreamHandler(sys.stdout)
        handler.setFormatter(JsonFormatter())
        logger.addHandler(handler)
        logger.setLevel(logging.INFO)
    
    return logger
```

---

### 6.10 Grafana + Loki + Prometheus Configuration

**File: `observability/prometheus/prometheus.yml`**

```yaml
# Prometheus Configuration
# Scrapes metrics from the agent and control plane every 15 seconds.

global:
  scrape_interval: 15s      # How often to scrape targets
  evaluation_interval: 15s  # How often to evaluate alerting rules

# Alerting rules (optional — uncomment for production)
# rule_files:
#   - "alert_rules.yml"

scrape_configs:
  # Agent metrics (exposed on port 8001)
  - job_name: "agent"
    static_configs:
      - targets: ["agent:8001"]
        labels:
          service: "killswitch-agent"

  # Control Plane API metrics
  - job_name: "control-plane"
    static_configs:
      - targets: ["control-plane:8000"]
        labels:
          service: "control-plane"
```

**File: `observability/grafana/provisioning/datasources/datasources.yml`**

```yaml
# Auto-provision data sources on Grafana startup.
# No manual configuration needed — Grafana connects to Prometheus and Loki automatically.

apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true
```

**File: `observability/loki/loki-config.yml`**

```yaml
# Loki Configuration — lightweight log aggregation

auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h  # 7 days log retention
```

---

### 6.11 LLM Tracing with Langfuse

**File: `agent/observability/tracer.py`**

```python
"""
Langfuse LLM tracing — captures the agent's reasoning chain.

This goes beyond metrics and logs: it records the FULL conversation
between the agent and the LLM, including:
- What prompt was sent
- What the LLM responded
- Which tool the LLM chose and why
- What the tool returned
- How many tokens were used (and estimated cost)

Essential for post-incident analysis: "Why did the agent decide to
delete that file before the kill switch was triggered?"
"""

import os
from langfuse.callback import CallbackHandler


def get_langfuse_handler():
    """
    Create a Langfuse callback handler for LangChain/LangGraph.
    
    Returns None if Langfuse credentials are not configured
    (tracing is optional, not required for the kill switch to work).
    """
    public_key = os.getenv("LANGFUSE_PUBLIC_KEY")
    secret_key = os.getenv("LANGFUSE_SECRET_KEY")
    host = os.getenv("LANGFUSE_HOST", "https://cloud.langfuse.com")
    
    if not public_key or not secret_key:
        return None
    
    return CallbackHandler(
        public_key=public_key,
        secret_key=secret_key,
        host=host,
    )
```

---

### 6.12 Docker Compose (Full Stack)

**File: `docker-compose.yml`**

This single file starts the entire project — all services, databases, and observability tools.

```yaml
version: "3.9"

services:
  # ==========================================
  # DATABASES
  # ==========================================
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: killswitch
      POSTGRES_PASSWORD: killswitch_pass
      POSTGRES_DB: killswitch_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U killswitch"]
      interval: 10s
      timeout: 3s
      retries: 5

  # ==========================================
  # AI AGENT
  # ==========================================
  
  agent:
    build: ./agent
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_HOST=redis
      - DATABASE_URL=postgresql://killswitch:killswitch_pass@postgres:5432/killswitch_db
      - AGENT_DATABASE_URL=postgresql://killswitch:killswitch_pass@postgres:5432/killswitch_db
      - METRICS_PORT=8001
      - LANGFUSE_PUBLIC_KEY=${LANGFUSE_PUBLIC_KEY:-}
      - LANGFUSE_SECRET_KEY=${LANGFUSE_SECRET_KEY:-}
    ports:
      - "8001:8001"  # Prometheus metrics
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    volumes:
      - agent_workspace:/tmp/agent_workspace

  # ==========================================
  # WATCHDOG (separate process)
  # ==========================================
  
  watchdog:
    build: ./agent
    command: python -m kill_switch.watchdog
    environment:
      - REDIS_HOST=redis
      - DATABASE_URL=postgresql://killswitch:killswitch_pass@postgres:5432/killswitch_db
      - AGENT_ID=operator-bot-1
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy

  # ==========================================
  # CONTROL PLANE API
  # ==========================================
  
  control-plane:
    build: ./control_plane
    environment:
      - REDIS_HOST=redis
      - DATABASE_URL=postgresql://killswitch:killswitch_pass@postgres:5432/killswitch_db
      - OPERATOR_API_KEY=${OPERATOR_API_KEY:-dev-operator-key}
      - DASHBOARD_URL=http://localhost:5173
    ports:
      - "8000:8000"
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy

  # ==========================================
  # FRONTEND DASHBOARD
  # ==========================================
  
  dashboard:
    build: ./dashboard
    ports:
      - "5173:5173"
    environment:
      - VITE_API_URL=http://localhost:8000
      - VITE_WS_URL=ws://localhost:8000
      - VITE_API_KEY=${OPERATOR_API_KEY:-dev-operator-key}
    depends_on:
      - control-plane

  # ==========================================
  # OBSERVABILITY
  # ==========================================
  
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./observability/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=7d"

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./observability/loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_AUTH_ANONYMOUS_ENABLED=true
    volumes:
      - ./observability/grafana/provisioning:/etc/grafana/provisioning
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
      - loki

volumes:
  redis_data:
  postgres_data:
  prometheus_data:
  loki_data:
  grafana_data:
  agent_workspace:
```

**File: `database/init.sql`**

```sql
-- Kill Switch Database Schema
-- This runs automatically when PostgreSQL starts for the first time.

-- Audit Log — APPEND-ONLY (no UPDATE, no DELETE allowed)
CREATE TABLE IF NOT EXISTS audit_log (
    id          UUID PRIMARY KEY,
    timestamp   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    action      VARCHAR(100) NOT NULL,
    tool_name   VARCHAR(100),
    tool_args   JSONB,
    reason      TEXT,
    actor       VARCHAR(100) NOT NULL DEFAULT 'system'
);

-- Index for common queries
CREATE INDEX idx_audit_log_timestamp ON audit_log (timestamp DESC);
CREATE INDEX idx_audit_log_action ON audit_log (action);
CREATE INDEX idx_audit_log_tool ON audit_log (tool_name);

-- Agent Task History
CREATE TABLE IF NOT EXISTS agent_tasks (
    id          UUID PRIMARY KEY,
    task        TEXT NOT NULL,
    status      VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    result      TEXT,
    error       TEXT
);

-- Prevent mutations on audit_log (enforce append-only)
CREATE OR REPLACE FUNCTION prevent_audit_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Audit log is append-only. UPDATE and DELETE are not permitted.';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_no_update
    BEFORE UPDATE ON audit_log
    FOR EACH ROW
    EXECUTE FUNCTION prevent_audit_mutation();

CREATE TRIGGER audit_no_delete
    BEFORE DELETE ON audit_log
    FOR EACH ROW
    EXECUTE FUNCTION prevent_audit_mutation();
```

---

## 7. Workflow Diagrams

### 7.1 Normal Agent Execution Flow

```
┌─────────┐     ┌───────────┐     ┌──────────────┐     ┌──────────────┐
│  User    │────►│  Control  │────►│    Agent      │────►│   Tool       │
│  sends   │     │  Plane    │     │  (LangGraph)  │     │  Execution   │
│  task    │     │  API      │     │               │     │              │
└─────────┘     └───────────┘     └──────┬───────┘     └──────────────┘
                                         │                      ▲
                                         │                      │
                                         ▼                      │
                                  ┌──────────────┐              │
                                  │  Kill Switch  │──── OK ─────┘
                                  │  Guard        │
                                  │  (Redis check)│
                                  └──────────────┘
                                         │
                                    BLOCKED? ──► Exception ──► Agent Stops
```

### 7.2 Kill Switch Activation Flow

```
Step 1: Operator clicks "HARD KILL" on React Dashboard
         │
Step 2:  ├──► POST /api/kill-switch/kill (FastAPI)
         │
Step 3:  ├──► Redis: SET killswitch:global "1"
         │    Redis: PUBLISH killswitch:events "hard_kill|operator|reason"
         │
Step 4:  ├──► PostgreSQL: INSERT INTO audit_log (...)
         │
Step 5:  ├──► WebSocket: broadcast "hard_kill" to all clients
         │
Step 6:  └──► Dashboard updates: status indicator turns RED

         ... meanwhile ...

Step 7:  Agent's next tool call hits Kill Switch Guard
         │
Step 8:  ├──► Guard reads Redis: killswitch:global == "1"
         │
Step 9:  ├──► Guard raises HardKillException
         │
Step 10: ├──► Agent loop terminates
         │
Step 11: └──► LangGraph checkpoints state (can resume later)
```

### 7.3 Watchdog Auto-Kill Flow

```
Normal:  Agent sends heartbeat every 10s ──► Redis SETEX (30s TTL)
         Watchdog checks every 10s          ──► Redis key exists? YES ──► OK

Failure: Agent crashes / infinite loop
         │
         ├──► Agent stops sending heartbeats
         │
         ├──► Redis key expires after 30s (TTL)
         │
         ├──► Watchdog check: Redis key exists? NO
         │    Consecutive failures: 1/6
         │
         ├──► ... 60 seconds pass (grace period) ...
         │    Consecutive failures: 6/6
         │
         ├──► Watchdog triggers auto-kill:
         │    ├── Redis: SET killswitch:global "1"
         │    ├── PostgreSQL: audit_log INSERT
         │    └── Redis: PUBLISH watchdog_auto_kill event
         │
         └──► Dashboard shows: "Agent auto-killed by watchdog"
```

### 7.4 Capability Kill Flow

```
Operator: "I want to block DB writes but keep everything else running"
         │
         ├──► POST /api/kill-switch/capability
         │    { tool_name: "db_writer", action: "disable" }
         │
         ├──► Redis: HSET killswitch:capabilities db_writer "disabled"
         │
         ├──► Agent continues running...
         │
         ├──► Agent decides to call db_writer("INSERT INTO ...")
         │
         ├──► Guard checks: is_capability_disabled("db_writer")? YES
         │
         ├──► Guard raises CapabilityDisabledException
         │
         ├──► Agent receives error, tries alternative approach
         │    (or reports to user that DB writes are blocked)
         │
         └──► Other tools (file_manager, api_caller) continue working
```

---

## 8. Environment Variables

**File: `.env.example`**

```bash
# ===========================================
# AI Agent Kill Switch — Environment Variables
# ===========================================

# --- LLM ---
OPENAI_API_KEY=sk-your-api-key-here
OPENAI_MODEL=gpt-4o

# --- Redis ---
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0

# --- PostgreSQL ---
DATABASE_URL=postgresql://killswitch:killswitch_pass@localhost:5432/killswitch_db
AGENT_DATABASE_URL=postgresql://killswitch:killswitch_pass@localhost:5432/killswitch_db

# --- Control Plane ---
OPERATOR_API_KEY=dev-operator-key

# --- Agent ---
AGENT_WORKSPACE=/tmp/agent_workspace
AGENT_ID=operator-bot-1
CODE_EXECUTION_TIMEOUT=30
METRICS_PORT=8001

# --- Dashboard ---
VITE_API_URL=http://localhost:8000
VITE_WS_URL=ws://localhost:8000
VITE_API_KEY=dev-operator-key

# --- Langfuse (optional — LLM tracing) ---
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com

# --- Grafana ---
GF_SECURITY_ADMIN_PASSWORD=admin
```

---

## 9. How to Run the Project

### Prerequisites

- Docker and Docker Compose installed
- An OpenAI API key (or compatible LLM API key)
- Git

### Step 1: Clone and Configure

```bash
git clone https://github.com/your-username/killswitch-agent.git
cd killswitch-agent

# Copy environment template and fill in your API keys
cp .env.example .env
# Edit .env and add your OPENAI_API_KEY
```

### Step 2: Start Everything

```bash
# Start the full stack (agent, API, dashboard, Redis, PostgreSQL,
# Prometheus, Grafana, Loki, watchdog) — all with one command:
docker compose up --build
```

### Step 3: Access the Services

| Service | URL | Purpose |
|---------|-----|---------|
| React Dashboard | http://localhost:5173 | Operator control panel |
| Control Plane API | http://localhost:8000/docs | Swagger API documentation |
| Grafana | http://localhost:3000 | Metrics and log dashboards |
| Prometheus | http://localhost:9090 | Raw metrics query |
| Loki | http://localhost:3100 | Log query API |

### Step 4: Test the Kill Switch

```bash
# Activate the hard kill via CLI (or use the dashboard)
curl -X POST http://localhost:8000/api/kill-switch/kill \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-operator-key" \
  -d '{"reason": "Testing the kill switch", "actor": "operator"}'

# Check status
curl http://localhost:8000/api/kill-switch/status \
  -H "X-API-Key: dev-operator-key"

# Revive the agent
curl -X POST http://localhost:8000/api/kill-switch/revive \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-operator-key" \
  -d '{"reason": "Test complete", "actor": "operator"}'

# Disable a specific tool
curl -X POST http://localhost:8000/api/kill-switch/capability \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-operator-key" \
  -d '{"tool_name": "db_writer", "action": "disable", "actor": "operator", "reason": "Investigating DB issues"}'
```

---

## 10. Testing Strategy

### Unit Tests

**File: `tests/test_kill_switch_guard.py`**

```python
"""Unit tests for the Kill Switch Guard."""

import pytest
from unittest.mock import patch, MagicMock
from agent.guard.kill_switch_guard import (
    KillSwitchGuard,
    HardKillException,
    CapabilityDisabledException,
    GuardrailViolation,
)


@pytest.fixture
def guard():
    with patch("agent.guard.kill_switch_guard.KillSwitchStateManager") as mock_sm:
        with patch("agent.guard.kill_switch_guard.AuditLogger"):
            g = KillSwitchGuard()
            g.state_manager = mock_sm()
            yield g


class TestHardKill:
    def test_blocks_when_global_kill_active(self, guard):
        guard.state_manager.is_killed.return_value = True
        
        with pytest.raises(HardKillException):
            guard.check("file_manager", {"action": "read", "path": "test.txt"})
    
    def test_allows_when_kill_not_active(self, guard):
        guard.state_manager.is_killed.return_value = False
        guard.state_manager.is_capability_disabled.return_value = False
        
        result = guard.check("file_manager", {"action": "read", "path": "test.txt"})
        assert result is True


class TestCapabilityKill:
    def test_blocks_disabled_tool(self, guard):
        guard.state_manager.is_killed.return_value = False
        guard.state_manager.is_capability_disabled.return_value = True
        
        with pytest.raises(CapabilityDisabledException):
            guard.check("db_writer", {"query": "SELECT 1"})
    
    def test_allows_enabled_tool(self, guard):
        guard.state_manager.is_killed.return_value = False
        guard.state_manager.is_capability_disabled.return_value = False
        
        result = guard.check("db_writer", {"query": "SELECT 1"})
        assert result is True


class TestGuardrails:
    def test_blocks_drop_table(self, guard):
        guard.state_manager.is_killed.return_value = False
        guard.state_manager.is_capability_disabled.return_value = False
        
        with pytest.raises(GuardrailViolation):
            guard.check("db_writer", {"query": "DROP TABLE users"})
    
    def test_blocks_delete_without_where(self, guard):
        guard.state_manager.is_killed.return_value = False
        guard.state_manager.is_capability_disabled.return_value = False
        
        with pytest.raises(GuardrailViolation):
            guard.check("db_writer", {"query": "DELETE FROM users"})
    
    def test_allows_safe_select(self, guard):
        guard.state_manager.is_killed.return_value = False
        guard.state_manager.is_capability_disabled.return_value = False
        
        result = guard.check("db_writer", {"query": "SELECT * FROM users WHERE id = 1"})
        assert result is True
    
    def test_blocks_internal_url(self, guard):
        guard.state_manager.is_killed.return_value = False
        guard.state_manager.is_capability_disabled.return_value = False
        
        with pytest.raises(GuardrailViolation):
            guard.check("api_caller", {"url": "http://169.254.169.254/latest/meta-data"})
    
    def test_blocks_fork_bomb(self, guard):
        guard.state_manager.is_killed.return_value = False
        guard.state_manager.is_capability_disabled.return_value = False
        
        with pytest.raises(GuardrailViolation):
            guard.check("code_executor", {"code": ":(){ :|:& };:"})
```

**File: `tests/test_state_manager.py`**

```python
"""Unit tests for the Redis-backed Kill Switch State Manager."""

import pytest
from unittest.mock import patch, MagicMock
from kill_switch.state_manager import KillSwitchStateManager


@pytest.fixture
def manager():
    with patch("kill_switch.state_manager.redis.Redis") as mock_redis:
        m = KillSwitchStateManager()
        m.redis = mock_redis()
        yield m


class TestHardKillState:
    def test_activate_sets_redis_key(self, manager):
        manager.activate_hard_kill(actor="test", reason="unit test")
        manager.redis.pipeline().set.assert_called()
    
    def test_is_killed_returns_true_when_active(self, manager):
        manager.redis.get.return_value = "1"
        assert manager.is_killed() is True
    
    def test_is_killed_returns_false_when_inactive(self, manager):
        manager.redis.get.return_value = None
        assert manager.is_killed() is False


class TestCapabilityState:
    def test_disable_sets_hash_field(self, manager):
        manager.disable_capability("db_writer", actor="test")
        manager.redis.hset.assert_called_with(
            "killswitch:capabilities", "db_writer", "disabled"
        )
    
    def test_is_disabled_returns_true(self, manager):
        manager.redis.hget.return_value = "disabled"
        assert manager.is_capability_disabled("db_writer") is True
    
    def test_is_disabled_returns_false(self, manager):
        manager.redis.hget.return_value = "enabled"
        assert manager.is_capability_disabled("db_writer") is False


class TestHeartbeat:
    def test_send_heartbeat_sets_key_with_ttl(self, manager):
        manager.send_heartbeat("agent-1", ttl=30)
        manager.redis.setex.assert_called_with(
            "killswitch:heartbeat:agent-1", 30, "alive"
        )
    
    def test_is_alive_returns_true_when_key_exists(self, manager):
        manager.redis.exists.return_value = 1
        assert manager.is_agent_alive("agent-1") is True
    
    def test_is_alive_returns_false_when_key_missing(self, manager):
        manager.redis.exists.return_value = 0
        assert manager.is_agent_alive("agent-1") is False
```

### Running Tests

```bash
# Run all tests
docker compose exec agent pytest tests/ -v

# Run specific test file
docker compose exec agent pytest tests/test_kill_switch_guard.py -v

# Run with coverage report
docker compose exec agent pytest tests/ --cov=agent --cov=kill_switch --cov-report=html
```

---

## 11. Sprint Plan — 4 Week Timeline

### Sprint Overview

```
Week 1: FOUNDATION          ████████████████████████████████████
Week 2: KILL SWITCH CORE    ████████████████████████████████████
Week 3: OBSERVABILITY & UI  ████████████████████████████████████
Week 4: INTEGRATION & DEMO  ████████████████████████████████████
```

---

### SPRINT 1 — Foundation (Week 1)

**Goal:** Set up infrastructure, build the basic agent, and establish the development workflow.

| Day | Task | Deliverable |
|-----|------|-------------|
| Day 1 | Project setup: Git repo, directory structure, Docker Compose skeleton, `.env` template | Working `docker compose up` with Redis + PostgreSQL + empty API |
| Day 2 | PostgreSQL schema: `audit_log` table, `agent_tasks` table, append-only triggers, seed data | Database running with correct schema |
| Day 3 | Agent core: LangGraph graph definition, state schema, system prompt, basic ReAct loop | Agent can receive a task and call the LLM (no tools yet) |
| Day 4 | Agent tools: Implement `file_manager` and `db_writer` with path traversal protection and SQL injection guards | Agent can read files and query the database |
| Day 5 | Agent tools: Implement `api_caller` and `code_executor` with sandboxing | All 4 tools working; agent can execute multi-step tasks |

**Sprint 1 Definition of Done:**
- [ ] Docker Compose starts Redis, PostgreSQL, and the agent
- [ ] Agent receives a natural language task and executes it using tools
- [ ] All 4 tools work independently (manual testing)
- [ ] Database schema is correct with append-only enforcement

---

### SPRINT 2 — Kill Switch Core (Week 2)

**Goal:** Implement all three kill switch levels, the guard, the watchdog, and the control plane API.

| Day | Task | Deliverable |
|-----|------|-------------|
| Day 1 | Kill Switch State Manager: Redis-backed global kill, soft kill, and capability flags | State manager with full CRUD + heartbeat |
| Day 2 | Kill Switch Guard: Implement `KillSwitchGuard` class, integrate into agent's `guard_check_node` | Guard intercepts every tool call and checks Redis |
| Day 3 | Guardrails: SQL pattern blocking, URL blocking, shell command blocking, path traversal guards | Dangerous operations blocked automatically |
| Day 4 | Audit Logger: PostgreSQL append-only logging, event recording for all guard decisions | Every kill switch event is recorded immutably |
| Day 5 | Watchdog + Events: Heartbeat monitoring, auto-kill on timeout, Redis Pub/Sub event system; Control Plane API: FastAPI endpoints for kill/revive/disable/status/audit | API accepts kill commands, watchdog auto-kills on heartbeat loss |

**Sprint 2 Definition of Done:**
- [ ] Hard kill: POST /kill stops the agent within 1 tool call cycle
- [ ] Soft kill: agent finishes current task but takes no new ones
- [ ] Capability kill: disabling `db_writer` blocks SQL but allows file ops
- [ ] Watchdog auto-kills agent after 60s heartbeat loss
- [ ] All events appear in the audit log
- [ ] Unit tests pass for guard, state manager, and guardrails

---

### SPRINT 3 — Observability & Frontend (Week 3)

**Goal:** Build the React dashboard, set up Prometheus + Grafana + Loki, and add LLM tracing.

| Day | Task | Deliverable |
|-----|------|-------------|
| Day 1 | Prometheus metrics: Define counters/gauges/histograms, expose `/metrics` endpoint, configure Prometheus scraping | Prometheus collects agent metrics |
| Day 2 | Structured logging + Loki: JSON logger, Loki configuration, Grafana Loki data source | Logs queryable in Grafana |
| Day 3 | Grafana dashboards: Pre-built dashboard for kill switch status, tool call rate, error rate, heartbeat status | Grafana shows live agent metrics |
| Day 4 | React dashboard: Project setup, KillSwitchPanel component (big red button), CapabilityToggles, API client | Dashboard can activate/deactivate kill switch |
| Day 5 | React dashboard: AuditLog component, MetricsChart (Recharts), WebSocket integration for real-time updates | Full operator dashboard with live data |

**Sprint 3 Definition of Done:**
- [ ] React dashboard shows kill switch status and allows one-click kill/revive
- [ ] Capability toggles enable/disable individual tools from the UI
- [ ] Audit log streams in real-time via WebSocket
- [ ] Grafana shows: tool call rate, error rate, kill switch activations, agent uptime
- [ ] Loki shows: structured JSON logs queryable by action, tool_name, etc.
- [ ] Langfuse integration captures LLM reasoning traces (optional)

---

### SPRINT 4 — Integration, Testing & Demo (Week 4)

**Goal:** End-to-end integration testing, documentation, demo scenarios, and polishing.

| Day | Task | Deliverable |
|-----|------|-------------|
| Day 1 | Integration testing: End-to-end test for hard kill, soft kill, capability kill, watchdog auto-kill, and guardrail blocking | All flows work end-to-end |
| Day 2 | Edge case testing: Concurrent kills, Redis connection failure handling, PostgreSQL connection failure handling, agent crash recovery from LangGraph checkpoint | Resilient to infrastructure failures |
| Day 3 | Demo scenarios: Create 3 scripted demo scenarios (runaway agent, privilege escalation attempt, graceful maintenance shutdown) with pre-recorded agent tasks | Demo-ready scenarios |
| Day 4 | Documentation: Finalise README, API docs (Swagger), architecture diagrams, and add inline code comments | Complete documentation |
| Day 5 | Final polish: UI/UX improvements, Grafana dashboard refinement, performance testing (guard check latency < 1ms), record demo video | Project is complete and presentable |

**Sprint 4 Definition of Done:**
- [ ] All integration tests pass
- [ ] Guard check latency is < 1ms (p99)
- [ ] 3 demo scenarios work end-to-end
- [ ] README is complete with setup instructions
- [ ] Docker Compose starts the full stack with a single command
- [ ] Demo video recorded (optional but recommended)

---

### Sprint Velocity and Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| OpenAI API rate limits during testing | Medium | Medium | Use mock LLM responses for unit tests; only use real API for integration |
| Redis connection instability | Low | High | Implement retry logic with exponential backoff in state manager |
| LangGraph checkpoint issues | Medium | Medium | Start with in-memory checkpointing, upgrade to PostgreSQL checkpointer in Sprint 3 |
| Frontend WebSocket disconnections | Medium | Low | Auto-reconnect with 3s delay (already implemented in client.ts) |
| Docker memory pressure (many services) | Medium | Medium | Allocate at least 8GB RAM to Docker; use Alpine images where possible |

---

## Final Checklist

Before declaring the project complete, verify every item:

- [ ] `docker compose up --build` starts all 9 services without errors
- [ ] Agent executes multi-step tasks using all 4 tools
- [ ] Hard kill stops the agent within 1 second
- [ ] Soft kill drains current task then stops
- [ ] Capability kill blocks specific tools while others work
- [ ] Watchdog auto-kills on heartbeat loss (60s grace period)
- [ ] Guardrails block DROP TABLE, internal URLs, and fork bombs
- [ ] React dashboard shows real-time kill switch status
- [ ] React dashboard allows one-click kill/revive/capability toggle
- [ ] Audit log records every event immutably
- [ ] Prometheus collects metrics from the agent
- [ ] Grafana displays metrics dashboards
- [ ] Loki stores and queries structured logs
- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] Guard check latency is < 1ms (p99)
- [ ] README covers setup, architecture, and API documentation

---

> **Built for learning, designed for production.** This project demonstrates that AI safety is not an afterthought — it is an architecture decision that must be made from day one.