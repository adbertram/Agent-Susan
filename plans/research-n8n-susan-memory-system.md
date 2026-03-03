# Technical Research: Susan n8n Memory Management System

## Files Analyzed

| File | Type | Key Information | Verified |
|------|------|-----------------|----------|
| `/Users/adam/.n8n/database.sqlite` | SQLite DB | n8n workflow and credential storage | YES |
| `/Library/LaunchDaemons/com.n8n.server.plist` | LaunchDaemon plist | n8n service config, env vars | YES |
| `/opt/claude-agents/Susan/memories.json` | JSON array | Current memory storage (empty) | YES |
| `/opt/homebrew/lib/node_modules/n8n/package.json` | Node package | n8n v1.64.3 | YES |
| `/Users/adam/Dropbox/GitRepos/clawdbot/docs/concepts/memory.md` | Markdown docs | OpenClaw memory architecture reference | YES |

## Workflow IDs (n8n database)

| Workflow Name | Workflow ID | Node Count | Purpose |
|---------------|-------------|-----------|---------|
| Susan (main) | U7cK5XlQqmgG9CWlrB6wM | 39 nodes | Main workflow: routing, agents, memory, image analysis, comments |
| Susan - Send Email Reply | CNt1yTB4mXNUH2bg | 2 nodes | Sub-workflow: execute workflow trigger + email send |
| Susan - Send Slack Message | IXNMpAJZI3od7jsa | TBD | Sub-workflow: send Slack messages |

## Current Memory Implementation Analysis

### Read Memories Node (n8n toolCode)
**ID:** `91c95fff-efe2-46c6-9d13-94dfbaeee761`
**Type:** `@n8n/n8n-nodes-langchain.toolCode` (v1.2)
**Position:** [2000, -864]

**Current Implementation:**
```javascript
const fs = require('fs');
const MEMORY_FILE = '/opt/claude-agents/Susan/memories.json';
const MAX_ENTRIES = 20;
try {
  const data = fs.readFileSync(MEMORY_FILE, 'utf8');
  const memories = JSON.parse(data);
  if (!memories || memories.length === 0) return 'No memories saved yet.';
  const recent = memories.slice(-MAX_ENTRIES);
  const prefix = memories.length > MAX_ENTRIES
    ? '(' + memories.length + ' total memories, showing last ' + MAX_ENTRIES + ')\n' : '';
  return prefix + recent.map(m => '[' + m.date + '] ' + m.category + ': ' + m.content).join('\n');
} catch (e) {
  if (e.code === 'ENOENT') return 'No memories saved yet.';
  return 'Error reading memories: ' + e.message;
}
```

**Status:** TO BE REPLACED - returns plain text, no search capability

### Save Memory Node (n8n toolCode)
**ID:** `6372c151-6e10-4e4a-863b-db1b366c2a9b`
**Type:** `@n8n/n8n-nodes-langchain.toolCode` (v1.2)
**Position:** [2128, -864]

**Current Implementation:**
```javascript
const fs = require('fs');
const MEMORY_FILE = '/opt/claude-agents/Susan/memories.json';
// Reads/writes JSON array with {date, category, content} objects
// 300-char content cap enforced
// Appends entries to array
```

**Status:** TO BE REPLACED - simple append-only, no indexing, no temporal decay

### Simple Memory Node (n8n memoryBufferWindow)
**ID:** `b22f65cb`
**Type:** `n8n-nodes-base.memoryBufferWindow`

**Status:** TO BE REMOVED - shared sessionKey across channels creates isolation issues

## Susan AI Agent Node Configuration

**ID:** `6e8ee7fe-bce9-4d86-9b25-09635328cc31`
**Type:** `@n8n/n8n-nodes-langchain.agent` (v3.1)
**Position:** [2384, -1088]

