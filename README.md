# Harness Engineering in Agentic AI

A small, readable demo of **harness engineering**: the control layer around LLM agents that makes them safer, cheaper, and easier to inspect.

This is not a chatbot wrapper. It is a production-shaped support pipeline:

```
Customer question
        │
        ▼
   🛡️ Guardrail          deterministic safety check
        │
        ▼
   🧭 Router             deterministic routing (no LLM)
        │
        ├── technical agent
        ├── billing agent
        └── general agent
        │
        ▼
   ✅ Reviewer           LLM quality gate
        │
        ▼
   Final answer + execution trace
```

The agents write drafts. The **harness** decides whether the request is allowed, which specialist should handle it, whether the draft is good enough, and what the operator can see.

---

## Why this exists

Most agent demos start with “give an LLM tools and let it decide.” That is useful for prototypes. It is a poor default for production.

**Harness engineering** is the practice of putting ordinary software around the model:

| Layer | Job | In this repo |
| --- | --- | --- |
| Guardrails | Block unsafe or out-of-policy requests before they reach a model | Phrase checks in `graph.py` |
| Routing | Send work to the right specialist without paying for an LLM call | Keyword router in `graph.py` |
| Specialists | Narrow prompts, not one mega-agent | `agents.py` |
| Review | Catch unsafe, vague, or overconfident drafts | Reviewer agent |
| Trace | Show *what actually ran* | `trace` on the graph state |

The teaching point: **if a rule is simple and predictable, use Python.** Save the LLM for language work.

---

## What you get when you run it

A FastAPI page where you type a support question. The UI returns:

1. **Final answer** — after review (or a hard block from the guardrail)
2. **Route** — `technical`, `billing`, `general`, or blocked
3. **Harness execution trace** — the steps the graph actually took

Example traces:

```
1. Guardrail checked the request
2. Router selected: technical
3. Technical Agent created a draft
4. Reviewer Agent checked the answer
```

If the question is blocked, the graph ends after the guardrail. No specialist is called.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Browser  (templates/index.html + static/)              │
│  POST /api/chat  { "message": "..." }                   │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│  app.py  FastAPI                                        │
│  load_dotenv → run_support_system(question)             │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│  graph.py  LangGraph StateGraph                         │
│                                                         │
│  SupportState                                           │
│    question, route, draft, final_answer,                │
│    blocked, trace                                       │
│                                                         │
│  START → guardrail ─┬─ blocked → END                    │
│                     └─ ok → router                      │
│                              │                          │
│                              ├─ technical_agent         │
│                              ├─ billing_agent           │
│                              └─ general_agent           │
│                                       │                 │
│                                       ▼                 │
│                                    reviewer → END       │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│  agents.py  Groq ChatGroq  (openai/gpt-oss-20b)         │
│  technical / billing / general / reviewer prompts       │
└─────────────────────────────────────────────────────────┘
```

### Shared state

Every node reads and writes a typed dict (`SupportState`). That is the harness contract:

- `question` — original user text
- `route` — which specialist was chosen
- `draft` — specialist output before review
- `final_answer` — what the user sees
- `blocked` — guardrail result
- `trace` — human-readable log of each hop

---

## How each layer works

### 1. Guardrail (deterministic)

`guardrail_node` scans the question for unsafe phrases (passwords, stolen credentials, full card numbers).

- If blocked: set `final_answer` to a refusal and jump to `END`.
- If allowed: continue to the router.

This is intentionally *not* an LLM. Safety that you can unit-test should not depend on model mood.

### 2. Router (deterministic)

`router_node` classifies with keyword lists:

| Route | Signals |
| --- | --- |
| `technical` | error, bug, login, api, install, code, server, … |
| `billing` | price, payment, refund, bill, subscription, plan, … |
| `general` | everything else |

The comment in the code is the design rule: production systems should not spend an LLM call on every cheap decision.

### 3. Specialist agents (LLM)

Three narrow roles in `agents.py`:

- **Technical** — login, API errors, install, bugs. Simple steps. No invented account data.
- **Billing** — demo policy only (Starter $10, Pro $25). Never claims a refund is already approved. Never asks for card numbers or passwords.
- **General** — polite answers; defers specialist topics.

Each agent returns a **draft**, not the user-facing answer.

### 4. Reviewer (LLM quality gate)

`reviewer_agent` checks the draft for clarity, relevance, unsafe requests, unsupported guarantees, and extra complexity. It rewrites if needed and returns only the final answer.

That split — *generate then inspect* — is the harness idea in miniature.

---

## Project layout

```
Harness-in-Agentic-AI/
├── app.py              # FastAPI app, `/` and `/api/chat`
├── graph.py            # LangGraph harness: state, nodes, edges
├── agents.py           # Groq LLM specialists + reviewer
├── templates/
│   └── index.html      # Demo UI
├── static/
│   ├── style.css
│   └── script.js       # Calls /api/chat, renders answer + trace
├── pyproject.toml      # Dependencies (uv / pip)
├── uv.lock
├── LICENSE             # Apache-2.0
└── main.py             # uv scaffold leftover; not the app entrypoint
```

Read in this order if you are learning the idea:

1. `graph.py` — the harness
2. `agents.py` — the models inside it
3. `app.py` + `static/script.js` — how the trace is shown

---

## Quick start

**Requirements:** Python 3.13+, a [Groq](https://console.groq.com/) API key.

### 1. Clone and enter the repo

```bash
git clone https://github.com/<you>/Harness-in-Agentic-AI.git
cd Harness-in-Agentic-AI
```

### 2. Create a virtualenv and install

With [uv](https://docs.astral.sh/uv/):

```bash
uv sync
```

Or with pip:

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

pip install -e .
```

