# Implementation Plan: Susan n8n Memory Management System

## Summary

Susan's existing memory system (JSON append-only file, unused Simple Memory buffer) is completely broken with zero retained memories despite daily use. The new system replaces all three existing memory nodes with a hybrid BM25 + vector search architecture: file-first markdown storage at `/opt/claude-agents/Susan/memory/`, a SQLite FTS5 index at `/opt/claude-agents/Susan/memory.sqlite`, and lazy Gemini embeddings cached in `.embeddings.json`. A new "Susan - Memory Manager" n8n sub-workflow hosts the search and save logic. Two tool nodes in the main Susan workflow wire it to the AI agent. Temporal decay (30-day half-life) and hybrid scoring (0.7 vector / 0.3 BM25) ensure recent, semantically relevant memories surface first.

## Why This Approach

- **Simplest path**: Reuses n8n's existing bundled packages (sqlite3 v5.1.7, @langchain/google-genai v2.1.10) via full-path `require()` — no new npm installs
- **Sub-workflow pattern**: Follows the established `toolWorkflow` pattern already used by Reply via Email (workflow `Mnd60jujVIHSFXTM`), keeping the main Susan workflow topology minimal
- **File-first**: Markdown files are human-readable and survive SQLite corruption; the SQLite index is rebuilable from markdown
- **Rip and replace**: Eliminates root cause (broken append-only JSON) rather than patching around it
- **Alternatives considered**: Python CLI sidecar (rejected — adds dependency), n8n community memory nodes (rejected — lack hybrid search), direct Code nodes in Susan main (rejected — messy topology, harder to test)

## Prerequisites

- SSH access to adam-server as adam
- n8n API key (confirmed working): from `sqlite3 ~/.n8n/database.sqlite "SELECT apiKey FROM user_api_keys LIMIT 1;"`
- `sudo` access on adam-server for LaunchDaemon plist edit and `launchctl` restart
- GOOGLE_API_KEY value (the actual key from the googlePalmApi credential `Jl8ZtxGiW3UysVF4`) — needed for LaunchDaemon env var
- n8n running at `http://localhost:5678` on adam-server
- Verified paths:
  - n8n binary: `/usr/local/lib/node_modules/n8n/bin/n8n`
  - Node.js (used by LaunchDaemon): `/opt/homebrew/opt/node@22/bin/node` (v22.22.0)
  - sqlite3: `/usr/local/lib/node_modules/n8n/node_modules/sqlite3` (v5.1.7)
  - @langchain/google-genai: `/usr/local/lib/node_modules/n8n/node_modules/@langchain/google-genai` (v2.1.10)
  - LaunchDaemon plist: `/Library/LaunchDaemons/com.n8n.server.plist`

## Implementation Steps

### Step 1: Retrieve the Google API Key from n8n Credentials

**On:** adam-server (SSH)

The Google API key is stored encrypted in n8n's database. The encryption key is in `~/.n8n/config`. Extract the key for use in the LaunchDaemon plist.

**Action:** Run in n8n Code node (or via the n8n UI manually) to expose the credential value:

```javascript
// Run this as a one-time Code node in a scratch n8n workflow
// This uses n8n's internal credential system — it will decrypt and show the key
// Alternatively: look up the key in the n8n UI under Settings > Credentials > Google Gemini(PaLM) Api account
```

**Practical approach:** In the n8n UI at `http://100.117.198.37:5678`, go to Credentials, open "Google Gemini(PaLM) Api account" (ID: `Jl8ZtxGiW3UysVF4`), and copy the API key value.

**Verify:** You have a string starting with `AIza...` that is 39 characters long.

---

### Step 2: Update LaunchDaemon Plist (NODE_PATH + GOOGLE_API_KEY)

**File:** `/Library/LaunchDaemons/com.n8n.server.plist`
**Requires:** `sudo` on adam-server — ask Adam to run these commands manually

**Action:** Add two new `<key>/<string>` pairs inside the `<dict>` block under `EnvironmentVariables`. The current plist ends its `EnvironmentVariables` dict before `</dict>`. Insert after the existing `NODE_FUNCTION_ALLOW_BUILTIN` entry:

```xml
<key>NODE_PATH</key>
<string>/usr/local/lib/node_modules/n8n/node_modules</string>
<key>GOOGLE_API_KEY</key>
<string>YOUR_ACTUAL_GOOGLE_API_KEY_HERE</string>
```

The full updated `EnvironmentVariables` dict (showing insertion point):
```xml
<key>EnvironmentVariables</key>
<dict>
    <!-- existing entries ... -->
    <key>NODE_FUNCTION_ALLOW_BUILTIN</key>
    <string>*</string>
    <!-- ADD THESE TWO: -->
    <key>NODE_PATH</key>
    <string>/usr/local/lib/node_modules/n8n/node_modules</string>
    <key>GOOGLE_API_KEY</key>
    <string>AIza...YOUR_KEY...</string>
    <!-- existing PATH entry follows -->
    <key>PATH</key>
    <string>/opt/homebrew/opt/node@22/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin</string>
</dict>
```

**After editing, reload the LaunchDaemon (requires sudo — ask Adam):**
```bash
sudo launchctl unload /Library/LaunchDaemons/com.n8n.server.plist
sudo launchctl load /Library/LaunchDaemons/com.n8n.server.plist
```

**Verify:** Wait 10 seconds, then:
```bash
curl -s http://localhost:5678/healthz
# Expected: {"status":"ok"}
```

---

### Step 3: Create Memory Directory Structure on adam-server

**On:** adam-server (SSH, no sudo needed)

**Action:**
```bash
mkdir -p /opt/claude-agents/Susan/memory
touch /opt/claude-agents/Susan/memory/MEMORY.md
touch /opt/claude-agents/Susan/memory/.embeddings.json
echo '{}' > /opt/claude-agents/Susan/memory/.embeddings.json
chmod 755 /opt/claude-agents/Susan/memory
chmod 644 /opt/claude-agents/Susan/memory/MEMORY.md
chmod 644 /opt/claude-agents/Susan/memory/.embeddings.json
```

**MEMORY.md initial content** (empty except for header):
```markdown
# Susan's Memory

This file contains Susan's curated long-term memories about Adam, his business, preferences, and patterns. Updated organically by Susan during consolidation.
```

**Verify:**
```bash
ls -la /opt/claude-agents/Susan/memory/
# Expected: MEMORY.md and .embeddings.json exist, directory owned by adam
```

---

### Step 4: Initialize SQLite Database Schema

**On:** adam-server (SSH)

**Action:** Create a bootstrap script and run it:

```bash
cat > /tmp/init_memory_db.js << 'EOF'
const sqlite3 = require('/usr/local/lib/node_modules/n8n/node_modules/sqlite3').verbose();
const DB_PATH = '/opt/claude-agents/Susan/memory.sqlite';

const db = new sqlite3.Database(DB_PATH, (err) => {
  if (err) { console.error('DB open error:', err.message); process.exit(1); }
  console.log('Opened:', DB_PATH);
});

db.serialize(() => {
  // FTS5 full-text search index
  db.run(`CREATE VIRTUAL TABLE IF NOT EXISTS memory_fts USING fts5(
    id UNINDEXED,
    content,
    date UNINDEXED,
    category UNINDEXED,
    source_file UNINDEXED
  )`, (err) => { if (err) console.error('FTS5 create error:', err.message); else console.log('memory_fts: OK'); });

  // Metadata table for structured queries
  db.run(`CREATE TABLE IF NOT EXISTS memory_entries (
    id TEXT PRIMARY KEY,
    date TEXT NOT NULL,
    category TEXT NOT NULL,
    source_file TEXT,
    content TEXT NOT NULL,
    embedding_cached INTEGER DEFAULT 0,
    created_at TEXT DEFAULT (datetime('now'))
  )`, (err) => { if (err) console.error('memory_entries error:', err.message); else console.log('memory_entries: OK'); });

  // Embeddings cache — keyed by SHA256 of content
  db.run(`CREATE TABLE IF NOT EXISTS embeddings_cache (
    content_hash TEXT PRIMARY KEY,
    embedding TEXT NOT NULL,
    cached_at TEXT DEFAULT (datetime('now'))
  )`, (err) => { if (err) console.error('embeddings_cache error:', err.message); else console.log('embeddings_cache: OK'); });

  // Daily logs metadata for retention management
  db.run(`CREATE TABLE IF NOT EXISTS daily_logs (
    date TEXT PRIMARY KEY,
    path TEXT NOT NULL,
    last_indexed TEXT,
    entry_count INTEGER DEFAULT 0
  )`, (err) => { if (err) console.error('daily_logs error:', err.message); else console.log('daily_logs: OK'); });

  // System state (consolidation tracking, last sync)
  db.run(`CREATE TABLE IF NOT EXISTS system_state (
    key TEXT PRIMARY KEY,
    value TEXT,
    updated_at TEXT DEFAULT (datetime('now'))
  )`, (err) => { if (err) console.error('system_state error:', err.message); else console.log('system_state: OK'); });
});

db.close((err) => {
  if (err) console.error('Close error:', err.message);
  else console.log('DB initialized successfully');
});
EOF

/opt/homebrew/opt/node@22/bin/node /tmp/init_memory_db.js
```

**Verify:**
```bash
/opt/homebrew/opt/node@22/bin/node -e "
const sqlite3 = require('/usr/local/lib/node_modules/n8n/node_modules/sqlite3').verbose();
const db = new sqlite3.Database('/opt/claude-agents/Susan/memory.sqlite');
db.all(\"SELECT name FROM sqlite_master WHERE type IN ('table','index') ORDER BY name\", (err, rows) => {
  console.log(rows.map(r => r.name).join(', '));
  db.close();
});
"
# Expected: daily_logs, embeddings_cache, memory_entries, memory_fts, system_state
```

