# Building with the Claude API

Companion code from the [Anthropic: Building with the Claude API](https://www.anthropic.com/learn) course. Contains 25 Jupyter notebooks covering API basics, prompt engineering, tool use, RAG pipelines, and advanced features, plus an MCP-based CLI chat capstone project.


## Notebook Index

| # | Section | Topic | Notebook |
|---|---------|-------|----------|
| 01 | Intro | Basic chat completions | `01-intro-basic-chat.ipynb` |
| 02 | Evaluation | Prompt grading framework | `02-eval-prompt-grader.ipynb` |
| 03 | Prompt Eng | Evaluation exercise (blank) | `03-prompteng-evaluation-exercise.ipynb` |
| 04 | Prompt Eng | Evaluation exercise (completed) | `04-prompteng-evaluation-completed.ipynb` |
| 05 | Tools | Schemas and first tool call | `05-tools-schemas-and-first-call.ipynb` |
| 06 | Tools | Agentic loop | `06-tools-agentic-loop.ipynb` |
| 07 | Tools | Multi-tool agent | `07-tools-multi-tool-agent.ipynb` |
| 08 | Tools | Tool choice parameter | `08-tools-tool-choice.ipynb` |
| 09 | Tools | Structured data extraction | `09-tools-structured-data.ipynb` |
| 10 | Tools | Streaming with tools | `10-tools-streaming.ipynb` |
| 11 | Tools | Text editor tool | `11-tools-text-editor.ipynb` |
| 12 | Tools | Web search tool | `12-tools-web-search.ipynb` |
| 13 | RAG | Text chunking strategies | `13-rag-chunking.ipynb` |
| 14 | RAG | Embeddings with Voyage AI | `14-rag-embeddings.ipynb` |
| 15 | RAG | Vector database (ChromaDB) | `15-rag-vector-db.ipynb` |
| 16 | RAG | BM25 keyword search | `16-rag-bm25.ipynb` |
| 17 | RAG | Hybrid search | `17-rag-hybrid-search.ipynb` |
| 18 | RAG | Reranking results | `18-rag-reranking.ipynb` |
| 19 | RAG | Contextual retrieval | `19-rag-contextual-retrieval.ipynb` |
| 20 | Features | Extended thinking | `20-feat-extended-thinking.ipynb` |
| 21 | Features | Image analysis | `21-feat-image-analysis.ipynb` |
| 22 | Features | PDF analysis | `22-feat-pdf-analysis.ipynb` |
| 23 | Features | Citations | `23-feat-citations.ipynb` |
| 24 | Features | Prompt caching | `24-feat-prompt-caching.ipynb` |
| 25 | Features | Code execution (sandbox) | `25-feat-code-execution.ipynb` |
| 26 | Capstone | MCP CLI chat client | [`26-cli-mcp-chat/`](./26-cli-mcp-chat/) |


## Setup

### Prerequisites

- Python 3.10+
- An [Anthropic API key](https://console.anthropic.com/)
- (Optional) A [Voyage AI API key](https://www.voyageai.com/) for the RAG/embeddings notebooks (14-19)

### Installation

```bash
# Clone and enter the repo
git clone https://github.com/andisab/building-with-claude-api.git
cd building-with-claude-api

# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install anthropic python-dotenv jupyter

# For RAG notebooks (13-19), also install:
pip install voyageai chromadb rank-bm25 cohere

# Copy and fill in your API key
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY

# Launch Jupyter
jupyter notebook
```

### CLI Capstone Project

See [`26-cli-mcp-chat/README.md`](./26-cli-mcp-chat/README.md) for setup and usage of the MCP chat client.


## Notes

- **`images.zip`** is excluded from the repository. Notebook 21 (image analysis) requires you to supply your own image file.
- **`COURSE_NOTES.md`** contains formatted notes from the course lectures.
- Supporting files (PDFs, CSVs) are in the `assets/` directory.
