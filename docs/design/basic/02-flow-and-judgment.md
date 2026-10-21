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
| 1 | 定期の実行の定義が渡したパスから、マシンの設定を読む（[ADR-20261015-005][adr-settings]） | 作業に入らず止まる（[SAFE-4][safety]）。記録の置き場が分からないので、点検の依頼があっても結果を書けない（人の `check` は待つ時間を過ぎたら止まる。「[導入と点検][design-setup]」の「点検だけで動かす仕組み」） |
| 2 | 記録の置き場の lock を取る。lock には PID・ホスト名・起動時刻を書き、PID が生きていない lock は古い lock として奪う（[運用の規則][flow]）。取ったら、手動の 1 回実行と点検の依頼のファイルを見て、起こされ方（定期・手動・点検の依頼）を決める（「[導入と点検][design-setup]」の「見送った実行と、起動しなかった定期の実行」）。続けて、前の実行が残した AI のセッションを止めて片付ける（「[AI のセッション][design-ai]」の「前の実行が残したセッション」） | 実行を見送り、見送ったことを記録に残す（[運用の規則][flow]）。点検の依頼は、動いている実行が終わりに行う（順 10） |
| 3 | 全体の止める合図（リポの変数 `DEV_AUTOPILOT_RUN`）を読む（[運用の規則][flow]）。読む先と認証は、マシンの設定の `repository.*` と `githubApp.*`（`null` なら PAT）で決める（[設定][design-settings]） | 値が `run` でないか読めなければ、作業に入らず止まる（[ADR-20261017-007][adr-stop]）。点検の依頼のファイルがあれば、順 4 の設定の読み込みと順 5 の点検だけを行ってから止まる。試しのセッションと書き込みの権限の試しは行わず、未確認の warn にする（「[導入と点検][design-setup]」の「点検だけで動かす仕組み」） |
| 4 | 進行 Bot の clone で fetch し、base ブランチの先端からリポの設定を読む。1 回の実行は、この版の設定で動く（[ADR-20261017-007][adr-stop] の決定 4） | 読めないか形が違えば、作業に入らず止まる。点検の依頼のファイルがあれば、読めないことを fail として結果に書く |
| 5 | 点検を回す。作業場所の残骸の見分け（下の「作業場所の残骸」）を含む。試しのセッションが要るとき（「[導入と点検][design-setup]」の「組と回数」）は、ここで回す（[SETUP-2][setup]） | fail が 1 つでもあれば、作業に入らず止まる（[SETUP-2][setup]）。点検の依頼のファイルがあれば、結果を書いて、作業に入らずに止まる |
| 6 | 基線（base ブランチの先端の必須チェック）を読む（[IMPL-9][impl]） | 緑でなければ、新しい着手・最初の実装・載せ直しだけを止め、ほかは続ける（[ADR-20261015-006][adr-flow] の決定 2） |
| 7 | Project の Issue と、結びついた PR と、印と、必須チェックの結果を読み、Issue ごとに状態を組み立てる（下の「状態の組み立て」） | API が失敗したら、作業を増やさずに記録して止まる（[SAFE-4][safety]） |
| 8 | Issue ごとに、「次の工程」の表で工程を決めて進める。Issue を進める順は下の「Issue を進める順」 | — |
| 9 | Issue を 1 つ進めるごとに、所要時間の上限（`run.maxMinutes`）を見る。上限の内なら順 8 に戻って次の Issue を進め、超えたら、残りの Issue は次の実行に回す（[FLOW-4][flow]） | — |
| 10 | 点検の依頼のファイルがあれば（実行の途中で人が `check` を呼んだ）、順 3 の合図を読み直し、順 5 の点検だけを行って結果を書く。記録と、状態の表示のファイル（`status.json`。「[導入と点検][design-setup]」の「状態の表示」）を書き、lock を外す（[FLOW-5][flow]） | — |

- 全体の止める合図は、順 3 のほかに、AI のセッションを起動する前と、GitHub に書き込む前にも読む。AI のセッションが動いている間は `run.stopSignalPollSeconds` ごとに読む（[運用の規則][flow]）
- 書かずに試す合図（`run.dryRun`）が `true` の間は、GitHub に書く代わりに、書くつもりだった内容を手元の記録に残す。読むことと AI のセッションは行う（[運用の規則][flow]）。点検の書き込みの権限の試し（書けないことを確かめる試し）は例外で、`true` の間も行う（「[導入と点検][design-setup]」の「PAT の権限」）
- 進行 Bot が起こす子プロセス（`gh`・`git`・`lsof` など。AI のセッションを除く）は、1 つずつ `run.commandMinutes` を過ぎたら止める。読む工程なら作業を増やさずに記録して止まり、push なら実行の失敗として次の実行に持ち越す（[設定][design-settings]、[SAFE-4][safety]）
- 書かずに試すのは、受け入れ工程（段階 4）の最初だけ（[ADR-20261021-002][adr-dry]）。書かずに試す間は印を書かないので、手元の記録の「書くつもりだった受け入れの下書き」を、同じ Issue で受け入れに使う文（題名・本文・コメント）のハッシュが同じ間は、書いたものとして扱う（[ADR-20261015-006][adr-flow] の決定 4 の例外）。扱うのは、受け入れの相手を選ぶとき（次の工程の表の順 9）と、選定の結果を作るときだけ
  - 書くつもりだった `needs-human`（`quarantine` と `retry-limit`）も、同じハッシュの間は書いたものとして扱い、受け入れ直さない。受付 AI が拒否した Issue は、次の実行で `sonnet` で受け入れ直す（[IMPL-29][impl]）
  - 受け入れの相手は、書いたものとして扱う Issue を除いて、Issue の番号の小さい順に `limits.intakePerRun` 件まで。実行を重ねると、番号の大きい Issue にも下書きが出る。記録が無ければ受け入れ直す（受け入れを黙って飛ばさない）
  - 選定の結果は、書いたものとして扱う下書きがある Issue について、候補の条件（[FLOW-1][flow]）から人の承認にかかわる条件（`agent-ready` があること、付けた人の権限、受け入れの下書きがあり人が `agent-ready` を付けていること）を除いて確かめる。満たすか、どの条件で外れたか、着手する順を、手元の記録にだけ残す
  - `run.dryRun` が `false` の実行は、書くつもりだった内容を読まない。書き始めた最初の実行から、GitHub に受け入れの印が無い Issue を改めて受け入れる（書かずに試した下書きは投稿しない）
  - 人は、状態の表示（「[導入と点検][design-setup]」の「状態の表示」）で、書いたものとして扱う下書きと選定の結果を読み、書き始めるまで `agent-ready` を付けない。異議は、記録の置き場の `dry-run-objections.md` に、`## 異議` の見出しで書き足す（受け入れ工程（段階 4）の出口で、GitHub の異議と合わせて数える。[段階の表][setup]）

## Issue を進める順

1 回の実行の中で、Issue を次の順に進める。同じ組の中は、Issue の番号の小さい順にする。ただし、新しい着手（組 5）はマイルストーンの版の小さい順に並べ、同じ版の中を Issue の番号の小さい順にする（[FLOW-1][flow]）。マージの判定だけは PR の番号の小さい順にする（[MERGE-5][merge]）。

![Issue を進める順][fig-issue-order]

| 順 | 組 | 理由 |
| --- | --- | --- |
| 1 | 片付け（結びついた PR がマージされた Issue） | 進行中の Issue の数の上限（`limits.wip`）を先に空け、依存していた PR の載せ直しにつなぐ（[MERGE-7][merge]） |
| 2 | 進行中の Issue（Status が `In Progress` から `Ready to Merge`） | 始めた作業を先に進める。新しい着手は進行中の Issue の数の上限で止まるので、進行中の Issue が先に動かないと流れが詰まる（[FLOW-3][flow]） |
| 3 | 承認の確定（人が `agent-ready` を付けた Issue） | 軽い工程で、着手の前に要る（[FLOW-18][flow]） |
| 4 | 受け入れ（受け入れの印が無い Issue） | 人が読む下書きを早く渡す。数は 1 回の実行の件数の上限（`limits.intakePerRun`）で抑える（[TEXT-12][text]） |
| 5 | 新しい着手（候補の Issue） | 上の組を進めた後に、空いた分だけ着手する。候補の順は、マイルストーンの版の小さい順（[FLOW-1][flow]） |

- 承認の確定（組 3）で `intake-final` の印を書いた Issue は、同じ実行の新しい着手（組 5）の候補に入れる。組を進めるたびに状態を GitHub から読み直さず、その実行で書いた印を、組み立てた状態に足す

