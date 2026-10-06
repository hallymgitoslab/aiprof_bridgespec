# AI Professor Lite Bridge API 仕様 v0.1

**状態:** Draft  
**基準日:** 2026-10-07  
**英語版:** [BRIDGE_SPEC_EN.md](BRIDGE_SPEC_EN.md)

## 1. 範囲

本仕様は、AI Professor Lite が Moodle と非 Moodle プラットフォームで同じ公開フローを利用するために Bridge が提供する最小互換契約を定義します。

Bridge は Moodle 全体を実装せず、現在の AI Professor Lite 公開フローに必要な endpoint と Web Service function のみを実装します。

## 2. Base URL と endpoint

AI Professor Lite には Bridge Base URL と API token を設定します。

Base URL 配下に次の相対パスが必要です。

~~~text
/webservice/rest/server.php
/webservice/upload.php
~~~

REST endpoint は POST を受け付ける必要があります。GET はエラー JSON でも構いませんが、存在確認のため 404 以外の応答を推奨します。

## 3. 認証

REST:

~~~text
wstoken=<token>
wsfunction=<function>
moodlewsrestformat=json
~~~

Upload:

~~~text
token=<token>
file_1=<multipart file>
~~~

token 比較は timing-safe を推奨し、ログ、HTML、エラーへ token を出力してはいけません。

## 4. 必須 function

最低限、次を提供します。

- core_webservice_get_site_info
- core_enrol_get_users_courses
- core_course_get_contents
- mod_simplevideotracker_create_activity
- mod_simplevideotracker_set_video

旧 VideoTracker 互換として次の alias を追加しても構いません。

- mod_videotracker_create_activity
- mod_videotracker_set_video

AI Professor Lite は site info の function 一覧から videotracker または simplevideotracker component を選択できます。

## 5. create_activity

| フィールド | 必須 | 意味 | 既定値 |
|---|---:|---|---:|
| courseid | Y | course としてマッピングする対象 | - |
| sectionnum | Y | section または論理公開位置 | - |
| name | Y | 講義タイトル | - |
| intro | N | 紹介 HTML/text | 空 |
| completionpercent | N | 完了基準 | 80 |
| preventseeking | N | 未検証位置への前方 seek 制限 | true |
| seektolerance | N | seek 許容秒数 | 3 |
| heartbeat | N | heartbeat 秒数 | 5 |

現在の AI Professor Lite publisher は courseid、sectionnum、name、intro を送信します。残りは Bridge 側の既定値を利用できます。

成功応答には最低限次を含めます。

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

cmid は後から同じ activity を解決できる安定した識別子でなければなりません。

## 6. ファイルアップロード

/webservice/upload.php は multipart upload を受け取り JSON 配列を返します。AI Professor Lite は先頭要素の itemid または draftitemid を使用します。

~~~json
[
  {
    "itemid": 456,
    "filename": "lecture.mp4"
  }
]
~~~

一時 draft ID は token と関連付け、期限切れ・削除ポリシーを持つことを推奨します。

## 7. set_video

| フィールド | 必須 | 意味 |
|---|---:|---|
| cmid | Y | 対象 activity |
| draftitemid | Y | upload が返した一時ファイル ID |
| duration | N | 信頼できる動画長(秒) |

duration がある場合は有限の正数として検証します。

成功応答は最低限次を推奨します。

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

実装は filename、URL、duration、changed 等のメタデータを追加できます。

## 8. プラットフォームマッピング

| 共通概念 | Moodle | Rhymix | WordPress | GnuBoard5 |
|---|---|---|---|---|
| Course | course | 指定 board/module | 論理公開先 | bo_table |
| Activity/cmid | course module id | document_srl または bridge id | post_id または bridge id | wr_id または bridge id |
| User | user.id | member_srl | WP user ID | mb_id |
| Video | Moodle file area | Rhymix attachment | Media Library | board file |
| Progress | plugin table | bridge progress table | bridge progress table | bridge progress table |

内部 ID 形式は異なっても、外部 contract は安定させます。

## 9. エラー

Moodle 形式の JSON を推奨します。

~~~json
{
  "exception": "moodle_exception",
  "errorcode": "invalidparameter",
  "message": "..."
}
~~~

運用ログでは認証、入力検証、内部エラーを区別できるようにします。

## 10. Idempotency

create_activity は重複作成の危険があります。可能であれば external_id 等の idempotency 手段を提供します。

作成結果が不明な場合、AI Professor Lite は無条件の再作成より運用者による確認を優先します。

set_video の同一操作を繰り返しても状態を壊さない実装を推奨します。

## 11. セキュリティ

HTTPS、長い乱数 token、拡張子/MIME 検証、一時ファイル削除、HTML sanitization、サーバー側進捗検証を推奨します。path traversal と secret のログ出力は禁止します。
