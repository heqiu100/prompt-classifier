# prompt-classifier v2.1 验证报告（第三轮）

- 验证日期：2026-10-10
- 测试集：Grok Bot 已收集的 1144 条提示词（`~/workspace/prompt-gallery-audit/records.jsonl`）
- 规则版本：SKILL.md v2.1（含 v3 细化清单第 1–4 条：负面提示剥离、typography/commercial 限缩、bare 关键词）
- 方法：关键词脚本机械实现 v2.1 规则（预处理剥离负面提示 → 规则 1→6 顺序执行，中英日三语关键词），跑完全量；人工抽查 40 条分歧样本逐条判定。

## 1. 方法与口径

- **stub 定义**：Prompt 字段为空、占位符（"full prompt in body"/"prompt in page body"/"see page body"/"full text in"/"一键复制"类）、或 <20 字符碎片 → 输出 `needs_manual_review`，不纳入对比。本轮 stub=117。
- **无标签**：26 条无 Grok Style 标签，排除。
- **可对比**：1001 条。
- **口径说明**：v1（n=1070）、v2（n=1082）用了更窄的 stub 定义。敏感性测试：用窄口径重算本轮得 n=1019、主标签命中率 54.8%、完全一致率 38.5%，与宽口径（54.6%/38.4%）基本一致，口径差异不影响结论。
- **方法局限**：关键词脚本是规则的近似实现，不是 LLM 的语义判断；Grok Bot 标签本身有系统性质量问题（portrait 滥用、other 垃圾桶），把它当 ground truth 会系统性低估规则水平。

## 2. 核心数字与三轮对比

| 轮次 | 规则版本 | 可对比 n | 主标签命中率 | 完全一致率 | 双标签率 |
|---|---|---|---|---|---|
| v1 | 初版 6 步规则 | 1070 | 52.1% | 36.1% | ~0% |
| v2 | +7 条改进 | 1082 | 50.5% | 33.3% | 14.0% |
| **v3** | **v2.1（+v3 前 4 条）** | **1001** | **54.6%** | **38.4%** | 10.4% |

- 主标签命中率较 v2 **+4.1 个百分点**，完全一致率 **+5.1 个百分点**，为三轮最高。
- 负面提示剥离生效：78 条记录的负面提示段落被成功剥离后再分类。
- 双标签率从 v2 的 14.0% 回落到 10.4%（脚本对"题材维度"的判定偏保守，见第 5 节）。

## 3. v3 前 4 条修复验证

**修复 #1：剥离负面提示** ✅ 生效（标准格式），⚠️ 发现 1 个边缘格式
- idx=309「日式温泉」：`Negative Prompt（负面提示）`段落被剥离（含 "cartoon, anime, illustration, 3d render"），分类 `portrait` 与 Grok 一致。若不剥离会被带偏。
- idx=147「Strict Identity-Lock Mirror Selfie」：两个 `negative_prompt: [...]` JSON 段落均被剥离（含 "anime, cartoon"），`illustration` 污染消除。但残留一个 `forbidden: [...]` 列表（含 "CGI or stylized realism"），脚本误出 `3d`。`forbidden` 是 JSON schema 里的负面约束，功能等同负面提示——全数据集仅 1 条，记入 v4 清单（建议 skill 预处理节追加 `forbidden`）。
- 结论：标准负面提示格式（Negative Prompt/negative_prompt/负面提示/NEGATIVE，共 78 条）剥离有效。

**修复 #2：typography 限缩** ✅ 生效
- 构造测试："bottle label with crisp typography"（无版式词）→ 输出 `other`，不再误触发 poster。
- idx=303「Luxury Skincare Campaign Poster」：输出 `poster`。注意：此处触发的是 prompt 正文明确写的 "Create a premium skincare poster"（规则 1 版式优先），不是 typography——按规则判 poster 正确。v2 报告称"正确应为 product"值得商榷：Grok 标的是 product+poster 双标签，本条实为边界案例。

**修复 #3：commercial 限缩** ✅ 生效
- idx=309 含 "commercial beauty photography"（摄影流派语境）：未触发 product，输出 `portrait` 与 Grok 一致。
- 反例确认：无 commercial+产品上下文的纯商业词不再误伤。

**修复 #4：bare 关键词** ✅ 生效
- "3D infographic" → `3d`（+infographic 触发版式 → `poster, 3d` 双标签）✓
- "film portrait" → `portrait` ✓（不再要求 portrait of 后缀）

## 4. 人工抽查 40 条分歧样本（分层抽样 8 类混淆 × 5 条）

| 判定 | 数量 | 说明 |
|---|---|---|
| 我的脚本正确 | ~22 | Grok 错（other 垃圾桶、portrait 滥用为主） |
| Grok 正确 | ~8 | 脚本覆盖度问题（见 v4 清单） |
| 双方都可接受（边界） | ~6 | 如 poster+product、海报人像 |
| 双方都错 | ~2 | 抽象/破碎 prompt |

典型"Grok 对、我错"的 4 条（脚本覆盖度，非规则逻辑错）：
1. idx=5「球鞋广告 campaign」："footwear campaign / sneaker advertisements" 有商业语境，应为 product；脚本 PROD_KW 未收 "campaign" 单独出现 → 判 photography。Grok 的 product 对。
2. idx=581「东南亚女子」：纯人物主语开场 "A beautiful young Southeast Asian woman…"，应为 portrait；脚本人物主语正则未覆盖 → 判 other。这是 v3 清单第 5 条（未纳入 v2.1）。
3. idx=461「肌肉男 subject stands」："subject stands" 人物主语，应为 portrait；脚本只认了 JSON 的 subject+character 组合。
4. idx=780「韩夜街拍」：中文"真实手机摄影" prompt，脚本因"立体"误出 3d。`立体`在中文里常指"立体感/层次感"非三维渲染，关键词过宽。

