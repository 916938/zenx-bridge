# 上游同步策略（soft fork）

本文档记录本仓库与上游 `Tencent/BrowserSkill` 的关系定位、量化结论、同步规程和移植清单。

**定位：下游发行版（soft fork）。** 保留上游 remote 并持续吸收其修复，但对外身份、版本线和支持范围由我们自己定义。

**两个仓库各管一半：本文件是"记录"**（这一轮挑了什么、跳过什么、验证结果如何）；
**可执行的操作手册在兄弟仓库** —— `../zenx-bridge-skill/skill/references/upstream-sync.md`
（流程、不可孤立移植的功能线表、验证矩阵、环境坑），配套的只读辅助脚本
`../zenx-bridge-skill/scripts/upstream_sync.py`（`triage` / `applied` / `order` / `table`）。
动手前先读手册，收尾时更新本文件。任一侧出现**通用性**的结论（新的失败模式、新的功能线入口、
验证命令变化），另一侧要同步更新。仓库间的分工与互链规则见 `AGENTS.md` 的
"Companion repositories" 一节。

---

## 1. 三项既定决策

| # | 决策 | 含义 |
|---|------|------|
| 1 | **协议层必须持续与上游保持兼容** | 不修改握手/协议版本语义。这是低成本同步的前提：一旦协议分叉，cherry-pick 就从"小改"变成"必须重写"。 |
| 2 | **remote / server 模式不纳入移植范围** | 上游 standalone server（公网监听 + 设备配对 + TLS）与我们"只绑 loopback"的不变量冲突。当前基线合并已带入该代码，但**冻结**：不再接受其后续变更。zenx 未来若有远程需求，按自有方式实现，不继承上游该部分设计。 |
| 3 | **保留 Tencent remote，持续吸收修复** | 上游仍在高频修真 bug。仅当出现以下两种情况之一才考虑切断：① 上游在协议层做不兼容变更；② 上游强制默认走 server 模式。 |

---

## 2. 分歧量化（2026-09-19 实测）

| 指标 | 值 |
|------|-----|
| 我方独有提交 | 50（对上游 49） |
| 文件级差异 | 204 文件，+12973 / −13990 |
| **双方都改过的文件**（真实冲突面） | **28** |
| 我方独有代码 | 约 2024 行：`cli/invoke.rs` 671 + `cli/templates.rs` 670 + `daemon/templates.rs` 344 + `protocol/template.rs` 276 + `cli/completion.rs` 63 |
| daemon / protocol 净增 | 1162 行 |
| 版本 | 我方 CLI & 扩展 `0.2.3`；上游 `0.3.0` |
| **协议层** | `crates/bsk-protocol/src/system.rs` 握手逻辑**零分歧**；schema 仅差 `screenshot_full_page`（我方滞后，非有意分歧） |

身份层面其实早已分离：

- `cli/update.rs` 的更新源指向 `916938/zenx-bridge/releases`；
- `Cargo.toml` 的 `repository` 为我方；
- 扩展 manifest 含上游没有的 `cookies` 权限；
- `zenxbrowser`、`browserskill-pro` 已依赖 fork-only 的 CLI 面（`--browser-id`、`tab observe`、`browsers close`、`profile_account_id`），在上游构建上跑不起来。

---

## 3. 同步规程

### 3.1 不再整树 merge，改为定向 cherry-pick

整树 merge 的代价是每次人工裁决 28 个重叠文件（含 `App.tsx`、`connection-controller.ts`、`daemon/ws.rs`、`protocol/method.rs`、`skill/SKILL.md`）。改为：

1. `git fetch Tencent`（HTTPS 不通时改 SSH，见兄弟仓库的手册 §9）
2. 用 `git log --oneline Tencent/main --since=<上次同步日>` 列出候选；或用
   `python3 ../zenx-bridge-skill/scripts/upstream_sync.py triage --markdown`，
   它一次给出每个提交的规模、是否触碰冻结路径、与我方改动的重叠度
