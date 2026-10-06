# AI Professor Lite Bridge Specification

This repository defines the common contract used to keep AI Professor Lite platform bridges and verified video-learning progress behavior consistent.

**Status:** v0.1 Draft  
**Updated:** 2026-10-07

Languages: [한국어](README.md) | **English** | [简体中文](README_ZH.md) | [日本語](README_JA.md)

## Purpose

AI Professor Lite uses a Moodle-oriented publishing flow. Other platforms such as WordPress, Rhymix, and GnuBoard should not require platform-specific branches in the AI Professor Lite core. Instead, each bridge implements the Moodle-compatible Web Service subset and Simple Video Tracker contract required by the publisher.

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

## Specifications

- [Bridge API Specification](docs/BRIDGE_SPEC_EN.md)
- [Video Progress Specification](docs/PROGRESS_SPEC_EN.md)
- [Shared Progress Test Vectors](test-vectors/progress-v0.1.json)

The English specification is the normative reference for international collaboration. Korean, Chinese, and Japanese files should remain synchronized translations.

## Current Implementations

| Implementation | Role | Confirmed baseline |
|---|---|---|
| simplevideotracker | Moodle native activity module | Reference Moodle implementation |
| aiprof-rx | Rhymix bridge | 0.2.2 / Rhymix 2.1.3+ / PHP 7.4+ |
| aiprof-wordpress | WordPress bridge | 0.1.1 |
| aiprof-gnuboard | GnuBoard5 bridge | 0.1.4 / GnuBoard5 5.4+ / PHP 7.4+ |

This matrix is a snapshot of the repositories as of 2026-10-07.

## Design Principles

1. Minimize platform-specific branching in AI Professor Lite.
2. Implement only the Moodle API subset required by the publisher.
3. Let each bridge use its platform-native users, posts, files, and database APIs.
4. Treat verified progress as server-authoritative.
5. Do not double-count replayed ranges.
6. Limit excessive forward seeking on the server.
7. Use shared test vectors so all bridges produce equivalent results for equivalent inputs.

## Change Policy

A common-contract change should review all of the following together:

- AI Professor Lite Moodle client
- Moodle Simple Video Tracker
- Rhymix bridge
- WordPress bridge
- GnuBoard5 bridge
- all four language specifications
- shared test vectors

Breaking changes require a new specification version.
