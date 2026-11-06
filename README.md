# lead-hunter-timer

每天 **北京时间 08:00** 唤醒私有仓库 `overseas-lead-hunter` 的日报发送任务。

## 为什么单独建这个公开仓库

GitHub Actions 的 `schedule` 对私有仓库会明显延迟（实测配置 UTC 00:00，
实际被排到 UTC 05:19~07:55，晚 5~8 小时）。公开仓库的定时任务优先级更高。
本仓库**不含任何客户数据**，只有一个触发动作和一个加密 Secret。

## 工作原理

`.github/workflows/trigger.yml` 在下列时间点（UTC）运行，向私有仓库发送
`workflow_dispatch`：

| UTC | 北京时间 | 作用 |
|---|---|---|
| 23:58 | 07:58 | 预启动，sleep 到 08:00 再触发，避开整点拥堵 |
| 00:00 | 08:00 | 整点 |
| 00:03 | 08:03 | 兜底 |
| 00:08 | 08:08 | 兜底 |
| 00:15 | 08:15 | 兜底 |

私有仓库 `daily.yml` 内有「今天已送达就 20 秒退出」的幂等闸门，
因此多个时间点都跑起来也**不会重复发日报**。

## Secret

`DISPATCH_TOKEN`：一个带 `repo, workflow` 权限的 GitHub token，
以 `PyNaCl SealedBox` 加密后存放，只在 workflow 运行时注入环境变量，
不出现在日志和代码里。

## 保活（防止 60 天后自动停摆）

GitHub 会在仓库连续 60 天无活动后自动禁用 `schedule`。本 workflow 最后一步
每天往 `LAST_RUN.txt` 写一行日期（一天最多一个 commit，`continue-on-error`），
保证仓库始终有活动，无需人工维护。文件日期停更即代表保活失败，需要手动 push 一次。

触发动作排在 checkout 与保活之前，准点优先；保活失败不影响当天日报。

## 手动测试

在 Actions 页对 `08:00 Daily Trigger` 点 `Run workflow`，
或调用：

```
POST /repos/lasby66available/lead-hunter-timer/actions/workflows/trigger.yml/dispatches
{"ref":"main"}
```

成功标志：日志里出现 `dispatch HTTP 204`，且私有仓库 Actions 新增一条
`workflow_dispatch` 运行记录。

> 本地备份与维护说明见 `overseas-lead-hunter/tools/timer-workflow.yml`，
> 完整方案见 `overseas-lead-hunter/docs/punctual-setup.md`。
