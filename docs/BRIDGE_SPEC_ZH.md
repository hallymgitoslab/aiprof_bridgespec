# AI Professor Lite Bridge API 规范 v0.1

**状态：** Draft  
**基准日期：** 2026-10-07  
**英文规范：** [BRIDGE_SPEC_EN.md](BRIDGE_SPEC_EN.md)

## 1. 范围

本规范定义 AI Professor Lite 在 Moodle 与非 Moodle 平台上复用同一发布流程时，Bridge 必须提供的最小兼容契约。

Bridge 不需要实现完整 Moodle，只实现当前 AI Professor Lite 发布流程所需的 endpoint 与 Web Service function。

## 2. Base URL 与 endpoint

AI Professor Lite 配置 Bridge Base URL 和 API token。

Base URL 下必须提供：

~~~text
/webservice/rest/server.php
/webservice/upload.php
~~~

REST endpoint 必须支持 POST。GET 可以返回错误 JSON，但建议不要返回 404，便于管理员检查 endpoint 是否存在。

## 3. 认证

REST 调用：

~~~text
wstoken=<token>
wsfunction=<function>
moodlewsrestformat=json
~~~

上传调用：

~~~text
token=<token>
file_1=<multipart file>
~~~

建议使用 timing-safe token 比较。不得在日志、HTML 或错误消息中泄露 token。

## 4. 必需 function

Bridge 至少提供：

- core_webservice_get_site_info
- core_enrol_get_users_courses
- core_course_get_contents
- mod_simplevideotracker_create_activity
- mod_simplevideotracker_set_video

为兼容旧版 VideoTracker，可以另外提供：

- mod_videotracker_create_activity
- mod_videotracker_set_video

AI Professor Lite 可以根据 site info 返回的 function 列表选择 videotracker 或 simplevideotracker component。

## 5. create_activity

| 字段 | 必需 | 含义 | 默认值 |
|---|---:|---|---:|
| courseid | Y | 映射为 course 的目标 | - |
| sectionnum | Y | section 或逻辑发布位置 | - |
| name | Y | 课程标题 | - |
| intro | N | 课程介绍 HTML/text | 空 |
| completionpercent | N | 完成阈值 | 80 |
| preventseeking | N | 阻止跳到未验证区域 | true |
| seektolerance | N | seek 容差秒数 | 3 |
| heartbeat | N | heartbeat 秒数 | 5 |

当前 AI Professor Lite 发布器发送 courseid、sectionnum、name、intro，其余字段可由 Bridge 使用默认值。

成功响应至少应包含：

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

cmid 必须是可以稳定重新定位该课程 activity 的标识符。

## 6. 文件上传

/webservice/upload.php 接收 multipart 文件并返回 JSON 数组。AI Professor Lite 从第一项读取 itemid 或 draftitemid。

~~~json
[
  {
    "itemid": 456,
    "filename": "lecture.mp4"
  }
]
~~~

临时 draft ID 应与当前 token 绑定，并具有过期与清理策略。

## 7. set_video

| 字段 | 必需 | 含义 |
|---|---:|---|
| cmid | Y | activity 标识 |
| draftitemid | Y | upload 返回的临时文件 ID |
| duration | N | 可信的视频时长秒数 |

duration 提供时必须验证为有限正数。

成功响应建议至少包含：

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

不同平台可增加 filename、URL、duration、changed 等元数据。

## 8. 平台映射

| 公共概念 | Moodle | Rhymix | WordPress | GnuBoard5 |
|---|---|---|---|---|
| Course | course | 指定 board/module | 逻辑发布目标 | bo_table |
| Activity/cmid | course module id | document_srl 或 bridge id | post_id 或 bridge id | wr_id 或 bridge id |
| User | user.id | member_srl | WP user ID | mb_id |
| Video | Moodle file area | Rhymix attachment | Media Library | board file |
| Progress | plugin table | bridge progress table | bridge progress table | bridge progress table |

内部 ID 可以不同，但外部 contract 必须稳定。

## 9. 错误

建议使用 Moodle 风格 JSON：

~~~json
{
  "exception": "moodle_exception",
  "errorcode": "invalidparameter",
  "message": "..."
}
~~~

日志应能够区分认证、参数验证与内部错误。

## 10. Idempotency

create_activity 存在重复创建风险。条件允许时 Bridge 应支持 external_id 或等价 idempotency 机制。

当创建结果不确定时，AI Professor Lite 应优先要求人工确认，而不是盲目自动重建。

set_video 的重复调用不应破坏数据。

## 11. 安全

建议 HTTPS、长随机 token、文件扩展名与 MIME 验证、临时文件清理、HTML sanitization 与服务端进度验证。必须阻止路径穿越，并禁止记录 secret。