### 3. Environment

Create a `.env` in the project root (this file is gitignored):

```env
GROQ_API_KEY=your_groq_api_key_here
```

The model is `openai/gpt-oss-20b` via `langchain-groq`. Change `MODEL_NAME` in `agents.py` if you want a different Groq model.

### 4. Run the demo

```bash
python app.py
```

Open [http://127.0.0.1:8080](http://127.0.0.1:8080).

The server binds to `0.0.0.0:8080` with reload enabled.

---

## Try these questions

The UI has the same examples as chips.

| Intent | Question | What the harness should do |
| --- | --- | --- |
| Technical | `I am getting an API login error. How can I fix it?` | Route → technical → review |
| Billing | `How much is the Pro plan and can I get a refund?` | Route → billing; reviewer should not invent an approved refund |
| General | `What can your support team help me with?` | Route → general |
| Guardrail | A request that asks for a password or a full card number | Block; no specialist call |

Watch the **Harness Execution Trace** after each run. That panel is the point of the demo.

---

## API

```http
POST /api/chat
Content-Type: application/json

{ "message": "How much is the Pro plan?" }
```

```json
{
  "route": "billing",
  "answer": "...",
  "trace": [
    "Guardrail checked the request",
    "Router selected: billing",
    "Billing Agent created a draft",
    "Reviewer Agent checked the answer"
  ]
}
```

You can call this from curl or any HTTP client; the page is only a viewer.

---

## Design choices worth copying

These are the ideas this repo is meant to make obvious:

1. **State is explicit.** Nodes do not hide data in globals. `SupportState` is the contract.
2. **Cheap decisions stay in code.** Guardrail and router are Python. They are fast, free, and testable.
3. **Agents are specialists.** Narrow prompts beat one omniscient agent for support work.
4. **Draft ≠ answer.** A reviewer sits between generation and the user.
5. **Trace is first-class.** If you cannot say which node ran, you do not have a harness — you have a prompt.
6. **Fail closed on safety.** Blocked requests skip the rest of the graph.

What this demo does **not** include (on purpose): tool calling, memory, RAG, auth, rate limits, evals, or a real moderation API. Those belong in a later layer of the same harness, not in the first explanation.

---

## Extending it

Small next steps if you want to grow this into a real harness:

- Replace the keyword lists with tests + a classifier only when rules fail.
- Swap the phrase guardrail for a moderation / PII library.
- Add evals: same questions, assert route, assert “refund not approved,” assert blocked traces.
- Log `trace` and `route` as structured events.
- Put policy (prices, refunds) in data, not only in the billing prompt.

---

## License

Apache License 2.0. See [LICENSE](LICENSE).