3. 按 §4 清单筛选：只挑 bugfix / 非 remote 的通用改进
4. `git cherry-pick <hash>`（冲突面降到 2–3 个文件）；挑多个前用
   `upstream_sync.py order <hash>...` 确认真实拓扑次序
5. 更新 §5 已移植清单（表格行可用 `upstream_sync.py table` 生成）

### 3.2 冲突热点（下次同步优先检查）

```
apps/extension/src/entrypoints/background.ts        # 上游 remote 接线 vs 我方 template client
apps/extension/src/entrypoints/popup/App.tsx        # 上游连接设置 UI vs 我方 useDaemonPort / profile account
apps/extension/src/session-manager/manager.ts
apps/extension/src/tools/dispatcher.ts              # 我方 browser.close / browser-tabs 分支
apps/extension/src/lib/connection-controller.ts     # 我方 profile-account 读取
crates/bsk-cli/src/daemon/ws.rs                     # 我方 template RPC
crates/bsk-protocol/src/method.rs
packages/i18n/src/locales/*/extension.json          # 双方都在加 key
skill/SKILL.md                                      # build.rs 会复制到 crates/bsk-cli/skill/SKILL.md
```

### 3.3 版本号与安装 URL 一律取我方

`Cargo.toml`、`Cargo.lock`、`apps/extension/package.json`、`packages/dsh-plugin-browserskill/package.json` 的版本号冲突一律保留我方；`README.md`、`README.zh-CN.md`、`AGENT_INSTALL.md`、`crates/bsk-cli/README.md` 的安装 URL 一律保留 `916938/zenx-bridge`，只吸收上游新增的 PATH 提示等附加说明。

> 版本号升级是独立的发布动作，不在同步提交里顺手改。

### 3.5 版本线规则

**我方版本号始终严格大于最后一次同步的上游版本。** 上游 `0.3.0` → 我方 `0.4.0`。这样单看版本号就能判断跑的是哪个发行版。

不要用 `0.3.0+zenx.1` 这类 build metadata 做区分，两个原因：

1. semver 比较**忽略** build metadata，`0.3.0+zenx.2` 与 `0.3.0+zenx.1` 判定为相等 → 自动更新会永远认为无需升级；
2. `scripts/release.mjs` 的版本正则 `^\d+\.\d+\.\d+(-[\w.]+)?$` 直接拒绝 `+`。

若将来我方号码即将与上游新版本相撞，继续向上跳一个 minor，而不是复用上游号码。

### 3.4 验证清单

```bash
cargo check -p bsk --locked
cargo test --workspace --locked --no-fail-fast
pnpm install --frozen-lockfile
pnpm --filter @browser-skill/extension exec wxt prepare
pnpm --filter @browser-skill/extension compile
pnpm ext:test
```

---

## 4. 不移植清单（Not ported）

以下上游内容**明确不跟进**，未来 cherry-pick 时跳过：

| 范围 | 路径 | 理由 |
|------|------|------|
| standalone server | `crates/bsk-cli/src/daemon/remote/**` | 公网监听口，违反 loopback 不变量 |
| 设备配对 / 凭据 | `apps/extension/src/transport/remote-authorization.ts`、`remote-storage.ts`、`remote-endpoint.ts` | 同上，属远程信任面 |
| 远程连接文档 | `docs/remote-extension-connection.md` | 不对外承诺该能力 |
| 远程模式测试 | `crates/bsk-cli/tests/remote_server.rs` | 见 §6 已知失败 |

现有树中的这部分代码是**基线合并的副产品，不是我们的支持范围**：不写进我方文档、不作为默认路径、不承诺行为。

---

## 5. 已移植清单（Ported）

### 5.1 2026-09-19 基线合并（`7edb391`）

一次性合入上游 49 个提交（我方 50 个），19 个文件冲突 / 30 个冲突块。主要吸收：

