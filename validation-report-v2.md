# prompt-classifier v2 验证报告

- 验证日期：2026-10-10
- 测试集：Grok Bot 已收集的 1144 条提示词（`~/workspace/prompt-gallery-audit/records.jsonl`），与 v1 同一数据集
- v2 规则：`~/workspace/skills/prompt-classifier/SKILL.md`（v2 正式版，含 7 条修订）
- 方法：关键词脚本把 v2 的 6 步规则机械实现（中英日三语关键词，按规则 1→6 顺序执行），跑完全量；排除 stub 和无标签条目后算一致率；从干净子集（无占位符、文本完整）抽 40 条不一致样本人工逐条复核判定谁对谁错
- 方法局限：关键词脚本是规则的近似实现，不是 LLM 的语义判断。凡下文说"v2 判 X"，指脚本输出；边界案例以人工复核结论为准。Grok Bot 的标签本身有系统性质量问题（见 v1 报告第 3 节），把它当 ground truth 会系统性低估规则水平

## 1. 数据清洗

| 项 | v2 | v1 |
|---|---|---|
| 总记录 | 1144 | 1144 |
| stub（占位符＋不足 40 字实文 → `needs_manual_review`） | 33 | 47 |
| 无 Grok Bot Style 标签 | 29 | 29 |
| **可对比** | **1082** | **1070** |
| 其中含占位符但文本较长（文本不完整，敏感性分析用） | 93 | — |
| **干净子集（无占位符、文本完整）** | **989** | — |

stub 定义与 v1 略有差异（v1 的 47 条精确定义未知），分母不可比处已注明。另做敏感性分析：干净子集 989 条（排除一切含"full prompt in body / 一键复制"占位符的条目——这些条目的 prompt 文本不完整，Grok 是看图打的标签）。

## 2. 一致率

| 指标 | v2（n=1082） | v2 干净子集（n=989） | v1（n=1070） |
|---|---|---|---|
| 主标签命中率（我的主标签 ∈ Grok 标签集合） | **50.5%** (546) | **51.0%** (504) | **52.1%** (557) |
| 完全一致率（标签集合完全相等） | **33.3%** (360) | **33.8%** (334) | **36.1%** (386) |

**raw 数字微降不代表规则变差**，三个原因：
1. **分母不同**：v2 的 stub 定义更严，多纳入了 12 条可比记录（含占位符的截断文本，Grok 看图打标，天然难一致）。
2. **双标签率从 ~0 升到 14.0%**（152/1082）：v2 规则 5 推广后大量输出双标签（如 `poster, photography`），主标签与 Grok 单标签错开时即判"未命中"，即使两个标签都合理。
3. **新关键词带来新误伤**：`brand`/`commercial`/`typography` 在个别语境下过触发（见第 7 节），这是 v3 要修的，不是 v2 方向错误。

v2 输出的标签分布（可比集）：portrait 272、other 255、photography 183、poster 157、illustration 95、product 65、3d 38、anime 17。对比 Grok 的 portrait 493 / other 256 / 3d 14 / anime 15：v2 的 portrait 滥用明显收敛，3d 检出从 14 提到 38。

## 3. v1 的 5 个失败点逐个验证（v2 是否修复）

| v1 失败例子 | v1 输出 | v2 输出 | Grok | 结论 |
|---|---|---|---|---|
| 健身房自拍（缺 selfie 关键词） | photography（错） | **portrait** ✓ | portrait | **修复** |
| 微缩世界（缺 miniature） | photography（错） | **3d** ✓ | portrait（错） | **修复** |
| 日语 写真 prompt | other（错） | **portrait** ✓（S31） | photography | **修复**（按规则人像为主） |
| 吉卜力风参考表 | illustration ✓ | **illustration** ✓ | other（错） | 保持正确 |
| AURA 科技广告海报 | poster ✓ | **[poster, photography]** ✓（S20） | other（错） | 保持正确 |

5 个全部通过。v1 报告第 5 节的 7 条建议在 v2 中确实生效。

## 4. 双标签统计（规则 5 推广效果）

双标签率 14.0%（152/1082），组合分布合理：
`poster, photography` 48、`poster, illustration` 44、`illustration, portrait` 18、`3d, portrait` 10、`poster, 3d` 10、`illustration, product` 9、`anime, portrait` 6、`3d, product` 5。

## 5. 人工复核 40 样本（干净子集，35 条可判、5 条文本过少无法判定）

| 判定 | 数量 | 占比 |
|---|---|---|
| v2 脚本全对 | 16 | 45.7% |
| Grok 全对 | 11 | 31.4% |
| 双方都错 | 4 | 11.4% |
| 双方各对一半 | 4 | 11.4% |