**Key Attributes:**
- **Prompt Type:** define
- **System Message:** 2200+ lines (comprehensive instructions for Donna-style assistant, tool usage, session management, self-improvement)
- **Tools Connected:**
  - Read Memories (tool ID: `91c95fff`)
  - Save Memory (tool ID: `6372c151`)
  - List Available Agents (Claude Code tool)
  - Run Claude Code Agent (Claude Code tool)
  - Send a message in Slack (tool ID: `23c2d615`)
  - Ask Adam for Approval
  - Reply via Email (tool ID: `email-reply-tool-001`)
  - Plus n8n management tools (Get Workflow, Update Workflow, Execute Workflow, etc.)

**LLM Model:** Google Gemini (via `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)

## Sub-Workflow Call Pattern

### Trigger: Execute Workflow Trigger Node
**Type:** `n8n-nodes-base.executeWorkflowTrigger`
**Key Parameter:** `inputSource: "passthrough"` — accepts input from parent workflow

### Implementation in Susan Main Workflow
Email Reply sub-workflow is connected as a **toolWorkflow** (special n8n node):

```json
{
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "name": "Reply via Email",
  "parameters": {
    "name": "reply_via_email",
    "description": "Send an email reply from susan@adamtheautomator.com",
    "workflowId": {
      "__rl": true,
      "value": "Mnd60jujVIHSFXTM",
      "mode": "id"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "toEmail": "= $fromAI('toEmail', ...)",
        "subject": "= $fromAI('subject', ...)",
        "body": "= $fromAI('body', ...)",
        "fromEmail": "susan@adamtheautomator.com"
      },
      "schema": [
        { "id": "fromEmail", "required": true, "type": "string" },
        { "id": "toEmail", "required": true, "type": "string" },
        { "id": "subject", "required": true, "type": "string" },
        { "id": "body", "required": true, "type": "string" }
      ]
    }
  }
}
```

**Sub-Workflow (Send Email Reply) Structure:**
- **Trigger node:** Execute Workflow Trigger (id: `trigger-email`)
- **Action node:** Email Send (id: `email-send`, credentials: Susan SMTP DreamHost)
- **Connection:** trigger-email → email-send

### Pattern for Memory Manager Sub-Workflow
The Memory Manager sub-workflow will follow this same pattern:
1. Create new workflow: "Susan - Memory Manager"
2. Add Execute Workflow Trigger node (inputSource: passthrough)
3. Add toolCode nodes for search/save operations
4. Return JSON output with results
5. Register as toolWorkflow in main Susan agent

## n8n Environment Configuration

### Node.js Setup
- **Node.js Version:** v22.22.0 (used by n8n LaunchDaemon)
- **n8n Version:** 1.64.3
- **n8n Install Path:** `/opt/homebrew/lib/node_modules/n8n`
- **LaunchDaemon Path:** `/Library/LaunchDaemons/com.n8n.server.plist`

### Current Environment Variables (plist)
```xml
<key>NODE_FUNCTION_ALLOW_BUILTIN</key>
<string>*</string>
```

**To Add:**
```xml
<key>NODE_PATH</key>
<string>/opt/homebrew/lib/node_modules/n8n/node_modules</string>
```

### Module Resolution
- **sqlite3 path:** `/opt/homebrew/lib/node_modules/n8n/node_modules/sqlite3` (v5.1.7)
- **@langchain/google-genai path:** `/opt/homebrew/lib/node_modules/n8n/node_modules/@langchain/google-genai` (v0.1.0)
- **GoogleGenerativeAIEmbeddings class:** Available in `@langchain/google-genai`
- **FTS5:** Available in sqlite3 — tested and confirmed working

### Require Strategy for Code Nodes
Code nodes in n8n have access to:
1. `fs` module (native) — readFileSync, writeFileSync, appendFileSync
2. Full path require: `require('/opt/homebrew/lib/node_modules/n8n/node_modules/sqlite3')`
3. Environment variables accessible via `process.env`

## Storage Layout (File System)

### Current
- `/opt/claude-agents/Susan/memories.json` - JSON array (empty, 3 bytes)
- Permissions: rw-r--r-- (644), owned by adam:staff

### New Structure
```
/opt/claude-agents/Susan/
├── memory.sqlite                  # SQLite index + embeddings cache
├── memory/
│   ├── MEMORY.md                  # Curated long-term memories (natural prose)
│   ├── YYYY-MM-DD.md              # Daily logs (append-only markdown)
│   └── .embeddings.json           # Cached embeddings {content_hash: [768-dim vector]}
└── memories.json                  # (DEPRECATED - will be removed)
```

### Directory Permissions
- Create with `drwxr-xr-x` (755) — readable/searchable by all, writable by adam
- File permissions: `rw-r--r--` (644)

## OpenClaw Memory Algorithm Reference

### Hybrid Search: Vector + BM25
From `/Users/adam/Dropbox/GitRepos/clawdbot/docs/concepts/memory.md`:

**Configuration (our choices):**
```
Vector Weight: 0.7 (semantic understanding)
Text Weight: 0.3 (keyword relevance)
Candidate Multiplier: 4x (retrieve 4x candidates, merge, take top-K)
```

**Merging Strategy:**
1. Retrieve top candidates from BM25 (SQLite FTS5)
2. Retrieve top candidates from vector search (in-memory cosine similarity)
3. Convert BM25 ranks to normalized scores: `MAX(0, 1 - ABS(rank) / max_rank)`
4. Compute weighted final score: `0.7 × vector_score + 0.3 × bm25_score`
5. Apply temporal decay multiplier
6. Return top-K results sorted by final score

### BM25 (Full Text Search)
- **SQLite Implementation:** FTS5 table with MATCH queries
- **Query Pattern:** `SELECT content, rank FROM fts_table WHERE fts_table MATCH ? ORDER BY rank`
- **Rank Value:** Negative floats (-5.2, -2.1, etc) — lower = more relevant
- **Normalization:** `MAX(0, 1 - ABS(rank) / max_possible_rank)`

**Example Normalization:**
| BM25 Rank | Normalized Score |
|-----------|------------------|
| -0.5 | 0.99 |
| -2.5 | 0.95 |
| -5.0 | 0.90 |
| -10.0 | 0.80 |

### Vector Search
- **Model:** Gemini `text-embedding-004` (768 dimensions)
- **Method:** Pure JavaScript cosine similarity (in-memory)
- **Lazy Computation:** Embeddings generated only during search, cached in JSON
- **Similarity Formula:**
  ```
  similarity = (A·B) / (|A| × |B|)
  where:
    A·B = sum(A[i] × B[i])
    |A| = sqrt(sum(A[i]²))
    |B| = sqrt(sum(B[i]²))
  ```

### Temporal Decay
- **Model:** Exponential decay with 30-90 day half-life
- **Formula:** `decay_score = 2^(-age_days / half_life_days)`
- **Applied:** Multiplied against final hybrid score
- **Examples (30-day half-life):**
  | Date | Days Old | Decay Score |
  |------|----------|-------------|
  | Today | 0 | 1.0000 |
  | 4 days ago | 4 | 0.9117 |
  | 10 days ago | 10 | 0.7937 |
  | 30 days ago | 30 | 0.5000 |
  | 90 days ago | 90 | 0.1250 |

## API Integration Details

### Google Gemini Embeddings (LangChain)
**Module:** `@langchain/google-genai` v0.1.0
**Class:** `GoogleGenerativeAIEmbeddings`

**Constructor Signature:**
```javascript
new GoogleGenerativeAIEmbeddings({
  modelName: "text-embedding-004",     // 768 dimensions
  googleApiKey: "GOOGLE_API_KEY"        // From env or credential
})
```

**Available Methods:**
- `embedDocuments(texts: string[])` — Batch embed multiple texts, returns `number[][]`
- `embedQuery(text: string)` — Embed single query, returns `number[]`

**Credential in n8n:**
- **ID:** `Jl8ZtxGiW3UysVF4`
- **Name:** "Google Gemini(PaLM) Api account"
- **Type:** `googlePalmApi` (encrypted in database)
- **Access:** Via n8n credential system (read during agent execution)

**Important:** googlePalmApi credential may be for older Palm API. Need to verify if it's compatible with Gemini embedding models, or if we need to add GOOGLE_API_KEY env var to LaunchDaemon plist.

### Consolidation LLM Call
**Model:** Google Gemini Pro (via existing chat model node)
**Pattern:** Direct API call with specific consolidation prompt
**Credentials:** Same as main Susan agent (already configured)

## SQLite Schema (memory.sqlite)

```sql
-- Full-text search index
CREATE VIRTUAL TABLE memory_fts USING fts5(
  id UNINDEXED,
  content,
  date UNINDEXED,
  category UNINDEXED,
  source UNINDEXED
);