| 类别 | 代表提交 |
|------|---------|
| 后台标签执行与截图 | `f85876a` `6347ccf` `84f7bf7` `fb2d161` `c772fa0` `a7540c3` |
| 导航就绪与取消重定向 | `237667b` `c26560a` `d3f9e91` |
| VOM 兄弟上下文缓存（性能） | `f1e8374` |
| 输入就绪与原生输入结果 | `1edee0f` `1740a1d` `c8bfb98` `3519874` |
| 全页截图修复 | `ee5afd3` `3af3fe3` `37393f3` |
| dsh 插件输出排空 | `d947823` |
| 远程连接（冻结，见 §4） | `f925bde` `18aa202` `2647f72` |

合并保留的我方能力：`bsk invoke`、`templates`、`completion`、`browser.close`、`browser-tabs`、`profile account id`、`since` 游标、smart labels。

### 5.2 后续 cherry-pick

| 提交 | 日期 | 内容 |
|------|------|------|
| `eb91433` → `53126fc` | 2026-09-25 | fix(extension): bound request-help cleanup wait |
| `bfa5e52` → `e2dfaa9` | 2026-09-25 | fix: preserve startup ownership and isolate pending cleanup（`AgentWindowApi.create` 改返回 `{windowId, initialTabIds}`；冲突合并保留我方 `allocatingAgentWindows`/`pendingAgentWindows`；dsh 侧去掉依赖未移植功能的 `requestFor` 行；我方两个 fork 测试的 mock 与旧断言随之更新） |
| `214b1eb` → `307a116` | 2026-09-25 | fix(daemon): let navigation timeout results outlive the transport deadline |
| `6a97ac0` → `1bf0740` | 2026-09-25 | fix(cli): isolate Windows daemon startup from caller jobs（detach 链基础：`windows_process.rs`、`daemon/start/windows.rs`、Job breakaway 验证；干净合入，我方对相关文件零改动） |
| `58eb44b` → `286d6f5` | 2026-09-25 | fix(cli): preserve shared daemons after launcher timeout（detach 链后续，随 `6a97ac0` 一并移植） |
| `14afdbc` → `ae88a1a` | 2026-09-25 | fix(extension): preserve popup observation tails and safe cleanup（仅 `session.ts` 的 2 行对我方有效，popup 主体随 task-popups 跳过；在我方树上为行为等价改动） |
| `8519221` → `d7c5bfc` | 2026-09-25 | fix(extension): bound renderer reads and stop fallback reads after a timeout |
| `a317bf8` → `aff018b` | 2026-09-25 | fix(extension): keep frame discovery consistent after a read timeout |
| `e262457` → `f88afc5` | 2026-09-25 | fix(extension): refuse renderer reads while a timed-out read is still running（去掉属 `9cde489` 的 `claimAttempts`/`attachmentVersions` 字段声明） |
| `ad6dd44` → `c021ffb` | 2026-09-25 | fix(extension): preserve read budgets and isolate stalled frame reads（`stuckReads` 被上游重构为 `CdpReadGate`；测试冲突只保留导航预算用例，"bounded debugger cleanup" 两个用例依赖未移植的 `80dd02a`） |
| `cba47d4` → `5a40f6b` | 2026-09-25 | fix(extension): retain observations after optional root layout timeout |
| `7f4685d` → `8e7261d` | 2026-09-25 | fix(extension): treat overlay host as the click blocker |
| `8334b16` → `5e09b13` | 2026-09-25 | fix(extension): accept lowercase special keys in press |
| `225f5fe` → `9e4b393` | 2026-09-25 | fix(extension): treat ARIA loading tokens as case-insensitive |
| `46800bd` → `f27cb85` | 2026-09-25 | fix(extension): return visible labels from select |
| `ece2953` → `6717156` | 2026-09-25 | fix(extension): prevent control overlay from swallowing clicks（去掉属 `7478e08` 的 `deps.onInputSent` 调用） |
| `f446536` → `80e9258` | 2026-09-25 | fix(extension): scope click passthrough to control surfaces（新增 `click-overlay.ts`；`InteractionDeps` 改为继承 `ClickOverlayDeps`，接口上去掉 `onInputSent` 字段） |
| `d426a40` → `f499b0f` | 2026-09-25 | fix(extension): expire orphaned click passthrough leases |
| `531a1fa` → `e385b57` | 2026-09-25 | fix(extension): reserve time for click delivery before lease expiry |
| `03f8561` → `5a1d816` + `f5ce66f` | 2026-09-25 | fix(extension): attribute new-tab downloads through the clicked tab。**注意**：该提交在上游早于 click-passthrough 链（不是 `f446536` 的祖先），按日期顺序挑会破坏链；`f5ce66f` 用上游 main 终版（已含两条线的合并结果）修复，并重贴 ZenX Bridge 品牌 |