---

### CHECKPOINT: Infrastructure Ready (Steps 1-4)

**Run on adam-server:**
```bash
# 1. n8n is running
curl -s http://localhost:5678/healthz
# Expected: {"status":"ok"}

# 2. Memory directory exists
ls /opt/claude-agents/Susan/memory/
# Expected: MEMORY.md  .embeddings.json

# 3. SQLite DB has all tables
/opt/homebrew/opt/node@22/bin/node -e "
const sqlite3 = require('/usr/local/lib/node_modules/n8n/node_modules/sqlite3').verbose();
const db = new sqlite3.Database('/opt/claude-agents/Susan/memory.sqlite');
db.all(\"SELECT name FROM sqlite_master WHERE type='table'\", (e,r) => { console.log(r.map(x=>x.name)); db.close(); });
"
# Expected: ['memory_fts', 'memory_entries', 'embeddings_cache', 'daily_logs', 'system_state']

# 4. NODE_PATH resolves modules
/opt/homebrew/opt/node@22/bin/node -e "process.env.NODE_PATH='/usr/local/lib/node_modules/n8n/node_modules'; require('module').Module._initPaths(); const s=require('sqlite3'); console.log('sqlite3 via NODE_PATH:', typeof s.Database)"
# Expected: sqlite3 via NODE_PATH: function
```

---

### Step 5: Create "Susan - Memory Manager" Sub-Workflow

**Method:** n8n REST API POST from adam-server

This creates the new sub-workflow skeleton that will host the three Code nodes (Search, Save, Consolidate/Status).

**Action:** Run this script on adam-server:

```bash
N8N_KEY=$(sqlite3 ~/.n8n/database.sqlite "SELECT apiKey FROM user_api_keys LIMIT 1;")

curl -s -X POST 'http://localhost:5678/api/v1/workflows' \
  -H "X-N8N-API-KEY: $N8N_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Susan - Memory Manager",
    "nodes": [
      {
        "id": "mem-trigger-001",
        "name": "Execute Workflow Trigger",
        "type": "n8n-nodes-base.executeWorkflowTrigger",
        "typeVersion": 1,
        "position": [240, 300],
        "parameters": { "inputSource": "passthrough" }
      },
      {
        "id": "mem-router-001",
        "name": "Route Action",
        "type": "n8n-nodes-base.switch",
        "typeVersion": 3,
        "position": [460, 300],
        "parameters": {
          "mode": "rules",
          "rules": {
            "values": [
              { "conditions": { "options": { "caseSensitive": false }, "conditions": [{ "leftValue": "={{ $json.action }}", "rightValue": "search", "operator": { "type": "string", "operation": "equals" } }] } },
              { "conditions": { "options": { "caseSensitive": false }, "conditions": [{ "leftValue": "={{ $json.action }}", "rightValue": "save", "operator": { "type": "string", "operation": "equals" } }] } },
              { "conditions": { "options": { "caseSensitive": false }, "conditions": [{ "leftValue": "={{ $json.action }}", "rightValue": "consolidate", "operator": { "type": "string", "operation": "equals" } }] } },
              { "conditions": { "options": { "caseSensitive": false }, "conditions": [{ "leftValue": "={{ $json.action }}", "rightValue": "status", "operator": { "type": "string", "operation": "equals" } }] } }
            ]
          },
          "fallbackOutput": "none"
        }
      },
      {
        "id": "mem-search-001",
        "name": "Search Memories",
        "type": "n8n-nodes-base.code",
        "typeVersion": 2,
        "position": [700, 160],
        "parameters": { "jsCode": "// PLACEHOLDER - replaced in Step 6" }
      },
      {
        "id": "mem-save-001",
        "name": "Save Memory",
        "type": "n8n-nodes-base.code",
        "typeVersion": 2,
        "position": [700, 300],
        "parameters": { "jsCode": "// PLACEHOLDER - replaced in Step 7" }
      },
      {
        "id": "mem-consolidate-001",
        "name": "Consolidate Memories",
        "type": "n8n-nodes-base.code",
        "typeVersion": 2,
        "position": [700, 440],
        "parameters": { "jsCode": "// PLACEHOLDER - replaced in Step 8" }
      },
      {
        "id": "mem-status-001",
        "name": "Memory Status",
        "type": "n8n-nodes-base.code",
        "typeVersion": 2,
        "position": [700, 580],
        "parameters": { "jsCode": "// PLACEHOLDER - replaced in Step 9" }
      }
    ],
    "connections": {
      "Execute Workflow Trigger": {
        "main": [[{ "node": "Route Action", "type": "main", "index": 0 }]]
      },
      "Route Action": {
        "main": [
          [{ "node": "Search Memories", "type": "main", "index": 0 }],
          [{ "node": "Save Memory", "type": "main", "index": 0 }],
          [{ "node": "Consolidate Memories", "type": "main", "index": 0 }],
          [{ "node": "Memory Status", "type": "main", "index": 0 }]
        ]
      }
    },
    "settings": { "callerPolicy": "workflowsFromSameOwner" },
    "active": false
  }'
```

**Capture the returned workflow ID** (e.g., `AbCdEfGhIjKlMnOp`) — you will need it in Step 10 when wiring the toolWorkflow nodes.

**Verify:**
```bash
N8N_KEY=$(sqlite3 ~/.n8n/database.sqlite "SELECT apiKey FROM user_api_keys LIMIT 1;")
curl -s 'http://localhost:5678/api/v1/workflows?limit=50' -H "X-N8N-API-KEY: $N8N_KEY" | python3 -c "
import json,sys
d=json.load(sys.stdin)
[print(w['id'], w['name']) for w in d['data'] if 'Memory' in w['name']]
"
# Expected: <new-id> Susan - Memory Manager
```

---

### Step 6: Implement Search Memories Code Node

**Node ID:** `mem-search-001` in the Susan - Memory Manager workflow

**Note on async:** The Gemini embedding API calls (`embedQuery`, `embedDocuments`) return Promises and require `await`. The sqlite3 package uses callbacks which are wrapped in Promises via `dbAll()`. n8n Code nodes with `typeVersion: 2` support `async`/`await` when the top-level code returns a Promise — this discovery preference for sync code does not apply here. The implementation below uses async/await throughout.

**Action:** Update the node's `jsCode` via the n8n API (or UI) with the full async hybrid search implementation:

