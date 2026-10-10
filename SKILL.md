---
name: prompt-classifier
description: >
  Classify a text-to-image prompt into the collector's Style taxonomy
  (photography / portrait / poster / illustration / 3d / anime / product / other).
  Use whenever a prompt needs a Style label: the Muse 提示词收集器 collection
  rounds, Notion gallery writes, or any prompt-organizing task. Multi-label
  allowed (Notion Style is multi-select). Deterministic rules + real examples;
  no API calls needed.
---

# Prompt Classifier — 生图提示词风格分类（v2.6）

> 修订依据：2026-10-10 对 Grok Bot 已收集 1144 条的七轮验证报告
> （`validation-report.md` / `-v2` / `-v3` / `-v4` / `-v5` / `-v6` / `-v7`），v2.6 补中文版式词
> 与人物信号变体（杂志/报刊/版式、confident 开场、Prompt 👇 前缀剥离）。

## Purpose

给一条文生图提示词打 Style 分类标签。标签集合固定为 8 个（与 Notion「Muse 提示词收集器」数据库的 Style 多选字段一致）：

`photography` `portrait` `poster` `illustration` `3d` `anime` `product` `other`

## Taxonomy（定义＋真实例子，均来自已收集数据；★=边界案例）

- **photography** — 写实摄影感的场景/静物/纪实。关键词：photograph, documentary, 35mm, realistic, studio lighting, candid, landscape, 写真（日）, 实拍/生活照/纪实（中）。
  例：纪实渔贩（鱼贩在冰上摆鲭鱼）、便利店夜景、世界时钟墙、日间清亮餐车广场生活照（中文"写真"）、洪堡企鹅新闻纪实（"真实新闻纪录片摄影"）。
  ★ 无商业词的单品静物归这里不归 product（如 ceramic coffee mug still life）。
- **portrait** — 人是绝对主角的肖像（面部/半身特写、自拍、时尚大片）。关键词：portrait（bare，如 film portrait / comic-book portrait）, portrait of, headshot, editorial portrait, selfie / mirror selfie / 自拍 / 自撮り / セルフィー, facing the camera, looking at / into the camera, toward the camera, close-up / upper body（指人）, subject.character（JSON 结构）, 人像/肖像（中）, ポートレート（日）。
  例：红光人像（白光束划过脸）、戏剧性女性肖像、低角度棒棒糖女孩时尚大片、浴室镜面自拍（selfie→portrait，Grok 曾误标 other）、JSON 结构 mirror selfie（subject 字段即人物主体信号）、狮子旁女人面部 close-up（人脸绝对主角）、facing the camera 的男士全身像。
  ★ 油画风 close-up 人像 → `illustration, portrait` 双标签（见规则 5）。
- **poster** — 海报/封面/信息图版式，含标题文字排版意图。关键词：poster, cover（限缩见下）, infographic, exploded view diagram, tagline, headline, layout（需设计语境）, 海报/封面/信息图/杂志/报刊/版式（中）, ポスター（日）。
  注意：`typography` 单独出现不触发 poster（如瓶身标签文字）；需同时有 poster/cover/infographic/layout/headline 或明确的排版意图。
  `layout` 在渲染质检语境（如 "Consistent layout & sharp text" 手机 UI 渲染清单）中不触发 poster。
  `cover` 仅在 album cover / magazine cover / book cover / cover design 等复合词中触发 poster；
  "cover art quality"（如 "graphic-novel cover art quality"）是质感描述，不是版式意图，不触发。
  例：VR 耳机爆炸图海报、宠物联名海报、Y2K 杂志封面风、科技广告海报（advertisement poster＋排版意图，标题有误导性时仍以正文为准）、照片→手绘 sketch 杂志风页面（illustration, poster）。
  ★ "Ultra-photorealistic cinematic album cover"：photorealistic 只是质感修饰，album cover 版式意图明确 → 主标签 poster，photography 作次标签（见规则 1）。