順は暫定。マージ判定工程（段階 8）までの記録（[FLOW-5][flow]）で、所要時間の上限に当たって次の実行に回された組を見て見直す。

## 状態の組み立て

進行 Bot は、毎回の実行で次のものを読み、Issue ごとの状態を組み立て直す。判定には GitHub の状態と印だけを使い、手元の記録は上限の数え上げにだけ使う（[ADR-20261015-006][adr-flow] の決定 4）。

- 読むのは、設定した Project（[FLOW-1][flow]）の項目のうち、当てる先のリポ（マシンの設定の `repository.*`）の Issue だけ。ほかのリポの Issue と、Project に入った PR の項目は読まない（PR は、結びつきの印から Issue ごとに読む）
- 閉じた Issue は、片付け（次の工程の順 1）と、着手した後に閉じられた Issue（順 3）と、作業場所の残骸の見分けのためにだけ読む。受け入れ・承認の確定・着手の対象にしない
- Status が空の項目は `Todo` とみなす

![状態の組み立て][fig-state-assembly]

| 読むもの | 読む先 | 判定での使い方 |
| --- | --- | --- |
| Status と Agent 欄 | Project（GraphQL） | 今の工程と、人の意思表示（`human`・`paused`） |
| ラベル、マイルストーン、依存（blocked by）、開閉。ラベル（`agent-ready`・`needs-human`・`plan:*`）を付けた人と外した人と、その時刻 | Issue と Issue の timeline（`LabeledEvent`・`UnlabeledEvent`） | 候補の条件を満たすか（[FLOW-1][flow]）、作業を止めるか（[TEXT-8][text]）、付けた人と外した人が push 以上の権限を持つか、`agent-ready` を付けた時刻（[TEXT-10][text]） |
| 印 | Issue のコメント（`intake` `intake-final` `plan` `note`）と、PR のコメント（`review` `fix` `merge` `note`） | どこまで進んだか。投稿者・未編集・署名の 3 つを満たす印だけを読む。`intake` の印だけは、人が下書きを直す前提なので、未編集の条件を掛けない（[IMPL-20][impl]、[印と固定の見出しの一覧][markers]） |
| 人のコメント | Issue と PR のコメント（`## 作業指示` などの固定の見出しのコメントと、受け入れの下書きのコメント）と、その編集の記録（`userContentEdits`） | 渡す文（作業指示コメントか、承認された下書き。[TEXT-1][text]）、書いた人と編集した人が push 以上の権限を持つか、最後に編集した時刻（[TEXT-10][text]）、人の却下と判定と違う判断（[FLOW-14][flow]、[MERGE-3][merge]）、異議（`## 異議`。段階の出口に数える。[段階の表][setup]） |
| 結びついた PR | PR の一覧。結びつきは、ブランチ名・同じリポ・結びつきの印の 3 つで決める（[IMPL-6][impl]） | 開いているか、マージされたか、draft か、head の SHA |
| 必須チェックの結果 | PR の head の SHA と、base ブランチの先端のチェック | CI が緑か（[IMPL-27][impl]）、基線が緑か（[IMPL-9][impl]） |
| 手元の記録 | 記録の置き場 | 実行の失敗の回数、受付 AI の拒否、累計と 1 日の利用枠。上限の数え上げにだけ使う。書かずに試す間は、書くつもりだった受け入れの下書きも読む（上の「1 回の実行の流れ」。[ADR-20261021-002][adr-dry]） |

## 次の工程

Issue ごとに、表を上から順に見て、最初に当てはまった行の工程に進む。
「重い」は AI のセッションを起動する工程で、Issue ごとに 1 回の実行で 1 つまで。「軽い」は起動しない工程で、重い工程の前後に同じ実行で続けてよい（[FLOW-4][flow]、[ADR-20261015-006][adr-flow] の決定 1）。
push した実行（実装・修正・載せ直し）では、push した Issue をそれ以上進めない。CI の結果が要る工程（収束・マージの判定）は、CI の結果が出ている実行でだけ進める（[FLOW-4][flow]）。
渡す文を AI に渡す工程（計画・実装・レビュー・修正・衝突の解消）は、起動の前に、渡す文のハッシュが最新の `intake-final` の印と同じかを確かめ、違えば `needs-human` にする（`brief-changed`。[TEXT-4][text] の「着手時と対応ループの各回」）。渡す文は、承認の確定と同じ規則で選び、push 以上の人の作業指示コメントが 2 つ以上なら `needs-human` にする（`multiple-instructions`。[ADR-20261021-003][adr-reapprove]）。

### 止める・待つ・触らない（先に見る）

![次の工程：止める・待つ・触らない][fig-stops]

| 順 | 条件 | 次の工程 |
| --- | --- | --- |
| 1 | 結びついた PR がマージされた | 片付け（軽い）：Issue を閉じ、作業場所を消す。Agent 欄とラベルは書き換えない。順 2（触らない）の除外より先に見る（[FLOW-27][flow]、[MERGE-7][merge]、[ADR-20261015-006][adr-flow] の決定 3）。依存していた PR の載せ直しは、片付けの中で行わず、依存していた PR の Issue で、下の「作業中」の表を見る前の確かめで行う（基線の条件と、順 2 の除外を通すため） |
| 2 | Agent 欄が `human` か `paused`。または `needs-human` がある | 触らない（[Issue の状態の遷移][flow]の決まり、[FLOW-1][flow]） |
| 3 | 着手した後（Status が `In Progress` 以降）に、Issue が閉じられた・別のリポに移された・`agent-ready` かマイルストーンが外された | `needs-human` にし（`issue-changed`）、作業ブランチと PR は残す（[TEXT-8][text]） |
| 4 | 結びつく PR がマージされずに閉じられた。または、ブランチ名が一致するのに結びつきの印が無い PR がある | 閉じた PR なら `needs-human` にする（`pr-closed`）。ただし、PR が閉じられた後に、push 以上の人が `needs-human` を外して Status を `Todo` に戻していれば、閉じた PR を結びつきから外し、計画からやり直す（下の「戻した Issue のやり直し」）。印の無い PR は、PR の作者が `safety.botAccounts` のどれかで、head が作業ブランチ、base が base ブランチで、ほかに結びつく PR が無ければ、PR を作った直後に切れた続きとして、結びつきの印（`note kind=link`）を Issue と PR に書いて続ける（軽い。[運用の規則][flow]の「GitHub の状態から復旧する」）。そうでなければ `needs-human` にする（`unlinked-pr`。[IMPL-6][impl]） |
| 5 | 進行 Bot の印のうち、未編集（`intake` の印には掛けない）か署名の条件を満たさないものがある。または、`review` の印の `round` が欠けている | `needs-human` にし、`note kind=stop reason=marker-invalid` を書く（[IMPL-20][impl]、[IMPL-18][impl]）。印の置き場が PR のときは、PR と、結びつく Issue の両方に `needs-human` を付ける。進行 Bot が `needs-human` を読むのは Issue だけで、PR だけに付けても止まらないため |
| 6 | Issue に紐づく作業場所の残骸がある（下の「作業場所の残骸」） | `needs-human` にする（`workspace-residue`。[FLOW-29][flow]） |
| 7 | 依存（blocked by）の相手に、閉じていない Issue がある。かつ、`intake-final` の印があるか、Status が `In Progress` 以降 | 待つ（欄は変えない）。待つことを、`note kind=wait` の印付きのコメントで 1 回だけ書く（[FLOW-10][flow]）。作業中なら、レビューと修正も止める（[FLOW-33、FLOW-34][flow]）。`intake-final` の印の前（受け入れと承認の確定）は、依存で止めない（[TEXT-12][text] の受け入れの対象は依存を問わない） |
| 8 | Status が、印と PR から許す値（下の「印と PR から許す Status」）に無い | 人が戻す手順（`needs-human` を外して `Todo` か `In Review` に戻す）に当たり、`needs-human` を外した人が push 以上の権限を持つなら、Status に従う。push 未満の人が外したなら、`needs-human` を付け直す（`unauthorized-resume`。[TEXT-10][text]）。ほかは想定外の Status として `needs-human` にする（`unexpected-status`。[Issue の状態の遷移][flow]の決まり、[SAFE-4][safety]） |

