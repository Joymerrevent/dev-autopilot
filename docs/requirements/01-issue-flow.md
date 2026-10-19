# Issue の流れと運用（棚卸し・並列・間隔・状態の遷移・運用の規則）

## 棚卸し（進行 Bot）

- FLOW-1 設定した Project（STRUCT-2）の Issue を読み、次の条件をすべて満たすものだけを候補にする
  - 人が付けたラベル `agent-ready` がある
  - マイルストーンがある。複数あれば、版の小さいマイルストーンを先にする
  - Project の Status が Todo
  - Issue 依存（blocked by）の相手がすべて閉じている
  - `needs-human` ラベルが無い（人が外すまで再着手しない）。Project の Agent 欄が `human`（人が引き取った）でも `paused`（人が一時停止した）でもない
  - 受け入れの下書きコメント（進行 Bot の投稿、`intake` の印。「[印と固定の見出しの一覧][f04]」）があり、人が `agent-ready` を付けている（TEXT-12）。渡す文は、作業指示コメントがあれば作業指示コメント、無ければ承認された下書きで、Issue の本文は著者を問わず渡さない（TEXT-1、[ADR-20261016-002][adr-draft]）。ただし `quarantine=found` の Issue では、人の作業指示コメントだけを渡す（TEXT-12）
  - `agent-ready` を付けた人が push 以上の権限を持つ（Issue の timeline で確かめる。TEXT-10）
- FLOW-2 候補を着手前の予測で待たせない。待たせるのは人が入れた依存だけ。触る範囲の重なりは着手後に実際の変更ファイルで見て、警告のコメントに留める（FLOW-8）
- FLOW-3 1 回の実行で着手する件数に上限を持つ（最初は 1 件）。進行中の Issue（`In Progress` 以降で `Done` でない）の数にも上限を持つ（WIP の上限。マージ判定工程（段階 8）までは 1）。上限に達していれば新しい Issue に着手しない
- FLOW-4 **1 回の実行で進める範囲**。進行 Bot は、AI のセッションを起動する工程（重い工程：受け入れ、計画、実装と PR の作成、レビュー（批評を含む）、修正）を、Issue ごとに 1 回の実行で 1 つだけ進める。
  AI のセッションを起動しない工程（軽い工程：承認の確定、収束、マージの判定、載せ直し、催促、片付け）は、重い工程の前後に同じ実行で続けてよい。CI の結果が要る工程（収束、マージの判定）は CI の結果が出ている実行でだけ進め、push した実行では push した Issue を次の実行まで進めない（[ADR-20261015-006][adr-flow]）。
  進行中の Issue は、毎回の実行で GitHub の状態（Status、PR の有無と draft、最新の印の種類と `head`、必須チェックの結果）から「次の工程」を決定的に決めて再開する。手元の記録は判定に使わない（上限の数え上げにだけ使う。「運用の規則」）。
  CI は実行の中で待たず、次回の実行で結果を読む（IMPL-27）。1 回の実行の所要時間の上限を設定に持ち、超えたら残りの Issue は次回に回す（lock を長く持たない）。
  1 日 1 回の間隔では 1 Issue に 1 週間ほどかかる。間隔を狭める判断は FLOW-12
- FLOW-5 実行の記録（選んだ Issue、待たせた Issue と理由、使った時間）を追跡しないフォルダに残す。
  AI のセッション 1 回ごとの記録は欄を固定し、後から集計できる形にする：実行 ID、Issue 番号、ラウンド、役（受け入れ・計画・作業・レビュー・批評）、指定したモデルと effort、実際に使われたモデル（出力（IMPL-14 の `stream-json`）の `modelUsage` から取る。安全側のフォールバックで別のモデルに回ったことを検知する）、費用、ターン数、実行の成否、作業の成否。
  IMPL-14 の `usage` と FLOW-13 の指標はこの欄から取る（Langfuse のような観測の道具が持つ「セッションごとのトレースに費用と成否を結びつける」形を、道具は入れずに欄として写す）

## 並列と競合

