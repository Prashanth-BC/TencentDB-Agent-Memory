# TODO: harness-driven LLMRunner (fork of TencentDB Agent Memory)

## Where things stand
- Forked `TencentCloud/TencentDB-Agent-Memory` → `Prashanth-BC/TencentDB-Agent-Memory`
  (origin = fork, upstream = real repo, for pulling their updates later).
- Cloned to `~/workspace/TencentDB-Agent-Memory`.
- Default branch is `feat/server_team` (their team/user/ACL work), not `main`.
- Working branch: `feat/harness-llm-runner`, pushed to origin, currently empty (no code yet).
- Old `~/workspace/memory-pipeline` scaffold was deleted — abandoned in favor of forking
  their real codebase instead of building fresh.

## The goal
TDAI's L1 (atom) / L2 (scenario) / L3 (persona) distillation always calls an LLM via
`LLMRunnerFactory.createRunner().run()`. For the standalone/Gateway path (what Claude Code,
Codex etc. use via the proxy), that's `StandaloneLLMRunner` → `@ai-sdk/openai` → a server-held
`MEMORY_LLM_API_KEY`. We want distillation to use whatever model the attached harness session
is already running, no second key — same principle as fornix's `sleep_batch`/`sleep_commit`
(the storage service never calls a model itself).

OpenClaw's host already does this in-process (`OpenClawLLMRunner` wraps `runEmbeddedPiAgent`,
the host's own agent). The gap is only for separate-process CLIs (Claude Code, Codex) talking
through the Gateway/proxy — they have no open channel back into their process to ask "run this
completion." The realistic channel is **MCP sampling** (`sampling/createMessage`): if MemoryCore
ran as (or alongside) an MCP server, it could ask the *connected client* to sample instead of
calling a hosted API.

## Plan
1. Decide: new `hostType: "mcp"` with its own `HostAdapter`, or extend the existing
   `StandaloneLLMRunner` behind a config flag (`MEMORY_LLM_MODE=sampling`) so the Gateway/proxy
   path can opt in without introducing a fourth host? (open question — pick one before coding)
2. Implement `SamplingLLMRunner implements LLMRunner`:
   - `run(params: LLMRunParams): Promise<string>` → build an MCP `sampling/createMessage`
     request from `params.prompt` (+ tool definitions when `enableTools: true`), send it to
     the client connection tied to the current `RuntimeContext.sessionId`, return the text.
   - Needs MemoryCore to actually hold an MCP server connection per session to sample against
     — check whether `MemoryProxy` already terminates an MCP connection anywhere, or whether
     this means running MemoryCore itself as an MCP server (new transport work).
3. Trigger side is already fine as-is: `flushSession(sessionKey)` (used by `POST /session/end`)
   lets the harness force immediate L1 processing instead of waiting on the idle timer / cron.
   Decide whether idle-timer and cron-triggered L1/L2/L3 runs should be disabled entirely when
   running in sampling mode (no key to fall back to if nothing is attached), or just skipped
   silently.
4. Wire `enableTools: true` (used by L2 scene / L3 persona, which do file read/write/edit)
   through sampling too — MCP sampling responses are text-only, so the tool-call loop
   (`MAX_TOOL_ITERATIONS`, local `read`/`write`/`edit` tools in `llm-runner.ts`) needs
   re-checking: does MCP sampling support tool use in its negotiated capabilities, or does the
   tool loop have to move server-side (MemoryCore calls the tools itself, only delegates the
   *text generation* step to sampling)?
5. Tests: at minimum, a fake MCP client that answers sampling requests deterministically,
   exercised against L1 extraction, to confirm no `MEMORY_LLM_API_KEY` is read on that path.
6. Docs: note in README/INSTALL.md that this is a fork-specific mode, not upstream behavior,
   so future `git fetch upstream` + merges don't silently revert it without noticing.

