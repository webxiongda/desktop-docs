# TabFlow 二开实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 subagent-driven-development（推荐）或 executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 把 `lwouis/alt-tab-macos` 二开为 `webxiongda/tabflow`，完成品牌、仓库身份、构建标识和文档的切换，同时保留 GPL-3.0 与上游功能。

**架构：** 先创建 GitHub fork 并克隆到 `桌面/tabflow/`，再用可执行的残留扫描脚本作为验收测试驱动批量替换标识，最后重建品牌图、改写自有 README、跑 Xcode 工程校验与构建。所有改动限定在标识、文案、元数据，不改 `src/` 业务逻辑与授权后端。

**技术栈：** Swift / Xcode 工程、bash、Python 标准库（`scripts/build_readme_svg.py`）、GitHub CLI、ripgrep、`edit` / `write` 文件工具。

**规格：** `docs/superpowers/specs/2026-11-26-tabflow-rebrand-design.md`

## 全局约束

- 上游许可证为 GPL-3.0；TabFlow 必须继续 GPL-3.0 发布并保留上游署名。
- 品牌名：TabFlow；仓库：`webxiongda/tabflow`；本地目录：`桌面/tabflow/`。
- Bundle ID：`com.webxiongda.tabflow`；Xcode product name：`TabFlow`。
- `DOMAIN`、`API_DOMAIN`、Sparkle、AppCenter、Pro 后端地址第一刀保持上游值，README 标注待迁移，不伪造域名。
- 不改 `src/` 里的窗口切换、快捷键、面板、Pro/license 业务逻辑；只改字符串、注释、元数据。
- 文件编辑只用 `edit` / `write`，不用 `sed` / `perl` / Python 脚本做原地改写。
- 提交信息用英文前缀 + 中文正文，例如 `feat: 切换 TabFlow 品牌标识`。

## 审查重点（Review Focus）

1. `com.lwouis` 或 `alt-tab-macos` 作为 bundle ID / 工程产物名残留 → 应用身份仍是上游，验收脚本必须零命中（除许可证和致谢）。
2. README / 菜单 / About / SVG 仍显示 AltTab → 用户看到的仍是别人的产品，验收脚本与 SVG 重生成必须覆盖。
3. `appcast.xml` 或发布脚本仍指向 `lwouis/alt-tab-macos/releases` → 更新会拉上游包，验收脚本必须命中并报失败。
4. GPL-3.0 许可证或上游致谢被误删 → 法律风险，验收脚本必须确认 `LICENCE.md` 与 README 致谢段存在。
5. Xcode 工程重命名后引用断裂（bridging header / entitlements / xcconfig）→ 构建失败，`xcodebuild -list` 和 Debug 构建必须验证。

---

## 任务 1：创建并连通 TabFlow 仓库

**文件：**
- 创建：`tabflow/`（整个克隆目录）

- [ ] **步骤 1：用 GitHub API 创建 fork 并命名**

运行：
```bash
gh api repos/lwouis/alt-tab-macos/forks -f name=tabflow -f organization=webxiongda
```
预期：返回 JSON 中 `full_name` 为 `webxiongda/tabflow`，`parent.full_name` 为 `lwouis/alt-tab-macos`。

- [ ] **步骤 2：克隆到桌面工作区**

运行：
```bash
git clone https://github.com/webxiongda/tabflow.git tabflow
git -C tabflow remote add upstream https://github.com/lwouis/alt-tab-macos.git
```
预期：`tabflow/` 存在，`origin` 指向自己的仓库，`upstream` 指向上游。

- [ ] **步骤 3：验证远端**

运行：
```bash
git -C tabflow remote -v
gh api repos/webxiongda/tabflow --jq '.full_name + " | " + .license.spdx_id'
```
预期：远端正确；输出 `webxiongda/tabflow | GPL-3.0`。

- [ ] **步骤 4：Commit（本任务无文件改动，跳过）**

无需 commit。

## 任务 2：增加品牌残留验收脚本并先跑红

**文件：**
- 创建：`tabflow/scripts/verify_tabflow_brand.sh`

- [ ] **步骤 1：编写验收脚本**

