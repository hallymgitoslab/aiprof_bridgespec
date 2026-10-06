# 视频学习进度验证规范 v0.1

**状态：** Draft  
**基准日期：** 2026-10-07  
**英文规范：** [PROGRESS_SPEC_EN.md](PROGRESS_SPEC_EN.md)

## 1. 目标

本规范为 Moodle、Rhymix、WordPress、GnuBoard5 定义统一的验证进度语义，使相同输入尽量得到相同结果。

本规范不统一各平台的存储 API，只统一结果含义。

## 2. 默认值

- heartbeat：5 秒
- seek grace/tolerance：3 秒
- completion threshold：80%
- 默认只为已登录用户保存个人进度

平台可以配置这些值，但测试必须明确使用的值。

## 3. 服务端权威

服务端不得把客户端 current_time 直接当作累计观看时间。

至少维护以下等价状态：

- watched_ranges
- watched_seconds
- frontier
- last_position
- last_heartbeat
- completed

## 4. Heartbeat 验证

如果没有历史记录，第一次请求只建立基准状态，不增加验证观看时间。

已有记录且 playing=true 时：

~~~text
max_forward = last_position + max(1, elapsed_seconds) + seek_grace
allowed_position = min(client_current_time, max_forward)
~~~

只有 allowed_position 大于 last_position 时才加入新的 watched range。

playing=false 时不得增加验证观看时间：

~~~text
allowed_position = min(client_current_time, frontier + seek_grace)
~~~

暂停请求可以把位置拉回已验证区域，但不新增 watched range。

## 5. 区间合并

所有区间必须 clamp 到 duration。错误区间以及 end <= start 的区间应丢弃。

排序后合并重叠或相邻区间。当前 baseline 使用：

~~~text
next.start <= current.end + 1
~~~

watched_seconds 为合并后各区间长度之和，因此重复观看不会重复累计。

## 6. Frontier

frontier 表示从 0 秒开始连续验证到的最远位置。

如果下一个区间的 start 大于 frontier + 1，则连续性中断。

客户端可以利用 frontier 立即恢复异常前跳，但最终以服务端结果为准。

## 7. 完成判定

~~~text
percentage = watched_seconds / duration * 100
completed = percentage >= completion_threshold
~~~

duration 为 0 或无效时不得判定完成。

## 8. Seek

客户端建议阻止超过 frontier + tolerance 的前跳并恢复到已验证位置。

但客户端限制不是安全边界。服务端必须假设存在开发者工具或直接 API 调用，并执行同样的前进量验证。

## 9. 播放事件

建议：

- play：同步后开始 heartbeat
- pause：停止 heartbeat 并同步
- ended：停止 heartbeat 并最终同步
- pagehide/beforeunload：支持时使用 keepalive 或尽力发送最终状态

即使客户端事件丢失，服务端也不得允许异常进度增长。

## 10. 媒体替换

同一 activity 的视频内容发生变化时，旧进度不得自动套用到新内容。必须有 reset 或明确 migration 策略。

Moodle Simple Video Tracker 参考实现会在内容变化时重置进度，仅 duration 变化时可执行重新计算。Bridge 建议保持同样语义。

## 11. 并发

同一用户、同一课程可能出现重叠 heartbeat。实现应使用 transaction、row lock、application lock 或等价手段防止 lost update。

还需要考虑媒体替换与 heartbeat 并发。

## 12. 公共测试

各 Bridge 测试应复用 [progress-v0.1.json](../test-vectors/progress-v0.1.json)。

各平台存储格式可以不同，但 watched_seconds、frontier、position、completed、percentage 的语义应一致。
