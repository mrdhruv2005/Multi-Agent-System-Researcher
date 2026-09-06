# ResearchMind — Multi-Agent Research Pipeline

A four-stage research assistant that takes a topic, searches the live web, reads the most
relevant source in depth, drafts a structured report, and then critiques its own output.

Built with LangChain tool-calling agents and LCEL chains over Mistral AI, with Tavily for
search. Deployed on Streamlit.

**[Live demo](https://multi-agent-system-researcher-jdg4vvoihvh5n89excwvhw.streamlit.app/)**

---

## How it works

Four stages run in sequence, each consuming the previous stage's output. `pipeline.py`
threads a `state` dictionary through all four:

```
topic
  │
  ├─▶ 1. SEARCH AGENT      tool-calling agent + Tavily
  │      └─ state["search_results"]      titles, URLs, snippets (top 5)
  │
  ├─▶ 2. READER AGENT      tool-calling agent + BeautifulSoup scraper
  │      └─ state["scraped_content"]     full text of the most relevant page
  │
  ├─▶ 3. WRITER CHAIN      LCEL: prompt | llm | StrOutputParser
  │      └─ state["report"]              structured Markdown report
  │
  └─▶ 4. CRITIC CHAIN      LCEL: prompt | llm | StrOutputParser
         └─ state["feedback"]            score /10, strengths, improvements, verdict
```

### Two agents, two chains — and the distinction matters

**Stages 1 and 2 are agents.** Each is built with `create_agent` and given a tool. The
model decides when to call its tool and with what arguments — the Reader agent, for
instance, is handed the search results and chooses which URL is worth scraping. That
decision is the model's, not the code's.

**Stages 3 and 4 are LCEL chains** (`prompt | llm | StrOutputParser`). They have no
tools and make no decisions about control flow. They are deterministic
prompt-in/text-out transformations, which is all they need to be.

Using an agent where a chain suffices adds latency and non-determinism for nothing. The
split here is deliberate: tools where a choice is required, chains where it isn't.

### Agents and tools

```python
llm = ChatMistralAI(model="mistral-small-latest", temperature=0)

def build_search_agent():
    return create_agent(model=llm, tools=[web_search])

def build_reader_agent():
    return create_agent(model=llm, tools=[scrape_url])
```

`temperature=0` throughout — for research synthesis, reproducibility matters more than
variety.

| Tool | Implementation |
|------|----------------|
| `web_search(query)` | Tavily API, `max_results=5`, returns title + URL + 300-char snippet per result |
| `scrape_url(url)` | `requests` with browser user-agent and 8s timeout, `BeautifulSoup` strips `script`/`style`/`nav`/`footer`, returns first 3,000 chars of clean text |

Both tools fail soft: `scrape_url` catches exceptions and returns an error string rather
than raising, so one dead link cannot kill the pipeline mid-run.

### The Critic

The Critic reviews the finished report and returns a fixed format:

```
Score: X/10

Strengths:
- ...

Areas to Improve:
- ...

One line verdict:
...
```

**The Critic's output is advisory — it is shown to the user, not fed back into the
Writer.** See [Limitations](#limitations); this is the most obvious next thing to build.

---

## Report structure

The Writer is prompted for a consistent shape:

- **Introduction**
- **Key Findings** — minimum three, each explained
- **Conclusion**
- **Sources** — every URL encountered during research

Because the Writer receives both the search snippets and the scraped page text, findings
trace back to material actually retrieved during this run rather than to the model's
training data.

---

## Architecture notes

**Search snippets are truncated before being handed to the Reader.**
`state['search_results'][:800]` — enough for the model to judge relevance, short enough
to leave context budget for the scraped page. Passing the full search output would crowd
out the content that actually matters.

**Search and scrape are separate stages.** Search returns breadth (five shallow
snippets); scraping returns depth (one full page). Merging them into one agent would
force a single tool to do both jobs and make the relevance decision implicit.

**Streamlit keep-awake.** `.github/workflows/keep-awake.yml` pings the deployed app on
a schedule so the free-tier Streamlit instance doesn't sleep between visits.

---

## Running it

```bash
git clone https://github.com/mrdhruv2005/Multi-Agent-System-Researcher.git
cd Multi-Agent-System-Researcher

python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Create a `.env`:

```env
MISTRAL_API_KEY=your_mistral_key
TAVILY_API_KEY=your_tavily_key
```

Then either the UI:

```bash
streamlit run app.py
```

or the CLI, which prints each stage as it completes:

```bash
python pipeline.py
```

---

## Repository structure

```
├── app.py                            # Streamlit UI
├── pipeline.py                       # Four-stage orchestration + CLI entry point
├── agents.py                         # Agent builders, writer chain, critic chain
├── tools.py                          # web_search (Tavily), scrape_url (BeautifulSoup)
├── requirements.txt
└── .github/workflows/keep-awake.yml  # Scheduled ping to prevent Streamlit sleep
```

---

## Limitations

**The Critic does not close the loop.** This is the main architectural gap. The Critic
scores the report and its feedback is displayed, but nothing routes that feedback back
into the Writer for revision. A genuine iterative refinement loop would re-invoke the
Writer with the critique attached and repeat until the score crossed a threshold or an
iteration cap was reached. As built, the pipeline is a linear sequence with a review
step at the end — not a feedback loop.

Implementing that properly is what a graph framework such as LangGraph is designed for:
conditional edges, cycles, and explicit state transitions. The current sequential
implementation would need restructuring to support it.

**Single source per report.** The Reader scrapes exactly one URL — the one it judges most
relevant. Search finds five, so four are used only as snippets. Scraping the top three
and synthesizing across them would produce better-grounded reports at roughly 3× the
scraping cost.

**No retrieval or embedding layer.** Scraped text goes straight into the prompt. There is
no chunking, no vector store, no semantic retrieval — so the amount of source material
usable per report is bounded by the model's context window rather than by what was found.

**Other constraints:**

- Truncation limits are fixed, not adaptive: 800 chars of search results, 3,000 chars of
  scraped text, 5 search results.
- No caching. The same topic re-runs every API call from scratch.
- No structured output validation. The Critic's score is parsed from free text, so a
  format deviation is not caught.
- No retry on LLM failure. `tenacity` is in `requirements.txt` but is not wired in.
- Report quality is not measured. There is no evaluation harness, so "is this report
  good?" is answered only by the Critic's own opinion.
- Scraping respects neither `robots.txt` nor rate limits — fine for personal research,
  not for anything running at scale.

---

## Tech stack

Python · LangChain (`create_agent`, LCEL, `ChatPromptTemplate`, `StrOutputParser`) ·
`langchain-mistralai` · Mistral AI (`mistral-small-latest`) · Tavily Search API ·
BeautifulSoup4 · Streamlit · python-dotenv · GitHub Actions
