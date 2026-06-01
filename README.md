# ResearchMind

An autonomous multi-agent research pipeline built with Streamlit and LangChain. 

You give it a topic, and it spins up four specialized agents to investigate, read, draft, and review a complete research report.

## How it works

The pipeline runs sequentially using LangChain and Mistral:
1. **Search Agent**: Uses Tavily to query the web and find recent, relevant sources.
2. **Reader Agent**: Scrapes the best URLs to extract in-depth content.
3. **Writer Chain**: Synthesizes the scraped data into a structured markdown report.
4. **Critic Chain**: Reviews the final draft, scoring it and suggesting improvements.

## Setup

1. Clone the repo:
   ```bash
   git clone https://github.com/mrdhruv2005/Multi-Agent-System-Researcher.git
   cd Multi-Agent-System-Researcher
   ```

2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

3. Set up your API keys:
   Create a `.env` file in the root directory and add your keys:
   ```env
   OPENAI_API_KEY=your_openai_key
   TAVILY_API_KEY=your_tavily_key
   MISTRAL_API_KEY=your_mistral_key
   ```
   *(Note: The codebase uses Mistral for the core LLM logic, but ensure you have the keys required by your specific LangChain setup).*

4. Run the app locally:
   ```bash
   streamlit run app.py
   ```

## Deployment

This app is set up for easy deployment on [Streamlit Community Cloud](https://streamlit.io/cloud). 
- **Python version**: 3.11 is recommended.
- **Secrets**: Add your `.env` variables to the Streamlit App Settings -> Secrets panel before deploying.
- **Sleep prevention**: A GitHub Action (`keep-awake.yml`) is included in this repo. It pings the app every hour so the free tier doesn't spin it down from inactivity. 

## License

MIT
