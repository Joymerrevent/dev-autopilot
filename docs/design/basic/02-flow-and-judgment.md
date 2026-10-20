# 状態遷移と判定

進行 Bot が、毎回の実行で GitHub の状態を読み、Issue ごとに「次の工程」を決めて進める仕組みを決める。
遷移そのもの（正常な順と止まる遷移）は[要件の Issue の状態の遷移][flow]が正本で、基本設計は、何をどの順に読み、どの条件でどの工程に進むかを表にする。

## 工程ごとの PDCA

![工程ごとの PDCA][fig-pdca]

AI が受け持つ工程を、段取り（P）・作業（D）・検証（C）・直す（A）で見ると、次のとおり。AI で回すループのエージェント定義は、ループごとのフォルダに置く（[STRUCT-9][struct]）。

| 工程 | P（段取り） | D（作業） | C（検証） | A（直す） | AI で回すループ |
| --- | --- | --- | --- | --- | --- |
| 受け入れ | 進行 Bot（検疫と、受け入れる Issue を選ぶ処理） | 受付 AI（作業指示の下書き） | 人が下書きを読む | 人が下書きを直すか、`## 作業指示` のコメントを書く | 置かない。人が必ず読むので、AI の検証を足しても人の手間は減らない（定義は `brief/`） |
| 計画 | 作業指示 | 計画 AI | レビュー AI と、人（マージ判定工程（段階 8）の前は、人がすべての計画を読む） | 計画 AI が計画し直す（上限は `limits.planRevisions`） | ある。同じ回の実行の中で回す（定義は `plan/`。[ADR-20261019-003][adr-replan]） |
| 実装と修正 | 計画 | 作業 AI | 機械の検査、レビュー AI（3 つの軸。見落としの軸は記録だけ）、批評 AI | 作業 AI が修正し、次のラウンドでレビューする | ある。往復は `limits.roundTrips` の回数まで（定義は `change/`） |
| マージの判定 | 判定の条件 | 進行 Bot | 人がマージする | 判定の条件を ADR で直す | 置かない。進行 Bot の判定は決まった処理で、同じ入力なら同じ結果になる |

- 前の工程の作業の結果が、次の工程の段取りになる（受け入れの作業指示が計画の段取りに、計画が実装の段取りになる）
- 新しい AI の検証は、記録だけに残して人の判断と比べてから、流れを止める判定に使う（計画のレビュー、見落としの軸、マージの判定）。この進め方も、段階の表で回す PDCA になっている（[段階の表][setup]）

## 1 回の実行の流れ

![1 回の実行の流れ][fig-run-flow]

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 定期の実行の定義が渡したパスから、マシンの設定を読む（[ADR-20261015-005][adr-settings]） | 作業に入らず止まる（[SAFE-4][safety]） |
| 2 | 記録の置き場の lock を取る。lock には PID・ホスト名・起動時刻を書き、PID が生きていない lock は古い lock として奪う（[運用の規則][flow]） | 実行を見送り、見送ったことを記録に残す（[運用の規則][flow]） |
| 3 | 全体を止める合図（リポの変数 `DEV_AUTOPILOT_RUN`）を読む（[運用の規則][flow]） | 値が `run` でないか読めなければ、作業に入らず止まる（[ADR-20261017-007][adr-stop]）。点検の依頼のファイルがあれば、4 と 5 の点検だけを行ってから止まる。試しのセッションと PAT の書き込みの試しは行わず、未確認の warn にする（「[導入と点検][design-setup]」の「点検だけで動かす仕組み」） |
| 4 | 進行 Bot の clone で fetch し、base ブランチの先端からリポの設定を読む。1 回の実行は、この版の設定で動く（[ADR-20261017-007][adr-stop] の決定 4） | 読めないか形が違えば、作業に入らず止まる |
| 5 | 点検を回す。作業場所の残骸の見分け（下の「作業場所の残骸」）を含む。試しのセッションが要るとき（「[導入と点検][design-setup]」の「組と回数」）は、ここで回す（[SETUP-2][setup]） | fail が 1 つでもあれば、作業に入らず止まる（[SETUP-2][setup]）。点検の依頼のファイルがあれば、結果を書いて、作業に入らずに止まる |
| 6 | 基線（base ブランチの先端の必須チェック）を読む（[IMPL-9][impl]） | 緑でなければ、新しい着手・最初の実装・載せ直しだけを止め、ほかは続ける（[ADR-20261015-006][adr-flow] の決定 2） |
| 7 | Project の Issue と、結びついた PR と、印と、必須チェックの結果を読み、Issue ごとに状態を組み立てる（下の「状態の組み立て」） | API が失敗したら、作業を増やさずに記録して止まる（[SAFE-4][safety]） |
| 8 | Issue ごとに、「次の工程」の表で工程を決めて進める。Issue を進める順は下の「Issue を進める順」 | — |
| 9 | Issue を 1 つ進めるごとに、所要時間の上限（`run.maxMinutes`）を見る。上限の内なら 8 に戻って次の Issue を進め、超えたら、残りの Issue は次の実行に回す（[FLOW-4][flow]） | — |
| 10 | 記録と、状態の表示のファイル（`status.json`。「[導入と点検][design-setup]」の「状態の表示」）を書き、lock を外す（[FLOW-5][flow]） | — |