```javascript
// === SEARCH MEMORIES (ASYNC VERSION WITH VECTOR EMBEDDINGS) ===
// n8n Code node typeVersion 2 supports async/await

const fs = require('fs');
const crypto = require('crypto');

const SQLITE3_PATH = '/usr/local/lib/node_modules/n8n/node_modules/sqlite3';
const LANGCHAIN_PATH = '/usr/local/lib/node_modules/n8n/node_modules/@langchain/google-genai';
const DB_PATH = '/opt/claude-agents/Susan/memory.sqlite';
const EMBEDDINGS_CACHE_PATH = '/opt/claude-agents/Susan/memory/.embeddings.json';
const HALF_LIFE_DAYS = 30;
const VECTOR_WEIGHT = 0.7;
const BM25_WEIGHT = 0.3;
const CANDIDATE_MULTIPLIER = 4;

const query = items[0].json.query || '';
const limit = parseInt(items[0].json.limit) || 10;

if (!query.trim()) {
  return [{ json: { results: 'No query provided.', count: 0 } }];
}

function cosineSimilarity(a, b) {
  let dot = 0, magA = 0, magB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    magA += a[i] * a[i];
    magB += b[i] * b[i];
  }
  const denom = Math.sqrt(magA) * Math.sqrt(magB);
  return denom === 0 ? 0 : dot / denom;
}

function temporalDecay(dateStr) {
  try {
    const entryDate = new Date(dateStr);
    if (isNaN(entryDate.getTime())) return 0.5;
    const ageDays = (Date.now() - entryDate.getTime()) / (1000 * 60 * 60 * 24);
    return Math.pow(2, -ageDays / HALF_LIFE_DAYS);
  } catch(e) { return 0.5; }
}

function hashContent(text) {
  return crypto.createHash('sha256').update(text).digest('hex').slice(0, 16);
}

function loadEmbeddingsCache() {
  try {
    return JSON.parse(fs.readFileSync(EMBEDDINGS_CACHE_PATH, 'utf8')) || {};
  } catch (e) { return {}; }
}

function saveEmbeddingsCache(cache) {
  try { fs.writeFileSync(EMBEDDINGS_CACHE_PATH, JSON.stringify(cache), 'utf8'); } catch(e) {}
}

// Promisified DB query
function dbAll(db, sql, params) {
  return new Promise((resolve, reject) => {
    db.all(sql, params, (err, rows) => {
      if (err) reject(err); else resolve(rows || []);
    });
  });
}

async function run() {
  const sqlite3 = require(SQLITE3_PATH).verbose();
  const db = new sqlite3.Database(DB_PATH);

  // Promisify close
  const dbClose = () => new Promise(r => db.close(r));

  let bm25Results = [];
  let allEntries = [];

  const candidateLimit = limit * CANDIDATE_MULTIPLIER;
  const ftsQuery = query.replace(/['"]/g, ' ').replace(/[*+\-()]/g, ' ').trim();

  try {
    if (ftsQuery.length > 0) {
      bm25Results = await dbAll(db,
        `SELECT id, content, date, category, source_file, rank FROM memory_fts WHERE memory_fts MATCH ? ORDER BY rank LIMIT ?`,
        [ftsQuery, candidateLimit]
      );
    }
  } catch(e) { bm25Results = []; }

  try {
    allEntries = await dbAll(db,
      `SELECT id, content, date, category, source_file FROM memory_entries ORDER BY date DESC LIMIT ?`,
      [candidateLimit]
    );
  } catch(e) { allEntries = []; }

  await dbClose();

  if (allEntries.length === 0 && bm25Results.length === 0) {
    return [{ json: { results: 'No memories found yet.', count: 0 } }];
  }

  // BM25 scores
  const MAX_BM25_RANK = 50;
  const bm25ScoreMap = {};
  for (const row of bm25Results) {
    bm25ScoreMap[row.id] = Math.max(0, 1 - Math.abs(row.rank || 0) / MAX_BM25_RANK);
  }

  // Vector search (lazy embeddings with cache)
  let vectorScoreMap = {};
  let embCache = loadEmbeddingsCache();
  let cacheUpdated = false;

  try {
    const apiKey = process.env.GOOGLE_API_KEY || '';
    if (apiKey) {
      const { GoogleGenerativeAIEmbeddings } = require(LANGCHAIN_PATH);
      const embedder = new GoogleGenerativeAIEmbeddings({ modelName: 'text-embedding-004', apiKey });

      // Get query embedding
      const queryVec = await embedder.embedQuery(query);

      // Get/generate embeddings for all entries (batch uncached ones)
      const uncachedEntries = allEntries.filter(e => !embCache[hashContent(e.content)]);
      if (uncachedEntries.length > 0) {
        const batchTexts = uncachedEntries.map(e => e.content);
        // Process in batches of 20 to stay within rate limits
        const BATCH_SIZE = 20;
        for (let i = 0; i < batchTexts.length; i += BATCH_SIZE) {
          const batch = batchTexts.slice(i, i + BATCH_SIZE);
          try {
            const vecs = await embedder.embedDocuments(batch);
            for (let j = 0; j < batch.length; j++) {
              const key = hashContent(uncachedEntries[i + j].content);
              embCache[key] = vecs[j];
              cacheUpdated = true;
            }
          } catch(batchErr) {
            // Partial failure OK — continue with what we have
            break;
          }
        }
      }

      // Score all entries
      for (const entry of allEntries) {
        const key = hashContent(entry.content);
        const entryVec = embCache[key];
        if (entryVec && queryVec) {
          vectorScoreMap[entry.id] = cosineSimilarity(queryVec, entryVec);
        }
      }

      if (cacheUpdated) saveEmbeddingsCache(embCache);
    }
  } catch(embErr) {
    // Graceful degradation: BM25-only search continues
    vectorScoreMap = {};
  }

  // Build candidate map
  const candidateMap = {};
  for (const row of bm25Results) {
    candidateMap[row.id] = { ...row, bm25_score: bm25ScoreMap[row.id] || 0, vector_score: vectorScoreMap[row.id] || 0 };
  }
  for (const id in vectorScoreMap) {
    if (!candidateMap[id]) {
      const entry = allEntries.find(e => e.id === id);
      if (entry) candidateMap[id] = { ...entry, bm25_score: bm25ScoreMap[id] || 0, vector_score: vectorScoreMap[id] || 0 };
    }
  }
  // Fallback: include recent entries if nothing matched
  if (Object.keys(candidateMap).length === 0) {
    for (const entry of allEntries.slice(0, limit)) {
      candidateMap[entry.id] = { ...entry, bm25_score: 0, vector_score: 0 };
    }
  }

  // Score and rank
  const scored = Object.values(candidateMap).map(c => ({
    ...c,
    final_score: ((VECTOR_WEIGHT * (c.vector_score || 0)) + (BM25_WEIGHT * (c.bm25_score || 0))) * temporalDecay(c.date)
  }));
  scored.sort((a, b) => b.final_score - a.final_score);
  const topK = scored.slice(0, limit);

  if (topK.length === 0) {
    return [{ json: { results: 'No relevant memories found for that query.', count: 0 } }];
  }

  const lines = topK.map((m, i) => {
    const dateStr = m.date || 'unknown date';
    const cat = m.category || 'general';
    return `[${i+1}] ${dateStr} (${cat}): ${m.content}`;
  });

  const prose = `Found ${topK.length} relevant ${topK.length === 1 ? 'memory' : 'memories'}:\n\n` + lines.join('\n\n');
  return [{ json: { results: prose, count: topK.length } }];
}

return run();
```

**Verify:** In the n8n UI, manually execute the sub-workflow with test input:
```json
{ "action": "search", "query": "invoice preferences", "limit": 5 }
```
Expected: `{ results: "No memories found yet.", count: 0 }` (since DB is empty at this point)

---

### Step 7: Implement Save Memory Code Node

**Node ID:** `mem-save-001` in the Susan - Memory Manager workflow

**Action:** Update the node's `jsCode` with the full save implementation:

```javascript
// === SAVE MEMORY ===
// Input: $json.content (string), $json.category (optional, default 'general')
// Output: { saved: true, id: string, file: string, message: string }

const fs = require('fs');
const crypto = require('crypto');
const path = require('path');

const SQLITE3_PATH = '/usr/local/lib/node_modules/n8n/node_modules/sqlite3';
const DB_PATH = '/opt/claude-agents/Susan/memory.sqlite';
const MEMORY_DIR = '/opt/claude-agents/Susan/memory';

// Get input (n8n Code node exposes items array)
const rawInput = items[0].json;

// Auto-sanitize input (silent — per discovery decision)
let content = '';
let category = 'general';

if (typeof rawInput.content === 'string') {
  content = rawInput.content.trim();
} else if (typeof rawInput.query === 'string') {
  // Fallback: some callers pass 'query' (old tool pattern)
  const q = rawInput.query;
  // If it's JSON, try to extract content field
  try {
    const parsed = JSON.parse(q);
    content = (parsed.content || parsed.text || q).toString().trim();
    category = parsed.category || rawInput.category || 'general';
  } catch(e) {
    content = q.trim();
  }
}

if (rawInput.category) {
  category = rawInput.category.toString().toLowerCase().trim();
}

// Validate and sanitize category
const VALID_CATEGORIES = ['routing', 'preference', 'pattern', 'session', 'agent_gap', 'agent_improvement', 'general', 'business', 'contact', 'task'];
if (!VALID_CATEGORIES.includes(category)) {
  // Auto-correct to 'general' — silent sanitization
  category = 'general';
}

// Strip any JSON objects from content (per discovery: "auto-fix bad input")
if (content.startsWith('{') || content.startsWith('[')) {
  try {
    const parsed = JSON.parse(content);
    content = parsed.content || parsed.text || parsed.note || JSON.stringify(parsed);
  } catch(e) {
    // Not valid JSON, use as-is
  }
}

if (!content || content.length === 0) {
  return [{ json: { saved: false, message: 'No content provided' } }];
}

// Generate unique ID and date
const today = new Date().toISOString().split('T')[0]; // YYYY-MM-DD
const id = crypto.randomBytes(8).toString('hex');
const timestamp = new Date().toISOString();

// ---- Write to daily log file ----
const dailyLogPath = path.join(MEMORY_DIR, `${today}.md`);
const logEntry = `\n## [${timestamp}] ${category}\n${content}\n`;

try {
  fs.appendFileSync(dailyLogPath, logEntry, 'utf8');
} catch(e) {
  return [{ json: { saved: false, message: `Failed to write log: ${e.message}` } }];
}

// ---- Index in SQLite ----
function dbRun(db, sql, params) {
  return new Promise((resolve, reject) => {
    db.run(sql, params, function(err) {
      if (err) reject(err); else resolve(this);
    });
  });
}

async function saveToDb() {
  const sqlite3 = require(SQLITE3_PATH).verbose();
  const db = new sqlite3.Database(DB_PATH);
  const dbClose = () => new Promise(r => db.close(r));

  try {
    // Insert into memory_entries
    await dbRun(db,
      `INSERT OR REPLACE INTO memory_entries (id, date, category, source_file, content, embedding_cached) VALUES (?, ?, ?, ?, ?, 0)`,
      [id, today, category, dailyLogPath, content]
    );

    // Insert into FTS5 table
    await dbRun(db,
      `INSERT INTO memory_fts (id, content, date, category, source_file) VALUES (?, ?, ?, ?, ?)`,
      [id, content, today, category, dailyLogPath]
    );

    // Update daily_logs metadata
    await dbRun(db,
      `INSERT INTO daily_logs (date, path, last_indexed, entry_count)
       VALUES (?, ?, datetime('now'), 1)
       ON CONFLICT(date) DO UPDATE SET
         last_indexed = datetime('now'),
         entry_count = entry_count + 1`,
      [today, dailyLogPath]
    );

    await dbClose();
    return { saved: true, id, file: dailyLogPath, message: `Memory saved: ${category}` };
  } catch(e) {
    await dbClose();
    // SQLite failed but file was written — return partial success
    return { saved: true, id, file: dailyLogPath, message: `Saved to file (DB index failed: ${e.message})` };
  }
}