- FLOW-6 Issue ごとに作業ブランチを分け、作業場所は AI のセッションごとに作る（[ADR-20261013-001][adr-workspace]）ので、同時に複数の Issue を進められる
- FLOW-7 競合の正は人が入れる Issue 依存（blocked by）。進行 Bot はそれに従う
- FLOW-8 競合の予測は AI にさせない。作業 AI の commit が出た時点で、進行 Bot がその PR の変更ファイルを他の進行中の dev-autopilot の PR の変更ファイルと機械で比べ、重なりがあれば「待つ」ではなく Issue に「警告のコメント」を 1 回書く（FLOW-2。1 回かどうかは `note` の印（`kind=overlap`）で判定する）
- FLOW-9 先行の PR がマージされたら、依存していた PR を `develop` に載せ直してから続ける
- FLOW-10 依存で待つ Issue は Status・Agent 欄・ラベルを変えない（`Blocked` にしない。`Blocked` は人が引き取るまで触らない意味で使う）。GitHub の Issue 依存（blocked by）の表示が印になる。
  進行 Bot は待たせた理由を記録に残し、Issue には 1 回だけコメントする（同じ理由で毎回コメントしない。1 回かどうかは `note` の印（`kind=wait`）で判定する）。先行が閉じたら次回の実行で自動で続ける。
  作業中（`In Progress` 以降）に人が依存を足したときも同じで、その Issue のレビューと対応ループを止めて待ち、作業ブランチと PR は残す。先行がマージされたら FLOW-9 のとおり載せ直してから続ける

## 間隔の運用

- FLOW-11 間隔と件数の上限はリポの設定 1 つで変える（cron の設定と、進行 Bot の上限の両方。STRUCT-2）
- FLOW-12 狭める条件：連続 7 回の実行で `needs-human` が 0 件、利用枠の消費が上限内。1 段ずつ狭める（1 日 1 回 → 12 時間 → 6 時間）
- FLOW-13 測る指標を記録に残す：マージ率（着手した Issue のうちマージに届いた割合）、往復の回数、人が却下した指摘の割合（誤検知率）、**マージ後の revert 率**（Codex 6.1%・Devin 14.5%・人 11.5% という調査がある）、1 Issue あたりの利用枠。
  revert 率は「マージした後に壊れていた」を測る唯一の値なので、進行 Bot のマージの判定を信頼してよいか（マージ判定工程（段階 8）の出口と、将来マージの実行を進行 Bot に移すか）の判断に使う
- FLOW-14 **指標の記録は GitHub に置く**。却下：人が PR に「`## 却下 R<n>`」の見出しでコメントし、却下する指摘の見出しを列挙する。進行 Bot は、push 以上の権限を持つ人が書き、進行 Bot の印を持たないこの見出しのコメントだけを数える（AI の出力に同じ見出しがあっても数えない。IMPL-20。検疫（TEXT-11）を通し、AI には渡さない）。
  一致：MERGE-3 の数え方。手元の記録は集計の写しであって正ではない

## Issue の状態の遷移

Issue の状態は 3 つの欄で表し、正は GitHub に置く（「運用の規則」）。進行 Bot は毎回の実行でこの 3 つから状態を組み立て直す。

- Project の Status：`Todo` → `In Progress` → `In Review` → `Ready to Merge` → `Done`。止まったときは `Blocked`（SETUP-5）
- Project の Agent 欄：空 / `worker` / `reviewer` / `human` / `paused`。「いま動いている役、または次に動く役」を表す（SETUP-6、IMPL-7）
- ラベル：`agent-ready`（人が処理対象にした）、`needs-human`（人が引き取るまで進行 Bot は触らない）、`from-review`（🟢 を起票した Issue）（SETUP-8）、`plan:fable` / `plan:opus`（人が計画 AI のモデルを上書きする。無ければ受け入れの印に従う。IMPL-10）

表の読み方：Status・Agent 欄・ラベルの列は遷移の後の値。「変えない」はその遷移で触らない欄。「順」は正常な順の並び。「番号」は要件の番号で、ほかの文書は番号で遷移を指す（振り直さない。[README][readme] の「要件を変えるとき」）。番号は正常な順が FLOW-15〜FLOW-27、止まる遷移が FLOW-28〜FLOW-34。

正常な順（上から下へ）：

