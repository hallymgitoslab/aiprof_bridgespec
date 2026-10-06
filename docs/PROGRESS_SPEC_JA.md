# 動画学習進捗仕様 v0.1

**状態:** Draft  
**基準日:** 2026-10-07  
**英語版:** [PROGRESS_SPEC_EN.md](PROGRESS_SPEC_EN.md)

## 1. 目的

Moodle、Rhymix、WordPress、GnuBoard5 で同等の入力から同等の検証済み視聴進捗を得るための共通 baseline を定義します。

各プラットフォームの保存 API は統一せず、結果の意味を統一します。

## 2. 既定値

- heartbeat: 5秒
- seek grace/tolerance: 3秒
- completion threshold: 80%
- 個人進捗は原則としてログインユーザーのみ保存

設定で変更可能ですが、テストでは使用値を明示します。

## 3. サーバー権威

client current_time をそのまま累積視聴時間として信用してはいけません。

最低限、次と同等の状態を保持します。

- watched_ranges
- watched_seconds
- frontier
- last_position
- last_heartbeat
- completed

## 4. Heartbeat 検証

既存 progress がない場合、最初の要求は基準状態だけを作り、検証済み視聴時間を増加させない動作を baseline とします。

既存 record があり playing=true の場合:

~~~text
max_forward = last_position + max(1, elapsed_seconds) + seek_grace
allowed_position = min(client_current_time, max_forward)
~~~

allowed_position > last_position の場合のみ watched range を追加します。

playing=false の paused heartbeat は進捗を増加させてはいけません。

~~~text
allowed_position = min(client_current_time, frontier + seek_grace)
~~~

pause 時は保存位置を検証済み領域へ戻せますが、新規 range は追加しません。

## 5. Range merge

range は duration 内に clamp します。不正な range と end <= start は破棄します。

sort 後、重複または隣接 range を merge します。現 baseline は次を使います。

~~~text
next.start <= current.end + 1
~~~

watched_seconds は merge 後の range 長の合計なので、再生済み区間を再生しても重複加算しません。

## 6. Frontier

frontier は動画 0 秒から連続して検証された最遠位置です。

次の range の start が frontier + 1 を超えると連続性が切れます。

client は frontier を使って過度な前方 seek を戻せますが、最終的な権威は server response です。

## 7. 完了判定

~~~text
percentage = watched_seconds / duration * 100
completed = percentage >= completion_threshold
~~~

duration が 0 または無効な場合は完了にしません。

## 8. Seek

client は frontier + tolerance を超える前方 seek を検出した場合、検証済み位置へ戻すことを推奨します。

ただし client 制御は security boundary ではありません。developer tools や直接 API 呼び出しを想定し、server 側でも前進量を制限します。

## 9. 再生イベント

推奨動作:

- play: 状態同期後 heartbeat 開始
- pause: heartbeat 停止後同期
- ended: heartbeat 停止後最終同期
- pagehide/beforeunload: 利用可能なら keepalive または best-effort 最終送信

client event が失われても server が異常な進捗増加を許可しないことが重要です。

## 10. メディア差し替え

同じ activity の動画内容が変更された場合、旧進捗を無関係な新動画へ引き継いではいけません。reset または明確な migration 方針が必要です。

Moodle Simple Video Tracker 基準実装は動画 content が変わると進捗を reset し、duration だけが変わった場合は再計算できます。Bridge も同じ意味を維持することを推奨します。

## 11. 同時実行

同一ユーザー・同一講義に複数 heartbeat が重なる可能性があります。transaction、row lock、application lock 等で lost update を防ぐことを推奨します。

動画差し替えと heartbeat の同時実行も考慮します。

## 12. 共通テスト

各 Bridge の test suite で [progress-v0.1.json](../test-vectors/progress-v0.1.json) を再利用します。

保存形式は異なっても watched_seconds、frontier、position、completed、percentage の意味は一致させます。