用 `write` 创建 bash 脚本，`set -euo pipefail`，在项目根目录执行以下断言，任一失败即退出 1：

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
fail=0
check_clean() {
  local pattern="$1" label="$2"; shift 2
  local args=(. --glob '!.git/**')
  local allow
  for allow in "$@"; do args+=(--glob "!$allow"); done
  local hits
  hits=$(rg -l "$pattern" "${args[@]}" || true)
  if [ -n "$hits" ]; then
    printf 'FAIL %s:\n%s\n' "$label" "$hits"
    fail=1
  fi
}
check_clean 'com\.lwouis' '残留上游 bundle ID'
check_clean 'lwouis/alt-tab-macos' '残留上游仓库 URL' 'README.md' 'changelog.md' 'LICENCE.md'
check_clean 'AltTab' '残留 AltTab 产品名' 'README.md' 'changelog.md' 'LICENCE.md'
check_clean 'github\.com/lwouis' '残留上游 GitHub 链接' 'README.md' 'changelog.md'
check_clean 'alt-tab\.app' '残留上游域名' 'docs/README_NOTE.md' 'config/base.xcconfig' 'appcast.xml' 'scripts/**' 'src/**'
rg -q 'TabFlow' README.md || { echo 'FAIL README 缺少 TabFlow'; fail=1; }
rg -q 'GPL-3\.0' LICENCE.md || { echo 'FAIL 许可证丢失'; fail=1; }
rg -q 'webxiongda/tabflow' README.md || { echo 'FAIL README 缺少新仓库'; fail=1; }
exit $fail
```

- [ ] **步骤 2：运行脚本，确认当前失败**

运行：`bash tabflow/scripts/verify_tabflow_brand.sh`  
预期：FAIL，列出 `AltTab`、`com.lwouis`、`lwouis/alt-tab-macos` 等残留。

- [ ] **步骤 3：Commit 测试脚本**

```bash
git -C tabflow add scripts/verify_tabflow_brand.sh
git -C tabflow commit -m "test: 增加 TabFlow 品牌残留验收脚本"
```

## 任务 3：切换工程与代码标识

**文件：**
- 修改：`tabflow/config/base.xcconfig`
- 修改：`tabflow/Info.plist`
- 修改：`tabflow/package.json`
- 修改：`tabflow/tabflow.xcodeproj/project.pbxproj`
- 修改：`tabflow/src/App.swift`
- 重命名：`tabflow/alt-tab-macos.xcodeproj` → `tabflow/tabflow.xcodeproj`
- 重命名：`tabflow/alt-tab-macos-Bridging-Header.h` → `tabflow-Bridging-Header.h`
- 重命名：`tabflow/alt_tab_macos.entitlements` → `tabflow.entitlements`

- [ ] **步骤 1：替换 xcconfig 中的工程身份**

在 `config/base.xcconfig` 中把 `PRODUCT_NAME = AltTab` 改为 `PRODUCT_NAME = TabFlow`，把 `PRODUCT_BUNDLE_IDENTIFIER = com.lwouis.alt-tab-macos` 改为 `com.webxiongda.tabflow`，把 `SWIFT_OBJC_BRIDGING_HEADER` 和 `CODE_SIGN_ENTITLEMENTS` 的文件名改为步骤 3 的重命名结果。`DOMAIN` 与 `API_DOMAIN` 保持 `alt-tab.app` 不动。

- [ ] **步骤 2：替换 package.json 身份**

把 `name` 改为 `tabflow`，把 `license` 从 `MIT` 改为 `GPL-3.0`，其余字段不动。

- [ ] **步骤 3：重命名桥接头与 entitlements 文件**

用 `edit` 的 `MV` 操作执行三个重命名（含 xcodeproj 目录），随后把 `project.pbxproj` 中所有旧文件名与目录引用同步为新名称；不要改文件内部代码。

- [ ] **步骤 4：替换 App.swift 可见标识**

把 `static let repository` 的值改为 `https://github.com/webxiongda/tabflow`；把注释里“多个 AltTab 版本”改为“多个 TabFlow 版本”。不改任何其他逻辑。

- [ ] **步骤 5：按残留清单替换其余字符串**

运行 `rg -n 'AltTab|alt-tab-macos|com\.lwouis|github\.com/lwouis' tabflow --glob '!.git/**' --glob '!LICENCE.md' --glob '!changelog.md'`，对每一处命中用 `edit` 做等价替换：`AltTab` → `TabFlow`，`alt-tab-macos` → `tabflow`（路径引用按实际重命名结果），`com.lwouis.alt-tab-macos` → `com.webxiongda.tabflow`，`github.com/lwouis/alt-tab-macos` → `github.com/webxiongda/tabflow`。`alt-tab.app` 域名保持不动，留给任务 5 写说明。

- [ ] **步骤 6：验证工程可识别**

运行：
```bash
xcodebuild -list -project tabflow/tabflow.xcodeproj
```
预期：输出中包含 `TabFlow` target/scheme；若本机 Xcode 缺失或报环境错误，记录原文并继续，不用改代码绕过。

