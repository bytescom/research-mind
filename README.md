# 🔬 ResearchMind: Autonomous Multi-Agent AI Research System

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![LangChain](https://img.shields.io/badge/LangChain-v0.2%2B-green.svg)](https://www.langchain.com/)
[![OpenAI](https://img.shields.io/badge/Model-GPT--4o--mini-orange.svg)](https://openai.com/)
[![Streamlit](https://img.shields.io/badge/UI-Streamlit-red.svg)](https://streamlit.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

An autonomous multi-agent research platform that coordinates specialized AI agents (Search, Reader, Writer, Critic) using LangChain and OpenAI to browse the live internet, sanitize and extract deep web content, synthesize comprehensive technical reports, and perform automated peer reviews.

---

## 🌟 Key Features

- **Autonomous Multi-Agent Coordination:** Decomposes complex research tasks into dedicated agents collaborating through an in-memory shared state blackboard.
- **Live Internet Search:** Integrates the **Tavily Search API** to fetch recent, high-credibility web resources and structured snippets.
- **Deep Content Extraction:** Automated DOM parsing with **BeautifulSoup4** that sanitizes HTML (stripping scripts, styling, navigation, and footers) to extract up to 3,000 characters of clean context.
- **Structured Academic Reports:** Generates structured Markdown reports formatted with Introduction, Key Findings ($\ge 3$ detailed points), Conclusion, and verified Sources.
- **Automated Peer Review & Scoring:** A dedicated Critic Chain evaluates report rigor, outputting an objective score (/10), strengths, areas for improvement, and a one-line verdict.
- **Interactive Streamlit Dashboard:** Custom dark-mode UI with real-time reactive status cards (`WAITING` ➔ `● RUNNING` ➔ `✓ DONE`), raw telemetry expanders, and instant Markdown export.

---

## 🏗️ System Architecture

```
                               ┌────────────────────────────────┐
                               │       User Topic Request       │
                               └────────────────────────────────┘
                                                │
                                                ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    SHARED STATE BLACKBOARD                                    │
│                                                                                               │
│  1. SEARCH AGENT (create_agent) ──▶ Queries Tavily API ──▶ Writes state["search_results"]     │
│             │                                                                                 │
│             ▼                                                                                 │
│  2. READER AGENT (create_agent) ──▶ BS4 Deep Scraping  ──▶ Writes state["scraped_content"]    │
│             │                                                                                 │
│             ▼                                                                                 │
│  3. WRITER CHAIN (LCEL Sequence)──▶ Synthesizes Report ──▶ Writes state["report"]             │
│             │                                                                                 │
│             ▼                                                                                 │
│  4. CRITIC CHAIN (LCEL Sequence)──▶ Evaluates & Scores ──▶ Writes state["feedback"]           │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
                                                │
                                                ▼
                               ┌────────────────────────────────┐
                               │   Streamlit Web UI / CLI /     │
                               │   One-Click Markdown Download  │
                               └────────────────────────────────┘
```

---

## 🧩 Agent Breakdown

| Agent / Chain | Architecture | Tooling | Role |
| :--- | :--- | :--- | :--- |
| **Search Agent** | Tool-calling Agent (`create_agent`) | Tavily Search API | Scours the live web and retrieves top 5 authoritative candidates with URLs and snippets. |
| **Reader Agent** | Tool-calling Agent (`create_agent`) | BeautifulSoup4 (`scrape_url`) | Inspects candidate links, selects the most relevant URL, and sanitizes body content. |
| **Writer Chain** | LCEL Runnable Sequence | OpenAI GPT-4o-mini | Merges broad snippets with deep scraped text to compile an academic 4-part research brief. |
| **Critic Chain** | LCEL Runnable Sequence | OpenAI GPT-4o-mini | Evaluates the draft report as an objective peer reviewer, scoring quality and citing gaps. |

---

## 🛠️ Tech Stack

- **Core:** Python 3.10+
- **Orchestration:** LangChain (`langchain`, `langchain-core`, `langchain-openai`)
- **LLM Engine:** OpenAI `gpt-4o-mini` (pinned to `temperature=0` for factual fidelity)
- **Search & Scraping:** Tavily Python Client, BeautifulSoup4, Requests, lxml, html5lib
- **Frontend:** Streamlit with custom CSS design system (`Syne`, `DM Mono`, `DM Sans` fonts)
- **Config & Environment:** `python-dotenv`

---

## 🚀 Getting Started

### Prerequisites

- Python 3.10 or higher
- [OpenAI API Key](https://platform.openai.com/)
- [Tavily API Key](https://tavily.com/) (Free tier available)

### Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/mepan/multi_agent_system.git
   cd multi_agent_system
   ```

2. **Create and activate a virtual environment:**
   ```bash
   # Using uv (Recommended for speed):
   uv venv
   .venv\Scripts\activate          # On Windows PowerShell
   # source .venv/bin/activate     # On Linux/macOS

   # Or using standard venv:
   python -m venv .venv
   .venv\Scripts\activate
   ```

3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   # Or with uv: uv pip install -r requirements.txt
   ```

4. **Configure environment variables:**
   Create a `.env` file in the project root:
   ```env
   OPENAI_API_KEY="your-openai-api-key"
   TAVILY_API_KEY="your-tavily-api-key"
   ```

---

## 💻 Usage

### Option 1: Streamlit Web Dashboard (Recommended)

Launch the interactive web UI:
```bash
streamlit run app.py
```
Open **`http://localhost:8501`** in your browser, enter your research topic, and click **⚡ Run Research Pipeline**.

### Option 2: Terminal / CLI Mode

Run the terminal pipeline directly:
```bash
python pipeline.py
```
Type any topic (e.g., *"Breakthroughs in Quantum Computing 2025"*) to view live step-by-step logs as each agent executes.

---

## 📂 Project Structure

```
multi_agent_system/
├── .env                  # API keys (TAVILY_API_KEY, OPENAI_API_KEY - never commit)
├── .gitignore            # Excludes secrets, venvs, and cache files
├── requirements.txt      # Project dependencies
├── tools.py              # Custom agent tools: web_search & scrape_url
├── agents.py             # Agent configurations (Search, Reader) & LCEL chains (Writer, Critic)
├── pipeline.py           # State orchestrator & CLI runner
├── app.py                # Streamlit web application
├── STEPS.md              # Detailed developer build log and engineering study
└── README.md             # Project documentation
```

---

## 💡 Engineering Highlights

- **Context-Window Optimization:** Web scraping frequently dumps megabytes of raw HTML boilerplate. The scraper strips `<script>`, `<style>`, `<nav>`, and `<footer>` tags and caps plain text to 3,000 characters, safeguarding LLM context limits and preventing token spikes.
- **Modern `create_agent` vs. Legacy ReAct:** Built on LangChain's native state-graph agent compiler, utilizing structured tool calls instead of fragile string regex matching.
- **Zero-Temperature Determinism:** All reasoning chains run at `temperature=0`, eliminating creative hallucination when citing external URLs.

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
