# Field Notes: The layer above the harness: what a team repo can put on your machine

**Series framing.** This is a Field Notes piece, not a numbered teardown.
Every teardown in this series dissects a system that runs an agent: a loop,
tools, approvals, a session lifecycle. Tencent's TeamAI CLI runs no agent
at all. It never executes a turn and calls no model for its core work; it
is a *team layer* — one Git repository holding a team's shared skills,
rules, instructions, subagent definitions, environment variables, hooks,
MCP servers, and model settings, plus a CLI (`teamai`) that renders those
canonical assets into the on-disk formats of the agent tools a team's
members already use, and keeps every machine reconciled. There is no turn
to trace because the loops belong to other harnesses — several of which
this series has already taken apart. What TeamAI offers instead is a
complete, source-readable answer to a question no single-harness teardown
can ask: **when the unit of configuration is a team rather than a person,
where does the trust boundary actually sit?** That is a Field Notes
question, and it is the one this piece answers.

> **Pin:** `Tencent/teamai-cli` @ `208b97bb6218573b63d2445ac74f3a09999dd30d`
> (2026-10-18), MIT. npm package `teamai-cli` v0.22.0, bin `teamai`.
> Cloned and read, never executed. The codebase at the pin is ~150k lines
> of production TypeScript against ~221k lines of tests; breadth claims
> in coverage of the project (including "20+ tools") are checked against
> the source wherever they appear below.

## One canonical form, rendered in both directions

Everything shared lives in the team repo in one canonical form:
`teamai.yaml` at the root, skills as whole directories under `skills/`,
rules as Markdown, subagents as `agents/<name>.yaml`, hooks and MCP
servers as YAML lists, env and secret declarations under `env/`, plus a
knowledge base of docs and learnings.[^1] The canonical forms are
deliberately tool-neutral. A subagent is an `AgentSpec` — name,
description, instructions, optional model and tools, with per-tool
differences preserved under `tool_extras` — rendered per tool as Markdown
frontmatter for most, TOML for the Codex family, `.agent.md` for Copilot,
JSON for Kiro.[^2] One MCP schema renders into **eight** distinct config
shapes, including Codex TOML tables and OpenCode's `type: local/remote`
form.[^3] Rules have dedicated format modules for Cursor, CodeBuddy,
Qoder, Trae, Kiro, Copilot, and others, and can be inlined as a context
block for tools with no rules surface at all.[^4] Hooks are declared once
using Claude's event names as a lingua franca.

The rendering runs in both directions, and that is what makes the write
path possible: `push` parses a member's native tool files *back* into
canonical form — agents, for example, are read from each tool's own
format and merged into a single `AgentSpec` — rather than asking members
to edit YAML by hand. Scoping is equally systematic: an asset under a
`<namespace>/` directory is delivered only to members whose role or
project declares it, namespace entries override same-named root entries,
and the same key in two active namespaces is an error, not a
coin flip.[^5]

## The distribution loop: review on the way in, a hook on the way out

`teamai push` turns local edits into a reviewed branch and pull request.
Handlers stage the reverse-rendered changes into the team-repo clone;
the branch is `teamai/push/<username>/<timestamp>`, the commit path
skips local Git hooks, and PR creation goes through a provider layer —
the `gh`, `gf`, and `cnb` CLIs, direct REST for GitLab and GitCode, and
a generic Git provider that cannot create PRs at all.[^6][^7]
Re-running against an existing PR's branch rebuilds it and force-pushes
with `--force-with-lease`; if anything fails after files were copied,
the clone is rolled back, so a failed push leaves no half-staged state.
Nothing reaches teammates from `push` until a reviewer merges.

`teamai pull` is the delivery engine (~3,800 lines in `src/pull.ts`):
refresh the clone, then deliver per scope — user scope and each project
scope — with an unchanged-revision fast path. A **delivery ledger**
records what TeamAI wrote and its hash; a delivered copy the member has
since edited is *kept and named*, never silently overwritten — the
member shares the edit with `push`, or deletes the copy to take the team
version. Hooks and MCP state are reconciled on every pull, outside the
fast path, so a broken local state self-heals even when nothing else
changed.[^8]

