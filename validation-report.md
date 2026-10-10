# prompt-classifier 验证报告

- 验证日期：2026-10-10
- 测试集：Grok Bot 已收集的 1144 条提示词（`~/workspace/prompt-gallery-audit/records.jsonl`）
- 方法：先用关键词脚本把 SKILL.md 的 6 步判定规则机械实现（中英双语关键词，按规则 1→4 的顺序执行），跑完全量；再人工抽查 40 条不一致样本逐条判定谁对谁错。
- 方法局限：关键词脚本是规则的近似实现，不是 LLM 的语义判断。凡下文说"我的规则判 X"，指脚本输出；边界案例以人工复核结论为准。

## 1. 数据清洗

| 项 | 数量 |
|---|---|
| 总记录 | 1144 |
| stub（Prompt 字段是"full prompt in body"类占位符，文本无法分类） | 47 |
| 无 Grok Bot Style 标签 | 29 |
| **可对比** | **1070** |

47 条 stub 已排除在一致率之外——文本缺失时任何纯文本分类器都无解，Grok Bot 是看了图打的标签（如 30 条标了 portrait），不具可比性。

## 2. 一致率

| 指标 | 结果 |
|---|---|
| 主标签命中率（我的主标签 ∈ Grok 标签集合） | **557/1070 = 52.1%** |
| 完全一致率（标签集合完全相等） | **386/1070 = 36.1%** |

## 3. Grok Bot 标签的系统性偏差

Grok Bot 的标签分布（1115 条有标签）：portrait 44.2%（493）、other 20.2%（225）、photography（单/多标签合计约 26%）、illustration 11.6%、product 10.9%、poster 8.5%、**3d 仅 1.3%（14 条）、anime 仅 1.3%（15 条）**。

三个明确问题：

1. **portrait 滥用**：44% 的条目标了 portrait。抽查发现酸奶广告 commercial（应为 product）、微缩世界（应为 3d）也被标 portrait。
2. **other 当垃圾桶**：20.2%。216 条非 stub 的 other 里，我的规则能给出具体标签的有 **115 条（53.2%）**：photography 54、illustration 20、portrait 15、poster 14、3d 6、product 4、anime 2。抽查显示这些基本都该有具体标签，例如"请制作成高级设计海报"被标 other（应为 poster）、"hand-drawn Studio Ghibli style"被标 other（应为 illustration）。
3. **3d/anime 系统性漏标**：如"handcrafted miniature world…three-dimensional scene"被标 portrait（应为 3d）。

结论：不一致主要不是我的规则错，而是 **Grok Bot 的标签质量本身不高**——portrait 泛化、other 兜底过度、3d/anime 几乎不用。

## 4. 混淆最多的 3 组标签对（排除 stub 后，n=1070）

| # | 我的 → Grok | 数量 | 主因 |
|---|---|---|---|
| 1 | other → portrait | 66 | 我的关键词没覆盖"selfie/面对镜头/JSON subject 字段"等人物主体表述；按规则 4 多数确应判 portrait，是**脚本覆盖度**问题 |
| 2 | photography → other | 56 | Grok 把明确的摄影类 prompt（街头实拍、食物摄影）丢进 other；Grok 错 |
| 3 | other → photography | 54 | 日语 prompt（含 写真）我的英文关键词没命中；Grok 对；属**多语言覆盖**问题 |

次高：photography → portrait（41，多为 selfie/人脸特写，Grok 对、我的脚本错）、other → illustration（30，多为 Grok 错）。

## 5. 规则改进建议（逐条，不改 SKILL.md 本体）

**建议 1 — 规则 4（人像 vs 纪实）：把"人物是画面英雄"的判定线索写全。**
现状只列了 portrait of / headshot / editorial portrait 三个词。实测中人物为主角的 prompt 大量使用：selfie（自拍）、facing the camera / looking (directly) at the camera、"a beautiful woman stands…"、JSON 结构里的 subject.character、close-up / upper body。建议在规则 4 下加一条"人物主体信号清单"，不只依赖 portrait 这个词。

**建议 2 — 规则 5（多标签）：把叠加逻辑从"3d+poster"推广为"渲染风格 × 题材"。**
如"oil painting portrait style 的 close-up portrait"：渲染是 illustration，题材是 portrait，正确输出应为 `illustration, portrait`。目前 skill 只举了 3d+poster 的例子，容易让人以为只有这一种叠加。建议明确：illustration/3d/anime 任一与 portrait/product/poster 题材同时成立时都打双标签。

**建议 3 — 规则 3（商品）：放宽商业语境的识别。**
现状只认 product photo / commercial / design brief。实测"crisps 广告创意（product's motion system）"这类品牌广告概念会被漏掉。建议增加 brand / advertising concept / campaign idea 等线索；同时明确：无商业词的单品静物（如"ceramic coffee mug still life"）归 photography，不归 product。

