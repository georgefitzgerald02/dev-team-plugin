# dev-team plugin

A Claude plugin that gives every Claude Code session, in any repo, a small
engineering team. Describe a change in the Claude app's **Code** tab and an
orchestrator plans it, delegates the work to specialist subagents, and
finishes with a tested, reviewed **draft pull request**.

| Component | Type | Role |
|---|---|---|
| `ship-feature` | Skill | Orchestrator. Triggers on any request to build, change or fix code. Plans, delegates, runs the review loop, opens the PR. Never edits code itself. |
| `developer` | Agent (Opus) | Implements one scoped task to make failing tests pass. |
| `tester` | Agent (Sonnet) | Writes failing tests first, then verifies with the full suite. Test files only. |
| `reviewer` | Agent (Opus) | Independent code review. **Read-only.** |
| `designer` | Agent (Opus) | UI changes: design spec before, screenshot QA after. **Read-only.** |
| `security-reviewer` | Agent (Opus) | Auth, input handling, secrets, dependencies. **Read-only.** |
| `docs-writer` | Agent (Sonnet) | Keeps README, docs and CHANGELOG in step. Docs only. |

The core trio (tester, developer, reviewer) runs on every change; the other
three join only when the orchestrator decides the change needs them.

## Repository layout

```
.claude-plugin/
  marketplace.json                 makes this repo a plugin marketplace
plugins/dev-team/
  .claude-plugin/plugin.json       plugin manifest
  skills/ship-feature/SKILL.md     orchestrator
  agents/*.md                      the six subagents
.github/workflows/
  package-plugin.yml               validates and builds dev-team-plugin.zip
```

---

## Setup

