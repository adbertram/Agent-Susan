# Discovery: Implement Memory Management System in Susan n8n Workflow

## Codebase Context

### Key Files
- `/opt/claude-agents/Susan/memories.json` - Current memory storage (empty array `[]`, 3 bytes)
- `/opt/claude-agents/Susan/CLAUDE.md` - Susan's instructions on adam-server
- Susan n8n workflow (ID: `U7cK5XlQqmgG9CWlrB6wM`) — 39 nodes, main workflow

### Existing Memory Architecture (TO BE REMOVED)
- **Read Memories node** (`toolCode`, id `91c95fff`): Reads `/opt/claude-agents/Susan/memories.json` via `fs.readFileSync`, returns last 20 entries as plain text
- **Save Memory node** (`toolCode`, id `6372c151`): Appends JSON entries `{date, category, content}` to the same file, 300-char content cap
- **Simple Memory node** (`memoryBufferWindow`, id `b22f65cb`): n8n's built-in conversation buffer, `sessionKey: "susan-session"`, 10-message window
- The current system is completely broken — `memories.json` is empty despite running since Feb 12

### Infrastructure on adam-server
- **SQLite FTS5 with BM25**: Available via n8n's bundled sqlite3 (v5.1.7) at `/usr/local/lib/node_modules/n8n/node_modules/sqlite3`
- **Google AI Embeddings**: `@langchain/google-genai` available in n8n's node_modules, credential `Jl8ZtxGiW3UysVF4`
- **Node.js**: v22.22.0 (n8n), v25.6.0 (system)
- **n8n config**: `NODE_FUNCTION_ALLOW_BUILTIN=*`, LaunchDaemon at `/Library/LaunchDaemons/com.n8n.server.plist`
- **Storage**: /opt at 88% used (25Gi free)
- **Triggers**: IMAP email, Slack messages, Schedule (daily 9am)
- **AI Agent**: Gemini Pro Latest (128k context), connected to memory tools via `ai_tool` connection type

## Q&A Results

### Wave: Problem Validation

**Q:** Current memories.json is empty despite Susan running daily. Fix adoption first, full system now, or parallel tracks?
**A:** Remove existing memory system fully — build the new one from scratch.

**Q:** Simple Memory node uses shared sessionKey across all channels. Include session isolation fix?
**A:** Remove existing memory system entirely (including Simple Memory node).

**Q:** Does pre-compaction flush apply to n8n given workflows are stateless between runs?
**A:** Disregard any existing memory system. (Context: n8n workflows are stateless, no in-flight compaction needed.)

