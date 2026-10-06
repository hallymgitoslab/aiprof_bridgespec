# AI Professor Lite Bridge Specification

本仓库用于定义 AI Professor Lite 多平台桥接器与视频学习进度验证的一致性规范。

**状态：** v0.1 Draft  
**更新日期：** 2026-10-07

语言：[한국어](README.md) | [English](README_EN.md) | **简体中文** | [日本語](README_JA.md)

## 目的

AI Professor Lite 以 Moodle 发布流程为基础。对于 WordPress、Rhymix、GnuBoard 等平台，不建议在 AI Professor Lite 核心中分别加入平台分支，而是由各平台桥接器实现发布器所需的 Moodle-compatible Web Service 子集与 Simple Video Tracker 契约。

~~~text
AI Professor Lite
        |
        | Moodle-compatible contract
        v
+-------------------------------+
| Platform Bridge               |
| - Moodle                      |
| - WordPress                   |
| - Rhymix                      |
| - GnuBoard5                   |
| - future adapters             |
+-------------------------------+
        |
        v
Native users / posts / files / progress
~~~

## 规范文档

- [Bridge API 规范](docs/BRIDGE_SPEC_ZH.md)
- [视频进度规范](docs/PROGRESS_SPEC_ZH.md)
- [共享测试向量](test-vectors/progress-v0.1.json)

国际协作时以英文规范为基准。韩文、中文、日文文档应作为同步翻译版本维护。

## 当前实现

| 实现 | 角色 | 当前确认基线 |
|---|---|---|
| simplevideotracker | Moodle 原生活动模块 | Moodle 参考实现 |
| aiprof-rx | Rhymix bridge | 0.2.2 / Rhymix 2.1.3+ / PHP 7.4+ |
| aiprof-wordpress | WordPress bridge | 0.1.1 |
| aiprof-gnuboard | GnuBoard5 bridge | 0.1.4 / GnuBoard5 5.4+ / PHP 7.4+ |

本表为 2026-10-07 的仓库状态快照。

## 设计原则

1. 尽量减少 AI Professor Lite 中的平台专用分支。
2. 仅实现发布流程实际需要的 Moodle API 子集。
3. 各 bridge 使用平台原生的用户、文章、文件与数据库体系。
4. 学习进度以服务端验证结果为准。
5. 重复播放区间不得重复累计。
6. 服务端必须限制异常的向前跳播。
7. 所有 bridge 使用相同测试向量验证一致行为。

## 变更策略

修改公共契约时，应同时检查 AI Professor Lite Moodle client、Moodle Simple Video Tracker、Rhymix/WordPress/GnuBoard bridge、四种语言文档以及共享测试向量。

破坏兼容性的修改应发布新的 spec version。