return saveToDb().then(result => [{ json: result }]);
```

**Verify:** Execute sub-workflow with:
```json
{ "action": "save", "content": "preference: Adam likes brief Slack replies", "category": "preference" }
```
Expected: `{ "saved": true, "id": "...", "file": "/opt/claude-agents/Susan/memory/2026-02-19.md", "message": "Memory saved: preference" }`

Then verify the file exists:
```bash
ls /opt/claude-agents/Susan/memory/
cat /opt/claude-agents/Susan/memory/*.md
```

---

### Step 8: Implement Consolidate Memories Code Node

**Node ID:** `mem-consolidate-001` in the Susan - Memory Manager workflow

**Action:** Update the node's `jsCode`:

```javascript
// === CONSOLIDATE MEMORIES ===
// Reads daily logs from past 90 days, calls Gemini to synthesize patterns,
// updates MEMORY.md, purges old log files. Silent operation.
// Input: $json.action = "consolidate"
// Output: { consolidated: true, entries_processed: int, message: string }

const fs = require('fs');
const path = require('path');
const https = require('https');

const MEMORY_DIR = '/opt/claude-agents/Susan/memory';
const MEMORY_MD = path.join(MEMORY_DIR, 'MEMORY.md');
const SQLITE3_PATH = '/usr/local/lib/node_modules/n8n/node_modules/sqlite3';
const DB_PATH = '/opt/claude-agents/Susan/memory.sqlite';
const RETENTION_DAYS = 90;

function dbRun(db, sql, params) {
  return new Promise((resolve, reject) => {
    db.run(sql, params, function(err) { if (err) reject(err); else resolve(this); });
  });
}
function dbAll(db, sql, params) {
  return new Promise((resolve, reject) => {
    db.all(sql, params, (err, rows) => { if (err) reject(err); else resolve(rows || []); });
  });
}

// Call Gemini directly via HTTPS (avoids n8n LangChain overhead)
function callGemini(prompt) {
  return new Promise((resolve, reject) => {
    const apiKey = process.env.GOOGLE_API_KEY || '';
    if (!apiKey) { reject(new Error('No GOOGLE_API_KEY')); return; }

    const body = JSON.stringify({
      contents: [{ parts: [{ text: prompt }] }],
      generationConfig: { temperature: 0.3, maxOutputTokens: 2048 }
    });

    const options = {
      hostname: 'generativelanguage.googleapis.com',
      path: `/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`,
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Content-Length': Buffer.byteLength(body) }
    };

    const req = https.request(options, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => {
        try {
          const parsed = JSON.parse(data);
          const text = parsed?.candidates?.[0]?.content?.parts?.[0]?.text || '';
          resolve(text);
        } catch(e) { reject(new Error('Gemini parse error: ' + data.slice(0, 200))); }
      });
    });
    req.on('error', reject);
    req.write(body);
    req.end();
  });
}

async function consolidate() {
  const sqlite3 = require(SQLITE3_PATH).verbose();
  const db = new sqlite3.Database(DB_PATH);
  const dbClose = () => new Promise(r => db.close(r));

  // Get all entries from past 90 days
  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - RETENTION_DAYS);
  const cutoffStr = cutoffDate.toISOString().split('T')[0];

  const recentEntries = await dbAll(db,
    `SELECT date, category, content FROM memory_entries WHERE date >= ? ORDER BY date ASC`,
    [cutoffStr]
  );

  if (recentEntries.length === 0) {
    await dbClose();
    return [{ json: { consolidated: false, entries_processed: 0, message: 'No entries to consolidate' } }];
  }

  // Read current MEMORY.md
  let currentMemory = '';
  try { currentMemory = fs.readFileSync(MEMORY_MD, 'utf8'); } catch(e) {}

  // Build consolidation prompt
  const entriesText = recentEntries.map(e =>
    `[${e.date}] ${e.category}: ${e.content}`
  ).join('\n');

  const prompt = `You are Susan, Adam's personal AI assistant. You are reviewing recent memory entries to update your long-term memory file.

CURRENT MEMORY.MD:
${currentMemory || '(empty)'}

RECENT ENTRIES (${recentEntries.length} entries from the past ${RETENTION_DAYS} days):
${entriesText}

TASK: Update MEMORY.md by:
1. Keeping all important long-term facts about Adam, his preferences, his business, and his patterns
2. Adding new insights from the recent entries that aren't already captured
3. Removing outdated or superseded information
4. Keeping it as natural prose — readable narrative, not bullet points or YAML
5. Organizing by topic naturally (e.g., communication preferences, business context, routing shortcuts)
6. Keep it concise — this is a memory aide, not a journal

Return ONLY the updated MEMORY.md content. Start with the header "# Susan's Memory" and write the updated file.`;

  let updatedMemory = '';
  try {
    updatedMemory = await callGemini(prompt);
  } catch(e) {
    await dbClose();
    return [{ json: { consolidated: false, entries_processed: recentEntries.length, message: `Gemini call failed: ${e.message}` } }];
  }

  // Write updated MEMORY.md
  if (updatedMemory.trim()) {
    fs.writeFileSync(MEMORY_MD, updatedMemory.trim() + '\n', 'utf8');
  }

  // Purge log files older than 90 days (but keep entries in DB for hybrid search)
  const now = new Date();
  const files = fs.readdirSync(MEMORY_DIR);
  let purgedCount = 0;
  for (const file of files) {
    if (!file.match(/^\d{4}-\d{2}-\d{2}\.md$/)) continue;
    const fileDate = new Date(file.replace('.md', ''));
    const ageDays = (now - fileDate) / (1000 * 60 * 60 * 24);
    if (ageDays > RETENTION_DAYS) {
      try {
        fs.unlinkSync(path.join(MEMORY_DIR, file));
        purgedCount++;
      } catch(e) {}
    }
  }

  // Update system_state with last consolidation time
  await dbRun(db,
    `INSERT INTO system_state (key, value, updated_at) VALUES ('last_consolidation', datetime('now'), datetime('now'))
     ON CONFLICT(key) DO UPDATE SET value = datetime('now'), updated_at = datetime('now')`,
    []
  );

  await dbClose();

  return [{ json: {
    consolidated: true,
    entries_processed: recentEntries.length,
    files_purged: purgedCount,
    message: `Consolidated ${recentEntries.length} entries, purged ${purgedCount} old log files`
  }}];
}

return consolidate();
```

---

### Step 9: Implement Memory Status Code Node

**Node ID:** `mem-status-001` in the Susan - Memory Manager workflow

**Action:** Update the node's `jsCode`:

```javascript
// === MEMORY STATUS ===
// Returns health and statistics for the memory system
// Input: $json.action = "status"
// Output: { total_entries, last_consolidation, db_size_kb, memory_md_size_kb, log_files, health }

const fs = require('fs');
const path = require('path');

const SQLITE3_PATH = '/usr/local/lib/node_modules/n8n/node_modules/sqlite3';
const DB_PATH = '/opt/claude-agents/Susan/memory.sqlite';
const MEMORY_DIR = '/opt/claude-agents/Susan/memory';
const MEMORY_MD = path.join(MEMORY_DIR, 'MEMORY.md');

function dbGet(db, sql, params) {
  return new Promise((resolve, reject) => {
    db.get(sql, params, (err, row) => { if (err) reject(err); else resolve(row); });
  });
}

async function getStatus() {
  const sqlite3 = require(SQLITE3_PATH).verbose();
  const db = new sqlite3.Database(DB_PATH);
  const dbClose = () => new Promise(r => db.close(r));

  let totalEntries = 0;
  let lastConsolidation = 'never';
  let embeddingsCached = 0;
  let dbSizeKb = 0;

  try {
    const countRow = await dbGet(db, 'SELECT COUNT(*) as cnt FROM memory_entries', []);
    totalEntries = countRow?.cnt || 0;
  } catch(e) {}

  try {
    const stateRow = await dbGet(db, "SELECT value FROM system_state WHERE key = 'last_consolidation'", []);
    lastConsolidation = stateRow?.value || 'never';
  } catch(e) {}

  try {
    const embRow = await dbGet(db, 'SELECT COUNT(*) as cnt FROM embeddings_cache', []);
    embeddingsCached = embRow?.cnt || 0;
  } catch(e) {}

  await dbClose();

  // DB file size
  try {
    const stats = fs.statSync(DB_PATH);
    dbSizeKb = Math.round(stats.size / 1024);
  } catch(e) {}

  // MEMORY.md size
  let memoryMdSizeKb = 0;
  try {
    const stats = fs.statSync(MEMORY_MD);
    memoryMdSizeKb = Math.round(stats.size / 1024);
  } catch(e) {}

  // Count daily log files
  let logFiles = 0;
  let oldestLog = null;
  try {
    const files = fs.readdirSync(MEMORY_DIR).filter(f => f.match(/^\d{4}-\d{2}-\d{2}\.md$/));
    logFiles = files.length;
    if (files.length > 0) {
      oldestLog = files.sort()[0].replace('.md', '');
    }
  } catch(e) {}

  // Health assessment
  let health = 'healthy';
  if (totalEntries === 0) health = 'empty';
  else if (lastConsolidation === 'never' && totalEntries > 50) health = 'needs_consolidation';

  const status = {
    total_entries: totalEntries,
    embeddings_cached: embeddingsCached,
    last_consolidation: lastConsolidation,
    db_size_kb: dbSizeKb,
    memory_md_size_kb: memoryMdSizeKb,
    log_files: logFiles,
    oldest_log: oldestLog,
    health: health
  };

  const prose = `Memory System Status:
- Total entries indexed: ${totalEntries}
- Embeddings cached: ${embeddingsCached}
- Last consolidation: ${lastConsolidation}
- Database size: ${dbSizeKb} KB
- MEMORY.md size: ${memoryMdSizeKb} KB
- Daily log files: ${logFiles}${oldestLog ? ' (oldest: ' + oldestLog + ')' : ''}
- Health: ${health}`;

  return [{ json: { ...status, prose } }];
}

return getStatus();
```

---

### CHECKPOINT: Sub-Workflow Functional (Steps 5-9)

Before proceeding to wire the tools into the main Susan workflow, verify the sub-workflow operates correctly.

**In the n8n UI**, open "Susan - Memory Manager" and run with manual trigger:

**Test 1 — Save:**
```json
{ "action": "save", "content": "preference: Adam likes brief Slack replies, not verbose explanations", "category": "preference" }
```
Expected: `{ "saved": true, "id": "...", "file": "...2026-02-19.md" }`

**Test 2 — Search (after save):**
```json
{ "action": "search", "query": "Adam communication preferences Slack", "limit": 5 }
```
Expected: `{ "results": "Found 1 relevant memory:\n\n[1] 2026-02-19 (preference): preference: Adam likes brief Slack replies...", "count": 1 }`

**Test 3 — Status:**
```json
{ "action": "status" }
```
Expected: `{ "total_entries": 1, "health": "empty" or "healthy", "log_files": 1 }`

**Verify files on adam-server:**
```bash
ls -la /opt/claude-agents/Susan/memory/
cat /opt/claude-agents/Susan/memory/*.md
/opt/homebrew/opt/node@22/bin/node -e "
const sqlite3 = require('/usr/local/lib/node_modules/n8n/node_modules/sqlite3').verbose();
const db = new sqlite3.Database('/opt/claude-agents/Susan/memory.sqlite');
db.all('SELECT id, date, category, content FROM memory_entries', (e,r) => { console.log(JSON.stringify(r,null,2)); db.close(); });
"
```

---

### Step 10: Wire Tool Nodes into Susan Main Workflow

**Workflow:** Susan (ID: `U7cK5XlQqmgG9CWlrB6wM`)
**Method:** n8n REST API PUT to update workflow

This step:
1. Removes nodes: `Read Memories` (91c95fff), `Save Memory` (6372c151), `Simple Memory` (b22f65cb)
2. Adds three new `toolWorkflow` nodes pointing to the Memory Manager sub-workflow
3. Updates connections

**Required:** The Memory Manager sub-workflow ID from Step 5 (e.g., `AbCdEfGhIjKlMnOp`)

#### 10a: New toolWorkflow node definitions

Add these three nodes to the Susan main workflow. They replace the three removed nodes at similar positions:

**Search Memories toolWorkflow** (replaces old Read Memories at position [2000, -864]):
```json
{
  "id": "mem-search-tool-001",
  "name": "Search Memories",
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "typeVersion": 2,
  "position": [2000, -864],
  "parameters": {
    "name": "search_memories",
    "description": "Search Susan's memory system using a natural language query. Returns the most relevant memories about Adam's preferences, business context, routing shortcuts, and past interactions. Call this whenever you need to recall something or check what you know about a topic. Provide a descriptive query explaining what you are looking for.",
    "workflowId": {
      "__rl": true,
      "value": "MEMORY_MANAGER_WORKFLOW_ID",
      "mode": "id"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "action": "search",
        "query": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI(\"query\", \"Natural language description of what memories to search for\", \"string\") }}",
        "limit": "10"
      },
      "matchingColumns": [],
      "schema": [
        { "id": "action", "displayName": "action", "required": true, "defaultMatch": false, "canBeUsedToMatch": false, "display": true, "type": "string", "removed": false },
        { "id": "query", "displayName": "query", "required": true, "defaultMatch": false, "canBeUsedToMatch": false, "display": true, "type": "string", "removed": false },
        { "id": "limit", "displayName": "limit", "required": false, "defaultMatch": false, "canBeUsedToMatch": false, "display": true, "type": "string", "removed": false }
      ],
      "attemptToConvertTypes": false,
      "convertFieldsToString": true
    }
  }
}
```

**Save Memory toolWorkflow** (replaces old Save Memory at position [2128, -864]):
```json
{
  "id": "mem-save-tool-001",
  "name": "Save Memory",
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "typeVersion": 2,
  "position": [2128, -864],
  "parameters": {
    "name": "save_memory",
    "description": "Save a memory to Susan's persistent memory system. Use this to remember preferences, routing decisions, patterns, session references, or anything Adam has corrected. Write plain text (1-3 sentences). Never save JSON, raw API responses, or technical data. Categories: routing, preference, pattern, session, agent_gap, agent_improvement, business, contact, general.",
    "workflowId": {
      "__rl": true,
      "value": "MEMORY_MANAGER_WORKFLOW_ID",
      "mode": "id"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "action": "save",
        "content": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI(\"content\", \"The memory to save as plain text (1-3 sentences)\", \"string\") }}",
        "category": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI(\"category\", \"Memory category: routing, preference, pattern, session, agent_gap, agent_improvement, business, contact, or general\", \"string\") }}"
      },
      "matchingColumns": [],
      "schema": [
        { "id": "action", "displayName": "action", "required": true, "defaultMatch": false, "canBeUsedToMatch": false, "display": true, "type": "string", "removed": false },
        { "id": "content", "displayName": "content", "required": true, "defaultMatch": false, "canBeUsedToMatch": false, "display": true, "type": "string", "removed": false },
        { "id": "category", "displayName": "category", "required": false, "defaultMatch": false, "canBeUsedToMatch": false, "display": true, "type": "string", "removed": false }
      ],
      "attemptToConvertTypes": false,
      "convertFieldsToString": true
    }
  }
}
```

**Memory Status + Consolidate toolWorkflow** (new tool at position [2256, -864]):
```json
{
  "id": "mem-status-tool-001",
  "name": "Memory Manager",
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "typeVersion": 2,
  "position": [2256, -864],
  "parameters": {
    "name": "memory_manager",
    "description": "Check memory system health (action: status) or trigger memory consolidation (action: consolidate). Use 'status' to see total entries, last consolidation time, and system health. Use 'consolidate' when you want to synthesize recent daily logs into MEMORY.md — do this silently when you judge it is appropriate (e.g., after 20+ new memories have been saved).",
    "workflowId": {
      "__rl": true,
      "value": "MEMORY_MANAGER_WORKFLOW_ID",
      "mode": "id"
    },
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "action": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI(\"action\", \"Either 'status' to check health or 'consolidate' to run consolidation\", \"string\") }}"
      },
      "matchingColumns": [],
      "schema": [
        { "id": "action", "displayName": "action", "required": true, "defaultMatch": false, "canBeUsedToMatch": false, "display": true, "type": "string", "removed": false }
      ],
      "attemptToConvertTypes": false,
      "convertFieldsToString": true
    }
  }
}
```

#### 10b: Apply the workflow update via API

**Script to run on adam-server** (after confirming Memory Manager workflow ID):

```bash
N8N_KEY=$(sqlite3 ~/.n8n/database.sqlite "SELECT apiKey FROM user_api_keys LIMIT 1;")
MEMORY_WF_ID="<REPLACE_WITH_ACTUAL_MEMORY_MANAGER_WORKFLOW_ID>"

# Get current Susan workflow
curl -s "http://localhost:5678/api/v1/workflows/U7cK5XlQqmgG9CWlrB6wM" \
  -H "X-N8N-API-KEY: $N8N_KEY" > /tmp/susan_current.json

echo "Current node count: $(python3 -c "import json; d=json.load(open('/tmp/susan_current.json')); print(len(d['nodes']))")"
```

The actual node additions and removals must be done by constructing the updated workflow JSON. Given the large size of the Susan workflow (87KB), the safest approach is to:

1. Download the current workflow JSON
2. Use a Python script to add new nodes, remove old ones, and update connections
3. PUT the updated JSON back

**Python script to transform the workflow** (run locally or on adam-server):

```python
import json

with open('/tmp/susan_current.json') as f:
    workflow = json.load(f)

MEMORY_WF_ID = "REPLACE_WITH_ACTUAL_ID"

# IDs to remove
REMOVE_IDS = {
    '91c95fff-efe2-46c6-9d13-94dfbaeee761',  # Read Memories (toolCode)
    '6372c151-6e10-4e4a-863b-db1b366c2a9b',  # Save Memory (toolCode)
    'b22f65cb-33d5-4b75-84d5-30515670d5df'   # Simple Memory (memoryBufferWindow)
}

# Remove old nodes
workflow['nodes'] = [n for n in workflow['nodes'] if n['id'] not in REMOVE_IDS]

# Add new toolWorkflow nodes
new_nodes = [
    {
        "id": "mem-search-tool-001",
        "name": "Search Memories",
        "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
        "typeVersion": 2,
        "position": [2000, -864],
        "parameters": {
            "name": "search_memories",
            "description": "Search Susan's memory system using a natural language query. Returns the most relevant memories about Adam's preferences, business context, routing shortcuts, and past interactions. Call this whenever you need to recall something or check what you know about a topic. Provide a descriptive query explaining what you are looking for.",
            "workflowId": {"__rl": True, "value": MEMORY_WF_ID, "mode": "id"},
            "workflowInputs": {
                "mappingMode": "defineBelow",
                "value": {
                    "action": "search",
                    "query": '={{ /*n8n-auto-generated-fromAI-override*/ $fromAI("query", "Natural language description of what memories to search for", "string") }}',
                    "limit": "10"
                },
                "matchingColumns": [],
                "schema": [
                    {"id": "action", "displayName": "action", "required": True, "defaultMatch": False, "canBeUsedToMatch": False, "display": True, "type": "string", "removed": False},
                    {"id": "query", "displayName": "query", "required": True, "defaultMatch": False, "canBeUsedToMatch": False, "display": True, "type": "string", "removed": False},
                    {"id": "limit", "displayName": "limit", "required": False, "defaultMatch": False, "canBeUsedToMatch": False, "display": True, "type": "string", "removed": False}
                ],
                "attemptToConvertTypes": False,
                "convertFieldsToString": True
            }
        }
    },
    {
        "id": "mem-save-tool-001",
        "name": "Save Memory",
        "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
        "typeVersion": 2,
        "position": [2128, -864],
        "parameters": {
            "name": "save_memory",
            "description": "Save a memory to Susan's persistent memory system. Use this to remember preferences, routing decisions, patterns, session references, or anything Adam has corrected. Write plain text (1-3 sentences). Never save JSON, raw API responses, or technical data. Categories: routing, preference, pattern, session, agent_gap, agent_improvement, business, contact, general.",
            "workflowId": {"__rl": True, "value": MEMORY_WF_ID, "mode": "id"},
            "workflowInputs": {
                "mappingMode": "defineBelow",
                "value": {
                    "action": "save",
                    "content": '={{ /*n8n-auto-generated-fromAI-override*/ $fromAI("content", "The memory to save as plain text (1-3 sentences)", "string") }}',
                    "category": '={{ /*n8n-auto-generated-fromAI-override*/ $fromAI("category", "Memory category: routing, preference, pattern, session, agent_gap, agent_improvement, business, contact, or general", "string") }}'
                },
                "matchingColumns": [],
                "schema": [
                    {"id": "action", "displayName": "action", "required": True, "defaultMatch": False, "canBeUsedToMatch": False, "display": True, "type": "string", "removed": False},
                    {"id": "content", "displayName": "content", "required": True, "defaultMatch": False, "canBeUsedToMatch": False, "display": True, "type": "string", "removed": False},
                    {"id": "category", "displayName": "category", "required": False, "defaultMatch": False, "canBeUsedToMatch": False, "display": True, "type": "string", "removed": False}
                ],
                "attemptToConvertTypes": False,
                "convertFieldsToString": True
            }
        }
    },
    {
        "id": "mem-status-tool-001",
        "name": "Memory Manager",
        "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
        "typeVersion": 2,
        "position": [2256, -864],
        "parameters": {
            "name": "memory_manager",
            "description": "Check memory system health (action: status) or trigger memory consolidation (action: consolidate). Use 'status' to see total entries, last consolidation time, and system health. Use 'consolidate' when you want to synthesize recent daily logs into MEMORY.md — do this silently when you judge it is appropriate.",
            "workflowId": {"__rl": True, "value": MEMORY_WF_ID, "mode": "id"},
            "workflowInputs": {
                "mappingMode": "defineBelow",
                "value": {
                    "action": '={{ /*n8n-auto-generated-fromAI-override*/ $fromAI("action", "Either \'status\' to check health or \'consolidate\' to run consolidation", "string") }}'
                },
                "matchingColumns": [],
                "schema": [
                    {"id": "action", "displayName": "action", "required": True, "defaultMatch": False, "canBeUsedToMatch": False, "display": True, "type": "string", "removed": False}
                ],
                "attemptToConvertTypes": False,
                "convertFieldsToString": True
            }
        }
    }
]

