# TabFlow 二开设计规格

日期：2026-11-26  
状态：设计已获用户批准，待书面规格审阅

## 1. 目标

把 `lwouis/alt-tab-macos` 二开成用户自己的 macOS 窗口切换程序 **TabFlow**：独立 GitHub 仓库、独立应用名、独立 bundle ID、独立文档入口。保留上游全部功能与许可证义务。

成功标准：

1. GitHub 上存在 `webxiongda/tabflow`，本地仓库 `桌面/tabflow/` 与其连通。
2. 构建产物以 TabFlow 名义出现：应用名、bundle ID、About、菜单、更新源、仓库链接均指向新品牌。
3. 除许可证、上游署名、git 历史和测试夹具外，源码与文档中不再把产品称为 AltTab，也不再指向 `lwouis/alt-tab-macos`。
4. 不破坏上游窗口切换、快捷键、权限引导、Pro 授权相关代码路径。

## 2. 范围

### 2.1 纳入

- GitHub fork / 新仓库创建与本地克隆。
- 品牌与工程标识替换：应用名、仓库 URL、bundle ID、xcconfig、package.json、脚本、README、docs、changelog、appcast、Info.plist、菜单文案、About、品牌 SVG 文案。
- 首页 README 改写为 TabFlow 自有口吻，同时保留上游署名与 GPL-3.0 声明。
- 本地验证：残留扫描、Xcode 工程识别、构建（环境允许时）。

### 2.2 不纳入

- 上游 `src/` 里的窗口追踪、快捷键、面板渲染、Pro/license 业务逻辑改动。
- 新的付费、授权、崩溃上报或自动更新后端。
- 上游 commit 历史重写。
- 未获用户提供的自有域名、Sparkle 密钥、AppCenter Secret 的真实接入。

## 3. 品牌映射

| 项 | 上游 | TabFlow |
|---|---|---|
| 产品名称 | AltTab | TabFlow |
| GitHub 仓库 | `lwouis/alt-tab-macos` | `webxiongda/tabflow` |
| Bundle ID | `com.lwouis.alt-tab-macos` | `com.webxiongda.tabflow` |
| Xcode product name | `AltTab` | `TabFlow` |
| `package.json.name` | `alt-tab-macos` | `tabflow` |
| 本地目录 | `alt-tab-macos` | `tabflow` |
| 官网/更新站 | `alt-tab.app` | 暂时保留上游值并在 README 标注待替换，不伪造域名 |

说明：用户尚未提供自有域名与服务端。为避免制造假链接或破坏现有更新/授权链路，`DOMAIN`、`API_DOMAIN`、Sparkle、AppCenter、Pro 后端地址在第一刀保持上游值；README 明确标注这是待迁移项。该功能开关不影响品牌可见部分。

## 4. 文件级改动

1. 仓库元数据
   - 新建 GitHub 仓库 `webxiongda/tabflow`，可见性 public（GPL-3.0 要求源码可得）。
   - 保留上游 `LICENCE.md`、`changelog.md` 历史内容，只在顶部追加 TabFlow 说明。
2. 构建标识
   - `config/base.xcconfig`：`PRODUCT_NAME`、`PRODUCT_BUNDLE_IDENTIFIER`、`DOMAIN`、`API_DOMAIN`。
   - `alt-tab-macos.xcodeproj/project.pbxproj`：工程名、target 名、产物名、 entitlements / bridging header 文件名引用（若重命名为 `tabflow.*` 则同步引用）。
   - `Info.plist`：版权可见文案、`NSHumanReadableCopyright` 保留 GPL-3.0 说明。
   - `package.json`：`name` 改为 `tabflow`，`license` 与仓库实际 GPL-3.0 对齐。
3. 代码内可见标识
   - `src/App.swift`：`repository` 静态常量改为新仓库 URL；注释中“多个 AltTab 版本”改为“多个 TabFlow 版本”。
   - `Endpoints` / URL 构造处：保留后端域名，但把所有用户可见的 GitHub / 官网链接改为 TabFlow。
   - 仅替换字符串与注释，不改调用关系。
4. 文档与发布
   - `README.md`：重写为 TabFlow 自有介绍 + badge + 快速开始 + 上游致谢 + GPL-3.0。
   - `docs/readme/README.md`、`docs/contributing.md`、`docs/contributors.md`：链接与措辞同步。
   - `docs/readme/main.svg`：由 `scripts/build_readme_svg.py` 重新生成，替换标题文案与外链；保留动画与计数标记。
   - `appcast.xml`、`scripts/update_appcast.sh`、`scripts/update_website.sh`、`scripts/update_readme_and_website.sh`：release 下载 URL 与 changelog URL 改到新仓库；域名保持上游配置不变。
   - `changelog.md` 顶部追加 TabFlow fork 说明，历史条目原样保留。
5. 资源与脚本
   - `resources/` 与 `docs/readme/` 中可见的 AltTab 文字按同一映射替换。
   - 构建脚本里产品名变量默认值改为 TabFlow。

## 5. 约束与风险

- **许可证**：上游是 GPL-3.0。TabFlow 必须继续以 GPL-3.0 发布，保留上游版权与源码可得性；不能闭源或删署名。
- **Pro / 授权**：上游含 Pro 与 license client。第一刀不改其逻辑，也不替换后端；避免把别人的商业服务指向伪地址。
- **Sparkle / 更新**：appcast 签名密钥与 ED key 属于上游发布体系。TabFlow 第一刀不生成新签名，自建更新链路留待用户提供密钥。
- **Xcode / 签名**：本机构建可能因缺少上游证书或签名配置失败；验证以 `xcodebuild -list` 与可行的 Debug 构建为准，失败要如实报告。
- **品牌图**：`main.svg` 含动画与内嵌位图，必须用上游生成脚本重建，不手改 SVG。

## 6. 验收步骤

1. `git -C tabflow remote -v` 显示 `webxiongda/tabflow`。
2. 残留扫描：
   - `rg -n "AltTab|alt-tab-macos|lwouis/alt-tab-macos|alt-tab\\.app" tabflow` 的命中必须只出现在 LICENSE / changelog 历史 / 上游致谢 / 明确标注待迁移的域名配置处。
   - `rg -n "com\\.lwouis" tabflow` 无命中。
3. `xcodebuild -list -project tabflow/alt-tab-macos.xcodeproj`（或重命名后的工程）输出 TabFlow target。
4. 若本机 Xcode 可用：`xcodebuild -scheme TabFlow -configuration Debug build` 成功；若失败，记录具体错误并区分环境问题和改动问题。
5. README 首屏可读，包含 TabFlow 名称、安装/构建方式、GPL-3.0 与上游致谢。

## 7. 非目标

- 不新增专属功能、不做 UI 重设计、不迁移服务端。
- 不修改上游功能代码来“适配”品牌；品牌只通过标识、文案、元数据体现。
- 不在未获授权的情况下向用户 GitHub 之外发布、提 PR 或发版。
