# ADR 0006 — Children inherit the parent's project-trust decision

## Status

Accepted (2026-11-03).

## Context

Tau 0.3.7 added project-input trust (upstream #541). A session resolves trust
for its cwd. A headless child resolves that decision on its own and lands on
`trusted=False`. Tau then builds the child's resource plan with
`project_resources_enabled=False`. The child loses the project's skills, the
project's AGENTS.md, and the project's extensions.

This extension built every child without `trust_override`. So every subagent in
an approved repository lost those resources. Nothing failed loudly. The child's
prompt was simply smaller.

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

Read `context.project_trusted` when a child spawns. Pass it as
`trust_override="approve"` when it is true, and `"decline"` when it is false.
Never pass `"approve"` unconditionally: that reopens the hole the host's gate
exists to close.

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

Children in an approved repository see the project's skills and its AGENTS.md.
Children in a declined repository behave as before.

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