workflow['nodes'].extend(new_nodes)

# Update connections: remove old nodes from connections, add new ones
conns = workflow.get('connections', {})

# Remove old node connection sources
for old_name in ['Read Memories', 'Save Memory', 'Simple Memory']:
    conns.pop(old_name, None)

# Remove old nodes as targets in other nodes' connections
for src_name, src_conns in conns.items():
    for conn_type, conn_list in src_conns.items():
        for i, group in enumerate(conn_list):
            conn_list[i] = [c for c in group if c.get('node') not in ['Read Memories', 'Save Memory', 'Simple Memory']]

# Add new tool connections to Susan agent (ai_tool type)
for new_node_name in ['Search Memories', 'Save Memory', 'Memory Manager']:
    conns[new_node_name] = {
        'ai_tool': [[{'node': 'Susan', 'type': 'ai_tool', 'index': 0}]]
    }

workflow['connections'] = conns

# Save transformed workflow
with open('/tmp/susan_updated.json', 'w') as f:
    json.dump(workflow, f, indent=2)

print(f"Updated workflow: {len(workflow['nodes'])} nodes")
print("Removed:", REMOVE_IDS)
print("Added: Search Memories, Save Memory, Memory Manager (toolWorkflow)")
```

**Then PUT the updated workflow:**
```bash
N8N_KEY=$(sqlite3 ~/.n8n/database.sqlite "SELECT apiKey FROM user_api_keys LIMIT 1;")
curl -s -X PUT "http://localhost:5678/api/v1/workflows/U7cK5XlQqmgG9CWlrB6wM" \
  -H "X-N8N-API-KEY: $N8N_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/susan_updated.json | python3 -c "import json,sys; d=json.load(sys.stdin); print('Updated at:', d.get('updatedAt')); print('Node count:', len(d.get('nodes',[])))"
