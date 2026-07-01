# 📈 StockPicker — A Hierarchical Multi-Agent Stock Analyst

StockPicker is an autonomous, multi-agent AI system built with [CrewAI](https://crewai.com) that researches the stock market and recommends a single company worth investing in — end to end, with no human in the loop.

Point it at a sector (e.g. *Pharmaceuticals*), and a team of specialised AI agents will:

1. Scan the latest financial news to find companies that are currently trending.
2. Perform deep financial research on each of those companies.
3. Compare them and pick the single best investment opportunity.
4. Push a notification to your phone with the decision and a one-line rationale.
5. Write a detailed report explaining *why* it chose that company — and why it rejected the others.

The two things that make this project interesting are how it uses **a hierarchical (manager-led) agent structure** and **persistent memory**. Those are explained first, because they are the heart of the design.

---

## 🧠 1. The Hierarchical Agent Structure

Most simple CrewAI projects run agents **sequentially** — Agent A finishes, hands its output to Agent B, and so on in a fixed, hard-coded order. StockPicker does *not* do this. It uses CrewAI's **hierarchical process**, where a dedicated **Manager agent** sits on top and orchestrates everything.

### How it works

```
                        ┌─────────────────────────┐
                        │      MANAGER AGENT      │
                        │  (Project Manager LLM)  │
                        │   allow_delegation=True │
                        └───────────┬─────────────┘
                                    │  delegates & coordinates
              ┌─────────────────────┼──────────────────────┐
              ▼                     ▼                       ▼
   ┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐
   │ Trending Company   │ │ Financial          │ │ Stock Picker       │
   │ Finder             │ │ Researcher         │ │                    │
   │                    │ │                    │ │                    │
   │ 🔎 SerperDevTool   │ │ 🔎 SerperDevTool   │ │ 📲 Push Notify     │
   │ finds trending     │ │ deep-dives each    │ │ picks best + alerts│
   │ companies in news  │ │ company            │ │ user               │
   └────────────────────┘ └────────────────────┘ └────────────────────┘
```

Instead of the developer wiring the agents together in a fixed sequence, the **Manager agent decides** who does what and in which order, delegating each task to the worker agent best suited for it, collecting their results, and driving the crew toward the overall goal: *pick the best company for investment.*

### Why this matters

- **Delegation is dynamic.** The manager is created with `allow_delegation=True`, giving it CrewAI's built-in *delegate work* and *ask question* tools. It can hand sub-tasks to the specialist agents and query them for clarification.
- **The workers stay focused.** Each worker agent (Finder, Researcher, Picker) is a specialist with a narrow role, goal, and backstory. They don't need to know about the big picture — the manager holds that.
- **It scales.** Adding a new capability is as simple as adding another specialist agent; the manager will fold it into its coordination without the pipeline being re-wired by hand.

### Where it's defined

In [`src/stock_picker/crew.py`](src/stock_picker/crew.py), the crew is assembled like this:

```python
manager = Agent(
    config=self.agents_config['manager'],
    allow_delegation=True          # ← the manager can delegate to workers
)

return Crew(
    agents=self.agents,            # the three worker agents
    tasks=self.tasks,
    process=Process.hierarchical,  # ← hierarchical, not sequential
    manager_agent=manager,         # ← our custom manager sits on top
    memory=True,
    verbose=True,
    tracing=True,
)
```

The key lines are `process=Process.hierarchical` and `manager_agent=manager`. Note that the manager is **not** in the worker `agents` list — it is a separate coordinator. The manager's persona (an "experienced and highly effective project manager who can delegate tasks to the right people") is defined in [`src/stock_picker/config/agents.yaml`](src/stock_picker/config/agents.yaml).

---

## 💾 2. Memory

Every agent in this crew — and the crew itself — has **memory enabled**. This is what lets StockPicker *learn and stay consistent* rather than starting from a blank slate every run.

### How it's enabled

- Each worker agent is created with `memory=True`:
  ```python
  Agent(config=..., tools=[SerperDevTool()], memory=True)
  ```
- The crew is also created with `memory=True`, which turns on CrewAI's full memory system.

### What memory does here

CrewAI's memory system combines several layers that work together automatically:

| Memory type | Backed by | Purpose in StockPicker |
|-------------|-----------|------------------------|
| **Short-term** | ChromaDB + RAG | Holds context during the current run so agents share what they've each discovered |
| **Long-term** | SQLite | Persists insights *across runs*, so the crew remembers what it learned last time |
| **Entity** | RAG | Tracks the companies, tickers, and people it encounters |
| **Contextual** | — | Weaves the above together into coherent responses |

A concrete, practical payoff: the agents are instructed to **"always pick new companies — don't pick the same company twice."** Long-term memory is what makes that instruction actually stick from one run to the next — the crew remembers which companies it already recommended and moves on to fresh opportunities.

Memory data is written to a local store the first time you run the crew. You can clear it anytime with:

```bash
crewai reset-memories -a      # reset all memory
crewai reset-memories -l      # long-term only
crewai reset-memories -s      # short-term only
```

---

## 🔁 3. The Full Pipeline — What It Actually Does

Under the manager's coordination, three tasks flow into one another (each task's output becomes context for the next), defined in [`src/stock_picker/config/tasks.yaml`](src/stock_picker/config/tasks.yaml):

