# 🔥 XS 热点开源产品集

> 从 2026-11-20 的 GitHub Trending / X 技术热榜里，挑出 5 个正在爆的方向，做成 5 个能直接跑、能涨 star 的开源产品。

数据的 originated from real signals — 不是拍脑袋想的点子。每个产品独立仓库、纯前端、零后端，`npm install && npm run dev` 即可运行。

## 为什么是这 5 个

| 热点主线 | 榜单证据（2026-11-20） |
|---|---|
| 反 AI 味 / 设计品质 | `impeccable` +15.2k、`unslop` +15.1k、`hallmark` +14.1k |
| 代码知识图谱 | `graphify` +10.6k/月、`codegraph` +119/天 |
| 文档转 Markdown | `microsoft/markitdown` 今日 +434（全场第一） |
| Agent Skills 生态 | `mattpocock/skills` +19.7k、`anthropics/skills` +11.8k |
| Token 经济学 | MSFT `TokenEconomizer` 上 X 热榜、`OmniRoute` +5.7k |

## 5 个产品

| 产品 | 一句话 | 热点 | 目录 |
|---|---|---|---|
| 🧹 **SlopCheck** | 给文案和代码测「AI 味」，评分 + 高亮 + 一键去味 | 反 AI 味 | [`slopcheck/`](./slopcheck/) |
| 🕸️ **CodeGraph** | 把代码库变成可交互的依赖图谱，揪出循环依赖和孤儿文件 | 代码图谱 | [`codegraph/`](./codegraph/) |
| 📄 **Drop2MD** | 拖入文件，本地转成干净 Markdown，隐私不上传 | markitdown | [`drop2md/`](./drop2md/) |
| 🧰 **SkillVault** | 管理、筛选、开关你的 Agent Skills，一键生成安装脚本 | Skills 生态 | [`skillvault/`](./skillvault/) |
| 💰 **TokenLens** | 分析 Claude Code 会话日志，找出 token 都花在哪、怎么省 | Token 经济学 | [`tokenlens/`](./tokenlens/) |

## 统一技术栈

React + TypeScript + Vite，纯前端，依赖极简；图表 / 图谱全部手写 SVG，不引重库。每个产品都内置示例数据，打开即见效果。

## 快速开始

```bash
cd slopcheck   # 或 codegraph / drop2md / skillvault / tokenlens
npm install
npm run dev
```

## License

MIT
