# CLAUDE.md -- Building with the Claude API

## Project Overview

Companion code from the Anthropic "Building with the Claude API" course. 25 Jupyter notebooks + 1 CLI capstone project covering the Anthropic API, prompt engineering, tool use, RAG, and advanced features.

See [README.md](./README.md) for the full notebook index and setup instructions.

## File Structure

```bash
eza -TL 2 --icons --git-ignore
```

### Section Breakdown

- **01**: Intro -- basic API chat completion
- **02**: Evaluation -- model-graded prompt evaluation framework
- **03-04**: Prompt Engineering -- evaluation exercise (03 = blank, 04 = completed)
- **05-12**: Tools -- progressive build from schemas to agentic loops, multi-tool agents, streaming, text editor, and web search
- **13-19**: RAG -- chunking, embeddings (Voyage AI), vector DB (ChromaDB), BM25, hybrid search, reranking, contextual retrieval
- **20-25**: Features -- extended thinking, image analysis, PDF analysis, citations, prompt caching, code execution
- **26-cli-mcp-chat/**: Capstone MCP CLI chat client with document tools, resources, and prompts

### Key Relationships

- Notebooks 05-08 form a progressive sequence building the same tool-use agent
- Notebook 03 is the blank exercise version of 04
- Notebooks 13-19 build a RAG pipeline incrementally (chunking -> embeddings -> vector DB -> BM25 -> hybrid -> reranking -> contextual)

## Configuration

- `.env` at root: `ANTHROPIC_API_KEY` (not committed)
- `.env.example` at root and in `26-cli-mcp-chat/`: placeholder templates
- RAG notebooks require `VOYAGE_API_KEY` and optionally `COHERE_API_KEY`
- All notebooks use `python-dotenv` to load environment variables

## Common Commands

```bash
# Run notebooks
jupyter notebook

# Clear all notebook outputs
jupyter nbconvert --clear-output --inplace *.ipynb

# Run CLI capstone (from 26-cli-mcp-chat/)
uv run main.py
```

## Implementation Notes

- Notebook 21 (images) requires a user-supplied image file; `images.zip` is gitignored
- Notebook 25 (code execution) uses beta API headers for the sandbox feature
- The CLI project's `mcp_server.py` has a note about adding a "summarize" prompt as a future enhancement
- Supporting assets (PDFs, CSVs) live in `assets/`
