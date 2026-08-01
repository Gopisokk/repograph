# repograph

<p align="center">
  <strong>Give your AI coding agent a real map of your codebase — not a pile of text chunks.</strong><br>
  A local-first repository graph engine that indexes every symbol and its relationships, so your agent finds the right code in one shot.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/version-0.1.0-blue?style=flat-square" alt="version">
  <img src="https://img.shields.io/badge/tests-677%20passed-brightgreen?style=flat-square" alt="tests">
  <img src="https://img.shields.io/badge/coverage-80.68%25%20branch-brightgreen?style=flat-square" alt="coverage">
  <img src="https://img.shields.io/badge/python-3.11%2B-blue?style=flat-square" alt="python">
  <img src="https://img.shields.io/badge/license-MIT-yellow?style=flat-square" alt="license">
</p>

<p align="center">
  <sub>Python 3.11+ · macOS · Linux · Windows · Your code never leaves your machine</sub>
</p>

---

## What is repograph?

When you ask an AI coding agent *"where does authentication happen?"*, it typically greps through files, reads dozens of them, and burns thousands of tokens rebuilding context that already exists on your disk.

**repograph** solves this by reading your repository **once** and building a structured graph of it — every function, class, and method, plus the typed relationships between them: what calls what, what extends what, what returns what. Your agent asks the graph a question and gets back *exactly the definition it needed, plus its connected context*, within a token budget.

### How it's different from plain text search

| Approach | What you get |
|---|---|
| `grep` / text search | Partial matches, middle of functions, noise |
| RAG chunk retrieval | Random text windows, broken definitions |
| **repograph** | **Complete symbol definitions + typed relationships** |

Three things make repograph stand out:

- **Symbols, not text spans.** A result is always a complete function or class definition — never the middle of a block.
- **Typed relationships.** `CALLS`, `EXTENDS`, `IMPLEMENTS`, `HAS_TYPE`, `RETURNS` — resolved across files, so callers and callees come back with the answer.
- **Fully local.** One SQLite file inside your project. No account, no cloud upload, no telemetry.

---

## Install

Requires **Python 3.11+**.

```bash
uv tool install repograph   # recommended
# or: pipx install repograph
# or: pip install repograph
```

For development (from source):

```bash
git clone https://github.com/Gopisokk/repograph
cd repograph
pip install -e .
```

This installs two commands:
- **`sg`** — the CLI for indexing, searching and managing your graph
- **`sg-mcp`** — the MCP server your AI agent connects to

```bash
sg --version    # 0.1.0
```

---

## Quick Start

### 1. Build the index

Run this inside any repository you want to index:

```bash
$ sg index .
Indexed into .sg/index.sqlite
  parsed files:        349
  resolved references: 36711
  new: 350
```

The entire index lives in `.sg/index.sqlite` inside your project. Delete it and it's gone — your source files are never modified.

### 2. Connect your AI agent

```bash
$ sg init --agent all
Wrote .mcp.json
Wrote CLAUDE.md
Wrote .cursor/mcp.json
Wrote AGENTS.md
Wrote .vscode/mcp.json
Wrote .github/copilot-instructions.md
Wrote opencode.json
Wrote .gemini/settings.json
Wrote GEMINI.md
Wrote ~/.codex/config.toml
```

`sg init` detects which AI coding agents you have installed and writes two things per agent:
1. The MCP server configuration (so the agent can call the tools)
2. An instruction block in the file that agent actually reads (`CLAUDE.md` for Claude, `GEMINI.md` for Gemini, `AGENTS.md` for Codex/Cursor, etc.)

Both matter. An agent with the tools registered but no instructions keeps grepping. The instruction block tells it to reach for the index first.

### 3. Restart your editor and ask it something

After restarting, your agent has **15 new tools** and will use the graph index instead of grepping. Test it from the terminal:

```bash
$ sg search "auth flow"
Authenticator.__init__ (method) — auth.py [score=0.266 sources=fts]
validate_token (function) — auth.py [score=0.216 sources=fts]
create_session (function) — auth.py [score=0.199 sources=fts]
handle_request (function) — api.py [score=0.032 sources=fts]
AdminAuthenticator.login (method) — admin.py [score=0.029 sources=fts]
```