-- Metadata and embeddings cache
CREATE TABLE memory_entries (
  id TEXT PRIMARY KEY,
  date TEXT NOT NULL,
  category TEXT NOT NULL,
  source TEXT,
  content TEXT NOT NULL,
  embedding_cached BOOLEAN DEFAULT 0
);

CREATE TABLE embeddings_cache (
  content_hash TEXT PRIMARY KEY,
  embedding BLOB,  -- JSON array [768 floats]
  cached_at TEXT
);

-- Daily logs metadata
CREATE TABLE daily_logs (
  date TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  last_indexed TEXT,
  entry_count INTEGER
);
```

## Code Node Requirements & Constraints

### What Works in n8n Code Nodes
✓ `require('fs')` — file operations (readFileSync, writeFileSync, appendFileSync)
✓ `require('/opt/homebrew/lib/node_modules/n8n/node_modules/sqlite3')`
✓ JSON parsing/stringification
✓ Array and object operations
✓ String manipulation
✓ Math operations
✓ Process environment (`process.env.NODE_PATH`, etc)
✓ setTimeout/async operations (though synchronous preferred)

### What Needs Testing
- `@langchain/google-genai` require() in Code node context
- Encrypted credential decryption/access pattern
- File I/O within n8n's sandboxed environment

### Constraint: Synchronous Code
- n8n Code nodes for tools should be synchronous
- Use `fs.readFileSync()`, `fs.writeFileSync()`
- SQLite3 sync mode: `db.run()` with callbacks (not promises)
- Embedding calls must be deferred to search-time only (lazy)

## n8n Sub-Workflow Execution Flow

1. **Parent Workflow (Susan)** — AI agent calls `Reply via Email` tool
2. **Tool Definition (toolWorkflow node)** — Maps AI-generated parameters to sub-workflow inputs
3. **Execute Workflow Trigger** — Sub-workflow receives `$input` with mapped parameters
4. **Action Nodes** — Process the input (e.g., send email)
5. **Return Value** — Final node result returned to parent workflow/AI agent

**Key Points:**
- `inputSource: "passthrough"` — sub-workflow receives full input object
- Input is available as `$json` in sub-workflow nodes
- No explicit return statement needed — last node's output becomes return value
- Parameters must match schema defined in parent's toolWorkflow node

## Credential Handling in n8n

### Encrypted Storage
- Credentials stored in `credentials_entity` table with encrypted `data` field
- Decryption handled automatically by n8n when credentials are referenced
- n8n resolves credential IDs to actual values during node execution

### Credential Types Available
- `googlePalmApi` — Google Gemini/PaLM API (ID: `Jl8ZtxGiW3UysVF4`)
- `imap`, `smtp` — Email (DreamHost)
- `slackApi` — Slack
- `gmailOAuth2` — Gmail
- Plus others (Airtable, Shippo, eBay, Bricklink, etc.)

### For Google Embeddings
**Option 1:** Use `@langchain/google-genai` with GOOGLE_API_KEY env var (requires LaunchDaemon update)
**Option 2:** Access existing googlePalmApi credential via n8n's credential system (requires testing)

## Memory System Integration Points

### Where Memory Needed
1. **Incoming Email** — YES (memory_needed: true)
2. **Incoming Slack** — YES (memory_needed: true)
3. **Scheduled Blog Comment Review** — NO (memory_needed: false, skip to comment processing)
4. **Scheduled Daily Task** (9am) — NO (memory_needed: false)

### Memory Flag in Format Node
New property to add to message payload:
```javascript
{ 
  text: "...",
  incoming_channel: { ... },
  memory_needed: true,  // or false for scheduled tasks
  source: "email|slack|scheduled"
}
```

## Implementation Verification Checklist

- [x] SQLite FTS5 available in bundled sqlite3 (v5.1.7)
- [x] @langchain/google-genai available (v0.1.0)
- [x] GoogleGenerativeAIEmbeddings class available
- [x] fs module accessible in Code nodes
- [x] Can require('/opt/homebrew/lib/node_modules/n8n/node_modules/sqlite3')
- [x] Can read/write files to /opt/claude-agents/Susan/
- [x] Cosine similarity algorithm works (tested with 768-dim vectors)
- [x] BM25 rank normalization formula confirmed
- [x] Temporal decay formula confirmed
- [x] Sub-workflow execution pattern understood
- [ ] GooglePalmApi credential compatible with text-embedding-004 (NEEDS TESTING)
- [ ] @langchain/google-genai credential mapping in Code nodes (NEEDS TESTING)

## Key Dependencies & Versions

| Component | Version | Path |
|-----------|---------|------|
| Node.js (n8n) | 22.22.0 | `/opt/homebrew/opt/node@22/bin/node` |
| n8n | 1.64.3 | `/opt/homebrew/lib/node_modules/n8n` |
| sqlite3 | 5.1.7 | `/opt/homebrew/lib/node_modules/n8n/node_modules/sqlite3` |
| @langchain/google-genai | 0.1.0 | `/opt/homebrew/lib/node_modules/n8n/node_modules/@langchain/google-genai` |

## Risk & Mitigation

| Risk | Severity | Mitigation |
|------|----------|-----------|
| API rate limits (embeddings) | Medium | Lazy compute, cache, fallback to BM25-only |
| SQLite database corruption | Low | Regular backups, readonly snapshots before consolidation |
| n8n upgrade breaking paths | Medium | Document dependency paths, test after upgrade |
| Google API key exposure | High | Keep in LaunchDaemon env var or use n8n credential system |
| Concurrent file writes | Low | Low usage volume, acceptable race condition risk |
| Memory database grows too large | Medium | 90-day rolling retention, monitoring alerts |

## Next Steps for Implementation

1. **Phase 1: Setup**
   - Create `/opt/claude-agents/Susan/memory/` directory
   - Create `memory.sqlite` with schema
   - Create empty `.embeddings.json` cache file

2. **Phase 2: Sub-Workflow**
   - Create "Susan - Memory Manager" workflow
   - Implement Search Memories Code node
   - Implement Save Memory Code node

3. **Phase 3: Integration**
   - Replace Read Memories node with search-capable version
   - Replace Save Memory node with new writer
   - Remove Simple Memory node
   - Update AI agent system prompt with new tool descriptions

4. **Phase 4: Testing**
   - Test in scratch workflow first (Code node execution)
   - Test sub-workflow calls
   - Test embedding generation and caching
   - Test BM25 + vector hybrid search
   - Test temporal decay application

5. **Phase 5: Deployment**
   - Update LaunchDaemon plist (add NODE_PATH if needed)
   - Enable daily consolidation trigger
   - Monitor memory system health

