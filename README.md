# Helm

A desktop app that wraps [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
for issue-driven work on a GitLab project. One tab per issue, each tab a live
Claude session in its own git worktree. Buttons either type a `/flow` command
into that session or call GitLab and git directly, and every button says
which, so you always know what costs tokens and what does not.

This repository holds only the latest installer and these instructions. The
source lives elsewhere.

## Download

**Windows (x64):** open the [latest release](https://github.com/MarkusVGJensen/helm-releases/releases/latest)
and download `Helm_<version>_x64-setup.exe`. Or, with the GitHub CLI:

```
gh release download --repo MarkusVGJensen/helm-releases --pattern "*setup.exe"
```

Run the installer. It is not code-signed, so Windows SmartScreen may say it
"protected your PC": choose **More info › Run anyway**.

**macOS:** no build is published yet.

Before Helm is useful you need Claude Code, git, glab and the `flow` plugin.
[SETUP.md](SETUP.md) walks through all of it, Helm included, and is written
for Claude to follow. With Claude Code installed, say:

> Set up Helm on this machine by following
> https://raw.githubusercontent.com/MarkusVGJensen/helm-releases/main/SETUP.md

Claude installs the tools, the plugins and Helm, and writes the config. You
log in to GitLab and GitHub when it asks, answer its questions about your
project, and pick your editor in Helm's Settings at the end.

### Updates

Helm checks for a newer release at launch and offers to install it. It asks
through `gh`, so the check needs the [GitHub CLI](https://cli.github.com/)
installed and logged in (`gh auth login`). From 1.0.0 the check reads this
repository by default. An older install points elsewhere: open **Settings**
(the gear, or Ctrl+,) and set **Update source** to
`MarkusVGJensen/helm-releases`.

## What Helm is

Helm is the window. The work happens in Claude Code through the
[`flow` plugin](https://github.com/MarkusVGJensen/flow), which takes an issue
from worktree to merge request with two human gates, and reviews other
people's merge requests with line-anchored draft comments. `flow` delegates
to Anthropic's first-party plugins for planning, reviewing and simplifying.
Nothing about a particular project lives in Helm: the project's `CLAUDE.md`
and two small config files carry that.

```
┌─ Helm  my-project ▾ ┬─────────────────────────────────────────────────────────┐
│ New issue           │ #128 Fix header layout   ● Needs input  Configured      │
│ Session in folder…  │ Issue · MR !140 · Open in editor · Close tab            │
│                     │ Setup › Plan › ■Plan › Tests › Implement › Review › …   │
│                     ├─────────────────────────────────────────────────────────┤
│ ● Hub               │                                                         │
│ OTHER     + SESSION │   Claude Code session in D:\wt\128                      │
│ ● Notes             │                                                         │
│ ISSUES              │   (full terminal: type, scroll, answer gates)           │
│ ● #128 Fix header…  │                                                         │
│ ● #131 Retry uplo…  │                                                         │
│ REVIEWS   2 waiting │                                                         │
│ ● !137 Cache inva…  │                                                         │
│ CLOSED              │                                                         │
└─────────────────────┴─────────────────────────────────────────────────────────┘
```

### Issues

1. **New issue.** Paste a GitLab issue link or type its number. Helm shows the
   worktree and branch it will create.
2. **Start.** Helm makes the worktree and branch, runs your configure command
   in the background so the editor button works, opens a Claude tab and types
   `/flow:start-issue <id> <your note>`.
3. **Plan.** The plugin plans and stops for your approval. The **Plan** button
   shows the plan as a document, diagrams drawn, with a reply box and an
   **Approve plan** button.
4. **Build.** Failing tests first, then the implementation, a self-review, the
   build and the formatter. The timeline above the terminal says which step
   it is on.
5. **Review commits.** At the second gate, **Review commits** shows the
   branch commit by commit with its diff. Mark lines, write notes, and send
   them all back to the session as one message, or approve and let the
   plugin open the merge request.
6. **Done.** Once the merge request is merged, **Refetch** offers to remove
   the worktree, branch and tab. Nothing is removed without a tick.

### Reviews

The **Reviews** queue lists the open merge requests where you are a reviewer.
**Start review** checks out the branch in its own worktree and starts
`/flow:review-mr` on it, with a note of your own if you want to steer it.

Nothing reaches GitLab yet. When the review finishes, **Review comments**
shows Claude's comments on the merge request's diff, each under the lines it
is about, the way GitLab's own review does. Each has a severity from 1 (a
nit) to 5 (must not merge as it is) that only you see, and the list beside
the diff is sorted by it. Edit, delete, or add comments of your own by
clicking a line, then **Post to GitLab** sends them as draft notes. Nobody
sees them until you submit the review in GitLab.

### Around the edges

- A status dot per tab: amber while Claude works, magenta once it has sat
  still long enough to want an answer, with one system notification.
- A Hub tab on the default branch for questions about the codebase, plain
  Claude sessions, and notes per tab.
- CI status per merge request in the rail.
- Several repositories, each with its own settings and tabs.
- **Diagnostics** at the foot of the rail says what Helm found on this
  machine, with a Copy button for whoever is helping you.

## Requirements

- Windows 10 or 11, x64
- A Claude Code subscription or API access
- A GitLab project, with `glab` logged in to it
- git and Node.js 20+

## Getting help

Open **Diagnostics** in Helm, press Copy, and send that along with what you
were doing.
