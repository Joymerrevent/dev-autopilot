# 状態遷移と判定

進行 Bot が、毎回の実行で GitHub の状態を読み、Issue ごとに「次の工程」を決めて進める仕組みを決める。
遷移そのもの（正常な順と止まる遷移）は[要件の Issue の状態の遷移][flow]が正本で、基本設計は、何をどの順に読み、どの条件でどの工程に進むかを表にする。
収束とマージの判定の細かい表と、印の種類は、この文書に後で足す。

## 1 回の実行の流れ

![1 回の実行の流れ][fig-run-flow]

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 定期の実行の定義が渡したパスから、マシンの設定を読む（[ADR-20261015-005][adr-settings]） | 作業に入らず止まる（[SAFE-4][safety]） |
| 2 | 記録の置き場の lock を取る。lock には PID・ホスト名・起動時刻を書き、PID が生きていない lock は古い lock として奪う（[運用の規則][flow]） | 実行を見送り、見送ったことを記録に残す（[運用の規則][flow]） |
| 3 | 全体を止める合図（リポの変数 `DEV_AUTOPILOT_RUN`）を読む（[運用の規則][flow]） | 値が `run` でないか読めなければ、作業に入らず止まる（[ADR-20261017-007][adr-stop]） |
| 4 | 進行 Bot の clone で fetch し、base ブランチの先端からリポの設定を読む。1 回の実行は、この版の設定で動く（[ADR-20261017-007][adr-stop] の決定 4） | 読めないか形が違えば、作業に入らず止まる |
| 5 | 点検を回す。作業場所の残骸の見分け（下の「作業場所の残骸」）を含む（[SETUP-2][setup]） | fail が 1 つでもあれば、作業に入らず止まる（[SETUP-2][setup]） |
| 6 | 基線（base ブランチの先端の必須チェック）を読む（[IMPL-9][impl]） | 緑でなければ、新しい着手・最初の実装・載せ直しだけを止め、ほかは続ける（[ADR-20261015-006][adr-flow] の決定 2） |
| 7 | Project の Issue と、結びついた PR と、印と、必須チェックの結果を読み、Issue ごとに状態を組み立てる（下の「状態の組み立て」） | API が失敗したら、作業を増やさずに記録して止まる（[SAFE-4][safety]） |
| 8 | Issue ごとに、「次の工程」の表で工程を決めて進める。Issue を進める順は下の「Issue を進める順」 | — |
| 9 | 所要時間の上限（`run.maxMinutes`）を超えたら、残りの Issue は次の実行に回す（[FLOW-4][flow]） | — |
| 10 | 記録を書き、lock を外す（[FLOW-5][flow]） | — |

- 全体を止める合図は、手順 3 のほかに、AI のセッションを起動する前と、GitHub に書き込む前にも読む。AI のセッションが動いている間は `run.stopSignalPollSeconds` ごとに読む（[運用の規則][flow]）
- 書かずに試す合図（`run.dryRun`）が `true` の間は、GitHub に書く代わりに、書くつもりだった内容を手元の記録に残す。読むことと AI のセッションは行う（[運用の規則][flow]）

## Issue を進める順

1 回の実行の中で、Issue を次の順に進める。同じ組の中は、Issue の番号の小さい順にする。マージの判定だけは PR の番号の小さい順にする（[MERGE-5][merge]）。

![Issue を進める順][fig-issue-order]

| 順 | 組 | 理由 |
| --- | --- | --- |
| 1 | 片付け（結びついた PR がマージされた Issue） | 進行中の Issue の数の上限（`limits.wip`）を先に空け、依存していた PR の載せ直しにつなぐ（[MERGE-7][merge]） |
| 2 | 進行中の Issue（Status が `In Progress` から `Ready to Merge`） | 始めた作業を先に進める。新しい着手は進行中の Issue の数の上限で止まるので、進行中の Issue が先に動かないと流れが詰まる（[FLOW-3][flow]） |
| 3 | 承認の確定（人が `agent-ready` を付けた Issue） | 軽い工程で、着手の前に要る（[FLOW-18][flow]） |
| 4 | 受け入れ（受け入れの印が無い Issue） | 人が読む下書きを早く渡す。数は 1 回の実行の件数の上限（`limits.intakePerRun`）で抑える（[TEXT-12][text]） |
| 5 | 新しい着手（候補の Issue） | 上の組を進めた後に、空いた分だけ着手する。候補の順は、マイルストーンの版の小さい順（[FLOW-1][flow]） |

