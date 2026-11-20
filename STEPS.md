# Building ResearchMind: An Autonomous Multi-Agent AI Research System
*Developer Build Log & Engineering Notes*

---

## 1. Project Motivation & Core Concept

When asking a standard LLM a complex research question, two fundamental problems arise:
1. **Shallow Knowledge & Cutoffs:** The model answers from pre-trained memory or a single quick web snippet, leading to hallucinated details or outdated information.
2. **Cognitive Overload in Single Prompts:** Forcing a single prompt to search Google, evaluate search results, read long articles, synthesize facts, and format a professional report usually results in a generic, unfocused summary.

To solve this, I designed **ResearchMind**—an autonomous multi-agent research pipeline built with **LangChain** and **OpenAI**. 

Instead of relying on a single prompt, I split the workflow across four specialized entities collaborating via a **shared-state blackboard architecture**:

| Component | Architecture | Tooling / Model | Primary Responsibility |
| :--- | :--- | :--- | :--- |
| **1. Search Agent** | Tool-calling Agent (`create_agent`) | Tavily Search API | Scours the live internet for top 5 authoritative sources and snippets. |
| **2. Reader Agent** | Tool-calling Agent (`create_agent`) | BeautifulSoup4 (`scrape_url`) | Evaluates the search results, visits the most relevant URL, and extracts sanitized body text. |
| **3. Writer Chain** | LCEL Runnable Sequence | GPT-4o-mini | Synthesizes all gathered research into an exhaustive, structured 4-part Markdown report. |
| **4. Critic Chain** | LCEL Runnable Sequence | GPT-4o-mini | Acts as an impartial senior reviewer—evaluating the report, scoring it out of 10, and flagging gaps. |

---

## 2. Architecture & Data Flow

I chose a sequential state-passing pattern where each agent reads from and writes to an in-memory dictionary (`state`):

```
User Input (Research Topic)
            │
            ▼
┌─────────────────────────┐
│   Step 1: Search Agent  │ ──▶ Hits Tavily API ──▶ Writes state["search_results"]
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│   Step 2: Reader Agent  │ ──▶ Selects top URL & scrapes DOM ──▶ Writes state["scraped_content"]
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│   Step 3: Writer Chain  │ ──▶ Combines search + scraped text ──▶ Writes state["report"]
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│   Step 4: Critic Chain  │ ──▶ Evaluates report quality ──▶ Writes state["feedback"]
└─────────────────────────┘
            │
            ▼
Streamlit Interactive UI / Terminal Output / Markdown Export
```

---

## 3. Step-by-Step Implementation Walkthrough

### Step 1: Environment & Tooling Setup

I initialized the project environment using `uv` because of its speed compared to traditional `pip`:

```bash
uv venv
source .venv/Scripts/activate  # On Windows PowerShell: .venv\Scripts\activate
```

Next, I set up the environment variables in a root `.env` file:
- `OPENAI_API_KEY`: Powers the reasoning LLM (`gpt-4o-mini`).
- `TAVILY_API_KEY`: Powers the real-time search engine API.

To protect sensitive keys, I created a `.gitignore` immediately to ensure `.env` and virtual environments are never committed.

---

### Step 2: Building Agent Tools (`tools.py`)

Agents are only as capable as the tools they have access to. I created two focused, production-grade tools using LangChain's `@tool` decorator:

#### 1. Web Search (`web_search`)
Instead of wrestling with generic search scraping and anti-bot challenges, I integrated the **Tavily Search API**, which is designed specifically for LLM workflows:
- I configured `max_results=5` to gather diverse perspectives without overwhelming the context window.
- Each result parses the `title`, `url`, and a truncated 300-character snippet.
- The output joins the items with a clean delimiter (`----`) so the downstream agent can easily distinguish between candidates.

#### 2. Deep Web Scraper (`scrape_url`)
Once the Search Agent finds promising URLs, the Reader Agent needs to dive deeper than a short snippet.
- **Header Spoofing:** Configured a standard browser `User-Agent` (`Mozilla/5.0`) to avoid instant 403 Forbidden blocks.
- **DOM Sanitization:** Raw web pages are full of noise. I used BeautifulSoup's `decompose()` to strip out `<script>`, `<style>`, `<nav>`, and `<footer>` elements before extracting text.
- **Context Protection:** I trimmed the cleaned text to a maximum of 3,000 characters with `strip=True`. This provides ample depth for the LLM while preventing massive token spikes and unnecessary latency.
- **Defensive Error Handling:** Wrapped the request in a `try/except` block with an 8-second timeout, returning a descriptive error string rather than allowing an unhandled exception to crash the pipeline.

---

### Step 3: Crafting Specialized Agents & Chains (`agents.py`)

This file is the brain of the project.

#### Key Architectural Decision: `create_agent` vs Legacy ReAct
Older LangChain tutorials often rely on `create_react_agent`, which relies on brittle text-parsing loops (`Action:` / `Action Input:`). I deliberately chose LangChain's modern **`create_agent`** factory:
- It compiles internally into a robust state graph.
- It leverages OpenAI's native tool-calling protocol (passing structured JSON rather than parsing free-form strings).
- It handles multi-turn agent loops cleanly without infinite loop edge cases.

#### The Model Configuration
```python
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
```
I locked `temperature=0` across the entire pipeline. When writing factual research reports, consistency and adherence to source material far outweigh creative variance.

#### 1. The Search Agent
```python
def build_search_agent():
    return create_agent(model=llm, tools=[web_search])
```
Equipped solely with the `web_search` tool, this agent focuses entirely on identifying credible sources for the topic.

