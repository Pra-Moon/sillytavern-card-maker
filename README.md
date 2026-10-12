# sillytavern-card-maker

> **Use the sillytavern-card-maker skill.**
> 一个给 AI 助手（Cherry Studio / Claude Code / Cursor 等支持 Agent Skills 的客户端）使用的 **SillyTavern（酒馆）角色卡制作技能包**。

帮你从零做出——或安全改造——带**变量系统、状态栏、交互界面、创角表单、正则脚本**的完备角色卡，并打包成玩家导入即用的成品（PNG/JSON）。

## 这是什么

一个遵循 Agent Skills 规范（`SKILL.md`）的领域技能：给 AI 装上"酒馆写卡师傅"的完整方法论 + 可运行工具 + 现成模板。

- 🧠 **知识**：`references/` 11 份深度文档——MVU zod 变量规范、酒馆正则（含闸门校验）、EJS 提示词模板、前端界面、创角表单、小白X、卡文件格式、内容构筑、生态选型、**成人向事件演出设计**、**卡面与图像资产**
- 🛠️ **工具**：`scripts/` 5 个 Python CLI——3 个零依赖主流程工具 + 2 个维护工具（封面脚本需 Pillow）
  - `build_card.py` —— 卡工程 → 完备卡（世界书内嵌 `character_book`、正则入 `regex_scripts`、脚本入 `tavern_helper`，PNG `chara`/`ccv3` 元数据，**未知扩展字段全保真**）
  - `check_card.py` —— 工程/成品 30+ 项硬伤检查（条目约定、正则语法、闸门配套、localhost 残留、历史格式……）
  - `extract_card.py` —— 现有卡拆包为可编辑工程（兼容四种历史脚本存储格式）
  - `scan_duplicates.py` —— 世界书赘述体检：行级精确查重 + 句级模糊查重（跨条目 / 条目内），附条目字数排行
  - `apply_cover_text.py` —— AI 插画 → 768 卡面底图（裁平台水印、暗角渐变、标题叠层；需 Pillow）
- 📦 **模板**：`templates/` 18 个——卡/世界书骨架、正则四件套、闸门组、交互式状态栏 HTML、创角表单骨架、变量结构（含健壮版）、输出纪律条目、玩家说明、**改卡维护检查清单**
- 📊 **评测**：`evals/` 附 3 个实测评测用例

## 覆盖的技术栈（四组件生态）

| 组件 | 用途 |
|---|---|
| [酒馆助手 JS-Slash-Runner](https://github.com/N0VI028/JS-Slash-Runner) | 脚本/前端界面底座、消息 HTML 渲染、全套 API |
| [MVU (MagVarUpdate)](https://github.com/MagicalAstrogy/MagVarUpdate) | 按楼层维护变量，解析 AI 的 `<UpdateVariable>` 更新 |
| [提示词模板 ST-Prompt-Template](https://github.com/zonde306/ST-Prompt-Template) | 世界书/提示词里的 EJS 动态逻辑（分阶段人设等） |
| [小白X LittleWhiteBox](https://github.com/RT15548/LittleWhiteBox) | 可选路线：代码块渲染、变量管理 2.0、沉浸式模板 |

## 安装

把本仓库（整个文件夹）放进你所用客户端的技能目录即可：

| 客户端 | 技能目录 |
|---|---|
| Cherry Studio | `…/CherryStudio/Data/Skills/`（放入即自动注册） |
| Claude Code / Claude Desktop | `~/.claude/skills/` |
| 其他 Agent Skills 客户端 | 对应 `skills/` 目录 |

## 使用

直接在对话里说需求即可（技能会自动触发），例如：

- 「帮我做一张酒馆角色卡：都市校园恋爱题材，要有好感度变量、分阶段人设和交互状态栏」
- 「把这张卡加一个创角表单」（附上卡文件）
- 「帮我写一套酒馆正则：去除更新块、折叠显示、渲染状态栏」

也可以显式点名：**Use the sillytavern-card-maker skill.**

AI 会按技能的 8 步工作流产出：卡工程（`card.json`/`worldbook.json`/`regex.json`/`scripts/`）→ 打包成品卡 → 自检 → 玩家安装说明。

## 工具快速上手

```bash
# 拆包现有卡为工程
python -X utf8 scripts/extract_card.py <卡.png|json> <输出目录>

# 检查工程/成品
python -X utf8 scripts/check_card.py <工程目录|卡.json|卡.png>

# 打包（--image 提供底图则输出 PNG 卡）
python -X utf8 scripts/build_card.py <工程目录> --image 底图.png

# 世界书赘述体检（三种输入皆可：世界书 json / 卡工程目录 / 成品卡 json）
python -X utf8 scripts/scan_duplicates.py <卡工程目录|成品卡.json> [--top 25]

# AI 插画 → 卡面底图（需 Pillow）
python -X utf8 scripts/apply_cover_text.py <输入图> 底图.png --title "标 题" --sub "副标题"
```

除封面脚本需 Pillow 外，其余工具零第三方依赖（Python 3.10+）。Windows 中文环境建议 `python -X utf8` 运行以免编码问题。

## 质量与验证

- 用真实线上卡做过 **拆包→重建逐字段保真** 回归（含 1279 条世界书的巨型卡）
- 依据 **291 张真实公开卡语料** 批量校准（发现并修正了正则深度语义、扩展字段丢失、历史格式兼容等问题）
- 做过两轮子代理盲测：带技能 **100%** 通过率 vs 无技能基线 **47.8%**（3 用例 × 28 断言）

## 免责声明

- 本仓库是**制作工具与知识包**，不包含任何具体角色卡内容；不针对任何特定题材。
- 请在遵守所在平台规则与当地法律法规的前提下使用；不得用于制作违法或侵权内容。
- 用本工具制作的卡内容由制作者自负其责。

## 致谢

本技能的方法论与部分模板基于以下开源项目与作者的公开资料整理、并经实测校准（版权归各自作者所有）：

- [SillyTavern](https://github.com/SillyTavern/SillyTavern) 本体
- [N0VI028/JS-Slash-Runner](https://github.com/N0VI028/JS-Slash-Runner)（酒馆助手）
- [MagicalAstrogy/MagVarUpdate](https://github.com/MagicalAstrogy/MagVarUpdate)（MVU）
- [zonde306/ST-Prompt-Template](https://github.com/zonde306/ST-Prompt-Template)（提示词模板）
- [RT15548/LittleWhiteBox](https://github.com/RT15548/LittleWhiteBox)（小白X）
- StageDog（青空莉）：[tavern_resource](https://github.com/StageDog/tavern_resource)、[tavern_helper_template](https://github.com/StageDog/tavern_helper_template) 与系列教程——变量三件套、闸门/美化正则、工具脚本的源头之一

## 许可

[MIT](LICENSE)
