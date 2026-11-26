# dsh-live-compact fork：Windows 不再改写备份 ACL

- 状态：已实施（工作区改动，未提交）
- 日期：2026-11-26
- 上游基线：`Tranquillitatis-Lab/dsh-live-compact` v1.1.14（`20d7d94`）；工作区实际是 1.1.14 外加一个未发布 commit（`055f78d`）
- 本机现场：desktop profile 在装 fork 之前是 1.1.4；曾录到 190 次 PowerShell / 120 秒，`execFileSync` 把事件循环占满

## 0. 结论

Windows 上不再重建备份的 DACL，也不再启动 `powershell.exe` / `whoami.exe`。改动是 `shadow.js` 两处（新判定函数 + 早退）、`index.js` 三处（引入 + 两处恢复标记），外加一份 `FORK.md`；README、`package.json`、测试一条没动，所以上游合并面只有 `tightenShadow` 顶部两行和恢复处两行。

只关 ACL 不够：恢复记录会被标成「还没复核权限」，修剪前那道 32 条的门槛会把整轮修剪挡掉，长会话等于不修剪。所以恢复时同时把 `aclVerified` 记为已复核。

目标场景是单机单账户；共享目录、网络盘、指到别处的 `DSH_HOME` 都不提供隔离保证。

## 1. 为什么去掉

上游为什么有这套 ACL，和它值不值得留，是两件事：

- 1.0.6 时备份目录只有 `chmod 0700/0600`，在 Windows 上等于没做。issue #14 的外部审计把它定为 P0：「新建目录/文件的 ACL 由插件主动建立，非继承获得；非授权主体读取失败」。作者按验收标准实现了「完整替换 DACL + 写后读回」。
- 之后每次事故加一层：#25/#28（`icacls /restore` 普通用户恒失败）、#38（恢复路径逐文件收紧，223 文件 148 秒）、#67（`-EncodedCommand` 被拒）。复杂度是叠出来的，不是设计出来的。
- 对我们的场景：本机实测 `icacls`，`%USERPROFILE%`、`%USERPROFILE%\.dsh`、`...\compaction-spine\shadow` 三级的允许主体都是 SYSTEM + Administrators + 当前用户，profile 本身是受保护 ACL，重建一遍结果相同。边际保护接近零，成本是每个文件一次 PowerShell。
- 横向对比：`ranxianglei/billion-context` 不做 DACL，靠 profile 默认 ACL + 可选 AES-256-GCM；`T-Auto/dsh-ops` 只给工作区根加一条 WRITE_OWNER，每工作区一次、缓存、异步。

## 2. 1.1.4 的事故与 1.1.13 的修复

| 版本 | 恢复路径 | 修剪前校验 | 现场表现 |
|---|---|---|---|
| 1.1.4 | `readRawToolResult` 默认 `tighten: true`，每文件 2 次子进程 | 每个可见文件重新授权，无上限 | 约 50 个工具结果即 190 次 spawn / 120 秒 |
| 1.1.13+ | 恢复只读（`tighten: false`） | 上限 32 个文件，超过就整轮拒修剪 | 事故形态消失；小会话一次性 ≤64 次，长会话不修剪 |

这次改动要同时处理第二条：关掉 ACL 之后没人再把 `aclVerified` 置为 true，门槛会一直挡着。

## 3. 改动

### 3.1 `shadow.js`：判定 + 早退

`shadow.js:606` 新增：

```js
// fork：真实 Windows 上不改写备份 ACL，信任系统目录权限；见 FORK.md。
// 注入 runner/execFile 的调用仍是老路径，POSIX 不受影响。
export function shadowAclSkipped(access = {}) {
  const platform = access.platform ?? process.platform;
  return process.platform === "win32" && platform === "win32" && !access.runner && !access.execFile;
}
```

`shadow.js:1318`（`tightenShadow` 顶部，熔断检查之后、runner 构造之前）：

```js
if (shadowAclSkipped(access)) return;
```

三个条件缺一不可：

