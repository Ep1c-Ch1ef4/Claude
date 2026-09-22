---
name: pageindex
description: Build document retrieval with the PageIndex Python SDK — index PDFs into a reasoning-based tree, then query them with your own LLM or expose them as tools to an agent framework. Use when the user asks about PageIndex, PageIndexClient, vectorless or reasoning-based document RAG, or wants question answering with page-level citations over PDFs, reports, filings, or manuals.
---

# PageIndex

PageIndex turns a document into a **tree index** — a hierarchical table of contents with per-node summaries — and an LLM searches that tree by reasoning over it, instead of embedding chunks into a vector store. No vector database, no chunking, no similarity threshold to tune.

Two sides configure independently:

- **Index** — where documents are processed and stored: on the user's machine (`index="<model>"`) or in PageIndex Cloud (`index="cloud"`).
- **Retrieval** — either `client.chat()`, where PageIndex runs its document-QA agent against the user's model, or the agent tools, where the user's own agent framework drives retrieval.

Either index mode works with either retrieval path.

## Setup

```bash
pip install -U pageindex
```

Keys go in the environment, never in source:

```bash
export PAGEINDEX_API_KEY="..."   # cloud indexing only
export OPENAI_API_KEY="..."      # the user's own model, always needed
```

```python
from pageindex import PageIndexClient

# Cloud: PageIndex parses, OCRs, and stores the document
client = PageIndexClient(index="cloud", chat="gpt-5.6-sol")

# Local: everything runs on the user's machine, no PageIndex key
client = PageIndexClient(index="gpt-5.6-luna", chat="gpt-5.6-sol")
```

Omit `chat=` when the user's agent framework brings its own model (see [Agent frameworks](#agent-frameworks)).