## Key files already located (previous session)
- `MemoryCore/src/core/types.ts` — `LLMRunner`, `LLMRunnerFactory`, `HostAdapter`,
  `RuntimeContext` (has `agentContext?: "primary" | "subagent" | "cron" | "flush"` already).
- `MemoryCore/src/adapters/standalone/llm-runner.ts` — `StandaloneLLMRunner`, the one to
  replace/wrap. Vercel AI SDK (`generateText`/`streamText`), `MAX_TOOL_ITERATIONS = 20`,
  sandboxed `read`/`write`/`edit` tools scoped to `workspaceDir`.
- `MemoryCore/src/utils/pipeline-manager.ts` — `flushSession()` (harness-triggered immediate
  L1), `enqueueL1(sessionKey, triggerReason)`.
- `MemoryCore/src/core/state/types.ts` — task types: `"L1" | "L2" | "L3" | "flush" | "offload-l1"
  | "offload-l15" | "offload-l2"`.
- `MemoryCore/src/services/pipeline-worker.ts` — `case "flush"` falls back to `executeL1`.

## Codebase-memory trace (this session) — the actual wiring, more concrete than the plan above
- **`TdaiCore.wirePipelineRunners()`** (`MemoryCore/src/core/tdai-core.ts:671-776`) is the real
  decision point for step 1. It already has a config-flag-over-hostType pattern in production:
  `useStandaloneRunner = cfg.llm.enabled || hostAdapter.hostType !== "openclaw"`. The existing
  code comments explicitly say `hostType` branching "should be rare" and documents a case
  (`provider=proxy`) where `cfg.llm` must override the host's own runner factory. This is
  precedent for **option 2** (a config flag, e.g. `MEMORY_LLM_MODE=sampling`) over adding a new
  `hostType: "mcp"` — follow the grain of the existing code rather than widening the `hostType`
  union.
- Inside `wirePipelineRunners()`, the chosen `runnerFactory` is wrapped by
  `MetricTrackingRunnerFactory` (non-intrusive Kafka credit reporting, no-op without Kafka
  config) before `createRunner({ enableTools: false })` (L1) and
  `createRunner({ enableTools: true })` (L2/L3) are called. This decorator is factory-agnostic,
  so a new `SamplingLLMRunnerFactory` gets instrumentation for free — no changes needed there.
- **Second, independent integration point not in the original plan**: `TdaiCore.buildSkillLlmRunner()`
  (`MemoryCore/src/core/tdai-core.ts:992-1025`) constructs a `StandaloneLLMRunner` **directly**,
  bypassing `LLMRunnerFactory`/`HostAdapter` entirely, for skill extraction (always
  `enableTools: true`). Any sampling-mode work must special-case this path too, not just the
  factory override in `wirePipelineRunners()` — otherwise skill extraction keeps using
  `MEMORY_LLM_API_KEY` even when sampling mode is on everywhere else.
- `StandaloneHostAdapter.constructor` (`MemoryCore/src/adapters/standalone/host-adapter.ts:47-57`)
  always builds a `StandaloneLLMRunnerFactory` as the adapter's default `runnerFactory` — the
  fallback `wirePipelineRunners()` uses when no override applies.
- Net: a sampling-mode implementation touches at least 3 call sites
  (`wirePipelineRunners`, `buildSkillLlmRunner`, `StandaloneHostAdapter.constructor`), not just
  "swap the factory."

## Not yet decided (ask next session)
- hostType vs config-flag approach (step 1) — leaning config-flag per the trace above, confirm
  before coding.
- How `buildSkillLlmRunner`'s direct `StandaloneLLMRunner` construction should pick up sampling
  mode (new finding — needs its own decision, not just "wherever the factory is used").
- Whether to also port their `team`/`user`/ACL model (from `feat/server_team`) into fornix
  separately — this was a side-thread of the same conversation, unrelated to the LLMRunner
  work, don't conflate the two.