- 全体を止める合図は、手順 3 のほかに、AI のセッションを起動する前と、GitHub に書き込む前にも読む。AI のセッションが動いている間は `run.stopSignalPollSeconds` ごとに読む（[運用の規則][flow]）
- 書かずに試す合図（`run.dryRun`）が `true` の間は、GitHub に書く代わりに、書くつもりだった内容を手元の記録に残す。読むことと AI のセッションは行う（[運用の規則][flow]）
- 書かずに試す間は印を書かないので、判定に手元の記録を使わない決まり（[ADR-20261015-006][adr-flow] の決定 4）のまま、毎回、同じ Issue を受け入れ直す。利用枠は、1 回の実行で受け入れる件数の上限（`limits.intakePerRun`）と費用の上限で抑える。人は、最新の実行の記録にある下書きと選定の結果を読む。異議は、記録の置き場の `dry-run-objections.md` に、`## 異議` の見出しで書き足す（受け入れ工程（段階 4）の出口で、GitHub の異議と合わせて数える。[段階の表][setup]）

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
| ラベル、マイルストーン、依存（blocked by）、開閉。ラベル（`agent-ready`・`needs-human`・`plan:*`）を付けた人と外した人と、その時刻 | Issue と Issue の timeline（`LabeledEvent`・`UnlabeledEvent`） | 候補の条件を満たすか（[FLOW-1][flow]）、作業を止めるか（[TEXT-8][text]）、付けた人と外した人が push 以上の権限を持つか、`agent-ready` を付けた時刻（[TEXT-10][text]） |
| 印 | Issue のコメント（`intake` `intake-final` `plan` `note`）と、PR のコメント（`review` `fix` `merge` `note`） | どこまで進んだか。投稿者・未編集・署名の 3 つを満たす印だけを読む。`intake` の印だけは、人が下書きを直す前提なので、未編集の条件を掛けない（[IMPL-20][impl]、[印と固定の見出しの一覧][markers]） |
| 人のコメント | Issue と PR のコメント（`## 作業指示` などの固定の見出しのコメントと、受け入れの下書きのコメント）と、その編集の記録（`userContentEdits`） | 渡す文（作業指示コメントか、承認された下書き。[TEXT-1][text]）、書いた人と編集した人が push 以上の権限を持つか、最後に編集した時刻（[TEXT-10][text]）、人の却下と判定と違う判断（[FLOW-14][flow]、[MERGE-3][merge]）、異議（`## 異議`。段階の出口に数える。[段階の表][setup]） |
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
| 1 | 結びついた PR がマージされた | 片付け（軽い）：Issue を閉じ、作業場所を消す。Agent 欄とラベルは書き換えない。順 2 の除外より先に見る（[FLOW-27][flow]、[MERGE-7][merge]、[ADR-20261015-006][adr-flow] の決定 3）。依存していた PR の載せ直しは、片付けの中で行わず、依存していた PR の Issue の順 18 で行う（基線の条件と、順 2 の除外を通すため） |
| 2 | Agent 欄が `human` か `paused`。または `needs-human` がある | 触らない（[Issue の状態の遷移][flow]の決まり、[FLOW-1][flow]） |
| 3 | 着手した後（Status が `In Progress` 以降）に、Issue が閉じられた・別のリポに移された・`agent-ready` かマイルストーンが外された | `needs-human` にし（`issue-changed`）、作業ブランチと PR は残す（[TEXT-8][text]） |
| 4 | 結びつく PR がマージされずに閉じられた。または、ブランチ名が一致するのに結びつきの印が無い PR がある | `needs-human` にする。印の無い PR のときは `note kind=stop reason=unlinked-pr` を書く（[IMPL-6][impl]） |
| 5 | 進行 Bot の印のうち、未編集（`intake` の印には掛けない）か署名の条件を満たさないものがある。または、`review` の印の `round` が欠けている | `needs-human` にし、`note kind=stop reason=marker-invalid` を書く（[IMPL-20][impl]、[IMPL-18][impl]）。印の置き場が PR のときは、PR と、結びつく Issue の両方に `needs-human` を付ける。進行 Bot が `needs-human` を読むのは Issue だけで、PR だけに付けても止まらないため |
| 6 | Issue に紐づく作業場所の残骸がある（下の「作業場所の残骸」） | `needs-human` にする（`workspace-residue`。[FLOW-29][flow]） |
| 7 | 依存（blocked by）の相手に、閉じていない Issue がある | 待つ（欄は変えない）。待つことを、`note kind=wait` の印付きのコメントで 1 回だけ書く（[FLOW-10][flow]）。作業中なら、レビューと修正も止める（[FLOW-33、FLOW-34][flow]） |
| 8 | Status が、印と PR から決まる値と違う | 人が戻す手順（`needs-human` を外して `Todo` か `In Review` に戻す）に当たり、`needs-human` を外した人が push 以上の権限を持つなら、Status に従う。push 未満の人が外したなら、`needs-human` を付け直す（`unauthorized-resume`。[TEXT-10][text]）。ほかは想定外の Status として `needs-human` にする（`unexpected-status`。[Issue の状態の遷移][flow]の決まり、[SAFE-4][safety]） |

`needs-human` にするときは、どの契機でも Status を `Blocked` にする（[Issue の状態の遷移][flow]の決まり）。

### 着手まで（Status が `Todo`）

![次の工程：着手まで][fig-before-start]

| 順 | 条件 | 次の工程 |
| --- | --- | --- |
| 9 | 受け入れの印（`intake`）が無い。`from-review` の Issue は、マイルストーンが付くまで除く | `agent-ready` が無ければ、受け入れ。1 回の実行の件数の上限（`limits.intakePerRun`）まで（[FLOW-16][flow]、[TEXT-12][text]、[IMPL-26][impl]、[ADR-20261015-003][adr-intake]）。中の順番は下の「受け入れと承認の確定で確かめること」。`agent-ready` が先に付いていれば、受け入れをせずに `needs-human` を付け（`approved-before-intake`）、`agent-ready` を外して下書きを待つよう案内する |
| 10 | 受け入れの印があり、人が `agent-ready` を付けたが、`intake-final` の印が無い | 承認の確定（軽い）：下の「受け入れと承認の確定で確かめること」を満たせば、渡す文のハッシュを `intake-final` の印に書き、依存とマイルストーンを確定する。満たさなければ、`intake-final` を書かずに `needs-human` を付ける（[FLOW-18][flow]、[TEXT-4][text]、[TEXT-10][text]、[TEXT-12][text]） |
| 11 | `intake-final` の印があり、候補の条件をすべて満たす（[FLOW-1][flow]） | 計画・実装工程（段階 5）以降なら、着手（軽い）：Status を `In Progress`、Agent 欄を `worker` にし、続けて計画（重い）に進む。基線が緑で、着手の件数と進行中の Issue の数（下の「進行中の Issue の数え方」）が上限の内のときだけ（[FLOW-19][flow]、[FLOW-20][flow]、[FLOW-3][flow]、[IMPL-9][impl]）。着手の前に、渡す文のハッシュが `intake-final` の印と同じか（違えば `brief-changed`。[TEXT-4][text]）と、渡す文に秘密の形（鍵の形式、メールアドレスなど）が無いか（あれば `brief-sensitive`。[TEXT-7][text]）を確かめ、外れたら着手せずに `needs-human` を付ける。受け入れ工程（段階 4）では着手せず、候補の条件を満たしたこと（選定の結果）を手元の記録にだけ残す（[段階の表][setup]） |

#### 受け入れと承認の確定で確かめること