- `needs-human` にするときは、どの契機でも Status を `Blocked` にする（[Issue の状態の遷移][flow]の決まり）。`needs-human` は、いつも Issue に付ける。印の置き場が PR の契機（順 5、`session-not-stopped` など）では、PR にも付ける。進行 Bot が `needs-human` を読むのは Issue だけで、PR だけに付けても止まらないため
- 進行 Bot の印も `agent-ready` も無い Issue（人が Project で自分の作業に使っている Issue など）は、順 8 の確かめと、進行中の Issue の数え方の対象にしない

#### 印と PR から許す Status

| 印と PR | 許す Status | 違うときの扱い |
| --- | --- | --- |
| 計画の印も PR も無い | `Todo`。着手した後（Agent 欄が `worker`）は `In Progress` も | 順 8 |
| 計画の印があり、PR が無い | `In Progress` | 順 8 |
| draft の PR がある | `In Review`。PR を作った直後に切れた `In Progress` も | `In Progress` なら、次の工程の前に `In Review` に書き直す（軽い） |
| ready の PR がある | `Ready to Merge`。ready にした直後に切れた `In Review` と、載せ直しやレビューで戻した `In Review` も | 順 8 |
| `needs-human` がある | `Blocked` | 順 2 で触らない |
| `needs-human` が無く、`Blocked` | 人が戻す手順の途中とみなし、何もしない（状態の表示に出す） | — |

- 進行 Bot が Status を書く途中で切れた組（draft の PR があって `In Progress`）は、[運用の規則][flow]の「GitHub の状態から復旧する」として、次の工程の前に書き直す。人が戻す手順（`Todo` か `In Review`）のほかの値を人が書いたときは、順 8 で止める

#### 戻した Issue のやり直し

人が戻す手順（`needs-human` を外して `Todo` に戻す。[止まった状態から戻す手順][flow]）で戻した Issue を、次のように扱う。

- 計画の印が、最後に `needs-human` を外した時刻（timeline の `UnlabeledEvent`）より前なら、計画の印が無いものとして、着手の後に計画し直す（順 12）。止まった理由（`adr-required`、`plan-blocked-path` など）が計画にあることが多く、人が直したものを計画に入れるため。読むのは、最新の計画の印だけ
- マージされずに閉じた PR が、最後に `needs-human` を外した時刻より前に閉じられていれば、その PR を結びつきから外す。作業ブランチは、人が消してから戻す（残っていれば、push の手順の順 5 で `not-fast-forward` になる）

#### 承認のやり直し

承認した後に渡す文を直したいとき、人は文を直して `Todo` に戻し、`agent-ready` を付け直す（[止まった状態から戻す手順][flow]、[ADR-20261021-003][adr-reapprove]）。進行 Bot は、次のように扱う。

- 次をすべて満たす Issue を、承認のやり直しとして、承認の確定（順 10）に回す
  - `intake-final` の印がある
  - push 以上の人が `agent-ready` を付けた最新の時刻（timeline の `LabeledEvent`）が、最新の `intake-final` の印の投稿より後
  - Status が `Todo` で、`needs-human` が無く、開いている結びつく PR が無い
- 承認の確定と同じこと（下の「受け入れと承認の確定で確かめること」）を確かめ、満たせば新しい `intake-final` の印を新しいコメントで書く。時刻の確かめ（`brief-edited-after-approval`）は、最新の `agent-ready` を付けた時刻で見る
- 進行 Bot は、最新の `intake-final` の印だけを読む。渡す文のハッシュの照らし（`brief-changed`）も、最新の印で行う
- 計画の印が、最新の `intake-final` の印より前なら、計画の印が無いものとして、着手の後に計画し直す（順 12）
- 着手した後（Status が `Todo` でない）に付け直した `agent-ready` は、やり直しとして扱わない。渡す文が変わっていれば、ハッシュの照らしで止まる（`brief-changed`）。人は、戻す手順で `Todo` に戻してから付け直す

### 着手まで（Status が `Todo`）

![次の工程：着手まで][fig-before-start]

| 順 | 条件 | 次の工程 |
| --- | --- | --- |
| 9 | 受け入れの印（`intake`）が無い。`from-review` の Issue は、マイルストーンが付くまで除く | `agent-ready` が無ければ、受け入れ。1 回の実行の件数の上限（`limits.intakePerRun`）まで（[FLOW-16][flow]、[TEXT-12][text]、[IMPL-26][impl]、[ADR-20261015-003][adr-intake]）。中の順番は下の「受け入れと承認の確定で確かめること」。`agent-ready` が先に付いていれば、受け入れをせずに `needs-human` を付け（`approved-before-intake`）、`agent-ready` を外して下書きを待つよう案内する。書かずに試す間は、書いたものとして扱う下書きのある Issue を、同じ文の間は除く（上の「1 回の実行の流れ」） |
| 10 | 受け入れの印があり、人が `agent-ready` を付けたが、`intake-final` の印が無い。または、承認のやり直しに当たる（上の「承認のやり直し」） | 承認の確定（軽い）：下の「受け入れと承認の確定で確かめること」を満たせば、渡す文のハッシュを `intake-final` の印に書き、依存とマイルストーンを確定する。渡す文の秘密の形と検疫も、ここで確かめる（`intake-final` を書いた後に見つかると、着手した後に止まり、人が承認をやり直す手間が増えるため）。満たさなければ、`intake-final` を書かずに `needs-human` を付ける（[FLOW-18][flow]、[TEXT-4][text]、[TEXT-10][text]、[TEXT-12][text]） |
| 11 | `intake-final` の印があり、候補の条件をすべて満たす（[FLOW-1][flow]） | 計画・実装工程（段階 5）以降なら、着手（軽い）：Status を `In Progress`、Agent 欄を `worker` にし、続けて計画（重い）に進む。基線が緑で、着手の件数と進行中の Issue の数（下の「進行中の Issue の数え方」。着手しようとする Issue 自身は数えない）が上限の内のときだけ（[FLOW-19][flow]、[FLOW-20][flow]、[FLOW-3][flow]、[IMPL-9][impl]）。着手の前に、渡す文のハッシュが `intake-final` の印と同じか（違えば `brief-changed`。[TEXT-4][text]）を確かめ、外れたら着手せずに `needs-human` を付ける。受け入れ工程（段階 4）では着手せず、候補の条件を満たしたこと（選定の結果）を手元の記録にだけ残す（[段階の表][setup]）。書かずに試す間の選定の結果は、上の「1 回の実行の流れ」の決まりで作る |

#### 受け入れと承認の確定で確かめること

| 工程 | 確かめること | 外れたとき |
| --- | --- | --- |
| 受け入れ（順 9） | 題名・本文・コメントを検疫（[TEXT-11][text]）に通す | 受付 AI を起動せず、検疫の結果（見つかった種類と場所。見つかった文の中身は写さない）と、`quarantine=found` の `intake` の印だけを載せた「受け入れの下書き」コメントを書き、`needs-human` を付ける（`quarantine`。[TEXT-12][text]、[ADR-20261015-003][adr-intake]） |
| 受け入れ（順 9） | 検疫で見つからなければ、受付 AI（重い）を起動する | 進行 Bot は、検疫の結果、作業指示の下書き、依存とマイルストーンの候補、計画の難しさの判定と根拠、方針に係る要望かの判定と根拠（方針の要望のときだけ）を載せた「受け入れの下書き」コメントを書く（[TEXT-12][text]） |
| 受け入れ（順 9） | 受付 AI が返した文の欄（下書きの 4 つの欄、`planReason`、`policyReason`）を、投稿の前に検疫と漏えいの検査に通す（[TEXT-12][text] の「下書きも検疫に通す」、[ADR-20261016-002][adr-draft] の決定 1） | 見つかれば下書きを投稿せず、検疫の結果と `quarantine=found` の `intake` の印だけを載せた「受け入れの下書き」コメントを書き、`needs-human` を付ける（`quarantine`。上の行と同じ形） |
| 承認の確定（順 10） | `agent-ready` を付けた人と、下書きのコメントと読む作業指示コメントを編集した人が、push 以上の権限を持つ。`## 作業指示` のコメントは、書いた人が push 以上のものだけを読み、push 未満の人のものは無いものとして扱う（止めない。[ADR-20261021-003][adr-reapprove]） | push 未満の人が付けたラベルとした編集を無効として扱い、`needs-human` を付ける（`permission`。[TEXT-10][text]） |
| 承認の確定（順 10） | push 以上の人の `## 作業指示` のコメントが 1 つまで | `intake-final` を書かずに `needs-human` を付ける（`multiple-instructions`）。人は 1 つを残してほかのコメントを消すか見出しを外してから、`agent-ready` を付け直す（[TEXT-12][text]、[ADR-20261021-003][adr-reapprove]） |
| 承認の確定（順 10） | 渡す文（作業指示コメントか、直した下書き）の投稿と最後の編集が、`agent-ready` を付けた時刻（承認をやり直すときは、最新の時刻）より前 | `needs-human` を付ける（`brief-edited-after-approval`。人は文を直してから `agent-ready` を付け直す。[TEXT-10][text]） |
| 承認の確定（順 10） | `quarantine=found` の Issue では、`## 作業指示` のコメントがある。渡す文は作業指示コメントだけにする（`source=instruction`） | `needs-human` を付け直す（`quarantine`。[TEXT-12][text]、[ADR-20261015-003][adr-intake]） |
| 承認の確定（順 10） | 渡す文に、秘密の形（鍵の形式、メールアドレスなど）が無い（[TEXT-7][text]）。渡す文を検疫に通して見つからない（[TEXT-11][text]） | `intake-final` を書かずに `needs-human` を付ける（`brief-sensitive` か `quarantine`）。人は文を直してから `agent-ready` を付け直す（`intake-final` が無いので、承認の確定をやり直せる） |