> **Optional — better semantic search.** By default repograph uses full-text + graph search (no model needed, `0.83` definition accuracy). For `0.92`+ accuracy, install [Ollama](https://ollama.ai) and run `sg embed` — this computes local embeddings for every symbol.

---

## MCP Tools

When connected via MCP, your agent gets these 15 tools:

| Tool | What it does |
|---|---|
| `search_symbols` | Hybrid search: FTS + vector + graph reranking |
| `get_symbol` | Exact lookup by name and kind |
| `get_callers` | All functions that call a given symbol |
| `get_callees` | All functions called by a given symbol |
| `get_implementations` | All implementations of an interface |
| `get_subclasses` | All subclasses of a class |
| `get_references` | All references to a symbol |
| `get_file_symbols` | All symbols defined in a file |
| `get_related` | Graph neighborhood of a symbol |
| `search_by_type` | Find all symbols returning/using a type |
| `get_imports` | Import graph for a file |
| `explain_symbol` | Symbol + context + related in one shot |
| `index_status` | Current index health and counts |
| `reindex` | Trigger an incremental re-index |
| `memory_search` | Search session memory across conversations |

---

## Supported Languages

**Full graph support** — symbols, typed relationships and import resolution:

| Language | Extensions |
|---|---|
| Python | `.py` |
| TypeScript / TSX | `.ts .tsx` |
| JavaScript / JSX | `.js .jsx` |
| Go | `.go` |
| Rust | `.rs` |
| Java | `.java` |
| C# | `.cs` |
| C / C++ | `.c .h .cpp .hpp .cc .hh .cxx .hxx` |

**Text-indexed** — full-text searchable, no syntax tree:

`bash css dockerfile gql graphql hcl htm html json kt less md php proto rb scss sh sql svelte swift tf toml vue xml yaml yml`

---

## CLI Reference

```bash
sg index .                    # Build/update the index for current repo
sg index . --force            # Force full re-index from scratch
sg search "query"             # Search for symbols matching the query
sg search "query" --budget 4000  # Limit token budget of results
sg get MyClass                # Look up a symbol by exact name
sg watch .                    # Watch for file changes and re-index live
sg init                       # Auto-detect agents and write configs
sg init --agent claude        # Write config for Claude Code only
sg doctor .                   # Check index health and agent wiring
sg status                     # Show index stats
sg status --oneline           # One-line summary
sg embed                      # Compute vector embeddings (requires Ollama)
sg uninstall                  # Remove all configs written by sg init
```

---

## Architecture

repograph is built in four phases:

```
Repository files
       │
       ▼
  [Parse Phase]          tree-sitter parsers for each language
       │                 produces: symbols + raw edges
       ▼
  [Analysis Phase]       resolves imports, infers types, deduplicates
       │                 produces: typed relationship graph
       ▼
  [Storage Phase]        SQLite + sqlite-vec for hybrid index
       │                 FTS5 for text, vector table for embeddings
       ▼
  [Retrieval Phase]      hybrid search → graph expansion → reranker
                         produces: ranked symbol list within token budget
```

### Key design decisions

- **SQLite only.** No external database, no server process. The index is a single file that lives next to your code.
- **Tree-sitter for parsing.** Deterministic, fast, works without a language server. Grammar wheels ship with the package.
- **Symbols as the unit.** Every indexed item is a complete, named definition. Retrieval returns whole definitions, never partial chunks.
- **Graph expansion.** After an initial FTS/vector hit, the graph traversal pulls in callers, callees and related types — giving the agent context it would have had to fetch with follow-up calls.
- **Token budget.** Results are trimmed to fit a configurable budget so the agent context window is never blown.

---

## Privacy & Security

Your code stays on your machine. There is no account, no cloud upload, and no telemetry. The only network call repograph makes is to a local Ollama instance, and only if you have one running.

**Secret scanning** runs before anything is indexed:

- **Files skipped entirely:** `.env*`, `credentials.json`, `secrets.yml`, `.pem` / `.key` / `.p12` / `.jks`
- **Content redacted:** AWS keys, GitHub tokens, Slack tokens, Stripe keys, OpenAI / Anthropic / Google API keys, JWTs, private key blocks
- **PII redacted:** emails, IPv4 addresses, phone numbers, SSNs, card numbers that pass a Luhn check
- **Obvious test placeholders left alone:** `your-api-key`, `test_key`, etc.

The dashboard (if running) binds to `127.0.0.1` only.

---

## Troubleshooting

**My agent isn't using the tools.**
Restart your editor after `sg init` — MCP servers are only loaded at startup. Then run `sg doctor .` to verify both the index and agent config exist.

**Search returns nothing.**
Run `sg status --oneline`. If symbol counts are `0`, run `sg index .` and watch for skipped-file warnings.

**Results feel stale.**
Run `sg index .` again, or use `sg watch .` for live updates. `sg init` also installs a post-commit git hook that re-indexes in the background.

**Semantic search quality is low.**
The `0.92 / 0.97` figures require vector embeddings. Install [Ollama](https://ollama.ai), pull a model, then run `sg embed`. `sg doctor` shows which backend is active.

**Something is locked / index is corrupt.**
`sg doctor` shows lock state. A stale lock lives at `.sg/.index.lock`. To start clean: `rm -rf .sg/` — your source files are never touched.

---

## Benchmarks

Measured on three real open-source repositories at token budget 800:

| Repository | FTS+Graph | +Vectors | Baseline (grep) |
|---|---|---|---|
| Django (Python, 350k LOC) | 0.83 | 0.92 | 0.41 |
| VSCode (TypeScript, 1.2M LOC) | 0.81 | 0.91 | 0.38 |
| Kubernetes (Go, 800k LOC) | 0.79 | 0.97 | 0.35 |

*Definition accuracy: fraction of benchmark queries where the correct symbol definition appeared in the top result.*

---

## Project Status

`0.1.0` is an early but genuinely functional release:
- 677 tests passing at 80.68% branch coverage
- 11 language profiles
- 15 MCP tools
- Benchmark numbers are reproducible from the commands in `benchmarks/`

Known gaps (listed honestly):
- Reranker weights are tuned by grid search, not learned from real agent runs
- `export *` wildcard re-exports resolve the file but not always the specific symbol
- Vector search uses a flat index; no HNSW yet

---

## Contributing

Contributions are welcome — see `CONTRIBUTING.md`. The rule that matters:

> If you can't point to the line and the test, it isn't done.

```bash
uv sync --all-extras
uv run pytest -q                          # 677 passed
uv run pytest --cov --cov-fail-under=80   # coverage gate
uv run ruff check .                       # lint
```

Security issues: use GitHub's private vulnerability reporting rather than a public issue. See `SECURITY.md`.

---

## License

MIT — see `LICENSE`.

<p align="center"><sub>If repograph saves you tokens, a star helps.</sub></p>
