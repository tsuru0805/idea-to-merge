# 走一遍:给一个记忆存储加「按时间过滤召回」

这是个**完全虚构**的小项目,用来把[七步循环](../docs/workflow.md)走一遍。
项目假设:一个简单的「记忆存储」服务,把文本片段存进 SQLite,提供一个 `recall(query)` 做关键词召回。
现在要给它加一个功能:**召回时可以按时间范围过滤**。

> 角色:**Builder** = Claude Code,**Reviewer** = Codex(`codex exec` 调用),**你** = 维护者。

---

## 第 1 步 · 你提想法

> 「recall 现在只能全量搜,我想能限定『只搜最近 7 天』这种时间范围。」

---

## 第 2 步 · Builder 立工单

```markdown
# 工单 · recall 支持时间范围过滤

## 要做什么
recall(query, since=None, until=None):可选传时间范围,只召回 created_at 落在范围内的片段。

## 改哪些文件
- store.py — recall() 加 since/until 参数 + SQL where 条件
- tests/test_recall.py — 加时间过滤的用例

## ⚠️ 不改什么
- 不动 write() 写入路径
- 不动现有 recall(query) 无参调用的行为(向后兼容)

## 方案
recall 现有 SQL: SELECT ... WHERE text LIKE ?
加上: AND (created_at >= ? OR ? IS NULL) AND (created_at <= ? OR ? IS NULL)

## 怎么验证
- [ ] 无参 recall 行为不变(回归)
- [ ] since 过滤掉更早的片段
- [ ] until 过滤掉更晚的片段

## 验收标准
代码 + 测试通过 + 已 commit。运行时验证由你决定要不要做。
```

---

## 第 3 步 · 设计自审循环

Builder 先不写代码,把工单 + 方案 `codex exec` 给 Reviewer:

```bash
codex exec --sandbox read-only \
  "这是工单和方案(见上)。按 AGENTS.md 审设计:有什么坑、边界有没有说清。" < /dev/null
```

Reviewer 回:

```
## 裁决:需改后通过

## 最关键的一条
SQL 里的 created_at 字段当前存的是什么类型?方案默认能直接和传入的时间比较,但没说清格式。

## 阻断项
(无硬阻断,但下面一条不澄清会埋 bug)

## 可放行项
- 向后兼容用 "OR ? IS NULL" 的写法是对的,无参调用行为不变。

## 验证层级修正
- 方案目前是「设计存在」,尚无任何实现/测试证据。

## 下一步最小动作
确认 created_at 的存储格式(ISO 字符串 / Unix 时间戳),并在工单里写明 since/until 传入的预期格式,
否则字符串和时间戳混比会静默错。
```

Builder 核了一下 `store.py`,发现 `created_at` 存的是 ISO8601 字符串。于是在工单方案里补一句
「since/until 接受 ISO8601 字符串,和 created_at 同格式直接字典序比较」,再 `codex exec` 一次 → **通过**。

> 看到了吗:**一行代码还没写,一个会静默出错的格式 bug 已经被挡下来了。**

---

## 第 4 步 · 向你提案(闸门一)

Builder:「方案过了设计审。recall 加可选 since/until(ISO 字符串),向后兼容,不动写入。准备开写?」
你:「OK。」

---

## 第 5 步 · 落到代码

Builder 在隔离分支 `feature/recall-time-filter` 上改 `store.py` 和测试。本地跑测试通过。

---

## 第 6 步 · 代码自审循环

```bash
git add store.py tests/test_recall.py        # 只 add 本次改动的文件
git diff --staged | codex exec --sandbox read-only \
  "本次暂存 diff。按 AGENTS.md 审,重点:完成度声明、是否偏离工单、边界。"
```

Reviewer 回:

```
## 裁决:阻断

## 最关键的一条
新测试只覆盖了 since 和 until 单独传的情况,工单要求的「无参回归」用例没写——
向后兼容这条最该测的恰恰没测。

## 阻断项
- tests/test_recall.py:无 recall(query) 无参的回归用例。工单「不改什么」第二条专门点了向后兼容,缺测 = 不能放行。
- store.py:42 — until 的边界是 created_at <= ?,确认是要含端点(闭区间)?工单没说,代码默默选了闭区间。

## 可放行项
- since/until 的 SQL 写法正确,OR ? IS NULL 向后兼容成立。

## 验证层级修正
- 当前:代码存在 ✅ / 测试通过 ⚠️(覆盖不全)/ 已 commit ❌
- 不能声明「测试通过」——回归路径没覆盖。

## 下一步最小动作
补无参回归用例;在工单注明端点是闭区间(或按需求改成开区间)。

## 本轮命令清单
- git diff --staged
- sed -n '30,60p' store.py
- cat tests/test_recall.py
```

Builder 补了无参回归用例、在工单注明闭区间,`codex exec` 再审 → **通过**。
然后 commit,把裁决写进 message:

```
feat(store): recall 支持 since/until 时间范围过滤

向后兼容(无参行为不变),since/until 接受 ISO8601 字符串。

codex 回执:首审阻断(缺无参回归用例 + 端点语义未定),
已补回归用例、注明闭区间,二审通过。
完成层级:代码 ✅ 测试 ✅ 已 commit ✅ / 运行时未验。
```

---

## 第 7 步 · 向你汇报(闸门二)+ handoff

Builder:「recall 时间过滤做完了。完成层级:代码 ✅ 测试 ✅ 已 commit ✅,运行时**没验**(没起服务真跑一次)。要收工吗?」
你:「收工。」

Builder 走收工 checklist:写 handoff(见 [`../templates/handoff.md`](../templates/handoff.md))、更新工单状态、刷看板,
然后把这批收工文档整批再过一次 Reviewer——它会盯「有没有把『没验运行时』悄悄写成『已上线』」。过了,合并 `feature/recall-time-filter`。

---

## 这一遍挡住了什么

- **设计阶段**:一个 ISO 字符串 / 时间戳混比的静默 bug(代码还没写就挡下)。
- **代码阶段**:工单明确要的向后兼容回归用例漏写;一个没人定义过的「端点闭/开区间」语义。
- **全程**:完成度声明被逐层核对,「测试通过」在覆盖不全时被打回,「运行时没验」被如实说出。

没有一个洞是「Claude 写不出好代码」——它们是**自审挑不出、异构审查才挑得出**的那一类。这就是整套工作流的意义。