| 工程 | 確かめること | 外れたとき |
| --- | --- | --- |
| 受け入れ（順 9） | 題名・本文・コメントを検疫（[TEXT-11][text]）に通す | 受付 AI を起動せず、検疫の結果（見つかった種類と場所。見つかった文の中身は写さない）と、`quarantine=found` の `intake` の印だけを載せた「受け入れの下書き」コメントを書き、`needs-human` を付ける（`quarantine`。[TEXT-12][text]、[ADR-20261015-003][adr-intake]） |
| 受け入れ（順 9） | 検疫で見つからなければ、受付 AI（重い）を起動する | 進行 Bot は、検疫の結果、作業指示の下書き、依存とマイルストーンの候補、計画の難しさの判定と根拠、方針に係る要望かの判定と根拠（方針の要望のときだけ）を載せた「受け入れの下書き」コメントを書く（[TEXT-12][text]） |
| 承認の確定（順 10） | `agent-ready` を付けた人、`## 作業指示` のコメントを書いた人、下書きのコメントを編集した人が、push 以上の権限を持つ | push 未満の人が付けたラベルとした編集を無効として扱い、`needs-human` を付ける（`permission`。[TEXT-10][text]） |
| 承認の確定（順 10） | 渡す文（作業指示コメントか、直した下書き）の投稿と最後の編集が、`agent-ready` を付けた時刻より前 | `needs-human` を付ける（`brief-edited-after-approval`。人は文を直してから `agent-ready` を付け直す。[TEXT-10][text]） |
| 承認の確定（順 10） | `quarantine=found` の Issue では、`## 作業指示` のコメントがある。渡す文は作業指示コメントだけにする（`source=instruction`） | `needs-human` を付け直す（`quarantine`。[TEXT-12][text]、[ADR-20261015-003][adr-intake]） |

- `agent-ready` が受け入れの下書きより先に付いた Issue は、受け入れの対象から外れ（[TEXT-12][text]）、承認の確定の条件（受け入れの印がある）にも当たらないので、何もしないと進まないまま残る。人が下書きを読む前の承認を通さないため、`needs-human` にして知らせる（順 9）

#### 進行中の Issue の数え方

進行中の Issue の数の上限（`limits.wip`）で数える Issue は、着手した後で、閉じていない Issue。

- 着手したかは、計画の印か結びつく PR があるか、Status が `In Progress`・`In Review`・`Ready to Merge` かで見る
- `Blocked` の Issue も、計画の印か結びつく PR があれば数える。人の対処を待つ作業があるうちに、新しい作業を積み増さないため
- 受け入れで止まった Issue と、着手した直後に計画の前で止まった Issue（計画の印も PR も無い）は数えない。人が見る作業がまだ無いため
- 要件の FLOW-3 は、上の数え方を指す。「運用の規則」の記録が無いときの扱いは、計画の印か結びつく PR がある Issue に掛ける

### 作業中（Status が `In Progress` から `Ready to Merge`）

![次の工程：作業中][fig-in-progress]

作業中の Issue では、表を見る前に、PR の head の必須チェックの結果を、IMPL-27 の表の上の行から順に当てる。`action_required` があれば `needs-human` にし（`ci-action-required`）、`cancelled` `skipped` `neutral` `stale` のまま完了から `limits.ciStaleHours` を過ぎていれば `needs-human` にして（`ci-stale`）、表には進まない。ほかの結果（赤、まだ無い、動いている途中）は、表の順 15 と 16 で扱う（[IMPL-27][impl]、[ADR-20261016-001][adr-pr]）。

| 順 | 条件 | 次の工程 |
| --- | --- | --- |
| 12 | 計画の印（`plan`）が無い | 計画（重い。中の順番は下の「計画の工程の中の順番」）。基線が緑のときだけ（計画は新しい着手に含む。[IMPL-9][impl]、[ADR-20261015-006][adr-flow] の決定 2）。計画を投稿した実行では、実装に入らない（[FLOW-20][flow]、[IMPL-10][impl]） |
| 13 | 計画の印があり、結びつく PR が無い | 実装と PR の作成（重い）。渡す文のハッシュが `intake-final` の印と同じで、基線が緑のときだけ（[FLOW-21][flow]、[TEXT-4][text]、[IMPL-9][impl]）。ハッシュが違えば `needs-human` にする（`brief-changed`。[TEXT-4][text]） |
| 14 | PR の head に、まだ `review` の印が無い | 計画・実装工程（段階 5）なら何もしない（人がレビューする番）。レビュー工程（段階 6）以降なら、レビュー（重い）（[FLOW-21、FLOW-22][flow]、[ADR-20261015-006][adr-flow] の決定 3） |
| 15 | 最新の `review` の印が `fix` か、CI が赤で、そのラウンドの `fix` の印がまだ無い | 修正工程（段階 7）以降なら、修正（重い）。往復の回数がすでに上限（`limits.roundTrips`）に達しているか（`round-limit`）、同じ指摘の再発（直したはずの指摘が再び出た。`repeated-finding`）なら、修正せずに `needs-human` にする（[FLOW-22][flow]、[FLOW-28][flow]、[IMPL-23、IMPL-24][impl]。往復の数え方は下の「ラウンドと往復の数え方」）。渡す文のハッシュが `intake-final` の印と違えば、修正せずに `needs-human` にする（`brief-changed`。[TEXT-4][text]）。レビュー工程（段階 6）では AI は直さず、人の番にする（Agent 欄を空にする） |
| 16 | 最新の `review` の印が `pass` で、CI の結果がまだ出ていない | 待つ。次の実行で読み直す（[IMPL-27][impl]） |
| 17 | 最新の `review` の印が `pass` で、CI が緑。Status が `In Review` | 収束（軽い）：🟢 を Issue にしてから、PR を ready にし、Status を `Ready to Merge` にする（[FLOW-23][flow]、[IMPL-26、IMPL-27][impl]）。レビュー工程（段階 6）から行う |
| 18 | Status が `Ready to Merge` で、base ブランチが PR の head より先に進んでいる | 載せ直し（軽い）。基線が緑のときだけ。差分の patch-id が同じなら印を付け直し、変わったら Status を `In Review` に戻して再レビューに回す。衝突したら修正のセッション（重い）で解く（[FLOW-25][flow]、[MERGE-6][merge]） |
| 19 | Status が `Ready to Merge` で、head に `merge` の印が無く、CI が緑 | 修正工程（段階 7）以降なら、マージの判定（軽い）。修正工程（段階 7）は判定を記録にだけ残し、印を書かない。マージ判定工程（段階 8）以降は印を書く。`merge=ready` にするのは 1 回の実行で 1 件まで。`merge=ready` の印があってマージされていない PR がほかにあれば、判定しない（1 件目がマージされるまで待つ）（[FLOW-24][flow]、[MERGE-2、MERGE-3、MERGE-5][merge]、[ADR-20261016-001][adr-pr]） |
| 20 | `merge=ready` の PR が、マージも「判定と違う判断」のコメントも無いまま `limits.remindDays` 日を過ぎ、催促の印（`note kind=remind`）がまだ無い | 催促（軽い）（[MERGE-3][merge]） |
| 21 | どれにも当てはまらない | 何もしない。人の操作か CI を待つ |

