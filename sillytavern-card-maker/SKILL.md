---
name: "sillytavern-card-maker"
description: "为 SillyTavern（酒馆）制作与修改角色卡，尤其基于「酒馆助手(JS-Slash-Runner/TavernHelper) + MVU 变量框架(zod) + 提示词模板(ST-Prompt-Template/EJS) + 小白X」的高级变量卡：变量结构设计、世界书条目（变量列表/更新规则/输出格式/initvar）、酒馆正则（含闸门校验）、状态栏与交互式前端界面美化、创角/开局表单、内嵌脚本，并打包成含内嵌脚本/正则/世界书的完备角色卡（PNG/JSON），附带校验与玩家安装说明。只要用户提到酒馆、角色卡、写卡/做卡/改卡、MVU、变量卡、状态栏、前端界面、创角表单/开局表单、酒馆正则、酒馆助手、小白X、提示词模板、EJS 动态提示词，或要求把创意做成可导入 SillyTavern 的卡，就必须使用本技能。"
metadata:
  title: "sillytavern-card-maker"
  category: "Design/sillytavern-card"
  keywords: ["sillytavern","角色卡"]
---

# SillyTavern 角色卡制作

目标产物：一个**卡工程目录** + 由 `scripts/build_card.py` 打包出的**完备角色卡**——玩家只需导入一个 PNG/JSON，卡内已自带世界书、正则脚本和酒馆助手脚本；再安装两个开源扩展即可游玩（`酒馆助手`、`提示词模板`）。

## 技术栈四件套

| 组件 | 开源仓库 | 作用 | 谁需要安装 |
|---|---|---|---|
| 酒馆助手 (TavernHelper) | `N0VI028/JS-Slash-Runner` | 底座：在酒馆内运行脚本/前端界面、渲染消息中的 HTML 代码块为 iframe、提供 `getVariables`/`eventOn`/`injectPrompts` 等全套 API | 作者+玩家 |
| MVU 变量框架 (MagVarUpdate) | `MagicalAstrogy/MagVarUpdate` | 按楼层维护变量；解析 AI 输出里的 `<UpdateVariable>` 更新变量；自动在回复末尾附加 `<StatusPlaceHolderImpl/>` 占位符 | 作者+玩家（卡内嵌脚本自动引入） |
| 提示词模板 (ST-Prompt-Template) | `zonde306/ST-Prompt-Template` | 在提示词/世界书中用 EJS (`<%_ _%>`) 写条件逻辑，按变量动态发送提示词 | 作者+玩家 |
| 小白X (LittleWhiteBox) | `RT15548/LittleWhiteBox` | 另一套渲染+变量方案：代码块渲染、变量管理 2.0、小白板、剧情总结等 | 仅当用户点名小白X时 |

**默认技术栈**：酒馆助手 + MVU(zod) + 提示词模板（社区称"变量卡三件套"，示例卡与教程最完善）。只有用户明确要求"小白X"或其玩法需要小白X特性（剧情总结/小白板/OOC 四次元壁等）时，才走小白X路线；详见 `references/xiaobaix.md`。

## 工作流

### 0. 需求确认

先弄清（缺的可以问用户，但不要用一堆术语轰炸，用"玩法语言"问）：

1. **题材与玩法**：什么世界观、玩家扮演谁、核心乐趣是什么（恋爱养成 / 冒险 / 经营 / 系统流……）。
2. **变量需求**：需要记什么状态（好感度、时间、物品栏、任务、NPC……）。这决定一切，先想清楚。
3. **界面需求**：要不要状态栏？要不要可交互前端（点按钮买东西/接任务）？要不要**创角/开局表单**（填表选配置再开局，系统流卡几乎必备）？
4. **是否改现有卡**：是则用 `python scripts/extract_card.py <卡.png|json> <输出目录>` 先拆出卡内容再改（`card.json` 里自动生成的 `extensions_extra` 原样保留，重建时不会丢 `depth_prompt` 等字段）；改卡的维护流程与陷阱见下文「**修改与维护现有卡**」一章，动工前先读。

### 1. 建立卡工程目录

在会话工作区建立（`<卡名>` 用中文没问题）：

```
<卡名>/
├── card.json          # 角色卡本体字段（不含 extensions/character_book，打包时自动合并）
├── worldbook.json     # 世界书（ST 导入格式，含变量条目+正文档条目）
├── regex.json         # 酒馆正则数组（ST 导入格式；把 templates/regex-基础.json 与 regex-界面.json 合并成这一个文件）
├── scripts/           # 酒馆助手角色脚本，每个 .js 文件 = 一个脚本（文件名即脚本名）
│   ├── MVU.js         # 一行引入 MVU 本体
│   ├── 变量结构.js
│   └── ...
├── 界面/              # 可选：前端界面 HTML 源文件（供维护；打包时内联进正则）
│   └── 状态栏.html
├── tools/             # 可选：维护脚本（patch_*.py 幂等补丁、assemble_regex.py、audit_consistency.py、apply_cover_text.py），见「修改与维护现有卡」
├── meta.json          # 可选：{"scripts_disabled": [...], "script_buttons": {...}}
├── 说明.md            # 玩家安装/使用说明（用 templates/readme-template.md）
└── (底图.png)         # 可选：卡面图，打包 PNG 用
```

从 `templates/` 拷贝骨架再改，不要从零手写格式。

### 2. 内容构筑

角色卡字段、世界书正文档条目的写法与质量标准 → `references/content-craft.md`。

### 3. 变量体系（MVU zod）

顺序固定：**先想变量 → 写 schema → 写 initvar → 写三大变量提示词**。

1. `scripts/变量结构.js`：用 zod 4 写 `Schema` 并 `registerMvuSchema`。规则（幂等、`z.coerce.number()`、`z.record` 优于数组、`prefault`、用 `transform` 纠错而非拒绝……）→ `references/mvu.md`。
   数值设计三惯例：**"无上限"字段给大上限**（如 `num(0, 0, 999)`，防失控又能容纳长期成长）；**软阈值字段**（如精液容量 150ml）数值照常累积不硬截断、超限效果交给界面呈现（膨大/溢出）；**布尔字段容忍中文写法**（`preprocess` 把"是/否"等转 `true/false` 再 `z.boolean()`，否则 AI 写中文字符串会让整次更新被 catch）。
2. `worldbook.json` 中四个变量条目（名称与状态是硬接口，不能改名/改开关）：
   - `变量列表` — 启用，蓝灯，内容含 `{{format_message_variable::stat_data}}`
   - `[initvar]变量初始化勿开` — **必须禁用**（MVU 照样读取）
   - `[mvu_update]变量更新规则` — 按 schema 逐变量写更新条件
   - `[mvu_update]变量输出格式` — 固定模板（JSONPatch 格式），照抄 `templates/变量输出格式.yaml`
3. 多开局：在各 `alternate_greetings` 里用 `<UpdateVariable><initvar>…</initvar></UpdateVariable>` 块覆盖初始值。

