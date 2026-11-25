# Maccy 中文增强实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 subagent-driven-development（推荐）或 executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** fork Maccy 到本地，新增零依赖的中文智能搜索（分词 / 全拼 / 首字母），构建可运行 App 并完成全局自测。

**架构：** 上游 Maccy 为 SwiftUI + SwiftData 的 macOS 剪贴板管理器。本计划新增两个纯 Swift 单元：`ChineseIndex`（进程内索引，检索 token / 拼音 / 首字母）与 `ChineseMatcher`（查询分词、评分、range 映射），在 `Search.swift` 增加 `Mode.cn` 分支，由 `History` 维护索引生命周期。不新增依赖、不改数据模型、不持久化索引。

**技术栈：** Swift 5 / SwiftUI / SwiftData / XCTest / Xcode 27.2 / macOS 14+；零新增 SPM 依赖。

**规格：** `docs/superpowers/specs/2026-11-26-maccy-cn-enhance-design.md`

## 全局约束

- 上游 `p0deje/Maccy`，MIT License，保留 LICENSE 与上游署名。
- 本地目录 `桌面/maccy-cn/`，独立 git 仓库，`upstream` 指向 `p0deje/Maccy`，工作分支 `dev/cn-search`。
- 第一阶段保留 Maccy 名称与 bundle ID；不改上游功能逻辑。
- 不引入任何新 Swift Package 依赖；不修改 SwiftData schema；不持久化索引。
- 查询节流保持 `Throttler(minimumDelay: 0.2)` 不变。
- 超长文本截断上限 5000 字符（沿用 `fuzzySearchLimit`）。
- 拼音转换失败须降级原文 / 分词匹配，不阻断主流程。
- 不创建 GitHub 远端仓库、不 push、不提 PR、不发版。
- 不自动安装到 `/Applications`；只保证本机构建产物可运行。
- 交付前运行 `python3 -m unittest discover -s ~/.omp/agent/tests -v`，退出码 0 并在汇报中给出命令与关键输出。
- 注释：业务逻辑用中文，技术实现用英文。

## 审查重点（Review Focus）

规格隐含但测试可能漏掉、最可能伤到用户的五类输入 / 失败模式；每行都在对应任务里有测试钉住：

1. **空查询**：索引尚未建立或查询为空时，必须返回全部条目，而不是空列表 —— 任务 3 测试 `testEmptyQueryReturnsAll`。
2. **中文子串优先**：多音字导致拼音不准时，原文子串仍必须命中 —— 任务 3 测试 `testChineseSubstringWinsOverBadPinyin`。
3. **高亮 range 越界**：命中 range 必须落在原字符串边界内且对应命中文本 —— 任务 3 测试 `testHighlightRangesAreInBounds`。
4. **超长标题**：超过 5000 字符的标题不得崩溃或卡死 —— 任务 2 测试 `testOversizedTitleTruncated`。
5. **索引一致性**：新增 / 删除条目后索引须同步，否则搜索返回陈旧结果 —— 任务 4 测试 `testIncrementalIndexUpdateAfterAddAndDelete`。

## 文件结构

| 文件 | 职责 | 动作 |
|---|---|---|
| `maccy-cn/` | 上游 clone，本地 fork 基座 | 创建 |
| `maccy-cn/Maccy/ChineseIndex.swift` | 索引容器：分词、拼音、首字母；纯值类型，不耦合 SwiftData | 创建 |
| `maccy-cn/Maccy/ChineseMatcher.swift` | 查询分词、评分、range 映射；纯函数 | 创建 |
| `maccy-cn/Maccy/Search.swift` | 增加 `Mode.cn` 分支，调用 matcher | 修改 |
| `maccy-cn/Maccy/Observables/History.swift` | 索引生命周期：全量重建 + 增量更新 | 修改 |
| `maccy-cn/Maccy/en.lproj/GeneralSettings.strings`、`zh-Hans.lproj/GeneralSettings.strings` | 新模式显示文案 | 修改 |
| `maccy-cn/MaccyTests/ChineseIndexTests.swift` | 索引单元测试 | 创建 |
| `maccy-cn/MaccyTests/SearchChineseTests.swift` | 匹配、评分、range、回归测试 | 创建 |
| `maccy-cn/MaccyTests/HistoryChineseIndexTests.swift` | 索引生命周期集成测试 | 创建 |

---

### 任务 1：P0 基座 — clone 与构建跑通

**文件：**
- 创建：`maccy-cn/`（整个上游仓库）
- 修改：`maccy-cn/.gitignore`（如需忽略本地构建产物，沿用上游配置则不动）

- [ ] **步骤 1：clone 上游并配置 fork 基座**

```bash
cd /Users/polarbear/orca/projects/桌面
git clone https://github.com/p0deje/Maccy.git maccy-cn
git -C maccy-cn remote add upstream https://github.com/p0deje/Maccy.git
git -C maccy-cn checkout -b dev/cn-search
```

