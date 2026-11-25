# Maccy 中文增强二开设计规格

日期：2026-11-26  
状态：设计已获用户批准；规格已补充「本地交付 + 全局自测」要求，待最终审阅

## 1. 目标

把 GitHub star 排名第一的剪切板工具 `p0deje/Maccy`（22.5k star，Swift / macOS 14+，MIT）二开为本地自有版本 `桌面/maccy-cn/`。第一阶段交付 **P0 基座** 与 **P1 中文搜索增强**，后续 P2 AI / P3 自动化 / P4 UI 重做 / P5 同步与团队另行开规格。

成功标准：

1. `桌面/maccy-cn/` 是从上游 clone 的独立 git 仓库，`upstream` 指向 `p0deje/Maccy`，本地有 `dev/cn-search` 分支，Xcode Debug 构建通过。
2. 新增中文智能搜索模式 `cn`：支持中文分词、全拼、首字母检索；结果排序与高亮 `ranges` 正确。
3. 上游原有 exact / fuzzy / regexp / mixed 四种模式行为不变；现有测试全部通过，新增中文测试通过。
4. 不引入任何新的 Swift Package 依赖，不修改 SwiftData schema，不持久化索引。

## 2. 调研事实与依据

| 事实 | 来源 | 对设计的影响 |
|---|---|---|
| `p0deje/Maccy` 22.5k star 为同类第一，`hluk/CopyQ` 12.7k 第二 | GitHub 仓库搜索按 star 排序（2026-11-26） | 选定 Maccy 为上游 |
| MIT License | `LICENSE` | 允许 fork、改名、闭源衍生；须保留版权与许可声明 |
| 要求 macOS 14+；本机 macOS 27.2.1 / Xcode 27.2 / swift 可用 | `README.md` + 本机探测 | P0 构建可行 |
| `Search.swift` 仅有 exact / fuzzy / regexp / mixed；Fuse 按字符匹配，超 5000 字符截断 | 源码 | 中文搜索需新增独立匹配器，不改造 Fuse |
| `History.swift` 用 `Throttler(minimumDelay: 0.2)` 节流查询 | 源码 | 索引查询可承受毫秒级内存计算 |
| `HistoryItem.generateTitle()` 把标题截到 1000 字符 | 源码 | 索引只覆盖标题，与现有搜索口径一致 |
| 已有 `zh-Hans.lproj` / `zh-Hant.lproj` 本地化 | 源码目录 | 中文界面无需从零翻译，仅补新增模式文案 |

## 3. 范围

### 3.1 纳入

- 本地 clone Maccy、配置 `upstream` remote、建立 `dev/cn-search` 分支。
- `Search.swift` 新增 `Mode.cn`、`ChineseIndex`、`ChineseMatcher`。
- `History.swift` 建立进程内索引：加载时全量重建，新增 / 删除时增量更新。
- 设置面板 `searchMode` 增加「中文智能」选项，补 `en.lproj` / `zh-Hans.lproj` 文案。
- `MaccyTests/SearchChineseTests.swift` 单元测试。
- Xcode Debug 构建与测试验证。
- 本地交付：仓库 clone 到 `桌面/maccy-cn/`，Debug 构建产物在本机可运行（不自动安装到 `/Applications`，不推送 GitHub）。
- 全局自测：交付前运行 `python3 -m unittest discover -s ~/.omp/agent/tests -v`，要求退出码 0 并在汇报中给出命令与关键输出。

### 3.2 不纳入

- P2 AI 能力、P3 自动化规则、P4 UI 重做、P5 跨设备同步 / 团队共享。
- SwiftData schema 变更、索引落库、Core Data / SwiftData 迁移。
- 新的 Swift Package 依赖（拼音库等）。
- GitHub 远端仓库创建、push、PR 或发版（用户未授权，需另行确认）。
- 上游 commit 历史重写、上游功能逻辑重构。

## 4. 方案选择

| 方案 | 做法 | 优点 | 代价 |
|---|---|---|---|
| **A 零依赖 Foundation（已批准）** | `CFStringTokenizer` 分词 + `String.applyingTransform(.toLatin)` 生成拼音，进程内索引 | 零新依赖、离线、隐私、契合 Maccy 轻量哲学 | 多音字 / 生僻字可能不准 |
| B 引入 SPM 拼音库 | `PinyinSwift` 等 | 多音字更准 | 新增依赖与升级成本 |
| C AI 语义搜索 | Core ML / 云端 embedding | 能理解语义 | 属于 P2 范畴，越界 |

采用 **方案 A**。已知限制：方案 A 下「重庆」搜 `chongqing` 可能不准；若后续要求多音字准确度，升级到方案 B 需另开规格。

## 5. 文件级改动

1. **`Maccy/Search.swift`**
   - `Mode` 新增 `case cn`；`description` 增加本地化 key（如 `ChineseSmart`）。
   - `search(string:within:)` 的 `switch` 增加 `.cn` 分支，调用 `ChineseMatcher`。
   - 新增 `ChineseIndex`（索引容器）与 `ChineseMatcher`（查询与评分）。
2. **`Maccy/Observables/History.swift`**
   - 持有 `ChineseIndex` 实例。
   - `load()` 完成后对全部条目全量重建索引。
   - `add()` / `delete()` 后增量更新索引。
   - `updateSearchResults()` 把索引传给 `Search`；具体签名以现有代码为准，保持节流不变。