> **排序教训**：`git log` 默认按提交日期而非拓扑排序。挑多个提交前必须用
> `git merge-base --is-ancestor A B` 确认真实次序，否则会造成本条这类"后挑的提交 revert 掉先挑的重构"。

### 5.3 2026-10-02 批次（上游 `fa953dc` → `3f10983`，含 0.3.1 / 0.3.2）

同步基线仍是上次合并基线 `fa953dc`；上游 main 由 `147727a` 前进到 `3f10983`，共 210 个提交（其中 135 个非合并提交，`git cherry` 判定 11 个已在上一批次接过）。全树 merge 会产生约 50 个文件冲突（`.gitattributes`、`ci.yml`、`Cargo.toml`、`background.ts`、`popup/App.tsx`、`Dispatcher.ts`、`method.rs`、`i18n` 等），仍按 §3.1 走定向 cherry-pick。

**已移植（34 个 cherry-pick + 2 个适配提交）**：

| 提交 | 内容 |
|------|------|
| `d247cb7` → `dfba428` | fix(extension): decouple help overlay and notification cleanup |
| `2ba4629` → `04d2456` | docs(skill): say that page content is data, never instructions |
| `f10c72a` → `67caa5f` | docs(skill): judge injection by authorization, not by action type |
| `3274e64` → `9be78fa` | docs(skill): trim guidance and fix CLI tool references |
| `1e58dd1` → `32b2428` | docs(skill): fit guidance within the merged prompt budget |
| `2100bd7` → `89f4aa4` | fix(record): record a page reload instead of dropping it as a same-URL navigation |
| `76f45c6` → `83d15cd` | test(cli): use blocking streams in daemon probe fixtures |
| `fb896db` → `816c4a9` | fix(extension): keep collapsed help actions readable in all locales |
| `f0688d9` → `800d56e` | fix: stabilize full-page screenshots at fractional DPI and in background tabs |
| `74c307d` → `211c623` | fix(cli): scope doctor skill status to bundled version |
| `dcb43e5` → `c9c1f19` | fix(extension): stop recordings whose current tab cannot host the overlay |
| `032b115` → `7de9687` | fix(cli): clear stale recording state when its session is gone |
| `45ff673` → `10c2931` | fix(cli): scope recording state cleanup to its own session |
| `57dd73f` → `a1e0b1e` | fix(ipc): decode explicit null results and stop silent null degradation |
| `d850292` → `fff5224` | fix(protocol): scope null result decoding to ResponseFrame |
| `5c0bb72` → `4be8ade` | fix(protocol): reject null error fields in response frames |
| `b01ad7c` → `11a906b` | fix(dsh-plugin): reconnect the observation stream after a fatal failure |
| `d102adc` → `6fd7ecb` | fix(dsh-plugin): track observation stream outages explicitly |
| `8f485ab` → `8a6a56d` | fix(dsh-plugin): show observation outages on the collapsed capsule |
| `67629e8` → `c2067dc` | fix(emulate): scope remembered overrides to the session and attachment |
| `d7f7fb9` → `bb4bb59` | fix(extension): ignore stale overlay state by generation |
| `8e85b3e` → `4f660b1` | fix(input): honor persistent background execution leases |
| `58b50f9` → `b881ee2` | docs(changelog): note calls settling on extension reconnect |
| `00c1f18` → `0ae6a27` | fix(download): bound trigger lifetime and retire cancelled input（CI 冲突：上游 step 与我方既有 step 并存） |
| `78bd77e` → `736d8cf` | test(download): run cancellation regression in its own CI job（去掉引用上游 dsh 包名与 `scripts/check-dsh-package.mjs` 的 job） |
| `7efe13a` → `0f96967` | fix(download): preserve dispatched effects when the trigger fails（CI step 合并） |
| `c8269cc` → `6563799` | fix(click): emit every press in a DOM multi-click sequence（CI 取我方分 step 写法） |
| `fa18d3c` → `da16399` | fix(vom): preserve page actions beside nonmodal sidebars |
| `995f790` → `48f15d2` | fix(input): restore a dropped background lease before hidden input（代码取上游终版 `reapply()`，CHANGELOG 保留我方） |
| `b967b27` → `1838d01` | fix(download): fence dispatch and retire owned overlay resources（`chromium-cdp.ts` 保留我方，见下方适配提交） |
| `c61ae62` → `2285a94` | fix(extension): 防止 request-help URL 正则回溯阻塞（引入 `re2js` 依赖；丢弃上游 `references/help-and-recovery.md`） |
| `9ff0036` → `c5215cd` | fix(extension): 按维护者反馈约束 request-help URL 正则资源（同上） |
| `145d4fa` → `7f40a7b` | fix(extension): 按条件累计 URL 正则预算并防止轮询重叠（同上） |
| `4fde56f` → `58e38ff` | fix(vom): tolerate viewport rounding and honor AX modality（先补上同一批的 follow-up 提交后才干净合入） |
| — (`6bcb324`) | chore(extension): 补回 `b967b27` 依赖的 `CdpDispatchGuard` / `sendGuarded` / `command(beforeDispatch)`，但不引入属 `9cde489` 的 `attachmentVersions` |
| — (`17a7182`) | docs(skill): build.rs 重新生成的 `crates/bsk-cli/skill/SKILL.md` 副本 |