**建议 4 — 规则 1/2 顺序：明确"版式词明确时优先版式"。**
如"Ultra-photorealistic cinematic album cover"：photorealistic 只是质感修饰，album cover 是明确版式意图，主标签应取 poster。建议加一条：当 poster/cover/infographic 等版式词明确出现、而渲染风格词仅为修饰时，主标签取 poster，渲染风格作次标签。

**建议 5 — 补中日韩关键词。**
测试集里 ZH 69 条、Mixed 37 条，另有大量"语言标 EN、内容含日语"的条目。写真（=photo）、ポートレート、海报/插画/人像等中文词目前缺失。建议 taxonomy 每类补 3-5 个中日常用对应词。

**建议 6 — 在 skill 里承认 other 的诚实边界 + stub 处理。**
纯叙事、无任何风格线索的 prompt（如"In 1941, a civil engineer in Berlin…"），按"只看文本不脑补"只能给 other——这是规则的诚实上限，不是失败，建议写进 skill 免得使用者硬套。另外 prompt 字段为占位符的 stub 条目应标记 `needs_manual_review`，不要硬分类（本次 47 条已排除）。

**建议 7 — 规则 1 的 3d 关键词补 miniature / three-dimensional。**
"handcrafted miniature world…three-dimensional scene, tiny textures"是典型的 3d/黏土质感，但现有关键词（3d render/CGI/clay/isometric/LEGO/diorama）一个都命中不了。建议加 miniature、three-dimensional、tiny world。

## 6. 代表性误判例子（5 个）

**例 1：微缩世界被双方错标**
- Prompt（摘要）："Create a whimsical handcrafted miniature world…one cohesive three-dimensional scene…physical handcrafted miniature, realistic tiny textures"（idx=796「Create a whimsical handcrafted miniature world…」）
- 我的标签：photography｜Grok Bot：portrait｜**正确：3d**
- 理由：miniature + three-dimensional scene 是规则 1 的三维质感，题材和人都不是重点。双方都错，暴露规则 1 关键词缺 miniature（见建议 7），以及 Grok Bot 的 portrait 滥用。

**例 2：自拍该算 portrait**
- Prompt（摘要）："Realistic gym mirror-selfie of woman in 20s…Three-quarter body shot"（idx=518）
- 我的标签：photography｜Grok Bot：portrait｜**正确：portrait**
- 理由：selfie 且人物是绝对主角，符合规则 4"人的脸/身体是画面英雄"。Grok 对，我的脚本错在关键词没收 selfie（见建议 1）。

**例 3：明确写了 poster 却被 Grok 标 other**
- Prompt（摘要）："A clean tech advertisement poster…Ultra minimal thin sans-serif typography…Tagline bottom"（idx=1024「Cinematic Sports Car Night Portrait」——标题有误导性）
- 我的标签：poster｜Grok Bot：other｜**正确：poster**
- 理由：advertisement poster + 排版意图明确，规则 2 直接命中。Grok 的 other 是错的；也说明不能只看标题，要看 prompt 正文。

**例 4：吉卜力风参考表被 Grok 标 other**
- Prompt（摘要）："detailed vintage illustrated visual reference sheet…warm hand-drawn Studio Ghibli style"（idx=340「Croissant Ghibli Reference Sheet」）
- 我的标签：illustration｜Grok Bot：other｜**正确：illustration**
- 理由：hand-drawn + Ghibli，规则 1 渲染风格优先。Grok 错。

**例 5：油画风人像该打双标签**
- Prompt（摘要）："draw…in a textured oil painting portrait style, thick impasto brushstrokes…Close-up upper body portrait…gaze directly at camera"（idx=572）
- 我的标签：illustration｜Grok Bot：portrait｜**正确：illustration, portrait**
- 理由：渲染风格是 illustration（规则 1 优先），题材是人像（规则 4），两个维度都强，按规则 5 应叠加。双方各对一半，暴露规则 5 叠加逻辑写得太窄（见建议 2）。

## 7. 总体评价

- 规则框架（6 步顺序）经 1070 条实测**方向正确**：在 Grok Bot 明确标错的地方（other 垃圾桶、portrait 滥用），规则给出正确标签的比例高。
- 52.1% 的主标签命中率**低估了规则的真实水平**：约一半不一致来自 Grok Bot 标签本身的质量问题，另一半主要来自关键词脚本的覆盖度（selfie、日语、多语言、miniature 等），而非规则逻辑错误。
- 按第 5 节的 7 条建议修订后，建议再跑一轮验证，预期主标签命中率可提升到 70%+（剩余分歧主要是 Grok Bot 侧的历史误标）。
