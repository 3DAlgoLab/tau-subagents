# ADR 0006 — Children inherit the parent's project-trust decision

## Status

Accepted (2026-11-03).

## Context

Tau 0.3.7 added project-input trust (upstream #541). A session resolves trust
for its cwd. A headless child resolves that decision on its own and lands on
`trusted=False`. Tau then builds the child's resource plan with
`project_resources_enabled=False`. The child loses the project's skills, the
project's AGENTS.md, and the project's extensions.

This extension built every child without `trust_override`, so each child
resolved trust for its own cwd. What that costs depends on whether a saved
decision covers that cwd:

- A normal (non-isolated) child runs in the parent's cwd. The parent's approval
  is persisted in `<tau home>/trust.json`, and a saved decision for an ancestor
  path covers the child, so the child resolves trusted on its own.
- An `isolation: worktree` child runs in `$TMPDIR/tau-agent-<id>-<hex>`, which no
  saved decision covers. There the child resolves untrusted and its prompt loses
  the project's skills and AGENTS.md.
- A non-isolated child also loses them when the parent's approval is not in the
  store: a session-scoped approval, or a decision that was later removed.

Measured on tau 0.4.9b3, in this repository, with one probe project skill
committed and `--no-extensions -e <checkout>` so only that build was loaded: the
pre-port build `f2a8b4b` showed an isolated child only the user skill, and the
ported build `a10d9f9` showed it both skills. For a non-isolated child both
builds showed both skills, because the stored approval covers the shared cwd.
Nothing failed loudly in any of these cases. The child's prompt was smaller.

Two facts hid this from us:

- `pyproject.toml` allowed `tau-ai>=0.3.0`, and `uv.lock` froze 0.3.5. CI ran
  `uv run pytest`, so CI proved only that 0.3.5 worked.
- The suite does assert skill names in child prompts, but every such test ran
  against a Tau without the gate.

Tau 0.4 changed resource plans as well. `resource_paths_with_cwd` now treats a
plan as destination-bound: it replaces the plan's `cwd` with the session's cwd.
`skills: true` pinned discovery by passing `CodingSessionConfig.resource_paths`,
so that pin no longer works.

Tau 0.4.9 also stopped the agent loop from starting a provider request after a
cancel. The old code spent one call that answered with an empty, cancelled
stream. One test counted those calls, so its expectation moved from 3 to 2.

## Decision

Forward the parent's verdict through the documented child-session seam: read
`context.project_trusted` when a child spawns, and pass `trust_override="approve"`
when it is true, `"decline"` when it is false. Never pass `"approve"`
unconditionally: that reopens the hole the host's gate exists to close. For a
child that shares the parent's cwd the override usually changes nothing, because
the store already answers for that path; it also makes the inheritance explicit
instead of a side effect of `trust.json` state.

Raise the dependency floor to `tau-ai>=0.4.8`. That release exposes
`context.project_trusted` and `CodingSessionConfig.allowed_tool_names`. Older
releases are not a working target, because 0.3.7 introduced the gate that
breaks children. Add a CI job that resolves the newest tau-ai without the
lockfile, so upstream drift fails the pull request that hits it.

Implement `skills: true` inside this extension. Load the parent's skills with
`load_skills`, render Tau's own `<available_skills>` block with
`format_skills_for_prompt`, and start the child with `skills_enabled=False`.
Project context files still resolve against the child's cwd.

Replace the post-load mutation of `session._harness.config.tools` with
`allowed_tool_names`. Tau's child-session note names it as the supported seam:
it applies to the composed tool list and survives a reload, where the mutation
does not.

Keep project extensions off for children. `project_extensions_enabled` stays at
its default. Trust says the repository's inputs may shape a child's prompt.
Running the repository's code inside a child is a separate decision.

## Consequences

Isolated children now see the project's skills and its AGENTS.md. Non-isolated
children saw them before whenever a stored decision covered the shared cwd, and
still do not see them when the parent's approval is session-scoped.

Child prompts change for `skills: true`. The skills block now sits in the tail
of the assembled prompt, not in Tau's native position, so such a child no
longer shares a native child's prompt-cache prefix.

Test doubles must grow with Tau's session protocol. `_FakeSession` in
`test_integration_tui.py` copies an unpackaged fake from Tau's own tests, so it
breaks whenever Tau's sidebar reads a new session property. Tau 0.4.9 required
`has_provider_context_usage` and `system_prompt_files`. Fake providers must also
accept `session_id`, which the provider protocol gained in 0.3.7.

The suite now pins the seam itself, not only the symptom:
`test_children_inherit_the_parent_project_trust_decision` asserts the trust
verdict on every child config, and asserts the decline case as well.

The extension still reads `<cwd>/.tau/agents/*.md` and
`<cwd>/.tau/subagents.json` without checking trust. Those files can set a
child's tools, model, and isolation. We record that as a known gap. A follow-up
should gate both reads on the parent's decision, or document them as
deliberate.