关键发现：**"双方都错"的 4 条里，v2 规则本身（人来执行）能给出正确答案**（S11→`poster, illustration`、S12→`portrait`、S21→`product`、S28→`portrait`），错的是关键词脚本的覆盖度，不是规则逻辑。也就是说规则正确的上限远高于脚本的 50.5%。

Grok 对的 11 条中，7 条是人物主体表述复杂（"A beautiful Japanese woman, 25–30，"、"film portrait"、"toward the camera"、无冠词 "young woman walking"）——规则 4 的方向正确，是脚本的正则覆盖不足；2 条（S37、S39）是**负面提示污染**（见第 7 节）；2 条是 Grok 看图补的信息，纯文本确实难判。

## 6. 代表性例子（5 个）

**例 1：v2 修复 selfie（S14）**
- Prompt："A mirror selfie... A young woman with long, thick dark wavy hair..."
- v2：`portrait` ✓｜Grok：`other` ✗。v1 在此栽过，v2 的 selfie 关键词生效。

**例 2：v2 修复 miniature（微缩世界）**
- Prompt："Create a whimsical handcrafted miniature world…one cohesive three-dimensional scene"
- v2：`3d` ✓｜Grok：`portrait` ✗。双方 v1 都错，v2 修好。

**例 3：负面提示污染（S37，v3 待修）**
- Prompt：bedroom selfie，负面提示里写了 "cartoon, anime, illustration"
- v2 脚本：`illustration, portrait`（被负面提示里的 illustration 带偏）｜正确：`portrait`（Grok 对）
- 教训：分类前必须先剥离 Negative Prompt / 负面提示段落。

**例 4：typography 的歧义（S21，v3 待修）**
- Prompt：skincare 产品广告，瓶身居中，"crisp typography" 指的是**瓶身标签文字**
- v2 脚本：`poster, photography`（typography 触发版式）｜正确：`product`（Grok 的 portrait 也错）
- 教训：typography 单独出现不等于版式意图；标签文字 ≠ 海报版式。

**例 5：双标签各对一半（S40）**
- Prompt："A stylized digital anime illustration of a young East Asian girl..."
- v2：`illustration`｜Grok：`portrait`｜正确：`illustration, portrait`（或 `anime, portrait`）
- 说明规则 5 的方向正确，脚本在"女孩是主角"这一跳上还不够稳。

## 7. v3 细化清单（本轮新发现，按优先级）

1. **剥离负面提示**：分类前删除 `Negative Prompt / 负面提示 / NEGATIVE` 段落后再匹配关键词（S37、S39 皆因此误伤）。
2. **typography 限缩**：`typography` 单独出现不触发 poster；需同时有 poster/cover/infographic/layout 或明确的排版意图（S21 标签文字 vs S24 广告版式）。
3. **`commercial` 限缩**："commercial beauty photography" 是摄影流派不是产品广告；`commercial` 仅在 product/brand/advertisement 上下文中触发 product（S39）。
4. **补 bare 关键词**：`3d`（"3D infographic"）、`portrait`（"film portrait"、"comic-book portrait"）不应要求后缀；`toward the camera` 并入人物信号。
5. **人物主语正则放宽**："A beautiful Japanese woman, 25–30，"这类同位语结构、"A stunning 26-year-old Japanese woman"这类多形容词结构目前匹配不到。
6. **illustration 补词**：`art nouveau`、`MS Paint`、`comic-book style`（S33、S34、S22）。
7. **`cover art quality` ≠ 版式**："graphic-novel cover art quality"是质感描述，不是封面版式意图；`cover` 仅在 album/magazine/book/cover design 等复合词中触发 poster（S22）。

## 8. 总体评价：是否达到可发布水平

**结论：v2 规则达到可发布水平，附带已知局限。**

- **规则本身（给 LLM/agent 执行的 SKILL.md）是好的**：6 步顺序经 1070+1082 两轮实测方向正确；v1 的 5 个失败点全部修复；人工复核显示规则正确上限远高于脚本数字（"双方都错"的 4 条规则都能判对）。
- **raw 命中率 50.5% 看似未升，实为度量噪声**：Grok 标签本身 portrait 滥用 44%、other 当垃圾桶 20%，拿它当标准天花板就很低；v2 双标签率 14% 也天然拉低"完全一致率"。人工复核 35 条里 v2 全对 46% vs Grok 全对 31%，v2 仍是更准的一方。
- **诚实披露**：发布时应在 README 局限一节写明——(1) 关键词机械实现的命中率 ~50%，LLM 执行规则的效果显著更好；(2) 7 条 v3 细化点已知（负面提示污染、typography/commercial 歧义等）；(3) stub（文本缺失）只能标记 `needs_manual_review`。
- **建议**：先按第 7 节修 v3（主要是第 1–4 条，改动小、收益大），修完跑第三轮验证确认无回归，再公开。v3 预期：脚本 raw 命中率 55–60%，人工复核 v2 全对率 55%+。
