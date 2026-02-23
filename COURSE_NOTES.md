>[toc]


## Building w/Claude API
Notes from a video course on building with the Claude language model API. Covers API fundamentals, prompt engineering, tool use, RAG pipelines, MCP, Claude Code, computer use, and agent/workflow patterns.


## Overview
Claude's model families, capabilities, and selection criteria.

### Claude Model Families
---
Claude has three model families optimized for different priorities:

- **Opus** — Highest intelligence for complex, multi-step tasks requiring deep reasoning and planning. Trade-off: higher cost and latency.
- **Sonnet** — Balanced intelligence, speed, and cost efficiency. Strong coding abilities and precise code editing. Best for most practical use cases.
- **Haiku** — Fastest model optimized for speed and cost. No reasoning capabilities like Opus/Sonnet. Best for real-time interactions and high-volume processing.

**Selection framework**: Intelligence priority → Opus. Speed priority → Haiku. Balanced requirements → Sonnet. Common approach: use multiple models in the same application based on specific task requirements rather than single model selection.

All models share core capabilities: text generation, coding, image analysis. Main difference is optimization focus.


## API Fundamentals
Core mechanics of interacting with the Claude API: authentication, requests, conversations, streaming, and configuration.

### Accessing the API
---
API access flow is a 5-step process from user input to response display:

1. **Client → Server**: Client sends user text to developer's server. Never access the Anthropic API directly from client apps — keep the API key secret.
2. **Server → Anthropic API**: Server makes request using SDK (Python, TypeScript, JavaScript, Go, Ruby) or plain HTTP. Required parameters: API key, model name, messages list, `max_tokens` limit.
3. **Text generation** has 4 stages:
  - *Tokenization* — breaking input into tokens (words, word parts, symbols, spaces)
  - *Embedding* — converting tokens to number lists representing all possible word meanings
  - *Contextualization* — adjusting embeddings based on neighboring tokens to determine precise meaning
  - *Generation* — output layer produces probabilities for next word, model selects using probability + randomness, adds selected word, repeats
4. **Stop condition**: Model stops when `max_tokens` reached or special `end_of_sequence` token generated.
5. **Response**: API returns generated text + usage counts + `stop_reason` to server, server sends to client for display.

### Making a Request
---
Setup steps:

1. Install packages: `pip install anthropic python-dotenv`
2. Store API key in `.env` file with `ANTHROPIC_API_KEY="your_key"` (ignore in version control)
3. Load environment variable using `python-dotenv`
4. Create client: initialize `anthropic` client and define model variable

API request structure uses `client.messages.create()` with required arguments: `model`, `max_tokens`, `messages`. Messages are a list of dictionaries with `"role"` (user/assistant) and `"content"` fields.

```python
message = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=[{"role": "user", "content": "What is quantum computing?"}]
)
# Full response contains metadata; extract text with:
text = message.content[0].text
```

### Multi-Turn Conversations
---
Key limitation: Anthropic API stores no messages. Each request is independent with no memory of previous exchanges.

Solution requires two steps:
1. Manually maintain a message list in code
2. Send entire conversation history with every follow-up request

Message structure: list of dictionaries with `"role"` (user/assistant) and `"content"` fields. Helper functions needed:

- `add_user_message(messages, text)` — appends user message to history
- `add_assistant_message(messages, text)` — appends assistant response to history
- `chat(messages)` — sends message history to API and returns response

Without message history, responses lack context and continuity. With complete history, Claude maintains conversation context and provides relevant follow-ups.

### System Prompts
---
System prompts customize Claude's response style and tone by assigning a specific role or behavior pattern. Passed as a plain string via the `system` keyword argument.

Purpose: control *how* Claude responds rather than *what* it responds. Example: a math tutor role makes Claude give hints instead of direct answers.

Structure: first line typically assigns role (`"You are a patient math tutor"`), followed by specific behavioral instructions.

Technical implementation: create a params dictionary, conditionally add the `system` key if a prompt is provided, pass params to `create()` with `**` unpacking. Handle `None` case by excluding the system parameter entirely.

### Temperature
---
Temperature is a parameter (0–1) controlling randomness in text generation by influencing token selection probabilities.

Generation process: input text → tokenization → probability assignment to possible next tokens → token selection → repeat.

- **Temperature 0** — deterministic output, always selects highest-probability token
- **Higher temperature** — increases chance of selecting lower-probability tokens, more creative/unexpected outputs

Usage guidelines: low temperature (near 0) for data extraction and factual tasks requiring consistency; high temperature (near 1) for creative tasks like brainstorming, writing, jokes, marketing. Higher values don't guarantee different outputs, just increase probability of variation.

### Response Streaming
---
Streaming displays AI responses chunk-by-chunk as they're generated instead of waiting for the complete response. Solves the 10–30 second wait problem.

How it works: server sends user message to Claude → Claude immediately sends acknowledgment → stream of events follows, each containing text chunks → server forwards chunks to frontend for real-time display.

Event types:
- `message_start` — initial acknowledgment
- `content_block_start` — text generation begins
- `content_block_delta` — contains actual text chunks (most important)
- `content_block_stop` / `message_stop` — generation complete

Implementation:
- Basic: `client.messages.create(stream=True)` returns event iterator
- Simplified: `client.messages.stream()` with `.text_stream` property extracts just text
- Final message: `stream.get_final_message()` assembles all chunks for storage


## Controlling Output
Techniques beyond prompt modification for steering Claude's responses: pre-filling, stop sequences, and structured data extraction.

