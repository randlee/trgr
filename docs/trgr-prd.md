# trgr — Triage Graph: Product Requirements Document

**Status:** Draft for team handoff\
**Author:** Rand\
**Date:** October 27, 2026

## 1. Summary

trgr (Triage Graph) is a Rust CLI that injects known-problem knowledge into AI agent context at the right moment, scoped to the right agent. Tools and build systems emit stable error codes; trgr resolves those codes against a graph of curated entries and delivers the matching guidance to the agent that needs it, or escalates it to the phase lead.

In short: trgr is intelligent memory injection. It lets the expertise of a seasoned agent (for example, Phoenix, who diagnoses and fixes ATMBD problems across teams) ship alongside a tool, so other teams get that expertise without porting the agent itself.

## 2. Problem

- Complex tools (the motivating example is the ATMBD beads orchestration tool) fail in ways that require system-wide understanding. Individual agents on a team usually cannot diagnose them.
- Today, the fix knowledge lives in one expert agent's context, memories, and error logs. Porting an agent to other teams is impractical.
- Teams run up to 10 agents on a single repo. Broadcasting every error to every agent is noise; errors are individual (a compiler error belongs to the dev who caused it).
- Agents re-discover the same known issues repeatedly, wasting cycles, and nobody notices when an agent is stuck on something a lead must resolve.

## 3. Goals

1. Map stable error codes to curated guidance (problem, fix, runbook reference).
2. Deliver guidance only to the agent(s) it is relevant to, keyed on agent identity.
3. Escalate system-level problems to the phase lead with actionable instructions.
4. Let tool maintainers ship troubleshooting knowledge as part of a skill/tool package.
5. Keep the knowledge current: entries are retired when the underlying problem is fixed in code.

## 4. Non-goals (MVP)

- Parsing arbitrary tool output in real time via tool-use hooks (post-MVP).
- LLM-based fuzzy matching of errors (post-MVP, see Jev matcher).
- Debounce/suppression of repeated injections (post-MVP).
- A server or shared network database. trgr is a single local binary that runs to completion per invocation: no daemon, no Tokio, and SC runtime is out of scope. trgr's own logging uses SC observability.
- Coupling to any specific tool. ATMBD is the reference example only.

## 5. Users and roles

- **Dev agents:** receive guidance for errors they caused (e.g. compiler errors).
- **Phase lead / team lead:** receives escalations and tool/system-level problems.
- **Knowledge authors:** expert agents (e.g. Phoenix) or humans who record entries.
- **Tool maintainers:** ship trgr knowledge packs with their tool's installer/skill.

## 6. Core concepts

- **Code:** a stable, namespaced identifier emitted by a tool (example shape: `ATMBD-E042`, `RUST-E0502`). Codes are the contract between tools and trgr; trgr never needs to understand the tool itself.
- **Entry:** a graph record keyed by code containing: description of the problem, the fix, audience/routing, escalation rule, runbook reference (graph node or a document bundled in the skill), and applicable tool versions.
- **Category / layer:** a source of entries. Each category is a folder of Turtle files and becomes a named graph.
- **Identity:** the `ATM_IDENTITY` environment variable, set for every agent.

## 7. Data model and storage

### 7.1 Source of truth: Turtle in category folders

Knowledge is authored as Turtle (TTL) files organized by category, for example:

- `general/rust/` — common Rust error codes with a best-practices guide.
- `repo/` — repo-specific entries (e.g. repo-specific Rust error codes).
- `tools/<tool-name>/` — one category per tool, delivered and maintained with that tool's skill (e.g. `tools/atmbd/`).

Tool installers already drop many files on upgrade (ATMBD drops 100+), so shipping TTL rule files is straightforward.

### 7.2 Query store: embedded Oxigraph

- The `oxigraph` Rust crate is linked directly into the trgr binary. No server.
- All category folders compile into an Oxigraph store; each category is a named graph so codes are scoped by source and cannot collide.
- The store is a disposable, derived cache. TTL is always authoritative.
- Expected scale: a few hundred entries. Rebuild is expected to be sub-second.

