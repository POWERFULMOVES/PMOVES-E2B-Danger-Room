# PMOVES.AI Integration Dossier

_Last updated: 2026-05-20_

## Module

- **Name:** PMOVES-E2B-Danger-Room
- **Path (in parent):** `PMOVES-E2B-Danger-Room/` (git submodule)
- **Branch convention:** customizations on `PMOVES.AI-Edition-Hardened`; `main` tracks upstream `e2b-dev/E2B/main`.

## Purpose in PMOVES.AI

Cloud sandbox environment for safe code execution and agent validation runs. Danger Room is the **isolation pore** of the MOF lattice — AI-generated code, MissingLinc forensic queries, multi-engine TTS validation harnesses, and any other not-yet-trusted run executes here before touching production fleet state.

## PMOVES Overlay Surface

- **pmoves-integrations/ overlay path:** N/A — Danger Room is consumed via the E2B SDK (`e2b_code_interpreter.Sandbox`), not via docker-compose. No `pmoves-integrations/` shim needed.
- **Compose/profile wiring:** Sandboxes connect to the canonical PMOVES MCP server bundle via the **D-Proxy pattern** documented in [`pmoves/docs/operations/MCP_TOOLKIT.md`](https://github.com/POWERFULMOVES/PMOVES.AI/blob/main/pmoves/docs/operations/MCP_TOOLKIT.md) § 6. The host runs `docker mcp gateway run --profile pmoves_5090_web --transport sse --port 8090`; sandboxes reach it over HTTPS+SSE through Tailscale.
- **Env/secret inputs (passed via `Sandbox.create(envs=...)`):**
  - `MCP_GATEWAY_URL` — `https://<host>.tail-scale.ts.net:8090/mcp/sse`
  - `MCP_GATEWAY_AUTH_TOKEN` — sourced from `pmoves/env.shared` on the host
  - `E2B_API_KEY` — for the E2B SDK itself (set on the host calling `Sandbox.create`, not exposed to the sandbox)
- **Auth/JWT requirements:** none at the sandbox boundary today. The MCP gateway uses the SSE auth token (DNS-rebinding protection). Future: per-sandbox short-lived CHIT-signed tokens scoped to the agent's claim.

## Contracts and Topics

- **NATS subjects (planned, not yet wired):**
  - `danger.room.sandbox.created.v1` — published when an agent creates a sandbox
  - `danger.room.sandbox.destroyed.v1` — on sandbox close
  - `danger.room.tool.call.v1` — per MCP tool invocation through the host gateway (lands when BoTZ Gateway proxy ships per Lane C)
- **Supabase schema/tables touched:** none directly. Any agent-run results land via the standard PMOVES `content.raw.v1` envelope through the host services, not from within the sandbox.
- **MCP endpoints/skills:** sandboxes consume the 25-server surface of the `pmoves_5090_web` profile through the host SSE gateway. The set is curated at the profile level (`docker mcp profile show pmoves_5090_web`); current servers include `hostinger-mcp-server`, `cloud-run-mcp`, `cloudflare-{autorag,workers-builds,ai-gateway,audit-logs,browser-rendering,dns-analytics,one-casb,workers-bindings,observability,logpush,docs,container,graphql,radar}`, `docker-docs`, `dockerhub`, `context7`, `github-official`, `gitmcp`, `deepwiki`, `postman`, `openmesh`, `openapi-schema`.

## Boot Order and Health

- **Bring-up dependency order:**
  1. Host node has Docker Desktop + MCP Toolkit installed (`docker mcp version` ≥ v0.42.0)
  2. `make -C pmoves mcp-toolkit-bootstrap` — pull + import the `pmoves_5090_web` profile (idempotent)
  3. `make -C pmoves mcp-toolkit-secrets-sync` — populate docker-pass-style secrets from `pmoves/env.shared`
  4. `make -C pmoves mcp-toolkit-gateway-start` — start the SSE gateway listener (background)
  5. Application code calls `Sandbox.create({envs: ...})` with `MCP_GATEWAY_URL` + `MCP_GATEWAY_AUTH_TOKEN`
- **Health endpoints:**
  - Host gateway: `curl http://localhost:8090/healthz` (TODO confirm path with `docker mcp gateway run` upstream behavior)
  - Sandbox: standard E2B SDK liveness (`sandbox.is_running` in JS, `sandbox.is_running()` in Python)
- **Smoke targets:**
  - `make -C pmoves mcp-toolkit-status` — single-shot host-side health (profile/client/secret/gateway-PID)
  - Sandbox-side smoke: lands with Lane D-sandbox PR (verify `mcp list` inside sandbox shows 25-server surface)

## Hardening Notes

- **Image pinning / provenance:**
  - Profile artifact pinned by tag today (`docker.io/darkxside/pmoves_5090_web:latest`); pin to immutable SHA digest in production per `.claude/mcp.json` F-07 supply-chain note
  - Per-server images inside the profile carry their own SHA pins (visible via `docker mcp profile show`)
- **Secrets source:** PMOVES canonical pipeline — `env.tier-*` files produced by the secrets-funnel → `env.shared` aggregate → `pmoves/scripts/mcp-toolkit-secrets-sync.sh` → `docker mcp secret set` (docker-pass provider). OAuth-mediated Cloudflare servers need one-time interactive `docker mcp oauth authorize <server>` per node.
- **Network/security policy constraints:**
  - SSE auth token (`MCP_GATEWAY_AUTH_TOKEN`) prevents DNS rebinding
  - `--block-secrets` on by default — secret values can't exfil through tool args/responses
  - `--block-network` opt-in via `PMOVES_MCP_BLOCK_NETWORK=1` (lock tool containers from arbitrary outbound)
  - E2B sandboxes have their own outbound network controls (`allowOut`/`denyOut`) — lock sandboxes to only reach the host gateway if strict isolation needed

## Source Documentation

- **Upstream README:** `README.md` (PMOVES section at top, vanilla upstream below)
- **PMOVES Toolkit guide:** [`pmoves/docs/operations/MCP_TOOLKIT.md`](https://github.com/POWERFULMOVES/PMOVES.AI/blob/main/pmoves/docs/operations/MCP_TOOLKIT.md) (parent repo)
- **PMOVES docs index reference:** [`pmoves/docs/SUBMODULE_DOCS_DOSSIER.md`](https://github.com/POWERFULMOVES/PMOVES.AI/blob/main/pmoves/docs/SUBMODULE_DOCS_DOSSIER.md)
- **MOF architecture thesis:** [`pmoves/docs/architecture/PMOVES_MOF_ARCHITECTURE.md`](https://github.com/POWERFULMOVES/PMOVES.AI/blob/main/pmoves/docs/architecture/PMOVES_MOF_ARCHITECTURE.md)

## Owner / Audit

- **Owning lane:**
  - MCP Toolkit profile integration: 5090-CLAUDE (per `vision_5090_claude_max_level_inventory`)
  - Submodule pin promotion in parent repo: Z890-CLAUDE (per `project_z890_claude_submodule_worktree_lane`)
  - Persona overlay (when MissingLinc and others mint): Archon
- **Last integration audit run:** 2026-05-20 (this dossier update)
- **Open work surface:**
  - Lane D-sandbox — `PMOVES-E2B-Danger-Room-Desktop/template/template.py` additions: sandbox bootstrap script writing MCP client config pointing at host gateway
  - Lane C — BoTZ Gateway proxy in front of the host MCP gateway (lifts `danger.room.tool.call.v1` from "planned" to "shipped")
  - Lane F — Danger Room Desktop variant on Ubuntu 22.04 LTS (or Pop!_OS — TBD)

See the parent repo's plan file `~/.claude/plans/pmoves-5090-web-docker-mcp-profile-sunny-cascade.md` for the full lane orchestration.
