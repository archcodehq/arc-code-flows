# How this plugin is built

This document explains the structure of `arc-code-flows`, the Claude Code
plugin companion to [Arc Code](../npx) — not what the plugin *does* for the
user (that's in `README.md` and `skills/arc-code-flows/SKILL.md`), but how
its pieces fit together and why they're shaped the way they are.

## Why a plugin, and why this shape

Arc Code (the `arc-code-cli` npm package) writes static architecture
diagrams to `.arch/` in a project. Those diagrams exist to save tokens: a
future question like "how does this feature's flow work end to end" or
"what does this one method call" should be answerable by reading one small
`.mmd` file instead of re-parsing the whole codebase — regardless of what
the project actually is or what its routes/classes happen to be named. That
only pays off if Claude actually *knows* to look there — which it won't, by default, in an
arbitrary session.

A Claude Code plugin is the mechanism for teaching that behavior once,
reusably, to every session opened against a project that has `.arch/`,
instead of re-explaining it in every conversation. Two different triggers
are involved, so the plugin has two matching components:

1. **"This project has `.arch/`"** is a fact about the *environment*, not
   something the user types — it needs to be noticed automatically. That's
   a **hook**.
2. **"Here's exactly how to use the index, and how to stage a proposed
   change"** is detailed, multi-step procedural knowledge that only needs
   to be fully loaded when actually relevant. That's a **skill**.

The hook's job is narrow: notice `.arch/` and say so, briefly. The skill
carries the full procedure. This mirrors how the `ponytail` plugin (used as
the structural reference while building this one — see "How the structure
was verified" below) splits an always-on nudge from a more detailed,
separately-invokable skill, rather than cramming everything into one hook's
stdout or relying on the skill being found by description-matching alone.

## Directory layout

```
claude-plugin/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest (name, version, hooks path, mcpServers)
│   └── marketplace.json    # lets this folder be installed as a local marketplace
├── plugin.yaml             # declares what the plugin provides (hooks, skills, commands, mcp servers)
├── hooks/
│   ├── hooks.json          # maps the SessionStart event to detect-arch.js
│   └── detect-arch.js      # the hook's actual implementation
├── skills/
│   └── arc-code-flows/
│       └── SKILL.md        # the detailed, invokable skill
├── commands/
│   └── arch-plan.md        # the /arch-plan slash command (prompt, not code)
└── README.md                # user-facing: what it does, how to install it
```

Two manifests exist at the root for two different consumers:
`.claude-plugin/plugin.json` is what Claude Code's plugin loader reads
(name, version, where to find the hooks config, which MCP servers to
auto-register); `plugin.yaml` is a higher-level declaration of capabilities
(`provides_hooks`, `provides_skills`, `provides_commands`,
`provides_mcp_servers`) used for discovery/listing. Both need to stay in
sync by hand — nothing generates one from the other.

## The command: `commands/arch-plan.md`

`/arch-plan <description>` is a plain markdown prompt file — auto-discovered
by directory convention (`commands/<name>.md` → `/<plugin>:<name>`, same as
skills are discovered by directory, no manifest entry needed for discovery
itself), with `description`/`argument-hint` frontmatter and the user's typed
text after the command name available as `$ARGUMENTS` in the body. There is
no code here: the body is instructions telling Claude to (1) ground the
plan in `.arch/manifest.json` if one exists, so it doesn't propose
something that's already there under a different name, (2) author
`plan.md`/`requirements.md`/`architecture.md` as plain markdown strings
itself — no downstream piece does any reasoning, same division of labor as
the skill's "draw before you build," (3) describe the diagrams as data in
`schema/proposal.schema.json`'s existing shape, never hand-written Mermaid,
and (4) call the new `create_architecture_plan` MCP tool (or its CLI
fallback) to persist all of it.

This differs from the skill's existing "draw before you build" flow in one
way that matters: a plan is a **dedicated, explicit first step** ("plan an
architecture for X"), not something triggered reactively while already
mid-implementation, and what it produces is **persisted** (`.arch/plans/<hex-id>/`,
never cleaned up automatically) rather than disposable review scratch
(`.arch/.temp/<uuid>/`, deleted after approval). The skill is unchanged —
both coexist, used for different moments in a feature's life.

## The hook: `hooks/detect-arch.js`

Wired to the `SessionStart` event via `hooks/hooks.json`, with
`matcher: "startup|resume|clear|compact"` — it fires every time a session
starts or resumes, not just on a cold start, since `.arch/` could be
generated or regenerated mid-project-lifetime.

What it does, in order:
1. Check whether `<cwd>/.arch/manifest.json` exists. If not, print nothing
   and exit — a silent no-op is the right behavior for every project that
   has nothing to do with Arc Code, which is most of them.
2. If it exists, parse it just enough to report how many files were
   analyzed (best-effort — a malformed manifest still gets the reminder,
   just without that detail).
3. Print a short, fixed block of English text to stdout: treat `.arch/`
   as the source of truth for the project's current architecture and
   flows (over Claude's own inference), use the index before re-reading
   source, fall back to source if something isn't indexed, and draw
   proposed changes into `.arch/.temp/`, list them, and get explicit
   approval before touching real files. It ends by pointing at the
   `arc-code-flows` skill for the full procedure — the hook deliberately
   doesn't try to teach the whole thing itself.

**Why it only checks `cwd`, not parent directories:** Claude Code's session
working directory is normally the project root already. Walking up the
tree to find an `.arch/` could just as easily find an unrelated ancestor
project's `.arch/` and inject misleading context about the wrong codebase.
A missed detection (user working from a subdirectory) is a safer failure
mode than a wrong one.

**Why raw stdout, not a JSON envelope:** verified against a real installed
plugin's hook (see below) — on native Claude Code, `SessionStart` accepts
plain text on stdout directly as additional context. Only other events
(`SubagentStart`, and other host platforms like Copilot/Codex/Qoder) need
the `hookSpecificOutput.additionalContext` JSON wrapper. Since this plugin
only uses `SessionStart`, the hook writes raw text and nothing else.

**Why it never calls `arc-code` itself:** generating or regenerating
`.arch/` is an action with a side effect (it writes files). A hook running
silently at session start is not the place to decide that's warranted —
that decision belongs to an explicit command Claude runs with the user's
normal approval, same as any other write. The hook only *reads*
`manifest.json`; it never shells out to `npx arc-code`.

## The skill: `skills/arc-code-flows/SKILL.md`

This is where the actual procedure lives — starting from the principle
that `.arch/` (root files plus `domains/`) is the source of truth for the
project's current flows and architecture, over Claude's own inference from
skimming code. Concretely: reading `manifest.json`'s `index` to jump
straight to a mini-flow file, falling back to the resource-scoped
`domains/<name>/` diagrams for whole-feature questions, and the
"draw before you build" workflow: before implementing anything that would
add, remove, or move a route, function, class, or dependency, describe the
proposed result as JSON matching `arc-code-cli`'s
`schema/proposal.schema.json` and render it into a fresh `.arch/.temp/<uuid>/`
folder (a new UUID per request, not a fixed shared name — so one proposal
never collides with or overwrites another still pending review) — via the `arc-code` MCP
server's `render_proposal` tool if a project has one wired into its
`.mcp.json` (structured call, no file, no shell command), or
`npx arc-code-cli render-proposal <file.json>` otherwise. The skill never
has Claude hand-write the `.mmd` files — both paths call into the exact
same renderer Arc Code itself uses (`render_proposal` *is*
`render-proposal`'s logic, exposed as an MCP tool instead of a CLI
argument), so the output is guaranteed valid Mermaid even though Mermaid's
escaping rules differ by diagram type (see "Why it never calls `arc-code`
itself" above for the parallel reasoning on not shelling out silently —
rendering a proposal *is* the explicit, user-requested action that
justifies calling either one here). After rendering, the skill lists what each file shows
and gets the user's explicit approval before any source file is touched —
not just for large architectural changes; the trigger is "does this change
what `.arch/` would describe," which covers most real feature work. This
is the half of Arc Code's
originally-intended workflow (request → AI proposal → diagram → human
approval → implementation → verification) that runs inside Claude Code
itself; regenerating `.arch/` afterward is the verification half, and still
happens as an explicit command, not automatically.

The skill also covers `arc-code-cli diff-diagrams` (compares two git refs,
or a ref against the working tree, by running the same analyzer on both
and classifying every route/class/function/module as
added/modified/removed/unchanged) -- read-only, so unlike
`generate-diagrams` it needs no approval step before running. That
classification logic and the color-coded rendering live in `arc-code-cli`
itself (`src/diff/`), same as `render_proposal` -- this plugin's skill is
again just the consumer, deciding when to call it.

Same split for frontend: `arc-code-cli` detects a project's frontend
framework and, for React/Next.js, structurally analyzes it into
`components.mmd`/`pages.mmd` (`src/analyzer/components.ts`,
`src/analyzer/pages.ts`) -- entirely inside `arc-code-cli`, same renderer
pipeline as the backend diagrams. This plugin's skill only teaches Claude
*when* those two files exist (check `manifest.json`'s `frontend` field)
and how to read them; it carries no component/JSX-detection logic of its
own, same as it carries no route-detection logic for the backend side.

It's a full skill rather than folded entirely into the hook's output for
two reasons: it's long enough that injecting all of it into every single
session start would be wasteful when most turns in a session won't touch
architecture questions at all; and a skill can be invoked again later in a
session (by name, or by its own description matching) if context has
rolled off and Claude needs the procedure again — a one-shot hook message
can't be re-delivered that way.

## The MCP server: lives in `arc-code-cli`, not in this plugin

`render_proposal` and `create_architecture_plan` (the MCP tools the skill
and `/arch-plan` call) are both `npx arc-code-cli mcp` — code in the
`arc-code` repo (`src/mcp/`), not in this plugin. That placement is
deliberate: each tool is a thin wrapper around a plain function the
matching CLI command also calls (`renderProposal()` / `createArchitecturePlan()`),
so the actual logic (and its test coverage) has exactly one home regardless
of which interface reaches it. This plugin is just the *consumer* of those
tools, same as it's the consumer of `.arch/`'s diagrams — it doesn't
reimplement or re-validate what they do.

**This now auto-registers** — a reversal of an earlier decision. Until this
plugin shipped `/arch-plan`, wiring the server into a project was a
separate, per-project step (hand-editing that project's `.mcp.json`),
deliberately *not* something this plugin's install did automatically: "a
plugin enabling a new MCP server as a side effect of being installed would
be a much bigger, more surprising action than anything else here." That
reasoning held as long as every MCP call was optional polish on top of a
working CLI fallback inside a reactive skill. `/arch-plan` changes the
calculus: it's a command the user explicitly typed, expecting one result
with no extra setup step (the whole point of "describe a demand, get a
plan" is that there isn't a second thing to configure first) — so the
server is now registered via `.claude-plugin/plugin.json`'s `mcpServers`
field, pointing at `${CLAUDE_PLUGIN_ROOT}/../npx/dist/cli/index.js mcp`.
Both tools still fall back to their CLI equivalents if the MCP connection
isn't up for any reason (a fresh install before the first session restart,
a platform where plugin-registered servers aren't supported yet), so
nothing *requires* this registration to work — it only removes a manual
step that `/arch-plan` shouldn't need. The hook's own "never a silent side
effect" principle is unchanged: the hook still only reads `manifest.json`
and never shells out to anything.

## Distribution: local marketplace, not a published one

`.claude-plugin/marketplace.json` declares this single folder as its own
marketplace (`"source": "./"`), which lets it be installed during
development without publishing anywhere:

```bash
claude plugin marketplace add /home/alexandre/Documents/projects/arch/claude-plugin
claude plugin install arc-code-flows@arc-code-flows
```

This is deliberate for now — the plugin is young and tied to one person's
workflow with Arc Code. If it's ever meant for other people to install, the
marketplace would need to move to its own public git repo (the same pattern
`ponytail` and `claude-mem` use), not change in kind, just in where
`marketplace.json`'s `source` points.

## How the structure was verified

Every file shape here (what `plugin.json` needs, how `hooks.json` maps
events to commands, what a hook's stdout contract actually is on native
Claude Code) was confirmed by reading real, already-installed plugins on
this machine — `~/.claude/plugins/cache/ponytail/ponytail/` in particular —
rather than assumed from memory. `claude plugin validate` was also run
against both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`
and passed before this was considered done. If the plugin schema changes in
a future Claude Code version, that's the same place to go re-check: a real
installed plugin's files, not documentation that might lag the current
format.