- 順 14 と 15 の「そのラウンド」は、PR の head に対して最後に書いた `review` か `fix` の印の `round` で決める。ラウンドの数え方と、収束とマージの判定は、下の「判定」の節で決める
- 順 15 の、レビュー工程（段階 6）で AI が直さない扱いは、要件が決めていない。段階の表が対応ループを修正工程（段階 7）で足すことに合わせた（[段階の表][setup]）。同じく、順 17 の収束はレビュー工程（段階 6）から行い、順 19 のマージの判定は修正工程（段階 7）から行う。要件の「段階による違い」にも、段階 6 の扱いを書いた

### 計画の工程の中の順番

![計画の工程の中の順番][fig-plan-step]

順 12 の計画（重い）は、1 回の実行の中で、次の順に AI のセッションを起動する。工程は 1 つのまま（[FLOW-4][flow]）。
セッションを起動する前には、毎回、「[AI のセッション][design-ai]」の「起動の前の確かめ」を行う。全体の止める合図、1 日の上限、1 回の実行の上限のどれかで起動できなければ、計画を投稿せずに終わり、次の実行で計画の工程を 1 から始め直す（[ADR-20261018-002][adr-redo]、[ADR-20261019-003][adr-replan]）。1 Issue の累計の上限に当たったら、`needs-human` を付ける（`usage-limit`）。

| 順 | すること | 次に進む先 |
| --- | --- | --- |
| 1 | 計画 AI が計画を返す（印とラベルで決めたモデル。`models.plan` か `models.planHard`。ラベル `plan:*` は、付けた人が push 以上の権限を持つときだけ使う。[TEXT-10][text]） | `opus` が自己申告したら 2。しなければ 3 |
| 2 | 計画 AI が `fable` で計画し直す（1 回だけ。`models.planHard`） | 3 |
| 3 | 進行 Bot が、計画の `plan.paths` を push しないパスの一覧（IMPL-15）と照らす | 1 つでも当たれば、計画を投稿せず、`needs-human` を付ける（`plan-blocked-path`。[ADR-20261019-004][adr-policy]）。当たらなければ 4 |
| 4 | レビュー AI が計画をレビューし、`ok` か `fix` と指摘の一覧を返す（`models.planReview`） | `ok` なら 6。`fix` で、レビューの指摘で計画し直した回数が上限（`limits.planRevisions`）に達していなければ 5、達していれば 6 |
| 5 | 計画 AI が、渡す文・直す前の計画・指摘の一覧を受け取って計画し直す（直す前の計画と同じモデル。自己申告は無視して記録に残す） | 3 |
| 6 | 最後の計画を `plan` の印付きで投稿する | 計画を投稿した実行では、実装に入らない |

- マージ判定工程（段階 8）の前は、6 で投稿する計画コメントに、レビューの結果も、レビューの指摘で計画し直したことも書かない（人の判断に影響を与えないため）。直す前の計画、レビューの結果と指摘、やり直した計画は、手元の記録に残す（[IMPL-10][impl]）
- 2 で計画し直したことは、計画コメントに固定の文で書き添える（[ADR-20261018-002][adr-redo]）
- 計画 AI と計画のレビューのどのセッションも、失敗したとき（出力契約が解析できない、時間かターンか費用の上限など）は、実行の失敗として計画を投稿せずに終わり、次の実行で計画の工程を 1 から始め直す（[IMPL-14][impl]）。計画のレビューは、計画・実装工程（段階 5）から計画し直しに使うので、レビューが失敗した計画をそのまま投稿しない
- 上限が初期値（1 回）のとき、計画の工程は最大 5 つのセッション（計画 AI が 3 つ、計画のレビューが 2 つ）を起動し、1 セッションの上限 45 分なら最大 225 分かかる。1 回の実行の所要時間の上限（`run.maxMinutes` の初期値 180 分）を超えたら、ほかの Issue を次の実行に回す
- マージ判定工程（段階 8）以降に、6 で `fix` のまま上限に達した計画を止めるか（`needs-human` にするか）は、マージ判定工程（段階 8）に進む前に別の ADR で決める（[ADR-20261018-004][adr-planrev] の決定 3、[ADR-20261019-003][adr-replan] の決定 3）

## push の手順

![push の手順][fig-push]

作業 AI の commit を GitHub に届ける手順（[IMPL-15][impl]、[ADR-20261013-001][adr-workspace] の「取り込み」）。実装・修正・衝突の解消のセッションの後に行う。AI が書いた commit は、進行 Bot が検査してから、進行 Bot の clone から送る。

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 作業 AI の出力契約を取り出す（「[AI のセッション][design-ai]」の「役ごとの出力契約」） | 実行の失敗として次の実行に持ち越す。止まった理由（`stopReason`）を返したら、push せず、止まった理由で `needs-human` を付ける |
| 2 | 進行 Bot の clone で、AI の作業場所から作業ブランチを取り込み（`git fetch <置き場>/sessions/<セッションの名前>/worktree <ブランチ>`）、取り込んだ先端の SHA を控える。AI の作業場所の clone では git を実行しない（[ADR-20261013-001][adr-workspace]） | 取り込みが objects の検査で失敗したら、`needs-human` を付ける（`workspace-tampered`）。AI が objects を書き換えたおそれがあるため（FACT-18） |
| 3 | 差分（作業場所を作ったときの SHA から、控えた SHA まで）が触れるパスを、push しないパス（`push.blockedPaths`）と照らす | 当たれば push せず、`needs-human` を付ける（`blocked-path`） |
| 4 | 漏えいの検査を、差分、commit のメッセージと作者、PR の題名と本文の案に掛ける。検査は `plugin/src/` の中のものを使い、worktree の中の検査は使わない（[IMPL-15][impl]） | 引っかかれば push せず、`needs-human` を付ける（`leak`） |
| 5 | GitHub の API で、作業ブランチの今の先端（まだ無ければ base ブランチの先端）を読み、控えた SHA の祖先であることを確かめる（fast-forward で届くこと） | 祖先でなければ push せず、`needs-human` を付ける（`not-fast-forward`）。AI が履歴を書き換えたか、ほかの誰かが push したため |
| 6 | 全体の止める合図を読み直し、書かずに試す合図（`run.dryRun`）を見る（[運用の規則][flow]） | 合図が `run` でなければ push せずに止まる。`dryRun` が `true` なら push せず、送るつもりだった SHA を手元の記録に残す |
| 7 | 進行 Bot の clone から、deploy key で `git push origin <控えた SHA>:refs/heads/<型>/issue-<番号>` を送る。ブランチ名でなく SHA を送り、force push はしない（[IMPL-15][impl]） | 断られたら（ruleset など）、実行の失敗として次の実行に持ち越す |
| 8 | GitHub の API で、作業ブランチの先端が送った SHA になったことを読み戻す。初回は PR を draft で作る（[IMPL-11][impl]）。題名と本文は、4 で検査した案を使う | 読み戻した SHA が違えば、`needs-human` を付ける（`unrecoverable`） |