### Pre-filling Assistant Messages
---
Manually adding an assistant message at the end of the conversation steers the response direction. Claude sees it as already-authored content and continues from the exact endpoint.

How it works: assemble messages list with user prompt + manual assistant message → Claude continues response from exact end of pre-filled text → must stitch together pre-fill + generated response.

Example: pre-fill `"Coffee is better because"` → Claude continues with justification for coffee.

### Stop Sequences
---
Force Claude to halt generation when a specific string appears. Provide a stop sequence string in the chat function; when Claude generates that exact string, the response immediately stops. The generated stop sequence text is *not* included in the final output.

Example: prompt `"count 1 to 10"` + stop sequence `"five"` → output stops at `"four, "` (five not included). Refinement: stop sequence `", five"` → clean output `"one, two, three, four"`.

Both pre-filling and stop sequences provide precise control over response direction and length without changing core prompts.

### Structured Data Generation
---
Combines assistant message pre-filling + stop sequences to get raw output without Claude's natural explanatory headers/footers.

Problem: Claude automatically adds markdown formatting, headers, commentary when generating JSON/code/structured content.

Solution pattern:
1. User message = request for structured data
2. Assistant message pre-fill = opening delimiter (e.g., ` ```json `)
3. Stop sequence = closing delimiter (e.g., ` ``` `)

Claude sees the pre-filled message, assumes it already started the response, generates only requested content, stops when hitting the delimiter. Result: raw structured data output with no extra formatting. Works for any structured data type (JSON, Python code, lists, etc.).


## Prompt Engineering & Evaluation
Systematic approaches to writing, testing, and iterating on prompts for reliable, high-quality outputs.

### Prompt Evaluation Overview
---
Prompt engineering = techniques for writing/editing prompts to help Claude understand requests and desired responses.

Three paths after writing a prompt:
1. Test once/twice, deploy to production (**trap**)
2. Test with custom inputs, minor tweaks for corner cases (**trap**)
3. Run through evaluation pipeline for objective scoring (**recommended**)

Key takeaway: engineers commonly under-test prompts. Use evaluation pipelines to get objective performance scores before iterating and deploying.

### A Typical Eval Workflow
---
A 6-step iterative process for prompt improvement:

1. **Write initial prompt draft** — create baseline prompt to optimize
2. **Create evaluation dataset** — collection of test inputs (3 examples or thousands, hand-written or LLM-generated)
3. **Generate prompt variations** — interpolate each dataset input into prompt template
4. **Get LLM responses** — feed each prompt variation to Claude, collect outputs
5. **Grade responses** — use grader system to score each response (e.g., 1–10 scale), average scores for overall prompt performance
6. **Iterate** — modify prompt based on scores, repeat entire process, compare versions

No standard methodology exists. Many open-source/paid tools available. Can start simple with custom implementation.

### Generating Test Datasets
---
Goal: build prompt + generate test dataset + evaluate performance. Example: AWS code assistance prompt that outputs only Python, JSON config, or regex without explanations.

Dataset generation approaches: manual assembly or automated with Claude (use faster models like Haiku for generation). Dataset structure: array of JSON objects with `task` property describing user requests.

Generation process: prompt Claude to create test cases → use pre-filling with assistant message ` ```json ` → set stop sequence ` ``` ` → parse response as JSON → save to file. `generate_dataset()` function sends prompt to Claude, gets structured JSON response of test tasks, saves to `dataset.json`.

### Running the Eval
---
Eval execution: merging test cases with prompts, running through LLM, and grading outputs.

Three core functions:
- `run_prompt` — merges test case with prompt, sends to Claude, returns output
- `run_test_case` — calls `run_prompt`, grades result, returns summary dictionary
- `run_eval` — loops through dataset, calls `run_test_case` for each, assembles results

Basic prompt structure: `"Please solve the following task: [test_case_task]"` (v1 starting point). Current limitations: no output formatting instructions, hardcoded scoring (`score=10`), verbose Claude responses. Output format: array of objects containing Claude output, original test case, and score.

### Model-Based Grading
---
Evaluation system that takes model outputs and assigns objective scores (typically 1–10 scale, 10 = highest quality).

Three grader types:
- **Code graders** — programmatic checks (length, word presence, syntax validation, readability scores)
- **Model graders** — additional API call to evaluate original model output; highly flexible for quality/instruction-following assessment
- **Human graders** — person evaluates responses; most flexible but time-consuming and tedious

Implementation pattern for model graders: create detailed prompt requesting strengths/weaknesses/reasoning/score (not just score alone to avoid default middling scores) → use JSON response format with pre-filled assistant message and stop sequences → parse returned JSON for score and reasoning → calculate average scores across test cases.

### Code-Based Grading
---
Automated validation system for LLM outputs containing code, JSON, or regex.

Core implementation:
- `validate_json()` — attempts JSON parsing, returns 10 if valid, 0 if error
- `validate_python()` — attempts AST parsing, returns 10 if valid, 0 if error
- `validate_regex()` — attempts regex compilation, returns 10 if valid, 0 if error

Dataset must include a `"format"` key specifying expected output type (JSON/Python/RegEx). Prompt instructs model to respond only with raw code/JSON/regex — no comments, explanations, or commentary. Use pre-filled assistant message with code blocks and stop sequences.

Final score = `(model_score + syntax_score) / 2`, combining semantic evaluation with syntax validation.

### Prompt Engineering Techniques
---
Module structure: start with initial poor prompt → apply techniques step-by-step → evaluate improvements → observe performance gains.

Example goal: generate one-day meal plan for athletes based on height, weight, physical goal, dietary restrictions.

