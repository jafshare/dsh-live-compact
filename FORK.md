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

## 怎么确认生效

```sh
grep -n shadowAclSkipped shadow.js index.js
```

装进插件目录后完全退出应用再启动，任务管理器里不该再出现 `powershell.exe` 子进程。