順は暫定。マージ判定工程（段階 8）までの記録（[FLOW-5][flow]）で、所要時間の上限に当たって次の実行に回された組を見て見直す。

## 状態の組み立て

進行 Bot は、毎回の実行で次のものを読み、Issue ごとの状態を組み立て直す。判定には GitHub の状態と印だけを使い、手元の記録は上限の数え上げにだけ使う（[ADR-20261015-006][adr-flow] の決定 4）。

![状態の組み立て][fig-state-assembly]

| 読むもの | 読む先 | 判定での使い方 |
| --- | --- | --- |
| Status と Agent 欄 | Project（GraphQL） | 今の工程と、人の意思表示（`human`・`paused`） |
| ラベル、マイルストーン、依存（blocked by）、開閉、`agent-ready` を付けた人 | Issue と Issue の timeline | 候補の条件を満たすか（[FLOW-1][flow]）、作業を止めるか（[TEXT-8][text]）、付けた人が push 以上の権限を持つか（[TEXT-10][text]） |
| 印 | Issue のコメント（`intake` `intake-final` `plan` `note`）と、PR のコメント（`review` `fix` `merge` `note`） | どこまで進んだか。投稿者・未編集・署名の 3 つを満たす印だけを読む（[IMPL-20][impl]） |
| 結びついた PR | PR の一覧。結びつきは、ブランチ名・同じリポ・結びつきの印の 3 つで決める（[IMPL-6][impl]） | 開いているか、マージされたか、draft か、head の SHA |
| 必須チェックの結果 | PR の head の SHA と、base ブランチの先端のチェック | CI が緑か（[IMPL-27][impl]）、基線が緑か（[IMPL-9][impl]） |
| 手元の記録 | 記録の置き場 | 実行の失敗の回数、受付 AI の拒否、累計と 1 日の利用枠。上限の数え上げにだけ使う |

## 次の工程

Issue ごとに、表を上から順に見て、最初に当てはまった行の工程に進む。
「重い」は AI のセッションを起動する工程で、Issue ごとに 1 回の実行で 1 つまで。「軽い」は起動しない工程で、重い工程の前後に同じ実行で続けてよい（[FLOW-4][flow]、[ADR-20261015-006][adr-flow] の決定 1）。
push した実行（実装・修正・載せ直し）では、push した Issue をそれ以上進めない。CI の結果が要る工程（収束・マージの判定）は、CI の結果が出ている実行でだけ進める（[FLOW-4][flow]）。

### 止める・待つ・触らない（先に見る）

![次の工程：止める・待つ・触らない][fig-stops]

| 順 | 条件 | 次の工程 |
| --- | --- | --- |
| 1 | 結びついた PR がマージされた | 片付け（軽い）：Issue を閉じ、作業場所を消し、依存していた PR を載せ直す。Agent 欄とラベルは書き換えない。順 2 の除外より先に見る（[FLOW-27][flow]、[MERGE-7][merge]、[ADR-20261015-006][adr-flow] の決定 3） |
| 2 | Agent 欄が `human` か `paused`。または `needs-human` がある | 触らない（[Issue の状態の遷移][flow]の決まり、[FLOW-1][flow]） |
| 3 | 着手した後（Status が `In Progress` 以降）に、Issue が閉じられた・別のリポに移された・`agent-ready` かマイルストーンが外された | `needs-human` にする。作業ブランチと PR は残す（[TEXT-8][text]） |
| 4 | 結びつく PR がマージされずに閉じられた。または、ブランチ名が一致するのに結びつきの印が無い PR がある | `needs-human` にする。印の無い PR のときは `note kind=stop reason=unlinked-pr` を書く（[IMPL-6][impl]） |
| 5 | 進行 Bot の印のうち、未編集か署名の条件を満たさないものがある。または、`review` の印の `round` が欠けている | `needs-human` にし、`note kind=stop reason=marker-invalid` を書く（[IMPL-20][impl]、[IMPL-18][impl]） |
| 6 | Issue に紐づく作業場所の残骸がある（下の「作業場所の残骸」） | `needs-human` にする（[FLOW-29][flow]） |
| 7 | 依存（blocked by）の相手に、閉じていない Issue がある | 待つ（欄は変えない）。作業中なら、レビューと修正も止める（[FLOW-33、FLOW-34][flow]） |
| 8 | Status が、印と PR から決まる値と違う | 人が戻す手順（`needs-human` を外して `Todo` か `In Review` に戻す）に当たるなら、Status に従う。ほかは想定外の Status として `needs-human` にする（[Issue の状態の遷移][flow]の決まり、[SAFE-4][safety]、[TEXT-10][text]） |

