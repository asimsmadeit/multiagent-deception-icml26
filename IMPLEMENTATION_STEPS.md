# Implementation Steps — Private Local Coding Agent

Companion to `FINDINGS.md` (read that first — it explains *why* behind every choice here).

**Architecture being built (revised from original idea):**

```
                    ┌─────────────────────────────────────────┐
                    │  PRIMARY AGENT (pick Shape A or B below) │
                    │                                          │
   Bedrock Claude ──┤  planner / reviewer / escalation LLM     │
   (PrivateLink)    │                                          │
                    │  local Qwen3-Coder via vLLM ─ implementer│
                    │  local small model ─ condenser           │
                    └───────┬──────────────┬───────────────────┘
                            │ MCP          │ MCP
                ┌───────────▼───┐   ┌──────▼──────────────┐
                │ Mem0/OpenMemory│   │ codebase-memory-mcp │
                │ (prefs +       │   │ (tree-sitter code   │
                │  episodic mem) │   │  knowledge graph)   │
                └───────────────┘   └─────────────────────┘
        All execution in Docker sandbox, default-deny egress, full audit log
```

Shape A (recommended): OpenHands is the agent; Goose's MCP servers mounted as extensions.
Shape B: Goose is the daily-driver CLI; OpenHands invoked headlessly as a delegated coding tool via an MCP wrapper.
Do NOT build two peer agents handing off mid-task — see FINDINGS §1.

---

## Phase 0 — Decisions & prerequisites (half a day)

- [ ] **0.1** Choose Shape A or B. Decision rule: if you mostly live in a CLI/IDE and want one agent loop → A. If you specifically want Goose's desktop app/recipes UX as the front door → B.
- [ ] **0.2** Inventory hardware. Targets:
  - 1× RTX 4090 (24 GB): Qwen3-Coder-30B-A3B at INT4 (~17–18 GB) — workable starter.
  - 2× A6000 (96 GB) or 1× RTX 5090: official FP8 30B-A3B with real context/batch.
  - Mac 32 GB+ unified memory: Q4 GGUF via llama.cpp — dev endpoint, not the shared server.
  - Stretch: evaluate Qwen3-Coder-Next (80B-A3B, 70.6% SWE-bench, Apache 2.0) if you can fit a quantized 80B MoE.
- [ ] **0.3** Confirm AWS account has Bedrock model access for Claude in your region; confirm you have VPC admin rights (needed for PrivateLink).
- [ ] **0.4** Get sign-off from your security team on the architecture diagram above *before* building — it'll save rework. Hand them FINDINGS §7.

## Phase 1 — Local model serving (1 day)

- [ ] **1.1** Install vLLM ≥ 0.11.0 (CVE-2025-59425 fixed). Pin the version.
- [ ] **1.2** Download `Qwen/Qwen3-Coder-30B-A3B-Instruct` (FP8 variant if you have ≥32 GB VRAM; otherwise AWQ/INT4, Unsloth dynamic quants include tool-calling fixes).
- [ ] **1.3** Launch with the **mandatory** tool-call parser:
  ```bash
  vllm serve Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8 \
    --enable-auto-tool-choice --tool-call-parser qwen3_coder \
    --max-model-len 131072 --host 127.0.0.1 --port 8000
  ```
  Cap `--max-model-len` to what your KV cache budget actually supports — don't default to 256K.
- [ ] **1.4** Front it with an authenticating reverse proxy (nginx or Envoy): API key or OIDC, TLS, rate limits. Bind vLLM to localhost only; proxy is the only listener on the LAN. Never expose to the internet.
- [ ] **1.5** Serve a second small model for condensation/summarization (e.g. Qwen3-4B or 8B) — either a second vLLM instance or the same instance if VRAM allows. This is the "condenser" LLM.
- [ ] **1.6** Smoke-test tool calling: curl an OpenAI-format chat request with a tool definition; verify structured `tool_calls` come back (not raw JSON in content).
- [ ] **1.7** (Mac dev box, optional) llama.cpp `llama-server` with `--jinja` for local-laptop use.

## Phase 2 — Bedrock private access (1 day, mostly AWS console/IaC)