Delivery does not depend on members remembering any of this. On `init`,
TeamAI installs its own built-in hooks into each tool, all pointing at
one dispatcher; on SessionStart the dispatcher runs `pull({ silent:
true })` in the background with a 120-second budget, so a session that
starts after a merge simply starts with the new configuration. The same
pass injects context the files cannot carry — team rules for tools
without a rules surface, the *names* (never values) of team env variables
and declared secrets in scope, recently-merged-MR hints. Git itself is
hooked as well: post-checkout and post-merge hooks trigger delivery for
a new worktree or a `git pull`.[^9] The hook is the distribution channel.

## "20+ tools", counted three ways

Three counts coexist in the source, and they are not interchangeable:

- **20** — the product rows in the project's own README support matrix.
- **24** — the keys in the default `toolPaths` map, reached by counting
  internal and regional variants (codex-internal, claude-internal,
  tclaude, tcodex) as separate targets.[^10]
- **38** — the entries in `KNOWN_AGENTS`, a registry of skills
  directories that includes tools TeamAI does not otherwise manage.[^11]

Depth varies as much as the count. **Codex** is a full target — skills,
agents, hooks, MCP — and TeamAI goes beyond file writing: it drives
`codex app-server` over JSON-RPC to record the trust hashes of the hooks
it installed (and to mark a project trusted, unless the member has
explicitly marked it untrusted), so the hooks it writes are hooks Codex
will actually run.[^12] **OpenCode** is a full target via its own
mechanisms: rules activated through a glob in `opencode.json`, and —
since OpenCode has no hooks in its config model — an auto-loaded plugin
file that bridges OpenCode events back to the dispatcher.[^13]
**OpenClaw** is partial: skills, a marked instructions block in the
workspace `AGENTS.md` (user scope only), and a managed hook package; the
default map has no agents or MCP paths for it, and the README matrix
agrees.[^14] **Aider and Gemini CLI are not managed targets at this
pin**: each appears exactly once in production source, as an entry in
the `KNOWN_AGENTS` skills-directory registry, and nowhere else — not in
`toolPaths`, not in the agent-format tool list, not in the matrix.[^11]
The accurate statement is the matrix itself: twenty products, uneven
depth, a few tools receiving every resource type and some registry
entries amounting to a skills path only.

## The friction loop: a hard session is the one worth writing up

The improvement loop is a nudge, not an upload. At Stop time, TeamAI
scores the session for **friction** from four signals: the user
interrupted the agent, the user rejected a tool call, tools genuinely
failed and had to be retried, and a *correction* — a new prompt within
60 seconds of a Stop matching a correction keyword list. Interrupts,
rejections, and corrections are worth 20 points each; tool errors score
on a tier. A hint fires only when the score reaches 20 **and** the
session made at least 15 tool calls — a hard gate, so one rejected
command in a trivial session stays quiet — and at most once per
session.[^15]