### 4. 脚本

- `变量结构.js`（必需）：注册 schema。
- `MVU 引入`不需要单独文件——打包脚本会为 `scripts/` 里每个文件生成脚本条目；**MVU 本体的引入写成 `scripts/MVU.js`**，内容就一行 `import 'https://testingcf.jsdelivr.net/gh/MagicalAstrogy/MagVarUpdate/artifact/bundle.js';`。
- 可选：立即事件（`injectPrompts` + 世界书关键字联动）、事件监听（`Mvu.events.VARIABLE_UPDATE_ENDED` 纠错/回调）→ `references/mvu.md` 脚本节。

### 5. 正则

标准三件套 + 界面正则，模板在 `templates/regex-*.json`（改 id 即可用；组合规则见 `templates/README.md`）：

1. `[不发送]去除变量更新` — `仅格式提示词`，默认对所有楼层生效（`minDepth/maxDepth: null`）；若希望 AI 参考最近几楼的更新，改设 **`minDepth: 3`（或 4）**——深度自最新楼数起 0，写 `maxDepth` 会把方向弄反（历史坑点，见 `references/regex.md`）。
2. `[折叠]变量更新中` / `[折叠]完整变量更新` — `仅格式显示`，把更新块折叠成 `<details>`。
3. `[不发送]界面占位符` — `仅格式提示词`，把占位符从提示词中抹掉（不占 token）。
4. `[界面]状态栏` — `仅格式显示`，把占位符替换成前端界面（见下一步）。
5. （可选，推荐）`[闸门]变量块校验-放行` + `[变量]变量未更新警告` — 更新块缺失时给出可见提示；配套细节（READY 改名、数组顺序、界面正则配套改法）见 `references/regex.md` §⑥，模板 `templates/regex-闸门.json`。

正则字段语义、命名约定、语法坑 → `references/regex.md`。

### 6. 前端界面（前端美化）

三种档位，按需求选：

