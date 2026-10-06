# Setting up a machine for Helm

From a fresh Windows computer with Claude Code access to the full workflow:
Helm on top, the [`flow` plugin](https://github.com/MarkusVGJensen/flow)
driving each session, and Anthropic's first-party plugins doing review and
planning.

If you are Claude reading this for the user: do the steps in order, run the
check at the end, and ask for the values in steps 5 and 6 that you cannot
work out. Do not invent project paths.

## 1. Tools

| Tool | Why | Install |
| --- | --- | --- |
| Claude Code | the sessions | per [Anthropic's docs](https://docs.anthropic.com/en/docs/claude-code); run `claude` once and log in |
| Node.js 20+ | the status hooks Helm installs | `winget install OpenJS.NodeJS.LTS` |
| git | worktrees | `winget install Git.Git` |
| glab | GitLab from Helm and the plugin | `winget install glab.glab`, then `glab auth login` |
| GitHub CLI | Helm's update check | `winget install GitHub.cli`, then `gh auth login` |

## 2. The flow plugin

```
git clone https://github.com/MarkusVGJensen/flow ~/.claude/skills/flow
```

The path matters: Claude Code loads plugins placed under `~/.claude/skills/`,
and that is how the `/flow:…` commands appear in every session. Helm can do
this for you: **Settings › Plugins** says whether the plugin was found and
has a **Fetch flow** button that clones it there, and an **Update flow**
button afterwards.

## 3. First-party plugins

Inside any Claude Code session:

```
/plugin install pr-review-toolkit@claude-plugins-official
/plugin install code-review@claude-plugins-official
/plugin install feature-dev@claude-plugins-official
/plugin install code-simplifier@claude-plugins-official
/plugin install claude-md-management@claude-plugins-official
/plugin install hookify@claude-plugins-official
```

Add the language server plugin for your language too, for example
`clangd-lsp@claude-plugins-official` for C and C++.

`flow` delegates to these: `pr-review-toolkit` supplies the review lenses
`/flow:review-mr` fans out to, `feature-dev` the architect used for planning,
`code-review` and `code-simplifier` the quality passes.

## 4. Claude Code settings

Merge this into `~/.claude/settings.json`:

```json
{
  "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" },
  "teammateMode": "in-process",
  "enabledPlugins": {
    "pr-review-toolkit@claude-plugins-official": true,
    "code-review@claude-plugins-official": true,
    "feature-dev@claude-plugins-official": true,
    "code-simplifier@claude-plugins-official": true,
    "claude-md-management@claude-plugins-official": true,
    "hookify@claude-plugins-official": true
  }
}
```

The permission mode is your choice. With
`"permissions": { "defaultMode": "bypassPermissions" }`, `/flow:start-issue`
runs from approved plan to pushed commits without stopping to ask. Without
it, Claude stops for each permission prompt and the tab waits for you.

## 5. Project values for the plugin

Copy `~/.claude/skills/flow/flow.config.example.json` to
`~/.claude/flow.local.json` and fill in:

- `forge` (`glab`), `project` (`group/repo`), `defaultTarget` (`main` or `master`)
- `worktreeRoot`: keep it short, such as `D:/wt`, because long paths hit
  Windows' path limit
- `branchPattern`, for example `you/{issue}-{slug}`
- `verify.*`: the project's own syntax check, targeted test, full build and
  format commands. Read the project's `CLAUDE.md` and build scripts to fill
  them in.
- `review.floor`, `review.mute`: what the reviewer should not comment on
- `ci.*`: poll interval, stalled-job minutes, known flaky tests

If the project commits a shared `<repo>/.claude/flow.json`, most of this is
already there and only your personal values are left.

## 6. Helm

Download and run the installer from the
[latest release](https://github.com/MarkusVGJensen/helm-releases/releases/latest).

On first launch the Settings dialog opens; the gear in the rail, or Ctrl+,,
opens it again later. Confirm the prefilled values and add:

- **Repository**: your main checkout of the project. Worktrees are created
  from it.
- **Configure command**: what prepares a fresh worktree for the editor. For a
  CMake project on Windows this is `cmd.exe` with one line:
  `/c call "<path to vcvars64.bat>" >nul && <configure script>`
- **Editor**: pick one of the IDEs Helm found on the machine.
- **Update source**: `MarkusVGJensen/helm-releases`
- **First prompts**: leave the defaults unless you know why not.

Settings live in `~/.helm/config.json`, tabs in `~/.helm/state.json`. For
more than one repository, add a profile per repository in Settings.

## 7. The project itself

The conventions Claude follows come from the project's own `CLAUDE.md`: test
style, comment style, build rules, commit rules. Helm and `flow` read it;
they do not carry it. A project without a `CLAUDE.md` gets generic behaviour
from every agent.

## 8. Check

1. Launch Helm. No warning banner shows at the top: `claude`, `node`, `git`
   and `glab` were found, `glab` is logged in, and the `flow` plugin was found
   and is new enough. If something is wrong and the banner does not say
   enough, **Diagnostics** at the foot of the rail does.
2. The Hub tab appears and its dot turns amber within a few seconds. That
   proves the terminal, the hooks and the state watcher.
3. Open the Reviews queue. Your assigned merge requests are listed.
4. **New issue**: paste a real issue link and start it. Expect a worktree, a
   configure chip that reaches Configured, Claude stopping at the plan, and a
   **Plan** button that shows the plan.

## What does not travel between machines

- Claude asks "do you trust this folder" the first time it runs in a new
  worktree root. Answer once in the tab; Helm types the first prompt after.
- `claude`, `glab` and `gh` each need a one-time login per machine.