Technical setup: updated eval pipeline with flexible prompt evaluator class supporting concurrency (`max_concurrent_tasks`). Key components: `prompt_input_spec` dictionary defining required inputs, `extra_criteria` for validation, `output.html` formatted evaluation report.

Process: write initial prompt → interpolate test case inputs → run evaluation → apply engineering techniques → re-evaluate → repeat. Initial results: expect poor scores (~2.32) with basic prompts.

#### Being Clear and Direct
Use simple, direct language with action verbs in the first line of prompts to specify the exact task. Structure: action verb + clear task description + output specifications.

Examples: `"Write three paragraphs about how solar panels work"`, `"Generate a one day meal plan for an athlete..."`. Result: improved prompt performance (score increase from 2.32 to 3.92).

#### Being Specific
Adding guidelines or steps to direct model output in a particular direction.

- **Type A (Attributes)** — list qualities/attributes desired in output (length, structure, format)
- **Type B (Steps)** — provide specific steps for model to follow in reasoning

Type A controls output characteristics. Type B controls how model arrives at answer. Both often combined. Use Type A for almost all prompts; Type B for complex problems needing broader perspective. Example: meal planning prompt score jumped from 3.92 to 7.86 with guidelines.

#### XML Tags for Structure
Using XML tags to organize different content sections within prompts. Wrapping content in descriptive tags like `<sales_records>` or `<my_code>` helps Claude distinguish between different types of information. Use descriptive, specific tag names.

Benefits: makes prompt structure obvious, reduces confusion about content boundaries, improves output quality even for smaller content blocks.

#### Providing Examples
One-shot/multi-shot prompting: providing examples in prompts to guide model behavior. One-shot = single example, multi-shot = multiple examples.

Structure examples with XML tags containing sample input and ideal output. Key applications: corner case handling (sarcasm detection), complex output formatting (JSON structures), clarifying expected response quality/style.

Best practices: add context for corner cases, include reasoning explaining why output is ideal, use highest-scoring examples from evaluations as templates, place examples after main instructions.


## Tool Use
Enabling Claude to access external information and perform actions through function calling, schemas, and multi-turn tool orchestration.

### Introduction to Tool Use
---
Tool use enables Claude to access external information beyond training data. Default limitation: Claude only knows information from training data, lacks current/real-time information.

Tool use flow:
1. Send initial request to Claude + instructions for external data access
2. Claude evaluates if external data needed, requests specific information
3. Server runs code to fetch requested data from external sources
4. Send follow-up request to Claude with retrieved data
5. Claude generates final response using original prompt + external data

Example: user asks current weather → Claude requests weather data → server calls weather API → Claude receives data → Claude provides informed response.

### Project Overview: Reminder System
---
Goal: teach Claude to set time-based reminders through tool implementation.

Target interaction: User: `"Set reminder for doctor's appointment, week from Thursday"` → Claude: `"I will remind you at that point in time"`

Three core problems requiring tools:
1. **Time knowledge gap** — Claude knows current date but not exact time
2. **Time calculation errors** — Claude sometimes miscalculates time-based addition
3. **No reminder mechanism** — Claude understands the concept but lacks implementation

Three corresponding tools: current datetime tool, duration addition tool, reminder setting tool.

### Tool Functions
---
Tool functions are plain Python functions called by Claude when it determines additional data is needed. Must use descriptive function names and argument names. Should validate inputs and raise errors with meaningful messages (error messages are visible to Claude, allowing retry with corrected parameters).

```python
def get_current_datetime(date_format="%Y%m%d %H:%M:%S"):
    if not date_format:
        raise ValueError("date format cannot be empty")
    return datetime.now().strftime(date_format)
```

Workflow: Claude identifies need for information → calls tool function → receives result or error → may retry with corrections.

### Tool Schemas
---
Tool schemas are JSON schema specifications describing tool functions and their parameters for Claude.

Schema structure:
- `name` — tool identifier
- `description` — 3–4 sentences: what it does, when to use, what data it returns
- `input_schema` — JSON schema describing function arguments with types and descriptions

Schema generation trick: take tool function to Claude.ai → prompt: `"write valid JSON schema spec for tool calling for this function, follow best practices in attached documentation"` → attach Anthropic API docs → copy generated schema. Wrap schema dictionary with `ToolParam()` to prevent type errors.

### Handling Message Blocks
---
When tools are enabled, messages contain multiple blocks instead of just text blocks.

Tool response format: assistant message with text block (user-facing explanation) + tool use block (function name + arguments for tool execution).

Critical requirement: manually maintain conversation history. Multi-block handling: append entire `response.content` (all blocks) to messages list, not just text. Helper functions must support multiple blocks.

### Sending Tool Results
---
After executing the tool function, send results back to Claude in a follow-up request.

Tool result block structure:
- `tool_use_id` — matches ID from original tool use block to pair requests with results
- `content` — tool function output converted to string (usually JSON)
- `is_error` — boolean flag for function execution errors (default `false`)

Follow-up request requirements: include complete message history (original user message + assistant tool use message + new user message with tool result). Must include original tool schemas even if not using tools again. Tool result block goes in user message, not assistant message.

### Multi-Turn Conversations with Tools
---
Conversations where Claude uses multiple tools sequentially to answer a single query.

Tool chaining process: user asks question → Claude requests first tool → tool executed → result returned → Claude requests second tool → tool executed → result returned → Claude provides final answer.

Implementation: `while` loop that continues calling Claude until no more tool requests, checking each response for `tool_use` blocks. `run_conversation` function takes initial messages, loops through Claude calls, executes requested tools, adds results to conversation, continues until final response.