The private GitHub repository
[`georgefitzgerald02/dev-team-plugin`](https://github.com/georgefitzgerald02/dev-team-plugin)
is assumed to already exist and contain this plugin. Pick **one** of the two
installation routes below. Route A is recommended, because edits you push to
this repo reach your account automatically.

### Prerequisites (both routes)

1. A Claude **Pro, Max or Team** plan with Claude Code cloud sessions.
2. GitHub connected at [claude.ai/code](https://claude.ai/code), with the
   [Claude GitHub App](https://github.com/apps/claude) installed and granted
   access to the private `georgefitzgerald02/dev-team-plugin` repository.

### Route A: install from this repo as a marketplace (recommended)

1. Open **Customize → Plugins** on [claude.ai](https://claude.ai/customize/plugins)
   or in the Claude app.
2. Choose **Add → Add marketplace** and enter
   `georgefitzgerald02/dev-team-plugin`.
   - For a private repo, connect GitHub and give the Claude GitHub App
     access to this repo when the dialog asks.
3. Find **dev-team** in the marketplace and select **Add**.
4. Turn on **Sync automatically** for the marketplace, so pushes to `main`
   reach your account without you doing anything.

To update the plugin later, edit files under `plugins/dev-team/` and push to
`main`. You can do this from the GitHub app, or by asking Claude in a cloud
session on this repo. New sessions pick up the change. If one doesn't, use
**Check for updates** on the marketplace.

The manifest deliberately has **no `version` field**, so each commit counts
as a new version. If you add one, bump it on every change or updates stop
flowing.

### Route B: upload the zip built by GitHub Actions

Use this route if you'd rather not add a marketplace, or want to pin an exact
build.

1. Push to `main` (or run the workflow by hand: **Actions → Package plugin →
   Run workflow**). The workflow validates the plugin and builds
   `dev-team-plugin.zip`.
2. Download the zip from the **rolling release**, which works in Safari on
   iPad and iPhone:

   ```
   https://github.com/georgefitzgerald02/dev-team-plugin/releases/download/latest/dev-team-plugin.zip
   ```

   It's also on the repo's **Releases** page as "Latest build". Each workflow
   run also attaches it as an artifact. GitHub wraps artifacts in an extra
   zip, so unzip the artifact once before uploading, or just use the release.
3. In **Customize → Plugins → Add → Upload plugin**, choose
   `dev-team-plugin.zip`.

To update: push your change, wait for the workflow to finish, then
**remove** the old plugin and upload the new zip.

For a versioned build, push a tag:
`git tag v1.0.0 && git push origin v1.0.0`. That creates a `v1.0.0` release
with the zip attached.

### After either route

- **Remove any older copies.** If you uploaded the earlier `dev-team` skill
  under **Customize → Skills**, delete it, so two orchestrators don't
  compete.
- **Projects:** a Project's threads load only the plugins added to that
  Project. In each Project, open **Project settings → Plugins** and add
  **dev-team**.
- **Your PC:** Claude Code (v2.1.273 or later, signed in with your claude.ai
  account) syncs the plugin automatically. No install is needed.

### Check it works

Start a new session in the **Code** tab on any repo and send:

> Which dev-team agents do you have?

You should see `dev-team:developer`, `dev-team:tester`, `dev-team:reviewer`,
`dev-team:designer`, `dev-team:security-reviewer` and
`dev-team:docs-writer`. If they're missing, see
[Troubleshooting](#troubleshooting).

---

## Usage

In the Claude app: **Code** → pick a repo and branch → choose **Auto** (or
**Plan**, to approve the plan before any edits) → describe the change:

> Add a dark mode toggle to the settings page

> Fix https://github.com/<you>/<repo>/issues/42

The orchestrator then works through these steps:

1. Plans the change. It stops for your approval if the request is
   ambiguous, large, risky, or adds a dependency.
2. Has the tester write failing tests, then the developer make them pass.
3. Verifies the change, and runs designer QA for UI changes.
4. Gets an independent review, plus a security review where relevant, with
   at most 2 fix rounds.
5. Opens a draft PR and reports back in a few lines.

To force the workflow when the request is phrased loosely, say "use
dev-team:ship-feature to …". To skip it for a trivial change, just ask
directly; the orchestrator also skips itself for one-liners.

Review the PR in the GitHub app and comment on it. With **Auto-fix**
enabled on the PR, Claude responds to review comments and CI failures.

### Tips

- **Add a `CLAUDE.md` to each repo** listing install, test, lint and
  type-check commands. Without one, the orchestrator works them out from
  `package.json`, `Makefile`, CI files and so on, which is slower.
- **Add preview deployments** (Vercel, Netlify or Cloudflare Pages) so each
  PR gets a link you can open on an iPad. The orchestrator puts the preview
  URL in the PR when a deploy bot posts one.
- **Turn on branch protection** on `main` in your repos: require a PR and
  passing checks. The agents are told never to merge or push to `main`;
  branch protection makes that a hard rule.

## Customising

- **Change an agent:** edit `plugins/dev-team/agents/<name>.md`. The
  frontmatter sets `tools`, `model` and `description`; the body is the
  agent's instructions. Plugin agents ignore `hooks`, `mcpServers` and
  `permissionMode`.
- **Add an agent:** create `plugins/dev-team/agents/<new>.md`, then add a
  row for it to the team table in `skills/ship-feature/SKILL.md`, saying
  when to use it.
- **Change the workflow:** edit `plugins/dev-team/skills/ship-feature/SKILL.md`.
- **Validate before pushing** (optional, needs Claude Code installed):

  ```bash
  claude plugin validate .
  claude plugin validate ./plugins/dev-team
  ```

  The workflow runs the same checks on every push and PR. Warnings about a
  missing version, README, licence or icon only matter for publishing to
  Anthropic's public directory, so you can ignore them.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Agents missing in a cloud session | Start a **new** session (plugins load at session start). Check the plugin is toggled on in Customize → Plugins. If it's still missing, the orchestrator falls back to general-purpose subagents; to get the real agents, run the work in a **Project** with the plugin added under Project settings → Plugins. |
| Missing in a Project thread | Add it under **Project settings → Plugins**. Projects don't use your account's plugin list on their own. |
| "Add marketplace" can't see the repo | Give the Claude GitHub App access to this repo on GitHub (Settings → Applications → Claude → Repository access). |
| Update didn't arrive | Route A: **Check for updates** on the marketplace, and make sure there's no `version` field you forgot to bump. Route B: remove the plugin and upload the new zip. |
| Upload rejected | Use the zip from the workflow. It must contain `dev-team/.claude-plugin/plugin.json`; zipping the wrong folder level is the usual cause. |
| Orchestrator doesn't trigger | Say "use dev-team:ship-feature to …", or check for a competing older `dev-team` skill under Customize → Skills. |