3. **`Maccy/Models/HistoryItem.swift`、`HistoryItemDecorator.swift`**
   - 不改模型字段；仅提供稳定的条目标识与 `title` 作为索引输入。
4. **本地化文件**
   - `Maccy/en.lproj/GeneralSettings.strings`、`Maccy/zh-Hans.lproj/GeneralSettings.strings` 增加新模式的显示文案。
5. **新增测试**
   - `MaccyTests/SearchChineseTests.swift`。

## 6. 匹配规则

- **查询分词**：中文用 `CFStringTokenizer`（`.unitWord`），英文按空格切分；统一大小写与全 / 半角。
- **索引内容**：对每个 `title` 生成三元组 `{tokens, pinyin, initials}`：
  - `tokens`：分词结果数组；
  - `pinyin`：无音调全拼字符串；
  - `initials`：各 token 拼音首字母拼接。
- **匹配语义**：查询中的所有 token 都必须命中（AND）；每个 token 按以下顺序尝试：
  1. 原文子串命中 → 得分 0
  2. 分词精确命中 → 得分 1
  3. 全拼命中 → 得分 2
  4. 首字母命中 → 得分 3
- **排序**：总分升序；同分沿用现有 `Sorter` 的 `lastCopiedAt` 降序。
- **高亮**：把命中的 token range 映射回原 `String.Index`，写回 `SearchResult.ranges`，复用现有高亮视图。

## 7. 数据流与兜底

- **写入流**：复制 → `History.add()` → 索引增量更新。
- **查询流**：输入 → 现有 `Throttler(0.2s)` → `ChineseMatcher` → 排序 → 高亮。
- **兜底**：
  - 拼音转换失败 → 仅用原文 / 分词匹配；
  - 空查询 → 沿用原逻辑（返回全部）；
  - 超长文本 → 截断到 5000 字符（沿用 `fuzzySearchLimit`）；
  - 索引构建异常 → 降级为 `exact` 模式，不阻断主流程；
  - 索引与数据不一致 → 以 `load()` 全量重建为准。

## 8. 测试计划

`MaccyTests/SearchChineseTests.swift` 覆盖：

| 用例 | 输入 | 期望 |
|---|---|---|
| 中文子串 | 查询「剪贴板」，含「我的剪贴板历史」 | 命中 |
| 分词 AND | 查询「中文 搜索」，含「中文搜索增强」 | 命中；缺一词不命中 |
| 全拼 | 查询 `jianqieban`，含「剪切板」 | 命中 |
| 首字母 | 查询 `jqb`，含「剪切板」 | 命中 |
| 中英混合 | 查询「copy 图片」，含「copy 图片到剪贴板」 | 命中 |
| 空查询 | 空字符串 | 返回全部 |
| 超长文本 | 超过 5000 字符的标题 | 不崩溃、有结果或安全返回空 |
| 高亮 range | 命中后检查 `ranges` | 索引落在原字符串边界内且对应命中文本 |
| 回归 | 四种原有模式 | 行为不变 |

## 9. 约束与风险

- **许可证**：上游 MIT。保留上游 `LICENSE`、版权声明与致谢；衍生版本继续以 MIT 发布。
- **品牌与标识**：第一阶段保留 Maccy 名称与 bundle ID，只加功能；改名 / 换 bundle ID 属独立议题，需另行批准。
- **性能**：索引在 `load()` 时构建，几千条历史量级可接受；查询走内存，配合 0.2s 节流。避免在每次查询中重算全量拼音。
- **多音字**：方案 A 的已知限制，已在规格中明示，不承诺解决。
- **构建 / 签名**：本机 Xcode 27.2 可用，但 ad-hoc 签名与权限（辅助功能）可能影响运行验证；失败要区分环境问题与改动问题并如实报告。

## 10. 验收步骤

1. `git -C maccy-cn remote -v` 显示 `upstream` 指向 `p0deje/Maccy`；`git -C maccy-cn branch --show-current` 为 `dev/cn-search`。
2. `xcodebuild -list -project maccy-cn/Maccy.xcodeproj` 能列出 target。
3. `xcodebuild -scheme Maccy -configuration Debug build` 成功；失败则记录具体错误并区分环境与改动。
4. `xcodebuild test` 运行 `SearchChineseTests`，全部通过；原有测试无回归。
5. 手动运行 App：复制一条中文文本，按 ⇧⌘C 唤起，分别用中文、全拼、首字母搜索，确认命中项与高亮正确；切换回 exact 模式确认行为未变。
6. 本地交付确认：`桌面/maccy-cn/` 存在且可 `open` 构建出的 `Maccy.app`；说明如何运行，不擅自安装到系统目录或推送远端。
7. 全局自测：运行 `python3 -m unittest discover -s ~/.omp/agent/tests -v`，退出码 0，并在最终汇报中给出命令与关键输出。

## 11. 非目标

- 不新增 AI、同步、团队、自动化功能，不重做 UI。
- 不修改上游窗口管理、剪贴板监听、粘贴逻辑。
- 不在未获授权的情况下向 GitHub push、创建仓库、提 PR 或发版。
- 不伪造官网 / 更新源 / 签名密钥等无凭据可接入的配置。