- `agent-ready` が受け入れの下書きより先に付いた Issue は、受け入れの対象から外れ（[TEXT-12][text]）、承認の確定の条件（受け入れの印がある）にも当たらないので、何もしないと進まないまま残る。人が下書きを読む前の承認を通さないため、`needs-human` にして知らせる（順 9）

#### 進行中の Issue の数え方

進行中の Issue の数の上限（`limits.wip`）で数える Issue は、着手した後で、閉じていない Issue。

- 着手したかは、計画の印か結びつく PR があるか、Status が `In Progress`・`In Review`・`Ready to Merge` かで見る
- `Blocked` の Issue も、計画の印か結びつく PR があれば数える。人の対処を待つ作業があるうちに、新しい作業を積み増さないため
- 受け入れで止まった Issue と、着手した直後に計画の前で止まった Issue（計画の印も PR も無い）は数えない。人が見る作業がまだ無いため
- 要件の FLOW-3 は、上の数え方を指す
- 記録を失ったときの扱い（「[運用の規則][flow]」の `records-missing`）は、進行中の数え方と違い、着手した Issue（Status が `In Progress`・`In Review`・`Ready to Merge` か、計画の印か結びつく PR がある Issue）に掛ける（[ADR-20261015-006][adr-flow] の決定 4 の「進行中の Issue」）。着手の直後で計画の前の Issue も、失敗の回数と累計を数え直さないため

### 作業中（Status が `In Progress` から `Ready to Merge`）

![次の工程：作業中][fig-in-progress]

作業中の Issue では、表を見る前に、次の 3 つを順に見る。

- PR の head の必須チェックの結果を、IMPL-27 の表の上の行から順に当てる。`action_required` があれば `needs-human` にし（`ci-action-required`）、`cancelled` `skipped` `neutral` `stale` のまま完了から `limits.ciStaleHours` を過ぎていれば `needs-human` にして（`ci-stale`）、表には進まない。ほかの結果（赤、まだ無い、動いている途中）は、表の順 15（修正）と順 16（CI を待つ）で扱う（[IMPL-27][impl]、[ADR-20261016-001][adr-pr]）
- 依存の待ち（順 7 の `note kind=wait` の印）の後で、依存の相手がすべて閉じ、依存の相手の Issue に結びつく PR のマージの commit が、PR の head にまだ含まれていなければ、表を見る前に載せ直す（順 18 と同じ手順。基線が緑のときだけ。[FLOW-9][flow]、[FLOW-34][flow]、[MERGE-7][merge]）。Status が `In Review` のまま待っていた PR も、先行の変更を取り込んでからレビューと修正に戻すため
- Status が `Ready to Merge` で、PR の head が最新の `review` の印の `head` と違う（人の push か、修正。載せ直しで差分が同じときは、印を付け直すので当たらない）なら、Status を `In Review` に戻す（軽い）。新しい head をレビューし、収束（順 17）で 🟢 を Issue にしてから `Ready to Merge` に戻すため（[MERGE-6][merge]、[IMPL-26][impl]）

| 順 | 条件 | 次の工程 |
| --- | --- | --- |
| 12 | 計画の印（`plan`）が無い。または、最新の計画の印が、最後に `needs-human` を外した時刻より前（上の「戻した Issue のやり直し」）か、最新の `intake-final` の印より前（上の「承認のやり直し」） | 計画（重い。中の順番は下の「計画の工程の中の順番」）。基線が緑のときだけ（計画は新しい着手に含む。[IMPL-9][impl]、[ADR-20261015-006][adr-flow] の決定 2）。計画を投稿した実行では、実装に入らない（[FLOW-20][flow]、[IMPL-10][impl]） |
| 13 | 計画の印があり、結びつく PR が無い | 実装と PR の作成（重い）。基線が緑のときだけ（[FLOW-21][flow]、[IMPL-9][impl]）。作業ブランチが origin にすでにあれば（push の後、PR を作る前に切れた）、実装し直さず、origin の作業ブランチに push の手順の順 3 と 4（照らしと漏えいの検査）を掛けてから、順 8 の PR の作成だけを行う（軽い。[運用の規則][flow]の「GitHub の状態から復旧する」） |
| 14 | PR の head に、まだ `review` の印が無い | 計画・実装工程（段階 5）なら何もしない（人がレビューする番）。レビュー工程（段階 6）以降なら、レビュー（重い）。修正の後と人の push の後の再レビューも、ここに当たる（[FLOW-21、FLOW-22][flow]、[ADR-20261015-006][adr-flow] の決定 3） |
| 15 | 最新の `review` の印が `fix` か、CI が赤で、そのラウンドの `fix` の印がまだ無い | 修正工程（段階 7）以降なら、修正（重い）。往復の回数がすでに上限（`limits.roundTrips`）に達しているか（`round-limit`）、同じ指摘の再発（直したはずの指摘が再び出た。`repeated-finding`）なら、修正せずに `needs-human` にする（[FLOW-22][flow]、[FLOW-28][flow]、[IMPL-23、IMPL-24][impl]。往復の数え方は下の「ラウンドと往復の数え方」）。レビュー工程（段階 6）では、AI は直さず、人の番にする（Agent 欄を空にする。[Issue の状態の遷移][flow]の「段階による違い」）。往復の上限と同じ指摘の再発は、AI が直す往復の歯止めなので、段階 6 では掛けない |
| 16 | 最新の `review` の印が `pass` で、CI の結果がまだ出ていない | 待つ。次の実行で読み直す（[IMPL-27][impl]） |
| 17 | 最新の `review` の印が `pass` で、CI が緑。Status が `In Review` | 収束（軽い）：🟢 を Issue にしてから、PR を ready にし、Status を `Ready to Merge` にする（[FLOW-23][flow]、[IMPL-26、IMPL-27][impl]）。レビュー工程（段階 6）から行う |
| 18 | Status が `Ready to Merge` で、base ブランチが PR の head より先に進んでいる | 載せ直し（軽い）。基線が緑のときだけ。差分の patch-id（行番号を除いた差分から git が作る、変更の中身の指紋）が同じなら印を付け直し、変わったら Status を `In Review` に戻して再レビューに回す。衝突したら、修正工程（段階 7）以降は修正のセッション（重い）で解き、レビュー工程（段階 6）では人の番にする（Agent 欄を空にし、衝突したことを Issue にコメントする）（[FLOW-25][flow]、[MERGE-6][merge]） |
| 19 | Status が `Ready to Merge` で、head に `merge` の印が無く、CI が緑で、base ブランチが PR の head より先に進んでいない | 修正工程（段階 7）以降なら、マージの判定（軽い）。修正工程（段階 7）は判定を記録にだけ残し、印を書かない。マージ判定工程（段階 8）以降は印を書く。`merge=ready` にするのは 1 回の実行で 1 件まで。`merge=ready` の印があってマージされていない、開いている PR がほかにあれば、`merge=ready` になる判定を次の実行に回す（1 件目がマージされるまで待つ）。`merge=human` の判定は 1 件の上限に数えず、書く（[FLOW-24][flow]、[MERGE-2、MERGE-3、MERGE-5][merge]、[ADR-20261016-001][adr-pr] の決定 2） |
| 20 | `merge=ready` の PR が、マージも「判定と違う判断」のコメントも無いまま `limits.remindDays` 日を過ぎ、催促の印（`note kind=remind`）がまだ無い | 催促（軽い）（[MERGE-3][merge]） |
| 21 | どれにも当てはまらない | 次の待ちに当たれば、何もしない：基線が緑でない（新しい着手・最初の実装・載せ直しを止めている）、着手の件数か進行中の Issue の数が上限、人のレビューか修正の番（段階 5・6）、CI の結果、人のマージ（`merge` の印のある PR と、段階 6・7 の `Ready to Merge` の PR）。どれにも当たらなければ、GitHub の状態から次の工程を決められない組み合わせとして `needs-human` にする（`unrecoverable`。[運用の規則][flow]） |

