# wt

`wt` manages git worktrees across several repos. Each worktree gets its own tab in one shared [zellij](https://zellij.dev) session, and each tab opens a Claude Code session next to a shell. It is a single Python script with no dependencies outside the standard library.

## Requirements

- Python 3.11 or later (for `tomllib`)
- git 2.31 or later
- zellij (tested with 0.45)
- Optional: [`gh`](https://cli.github.com), to check out GitHub pull requests and issues by number or URL
- Optional: a [Linear](https://linear.app) API key, to name branches after Linear issues

## Install

```sh
git clone https://github.com/ansermino/wt.git ~/src/wt
mkdir -p ~/.local/bin
ln -s ~/src/wt/wt ~/.local/bin/wt
```

`~/.local/bin` must be on your `PATH`. To update, run `git -C ~/src/wt pull`.

Then set up the config:

```sh
mkdir -p ~/.config/wt/layouts
cp ~/src/wt/examples/config.toml ~/.config/wt/config.toml
cp ~/src/wt/examples/layouts/default.kdl ~/.config/wt/layouts/default.kdl
```

Edit `~/.config/wt/config.toml` and add a `[repos.<name>]` entry for each repo. `wt clone <url>` adds the entry for you.

### Zellij keybindings (optional)

Add these to `~/.config/zellij/config.kdl`, inside the `keybinds` block. Alt-w then prompts for a new worktree and Alt-x prompts for one to remove:

```kdl
shared_except "locked" {
    bind "Alt w" {
        Run "wt" "new" {
            floating true
            close_on_exit true
        }
    }
    bind "Alt x" {
        Run "wt" "rm" {
            floating true
            close_on_exit true
        }
    }
}
```

## Commands

| Command | What it does |
|---------|--------------|
| `wt new ABC-123` | Creates a worktree for the ticket, runs the post-create hook, and opens the tab `ABC-123`. |
| `wt new my-branch` | Same for a plain branch name. `branch_prefix` is added unless the name already has a `/` or the branch exists. |
| `wt new 123` | If #123 is a PR, checks out its head branch and opens the tab `PR-123`. If it is an issue, uses the issue's branch and opens the tab `GH-123`. Also accepts `#123` or the URL. Needs `gh`. |
| `wt rm ABC-123` | Runs the pre-remove hook, removes the worktree, and closes its tab. |
| `wt open ABC-123` | Opens or focuses the tab of an existing worktree. With no argument, uses the current directory's worktree. |
| `wt ls` | Lists the repo's worktrees: tab name, branch, path. |
| `wt claude` | Starts Claude in the current worktree: with a pending `--prompt`, a new session named after the tab that opens with that message; otherwise `claude --continue`, falling back to a new session. Layouts run this as the Claude pane's command. |
| `wt sync` | Adds a tab for each worktree of each configured repo that lacks one, plus the `extra_tabs`. Reports tabs that match no worktree but does not close them. |
| `wt clone <url> [dir]` | Bare-clones into `./<repo-name>` (or `dir`) with the main worktree at `work/<default-branch>`, adds a `[repos.<name>]` entry to the config, and opens its tab. |
| `wt` | Outside zellij: attaches the session, creating it with a tab per worktree of every repo. Inside zellij: same as `wt sync`. |

`new`, `rm`, `open` and `ls` act on one repo: `--repo <name>` if given, else the repo containing the current directory, else `default_repo`. `wt new` and `wt rm` with no argument prompt for one.

| Flag | Command | Effect |
|------|---------|--------|
| `-b <base>` | `new` | Base for a new branch (default: `base` in the config, `origin/main`). Ignored if the branch exists. |
| `-n` | `new` | Dry run: print the branch, path and tab without changing anything. |
| `--no-focus` | `new`, `open` | Open the tab in the background instead of switching to it. An existing tab is left unfocused. |
| `--prompt <text>` | `new` | Start the new tab's Claude session with `<text>` as its first message. Only for a new worktree: if the branch is already checked out, `wt` exits with an error instead of focusing it. Needs a layout whose Claude pane runs `wt claude`. |
| `-f` | `rm` | Remove even with uncommitted changes. |
| `-d` | `rm` | Also delete the branch if it is merged into `base`. |

## How names are chosen

- A ticket key is any `KEY-123`. `wt` uses an existing local or origin branch for that ticket first. Otherwise it asks Linear for the issue's branch name, using the key in `~/.config/wt/linear-api-key` (chmod 600) or `$LINEAR_API_KEY`.
- A GitHub issue uses the branch linked to it on GitHub first, then a local or origin branch named `123-…` (with or without `branch_prefix`), else a new `branch_prefix` + `123-<title>`. With `link_issue_branches = true`, `wt` creates that new branch on GitHub with `gh issue develop` so it shows as linked on the issue. This needs `base` to be an `origin/` branch.
- Directory names are lowercase, so a directory can serve as a Docker Compose project name.
- A tab is named after the ticket key in its directory (`ABC-123`), else after the directory. The main checkout's tab is `main`.
- Tabs of repos other than `default_repo` carry `tab_prefix` (`o/ABC-123`), so the same ticket in two repos gets two tabs.
- If the ticket or branch is already checked out, `wt new` focuses that tab instead.

## Files

| Path | Purpose |
|------|---------|
| `~/.config/wt/config.toml` | Global settings plus a `[repos.<name>]` entry per repo, in tab order. See `examples/config.toml`. |
| `~/.config/wt/layouts/<repo>.kdl` | The panes of one worktree tab. Falls back to `default.kdl`, then to a built-in Claude + shell split. The Claude pane should run `wt claude`, so `--prompt` reaches it. |
| `~/.config/wt/hooks/<repo>/post-create`, `pre-remove` | Executables run inside the worktree with `WT_REPO`, `WT_NAME`, `WT_PATH`, `WT_BRANCH` and `WT_MAIN` set. A failing hook aborts the command. |

`$XDG_CONFIG_HOME` and `$XDG_CACHE_HOME` are honoured. Generated zellij layouts go to `~/.cache/wt/`.