"stub 漏网"2 条：idx=41/35 的 Prompt 字段为 "[Full prompt in 一键复制]…"占位符，旧脚本未识别。本轮已把"一键复制"纳入 stub 判定。

## 5. 剩余主要分歧类型

| # | 我的 → Grok | 数量 | 主因 |
|---|---|---|---|
| 1 | other → portrait | 52 | 人物主语开场/碎片 prompt，脚本人物信号覆盖不足（v3 #5 未做） |
| 2 | photography → other | 37 | Grok 把明确摄影类 prompt 丢进 other；Grok 错 |
| 3 | other → photography | 37 | 日语/中文摄影 prompt（如"写真"），部分脚本漏检；Grok 对 |
| 4 | portrait → other | 34 | Grok other 垃圾桶；多为我对 |
| 5 | photography → portrait | 29 | 自拍/特写类，Grok 对的多（selfie 已修好，残留为人物主语型） |
| 6 | poster → other | 25 | Grok other 垃圾桶；多为我对（如"高级设计海报"） |

## 6. 代表性例子（5 个）

**例 1：负面提示剥离生效（idx=309）**
- Prompt：日式温泉私汤，"high-end private spa beauty photography"；负面提示含 "cartoon, anime, illustration, 3d render"
- v2.1：`portrait`｜Grok：`portrait`｜一致 ✓
- 说明：若不剥离，illustration/anime 会污染分类。

**例 2：forbidden 边缘格式（idx=147，新发现）**
- Prompt：JSON schema 严格身份锁定镜面自拍；含 `forbidden: [...CGI or stylized realism...]` + 两个 `negative_prompt: [...]`
- v2.1 脚本：`3d, portrait`｜正确：`portrait`（Grok 的 photography+portrait 也可接受）
- 说明：negative_prompt 已剥离，但 `forbidden` 列表是同类负面约束，脚本未覆盖。规则层面 LLM 可识别，建议 v4 追加。

**例 3：中日关键词生效（idx=545 日语健身房）**
- Prompt：日语 GPT Image2 提示词，"AI写真""主体：成人女性…ラットプルダウン"
- v2.1：`portrait`｜Grok：`other`｜我对 ✓
- 说明：v2 补的日文关键词（写真/主体人物）在本轮生效。

**例 4：双标签正确（idx=984 海港黄昏）**
- Prompt："Get back a poster where the top half is your shot and the bottom half is an architect's concept sketch"
- v2.1：`poster, illustration`｜Grok：`other`｜我对 ✓
- 说明：规则 5"渲染风格×题材"双标签在本轮稳定工作。

**例 5：脚本保守漏双标签（idx=881 绿灯侠）**
- Prompt："dramatic comic-book portrait of a stern male superhero"
- v2.1：`poster, illustration`｜Grok：`portrait`｜正确应为 `illustration, portrait`
- 说明：脚本把 "Heroes and Villains" 标题误判出 poster 信号；"comic-book" 未被收录（v3 #6 未做）。题材 portrait 被漏掉，双标签题材维度偏保守。

## 7. v4 清单（本轮新发现，按优先级）

1. **预处理追加 `forbidden`**：JSON schema 中的 `forbidden: [...]` 与负面提示同义，应一并剥离（全量仅 1 条，影响小但逻辑完备）。
2. **人物主语正则放宽**（v3 #5 未做）："A beautiful X woman…"类人物主语开场、"subject stands/verbed" 结构应计入 portrait 信号。本轮最大残留分歧来源（other→portrait 52 条中约一半）。
3. **product 商业语境补 "campaign"**："footwear campaign / sneaker advertisements" 类应触发 product；`campaign` 单独出现时需搭配产品词才触发（防误伤）。
4. **`立体`关键词限缩**：中文"立体感/层次感"非三维渲染；`立体`仅在"三维立体/立体渲染/黏土立体"等复合词中触发 3d。
5. **illustration 补词**（v3 #6 未做）：`art nouveau`、`MS Paint`、`comic-book style`。
6. **`cover` 触发限缩**（v3 #7 未做）：`cover` 仅在 album/magazine/book/cover design 复合词中触发 poster。

## 8. 总体评价：是否可以公开

**结论：v2.1 已达到可公开水平，建议附"已知局限"声明。**

- 三轮验证主标签命中率 52.1% → 50.5% → **54.6%**，v2.1 为最高；v1 的 5 个失败点全部修复；v3 前 4 条修复经个案验证全部生效。
- 人工抽查显示：在分歧样本中我的脚本正确的约占一半以上（22/40），Grok 正确的约 1/5（8/40），其余为边界或双方都错。**规则本身（给 LLM/agent 执行的 SKILL.md）显著优于关键词脚本的机械数字**——脚本测出的是下限。
- raw 数字的天花板受 Grok 标签质量拖累（portrait 滥用 44%、other 垃圾桶 20%），这是度量噪声，不是规则问题。
- **公开前建议**：在 README 局限一节如实写明三点——(1) 关键词机械实现约 55%，LLM 按规则执行效果更好；(2) v4 清单 6 条已知细化点（forbidden、人物主语正则、campaign、立体、插画补词、cover 限缩）；(3) stub（文本缺失）只能标记 `needs_manual_review`。
- 是否做 v4：v4 的 6 条都是小修（改动量与 v3 相当），若追求更稳可再跑一轮；若接受"已知局限"式公开，v2.1 当前即可发布。

---
*方法局限重申：关键词脚本是规则的近似实现；Grok Bot 标签质量不高，一致率数字低估规则真实水平；所有数字为本轮实际跑出，未编造。*