- 順 14（レビュー）と順 15（修正）の「そのラウンド」は、PR の head に対して最後に書いた `review` か `fix` の印の `round` で決める。ラウンドの数え方と、収束とマージの判定は、下の「判定」の節で決める
- レビュー工程（段階 6）で AI が直さない扱い（順 15）は、要件の「[段階による違い][flow]」が決めている。順 17 の収束はレビュー工程（段階 6）から行い、順 19 のマージの判定は修正工程（段階 7）から行う（[段階の表][setup]）
- 収束（順 17）で Issue にする 🟢 は、その PR のすべてのラウンドの `review` のコメント（進行 Bot が書き、未編集のもの）の 🟢（批評 AI の統合で 🟡 から下げたものを含む）。指摘の ID ごとに 1 件にまとめる。読み方は「[AI のセッション][design-ai]」の「レビューのコメントの形」（[IMPL-26][impl]）

### 計画の工程の中の順番

![計画の工程の中の順番][fig-plan-step]

順 12 の計画（重い）は、1 回の実行の中で、次の順に AI のセッションを起動する。工程は 1 つのまま（[FLOW-4][flow]）。
セッションを起動する前には、毎回、「[AI のセッション][design-ai]」の「起動の前の確かめ」を行う。全体の止める合図、1 日の上限、1 回の実行の上限のどれかで起動できなければ、計画を投稿せずに終わり、次の実行で計画の工程を 1 から始め直す（[ADR-20261018-002][adr-redo]、[ADR-20261019-003][adr-replan]）。1 Issue の累計の上限に当たったら、`needs-human` を付ける（`usage-limit`）。

| 順 | すること | 次に進む先 |
| --- | --- | --- |
| 1 | 計画 AI が計画を返す（モデルは `intake-final` の印の `plan=`。承認したときのラベル `plan:*` で上書き済みの値で、承認の後に付いたラベルは読まない。`models.plan` か `models.planHard`。[TEXT-10][text]、[ADR-20261018-002][adr-redo]） | `opus` が自己申告したら 2。しなければ 3 |
| 2 | 計画 AI が `fable` で計画し直す（1 回だけ。`models.planHard`） | 3 |
| 3 | 進行 Bot が、計画の `plan.paths` を push しないパスの一覧（IMPL-15）と照らす | 1 つでも当たれば、計画を投稿せず、`needs-human` を付ける（`plan-blocked-path`。[ADR-20261019-004][adr-policy]）。当たらなければ 4 |
| 4 | レビュー AI が計画をレビューし、`ok` か `fix` と指摘の一覧を返す（`models.planReview`） | `ok` なら 6。`fix` で、レビューの指摘で計画し直した回数が上限（`limits.planRevisions`）に達していなければ 5、達していれば 6 |
| 5 | 計画 AI が、渡す文・直す前の計画・指摘の一覧を受け取って計画し直す（直す前の計画と同じモデル。自己申告は無視して記録に残す） | 3 |
| 6 | 最後の計画の文の欄を、検疫（[TEXT-11][text]）と漏えいの検査に通してから、`plan` の印付きで投稿する | 計画を投稿した実行では、実装に入らない。検疫か漏えいの検査で見つかれば、投稿せずに `needs-human` を付ける（`quarantine` か `leak`）。計画の印が無いので、人が戻せば計画からやり直す |

- マージ判定工程（段階 8）の前は、順 6 で投稿する計画コメントに、レビューの結果も、レビューの指摘で計画し直したことも書かない（人の判断に影響を与えないため）。直す前の計画、レビューの結果と指摘、やり直した計画は、手元の記録に残す（[IMPL-10][impl]）
- 順 2 で計画し直したことは、計画コメントに固定の文で書き添える（[ADR-20261018-002][adr-redo]）
- 計画 AI と計画のレビューのどのセッションも、失敗したとき（出力契約が解析できない、時間かターンか費用の上限など）は、実行の失敗として計画を投稿せずに終わり、次の実行で計画の工程を 1 から始め直す（[IMPL-14][impl]）。計画のレビューは、計画・実装工程（段階 5）から計画し直しに使うので、レビューが失敗した計画をそのまま投稿しない
- 上限が初期値（1 回）のとき、計画の工程は最大 5 つのセッション（計画 AI が 3 つ、計画のレビューが 2 つ）を起動し、1 セッションの上限 45 分なら最大 225 分かかる。1 回の実行の所要時間の上限（`run.maxMinutes` の初期値 180 分）を超えたら、ほかの Issue を次の実行に回す
- マージ判定工程（段階 8）以降に、順 6 で `fix` のまま上限に達した計画を止めるか（`needs-human` にするか）は、マージ判定工程（段階 8）に進む前に別の ADR で決める（[ADR-20261018-004][adr-planrev] の決定 3、[ADR-20261019-003][adr-replan] の決定 3）

## push の手順

![push の手順][fig-push]

作業 AI の commit を GitHub に届ける手順（[IMPL-15][impl]、[ADR-20261013-001][adr-workspace] の「取り込み」）。実装・修正・衝突の解消のセッションの後と、進行 Bot だけで行う載せ直しの後に行う。AI が書いた commit は、進行 Bot が検査してから、進行 Bot の clone から送る。

照らす範囲は、PR が base ブランチに足す commit（進行 Bot の clone で fetch した base ブランチの先端の SHA を `<base>` として、`git rev-list <base>..<控えた SHA>`）にする。載せ直しと衝突の解消で取り込んだ base の commit は、範囲に入らない。人が base ブランチに入れた変更を、AI の変更として止めないため。

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 作業 AI の出力契約を取り出す（「[AI のセッション][design-ai]」の「役ごとの出力契約」）。進行 Bot だけで行う載せ直しでは、とばす | 取り出せなければ、実行の失敗として次の実行に持ち越す。取り出せて、止まった理由（`stopReason`）を返していれば、push せず、止まった理由で `needs-human` を付ける |
| 2 | 進行 Bot の clone で、AI の作業場所から作業ブランチを取り込み（`git fetch <置き場>/sessions/<セッションの名前>/worktree <ブランチ>`）、取り込んだ先端の SHA を控える。AI の作業場所の clone では git を実行しない（[ADR-20261013-001][adr-workspace]）。控えた SHA が、作業場所を作ったときの SHA と同じ（新しい commit が無い）なら、push しない | 取り込みが objects の検査で失敗したら、`needs-human` を付ける（`workspace-tampered`）。AI が objects を書き換えたおそれがあるため（FACT-18、FACT-57）。作業 AI が `done` を返したのに新しい commit が無ければ、`needs-human` を付ける（`no-new-commit`。全部 `unable` の修正と、直せなかった CI の赤を含む）。印を書かずに止めるのは、`fix` の印を同じ head に書くと次の工程の表のどの行にも当たらず、書かないと毎回同じ修正をやり直すため |
| 3 | 範囲の commit ごとに、差分が触れるパスを、push しないパス（`push.blockedPaths`）と照らす。merge の commit は、どの親とも違う部分（`git show --cc`）を照らす。あわせて、PR の全体の差分（`git diff <base>...<控えた SHA>`）も照らす | 当たれば push せず、`needs-human` を付ける（`blocked-path`）。途中の commit で触れて後の commit で戻しても、途中の commit が GitHub に届くので止める |
| 4 | 漏えいの検査を、範囲の commit ごとの差分（merge の commit は `git show --cc`）、commit ごとのメッセージと作者、PR の題名と本文の案に掛ける。検査は `plugin/src/` の中のものを使い、worktree の中の検査は使わない（[IMPL-15][impl]） | 引っかかれば push せず、`needs-human` を付ける（`leak`）。途中の commit で足して後の commit で消した秘密も、途中の commit が GitHub に届くので止める |
| 5 | 進行 Bot の clone から `git ls-remote` で、作業ブランチの今の先端を読み、控えた SHA の祖先であることを確かめる（fast-forward で届くこと）。作業ブランチがまだ無ければ、作業場所を作ったときに控えた base の SHA で確かめる（[IMPL-15][impl]、[SAFE-13][safety]） | 祖先でなければ push せず、`needs-human` を付ける（`not-fast-forward`）。AI が履歴を書き換えたか、ほかの誰かが push したため |
| 6 | 全体の止める合図を読み直し、書かずに試す合図（`run.dryRun`）を見る（[運用の規則][flow]） | 合図が `run` でなければ push せずに止まる。`dryRun` が `true` なら push せず、送るつもりだった SHA を手元の記録に残す |
| 7 | 進行 Bot の clone から、deploy key で `git push origin <控えた SHA>:refs/heads/<型>/issue-<番号>` を送る。ブランチ名でなく SHA を送り、force push はしない（[IMPL-15][impl]） | 断られたら（ruleset など）、実行の失敗として次の実行に持ち越す |
| 8 | `git ls-remote` で、作業ブランチの先端が送った SHA になったことを読み戻す。初回は PR を draft で作り、すぐに結びつきの印（`note kind=link`）を Issue と PR に書く（[IMPL-11][impl]、[IMPL-6][impl]）。題名は、順 4 で検査した案を、規約（`conventions.*`。Conventional Commits の形と言語）に照らしてから使う。外れていれば、Issue の題名から固定の形（`<作業ブランチの型>: <Issue の題名>`）で作る。本文は、「AI が生成した」の 1 行、`Closes #<Issue の番号>`、作業 AI が確かめた検査の結果（出力契約の `checks`）、本文の案の順に並べる | 読み戻した SHA が違えば、`needs-human` を付ける（`unrecoverable`）。PR を作れなければ、実行の失敗として次の実行に持ち越す（次の実行は順 13 で、PR の作成から続ける） |

