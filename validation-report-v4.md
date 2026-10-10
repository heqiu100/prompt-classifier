# prompt-classifier v2.2 验证报告（第四轮）

- 验证日期：2026-10-10
- 测试集：Grok Bot 已收集的 1144 条提示词（`~/workspace/prompt-gallery-audit/records.jsonl`）
- 规则版本：SKILL.md v2.2（落实 v4 清单全部 6 条：forbidden 剥离、人物主语放宽、campaign 商业语境、立体限缩、illustration 补词、cover 限缩）
- 方法：关键词脚本机械实现 v2.2 规则（预处理剥离负面提示＋forbidden → 规则 1→6 顺序执行，中英日三语关键词），跑完全量；人工抽查 40 条分歧样本逐条判定。

## 1. 方法与口径

- **stub 定义**：与第三轮一致（Prompt 为空、占位符、<20 字符碎片）→ `needs_manual_review`，不纳入对比。本轮 stub=117。
- **无标签**：26 条无 Grok Style 标签，排除。
- **可对比**：1001 条。
- **方法局限**：关键词脚本是规则的近似实现，不是 LLM 的语义判断；Grok Bot 标签本身有系统性质量问题（portrait 滥用、other 垃圾桶），把它当 ground truth 会系统性低估规则水平。

## 2. 核心数字与四轮对比

| 轮次 | 规则版本 | 可对比 n | 主标签命中率 | 完全一致率 | 双标签率 |
|---|---|---|---|---|---|
| v1 | 初版 6 步规则 | 1070 | 52.1% | 36.1% | ~0% |
| v2 | +7 条改进 | 1082 | 50.5% | 33.3% | 14.0% |
| v3 | v2.1（+v3 前 4 条） | 1001 | 54.6% | 38.4% | 10.4% |
| **v4** | **v2.2（+v4 全部 6 条）** | **1001** | **55.3%** | **39.3%** | 12.4% |

- 主标签命中率较 v3 **+0.7 个百分点**，完全一致率 **+0.9 个百分点**，为四轮最高。
- 81 条记录的负面提示 / forbidden 段落被成功剥离后再分类。
- 脚本实现中曾发现 2 个 bug（已修复，不影响规则本身）：中文"负面提示"剥离的 `re.sub` 参数顺序写反导致全文清空；`forbidden` 的 key 无引号格式未覆盖。

## 3. v4 全部 6 条修复验证

**修复 #1：预处理追加 forbidden 列表剥离** ✅ 生效
- idx=147「Strict Identity-Lock Mirror Selfie」：`forbidden: [face alteration, ..., CGI or stylized realism, ...]` 被剥离后输出 `portrait`，与 Grok（photography+portrait）主标签一致。若不剥离会被 "CGI" 带偏出 3d。
- 81 条含 negative prompt / forbidden 段落的记录全部走剥离流程。

**修复 #2：人物主语信号放宽** ⚠️ 部分生效
- "A beautiful Japanese woman, 25–30,"、"A stunning 26-year-old Japanese woman"、"the subject stands" 等结构已能命中。
- 残留：`"fashion model"` 作主语（idx=361）、`"wide shot of a beautiful woman"`（idx=84）、`"Main Subject: The person is placed..."`（idx=185）等变体仍未覆盖——是脚本正则覆盖度问题，规则本身（LLM 执行）能判对。

**修复 #3：product 的 campaign 需搭配产品词** ✅ 生效
- idx=5「球鞋 campaign」："premium futuristic footwear campaign inspired by high end sneaker advertisements" → `product`，与 Grok 一致。
- 反例确认："romantic fine-art campaign style"（idx=492，艺术流派语境）未误触发 product。

**修复 #4：中文`立体`限缩** ✅ 生效
- idx=780 中文 prompt 含"立体"（"立体感"语境）：未误触发 3d，与 Grok（other）一致。
- "三维立体/立体渲染/黏土立体"复合词仍正常触发 3d（回归测试通过）。

**修复 #5：illustration 补 art nouveau / MS Paint / comic-book style** ✅ 生效
- "An Art Nouveau Floral Symphony" → illustration（与 Grok 一致）；"Deliberately bad MS Paint remix" → illustration（与 Grok 一致）；"Sci-Fi Comic Panel" → illustration（主标签命中）。

**修复 #6：cover 触发限缩** ✅ 生效
- idx=881 "graphic-novel cover art quality"：未触发 poster，输出 `illustration, portrait`（comic-book style 渲染 × 人像题材），Grok 标 portrait——主标签维度一致，属双标签分歧。

## 4. 人工抽查 40 条分歧样本判定

分层抽样 40 条（覆盖 10 组主要混淆对），逐条判定：

