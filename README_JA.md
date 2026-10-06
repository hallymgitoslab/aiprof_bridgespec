# AI Professor Lite Bridge Specification

AI Professor Lite のマルチプラットフォーム Bridge と、検証済み動画学習進捗の動作を一貫させるための共通仕様リポジトリです。

**Status:** v0.1 Draft  
**Updated:** 2026-10-07

言語：[한국어](README.md) | [English](README_EN.md) | [简体中文](README_ZH.md) | **日本語**

## 目的

AI Professor Lite は Moodle 向けの公開フローを基準に動作します。WordPress、Rhymix、GnuBoard などのプラットフォームごとに AI Professor Lite 本体へ分岐を追加するのではなく、各 Bridge が必要な Moodle-compatible Web Service のサブセットと Simple Video Tracker 契約を実装します。

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

## 仕様書

- [Bridge API 仕様](docs/BRIDGE_SPEC_JA.md)
- [動画進捗仕様](docs/PROGRESS_SPEC_JA.md)
- [共通テストベクター](test-vectors/progress-v0.1.json)

国際共同開発では英語版を規範文書とし、韓国語・中国語・日本語版は同期した翻訳として管理します。

## 現在の実装

| 実装 | 役割 | 確認済み基準 |
|---|---|---|
| simplevideotracker | Moodle native activity module | Moodle 基準実装 |
| aiprof-rx | Rhymix bridge | 0.2.2 / Rhymix 2.1.3+ / PHP 7.4+ |
| aiprof-wordpress | WordPress bridge | 0.1.1 |
| aiprof-gnuboard | GnuBoard5 bridge | 0.1.4 / GnuBoard5 5.4+ / PHP 7.4+ |

この表は 2026-10-07 時点のリポジトリ状態のスナップショットです。

## 設計原則

1. AI Professor Lite 本体のプラットフォーム別分岐を最小化します。
2. 公開処理に必要な Moodle API のサブセットだけを実装します。
3. 各 Bridge は各 CMS のユーザー、投稿、ファイル、DB 機構を利用します。
4. 進捗判定はサーバー権威型とします。
5. 再生済み区間の重複カウントを行いません。
6. 過度な前方シークはサーバー側で制限します。
7. 共通テストベクターにより全 Bridge の結果を揃えます。

## 変更方針

共通契約を変更する場合は AI Professor Lite Moodle client、Moodle Simple Video Tracker、Rhymix/WordPress/GnuBoard Bridge、4言語文書、共通テストベクターをまとめて確認します。

互換性を壊す変更は新しい spec version として分離します。