- 載せ直し（[MERGE-6][merge]）は、AI のセッションを起こさずに、進行 Bot の clone で base ブランチを作業ブランチに取り込み（`git merge`）、順 3〜8 を同じく行う。衝突したら取り込みをやめ、修正のセッション（作業 AI の衝突の解消）で解く（レビュー工程（段階 6）では人の番にする）。差分の patch-id を比べて、印を付け直すか再レビューに回すかは、上の「作業中」の順 18
- 進行 Bot の clone の git は、ホームの個人設定とシステムの設定を読まず（`GIT_CONFIG_GLOBAL=/dev/null`、`GIT_CONFIG_NOSYSTEM=1`）、`core.hooksPath` を空のフォルダにし、署名しない（`commit.gpgsign=false`）で動かす（[SAFE-16][safety]）。載せ直しの merge の commit の作者は、投稿に使うアカウントの名前と、GitHub の noreply のアドレスにする（作業 AI の commit と同じ。「[AI のセッション][design-ai]」の「環境変数」）。手元の Mac で、人の鍵で署名した commit を送らないため
- push した実行では、push した Issue を次の実行まで進めない（上の「次の工程」）
- 順 2 で控えた SHA を、順 3〜8 のすべてで使う。途中で作業ブランチを読み直さないので、検査した commit と送る commit が同じになる

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
- `merge=ready` にするのは、1 回の実行で 1 件まで。2 件目以降の、`merge=ready` になるはずの PR は、判定を次の実行に回し、印を書かない。次の実行でも、1 件目が開いていてマージされていなければ、2 件目以降の `merge=ready` は書かない。`merge=human` の判定は 1 件の上限に数えず、書く。1 件目が閉じられたら、待つのをやめる（[MERGE-5][merge]、[ADR-20261016-001][adr-pr] の決定 2）
- 修正工程（段階 7）は、判定を手元の記録にだけ残し、印を書かない。マージ判定工程（段階 8）から印を書く（[FLOW-24][flow]）。計画・実装工程（段階 5）とレビュー工程（段階 6）では判定しない

### 同じ指摘の再発の判定

![同じ指摘の再発の判定][fig-repeat]

修正の後のラウンドで、前のラウンドと同じ指摘がもう一度出たかを、`review` の印の `findings` で見分ける（[ADR-20261019-001][adr-repeat]）。

- 進行 Bot は、`review` の印を書く前に、批評 AI の統合の後に残った 🔴 🟡 の指摘ごとに ID を計算し、`findings` に書く。ID は「軸」「正規化した見出し」「ファイルのパス」を改行でつないだ文字列の SHA-256 の先頭 8 文字（16 進）。正規化は、Unicode の NFKC にそろえ、先頭の重要度の記号と空白を除き、英字を小文字にする（[ADR-20261019-001][adr-repeat] の決定 1）
- 新しいラウンドの `findings` の ID が、前のどれかのラウンドの `findings` にもあれば、同じ指摘の再発とする。間が空いていても数える。同じラウンドの中の重複は数えない（[ADR-20261019-001][adr-repeat] の決定 2）
- 同じ指摘の再発なら、往復の上限と同じく、修正せずに Status を `Blocked` にして `needs-human` を付け（`repeated-finding`）、再び出た指摘の見出しと、出たラウンドを Issue にコメントする（[IMPL-24][impl]、[FLOW-28][flow]）

| ラウンド | `findings`（例） | 判定 |
| --- | --- | --- |
| 1 | A、B | 前のラウンドが無い。修正に進む |
| 2 | C | A も B も出ていない。修正に進む |
| 3 | A | A はラウンド 1 にも出ている。同じ指摘の再発として `needs-human` にする |

## 印の種類

UNDECIDED-39（進行 Bot のほかのコメントに付ける印の種類）を決め、`stop` の印の理由の記号をそろえる。
印の形と欄の正本は[印と固定の見出しの一覧][markers]で、基本設計は、どのコメントにどの印を付けるかと、記号の一覧を決める。進行 Bot が投稿するコメントは、すべて署名付きの印を 1 つ持つ（[印と固定の見出しの一覧][markers]）。

### 印の署名

印の `sig`（[IMPL-20][impl] の (c)）は、HMAC-SHA256 で作る。

- 鍵：秘密の置き場の `marker-signing-key`（32 バイトの乱数を base64 で書いたもの。[設定][design-settings]の「秘密の置き場のファイル」）。人が作り（`setup` が作り方のコマンドを示す。例：`openssl rand -base64 32`）、進行 Bot だけが読む
- 署名する文字列：1 行目に印の種類、2 行目にリポ（`owner/repo`）、3 行目に置き場の Issue か PR の番号、4 行目から `sig` 以外の欄を `<欄>=<値>` の形で欄の名前の順（Unicode の符号の順）に 1 行ずつ並べ、改行（LF）でつなぐ（種類・リポ・番号は欄として書かず、署名にだけ入れる。[ADR-20261015-004][adr-markers] の決定 1）
- `sig` の値：HMAC-SHA256 の結果の 16 進の 64 文字
- 鍵を替えると、前の鍵で書いた印が読めなくなる。替えるときは、進行中の Issue が無いときを選ぶ

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
| `raise` | 作業 AI のモデルを `sonnet` から `opus` に上げたとき（[IMPL-25][impl]） | Issue | `from=<別名>`、`to=<別名>` | 同じ `kind` の有効な印があれば書かない（1 つの Issue で上げるのは 1 回） |
| `duplicate` | 🟢 を Issue にするときに、同じ見出しの開いている Issue があり、新しく作らずに PR 番号を足すとき（[IMPL-26][impl]） | すでにある 🟢 の Issue | `pr=<PR の番号>` | 同じ番号の有効な印があれば書かない |

