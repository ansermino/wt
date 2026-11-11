# AGENTS.md

`wt` is one executable Python script, `wt`, plus `README.md` and `examples/`. There is no build, package or test suite.

## Rules

- Standard library only, Python 3.11+. No third-party imports.
- Keep it one file. Users install it by symlinking `wt` onto their `PATH`.
- No personal or employer-specific data anywhere in the repo: no usernames, home paths, company or project names, real ticket keys or tokens. Examples use `ABC-123`, `myapp` and `me/`.
- User-facing failures raise `Die(message)`; `main()` prints them as `wt: <message>` and exits 1. Never let a traceback reach the user for an expected error.
- External commands go through `run()` / `git()`; `zellij action` calls go through `zj()`.
- Config keys are read with `cfg.get(key, default)` and documented in `examples/config.toml` and the README's tables. A new command, flag or config key updates the module docstring (it is the `--help` text), the README and the example config in the same change.
- Comments state the non-obvious reason only.

## Checking a change

```sh
python3 -m py_compile wt
./wt --help
./wt new -n some-branch      # dry run inside any git repo: prints branch, path and tab, changes nothing
./wt ls
```

`sync`, `open` and the tab handling need a live zellij session; run them inside one. Before committing, grep the diff for anything identifying (see Rules).