#### 2026-10-02 跳过及原因（重评估入口）

| 提交 | 原因 |
|------|------|
| `fd5aeee` / `83cc415` | 下载 target-blank 的**早期**实现（`expectedUrl`），已被上游终版（我方 `5a1d816`+`f5ce66f` 落的 `popupUrls`/`markDispatched`）取代，再挑会把终版代码回退到早期写法 |
| `2fc2704` | 给 session.stop 方案加 `requestId`，依赖未携带的 durable stop 恢复线（`abf0d3a`） |
| `2aca02a` | 依赖 `overlay-version.ts` 与 `popupSnapshotRefreshers`（popup 归因线），我方用 `overlayGeneration` 计数器 |
| `ca02b26` / `a6c3498` / `bedcf90` | 依赖 `start_session_recoverable`、`preserve_cleanup`、`unresponsive` 标记（可恢复会话启动线 `8e357f3` / `abf0d3a`）；`bedcf90` 单独有 30 个冲突块 / 11 文件，需先整条吸收该线 |
| `507522f` / `72986b2` | 上游把启动恢复提示抽成 `daemon/start_error.rs` 模块，我方内联在 `cli/ensure_daemon.rs` / `cli/doctor.rs`，单独挑需连带引入该模块与全部接线 |
| `fc7561b` `1c74821` `15c0bab` `eb9b142` `b56bb43` `77b78b8` `c7c3693` `b3b133e` `d691cc1` | Windows 自更新/接管链（9 提交）。我方 `cli/update.rs` 与上游终版相差 1566 行（含我方 release 源），且引入新的 `update/state.rs`、`daemon/start/handover.rs`。属 feature 级改造 + 发布链路风险，**建议单独立项**而不是搭在本轮同步里 |
| `b4c41ee` `f443309` `6f6e09e` `a965e6d` `bbde2d7` `2b01a15` `bf5a8d9` `ccaa317` `b8b744a` | dsh 插件（runner 收尾、截图卡片、DSH 0.2 宿主、归档祖先）。同 §6 既有结论：dsh 测试基线即失败，真正修复入口是 SDK `0.1.0-rc.6 → 0.1.5-rc.3` 迁移线 |
| `02e802f` `47f5765` `240bf1d` `45a3bf1` | skill 打包/reference bundles 线（`skill_install/bundle.rs`、`sync.rs`、`legacy-digests.txt`、`.gitattributes`），我方未采用该结构 |
| `f360a34` `0ac871b` `e8c5cfc` `47cf368` `770f7a1` `8a7f280` | i18n 多语言包（新增 7 个 locale + `import.meta.glob` 自动注册），feature 线；单独挑会留下无构建入口的目录 |
| `ceb5f65` `da0d038` `ed7f85a` `aa96082` | skill 文档改动，我方 `SKILL.md` 已按 ZenX 品牌重排章节（"Before starting a session" 等），需要的 parts 应在单独的文档轮次手工合并 |
| `bbada7b` `dc80625` | 仅涉及 `scripts/check-crate-skill.mjs` / `check-dsh-package.mjs`，我方无对应脚本 |
| `1e2b175` `a5d9201` `476500b` `3383a4d` | 隐私声明 / README 与版本号。README、CHANGELOG、版本号一律保留我方（§3.3）；release 提交不落在同步批次里 |
| `a3f8200` 及其余 `feat(debug)`/`fix(debug)`（16 个） | 上游"网站排障取证据"功能线（`apps/extension/src/debug/**`、`crates/bsk-cli/src/cli/debug.rs`、`protocol/tools/debug.rs`），我方树中零这些模块，属新功能而非修复 |

