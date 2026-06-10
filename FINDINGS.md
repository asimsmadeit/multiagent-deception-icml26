# Private Local Coding Agent — Research Findings & Architecture Audit

**Date:** June 10, 2026
**Inputs:** Deep-research pass (24 web sources, 119 claims extracted, 25 adversarially verified 3-vote, 24 confirmed / 1 refuted), supplemental research on memory systems + serving stacks, and a file-level audit of the Goose and OpenHands checkouts in `~/Downloads`.

---

## TL;DR Verdict

**The idea is worth building, but with one major revision.** The parts that hold up under evidence:

- ✅ **Model routing** (Bedrock Claude for planning, local model for implementation) — standard production practice, well validated.
- ✅ **Shared persistent memory over MCP** — the right integration seam; must be built externally since OpenHands has no native cross-task memory.
- ✅ **Bedrock for enterprise privacy** — PrivateLink + scoped endpoint policies + contractual no-training terms all check out.
- ✅ **Exceeding Copilot on long-context/codebase understanding** — achievable via a tree-sitter codebase knowledge graph + condensation + persistent memory, none of which Copilot does well.

The parts that don't hold up:

- ❌ **Two peer agent frameworks (Goose + OpenHands) handing off to each other** — the evidence is strongly against this exact shape. Keep both tools, but make one the orchestrator and consume the other as a tool/subagent, not a peer.
- ❌ **Distilling/finetuning a local model on Bedrock Claude planning traces** — this violates Anthropic's Commercial ToS (applies on Bedrock too). Distill from an *open* teacher instead; this is a validated, even better-documented path.
- ⚠️ **"Comparable to Claude Code, within reason"** — be realistic: locally hostable coders score ~51–62% SWE-bench Verified vs ~70–77% for frontier; on Terminal-Bench the gap is ~40 points. The hybrid routing design is exactly how you close most of the *practical* gap, because Bedrock Claude handles the hard 20%.

---

## 1. The dual-agent architecture: evidence says simplify

This was the most heavily verified area, and the findings were unanimous (3-0 verification votes):

- **MAST (NeurIPS 2025, arXiv:2503.13657):** "Despite enthusiasm for Multi-Agent LLM Systems (MAS), their performance gains on popular benchmarks are often minimal." The taxonomy identifies **14 failure modes** in 3 categories — system design issues, **inter-agent misalignment**, and task verification — across coding tasks and multiple model families. Prompt-level fixes recovered only +9–16%; the failures are structural.
- **Anthropic (Jan 2026):** "Multi-agent implementations typically use 3-10x more tokens than single-agent approaches for equivalent tasks." Multi-agent is justified only for context protection, parallel search, or genuine tool/domain specialization. Critically: **"Each handoff loses context"** — and Anthropic's recommended fix is to avoid planner-vs-implementer decomposition entirely, not to patch it with shared memory. Your memory layer *mitigates* but does not *eliminate* this failure class.
- **Survey of 13 production coding agents (arXiv:2604.03515):** sub-agent delegation is first-class in 5 of 13 agents — OpenHands has it natively via `AgentDelegateAction`. **All 10 agents that do multi-model routing do it inside a single scaffold; none use two separate agent frameworks.**
- Corroborating: arXiv:2604.02460 shows single agents match multi-agent systems under equal token budgets.

**What this means for your design:** Goose-orchestrates-OpenHands-as-a-peer is the weakest link. Pick one *primary* framework and consume the other's capabilities through MCP (which both speak fluently). Two viable shapes, in order of recommendation:

| Shape | How | Trade-off |
|---|---|---|
| **A (recommended): OpenHands primary** | OpenHands is the agent; its native LLM profiles give you planner/implementer/condenser model split in one scaffold; mount Goose's MCP servers (memory, computer-controller) as extensions for workspace/desktop tasks | Single agent loop = no handoff loss; loses Goose's nicer CLI/desktop UX |
| **B: Goose primary, OpenHands as a delegated tool** | Goose is your daily-driver CLI/desktop agent; for heavy multi-file coding tasks it invokes OpenHands headlessly (REST API or `openhands-sdk`) wrapped as an MCP tool, receiving back a structured result | Keeps Goose UX; handoff still exists but is one-directional, task-shaped ("here's a spec, return a diff"), which is the *least* harmful handoff form |