1. **真实 `process.platform === "win32"`**：POSIX 上一律不跳过。`p0-gate.mjs` 里有一条用例在 POSIX 上模拟 `platform: "win32"` 且不注入执行器、断言必须抛 `ESHADOWACCESS`，靠这个条件继续走老路径，测试不用改。
2. **`platform === "win32"`**：`access.platform` 显式传别的平台时按老规则处理，注入 runner 的用例 `windows` 本来就是 false。
3. **`!access.execFile`**：上游测试全部通过注入 `execFile` 的假执行器来验证 ACL，带上它这些用例仍走老路径。生产调用不传 `execFile`。

### 3.2 `index.js`：恢复记录直接标已复核

`index.js:43` 引入 `shadowAclSkipped`；`index.js:3096` 与 `index.js:3142` 两处恢复记录，成功分支从 `aclVerified: false` 改为 `aclVerified: shadowAclSkipped(access)`。

不加这两行的后果：

- `rawVerifiedSet` 里「待复核 > 32」的门槛一直成立，`runPruneAfterCapture` 对缺失 id 抛错，宿主整轮跳过修剪，日志打 `spine: prune skipped: raw backup missing`。
- 修剪跳过之后，切片前 `commitSelectedSpan` 仍按 span 内每条工具结果各 spawn 两次。

加了之后这条链就顺了：待复核计数恒为 0，32 条门槛不触发，修剪正常跑，切片不再重复授权。

校验没有减少：`rawVerifiedSet` 每轮仍然逐条读 raw/meta、比对 path / bytes / sha256；切片前 `verifiedRaw` 仍然读文件核 hash；文件身份、锁、rotation 全部不变。失败分支（`record.ok` 为假）保持 `aclVerified: false`，不冒充已复核。

### 3.3 `FORK.md`

新建，写清：改了什么、没改什么、安全边界、升级上游、怎么确认生效。原提案要整段改写 README，实际没动——README 是上游每次发版都会碰的高频冲突区，说明放独立文件冲突面为 0。

### 3.4 不改的部分

- `package.json`：`name`、`version` 都不动。版本号跟着上游走，合并后自然变成上游版本。
- `README.md`：不动。上游的 Windows 段落对本 fork 不适用，`FORK.md` 里写明。
- 测试：不动。`test/windows-acl.test.mjs`、`test/issue-20.mjs` 等继续覆盖注入路径。
- `WINDOWS_ACL_SCRIPT`（约 3.3KB）：留在包里不执行。
- `index.js` 里两处 `shadowAccessFused()` 守卫：Windows 上成为死分支，保留。

## 4. 行为影响

| 项 | 改动前（1.1.14） | 改动后 |
|---|---|---|
| Windows ACL 子进程 | ≤32 个文件 × 2 次（一次）；>32 时不跑但拒修剪 | 0 |
| 宿主修剪 | 小会话做完复核后照常；长会话整轮跳过 | 恒常执行 |
| 切片前的重授权 | 对未复核文件逐个 spawn | 跳过，仍读文件核 hash |
| 失败语义 | 权限不达标抛 `ESHADOWACCESS` 并熔断本进程后续备份 | 只剩写失败、哈希不符、容量拒绝；真实 I/O 错误仍原样抛出 |
| 安全边界 | 默认根等价 profile ACL；自定义根有隔离 | 默认根 = profile ACL；自定义根无隔离 |
| POSIX | 0700 / 0600 + 读回校验 | 不变 |
| 测试注入路径 | 老路径 | 老路径（条件里排除了注入） |

真实读写错误与 ACL 是两条独立的路：`openSync` / `fsyncSync` / `renameSync` 失败时原错误上抛，上层记 `spine: raw backup failed: <message>`、`reason: "unwritable"`，只有 `EAGAIN` / `EBUSY` / `EINTR` / `EIO` 标可重试；切片前备份写不出来则抛 `spine: shadow backup was not written`，原段不动。

要记账的一处代价：以前长会话（>32 个待复核文件）修剪整轮跳过，一个文件都不读；现在每轮修剪前会把可见备份读一遍并核 hash，最大会话（1331 个文件）约 0.3 秒，一两百个文件的会话 20–30 毫秒。这是恢复「真的会修剪」换来的，不是回归。

## 5. 上游合并

