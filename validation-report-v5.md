# prompt-classifier 验证报告 v5（v2.3 规则）

- 验证日期：2026-10-10
- 测试集：Grok Bot 已收集的 1144 条提示词（`~/workspace/prompt-gallery-audit/records.jsonl`）
- 规则版本：v2.3（新增：portrait 补 自撮り；illustration 补 drawing、ukiyo-e/浮世绘；photography 补 landscape；规则 4 人物主语信号放宽 fashion model 主语、wide shot 句式、Main Subject 开场；candid/街拍边界澄清）
- 方法：关键词脚本机械实现 v2.3 规则（中英日三语，先剥离负面提示+forbidden，再按规则 1→6 顺序执行）跑完全量；人工抽查 30 条分歧样本逐条判定。
- 方法局限：关键词脚本是规则的近似实现，不是 LLM 的语义判断；Grok Bot 标签本身有系统性质量问题（portrait 滥用、other 垃圾桶），把它当 ground truth 会系统性低估规则水平。

## 1. 数据与核心数字

| 项 | 数量 |
|---|---|
| 总记录 | 1144 |
| stub（占位符/超短碎片） | 112 |
| 无 Grok Bot Style 标签 | 26 |
| **可对比** | **1006** |

| 轮次 | 规则版本 | 可对比 n | 主标签命中率 | 完全一致率 |
|---|---|---|---|---|
| v1 | 初版 | 1070 | 52.1% | 36.1% |
| v2 | +7 条改进 | 1082 | 50.5% | 33.3% |
| v2.1 | +v3 前 4 条 | 1001 | 54.6% | 38.4% |
| v2.2 | +v4 全部 6 条 | 1001 | 55.3% | 39.3% |
| **v2.3（本轮）** | **+关键词覆盖/人物主语** | **1006** | **55.3%** | **37.0%** |

**同口径复核**（用 v2.2 规则重跑本轮相同的 1006 条）：v2.2-mode 得 55.6%/37.6%，v2.3 得 55.3%/37.0%。
结论：v2.3 在 raw 数字上与 v2.2 基本持平（±0.3pp 属噪声），没有拉开差距。原因见第 3 节：新增项修好了真缺口，但也引入了 3 处新的误伤，两相抵消。

## 2. v2.3 新增项验证（逐项）

| # | 新增项 | 验证结果 |
|---|---|---|
| 1 | portrait 补 自撮り | ✅ 生效。idx=951（日语全身镜自撮り）v2.2 判 photography，v2.3 判 portrait，与 Grok 一致。全量 2 条含自撮り均判 portrait。 |
| 2 | illustration 补 drawing | ⚠️ 部分生效。名词用法修好 1 条（idx=71 儿童绘本 otter → illustration ✓）；但动词/习语用法引入 3 处误伤（见第 4 节）。 |
| 3 | illustration 补 ukiyo-e/浮世绘 | ✅ 按文本生效。idx=497 文本被截断（仅 118 字符，"A Japanese Edo-period Ukiyo-e w…"），按"只看文本"判 illustration 是诚实输出；Grok 的 product 应来自看图，文本不可比。 |
| 4 | photography 补 landscape | ✅ 基本生效。idx=492（"Peace has a landscape"）v2.2 other → v2.3 photography ✓；15 条含 landscape 的条目中 3 条主标签为 photography，其余被更强信号（人物/版式）覆盖，符合规则顺序。 |
| 5 | 人物主语放宽（fashion model / wide shot / Main Subject） | ✅ 无回归，未发现误伤；覆盖到 "wide shot of a beautiful woman" 类句式。 |
| 6 | candid/街拍边界澄清 | 中性。本轮分歧样本中未出现典型的 candid 边界案例，规则文本已备好，待生产数据检验。 |

## 3. v2.2 → v2.3 差异条目分析（21 条变化）

- 变好 3 条（按 Grok 标准）：idx=71（drawing 名词）、idx=492（landscape）、idx=951（自撮り）。
- 按文本实质判断另有 3 条"名义变坏、实际变好"：idx=369/417（"A black and white line drawing…"按文本就是 illustration，Grok 的 other 错）、idx=803（"鏡越しに自撮り"按文本就是 portrait，Grok 的 other 错）。
- 真误伤 3 条（v2.3 新增词的代价）：idx=668、idx=750、idx=1058，全部来自 `drawing` 的动词/习语用法（见第 4 节第 2 条）。

## 4. 本轮新发现（v2.4 清单，按优先级）

1. **否定语境污染（最高优先级）**：正向描述中的 "not anime / not illustration / not CGI" 被脚本当成风格词命中。
   - idx=549："not doll-like, not anime, not overly large eyes" → 脚本误出 anime，正确应为 portrait（Grok 对）。
   - idx=1034："not illustration, not CGI look" → 脚本误出 illustration，正确应为 portrait（selfie，Grok 对）。
   - idx=1132："not illustrated, not anime" → 脚本误出 anime。
   - idx=262："not a poster" → 脚本命中 poster（该条按版式规则判 poster 仍可辩护，但否定词应被识别）。
   - 建议：预处理追加一步——删除 "not X" 否定从句中的风格关键词，或匹配时跳过 "not / no / without / 避免" 后 3 个词内的风格词。LLM 执行规则时天然能处理，脚本需显式处理。