**Windows 生命周期测试的运行环境**：`windows_daemon_start` / `windows_update` 的多数用例要求宿主进程**不在限制性 Job 内**。在 IDE/沙箱 shell 里直接 `cargo test` 会因 `Access is denied (os error 5)`（breakaway 被拒）大片失败，属环境限制而非代码问题。正确跑法：`powershell -File scripts/test-windows-daemon.ps1`（与 CI 相同的 WMI 独立宿主，2026-09-25 全绿；偶发 `launcher did not exit within 8s` 超时重跑即可）。

#### 2026-09-25 跳过及原因（重评估入口）

| 提交 | 原因 |
|------|------|
| `abf0d3a` | 与上游"可恢复会话启动"功能线（`session-starts.ts`、`start-journal.ts`，`8e357f3`）深度耦合，我方未携带该功能 |
| `80dd02a` | 依赖上游任务 UI 功能线（`task-preview.ts`、`ui-activity.ts`、`claimAttempts`，`9cde489`/`a15e857`），我方未携带 |
| `345d703` / `69dfd06` | 均修复上游 browser rename 功能（`05db608`），我方未携带 |
| `7b74596` / `7478e08` / `64245fb` | popup 观察/归因线（`task-popups.ts`、`observedTabs`、`releaseObservedTab`），我方整体未携带该子系统；`7b74596` 曾试挑后 revert（残余部分离开 `task-popups` 即为死代码）。若需要 popup 归因，应整条吸收 `7478e08` 起的功能线 |
| `9cde489` | 任务 UI 线（引入 `claimAttempts`/`attachmentVersions`），且被 `80dd02a` 依赖 |

**教训**：上游 0.3.1 的修复大量挂在功能线（可恢复启动、任务 UI/rename）上，逐修复 cherry-pick 的命中率低。下次同步前应先决定是否整条吸收某个功能线，再批量移植。

---

## 6. 已知失败