- 載せ直し（[MERGE-6][merge]）は、AI のセッションを起こさずに、進行 Bot の clone で base ブランチを作業ブランチに取り込み（`git merge`）、3〜8 を同じく行う。衝突したら取り込みをやめ、修正のセッション（作業 AI の衝突の解消）で解く。差分の patch-id を比べて、印を付け直すか再レビューに回すかは、上の「作業中」の順 18
- push した実行では、push した Issue を次の実行まで進めない（上の「次の工程」）
- 手順 2 で控えた SHA を、手順 3〜8 のすべてで使う。途中で作業ブランチを読み直さないので、検査した commit と送る commit が同じになる

## 判定

次の工程の表の順 14〜20 で使う判定を決める。判定には、GitHub にある署名付きの印と、PR と必須チェックの状態だけを使う（[ADR-20261015-006][adr-flow] の決定 4）。

### ラウンドと往復の数え方

![ラウンドと往復の数え方][fig-rounds]

- ラウンドは、PR のレビューの 1 回。`review` の印の `round` は 1 から始め、レビューするたびに 1 増やす。欠けていたら `needs-human` にする（[IMPL-18][impl]）
- 修正の後の再レビューと、載せ直しで差分（patch-id）が変わったときの再レビューは、新しいラウンドにする。載せ直しで差分が同じときは、ラウンドを増やさず、同じ内容の印を新しい head に付け直す（[MERGE-6][merge]）
- `fix` の印の `round` は、直した対象の `review` の印の `round` と同じにする（[IMPL-27][impl]）
- 往復の回数は、最新の `review` の印の `round` から 1 を引いた数。最新のラウンドが `fix` か CI が赤で、往復の回数がすでに上限（`limits.roundTrips`）に達していれば、修正せずに `needs-human` にする（[IMPL-24][impl]、[FLOW-28][flow]）

| ラウンド | 起きたこと | 往復の回数 |
| --- | --- | --- |
| 1 | 最初のレビューが `fix` を返し、修正する | 0 回 |
| 2 | 再レビューが `fix` を返し、修正する | 1 回 |
| 3 | 再レビューが `fix` を返し、修正する | 2 回 |
| 4 | 再レビューが `fix` を返す | 3 回。上限（3 回）に達しているので、修正せずに `needs-human` にする |

### 収束の判定

![収束の判定][fig-convergence]

PR が収束した（`Ready to Merge` に進めてよい）かを、次の順に確かめる（[FLOW-23][flow]、[IMPL-27][impl]）。

| 順 | 確かめること | 当てはまらないとき |
| --- | --- | --- |
| 1 | 最新の `review` の印の `head` が、PR の head と同じ | その head はまだレビューしていない。次の工程の表の順 14（レビュー）に進む |
| 2 | 最新の `review` の印の `verdict` が `pass` | 順 15（修正）に進む |
| 3 | 必須チェック（`checks.required`）のすべてが、PR の head に対して `success` | 結果ごとの扱いは IMPL-27 の表のとおり。`action_required` は `needs-human`、`failure` と `timed_out` は修正、まだ無いか動いている途中なら待つ、`cancelled` などは待って `limits.ciStaleHours` を過ぎたら `needs-human`（[IMPL-27][impl]） |

- 必須チェックは、チェックの名前・チェックを出した app・base 側のワークフローのパスの 3 つがすべて一致するものだけを数え、同じ名前が複数あれば最新の 1 件を採る。一致するチェックが無ければ、まだ無いものとして待つ（[IMPL-27][impl]、[ADR-20261016-001][adr-pr]）
- 3 つとも満たせば収束。1 回目のレビューから `pass` なら、1 回で収束する（[IMPL-27][impl]）

### マージの判定

![マージの判定][fig-merge]

Status が `Ready to Merge` の PR を、マージしてよいかで判定する（[MERGE-2、MERGE-3][merge]）。判定の時点で CI が緑でなければ、判定を次の実行に回し、印を書かない（`merge=human` にもしない。[MERGE-3][merge]）。
CI が緑なら、次の条件をすべて確かめ、満たさない条件の記号を `merge` の印の `reasons` に書く。

| 記号（`reasons`） | 満たさない条件 | 確かめ方 |
| --- | --- | --- |
| `not-bot-pr` | dev-autopilot が作った PR でない | PR に、進行 Bot の署名付きの結びつきの印が無い（[MERGE-2][merge]、[IMPL-6][impl]） |
| `base` | 向き先が base ブランチでない（`release/*` ブランチと `main` 向けの PR を含む） | PR の向き先の名前を、マシンの設定の `baseBranch` と比べる（[MERGE-2、MERGE-4][merge]） |
| `not-converged` | 収束していない | 上の「収束の判定」を、判定の時点でもう一度確かめる（[MERGE-2][merge]） |
| `leak` | 漏えいの検査に引っかかった | 判定の時点で、最終の head の diff 全体に検査を掛け直す（[IMPL-15][impl]） |
| `dropped-high` | 批評 AI が 🔴 を再現できなかったラウンドがある | 最新の `review` の印の `dropped_high`（PR ごとの累計）が 1 以上。数え方は「[AI のセッション][design-ai]」の「批評 AI」の統合の規則（[IMPL-22][impl]） |
| `path` | 許可一覧（`merge.allowPaths`）に無いパスを触った | 進行 Bot の clone で取った `git diff --raw <base>...<head>` の一覧と比べる（[MERGE-2、MERGE-4][merge]） |
| `diff-kind` | 状態が追加（A）か変更（M）でないもの（削除・改名・コピー・種類の変更など）がある。または、mode が `100644` でない | 同じ `git diff --raw` の状態と mode を見る。状態は、許す 2 つ（A と M）で照らし、並べていない状態も通さない（[MERGE-2、MERGE-4][merge]） |
| `policy` | 方針に係るパス（`merge.policyPaths`）を触った | 同じ `git diff --raw` の一覧と比べる（[MERGE-2][merge]） |

- 当てはまった記号をすべて `,` でつないで `reasons` に書く。1 つも無ければ `merge=ready`、1 つでもあれば `merge=human` にする（[MERGE-3][merge]）
- `merge=ready` にするのは、1 回の実行で 1 件まで。2 件目以降は判定を次の実行に回し、印を書かない。次の実行でも、1 件目がマージされていなければ、2 件目以降は判定しない（[MERGE-5][merge]、[ADR-20261016-001][adr-pr]）
- 修正工程（段階 7）は、判定を手元の記録にだけ残し、印を書かない。マージ判定工程（段階 8）から印を書く（[FLOW-24][flow]）。計画・実装工程（段階 5）とレビュー工程（段階 6）では判定しない

### 同じ指摘の再発の判定

![同じ指摘の再発の判定][fig-repeat]

修正の後のラウンドで、前のラウンドと同じ指摘がもう一度出たかを、`review` の印の `findings` で見分ける（[ADR-20261019-001][adr-repeat]）。

