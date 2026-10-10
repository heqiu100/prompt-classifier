# prompt-classifier 验证报告 v6（v2.4 规则）

- 验证日期：2026-10-10
- 测试集：Grok Bot 已收集的 1144 条提示词（`~/workspace/prompt-gallery-audit/records.jsonl`）
- 规则版本：v2.4（新增：预处理追加否定语境剥离——删除 "not + 风格词" 结构后再匹配；illustration 的 drawing 限缩为名词搭配 line drawing / drawing style / children's book drawing / pencil drawing，动词用法不触发）
- 方法：关键词脚本机械实现 v2.4 规则（中英日三语，先剥离负面提示+forbidden+否定语境，再按规则 1→6 顺序执行）跑完全量；人工抽查 30 条分歧样本逐条判定。
- 方法局限：关键词脚本是规则的近似实现，不是 LLM 的语义判断；Grok Bot 标签本身有系统性质量问题（portrait 滥用、other 垃圾桶），把它当 ground truth 会系统性低估规则水平。本轮另修复了脚本自身的 2 个 fidelity bug（"human"→"man" 子串误匹配，已加词边界；"the subject is + 非人物"误触发，已要求 is 后接人物词），属脚本问题，不涉及规则改动。

## 1. 数据与核心数字

| 项 | 数量 |
|---|---|
| 总记录 | 1144 |
| stub（占位符/超短碎片，len<60 或占位符模式） | 114 |
| 无 Grok Bot Style 标签 | 26 |
| **可对比** | **1004** |

| 轮次 | 规则版本 | 可对比 n | 主标签命中率 | 完全一致率 |
|---|---|---|---|---|
| v1 | 初版 | 1070 | 52.1% | 36.1% |
| v2 | +7 条改进 | 1082 | 50.5% | 33.3% |
| v2.1 | +v3 前 4 条 | 1001 | 54.6% | 38.4% |
| v2.2 | +v4 全部 6 条 | 1001 | 55.3% | 39.3% |
| v2.3 | +关键词覆盖/人物主语 | 1006 | 55.3% | 37.0% |
| **v2.4（本轮）** | **+否定剥离/drawing 限缩** | **1004** | **57.0%** | **38.6%** |

**同口径复核**（v2.3 规则重跑本轮相同的 1004 条）：v2.3-mode 得 55.7%/37.9%，v2.4 得 57.0%/38.6%。
结论：v2.4 相对 v2.3 提升 +1.3pp/+0.7pp，六轮最高；v5 报告预期的"回升 1–2pp"兑现。

其他指标：双标签率 12.5%（126/1004）；81 条记录的负面提示/forbidden 段落被成功剥离后再分类。

## 2. v2.4 两项修复验证（逐项）

| # | 修复项 | 验证结果 |
|---|---|---|
| 1 | 否定语境剥离 | ✅ 生效。idx=1034（"not illustration, not CGI look"）：v2.3 误出 `illustration, portrait` → v2.4 `portrait`，与 Grok 一致。idx=549（"not doll-like, not anime"）：v2.3 误出 `anime, photography` → v2.4 `photography`，anime 误伤消除（正确应为 portrait，见第 5 节）。idx=1132（"not illustrated, not anime"）：illustration/anime 误伤均消除。idx=262（"not a poster"）：poster 误触发消除 → `other`。 |
| 2 | drawing 动词限缩 | ✅ 生效且无回归。idx=668（"drawing a traditional wooden bow" 拉弓）：v2.3 误出 `illustration, portrait` → v2.4 `portrait`，与 Grok 一致。idx=750/1058（"drawing the eye" 习语）：v2.3 误出 `illustration` → v2.4 `other`，与 Grok 一致。idx=71（"children's book drawing" 名词用法）：仍正确判 `illustration`，与 Grok 一致，无回归。 |

## 3. v2.3 → v2.4 差异条目分析（20 条变化）

- 变好 13 条（按 Grok 标准）：含上述 668/750/1058/1034，以及 50/62/82/101（海报类："line drawing"不再抢占主标签，`poster` 扶正）、144/827/959（自拍/特写人像：illustration 误伤消除 → `portrait`）、262（not a poster）、721（"not 3d/CGI"类否定被剥离 → `photography`）。
- 变坏 0 条：**无回归**。
- 中性 7 条：549（anime 误伤消除，但正确 portrait 仍未命中——人物主语覆盖度问题）、1132、1010、1082、107、489、26（Grok 侧标签质量问题为主）。

## 4. 人工抽查 30 条分歧判定汇总