`needs-human` にするときは、どの契機でも Status を `Blocked` にする（[Issue の状態の遷移][flow]の決まり）。

### 着手まで（Status が `Todo`）

![次の工程：着手まで][fig-before-start]

| 順 | 条件 | 次の工程 |
| --- | --- | --- |
| 9 | 受け入れの印（`intake`）が無い。`from-review` の Issue は、マイルストーンが付くまで除く | 受け入れ（重い）。1 回の実行の件数の上限（`limits.intakePerRun`）まで（[FLOW-16][flow]、[TEXT-12][text]、[IMPL-26][impl]） |
| 10 | 受け入れの印があり、人が `agent-ready` を付けたが、`intake-final` の印が無い | 承認の確定（軽い）：`agent-ready` を付けた人の権限を確かめ、渡す文のハッシュを `intake-final` の印に書き、依存とマイルストーンを確定する（[FLOW-18][flow]、[TEXT-4][text]、[TEXT-10][text]、[TEXT-12][text]） |
| 11 | `intake-final` の印があり、候補の条件をすべて満たす（[FLOW-1][flow]） | 着手（軽い）：Status を `In Progress`、Agent 欄を `worker` にし、続けて計画（重い）に進む。基線が緑で、着手の件数と進行中の Issue の数が上限の内のときだけ（[FLOW-19][flow]、[FLOW-20][flow]、[FLOW-3][flow]、[IMPL-9][impl]） |

### 作業中（Status が `In Progress` から `Ready to Merge`）

![次の工程：作業中][fig-in-progress]

| 順 | 条件 | 次の工程 |
| --- | --- | --- |
| 12 | 計画の印（`plan`）が無い | 計画（重い）。計画を投稿した実行では、実装に入らない（[FLOW-20][flow]、[IMPL-10][impl]） |
| 13 | 計画の印があり、結びつく PR が無い | 実装と PR の作成（重い）。渡す文のハッシュが `intake-final` の印と同じで、基線が緑のときだけ（[FLOW-21][flow]、[TEXT-4][text]、[IMPL-9][impl]）。ハッシュが違えば `needs-human` にする（[TEXT-4][text]） |
| 14 | PR の head に、まだ `review` の印が無い | 計画・実装工程（段階 5）なら何もしない（人がレビューする番）。レビュー工程（段階 6）以降なら、レビュー（重い）（[FLOW-21、FLOW-22][flow]、[ADR-20261015-006][adr-flow] の決定 3） |
| 15 | 最新の `review` の印が `fix` か、CI が赤で、そのラウンドの `fix` の印がまだ無い | 修正工程（段階 7）以降なら、修正（重い）。往復の回数が上限（`limits.roundTrips`）を超えるか、振動なら、修正せずに `needs-human` にする（[FLOW-22][flow]、[FLOW-28][flow]、[IMPL-23、IMPL-24][impl]）。渡す文のハッシュが `intake-final` の印と違えば、修正せずに `needs-human` にする（[TEXT-4][text]）。レビュー工程（段階 6）では AI は直さず、人の番にする（Agent 欄を空にする） |
| 16 | 最新の `review` の印が `pass` で、CI の結果がまだ出ていない | 待つ。次の実行で読み直す（[IMPL-27][impl]） |
| 17 | 最新の `review` の印が `pass` で、CI が緑。Status が `In Review` | 収束（軽い）：🟢 を Issue にしてから、PR を ready にし、Status を `Ready to Merge` にする（[FLOW-23][flow]、[IMPL-26、IMPL-27][impl]） |
| 18 | Status が `Ready to Merge` で、base ブランチが PR の head より先に進んでいる | 載せ直し（軽い）。基線が緑のときだけ。差分の patch-id が同じなら印を付け直し、変わったら Status を `In Review` に戻して再レビューに回す。衝突したら修正のセッション（重い）で解く（[FLOW-25][flow]、[MERGE-6][merge]） |
| 19 | Status が `Ready to Merge` で、head に `merge` の印が無く、CI が緑 | マージの判定（軽い）。修正工程（段階 7）は判定を記録にだけ残し、印を書かない。マージ判定工程（段階 8）以降は印を書く。`merge=ready` にするのは 1 回の実行で 1 件まで（[FLOW-24][flow]、[MERGE-2、MERGE-3、MERGE-5][merge]） |
| 20 | `merge=ready` の PR が、マージも「判定と違う判断」のコメントも無いまま `limits.remindDays` 日を過ぎ、催促の印（`note kind=remind`）がまだ無い | 催促（軽い）（[MERGE-3][merge]） |
| 21 | どれにも当てはまらない | 何もしない。人の操作か CI を待つ |