### Implementing Multiple Turns
---
`stop_reason` field indicates why Claude stopped generating text. `stop_reason = "tool_use"` means Claude wants to call a tool.

**`run_conversation` function**: calls Claude with messages + available tools → adds assistant response to history → checks `stop_reason` → if not `"tool_use"`, breaks loop → if `tool_use`, calls `run_tools` → adds tool results as user message → repeats.

**`run_tools` function**: filters `message.content` for blocks with `type="tool_use"` → iterates through each tool request → runs appropriate tool function via `run_tool` helper → creates `tool_result` blocks with `type`, `tool_use_id`, `content` (JSON-encoded output), `is_error` → returns list of all tool result blocks.

**`run_tool` function**: dispatcher using if statements to match tool names to functions. Error handling: success → `is_error=false`, content=tool_output; failure → `is_error=true`, content=error_message.

### Using Multiple Tools
---
After initial framework setup, adding new tools is a simple pattern: schema + routing + implementation.

Process: (1) add tool schemas to `run_conversation`'s tools list, (2) add conditional cases in `run_tool` to handle new tool names, (3) implement actual tool functions.

Tool chaining: AI can use multiple tools sequentially in a single conversation (e.g., calculate date first, then set reminder with result). Assistant responses can contain multiple blocks: text blocks + tool use blocks in same message.

### The Batch Tool
---
Enables Claude to run multiple tools in parallel within a single assistant message instead of separate sequential requests.

Problem: Claude can technically send multiple tool use blocks in one message but rarely does in practice.

Solution: create a batch tool schema that takes a list of invocations (each containing tool name + arguments). Claude calls the batch tool with an array of desired tool executions. `run_batch` function iterates through invocations, extracts tool name and JSON-parsed arguments, calls `run_tool` for each, returns `batch_output` list. Result: single request-response cycle instead of multiple sequential rounds.

### Tools for Structured Data Extraction
---
Alternative to prompt-based extraction: define a JSON schema for a tool where inputs = desired data structure → send prompt + schema → Claude calls tool with structured arguments → extract JSON from tool use block (no tool result needed).

Critical: force tool calling using `tool_choice` parameter: `tool_choice = {"type": "tool", "name": "your_tool_name"}`. This ensures Claude always calls the specified tool.

Access structured data from `response.content[0].input`. Prompt-based methods are simpler for quick extractions; tools are better for complex/reliable extractions.

### Fine-Grained Tool Calling
---
Tool streaming: standard streaming returns `content_block_delta` events; tool streaming adds `input_json_delta` events with `partial_json` (chunk) and `snapshot` (cumulative sum).

Default behavior: Claude generates JSON chunks → API buffers until complete top-level key-value pair → validates JSON against schema → sends chunks. Results in delays followed by burst of chunks.

Fine-grained mode (`fine_grained: true`): disables API-side JSON validation → sends chunks immediately as generated → provides traditional streaming experience → requires client-side error handling for invalid JSON.

Trade-offs: default = slower but validated JSON; fine-grained = faster streaming but potential invalid JSON.

### The Text Edit Tool
---
Built-in Claude tool for file/text operations (read, write, create, replace, undo). Only the JSON schema is built into Claude; implementation must be custom-coded.

Schema stub sent to Claude gets auto-expanded to full schema. Schema type string varies by Claude model version (3.5 vs 3.7 have different dates).

Workflow: send minimal schema stub → Claude expands internally → Claude sends tool use requests → custom implementation executes file operations → results sent back. Enables replicating AI code editor functionality through API calls.

### The Web Search Tool
---
Built-in Claude tool for searching the web. No custom code needed — Claude handles search execution automatically.

Schema requirements: `type: "web_search_20250305"`, `name: "web_search"`, `max_uses`: number (default 5), `allowed_domains`: optional list to restrict search.