```

Expected: `Updated at: 2026-02-19T...` and node count should be previous count - 3 + 3 = same (39 nodes).

---

### Step 11: Update Standardize Incoming Channel Node (memory_needed flag)

**Node:** `standardize-channel-001` in Susan main workflow

**Action:** Add `memory_needed` to the return object at the end of the node. The logic is:
- `comment_review` type → `memory_needed: false`
- All other types (email, slack) → `memory_needed: true`

**Change the final return statement** from:
```javascript
return [{
  json: {
    text,
    incoming_channel
  },
  binary
}];
```

**To:**
```javascript
// Memory is needed for interactive channels, not scheduled tasks
const memory_needed = incoming_channel.type !== 'comment_review';

return [{
  json: {
    text,
    incoming_channel,
    memory_needed
  },
  binary
}];
```

**Apply via n8n API** — re-download the current workflow (which now includes Step 10's toolWorkflow nodes), modify with the Python script below, PUT. The node ID is `standardize-channel-001`.

**Re-download the current workflow** (REQUIRED — ensures you have Step 10's changes, not stale data):
```bash
N8N_KEY=$(sqlite3 ~/.n8n/database.sqlite "SELECT apiKey FROM user_api_keys LIMIT 1;")
curl -s "http://localhost:5678/api/v1/workflows/U7cK5XlQqmgG9CWlrB6wM" \
  -H "X-N8N-API-KEY: $N8N_KEY" > /tmp/susan_current.json
echo "Re-downloaded post-Step-10 workflow: $(python3 -c "import json; d=json.load(open('/tmp/susan_current.json')); print(len(d['nodes']), 'nodes')")"
```

**Python script to apply the change** (run on adam-server after re-downloading the workflow to `/tmp/susan_current.json`):

```python
import json, re

with open('/tmp/susan_current.json') as f:
    workflow = json.load(f)

# Locate the standardize-channel-001 node
node = next((n for n in workflow['nodes'] if n['id'] == 'standardize-channel-001'), None)
if not node:
    raise SystemExit('ERROR: standardize-channel-001 not found in workflow')

old_return = """return [{
  json: {
    text,
    incoming_channel
  },
  binary
}];"""

new_return = """// Memory is needed for interactive channels, not scheduled tasks
const memory_needed = incoming_channel.type !== 'comment_review';

return [{
  json: {
    text,
    incoming_channel,
    memory_needed
  },
  binary
}];"""

original = node['parameters']['jsCode']
updated = original.replace(old_return, new_return)
if updated == original:
    raise SystemExit('ERROR: return block not found — check exact whitespace in jsCode')

node['parameters']['jsCode'] = updated

with open('/tmp/susan_updated.json', 'w') as f:
    json.dump(workflow, f, indent=2)

print('Updated standardize-channel-001 jsCode — ready to PUT')
```

**Then PUT the updated workflow:**
```bash
N8N_KEY=$(sqlite3 ~/.n8n/database.sqlite "SELECT apiKey FROM user_api_keys LIMIT 1;")
curl -s -X PUT "http://localhost:5678/api/v1/workflows/U7cK5XlQqmgG9CWlrB6wM" \
  -H "X-N8N-API-KEY: $N8N_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/susan_updated.json | python3 -c "import json,sys; d=json.load(sys.stdin); print('Updated at:', d.get('updatedAt'))"
```

**Verify:** After the update, run a manual trigger and inspect the output of the Standardize Incoming Channel node to confirm `memory_needed: true` appears in the JSON output.

---

### Step 12: Update Susan Agent System Prompt

**Node:** Susan AI agent (`6e8ee7fe-bce9-4d86-9b25-09635328cc31`)

**Action:** Replace the memory section in `options.systemMessage`. The current text to find and replace:

**FIND:**
```
## #1 Skill: Continuous Learning
You HATE doing things twice. After every interaction:
- Notice what worked, what was slow, what Adam corrected
- Save learnings using the "Save Memory" tool — short plain-text notes only
- Memories are cues: routing shortcuts, preferences, patterns, session references
- NEVER save JSON, raw API responses, workflow definitions, or technical data as memories
- ALWAYS call "Read Memories" at the start to recall what you've learned
- Apply learnings proactively. Each conversation makes you smarter.
```

**REPLACE WITH:**
```
## #1 Skill: Continuous Learning
You HATE doing things twice. After every interaction:
- Notice what worked, what was slow, what Adam corrected
- Save learnings using the "Save Memory" tool — plain text notes, 1-3 sentences
- NEVER save JSON, raw API responses, workflow definitions, or technical data as memories
- Categories: routing, preference, pattern, session, agent_gap, agent_improvement, business, contact, general
- Apply learnings proactively. Each conversation makes you smarter.

## Memory System
Your memory uses hybrid search (semantic + keyword) with temporal decay. Recent memories surface more strongly than old ones.

**Search Memories**: Call this at the start of EVERY interactive conversation (when memory_needed is true). Provide a descriptive query about what you are looking for — e.g., "Adam invoice preferences Accountant agent routing" rather than just "memories".

**Save Memory**: Save meaningful learnings after EVERY interaction that reveals something new about Adam, his preferences, or your routing decisions. Write naturally: "Adam prefers the Accountant agent for all FreshBooks invoice questions" not "invoice:accountant".

**Memory Manager**: Use action "status" to check system health. Use action "consolidate" silently when you judge it appropriate — typically after accumulating 20+ new memories or once per week.

