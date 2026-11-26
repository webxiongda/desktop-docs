# XS 热点开源产品 · 实现计划

> 面向执行者：每个产品独立目录、独立仓库、可 `npm run dev` 运行、`npm run build` 通过。

**目标：** 根据 2026-11-20 GitHub/X 开发者热点，做出 5 个能涨 star、好用的开源前端产品。

**技术栈：** Vite + React + TypeScript，纯前端，依赖尽量少。

**规格依据：** 用户已通过确认表单批准 5 个产品方向（plan=all）。

## 全局约束
- 工作目录：`/Users/polarbear/orca/projects/桌面/`，每个产品一个子目录。
- 环境变量 / 凭据一律不写进代码、README、配置。
- 注释：业务逻辑用中文，技术实现用英文。
- 每个产品独立 `git init`，含 README（含 badges、特性、快速开始）、MIT LICENSE、.gitignore。
- 界面必须精致（现代感、动效、响应式），第一眼决定 star。

## 产品清单与验收方式

| 产品 | 目录 | 目标 | 验证方式 |
|---|---|---|---|
| SlopCheck | `slopcheck/` | AI 味检测器：评分 + 高亮 + 改法 + 一键去味 + 分享卡 | build 通过；输入示例文案能出分和高亮 |
| CodeGraph | `codegraph/` | 代码依赖图谱：AST 解析 + 力导向图 + 环/孤儿/枢纽分析 | build 通过；内置示例能渲染出图 |
| Drop2MD | `drop2md/` | 拖拽转 Markdown：本地转换 + 预览 + 下载 | build 通过；拖入示例文件能出 md |
| SkillVault | `skillvault/` | Agent Skills 管理器：解析/筛选/开关/安装命令 | build 通过；内置 skill 数据能展示和筛选 |
| TokenLens | `tokenlens/` | Token 花费分析：解析会话日志 + 图表 + 省钱建议 | build 通过；内置示例能出图和建议 |

## 执行步骤
1. 并行实现 5 个产品（脚手架 → 核心逻辑 → UI → README → build 自验）。
2. 统一验收：逐个 `npm install && npm run build`。
3. 浏览器逐个打开截图，确认界面可用、不是空壳。
4. 汇总产出给用户，说明每个产品蹭的热点与涨星逻辑。