- `cargo test -p bsk --test remote_server`：Windows 上 1–2 例不稳定失败（`authorization_file_contention_does_not_block_socket_messages` 断言 401 vs 503；`sixty_four_online_browsers_leave_http_exchange_capacity_available` 报 `Access is denied (os error 5)`）。该测试文件与 `daemon/remote/**` 均相对上游零改动，属 Windows 文件争用语义差异；按 §4 该项不在我们的支持范围，不修。
- `pnpm --filter @916938/zenx-bridge-dsh-plugin test`：`tests/skill.test.ts` 2 例 + `tests/lazy-tools.test.ts` 7 例失败（2026-09-25 在同步前基线 `e908115` 上复现，属**既有失败**，非同步引入）。

  **lazy-tools 修复组评估结论（2026-09-25，已实测并回滚，判定为「未修复」）**：

  | 评估项 | 结论 |
  |--------|------|
  | 范围 | `2eee76f` → `e1a4018` → `a4ac06e` → `963f694`（线性链，全部只动 dsh 插件），另需 `start-journal.ts`（自包含，来自 `8e357f3`）作测试 fixture |
  | 场景对应 | 对应：lazy-tools 7 例（`armLazyTools`/`hasSuccessfulSkillInvocation`/`apply()` 接线）确实就是这批修复覆盖的场景 |
  | 修复前 | 9 例失败（skill 2 + lazy-tools 7） |
  | 修复后 | **33 例失败**：新代码与新测试针对 dsh SDK `0.1.5-rc.3`，我方装在 `0.1.0-rc.6`；叠加 SDK 升级提交 `b8b744a` 后仍失败（客户端组件等还需上游更多迁移提交） |
  | 结论 | **未修复，且不可孤立移植**。这 9 例的修复入口是整条 dsh SDK `0.1.0-rc.6 → 0.1.5-rc.3` 迁移线，是独立于本次同步的工程项，需单独立项评估（含 `plugin-id.test.ts` 的 Cordis 插件 id 断言） |
- 其余目标全绿（2026-09-25：`cargo test --workspace --no-fail-fast` 仅 remote_server 失败；`pnpm ext:test` 2169 passed / 104 skipped / 0 failed，131 files；`scripts/test-windows-daemon.ps1` WMI 宿主全绿）。
- 2026-10-02 同步批次的验证结果：`cargo check -p bsk --locked` 通过；`scripts/test-windows-daemon.ps1` WMI 宿主全绿（startup 10 + launcher 1 + update 3）；在 IDE 内直接 `cargo test` 时 `windows_daemon_start` / `windows_update` 仍报 `Access is denied (os error 5)`，属本节已记录的环境限制。`pnpm --filter @browser-skill/extension compile` 通过；`vitest run` 2394 用例 / 2277 passed / 0 failed。
- 本批次新增失败：无；`src/long-screenshot/png.test.ts` 的 12 万扫描线用例在并发满载时会退化为 `STACK_TRACE_ERROR`，单独重跑通过（与 §6 已记录的偶发一致）。
- `src/long-screenshot/*` 两个用例在全量并发跑时偶发失败，单独重跑通过（负载相关，非回归）。

---

## 7. 待办（soft fork 身份步骤，未包含在本轮同步）

1. ~~版本线独立：与上游 `0.3.0` 显式区分。~~ **已完成** —— 当前 `0.4.0`，规则见 §3.5。
2. 对外品牌去 "BrowserSkill"（上游商标）。CLI 二进制名 `bsk` **保留**——`zenxbrowser` 与 `browserskill-pro` 全部脚本硬编码依赖，改名收益不抵成本；且我们不 `cargo publish`，不存在 crate 名冲突。
3. `LICENSE`：保留 Tencent 的 MIT 版权行，另起一行加我方 copyright 与 modified 说明（MIT 硬性要求）。
4. `browserskill-pro` / `zenxbrowser` 文档写明"依赖 fork 构建，非上游 BrowserSkill"，列出 fork-only 命令。
5. `AGENTS.md` 不变量 #2 需改写为"默认且受支持的模式是 loopback；上游 remote/server 模式虽在树中但不受支持"。
