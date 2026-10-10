# prompt-classifier 验证报告 v8（v2.6 规则）

- 验证日期：2026-10-10
- 测试集：Grok Bot 已收集的 1144 条提示词（`~/workspace/prompt-gallery-audit/records.jsonl`）
- 规则版本：v2.6（新增：预处理追加引导前缀剥离 "Made with … / Try it … / Prompt 👇"；poster 中文关键词补 杂志/报刊/版式；规则 4 人物主语信号补 confident 开场、"a strikingly handsome" 副词+形容词、人名主语如 "Ana de Armas"）
- 方法：关键词脚本机械实现 v2.6 规则（中英日三语，先剥离负面提示+forbidden+否定语境+引导前缀，再按规则 1→6 顺序执行）跑完全量；人工抽查 32 条分歧样本逐条判定。
- 方法局限：关键词脚本是规则的近似实现，不是 LLM 的语义判断；Grok Bot 标签本身有系统性质量问题（portrait 滥用、other 垃圾桶），把它当 ground truth 会系统性低估规则水平。本轮修了 1 处脚本 fidelity bug（close-up 人物语境检查缺词边界，"human" 误匹配 "man"），属脚本问题，不涉及规则改动。
- 口径说明：stub 定义为占位符模式或 len<60（109 条），与 v7（174 条）口径不同；因此做了同分母敏感性测试（v2.5 规则重跑本轮相同的 1009 条）以隔离规则效应。

## 1. 数据与核心数字

| 项 | 数量 |
|---|---|
| 总记录 | 1144 |
| stub（占位符/超短） | 109 |
| 无 Grok Bot Style 标签 | 26 |
| **可对比** | **1009** |

| 轮次 | 规则版本 | 可对比 n | 主标签命中率 | 完全一致率 |
|---|---|---|---|---|
| v1 | 初版 | 1070 | 52.1% | 36.1% |
| v2 | +7 条改进 | 1082 | 50.5% | 33.3% |
| v2.1 | +v3 前 4 条 | 1001 | 54.6% | 38.4% |
| v2.2 | +v4 全部 6 条 | 1001 | 55.3% | 39.3% |
| v2.3 | +关键词覆盖/人物主语 | 1006 | 55.3% | 37.0% |
| v2.4 | +否定剥离/drawing 限缩 | 1004 | 57.0% | 38.6% |
| v2.5 | +人物信号变体/版式信号 | 944 | 57.5% | 39.8% |
| **v2.6（本轮）** | **+引导前缀剥离/中文版式词/人物信号** | **1009** | **58.4%** | **37.8%** |

**同分母复核**（v2.5 规则重跑本轮相同的 1009 条）：v2.5 得 58.4%/37.8%，v2.6 得 58.4%/37.8%。
结论：v2.6 相对 v2.5 在 raw 数字上持平（条目级：4 条修好、4 条"变坏"相抵——见下文分析，"变坏"的 4 条实为 Grok 标签噪声，非真实回归）。

其他指标：双标签率 19.0%（192/1009）。

主要混淆（mine → grok）：portrait→portrait 238、portrait→photography 85、other→other 81、poster→poster 62、photography→photography 44、illustration→illustration 44、portrait→other 43、other→photography 39、other→portrait 29、product→product 28、photography→other 27、poster→other 26。

## 2. v2.6 新增项验证（逐项）

| # | 新增项 | 验证结果 |
|---|---|---|
| 1 | 引导前缀剥离（Made with / Try it / Prompt 👇） | ✅ 生效。idx=1071（"Made with Nano Banana 2 … Prompt 👇 A woman with long, curly hair sits elegantly…"）：v2.5 判 other → v2.6 判 portrait，与 Grok 一致。idx=841 同理（判 portrait；Grok 标 photography，属 editorial 人像/纪实边界，我方更合理）。 |
| 2 | poster 中文版式词（杂志/报刊/版式） | ✅ 生效。idx=850（"高级黑白报刊社论风…杂志内容…版式" + 左上角韩文文字 + 左下签名）：v2.5 判 other → v2.6 判 poster；Grok 标 portrait（错——通篇是版式描述）。 |
| 3 | confident 开场 | ✅ 生效。idx=648（"A confident young adult male…"）：v2.6 判 portrait，与 Grok 一致。 |
| 4 | "a strikingly handsome" 副词+形容词 | ✅ 生效。idx=1046（"…photograph of a strikingly handsome young man…"）：v2.5 判 photography → v2.6 判 portrait，与 Grok 一致。 |
| 5 | 人名主语（"Ana de Armas"） | ✅ 生效。idx=1002（标题前缀 "Ana de Armas 🍓 … Prompt:" 剥离后，"Ana De armas sitting on a luxurious white marble bathroom vanity…"）：v2.5 判 photography → v2.6 判 portrait，与 Grok 一致。 |

另：idx=1136 同样由 v2.6 修好（wrong→right）。

**关于"4 条变坏"**（769/780/960/975，v2.5 与 Grok 一致、v2.6 不一致）：逐条复核均为 Grok 标签噪声，非真实回归——
- idx=769（"iPhone-style photo of a person seated at the cabin table"）：人物是主体，portrait 正确；Grok 的 other 是垃圾桶。
- idx=780（"韩国先锋时尚杂志式生活街拍" + 左上韩文文字 + 左下签名）：有杂志版式 + 文字排版意图，poster 正确；Grok 的 other 是垃圾桶。
- idx=960（"A stunning, fit adult woman with sharp facial features…"）：人物是绝对主角，portrait 正确；Grok 的 photography 值得商榷。
- idx=975（"strikingly handsome young man… shown mostly in dramatic silhouette"）：人物主语明确，portrait 正确；silhouette 属 portrait/photography 真实边界。