- **illustration** — 手绘/插画质感（水彩、扁平插画、混合媒介、版画）。关键词：illustration / illustrated, watercolor, hand-drawn, ink, woodcut, sketch, drawing（仅名词搭配，如 line drawing / drawing style / children's book drawing / pencil drawing）, art nouveau, MS Paint, comic-book style, ukiyo-e / 浮世绘, 插画/插图/手绘（中）, イラスト（日）。
  注意：`drawing` 作动词或习语时不触发 illustration（如 "drawing a traditional wooden bow" 拉弓、"drawing the eye / drawing attention"）。
  例：成都吃货暴走地图（插画风城市美食地图）、吃西瓜的手绘插画风女孩、粉红底木刻风小狗（hand drawn crosshatching＋woodcut）、吉卜力风参考表（hand-drawn Studio Ghibli style，Grok 曾误标 other）。
  ★ 水彩画的人像也是 illustration 不是 portrait（风格优先）；油画风人像 → `illustration, portrait`。
- **3d** — 三维渲染/CGI/黏土/乐高/微缩质感。关键词：3d（bare，如 3D infographic）, 3d render, CGI, clay, isometric, LEGO, diorama, miniature, three-dimensional, tiny world, 三维/黏土（中）。
  注意：中文`立体`仅在"三维立体/立体渲染/黏土立体"等复合词中触发 3d；"立体感/层次感"指画面层次，不是三维渲染，不触发。
  例：夸张 3D 漫画像角色、等距漂浮国家地图透视、乐高小人镜面自拍、手作微缩世界（miniature＋three-dimensional scene，双方曾误标，Grok 标成 portrait）、毛毡青蛙沙滩（felt 手作质感）。
  ★ 3D 城市旅行指南信息图 → `3d, poster` 双标签。
- **anime** — 日漫/吉卜力/Q 版二次元风格。关键词：anime, Ghibli, chibi, manga, 二次元/动漫/漫画（中）, アニメ（日）。
  例：旅行照转吉卜力风明信片、梦幻 Q 版拼贴肖像、trending anime art style（confident line-work）。
- **product** — 商品为绝对主角的商业摄影/广告创意。关键词：product photo, campaign（需搭配产品词，如 footwear campaign / sneaker advertisements）, design brief, brand, advertising concept, brand campaign idea, 商品/商拍/广告（中）。
  注意：`commercial` 仅在 product/brand/advertisement 上下文中触发 product；"commercial beauty photography" 是摄影流派，不算。
  例：儿童玩偶设计 brief、拉面爆炸分解图、黑香水低角度广告大片、酸奶广告 commercial（Grok 曾误标 portrait）、脆片品牌广告创意（product's motion system）。
  ★ 无商业词的单品静物 → photography，不是 product。
- **other** — 以上都不贴切时才用（字体艺术、梗图、纯模板），**以及纯叙事、无任何风格线索的 prompt**（如 "In 1941, a civil engineer in Berlin…"）——这是"只看文本不脑补"原则下的诚实上限，不是分类失败。
  例：米粒刻字微距、浴缸里的动物。

## 预处理（分类前先做）

- **剥离负面提示**：先删除 `Negative Prompt` / `负面提示` / `NEGATIVE` 等段落及其内容，
  以及 JSON 结构中的 `forbidden: [...]` 列表（功能等同负面提示），再匹配关键词。
  负面提示里常出现 "cartoon, anime, illustration, blurry" 等词，会污染分类（S37、S39 曾因此误伤）。
- **剥离否定语境**：删除 "not + 风格词" 结构（如 "not anime"、"not illustration"、"not CGI"、"not a poster"、"not cartoon"），
  再匹配关键词。否定词后的风格词是"不要什么"，不是"要什么"——"not doll-like, not anime" 不应判 anime（idx=549 曾因此误伤）。
- **剥离引导前缀**：删除 "Made with … / Try it … / Prompt 👇" 类引导语前缀后再分类。
  这类前缀会把真正的人物开场推出识别窗口（idx=1071/841 曾因此漏检）。

## Decision Rules（按顺序执行）

1. **版式明确时优先版式**：poster / cover / infographic 等版式词明确出现、而渲染风格词（如 photorealistic / cinematic）仅为质感修饰时，主标签取 `poster`，渲染风格作次标签。
   例："Ultra-photorealistic cinematic album cover" → `poster, photography`。
