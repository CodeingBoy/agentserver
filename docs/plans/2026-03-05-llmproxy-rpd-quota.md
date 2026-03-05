# LLM Proxy RPD Quota Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add per-workspace requests-per-day (RPD) rate limiting to the llmproxy, with 3-layer config priority: code default < env var < DB workspace_quotas table.

**Architecture:** llmproxy gets a new `workspace_quotas` table in its own DB. Before proxying each Anthropic API request, it checks today's request count against the effective max_rpd for that workspace. Management APIs under `/internal/` allow agentserver to set per-workspace quotas. Sandbox tokens only grant access to `/v1/*` (LLM proxy), not to `/api/*` or `/internal/*`.

**Tech Stack:** Go, PostgreSQL, chi router, Helm

---

### Task 1: DB Migration — `workspace_quotas` table

**Files:**
- Create: `internal/llmproxy/migrations/002_workspace_quotas.sql`

**Step 1: Create migration file**

```sql
CREATE TABLE workspace_quotas (
    workspace_id TEXT PRIMARY KEY,
    max_rpd      INTEGER,
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Step 2: Commit**

```bash
git add internal/llmproxy/migrations/002_workspace_quotas.sql
git commit -m "feat(llmproxy): add workspace_quotas migration for RPD limit"
```

---

### Task 2: Config — add `DefaultMaxRPD`

**Files:**
- Modify: `internal/llmproxy/config.go`

**Step 1: Add field and env var loading**

Add `DefaultMaxRPD int` to `Config` struct. In `LoadConfigFromEnv`, parse `LLMPROXY_DEFAULT_MAX_RPD` env var (default 0 = unlimited).

**Step 2: Commit**

```bash
git add internal/llmproxy/config.go
git commit -m "feat(llmproxy): add DefaultMaxRPD config from env"
```

---

### Task 3: Types — add `WorkspaceQuota` struct

**Files:**
- Modify: `internal/llmproxy/types.go`

**Step 1: Add struct**

```go
type WorkspaceQuota struct {
    WorkspaceID string    `json:"workspace_id"`
    MaxRPD      *int      `json:"max_rpd"`
    UpdatedAt   time.Time `json:"updated_at"`
}
```

**Step 2: Commit**

```bash
git add internal/llmproxy/types.go
git commit -m "feat(llmproxy): add WorkspaceQuota type"
```

---

### Task 4: Store — quota CRUD + daily request count

**Files:**
- Modify: `internal/llmproxy/store_queries.go`

**Step 1: Add four methods**

- `GetWorkspaceQuota(workspaceID string) (*WorkspaceQuota, error)` — SELECT from workspace_quotas, return nil on ErrNoRows
- `SetWorkspaceQuota(workspaceID string, maxRPD *int) error` — UPSERT into workspace_quotas
- `DeleteWorkspaceQuota(workspaceID string) error` — DELETE from workspace_quotas
- `CountTodayRequests(workspaceID string) (int64, error)` — `SELECT COUNT(*) FROM usage WHERE workspace_id = $1 AND created_at >= date_trunc('day', NOW())`

**Step 2: Commit**

```bash
git add internal/llmproxy/store_queries.go
git commit -m "feat(llmproxy): add workspace quota CRUD and daily request count"
```

---

### Task 5: RPD enforcement in proxy handler

**Files:**
- Modify: `internal/llmproxy/anthropic.go`

**Step 1: Add RPD check after token validation**

In `handleAnthropicProxy`, after sandbox validation succeeds and before body reading (between current steps 1 and 2), add RPD check:

1. Only check for `/v1/messages` endpoint (same as `isMessagesEndpoint`)
2. Resolve effective max_rpd: if store exists, try `GetWorkspaceQuota`; if quota has `MaxRPD != nil`, use it; else fall back to `s.config.DefaultMaxRPD`
3. If effective max_rpd > 0, call `CountTodayRequests`; if count >= max_rpd, return 429 with JSON body `{"error":"rpd_exceeded","message":"...","quota":{"current":N,"max":M}}`

**Step 2: Commit**

```bash
git add internal/llmproxy/anthropic.go
git commit -m "feat(llmproxy): enforce RPD quota before proxying requests"
```

---

### Task 6: Internal management API routes

**Files:**
- Modify: `internal/llmproxy/server.go`

**Step 1: Add `/internal/quotas` routes**

In `Routes()`, add a new route group:

```go
r.Route("/internal", func(r chi.Router) {
    r.Use(s.requireStore)
    r.Get("/quotas/{workspace_id}", s.handleGetWorkspaceQuota)
    r.Put("/quotas/{workspace_id}", s.handleSetWorkspaceQuota)
    r.Delete("/quotas/{workspace_id}", s.handleDeleteWorkspaceQuota)
})
```

**Step 2: Implement handlers**

- `handleGetWorkspaceQuota` — returns `{"quota": ..., "default_max_rpd": N}` (both DB override and config default)
- `handleSetWorkspaceQuota` — accepts `{"max_rpd": N}`, calls `SetWorkspaceQuota`
- `handleDeleteWorkspaceQuota` — calls `DeleteWorkspaceQuota`, returns 204

**Step 3: Commit**

```bash
git add internal/llmproxy/server.go
git commit -m "feat(llmproxy): add internal quota management API"
```

---

### Task 7: Route protection — sandbox tokens only for /v1/*

**Files:**
- Modify: `internal/llmproxy/server.go`

**Step 1: Verify route isolation**

Current state: `/v1/*` uses `handleAnthropicProxy` which validates `x-api-key` (sandbox proxy token). `/api/*` and the new `/internal/*` have no auth — they rely on network isolation (only agentserver can reach them).

Ensure the new `/internal/*` routes do NOT go through any sandbox token check. This is already the case since they're separate route groups. No code change needed — just verify.

**Step 2: Commit (if any cleanup needed)**

---

### Task 8: Helm values + deployment template

**Files:**
- Modify: `deploy/helm/agentserver/values.yaml`
- Modify: `deploy/helm/agentserver/templates/llmproxy.yaml`

**Step 1: Add default to values.yaml**

Under `llmproxy:`, add:

```yaml
llmproxy:
  # ...existing fields...
  defaultMaxRpd: 0  # requests per day per workspace, 0 = unlimited
```

**Step 2: Inject env var in llmproxy.yaml**

Add to the llmproxy container `env:` section:

```yaml
- name: LLMPROXY_DEFAULT_MAX_RPD
  value: {{ .Values.llmproxy.defaultMaxRpd | default 0 | quote }}
```

**Step 3: Commit**

```bash
git add deploy/helm/agentserver/values.yaml deploy/helm/agentserver/templates/llmproxy.yaml
git commit -m "feat(helm): add llmproxy defaultMaxRpd config"
```

---

### Task 9: Build verification

**Step 1: Verify compilation**

```bash
cd /root/agentserver && go build ./...
```

**Step 2: Final commit if needed**

---
