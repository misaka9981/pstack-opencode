---
name: how
description: "Use for \"how does X work\", code walkthroughs before changing something, and placement / ownership / layering questions (\"where should this live\", \"which package owns this\", \"is this the right layer\"). Explains subsystem architecture, runtime flow, onboarding mental models. Use why for motivation."
disable-model-invocation: true
metadata:
  opencode/autoinvoke: false
slash: true
---

# How

Explore the codebase to answer "how does X work?" questions. Produce architectural explanations at the level of a senior engineer onboarding onto a subsystem, enough to build a working mental model, not so much that it reads like annotated source code.

Each spawn below names a role line in `~/.config/opencode/pstack-roles.md` and a default type. Use the type on that line, or the default when the file or the line is missing. If the `task` tool rejects a configured type as unresolvable, fall back to the closest valid type from its error message and say so.

## Step 1. Assess Complexity

If the scope is ambiguous, state your interpretation and explore. The user can redirect.

- **Simple** (a single module, a small utility, a narrow question such as "how does function X work"): no explorers. One explainer explores and explains in a single pass. Go to Step 2b.
- **Complex** (a subsystem spanning multiple files or services, a cross-cutting feature, a full architectural overview): spawn parallel explorers first, then hand off to the explainer. Go to Step 2a.

When in doubt, take the simple path.

## Step 2a. Explore (complex questions only)

Decompose the question into 2 to 4 exploration angles, each a distinct slice of the subsystem so explorers don't duplicate work. Example split for "how does the rate limiter work?":

- Explorer 1: data model and state management
- Explorer 2: request path and enforcement
- Explorer 3: configuration and metrics infrastructure

The right decomposition depends on the question. Narrow questions: 2 explorers is fine. Broad subsystems: up to 4.

Spawn all explorers in a single message:

- `subagent_type`: `explore` (read-only; or your configured `how-explorer` type, default `explore`)

Each explorer gets the prompt in `references/explorer-prompt.md` with its angle filled in. Each explorer should:

- Start broad: Glob for relevant directories, Grep for key types/interfaces/class names
- Follow the thread: from an entry point, trace the call chain (callers, callees, data flow, type definitions)
- Read the actual code, don't guess from file names
- Stop when it can describe the full path from input to output (or trigger to effect) without hand-waving any step
- Note things that are surprising, non-obvious, or that a newcomer would get wrong

Each explorer returns structured findings: components found, flow traced, files read, anything non-obvious. Overlap between explorers is fine; the explainer reconciles.

Then go to Step 3.

## Step 2b. Direct Explain (simple questions)

Spawn one subagent via the task tool that explores and explains in one pass:

- `subagent_type`: `general` (or your configured `how-explainer` type, default `general`)

Build its prompt from `references/explainer-prompt.md` without the explorer-findings section. Go to Step 4.

## Step 3. Synthesize (complex questions only)

Once all explorers have returned, spawn one subagent via the task tool to synthesize their findings into one explanation:

- `subagent_type`: `general` (or your configured `how-explainer` type, default `general`)

Build its prompt from `references/explainer-prompt.md` with every explorer's findings filled in. The explainer reconciles overlapping findings, resolves contradictions, and weaves the slices into a unified picture.

## Step 4. Present

Present the explainer's output to the user. Light edits for clarity or context from the conversation are fine. Do not substantially rewrite it.

## Output Format

The explanation uses the sections defined in `references/explainer-prompt.md`, dropping any that do not apply: Overview, Key Concepts, How It Works, Where Things Live, Gotchas.