| 順 | 番号 | 契機 | 変える人 | Status | Agent 欄 | ラベル | 根拠 |
|---|---|---|---|---|---|---|---|
| 1 | FLOW-15 | Issue が起票され、Project に入る | 自動（Project の自動追加。SETUP-1） | `Todo` | 空 | 無し | SETUP-1 |
| 2 | FLOW-16 | 受け入れ：検疫、作業指示の下書き、依存とマイルストーンの候補を「受け入れの下書き」コメントに書く（検疫で見つかったら、検疫の結果と印だけを書き、FLOW-29 の遷移で `needs-human` にする。TEXT-12） | 進行 Bot | 変えない | 変えない | 変えない | TEXT-12 |
| 3 | FLOW-17 | 下書きを読み（必要なら直し）、処理対象にすると判断する | 人（コラボレータ） | 変えない | 変えない | `agent-ready` | TEXT-9、TEXT-12 |
| 4 | FLOW-18 | `agent-ready` を検出し、渡す文のハッシュを取って `intake-final` の印を新しいコメントで投稿し（TEXT-12）、依存（blocked by）とマイルストーンを候補（人が直した値があればそれ）で確定する | 進行 Bot | 変えない | 変えない | 変えない | TEXT-12、FLOW-1、SETUP-7 |
| 5 | FLOW-19 | 棚卸しで候補に選び、基線が緑なら着手する | 進行 Bot | `In Progress` | `worker` | 変えない | FLOW-1、IMPL-9 |
| 6 | FLOW-20 | 計画 AI（印とラベルで決めたモデル）が計画を返し、進行 Bot が `plan` の印付きで Issue にコメントする。計画 AI の `opus` が自己申告したら、同じ回の実行で続けて `fable` で計画し直し、やり直した計画だけを投稿する（IMPL-10。[ADR-20261018-002][adr-redo]）。計画を投稿した実行では、どの段階でも実装に入らない（重い工程は 1 回の実行で 1 つ。FLOW-4）。計画を投稿する前に、計画が変える見込みのパスを push しないパスの一覧と照らし、当たれば計画を投稿せずに `needs-human` にする（IMPL-10、[ADR-20261019-004][adr-policy]）。当たらなければ、レビュー AI が計画をレビューし、直すべき点があれば、同じ回の実行で続けて計画 AI に計画し直させる（上限は設定。マージ判定工程（段階 8）の前は、レビューの結果も計画し直したことも Issue に書かない。IMPL-10、[ADR-20261018-004][adr-planrev]、[ADR-20261019-003][adr-replan]）。マージ判定工程（段階 8）の前は、人がすべての計画を読み、誤っていれば次回の実行までに止める（Agent 欄 `paused` か `needs-human`）。マージ判定工程（段階 8）以降は計画を記録として残す（IMPL-10） | 進行 Bot | 変えない | 変えない | 変えない | IMPL-10 |
| 7 | FLOW-21 | draft PR を作る（計画・実装工程（段階 5）ではレビュー AI が無いので、人がレビューする。Agent 欄は空） | 進行 Bot | `In Review` | `reviewer`（レビュー工程（段階 6）以降。計画・実装工程（段階 5）は空） | 変えない | IMPL-6、IMPL-7 |
| 8 | FLOW-22 | 対応ループ：🔴 🟡 があるか、CI が赤（IMPL-27）なら作業 AI が直す。載せ直し（MERGE-6）で差分が変わったときもここに戻る | 進行 Bot | 変えない | `worker`（直している間）→ `reviewer`（再レビュー中） | 変えない | MERGE-6 |
| 9 | FLOW-23 | 収束（CI 緑かつ `verdict=pass`）。🟢 を Issue にしてから PR を ready にする | 進行 Bot | `Ready to Merge` | 空 | 変えない | IMPL-27、IMPL-26、IMPL-11 |
| 10 | FLOW-24 | マージの可否を判定し、印（`merge=ready` / `merge=human`）を PR に書く（修正工程（段階 7）は記録にだけ残し印を書かない。shadow 判定） | 進行 Bot | 変えない | 変えない | 変えない | MERGE-3、「[段階][f06]」 |
| 11 | FLOW-25 | 人が別の PR をマージし base が進んだ。載せ直して、差分が同じなら印を付け直し（Status はそのまま）、変わったら FLOW-22 に戻る（Status を `In Review` に戻す） | 進行 Bot | `Ready to Merge` か `In Review` | 空か `reviewer` | 変えない | MERGE-5、MERGE-6 |
| 12 | FLOW-26 | 印を見てマージする | 人 | 変えない（進行 Bot が次回の実行で検出するまで） | 変えない | 変えない | MERGE-1、MERGE-5 |
| 13 | FLOW-27 | マージを検出し、Issue を閉じ、作業場所（clone と worktree）をフォルダごと消す。Issue を閉じると Project が `Done` にする | 進行 Bot | `Done` | 空 | 変えない | MERGE-7 |