### 7.3 Staleness and rebuild

- trgr detects changed or out-of-date sources and rebuilds when triggered. In practice this happens inside the hook.
- Requirements:
  - Staleness check by last-modified time of the TTL files.
  - Concurrency safety: multiple agents may detect staleness simultaneously. Use a lock so one process rebuilds; others wait briefly or use the previous store.
  - Atomic swap: build into a temporary location and swap in, so no reader sees a partial store.
  - Given the small scale, an in-memory rebuild per invocation is an acceptable alternative; the team should benchmark and choose.

### 7.4 Entry fields (minimum)

| Field | Purpose |
| --- | --- |
| code | Stable namespaced identifier |
| category | Source layer (named graph) |
| summary | What the problem is |
| fix | What to do |
| route | `causing-agent` or `phase-lead` (or both) |
| escalation | Rule describing when/whom to escalate and what to tell each party |
| runbook | Link to a graph node or a skill-bundled step-by-step document |
| applies\_to\_versions | Tool version range the entry is valid for |
| author / provenance | Who recorded it, from what evidence |

**Future reuse note — SC Oxigraph crate.** Oxigraph does not provide a TTL-folder-to-store sync layer. trgr will implement one internally: map category folders to named graphs, detect staleness via file modified times, and rebuild (in-memory load or temp build plus atomic swap). Once a second consumer needs it (likely grep-jev's Roslyn/syn graphs), extract it into a reusable SC ecosystem crate, with PyO3 bindings so it is easy to use from Python.

## 8. Detection and delivery

### 8.1 MVP: diagnostic mode + compaction hook

1. The tool's own CLI (e.g. ATMBD) runs in a special diagnostic mode that inspects current system state and emits error codes.
2. trgr resolves those codes to entries, filtered by identity and routing.
3. Matching guidance is injected through the existing context-injection hook that fires on compaction/restart, into the phase lead's context.

**Ingestion schema.** trgr defines its own ingestion schema; tools format emitted data to it. In-house tools (ATM and others using SC observability logging) emit directly or via log-file parsing. Tools that will not change (e.g. cargo) get trgr-side adapters.

### 8.2 Delivery adapters (keep open)

Lookup is a pure function: code(s) + identity → entries. Delivery is a pluggable adapter over it. Supported or planned adapters:

- Startup / compaction hook (MVP).
- Tool-use hook, injected as a memory (post-MVP).

The design must not hard-wire one delivery point.

## 9. Routing and identity

- **Code errors caught in a hook** (e.g. a compiler error) route to the agent that triggered them, identified by `ATM_IDENTITY`.
- **Tool/system errors found by a diagnostic scan** (e.g. ATMBD task-graph problems) route to the phase lead by default. Individual agents generally lack the system-wide understanding to fix these.
- Identity is used primarily for **filtering**: an agent should not see entries not relevant to it.

## 10. Escalation

- Escalation rules are encoded in the graph, per error type.
- When an escalatable error is detected in another agent's context, trgr can act on one or both agents:
  - Inject a message telling the dev agent to stop and wait.
  - Perform an ATM send to the team/phase lead.
- The escalation message must be self-contained: what the problem is, what the lead needs to do, and a pointer to more information (graph node or skill runbook).
- All notifications and escalations are normal ATM messages with sender trgr@tool. trgr registers with ATM's virtual tool team; its send permissions are whatever the tool team allows.

## 10a. Runtime state and cross-agent correlation

trgr keeps two kinds of data with different lifecycles: shared static knowledge (one graph, derived, disposable) and trigger events (authoritative, append-only, never wiped by a rebuild).

- **Location:** a gitignored `.trgr/` folder in the repo.
- **Per-agent event log:** one append-only JSONL file per agent, keyed by `ATM_IDENTITY`. Each event records code, detected time, notified time, and escalated time. Agents never write the same file, so no locking is needed.
- **Debounce:** before injecting, trgr checks the last notified time for that agent and code.
- **Readability:** logs are readable by the team lead, other agents, and trgr itself.
- **Cross-agent correlation:** if the same error appears in several agents' worktrees (e.g. dev1, dev2, dev3), it is likely in shared code and should be fixed elsewhere. A cheap mechanical match (same code across worktrees) nominates candidates; a Jev agent reviews snapshots of each incident and decides whether they are the same issue and whether to escalate.
- **Knowledge filtering:** identity scoping is a query filter over the one shared graph (entries declare audience/role), not separate per-agent graphs.

## 11. Knowledge lifecycle

### 11.1 Authoring

- `trgr add` records a new entry into the appropriate category's TTL.
- A companion skill carries general knowledge of how to author good entries for a tool.
- Initial ATMBD pack is built by Phoenix from his knowledge, memories, and observed error logs.

### 11.2 Retirement

- Fixing the error in the tool's code is always preferred over carrying a workaround.
- When a tool release fixes an error, its trigger must be removed. Mechanisms:
  - Version scoping (`applies_to_versions`) so entries auto-retire after upgrade.
  - Tool packs simply omit fixed entries in the next release drop.

## 12. CLI surface (proposed)

- `trgr lookup <code>...` — resolve codes for the current identity.
- `trgr inject --adapter <compact|tooluse>` — produce injection payload for a hook.
- `trgr scan --tool <name>` — run a tool's diagnostic mode and resolve results.
- `trgr add` — record a new entry.
- `trgr build` / `trgr check` — explicit rebuild and staleness check.
- `trgr validate` — lint TTL packs (schema, duplicate codes, missing runbooks, version ranges). Plus trgr doctor: flags codes present in more than one layer so an agent can record a precedence or removal decision in the graph.

Exact verbs are for the team to finalize.

## 13. Post-MVP roadmap

1. **Tool-use hook monitoring:** detect codes as tools run, not only at compaction.
2. **Jev matcher:** pipe all cargo errors through a Jev agent (local small model) to pattern-match against known issues/triggers, covering errors without explicit codes.
3. **Debounce:** track last-injected per (identity, code); re-inject only on recurrence after a fix or persistence past a threshold, so agents aren't nagged about things they already know.
4. **Effectiveness signal:** record whether the error cleared after injection; feed debounce and entry quality.

## 14. Success criteria

- A team without Phoenix can resolve the known ATMBD failure set using trgr guidance alone.
- Dev agents receive only entries for errors they caused; no cross-agent noise.
- Escalations reach the phase lead with enough information to act without follow-up questions.
- Store rebuild is effectively instantaneous at expected scale and safe under 10 concurrent agents.
- Fixed issues disappear from injection after the corresponding tool upgrade.

## 15. Decisions

1. **Layer precedence:** default is to inject all matching entries. trgr doctor flags overlaps; an agent records in the graph either an explicit precedence (e.g. prefer repo) or removal of the redundant entry.
2. **Code namespaces:** each tool owns its codes via its existing error-ID ledger, versioned with semver so derivative tools and skills stay consistent. Entries declare the ledger versions they apply to.
3. **Change detection:** last-modified time of TTL files. Persistent shared store vs in-memory rebuild per call is left to benchmarking; both are cheap at expected scale.
4. **Escalation attribution:** messages come from trgr@tool, not the dev agent.
5. **Action trust boundary:** trgr acts as a registered member of ATM's tool team.
6. **QA findings triage:** out of scope. trgr is a separate system; the old QA triage graph has been superseded by ATM beads.
7. **Diagnostic contract:** trgr owns an ingestion schema; adapters for unchangeable tools such as cargo; log parsing for SC observability logs.

Remaining for the team: the exact JSON shape of the ingestion schema and event records, and the debounce thresholds.

## 16. Deliverables

1. trgr Rust binary with embedded Oxigraph.
2. TTL ontology/schema for entries.
3. Compaction-hook adapter integrated with the existing context-injection hook.
4. Diagnostic-mode output contract and ATMBD reference implementation.
5. Initial ATMBD knowledge pack authored by Phoenix.
6. Authoring skill and `trgr add` / `trgr validate`.
7. Documentation: author guide, tool-integration guide.