- 順 14 と 15 の「そのラウンド」は、PR の head に対して最後に書いた `review` か `fix` の印の `round` で決める。印の数え方と、収束とマージの判定の細かい表は、この文書に後で足す
- 順 15 の、レビュー工程（段階 6）で AI が直さない扱いは、要件が決めていない。段階の表が対応ループを修正工程（段階 7）で足すことに合わせた（[段階の表][setup]）

## 作業場所の残骸

UNDECIDED-27（作業場所の残骸の決め方）を決める。セッションのフォルダ（`<置き場>/sessions/<セッションの名前>/`）は、同じ名前で作り直すか片付けるまで残るので、作業中の Issue のフォルダがあるのは普通の状態（[ADR-20261013-001][adr-workspace]）。
セッションの名前は、役と、Issue か PR の番号を略さずに書く（例：`work-issue-<番号>`、`review-pr-<番号>-round-<ラウンド>`。役ごとの名前の一覧は、AI のセッションの文書で決める）。進行 Bot は、名前から役と番号を読み、次の表で見分ける。

![作業場所の残骸の見分け方][fig-residue]

| フォルダ | 扱い |
| --- | --- |
| 名前から Issue か PR を引け、その Issue が作業中（閉じておらず、結びつく PR がマージも閉じもされていない） | 普通の状態。作り直すか片付けるまで残す（[ADR-20261013-001][adr-workspace]） |
| 名前から引けた PR がマージされた | 片付け（次の工程の順 1）で消す。片付けが途中で止まって残ったものも、次の実行の片付けで消す（[FLOW-27][flow]、[MERGE-7][merge]） |
| 名前から引けた Issue が、PR がマージされないまま閉じられたか、結びつく PR がマージされずに閉じられた | Issue に紐づく残骸。消さずに Issue を `needs-human` にし、人が中身を確かめてから消す（[FLOW-29][flow]） |
| 消す前に読んだ `commondir` か `gitdir` が、作ったときの値（作業場所のパスから決まる値）と違う | 消さずに Issue を `needs-human` にし、`check` で warn にする（[ADR-20261013-001][adr-workspace]、[SAFE-12][safety]） |
| 名前から Issue か PR を引けない。または、引けた番号が設定した Project に無い | Issue に紐づかない残骸。点検を fail にし、実行は作業に入らずに止まる。人が中身を確かめて消す（[運用の規則][flow]、[SETUP-2][setup]、[SAFE-4][safety]） |

進行 Bot が、名前から引けない残骸を消さないのは、何のフォルダか分からないものを消すと、人が調べる材料を失うため。作業に入らないのは、判定できない状態で作業を増やさないため（[SAFE-4][safety]）。

[adr-flow]: ../../adr/adr-20261015-006-run-stages-baseline-agent-field-and-local-records.md
[adr-settings]: ../../adr/adr-20261015-005-settings-values-and-machine-settings.md
[adr-stop]: ../../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-workspace]: ../../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[flow]: ../../requirements/01-issue-flow.md
[impl]: ../../requirements/02-implement-review-fix.md
[merge]: ../../requirements/03-merge.md
[text]: ../../requirements/04-external-text-and-markers.md
[safety]: ../../requirements/05-safety.md
[setup]: ../../requirements/06-setup-and-stages.md
[fig-run-flow]: ./diagrams/02-run-flow.drawio.svg
[fig-issue-order]: ./diagrams/02-issue-order.drawio.svg
[fig-state-assembly]: ./diagrams/02-state-assembly.drawio.svg
[fig-stops]: ./diagrams/02-next-step-stops.drawio.svg
[fig-before-start]: ./diagrams/02-next-step-before-start.drawio.svg
[fig-in-progress]: ./diagrams/02-next-step-in-progress.drawio.svg
[fig-residue]: ./diagrams/02-workspace-residue.drawio.svg
