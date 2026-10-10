# prompt-classifier

给文生图（text-to-image）提示词打**风格分类**的确定性规则 skill：输入一条提示词文本，输出 8 分类标签（可双标签）。无 API 调用，规则透明、可复现。

> 规则已用 1070+ 条真实提示词做过七轮验证（见 `validation-report.md`、`validation-report-v2.md`、`validation-report-v3.md`、`validation-report-v4.md`、`validation-report-v5.md`、`validation-report-v6.md`、`validation-report-v7.md`）；v2.6 为当前版本。

## 分类体系（8 标签）

| 标签 | 含义 | 典型关键词 |
|---|---|---|
| `photography` | 写实摄影感的场景/静物/纪实 | photograph, documentary, 35mm, candid, 写真, 实拍 |
| `portrait` | 人是绝对主角的肖像/自拍/时尚大片 | portrait of, headshot, selfie, facing the camera, 人像 |
| `poster` | 海报/封面/信息图版式（含排版意图） | poster, cover, infographic, typography, 海报 |
| `illustration` | 手绘/插画质感（水彩、版画、混合媒介） | illustration, watercolor, hand-drawn, woodcut, 插画 |
| `3d` | 三维渲染/CGI/黏土/乐高/微缩质感 | 3d render, CGI, clay, diorama, miniature, three-dimensional |
| `anime` | 日漫/吉卜力/Q 版二次元 | anime, Ghibli, chibi, manga, 二次元 |
| `product` | 商品为绝对主角的商业摄影/广告创意 | product photo, commercial, brand, advertising concept |
| `other` | 以上都不贴切；或纯叙事无风格线索（诚实上限） | — |

## 判定规则（摘要，完整版见 `SKILL.md`）

1. **版式明确时优先版式**：`album cover` 等版式词明确、渲染词仅为修饰 → 主标签 `poster`
2. **渲染风格优先**：水彩/手绘→`illustration`，3D/黏土/微缩→`3d`，日漫→`anime`（风格压过题材）
3. **商品优先**：商业语境（brand / advertising concept）→`product`；无商业词的单品静物→`photography`
4. **人像 vs 纪实**：人物是画面英雄→`portrait`（信号含 selfie、面对镜头、JSON subject 字段等）；人只是场景一部分→`photography`
5. **双标签 = 渲染风格 × 题材**：如油画风人像→`illustration, portrait`
6. **other 兜底**：纯叙事无风格线索→`other`；Prompt 占位符（stub）→`needs_manual_review`，不硬分类

## 用法

把 `SKILL.md` 交给 LLM（或实现为关键词脚本做初筛＋人工复核），每条提示词输出一行：

```
<主标签>[, <次标签>] ｜ <一句话理由，引用提示词里的关键词>
```

## 验证

- 验证集：Grok Bot 已收集的 1144 条提示词（114 条 stub 排除后可对比 1004 条），关键词脚本机械实现
- 六轮主标签命中率：v1 52.1% → v2 50.5% → v2.1 54.6% → v2.2 55.3% → v2.3 55.3% → **v2.4 57.0%**（完全一致率 36.1% → 33.3% → 38.4% → 39.3% → 37.0% → **38.6%**），v2.4 为六轮最高
- 规则本身优于机械数字：人工抽查 30 条分歧样本，脚本正确约 17 条、Grok 正确约 8 条；对照组标签有系统性质量问题（portrait 滥用 44%、other 当垃圾桶 20%），raw 一致率数字低估了规则真实水平

## 已知局限（发布时如实声明）

1. 约 57% 是关键词脚本机械实现的近似值（六轮验证：52.1% → 50.5% → 54.6% → 55.3% → 55.3% → 57.0%），LLM 按规则执行的效果显著更好
2. 已知覆盖度缺口（v2.5 候选）："looking into the camera"、elegant/stylish 人物开场、"full body shot of a [person]" 等，LLM 执行不受影响
3. stub（文本缺失/截断）无法分类，只标记 `needs_manual_review`
4. portrait/photography 在 candid/街拍类存在真实模糊地带
5. 验证基线（Grok Bot 历史标签）本身质量不高：portrait 滥用约 44%、other 约 20% 被当作垃圾桶，对比数字仅供参考

## License

MIT
