---
description: Plan a new architecture from a natural-language demand -- writes plan.md/requirements.md/architecture.md plus Mermaid diagrams to a permanent .arch/plans/<hex-id>/ folder, which the Arc Code Architecture Explorer VS Code extension opens automatically
argument-hint: <description of the demand>
---

The user wants to plan a new piece of architecture from a plain-language
description, not implement anything yet. `$ARGUMENTS` is that description.

If `$ARGUMENTS` is empty, ask the user what they want to plan and stop --
do not guess a demand.

## 1. Ground the plan in what already exists

If `.arch/manifest.json` exists in this project, read it (same as the
`arc-code-flows` skill does) before inventing anything: check `index` and
`resources` for routes/classes/functions/modules the demand might touch or
duplicate, and note `frontend.analyzed` if relevant. A plan that proposes a
class or route that already exists under a different name is a bad plan.
If there's no `.arch/manifest.json`, proceed without it -- planning doesn't
require a prior `generate-diagrams` run.

## 2. Write the plan content yourself

Compose three markdown documents as plain strings (you author the
reasoning here -- nothing downstream does any thinking):

- **plan.md**: demand name, original description, objective, context,
  scope, functional requirements, non-functional requirements,
  constraints, assumptions, architectural decisions, risks,
  implementation strategy, next steps.
- **requirements.md**: functional requirements, non-functional
  requirements, business rules, dependencies, acceptance criteria.
- **architecture.md**: overview, components, responsibilities,
  communication between components, main flows, technologies, decisions.

Then describe the proposed architecture as data, in the exact same shape
`schema/proposal.schema.json` already defines (the `arc-code-flows` skill
uses this same shape for in-flight proposals) -- `architecture`,
`routes`, `sequence`, `classes`, `interfaces`, `dependencies`, `functions`,
optionally `domains` for a multi-resource demand. **Never hand-write
Mermaid.** Pick only the tags the demand's complexity actually calls for:
a single CRUD endpoint might need just `routes` + `sequence`; a
multi-service platform might want all of them, scoped under `domains`.

## 3. Persist it

Call the `arc-code` MCP server's `create_architecture_plan` tool if it's
connected (check `/mcp` or the project's `.mcp.json` the same way the skill
does for `render_proposal`) with:

```json
{
  "projectDir": "<absolute project root>",
  "title": "<short name of the demand>",
  "description": "<the original demand>",
  "planMarkdown": "<plan.md content>",
  "requirementsMarkdown": "<requirements.md content>",
  "architectureMarkdown": "<architecture.md content>",
  "...": "the diagram tags from step 2"
}
```

If the MCP server isn't connected, write that same object to a temp JSON
file and run:

```bash
npx arc-code-cli create-plan <that-file.json>
```

(run from the project root, so `projectDir` resolves correctly -- the CLI
takes it from `process.cwd()`, not from the JSON). Both paths write to a
fresh, permanent `.arch/plans/<8-hex-id>/` folder -- unlike
`render_proposal`'s disposable `.arch/.temp/<uuid>/`, this one is never
deleted; it's the persisted history of this planning session.

## 4. Report back

Read the path and file list either call reports and tell the user:

- The plan's id and folder path.
- One line per file written and what it contains.
- That if they have the Arc Code Architecture Explorer extension open on
  this workspace in VS Code, it will open the generated diagrams on its
  own within about a second -- nothing else to run.

This command only plans. It never writes or edits real source files, and
it never needs approval the way implementing a change does -- the plan
itself *is* the artifact being produced. Implementing what the plan
describes is a separate, later step (same "draw before you build"
discipline the `arc-code-flows` skill already uses for smaller, in-flight
proposals).
