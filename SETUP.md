# Setting up a machine for Helm

From a Windows or macOS computer with Claude Code to the full workflow: Helm on top,
the [`flow` plugin](https://github.com/MarkusVGJensen/flow) driving each
session, and Anthropic's first-party plugins doing review and planning.

The quickest way through is to let Claude do it. In Claude Code, say:

> Set up Helm on this machine by following
> https://raw.githubusercontent.com/MarkusVGJensen/helm-releases/main/SETUP.md

## For Claude

Do the steps in order. Each says who does it:

- **Claude**: run it yourself, in PowerShell.
- **The person**: it needs their keyboard (a login or a choice). Say exactly
  what to type, and wait. A command they can run inside this session is
  written as `! <command>`.

Ask for every value you cannot work out from the machine or the project. Never
invent a project path, a GitLab project or a build command. Skip a step whose
result is already in place, and say that you did.

The steps are written for Windows and PowerShell. **On a Mac**, follow
[On a Mac](#on-a-mac) at the end wherever a step differs: the tools, the
install and the paths.

A tool installed in step 1 is not on this session's `PATH` until a new shell
starts. Refresh it before using the tool:

```powershell
$env:PATH = [Environment]::GetEnvironmentVariable("PATH", "Machine") + ";" + [Environment]::GetEnvironmentVariable("PATH", "User")
```

## 1. Tools (Claude)

Install what `Get-Command` cannot find:

| Tool | Why | Install |
| --- | --- | --- |
| Node.js 20+ | the status hooks Helm installs | `winget install --id OpenJS.NodeJS.LTS -e` |
| git | worktrees | `winget install --id Git.Git -e` |
| glab | GitLab from Helm and the plugin | `winget install --id GLab.GLab -e` |
| GitHub CLI | Helm's update check | `winget install --id GitHub.cli -e` |

Add `--accept-source-agreements --accept-package-agreements` so `winget`
does not stop to ask. Claude Code itself is already here, since you are
running in it.

## 2. Logins (the person)

Ask the person to run these, one at a time, and to say when each is done:

```
! glab auth login
! gh auth login
```

For a GitLab instance other than gitlab.com, the first is
`! glab auth login --hostname <host>`; ask which host. Check both afterwards
with `glab auth status` and `gh auth status`.

## 3. The flow plugin (Claude)

```powershell
git clone https://github.com/MarkusVGJensen/flow "$HOME\.claude\skills\flow"
```

The path matters: Claude Code loads plugins placed under `~/.claude/skills/`,
and that is how the `/flow:…` commands appear in every session. If the folder
is already a clone, run `git -C "$HOME\.claude\skills\flow" pull` instead.

## 4. First-party plugins (Claude)

```powershell
claude plugin marketplace list
```

If `claude-plugins-official` is not listed, add it with
`claude plugin marketplace add anthropics/claude-plugins-official`. Then:

```powershell
foreach ($p in "pr-review-toolkit", "code-review", "feature-dev", "code-simplifier", "claude-md-management", "hookify") {
  claude plugin install "$p@claude-plugins-official" --scope user
}
```

Add the language server plugin for the project's language the same way, for
example `clangd-lsp` for C and C++ or `pyright-lsp` for Python.

`flow` delegates to these: `pr-review-toolkit` supplies the review lenses
`/flow:review-mr` fans out to, `feature-dev` the architect used for planning,
`code-review` and `code-simplifier` the quality passes. They load in new
sessions, not in this one.

## 5. Claude Code settings (Claude, one question to the person)

Merge these keys into `~/.claude/settings.json`, keeping everything already
there:

```json
{
  "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" },
  "teammateMode": "in-process"
}
```

Then ask the person about the permission mode. With
`"permissions": { "defaultMode": "bypassPermissions" }`, `/flow:start-issue`
runs from approved plan to pushed commits without stopping to ask. Without
it, Claude stops at each permission prompt and the tab waits for them. Set it
only if they say yes.

## 6. Project values for the plugin (Claude, asking the person)

Ask for the main checkout of the project if you are not already in it. Copy
`~/.claude/skills/flow/flow.config.example.json` to `~/.claude/flow.local.json`
and fill in:

- `forge`: `glab`
- `project`: the GitLab path, `group/repo`. `git remote get-url origin` in
  the checkout gives it.
- `defaultTarget`: the branch merge requests go into, usually `main` or
  `master`
- `worktreeRoot`: a short folder such as `D:/wt`, because long paths hit
  Windows' path limit. Ask which drive.
- `branchPattern`: for example `<their initials>/{issue}-{slug}`. Ask.
- `verify.*`: the project's own syntax check, targeted test, full build and
  format commands. Read the project's `CLAUDE.md` and build scripts, propose
  them, and let the person correct them.
- `review.floor`, `review.mute` and `ci.*`: keep the example's values unless
  the person says otherwise.

If the project commits a shared `<repo>/.claude/flow.json`, most of this is
already there; `flow.local.json` then only needs what differs for this
person.

## 7. Install Helm (Claude)

```powershell
$rel = Invoke-RestMethod https://api.github.com/repos/MarkusVGJensen/helm-releases/releases/latest
$asset = $rel.assets | Where-Object name -like "*setup.exe" | Select-Object -First 1
$out = Join-Path $env:TEMP $asset.name
Invoke-WebRequest $asset.browser_download_url -OutFile $out
Start-Process $out -ArgumentList "/P" -Wait
```

`/P` installs without questions, showing only a progress bar. It installs for
this user, so it needs no administrator rights, and Helm appears in the Start
menu.

## 8. Helm's settings (Claude)

Helm reads `~/.helm/config.json` at launch; anything left out takes its
default. Write the project's values there, and the same file to
`~/.helm/profiles/default.json` so the repository switcher lists it. If the
file exists already, change only these keys:

```json
{
  "repo": "D:\\path\\to\\main\\checkout",
  "worktreeRoot": "D:\\wt",
  "project": "group/repo",
  "defaultTarget": "main",
  "branchPattern": "ab/{issue}-{slug}",
  "name": "my-project",
  "configure": { "program": "", "args": [] }
}
```

- `repo`, `worktreeRoot`, `project`, `defaultTarget` and `branchPattern`: the
  values from step 6, with backslashes in Windows paths.
- `name`: what the repository is called in Helm's switcher.
- `configure`: what prepares a fresh worktree for the editor, run in each new
  worktree. Leave the program empty when the project needs nothing. Each
  argument is one string. For a CMake project that needs the Visual Studio
  environment:
  `{ "program": "cmd.exe", "args": ["/c call \"<path to vcvars64.bat>\" >nul && <configure script>"] }`.
  Work it out from the project's build scripts and `CLAUDE.md`, and confirm
  it with the person.

## 9. First launch (the person)

Ask the person to start Helm from the Start menu, then:

1. Open **Settings** (the gear in the rail, or Ctrl+,) and pick their
   **Editor** from the IDEs Helm found on the machine.
2. Read the other values through once; they are the ones from step 8.

## 10. The project itself

The conventions Claude follows come from the project's own `CLAUDE.md`: test
style, comment style, build rules, commit rules. Helm and `flow` read it;
they do not carry it. If the project has none, tell the person: every agent
will behave generically until one is written.

## 11. Check

Claude checks:

- `claude plugin list` shows the plugins from step 4.
- `~/.claude/skills/flow/.claude-plugin/plugin.json` exists.
- `glab auth status` and `gh auth status` succeed.
- `~/.helm/config.json` has `repo`, `worktreeRoot` and `project` filled in.

The person checks, in Helm:

1. No warning banner at the top: `claude`, `node`, `git` and `glab` were
   found, `glab` is logged in, and `flow` was found and is new enough. If
   something is wrong and the banner does not say enough, **Diagnostics** at
   the foot of the rail does, with a Copy button.
2. The Hub tab appears and its dot turns amber within a few seconds. Claude
   asks once whether to trust the worktree folder; answer it in the tab.
3. The Reviews queue lists their assigned merge requests.
4. **New issue**: paste a real issue link and start it. Expect a worktree,
   Claude stopping at the plan, and a **Plan** button that shows it.

## On a Mac

Everything not listed here is the same as on Windows, run in the default
shell (zsh) instead of PowerShell; `~` is the home folder on both.

- **Step 1, tools.** Install [Homebrew](https://brew.sh) first if `brew` is
  missing; its installer asks for the person's password, so ask them to run
  it. Then: `brew install node git glab gh`. There is no `PATH` to refresh.
- **Step 3, flow.** `git clone https://github.com/MarkusVGJensen/flow ~/.claude/skills/flow`
- **Step 4, plugins.** The same `claude plugin install` commands, one per
  plugin:
  `for p in pr-review-toolkit code-review feature-dev code-simplifier claude-md-management hookify; do claude plugin install "$p@claude-plugins-official" --scope user; done`
- **Step 6, project values.** `worktreeRoot` is a short folder in the home
  folder, such as `~/wt`, written out in full (`/Users/<name>/wt`).
- **Step 7, install Helm.**

  ```sh
  url=$(curl -s https://api.github.com/repos/MarkusVGJensen/helm-releases/releases/latest \
    | grep -o '"browser_download_url": *"[^"]*_universal\.dmg"' | cut -d'"' -f4)
  curl -L -o /tmp/Helm.dmg "$url"
  hdiutil attach -nobrowse -quiet /tmp/Helm.dmg -mountpoint /tmp/helm-dmg
  ditto /tmp/helm-dmg/Helm.app /Applications/Helm.app
  hdiutil detach -quiet /tmp/helm-dmg
  xattr -dr com.apple.quarantine /Applications/Helm.app
  ```

  Helm is not notarized by Apple. The last line lets it open without the
  "cannot be opened" warning; tell the person you ran it and why.
- **Step 8, settings.** The same files, `~/.helm/config.json` and
  `~/.helm/profiles/default.json`, with `/` paths. `configure` is whatever the
  project needs before a build, often nothing; for a shell line, use
  `{ "program": "/bin/zsh", "args": ["-lc", "<command>"] }`.
- **Step 9, first launch.** Helm is in Applications and Launchpad. The
  Editor list shows the IDEs Helm found, Xcode among them.

## Updates

Helm checks this repository for a newer release at launch, through `gh`, and
offers to install it. To update `flow`, use **Update flow** in Helm's
Settings, or `git -C ~/.claude/skills/flow pull`.