止まる遷移（どの段階からでも起きうる）：

| 番号 | 契機 | 変える人 | Status | Agent 欄 | ラベル | 根拠 |
|---|---|---|---|---|---|---|
| FLOW-28 | 往復の上限（3 回）か、同じ指摘の再発（直したはずの指摘が再び出た） | 進行 Bot | `Blocked` | 変えない | `needs-human` | IMPL-24 |
| FLOW-29 | ADR の起票が要る、同じ実行の失敗が 3 回、累計の利用枠の上限、漏えいの検査、push しないパス（進行 Bot・次のセッション・CI が実行するか、規則として読むパス。IMPL-15）への変更、渡す文の変更、検疫、Issue の状態の変化、ラベルや編集の権限不足、復旧できない状態、Issue に紐づく作業場所の残骸（UNDECIDED-27）、印が無効か `review` の `round` の欠け（IMPL-20、IMPL-18）、結びつきの印の無い PR かマージされずに閉じられた PR（IMPL-6）、CI の `action_required` か、取り消しなどが時間を過ぎても変わらない（IMPL-27）、作業場所の `commondir` か `gitdir` が作ったときと違う（SAFE-12）、記録を失った（「運用の規則」）、想定外の Status（SAFE-4） | 進行 Bot | `Blocked` | 変えない | `needs-human` | IMPL-6、IMPL-8、IMPL-14、IMPL-15、IMPL-18、IMPL-20、IMPL-27、SAFE-4、SAFE-12、TEXT-4、TEXT-7、TEXT-8、TEXT-10、TEXT-11、SETUP-2、「運用の規則」 |
| FLOW-30 | 実行の失敗（利用枠切れ、時間の上限、ターンの上限、`claude` の異常終了、出力契約の欠け、受付 AI の拒否、依存の導入でレジストリにつながらない（IMPL-1）。3 回未満。次回に持ち越す） | 進行 Bot | 変えない | 変えない | 変えない | IMPL-14 の表 |
| FLOW-31 | 人が引き取る | 人 | 変えない | `human` | 変えない | 本節の決まり |
| FLOW-32 | 人が一時停止する | 人 | 変えない | `paused` | 変えない | 「運用の規則」 |
| FLOW-33 | 着手前：先行の Issue（blocked by）が閉じていないので候補に選ばない。待ち | 進行 Bot（判定だけ。欄は変えない） | 変えない（`Todo`） | 変えない | 変えない | FLOW-1、FLOW-7、FLOW-10 |
| FLOW-34 | 作業中：人が依存を足し、先行がまだ閉じていない。レビューと対応ループを止めて待ち、作業ブランチと PR は残す。先行がマージされたら載せ直して続ける | 進行 Bot（判定だけ。欄は変えない） | 変えない | 変えない | 変えない | FLOW-9、FLOW-10 |

止まった状態から戻す手順（人だけが行う）：

- `needs-human`：原因を直してからラベルを外し、Status を戻す。PR が無ければ `Todo`（棚卸しで選び直す）、PR があれば `In Review`（進行 Bot が次回の実行で PR の状態から再開する）。Agent 欄が `human` なら `worker` に戻す
- `human`：返すときは Agent 欄を `worker` に戻す。Status は触った内容に合わせて人が直す
- `paused`：再開するときは Agent 欄を空か `worker` に戻す
- 依存の待ち（FLOW-33・FLOW-34）：人の操作は要らない。先行の Issue が閉じれば、次回の実行で進行 Bot が続ける。待ちを解きたければ人が依存（blocked by）を外す

決まり：

