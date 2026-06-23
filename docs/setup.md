# Setup · 装 Codex CLI + 让 Claude Code 内部调用它

这是整套工作流的技术核心:**Claude Code 在自己的会话里,用 Bash 直接起一个 Codex 非交互会话来审查自己的改动**。

> ⚠️ 下面的命令和参数请以你机器上 `codex --help` / `codex exec --help` 的实际输出为准——
> CLI 版本会变,本文写的是**形状和思路**,具体 flag 名以本地为权威。

---

## 1. 装 Codex CLI

Codex 是 OpenAI 的命令行 agent。常见装法:

```bash
# npm
npm install -g @openai/codex

# 或 Homebrew(macOS)
brew install codex
```

装完确认:

```bash
codex --version
codex --help
```

首次用要登录 / 配置 API key(按 `codex` 交互提示走一遍即可)。

---

## 2. 关键机制:Codex 自动读 `AGENTS.md`

这是整套工作流能成立的根:**Codex 会读取工作目录里的 `AGENTS.md`,把它当成项目级系统规则**(此行为以你当前 CLI 版本为准,可用 `codex exec --help` 或官方文档确认)。

所以你**不需要**每次把一长串审查规则塞进 prompt——只要:

1. 把本仓的 [`../AGENTS.md`](../AGENTS.md) 拷到你项目根目录;
2. 按你项目实际改里面的保护区路径、红线;

之后任何在该目录起的 Codex 会话,都自动带着「我是只读 reviewer、这些是保护区、裁决用固定格式」的规则。

---

## 3. 非交互调用:`codex exec`

交互式 `codex` 是给人坐在终端前用的。**让另一个 agent 调用,要用非交互模式**——一条命令进、结果出、不等键盘:

```bash
codex exec "审查当前暂存区的 diff,按 AGENTS.md 的裁决格式输出。"
```

`codex exec` 会:

- 读 cwd 的 `AGENTS.md`;
- 跑你给的任务(它可以自己 `git diff`、`rg`、`cat` 等只读命令收集上下文);
- 把结果打到 stdout。

Claude Code 通过它的 **Bash 工具**跑这条命令、把 stdout 当成「Codex 的裁决」读回来。对 Claude Code 来说,这和跑任何别的 shell 命令没区别——这就是「内部调用」的全部魔法。

---

## 4. 把审查方锁成只读(重要)

审查方**不该有写权限**——它的职责是挑刺,不是动手。用 `--sandbox read-only` 把这次 exec 锁成只读:

```bash
codex exec --sandbox read-only \
  "审查暂存区 diff,按 AGENTS.md 裁决格式输出。"
```

> **别给 `codex exec` 加 `--ask-for-approval`(`-a`)。** 这个审批 flag 只挂在顶层交互式 `codex` 上,
> `codex exec` 子命令**不收**,照抄会报 `error: unexpected argument '-a' found`。
> 原因是 `codex exec` 本来就是非交互模式,审批策略自动是 never——不需要、也不能再手动指定。
> 只读用 `--sandbox read-only` 就够了。具体可用 flag 以你机器上 `codex exec --help` 为准。
>
> 思路不变:**reviewer = 只读**。AGENTS.md 里也用文字再钉一遍「不写、不 commit、不 push」,双保险。

---

## 5. 在 Claude Code 里怎么用

你只要在会话里跟 Claude Code 说清楚循环,它就会在合适的节点自己调:

> 「做完一段就用 `codex exec` 起 review,按 AGENTS.md 走;Codex 阻断你就改,改完再审,直到放行;然后把裁决写进 commit message 再来找我。」

也可以把它固化成一个 Claude Code 的**自定义 slash command** 或项目说明(`CLAUDE.md`),让「做完→自审」变成默认动作,不用每次叮嘱。

一次典型的内部调用,Claude Code 跑的命令长这样:

```bash
git add path/to/changed_file        # 只 add 本次改动的文件,别用 git add -A 把无关改动也拖进来
git diff --staged | codex exec --sandbox read-only \
  "这是本次改动的暂存 diff。按 AGENTS.md 的裁决格式审查,重点核完成度声明有没有超出证据层级。"
```

读回裁决 → 是「阻断 / 需改后通过」就改 → 再审 → 「通过」后把要点写进 commit message。

---

## 下一步

- 循环每一步在做什么:[`workflow.md`](workflow.md)
- 为什么这么设计 / 什么时候别用:[`why-it-works.md`](why-it-works.md)
- 跟着一个真例子走:[`../examples/`](../examples/)
