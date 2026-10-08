# AGENTS.md — instructions for Codex and other coding agents

## Mission

Build **pi-live-context**, a local, live, bidirectional conversation-context inspector/editor for the **Pi coding agent** (https://github.com/earendil-works/pi).

The user wants to see EVERYTHING Pi can actually observe about an agent conversation, without truncating or silently rewriting it: full system instructions, tool definitions, user/assistant messages, structured content blocks, thinking blocks if provided, tool invocations with arguments and results, session metadata, changes, and compaction. They want to modify the *future model-facing conversation* from a locally hosted browser UI. The project must distinguish faithfully between (a) original stored history, (b) effective request-time transcript, (c) provider-adapted request payload, and (d) model-server KV cache, which Pi cannot inspect.

This is an **implementation assignment**, not a request for further reconnaissance-only caveats. Produce working code, a demo, and tests. When an upstream API prevents an operation, implement the nearest accurate behavior and expose the limitation explicitly in the UI and documentation.

## Ground-truth references (check the installed version)

- Pi source: https://github.com/earendil-works/pi
- Extensions: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md
- Extension event and API types: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts
- Session format: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/session-format.md
- Session manager: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts
- Messages: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/message-types.md
- SDK: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md
- Examples: https://github.com/earendil-works/pi/tree/main/packages/coding-agent/examples/extensions
- Read docs/CONTEXT_AND_DESIGN.md and docs/CODEX_TASK.md in this repository before coding.

Pi changed substantially in 2026. Use `@earendil-works/pi-coding-agent`, not obsolete `@mariozechner/*` examples. Verify signatures from the installed package. Do not assume a pre-2026 extension event contract.

## Essential conceptual invariants

1. **Original session data is append-only** JSONL with a tree of `id` and `parentId` entries. Stored order is not the same as active model context. Show both.
2. **Effective context is a projection** of the active branch, compactions, context edits, prompt/tool state, and request-time transforms. Show the assembled full transcript and identify the moment it was captured.
3. **Provider payload is another layer.** If observing `before_provider_request`, label it as a Pi-visible provider payload, not verbatim network bytes, and do not assume it matches the generic transcript.
4. **KV cache is not JSON history.** No provider's internal activation/cache memory is inspectable via Pi merely because prompts were sent. Never imply otherwise.
5. **Raw means RAW.** Preserve every property and content block accessible through public APIs, including unknown types, images represented as data/metadata, opaque signatures, usage fields, and provider-specific extras. A pretty view is additive; always offer literal JSON.
6. **Live means live.** Existing viewers must receive changes when messages stream, finish, when tools complete, when branches change, and on session switches. Show partial/streaming vs finalized/persisted status honestly.
7. **Bidirectional means the NEXT real model request uses approved changes**, not just that an HTML field changed. Durable edits must survive browser reload and (once applied) Pi restart. Keep the untouched original.
8. **Editing cannot retroactively change provider requests already submitted**, nor remove information from a cache already populated. Surface this boundary rather than disguising it.
9. **No manual on-disk JSONL rewrite.** This conflicts with Pi's live in-memory authority and append-only session manager. Prefer supported context-edit drafts and session interfaces.
10. **No exfiltration.** All raw prompts and tool results may contain secrets. Keep every interaction local and never send telemetry or captured data to third parties.

## Required UI modes

- **History / original:** all persisted or in-memory session entries, full chronological append order, with their `id`, `parentId`, timestamps, roles, type, and raw JSON; distinguish inactive branches, model/usage metadata, and custom entries.
- **Active branch:** the selected path through the session tree with provenance labels.
- **Effective transcript:** the complete Pi-visible full transcript from the latest `context_with_system` event, including system messages and tool declarations; show exact snapshot and request sequence number, not just a recomputed guess. Before first request show an explicitly labeled provisional projection, if available.
- **Provider request (optional but desirable):** last Pi-visible `before_provider_request` payload; distinguish it from actual HTTP frames and server state.
- Allow expanding individual structured blocks and copying/exporting original JSON, searching, navigating diffs, and watching updates without refreshing.

Do not truncate fields silently. Huge values may be collapsed in presentation, but accessible intact through deliberate expand/download operations.

## Editing requirements

- MVP: edit or omit the `content` of an eligible **user, assistant, toolResult, or custom_message** entry on the active branch. Preserve role, metadata and original content.
- Explicit **Save/Apply** with optimistic concurrency (session ID, active leaf, snapshot revision, expected original target). Display pending vs applied vs rejected.
- Use Pi's native `context_edit` entries where the supported extension API allows them (e.g. `turn_end` and `agent_before_settle` boundary drafts). `ExtensionContext.sessionManager` is read-only; DO NOT cast it to a writable manager and call private methods to fake support.
- A `context_with_system` handler can transform the request-local transcript, but is NOT by itself a durable edit. If a pending edit needs request-time overlay until the next boundary commit, show this transitional state and test it. Do not claim durability until actually committed.
- If edits cannot be committed while Pi is idle through the current supported API, state this clearly and implement safe queueing to the next supported boundary, with precise confirmation and restart/recovery semantics. Never silently call it instantaneous persistence.
- System-prompt/tool declaration edits are a **separate capability**. `context_edit` does not support them. Consider a dedicated request-local prompt transform, if tested; show whether it is ephemeral, persistent or unsupported. Never pretend a system message was edited in the session log when it was not.
- Reject stale, out-of-branch, invalid-role, malformed, and in-flight race edits. Do not mutate a provider request already in progress. No unrequested auto-continue/model calls.
- Display source, edit entry, effective replacement/omission, applied sequence and rollback/reversion method. Revert by adding a new edit rather than destroying historical records, if API semantics permit.

## Implementation guidance

- Start with a Pi **extension** (TypeScript) loaded by `pi --extension ./extensions/index.ts`; do not fork the Pi harness.
- Pi extension factory should register handlers only; long-lived sockets/server/watchers start on `session_start` and stop on `session_shutdown`. Prevent duplicate servers on reload.
- Bind a small Node HTTP server to `127.0.0.1` (optional configurable loopback port), serve the web app from the same origin, and use SSE or WebSocket for updates. A REST command endpoint is fine for edits. No mandatory cloud/server account.
- Full initial snapshot on connection + ordered changes/events with increasing revisions + resync after reconnect. Avoid poll-only designs, lost event races, or browser-only state as the authority.
- Capture request-time `context_with_system` without changing returned messages in read-only mode. Review `message_update`, `message_end`, `tool_execution_*`, session events, compaction events, and `agent_settled` for their real contracts; do not assume every live event is persisted.
- Pi may have sessions in memory when `--no-session` is used. Support that mode as live-only and clearly label lack of disk persistence.
- Encapsulate Pi-specific event code, an immutable snapshot/provenance model, edit coordinator, HTTP transport, and browser UI in separate modules.
- Secure the local editor: bind ONLY to loopback by default, no permissive CORS, validate Host/Origin and mutating Content-Type, use an unguessable instance token for API and event access, reject cross-origin writes; never inject untrusted model text as HTML. Document security implications.
- Prefer minimal runtime dependencies. TypeScript, Node and Pi SDK types; test with a fake/no-cost provider or mocked events. Avoid committing user session data, secrets, credentials, or raw captured transcripts.

## Acceptance criteria

1. `npm install` and documented dev/test commands work in a fresh Codex Linux environment without a proprietary provider API key.
2. Launch Pi with the extension and open a 127.0.0.1 URL; original session messages and system/tool transcript become inspectable.
3. A mock conversation with multiple tools, results, thinking if present, metadata, a new branch and compaction can be viewed **without dropped JSON properties**.
4. As the agent streams, the browser updates automatically and partial records cannot be mistaken for committed records.
5. Demonstrate changing a prior supported message through the browser; the very next model request reflects the approved change; original history remains inspectable; evidence verifies whether and when the edit is durably stored.
6. Reload the browser, switch branches/sessions, and restart Pi; demonstrate correct reconstruction and appropriate read/write limitations.
7. Verify unauthorized cross-origin requests fail.
8. Tests prove actual Pi-facing behavior, not only UI state or mocked browser calls.
9. README has reproducible setup, use, architecture, test commands, and an honest limitations table.
10. Run lint/typecheck/tests, fix failures, and report exact results.

**Priority:** functional inspection > provenance/fidelity > safe editing > nice styling. A partial implementation that reports its limitations truthfully is preferable to a polished mock-up.

## How to work

Implement in small, reviewable commits. Do not spend the whole task writing an architecture essay or exhaustively reading Pi source. Start from the linked docs and type declarations. Inspect source only to resolve a specific API question, especially persistence and edits. Add fixtures and tests early. If blocked, leave the repo in a runnable, independently useful state, explain the blocker precisely, and propose a tested next step. Work on this repo; upstream Pi remains a dependency, not a fork.