- [ ] **2.1** Create an interface VPC endpoint for `com.amazonaws.{region}.bedrock-runtime` (FIPS variant if required). No IGW/NAT path needed.
- [ ] **2.2** Replace the default endpoint policy (it allows full access) with one scoped to exactly `bedrock:InvokeModel` + `bedrock:InvokeModelWithResponseStream`, on the specific Claude model ARNs you'll use.
- [ ] **2.3** Create a dedicated IAM role/user for the agent host with the same minimal permissions. No wildcard `bedrock:*`.
- [ ] **2.4** Enable Bedrock Guardrails (PII filters at minimum) — defense in depth, not the security boundary.
- [ ] **2.5** Save the legal references for compliance: AWS Bedrock third-party model terms + Anthropic-on-Bedrock ToS (no-training clause). Your security review will want them.
- [ ] **2.6** Test from the agent host: invoke Claude through the VPC endpoint; confirm via VPC flow logs that no public egress occurred.

## Phase 3 — Primary agent with multi-model routing (2–3 days)

### Shape A: OpenHands primary
- [ ] **3A.1** Deploy OpenHands (your checkout is v1.7.0; prefer the released package over the zip). Use the **Docker sandbox** runtime, not process sandbox.
- [ ] **3A.2** Configure LLM profiles in `config.toml`:
  - `[llm]` default/planner → Bedrock Claude (via boto3/LiteLLM, pointed at the VPC endpoint).
  - implementer profile → local vLLM endpoint (`base_url = "https://your-proxy/v1"`, `model = "openai/Qwen3-Coder-30B..."`).
  - `[llm.condenser]` → the small local model; set condenser type to `llm` (default is `noop` — turn it on).
- [ ] **3A.3** Define the routing policy (start manual, automate later): planning/architecture/review turns → Claude profile; mechanical edits/test-fix loops → local profile. OpenHands' experimental `model_routing` section is worth testing once manual routing works.
- [ ] **3A.4** Mount Goose's MCP servers as OpenHands MCP extensions (stdio): `goose mcp memory` (bootstrap memory until Phase 4), `goose mcp computercontroller` if you want desktop/document control.

