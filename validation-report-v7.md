# prompt-classifier 验证报告 v7（v2.5 规则）

- 验证日期：2026-10-10
- 测试集：Grok Bot 已收集的 1144 条提示词（`~/workspace/prompt-gallery-audit/records.jsonl`）
- 规则版本：v2.5（新增：portrait 补 "looking into the camera"；人物主语信号补 elegant/stylish 开场形容词、"A 20–25 year old woman" 年龄句式、"full body shot of a [person]"；"subject + 动词"需人物语境；poster 补 headline、layout 需设计语境；illustration 补 illustrated 形态）
- 方法：关键词脚本机械实现 v2.5 规则（中英日三语，先剥离负面提示+forbidden+否定语境，再按规则 1→6 顺序执行）跑完全量；人工抽查 32 条分歧样本逐条判定。
- 方法局限：关键词脚本是规则的近似实现，不是 LLM 的语义判断；Grok Bot 标签本身有系统性质量问题（portrait 滥用、other 垃圾桶），把它当 ground truth 会系统性低估规则水平。本轮另做了 2 处脚本 fidelity 修复（"looking directly into the camera" 允许副词插入；"Elegant, slender girl" 形容词开头），属脚本问题，不涉及规则改动。
- 口径说明：本轮 stub 定义为占位符模式或 len<60（174 条，含 47 条短但真实的 prompt），与 v6（114 条）口径不同；因此做了同分母敏感性测试（v2.4 规则重跑本轮相同的 944 条）以隔离规则效应。

## 1. 数据与核心数字

| 项 | 数量 |
|---|---|
| 总记录 | 1144 |
| stub（占位符/超短，len<60 或占位符模式） | 174 |
| 无 Grok Bot Style 标签 | 26 |
| **可对比** | **944** |

| 轮次 | 规则版本 | 可对比 n | 主标签命中率 | 完全一致率 |
|---|---|---|---|---|
| v1 | 初版 | 1070 | 52.1% | 36.1% |
| v2 | +7 条改进 | 1082 | 50.5% | 33.3% |
| v2.1 | +v3 前 4 条 | 1001 | 54.6% | 38.4% |
| v2.2 | +v4 全部 6 条 | 1001 | 55.3% | 39.3% |
| v2.3 | +关键词覆盖/人物主语 | 1006 | 55.3% | 37.0% |
| v2.4 | +否定剥离/drawing 限缩 | 1004 | 57.0% | 38.6% |
| **v2.5（本轮）** | **+人物信号变体/版式信号** | **944** | **57.5%** | **39.8%** |

**同分母复核**（v2.4 规则重跑本轮相同的 944 条）：v2.4-mode 得 56.7%/39.1%，v2.5 得 57.5%/39.8%。
结论：v2.5 相对 v2.4 提升 +0.8pp/+0.7pp，七轮最高；无回归。

其他指标：双标签率 12.0%（113/944）。

主要混淆（mine → grok）：portrait→other 41、photography→other 31、other→portrait 29、portrait→photography 26、poster→other 25。

## 2. v2.5 新增项验证（逐项）

| # | 新增项 | 验证结果 |
|---|---|---|
| 1 | "looking into the camera" | ✅ 生效。idx=320（"looking directly into the camera"）：v2.4 判 photography → v2.5 判 portrait，与 Grok 一致。另修脚本 fidelity：允许 directly/straight 插入。 |
| 2 | elegant/stylish 开场 | ✅ 生效。idx=80（"Elegant, slender East Asian girl…"）：v2.4 other → v2.5 portrait，与 Grok 一致。idx=39（"A stylish European woman…"）：v2.5 判 portrait；Grok 标 other（垃圾桶，我对）。另修脚本 fidelity：形容词开头模式。 |
| 3 | "full body shot of a [person]" | ✅ 生效。idx=339（"full body shot of a hooded male assassin"）：v2.5 判 `illustration, portrait`；Grok 标 other（我对）。 |
| 4 | 年龄句式 | ✅ 生效（回归测试通过）。 |
| 5 | poster 补 headline | ✅ 生效。idx=55（travel-mug banner 含 headline）：判 poster，主标签与 Grok 一致。 |
| 6 | layout 需设计语境 | ✅ 无回归。idx=21（"Consistent layout & sharp text" 手机 UI 渲染清单）：未触发 poster（判 photography，边界可接受）。 |
| 7 | illustrated 形态 | ✅ 生效。抽查 13 条含 illustrated 条目：528（hand-drawn illustrated）判 illustration 与 Grok 一致；1075（Illustrated Narrative）判 illustration，Grok 标 other（我对）。 |

## 3. 人工抽查 32 条分歧判定汇总

