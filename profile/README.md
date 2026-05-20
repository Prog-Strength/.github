# Prog Strength

**An AI-native weight lifting assistant for serious lifters who care about progressive overload.**

Prog Strength is an open-source workout tracker built around a single idea: the data you collect about your lifts should *talk back to you*. Instead of static logs, charts, and forms, you get a conversational AI agent with full access to your lifting history — so you can ask the questions you actually care about and get answers grounded in your real numbers.

---

## The Problem

Lifters who care about progressive overload usually end up doing one of two things:

1. **Logging in form-heavy apps** that are tedious enough most people quit after a few weeks. Without consistent data, there's no way to prove you're actually getting stronger.
2. **Asking general-purpose chatbots** (ChatGPT, etc.) for lift recommendations — which works in spurts but has no persistent memory of your training, no awareness of last session's RPE, and no structured access to the numbers.

Progressive overload is fundamentally a data problem, and the tools available today either collect the data badly or reason about it badly.

## What Prog Strength Does

A frictionless workout logger paired with a domain-specialized AI agent that has structured tool access to your training history.

You can ask things like:

- *"Given my last four bench sessions, what weight should I work up to today for 4×8?"*
- *"What's my estimated 1RM on squat right now, and how has it trended over the last 12 weeks?"*
- *"Am I actually progressing on overhead press, or have I been stuck?"*

The agent answers using tools exposed by an MCP (Model Context Protocol) server that has read access to your data via the Prog Strength API. Because it reasons over structured data — not chat history — its answers are grounded in your actual numbers.

Logging is designed to disappear: photograph a whiteboard or notebook, paste plain-text notes from your phone, and the agent parses it into a structured workout for you to confirm.

---

## Who This Is For

- **Lifters who already track progressive overload** and are frustrated with form-driven logging apps.
- **People who already ask ChatGPT for lift recommendations** and want a tool designed for the job.
- **Data-minded gymgoers** who want dashboards, trends, and estimated 1RMs without maintaining a spreadsheet.
- **Anyone logging on paper or whiteboards** who wants their notes digitized without retyping.

Not aimed at casual users looking for a beginner's program. It assumes you know what progressive overload is and care about the numbers.

---

## Core Features

- **Conversational AI assistant** — natural-language interface backed by MCP tools for querying lift history, computing 1RM estimates, surfacing trends, and recommending next-session targets.
- **Frictionless logging** — image parsing (photograph a whiteboard), note parsing (paste plain text). Manual entry exists but is explicitly the fallback.
- **Progress visualization** — per-lift progression, volume and intensity dashboards, plateau detection.
- **Training history as a first-class data layer** — owned by the user, exportable, and the grounding for every agent interaction.

---

## Architecture

Prog Strength is split across multiple repositories. The split follows deployment boundaries and language choice, not service-oriented complexity for its own sake.

```
┌─────────────────────┐         ┌─────────────────────┐
│  Web (Next.js)      │         │  Mobile (Expo)      │
│  prog-strength-web  │         │ prog-strength-mobile│
└──────────┬──────────┘         └──────────┬──────────┘
           │                               │
           └───────────────┬───────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              │ chat                    │ direct
              ▼                         │ reads/writes
   ┌─────────────────────────┐          │
   │  Agent (FastAPI)        │          │
   │  wraps Claude           │          │
   │  prog-strength-agent    │          │
   └────────────┬────────────┘          │
                │ MCP                   │
                ▼                       │
   ┌─────────────────────────┐          │
   │  MCP server (FastMCP)   │          │
   │  prog-strength-mcp      │          │
   └────────────┬────────────┘          │
                │                       │
                └───────────┬───────────┘
                            ▼
              ┌──────────────────────────┐
              │      Go Chi API          │
              │    prog-strength-api     │
              └────────────┬─────────────┘
                           ▼
                     ┌──────────┐
                     │  SQLite  │
                     └──────────┘
```

**Key invariant:** Only the Go API ever reads from or writes to SQLite. The MCP server reaches the database exclusively through the API. This keeps the data layer single-writer, centralizes auth and validation, and avoids the mess of multiple services holding direct DB connections.

The **agent** owns no per-conversation state — `/chat` is stateless, the frontend sends full message history each turn — and injects the authenticated `user_id` (from a JWT minted by the API's OAuth callback) into every tool call so Claude can never spoof another user.

### Repositories

| Repository | Purpose | Stack |
|---|---|---|
| `prog-strength-api` | Core HTTP API. Auth, workout CRUD, derived metrics. Sole reader/writer of the database. | Go, Chi, SQLite |
| `prog-strength-agent` | User-facing chat service. Wraps Claude with the MCP tool layer; JWT-validated, persistent MCP session. | Python, FastAPI |
| `prog-strength-mcp` | MCP server exposing tools to the agent. Transparent forwarder to the Go API. | Python, FastMCP |
| `prog-strength-web` | Web frontend. | Next.js, TypeScript |
| `prog-strength-mobile` *(in development)* | iOS mobile frontend. | React Native, Expo |
| `prog-strength-infra` | Infrastructure-as-code for all environments. | (IaC) |
| `.github` | This profile and shared org-level configuration. | — |

### Why these choices

- **Go + Chi for the API** — fast, simple, deploys as a single binary, fits the read-heavy structured workload of workout data.
- **SQLite** — each user's data is small and well-bounded; SQLite is more than sufficient and pairs cleanly with a single-writer design.
- **Python for agent + MCP** — the Python MCP ecosystem is the most mature, and these layers are glue code, not the hot path.
- **Next.js + Expo for frontends** — shared API contract and type shapes between them; the iOS app talks to the same Go API and Python agent as the web app.
- **Multi-repo** — each repo has a distinct deployment lifecycle and toolchain. They're not microservices; they're separately-deployed components of one app.

---

## Status

Early development. This is a portfolio side project being built in the open. The web app and backend stack are live at `progstrength.fitness`; the iOS app is actively in development. Expect breaking changes until the first tagged release.

## License

Prog Strength is fully open-source. See individual repositories for license details.
