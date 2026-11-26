# Fork 说明：Windows 上不改写备份 ACL

本仓库是 `Tranquillitatis-Lab/dsh-live-compact` 的分支，唯一的代码差异是：**Windows 上不再重建备份目录/文件的 DACL，也不再为此启动 PowerShell / whoami。**

## 改了什么

- `shadow.js` 新增 `shadowAclSkipped(access)`；`tightenShadow` 在熔断检查之后直接返回。
- `index.js` 恢复记录时，Windows 上直接把 `aclVerified` 记为已复核，修剪和切片不再等一次不会发生的权限复核。
- 判定条件同时要求真实 Windows、未注入 `runner`、未注入 `execFile`，所以 POSIX 行为和上游一致，注入式测试仍走原路径。

## 没改什么

- 备份内容、哈希/字节校验、文件身份校验、切分策略、卡片格式、worker 协议都不变。
- POSIX 的目录 0700 / 文件 0600 及读回校验不变。
- 上游的 DACL 脚本仍留在包里，不再执行。

## 安全边界

备份的可见性由用户目录的系统 ACL 决定：`%USERPROFILE%` 默认只允许当前用户、SYSTEM、Administrators，`DSH_HOME` 在默认位置时继承同一组权限。

共享目录、网络盘，以及把 `DSH_HOME` / `SPINE_SHADOW_DIR` 指到其他位置的情况，本 fork 不提供隔离保证。需要隔离时请自己保证目录权限。

上游 README 的「Windows 备份权限」一节描述的是上游行为，对本 fork 不适用。

## 升级上游

```sh
git fetch upstream
git merge upstream/main
```

冲突面只有 `tightenShadow` 顶部那几行和恢复处的两行；上游若重构，把 `if (shadowAclSkipped(access)) return;` 放回新入口即可。

合并后跑 `npm run test:gate`。上游改动测试注入方式（不再用 `access.execFile`）时，`shadowAclSkipped` 的条件要跟着改，否则注入式用例会走新路径。

## 安装

桌面端在「设置 → 插件」里填写依赖写法。**不要贴 SSH 克隆地址**（`git@github.com:...`，或去掉 `git@` 的 `github.com:...`）——pnpm 会把 `github.com:` 当成协议名，直接报 `ERR_PNPM_UNSUPPORTED_PROTOCOL`。

| 写法 | 说明 |
|---|---|
| `github:jafshare/dsh-live-compact` | 从 GitHub 装，取的是已 push 的 `main`。改完要 push 再重装 |
| `git+https://github.com/jafshare/dsh-live-compact.git` | 同上，走 HTTPS |
| `file:D:/Workspace/AI/dsh-live-compact` | 从本地工作区装。拷一份进 profile，改完要重装 |
| `link:D:/Workspace/AI/dsh-live-compact` | 软链到本地工作区，改完重启即生效；但仓库没有自己的 `node_modules`，`@deepseek-ai/*` 要靠 profile 的父目录解析，用之前先确认能 import 到宿主包 |

装到 `%USERPROFILE%\.dsh\profiles\desktop\node_modules\dsh-live-compact`，装完**完全退出应用再启动**。

合并上游之后：GitHub 装法要 push 再装一次；`file:` 装法要重装一次；`link:` 装法直接重启。

## 怎么确认生效

版本号仍是 1.1.14，插件页显示的号码和上游一样，看不出装的是哪份，所以查文件：

```sh
grep -n shadowAclSkipped "%USERPROFILE%/.dsh/profiles/desktop/node_modules/dsh-live-compact/shadow.js"
```

有输出就是这份 fork。启动后任务管理器里 `node.exe` 下不该再出现 `powershell.exe` 子进程。