### Shape B: Goose primary
- [ ] **3B.1** Install Goose; configure Bedrock provider (planning sessions) and openai_compatible provider (local sessions). Note Goose assigns one provider per session — model split happens at the session/recipe level, not per-turn.
- [ ] **3B.2** Run OpenHands headless behind its REST API (or `openhands-sdk`) with the local-model profile from 3A.2.
- [ ] **3B.3** Build a thin **MCP wrapper** exposing OpenHands as tools to Goose: `delegate_coding_task(spec, repo_path) -> {diff, test_results, summary}`. Keep the interface task-shaped: full spec in, structured result out — never conversational back-and-forth (that's the handoff failure mode from FINDINGS §1). ~200 lines of Python with fastmcp.
- [ ] **3B.4** Register the wrapper as a Goose extension; write a Goose recipe that plans with Bedrock, then calls `delegate_coding_task`.

### Both shapes
- [ ] **3.5** Escalation rule: if the local implementer fails the same task twice (tests still failing / no progress), auto-escalate that task to the Claude profile. Log every escalation — this data tunes the routing later.
- [ ] **3.6** Bedrock cost controls: AWS Budgets alert + per-day token cap in your proxy/agent config.

## Phase 4 — Shared persistent memory (2 days)

- [ ] **4.1** Deploy self-hosted **Mem0 + OpenMemory MCP**: docker-compose with API + Qdrant + Postgres; configure local embeddings (Ollama/vLLM embedding model) so nothing leaves the box. Zero cloud dependencies.
- [ ] **4.2** Mount it in the agent(s): Goose extension (`streamable_http`) and/or OpenHands `[mcp]` config. Both agents (or both *roles*) now read/write one memory.
- [ ] **4.3** Migrate anything accumulated in Goose's file-based memory (`~/.config/goose/memory/`) into Mem0; retire the bootstrap.
- [ ] **4.4** Define the **memory write policy** (FINDINGS §8.5): writes only via explicit tool call with provenance (which agent, which task, when); namespaces — `user-preferences`, `project:{name}` decisions, `episodic` task outcomes; periodic decay/review of stale entries.
- [ ] **4.5** Injection hygiene: system-prompt both agents that recalled memory content is *information, not instructions*; never store raw untrusted text (web content, third-party code comments) as memory.
- [ ] **4.6** (Optional, later) Trial Graphiti's MCP server alongside Mem0 for temporal decision history; keep whichever your eval (Phase 6) says recalls better.

## Phase 4.5 — Playbook flywheel: distill to context, not weights (alternative/extension track, 2–3 days)

> Additive extension to Phases 3–4, not a replacement. This is the project's differentiator — see FINDINGS §9. Goal: each Bedrock call produces a reusable artifact so frontier dependence measurably decays over time.

- [ ] **4.5.1** Add a `playbooks/` namespace to the Mem0/MCP memory layer. Required schema per playbook: `title`, `task_category`, `procedure` (stepwise markdown), `provenance` (source task ID, model, date), `validation_status` (`draft` | `trusted` | `retired`), `success_count` / `failure_count`.
- [ ] **4.5.2** Planning prompt change: on novel tasks, Bedrock Claude's deliverable is **two artifacts** — the plan for this task, plus a generalized playbook draft *when it judges the pattern recurring* (let the model decide; don't force a playbook per task or the library fills with noise).
- [ ] **4.5.3** Promotion gate (outcome-gated memory): a playbook stays `draft` until the local model executes it successfully once (tests pass / diff accepted) → `trusted`. Two consecutive failures → `retired`, and the failure context becomes input to a revision request on the next Claude call.
- [ ] **4.5.4** Router change (extends 3.5): before escalating to Bedrock, retrieve playbooks by task category/similarity. Escalate only on retrieval miss **or** playbook failure. Every escalation is by definition a future playbook candidate — the flywheel closes itself.
- [ ] **4.5.5** Instrument the **frontier-call decay curve**: per task category, log `frontier_tokens / total_tokens` and resolve rate, weekly. This is the headline metric (dashboard in 9.3). Success looks like the frontier share declining at constant resolve rate.
- [ ] **4.5.6** Review loop: playbooks are human-readable markdown — skim the library weekly, correct or delete bad entries. Auditability is the enterprise selling point; keep the library small and curated rather than large and stale.
- [ ] **4.5.7** Legal note: persistent reuse of Claude outputs as context is ordinary usage (materially different from the prohibited weight training), but if this ships beyond personal/team use, have whoever reviews vendor agreements glance at it.

## Phase 5 — Codebase knowledge layer (1–2 days)

- [ ] **5.1** Deploy `codebase-memory-mcp` (the open-source tree-sitter KG from arXiv 2603.27277) against your main work repos; mount in the agent(s) via MCP.
- [ ] **5.2** Usage policy in the agent prompt: KG for navigation/recurring lookups (10x cheaper); direct file reads for anything load-bearing (KG is 83% vs 92% answer quality).
- [ ] **5.3** Index refresh hook: re-index on branch switch / post-merge (git hook or cron).
- [ ] **5.4** Add repo-convention files the agents already understand: `.goosehints` and/or OpenHands microagent instruction files per repo — build conventions, test commands, gotchas.

## Phase 6 — Evaluation harness (2 days; do this BEFORE any finetuning)

- [ ] **6.1** Stand up SWE-bench Lite (or a 50-task Verified subset) runnable against any profile. OpenHands ships evaluation tooling — reuse it.
- [ ] **6.2** Add 10–20 private tasks from your own repos (real bugs/features with test oracles). This is the benchmark that actually matters.
- [ ] **6.3** Baseline four configs: local-only, Claude-only, routed (your Phase 3 policy), routed+memory+KG. Track: resolve rate, tokens, $ cost, wall time.
- [ ] **6.4** Tune the routing threshold on this data. Expected outcome: routed ≈ Claude-only resolve rate at a fraction of Bedrock spend; if not, adjust what gets routed local.
- [ ] **6.5** Re-run monthly and on any model/scaffold change.