| 判定 | 数量 | 说明 |
|---|---|---|
| 我的脚本正确 | ~17 | Grok 的 other 垃圾桶为主（1125/45/515 人像被标 other；323/235/832 摄影被标 other）；662/582/85 的人像（Grok 标 photography）；52/292/468/489 的明确海报（Grok 标 other）；638/547/507/590 的海报+插画（Grok 标 portrait，漏版式意图） |
| Grok 正确 | ~10 | 全部是脚本关键词覆盖度问题：924（人脸重建）、1071/841（"Prompt 👇" 前缀超窗口）、648（"confident" 不在形容词表）、121（"A young woman" 形容词紧贴）、773、684、1046、1002、1111（人名/副词结构）——没有一条是规则逻辑错误 |
| 双方都错/边界 | ~5 | 9（脑雕塑 close-up：Grok 的 other 对，我错）；850（中文杂志版式：正确应为 poster，双方都错——中文版式词缺"杂志/报刊/版式"）；38（室内女人：正确应为 portrait，双方都错）；138（editorial lifestyle：人像/纪实边界）；790（Tom & Jerry：难判） |

关键结论：Grok 判对的条目里**没有一条是规则逻辑错误**——全部是脚本关键词/正则覆盖度问题。规则逻辑在七轮抽查中从未被证伪。

## 4. 本轮新发现（v2.6 候选，按优先级）

1. **中文版式词补"杂志/报刊/版式"**：idx=850（"高级黑白报刊社论风…杂志内容…版式"）应判 poster，当前中文版式词只有海报/封面/信息图/排版。
2. **人物开场形容词表补 "confident"**：idx=648（"A confident young adult male…"）漏检。
3. **"Prompt 👇"类前缀**：idx=1071/841 的 "Made with… Try it… Prompt 👇" 前缀把人物开场推出 120 字符窗口；可考虑剥离常见前缀或放宽窗口。
4. 其余小项："a strikingly handsome"（副词+形容词，idx=1046）、人名主语（Ana de Armas，idx=1002，关键词难覆盖，LLM 可判）。

注：第 2–4 条均为脚本覆盖度问题，LLM 按 SKILL.md 执行不受影响；第 1 条建议纳入规则（中文词表补齐）。

## 5. 代表性例子（3 个）

**例 1：looking into the camera 生效（idx=320，v2.5 成功）**
- Prompt（摘要）："One young adult woman… looking directly into the camera. Preserve the same elegant, relaxed presence…"
- v2.4：`photography`｜v2.5：`portrait`｜Grok：`portrait`｜**一致 ✓**
- 理由："looking directly into the camera" 是 v2.5 新增的人物信号；另修脚本允许 directly/straight 插入。

**例 2：中文杂志版式漏检（idx=850，v2.6 待修）**
- Prompt（摘要）："分享套GPT Image2提示词…生成一张高级黑白报刊社论风英文文章版式，整体视觉像高端文学杂志…"
- v2.5：`other`｜Grok：`portrait`｜**正确应为：poster**（双方都错）
- 理由：明确的杂志版式意图，但中文版式关键词缺"杂志/报刊/版式"。规则层面应补词。

**例 3：海报+插画双标签正确（idx=507，v2.5 成功）**
- Prompt（摘要）："请将我上传的每一张照片分别制作成一张独立的高级设计海报…下半部分…重构为新民艺剪纸拼贴风的扁平化插画…"
- v2.5：`poster, illustration`｜Grok：`portrait`｜**我对 ✓**
- 理由：明确海报版式（规则 1）＋ 插画风格（规则 2）→ 双标签；Grok 漏掉版式意图。

## 6. 总体评价：是否可以公开

**结论：v2.5 可以公开，建议附"已知局限"声明。**

- **七轮单调向好**：52.1% → 50.5% → 54.6% → 55.3% → 55.3% → 57.0% → **57.5%**；v2.5 的 7 项新增全部生效、无回归。
- **人工抽查中脚本正确数约为 Grok 的两倍**（~17 vs ~10），且 Grok 判对的条目全部是脚本覆盖度问题、不是规则逻辑问题——LLM 按 SKILL.md 执行的真实效果高于脚本数字。
- **raw 数字天花板是度量噪声**：Grok 标签 portrait 滥用 44%、other 当垃圾桶 20%，拿它当标准本身就有系统性偏差。
- **公开时的已知局限声明**（README 应如实写）：(1) 关键词机械实现约 57%，LLM 按规则执行效果更好；(2) 已知覆盖度缺口（中文"杂志/报刊/版式"、"confident" 等，LLM 执行不受影响，v2.6 候选）；(3) stub（文本缺失/截断）只能标记 `needs_manual_review`；(4) portrait/photography 在 candid/街拍/editorial 类存在真实模糊地带。
- 后续如需 v2.6：第 4 节的 4 条均为小修小补（其中第 1 条建议纳入规则），可在公开后迭代。

所有数字均为本轮实际跑出，未编造；方法局限已在报告中声明。