1. `git remote add upstream https://github.com/Tranquillitatis-Lab/dsh-live-compact.git`（当前 clone 只有 `origin`，没有 tag）。
2. 发版时 `git fetch upstream && git merge upstream/main`。
3. 冲突面只有两处：`tightenShadow` 顶部两行；恢复处的两行。
4. 上游若改了注入方式（不再用 `access.execFile`），`shadowAclSkipped` 的条件会失效、注入式用例会走新路径，需要同步改条件。
5. 上游若把 ACL 挪到别的入口（例如 `writeShadowFile` 直接调 runner，绕过 `tightenShadow`），早退点失效，要跟到新入口补一行。
6. 合并后跑 `npm run test:gate`；本机需要先把 `node_modules` 指向宿主包（junction 到 `~/.dsh/profiles/node_modules`），否则 `@deepseek-ai/*` 解析不到。

## 6. 验证

已做：

- `test/windows-acl.test.mjs` 30 通过 / 0 失败；`test/issue-20.mjs` 11 / 0。
- 整条 `test:gate` 与改动前逐条对比，失败集合完全一致（本机缺配对的宿主包版本，`p0-12`、`p0-17` 本来就失败，与本次无关）。
- 探针（真实 Windows，未注入）：恢复后 `aclVerified === true`，`rawVerifiedSet` 返回该 id；真实写入 29ms，内容与大小正确。
- 性能实测（本机，热缓存）：ACL 脚本一次 284–325ms；写 64KB 备份 15–22ms；修剪前校验 103 个文件 23ms、115 个 26ms、1331 个 273–344ms。

待做（由用户自己装）：

1. 用桌面「设置 → 插件」填本工作区路径，或直接拷到 `profiles/desktop/node_modules/dsh-live-compact`，替换掉 1.1.4。
2. 完全退出应用再启动。
3. 确认装的是这份：插件目录里 `grep -n shadowAclSkipped shadow.js index.js`。
4. 长会话跑一轮，任务管理器里 `node.exe` 下不应再有 `powershell.exe`；日志里不应再出现 `prune skipped: raw backup missing`，`spine-cuts.jsonl` 应继续增长。

## 7. 风险与回退

- **共享目录 / 自定义根没有隔离**：决策的一部分，不是缺陷。同账户进程、管理员、备份软件、云同步本来也不在防护范围内。
- **默认根被系统策略放宽时没有检测**：没有一次性 `icacls` 读校验。需要时可加，每进程一次。
- **上游把 ACL 挪走**：早退点失效，回到每文件 spawn；靠现场验证里的 `powershell.exe` 检查发现。
- **未来有人给生产路径传 `execFile`**：短路静默失效，注释里已写明这个条件只对注入生效。
- **回退要两处一起退**：删早退的同时把恢复处两行改回 `aclVerified: false`，否则记录会冒充已复核、绕过那道门槛；只删早退等于回到 1.1.14 原行为，但那样恢复标记必须同步还原。

## 附录：证据

- 现场：1.1.4，190 次 spawn / 120 秒，84.5 秒在 `spawnSync`，栈为 `recoverVisibleRawBatch → readRawToolResult → tightenShadow → defaultShadowAccessRunner`。
- 上游修复：#38（恢复只读，1.1.5 的 `152402e`）、`37f15f6`（reauth 限 32）、`90bf6c4`（保存事务进 worker）、`853d95d`（`-Command`）。
- 审计来源：issue #14（P0-1）；后续 #25 / #28 / #38 / #67。
- 对照实现：`ranxianglei/billion-context`（无 DACL，AES-256-GCM 可选）、`T-Auto/dsh-ops`（单条 WRITE_OWNER，每工作区一次）。
- 代码位置（1.1.14 + 本 fork）：`shadow.js:606` `shadowAclSkipped`、`shadow.js:1318` 早退、`shadow.js:1211`（上游 1.1.14 为 1204，1.1.13 为 1165）`defaultShadowAccessRunner`、`index.js:3096` / `index.js:3142` 恢复标记、`index.js:3208` 32 条门槛、`raw.js:293` `runPruneAfterCapture`。