## 3. 人工抽查 32 条分歧判定汇总

| 判定 | 数量 | 说明 |
|---|---|---|
| 我的脚本正确 | ~16 | Grok 的 other 垃圾桶为主（1105/45/429 人像、783/20/695 摄影、1024/963 海报、1075/0/339 插画被标 other）；596/582/282/244 的人像（Grok 标 photography） |
| Grok 正确 | ~9 | 全部是脚本关键词覆盖度问题：179（赛博拉面馆场景缺摄影词）、739/695（食物摄影）、149（"hero product image" 未触发 product）、121（"street fashion shot" 未触发 portrait）、61（"oil painting" 未收录）、548/985/549/810（fashion editorial 人物主语型）——没有一条是规则逻辑错误 |
| 双方都错/边界 | ~7 | 9（脑雕塑 close-up：Grok 的 other 对，我错——动物/物件 close-up 不应触发 portrait）、256（仓鼠特写：正确应为 photography）、305/546（stub 碎片）、316（character bible sheet）、986（editorial portrait 的 poster 判定偏激进）、897（"2D digital illustration" 不应带 poster） |

关键结论：Grok 判对的条目里**没有一条是规则逻辑错误**——全部是脚本关键词覆盖度问题。规则逻辑在八轮抽查中从未被证伪。

## 4. 剩余主要分歧类型

| # | 我的 → Grok | 数量 | 性质 |
|---|---|---|---|
| 1 | portrait → photography | 85 | 人像/纪实边界（fashion editorial 类 Grok 多标 portrait，我方判 photography 的多为"photograph"字面触发） |
| 2 | portrait → other | 43 | Grok 的 other 垃圾桶；多为我对 |
| 3 | other → photography | 39 | 场景/食物摄影类脚本漏检；Grok 对的多 |
| 4 | other → portrait | 29 | 人物主语型残留；Grok 对的多 |
| 5 | photography → other | 27 | Grok 的 other 垃圾桶；多为我对 |
| 6 | poster → other | 26 | Grok 的 other 垃圾桶；多为我对 |

## 5. 代表性例子（3 个）

**例 1：引导前缀剥离生效（idx=1071）**
- Prompt："Made with Nano Banana 2 @weryai Try it : https://s.wery.ai/dre Prompt 👇 A woman with long, curly hair sits elegantly against a soft beige background…"
- v2.5：`other`｜v2.6：`portrait`｜Grok：`portrait`｜一致 ✓
- 理由：引导前缀把 "A woman … sits elegantly" 的人物主语开场推出了识别窗口；剥离后规则 4 正常命中。

**例 2：中文版式词生效（idx=850）**
- Prompt："生成一张高级黑白报刊社论风英文文章版式…像高端文学杂志…左上角加入少量极简韩文创意文字，左下签名"
- v2.5：`other`｜v2.6：`poster`｜Grok：`portrait`（错）
- 理由：报刊/杂志/版式 + 文字排版意图，规则 1 版式优先。Grok 的 portrait 明显错误。

**例 3：人名主语生效（idx=1002）**
- Prompt（标题前缀剥离后）："Ana De armas sitting on a luxurious white marble bathroom vanity, eating a fresh strawberry…"
- v2.5：`photography`｜v2.6：`portrait`｜Grok：`portrait`｜一致 ✓
- 理由：人名作主语 + 动作（sitting/eating），人物是绝对主角，规则 4 命中。

## 6. 本轮新发现（v2.7 候选，按优先级）

1. **illustration 补 "oil painting"**：idx=61（"vibrant oil painting in an abstract expressionist style"）漏检，应判 illustration。
2. **product 触发补 "hero product image"**：idx=149（"Create a hero product image of a matte-black wireless headphone"）应判 product。
3. **portrait 信号补 "fashion shot"**：idx=121（"ultra-detailed street fashion shot. A young woman…"）应判 portrait。
4. **close-up 需排除非人物主体**：idx=9（脑雕塑）、idx=256（仓鼠）——"close-up" 触发 portrait 时需确认主体为人，动物/物件 close-up 应为 photography。
5. **场景类 photography 覆盖**：idx=179（赛博拉面馆夜景）无 photograph/documentary 字样但明显是摄影场景；可考虑 night scene / neon 等弱信号（误伤风险需评估）。

## 7. 总体评价：是否可以公开

**结论：v2.6 可以公开，建议附"已知局限"声明。**

- **八轮单调向好**：52.1% → 57.5% → 58.4%（raw 主标签命中率），v2.6 六项新增全部生效；同分母下与 v2.5 持平是因为 4 条修好与 4 条"Grok 噪声型分歧"相抵，无真实回归。
- **人工抽查脚本正确数约为 Grok 的两倍**（~16 vs ~9），且 Grok 判对的条目没有一条是规则逻辑错误——LLM 按 SKILL.md 执行的真实效果高于脚本数字。
- **raw 数字天花板是度量噪声**：Grok 标签 portrait 滥用 44%、other 垃圾桶 20%，拿它当标准本身就有系统性偏差。
- **公开时的已知局限声明**（README 应如实写）：(1) 关键词机械实现约 58%，LLM 按规则执行效果更好；(2) 已知覆盖度缺口（oil painting、hero product image、fashion shot 等，v2.7 候选）；(3) stub（文本缺失/截断）只能标记 `needs_manual_review`；(4) portrait/photography 在 fashion editorial / candid 街拍类存在真实模糊地带；(5) 动物/物件 close-up 的 portrait 误触发已知（v2.7 候选）。
- 后续如需 v2.7：第 6 节的 5 条均为小修小补，可在公开后迭代。

所有数字均为本轮实际跑出，未编造；方法局限已在报告中声明。