Response structure: text blocks (Claude's explanatory text), tool use blocks (search queries), web search result blocks (title, URL), citation blocks (specific text supporting statements).

Use case: restricting to `nih.gov` for medical advice ensures scientifically-backed information.


## RAG (Retrieval Augmented Generation)
Techniques for querying large documents by chunking, embedding, and retrieving relevant context for LLM prompts.

### Introduction to RAG
---
RAG = technique for querying large documents using language models. Problem: how to extract specific information from 100–1000+ page documents without hitting context limits.

**Option 1 (Direct approach)**: place entire document text directly into prompt. Limitations: hard token limits, decreased effectiveness with longer prompts, higher costs, slower processing.

**Option 2 (RAG approach)**: two-step process — (1) break document into small chunks, (2) for user questions, find most relevant chunks and include only those in prompt.

\+ Scales to large/multiple documents
\+ Smaller prompts, lower costs, faster processing
\+ Model focuses on relevant content
\- More complexity, requires preprocessing
\- Needs search mechanism to find relevant chunks
\- No guarantee chunks contain complete context

### Text Chunking Strategies
---
Chunking quality directly impacts RAG performance. Poor chunking leads to irrelevant context retrieval.

**1. Size-Based Chunking** — dividing text into equal-length strings.
\+ Easy to implement, most common in production
\- Cut-off words, lacks context
Overlap strategy: include characters from neighboring chunks to preserve context. Creates text duplication but improves chunk meaning.

**2. Structure-Based Chunking** — dividing based on document structure (headers, paragraphs, sections). Best for structured documents (markdown, HTML). Limitation: requires guaranteed document formatting.

**3. Semantic-Based Chunking** — using NLP to group related sentences/sections. Most advanced technique. Groups consecutive sentences based on semantic similarity. Complex implementation.

Rule: no universal best chunking method — depends on document structure guarantees and specific use case.

### Text Embeddings
---
Text embeddings = numerical representation of text meaning generated by embedding models. An embedding model takes text input and outputs a long list of numbers (range -1 to +1). Each number theoretically scores different aspects (happiness, topic relevance, etc.) but actual meaning is unknown.

Semantic search uses text embeddings to find text chunks related to user questions in RAG pipelines. Enables semantic similarity matching rather than keyword matching.

RAG pipeline process: extract text chunks → user submits query → find related chunks using semantic search → add relevant chunks as context to prompt.

Implementation: Anthropic recommends Voyage AI for embedding generation. Requires separate account/API key. Free to start, easy integration via SDK.

### The Full RAG Flow
---
A 7-step process combining text chunking, embeddings, and vector search:

1. **Text Chunking** — split source documents into separate text pieces
2. **Generate Embeddings** — convert text chunks into numerical vectors using embedding models
3. **Normalization** — scale vector magnitudes to 1.0 (handled automatically by embedding APIs)
4. **Vector Database Storage** — store embeddings in specialized database optimized for numerical vector operations
5. **Query Processing** — convert user question into embedding using same model
6. **Similarity Search** — find most similar stored embeddings using cosine similarity
7. **Prompt Assembly** — combine user question with retrieved relevant text chunks, send to LLM

Key math: **cosine similarity** = cosine of angle between vectors, returns values -1 to 1, closer to 1 means more similar. **Cosine distance** = 1 minus cosine similarity, values closer to 0 mean higher similarity.

Process flow: pre-processing (steps 1–4) → user query → real-time retrieval (steps 5–7) → LLM response.

### Implementing the RAG Flow
---
Practical walkthrough of the 5-step retrieval process:

1. **Text Chunking** — split document into sections using `chunk_by_section` function
2. **Embedding Generation** — create vector representations for each chunk via `generate_embedding` (supports single string or list)
3. **Vector Store Population** — create vector index instance, loop through chunk-embedding pairs using `zip()`, store each pair with `store.add_vector(embedding, {content: chunk})`
4. **Query Processing** — user asks question, generate embedding for user query
5. **Similarity Search** — use `store.search(user_embedding, 2)` to find 2 most relevant chunks; returns results with cosine distances

Key: store original text with embeddings for meaningful retrieval results.

### BM25 Lexical Search
---
BM25 (Best Match 25) = lexical search algorithm commonly used in RAG pipelines to complement semantic search.

Problem with semantic search alone: can miss exact term matches, returning irrelevant results even when specific terms appear frequently in certain documents.

Hybrid search approach: combine semantic search (embeddings/vector database) with lexical search (BM25) in parallel, then merge results.

BM25 algorithm steps:
1. Tokenize user query into separate terms
2. Count frequency of each term across all text chunks
3. Assign relative importance based on usage frequency (rare terms = higher importance, common terms like `"a"` = lower importance)
4. Rank text chunks by how often they contain higher-weighted terms

Both semantic and lexical search systems use similar APIs (`add_document`, `search`), making them easy to combine.

### Multi-Index RAG Pipeline
---
System combining semantic search (vector index) and lexical search (BM25 index) for improved retrieval accuracy. A `Retriever` class wraps both indexes and merges results.

**Reciprocal Rank Fusion (RRF)** = technique for merging results from different indexes. Formula: `RRF_score = sum of (1/(rank + 1))` across all search methods for each document. Documents ranked by highest combined score.

Example: vector search returns `[doc2, doc7, doc6]`, BM25 returns `[doc6, doc2, doc7]`. After RRF calculation, final ranking becomes `[doc2, doc6, doc7]` because doc2 ranked high in both methods.

\+ Improved search accuracy by combining different paradigms
\+ Modular design with standardized API (`search()` and `add_document()` methods)
\+ Easy to extend with additional search indexes

### Reranking Results
---
Post-processing step using an LLM to reorder search results by relevance after initial retrieval.

Process: run vector + BM25 search → merge results → pass to LLM with prompt asking to rank documents by relevance → get reordered results. Use document IDs instead of full text for efficiency. Assistant message pre-fill + stop sequence ensures structured JSON output.

Trade-offs: increases search accuracy by leveraging LLM's understanding of semantic relevance; increases latency due to additional LLM call. Particularly effective when initial retrieval methods miss nuanced query intent (e.g., `"ENG team"` vs `"engineering team"`).

### Contextual Retrieval
---
Technique to improve RAG accuracy by adding context to document chunks *before* embedding.

Problem: when documents are split into chunks, individual chunks lose context from the original document. Solution: pre-processing step that adds contextual information to each chunk before inserting into retriever database.

Process:
1. Take individual chunk + original source document
2. Send to LLM with prompt asking to generate situating context
3. LLM generates brief context explaining chunk's relationship to larger document
4. Join generated context with original chunk = "contextualized chunk"
5. Use contextualized chunk as input to vector/BM25 indexes

**Large document handling**: if source document is too large for single prompt, use selective context strategy: include starter chunks (1–3) from document beginning for summary/abstract, plus chunks immediately before target chunk for local context, skip middle chunks.


## Advanced Features
Extended thinking, vision, PDF support, citations, prompt caching, and code execution capabilities.

### Extended Thinking
---
Extended thinking allows Claude reasoning time before generating the final response. Displays a separate thinking process visible to users. Increases accuracy for complex tasks but adds cost (charged for thinking tokens) and latency.

- `thinking_budget` — minimum 1024 tokens allocated for thinking phase
- `max_tokens` must exceed thinking budget (e.g., budget 1024 requires `max_tokens ≥ 1025`)

Response structure: **thinking block** (reasoning text + cryptographic signature) + **text block** (final response). Signature prevents tampering with thinking text (safety measure). Redacted thinking blocks contain encrypted text flagged by safety systems, provided for conversation continuity.

When to use: enable after prompt optimization fails to achieve desired accuracy. Use prompt evals to determine necessity.

### Image Support
---
Claude can process images within user messages for analysis, comparison, counting, and description tasks.

Limitations: max 100 images per request; size/dimension restrictions apply; images consume tokens (charged based on pixel height/width).

Image block structure: special block type within user messages holding either raw image data (base64) or URL reference. Multiple image blocks allowed per message.

Critical success factor: strong prompting techniques required for accurate results. Techniques include step-by-step analysis instructions, one-shot/multi-shot examples, clear guidelines and verification steps, structured analysis frameworks.

Implementation: base64 encode image data → create message with image block (`type: image`, `source: base64`, `media_type`, `data`) followed by text block containing detailed prompt instructions.

### PDF Support
---
Claude can read PDF files directly using similar code to image processing. Key changes: file type = `"document"` instead of `"image"`, media type = `"application/pdf"` instead of `"image/png"`. Claude can read text, images, charts, tables, and mixed content from PDFs.

### Citations
---
Citations allow Claude to reference source documents and show where information comes from.

Citation types:
- `citation_page_location` — for PDF documents: document index, title, start/end page, cited text
- `citation_char_location` — for plain text: character position in text block

Implementation: add `"citations": {"enabled": true}` to request + add `"title"` field to identify source document. Works with both PDF files and plain text sources.

Response structure: content becomes list of text blocks, some containing `citations` arrays with location data. Enables citation popups/overlays showing source document, page numbers, and exact cited text.

### Prompt Caching
---
Speeds up Claude's responses and reduces costs by reusing computational work from previous requests.

Normal flow: user sends message → Claude processes input → generates output → **discards all processing work** → ready for next request. Problem: follow-up requests with identical content repeat all computational work.

Solution: prompt caching stores results of input processing in temporary cache. Identical input in subsequent requests → Claude retrieves cached work instead of reprocessing.

#### Rules of Prompt Caching
- Cache duration = 1 hour maximum
- Requires manual cache breakpoint addition to message blocks
- Text block must use longhand format: `content = [{"type": "text", "text": "content", "cache_control": {...}}]`
- Cache scope = all content up to and including breakpoint
- Any change before breakpoint invalidates entire cache
- Content processing order: tools → system prompt → messages
- Maximum breakpoints = 4 per request
- Minimum cache threshold = 1024 tokens

#### Prompt Caching Implementation
Modify chat function to enable caching by default for tools and system prompts.

**Tool schema caching**: add `cache_control` field with type `"ephemeral"` to last tool in list. Best practice: create copy of tools list, clone last schema, add cache control.

**System prompt caching**: wrap system prompt in text block dictionary with `cache_control` type `"ephemeral"`.

Token usage patterns: `cache_creation_input_tokens` (written to cache on first use), `cache_read_input_tokens` (retrieved from cache on subsequent requests). Partial cache reads possible when some content matches.

### Code Execution and the Files API
---
**Files API**: upload files ahead of time and reference them later via file ID instead of including raw data in each request. Upload file → get metadata object with ID → use ID in future requests.

**Code Execution**: server-based tool where Claude executes Python code in isolated Docker containers. No implementation needed, just include predefined tool schema. Claude can run code multiple times, interpret results, generate final response.

Key constraints: Docker containers have no network access; data input/output relies on Files API integration.

Combined workflow: upload file via Files API → get file ID → include ID in container upload block → ask Claude to analyze → Claude writes/executes code with access to uploaded file → returns analysis and results. Claude can generate files (plots, reports) inside the container that can be downloaded using file IDs.


## MCP (Model Context Protocol)
A communication layer providing Claude with context and tools through standardized client-server architecture.

### Introduction to MCP
---
MCP = communication layer providing Claude with context and tools without requiring developers to write tedious code.

Architecture: MCP client connects to MCP server. Server contains tools, resources, and prompts as internal components.

Problem solved: eliminates burden of authoring/maintaining numerous tool schemas and functions for service integrations. Example: a GitHub chatbot would require implementing tools for repositories, pull requests, issues, projects — significant developer effort. MCP server handles tool definition and execution instead.

Key benefits: developers avoid writing tool schemas and function implementations themselves. MCP and tool use are complementary — MCP handles *who* does the work (server vs developer), both still involve tools.

### MCP Clients
---
MCP client = communication interface between your server and MCP server, providing access to server's tools. Transport agnostic: client/server can communicate via multiple protocols (stdio, HTTP, WebSockets). Common setup: client and server on same machine using standard I/O.

Key message types:
- `list tools request` / `list tools result` — client asks server for available tools
- `call tool request` / `call tool result` — client asks server to run tool with arguments

Typical flow: user queries server → server requests tool list via MCP client → MCP server responds → server sends query + tools to Claude → Claude requests tool execution → MCP client sends call tool request → MCP server executes (e.g., GitHub API call) → results flow back through chain.

### MCP Project Setup
---
CLI-based chatbot project teaching MCP client-server interaction through hands-on implementation.

Project components: MCP client (connects to custom MCP server), MCP server (provides 2 tools: read document, update document), document collection (fake documents stored in memory only).

Key distinction: normal projects implement either client OR server, not both. Setup: download starter code → follow `readme.md` → add API key to `.env` → install dependencies → run with `uv run main.py` or `python main.py`.

### Defining Tools with MCP
---
MCP Python SDK creates tools through decorators rather than manual JSON schemas. The `@mcp.tool` decorator auto-generates tool JSON schemas from Python function definitions.

Syntax: `@mcp.tool(name="tool_name", description="description")` + function with typed parameters using `Field()` for argument descriptions.

```python
@mcp.tool(name="read_doc_contents", description="Read document contents")
def read_doc_contents(doc_id: str = Field(description="Document identifier")):
    if doc_id not in docs:
        raise ValueError(f"Document {doc_id} not found")
    return docs[doc_id]
```

Required imports: `Field` from `pydantic` for parameter descriptions, `mcp` package for server and tool decorators.

### The Server Inspector
---
MCP Inspector = in-browser debugger for testing MCP servers without connecting to applications. Access: run `mcp dev [server_file.py]` → opens server on port → navigate to provided URL.

Interface: left sidebar has connect button → top menu shows resources/prompts/tools → tools section lists available tools → click tool to open right panel for manual testing. Testing workflow: connect → navigate to tools → select → input parameters → run → verify output.

### Implementing a Client
---
MCP Client = wrapper class around client session for resource cleanup and connection management. Client session = actual connection from MCP Python SDK, requires resource cleanup on close.

Key functions:
- `list_tools()` — `await self.session.list_tools()`, return `result.tools`
- `call_tool()` — `await self.session.call_tool(tool_name, tool_input)`

Usage flow: client gets tool definitions to send to Claude, then executes tools when Claude requests them. Other code calls client functions to interact with MCP server, enabling Claude to inspect/edit documents.

### Defining Resources
---
MCP resources allow servers to expose data to clients for read operations.

Resource types: **direct** (static URI like `"docs://documents"`) and **templated** (parameterized URI like `"docs://documents/{doc_id}"`).

Implementation: use `@mcp.resource` decorator with URI and MIME type parameters. MIME types hint to client about returned data format (`application/json` for structured data, `text/plain` for plain text). Templated resource URI parameters are automatically parsed by SDK and passed as keyword arguments.

Resources vs tools: resources provide data proactively (fetch document contents when `@` mentioned); tools perform actions reactively (when Claude decides to call them).

### Accessing Resources
---
Client-side function to request and parse resources from MCP server. Import `AnyURL` from `pydantic` → call `await self.session.read_resource(AnyURL(uri))` → extract first element from `result.contents[0]` → check `resource.mime_type` for parsing strategy.

Content parsing: if `mime_type == "application/json"` → return `json.loads(resource.text)`; otherwise → return `resource.text`.

### Defining Prompts
---
MCP prompts = pre-defined, tested prompt templates that MCP servers expose to client applications for specialized tasks.

Purpose: instead of users writing ad-hoc prompts, server authors create high-quality, evaluated prompts tailored to their server's domain.

Implementation: `@mcpserver.prompt` decorator with name/description → define function returning list of messages (`base.UserMessage` objects). Example: document formatting prompt that takes document ID, instructs Claude to read document using tools, reformat to markdown, and save changes.

Client integration: prompts appear as autocomplete options (slash commands) in client applications.

### Prompts in the Client
---
- `list_prompts` — `await self.session.list_prompts()`, return `result.prompts`
- `get_prompt` — `await self.session.get_prompt(prompt_name, arguments)`, return `result.messages`

Workflow: define prompt in MCP server with expected arguments → client calls `get_prompt` with prompt name + arguments dictionary → arguments passed as keyword arguments to prompt function → function interpolates arguments into prompt text → returns messages array for direct feeding to LLM.


## Claude Code
Anthropic's terminal-based coding assistant: setup, usage patterns, MCP integration, parallelization, and automated debugging.

### Overview
---
Anthropic deploys two key applications: **Claude Code** (terminal-based coding assistant) and **Computer Use** (toolset expanding Claude beyond text generation). Both demonstrate agent concepts and provide practical examples for agent design.

### Setup
---
Claude Code = terminal-based coding assistant that helps with code-related tasks. Core capabilities: search/read/edit files + advanced tools (web fetching, terminal access) + MCP client support for expanded functionality.

Setup: install Node.js → `npm install` → execute `claude` command → login to Anthropic account. Full guide at `docs.anthropic.com`.

### Claude Code in Action
---
Claude Code functions as a collaborative engineer, not just a code generator. Key capabilities: project setup, feature design, code writing, testing, deployment, error fixing.

Setup workflow: download project → open in editor → run `claude` → ask Claude to read README and execute setup → run `init` command (Claude scans codebase for architecture/style, creates `claude.md`). `claude.md` is automatically included as context for future requests.

Memory types: project (shared), local, user memory files. Use `#` to add specific notes to memory.

#### Effective Prompting Strategies
**Method 1 — Three-step workflow**:
1. Identify relevant files, ask Claude to analyze them
2. Describe feature, ask Claude to plan solution (no code yet)
3. Ask Claude to implement the plan

**Method 2 — Test-driven development**:
1. Provide relevant context
2. Ask Claude to suggest tests for the feature
3. Select and implement chosen tests
4. Ask Claude to write code until tests pass

Core principle: Claude Code = effort multiplier. More detailed instructions = significantly better results.

### Enhancements with MCP Servers
---
Claude Code has an embedded MCP client that can connect to MCP servers. Integration via: `claude mcp add [server-name] [startup-command]`.

Example: document processing server exposing "Document Path to Markdown" tool, allowing Claude Code to read PDF/Word documents. Common use cases: production monitoring (Sentry), project management (Jira), communication (Slack), custom workflow tools.

### Parallelizing Claude Code
---
Running multiple Claude instances simultaneously via **Git work trees** = feature creating complete project copies in separate directories, each corresponding to different Git branches.

Workflow: create work tree → assign task to Claude instance → work in isolation → commit changes → merge back to main branch. Claude automatically resolves merge conflicts.

Custom commands: automate work tree creation via `.claude/commands` directory with markdown files containing prompts and `$ARGUMENTS` placeholder. Scales to unlimited parallel instances based on developer's capacity to manage simultaneous tasks.

### Automated Debugging
---
Using Claude to automatically detect, analyze, and fix production errors without manual intervention.

Core workflow:
1. GitHub Action runs daily to check production environment
2. Fetches CloudWatch logs from last 24 hours
3. Claude identifies and deduplicates errors
4. Claude analyzes each error and generates fixes
5. Creates pull request with proposed solutions

Requirements: repository access, cloud logging service, AI coding assistant, CI/CD pipeline integration. Catches production-only errors (configuration differences between environments) and reduces manual debugging time.


## Computer Use
Claude's ability to interact with computer interfaces through visual observation and programmatic control.

### Overview
---
Claude can interact with computer interfaces through visual observation and control actions: takes screenshots, clicks buttons, types text, navigates interfaces, follows multi-step instructions autonomously.

Runs in an isolated Docker container. User provides instructions via chat → Claude observes screen visually and executes actions → generates reports on completion/results.

Primary use cases: automated QA testing, UI interaction testing, repetitive computer tasks, bug identification through systematic testing.

### How Computer Use Works
---
Computer use follows the identical tool use flow: user sends message + tool schema → Claude responds with tool use request → server executes code → result sent back.

Special tool schema sent to Claude (small schema expands to larger structure). Expanded schema includes action function with arguments: mouse move, left click, screenshot, etc. Claude sends tool use requests; developers must fulfill them via a computing environment (typically Docker container).

Key points: Claude doesn't directly manipulate computers. Computer use = tool system + developer-provided computing environment. Anthropic provides a reference implementation (Docker container with pre-built mouse/keyboard execution code).


## Agents & Workflows
Strategies for handling complex tasks: pre-defined workflow patterns vs flexible agent-based approaches.

### Workflows vs Agents
---
**Workflows** = pre-defined series of calls to Claude with known exact steps. **Agents** = flexible approach using basic tools that Claude combines to complete unknown tasks.

Decision rule: use workflows when you have precise task understanding and know exact step sequence. Use agents when task details are unclear.

| | Workflows | Agents |
|---|---|---|
| Task division | Break big tasks into specific subtasks | Handle varied challenges creatively |
| Testing | Easier due to known execution sequence | Harder since path is unpredictable |
| User experience | Require specific inputs | Create own inputs, can request additional |
| Success rates | Higher completion rates | Lower due to delegated complexity |

Recommendation: prioritize workflows for reliability. Use agents only when flexibility is truly required.

### Evaluator-Optimizer Pattern
---
Example workflow: image to 3D model converter.
1. Claude describes uploaded image in detail
2. Claude uses CADQuery to model object from description
3. Create rendering of model
4. Claude compares rendering to original image
5. If inaccurate, repeat from step 2 with feedback

Producer generates output, evaluator assesses quality, loop continues until evaluator accepts.

### Parallelization Workflows
---
Breaking one complex task into multiple simultaneous subtasks, then aggregating results.

Example: material selection — instead of one large prompt evaluating all materials, use separate parallel requests each evaluating one material's suitability, then a final aggregation step.

\+ Each subtask handles one specific analysis
\+ Individual prompts can be improved/evaluated separately
\+ Easy to add new subtasks without affecting existing ones
\+ Reduces confusion from overly complex single prompts

### Chaining Workflows
---
Breaking large tasks into a series of distinct sequential steps rather than a single complex prompt.

Primary use case: when Claude consistently ignores constraints in complex prompts despite repetition. Common with long prompts containing many "don't do X" requirements.

Solution: Step 1 — send initial prompt, accept imperfect output. Step 2 — follow-up prompt asking Claude to rewrite based on specific violations found.

### Routing Workflows
---
Categorizes user input to determine the appropriate processing pipeline.

Mechanism: initial request to Claude categorizes input into predefined genres/categories → based on response, system routes to specialized pipeline with customized prompts/tools.

Example: social media video script generation where programming topics get educational treatment (definitions, explanations) while entertainment topics get trendy language and engaging hooks.

### Agents and Tools
---
Agents create plans using provided tools when exact steps are unknown.

Tool abstraction principle: provide generic/abstract tools rather than hyper-specialized ones. Example: Claude Code uses `bash`, `web_fetch`, `file_write` (abstract) rather than `refactor_tool`, `install_dependencies` (specialized).

Agent behavior: can request additional information when needed, combines tools creatively. Design approach: give agents abstract tools that can be pieced together rather than single-purpose specialized tools.

### Environment Inspection
---
Agents evaluating their environment and action results to understand progress and handle errors.

Core concept: after each action, agents need feedback mechanisms beyond basic tool returns. Computer use example: Claude takes a screenshot after every action to see how the environment changed. Code editing example: agents must read current file contents before modifying.

Social media video agent examples: use Whisper CPP via bash to generate timestamped captions and verify dialogue placement; use FFmpeg to extract video screenshots at intervals and inspect visual results; validate video creation meets expectations before posting.

Key benefit: enables agents to gauge progress, detect errors, and adapt to unexpected results rather than operating blindly.