- `raise` と `duplicate` は、要件の印の一覧の種類（`wait` `overlap` `remind` `link` `stop`）に寄せず、基本設計で足した。`raise` を `stop` に寄せると止まったと読まれる。`duplicate` を `link` に寄せると、PR と Issue の結びつき（[IMPL-6][impl] の判定に使う）を誤る
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
| `leak` | 漏えいの検査に引っかかった（push の前、計画を投稿する前など。[IMPL-15][impl]） |
| `blocked-path` | diff が、push しないパスに触れている（[IMPL-15][impl]） |
| `not-fast-forward` | push する SHA が、GitHub の作業ブランチの今の先端（まだ無ければ、作業場所を作ったときに控えた base の SHA）の子孫でない（[IMPL-15][impl]、[SAFE-13][safety]、上の「push の手順」） |
| `no-new-commit` | 作業 AI が作業を終えた（`done`）と返したのに、新しい commit が無い（実装・修正・衝突の解消。上の「push の手順」の順 2） |
| `plan-blocked-path` | 計画が変える見込みのパスが、push しないパスに当たる（[IMPL-10][impl]、[ADR-20261019-004][adr-policy]） |
| `approved-before-intake` | 受け入れの下書きの前に、`agent-ready` が付いた（[TEXT-12][text]。人が下書きを読む前の承認を通さない） |
| `brief-changed` | 渡す文のハッシュが、承認したときと違う（[TEXT-4][text]） |
| `brief-edited-after-approval` | 渡す文（作業指示コメントか、直した下書き）の投稿か最後の編集が、`agent-ready` を付けた時刻より後（[TEXT-10][text]） |
| `brief-sensitive` | 渡す文に、事業データか認証情報の形が混じっている（承認の確定で確かめる。[TEXT-7][text]） |
| `quarantine` | 受け入れの検疫で見つかった。受付 AI が返した文の欄で見つかった。見つかった Issue に `## 作業指示` のコメントが無いまま `agent-ready` が付いた（[TEXT-12][text]）。承認の確定か、AI に渡す直前か、計画を投稿する前の検疫で見つかった（[TEXT-11][text]、[AI のセッション][design-ai]の「起動の前の確かめ」） |
| `issue-changed` | 作業中に、Issue が閉じられた・移された・`agent-ready` かマイルストーンが外された（[TEXT-8][text]） |
| `permission` | `agent-ready` を付けた人か、下書きのコメントか読む作業指示コメントを編集した人が push 未満（[TEXT-10][text]、[FLOW-29][flow]）。push 未満の人が書いた作業指示コメントは読まないので、止めない |
| `unauthorized-resume` | push 未満の人が `needs-human` を外した（[TEXT-10][text]） |
| `unlinked-pr` | ブランチ名が一致するのに、結びつきの印が無い PR がある（[IMPL-6][impl]） |
| `pr-closed` | 結びつく PR が、マージされずに閉じられた（[IMPL-6][impl]） |
| `marker-invalid` | 進行 Bot の印が、未編集（`intake` の印には掛けない）か署名の条件を満たさない。または `review` の印の `round` が欠けている（[IMPL-20、IMPL-18][impl]） |
| `ci-action-required` | CI のワークフローの実行に、人の承認が要る（[IMPL-27][impl]） |
| `ci-stale` | CI の結果が、取り消しなどのまま時間を過ぎても変わらない（[IMPL-27][impl]） |
| `workspace-residue` | Issue に紐づく作業場所の残骸がある（下の「作業場所の残骸」） |
| `workspace-tampered` | 作業場所の `commondir` か `gitdir` が、作ったときの値と違う。または、取り込みの fetch が objects の検査で失敗した（[SAFE-12][safety]、[ADR-20261013-001][adr-workspace]、上の「push の手順」） |
| `local-settings` | AI のセッションの作業場所の worktree に `.claude/settings.local.json` がある（FACT-56、[SAFE-2][safety] の (a)、[AI のセッション][design-ai]の「起動の前の確かめ」） |
| `draft-format` | 承認の確定で、受け入れの下書きの依存とマイルストーンの候補の 2 行を決まった形で読めない（[AI のセッション][design-ai]の「コメントの形」） |
| `multiple-instructions` | push 以上の人の作業指示コメントが 2 つ以上ある（承認の確定、着手、対応ループの各回。[TEXT-12][text]、[ADR-20261021-003][adr-reapprove]） |
| `settings-tampered` | セッションの安全設定のファイルが、作ったときのハッシュと違う（起動の直前か、セッションの後。[SAFE-2][safety] の (e)、[AI のセッション][design-ai]の「セッションごとの安全設定」） |
| `records-missing` | 着手した Issue の手元の記録が無い（[運用の規則][flow]。見分け方は「[AI のセッション][design-ai]」の「起動の前の確かめ」） |
| `unrecoverable` | GitHub の状態から、次の工程を決められない（次の工程の表の順 21。[運用の規則][flow]）。または、push した後に読み戻した作業ブランチの先端が違う |
| `unexpected-status` | Status が、印と PR から決まる値と違い、人が戻す手順にも当たらない（[SAFE-4][safety]） |

## 作業場所の残骸

UNDECIDED-27（作業場所の残骸の決め方）を決める。セッションのフォルダ（`<置き場>/sessions/<セッションの名前>/`）は、同じ名前で作り直すか片付けるまで残るので、作業中の Issue のフォルダがあるのは普通の状態（[ADR-20261013-001][adr-workspace]）。
セッションの名前は、役と、Issue か PR の番号を略さずに書く（例：`work-issue-<番号>`、`review-pr-<番号>-round-<ラウンド>-<軸>`。役ごとの名前の一覧は「[AI のセッション][design-ai]」の「セッションの名前」）。進行 Bot は、名前から役と番号を読み、次の表で見分ける。

![作業場所の残骸の見分け方][fig-residue]

| フォルダ | 扱い |
| --- | --- |
| 名前から Issue か PR を引け、その Issue が作業中（閉じておらず、結びつく PR がマージも閉じもされていない） | 普通の状態。作り直すか片付けるまで残す（[ADR-20261013-001][adr-workspace]） |
| 名前から引けた PR（Issue の名前のフォルダは、結びつく PR）がマージされた | 片付け（次の工程の順 1）で消す。片付けが途中で止まって残ったものも、次の実行の片付けで消す（[FLOW-27][flow]、[MERGE-7][merge]） |
| 名前から引けた Issue が、PR がマージされないまま閉じられたか、結びつく PR がマージされずに閉じられた | Issue に紐づく残骸。消さずに Issue を `needs-human` にし（`workspace-residue`）、人が中身を確かめてから消す（[FLOW-29][flow]） |
| 消す前に読んだ `commondir` か `gitdir` が、作ったときの値（作業場所のパスから決まる値）と違う | 消さずに Issue を `needs-human` にし（`workspace-tampered`）、`check` で warn にする（[ADR-20261013-001][adr-workspace]、[SAFE-12][safety]） |
| 短い一時フォルダの根（`sessionTmp.root`）の下にあり、実行の記録にセッションの名前との組が無い | Issue に紐づかない残骸。点検を fail にし、実行は作業に入らずに止まる。人が中身を確かめて消す（[ADR-20261017-006][adr-tmp]、[AI のセッション][design-ai]の「短い一時フォルダ」） |
| 名前から Issue か PR を引けない。または、引けた Issue（PR の名前のフォルダは、結びつきの印から引いた Issue）が設定した Project に無い | Issue に紐づかない残骸。点検を fail にし、実行は作業に入らずに止まる。人が中身を確かめて消す（[運用の規則][flow]、[SETUP-2][setup]、[SAFE-4][safety]） |

進行 Bot が、名前から引けない残骸を消さないのは、何のフォルダか分からないものを消すと、人が調べる材料を失うため。作業に入らないのは、判定できない状態で作業を増やさないため（[SAFE-4][safety]）。

## 手元の記録

![手元の記録][fig-records]

進行 Bot は、記録の置き場（`paths.records`。[設定][design-settings]の「マシンの中のフォルダ」）に、実行と AI のセッションの記録を残す（[FLOW-5][flow]）。記録は、判定に使わず、上限の数え上げと、段階の出口の数え上げと、人が読むためにだけ使う（[ADR-20261015-006][adr-flow] の決定 4）。例外は、書かずに試す間の受け入れ（上の「1 回の実行の流れ」。[ADR-20261021-002][adr-dry]）。AI は sandbox の中から読めず、書けない。

### 置くもの

