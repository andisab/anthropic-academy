# CLAUDE.md -- Anthropic API Course Collection

## Project Overview

Multi-course repository with companion code from Anthropic learning materials.

- **`building-with-claude-api/`** -- 25 Jupyter notebooks + 1 CLI capstone covering the Anthropic API, prompt engineering, tool use, RAG, and advanced features
- **`claude+cloud/`** -- 2 notebooks on using Claude through AWS Bedrock and Google Vertex AI

See [building-with-claude-api/README.md](./building-with-claude-api/README.md) for the full notebook index and setup instructions.

## File Structure

```bash
eza -TL 2 --icons --git-ignore
```


## building-with-claude-api/

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

### Configuration
- `building-with-claude-api/.env`: `ANTHROPIC_API_KEY` (not committed)
- `building-with-claude-api/.env.example` and `building-with-claude-api/26-cli-mcp-chat/.env.example`: placeholder templates
- RAG notebooks require `VOYAGE_API_KEY` and optionally `COHERE_API_KEY`
- All notebooks use `python-dotenv` to load environment variables

### Common Commands

```bash
# Run notebooks (from building-with-claude-api/)
jupyter notebook

# Clear all notebook outputs (from building-with-claude-api/)
jupyter nbconvert --clear-output --inplace *.ipynb

# Run CLI capstone (from building-with-claude-api/26-cli-mcp-chat/)
uv run main.py
```

### Implementation Notes
- Notebook 21 (images) requires a user-supplied image file; `images.zip` is gitignored
- Notebook 25 (code execution) uses beta API headers for the sandbox feature
- The CLI project's `mcp_server.py` has a note about adding a "summarize" prompt as a future enhancement
- Supporting assets (PDFs, CSVs) live in `building-with-claude-api/assets/`

### Pre-existing Path Notes
These are from the original course materials and are not caused by the restructure:
- RAG notebooks (13-19) reference `./report.md` -- this file is gitignored and user-supplied, never committed
- Notebook 23 references `earth.pdf` but the actual asset is `assets/earth-sample.pdf`
- Notebook 21 uses placeholder `"your-image.png"` -- intentional, documented in its markdown cell


## claude+cloud/

Two standalone notebooks demonstrating Claude access through cloud provider SDKs:

- **`anthropic+bedrock.ipynb`** -- AWS Bedrock integration using `anthropic[bedrock]` and `boto3`
- **`anthropic+vertex.ipynb`** -- Google Vertex AI integration using `anthropic[vertex]` and `google-cloud-aiplatform`

### Configuration
- No `.env` file needed -- both notebooks use cloud SDK authentication (AWS credentials via `boto3`, Google ADC via `gcloud`)
- `screenshots/` contains reference images for Bedrock/Vertex setup steps