- 進行 Bot は、`review` の印を書く前に、批評 AI の統合の後に残った 🔴 🟡 の指摘ごとに ID を計算し、`findings` に書く。ID は「軸」「正規化した見出し」「ファイルのパス」を改行でつないだ文字列の SHA-256 の先頭 8 文字（16 進）。正規化は、Unicode の NFKC にそろえ、先頭の重要度の記号と空白を除き、英字を小文字にする（[ADR-20261019-001][adr-repeat] の決定 1）
- 新しいラウンドの `findings` の ID が、前のどれかのラウンドの `findings` にもあれば、同じ指摘の再発とする。間が空いていても数える。同じラウンドの中の重複は数えない（[ADR-20261019-001][adr-repeat] の決定 2）
- 同じ指摘の再発なら、往復の上限と同じく、修正せずに Status を `Blocked` にして `needs-human` を付け、再び出た指摘の見出しと、出たラウンドを Issue にコメントする（[IMPL-24][impl]、[FLOW-28][flow]）

| ラウンド | `findings`（例） | 判定 |
| --- | --- | --- |
| 1 | A、B | 前のラウンドが無い。修正に進む |
| 2 | C | A も B も出ていない。修正に進む |
| 3 | A | A はラウンド 1 にも出ている。同じ指摘の再発として `needs-human` にする |

## 印の種類

UNDECIDED-39（進行 Bot のほかのコメントに付ける印の種類）を決め、`stop` の印の理由の記号をそろえる。
印の形と欄の正本は[印と固定の見出しの一覧][markers]で、基本設計は、どのコメントにどの印を付けるかと、記号の一覧を決める。進行 Bot が投稿するコメントは、すべて署名付きの印を 1 つ持つ（[印と固定の見出しの一覧][markers]）。

### 印が付く順

![印が付く順][fig-markers]

正常な順で、Issue と PR のコメントに付く印。

| 順 | 印 | 置き場 | 書くとき |
| --- | --- | --- | --- |
| 1 | `intake` | Issue | 受け入れの下書きを投稿するとき（次の工程の順 9） |
| 2 | `intake-final` | Issue | 人が `agent-ready` を付けた後、承認を確定するとき（順 10） |
| 3 | `plan` | Issue | 計画を投稿するとき（順 12） |
| 4 | `note kind=link` | Issue と PR | PR を作った直後に、両方に書く（順 13。[IMPL-6][impl]） |
| 5 | `review` | PR | ラウンドごとに、レビューの結果を投稿するとき（順 14。レビュー工程（段階 6）から） |
| 6 | `fix` | PR | 修正を push した後（順 15。修正工程（段階 7）から） |
| 7 | `merge` | PR | マージの判定のとき（順 19。マージ判定工程（段階 8）から） |

このほかに、`note` の印を、待つとき・重なりを警告するとき・催促するとき・止めるとき・モデルを上げたとき・🟢 が重複したときに書く（下の「note の種類」）。

### note の種類

| `kind` | 書くとき | 置き場 | 欄 | 1 回だけ書く判定 |
| --- | --- | --- | --- | --- |
| `wait` | 依存の相手が閉じていないので待つとき（[FLOW-10][flow]） | Issue | — | 同じ `kind` の有効な印があれば書かない |
| `overlap` | 触る範囲が、ほかの進行中の PR と重なったとき。作業 AI の commit を取り込んだとき（「push の手順」の 2）に、変更ファイルの一覧をほかの進行中の PR の変更ファイルと比べる（[FLOW-8][flow]） | Issue | — | 同じ `kind` の有効な印があれば書かない |
| `remind` | `merge=ready` の PR が、マージも「判定と違う判断」のコメントも無いまま日数を過ぎたとき（[MERGE-3][merge]） | PR | — | 同じ `kind` の有効な印があれば書かない |
| `link` | PR を作った直後（[IMPL-6][impl]） | Issue と PR | Issue には `pr=<PR の番号>`、PR には `issue=<Issue の番号>` | 同じ番号の有効な印があれば書かない |
| `stop` | `needs-human` を付けるとき | Issue か PR | `reason=<記号>`（下の「stop の理由の記号」） | 1 回だけにしない。`needs-human` を付けるたびに書く |
| `raise`（足す） | 作業 AI のモデルを `sonnet` から `opus` に上げたとき（[IMPL-25][impl]） | Issue | `from=<別名>`、`to=<別名>` | 同じ `kind` の有効な印があれば書かない（1 つの Issue で上げるのは 1 回） |
| `duplicate`（足す） | 🟢 を Issue にするときに、同じ見出しの open な Issue があり、新しく作らずに PR 番号を足すとき（[IMPL-26][impl]） | すでにある 🟢 の Issue | `pr=<PR の番号>` | 同じ番号の有効な印があれば書かない |

- `raise` と `duplicate` を足し、今ある種類に寄せなかった。`raise` を `stop` に寄せると止まったと読まれる。`duplicate` を `link` に寄せると、PR と Issue の結びつき（[IMPL-6][impl] の判定に使う）を誤る
- `stop` だけは、`needs-human` を付けるたびに書く。`needs-human` の Issue は進行 Bot が触らないので、同じ理由で重ねて書くことは無い。人が `needs-human` を外した後に、同じ理由で再び止まったとき、前の印を見て書かないと、今回止まったことが人に伝わらないため

### stop の理由の記号

`needs-human` を付けるときに書く `note kind=stop` の `reason` の一覧。止まる遷移（[FLOW-28、FLOW-29][flow]）の契機ごとに 1 つ決める。