- [ ] **步骤 7：Commit**

```bash
git -C tabflow add -A
git -C tabflow commit -m "refactor: 切换 TabFlow 工程与代码标识"
```

## 任务 4：改写发布文档并重建品牌图

**文件：**
- 修改：`tabflow/README.md`
- 修改：`tabflow/docs/readme/README.md`
- 修改：`tabflow/docs/contributing.md`
- 修改：`tabflow/docs/contributors.md`
- 修改：`tabflow/changelog.md`（仅顶部追加）
- 修改：`tabflow/appcast.xml`
- 修改：`tabflow/scripts/update_appcast.sh`
- 修改：`tabflow/scripts/update_website.sh`
- 修改：`tabflow/scripts/update_readme_and_website.sh`
- 修改：`tabflow/scripts/build_readme_svg.py`
- 重新生成：`tabflow/docs/readme/main.svg`

- [ ] **步骤 1：重写 README**

用 `write` 覆盖 `README.md`，首屏含：TabFlow 一句话定位、平台 macOS 12+、快捷键/窗口切换特性列表、从源码构建命令、GPL-3.0 声明、`Forked from lwouis/alt-tab-macos` 上游致谢、`webxiongda/tabflow` badge。不要写任何未提供的域名或下载站。

- [ ] **步骤 2：同步 docs 文案**

用 `edit` 把 `docs/readme/README.md`、`docs/contributing.md`、`docs/contributors.md` 里的产品名称、仓库链接、官网链接换成 TabFlow / `webxiongda/tabflow`；`alt-tab.app` 域名只在“待迁移”说明里保留。

- [ ] **步骤 3：更新 changelog 顶部**

在 `changelog.md` 最前面追加一段：`## TabFlow fork — 2026-11-26` + “从 lwouis/alt-tab-macos fork，品牌与工程身份切换为 TabFlow，功能逻辑未改动。” 历史条目原样保留。

- [ ] **步骤 4：更新 appcast 与发布脚本**

把 `appcast.xml` 和 `scripts/update_appcast.sh`、`scripts/update_website.sh`、`scripts/update_readme_and_website.sh` 中的 release 下载 URL 与 GitHub 仓库 URL 改到 `webxiongda/tabflow`；changelog URL 改到新仓库 raw 地址；域名变量保持上游值不动。

- [ ] **步骤 5：重建品牌 SVG**

先读 `scripts/build_readme_svg.py`，把标题/字幕文案改成 TabFlow，把外链改成 `https://github.com/webxiongda/tabflow`，再运行：
```bash
cd tabflow && python3 scripts/build_readme_svg.py --stars 16K
```
预期：`docs/readme/main.svg` 更新；用 `rg -n 'AltTab|alt-tab' tabflow/docs/readme/main.svg` 检查零命中。

- [ ] **步骤 6：写域名待迁移说明**

用 `write` 创建 `tabflow/docs/README_NOTE.md`，说明 `DOMAIN` / `API_DOMAIN` / Sparkle / AppCenter / Pro 后端仍使用上游配置，原因与迁移前置条件（自有域名、密钥、服务端）。这是验收脚本唯一允许出现 `alt-tab.app` 的文档。

- [ ] **步骤 7：Commit**

```bash
git -C tabflow add -A
git -C tabflow commit -m "docs: 建立 TabFlow 自有文档与发布配置"
```

## 任务 5：跑通验收并推送

**文件：**
- 无新增文件；运行验证并根据输出修复

- [ ] **步骤 1：跑品牌残留验收**

运行：`bash tabflow/scripts/verify_tabflow_brand.sh`  
预期：退出码 0，无 FAIL 行。

- [ ] **步骤 2：跑 Xcode 构建**

运行：
```bash
xcodebuild -scheme TabFlow -configuration Debug -project tabflow/tabflow.xcodeproj build -derivedDataPath /tmp/tabflow-derived
```
预期：BUILD SUCCEEDED。若因缺少上游证书/密钥失败，记录错误原文，只允许修工程引用类问题，不得改业务代码绕过。

- [ ] **步骤 3：检查 git 状态与远端**

运行：
```bash
git -C tabflow status --short
git -C tabflow log --oneline -5
git -C tabflow remote -v
```
预期：工作区干净，最近提交为本计划产生的 3 个 commit，`origin` 为 `webxiongda/tabflow`。

- [ ] **步骤 4：推送到自己的仓库**

运行：`git -C tabflow push -u origin HEAD:master`  
预期：远端 `master` 与本地一致。

- [ ] **步骤 5：Commit（无新增改动，跳过）**

无需 commit。