- `needs-human` を付けるときは、どの契機でも Status を `Blocked` にする（`Todo` のままだと翌日また選ばれるため）
- 進行 Bot は Agent 欄を `human` と `paused` にしない。この 2 つは人の意思表示で、進行 Bot は読むだけ。`human` の Issue と PR には進行 Bot は触らない
- 例外として、片付け（FLOW-27、MERGE-7）は `needs-human`・`human`・`paused` の除外より先に判定する。PR がマージされていれば Issue を閉じて作業場所を消し、Agent 欄とラベルは書き換えない（マージは人の最後の操作なので、人の意思表示に反しない。[ADR-20261015-006][adr-flow]）
- 進行 Bot が起票する 🟢 の Issue（`from-review`）は、Project に入れて Status を `Todo` にし、マイルストーンと `agent-ready` は付けない。人がマイルストーンを付けたら受け入れ（TEXT-12）が下書きを用意し、人が `agent-ready` を付けるかを決める（IMPL-26）
- 段階による違い：計画・実装工程（段階 5）では FLOW-21 の Agent 欄が空（人がレビューする番。Status `In Review` と設定の `stage` で表し、`/dev-autopilot` の状態の表示で「人の番」と出す。[ADR-20261015-006][adr-flow]）、修正工程（段階 7）では FLOW-24 の印を書かない（shadow）。段階ごとの振る舞いは設定の `stage` で切り替える（STRUCT-2）
- Status と Agent 欄の更新はすべて進行 Bot が GraphQL で行い、AI のセッションは触らない（IMPL-5、「[Issue と PR の本文の扱い][f04]」）。更新は冪等で、すでにその値なら何もしない
- 進行 Bot は Status と Agent 欄を変えた人を確かめない（値を最後に変えた人は API で取れない。FACT-27 と FACT-28。[10-verified-facts.md][f10]）。Status は、印と PR の状態から決まる値と違えば、人が戻す手順（`needs-human` を外して `Todo` か `In Review` に戻す）に当たるときだけ従い、`needs-human` を外した人の権限で確かめる（TEXT-10）。ほかの違いは想定外の Status として `needs-human`（SAFE-4）。Agent 欄は、止める向き（`human`・`paused`）にも再開の向きにも従う（Project に書ける人がリポの push 以上の人に限られることを運用の前提にする。[ADR-20261015-004][adr-markers]）
- 人は進行 Bot のコメントを編集しない（印が無効になり `needs-human` になる。IMPL-20）。誤りは新しいコメントで補う。受け入れの下書き（`intake` の印）だけは、人が直してよい（TEXT-12）

## 運用の規則

- 状態の正は GitHub に置く（PR の有無と draft、印、Status、Agent 欄、ラベル）。進行 Bot は毎回そこから状態を組み立てる。各手順は「すでに済んでいれば何もしない」（PR があれば作らない、コメントがあれば足さない）
- 手元の記録は判定に使わず、実行をまたぐ上限の数え上げにだけ使う：同じ Issue で続いた実行の失敗の回数、受付 AI が拒否した Issue の次のモデル、Issue ごとの累計の利用枠、1 日の利用枠（[ADR-20261015-006][adr-flow]）
- 記録を失ったときは、上限が黙って効かなくなる側に倒さない。`setup` は記録のフォルダに、作った日時を書いた印のファイルを置く。印のファイルが無ければ `check` は fail にする（人が `setup` をやり直して、失ったことを認める）。
  進行中の Issue（`In Progress` 以降、または計画の印がある）に記録が無ければ、実行の失敗は 1 回目で `needs-human`、累計の利用枠は次のセッションを起こさずに `needs-human`（`note kind=stop reason=records-missing`）。拒否の記録が無ければ受付 AI は `haiku`（完全な ID。IMPL-29）で起動する。当日の利用枠の記録が無ければ、当日の残りの AI のセッションを起こさない
- 多重起動を lockfile で防ぐ。人の `/dev-autopilot` の手動実行は、定期の実行と同じジョブを起こし（macOS は `launchctl kickstart`（`-k` なし。`-k` は動いている進行 Bot を殺す）。Linux は登録した仕組みで同じジョブを起こす操作。[ADR-20261015-001][adr-read]、[ADR-20261017-003][adr-plugin]）、同じ lock を取る。
  `setup` が進行 Bot のプログラムの写しを作り直すときも、同じ lock を取る（STRUCT-3）。lock には PID・ホスト名・起動時刻を書き、PID が生きていなければ古い lock として奪う（電源断の後に全実行が「多重起動」で止まり続けないため）。
  lock で見送った実行と、マシンのスリープや停止で定期の実行が起動しなかった日は記録に残し、`check` が warn に出す（受け入れ工程（段階 4）の「7 回の実行」はこれらを除いて数える）