Either way: **the shared memory layer stays** — it's valuable for cross-*session* continuity and preference learning regardless of agent count. It just shouldn't be load-bearing for intra-task handoffs.

## 2. What the repo audit found (Goose 1.37.0, OpenHands 1.7.0)

Both repos in `~/Downloads` already cover more of your wishlist than expected:

**Goose** (`goose-main/`, Rust, Apache 2.0):
- Bedrock provider built in (`crates/goose/src/providers/bedrock.rs`), defaults to Claude Sonnet on Bedrock, IAM credential chain.
- OpenAI-compatible provider for local vLLM/SGLang/Ollama (`providers/openai_compatible.rs`).
- **Built-in Memory MCP server** (`crates/goose-mcp/src/memory/mod.rs`): `remember_memory`/`retrieve_memories`, local (`.goose/memory/`) + global (`~/.config/goose/memory/`) scopes. File-based, no semantic search — fine as a seed, not the long-term answer.
- Auto context compaction at 80% threshold (`context_mgmt/`), subagents (`agents/subagent_handler.rs`), recipes with sub-recipes, full MCP client (rmcp 1.4), REST server (`goose-server`), `.goosehints` repo conventions.
- Execution is **local subprocess** (process groups, env filtering) — no Docker sandbox by default. This matters for the security section below.
- No per-role multi-model assignment at agent level (one provider per session).

**OpenHands** (`OpenHands-main/`, Python, MIT):
- Bedrock via boto3/LiteLLM; any OpenAI-compatible endpoint via `base_url`.
- **Per-role LLM profiles** (`app_server/settings/llm_profiles.py`, up to 10) and a **separate condenser LLM config** — i.e., your "frontier plans, cheap model condenses/implements" split is a config exercise, not a build.
- Condenser types: noop / recent / llm / amortized / llm_attention (`config.template.toml:238-300`).
- MCP client via fastmcp (`app_server/mcp/mcp_router.py`).
- **Docker sandbox is the primary runtime** (`sandbox/docker_sandbox_service.py`), plus process/remote/Kubernetes options.
- Headless/programmatic use via `openhands-sdk` (Agent, LocalWorkspace) and the FastAPI REST surface.
- **No persistent cross-task memory** — verified both in the repo and in the literature (arXiv:2604.03515 §4.3.4): microagents load static instruction files but the agent never writes back. Your memory layer must be external.

**Integration seam:** both are MCP clients, and Goose's MCP servers run as plain processes (`goose mcp memory`). A standalone memory MCP server that both mount is plug-and-play — no Rust↔Python bridging needed.

## 3. Local model reality check (verified leaderboard numbers)

All from swebench.com Verified leaderboard (scaffold-sensitive — only compare within the same scaffold):

