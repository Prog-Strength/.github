# Prog Strength

**An AI-native weight lifting assistant for serious lifters who care about progressive overload.**

Prog Strength is an open-source workout tracker built around a single idea: the data you collect about your lifts should *talk back to you*. Instead of static logs, charts, and forms, Prog Strength gives you a conversational AI agent with full access to your lifting history — so you can ask the questions you actually care about and get answers grounded in your real numbers.

---

## The Problem

Most lifters who care about progressive overload end up doing one of two things:

1. **Logging workouts in apps with rigid, form-heavy interfaces** — which is tedious enough that most people stop logging after a few weeks. Without consistent data, there's no way to prove you're actually getting stronger.
2. **Asking general-purpose chatbots (ChatGPT, etc.) for lift recommendations** — which works in spurts but has no persistent memory of your training history, no awareness of your last session's RPE, and no structured access to the numbers it would need to give a real answer.

Neither of these is good enough for someone who treats lifting seriously. Progressive overload is fundamentally a data problem, and the tools available today either collect the data badly or reason about it badly.

## What Prog Strength Does

Prog Strength combines a frictionless workout logger with a domain-specialized AI agent that has structured tool access to your entire training history.

You can ask things like:

- *"Given my last four bench sessions, what weight should I work up to today for 4×8?"*
- *"What's my estimated 1RM on squat right now, and how has it trended over the last 12 weeks?"*
- *"Am I actually progressing on overhead press, or have I been stuck?"*
- *"How does today's planned volume compare to my average for this lift?"*

The agent answers using tools exposed by an MCP (Model Context Protocol) server that has read access to your lifting data via the Prog Strength API. Because the agent reasons over structured data — not a fuzzy chat history — its answers are grounded in your actual numbers, not guesses.

Logging is designed to disappear. Snap a photo of a whiteboard or notebook, paste in plain-text notes from your phone, and the agent will parse it into a structured workout for you to confirm. The less friction between finishing a set and having it recorded, the more consistently the app gets used — and consistency is what makes the data useful.

---

## Who This Is For

- **Lifters who already track progressive overload** and are frustrated with form-driven logging apps.
- **People who already ask ChatGPT for lift recommendations** and want a tool that's actually designed for the job.
- **Data-minded gymgoers** who want dashboards, trends, and estimated 1RMs without manually maintaining a spreadsheet.
- **Anyone who logs workouts on paper or whiteboards** and wants their notes digitized without retyping.

This is not aimed at casual users looking for a beginner's program. It assumes you know what progressive overload is and that you care about the numbers.

---

## Core Features

A deliberately small, focused feature set. Everything below is in scope; everything else is not.

### 1. Conversational AI Lifting Assistant
- Natural-language interface as the primary way to interact with your data.
- Backed by an MCP server with tools for querying lift history, computing 1RM estimates, surfacing trends, and recommending next-session targets.
- Answers grounded in *your* data, not generic advice.

### 2. Frictionless Workout Logging
- **Image parsing** — photograph a whiteboard or notebook workout log and have it parsed into a structured workout.
- **Note parsing** — paste plain-text workout notes and have them structured automatically.
- **Manual entry** — available but explicitly secondary; the goal is to make the other paths good enough that manual entry is the fallback.

### 3. Progress Visualization
- Per-lift progression charts (weight × reps over time, estimated 1RM).
- Volume and intensity dashboards.
- Plateau detection and trend summaries.

### 4. Personal Training History as a First-Class Data Layer
- Every interaction with the agent is grounded in your structured history.
- Data is owned by the user and exportable.

---

## Architecture

Prog Strength is intentionally split across multiple repositories. The split is driven by deployment boundaries and language choice, not by service-oriented complexity for its own sake.

```
┌──────────────────────┐      ┌──────────────────────┐
│  Web App / Mobile    │─────▶│  AI Agent + MCP      │
│  (frontend repos)    │      │  (prog-strength-mcp) │
└──────────────────────┘      └──────────┬───────────┘
           │                              │
           │                              ▼
           │                  ┌──────────────────────┐
           └─────────────────▶│  Go Chi API          │
                              │  (prog-strength-api) │
                              └──────────┬───────────┘
                                         ▼
                                  ┌─────────────┐
                                  │   SQLite    │
                                  └─────────────┘
```

**Key invariant:** Only the Go API ever reads from or writes to SQLite. The MCP server reaches the database exclusively through the API. This keeps the data layer single-writer, makes auth and validation centralized, and avoids the usual mess that comes with multiple services holding direct DB connections.

### Repositories

| Repository | Purpose | Stack |
|---|---|---|
| `prog-strength-api` | Core HTTP API. Handles auth, workout CRUD, derived metrics, and is the sole reader/writer of the database. | Go, Chi, SQLite |
| `prog-strength-mcp` | MCP server exposing tools to the AI agent. Forwards every data request to the Go API. | Python |
| `prog-strength-infra` | Infrastructure-as-code for all environments. | (IaC) |
| `prog-strength-web` *(planned)* | Web frontend. | TBD |
| `prog-strength-mobile` *(planned)* | Mobile frontend. | TBD |
| `.github` | This profile and shared org-level configuration. | — |

### Why these choices

- **Go + Chi for the API** — fast, simple, easy to deploy as a single binary, and a good fit for the read-heavy, structured workload of workout data.
- **SQLite** — this is a personal-data app where each user's data is small and well-bounded. SQLite is more than sufficient, dramatically simplifies operations, and pairs cleanly with a single-writer design.
- **Python for the MCP server** — the Python MCP ecosystem is the most mature, and the MCP layer is glue code, not the hot path.
- **Multi-repo** — each repo has a distinct deployment lifecycle and toolchain. Splitting them keeps CI fast and boundaries clean. They're not microservices; they're separately-deployed components of one app.

---

## Status

Early development. This is a portfolio side project being built in the open. The repo layout is in place; feature work is underway. Expect breaking changes until the first tagged release.

## License

Prog Strength is fully open-source. See individual repositories for license details.
