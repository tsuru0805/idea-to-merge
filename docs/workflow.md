# Workflow · 七步循环逐步详解

这套循环有两个角色 + 一个人:

- **Builder(做工方)** = Claude Code。读代码、写代码、commit、push、写工单和 handoff。
- **Reviewer(审查方)** = Codex,经 `codex exec` 内部调用。只读,只挑刺、只裁决,不动手。
- **你(人类)** = 维护者。在两个闸门拍板:提案通过、收工。

> 主线是**内部调用**:Builder 在自己会话里直接 `codex exec` 起 Reviewer,不需要你在两个终端之间搬消息。
> 当裁决有歧义、或触到需要你定夺的边界时,才回退到「人类传话」——Builder 把 Codex 的裁决文本转给你,你拍板。

---

## 第 1 步 · 你提想法

随口说都行:「我想给 X 加个 Y」。不用写得多正式——把它变成可审的 spec 是 Builder 的活。

## 第 2 步 · Builder 立工单

Builder 把你的想法写成一份**工单**(见 [`../templates/ticket.md`](../templates/ticket.md)):要做什么、改哪些文件、**不改什么**、怎么验证、验收标准。

为什么先立工单:**审查需要一个可对照的 spec**。没有 spec,Reviewer 只能审「代码对不对」,审不了「做的是不是你要的那件事」。工单就是后面每一轮 review 的标尺。

## 第 3 步 · 设计自审循环(代码之前)

Builder **先不写代码**,把方案 `codex exec` 给 Reviewer 审:

```bash
codex exec --sandbox read-only \
  "这是工单 + 设计方案。按 AGENTS.md 审:架构有什么坑、哪些边界没说清、哪些『轻描淡写』要展开。"
```

Reviewer 出裁决(通过 / 需改后通过 / 阻断)。**阻断就改方案、再审**,循环到放行。

> 这一步最值钱:**在没写一行代码前就把设计漏洞挑出来**,比写完再返工便宜得多。

## 第 4 步 · 向你提案(闸门一)

设计过了 Reviewer 这关,Builder 才来找你:「原方案是这样,我准备这么做,不动 X/Y/Z。」
**你说 OK,才落到代码。** 这是第一个人类闸门——机器之间审过了,但「要不要做、是不是你想要的」由你定。

## 第 5 步 · 落到代码

Builder 写代码。改动严格对着第 2 步的工单——**超出 spec 范围的改动要先回来说**,不能默默夹带。

## 第 6 步 · 代码自审循环

写完(本地测过)再把 diff `codex exec` 给 Reviewer:

```bash
git diff --staged | codex exec --sandbox read-only \
  "本次改动暂存 diff。按 AGENTS.md 审,重点:完成度声明有没有超出证据层级、有没有碰保护区、有没有偏离工单。"
```

Reviewer 裁决。**阻断 / 需改后通过就改,再审,循环到通过。**
Reviewer 的回执**要核查、不无脑同意**——Builder 觉得它判断有误,就摆证据二审 / 反驳,目的是把工程推对,不是过关。

## 第 7 步 · 向你汇报(闸门二)+ 收工

Builder 向你汇报:做了什么、核了什么、**哪些没核**(用[完成层级](../conventions/truth-hierarchy.md)说清,别把「代码写完」说成「跑通了」)。
**你说收工**,Builder 才写 **handoff**(见 [`../templates/handoff.md`](../templates/handoff.md))——交接给下一个会话:做到哪、为什么这么做、踩了什么坑、还差什么。

handoff **再过一次 Reviewer**(它最容易把「打算修的」写成「已修」、把没验证的写成闭环)。过了再合并。

---

## 留痕:裁决进 commit message,不写报告文件

每轮 review 的结论**不另存报告文件**——报告文件会像 handoff 一样越堆越多、还会漂。
审计轨迹靠 commit message:

```
feat: 给 X 加 Y

codex 回执:需改后通过 → 已按裁决补 Z 的边界校验,二审通过。
完成层级:代码 ✅ 测试 ✅ 已 commit ✅ / 运行时未验 / 用户体感未验。
```

git log 不可变、不漂移、零维护。只有刻意的里程碑分析(如大型重构 audit)才值得单独存一个文件。

---

## 什么时候回退到人类传话

内部调用是主线,但这几种情况 Builder 应该把裁决转给你、停下等你拍板:

- Reviewer 的裁决**有歧义**(同一条建议能有两种理解);
- 改动触到**架构级 / 不可逆 / 触红线**的东西;
- Builder 和 Reviewer **僵住了**(来回几轮没收敛)——别让两个 agent 死磕,交给人。

判断标准见 [`why-it-works.md`](why-it-works.md) 的「人类闸门」一节。
