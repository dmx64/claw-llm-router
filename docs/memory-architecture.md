# Memory Architecture — Shared Agent Memory System

## Overview

Centralized memory system serving multiple AI agents (Alika, Agime, Dina) through a shared Memelord service with semantic search, RL reranking, and automatic failover.

## Architecture Diagram

```
┌─────────────────────────────────────────────────┐
│              MASTER: ah (192.168.1.220)          │
│                                                  │
│  ┌───────────────────────────────────────────┐   │
│  │  Memelord Service (0.0.0.0:9377)          │   │
│  │  ├── SQLite + sqlite-vec (768d vectors)   │   │
│  │  ├── FTS5 full-text search                │   │
│  │  ├── Embedding: nomic-embed-text (Ollama) │   │
│  │  ├── RL rerank (mem_rl_weights)           │   │
│  │  └── 7754+ chunks, 25K cached embeddings  │   │
│  └──────────┬────────────────────────────────┘   │
│             │                                    │
│  ┌──────────┴────────────────────────────────┐   │
│  │  OpenClaw Memory Files (source)           │   │
│  │  ├── MEMORY.md (HOT index, ≤8K)          │   │
│  │  ├── memory/topics/*.md (WARM)            │   │
│  │  ├── memory/archive/ (COLD)               │   │
│  │  └── memory/YYYY-MM-DD.md (daily notes)   │   │
│  └───────────────────────────────────────────┘   │
└──────────────┬──────────────┬────────────────────┘
               │              │
          LAN :9377      rsync 04:00
               │              │
    ┌──────────┴───┐    ┌─────┴───────────────────┐
    │   AGENTS:    │    │  BACKUP: agime           │
    │              │    │  /memory-backup/          │
    │  • Alika     │    │  ah-main.sqlite           │
    │    (ah)      │    │                           │
    │  • Agime     │    │  + own memelord :9377     │
    │    (99)      │    │    (896 local chunks)     │
    │  • Dina      │    │                           │
    │    (237)     │    │  Failover: if ah down →   │
    │              │    │  agime serves from backup  │
    └──────────────┘    └───────────────────────────┘
```

## Request Flow

```
User Message
    │
    ▼
┌──────────────────────────────┐
│ Agent receives message       │
│ POST http://ah:9377/query    │
│ {"message": "<user text>"}   │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ Memelord Pipeline            │
│                              │
│ 1. Text → embedding (~70ms) │
│    nomic-embed-text (Ollama) │
│                              │
│ 2. Hybrid search:           │
│    • sqlite-vec (vector)    │
│    • FTS5 (keywords)        │
│                              │
│ 3. RL rerank                │
│    mem_rl_weights table     │
│    (learns from feedback)   │
│                              │
│ 4. Return top 5 chunks     │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ Agent injects chunks into   │
│ context → generates reply   │
│                              │
│ POST /feedback              │
│ {used_ids, all_ids}         │
│ → RL weights updated        │
└──────────────────────────────┘
```

## Memory Layers (4-tier)

| Layer | What | Location | TTL | Access |
|-------|------|----------|-----|--------|
| **HOT** | MEMORY.md — index, current week | File (≤8K) | Always current | Every session |
| **WARM** | Topic files (models, security, calls, crypto, people) | memory/topics/*.md | Updated as needed | On-demand |
| **COLD** | Archived daily notes >14 days, weekly digests | memory/archive/ | Permanent | Search only |
| **VECTOR** | Embeddings + RL weights + FTS index | SQLite + sqlite-vec | Auto-updated | Memelord API |

## Database Schema

```sql
-- Core tables
chunks (id TEXT PK, source TEXT, text TEXT, metadata JSON)
chunks_vec (rowid, vector FLOAT[768])  -- sqlite-vec
chunks_fts (text)                       -- FTS5 full-text

-- Learning tables  
mem_rl_weights (chunk_id TEXT, weight REAL, hits INT, last_used TIMESTAMP)
mem_query_log (query TEXT, chunks_returned JSON, chunks_used JSON, ts TIMESTAMP)

-- Caching
embedding_cache (text_hash TEXT PK, vector BLOB, model TEXT, ts TIMESTAMP)
```

## API Endpoints

### `GET /health`
```json
{
  "ok": true,
  "service": "memelord",
  "chunks": 7754,
  "vectors": 7754,
  "embedding_cache": 25508,
  "embed_model": "all-minilm",
  "vector_dims": 384
}
```

### `POST /query`
```json
// Request
{"message": "what is the AParser status?"}

// Response
{
  "action": "INJECT",
  "chunks": [
    {"id": "abc123", "text": "AParser: 143K keys...", "score": 0.87},
    ...
  ]
}
```

### `POST /feedback`
```json
{
  "used_chunk_ids": ["abc123", "def456"],
  "all_chunk_ids": ["abc123", "def456", "ghi789"]
}
```

## LLM Router Integration

The memory system works alongside the LLM Router for complete agent infrastructure:

```
User Message
    │
    ├──→ Memelord (memory retrieval, ~70ms)
    │
    └──→ LLM Router (model selection, ~50ms)
              │
              ├── Semantic embedding classification
              │   (nomic-embed-text, 768d vectors)
              │
              └── Routes to optimal backend:
                  ├── groq (simple Q&A) 
                  ├── claude (code)
                  ├── grok (math/reasoning)
                  ├── gemini (translation/summary)
                  ├── together (security/pentest)
                  ├── kimi (creative)
                  └── local/ollama (private/PII)
```

## Failover Strategy

1. **Primary**: All agents → `http://192.168.1.220:9377` (ah master)
2. **Nightly backup**: rsync SQLite to agime `/home/dmx/memory-backup/ah-main.sqlite`
3. **If ah down**: Agime serves from local backup copy
4. **Agent resilience**: If memelord unreachable → agent works without memory (graceful degradation)
5. **Max data loss**: 24 hours (nightly sync interval)

## Firewall Rules

```bash
# ah (master)
ufw allow from 192.168.1.0/24 to any port 9377 proto tcp  # Memelord LAN
ufw allow from 192.168.1.0/24 to any port 8081 proto tcp  # LLM Router LAN

# agime (backup)
ufw allow from 192.168.1.220 to any port 9377 proto tcp   # ah → agime sync
```

## Systemd Service

```ini
[Unit]
Description=Memelord Memory Hook Service
After=network.target

[Service]
Type=simple
User=dmx
WorkingDirectory=/home/dmx/memory-stack
ExecStart=/usr/bin/python3 memory-service.py
Restart=on-failure
RestartSec=5
Environment=PYTHONUNBUFFERED=1
Environment=MEMELORD_BIND=0.0.0.0

[Install]
WantedBy=multi-user.target
```

## Performance

| Operation | Latency |
|-----------|---------|
| Embedding (nomic-embed-text) | ~70ms |
| Vector search (sqlite-vec) | ~5ms |
| FTS5 keyword search | ~2ms |
| RL rerank | ~1ms |
| Full query pipeline | ~80-100ms |
| LLM Router classification | ~50ms |
| End-to-end (memory + routing) | ~150ms |

## Monitoring

```bash
# Check memelord health
curl http://192.168.1.220:9377/health

# Check router health  
curl http://192.168.1.220:8081/health

# Check backup freshness
ssh -p 2223 dmx@192.168.1.99 "ls -la /home/dmx/memory-backup/ah-main.sqlite"

# Check systemd status
systemctl status memelord.service
```

---

*Last updated: 2026-03-22 · Maintained by Alika*