#### 2. The Reader Agent
```python
def build_reader_agent():
    return create_agent(model=llm, tools=[scrape_url])
```
Equipped with `scrape_url`, this agent reads the upstream search results, picks the single most authoritative link, and retrieves detailed content.

#### 3. The Writer Chain (LCEL)
The writer does not require tool execution, so wrapping it in an agent loop would be redundant and slow. Instead, I built a deterministic LCEL (LangChain Expression Language) pipeline:
```python
writer_chain = writer_prompt | llm | StrOutputParser()
```
The prompt explicitly enforces an academic structure:
- **Introduction**
- **Key Findings** (minimum 3 well-explained points)
- **Conclusion**
- **Sources** (complete list of URLs gathered)

#### 4. The Critic Chain (LCEL)
To ensure output quality, I implemented an automated review step inspired by human peer-review workflows:
```python
critic_chain = critic_prompt | llm | StrOutputParser()
```
The critic prompt instructs the model to act as a strict research lead and output a structured assessment:
- `Score: X/10`
- `Strengths:`
- `Areas to Improve:`
- `One line verdict:`

---

### Step 4: The Pipeline Orchestrator (`pipeline.py`)

With the individual agents and chains defined, `pipeline.py` acts as the conductor. 

I implemented `run_research_pipeline(topic: str) -> dict`:
1. **Initializes State:** Creates an empty `state = {}` dictionary.
2. **Step 1 (Search):** Prompts the Search Agent. It executes Tavily searches and appends results to `state["search_results"]`.
3. **Step 2 (Deep Reading):** Passes `state["search_results"][:800]` to the Reader Agent. It selects the best URL, scrapes it, and saves the text to `state["scraped_content"]`.
4. **Step 3 (Drafting):** Aggregates both the broad snippets and deep scraped text into a single context payload, invoking `writer_chain` to write `state["report"]`.
5. **Step 4 (Criticism):** Feeds the draft into `critic_chain`, attaching the final critique to `state["feedback"]`.
6. **Returns State:** The complete dictionary containing intermediate telemetry and final deliverables is returned.

I also added an interactive CLI entrypoint (`if __name__ == "__main__":`) with formatted console logging so I could test and debug the workflow directly from my terminal.

---

### Step 5: Building the UI (`app.py`)

To make the system user-friendly, I built a web application using **Streamlit**:

- **Custom Aesthetic:** Rather than using default Streamlit styles, I injected custom CSS featuring dark-mode backgrounds (`#0a0a0f`), subtle orange accents (`#ff8c32`), and modern typography using the Google Fonts `Syne`, `DM Mono`, and `DM Sans`.
- **Two-Column Responsive Layout:**
  - **Left Column:** Clean input card with topic submission and clickable example pills (*"LLM agents 2025"*, *"CRISPR gene editing"*, *"Fusion energy progress"*).
  - **Right Column:** Live visual pipeline cards that reactively update their state badges (`WAITING` ➔ `● RUNNING` ➔ `✓ DONE`).
- **Telemetry Expanders:** Provided collapsible inspection panels for raw Tavily search outputs and raw BeautifulSoup scraped text for transparency.
- **Report & Critique Panels:** Rendered the final research report in native Markdown alongside a green-accented Critic Feedback card.
- **Export Functionality:** Added an instant `st.download_button` allowing users to save the generated report as a timestamped `.md` file.

---

## 4. Engineering Challenges & What I Learned

### 1. Handling Anti-Scraping Defenses
**Problem:** Several major websites block automated Python requests with `403 Forbidden` or require client-side JavaScript execution (Cloudflare, CAPTCHAs).  
**Solution:** I added a browser User-Agent header, wrapped the scraper in an 8-second timeout, and handled exceptions cleanly. If a domain blocks the reader, the agent surfaces the error gracefully without terminating the pipeline, and the Writer Chain still generates a comprehensive report using the remaining Tavily snippets.

### 2. Balancing Context Window with Information Depth
**Problem:** Full-page HTML dumps waste thousands of tokens on navigation links, style definitions, and tracking scripts, rapidly inflating costs and diluting LLM attention.  
**Solution:** Combining BeautifulSoup tag decomposition with a strict 3,000-character cap struck the right balance—preserving rich factual content while keeping prompt tokens and latency low.

### 3. Avoiding Agent Hallucination in Multi-Agent Handoffs
**Problem:** In early iterations, the Writer Chain occasionally synthesized unsupported facts or hallucinated generic source links.  
**Solution:** Setting `temperature=0.0` and enforcing strict prompt templates—mandating that the model cite only the URLs provided in `research_combined`—completely eliminated fabricated references.

---

## 5. How to Run the Project

### CLI Mode (Terminal)
```bash
cd multi_agent_system
.venv\Scripts\activate       # On Linux/macOS: source .venv/bin/activate
python pipeline.py
```

### Web UI Mode (Streamlit Dashboard)
```bash
cd multi_agent_system
.venv\Scripts\activate
streamlit run app.py
```
Open **`http://localhost:8501`** in your browser, enter any research query, and watch the agents collaborate in real time.

---

## 6. Future Roadmap

- [ ] **Parallel Reading:** Upgrading the Reader Agent to scrape and summarize top 3 URLs concurrently using `asyncio`/`aiohttp`.
- [ ] **Iterative Self-Correction:** Allowing the Writer Chain to automatically revise its draft if the Critic Chain assigns a score below 7/10.
- [ ] **Vector Store Integration:** Persisting scraped content into ChromaDB or FAISS to support multi-turn conversational Q&A over past research sessions.