| 档位 | 做法 | 适用 |
|---|---|---|
| 纯文本状态栏 | 正则把占位符替换为 `💖 好感度: {{format_message_variable::stat_data.角色.好感度}}` | 最快 |
| 静态美化状态栏 | 占位符替换为 `<style>…</style><div>…宏…</div>` | 简单美化 |
| 交互式前端（推荐上限） | 占位符替换为 ```html 代码块，内含完整 `<head><body>`+JS：读变量、点按钮改变量、写"系统日志"变量供 AI 感知 | 游戏化玩法 |
| 创角/开场表单 | first_mes 放占位符 → `[开场]` 正则渲染 HTML 表单 → 提交（模拟发送或 `createChatMessages`）→ 协议条目让 AI 首轮 JSONPatch 初始化变量 | 系统流/经营/冒险卡（见 `references/opening-forms.md`） |

要点（细节 → `references/frontend.md`）：

- 前端以**消息内嵌 HTML 代码块**分发：代码块文本包含 `<head>`/`<body` 才会被酒馆助手渲染为无沙盒 iframe。默认**内联**进正则替换内容，卡自包含、不依赖 CDN 托管。
- iframe 里自带 `$`(jQuery)、`_`(lodash)、`z`(zod)、`toastr`、`Vue`、`YAML` 和全套 TavernHelper 函数；用 `await waitGlobalInitialized('Mvu')` 等 MVU 就绪，再用 `Mvu.getMvuData({type:'message', message_id: getCurrentMessageId()})` 读写本层变量，写完 `await Mvu.replaceMvuData(...)` 落盘。
- 玩家通过界面改变量时，把操作写进一个变量（如 `系统日志`），否则 AI 会看到"变量无故变化"并试图"修正"它。
- 样式规范：宽度自适应、禁用 `vh`/`min-height` 撑高、禁横向滚动条、深色浅色主题都要可读（用透明背景或自适应风格）。
- **响应式用 `@container` 查询**（PC/安卓双端一次搞定）：外层声明 `container-type: inline-size`，窄屏块 `@container (max-width: 420px)` 里改 `flex-direction` 与仪表方向；flex 子项**必须 `min-width: 0`**（`flex-basis: auto` 是换行错位的重灾区）。细节与坑 → `references/frontend.md` §7.0。
- **截图回归**：无头 Chrome 渲染整张卡的完整 HTML 极易崩（`Abnormal renderer termination`）——改用**隔离预览法**（只抽 `<style>` + 目标函数 + 极简外壳，几秒出图），并给用户发**两态对照**图。做法 → `references/frontend.md` §8.1。

### 6.5 女主状态栏：样式惯例与情景特效工程（实战沉淀，制作/改卡时遵守）

#### 女主状态栏样式惯例（工作流规则）

- **没有女主状态栏的卡** → 直接按「渊触按摩店」的女主小状态栏样式新建：参考 `templates/statusbar-女主参考.html`（快照）或卡工程原件（`agent work doc WB/渊触按摩店/界面/状态栏.html`，**取最新版为准**）。
- **已有女主状态栏的卡** → **先按原卡的样式继续改**，不要擅自换整套风格；同时把原卡预览图（优先从卡工程 `输出/预览-*.png` 取，没有就本地截图）发给用户，由用户决定是否更换为新样式。
- 样式特征清单（复刻/新建时逐项对齐）：
  - 粉紫浅色卡式（`--bg/--card/--line` CSS 变量 + 透明背景），卡片可折叠（点标题展开/收起）；
  - 字体层次：楷体标题（大字号 + 宽字距）／宋体名字（14px 600）／衬线数字（`tabular-nums`）；正文无衬线 11~13px 分级；
  - 数值条：彩色渐变胶囊条（`.m-*` 类，高约 7px）+ 右侧数字；部位级细条用 46×5px 迷你条（`.yc-mini`）；
  - 区块骨架：标题行（店铺/当前预约/女主/触手们…）→ 卡片分组 → 底部小字说明；区块标题用固定词（如「店主」），卡内显示 `{{user}}`；
  - 组件命名统一 `yc-*` 前缀；卡片特效见下。

#### 情景特效（状态 → 视觉映射，class toggle 驱动）

| 状态源 | 特效 |
|---|---|
| 忙碌/服务中 | 卡片转灰（`.yc-busy`：背景/边框/文字降饱和，保留工作痕迹特效） |
| 高亢（快感/发情 ≥ 阈值） | 渐粉底 + 内晕呼吸（`.yc-blush`） |
| 接近极限（≥ 更高阈值） | 高光汗珠沿细湿痕缓滑（`.yc-sweat`），不要画成一排蓝点 |
| 沾污（沾精等） | 少量液柱缓慢下淌（`.yc-cumdrip`：4 道、靠边分布、峰值透明度 ≤0.8） |
| 工作痕迹（触手服务中） | 湿光斑呼吸 + 底缘垂挂液滴（`.yc-gloop`：滴 + 2px 拉丝，"藕断丝连"） |
| 数值超阈值 | SVG 组件形态变化（膨大 `scale` / 倒灌填充 / 溢液） |

流体特效设计原则（都踩过坑）：

1. **克制优先**：特效不能遮挡信息——液体滴落宁少勿多（4 道封顶）、靠边分布、低透明度且快速淡出，保证名字/数值可读；
2. **参考现实质感**：粘稠液体（精液/黏液）是"挂着慢慢淌"——长时长（5s±）+ 粘滞缓动 `cubic-bezier(.5,.05,.5,.95)` + 上窄下圆的液柱/拉丝造型；不要画成高速滴答的水滴（"渗水感"会被用户打回）；
3. **相位错开**：多滴用不同 `animation-delay`，不要同步摆动；
4. `prefers-reduced-motion: reduce` 媒体查询**全量覆盖**所有新动画。

#### 工程模式（保稳定、防翻车）

- **幂等渲染**：结构层用 **signature 缓存**——把影响结构的数据 JSON 化比较，未变化直接 `return`（避免每秒重建 DOM 导致动画重置、展开状态丢失）；数值层每帧只更新样式与文本。signature **只含结构字段**（状态/标签/列表），快感这类每秒跳动的值不参与（否则缓存永远失效）。
- **SVG 组件**：`viewBox` 固定坐标绘制；液面用 `clipPath` 裁剪 + `rect` 的 `y/height` 随数值更新（配 `transition: y, height` 平滑）；"膨大"用 `transform: scale()`（`transform-origin: center`）；多态切换直接切 class/`display`，**不要重建 SVG**；`clipPath/gradient` 的 id 在同一文档内必须唯一。
- **布局韧性**：图片类图元 `flex-shrink: 0`；文本栏用 `flex: 1 1 0%; min-width: …`——**不要用 `flex-basis: auto`**，否则某一帧文案变长（如超量提示）会把并排布局挤到换行（实测坑）；长文本容器加 `overflow-wrap: anywhere`。
- **回归测试工具链**（建议每张卡都配）：`tools/make_demo.py` 生成带"阶段切换按钮"的 `界面/_demo.html`（mock 数据 + 各阶段场景）→ headless Chrome 截图（`--headless=new --screenshot`）→ 用 PIL 裁切特写核对细节 → 逐版本存档（`输出/预览-*.png` 兼顾交付）。每改一次界面跑一轮，动画/布局问题本地即暴露。

### 6.6 女主状态栏：腔内差分图与外部世界书兼容（实战沉淀）

做女性主角的「子宫 / 小穴 / 腔内」断面差分状态栏时，按这一节执行；细节与代码片段 → `references/female-statusbar-cavity.md`。

- **单图优先，不默认拆腔**：用户说「独立腔内」时先确认是「一张图看不同状态」还是「两个视口分别判定」。多数情况要的是单图全断面；拆成子宫腔/小穴腔后规则、ml、插入判定都容易打架，用户下一轮很可能要求合回去。
- **先做状态 × 插入矩阵**：`标准/排卵/危险/受精/妊娠/临盆/精液少/精液中/精液多` × `未插入/插入` = 18 帧；每帧固定 id，下拉与自动都必须覆盖全部 18 张，不能只用其中几张。用纯函数跑 9×2 断言 `missing=[]`。
- **背景处理**：源图可能是白底或**黑底**；从边框做 flood fill 去底，阈值约 60~70，保留内部深色描线。不要全图按颜色抠，也不要把紧裁切当边框问题。
- **边框不能压图**：视口不要用 `border`（占内容盒会盖住图片），改用外圈 `box-shadow: 0 0 0 1px …` + 内衬 `gutter`（桌面 6px / 窄屏 4px）；视口尺寸 = 帧尺寸 + 2×gutter，图片偏移再加 gutter。源素材本身被裁掉的部分，CSS 补不回来。
- **窄屏按宽度不按 UA**：`matchMedia('(max-width: 600px)')`，图片、视口、偏移、stage 高度一起 ×0.5；不要用 `/Android/`。
- **状态归一**：外部独立世界书可能写「正常/发情/易孕/着床/怀孕/分娩/内有精液/大量精液/灌满/溢出」等，不要只认自己的枚举。合并读取 `腔内.状态`、`外部状态`、`状态文本`、`当前状态`、`腔状态`、`精液状态`、`子宫状态`、`小穴状态`、`宫内状态`、`阴道状态`、`体内精液.状态/描述` 等，关键词映射 + 回退。
- **插入 vs 内射**：`插入/抽送/性交/抽插/挺入` 才选 `_插入` 图；`内射` 是事件，选精液图，不要因为“内射”两字继续显示插入。
- **判定只选一种模型**：单腔版统一按 `总ml`，受孕看「有内射 + 排卵/危险期」，不区分具体子宫/阴道；拆腔版才用「宫颈口闸门 + 宫腔ml/阴道ml」，且只能有一个 ml 权威源。用户说“不要区分”时，变量/规则/UI 三处一起回单腔。
- **强制更新协议**：外部「变量强制更新 4.0」是审计层，不要原样导入 12K 通用文本；做卡专用精简条目，引用 `[mvu_update]变量更新规则` 作为唯一字段源，输出 `<BianLiang> + <Analysis> + <JSONPatch>`，无变化 `[]`。
- **同步清单**：自省标题 `getUserName() → substituteParams('{{user}}') → {{user}}`；年龄从 Persona/档案模板同步且不覆盖剧情年龄；UI 标题/按钮/下拉/caption/reason/JS key 一起改。
- **验收**：`check_card` 0 错误、`zod_validate.cjs` LOST=0、`verify_png` 三源一致、18 帧测试、别名测试、800px/420px 截图各一张。

### 7. 打包与自检

```bash
python scripts/check_card.py <卡工程目录>          # 先校验（也可校验成品卡）
python scripts/build_card.py <卡工程目录> --image 底图.png   # 无底图则只出 JSON
```

- `build_card.py` 把 `card.json` + `worldbook.json`（转成 `character_book` 内嵌）+ `regex.json` + `scripts/*.js` 合并为 `输出/<卡名>.json` 与 `.png`（`chara`/`ccv3` 两个 PNG 元数据块），另导出独立的 `世界书.json`、`正则.json` 备用。玩家导入卡后：世界书随卡自动建立；正则会在首次使用时弹窗询问是否启用（ST 的 scoped regex 机制）；酒馆助手脚本直接从卡读取。
- `check_card.py` 检查格式硬伤（字段缺失、变量条目名称/开关、`[initvar]` 未禁用、正则语法、脚本引用、JSON 结构），报错信息直接可修。
- **无工程目录、只有一个 PNG/JSON 时的独立工具**（v9 沉淀，也适用于改现成卡）：
  - `scripts/zod_validate.cjs <card.json> [initvar_comment]` —— vm 沙箱加载卡内 ZOD、解析 initvar、**叶子级 LOST/GAINED 覆盖 diff**（LOST 必须 0）。需 `npm i zod js-yaml`（lodash 有垫片）。
  - `scripts/repack_png.py <base.png> <card.json> <out.png>` —— 只替换 `chara`/`ccv3` 两个 `tEXt` chunk，其余 chunk 原样保留（要求底图已带这两个 chunk）。
  - `scripts/verify_png.py <card.png> [source.json]` —— 回读 PNG，断言 `chara == ccv3 == 源 JSON`，并复核条目数与关键关键词。

### 8. 交付说明

用 `templates/readme-template.md` 写 `说明.md`，必须包含玩家侧要求：
装 `酒馆助手` 与 `提示词模板` 两个扩展（酒馆助手在扩展面板按 URL 安装；提示词模板安装地址 `https://codeberg.org/zonde306/ST-Prompt-Template`），开启酒馆助手的渲染开关，并说明玩法与命令。

可选增强：挂一个 `自动安装插件` 脚本（`import 'https://testingcf.jsdelivr.net/gh/StageDog/tavern_resource/dist/酒馆助手/自动安装插件/index.js'`），玩家缺依赖时弹窗一键安装；同类工具脚本（自动允许卡内正则、角色卡绑定预设等）见 `references/ecosystem.md`。发布前用 `check_card.py` 扫掉 localhost 等开发残留。

## 修改与维护现有卡（改卡是高频任务，比新建更容易翻车）

### 标准作业流程（每轮照做，清单见 `templates/维护检查清单.md`）

1. **备份 + 外部改动检查**：先建 `备份-<主题>前/`，拷入将动的文件（card.json / worldbook.json / 说明.md / 相关 scripts、界面、tools）与 `输出/` 成品。拷前先确认文件是否被**外部手改**过（看修改时间戳、grep 目标关键词）——用户可能自己动过工程文件（实测发生过：用户手动改了 schema/initvar/界面/测试并保持了一致），发现外部改动后**以文件现状为基准**叠加，不要按记忆/旧态覆盖。
2. **写幂等补丁脚本**（`tools/patch_<主题>.py`），不要直接手改大 JSON：
   - 每条替换先 **assert 锚点恰好出现 1 次**，全部断言通过再落盘（改一半失败＝半坏状态）；
   - 脚本开头做幂等检查（检测到已打过则跳过），可安全重复运行；
   - worldbook 用 JSON 感知替换（load → 改 content → dump，保 CRLF）；card.json 用保形字符串替换（保原格式与结尾换行）；`界面/*.html` 改完后重跑 `tools/assemble_regex.py` 重组 regex.json。
   - **变量源文件与世界书条目是双份内容**：`tools/content/initvar.yaml` → `[initvar]` 条目、`tools/content/变量更新规则.yaml` → `[mvu_update]变量更新规则` 条目，改完源文件**必须重跑对应同步补丁**（补丁脚本里带"内容同步"段、幂等跳过），漏同步 = 成品卡里还是旧规则（实测被 `audit_consistency.py` 的"双份一致"检查抓出过 3 项失败）。
   - 每轮交付把 **character_version +0.1**（字符串亦可用 1.10），并让 `audit_consistency.py` 断言该版本号——成品 PNG/JSON 忘了重建时能立刻发现。
3. **改完必跑**（顺序固定）：`check_card.py <工程>` → `build_card.py <工程> --image 底图.png` → `check_card.py 输出/<卡名>.json` → `tools/audit_consistency.py`（工程自带的扩展审计）。
4. **收尾**：版本号 +0.1；`说明.md` / creator_notes 同步；新机制往 audit 关键词表追加（防回归）。
5. **"一个变量改四处"纪律**（本项目最高频的漏改）：动一个变量时，**schema / initvar / 更新规则 / 状态栏渲染**这四处通常都要同步——只改 schema 会"字段有了但界面不显示"，只改规则会"AI 写进了但被剥离"。若该卡没有工程目录（只有一个 PNG），先按 `references/card-format.md` §5.1 定位 schema 再动手。改完**逐处 grep 变量名**确认四处都在。

### 锚点与格式陷阱（被坑过，先看再改）

- **全角/半角冒号**：中文文本里「不空转：」与「不空转:」是两个字符——锚点必须先从文件 dump 原文核对，勿凭记忆写。
- JSON 里换行是 `\n` 两个字符的转义；往 JSON 字符串插内容时注意写进去的是字面 `\n` 还是真换行。
- 分隔条等号按原文（`===世界设定结束===` 是 3 个等号）；uid 数字字符串键勿数错。
- 保持文件原行尾（CRLF/LF）、BOM、结尾换行习惯；dump 前后 diff 验证。

### 世界书条目增删的位置惯例

- 新条目插在分隔条「===世界设定结束===」（3 个等号；该条目 content 为空，分隔文字就在 comment 里）**之前**；分隔条 uid 让位重编（如 40→42），保持显示有序。
- uid 取空闲号；新条目字段集直接复制同位置同类型现成条目（constant / position / depth / role 一致），只改 uid、comment、content、order。

### 变体起点：优先做成开局，不要维护第二张卡

想做「另一条线 / 另一种起点」的变体时，**直接加 `alternate_greetings` 开局**（各自带完整 `<initvar>`），不要另开工程维护第二张卡——实测：独立变体卡最终被放弃、内容合并回主卡开局。开工前先问用户要哪种。

### 常驻 vs 关键词触发（省 token 的大头）

- **一次性/流程性条目**（如创角表单协议）不应常驻：改绿灯关键词触发（`key` 如 `["开局配置", "OpeningConfig"]`），提交表单后自动生效数轮，不再每轮占位（实测省约 1900 字/轮）。
- **大块背景设定**（世界历史、国家地理等千字以上条目）默认**关键词触发**、不要常驻：key 挑"只会因剧情被提起"的具体专有名词（如 `明斯克条约/世界大战/联合国/美国`），避免 `历史/战争` 这类泛词（容易伪常驻、白费 token）。实测一张卡把两条背景设定（约 3160 字）改为触发后，每轮常驻从 ~15850 字降到 ~12950 字。
- **与变量系统重复的原条目**（结算数值、生成概率表等）做"单一权威源 + 短指针"精简：数值规则以「变量更新规则」为准，原条目改引用式并保留独有信息（示例、流程、生态细节）；修文案时注意跨条目**专有名词一致性**（实测抓到过"大众点评 vs 夜莺之声"两个应用名分裂），把这类词加进审计关键词。
- 转换时必须**补一个小的「兜底」常驻条目**，保住直启 / 老存档等边缘路径（如「未用表单直接开聊 → 用 [initvar] 默认值」，约 150 字）。
- 跨幕关系、长期状态相关的条目**必须常驻**，不能绿灯。

### 赘述体检与去重（`scripts/scan_duplicates.py`）

- 体检：行级精确查重 + 句级模糊查重（difflib ≥0.55），跨条目与条目内各扫一遍，附条目字数排行（常驻/条件分开）。
- 处理原则：**单一权威源 + 短指针**——数值只留「变量更新规则」、定义只留一个机制条目，其余改「见『XXX』」；**咒语型短指针**（如 `见「遭遇导演术」`）是执行锚点，只压短不删；例子与味道句不删；按节省字数排优先级执行。

### Token 浪费审计方法学（v8 沉淀）

`scripts/scan_duplicates.py` 适合抓显而易见的整行/整句重复。但真正贵的浪费有三类，行级查重抓不到：

1. **跨条目 12-gram 重叠**：提取所有 `enabled && constant` 条目，对每对做去掉空白的 set intersection，长 n-gram (n=12)。重叠 > 25 处基本是逐字重复。`scripts/inspect_overlap.py` 实现。
2. **条目内部自覆盖**：对单条目做 n-gram 计数（cnt ≥ 2 的 gram 在文中占多少字符）。20% 以上说明有内部重复（如同义段、列表项同前缀）。`scripts/internal_dup.py`。
3. **ZOD 往返测试**：`Schema.parse(initvar).parse()` 对比前后叶子集合。被 ZOD 剥离 = 字段无人接收；值被改写 = initvar 写了与 schema 不符的字符串。`scripts/roundtrip_check.py`。

发现后处置分类：

| 类型 | 处置 | 例 |
|---|---|---|
| 🔴 功能缺失（如某条目 `enabled:false`） | 立即启用，否则整条规则从未注入 | `<扶她系统>` 被禁用 |
| 🔴 数据结构错（缩进/重键/YAML 类型） | 修，让 ZOD 通过 | 同一字段出现两次、缩进错位、initvar 是 object 但 ZOD 期望 string |
| 🔴 数值不一致（如「非洲女性 80%」但世界无非洲） | 改至与世界观一致 | 改成「妃洲女性与混血 80%」 |
| 🟡 Token 重复（同一段跨条目） | 抽离到权威条目，其它改为短指针 | 「扶她生理/忍耐力」700 字符抽到 entry[29]，entry[2] 改"见扶她系统" |
| 🟡 元数据错（指向不存在的条目） | 改至真实存在的条目名 | "世界书中的「世界观（分级）」" → "世界书<分级>" |
| 🟡 UI 与后端不同步（状态栏 mockData 缺字段） | 补齐 mockData 兜底 | 主角/痕迹 12 字段在 mockData 全量存在，0 token 成本（仅格式显示） |

**主动拒绝的优化**：每条候选理由要写"为什么不修"，避免后续维护者抓瞎。常见反模式：

- 强合"看起来重复"的规则段（实际每段带类型专属时机条款）
- 把 `constant + keys` 改 `keys`-only（keywords 与变量名重叠会偶发不注入）
- 删除 disabled `[initvar]` 内的注释示例（disabled 文本 token 成本 = 0，留着讲解更有价值）
- 把 initvar YAML 与规则散文强行合并（结构化数据 vs 散文语义不同）

### 关键纪律补充（v8 round-2 沉淀）

- **不要往 `system_prompt` 塞规则**：SillyTavern 的 `system_prompt` **会覆盖**酒馆默认主系统提示词（其带 `{{persona}}` 等内置宏与酒馆插件 hooks），塞自定义内容会让酒馆的内置能力（如角色档案感知、插件交互提示）失能。所有"硬性规则"应放在 `constant: true` 的世界书条目里。如果历史上有内容混进 `system_prompt`，**还原为空字符串**，信息挪到世界书对应条目（措辞可微调）。
- **initvar YAML 缩进的连锁反应**：给某字段加 nested 子键时，必须保证子键缩进 = 父字段缩进 + 2。如果子键缩进仍是父字段缩进，YAML 会把它们当父字段的同级键，父字段留作 null。**典型表现**：脚本说"应用了补丁"，但 ZOD 校验仍报"expected object, received null"。
- **同步 ZOD 缓存副本**：card.json 的 `tavern_helper.scripts[1]` 是 schema 真源；本地调试用的 `_zod_full.cjs` 是离线副本（去掉 `import`、`$(() => …})`、stub `_` 和 `registerMvuSchema`）。**改 card.json 后必须重跑 `sync_zod.py`**，否则 round-trip 校验是过时的，会给出假阴性。
- **术语对齐要给可变量化的占位符**：用户称谓/专有名词（如"女男爵"）在 5 处以上散落时，应改用 `{{user}}` 占位符，让酒馆自动替换为用户档案中的实际姓名/称谓。**例外**：NPC 称呼的具体词汇（如"夫人/女男爵"两个并列选项）不可改占位符，否则会出现"称呼{{user}}为'夫人'或'{{user}}'"的语法怪象。
- **状态栏 UI 术语与 JS 内部标识符要同步**：用户的术语改动（如"本我→自省"）必须同时改 UI 可见文本（菜单、标题、tab id、错误提示）和 JS 内部字符串（menu 数组、`renderedTabs` key、`currentMenu === "..."` 判断）。错一处 tab 就跳不到。
- **分腔室量化内射**：精液/液体类状态字段用「分腔室 ml 计」而非自由文本：阴道ml / 宫腔ml / 后穴ml / 口腔ml / 总ml + 最近注入时间 + 对象。ZOD 给数值字段加 `.transform(v => _.clamp(v, 0, MAX)).prefault(0)`。配合一条子规则说明流向（男性仅入阴道；扶她 ≥ 阈值时按比例入宫；时间衰减）。

### 新增「资源类」机制 & 历法粒度改造（v9 沉淀）

**新资源别塞进货币节点**：给一张已有 `金钱: {金币,银币,铜币}` 的卡加"粮草/木材/铁矿"等新资源时，**必须另开独立顶层节点**，不要加进 `金钱` 的子键。原因是金币渲染通常走 `Object.entries(appData["金钱"])` 的泛型循环 + `coinClasses` 映射（`金币→coin-gold` 等）——多出的第 4 个键会落进 `else` 分支渲染成裸标签，还可能污染货币进位/换算逻辑。独立节点则可完全自定义渲染。
落地时按「一个变量改四处」同步：`initvar` 顶层键 → ZOD `z.object` → 世界书规则（定义/推导公式/更新时机/阈值判定）→ 状态栏渲染块 + CSS + `mockData`。渲染块位置锚定在货币块之后（用 `secDiv.innerHTML += moneyHtml;\n container.appendChild(secDiv);\n }` 这类唯一收尾行做插入点）。

**"可支撑天数"类派生值要处理除零**：`Math.floor(总量 / 日均消耗)` 在 `日均消耗 = 0` 时会得到 `Infinity`，须显式判空并显示 `∞`；同时把 `Math.min(days, 9999)` 兜住极端值，避免 UI 撑破。

**三色态用 CSS 变量而非硬编码**：状态色抽成 `--color-<res>` / `--color-<res>-warn` / `--color-<res>-out`，渲染时只切 class（`-ok` / `-warn` / `-out`），便于后续统一换肤。

### 资源↔货币的兑换通道：缺资源时用钱买（v10 沉淀）

给"粮草/木材/铁矿"这类**非货币资源**加"缺了就用钱买"的机制时，最容易翻车的是**双栏记账打架**。正确做法：

1. **双栏同时记账**：这笔交易要**同时**写两处——
   - 资源消耗栏：`本周消耗.粮草.明细.采买粮草 = 采购量`（**同时资源库存入库**，不能只记消耗不入库）
   - 货币消耗栏：`本周消耗.货币.明细.采买粮草 = 采购量 × 单价`（**同时从金钱扣款**）
   - **关键**：因为资源与货币**同时**增减，所以**不需要改写"净额"公式**——净额（资源净额 / 货币净额）会自然反映这笔交易。若贪图省事只扣钱或只记消耗，净额就会算错。
   - 明细键名两栏取**同一个名字**（如都叫"采买粮草"），便于阅读时对照。

2. **定价锚定世界观的物价条目，不要拍脑袋**：先从世界书的物价条目取基准区间（如粮价锚定黑麦/燕麦的"5~8 铜币/公斤"），再按行情分档（常态 / 战时 / 饥荒），并写明"大量采购可议价"。定价档位要给出**具体数值区间**，模型才能自洽。

3. **补足量要有安全余量**：只补到"刚好够"会导致下轮立刻又告急。算法用 `缺口 = max(0, 阈值天数 × 日均消耗 − 库存)`，再加一个**安全库存**（如 `日均消耗 × 3`，落到变量 `安全库存天数`），避免震荡。

4. **付款不足要有降级链**：写清"钱不够怎么办"的顺序——动用余币 → 借债（产生利息）→ 变卖资产 → **削量采买**（买到多少算多少，缺口转入饥荒/逃奴剧情线）。否则模型会默认"钱无限"。

5. **与既有"告急处理"去重**：若卡里已有"资源不足须安排采买"之类的散句，把这个新机制定位为它的**货币化执行手段**（"告急处理要求采买，本机制规定怎么买、怎么记账、怎么付款"），并把旧散句改成**指向新规则**，消除并行措辞。

6. **检查 UI 是否需要改**：先看结算表明细的渲染是"固定键"还是"动态遍历 `Object.keys(明细)`"。若为后者（常见），新增明细键会**自动渲染**，UI 与 mockData 都不用动——动手改 UI 前先确认这一点，能省一大圈。

### 历法粒度改造（月结算 → 周结算）必须全链一致

把"每月结算"改成"每周结算"是**高翻车改造**，因为量级文案散落在规则条目各处。按此清单逐项做，最后原文 grep 残留：

1. **历法字段**：世界节点补 `星期: 1`（1~7）；ZOD 加 `z.coerce.number().int().transform(v => _.clamp(v, 1, 7)).prefault(1)`。
2. **结算节点改名**：`月结算` → `周结算`；子键 `本月消耗/收益/净额` → `本周…`。
3. **日期结构扩展**：`上次结算日期` 补 `周序号`（YAML 块式，注意是 block 不是 flow mapping——锚点要按实际缩进写）。
4. **历史条目键格式**：`"春3月初结"` → `"1500年3月第1周"`。
5. **规则条目量级重算**：把所有"每月/月"的量级改成"每周/周"，并**重算数值**（口粮、牲畜、战马、薪饷、维护、结算示例），不能只改字面。参考换算：每月 × 7/30 ≈ 每周。
6. **状态栏**：`appData["月结算"]` → `["周结算"]`、`createAccordion("周结算", …)`、所有"本月 X"标签 → "本周 X"、`mockData` 键同步。
7. **收尾 grep**：`月结算` / `本月` / `月结` / `每月` / `一整个月` 全卡计数必须归零（含 initvar、ZOD、所有世界书条目、状态栏 JS 与 mockData）。

### 长期机制型世界书条目（如"诱导/骚扰/调教类慢性机制"）

写这类**非单次事件、而是长期缓慢生效**的机制条目时，用这套骨架（实测可用）：

1. **适用对象**：明确"主体是谁、客体是谁、前提条件"（如"主角名下已确立为亲信且好感度 ≥ N 者"），避免模型让路人甲凭空发作。
2. **分级阶梯**：`L0 日常 → L1 轻 → L2 → L3 关键 → L4 越界 → L5 惯性化`，逐级解锁、**禁止跳级**、主角抗拒可回退一级。阶梯给模型"下一步该做什么"的明确抓手。
3. **隐性刻度**：允许模型在叙事中自行把握的软性计量（如"好奇值""张力"），说明其增减触发点，但不写进变量（省 token、不增加 ZOD 负担）。
4. **数值联动表**：把每一级映射到**既有变量**的具体增减（如 性瘾 +1~3、性欲额外 +3~8、顺从度 +1~3），做到文字与数值互证。**优先复用现有变量**，不轻易新增。
5. **硬性约束**：与卡内最高优先级条目（如《性感受限制器》）挂钩，声明"本机制只推动底层累积，不代表可以秒高潮"；并写明**主角反抗必须有效**（可暂缓/回退/反制），避免机制变成不可逆宿命。
6. **交叉引用**：列出与其它条目的接口（性格原型决定推进方式、可作某支线的激活条件之一等）。
7. **接入召回**：把新条目名加进 `[世界观召回]` 常驻条目的清单（`…、<扶她系统>、<性教学>之中的内容。`），否则模型不知道要去读它；并可在「变量更新规则」对应变量处补一行"受 <X> 机制推动，须同步更新…"。

**条目字段集**：直接深拷贝同族现有条目（如扶她系统 entry[29]）的 `extensions`，只改 `display_index`（= 当前条目数）、`depth`、`position`；`id` 取 `max(id)+1`，`insertion_order` 取同组 +1，`position='before_char'`、`constant=true`、`use_regex=true`。写脚本时**先做幂等检查**（同 `comment` 已存在即退出），避免重复注入。

### 验证要跑到"运行时"，不能只查字符串

字符串 grep 只能证明"文本在、且唯一"，证不了 schema 真能跑。**改完 ZOD 必须做运行时求值**：

- 用 `vm` 沙箱加载卡的 ZOD 脚本：剥掉 `import` 行、把 `export const Schema =` 换成 `var Schema =`、去掉末尾 `$(() => { registerMvuSchema(Schema) })`，注入 `z`（真 zod）、`_`（lodash，或只实现 `_.clamp` 的垫片）、`$: ()=>{}`。
- 用 `js-yaml` 解析 initvar 的 YAML（注意剥掉 ``` 围栏）→ `Schema.safeParse(data)` → 打印 `success` 与关键新字段值。
- 再做**叶子级往返 diff**：递归展平 initvar 与 parse 结果的所有叶子路径，比对 `lost`（被 schema 剥离 = 字段无接收者）与 `gained`（prefault 补齐）。**`lost` 必须为 0**；`gained` 只应是合理的默认值补齐。
- 状态栏 JS 用 `node --check` 前必须先**只抽出含 `Mock Data` 的那个 `<script>` 块**再检查——直接 check 整个 `replaceString`（含 ``` 围栏与 HTML）会得到假 `SyntaxError`。

### 打包与回读

PNG 卡打包 = 保留原图所有 chunk，仅替换 `tEXt` 里的 `chara` / `ccv3` 两个 chunk（新数据 = 关键字 + `\x00` + base64(JSON)，重算 CRC32）。**打包后必须回读校验**：重新解析 PNG，解 base64 反序列化，断言 `chara == ccv3 == 源 JSON`（三层相等），并复核 `entries` 数与末条 comment、关键 UI 关键词仍在新 payload 里。

## 关键纪律（都是机器消费的接口，错了不报错、只是静静地不工作）

- **条目名称里只有两个前缀是硬接口**：`[initvar]…`（MVU 靠名字识别初始值，且必须禁用）、`[mvu_update]…`（标记"只发给变量更新轮"的条目 = 支持额外模型解析模式）。其余名称（变量列表/更新规则/输出格式）是社区约定——实测卡会用 `[系统]当前变量情况`、`（EJS）MVU账本记录发送` 等自定义名，可接受，但别用 `get_message_variable` 旧宏。
- **变量列表是"内容"接口**：某个启用条目（推荐蓝灯 @D0）中必须含 `{{format_message_variable::stat_data}}`——名字不重要，内容才让 AI 每轮看到最新变量值。
- **输出格式固定为 `<UpdateVariable><Analysis/><JSONPatch/>` 结构**：照抄模板，改动等于让 AI 和 MVU 对不上码。
- **正则 `findRegex` 是 JS 正则字面量字符串**（如 `"/<update(?:variable)?>/gsi"`），字段名是英文（`scriptName`/`findRegex`/`replaceString`/`placement`/`markdownOnly`/`promptOnly`/`minDepth`/`maxDepth`）；`markdownOnly:true` = 仅格式显示，`promptOnly:true` = 仅格式提示词。
- **前端 HTML 必须含 `<body>` 或 `<head>` 标签**，否则酒馆助手不会渲染；且必须以 ``` 代码块包裹。
- **`_` 前缀变量 AI 不能更新；`$` 前缀变量对 AI 不可见**（两者都是 MVU/酒馆助手约定）：只给脚本/界面用的变量记得加前缀。
- **schema 每个字段都必须带兜底**（文本/数值 `.prefault(def)`，数值再加 `.catch(def)` 防 NaN）：字段缺失或非数字输入会让 zod 拒绝整次更新（玩家侧报「变量更新错误」）。新增字段必须与 initvar/开局 `<initvar>` 同步，并按 `references/mvu.md` §2.6 本地实测；initvar 里以 `{` 开头的值（如 `{{user}}`）必须加引号（否则 YAML 解析成对象 → `[object Object]`，见 §3.1）。
- **世界书 JSON 用 ST 格式**：`{"name": ..., "entries": {"<uid数字字符串>": {uid, key, keysecondary, comment, content, constant, disable, position(数字), order, depth, ...}}}`。`position` 数字表：0=Character Before，1=Character After，2=AN Top，3=AN Bottom，4=@D(指定深度)，5=EM Top，6=EM Bottom，7=Outlet。
- **卡 JSON 是 V2 (`chara_card_v2`) 结构**，变量相关内容全在 `data` 下；脚本在 `data.extensions.tavern_helper.scripts`，正则在 `data.extensions.regex_scripts`。打包脚本已处理。
- 修改现有卡前先 `extract_card.py` 拆包，改完再打包，不要凭空重写用户已有的卡；工程里 `card.json` 的 `extensions_extra`（承载 `depth_prompt` 等第三方扩展字段）**不要删**。
- 脚本内容可以带 ``` 围栏（会被自动剥离），但**不能是 JSON 导出格式**（`{"type":"script",…}` 整体粘贴会静默失效；`check_card.py` 会检出）。

## 参考索引

| 文档 | 何时读 |
|---|---|
| `references/ecosystem.md` | 需要解释组件分工/安装方式/版本要求/依赖来源，或要在酒馆助手与小白X之间做选型 |
| `references/card-format.md` | 要手改卡 JSON、PNG 元数据、世界书格式、字段映射时；**改现成卡找不到 zod schema 在哪**（在 `tavern_helper.scripts`，不在 regex_scripts）时见 §5.1 |
| `references/mvu.md` | 写变量结构(schema)、initvar、三大变量提示词、变量更新相关脚本与调试；**设施/类别随进度增长**（固定键 + `.catchall()`）见 §2.2.1 |
| `references/regex.md` | 写/改正则，理解 placement、仅格式显示/提示词、命名约定与写法坑 |
| `references/prompt-template.md` | 用 EJS 写动态提示词/分阶段人设/绿灯预处理 |
| `references/frontend.md` | 写状态栏/交互界面、iframe 环境 API、样式规范、分发方式、结构化消息协议与按钮模式 |
| `references/opening-forms.md` | 做创角/开局表单：占位符体系、三种提交通道、协议条目写法 |
| `references/xiaobaix.md` | 走小白X路线：代码块渲染开关、变量管理 2.0、小白板、与酒馆助手混用注意 |
| `references/content-craft.md` | 写角色设定/世界书内容/开场白，追求"能卖"的叙事质量 |
| `references/adult-events.md` | 制作/修改含成人情节的卡：演出时长控制、事件构成与边界、触发密度、计分护栏、镜头聚焦、事件库模式、**用词条款（医学/民间粗俗/日系色情三套语域，按场景选择，禁隐晦代称）**、主角人设与人称策略、**"现实感限制器"（约束生理反应不夸张 + 四层冗余注入确保跨模型生效）**、**身体损耗状态的可逆/不可逆分层设计** |
| `references/female-statusbar-cavity.md` | 做女性主角「腔内/断面」差分状态栏：单图 vs 拆腔、状态×插入矩阵、黑底/白底去底、精灵图偏移、边框不压图、窄屏缩放、外部世界书状态别名、18 帧测试、单腔/拆腔判定与强制更新协议适配 |
| `references/cover-art.md` | 做/换卡面图：封面工作流（裁水印、文字叠层、rebuild 校验）、PIL 文字重影顺序坑、受限平台图像生成分诊与尺度边界 |

## 常见故障速查

| 症状 | 排查 |
|---|---|
| 变量没初始化 | `[initvar]` 条目名没含 `[initvar]` / 没禁用？开局消息有没有 `first_mes`？新开聊天才生效 |
| 日志"变量结构注册失败" | schema 脚本报错，看「酒馆助手的日志查看器」里 `【脚本|变量结构】` 开头的报错 |
| 变量更新报错（toastr/日志「变量更新错误」） | schema 字段缺兜底：新增字段没 `.prefault(def)`（旧存档缺键即炸）；数值字段收到非数字字符串得到 NaN、zod 在 transform 前就拒绝 → 需 `.catch(def)`。把两种兜底打包进 helper，并本地跑 schema 实测——见 `references/mvu.md` §2.5/§2.6 |
| 某变量显示成 `[object Object]`（如姓名） | initvar 里该值以 `{` 开头（如 `{{user}}`）被 YAML 解析成 flow mapping → 加引号：`姓名: '{{user}}'`。见 `references/mvu.md` §3.1 |
| AI 新增的设施/类别莫名消失（不报错） | schema 该字段是固定键 `z.object({…})`，未知键被 zod **静默剥离** → 链尾加 `.catchall(z.record(z.string(), …))`，并本地断言新键仍在。见 `references/mvu.md` §2.2.1 |
| 只靠世界书条目写约束，某些模型（DeepSeek/Gemini）照样违规 | 单层注入被忽略 → 同一约束做**四层冗余注入**（世界书条目 + 变量更新规则顶部 + `depth_prompt` + `description`/`system_prompt`），并写"优先级高于文风要求"。见 `references/adult-events.md` §11.2 |
| 界面在安卓/窄屏挤爆或换行错位 | 用 `@container (max-width: …)` 改布局，别用 `@media`；flex 子项补 `min-width: 0`（`flex-basis: auto` 是元凶）。见 `references/frontend.md` §7.0 |
| 无头 Chrome 截图卡死/`Abnormal renderer termination` | 完整卡 HTML 太重 → 用**隔离预览法**：只抽 `<style>`+目标函数+极简外壳。见 `references/frontend.md` §8.1 |
| 改现成卡时找不到 schema / 改了字段界面不认 | schema 在 `data.extensions.tavern_helper.scripts[N].content`（不在 regex_scripts）；改变量常需**四处同步**（schema+initvar+更新规则+状态栏）。见 `references/card-format.md` §5.1 |
| AI 不更新变量 | `变量列表`/`[mvu_update]变量输出格式` 条目没启用，或没在 D0 深度注入 |
| 状态栏不显示 | 占位符正则没启用？酒馆助手渲染开关没开？HTML 缺 `<body>`/`<head>`？加了闸门后请检查：界面正则要匹配 READY 且排在闸门之后 |
| AI 总是不输出更新块 | 加「变量输出格式强调」或「强制结尾输出」条目（`templates/`，蓝灯 @D0） |
| 你本地正常、玩家那边界面/脚本失效 | 检查卡内是否残留 localhost/开发地址导入（`check_card.py` 会警告）；玩家缺扩展时可用「自动安装插件」脚本引导 |
| 更新块大量出现在提示词 | `[不发送]去除变量更新` 正则的 `promptOnly` 没勾，或深度设置写反（应 null/null 或 minDepth，别用 maxDepth） |
| 玩家导入卡后无正则 | 首次弹窗"允许该角色的正则脚本"被拒绝 → 扩展设置→正则→重新允许 |
| 改完卡却发现成品没变化 | 忘记重新打包：`build_card.py` 才会更新 `输出/`；改世界书/正则/脚本/底图后必须重建 |
| 补丁脚本锚点找不到（count=0） | 全角/半角冒号、等号数量、JSON 转义差异——先从文件 dump 原文再定锚点，勿凭记忆写 |
| 新机制改动后担心静默回归 | 每加机制就往 `tools/audit_consistency.py` 的 KEYWORDS 追加一条，成品抽查会连带校验 |
| 图像生成报 internal server error | 先用无害提示词对照测试分诊：能出图＝内容审核（走单变量降级法）、也报错＝服务波动稍后重试；详见 `references/cover-art.md` |
| 改了 ZOD 但"grep 都在、运行时却报错" | 字符串 grep 证不了 schema 可跑 → 跑 `scripts/zod_validate.cjs <card.json>`：vm 沙箱加载 ZOD、解析 initvar、做叶子级 LOST/GAINED diff（LOST 必须为 0） |
| 新资源（粮草/木材等）在状态栏渲染成裸标签 | 该资源被塞进了货币节点的泛型 `Object.entries` 循环 → 另开独立顶层节点 + 自定义渲染块 |
| 历法粒度改了（月→周）但界面上仍有"本月" | 全链未同步：initvar / ZOD / 所有世界书条目 / 状态栏 JS + mockData 四处都要改，收尾用 grep 把 `本月/月结算/月结/每月` 计数清零 |
| `node --check` 报 `SyntaxError: Unexpected number` 但代码看着没问题 | 检查的是整个 `replaceString`（含 ``` 围栏与 HTML）→ 只抽出含 `Mock Data` 的那个 `<script>` 块再 check |
| 补丁脚本报「锚点未找到」但字段明显存在 | YAML/缩进/字段名与记忆不符 → 一定先 `dump` 原文（`repr()` 打印）再写锚点；实测 initvar 顶层节点子键是 **2 空格**缩进（不是 4），ZOD 里的字段常已被用户加了 `.clamp()` 变得更长 |
| 差分图被边框线压住 | `border` 占内容盒 → 改外圈 `box-shadow` + `gutter`（桌面 6px / 窄屏 4px），视口=帧+2×gutter。见 `references/female-statusbar-cavity.md` §4 |
| 18 张差分图只显示了几张 | 只按状态枚举匹配，没按插入/精液量分支 → 建 9×2 矩阵 + 纯函数断言 `missing=[]`。见 `references/female-statusbar-cavity.md` §2.1 |
| 外部世界书写「发情/大量精液」不显示对应图 | 只认自己的枚举 → 合并读取多字段 + 别名关键词 + 回退；插入与内射分开。见 `references/female-statusbar-cavity.md` §3 |
| 拆成子宫腔/小穴腔后判定打架 | 两套 ml/两套插入判定 → 回到单腔按 `总ml`，或只留一个 ml 权威源 + 宫颈口闸门。见 `references/female-statusbar-cavity.md` §6 |
| 强制更新协议每轮输出巨长、正文被挤掉 | 原样导入 12K 通用协议 → 做卡专用精简条目，引用变量更新规则，输出 `<BianLiang>+<Analysis>+<JSONPatch>`。见 `references/female-statusbar-cavity.md` §7 |
| 性文本写成「花径/蜜液/下面/那里」 | 用了委婉代称 → 按场景选医学/民间粗俗/日系色情三套语域之一，旁白点名器官与动作；禁词清单见 `references/adult-events.md` §7 |
| 新增资源明细在结算表不显示 | 先确认明细渲染是动态 `Object.keys(明细)` 还是固定键；动态遍历则自动出现，无需改 UI |
| 玩家反馈「事件两三句就结束 / 围观群众抢戏 / 数值随演出膨胀」 | 成人卡演出系统的三个经典缺陷，配方见 `references/adult-events.md` |

更详细的排错（含日志文本对照）见 `references/mvu.md` 的诊断节与 `references/ecosystem.md`。