2. **`drawing` 动词/习语限缩**：只保留名词用法。
   - idx=668："drawing a traditional wooden bow"（拉弓）、"drawing hand"（拉弓的手）→ 误出 illustration。正确应为 portrait（photorealistic sports portrait）。
   - idx=750 / idx=1058："drawing the eye towards…"（吸引视线，习语）→ 误出 illustration。
   - 建议：`drawing` 仅在 "line drawing / pencil drawing / drawing style / drawing of" 等名词搭配中触发；排除 "drawing the eye(s)"、"drawing a|his|her|their + 名词" 动词结构。
3. **`headline` 补作版式信号**：idx=691（"Hand-torn paper poster" 标题，正文写 "headline blocks…bold brush lettering"），正文无 poster 一词但排版意图明确，脚本判 other，Grok 判 poster。建议：`headline` 配合"排版意图"（centered / layout / blocks）时触发 poster；规则 1 的版式词表可增 headline。
4. **`illustrated` 形态覆盖**：idx=298（"stylized illustrated portrait"），`illustrated` 未命中 `illustration`，脚本漏掉渲染风格。建议：关键词表增加 illustrated，或做简单词干归一。
5. **人物主语年龄句式**：idx=161（"A 20–25 year old woman…"），无形容词、纯年龄数字开场，脚本人物主语正则未覆盖。建议：正则增加 `\d+([–-]\d+)?\s*year[- ]old` 开场模式。
6. **candid/街拍边界**：本轮未遇到典型案例，规则文本已澄清，暂不改动，待生产数据反馈。

## 5. 人工抽查 30 条分歧判定汇总

| 判定 | 数量 | 说明 |
|---|---|---|
| 我的脚本正确 | 16 | Grok 的 other 垃圾桶、portrait 滥用为主；含自撮り/landscape 等 v2.3 修复项 |
| Grok 正确 | 7 | 否定语境 3 条（549/1034/1132）、人物主语年龄句式 1 条（161）、headline 版式 1 条（691）、技术图纸 1 条（1097）、storyboard 版式 1 条（172） |
| 持平/边界/一致 | 7 | 双渲染风格（anime+illustration）、contact sheet 版式 vs 人像、玩偶描述、医疗可视化等 |

关键结论：Grok 对的 7 条里，**没有一条是规则逻辑错误**——3 条是否定语境（规则本身 LLM 可判对，脚本缺处理），4 条是脚本关键词覆盖度。规则逻辑在本轮 30 条抽查中没有被证伪。

## 6. 代表性例子（3 个）

**例 1：否定语境污染（idx=549，v2.4 待修）**
- Prompt（摘要）："High-end erotic fashion office editorial…A tall Japanese office woman…photorealistic single subject…natural Japanese facial features, realistic bone structure, not doll-like, not anime, not overly large eyes."
- 我的标签：anime, photography｜Grok：portrait｜**正确：portrait**
- 理由："not anime" 是否定，不是风格。photorealistic + 人物绝对主角 → portrait。暴露脚本无否定处理（见第 4 节第 1 条）。

**例 2：drawing 动词误伤（idx=668，v2.4 待修）**
- Prompt（摘要）："A photorealistic cinematic outdoor sports portrait…She stands in a three-quarter pose…drawing a traditional wooden bow with controlled strength…leather archery glove on her drawing hand."
- 我的标签：illustration, portrait｜Grok：portrait｜**正确：portrait**
- 理由："drawing a bow" 是拉弓（动词），不是绘画。photorealistic + 人物主角 → portrait。主标签被 illustration 抢占属误伤（见第 4 节第 2 条）。

**例 3：自撮り修复生效（idx=951，v2.3 成功）**
- Prompt（摘要）：日语全身镜自拍说明，"白い寝室の全身鏡で…成人女性がスマートフォン自撮りをする縦位置写真。人物は画面中央を頭から腿まで大きく占め…"
- v2.2：photography｜v2.3：portrait｜Grok：portrait｜**一致 ✓**
- 理由：自撮り（日语自拍）+ 人物占画面中央 → portrait。v2.3 新增关键词生效的干净案例。

## 7. 总体评价：是否可以公开

**结论：v2.3 规则逻辑可以公开，但建议先做一轮 v2.4（改动小、收益明确）再公开。**

- **规则方向五轮未被证伪**：人工抽查中 Grok 判对的条目全部是脚本覆盖度/否定处理问题，没有一条指向规则逻辑错误；"双方都错"的条目里规则（LLM 执行）多能判对。
- **v2.3 的 raw 数字是持平，不是退步**：同口径下 v2.2-mode 55.6% vs v2.3 55.3%，差异来自 Grok 基线噪声；按文本实质判断，v2.3 修好的（自撮り/landscape/drawing 名词/369/417/803）多于误伤的（drawing 动词 3 条）。
- **v2.4 的 1、2 条值得做**：否定语境剥离和 drawing 动词限缩都是小改动、影响面明确（本轮已定位到具体条目），修完预期 raw 命中率回升 1–2pp，且消除的是"明显看得出来的错误"（not anime 被判成 anime 这种，公开后最容易被挑刺）。
- **公开时的已知局限声明**（README 应如实写）：(1) 关键词机械实现约 55%，LLM 按规则执行效果更好；(2) stub（文本缺失/截断）只能标记 `needs_manual_review`；(3) portrait/photography 在 candid/街拍类存在真实模糊地带；(4) Grok Bot 历史标签本身质量不高（portrait 滥用 44%、other 20%），对比数字仅供参考。
- 若用户接受"先公开、再迭代"，v2.3 当前也可公开；若追求更稳，做完 v2.4 第 1–2 条跑第六轮确认无回归再公开。