预期：目录存在；`git -C maccy-cn remote -v` 同时显示 `origin` 与 `upstream` 均指向 `p0deje/Maccy`；当前分支为 `dev/cn-search`。

- [ ] **步骤 2：确认 Xcode 工程可识别**

运行：`xcodebuild -list -project maccy-cn/Maccy.xcodeproj`

预期：输出包含 `Maccy` target 与 schemes 列表，无 `command line tools` 相关报错。

- [ ] **步骤 3：Debug 构建基线**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj build -quiet`

预期：`BUILD SUCCEEDED`。若失败，记录完整错误；区分证书 / 签名 / 环境问题和代码问题，不擅自改上游构建配置。

- [ ] **步骤 4：运行上游现有测试基线**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj test -quiet`

预期：现有测试全部通过；若本机权限导致 UI / 辅助功能相关测试失败，记录具体用例并区分环境问题。

- [ ] **步骤 5：基线 commit**

在 `maccy-cn/` 独立子仓库中记录基线（该仓库是新建的独立仓库，不是共享 worktree）：

```bash
git -C maccy-cn add -A && git -C maccy-cn commit -m "chore: import Maccy upstream as cn-search base"
```

若上游工作区干净、无内容可提交，跳过此步并在执行记录中注明。

### 任务 2：ChineseIndex — 索引容器

**文件：**
- 创建：`maccy-cn/Maccy/ChineseIndex.swift`
- 测试：`maccy-cn/MaccyTests/ChineseIndexTests.swift`

- [ ] **步骤 1：编写失败的测试**

在 `ChineseIndexTests.swift` 中编写：

```swift
func testIndexStoresTokensPinyinAndInitials() {
    var index = ChineseIndex()
    index.update(id: "a", title: "剪切板历史")
    let entry = index.entry(for: "a")
    XCTAssertNotNil(entry)
    XCTAssertTrue(entry!.tokens.contains("剪切板"))
    XCTAssertTrue(entry!.pinyin.contains("jianqieban"))
    XCTAssertEqual(entry!.initials, "jqbls")
}

func testOversizedTitleTruncated() {
    var index = ChineseIndex()
    let long = String(repeating: "剪", count: 6000)
    index.update(id: "long", title: long)
    XCTAssertNotNil(index.entry(for: "long"))
    XCTAssertLessThanOrEqual(index.entry(for: "long")!.pinyin.count, 5000)
}
```

- [ ] **步骤 2：运行测试验证失败**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj test -only-testing:MaccyTests/ChineseIndexTests -quiet`

预期：FAIL，编译错误 `Cannot find type 'ChineseIndex' in scope`。

- [ ] **步骤 3：实现 `ChineseIndex`**

在 `ChineseIndex.swift` 定义：

```swift
struct ChineseIndex {
    struct Entry: Equatable {
        var tokens: [String]
        var pinyin: String
        var initials: String
    }

    private(set) var entries: [String: Entry] = [:]

    mutating func rebuild(from items: [(id: String, title: String)])
    mutating func update(id: String, title: String)
    mutating func remove(id: String)
    func entry(for id: String) -> Entry?
}
```

实现约束：
- 标题先截断到 5000 字符再做索引；
- 中文分词用 `CFStringTokenizer`（`kCFStringTokenizerUnitWord`），英文按空格切分，统一大小写与全 / 半角；
- 拼音用 `String.applyingTransform(.toLatin, reverse: false)` 去除声调后拼接；转换失败时该字段为空字符串；
- `initials` 为每个 token 拼音首字母依次拼接；
- 纯值类型，不 import SwiftData，不耦合 `HistoryItem`。

- [ ] **步骤 4：运行测试验证通过**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj test -only-testing:MaccyTests/ChineseIndexTests -quiet`

预期：`TEST SUCCEEDED`，2 个测试通过。

- [ ] **步骤 5：Commit**

在 `maccy-cn/` 中提交 `ChineseIndex.swift` 与 `ChineseIndexTests.swift`，信息：`feat: add Chinese index with tokenizer and pinyin`。

### 任务 3：ChineseMatcher 与 Search 模式

**文件：**
- 创建：`maccy-cn/Maccy/ChineseMatcher.swift`
- 修改：`maccy-cn/Maccy/Search.swift`
- 测试：`maccy-cn/MaccyTests/SearchChineseTests.swift`

- [ ] **步骤 1：编写失败的测试**

在 `SearchChineseTests.swift` 中编写（断言按规格取值）：