| Model | SWE-bench Verified | Notes |
|---|---|---|
| Claude 4.5 Opus (high reasoning) | 76.8% | bash-only scaffold |
| Claude 4 Sonnet (OpenHands scaffold) | 70.4% | your Bedrock planner class |
| Qwen3-Coder-480B-A35B (OpenHands) | 69.6% | open, but multi-GPU H100 class |
| **Qwen3-Coder-Next (80B-A3B, Feb 2026)** | **70.6%** (model card) | Apache 2.0, 3B active, 256K ctx — the standout local candidate |
| SWE-Hero-32B (distilled, NVIDIA) | 62.2% | see distillation section |
| Devstral Small 2512 | 56.4% | (72.2% in Mistral's own scaffold — scaffold sensitivity) |
| Qwen3-Coder-30B-A3B (OpenHands) | 51.6% | fits a single 24 GB GPU at INT4 |

Terminal-Bench 2.0 (tbench.ai, May 2026): best Claude entry 80.2% (Opus 4.7); open coder models sit around ≤40% (Qwen3-Coder-Next: 36.2%). **The terminal/agentic gap is much larger than the patch-writing gap** — another argument for routing agentic planning/driving to Bedrock and scoped implementation to the local model.

The refuted claim worth knowing: "top open models are within 4–7 points of Claude 4.5 Opus" failed verification 0-3. Treat open-model near-parity claims skeptically.

**Hardware (supplemental research):**
- Qwen3-Coder-30B-A3B at INT4 ≈ 17–18 GB → fits one RTX 4090 (24 GB). Official FP8 ≈ 30 GB → needs 2×A6000 (96 GB) comfortably, or a 32 GB RTX 5090 with tight KV budget.
- Mac M-series: Q4 GGUF (~18 GB) runs well on 32 GB+ unified memory via llama.cpp/MLX. Cap context — 256K KV cache is expensive.
- Qwen3-Coder-Next (80B-A3B): bigger footprint; check quantized sizing at implementation time.

## 4. Distillation: validated technique, but your teacher choice is illegal

- **The technique works:** NVIDIA SWE-Hero (arXiv:2604.01496, May 2026) distilled 300k execution-free + 13k execution-based agent trajectories from **Qwen3-Coder-480B** into Qwen2.5-Coder students via multi-turn SFT → 52.7% (7B), 60.8% (14B), 62.2% (32B) SWE-bench Verified. The trajectory dataset is **public on HuggingFace** (`nvidia/SWE-Hero-openhands-trajectories`) — and it was collected in an OpenHands scaffold, i.e., directly compatible with your stack.
- **But:** Anthropic's Commercial ToS — including the Anthropic-on-Bedrock terms that AWS's third-party-model page incorporates — prohibits using the services "to build a competing product or service, **including to train competing AI models**" without express approval. Anthropic actively enforces this.
- **Resolution:** distill from an open teacher (Qwen3-Coder-480B via a rented GPU burst, or skip collection entirely and use NVIDIA's published trajectories). Use Bedrock Claude for *inference-time planning only*, never as training data. You lose nothing: the open-teacher path is the one with published, verified results.

~62% is the realistic ceiling for a ≤32B local student. Also relevant: scaffold improvements recover real points without any training (EntroPO+R2E lifts Qwen3-Coder-30B from 51.6 → 60.4).

## 5. Shared memory layer: what to actually use

Verified: the memory layer **must be external** (finding §2), and MCP is the natural seam. Supplemental comparison of the self-hostable options:

| System | License | Self-host | MCP server for *shared* memory | Best at |
|---|---|---|---|---|
| **Mem0 / OpenMemory MCP** | Apache 2.0 | Docker Compose: API + Qdrant + Postgres (+Neo4j, +Ollama embeddings) | **Yes — purpose-built for "one local MCP memory shared across multiple clients"** | Preferences/facts, broadest adoption |
| Graphiti (Zep) | Apache 2.0 | Neo4j 5.26+ or FalkorDB; needs an extraction LLM (can be local) | Yes — official `mcp_server/` in repo | Temporal knowledge graph: "what did we decide and when did it change" |
| Letta (MemGPT) | Apache 2.0 | Docker + Postgres/pgvector | Indirect (third-party MCP wrappers); shared memory blocks are first-class *within* Letta | When agents themselves live in Letta — wrong fit as a passive sidecar |
| LangMem | MIT | Library only | No — you'd build the server | LangGraph ecosystems |
| Anthropic reference memory MCP | MIT | Trivial (JSON file) | Yes, but toy-grade | Demos only |
| Goose built-in memory MCP | Apache 2.0 | Already in your checkout | Yes (stdio) | Day-1 bootstrap before Mem0 is stood up |

Benchmark caveat: LoCoMo recall numbers are a vendor food-fight (Zep claimed 84% → Mem0's replication got 58.4% → Zep counterclaimed 75.1%). Don't choose on benchmark marketing; choose on architecture fit.

**Recommendation: Mem0/OpenMemory MCP** as the shared preference/episodic memory, **plus** a separate codebase-knowledge layer (next section). Graphiti is the runner-up if temporal "decision history" reasoning matters more to you than plug-and-play.

## 6. Codebase long-context: the verified technique

A tree-sitter-based **code knowledge graph exposed over MCP** (arXiv:2603.27277, "Codebase-Memory", Mar 2026; open-source `codebase-memory-mcp`, 159 languages in the shipped version):
- ~**10x fewer tokens** and 2.1x fewer tool calls than agentic file exploration,
- but 83% answer quality vs 92% for direct exploration (31 real repos).

Design implication (verified 3-0, confidence medium — single preprint, self-evaluated): use the KG for cheap recurring lookups, navigation, and cost control; **keep direct file exploration available for high-stakes answers.** Combined with OpenHands' LLM condenser and the persistent memory layer, this is your "beat Copilot on codebase context" stack.

## 7. Enterprise security: verified posture

**Bedrock (all verified against current AWS docs + legal terms):**
- Fully private access via **PrivateLink interface VPC endpoints** — no internet gateway, NAT, VPN; distinct endpoint services for control plane vs runtime (`com.amazonaws.{region}.bedrock-runtime`); FIPS variants in us-east-1/2, us-west-2, ca-central-1, GovCloud.
- **The default endpoint policy allows full access** — scope it to exactly `bedrock:InvokeModel` + `bedrock:InvokeModelWithResponseStream` as an exfiltration control.
- Legally binding no-training terms: "Anthropic may not train models on Customer Content from Services"; provider models run in AWS-controlled accounts the provider can't access. (Cite the legal terms page in any compliance review, not the marketing page. Qualifier: Anthropic receives aggregate usage-volume data, excluding content.)

**Prompt injection (arXiv:2506.08837, Google/Microsoft/IBM/ETH):**
- Model-level defenses "do not provide guarantees." The hard constraint: **once an agent has ingested untrusted input, it must be impossible for that input to trigger consequential actions.**
- Your planner/implementer split *resembles* the paper's Dual LLM pattern but **is not a security boundary** — the quarantined LLM in that pattern has *no tool access*, while your local implementer has full code execution. Your split is a cost-routing pattern. Security must come from sandboxing + egress control, structurally.
- Practical consequences: run the implementer in OpenHands' Docker sandbox (not Goose's local subprocess mode) for any task touching untrusted content (third-party code, web content, dependencies); default-deny network egress from the sandbox; treat MCP servers as part of the trust boundary.

**Local serving (supplemental, sourced):**
- **vLLM is the right default**: OpenAI-compatible, continuous batching, mature Qwen3-Coder support — but `--tool-call-parser qwen3_coder` is **mandatory** or tool calls won't parse. Pin ≥0.11.0 (CVE-2025-59425 api-key timing bypass fixed); ~6 high-severity CVEs since 2025, so patch cadence matters.
- **SGLang** if your workload is prefix-heavy multi-turn (RadixAttention, up to ~6.4x on shared-prefix workloads) or strict structured output.
- **llama.cpp** for Mac endpoints (use `--jinja` for Qwen3-Coder tool calls).
- **Avoid Ollama as a shared endpoint**: no auth at all, ~175k–300k internet-exposed instances found, CVE-2026-7482 ("Bleeding Llama," unauthenticated memory leak) among others. Fine on a personal laptop behind localhost.
- None of these ship real auth: bind to localhost/private VLAN, authenticating reverse proxy (nginx/Envoy + OIDC or API keys), TLS, no internet exposure, pinned versions.

## 8. Gaps in your original plan (things you were forgetting)

1. **Evaluation harness** — you can't tune the routing policy ("when does the local model handle it vs escalate to Bedrock?") without measurement. SWE-bench Lite subset + a Terminal-Bench slice + 10–20 tasks from your own repos.
2. **Routing/escalation policy with cost controls** — the local model fails on a meaningful fraction of tasks the planner could solve; you need explicit escalation triggers (N failed attempts, test-failure loops, confidence signals) and Bedrock spend caps/alerts.
3. **Audit logging for compliance** — every prompt, tool call, file write, and egress event logged locally; this is what actually makes the "enterprise-safe" claim defensible to a security team.
4. **Secrets hygiene** — secret-scanning/redaction before anything leaves for Bedrock (even over PrivateLink, your security team will ask); gitleaks/trufflehog hooks in the agent's edit path.
5. **Memory write policy** — unbounded automatic memory writes degrade recall and can launder injected instructions into persistent state. Gate writes (explicit tool call with provenance, periodic review/decay), and treat memory content as untrusted input when re-read.
6. **IDE integration** — Goose Desktop / OpenHands VS Code extension / ACP; without it you won't actually displace Copilot day-to-day.
7. **KV-cache and context budgeting** — 256K context sounds great until the KV cache evicts your batch; cap per-session context and lean on the condenser + KG instead.
8. **Scaffold tuning beats training** — before any finetuning, invest in the harness (better tool parsing, repair loops, repo map): it's worth ~9 points on a 30B model for zero training cost.

## 9. Standout thesis (alternative/extension track): distill to context, not weights

This section is additive — it doesn't replace the architecture above; it's the differentiator layered on top of it.

**The gap:** every hybrid agent treats the frontier model as a permanent component (static cost split, forever). The obvious fix — finetune the local model on Claude's outputs — is barred by Anthropic's ToS (§4). Everyone stops there. The unused channel is **context**: capture the frontier model's reasoning as reusable, auditable *playbooks* instead of weights.

**The mechanism:** when Bedrock Claude solves a novel planning problem, it produces two artifacts — the plan for this task, plus a generalized playbook ("how we add an endpoint to this service," "decision procedure for choosing a migration strategy here," "what broke last time + the check that catches it"). Playbooks live in the shared memory layer. Next similar task, the local Qwen model retrieves and executes the playbook **without calling Claude**.

**Why this wins:**
- **Legal distillation path.** Reusing outputs as working context is ordinary usage (CLAUDE.md files are exactly this); training competing model weights is not. Capability is captured in an artifact you own. (Caveat: persistent reuse sits in a grayer zone than nothing — materially different from weight training and almost certainly fine, but worth a vendor-agreement glance if this ships beyond personal/team use.)
- **Exploits a verified asymmetry.** Explicit procedures lift a 30B model far more than a frontier model — the weak model follows the plan instead of deriving it. The gap-closing mechanism targets exactly the model that needs it.
- **Compounds where finetuning can't.** A finetuned model is frozen and capped (~62%, SWE-Hero ceiling). A playbook library grows weekly, updates instantly when the codebase changes, and transfers for free to the next model generation.
- **Auditable.** "The agent's learned knowledge is a reviewable folder of procedures with provenance" is a compliance story no weight-based approach can tell. Humans can read, correct, or delete anything the system has learned.

**The novel metric: the frontier-call decay curve.** Track per task category what fraction of work required a Bedrock call, week over week, next to resolve rate. If the flywheel works, the curve declines — the system visibly weans itself off the expensive dependency as the library grows ("month 1: 40% of tokens frontier; month 3: 12%, same resolve rate"). Nobody reports this metric; every hybrid-routing writeup shows a static split. This is the result you can blog, publish, or pitch internally with a dollar figure attached.

**Thesis sentence:** *a coding agent that converts frontier-model calls into a permanent, auditable skill library, so each expensive call makes the next one less necessary.* Differentiated from Copilot (no memory), Claude Code (knowledge lives in rented weights), and local-agent projects (which accept the weak model's ceiling).

Implementation deltas are small — see Phase 4.5 in `IMPLEMENTATION_STEPS.md`.

## 10. Unresolved questions (researched but not settled)

- True mid-2026 gap between the largest open models (GLM-5, Kimi K2.5, DeepSeek V3.2) and Claude — the near-parity claim was refuted; verified comparisons date to Aug–Dec 2025.
- Whether Goose's orchestration adds enough over OpenHands' native delegation to justify shape B — this is a UX preference call, not an evidence call; the research found no verified evidence either way.
- Graphiti vs Mem0 recall quality — vendor benchmark dispute is unresolved; if it matters, run both against your own eval tasks.

---

## Sources (primary, verified)

- MAST failure taxonomy: arxiv.org/abs/2503.13657 (NeurIPS 2025)
- Anthropic multi-agent guidance: claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them (Jan 2026)
- 13-agent production survey: arxiv.org/pdf/2604.03515
- SWE-bench Verified leaderboard: swebench.com/verified.html
- SWE-Hero distillation: arxiv.org/html/2604.01496 + HF `nvidia/SWE-Hero-openhands-trajectories`
- Codebase-Memory tree-sitter KG over MCP: arxiv.org/abs/2603.27277
- Prompt-injection design patterns: arxiv.org/html/2506.08837v2
- Bedrock PrivateLink/VPC: docs.aws.amazon.com/bedrock/latest/userguide/usingVPC.html; legal: aws.amazon.com/legal/bedrock/third-party-models/
- Terminal-Bench 2.0: tbench.ai/leaderboard/terminal-bench/2.0
- Qwen3-Coder lineup: github.com/QwenLM/Qwen3-Coder; huggingface.co/Qwen/Qwen3-Coder-Next
- Memory systems: github.com/mem0ai/mem0; github.com/getzep/graphiti (mcp_server/); github.com/letta-ai/letta
- vLLM Qwen3-Coder recipe: docs.vllm.ai/projects/recipes/en/latest/Qwen/Qwen3-Coder-480B-A35B.html
- Anthropic-on-Bedrock Commercial ToS (no competing-model training): www-cdn.anthropic.com (Dec 2023 PDF) via aws.amazon.com/legal/bedrock/third-party-models/
