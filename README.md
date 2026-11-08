# lead-hunter-timer

每天 **北京时间 08:00** 唤醒私有仓库 `overseas-lead-hunter` 的日报发送任务。

## 为什么单独建这个公开仓库

Actions 分钟数免费且不限量 —— 本 workflow 会提前几小时启动并 `sleep`
最长近 5 小时，放在私有仓库会烧掉免费额度。公开仓库也**不含任何客户数据**，
只有一个触发动作、一个加密 Secret 和一个保活文件。

> 2026-11-08 更正：早期文档写「公开仓库 schedule 优先级更高」是错的。
> 11-07 实测本仓库配在 UTC 23:58~00:15 的 5 个点全部被排到
> UTC 04:28~05:53（北京 12:28~13:53）；11-08 上午更是 0 次 run。
> 现在不靠 `schedule` 准点，改用「提前唤醒 + sleep 到整点」。

## 工作原理

`.github/workflows/trigger.yml` 在前一天的低峰时段提前唤醒若干 job，
每个 job 计算「距离下一个 UTC 00:00（北京 08:00）还有多久」，
分段 sleep 到点后调用私有仓库的 `workflow_dispatch`：

```
POST /repos/lasby66available/overseas-lead-hunter/actions/workflows/daily.yml/dispatches
{"ref":"main","inputs":{}}
```

### 时间点（UTC）

| cron | UTC | 北京时间 | 作用 |
|---|---|---|---|
| `7 19 * * *` | 19:07 | 03:07 | 提前唤醒，睡 4h53m，最能扛住小时级延迟 |
| `13 20 * * *` | 20:13 | 04:13 | 提前唤醒，睡 3h47m |
| `19 22 * * *` | 22:19 | 06:19 | 提前唤醒，睡 1h41m |
| `37 23 * * *` | 23:37 | 07:37 | 提前唤醒，睡 23m |
| `52 23 * * *` | 23:52 | 07:52 | 提前唤醒，睡 8m |
| `3 0 * * *` | 00:03 | 08:03 | 迟到兜底，启动即补发 |
| `15 0 * * *` | 00:15 | 08:15 | 迟到兜底 |
| `40 1 * * *` | 01:40 | 09:40 | 最后一档兜底 |

全部避开 `:00`。UTC 小时 >= 18 时目标是「明天 00:00」；
已经过了 00:00 说明调度迟到，立即补发（绝不睡到明天，会撞 6 小时 job 上限）。
等待超过 5h50m 时本点直接放弃，不会退化成凌晨提前发信。

这样即使 GitHub 把 19:07 那个点晚排 4.5 小时，它也会在 23:37 启动、
睡 23 分钟，仍然在 **08:00:00** 按下按钮 —— 调度延迟被 sleep 吃掉了。

私有仓库 `daily.yml` 内有「今天已送达就 20 秒退出」的幂等闸门和
`concurrency` 组，因此多个点同时 dispatch 也**不会重复发日报**。

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

在 Actions 页对 `08:00 Daily Trigger（提前唤醒）` 点 `Run workflow`，或调用：

```
POST /repos/lasby66available/lead-hunter-timer/actions/workflows/trigger.yml/dispatches
{"ref":"main"}
```

手动触发会跳过等待步骤，立即 dispatch。成功标志：日志出现 `dispatch HTTP 204`，
且私有仓库 Actions 新增一条 `workflow_dispatch` 运行记录。
若当天日报已发过，私有仓库的幂等闸门会让发送步骤全部 skipped（这是正常的）。

> 本地备份与维护说明见 `overseas-lead-hunter/tools/timer-workflow.yml`，
> 完整方案（含 2026-11-08 事故复盘）见
> `overseas-lead-hunter/docs/punctual-setup.md`。
