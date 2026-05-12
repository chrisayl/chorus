---
description: Run a multi-LLM Chorus peer review on a task
argument-hint: [template] <work description>
---

Route this request to the Chorus MCP server. Use the `mcp__chorus__*` tools — do NOT try to do the review yourself.

**Arguments:** $ARGUMENTS

## Steps

### 1. Classify intent

Read $ARGUMENTS and decide which mode the user wants:

- **Review-only mode** — they want N AIs to look at *existing content* (a diff, a file, a paragraph, a proposal) and return independent verdicts. Trigger phrases: *"review the diff"*, *"review the staged changes"*, *"review my branch"*, *"review this file"*, *"second opinion on this draft"*, *"critique this proposal"*, *"is this implementation correct"*.
- **Doer mode** — they want Chorus to *produce* something (write code, draft an architecture, run TDD). Trigger phrases: *"implement"*, *"refactor"*, *"design a"*, *"propose"*, *"write tests for"*, *"red-green this"*.

If genuinely ambiguous, ask the user before continuing.

### 2. Resolve the template

- If $ARGUMENTS starts with a known template id (`code-review`, `review-only`, `architect-review`, `tri-review`, `red-green`, `bug-diagnose`), peel it off as the template.
- Otherwise:
  - **Review-only mode** → use `review-only` (parallel critique, all reviewers see the same artifact, no doer-as-intermediary).
  - **Doer mode** → call `mcp__chorus__list_templates`, show `name + description` for each, ask which to use. Don't guess — picking the wrong template wastes a multi-minute, multi-LLM run.

### 3. Prepare the artifact (review-only mode ONLY)

`review-only` REQUIRES an `artifact` string — the text being reviewed. Reviewers can't read your filesystem, so if you skip this step they get nothing to look at.

Capture the artifact before calling `create_chat`:

| User phrasing | Capture command (use the Bash tool) |
|---|---|
| "the staged diff" | `git diff --staged` |
| "the staged diff against main" / "vs main" | `git diff --staged main` |
| "my branch" / "this branch" / "branch vs main" | `git diff main...HEAD` |
| "this commit" / "commit X" | `git show <sha>` (ask if no sha given) |
| "this file" / specific path | `cat <path>` |
| "this folder" | enumerate via Glob, then `cat` each file in turn |

Run the relevant command, capture stdout. The daemon caps `artifact` at 1 MiB but reviewer ask-prompts cap at 256 KB, so prefer artifacts under ~200 KB. If the captured text is larger, truncate from the bottom and append a `[...truncated]` marker. If the captured text is empty, tell the user and stop.

### 4. Create the chat

- **Review-only mode**: `mcp__chorus__create_chat({ template: "review-only", work: "<user's framing of what to look for>", artifact: "<captured text>" })`
- **Doer mode / any other template**: `mcp__chorus__create_chat({ template, work: "<user's full arguments>" })`

Capture the returned `chatId`.

### 5. Wait for the result

Call `mcp__chorus__wait_for_chat({ chatId })`. Stream progress notifications back to the user as they arrive (template phases, reviewer verdicts).

### 6. Summarise

When the chat completes, report:
- The final verdict (approved / changes-requested / blocked)
- The 3–5 highest-priority findings, each with a one-line rationale and a `file:line` pointer if the reviewer gave one
- The Chorus cockpit URL for the run if available (typically `http://127.0.0.1:5050/runs/<chatId>`)

Do not paste the full transcript. The user can open the run page for that.

## Failure handling

- If `create_chat` returns `MCP server chorus not connected` or any tool throws a connection error, tell the user to run `chorus start` (or `chorus status` to check) and retry. Do not fall back to doing the review yourself.
- If the chat is `blocked` (waiting for human input), surface the question verbatim and stop — let the user respond via `mcp__chorus__resume_chat`.
- If the chat fails (`failed` status), report the error from `get_chat_status` and suggest the cockpit URL for the full log.

## Hard rules

- Never substitute your own review for a Chorus run when the user asked for `/chorus` — the whole point is cross-lineage second opinions.
- Never invent a template id. If $ARGUMENTS doesn't match a real template, ask.
- Never call `cancel_chat` unless the user explicitly asks to abort.
- Never skip the artifact-capture step in review-only mode. Reviewers can't read the filesystem; forgetting the artifact means they review nothing.
