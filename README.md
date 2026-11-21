# arc-code-flows

Claude Code plugin companion to [Arc Code](../npx) (the `arc-code-cli` npm package). It does not run Arc Code itself — it teaches Claude Code how to *consume* the `.arch/` diagrams Arc Code already generated, instead of re-reading source code to answer the same flow/function/class questions over and over. It's project-agnostic: it carries no knowledge of any particular project's domain, routes, or class names — only the generic shape of `.arch/manifest.json`.

## What it does

- **SessionStart hook** (`hooks/detect-arch.js`): if the current project has `.arch/manifest.json`, injects a short reminder into context automatically — no need to ask for it.
- **Skill** (`skills/arc-code-flows/SKILL.md`): the full detail — how to use `manifest.json`'s `index` to jump straight to the right mini-flow file, how to read `components.mmd`/`pages.mmd` for a React/Next.js project's component tree and UI routes (check `manifest.json`'s `frontend` field — Vue/Angular/Svelte are detected and reported but not yet analyzed), how to run `arc-code-cli diff-diagrams` to show what changed architecturally between two commits/branches/the working tree (color-coded added/modified/removed/unchanged, read-only), and the "draw before you build" workflow: before implementing anything that would add/remove/move a route, function, class, or dependency, describe it as JSON and render it into `.arch/.temp/`, list what each file shows, and get explicit approval before touching any real source file.
- **`/arch-plan <description>` command** (`commands/arch-plan.md`): plan a *new* architecture from a plain-language demand, as a dedicated first step rather than a side effect of implementing something. Writes `plan.md`/`requirements.md`/`architecture.md` plus the same Mermaid diagrams the skill renders, to a permanent `.arch/plans/<8-hex-id>/` folder — unlike the skill's disposable `.arch/.temp/<uuid>/`, a plan is never deleted automatically; it's a persisted record. If the [Arc Code Architecture Explorer](../arc-architecture-explorer) VS Code extension is open on the same workspace, it opens the generated diagrams on its own within about a second (it already watches `.arch/` in real time — no new channel between this plugin and VS Code, just a folder it was already watching).

Rendering a proposal (`render_proposal`) or persisting a plan (`create_architecture_plan`) uses the `arc-code` MCP server, which this plugin **registers automatically** on install (`.claude-plugin/plugin.json`'s `mcpServers` field) — no manual `.mcp.json` editing needed. Both tools fall back to a CLI equivalent (`npx arc-code-cli render-proposal` / `create-plan`) if the MCP connection isn't up for any reason. Neither this plugin, the skill, nor `/arch-plan` ever calls `arc-code-cli` as a silent side effect of a hook — a slash command run by the user is itself the explicit, user-visible step — and generating the real `.arch/` (`generate-diagrams`) stays a separate, explicit command either way.

## Local install (development)

This plugin isn't published to a marketplace — install it from this local path:

```bash
claude plugin marketplace add /home/alexandre/Documents/projects/arch/claude-plugin
claude plugin install arc-code-flows@arc-code-flows
```

After editing `hooks/`, `skills/`, `commands/`, or the `mcpServers` registration, start a new Claude Code session to pick up the change (plugins, and the MCP servers they register, load at session start). `arc-code-cli` must already be built (`cd ../npx && npm run build`) for the auto-registered `arc-code` MCP server to actually start — the registration points at `npx/dist/cli/index.js`, not the TypeScript source.

To remove:

```bash
claude plugin disable arc-code-flows
claude plugin marketplace remove arc-code-flows
```