- 止める合図：全体を止めるのは、当てる先のリポの変数（Actions の変数。名前は構造仕様で決める。例：`DEV_AUTOPILOT_RUN`）。値が `run` のときだけ動き、ほかの値・変数が無い・読めないときは止まる。
  `setup` の GitHub の準備が値を `stop` にして作り、人が `check` の後に `run` にする。人は GitHub の画面（スマートフォンを含む）か人の `gh` で変え、PR は要らない（変えられるのは push 以上の人。org のリポでは、変えた人が org の監査ログに残る）。
  進行 Bot の PAT は変数を読むだけで、変えられない（[ADR-20261017-007][adr-stop]）。その Issue だけを止めるのは、Project の Agent 欄 `paused`。
  進行 Bot は、全体の合図を、実行の最初・AI のセッションを起動する前・GitHub に書き込む前に読み、`run` を読めなければ起動も書き込みもしない。AI のセッションが動いている間も一定の間隔で読み（間隔は基本設計で決める）、
  `run` 以外の値を読んだら、AI のセッションを時間の上限と同じやり方（`SIGTERM`、猶予、`SIGKILL`、残ったコマンドも止める。IMPL-14）で止める。間の読み取りの失敗では止めない（書き込む前にもう一度読むため）。
  合図で止めた実行は、手元の記録に残し、lock を外して終わる。その回は実行の失敗に数えない（IMPL-14）。次の実行は、合図が `run` に戻った後、GitHub の状態から続ける
- 書かずに試す合図：リポの設定の `dryRun: true`（PR で変える。`dryRun` を切って実際に書き始めるのは、安全の境界を緩める向きの変更なので PR に残す。急いで書き込みを止めるのは、全体の止める合図で行う。[ADR-20261017-007][adr-stop]）。進行 Bot は GitHub への書き込み（コメント・ラベル・Project の欄・PR）をすべて手元の記録に「書くつもりだった内容」として残し、実際には書かない。読むことと AI のセッションは行う（受け入れ工程（段階 4）の最初と、新しい段階に入るときに使う）
- 進行 Bot は、毎回の実行の最初に、進行 Bot の clone（`<置き場>/controller/`）で fetch した base ブランチの先端から、リポの設定を読む。1 回の実行は 1 つの版の設定で動き、途中で読み直さない（全体の止める合図は除く）。
  人の作業コピー、AI の worktree、Plugin の写しの中のリポの設定は読まない（SAFE-2）。設定の変更は、base ブランチにマージされた後の次の実行から効く。読めない・形が違うときは作業に入らず止まる（SAFE-4。[ADR-20261017-007][adr-stop]）
- 進行 Bot が途中で死んだら、次回の実行が GitHub の状態から復旧する（FLOW-4 の「次の工程」の決め方）。復旧できない組み合わせ（GitHub の状態から次の工程を決められないもの）はその Issue を `needs-human`。作業場所はセッションごとに作り直すので、作業場所の有無は復旧の判定に使わない（[ADR-20261013-001][adr-workspace]）。Issue に紐づかない作業場所の残骸は `check` の fail（SETUP-2、UNDECIDED-27）

[f04]: ./04-external-text-and-markers.md
[f06]: ./06-setup-and-stages.md
[f10]: ./10-verified-facts.md
[adr-workspace]: ../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-markers]: ../adr/adr-20261015-004-signed-markers-and-actor-checks.md
[adr-flow]: ../adr/adr-20261015-006-run-stages-baseline-agent-field-and-local-records.md
[adr-draft]: ../adr/adr-20261016-002-intake-ai-drafts-every-issue.md
[adr-plugin]: ../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-stop]: ../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-redo]: ../adr/adr-20261018-002-plan-redo-in-same-run.md
[adr-planrev]: ../adr/adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[readme]: ./README.md
[adr-replan]: ../adr/adr-20261019-003-replan-on-plan-review-from-stage-5.md
[adr-policy]: ../adr/adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths.md
