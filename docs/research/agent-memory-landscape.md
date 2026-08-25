# AI Agent Context & Memory Persistence — Landscape Scan

Landscape scan of open-source projects focused on **AI agent context and memory persistence**.

**Filter criteria (both required):**

- ≥ 3,000 stars
- At least one commit on the repo's **default branch** since **2026-08-10** (14 days)

Activity was verified with GitHub's commit search, which indexes the default branch only — so these
are confirmed default-branch commits, not just repo-level `pushed` timestamps. Star counts and
commit dates are as of **2026-08-25**.

## Core: agent memory / context persistence

| Repo | ⭐ | Last default-branch commit | What it is |
|---|---|---|---|
| [thedotmack/claude-mem](https://github.com/thedotmack/claude-mem) | 91.7k | Aug 19 | Persistent context across sessions; captures + compresses + re-injects |
| [mem0ai/mem0](https://github.com/mem0ai/mem0) | 64.0k | Aug 24 | Universal memory layer for AI agents |
| [MemPalace/mempalace](https://github.com/MemPalace/mempalace) | 58.6k | Aug 24 (`develop`) | Open-source AI memory system, benchmark-focused |
| [DeusData/codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp) | 40.3k | Aug 21 | Codebase → persistent knowledge graph over MCP |
| [volcengine/OpenViking](https://github.com/volcengine/OpenViking) | 32.9k | Aug 24 | Self-evolving context database for AI agents |
| [getzep/graphiti](https://github.com/getzep/graphiti) | 30.3k | Aug 20 | Real-time temporal knowledge graphs for agents |
| [topoteretes/cognee](https://github.com/topoteretes/cognee) | 30.2k | Aug 24 | AI memory platform; self-hosted knowledge-graph engine |
| [supermemoryai/supermemory](https://github.com/supermemoryai/supermemory) | 29.0k | Aug 24 | Memory + context engine, Memory API |
| [rohitg00/agentmemory](https://github.com/rohitg00/agentmemory) | 27.4k | Aug 23 | Persistent memory for coding agents, benchmark-driven |
| [letta-ai/letta](https://github.com/letta-ai/letta) | 24.4k | Aug 23 | Stateful agents (MemGPT lineage) |
| [TencentCloud/TencentDB-Agent-Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory) | 24.2k | Aug 15 (`feat/server_team`) | Team-level memory hub: chat memory, wiki, code-graph |
| [vectorize-io/hindsight](https://github.com/vectorize-io/hindsight) | 21.0k | Aug 24 | Agent memory that learns |
| [mksglu/context-mode](https://github.com/mksglu/context-mode) | 20.1k | Aug 24 | Context-window optimization + session memory via MCP/hooks |
| [MemoriLabs/Memori](https://github.com/MemoriLabs/Memori) | 16.2k | Aug 21 | Agent-native memory infrastructure, enterprise-oriented |
| [NevaMind-AI/memU](https://github.com/NevaMind-AI/memU) | 14.3k | Aug 21 | Personal memory across agents |
| [EverMind-AI/EverOS](https://github.com/EverMind-AI/EverOS) | 12.4k | Aug 17 | Portable, local-first, Markdown-native memory layer |
| [MemTensor/MemOS](https://github.com/MemTensor/MemOS) | 11.0k | Aug 24 | Memory OS: persistent memory + hybrid retrieval |
| [semantica-agi/semantica](https://github.com/semantica-agi/semantica) | 10.6k | Aug 24 | Graph-native context infrastructure with provenance |
| [plastic-labs/honcho](https://github.com/plastic-labs/honcho) | 6.8k | Aug 24 | Memory library for stateful agents |
| [Gentleman-Programming/engram](https://github.com/Gentleman-Programming/engram) | 6.1k | Aug 17 | Agent-agnostic Go memory server (SQLite + FTS5, MCP) |
| [breferrari/obsidian-mind](https://github.com/breferrari/obsidian-mind) | 4.6k | Aug 24 | Self-organizing Obsidian vault as agent memory |
| [CaviraOSS/OpenMemory](https://github.com/CaviraOSS/OpenMemory) | 4.5k | Aug 24 | Local persistent memory store for LLM apps |
| [akitaonrails/ai-memory](https://github.com/akitaonrails/ai-memory) | 4.4k | Aug 24 | Long-term memory + handoff between agent CLIs |
| [Ar9av/obsidian-wiki](https://github.com/Ar9av/obsidian-wiki) | 3.3k | Aug 23 | Agents build/maintain a digital brain in an Obsidian wiki |
| [MemMachine/MemMachine](https://github.com/MemMachine/MemMachine) | 3.2k | Aug 20 | Universal memory layer, interoperable storage/retrieval |
| [letta-ai/letta-code](https://github.com/letta-ai/letta-code) | 3.1k | Aug 24 | Stateful coding agent with memory/identity |

## Adjacent (infrastructure or harness, not memory-first)

| Repo | ⭐ | Last commit | Note |
|---|---|---|---|
| [memgraph/memgraph](https://github.com/memgraph/memgraph) | 4.4k | Aug 24 (`master`) | Graph DB explicitly positioned for AI memory / GraphRAG |
| [EverMind-AI/Raven](https://github.com/EverMind-AI/Raven) | 3.6k | Aug 24 | Memory-first agent harness built on EverOS |

## Notes & caveats

- **Default branch ≠ `main` for three entries.** `TencentDB-Agent-Memory` uses `feat/server_team`,
  `mempalace` uses `develop`, and `memgraph` uses `master`. "Commit to main" was interpreted as
  "commit to the default branch".
- **Deliberately excluded:** general-purpose databases that merely market toward agent workloads
  (TiDB, Dolt, SurrealDB), and awesome-lists / curated link collections — neither category is a
  memory system itself.
- **Borderline, not included:** context-compression and capture tools such as
  `headroomlabs-ai/headroom`, `screenpipe/screenpipe`, and `OthmanAdi/planning-with-files`. They
  touch context persistence but aren't memory stores.
