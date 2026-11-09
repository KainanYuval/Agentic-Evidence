<!-- SPDX-License-Identifier: Apache-2.0 -->

# Integrate Agentic Process Evidence

How to project [Agentic Process Evidence](../../schemas/agentic-process-evidence/agentic-process-evidence.md) (APE) into this project's evidence records.

**Status.** Proposed mapping for contributors. It is not an ingestion adapter, and nothing in `src/` reads these artifacts. The v0.1 JSON Schema is not published yet ([schemas/evidence/v0.1](../../schemas/evidence/v0.1/README.md)). Whether this project adopts the APE format, keeps its own schema and maps both ways, or co-designs v1.0 is open ([DECISIONS.md](../../DECISIONS.md#open-decisions), [HANDOFF.md](../../HANDOFF.md#4-open-questions)). Until that decision, treat every field below as a projection onto the record model in [ARCHITECTURE.md](../../ARCHITECTURE.md#31-evidencerecord), not as a new schema.

The copies under `schemas/agentic-process-evidence/` and `schemas/agentic-session-log/` are the reference for this guide. Wording matches the APE spec; relative links were retargeted so they resolve in this repository. Alignment evidence is not part of this guide.

## The two artifacts

APE splits one governed outcome from the runs that produced it.

| Artifact | Unit | Spec |
|---|---|---|
| Process evidence | One in-toto Statement v1 about a subject (git commit, artifact digest, application version, or a session when the process is a single session) | [agentic-process-evidence.md](../../schemas/agentic-process-evidence/agentic-process-evidence.md) |
| Session log | One agent run: prompts, tool calls, responses | [agentic-session-log.md](../../schemas/agentic-session-log/agentic-session-log.md) |

A process may reference many session logs (`predicate.sessionsLogs`). A session belongs to one process. Facts about one run stay on the session log. Facts about the governed outcome stay on the statement.

This project records one security-relevant event per [evidence record](../glossary.md#evidence-and-trust). An APE statement is not a record. A session log is not a record. Integration projects both into records, and keeps the original artifacts addressable by `uri` + `digest`.

## Rules for the projection

Apply these before mapping any field. They come from [CONSTRAINTS.md](../../CONSTRAINTS.md#4-evidence-and-claims-constraints) and [ARCHITECTURE.md §4](../../ARCHITECTURE.md#4-trust-model).

1. **One run per statement.** `run_id` is the process, not the session. Every record projected from that statement and its session logs shares that `run_id`.
2. **Check bytes before reading a log.** Fetch each `sessionsLogs[]` target and compare the bytes to `sessionsLogs[].digest`. On mismatch, do not project that log. Keep the statement's reference as an asserted record and leave the timeline absent.
3. **Do not invent fields the source does not have.** No OpenTelemetry ids, no `descriptor_digest`, no `caused_by`, no `authz.decision`, no `delegation_chain`, no `side_effect`. Absence is a gap, or a claim that stays `insufficient_evidence`.
4. **Digests and references, not raw content.** Store prompts, tool arguments, retrieved text, and `processSummary` as digests and refs. Do not copy `timeline[].prompt` or `tool_input` into a record payload.
5. **Trust comes from the channel.** An APE runtime tool uploading its own statement and logs is `channel: self-report`, so every projected record is `trust: asserted`. Matching a digest inside the same producer's statement does not raise trust: that is one source agreeing with itself.
6. **`collected_at` is the recorder clock.** Copy source timestamps into `event_time` only. Do not copy `createdAt` or `timeline[].ts` into `collected_at`.
7. **Synthetic data only** in anything committed to this repository (fixtures, examples, issues).

`observed` is available only when this project's recorder saw the bytes at a tap it controls. The APE runtime tool is not that tap. `verified` is available only when a `reference.pin` from outside the agent matches, or two independent sources agree on the same fact. A signature on the in-toto statement, once checked against a key that is not the agent's, verifies that those bytes were signed. It does not verify that the predicate is true.

## Step 1 — Bind the statement to a run

From one statement, emit:

| Record | `event_time` | Payload taken from the statement |
|---|---|---|
| `run.start` | `predicate.startTimestamp` | `scenario_id` has no APE equivalent; leave it empty until a scenario wraps this process. Carry `predicate.traceId` as an asserted process id. It is not an OpenTelemetry `trace_id`. |
| `run.end` | `predicate.endTimestamp` | `predicate.result` as the process status (`COMPLETED`, `FAILED`, `CANCELLED`, `APPROVED`, `REJECTED`, `ALIGNED`, `MISALIGNED`). |

Identity on both records, all asserted:

| APE field | Record field | Do not |
|---|---|---|
| `predicate.owner` | `identity.principal` | Do not use `createdBy`. That is who assembled the statement. |
| `predicate.providers[].agent.name` | `identity.agent` | One statement may list several providers. Emit one identity per provider stack, and do not collapse them. |
| `predicate.providers[]` | [Provider](../../schemas/agentic-process-evidence/agent-identifier.md#provider) object, by digest or ref | Harness, agent id, and language models have no v0.1 identity fields. Keep the object with the record. Do not rename `inferenceProvider` / `resolved`. |
| — | `identity.delegation_chain` | Leave empty. `owner` plus agent name is not a delegation chain. `reviewers` are overseers, not a chain. |

`predicate.traceId` stays a string on the run. Do not write it into `trace.trace_id`. Fabricating span ids makes [correlation](../../ARCHITECTURE.md#5-correlation) look complete when the source has no trace tree.

## Step 2 — Project the rest of the statement

Statement fields that are summaries, not events:

| APE field | Projection | Stop here |
|---|---|---|
| `subject[]` | Asserted subject ref on `run.start`: `uri` + `digest` (`gitCommit`, `sha256`, or another digest-set key) | Do not invent a record kind for the SDLC subject. v0.1 kinds do not include one. |
| `sessionsLogs[]` | One asserted reference per entry (`uri`, `digest`) | The reference is not the log. Project the timeline only after Step 3's digest check. |
| `predicate.tools[]` | Asserted name + version list on the run | A name in this list is not a `tool.invocation`. Do not synthesize invocations for tools that never appear as `preToolUse`. |
| `contextArtifacts[]` | `input.received` with `input_ref` = `uri` (or a ref to inline `data`) and `input_digest` from `digest` | Do not use `retrieval.result`. A policy, guideline, or instruction is an input to the process, not a document the agent retrieved, unless a timeline event shows that retrieval. `tags` stay on the payload. |
| `custom` | Asserted, process-specific object on `run.start` (for example `baseCommit`, `requirements`) | No v0.1 kind. Do not spread `custom` keys into correlation fields. |
| `intents`, `processSummary` | Digest + ref on `run.end` | Prose is not a `caused_by` link and not a claim status. |
| `reviewers` | Asserted string list on `run.end` | Not `authz.decision`, and not proof a human saw the outcome. |
| `predicateType`, `_type` | Informational only | `predicateType` does not select a schema in this repo. `_type` must be `https://in-toto.io/Statement/v1` or the artifact is not this mapping's input. |
| `createdAt`, `createdBy` | Asserted statement metadata on `run.end` | Not `collected_at`, not `identity.principal`. |

`predicate.result` values `APPROVED` and `REJECTED` are the process verdict. They are not an `authz.decision` for any tool call. `ALIGNED` and `MISALIGNED` belong to an alignment check; this guide does not define that statement.

When `subject[].digest` and a session log's `affectedEntities` disagree, project both. Do not pick a winner. Two asserted values that conflict are a contradiction only for a claim that names both; neither becomes `verified`.

## Step 3 — Project each session timeline

After the digest check, walk `timeline[]` in order. Standard overlay fields are `eventId`, `event`, and `ts`. Every other property keeps the producer's name.

| `event` | Record kind | Join key | Payload |
|---|---|---|---|
| `beforeSubmitPrompt` | `input.received` | — | `input_digest` over the prompt bytes, `input_ref` if the producer stored the prompt elsewhere. Do not store `prompt`. |
| `preToolUse` | `tool.invocation` | `correlation.invocation_id` = producer `tool_use_id` | `tool_name`. `args_digest` over `tool_input`. No `descriptor_digest`. No `side_effect` unless the source itself classifies one. |
| `postToolUse` | `tool.result` only when the event has a status or an output digest | same `tool_use_id` | `status` and `output_digest` only from fields that are actually those facts. |
| `afterAgentResponse` | none | — | Response text is not `agent.decision`. Leave it in the source log. |
| `sessionEnd` | none | — | Session end is not `run.end`. The process may contain further sessions. |
| any other value | none | — | Keep the event in the source log. Do not coerce harness-specific names into a v0.1 kind. |

Also:

- `sessionId` goes on each projected record as an asserted session id. It is not `run_id`. Producer `conversation_id` may repeat `sessionId`; do not create a second identifier when they match.
- `parentSessionId` is an asserted pointer to another session. It is not `trace.parent_span_id` and not `caused_by`.
- `path_hashes` are optional file fingerprints on a tool event. They are not `descriptor_digest` and not `output_digest`. If the only post-tool fact is `path_hashes`, do not emit `tool.result`.
- Order records by recorder `collected_at` once a recorder exists. Until then, preserve source order and the source `ts` as `event_time`. A later `ts` does not create `caused_by`.

`agent.decision` is the kind that carries `caused_by`. Emit it only when the source has an explicit causal link. APE timeline events do not. Prompt-then-tool order is not that link.

## Step 4 — Assign trust

| How the bytes arrived | `source.channel` | `trust` |
|---|---|---|
| APE runtime tool, harness hook export, or CI upload of the producer's own artifacts | `self-report` | `asserted` |
| This project's recorder tap saw the hook or statement bytes | `interception` | `observed` |
| Digest or signature checked against a `reference.pin` whose source is outside the agent, or a second independent observer agrees | `reference-check` | `verified` |

The recorder assigns trust. A field inside the APE document that says the event was observed does not count.

Component for harness-emitted events: `agent-runtime`. Use `tool:<name>` only for a record the tool itself emitted, not for the harness's `preToolUse` wrapper. Do not use `recorder-tap` for the APE runtime tool.

## Step 5 — Gaps a faithful import still has

[Gap rules](../../ARCHITECTURE.md#6-gap-rules-v01) evaluate records. They do not read the APE document. Emitting the projection above leaves these holes, and that is the result:

| Rule | Why a faithful APE import trips it |
|---|---|
| G-AUTHZ-001 | Session logs and process evidence have no per-invocation allow/deny. `predicate.result` does not fill `authz.decision`. |
| G-DESC-001 | `tools[].name` and `tool_name` identify a tool by name. There is no descriptor digest. |
| G-DESC-003 | There is no `reference.pin` for the descriptor that was invoked, unless an importer adds one from outside the agent. |
| G-ID-001 | `delegation_chain` is empty. `owner` does not say on whose behalf the tool ran. |
| G-OUT-001 | `postToolUse` often has no status and no output digest. `path_hashes` do not close this gap. |
| G-INFL-001 | Fires only if a projection sets `side_effect: external` and also emits `retrieval.result`. Do not set either in order to make the rule fire. Without an explicit `caused_by`, an influence claim is `insufficient_evidence` anyway. |

Do not expect G-TRACE-001 unless a projected record contains a `parent_span_id`. Omitting the trace is not the same as pointing at a missing span. The execution path is still unsupported: there is no trace tree to walk.

G-DESC-002 fires only when an invocation carries a `descriptor_digest` that no `tool.descriptor.observed` record matches. Name-only APE tools never reach that rule; they stop at G-DESC-001.

## Step 6 — What you may conclude

Built-in claims need the elements in [ARCHITECTURE.md](../../ARCHITECTURE.md#34-claim). For a projection that is entirely `asserted`:

| Question | Status a faithful import supports |
|---|---|
| Did the producer assert this tool name, this owner, this subject digest, this timeline? | `supported_by_assertion_only`, and only when those fields are present |
| Was the tool call authorized? | `insufficient_evidence` (G-AUTHZ-001) |
| Which descriptor ran, and was it the expected one? | `insufficient_evidence` (G-DESC-001, and G-DESC-003 if a pin is required) |
| On whose behalf? | `insufficient_evidence` (G-ID-001) |
| Did a prompt or a context artifact cause the tool call? | `insufficient_evidence`. Order and `processSummary` are not influence. |
| Did the process "detect" a poisoned retrieval or a descriptor change? | No such status. Do not write one. |

`supported` requires at least one required element at `observed` or `verified`. A self-reported APE upload never reaches `supported` on its own.

`contradicted` requires a conflict in required elements (invocation digest against a pin, or an authz `deny` followed by the invocation). Disagreement between `tools[]` and the timeline is two assertions. Report both. Do not promote the disagreement to `contradicted` unless a claim's required elements actually conflict.

## Worked projection

Source shape (a single `preToolUse` inside a log whose digest already matched):

```json
{
  "eventId": "evt_01EXAMPLE",
  "event": "preToolUse",
  "ts": "2026-04-08T08:51:18.441Z",
  "tool_name": "Write",
  "tool_use_id": "toolu_01EXAMPLE",
  "conversation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

Projected record. `collected_at` and `record_digest` are the recorder's job, so they are omitted here. `delegation_chain` is empty on purpose.

```json
{
  "schema_version": "0.1.0",
  "kind": "tool.invocation",
  "source": { "component": "agent-runtime", "channel": "self-report" },
  "trust": "asserted",
  "event_time": "2026-04-08T08:51:18.441Z",
  "correlation": { "invocation_id": "toolu_01EXAMPLE" },
  "identity": {
    "principal": "dev@example.com",
    "agent": "cursor-agent",
    "delegation_chain": []
  },
  "payload": {
    "tool_name": "Write",
    "session_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  }
}
```

`args_digest` is absent because the snippet has no `tool_input`. Do not fill it. The same call still fails G-AUTHZ-001, G-DESC-001, and G-ID-001.

## What this guide does not add

- An adapter, a CLI flag, or a scenario. Those wait on a schema decision and on milestones M1–M3.
- Alignment evidence, agent authentication, or an intents algorithm.
- Enforcement. This project does not block a merge or a release because a statement is missing.
- A signature on this project's bundle. v0.1 bundle integrity is a SHA-256 manifest. Saying the APE statement was signed is a claim about the source artifact, checked only by the procedure in Step 4.
- New record kinds. Fields with nowhere to go stay on the original artifact, referenced by digest, until a schema change adds a kind.