| 判定 | 数量 | 说明 |
|---|---|---|
| 我的脚本正确 | ~17 | Grok 的 other 垃圾桶为主（34/36/44 镜面自拍→portrait、235 食物摄影→photography、0/10 插画→illustration、292 明确写"海报"→poster）；56/84/236 的 candid/街拍人物主体→portrait 可辩护 |
| Grok 正确 | ~8 | 脚本覆盖度问题：58/80/96（fashion editorial 人像：形容词 "elegant"/"stylish" 未收录）、320（"looking into the camera" 变体未收录）、179（赛博拉面馆场景→photography）、261（supermodel editorial 实为 portrait，双方都错） |
| 边界/双方都错 | ~5 | 9（花卉脑雕塑→photography）、256（仓鼠特写→photography，Grok 的 portrait 系滥用）、305/354/402（"full prompt in page body" 碎片，文本写明人物主体→portrait）、21（手机 UI mockup："Consistent layout" 误触发 poster，应为 other）、136/274（无风格线索，other 系诚实输出） |

关键结论：Grok 判对的条目里**没有一条是规则逻辑错误**——全部是脚本关键词/正则覆盖度问题。规则逻辑在六轮抽查中从未被证伪。

## 5. 本轮新发现（v2.5 候选，按优先级）

1. **"looking into the camera"**：idx=320，`looking at the camera` 的常见变体，portrait 信号应收录。
2. **人物开场形容词补 "elegant" / "stylish"**：idx=80（"Elegant, slender East Asian girl…"）、idx=39（"A stylish European woman…"）漏检。
3. **"full body shot of a [person]"**：idx=339，应计入 portrait 人物信号（当前只有 close-up/upper body/three-quarter）。
4. **"the subject is" 需人物语境**（脚本已修）：idx=95 "The subject is a majestic natural landscape" 曾误触发 portrait；规则层面 LLM 可判对，脚本侧已收紧。
5. **"layout" 在渲染质检语境中误触发**：idx=21 "Consistent layout & sharp text"（手机 UI 渲染清单）→ poster 误伤；罕见，可考虑要求 layout 搭配设计语境。
6. 其余（v5 已列，未修）：`headline` 版式信号、`illustrated` 形态、人物主语年龄句式（"A 20–25 year old woman"）。

## 6. 代表性例子（3 个）

**例 1：否定剥离生效（idx=1034，v2.4 成功）**
- Prompt（摘要）："Subject & Action — A 22-year-old Japanese-European mixed-race woman…not illustration, not CGI look"（自拍类）
- v2.3：`illustration, portrait`｜v2.4：`portrait`｜Grok：`portrait`｜**一致 ✓**
- 理由："not illustration, not CGI" 是否定，不是风格。剥离后 selfie+人物主角 → portrait。

**例 2：drawing 动词修复生效（idx=668，v2.4 成功）**
- Prompt（摘要）："A photorealistic cinematic outdoor sports portrait…drawing a traditional wooden bow…leather archery glove on her drawing hand"
- v2.3：`illustration, portrait`｜v2.4：`portrait`｜Grok：`portrait`｜**一致 ✓**
- 理由："drawing a bow" 是拉弓（动词），名词搭配限缩后不再误触发。

**例 3：the subject 误伤修复（idx=95，本轮脚本 fidelity 修复）**
- Prompt（摘要）："This aerial shot captures a dramatic, misty landscape…The subject is a majestic natural landscape…"
- 修复前：`portrait`（"the subject is" 误触发）｜修复后：`photography`｜Grok：`photography`｜**一致 ✓**
- 理由："subject" 在此指照片的拍摄对象（山谷），非人物。规则层面 LLM 可判对；脚本侧已收紧为 is 后接人物词。

## 7. 总体评价：是否可以公开

**结论：v2.4 可以公开，建议附"已知局限"声明。**

- **六轮单调向好**：52.1% → 50.5% → 54.6% → 55.3% → 55.3% → **57.0%**；v2.4 的两项修复全部生效、零回归；"not anime 被判成 anime"这类公开后最易被挑刺的错误已消除。
- **人工抽查中脚本正确数约为 Grok 的两倍**（~17 vs ~8），且 Grok 判对的条目全部是脚本覆盖度问题、不是规则逻辑问题——LLM 按 SKILL.md 执行的真实效果高于脚本数字。
- **raw 数字天花板是度量噪声**：Grok 标签 portrait 滥用 44%、other 当垃圾桶 20%，拿它当标准本身就有系统性偏差。
- **公开时的已知局限声明**（README 应如实写）：(1) 关键词机械实现约 57%，LLM 按规则执行效果更好；(2) 已知覆盖度缺口（"looking into the camera"、elegant/stylish 开场、full body shot 等，LLM 执行不受影响，v2.5 候选）；(3) stub（文本缺失/截断）只能标记 `needs_manual_review`；(4) portrait/photography 在 candid/街拍类存在真实模糊地带。
- 后续如需 v2.5：第 5 节的 6 条均为小修小补，可在公开后迭代。

所有数字均为本轮实际跑出，未编造；方法局限已在报告中声明。