The hint names the concrete signals ("you interrupted the AI twice, you
rejected 1 tool call…") and suggests `/teamai share what this session
taught me`. That loads a bundled skill under which the *agent* writes
the learning document — required frontmatter, Context / Solution /
Lessons / Related Skills sections — and `teamai contribute` publishes
it: queued durably on the local machine, indexed for recall immediately,
then committed directly to the **`teamai-learnings` orphan branch**. No
pull request. Learnings therefore trade the review gate for immediacy;
the mitigations are that a learning lands as searchable reference
material — it is not configuration and does not execute — and that
recall carries a voting system, with reads and adoptions credited as
upvotes and low-confidence learnings prunable by maintenance.[^16]
Recall itself is a local lexical index over learnings and codebase
knowledge — tokenized, IDF-weighted, with title and tag boosts and a
vote bonus. No embeddings, no external service, and it is **off by
default** until the team enables it or a member opts in.[^17]

## What leaves the machine

Apart from Git traffic to the team's own repo, the outbound surface is
short, and its defaults deserve to be read precisely:

- **Usage reporting — on by default.** A non-inline pull folds the local
  event log into `stats/<username>.yaml` on the `teamai-reports` orphan
  branch: skill invocation counts, intervention counts, prompt counts,
  token totals, daily session buckets, recall votes. The `UserStats`
  schema has no field for prompt text — the design is counts-only — and
  either the team or the member can switch it off.[^18]
- **Self-update — on by default.** A Stop hook checks the npm registry
  (12-hour cache); the effective policy defaults to `auto`, under which
  a newer version is installed globally **without prompting**. A team's
  `autoUpdate: false` and a member's local policy can override — but
  out of the box, the CLI replaces itself from the registry in the
  background.[^19]
- **Webhooks — off by default.** If the team enables them, session and
  sync events are POSTed to team-configured endpoints, optionally
  HMAC-SHA256-signed.[^20]

There is no third-party telemetry endpoint in the code: no analytics or
crash-reporting service is called. The destinations are the team's own
repo, the configured webhooks, and the npm registry. The richer local
log — tool names, timestamps, token totals, and a best-effort-redacted,
truncated first prompt and last assistant output — stays on the machine;
full transcripts move only through an explicit `teamai session push`
command.[^21]

## The trust composition: merged is applied

TeamAI's trust model is, in essence, *the team repo is trusted code*.
Merging a PR is the act of trust; after that, delivery is automatic and
several delivered things execute. A merged change can, at teammates'
next session start:

1. **Run shell commands as hooks.** Team hooks carry a free-form shell
   `command` per lifecycle event. `sharing.hooks.autoApply` defaults to
   **true**: pull installs them without asking, printing the commands as
   they are applied — a transparency log, not a confirmation. Restricting
   commands to a team-scripts directory (`requireTeamScripts`) defaults
   to **false**; a member-side kill switch (`TEAMAI_HOOKS_DISABLED`)
   exists, as does a root hooks file that can re-time even TeamAI's own
   built-in hooks.[^22]
2. **Run team Node scripts around sync.** `scripts.prePull` and
   `scripts.postPull` run inside an ordinary pull — including the silent
   SessionStart one — with the member's environment; `postDeploy` runs
   at checkout preparation. The one hard check is containment: the path,
   symlinks included, must stay inside the team repo. There is no
   signature, sandbox, or per-script consent, and the 90-second postPull
   "budget" bounds only how long pull *waits* — on expiry the script is
   detached and left running.[^23]
3. **Install MCP servers that tools will launch.** A stdio server is a
   `command` plus `args` — an executable. `sharing.mcp.autoApply`
   defaults to **true**, and the `allowedCommands` / `allowedHosts`
   allowlists default to **empty, which the code treats as no
   restriction**.[^24]
4. **Set environment variables in every shell**, via a per-scope
   `env.sh` the member's profile sources — data, not code, but data
   that steers every tool the member runs.
5. **Steer every session.** Rules, instructions, skills, and agent
   definitions are prompt-level influence by design; a skill is a whole
   directory, scripts included, copied into each tool's skills folder.
6. **Define workflows** with shell `command` steps — an execution
   surface of a lower order, since a workflow runs only when a member
   (or an agent following a skill) starts one.

The mitigations are real but differently shaped: the PR gate on the repo
itself, the transparency print, member-side overrides for most features,
and `teamai doctor` diagnostics. What the code does not offer is a
per-change consent step at delivery time. By default, merged is applied.

## Secrets: a careful design beside a plaintext channel

Two mechanisms sit side by side, and which one a team reaches for is a
matter of author discipline.

**`env/env.yaml` is plaintext in the shared repo, by design.** Its
schema is `{key, value}` with `value` a plain string; the project's own
design table lists the file as "values in plaintext," and values are
masked only in CLI display. Resolved values are delivered to every
member in scope and written as `export` lines into their `env.sh`. A
shared token pasted into this file is, in effect, published to the team,
full Git history included.[^25]

**The secrets feature (`env/secrets.yaml`) is the careful path.**
Declarations carry only a key, a description, and a "where to get one"
URL; a declaration that includes a `value:` — or any unknown key — is
rejected, so a value cannot ride along accidentally. Each member
supplies their own value at a prompt or from another env var, never as a
command-line argument, so it stays out of shell history. Values live in
a store keyed by the SHA-256 of the team repo's URL — a careful
canonicalization, so a copied or hostile repo cannot claim another
team's identity and receive its values — written atomically at mode
**0600**. That is **not encryption**, and the design doc says so itself:
anyone who can read the member's files can read the values.[^26]

Two qualifications complete the picture. When an MCP config references
`${GITHUB_TOKEN}`, pull writes the member's actual value, in plaintext,
into each tool's MCP config file — compensated carefully: the file is
tightened to 0600, and a resolved value is never written into a project
config that Git tracks or fails to ignore (pull withholds it and advises
untracking the file and rotating the token). And a resolved secret is
not bound to a host: whoever controls `mcp.yaml` chooses the command or
host that receives every member's token — a fact the project's own
design doc states plainly.[^27] The session and sharing paths, finally,
lean on a best-effort regex redactor whose own header calls it
"defense-in-depth, not a guarantee." The secrets feature is thoughtfully
built; the exposure is the older, simpler channel beside it, which
accepts any string as a value and cannot tell a shared endpoint URL from
a shared credential.

## What to take from it

A dotfiles repo with a sync script covers perhaps the first third of
this. What goes beyond that pattern is the engineering in the middle: a
canonical intermediate form with **bidirectional** rendering, so assets
are authored once and still round-trip through twenty tools' native
formats; **reconciliation rather than copying** — a delivery ledger that
preserves member edits, hooks and MCP state that self-heal on every
pull, namespace scoping with conflicts treated as errors; an
**orphan-branch data plane** that separates configuration under review
from knowledge and counts under trust; and a friction loop that treats
*a session the human had to fight* as the session worth writing up, and
says so to the agent at exactly the moment it can act on it.

The trust reading follows from the same code. None of the defaults
above is hidden, and the design consistently chooses team velocity with
member-side escape hatches over delivery-time consent. That is a
defensible posture for a team that reviews its configuration repo the
way it reviews production code — because that is what a merged change
to it becomes: code and configuration, on every laptop, at the next
session start.

---

## Endnotes

*Read at a pinned commit: Tencent/teamai-cli @ 208b97b (MIT), cloned and
read, never executed. Independent personal project. Views are my own.*

[^1]: The team-repo layout — canonical on-disk forms for every asset
    type:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/docs/product-overview.md
[^2]: `AgentSpec` — the canonical agent form, and `renderForTool`'s
    per-tool renderers (Markdown frontmatter, Codex TOML, Copilot
    `.agent.md`, Kiro JSON):
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/resources/agent-format.ts#L75
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/resources/agent-format.ts#L1024
[^3]: MCP rendering — the eight target formats and the Codex TOML
    renderer:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/resources/mcp-format.ts#L18
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/resources/mcp-format.ts#L324
[^4]: Rules — canonical Markdown, the per-tool format modules in
    `src/resources/` (`cursor-mdc.ts`, `kiro-steering.ts`,
    `copilot-instructions.ts`, …), and inlining as a context block for
    tools with no rules surface:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/resources/rules.ts#L1569
[^5]: Namespaces — entry resolution, override, and same-key conflict
    handling:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/namespaced-entries.ts
[^6]: `push` — PR creation with fallback, and the core staging flow
    with rollback on failure:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/push.ts#L344
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/push.ts#L1034
[^7]: Branch and commit mechanics — `pushRepoBranch` (with
    `--force-with-lease` on the reuse path), branch naming, and commits
    that skip local Git hooks:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/utils/git.ts#L750
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/utils/git.ts#L221
[^8]: `pull` — per-scope delivery; hooks and MCP reconciled outside the
    fast path; the delivery ledger and its edited-copy rule:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/pull.ts#L1242
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/resources/delivered-copies.ts#L557
[^9]: Built-in hooks and the SessionStart silent pull — the canonical
    dispatch spec, and the background handler with its 120 s budget:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/builtin-hooks.ts#L317
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/hook-handlers.ts#L115
[^10]: The default `toolPaths` map — 24 keys including internal
    variants:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/types.ts#L461
[^11]: `KNOWN_AGENTS` — the 38-entry skills-directory registry; Aider
    and Gemini CLI appear only here, and the README matrix lists
    exactly 20 product rows:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/known-agents.ts#L82
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/README.md#L125
[^12]: Codex hook trust — driving `codex app-server` to record trust
    hashes, respecting a member's explicit untrusted marking:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/codex-trust.ts#L191
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/codex-trust.ts#L259
[^13]: OpenCode — the auto-loaded hooks plugin that shells back to
    `teamai hook-dispatch`:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/opencode-hooks.ts#L94
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/opencode-hooks.ts#L390
[^14]: OpenClaw — the managed hook package (`teamai-status-report`)
    that loads only once `openclaw.json` enables it; no agents or MCP
    paths in the default map:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/openclaw-hooks.ts#L63
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/types.ts#L671
[^15]: Friction — the `SessionFriction` signals, the 15-tool-call hard
    gate, the score threshold and weights, and the hint construction:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/types.ts#L2016
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/contribute-check.ts#L461
[^16]: The share skill and the contribute path — local queue first,
    then one commit direct to the `teamai-learnings` orphan branch:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/skill-data/share/SKILL.md
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/utils/learnings-publish.ts#L516
[^17]: Recall — the local lexical index (IDF-weighted, title ×3 / tag
    ×2 boosts, vote bonus); no embeddings, no external service:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/utils/search-index.ts#L833
[^18]: Usage reporting — `reportUsageToTeam` folds local events into
    counts-only `UserStats` on the `teamai-reports` branch; the
    default-on check:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/team-push.ts#L933
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/types.ts#L309
[^19]: Self-update — the registry check, `doUpdate` under the `auto`
    policy, and the policy resolution order (local > team > default
    `auto`):
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/update.ts#L657
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/update-policy.ts#L10
[^20]: Webhooks — event POSTs to team-configured endpoints, optional
    HMAC-SHA256 signature; disabled by default in the sharing schema:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/webhook.ts#L50
[^21]: The local event log — its path, the redacted 200-char prompt
    summary and 500-char last output, and the redactor's own
    "defense-in-depth, not a guarantee" header:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/dashboard-collector.ts#L1472
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/utils/redact.ts
[^22]: Team hooks — the schema's free-form `command`, the sharing
    defaults (`autoApply` true, `requireTeamScripts` false), and the
    layered resolution gate with its transparency print and kill
    switch:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/types.ts#L107
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/resources/hooks.ts#L161
[^23]: Team scripts — declaration in team config, the containment-only
    path check, and the 90-second wait budget that detaches rather
    than kills:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/team-scripts.ts#L198
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/post-pull.ts#L49
[^24]: MCP sharing — `autoApply` default true; allowlists default
    empty and only restrict when populated:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/types.ts#L155
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/mcp-reconcile.ts#L244
[^25]: Env — the plaintext `{key, value}` schema, display-only
    masking, delivery into `env.sh`; the internal design table's
    "values in plaintext":
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/resources/env.ts#L34
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/docs/designs/management-backend.md#L51
[^26]: Secrets — declarations with no value field (a `value:` key
    invalidates the declaration); the URL-hash-keyed 0600 store; the
    design doc's "not encryption":
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/resources/secrets.ts#L23
    and https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/src/secret-store.ts#L174
[^27]: Resolved secrets in MCP configs — plaintext delivery, 0600
    tightening, withholding from Git-tracked configs, and the
    not-bound-to-a-host statement:
    https://github.com/Tencent/teamai-cli/blob/208b97bb6218573b63d2445ac74f3a09999dd30d/docs/designs/team-secrets.md#L157