2. **渲染风格优先**：提示词明确写了水彩/手绘 → `illustration`；3D/CGI/黏土/微缩 → `3d`；日漫/吉卜力 → `anime`。风格词压过题材（水彩画的人像也是 `illustration`，不是 `portrait`）。
3. **商品优先**：商品为绝对主角的商业图 → `product`（商业语境含 brand / advertising concept / campaign idea）。
   反例：无商业词的单品静物 → `photography`。
4. **人像 vs 纪实**：人的脸/身体是画面英雄 → `portrait`；人只是纪实场景中的一部分 → `photography`。
   人物主体信号清单（不限于 portrait 这个词）：selfie / mirror selfie / 自拍 / 自撮り、facing the camera / looking at / into the camera / toward the camera、
   人物主语开场（如 "A beautiful woman stands…"、"A beautiful Japanese woman, 25–30,"、"A stunning 26-year-old Japanese woman"、
   "A 20–25 year old woman" 年龄句式、"Elegant, slender East Asian girl…" / "A stylish European woman…" / "A confident young adult male…" elegant/stylish/confident 开场、
   "a strikingly handsome …" 副词+形容词结构、人名主语如 "Ana de Armas …"、
   "a fashion model …"、"wide shot of a beautiful woman …"、"full body shot of a [person]"、"Main Subject: The person is placed…"、
   "the subject stands…" / "subject + 动词"结构——需人物语境，"The subject is a landscape" 这类不算）、
   JSON 结构中的 subject.character、close-up / upper body / three-quarter body shot（指人时）。
   多语言：中文 人像/肖像/写真集、日文 ポートレート / 自撮り。
   边界：candid / 街拍类——人物是抓拍的明确主体（如街头行人特写）→ `portrait`；人物只是街景中的点缀 → `photography`。
5. **多标签 = 渲染风格 × 题材**：最多 2 个，第一个为主标签。渲染风格（illustration / 3d / anime）与题材（portrait / product / poster）两个维度都强时叠加，
   不只限于 3d+poster 一种组合。
   例：油画风 close-up 人像 → `illustration, portrait`；3D 城市信息图海报 → `3d, poster`；手绘杂志风页面 → `illustration, poster`。
6. **other 兜底＋诚实边界**：实在不贴切才用 `other`，不要滥用；纯叙事、无风格线索的 prompt 给 `other` 是诚实上限。
   **stub 处理**：Prompt 字段为占位符（如 "full prompt in body"）的条目文本缺失，任何纯文本分类器都无解，
   输出 `needs_manual_review`，不要硬分类。

## Output Contract

对每条提示词输出一行：

```
<主标签>[, <次标签>] ｜ <一句话理由，引用提示词里的关键词>
```

stub 条目输出：

```
needs_manual_review ｜ Prompt 字段为占位符，文本缺失无法分类
```

示例：
```
portrait ｜ 主体是"front-facing cinematic portrait"，人脸为绝对主角
3d, poster ｜ "isometric diorama" 渲染 + 明确海报版式
illustration, portrait ｜ "oil painting portrait style" 渲染风格 × 人像题材
poster ｜ "advertisement poster" + typography 排版意图明确（标题有误导性，以正文为准）
```

## Operating Rules

- 只看提示词文本本身，不脑补图片内容；**不能只看标题，要看 prompt 正文**。
- 提示词含多语言时以视觉描述为准，不以语言分类；中日关键词见各分类词表。
- 拿不准时选更具体的标签，不要丢给 `other`；真拿不准（纯叙事无风格线索）才用 `other`。
- 批量分类时保持同一标准，前后一致。

## 变更记录（v1 → v2 草稿）

