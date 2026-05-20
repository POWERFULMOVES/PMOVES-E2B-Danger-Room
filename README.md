# PMOVES.AI Edition — Danger Room

> This repository is the **PMOVES.AI fork of the E2B Code Interpreter SDK** (`e2b-dev/E2B`), customized for the PMOVES.AI fleet's sandboxed-agent surface ("Danger Room"). Upstream README continues below the PMOVES section.
>
> - **Upstream:** [e2b-dev/E2B](https://github.com/e2b-dev/E2B) — credit, license, and SDK docs stay there.
> - **PMOVES.AI parent repo:** [POWERFULMOVES/PMOVES.AI](https://github.com/POWERFULMOVES/PMOVES.AI) — this fork lives as the `PMOVES-E2B-Danger-Room` submodule.
> - **PMOVES branch convention:** PMOVES customizations live on [`PMOVES.AI-Edition-Hardened`](../../tree/PMOVES.AI-Edition-Hardened). `main` mirrors upstream for sync.
> - **Integration dossier:** [`PMOVES.AI_INTEGRATION.md`](PMOVES.AI_INTEGRATION.md) — contracts, env/secrets, MCP wiring, ownership.

## What this fork adds

Vanilla E2B gives you secure isolated sandboxes for AI-generated code. PMOVES.AI Danger Room layers on top of that:

- **Docker MCP Toolkit profile integration.** Sandboxes connect to the canonical PMOVES MCP server bundle (`docker.io/darkxside/pmoves_5090_web` — 25 servers covering Hostinger DNS, the full Cloudflare suite, GitHub, DockerHub, Postman, Context7, DeepWiki, Cloud Run, and more) over an SSE host gateway. **No Docker-in-Docker required.** See [the PMOVES Toolkit guide](https://github.com/POWERFULMOVES/PMOVES.AI/blob/main/pmoves/docs/operations/MCP_TOOLKIT.md) § 6 (D-Proxy pattern).
- **PMOVES branding + persona overlay.** Sandboxes carry PMOVES agent identity (per the parent repo's [agent taxonomy](https://github.com/POWERFULMOVES/PMOVES.AI/tree/main/PMOVES-agents.md)). Danger Room is where validation-runs happen for any PMOVES agent — including MissingLinc when minted.
- **CHIT-aware audit surface.** Sandbox lifecycle events are designed to publish on `danger.room.*` NATS subjects so the fleet's observability stack ([`pmoves/docs/audit/CHIT_INTEGRATION_STATUS.md`](https://github.com/POWERFULMOVES/PMOVES.AI/blob/main/pmoves/docs/audit/CHIT_INTEGRATION_STATUS.md)) captures the run. (Sandbox-side bootstrap lands in a follow-up PR — see [`PMOVES.AI_INTEGRATION.md`](PMOVES.AI_INTEGRATION.md) for current state vs. planned.)

## Quick start (PMOVES operator)

```bash
# On a fleet node (e.g., the 5090):
make -C pmoves mcp-toolkit-bootstrap        # pulls + imports the pmoves_5090_web profile
make -C pmoves mcp-toolkit-secrets-sync     # populates docker-pass-style secrets from env.shared
make -C pmoves mcp-toolkit-gateway-start    # starts the SSE gateway on :8090 (background)

# Then, in agent code that creates a Danger Room sandbox:
```

```python
from e2b_code_interpreter import Sandbox

sandbox = Sandbox.create(envs={
    "MCP_GATEWAY_URL": "https://pmoves-5090.tail-scale.ts.net:8090/mcp/sse",
    "MCP_GATEWAY_AUTH_TOKEN": os.environ["MCP_GATEWAY_AUTH_TOKEN"],  # from env.shared
})
# Inside the sandbox, the bootstrap script (lands in Lane D-sandbox PR) wires
# the local MCP client at $MCP_GATEWAY_URL — agent sees the 25-server surface.
```

## Why this fork and not upstream

PMOVES.AI is a [Metal-Organic Framework (MOF)](https://github.com/POWERFULMOVES/PMOVES.AI/blob/main/pmoves/docs/architecture/PMOVES_MOF_ARCHITECTURE.md) for distributed machine intelligence — every fleet node is a pore in the lattice. Danger Room is the **isolation pore** where AI-generated work runs before it touches the lattice. The customizations on this fork are PMOVES-specific (MCP Toolkit profile wiring, persona overlay, CHIT trail), not upstream-appropriate. We track upstream `e2b-dev/E2B/main` on the fork's `main` branch and rebase Hardened customizations forward when upstream lands material we want.

If you're not building on PMOVES.AI, **use [upstream e2b-dev/E2B](https://github.com/e2b-dev/E2B) directly** — that's the right place for the vanilla SDK.

---

<!-- ============================================== -->
<!-- Below: upstream e2b-dev/E2B README, preserved verbatim. -->
<!-- ============================================== -->

# Upstream README — E2B Code Interpreter SDK

<!-- <p align="center">
  <img width="100" src="/readme-assets/logo-circle.png" alt="e2b logo">
</p> -->

![E2B SDK Preview](/readme-assets/e2b-sdk-light.png#gh-light-mode-only)
![E2B SDK Preview](/readme-assets/e2b-sdk-dark.png#gh-dark-mode-only)

<h4 align="center">
  <a href="https://pypi.org/project/e2b/">
    <img alt="Last 1 month downloads for the Python SDK" loading="lazy" width="200" height="20" decoding="async" data-nimg="1"
    style="color:transparent;width:auto;height:100%" src="https://img.shields.io/pypi/dm/e2b?label=PyPI%20Downloads">
  </a>
  <a href="https://www.npmjs.com/package/e2b">
    <img alt="Last 1 month downloads for the JavaScript SDK" loading="lazy" width="200" height="20" decoding="async" data-nimg="1"
    style="color:transparent;width:auto;height:100%" src="https://img.shields.io/npm/dm/e2b?label=NPM%20Downloads">
  </a>
</h4>

<!---
<img width="100%" src="/readme-assets/preview.png" alt="Cover image">
--->
## What is E2B?
[E2B](https://www.e2b.dev/) is an open-source infrastructure that allows you to run AI-generated code in secure isolated sandboxes in the cloud. To start and control sandboxes, use our [JavaScript SDK](https://www.npmjs.com/package/@e2b/code-interpreter) or [Python SDK](https://pypi.org/project/e2b_code_interpreter).

> [!NOTE]
> This repository contains the core E2B SDK that's used in our main [E2B Code Interpreter SDK](https://github.com/e2b-dev/code-interpreter).

## Run your first Sandbox

### 1. Install SDK

JavaScript / TypeScript
```
npm i @e2b/code-interpreter
```

Python
```
pip install e2b-code-interpreter
```

### 2. Get your E2B API key
1. Sign up to E2B [here](https://e2b.dev).
2. Get your API key [here](https://e2b.dev/dashboard?tab=keys).
3. Set environment variable with your API key
```
E2B_API_KEY=e2b_***
```     

### 3. Execute code with code interpreter inside Sandbox

JavaScript / TypeScript
```ts
import { Sandbox } from '@e2b/code-interpreter'

const sandbox = await Sandbox.create()
await sandbox.runCode('x = 1')

const execution = await sandbox.runCode('x+=1; x')
console.log(execution.text)  // outputs 2
```

Python
```py
from e2b_code_interpreter import Sandbox

with Sandbox.create() as sandbox:
    sandbox.run_code("x = 1")
    execution = sandbox.run_code("x+=1; x")
    print(execution.text)  # outputs 2
```

### 4. Check docs
Visit [E2B documentation](https://e2b.dev/docs).

### 5. E2B cookbook
Visit our [Cookbook](https://github.com/e2b-dev/e2b-cookbook/tree/main) to get inspired by examples with different LLMs and AI frameworks.

## Self-hosting

Read the [self-hosting guide](https://github.com/e2b-dev/infra/blob/main/self-host.md) to learn how to set up the [E2B infrastructure](https://github.com/e2b-dev/infra) on your own. The infrastructure is deployed using Terraform. 

Supported cloud providers:
- 🟢 GCP
- 🚧 AWS
- [ ] Azure
- [ ] General linux machine