### Task 1 — Find Trending Companies
- **Agent:** Trending Company Finder
- **Tool:** `SerperDevTool` (Google search via [Serper.dev](https://serper.dev))
- **Does:** Reads the latest news for the chosen sector and identifies 2–3 companies currently trending, along with *why* each is in the news.
- **Output:** [`output/trending_companies.json`](output/trending_companies.json) — structured JSON validated against a Pydantic model (`TrendingCompanyList`).

### Task 2 — Research Trending Companies
- **Agent:** Financial Researcher
- **Tool:** `SerperDevTool`
- **Uses context from:** Task 1
- **Does:** Takes the trending companies and produces a comprehensive analysis of each — market position, future outlook, and investment potential.
- **Output:** [`output/research_report.json`](output/research_report.json) — structured JSON (`TrendingCompanyResearchList`).

### Task 3 — Pick the Best Company
- **Agent:** Stock Picker
- **Tool:** `send_push_notification` (custom tool, via Pushover)
- **Uses context from:** Task 2
- **Does:** Analyses all the research, selects the single best company for investment, **sends a push notification** to the user with the decision + one-sentence rationale, then writes a full report explaining the choice and why the other candidates were rejected.
- **Output:** [`output/decision.md`](output/decision.md) — a human-readable Markdown report.

### Structured outputs (Pydantic)

Data doesn't flow between agents as loose text — it's validated. [`crew.py`](src/stock_picker/crew.py) defines Pydantic models (`TrendingCompany`, `TrendingCompanyResearch`, and their list wrappers) that force each task's output into a strict, typed schema. This keeps the hand-off between agents reliable and machine-readable.

---

## 🛠️ Tech Stack & Key Components

| Component | What it is |
|-----------|-----------|
| **[CrewAI](https://crewai.com)** (`crewai[tools]==1.14.4`) | The multi-agent orchestration framework |
| **`SerperDevTool`** | Real-time Google search, giving agents live access to the latest news |
| **Custom Push Tool** | [`tools/push_tool.py`](src/stock_picker/tools/push_tool.py) — sends phone notifications via the [Pushover](https://pushover.net) API |
| **Pydantic** | Enforces structured, typed output between agents |
| **OpenAI GPT** | The LLM powering every agent (configurable in `agents.yaml`) |
| **[UV](https://docs.astral.sh/uv/)** | Fast Python dependency & environment management |
| **ChromaDB + SQLite** | Backing stores for the memory system |

---

## 📁 Project Structure

```
stock_picker/
├── knowledge/
│   └── user_preference.txt        # Facts about the user (name, role, interests)
├── output/                        # Generated on each run
│   ├── trending_companies.json    # Task 1 output
│   ├── research_report.json       # Task 2 output
│   └── decision.md                # Task 3 output — the final recommendation
├── src/stock_picker/
│   ├── config/
│   │   ├── agents.yaml            # Roles, goals & backstories for all 4 agents
│   │   └── tasks.yaml             # The 3 tasks and how they chain together
│   ├── tools/
│   │   └── push_tool.py           # Custom Pushover notification tool
│   ├── crew.py                    # Crew assembly: agents, tasks, hierarchy, memory
│   └── main.py                    # Entry point — sets inputs and kicks off the crew
├── pyproject.toml                 # Dependencies & CLI scripts
└── README.md
```

---

## 🚀 Getting Started

### Prerequisites
- Python `>=3.10, <3.14`
- An OpenAI API key
- A [Serper.dev](https://serper.dev) API key (for web search)
- *(Optional)* [Pushover](https://pushover.net) credentials, for the push-notification feature

### 1. Install UV
```bash
pip install uv
```

### 2. Install dependencies
From the project root:
```bash
crewai install
```

### 3. Configure your environment
Create a `.env` file in the project root:
```env
OPENAI_API_KEY=sk-...
SERPER_API_KEY=...
# Optional — enables the push notification in Task 3
PUSHOVER_USER=...
PUSHOVER_TOKEN=...
```

### 4. Run it
```bash
crewai run
```

This kicks off the crew. By default it analyses the **Pharmaceuticals** sector — change the `sector` input in [`src/stock_picker/main.py`](src/stock_picker/main.py) to target any sector you like:

```python
inputs = {
    'sector': 'Pharmaceuticals',
    "current_date": str(datetime.now().date())
}
```

When it finishes, check the [`output/`](output/) folder for the results — the final investment recommendation lands in `output/decision.md`.

---

## 🎛️ Customising

- **Change the sector** → edit `inputs` in [`main.py`](src/stock_picker/main.py)
- **Tune agent personas** → edit [`config/agents.yaml`](src/stock_picker/config/agents.yaml)
- **Change the workflow / tasks** → edit [`config/tasks.yaml`](src/stock_picker/config/tasks.yaml)
- **Swap the LLM** → change the `llm:` field for any agent in `agents.yaml`
- **Add tools or logic** → edit [`crew.py`](src/stock_picker/crew.py)

---

## ⚠️ Disclaimer

This project is for **educational and demonstration purposes only**. It is a technology demo of multi-agent AI systems — **not** financial advice. Do not make real investment decisions based on its output.

---

## 🙏 Acknowledgements

Built as part of learning the **CrewAI** framework, based on the agentic AI course materials by [Ed Donner](https://github.com/ed-donner/agents). The hierarchical-process and memory patterns showcased here are core CrewAI concepts.
