# OAB + GBrain Reference Architecture

Shared persistent memory for OpenAB multi-agent deployments.

## Architecture

```
  Discord / Slack / Telegram / LINE
    │            │            │
    ▼            ▼            ▼
┌────────┐ ┌────────┐ ┌────────┐
│OAB Pod │ │OAB Pod │ │OAB Pod │   ← any ACP CLI (kiro, claude, copilot, …)
│        │ │        │ │        │
│ gbrain │ │ gbrain │ │ gbrain │   ← MCP server per pod (gbrain serve)
│  serve │ │  serve │ │  serve │
└───┬────┘ └───┬────┘ └───┬────┘
    └──────────┼──────────┘
               ▼
     ┌───────────────────┐
     │  gbrain-postgres  │   pgvector/pgvector:pg16 · 5Gi PVC
     └───────────────────┘
```

All agents read/write the same DB. One agent writes a page → all others can query it instantly.

- **OpenAB**: https://github.com/openabdev/openab
- **GBrain**: https://github.com/garrytan/gbrain

---

## Option A — Manual Setup

For existing OAB deployments. No Helm changes needed.

```
┌─────────────────────────────────────────────────────────┐
│ 1. Deploy PostgreSQL                                    │
│                                                         │
│    kubectl create secret ─► gbrain-postgres-secret      │
│    kubectl apply          ─► Deployment + Service + PVC │
│                                                         │
│    Image: pgvector/pgvector:pg16                        │
│    Env:   POSTGRES_DB=gbrain                            │
│           POSTGRES_USER=gbrain                          │
│           POSTGRES_PASSWORD=(from secret)               │
│    PVC:   5Gi RWO                                       │
│    Svc:   ClusterIP :5432                               │
├─────────────────────────────────────────────────────────┤
│ 2. For each OAB agent pod:                              │
│                                                         │
│    kubectl exec <pod> ──┐                               │
│                         ├─► Install gbrain CLI          │
│    curl install.sh      │   → ~/.local/bin/gbrain       │
│                         ├─► Init database               │
│    gbrain init --url    │   → ~/.gbrain/config.json     │
│                         ├─► Write MCP config            │
│                         │                               │
│    ┌──────────┬─────────────────────────────┐           │
│    │ CLI      │ MCP config path             │           │
│    ├──────────┼─────────────────────────────┤           │
│    │ kiro     │ ~/.kiro/settings/mcp.json   │           │
│    │ claude   │ ~/.claude/server.json       │           │
│    │ copilot  │ ~/.copilot/mcp-config.json  │           │
│    └──────────┴─────────────────────────────┘           │
│                                                         │
│    MCP config content:                                  │
│    {"mcpServers":{"gbrain":{                            │
│      "command":"~/.local/bin/gbrain",                   │
│      "args":["serve"]}}}                                │
├─────────────────────────────────────────────────────────┤
│ 3. Verify                                               │
│                                                         │
│    kubectl exec <pod> -- gbrain stats                   │
│    kubectl exec <pod> -- gbrain query "test"            │
└─────────────────────────────────────────────────────────┘
```

Data lives on PVC — survives pod restarts when `persistence.enabled: true` (OAB default).

---

## Option B — Helm Integrated

Target state: GBrain as a built-in OAB Helm component.

```
values.yaml                          What the chart does
─────────────                        ──────────────────
gbrain:                              ┌──────────────────────────────────┐
  enabled: true          ──────────► │ 1. Deploy gbrain-postgres        │
                                     │    (Deployment + Svc + PVC +     │
  postgresql:                        │     Secret, auto-generated pw)   │
    image: pgvector/pgvector:pg16    │                                  │
    storage: 5Gi                     │ 2. Inject init container into    │
                                     │    every agent pod:              │
  # or use external:                 │    • install gbrain CLI          │
  # external:                        │    • gbrain init --url $DB_URL   │
  #   url: "postgresql://..."        │    • write MCP config            │
                                     │      (auto-detect CLI type)      │
                                     │                                  │
                                     │ 3. Inject DATABASE_URL env       │
                                     │    from secret into agent pods   │
                                     └──────────────────────────────────┘

MCP config auto-detection:
  command contains "kiro"    → ~/.kiro/settings/mcp.json
  command contains "claude"  → ~/.claude/server.json
  command contains "copilot" → ~/.copilot/mcp-config.json
  fallback                   → ~/.config/mcp/servers.json
```

---

## What GBrain gives your agents

```
┌─ Agent A writes ──────────────────────────────────────────┐
│                                                           │
│  gbrain put handoff/task-42   ← structured page           │
│  gbrain link handoff/task-42 agent/B --type assigned_to   │
│                                                           │
├─ Agent B queries ─────────────────────────────────────────┤
│                                                           │
│  gbrain query "what tasks are assigned to me?"            │
│  gbrain get handoff/task-42   ← full context              │
│                                                           │
└───────────────────────────────────────────────────────────┘

Capabilities:
  • Pages        markdown + frontmatter + tags
  • Search       hybrid (vector + keyword + graph)
  • Links        typed edges (assigned_to, depends_on, …)
  • Timeline     chronological events per entity
  • 30+ tools    exposed via MCP (brain_write, brain_query, …)
```