| `reason` | 止めた契機 |
| --- | --- |
| `round-limit` | 往復の回数が上限に達した（[IMPL-24][impl]） |
| `repeated-finding` | 同じ指摘の再発（直したはずの指摘が再び出た。[IMPL-24][impl]、[ADR-20261019-001][adr-repeat]） |
| `adr-required` | 作業 AI が、ADR の起票か規約の正典の変更が要ると返した（[IMPL-8][impl]） |
| `retry-limit` | 同じ Issue で実行の失敗が続き、回数の上限に達した（[IMPL-14][impl]） |
| `usage-limit` | 1 Issue の累計の費用に、次のセッションの費用の上限を足すと、上限を超える（[IMPL-14][impl]、[AI のセッション][design-ai]の「止め方と利用枠の数え方」） |
| `session-not-stopped` | AI のセッションを止めた後、コマンドの残りが止まったと確かめられない（[IMPL-14][impl]） |
| `leak` | 漏えいの検査に引っかかった（[IMPL-15][impl]） |
| `blocked-path` | diff が、push しないパスに触れている（[IMPL-15][impl]） |
| `not-fast-forward` | push する SHA が、GitHub の作業ブランチの今の先端（まだ無ければ base ブランチの先端）の子孫でない（[IMPL-15][impl]、上の「push の手順」） |
| `plan-blocked-path` | 計画が変える見込みのパスが、push しないパスに当たる（[IMPL-10][impl]、[ADR-20261019-004][adr-policy]） |
| `approved-before-intake` | 受け入れの下書きの前に、`agent-ready` が付いた（[TEXT-12][text]。人が下書きを読む前の承認を通さない） |
| `brief-changed` | 渡す文のハッシュが、承認したときと違う（[TEXT-4][text]） |
| `brief-edited-after-approval` | 渡す文（作業指示コメントか、直した下書き）の投稿か最後の編集が、`agent-ready` を付けた時刻より後（[TEXT-10][text]） |
| `brief-sensitive` | 渡す文に、事業データか認証情報の形が混じっている（[TEXT-7][text]） |
| `quarantine` | 受け入れの検疫で見つかった。または、見つかった Issue に `## 作業指示` のコメントが無いまま `agent-ready` が付いた（[TEXT-12][text]）。または、AI に渡す直前の検疫で見つかった（[TEXT-11][text]、[AI のセッション][design-ai]の「起動の前の確かめ」） |
| `issue-changed` | 作業中に、Issue が閉じられた・移された・`agent-ready` かマイルストーンが外された（[TEXT-8][text]） |
| `permission` | `agent-ready` を付けた人、作業指示コメントを書いた人、下書きのコメントを編集した人が push 未満（[TEXT-10][text]、[FLOW-29][flow]） |
| `unauthorized-resume` | push 未満の人が `needs-human` を外した（[TEXT-10][text]） |
| `unlinked-pr` | ブランチ名が一致するのに、結びつきの印が無い PR がある（[IMPL-6][impl]） |
| `pr-closed` | 結びつく PR が、マージされずに閉じられた（[IMPL-6][impl]） |
| `marker-invalid` | 進行 Bot の印が、未編集（`intake` の印には掛けない）か署名の条件を満たさない。または `review` の印の `round` が欠けている（[IMPL-20、IMPL-18][impl]） |
| `ci-action-required` | CI のワークフローの実行に、人の承認が要る（[IMPL-27][impl]） |
| `ci-stale` | CI の結果が、取り消しなどのまま時間を過ぎても変わらない（[IMPL-27][impl]） |
| `workspace-residue` | Issue に紐づく作業場所の残骸がある（下の「作業場所の残骸」） |
| `workspace-tampered` | 作業場所の `commondir` か `gitdir` が、作ったときの値と違う。または、取り込みの fetch が objects の検査で失敗した（[SAFE-12][safety]、[ADR-20261013-001][adr-workspace]、上の「push の手順」） |
| `settings-tampered` | セッションの安全設定のファイルが、作ったときのハッシュと違う（起動の直前か、セッションの後。[SAFE-2][safety] の (e)、[AI のセッション][design-ai]の「セッションごとの安全設定」） |
| `records-missing` | 進行中の Issue の手元の記録が無い（[運用の規則][flow]） |
| `unrecoverable` | GitHub の状態から、次の工程を決められない（[運用の規則][flow]） |
| `unexpected-status` | Status が、印と PR から決まる値と違い、人が戻す手順にも当たらない（[SAFE-4][safety]） |

## 作業場所の残骸

UNDECIDED-27（作業場所の残骸の決め方）を決める。セッションのフォルダ（`<置き場>/sessions/<セッションの名前>/`）は、同じ名前で作り直すか片付けるまで残るので、作業中の Issue のフォルダがあるのは普通の状態（[ADR-20261013-001][adr-workspace]）。
セッションの名前は、役と、Issue か PR の番号を略さずに書く（例：`work-issue-<番号>`、`review-pr-<番号>-round-<ラウンド>-<軸>`。役ごとの名前の一覧は「[AI のセッション][design-ai]」の「セッションの名前」）。進行 Bot は、名前から役と番号を読み、次の表で見分ける。

![作業場所の残骸の見分け方][fig-residue]

| フォルダ | 扱い |
| --- | --- |
| 名前から Issue か PR を引け、その Issue が作業中（閉じておらず、結びつく PR がマージも閉じもされていない） | 普通の状態。作り直すか片付けるまで残す（[ADR-20261013-001][adr-workspace]） |
| 名前から引けた PR がマージされた | 片付け（次の工程の順 1）で消す。片付けが途中で止まって残ったものも、次の実行の片付けで消す（[FLOW-27][flow]、[MERGE-7][merge]） |
| 名前から引けた Issue が、PR がマージされないまま閉じられたか、結びつく PR がマージされずに閉じられた | Issue に紐づく残骸。消さずに Issue を `needs-human` にし、人が中身を確かめてから消す（[FLOW-29][flow]） |
| 消す前に読んだ `commondir` か `gitdir` が、作ったときの値（作業場所のパスから決まる値）と違う | 消さずに Issue を `needs-human` にし、`check` で warn にする（[ADR-20261013-001][adr-workspace]、[SAFE-12][safety]） |
| 短い一時フォルダの根（`sessionTmp.root`）の下にあり、実行の記録にセッションの名前との組が無い | Issue に紐づかない残骸。点検を fail にし、実行は作業に入らずに止まる。人が中身を確かめて消す（[ADR-20261017-006][adr-tmp]、[AI のセッション][design-ai]の「短い一時フォルダ」） |
| 名前から Issue か PR を引けない。または、引けた番号が設定した Project に無い | Issue に紐づかない残骸。点検を fail にし、実行は作業に入らずに止まる。人が中身を確かめて消す（[運用の規則][flow]、[SETUP-2][setup]、[SAFE-4][safety]） |

進行 Bot が、名前から引けない残骸を消さないのは、何のフォルダか分からないものを消すと、人が調べる材料を失うため。作業に入らないのは、判定できない状態で作業を増やさないため（[SAFE-4][safety]）。

## 手元の記録

![手元の記録][fig-records]

進行 Bot は、記録の置き場（`paths.records`。[設定][design-settings]の「マシンの設定」）に、実行と AI のセッションの記録を残す（[FLOW-5][flow]）。記録は、判定に使わず、上限の数え上げと、段階の出口の数え上げと、人が読むためにだけ使う（[ADR-20261015-006][adr-flow] の決定 4）。AI は sandbox の中から読めず、書けない。

### 置くもの

