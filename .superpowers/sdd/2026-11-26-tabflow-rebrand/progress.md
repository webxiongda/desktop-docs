# SDD ledger — plan: docs/superpowers/plans/2026-11-26-tabflow-rebrand.md

Pre-flight: Task 2 创建的 `scripts/verify_tabflow_brand.sh` 由 Task 3/4 的改动喂数据，由 Task 5 消费执行；命名与断言取值以计划中的脚本块为准。其余任务无共享接口。
Pre-flight: 本环境的 `sdd-workspace` / `task-start` / `task-done` 脚本缺失，改用本账本手写记录；每步仍按计划跑命令并读取输出。
Pre-flight: fork 目标是用户账号 `webxiongda`（type=User），不是 organization；计划里的 `-f organization=webxiongda` 需按 GitHub API 实际行为调整，裁决见下。

Task 1: Ruling: 省略 fork API 的 `organization` 参数 — `webxiongda` 是 User，GitHub fork API 的 organization 仅用于组织； forks 默认落在当前认证用户下 — 如果错了代价是一次 API 报错，可立即改用 PATCH 重命名已创建的 fork。
Task 1: complete (fork webxiongda/tabflow created from lwouis/alt-tab-macos; clone at 桌面/tabflow; remotes origin+upstream verified; license GPL-3.0)
Task 2: Ruling: 验收脚本初版把 `GPL-3.0` 写成许可证断言，但 LICENCE.md 实际正文是 `GNU GENERAL PUBLIC LICENSE` — 改为检查许可证正文并允许 README 中的上游致谢 — 代价：若上游换了许可证，断言需同步更新。
Task 2: complete (commits cdd99b68; red run showed 199 files with upstream identifiers; final script rewritten and re-run green in Task 5)
Task 3: Ruling: 不把 appcast.xml 的 296 条历史 release URL 当作产品身份单独保留 — 统一改为 `webxiongda/tabflow` 以便未来重生成；历史二进制尚未发布 — 代价：旧版本更新链接暂时不可用，符合“第一刀不建发布后端”约束。
Task 3: complete (commit e20674a6; xcodebuild -list shows project tabflow, target tabflow, TabFlow scheme; PRODUCT_NAME=TabFlow; bundle ID com.webxiongda.tabflow)
Task 4: complete (commit 5ebf686f; README + README_NOTE + changelog header + regenerated main.svg; brand verification script EXIT=0)
Task 5: Ruling: Debug 构建默认因本机缺少名为 `Local Self-Signed` 的签名证书失败 — 使用 `CODE_SIGN_IDENTITY=- CODE_SIGNING_ALLOWED=NO` 做 ad-hoc 签名复验，BUILD SUCCEEDED，且不改工程配置 — 代价：构建产物未正式签名，不能用于分发。
Task 5: complete (verify script EXIT=0; xcodebuild -list shows TabFlow scheme; ad-hoc Debug build succeeded; bundle id com.webxiongda.tabflow; pushed 950c1a4e..5ebf686f to origin/master)
Final review: subagent reviewer unavailable (429 rate limit until 2026-11-26 12:20 UTC+8); self-review against the five Review Focus items completed — no Critical/Important findings; evidence: brand script EXIT=0, `xcodebuild -list` shows TabFlow scheme, ad-hoc Debug build SUCCEEDED, remote README starts with TabFlow, working tree clean.
Final: global-context self-test `python3 -m unittest discover -s ~/.omp/agent/tests -v` → Ran 20 tests, OK, exit 0.
Final: minor (deferred): none.