```swift
func testChineseSubstringWinsOverBadPinyin() {
    let results = Search().search(string: "剪贴板", within: [item("我的剪贴板历史")])
    XCTAssertEqual(results.count, 1)
}

func testTokenizeAndMatchFullPinyin() {
    let results = Search().search(string: "jianqieban", within: [item("剪切板")])
    XCTAssertEqual(results.count, 1)
}

func testInitialsMatch() {
    let results = Search().search(string: "jqb", within: [item("剪切板")])
    XCTAssertEqual(results.count, 1)
}

func testTokenAND() {
    let items = [item("中文搜索增强"), item("只有中文")]
    let results = Search().search(string: "中文 搜索", within: items)
    XCTAssertEqual(results.count, 1)
    XCTAssertTrue(results[0].object.title.contains("搜索"))
}

func testMixedChineseEnglish() {
    let results = Search().search(string: "copy 图片", within: [item("copy 图片到剪贴板")])
    XCTAssertEqual(results.count, 1)
}

func testEmptyQueryReturnsAll() {
    let results = Search().search(string: "", within: [item("甲"), item("乙")])
    XCTAssertEqual(results.count, 2)
}

func testHighlightRangesAreInBounds() {
    let results = Search().search(string: "剪贴板", within: [item("我的剪贴板历史")])
    let range = results[0].ranges[0]
    let title = results[0].object.title
    XCTAssertLessThanOrEqual(range.upperBound, title.endIndex)
    XCTAssertEqual(String(title[range]), "剪贴板")
}

func testExistingModesUnchanged() {
    XCTAssertEqual(Search.Mode.allCases.count, 5)
    XCTAssertTrue(Search.Mode.allCases.contains(.exact))
    XCTAssertTrue(Search.Mode.allCases.contains(.cn))
}
```

测试夹具 `item(_:)` 需构造 `HistoryItemDecorator`，沿用 `MaccyTests` 现有 helper；若不存在，用 `HistoryItem` + `HistoryItemDecorator` 的最小初始化方式构造，并把 `Defaults[.searchMode]` 设为 `.cn`。

- [ ] **步骤 2：运行测试验证失败**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj test -only-testing:MaccyTests/SearchChineseTests -quiet`

预期：FAIL，`.cn` 枚举值不存在。

- [ ] **步骤 3：实现 `ChineseMatcher`**

在 `ChineseMatcher.swift` 定义纯函数（不耦合 SwiftData）：

```swift
enum ChineseMatcher {
    static func tokenize(_ query: String) -> [String]
    static func score(token: String, in entry: ChineseIndex.Entry) -> Int?
    static func ranges(for token: String, in title: String) -> [Range<String.Index>]
}
```

实现约束：
- `score` 依次尝试原文子串 → 分词精确 → 全拼 → 首字母，返回 0 / 1 / 2 / 3，未命中返回 `nil`；
- 查询 token 全部命中才返回该条目（AND）；总分 = 各 token 得分之和，升序排列；
- `ranges` 必须落在 `title.startIndex...title.endIndex` 内，映射失败时返回空数组；
- 拼音字段为空时跳过拼音与首字母匹配。

- [ ] **步骤 4：在 `Search.swift` 接入新模式**

- `Search.Mode` 增加 `case cn`，`description` 返回 `NSLocalizedString("ChineseSmart", tableName: "GeneralSettings", comment: "")`；
- `search(string:within:)` 的 `switch` 增加 `.cn` 分支：查询为空时返回全部条目，非空时调用 `ChineseMatcher`；
- 保持其他四种模式实现不变。

- [ ] **步骤 5：运行测试验证通过**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj test -only-testing:MaccyTests/SearchChineseTests -quiet`

预期：`TEST SUCCEEDED`，8 个测试通过。

- [ ] **步骤 6：Commit**

在 `maccy-cn/` 中提交 `ChineseMatcher.swift`、`Search.swift`、`SearchChineseTests.swift`，信息：`feat: add Chinese smart search mode`。

### 任务 4：History 索引生命周期

**文件：**
- 修改：`maccy-cn/Maccy/Observables/History.swift`
- 测试：`maccy-cn/MaccyTests/HistoryChineseIndexTests.swift`

- [ ] **步骤 1：编写失败的测试**

```swift
func testIncrementalIndexUpdateAfterAddAndDelete() async throws {
    let history = History.shared
    try await history.load()
    let before = history.search(query: "增量测试") // 测试用辅助入口，沿用现有 search 路径
    XCTAssertEqual(before.count, 0)

    let item = HistoryItem()
    item.title = "增量测试条目"
    history.add(item)
    XCTAssertEqual(history.search(query: "增量测试").count, 1)

    history.delete(item)
    XCTAssertEqual(history.search(query: "增量测试").count, 0)
}
```

若现有 `History` 无公开 `search(query:)` 入口，测试改为通过 `history.searchQuery = "增量测试"` 后读取 `history.items`，并在赋值后等待 `Throttler` 的 0.2s 窗口；不得为了测试改动生产 API 可见性。