| ファイル | 1 件 | 欄 |
| --- | --- | --- |
| `runs.jsonl` | 1 回の実行（lock で見送った実行も） | 実行 ID、起動と終わりの時刻、終わり方（作業した・点検だけ・合図で止めた・点検の fail で止めた・lock で見送った）、段階（`run.stage`）、`run.dryRun`、リポの設定を読んだ base ブランチの SHA、点検の fail と warn、基線の結果（緑でない必須チェックの名前）、起動しなかった定期の実行の時刻、Issue ごとの結果（進めた工程、待たせた理由か止めた理由（stop の理由の記号）、マージの判定と理由の記号）、使った時間 |
| `sessions.jsonl` | AI のセッション 1 回 | [FLOW-5][flow] の固定の欄（実行 ID、Issue の番号、ラウンド、役、指定したモデルと effort、実際に使われたモデル（`modelUsage`）、費用、ターン数、実行の成否、作業の成否）。加えて、セッションの名前、PR の番号、起動と終わりの時刻、終わり方と費用の数え方（「[AI のセッション][design-ai]」の「費用の数え方」）、モデルごとのトークン、出力が分からないメッセージがあったか、拒否（`stop_reason: refusal`）で止まったか |
| `outputs/<実行 ID>/<セッションの名前>.json` | AI のセッションの出力契約 1 つ | 取り出した出力契約をそのまま残す。受け入れの判定、直す前の計画と計画のレビューの結果、計画 AI の自己申告、見落としの軸と比べるためのレビューの結果も、ここから読む |
| `streams/<実行 ID>/<セッションの名前>.jsonl` | AI のセッション 1 回の `stream-json` | 受け取った行をそのまま残す。人のホームに会話の記録を残さないので、会話の流れを読むときはここを読む（「[AI のセッション][design-ai]」の「起動の形」の `--no-session-persistence`） |
| `writes/<実行 ID>.jsonl` | GitHub に書いた 1 件（`run.dryRun` の間は、書くつもりだった 1 件） | 書く先（Issue か PR の番号）、種類（コメント・ラベル・Project の欄・PR・push）、本文か値、実際に書いたか |
| `dry-run-objections.md` | 人が書く異議 | `## 異議` の見出し（上の「1 回の実行の流れ」） |

- ファイルは 1 行に 1 件の JSON（JSON Lines）で、後ろに足すだけにし、書き直さない。書くたびに、ファイルを閉じる前にディスクへ書き出す（電源が切れても、前の行を失わないため）
- 最後の行が途中で切れていたら（書いている途中で電源が切れた）、切れた行を読まずに点検で warn に出す。途中の行が読めなければ点検を fail にする。数え上げが少なく出て、上限が効かなくなる側に倒さないため（[運用の規則][flow]）
- 点検の依頼と結果、状態の表示のファイル（`status.json`）、lock、印のファイルの置き場も記録の置き場（「[導入と点検][design-setup]」、[運用の規則][flow]）

### 段階の出口の数え方

段階の出口と、段階で記録する値（[段階の表][setup]）は、上の記録と GitHub から、数えるときに数える。数えるための欄を別に持たない（[ADR-20261018-003][adr-raise] と同じ考え方）。

| 段階 | 数えるもの | 数える元 |
| --- | --- | --- |
| 受け入れ工程（段階 4） | 実行の回数（lock で見送った実行と、起動しなかった定期の実行の時刻を除く）、投稿ありの実行 | `runs.jsonl`、`writes/` |
| 受け入れ工程（段階 4） | 人が `## 異議` の見出しで書いた回数 | Issue のコメントと `dry-run-objections.md` |
| 受け入れ工程（段階 4） | 方針に係る要望の判定と、人が `agent-ready` を付けたか（[ADR-20261019-004][adr-policy]） | `outputs/` の受付 AI、Issue の timeline |
| 計画・実装工程（段階 5） | 人が手直しせずにマージした PR（手直しは、人が PR に commit を足したか、close して作り直したこと） | PR の commit の作者、閉じた PR |
| 計画・実装工程（段階 5） | `plan=` の判定への異議、`fable` にした計画と `opus` の計画の違い、自己申告で捨てた計画 | `outputs/` の受付 AI と計画 AI、Issue のコメント |
| 計画・実装工程（段階 5）から修正工程（段階 7） | 計画のレビューの結果と、計画し直したこと | `outputs/` の計画のレビューと、直す前の計画 |
| 計画・実装工程（段階 5）から修正工程（段階 7） | 作業 AI の最初の実装のモデルと、Issue の種類（作業ブランチの型）ごとの結果 | `sessions.jsonl` の作業 AI、`review` の印、PR |
| レビュー工程（段階 6） | 一致率と却下率、`opus` と `fable` の比べ、見落としの軸の 2 つの数 | `outputs/` のレビュー AI（3 つの軸、比べるための実行、見落としの軸）、人のレビューの結果、`## 却下` のコメント |
| 修正工程（段階 7） | 往復の回数、`opus` に上げた件数、上げた後に Ready to Merge まで進んだ件数 | `review` の印、`sessions.jsonl` の作業 AI、Project の Status |
| 修正工程（段階 7）とマージ判定工程（段階 8） | マージの判定と人の操作が合ったか | `runs.jsonl` のマージの判定、PR のマージと「判定と違う判断」のコメント |

- 記録を消す時期は決めない。段階の出口と見直しに使うため。大きくなりすぎたら、人が古い実行の分を別の場所に移す（移した分は数え上げに使わない）

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
[adr-pr]: ../../adr/adr-20261016-001-ci-states-merge-ready-limit-and-pr-issue-link.md
[adr-repeat]: ../../adr/adr-20261019-001-repeated-findings-by-ids.md
[fig-rounds]: ./diagrams/02-rounds.drawio.svg
[fig-convergence]: ./diagrams/02-convergence.drawio.svg
[fig-merge]: ./diagrams/02-merge-judgment.drawio.svg
[fig-repeat]: ./diagrams/02-repeated-findings.drawio.svg
[markers]: ../../requirements/04-external-text-and-markers.md
[fig-markers]: ./diagrams/02-markers.drawio.svg
[design-ai]: ./03-ai-sessions.md
[struct]: ../../requirements/07-structure.md
[adr-redo]: ../../adr/adr-20261018-002-plan-redo-in-same-run.md
[adr-planrev]: ../../adr/adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[adr-replan]: ../../adr/adr-20261019-003-replan-on-plan-review-from-stage-5.md
[fig-pdca]: ./diagrams/02-pdca.drawio.svg
[fig-plan-step]: ./diagrams/02-plan-step.drawio.svg
[adr-policy]: ../../adr/adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths.md
[adr-intake]: ../../adr/adr-20261015-003-intake-scope-and-quarantine-report.md
[adr-tmp]: ../../adr/adr-20261017-006-per-session-short-tmpdir.md
[design-setup]: ./04-setup-and-check.md
[fig-push]: ./diagrams/02-push.drawio.svg
[fig-records]: ./diagrams/02-records.drawio.svg
[adr-raise]: ../../adr/adr-20261018-003-count-model-raise-at-stage-7.md
[design-settings]: ./01-settings.md