Model names follow [LiteLLM's convention](https://docs.litellm.ai/docs/providers) — bare for OpenAI, `anthropic/...`, `openrouter/...`, `openai/...` for an OpenAI-compatible server (with `OPENAI_BASE_URL`).

**Choosing a mode.** Local reads the PDF's text layer, so it only suits text-based PDFs; it is free and open source. Cloud runs production OCR and image understanding, so it is the one to use for scanned pages, charts, and diagrams, and it is the only mode with folders, block-level citations, metadata, and MCP. Cloud also accepts PPTX and Word; local is PDF only.

## Index a document

```python
doc_id = client.submit_document("./2023-annual-report.pdf", wait=True)["doc_id"]
```

`wait=True` blocks until the document is queryable. Without it, `submit_document` returns immediately and the caller polls:

```python
doc_id = client.submit_document("./report.pdf")["doc_id"]
client.get_document(doc_id)["status"]   # "completed" when done
```

Indexing is billed once per page on cloud, so **reuse `doc_id` across runs** — persist it rather than re-submitting the same file. `client.list_documents()` finds documents already indexed.

## Ask a question

`chat()` runs PageIndex's document-QA agent against the model set in `chat=`.

```python
answer = client.chat("What are the key findings?", doc_id=doc_id)
```

The first argument is a question string or a conversation list (`[{"role": "user", "content": ...}, ...]`). Scope the search with exactly one of:

```python
client.chat(messages, doc_id="doc_id_1")                    # one document
client.chat(messages, doc_id=["doc_id_1", "doc_id_2"])      # several
client.chat(messages, folder_id="my-folder-id")             # a folder (cloud)
client.chat(messages)                                       # the whole library
```

Useful arguments:

- `stream=True` — yields the answer in chunks, with the agent's reasoning interleaved. Suppress parts of that with `show_process={"thinking": False}`.
- `citations=True` — citations inline after each claim. With your own chat model: `<cite doc="report.pdf" page="12" block="p12_text_3"/>` tags (`block` on cloud documents with blocks, page-only on local) plus a closing "Sources" list. The managed cloud chat, without your own chat model: `<doc=report.pdf;page=12;block=p12_text_3>` tags only. `client.get_citations(answer)` parses both into a list of `{'document', 'doc_id', 'page', 'block_id', ...}`; `bbox`, `block_type` and `text` ride along only when the block could be read, so use `.get('bbox')`. `client.resolve_citations(...)` returns the same entries display-ready as `{'answer', 'citations'}`, where `answer` has each tag rewritten to a numbered markdown link and each entry adds `anchor` and `index`.
- `protocol="chat_completions" | "responses" | "messages"` — returns a provider-shaped response instead of a string, for dropping into an existing pipeline.

```python
for chunk in client.chat("Summarize this document", doc_id=doc_id, stream=True):
    print(chunk, end="", flush=True)
```

## Agent frameworks

When the user already has an agent, give it PageIndex's retrieval tools instead of calling `chat()`. Two pieces are needed, and they must match: the **system prompt** and the **tools**.

```python
client = PageIndexClient(index="cloud")        # no chat= — the framework's model answers
instructions = client.agent_instructions()     # the orchestration prompt
```

| Framework | Tools |
|---|---|
| OpenAI Agents SDK | `client.as_openai_tools()` |
| Anthropic SDK | `client.as_anthropic_tools()` |
| Claude Agent SDK | `client.as_claude_mcp()` (an MCP server config) |
| Anything else | `client.agent_tools()` — plain Python functions |

```python
from agents import Agent, Runner

agent = Agent(
    name="PageIndex",
    instructions=instructions,
    tools=client.as_openai_tools(),
    model="gpt-5.6-sol",
)
result = Runner.run_sync(agent, messages)
```

Point the agent at specific documents by putting a context block in the first user message:

```python
context = client.document_context(doc_id)        # or client.folder_context("my-folder-id")
messages = [
    {"role": "user", "content": context},
    {"role": "user", "content": "Summarize the auditor's concerns."},
]
```

This **steers** the agent; it does not restrict tool access to those documents.

Tools are read-only by default. `include_management=True` also exposes upload and delete — pass it to both `agent_instructions()` and the tool method so the prompt matches the tool set.

## Inspect and manage documents

```python
client.get_tree(doc_id)                      # the tree index; node_summary=True, include_text=False
client.get_page_content(doc_id, "5-7")       # raw markdown for a page range
client.get_block(doc_id, "p3_text_5")        # one block with its bbox (cloud)
client.get_document(doc_id)                  # name, status, pageNum, createdAt
client.list_documents(limit=10, offset=0)
client.delete_document(doc_id)
```

Read the tree when the goal is to understand or navigate structure (build a TOC, pick a section, drive custom retrieval). Use `chat()` or the agent tools when the goal is an answer — do not hand-roll retrieval over `get_tree()` output.

**Cloud only:** attach `metadata={"source": "web", "year": 2024}` on `submit_document`, and organize documents with folders:

```python
folder_id = client.create_folder("Research Papers")["folder"]["id"]
client.submit_document("./report.pdf", folder_id=folder_id)
client.list_documents(folder_id=folder_id)
client.list_folders(parent_folder_id="root")
```

`"root"` is the top level: `list_documents(folder_id="root")` returns documents in no folder.

## Gotchas

- Both `chat()` and indexing consume the user's own LLM credits; retrieval quality tracks the model in `chat=`, so do not silently downgrade it.
- `submit_document` without `wait=True` is not immediately queryable — poll `get_document(doc_id)["status"]` until `"completed"`.
- Folders, metadata, block-level citations, and MCP are cloud-only; on local they are unavailable, not merely degraded.
- Cloud failures raise `PageIndexAPIError`. Tool-level failures inside an agent are returned as JSON, not raised.

## Reference

Full documentation: https://docs.pageindex.ai — [client configuration](https://docs.pageindex.ai/sdk/client), [document processing](https://docs.pageindex.ai/sdk/documents), [LLM integration](https://docs.pageindex.ai/sdk/chat), [agent integration](https://docs.pageindex.ai/sdk/agents), [MCP](https://docs.pageindex.ai/mcp). Source: https://github.com/VectifyAI/PageIndex