**Q:** Use n8n's bundled packages directly, external Python CLI, or sub-workflow approach?
**A:** Use n8n internals (require() sqlite3 and langchain packages from n8n's node_modules).

### Wave: Success Criteria

**Q:** Daily log fidelity — verbatim, synthesized cues, or structured hybrid?
**A:** Synthesized cues — 1-3 sentence distillations per interaction.

**Q:** MEMORY.md format — sectioned by category, free-form prose, or YAML + prose?
**A:** Free-form prose — natural narrative, updated organically by Susan.

**Q:** Search mode — always pass query, recent + search, or context-aware with MEMORY.md always loaded?
**A:** Always pass query — pure retrieval based on Susan's description of what she needs.

**Q:** Include MMR deduplication?
**A:** Skip MMR — simple top-K by hybrid score. Add later if needed.

**Q:** Temporal decay model?
**A:** Strong decay — exponential over 30-90 days. Old memories fade fast.

**Q:** Return format for search results?
**A:** Full entries — complete memory entry (category, date, full content).

### Wave: Technical Constraints

**Q:** Disk space constraints?
**A:** Monitor, don't limit — reasonable defaults (50k cache, 1 year retention), add alerts if /opt > 95%.

**Q:** Module resolution for sqlite3?
**A:** Add NODE_PATH env var to LaunchDaemon plist — cleaner, requires sudo/restart.

**Q:** Prototype in actual n8n Code node first?
**A:** Test first — build a minimal test Code node in a scratch workflow to confirm sqlite3 require() and fs access.

**Q:** Embedding generation timing — sync, async, or lazy?
**A:** Lazy on search — compute embeddings only when search is performed, cache results.

**Q:** Vector search backend — pure JS cosine similarity, sqlite-vec, or skip vector?
**A:** Pure JS cosine similarity — load embeddings into memory and score. Fine for hundreds of entries.

**Q:** Implementation host — Code nodes, Python CLI, or sub-workflow?
**A:** n8n sub-workflow — create 'Susan - Memory Manager' following existing sub-workflow patterns.

### Wave: Edge Cases

**Q:** Seed MEMORY.md from CLAUDE.md or start fresh?
**A:** Start fresh — empty MEMORY.md, let Susan build organically.

**Q:** Memory behavior for scheduled vs interactive triggers?
**A:** Add `memory_needed` property to the Format Incoming Message node. Memory is NOT needed for blog comment review (scheduled trigger). Susan reads this flag.

**Q:** Per-entry character limit?
**A:** No hard limit — let Susan write as much as needed. Search handles relevance.

**Q:** Concurrent write safety?
**A:** Acceptable risk — usage volume is low, simultaneous writes are rare.

**Q:** Embedding API failure handling?
**A:** Degrade gracefully — save markdown successfully, mark embedding pending, fall back to BM25-only search.

**Q:** Daily log retention policy?
**A:** 90-day rolling — auto-delete logs older than 90 days. MEMORY.md captures important info first.

**Q:** Self-triggered consolidation?
**A:** Self-triggered allowed — Susan can call consolidation when she judges it's time.

### Wave: Integration Impact

**Q:** Tool interface — keep two tools, expand to four, or three?
**A:** Keep two tools — Read Memories (now with search) and Save Memory (now writing to markdown). Minimal topology change.

**Q:** Update system prompt for query-based retrieval?
**A:** Yes, update it — rewrite memory section to explain query-based retrieval.

**Q:** Consolidation hosting — separate workflow, integrate with backup, or inside Susan main?
**A:** Set in Format Incoming Channel node with `memory_needed: false` for scheduled triggers. (Consolidation itself via self-triggered tool call.)

**Q:** Add memory introspection status tool?
**A:** Yes, add status tool — total entries, last consolidation, DB size, health.

**Q:** Context injection format for search results?
**A:** Natural prose — paragraph summarizing relevant memories. Readable for Gemini.

### Wave: Implementation Preferences

**Q:** BM25 + vector search weight ratio?
**A:** Vector-heavy (0.7/0.3) — semantic understanding matters more.

**Q:** SQLite database location?
**A:** `/opt/claude-agents/Susan/memory.sqlite` — co-located with agent home.

**Q:** Embedding model?
**A:** text-embedding-004 (latest, 768 dims).

**Q:** Consolidation LLM approach?
**A:** Dedicated Gemini call — direct API call with specific consolidation prompt.

**Q:** Async pattern for Code nodes?
**A:** Keep synchronous — fs.readFileSync/writeFileSync with sqlite3 sync API.

**Q:** n8n upgrade risk mitigation?
**A:** Accept the risk — document dependency, fix path if n8n upgrade breaks it.

**Q:** Testing strategy?
**A:** Direct edit — edit live workflow. Daily 9am trigger is the only automatic risk.

**Q:** Implementation phasing?
**A:** All at once — build the complete system in one implementation.

### Wave: Risk Assessment

**Q:** Content validation in save tool?
**A:** Sanitize silently — auto-fix bad input (strip JSON, truncate, correct category).

**Q:** Consolidation notification visibility?
**A:** Silent always — plumbing shouldn't be visible unless something breaks.

**Q:** Default search result count?
**A:** 10 results.

**Q:** Index sync strategy when files change?
**A:** File watcher — background process watches memory directory and re-indexes changed files.

## Key Decisions

1. **Rip and replace**: Remove all existing memory nodes (Read Memories, Save Memory, Simple Memory). Build fresh.
2. **Architecture**: n8n sub-workflow ("Susan - Memory Manager") using n8n internal packages (sqlite3, @langchain/google-genai).
3. **Storage**: File-first with `/opt/claude-agents/Susan/memory/` for markdown files, `/opt/claude-agents/Susan/memory.sqlite` for index.
4. **Retrieval**: Hybrid BM25 + vector search (0.7/0.3 weighting), strong temporal decay, no MMR.
5. **Embeddings**: Gemini text-embedding-004, lazy (computed on search, cached), pure JS cosine similarity.
6. **Tools**: Two tools (Search Memories + Save Memory), `memory_needed` flag on incoming message formatting.
7. **Consolidation**: Self-triggered by Susan via dedicated Gemini API call, silent operation, 90-day log retention.
8. **Environment**: Add NODE_PATH to LaunchDaemon, synchronous code, direct edit live workflow.
9. **Output**: Natural prose search results, full entries, 10 results default.
10. **Extras**: Memory status introspection tool, file watcher for index sync, graceful embedding failure degradation.