| ファイル | 1 件 | 欄 |
| --- | --- | --- |
| `runs.jsonl` | 1 回の実行（lock で見送った実行も） | 実行 ID、起こされ方（定期・手動・点検の依頼）、起動と終わりの時刻、終わり方（作業した・点検だけ・合図で止めた・点検の fail で止めた・lock で見送った）、段階（`run.stage`）、`run.dryRun`、リポの設定を読んだ base ブランチの SHA、点検の fail と warn、基線の結果（緑でない必須チェックの名前）、起動しなかった定期の実行の時刻、Issue ごとの結果（進めた工程、待たせた理由か止めた理由（stop の理由の記号）、マージの判定と理由の記号）、使った時間 |
| `sessions.jsonl` | AI のセッション 1 回につき、始まりの行と終わりの行の 2 行（`kind` の欄が `start` か `end`） | 始まりの行（起動の前に書く）：実行 ID、`run.dryRun`、セッションの名前、Issue と PR の番号、役、短い一時フォルダの ID、安全設定のハッシュ、起動の時刻。終わりの行：[FLOW-5][flow] の固定の欄（実行 ID、Issue の番号、ラウンド、役、指定したモデルと effort、実際に使われたモデル（`modelUsage`）、費用、ターン数、実行の成否、作業の成否）、終わりの時刻、終わり方と費用の数え方（「[AI のセッション][design-ai]」の「費用の数え方」）、モデルごとのトークン、出力が分からないメッセージがあったか、拒否（`stop_reason: refusal`）で止まったか、終わった後の安全設定のハッシュ。試しのセッションは、Claude Code と写しと OS の版、安全設定と環境変数の名前のハッシュ、試しの結果（pass か fail と項目ごとの結果）も書く。ラウンドの欄は、役ごとに、計画と計画のレビューは計画の工程の中のラウンド、レビューと批評と修正は `review` の印の `round`、作業 AI の実装は 0 |
| `outputs/<実行 ID>/<セッションの名前>.json` | AI のセッションの出力契約 1 つ | 取り出した出力契約をそのまま残す。受け入れの判定、直す前の計画と計画のレビューの結果、計画 AI の自己申告、見落としの軸と比べるためのレビューの結果も、ここから読む |
| `streams/<実行 ID>/<セッションの名前>.jsonl` | AI のセッション 1 回の `stream-json` | 受け取った行をそのまま残す。人のホームに会話の記録を残さないので、会話の流れを読むときはここを読む（「[AI のセッション][design-ai]」の「起動の形」の `--no-session-persistence`） |
| `writes/<実行 ID>.jsonl` | GitHub に書いた 1 件（`run.dryRun` の間は、書くつもりだった 1 件） | 書く先（Issue か PR の番号）、種類（コメント・ラベル・Project の欄・PR・push）、本文か値、実際に書いたか。受け入れの下書きと、受け入れで付ける `needs-human` では、受け入れに使った文（題名・本文・コメント）のハッシュも書く（書かずに試す間に、同じ文の間は書いたものとして扱うため） |
| `dry-run-objections.md` | 人が書く異議 | `## 異議` の見出し（上の「1 回の実行の流れ」） |

- `.jsonl` のファイルは 1 行に 1 件の JSON（JSON Lines）で、後ろに足すだけにし、書き直さない。書くたびに、ファイルを閉じる前にディスクへ書き出す（電源が切れても、前の行を失わないため）。`outputs/` の `.json` は 1 つの出力契約を 1 つのファイルに書き、書き直さない
- `.jsonl` の最後の行が途中で切れていたら（書いている途中で電源が切れた）、切れた行を読まずに点検で warn に出す。途中の行が読めなければ点検を fail にする。数え上げが少なく出て、上限が効かなくなる側に倒さないため（[運用の規則][flow]）
- `dry-run-objections.md` には、`## 異議` の見出しの次の行に、書いた日（`YYYY-MM-DD`）を書く。受け入れ工程（段階 4）の出口で、数える期間に入るものだけを数えるため

### 記録の置き場のほかのファイル

| ファイル | 中身 | 書く人と消す時 |
| --- | --- | --- |
| `run.lock` | 実行の lock。JSON で、PID・ホスト名・起動の時刻 | 進行 Bot と `setup`（プログラムの写しを作り直すとき）。lock を外すときに消す |
| `records-created.json` | 記録の目印のファイル。JSON で、作った時刻と `setup` の版 | `setup`。消さない（無ければ点検が fail にする。[運用の規則][flow]） |
| `check-request.json` | 点検の依頼。JSON で、依頼の ID と時刻 | 人の `check` が置き、進行 Bot が結果を書いた後に消す |
| `check-result-<依頼の ID>.json` | 点検の結果。JSON で、項目ごとの fail・warn・pass | 進行 Bot。人の `check` が表示した後に消す。残ったものは、7 日を過ぎたら進行 Bot が消す |
| `run-request.json` | 手動の 1 回実行の依頼。JSON で、依頼の ID と時刻 | 人の `/dev-autopilot` が置き、進行 Bot が lock を取った後に読んで消す |
| `status.json` | 状態の表示（「[導入と点検][design-setup]」の「状態の表示」） | 進行 Bot が実行の終わりに書き直す |
| `launchd.out.log`、`launchd.err.log` | 定期の実行の出力（macOS） | launchd。大きくなりすぎたら人が消す |

- 実行 ID と依頼の ID は、UTC の時刻と乱数の組（`YYYYMMDDTHHMMSSZ-<16 進の 4 文字>`。例：`20261020T150000Z-3fa9`）。パスに入れても困らない文字だけにする。試しのセッションの名前の `<日時>` も同じ形（`check-20261020T150000Z-3fa9`）

### 段階の出口の数え方

段階の出口と、段階で記録する値（[段階の表][setup]）は、上の記録と GitHub から、数えるときに数える。数えるための欄を別に持たない（[ADR-20261018-003][adr-raise] と同じ考え方）。

| 段階 | 数えるもの | 数える元 |
| --- | --- | --- |
| 受け入れ工程（段階 4） | 実行の回数（起こされ方が定期で、終わり方が「作業した」の実行。lock で見送った実行・起動しなかった定期の実行・点検だけの実行・合図で止めた実行は数えない）、投稿ありの実行 | `runs.jsonl`、`writes/` |
| 受け入れ工程（段階 4） | 人が `## 異議` の見出しで書いた回数 | Issue のコメントと `dry-run-objections.md` |
| 受け入れ工程（段階 4） | 方針に係る要望の判定と、人が `agent-ready` を付けたか（[ADR-20261019-004][adr-policy]） | `outputs/` の受付 AI、Issue の timeline |
| 受け入れ工程（段階 4） | 受け入れた件数と、マイルストーンの候補を人が直した件数（[ADR-20261015-003][adr-intake] の確認方法）。人が下書きを直した件数を、コラボレータと外の人の Issue に分けて（[ADR-20261016-002][adr-draft] の確認方法） | `writes/` の下書き、Issue のコメントの編集の記録（`userContentEdits`） |
| 計画・実装工程（段階 5） | 人が手直しせずにマージした PR（手直しは、人が PR に commit を足したか、close して作り直したこと） | PR の commit の作者、閉じた PR |
| 計画・実装工程（段階 5） | `plan=` の判定への異議、`fable` にした計画と `opus` の計画の違い、自己申告で捨てた計画 | `outputs/` の受付 AI と計画 AI、Issue のコメント |
| 計画・実装工程（段階 5）から修正工程（段階 7） | 計画のレビューの結果と、計画し直したこと | `outputs/` の計画のレビューと、直す前の計画 |
| 計画・実装工程（段階 5）から修正工程（段階 7） | 作業 AI の最初の実装のモデルと、Issue の種類（作業ブランチの型）ごとの結果 | `sessions.jsonl` の作業 AI、`review` の印、PR |
| レビュー工程（段階 6） | 一致率と却下率、`opus` と `fable` の比べ、見落としの軸の 2 つの数（人の 🔴 🟡 のうち、3 つの軸が見落として見落としの軸が見つけたものの数と、見落としの軸だけが出して人が出さなかった 🔴 🟡 の数。[段階の表][setup]） | `outputs/` のレビュー AI（3 つの軸、比べるための実行、見落としの軸）、人のレビューの結果、`## 却下` のコメント |
| 修正工程（段階 7） | 往復の回数、`opus` に上げた件数、上げた後に Ready to Merge まで進んだ件数 | `review` の印、`sessions.jsonl` の作業 AI、Project の Status |
| 修正工程（段階 7）とマージ判定工程（段階 8） | マージの判定と人の操作が合ったか。修正工程（段階 7）は印を書かず毎回の実行で判定が重なるので、PR と head ごとに最後の判定を 1 件として数える | `runs.jsonl` のマージの判定、`merge` の印、PR のマージと「判定と違う判断」のコメント |
| マージ判定工程（段階 8） | revert（マージから 14 日以内に、`develop` にその squash の commit を revert する commit か、本文でその PR の番号を「戻す」と言及する fix の PR がマージされたもの） | base ブランチの commit の履歴（進行 Bot の clone）、マージされた PR の本文 |
| 並列化（段階 9） | 連続 7 回の実行で `needs-human` が 0 件か、利用枠が上限の内か（[FLOW-12][flow]） | `runs.jsonl`（止めた理由）、`sessions.jsonl`（費用） |

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
[adr-draft]: ../../adr/adr-20261016-002-intake-ai-drafts-every-issue.md
[adr-markers]: ../../adr/adr-20261015-004-signed-markers-and-actor-checks.md
[adr-dry]: ../../adr/adr-20261021-002-dry-run-only-at-intake-stage.md
[adr-reapprove]: ../../adr/adr-20261021-003-reapprove-by-relabel-and-ignore-outside-instructions.md
