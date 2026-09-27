---
name: setup-pstack
description: Configure which subagent types pstack uses per role, and optionally which model each role's agents run on. Detects the agent types available in this session and writes ~/.config/opencode/pstack-roles.md, an override layer the skills read. Use for /setup-pstack, "configure pstack agents", "pstack budget", or changing pstack's subagent choices.
disable-model-invocation: true
metadata:
  opencode/autoinvoke: false
slash: true
---

# Setup pstack

Write `~/.config/opencode/pstack-roles.md`, a plain file the pstack skills read on demand, that sets pstack's subagent type per role. The skills read it and fall back to their inline defaults when a line is absent, so this is an override layer, not a requirement. Optionally, per-agent model overrides go to `~/.config/opencode/opencode.json` (step 6).

opencode runs every subagent on the session model unless its agent file or its config entry (`agent.<name>.model` in `opencode.json`) sets a model, so by default diversity comes from the subagent type and the prompt, not the model. What this file configures is the **subagent type** each role fans out to: a reviewer type reads a diff differently than an architect type or a general delegate.

## Steps

### 1. Detect available subagent types

Enumerate the `subagent_type` values you can pass to a `task` call in this session. The dependable sources, in order:

1. opencode built-ins always present: `build`, `plan`, `general`, `explore`. pstack also ships its own agents in its repo's `agents/` dir: `poteto-agent`, `comment-sicko`, `code-reviewer`, `code-architect`; these must be installed for the role defaults in the skills to resolve.
2. Types contributed by enabled plugins and declared in this session. pstack's own agents (`poteto-agent`, `comment-sicko`, `code-reviewer`, `code-architect`) are not a plugin contribution: they arrive via the installer into `~/.config/opencode/agents/` (or a plugin install, if used).
3. Agent definitions opencode loads outside plugins (for example `~/.config/opencode/agents/`).

Never write a type you have not confirmed exists. A role line pointing at a type the `task` tool rejects breaks every delegation that reads it.

### 2. Load current state

The default role-to-type mapping is the shape shown in step 5 below. If `~/.config/opencode/pstack-roles.md` already exists, read it and treat its `# budget` line and its role values as the current choices. Otherwise start from those defaults. A line whose role is not in step 5, such as the retired `how-critics` role, is stale: drop it.

### 3. Budget, map, and confirm

**(a) Ask for a budget.** Prefer the `question` tool over free text. Offer these four options with these exact labels, and name the current budget when the file records one.

- `unlimited — keep max`
- `large — xhigh reasoning`
- `medium — high reasoning`
- `small — medium reasoning`

**(b) Apply it.** In this port the budget shapes the optional per-role **models** (step 6), not the subagent types, because opencode picks each model's own reasoning effort when no variant is set. `unlimited` leaves model choices alone. `large`, `medium`, and `small` append the effort variant `#xhigh`, `#high`, or `#medium` to every real `provider/model` id you write in step 6, when that provider exposes such a variant; otherwise record the budget and say the model was left at its default effort. `inherit-parent` and `auto` entries never change. If step 6 is skipped (types only), record the budget in the file as intent but note it changes nothing until models are pinned.

**(c) Show the roles and confirm.** Show every role with its current type, marking any type not in the detected set as needing a choice. Also list each line step 2 dropped. Ask whether to accept as-is or change specific roles, offering the detected types as the options. Prefer the `question` tool over free text. For panel roles (arena runners, architect runners, interrogate reviewers) the value is a list, and one subagent runs per entry, so the list length sets the fan-out; entries may repeat a type when you want volume over diversity. `arena cross-judge pool` is also a list, and Arena picks one value from it. `swarm workers` is the default type for every worker unless a race or comparison assigns another per arm.

### 4. Validate

Every type written must be in the detected set. If a chosen type is not available, stop and ask again.

### 5. Write the file

Write `~/.config/opencode/pstack-roles.md`, overwriting the whole file so re-runs stay idempotent. Shape:

```markdown
# pstack per-role subagent choices (overrides skill defaults)

One line per role or role group, `role[, role...]: type[, type...]`. Delete a line to fall back to the skill default.
# budget: unlimited (max)

feature, refactoring: poteto-agent
bug-fix: poteto-agent
perf-issue: poteto-agent
hillclimb: poteto-agent
judgment and prose: general
hardest tasks: general
how-explorer: explore
how-explainer: general
why-investigators: general
why-synthesizer: general
reflect-tooling: code-reviewer
reflect-judgment: general
arena runners: poteto-agent, code-reviewer, code-architect, general
arena cross-judge pool: code-reviewer, code-architect, general
swarm workers: poteto-agent
architect runners: code-architect, poteto-agent, code-reviewer, general
interrogate reviewers: code-reviewer, code-architect, poteto-agent, general
```

### 6. Models (optional)

Ask once whether the user wants per-role models. Most users run every subagent on the session model; that stays the default. Do not push per-role models.

On yes, determine the models actually available: read the `provider` and `model` keys in `~/.config/opencode/opencode.json` and the project `opencode.json`, and offer the `provider/model` ids configured there plus the current session model. Never offer a model id you have not seen in those files or in the session. Apply the step-3 budget by appending its effort variant (`#xhigh`, `#high`, or `#medium`) when the provider exposes one.

For each role the user wants to change, map the role's agent type(s) from step 5 to the chosen model and write `agent.<agent-name>.model` as a "provider/model" (optionally `provider/model#variant`) string in `~/.config/opencode/opencode.json`. This covers opencode built-ins (`general`, `explore`, `build`, `plan`) and pstack's custom agents (`poteto-agent`, `comment-sicko`, `code-reviewer`, `code-architect`). Model overrides are per agent, not per role: setting a model on an agent affects every role that uses it, so say so when a role's type list shares an agent with other roles.

`opencode.json` is a merge target, not a rewrite. Preserve every existing key (`provider`, `permission`, `mcp`, ...), create the `agent` object if absent, and touch only the `agent.<name>.model` entries for the chosen roles. Re-runs must be idempotent: same choices, same file.

Precedence: an agent file with its own `model` frontmatter beats the config override. Deleting the `agent.<name>.model` entry restores session-model inheritance.

### 7. Confirm

Re-read the file you wrote and echo the final table to the user. If step 6 wrote model overrides, echo them (agent name -> provider/model) and the precedence rule (agent file `model` frontmatter beats config; deleting the entry restores session-model inheritance). State the fallback rule (absent line = skill default), the recorded budget, and that rerunning `/setup-pstack` is always safe.

### 8. Offer a verification skill (optional)

Check whether the project has a way to drive the real app for proof (a `verify-*` skill, or an existing harness). If not, offer once: "want a project-local verification skill, so agents can drive the app the way a user does and prove changes work? I can generate one with /create-verification-skill." On yes, invoke `/create-verification-skill` (resolves wherever pstack is installed — workspace, user, or plugin). On no, move on without pushing.