- [ ] **步骤 2：运行测试验证失败**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj test -only-testing:MaccyTests/HistoryChineseIndexTests -quiet`

预期：FAIL，新增 / 删除后索引未更新，搜索结果不符合断言。

- [ ] **步骤 3：在 `History.swift` 接入索引生命周期**

- 持有 `private var chineseIndex = ChineseIndex()`；
- `load()` 完成后，用全部条目（pinned + unpinned）调用 `chineseIndex.rebuild(from:)`；
- `add()` 成功后调用 `chineseIndex.update(id:title:)`；
- `delete()` 成功后调用 `chineseIndex.remove(id:)`；
- `updateSearchResults()` 把索引传给 `Search`；保持 `Throttler(minimumDelay: 0.2)` 不变；
- 索引构建异常时降级为 `.exact` 模式，记录日志，不抛错到 UI。

条目 id 使用 `ObjectIdentifier(item)` 的字符串表示，保证同一对象跨增量操作稳定。

- [ ] **步骤 4：运行测试验证通过**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj test -only-testing:MaccyTests/HistoryChineseIndexTests -quiet`

预期：`TEST SUCCEEDED`。

- [ ] **步骤 5：Commit**

在 `maccy-cn/` 中提交 `History.swift` 与 `HistoryChineseIndexTests.swift`，信息：`feat: wire Chinese index into history lifecycle`。

### 任务 5：设置面板与本地化

**文件：**
- 修改：`maccy-cn/Maccy/en.lproj/GeneralSettings.strings`
- 修改：`maccy-cn/Maccy/zh-Hans.lproj/GeneralSettings.strings`

- [ ] **步骤 1：添加本地化文案**

在两个 `GeneralSettings.strings` 中增加键 `ChineseSmart`：
- 英文：`Chinese Smart`
- 简体中文：`中文智能`

保持原有键值顺序与文件格式，不改动其他条目。

- [ ] **步骤 2：验证文案可被读取**

运行：`rg -n "ChineseSmart" maccy-cn/Maccy/en.lproj/GeneralSettings.strings maccy-cn/Maccy/zh-Hans.lproj/GeneralSettings.strings`

预期：两个文件各命中 1 行，值与上步一致。

- [ ] **步骤 3：全量测试回归**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj test -quiet`

预期：`TEST SUCCEEDED`；任务 2–4 的测试与原有用例全部通过。

- [ ] **步骤 4：Commit**

在 `maccy-cn/` 中提交本地化文件，信息：`feat: localize Chinese smart search mode`。

### 任务 6：本地交付与全局自测

**文件：** 无源码改动；产出为可运行 App 与验证记录。

- [ ] **步骤 1：最终 Debug 构建**

运行：`xcodebuild -scheme Maccy -configuration Debug -project maccy-cn/Maccy.xcodeproj build -quiet`

预期：`BUILD SUCCEEDED`；记录 `Maccy.app` 产物路径（通常在 `~/Library/Developer/Xcode/DerivedData/.../Build/Products/Debug/Maccy.app`）。

- [ ] **步骤 2：本地运行验证**

运行：`open <上一步得到的 Maccy.app 路径>`

预期：App 在本机启动、菜单栏出现图标；按 ⇧⌘C 唤起面板，复制一条中文文本后用中文 / `jqb` / `jianqieban` 搜索，命中项与高亮正确；切回 `Exact` 模式行为未变。若辅助功能权限弹窗阻断，记录并说明需要用户在系统设置里授权，不擅自改系统权限。

- [ ] **步骤 3：全局上下文自测**

运行：`python3 -m unittest discover -s ~/.omp/agent/tests -v`

预期：退出码 0；记录关键输出（用例数与 `OK`）。失败则如实报告失败用例，不掩饰。

- [ ] **步骤 4：交付说明**

在最终汇报中给出：本地目录路径、运行方式、本次交付范围（P0 + P1）、已知限制（多音字）、P2–P5 的后续路径，以及全局自测的命令与输出。不执行 `/Applications` 安装、GitHub push 或发版。

---

## 自检记录（执行前由实现者复核）

- **规格覆盖度**：目标 1–4 分别由任务 1、3、3+5、任务 2–4 的“零依赖 / 不改 schema”约束覆盖；本地交付与全局自测由任务 6 覆盖。
- **步骤扫描**：每个步骤都给出确切命令、签名或断言；无“处理边界情况”类空指令。
- **类型一致性**：`ChineseIndex.Entry`、`ChineseMatcher.score/ranges`、`Search.Mode.cn` 在任务 2–4 中名称一致。
- **审查重点**：五条风险均有对应测试，分别位于任务 3（3 条）、任务 2（1 条）、任务 4（1 条）。
- **比例**：计划长度与规格相当，代码块仅用于签名与测试断言，未抄写实现。
