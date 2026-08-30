# x-cmd-action/x-cmd

> 安装 [x-cmd](https://github.com/x-cmd/x-cmd)（纯 POSIX shell 库，兼容 dash/ash/bash/zsh）到 `~/.x-cmd.root/`。**只做一件事** —— 从 [`x-cmd/action`](https://github.com/x-cmd/action) 里把 install 这一步抽出来，变成独立、零依赖的 action。

[English](./README.md)

## 为什么需要独立的 action？

`x-cmd/action` 是个**完整的复合 action** —— 装 x-cmd **+** 配 SSH + 配 git identity + 配 docker + 克隆 workspace repo + 上传 artifact。多数用户需要这些。但有时候你只想要 install：

- 想用 `x` 命令但不要 `x-cmd/action` 的其他表面
- 想把 x-cmd install 和自己其他自定义 setup step 拼起来
- 写下游 action，需要 x-cmd 但不需要 SSH / docker

这个 action 就是那个 install 步骤，没有别的。

## 用法

这个 step 跑完后，x-cmd 装在约定路径 `~/.x-cmd.root/` 下。两种调用方式：

```yaml
steps:
  - uses: x-cmd-action/x-cmd@v1

  # 姿势 1：当成外部命令 —— 每次新建子 shell
  - run: x-cmd cowsay "one-liner"

  # 姿势 2：source X，把 x 当 shell 函数用（更快、共享状态）
  - run: |
      . ~/.x-cmd.root/X
      x cowsay "loaded"
      x cowsay "faster, no subshell overhead"
```

`outputs.root` 是给诊断 / 非默认安装路径用的：

```yaml
- uses: x-cmd-action/x-cmd@v1
  id: x-cmd

- run: |
    echo "installed at ${{ steps.x-cmd.outputs.root }}"
    test -f "${{ steps.x-cmd.outputs.root }}/X"
```

## Inputs

| Input | 默认 | 说明 |
| --- | --- | --- |
| `stream` | `index.html` | 发布流。可选：`index.html`（stable）/ `x7`（alpha）/ `x6`, `x5`, `x4`, `x3`, `x2`, `x1`（实验版，x1 最旧，x6 最新）/ `x0`（社区开发版）。映射到 x-cmd 内部用的 `___X_CMD_GHACTION_X` 环境变量。 |

## Outputs

| Output | 说明 |
| --- | --- |
| `root` | x-cmd 安装的绝对路径（`X` 文件所在的目录）。 |

## 原理

action 只做一件事 —— 从 [`x-cmd/get`](https://github.com/x-cmd/get) `eval` 装 x-cmd：

```bash
eval "$(curl -fsSL \
    "https://raw.githubusercontent.com/x-cmd/get/main/$___X_CMD_GHACTION_X")"
```

完事。和 `x-cmd/action` 内部 `___x_cmd_ghaction_init_x_cmd`（`lib/index.sh` 第 12 行）做的一模一样，只是没包 SSH/git/docker 那层壳。

action **幂等**：`~/.x-cmd.root/X` 已存在就跳过 install。所以同一 job 多次调用，开销是 ~100ms（只查个文件），不是 ~5s（重下整套）。

## 对比

| | `x-cmd-action/x-cmd` | `x-cmd/action` |
| --- | --- | --- |
| 装 x-cmd | ✅ | ✅ |
| 配 SSH / git identity / docker | ❌ | ✅ |
| 上传 artifact | ❌ | ✅ |
| 克隆 workspace repo | ❌ | ✅（通过 `ws_owner_repo`） |
| 纯 shell，无 Node.js | ✅ | ✅ |
| Idempotent install | ✅ | ❌（目前 — 同样的 install 方法，但没加 skip 检查） |
| `root` 路径 output | ✅ | ❌ |

只想要 x-cmd → 用 `x-cmd-action/x-cmd`。想要全套 bootstrap → 用 `x-cmd/action`。

## 它跟其它 action 的关系

`x-cmd-action/x-cmd` 是 `x-cmd-action` org 里**几个同级 action 之一** —— 各自只做一件事，互相不依赖：

- **`x-cmd-action/x-cmd`** —— 装 x-cmd（这个 action）
- **`x-cmd-action/checkout`** —— 把 repo 克隆进 workspace
- **`x-cmd-action/gitmirror`** —— 跨平台同步 repo

需要哪个就拿哪个，可以自由组合，不会重叠。

**它不是 `x-cmd/action` 的"小子集"**。两者是不同的工具：

- **`x-cmd/action`**（独立仓库，不在这 org）装 x-cmd 并且**默认你会用 x-cmd 命令做剩下的事** —— `x gitb`、`x ws` 等等。action 的职责是"让 runner 准备好 x-cmd"，你的脚本的职责是"用 x-cmd 把活干完"。
- **`x-cmd-action/x-cmd`** 只装 x-cmd。它不假设你后面都用 x-cmd —— 剩下的用别的 action 就行。

CI 是"x-cmd 一统天下"风格 → 用 `x-cmd/action`。想要 x-cmd 同时和其它工具混用 → 用这个。

## 许可证

Apache 2.0 —— 见 [`LICENSE`](LICENSE)。

## 相关链接

- [x-cmd/action](https://github.com/x-cmd/action) —— 完整 bootstrap action，本 action 是从它抽出来的。
- [x-cmd/get](https://github.com/x-cmd/get) —— x-cmd 安装器（本 action 拉的就是这个）。
- [x-cmd-action/checkout](https://github.com/x-cmd-action/checkout) —— 纯 shell checkout action。
- [x-cmd-action/gitmirror](https://github.com/x-cmd-action/gitmirror) —— 跨平台 repo 镜像。