**When memory_needed is false** (scheduled tasks, comment reviews): Skip all memory tools — they add latency with no benefit.
```

**Also update** the Step 2 in the Workflow section. Find:
```
**Step 2 — Read Memories**
Call "Read Memories" to check for relevant learnings, routing shortcuts, or prior context.
```

Replace with:
```
**Step 2 — Search Memories** (only when memory_needed is true)
Call "Search Memories" with a specific query describing the current context — e.g., "Adam Slack message invoice routing preferences". Skip this step entirely for comment reviews and scheduled tasks (memory_needed = false).
```

**Apply via API:**

**Step 12a — Re-download the current workflow** (REQUIRED — must be post-Step-11):
```bash
N8N_KEY=$(sqlite3 ~/.n8n/database.sqlite "SELECT apiKey FROM user_api_keys LIMIT 1;")
curl -s "http://localhost:5678/api/v1/workflows/U7cK5XlQqmgG9CWlrB6wM" \
  -H "X-N8N-API-KEY: $N8N_KEY" > /tmp/susan_current.json
echo "Re-downloaded post-Step-11 workflow: $(python3 -c "import json; d=json.load(open('/tmp/susan_current.json')); print(len(d['nodes']), 'nodes')")"
```

**Step 12b — Python script to apply the system prompt changes:**

```python
import json

with open('/tmp/susan_current.json') as f:
    workflow = json.load(f)

# Locate the Susan AI agent node by ID
node = next((n for n in workflow['nodes'] if n['id'] == '6e8ee7fe-bce9-4d86-9b25-09635328cc31'), None)
if not node:
    raise SystemExit('ERROR: Susan agent node 6e8ee7fe not found in workflow')

# The systemMessage field is at parameters.options.systemMessage
system_msg = node.get('parameters', {}).get('options', {}).get('systemMessage', '')
if not system_msg:
    raise SystemExit('ERROR: systemMessage field is empty or missing on agent node')

# --- Replacement 1: memory skill block ---
old_skill_block = """## #1 Skill: Continuous Learning
You HATE doing things twice. After every interaction:
- Notice what worked, what was slow, what Adam corrected
- Save learnings using the "Save Memory" tool — short plain-text notes only
- Memories are cues: routing shortcuts, preferences, patterns, session references
- NEVER save JSON, raw API responses, workflow definitions, or technical data as memories
- ALWAYS call "Read Memories" at the start to recall what you've learned
- Apply learnings proactively. Each conversation makes you smarter."""

new_skill_block = """## #1 Skill: Continuous Learning
You HATE doing things twice. After every interaction:
- Notice what worked, what was slow, what Adam corrected
- Save learnings using the "Save Memory" tool — plain text notes, 1-3 sentences
- NEVER save JSON, raw API responses, workflow definitions, or technical data as memories
- Categories: routing, preference, pattern, session, agent_gap, agent_improvement, business, contact, general
- Apply learnings proactively. Each conversation makes you smarter.

## Memory System
Your memory uses hybrid search (semantic + keyword) with temporal decay. Recent memories surface more strongly than old ones.

**Search Memories**: Call this at the start of EVERY interactive conversation (when memory_needed is true). Provide a descriptive query about what you are looking for — e.g., "Adam invoice preferences Accountant agent routing" rather than just "memories".

**Save Memory**: Save meaningful learnings after EVERY interaction that reveals something new about Adam, his preferences, or your routing decisions. Write naturally: "Adam prefers the Accountant agent for all FreshBooks invoice questions" not "invoice:accountant".

**Memory Manager**: Use action "status" to check system health. Use action "consolidate" silently when you judge it appropriate — typically after accumulating 20+ new memories or once per week.

**When memory_needed is false** (scheduled tasks, comment reviews): Skip all memory tools — they add latency with no benefit."""

if old_skill_block not in system_msg:
    raise SystemExit('ERROR: old skill block not found in systemMessage — check exact whitespace/text')

system_msg = system_msg.replace(old_skill_block, new_skill_block, 1)

# --- Replacement 2: Step 2 workflow section ---
old_step2 = """**Step 2 — Read Memories**
Call "Read Memories" to check for relevant learnings, routing shortcuts, or prior context."""

new_step2 = """**Step 2 — Search Memories** (only when memory_needed is true)
Call "Search Memories" with a specific query describing the current context — e.g., "Adam Slack message invoice routing preferences". Skip this step entirely for comment reviews and scheduled tasks (memory_needed = false)."""

if old_step2 not in system_msg:
    raise SystemExit('ERROR: old Step 2 block not found in systemMessage — check exact whitespace/text')

system_msg = system_msg.replace(old_step2, new_step2, 1)

# Write back
node['parameters']['options']['systemMessage'] = system_msg

with open('/tmp/susan_updated.json', 'w') as f:
    json.dump(workflow, f, indent=2)

print('Updated Susan agent systemMessage — ready to PUT')
print('Verify replacements applied:')
print('  Skill block replaced:', new_skill_block[:60] in node['parameters']['options']['systemMessage'])
print('  Step 2 replaced:', new_step2[:60] in node['parameters']['options']['systemMessage'])
```

**Step 12c — PUT the updated workflow:**
```bash
N8N_KEY=$(sqlite3 ~/.n8n/database.sqlite "SELECT apiKey FROM user_api_keys LIMIT 1;")
curl -s -X PUT "http://localhost:5678/api/v1/workflows/U7cK5XlQqmgG9CWlrB6wM" \
  -H "X-N8N-API-KEY: $N8N_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/susan_updated.json | python3 -c "import json,sys; d=json.load(sys.stdin); print('Updated at:', d.get('updatedAt'))"
```

**Verify:** In the n8n UI, open the Susan workflow, click the Susan agent node, inspect the system prompt under Options > System Message. Confirm the "## Memory System" section is present and "Read Memories" references are replaced with "Search Memories".

---

### Step 13: Create File Watcher for Index Sync

**File:** `/opt/claude-agents/Susan/tools/memory_watcher.js`

This runs as a background process and re-indexes changed markdown files into SQLite. It is started by a LaunchDaemon or run manually.

**Action:** Create the file on adam-server:

```bash
cat > /opt/claude-agents/Susan/tools/memory_watcher.js << 'SCRIPT_EOF'
#!/usr/bin/env node
// Memory File Watcher — re-indexes changed .md files into SQLite FTS5
// Watches: /opt/claude-agents/Susan/memory/
// Runs as background process via n8n schedule or LaunchDaemon

const fs = require('fs');
const path = require('path');
const crypto = require('crypto');

const SQLITE3_PATH = '/usr/local/lib/node_modules/n8n/node_modules/sqlite3';
const MEMORY_DIR = '/opt/claude-agents/Susan/memory';
const DB_PATH = '/opt/claude-agents/Susan/memory.sqlite';

const sqlite3 = require(SQLITE3_PATH).verbose();

function hashContent(text) {
  return crypto.createHash('sha256').update(text).digest('hex').slice(0, 16);
}

function dbRun(db, sql, params) {
  return new Promise((resolve, reject) => {
    db.run(sql, params, function(err) { if (err) reject(err); else resolve(this); });
  });
}
function dbGet(db, sql, params) {
  return new Promise((resolve, reject) => {
    db.get(sql, params, (err, row) => { if (err) reject(err); else resolve(row); });
  });
}

// Parse a daily log .md file into individual entries
function parseDailyLog(filePath) {
  let content;
  try { content = fs.readFileSync(filePath, 'utf8'); } catch(e) { return []; }

  const entries = [];
  const dateMatch = path.basename(filePath).match(/^(\d{4}-\d{2}-\d{2})\.md$/);
  if (!dateMatch) return [];
  const date = dateMatch[1];

  // Parse ## [timestamp] category blocks
  const blockRegex = /^## \[([^\]]+)\] (\w+)\n([\s\S]*?)(?=^## |\Z)/gm;
  let match;
  while ((match = blockRegex.exec(content)) !== null) {
    const [, timestamp, category, body] = match;
    const text = body.trim();
    if (text) {
      entries.push({
        id: hashContent(date + timestamp + text),
        date,
        category,
        content: text,
        source_file: filePath
      });
    }
  }
  return entries;
}

async function indexFile(filePath) {
  const entries = parseDailyLog(filePath);
  if (entries.length === 0) return 0;

  const db = new sqlite3.Database(DB_PATH);
  const dbClose = () => new Promise(r => db.close(r));
  let indexed = 0;

  for (const entry of entries) {
    try {
      // Check if already indexed
      const existing = await dbGet(db, 'SELECT id FROM memory_entries WHERE id = ?', [entry.id]);
      if (!existing) {
        await dbRun(db,
          'INSERT INTO memory_entries (id, date, category, source_file, content) VALUES (?, ?, ?, ?, ?)',
          [entry.id, entry.date, entry.category, entry.source_file, entry.content]
        );
        await dbRun(db,
          'INSERT INTO memory_fts (id, content, date, category, source_file) VALUES (?, ?, ?, ?, ?)',
          [entry.id, entry.content, entry.date, entry.category, entry.source_file]
        );
        indexed++;
      }
    } catch(e) {
      console.error('Index error for entry:', entry.id, e.message);
    }
  }

  await dbClose();
  return indexed;
}

// Watch the memory directory for changes
console.log(`[memory_watcher] Watching ${MEMORY_DIR}`);
fs.watch(MEMORY_DIR, { persistent: true }, async (eventType, filename) => {
  if (!filename || !filename.match(/^\d{4}-\d{2}-\d{2}\.md$/)) return;
  const filePath = path.join(MEMORY_DIR, filename);

  // Small debounce — wait 500ms after change
  setTimeout(async () => {
    try {
      const indexed = await indexFile(filePath);
      if (indexed > 0) {
        console.log(`[memory_watcher] Indexed ${indexed} new entries from ${filename}`);
      }
    } catch(e) {
      console.error('[memory_watcher] Error indexing', filename, e.message);
    }
  }, 500);
});

