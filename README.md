# x-cmd-action/x-cmd

> Install [x-cmd](https://github.com/x-cmd/x-cmd) (pure-POSIX shell library) into `~/.x-cmd.root/`. **Does one thing only** — the install step from [`x-cmd/action`](https://github.com/x-cmd/action), factored out as a standalone, dependency-free action.

[中文文档](./README.cn.md)

## Why a separate action?

`x-cmd/action` is a **full composite** — it installs x-cmd **and** wires up SSH, git identity, docker, workspace repo, and artifact upload. Most users want all of that. But sometimes you only want the install:

- You want to use `x` commands without the rest of `x-cmd/action`'s surface
- You're composing x-cmd install with other custom setup steps
- You're writing a downstream action that depends on x-cmd but doesn't need SSH / docker

This action is exactly the install step, nothing more.

## Usage

After this step runs, x-cmd is at the conventional path `~/.x-cmd.root/`. Two ways to invoke it:

```yaml
steps:
  - uses: x-cmd-action/x-cmd@v1

  # Option 1: as an external command — fresh subshell per call
  - run: x-cmd cowsay "one-liner"

  # Option 2: source X, then use `x` as a shell function (faster, shared state)
  - run: |
      . ~/.x-cmd.root/X
      x cowsay "loaded"
      x cowsay "faster, no subshell overhead"
```

The `outputs.root` is exposed for diagnostics or non-default install paths:

```yaml
- uses: x-cmd-action/x-cmd@v1
  id: x-cmd

- run: |
    echo "installed at ${{ steps.x-cmd.outputs.root }}"
    test -f "${{ steps.x-cmd.outputs.root }}/X"
```

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `stream` | `index.html` | Release stream. Options: `index.html` (stable) / `x7` (alpha) / `x6`, `x5`, `x4`, `x3`, `x2`, `x1` (experimental, x1 oldest, x6 newest) / `x0` (community development). Maps to the `___X_CMD_GHACTION_X` env var used inside x-cmd. |

## Outputs

| Output | Description |
| --- | --- |
| `root` | Absolute path to the x-cmd install (where `X` lives). |

## How it works

The action does one thing — `eval` the x-cmd installer from [`x-cmd/get`](https://github.com/x-cmd/get):

```bash
eval "$(curl -fsSL \
    "https://raw.githubusercontent.com/x-cmd/get/main/$___X_CMD_GHACTION_X")"
```

That's it. Identical to what `x-cmd/action` does internally in `___x_cmd_ghaction_init_x_cmd` (`lib/index.sh` line 12), just without the surrounding SSH/git/docker wiring.

The action is **idempotent**: if `~/.x-cmd.root/X` already exists, the install step is skipped. So calling it multiple times in the same job costs ~100ms (just the file check) instead of ~5s (full download).

## Comparison

| | `x-cmd-action/x-cmd` | `x-cmd/action` |
| --- | --- | --- |
| Installs x-cmd | ✅ | ✅ |
| Sets up SSH / git identity / docker | ❌ | ✅ |
| Uploads artifacts | ❌ | ✅ |
| Clones workspace repo | ❌ | ✅ (via `ws_owner_repo`) |
| Pure shell, no Node.js | ✅ | ✅ |
| Idempotent install | ✅ | ❌ (currently — same install method, but no skip-check) |
| Output: `root` path | ✅ | ❌ |

Use `x-cmd-action/x-cmd` when you want x-cmd alone. Use `x-cmd/action` when you want the full bootstrap package.

## How it relates to other actions

`x-cmd-action/x-cmd` is **one of several sibling actions** in the `x-cmd-action` org — each does one thing, none depend on the others:

- **`x-cmd-action/x-cmd`** — install x-cmd *(this action)*
- **`x-cmd-action/checkout`** — clone a repo into the workspace
- **`x-cmd-action/gitmirror`** — sync repos across platforms

Pick whichever you need; combine freely; they don't overlap.

It is **not** a "minimal subset of `x-cmd/action`". Those are different tools:

- **`x-cmd/action`** (separate repo, not in this org) installs x-cmd AND assumes you'll do the rest of the CI with x-cmd commands themselves — `x gitb`, `x ws`, etc. The action's job is "make the runner x-cmd-ready". Your script's job is "do everything via x-cmd".
- **`x-cmd-action/x-cmd`** just installs x-cmd. It doesn't presume you'll use x-cmd for everything — use whatever other actions make sense for the rest of the job.

Use `x-cmd/action` when your CI is "x-cmd-everywhere". Use `x-cmd-action/x-cmd` when you want x-cmd alongside other tooling.

## License

Apache 2.0 — see [`LICENSE`](LICENSE).

## Related

- [x-cmd/action](https://github.com/x-cmd/action) — the full bootstrap action this one is extracted from.
- [x-cmd/get](https://github.com/x-cmd/get) — x-cmd installer (what this action fetches).
- [x-cmd-action/checkout](https://github.com/x-cmd-action/checkout) — pure-shell checkout action.
- [x-cmd-action/gitmirror](https://github.com/x-cmd-action/gitmirror) — cross-platform repo mirror.