| # | 验证报告建议 | 落实位置 |
|---|---|---|
| 1 | 规则 4 人物主体信号清单（selfie/面对镜头/JSON subject/close-up 等） | Taxonomy portrait 关键词＋规则 4 |
| 2 | 双标签推广为"渲染风格 × 题材" | 规则 5（原只举 3d+poster） |
| 3 | 商品放宽商业语境（brand/advertising concept）；单品静物→photography | Taxonomy product/photography＋规则 3 |
| 4 | 版式词明确时优先版式 | 新增规则 1（顺序提前） |
| 5 | 补中日关键词 | 各分类 Taxonomy 词表 |
| 6 | other 诚实边界＋stub 标记 needs_manual_review | Taxonomy other＋规则 6＋Output Contract |
| 7 | 3d 关键词补 miniature / three-dimensional / tiny world | Taxonomy 3d 关键词 |
| ＋ | 例子扩充到每类 6–8 个真实例子＋边界案例标注 | Taxonomy 全部分类 |

## 变更记录（v2 → v2.1）

| # | v3 细化清单 | 落实位置 |
|---|---|---|
| 1 | 剥离负面提示（分类前删除 Negative Prompt 段落） | 新增「预处理」节 |
| 2 | typography 限缩（单独出现不触发 poster） | Taxonomy poster 关键词＋注意 |
| 3 | commercial 限缩（仅 product/brand/advertisement 上下文触发） | Taxonomy product 关键词＋注意 |
| 4 | 补 bare 关键词（3d、portrait 不要求后缀；toward the camera） | Taxonomy 3d/portrait 关键词 |

## 变更记录（v2.1 → v2.2）

| # | v4 清单 | 落实位置 |
|---|---|---|
| 1 | 预处理追加 `forbidden` 列表剥离 | 「预处理」节 |
| 2 | 人物主语信号放宽（同位语/多形容词结构、subject+动词） | 规则 4 人物主体信号清单 |
| 3 | product 补 campaign 商业语境（需搭配产品词） | Taxonomy product 关键词＋注意 |
| 4 | 中文`立体`限缩（仅复合词触发；立体感不算） | Taxonomy 3d 关键词＋注意 |
| 5 | illustration 补 art nouveau / MS Paint / comic-book style | Taxonomy illustration 关键词 |
| 6 | `cover` 触发限缩（仅 album/magazine/book/cover design 复合词） | Taxonomy poster 关键词＋注意 |

## 变更记录（v2.2 → v2.3）

| # | 内容 | 落实位置 |
|---|---|---|
| 1 | 补关键词：自撮り（日语自拍）、drawing、ukiyo-e / 浮世绘、landscape | Taxonomy portrait / illustration / photography 关键词 |
| 2 | 人物主语信号再放宽（fashion model 主语、wide shot 句式、Main Subject 开场） | 规则 4 人物主体信号清单 |
| 3 | candid / 街拍边界澄清（抓拍主体→portrait，街景点缀→photography） | 规则 4 边界说明 |

## 变更记录（v2.3 → v2.4）

| # | 内容 | 落实位置 |
|---|---|---|
| 1 | 剥离否定语境（"not + 风格词"结构先删除再匹配） | 「预处理」节 |
| 2 | `drawing` 限缩为名词搭配（line drawing / drawing style 等）；动词/习语不触发 | Taxonomy illustration 关键词＋注意 |

## 变更记录（v2.4 → v2.5）

| # | 内容 | 落实位置 |
|---|---|---|
| 1 | "looking into the camera" 补作 portrait 信号 | Taxonomy portrait 关键词＋规则 4 |
| 2 | 人物开场补 elegant / stylish 形容词、年龄句式、full body shot | 规则 4 人物主体信号清单 |
| 3 | "subject + 动词" 需人物语境（"The subject is a landscape" 不算） | 规则 4 人物主体信号清单 |
| 4 | poster 补 headline 版式信号；layout 需设计语境 | Taxonomy poster 关键词＋注意 |
| 5 | illustration 补 illustrated 形态 | Taxonomy illustration 关键词 |

## 变更记录（v2.5 → v2.6）

| # | 内容 | 落实位置 |
|---|---|---|
| 1 | 中文版式词补"杂志/报刊/版式"（idx=850 高级黑白报刊社论风曾漏检） | Taxonomy poster 关键词 |
| 2 | 人物开场补 confident 形容词、"a strikingly handsome" 副词+形容词、人名主语 | 规则 4 人物主体信号清单 |
| 3 | 预处理追加引导前缀剥离（"Made with … / Try it … / Prompt 👇"） | 「预处理」节 |
