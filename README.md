# pi-live-context

**A live, bidirectional context viewer and editor for the [Pi coding agent](https://github.com/earendil-works/pi).**

Imagine being able to open a browser beside a running Pi session and inspect the *entire* conversation that Pi knows about—system instructions, messages, reasoning/thinking blocks when exposed, tool calls and results, compactions, and prompt/tool changes—as structured data, updating while the agent works. Then edit what the model will see on its **next** request, without destroying the original record.

That is what this project is intended to build.

> **Status:** Specification / starting repository. The extension and website have **not yet been implemented**. [AGENTS.md](AGENTS.md) contains the implementation brief for Codex and other coding agents.

## What should appear in the browser?

- **Original history:** Every session entry as originally recorded, including inactive branches, IDs, parent IDs, timestamps, tool input/output, metadata, and every available raw JSON field. No silent truncation or "helpful" rewriting.
- **Active context:** The branch and context Pi currently reconstructs, with compactions and edits applied. Show where its messages originated.
- **Last model-facing request:** A captured snapshot of the full Pi-visible transcript at the moment a request was made, including system instructions and tool definitions. Keep this distinct from a newly recomputed preview.
- **Live activity:** Streaming assistant output, tool execution and results, and session/branch changes, clearly indicating what is provisional versus finalized.
- **Editor:** Select a supported earlier message and replace or omit its contribution to *future* model requests. Show the untouched original alongside the effective edited version, with visible save/apply status.

Every view should have a usable **raw JSON** mode alongside any prettier presentation. Expensive or enormous data may be collapsed visually, but must remain accessible in full.

## What exactly is “context”?

Pi uses several related but different kinds of state. Mixing them together is how context viewers become misleading.

| Layer | What it is | Can we inspect it? |
| --- | --- | --- |
| **Session history** | An append-only tree of stored entries, usually on disk as JSON Lines | **Yes.** Pi exposes it through its session manager. |
| **Effective conversation** | The selected branch plus compactions, edits, prompt/tool state and other transformations | **Yes.** Pi can reconstruct it, and extensions can observe it when requests are prepared. |
| **Provider request** | The payload Pi's model adapter prepares for a particular model/provider | **Potentially.** Capture Pi-visible request data separately; do not claim it is raw network traffic. |
| **Provider KV cache** | Numerical inference-time attention cache maintained by the model server | **No.** This is not a JSON transcript, and Pi's extension API does not expose it. |

By default, Pi persists sessions under `~/.pi/agent/sessions/`, grouped by working directory. Entries are linked by `id` and `parentId`; the active model context follows *one branch* and may contain compaction summaries instead of older message text. Original entries remain in history. Pi can also run with in-memory-only sessions, in which case there is nothing durable to recover after exit.

Relevant upstream references: [session format](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/session-format.md), [message types](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/message-types.md), [extensions](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md), [SDK](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md).

## How editing should work

**The browser should edit future context, not rewrite history.**

Pi supports append-only `context_edit` entries that replace or omit the *content* of earlier eligible user, assistant, tool-result, and extension-message entries on the active branch. That gives us a promising native route to reversible, branch-sensitive edits while keeping original content intact.

There is an important API constraint: an ordinary extension sees a **read-only** session manager. Pi supports context-edit drafts at certain agent-run boundaries, but a browser click while Pi is idle cannot simply be assumed to commit a durable edit immediately. The implementation must demonstrate an actual supported write path, state clearly whether an edit is **pending**, **request-local**, **durably applied**, or **rejected**, and verify what the next real model request receives.

System instructions and tool definitions need distinct handling: the basic `context_edit` mechanism does **not** edit those entries. Never disguise a temporary request-time transformation as a persistent system-prompt edit. Already-submitted provider requests and remote caches cannot be retroactively changed.

The initial version should prioritize faithful inspection, then safe editing, then aesthetics.

## Intended architecture

- A **TypeScript Pi extension** loaded by Pi, not a fork of Pi itself.
- A local HTTP server bound to **127.0.0.1** with the viewer served from the same origin.
- Server-sent events or WebSockets for live updates, plus an authenticated local edit endpoint.
- Pi's session APIs and lifecycle events as the authority; no rewriting `.jsonl` files behind the running process.
- Explicit revision/session/branch checks to prevent applying stale browser edits.
- Strict loopback security, no telemetry or third-party resources. Raw conversation content may contain file contents, credentials, or secrets.

## Implement this with Codex Cloud

1. Create/select a Codex Cloud environment using **this** GitHub repository: `fettfrisur/pi-live-context`.
2. Ensure it can run Node.js, npm, and TypeScript; no commercial model API key should be necessary for the project's fixture and mock-provider tests.
3. Ask Codex to implement the project using [AGENTS.md](AGENTS.md). A suitable initial task is:

> Implement the Pi extension and loopback web viewer/editor specified in AGENTS.md and README.md. Start with an actual, running original-history and effective-context inspector with live updates; then implement safe context edits using supported Pi APIs. Add tests verifying Pi-facing behavior, document setup and limitations, and do not stop at a plan or nonfunctional mockup.

Codex should replace this status/setup guidance with actual installation, run, and test instructions once it has built the software.

## Design principles

**Fidelity over interpretation.** **Original history remains untouched.** **An edit is not considered applied until Pi actually uses it.** **The browser stays local.** **No fabricated access to provider internals.**

See [AGENTS.md](AGENTS.md) for the complete agent-facing requirements and acceptance criteria.