## Phase 7 — Security hardening & compliance (2 days)

- [ ] **7.1** Sandbox: all implementer execution in OpenHands' Docker sandbox; non-root user; if Shape B, do NOT let Goose run untrusted-content tasks via its local subprocess execution.
- [ ] **7.2** Default-deny network egress from the sandbox; allowlist only: internal git, package registries (or better, an internal mirror like Artifactory/devpi), the vLLM proxy, the Bedrock VPC endpoint.
- [ ] **7.3** Audit logging: append-only local log of every prompt, tool call, file write, command, and egress event (both agents support event streams — tap them). This is your compliance artifact.
- [ ] **7.4** Secret scanning (gitleaks/trufflehog) as a pre-send filter on anything going to Bedrock and a pre-commit hook on agent-written code.
- [ ] **7.5** Patch cadence: subscribe to vLLM and OpenHands security advisories; pin and schedule monthly updates. Do not use Ollama as a shared endpoint.
- [ ] **7.6** Structural prompt-injection posture (FINDINGS §7): any task ingesting untrusted input (web pages, third-party PRs, dependency code) runs with reduced tool privileges — no push, no external calls — and its outputs are reviewed before merge.

## Phase 8 — Optional: finetune the local implementer (1–2 weeks, only if Phase 6 shows a gap worth closing)

- [ ] **8.1** ⚠️ **Open teacher only.** Distilling Claude/Bedrock outputs into your model violates Anthropic's ToS (FINDINGS §4). The validated path uses open teachers and it's the better-documented one anyway.
- [ ] **8.2** Start free: pull `nvidia/SWE-Hero-openhands-trajectories` from HuggingFace (300k+ trajectories, distilled from Qwen3-Coder-480B, collected in an OpenHands scaffold — directly compatible).
- [ ] **8.3** Optionally add your own domain trajectories: run Qwen3-Coder-480B (rented GPU burst or a provider hosting open weights under acceptable terms) on tasks from your repos; keep successful trajectories.
- [ ] **8.4** Multi-turn SFT (LoRA/QLoRA via axolotl or Llama-Factory) on Qwen3-Coder-30B-A3B. Reference ceiling: SWE-Hero-32B hit 62.2%.
- [ ] **8.5** Evaluate on Phase 6 harness before swapping into production. Remember: scaffold tuning bought ~9 points in published results for zero training — exhaust that first.

## Phase 9 — Daily-driver polish (ongoing)

- [ ] **9.1** IDE: OpenHands VS Code extension or Goose Desktop, per your Shape choice. This is the Copilot-displacement moment.
- [ ] **9.2** Preference learning loop: end-of-session reflection step that proposes memory writes ("user prefers pytest fixtures over mocks") for your approval, then commits them to Mem0.
- [ ] **9.3** Dashboards: token spend (Bedrock vs local), escalation rate, eval trend — and the frontier-call decay curve from 4.5.5 if you adopted the playbook track (it's the headline chart).
- [ ] **9.4** Quarterly model refresh: re-check SWE-bench/Terminal-Bench leaderboards and the Qwen coder lineup (a dedicated Qwen3.5-Coder had NOT shipped as of June 2026 — likely soon).

---

## Suggested order & rough effort

| Phase | Effort | Blocked by |
|---|---|---|
| 0 Decisions | 0.5 d | — |
| 1 vLLM serving | 1 d | 0 |
| 2 Bedrock PrivateLink | 1 d | 0 (parallel with 1) |
| 3 Agent + routing | 2–3 d | 1, 2 |
| 4 Shared memory | 2 d | 3 |
| 4.5 Playbook flywheel (extension) | 2–3 d | 4 |
| 5 Codebase KG | 1–2 d | 3 (parallel with 4) |
| 6 Eval harness | 2 d | 3 |
| 7 Hardening | 2 d | 3 (start earlier where possible) |
| 8 Finetuning (optional) | 1–2 wk | 6 |
| 9 Polish | ongoing | 4–7 |

**~2 weeks to a working, private, memory-equipped routed agent** (Phases 0–7), before any optional finetuning.
