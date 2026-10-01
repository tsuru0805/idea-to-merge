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
codex exec "审查当前暂存区的 diff,按 AGENTS.md 的裁决格式输出。" < /dev/null
```

`codex exec` 会:

- 读 cwd 的 `AGENTS.md`;
- 跑你给的任务(它可以自己 `git diff`、`rg`、`cat` 等只读命令收集上下文);
- 把结果打到 stdout。

Claude Code 通过它的 **Bash 工具**跑这条命令、把 stdout 当成「Codex 的裁决」读回来。对 Claude Code 来说,这和跑任何别的 shell 命令没区别——这就是「内部调用」的全部魔法。

### 调用形态的三条硬规矩(都是挂死过才定的)

1. **单独一条、前台跑。** 别把 `codex exec` 串在 `git commit && git push && ...` 后面,也别丢后台。实测:串进 `&&` 链又被后台化时,它会 CPU 0%、零输出、卡十几分钟不退;单独一条前台跑,一两分钟正常出结果。先单独跑完 git 操作,再单独一条跑审查。
2. **每条都加 `< /dev/null`。** 有些 agent 运行环境会自作主张把长命令转去后台;后台没有 TTY,codex 会一直等标准输入,于是挂死。把标准输入关掉,就算被后台化也能跑完。**无例外、每条都加**,别靠「这条应该是前台」侥幸——我们家忘加过一次,又挂死了。
3. **默认模型被拒就用 `-m` 指定。** CLI 版本和服务端模型清单不同步时,默认模型可能直接被拒(报「需要更新版本的 Codex」之类)。临时绕法是 `codex exec -m <当前可用的模型名> ...`;要不要升级 CLI,由维护者决定,别让 AI 自己升级你的安装。

**管道喂 diff 的写法不用改**:`git diff --staged | codex exec ...` 的标准输入已经被 diff 占了,读完就走,不会挂。

挂死了怎么办:`pgrep -f '[c]odex exec'` 找到你自己起的那个审查进程,kill 掉(它只是只读审查,不是生产进程),然后按上面三条重跑。

---

## 4. 把审查方锁成只读(重要)

审查方**不该有写权限**——它的职责是挑刺,不是动手。用 `--sandbox read-only` 把这次 exec 锁成只读:

```bash
codex exec --sandbox read-only \
  "审查暂存区 diff,按 AGENTS.md 裁决格式输出。" < /dev/null
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

> 「做完一段就用 `codex exec` 起 review,按 AGENTS.md 走——单独一条前台跑,末尾加 `< /dev/null`;Codex 阻断你就改,改完再审,直到放行;它建议的修法你实现之后,终版再回递它一轮;然后把裁决写进 commit message 再来找我。」

也可以把它固化成一个 Claude Code 的**自定义 slash command** 或项目说明(`CLAUDE.md`),让「做完→自审」变成默认动作,不用每次叮嘱。

一次典型的内部调用,Claude Code 跑的命令长这样:

```bash
git add path/to/changed_file        # 只 add 本次改动的文件,别用 git add -A 把无关改动也拖进来
git diff --staged | codex exec --sandbox read-only \
  "这是本次改动的暂存 diff。按 AGENTS.md 的裁决格式审查,重点核完成度声明有没有超出证据层级。"
```

读回裁决 → 是「阻断 / 需改后通过」就改 → 再审 → 「通过」后把要点写进 commit message。

---

## 6. 什么时候别用 Codex,换一个审查者

Codex 跑在沙箱里,这是它当只读审查方的优点,也是它的边界:

- **审查对象涉及运行时实况**(服务管理器里挂了什么、端口通不通、进程在不在、定时任务、真实库)→ 沙箱看不见你的用户级服务、连不上本地端口,会报**假阻断**,也会因此漏掉真阻断。这类审查换成**能进真实环境**的审查者——比如在 Claude Code 里派一个订阅内的子 agent(指定强档模型),让它在你机器上自己去查、去 `/tmp` 里复现。
- **Codex 额度用完了** → 同样用订阅内子 agent 兜底,别停审,也别为了「异构」去走按量计费的裸 API。
- 派子 agent 审查时,prompt 末尾写「全部自己动手,不许再派子 agent」,并把 AGENTS.md 的审查规约和裁决格式一并给它。

完整说明见 [全景文档 A7](idea-to-merge.md#a7--审查方选择与兜底)。

---

## 下一步

- 循环每一步在做什么:[`workflow.md`](workflow.md)
- 为什么这么设计 / 什么时候别用:[`why-it-works.md`](why-it-works.md)
- 跟着一个真例子走:[`../examples/`](../examples/)