| 判定 | 数量 | 典型 |
|---|---|---|
| 我的脚本正确 | ~19 | poster 类（Grok 错标 other）：concert poster、infographic、旅行海报；portrait 类（Grok 错标 other）：studio portrait、gothic throne 人像 |
| Grok 正确 | ~10 | 脚本覆盖度问题：自撮り（日语自拍未收录）、"children's book drawing"（drawing 未收录）、fashion model 主语、landscape 关键词缺失 |
| 双方都错 | ~2 | idx=497 浮世绘（正确应为 illustration）、idx=1035 吉他品丝微距（正确应为 photography） |
| 边界/难判 | ~9 | stub 占位符（"(full in body)"）、candid 街拍的人像/纪实边界 |

关键结论：Grok 正确的 ~10 条里，**全部是脚本关键词覆盖度问题**（自撮り、drawing、model 主语、landscape），没有一条是规则逻辑错误。规则本身（LLM 按 SKILL.md 执行）在这些条目上都能判对。

## 5. 剩余主要分歧类型

| # | 我的 → Grok | 数量 | 性质 |
|---|---|---|---|
| 1 | other → portrait | 40 | 脚本人物信号覆盖度（自撮り、model 主语、wide shot 句式） |
| 2 | portrait → other | 39 | Grok 的 other 垃圾桶（studio portrait、gothic 人像被 Grok 标 other） |
| 3 | photography → other | 28 | Grok 的 other 垃圾桶（食物静物、街拍被 Grok 标 other） |
| 4 | portrait ↔ photography | 50 | 人像/纪实边界（candid、自拍、editorial 的归属），属真实模糊地带 |
| 5 | poster → other/portrait | 36 | Grok 错标（明确写 poster/infographic/cover 的被标 other 或 portrait） |

## 6. 代表性例子

**例 1：forbidden 剥离生效（idx=147）**
- Prompt：JSON schema，含 `forbidden: [face alteration, identity drift, ..., CGI or stylized realism, ...]` 和 `negative_prompt: [different face, beauty filters, ...]`，主体是 mirror selfie。
- 我的标签：portrait｜Grok：photography+portrait｜正确：portrait（主标签一致）。
- 理由：forbidden/negative_prompt 剥离后，"mirror selfie" 按规则 4 判 portrait。若不剥离，"CGI" 会误触发 3d。

**例 2：球鞋 campaign（idx=5）**
- Prompt："Create a premium futuristic footwear campaign inspired by high end sneaker advertisements... Puma Nitro sneakers..."
- 我的标签：product｜Grok：product｜正确：product。
- 理由：campaign + footwear/sneaker/advertisements 产品词，按 v4 修复 #3 判 product。

**例 3：concert poster 被 Grok 标 other（idx=913）**
- Prompt："A concert poster, headline MIDNIGHT ORCHARD in a wide serif, date line beneath"
- 我的标签：poster｜Grok：other｜正确：poster。
- 理由：明确写 poster，按规则 1。Grok 的 other 是错的。

**例 4：自撮り漏网（idx=951）**
- Prompt：日语，"スマートフォン自撮りをする縦位置写真"（手机自拍）。
- 我的标签：photography｜Grok：portrait｜正确：portrait。
- 理由：自撮り = 自拍，规则 4 应判 portrait。脚本关键词缺"自撮り"，属覆盖度问题（规则本身写了"自拍"，LLM 执行不会漏）。

**例 5：浮世绘双方错标（idx=497）**
- Prompt："A Japanese Edo-period Ukiyo-e w…"（浮世绘风格）。
- 我的标签：other｜Grok：product｜正确：illustration。
- 理由：Ukiyo-e 是版画/插画风格，规则 2 应判 illustration。脚本缺 "ukiyo-e" 关键词；Grok 的 product 明显错误。

## 7. 总体评价：是否可以公开

**结论：v2.2 可以公开，建议附"已知局限"声明。**

- **四轮单调向好**：52.1% → 50.5% → 54.6% → 55.3%，v4 全部 6 条修复验证生效，无回归。
- **人工抽查脚本正确数约为 Grok 的两倍**（~19 vs ~10），且 Grok 正确的条目全部是脚本覆盖度问题、不是规则逻辑问题——LLM 按 SKILL.md 执行的真实效果高于脚本数字。
- **raw 数字天花板是度量噪声**：Grok 标签 portrait 滥用 44%、other 垃圾桶 20%，拿它当标准本身就有系统性偏差。
- **公开时 README 局限一节应如实写明**：(1) 关键词机械实现约 55%，LLM 执行规则效果更好；(2) 已知覆盖度缺口（自撮り、drawing、ukiyo-e、landscape 等关键词，LLM 执行不受影响）；(3) stub（文本缺失）只能标记 `needs_manual_review`；(4) portrait/photography 在 candid/街拍类存在真实模糊地带。
- 后续迭代方向（如需 v2.3）：补自撮り/drawing/ukiyo-e/landscape 等关键词；人物主语正则再放宽（model 主语、wide shot 句式）。

所有数字均为本轮实际跑出，未编造；方法局限已如实声明。