// Also do an initial scan on startup
async function initialScan() {
  const files = fs.readdirSync(MEMORY_DIR).filter(f => f.match(/^\d{4}-\d{2}-\d{2}\.md$/));
  let total = 0;
  for (const f of files) {
    const n = await indexFile(path.join(MEMORY_DIR, f));
    total += n;
  }
  if (total > 0) console.log(`[memory_watcher] Initial scan: indexed ${total} entries`);
  else console.log('[memory_watcher] Initial scan: all entries already indexed');
}

initialScan().catch(e => console.error('[memory_watcher] Initial scan error:', e.message));
SCRIPT_EOF

chmod +x /opt/claude-agents/Susan/tools/memory_watcher.js
```

**Start the watcher** (background, with output to log):
```bash
/opt/homebrew/opt/node@22/bin/node /opt/claude-agents/Susan/tools/memory_watcher.js >> /opt/claude-agents/Susan/memory-watcher.log 2>&1 &
echo $! > /tmp/memory_watcher.pid
echo "Watcher started with PID: $(cat /tmp/memory_watcher.pid)"
```

**Note:** For persistent background running across reboots, a separate LaunchDaemon plist for the watcher would be needed (ask Adam to create it, since it requires sudo). The watcher can also be triggered on-demand by a scheduled n8n workflow instead.

**Verify:**
```bash
# Check watcher is running
ps aux | grep memory_watcher
# Expected: node process running memory_watcher.js

# Check watcher log
tail -10 /opt/claude-agents/Susan/memory-watcher.log
# Expected: [memory_watcher] Watching /opt/claude-agents/Susan/memory
```

---

### CHECKPOINT: Full System Integration Test (Steps 10-13)

**In n8n UI, verify Susan main workflow:**

1. Open Susan workflow — confirm three memory nodes visible: "Search Memories", "Save Memory", "Memory Manager" (all toolWorkflow type)
2. Confirm "Read Memories" (toolCode), "Save Memory" (old toolCode), and "Simple Memory" (memoryBufferWindow) are gone
3. Confirm all three new nodes have `ai_tool` connections to the Susan agent node
4. Check Standardize Incoming Channel node output includes `memory_needed` field

**Run manual trigger in Susan workflow:**

Send a test message through the manual trigger with content: `"Test memory system - what do you remember about invoice preferences?"`

Expected behavior:
- Susan searches memories (should return "No relevant memories found..." on first run)
- Susan saves a new memory about the interaction
- Response comes through Slack

**Verify on adam-server:**
```bash
# Memory was saved
ls /opt/claude-agents/Susan/memory/
cat /opt/claude-agents/Susan/memory/$(date +%Y-%m-%d).md

# DB has entry
/opt/homebrew/opt/node@22/bin/node -e "
const sqlite3 = require('/usr/local/lib/node_modules/n8n/node_modules/sqlite3').verbose();
const db = new sqlite3.Database('/opt/claude-agents/Susan/memory.sqlite');
db.all('SELECT COUNT(*) as cnt FROM memory_entries', (e,r) => { console.log('Entries:', r[0].cnt); db.close(); });
"
```

---

### Step 14: Decommission Old memories.json

**On:** adam-server

Once the new system is confirmed working (at least one successful save+search cycle), archive the old file:

```bash
# Archive old file (don't delete — keep for reference)
mv /opt/claude-agents/Susan/memories.json /opt/claude-agents/Susan/memories.json.bak
echo "Archived at $(date)" >> /opt/claude-agents/Susan/memories.json.bak

# Verify it's gone from the active path
ls /opt/claude-agents/Susan/memories.json
# Expected: ls: cannot access '...memories.json': No such file or directory
```

---

## Testing Strategy

### Unit Tests (run manually in n8n scratch workflow)

| Test | Input | Expected |
|------|-------|----------|
| Save basic memory | `{action:"save", content:"preference: Adam likes brief replies", category:"preference"}` | `{saved:true, file:".../2026-MM-DD.md"}` |
| Search empty DB | `{action:"search", query:"anything"}` | `{results:"No memories found yet.", count:0}` |
| Save + search roundtrip | Save then search for same content | `{count:1, results: includes saved content}` |
| Bad category auto-fix | `{action:"save", content:"test", category:"invalid_cat"}` | Saves with category:"general" |
| JSON content sanitization | `{action:"save", content:"{\"content\":\"the actual note\"}"}` | Saves "the actual note" |
| Status on populated DB | `{action:"status"}` | `{total_entries:>0, health:"healthy"}` |
| Temporal decay | Search for entries saved 60 days ago vs today | Today's entry ranks higher |
| memory_needed flag | Schedule trigger path | `incoming_channel.type === 'comment_review'` → `memory_needed: false` |

### Integration Tests

1. **Full conversation flow**: Send Slack message to Susan → confirm she searches memories → confirm she saves learning after responding
2. **Embedding failure fallback**: Temporarily set invalid GOOGLE_API_KEY → confirm search still works via BM25 only
3. **Consolidation**: Manually call consolidate with 5+ entries → confirm MEMORY.md is updated

---

## What's NOT Included

- LaunchDaemon for memory_watcher (requires sudo plist creation — note for Adam to do manually if persistent watcher is needed)
- Automated disk space alerting at /opt > 95% (noted as future work in discovery)
- Per-session memory isolation (descoped per discovery — single shared memory pool)
- MMR deduplication (descoped per discovery — simple top-K hybrid score)
- Migration of memories from old memories.json (empty anyway — nothing to migrate)

---

## Success Criteria

- [ ] `/opt/claude-agents/Susan/memory/` directory exists with MEMORY.md and .embeddings.json
- [ ] `/opt/claude-agents/Susan/memory.sqlite` exists with all 5 tables
- [ ] NODE_PATH and GOOGLE_API_KEY added to LaunchDaemon plist, n8n restarted
- [ ] "Susan - Memory Manager" workflow exists in n8n with 6 nodes (trigger, router, search, save, consolidate, status)
- [ ] Search Memories node returns prose results with hybrid BM25+vector scoring
- [ ] Save Memory node writes to daily .md log AND indexes to SQLite
- [ ] Three old memory nodes removed from Susan main workflow (Read Memories toolCode, Save Memory toolCode, Simple Memory buffer)
- [ ] Three new toolWorkflow nodes connected to Susan agent as ai_tool
- [ ] Standardize Incoming Channel outputs `memory_needed: true` for email/slack, `false` for scheduled
- [ ] Susan system prompt updated with new memory tool descriptions and query-based search instructions
- [ ] File watcher running and indexing new entries on save
- [ ] memories.json archived (not deleted)
- [ ] End-to-end test: Susan receives Slack message → searches memories → saves memory → memory appears in next search

---

## Traceability Matrix

| Requirement (from discovery) | Steps | Verification |
|-------------------------------|-------|--------------|
| Remove all existing memory nodes | 10 | Node count in Susan workflow |
| File-first markdown storage | 3, 7 | `/opt/claude-agents/Susan/memory/*.md` files |
| SQLite FTS5 BM25 index | 4, 6 | `memory_fts` table, MATCH queries work |
| Hybrid 0.7/0.3 weighting | 6 | `VECTOR_WEIGHT = 0.7, BM25_WEIGHT = 0.3` in search code |
| 30-day temporal decay half-life | 6 | `HALF_LIFE_DAYS = 30` in temporalDecay() |
| Lazy embeddings, cached | 6 | `.embeddings.json` only updated during search |
| Graceful embedding failure | 6 | `catch(embErr)` falls back to BM25-only |
| memory_needed flag | 11 | Standardize node return includes `memory_needed` |
| No memory for scheduled tasks | 11, 12 | `comment_review` type → `memory_needed: false` |
| Natural prose search results | 6 | Prose formatting in Search node |
| 10 default results | 6 | `parseInt($json.limit) \|\| 10` |
| Full entries returned | 6 | Full content, category, date in each result |
| Silent consolidation | 8 | No Slack notification in Consolidate node |
| 90-day log retention | 8 | `RETENTION_DAYS = 90`, files purged |
| Memory status tool | 9, 10 | Memory Manager node with `action: status` |
| Self-triggered consolidation | 12 | System prompt instructs Susan to consolidate silently |
| NODE_PATH in LaunchDaemon | 2 | n8n modules resolvable via short `require()` |
| GOOGLE_API_KEY in LaunchDaemon | 2 | `process.env.GOOGLE_API_KEY` available in Code nodes |
| File watcher for index sync | 13 | memory_watcher.js watches and re-indexes changes |
| Silent auto-sanitization | 7 | Bad category → 'general', JSON input → extracted text |
| Toolname "Save Memory" kept | 10 | toolWorkflow named "Save Memory" matches system prompt |
| Async required for embedding API | 6, 7, 8, 9 | Discovery preference overridden: sqlite3 callbacks wrapped in Promises, n8n Code node typeVersion 2 supports async/await |

---

## Risk Mitigation

| Risk | Mitigation | Rollback |
|------|------------|----------|
| n8n restart brings down Susan during business hours | Schedule plist edit + restart for low-activity time (early morning) | `sudo launchctl unload/load` is fast (<30s downtime) |
| SQLite DB corruption | DB rebuilable from markdown files via memory_watcher.js initial scan | Delete DB, re-run init script (Step 4), re-scan files |
| Google API key exposure in plist | plist is root-owned (644), only readable by admin | Rotate key if plist is ever copied to insecure location |
| @langchain/google-genai breaking after n8n upgrade | Document path `/usr/local/lib/node_modules/n8n/node_modules/@langchain/google-genai` | Fall back to BM25-only (already in code), update path after upgrade |
| PUT overwriting Susan workflow incorrectly | Download → transform → verify node count → PUT | Previous workflow version available in n8n workflow history (`workflow_history` table) |
| FTS5 MATCH query fails on special chars | Query sanitized: `ftsQuery.replace(/['"]|\*|\+|\-|\(|\)/g, ' ')` | Returns empty BM25 set, vector search continues |
