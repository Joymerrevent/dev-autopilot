# 1 回の実行の流れ

進行 Bot の 1 回の実行（順 1〜10）、GitHub の事実の読み方、状態の組み立て、次の工程、工程、push の手順、判定、全体の停止と通知の処理の作りと、人が呼ぶ `/dev-autopilot` の処理を決める。
順・状態・工程・判定・印・止めた理由の正本は、基本設計の「[状態遷移と判定][b02]」にある。本書は、正本の表を並べ直さず、ファイルの分け方、1 回の実行の中で持つ値、終わり方と終了コード、GitHub の事実を読む形、状態と工程の記号、作業の単位の見直しの止め方、GitHub に書くときの決まり、実行の失敗の数え方、実装の都合で止める理由の記号の扱いを決める。

## ファイルの一覧

| ファイル | 層 | 受け持つこと | 基本設計 |
| --- | --- | --- | --- |
| `main.ts` | 入口 | 引数（マシンの設定のパス）を読み、システムを作り、外との境を組み立てて `flow/run/run.ts` を呼び、返った終わり方の終了コードで終える | 「[状態遷移と判定][b02]」の「1 回の実行の流れ」 |
| `flow/run/run.ts` | 流れ | 順 1〜10 を、起こされ方ごとの順で呼ぶ。思わない失敗で止まったときの後始末 | 同上 |
| `flow/run/clone.ts` | 流れ | 進行 Bot の clone の fetch（順 4）と、取り込みの前の後始末 | 同上、「push の手順」の表の下 |
| `flow/run/gather.ts` | 流れ | 順 6・7 の GitHub の読み取り。基線の必須チェックと、作業の単位ごとの事実を作る | 「状態の組み立て」 |
| `flow/run/units.ts` | 流れ | 順 8・9。作業の単位を組の順に進め、作業の単位ごとに見直しを繰り返す | 「作業の単位を進める順」「次の工程」 |
| `flow/run/write.ts` | 流れ | GitHub に書くときの共通の処理（合図の読み直し、印のコメントの組み立て、記録） | 「1 回の実行の流れ」の表の下 |
| `flow/run/intake.ts` | 流れ | 受け入れの依頼の実行の、受付 AI を起こして結果を書く処理 | 「[AI のセッション][b03]」の「人の Skill から受け入れを呼ぶ形」の順 5〜7 |
| `flow/steps/brief.ts` | 流れ | 依頼の枠の工程（入口の検疫、下書き、書き直し、人の直しの検疫、無効な `agent-ready` を外す、`ignored-instruction`） | 「状態ごとの工程」 |
| `flow/steps/decide.ts` | 流れ | 決定済みの工程（決定の印、着手） | 同上 |
| `flow/steps/plan.ts` | 流れ | 計画の輪（順 1〜6） | 「計画の工程の中の順番」 |
| `flow/steps/change.ts` | 流れ | 作業 AI を起こす工程（変更の作成、検査の失敗の直し、指摘の直し、衝突の直し）と、人が push した作業ブランチから PR を作る工程 | 「状態ごとの工程」 |
| `flow/steps/review.ts` | 流れ | レビューと、載せ直しの後に印を付け直す工程 | 同上の検査合格 |
| `flow/steps/passed.ts` | 流れ | レビュー合格の (1)〜(5)（🟢 の起票、ready、載せ直し、マージの判定、催促） | 同上のレビュー合格 |
| `flow/steps/cleanup.ts` | 流れ | 片付けと、戻した後に前の PR を閉じる工程（「先に見る」の順 9） | 「状態ごとの工程」、「作業場所の残骸」 |
| `flow/steps/dependency.ts` | 流れ | 先の依頼の `wait` の印と、計画の見直し | 「先に見る」の順 13・14 |
| `flow/steps/stop.ts` | 流れ | 停止中にする（止めた先ごとの `stop` の印と、止めた理由のコメント） | 「先に見る」の表の下 |
| `flow/steps/display.ts` | 流れ | Project の表示の欄と、表示のラベル `security-suspected` | 「1 回の実行の流れ」の順 8 |
| `flow/push.ts` | 流れ | push の手順の順 1〜8 | 「push の手順」 |
| `flow/safety/signal.ts` | 流れ | 全体の止める合図と、全体の停止の Issue を読む | 「全体の停止の Issue」 |
| `flow/safety/suspect.ts` | 流れ | 攻撃の疑いにする（`security` の印。`where` が `text` なら全体の停止の Issue を開き、通知する） | 同上、「GitHub の外への通知」 |
| `flow/safety/halt.ts` | 流れ | 全体の停止の Issue を開くか `halt` の印を足し、動いているセッションを止める | 「全体の停止の Issue」 |
| `flow/safety/notify.ts` | 流れ | GitHub の外への通知と送り直し | 「GitHub の外への通知」 |
| `logic/state/states.ts` | 判断 | 主な流れの状態と軸の値の記号、Project の表示の名前（[FLOW-39][flow]）と主な流れの枠 | 「作業の単位の状態遷移」 |
| `logic/state/facts.ts` | 判断 | 作業の単位の事実の型（下の「GitHub の事実の読み方」で作るもの） | 「状態の組み立て」 |
| `logic/state/discount.ts` | 判断 | 戻した後の読み方と、数えない無効な印（組み立ての 1） | 同上の「戻した後の読み方」「数えない無効な印」 |
| `logic/state/main-flow.ts` | 判断 | 主な流れの状態と、依頼の中の状態（組み立ての 2・3） | 同上の「主な流れの状態の決め方」「依頼の中の状態の決め方」 |
| `logic/state/axes.ts` | 判断 | 重ねる 3 本の軸（組み立ての 4） | 同上の「重ねる軸の決め方」 |
| `logic/state/assemble.ts` | 判断 | 組み立ての 1〜4 を順に呼び、作業の単位の状態を返す | 同上 |
| `logic/state/next.ts` | 判断 | 次の工程（「先に見る」と「状態ごとの工程」の表） | 「次の工程」 |
| `logic/state/order.ts` | 判断 | 作業の単位を進める順、進行中の作業の単位の数え方、着手の条件 | 「作業の単位を進める順」「進行中の作業の単位の数え方」 |
| `logic/state/apply.ts` | 判断 | 工程で書いた印と GitHub の変化を、事実に足す | 「作業の単位を進める順」の表の下 |
| `logic/state/halt.ts` | 判断 | 全体の停止の Issue の見分けと、止める間か | 「全体の停止の Issue」 |
| `logic/state/reasons.ts` | 判断 | 止めた理由の記号の一覧（下の「止めた理由の記号」） | 「止めた理由の記号」 |
| `logic/judge/checks.ts` | 判断 | 必須チェックの読み方（3 つの一致、最新の 1 件、結果ごとの扱い） | 「主な流れの状態の決め方」の表の下、[IMPL-27][impl] |
| `logic/judge/rounds.ts` | 判断 | ラウンドの欠けと、往復の回数 | 「ラウンドと往復の数え方」 |
| `logic/judge/convergence.ts` | 判断 | レビュー合格の判定 | 「レビュー合格の判定」 |
| `logic/judge/repeat.ts` | 判断 | 同じ指摘の再発の判定（head の形の見分けを含む） | 「同じ指摘の再発の判定」 |
| `logic/judge/merge.ts` | 判断 | マージの判定と、`reasons` の記号の一覧 | 「マージの判定」 |
| `logic/push/paths.ts` | 判断 | push の範囲の commit が触れたパスを、変えてはいけないパスの 2 つの組と照らし、止める理由を決める | 「push の手順」の順 3 |
| `logic/push/overlap.ts` | 判断 | 変更ファイルの一覧の重なり（`overlap` の印） | 「状態ごとの工程」の表の下 |
| `logic/run/wake.ts` | 判断 | 起こされ方の決定と、依頼のファイルごとの扱い | 「[導入と点検][b04]」の「見送った実行と、起動しなかった定期の実行」 |
| `logic/run/missed.ts` | 判断 | 起動しなかった定期の実行の時刻 | 同上 |
| `logic/run/failures.ts` | 判断 | 実行の失敗と片付けの失敗が続いた回数と、重い工程を起こしてよいか | 「[AI のセッション][b03]」の「起動の前の確かめ」の表の下 |
| `logic/run/residue.ts` | 判断 | 作業場所の残骸の見分け（フォルダごとの扱いと扱う所） | 「作業場所の残骸」 |
| `entry/dev-autopilot.ts` | 入口 | `/dev-autopilot` の入口（引数「表示」と「1 回実行」） | 「[導入と点検][b04]」の「状態の表示」 |
| `flow/skill/dev-autopilot.ts` | 流れ | `/dev-autopilot` の処理 | 同上 |

- 「基本設計」の欄の節は、ことわりの無いものは「[状態遷移と判定][b02]」の節
- 工程のファイルは、状態の組み立ても次の工程の選び方も持たない。`logic/state/next.ts` が選んだ工程を行い、結果（下の「工程の結果」）を返すだけにする。工程を選ぶ規則を 1 か所に置き、単体テストで表の行ごとに確かめるため
- 判断の層のファイルは、時刻を引数で受ける。ラベルの付き外しと印の新しさを比べる判定が多く、組み合わせごとに単体テストで確かめるため
- 運用の記録の数え方（「[状態遷移と判定][b02]」の「運用の記録の数え方」）を数える処理は、今は作らない。今の入口（`main.ts`・`setup`・`check`・`/dev-autopilot`・intake）のどれも数えた値を使わないため。数えるための欄は記録に別に持たない決まりなので、数える入口を足すときに、記録と GitHub から数える処理を足す

## 1 回の実行の中で持つ値

`flow/run/run.ts` は、実行の最初に、実行の間の値（`RunContext`）を 1 つ作り、流れの層の処理に引数で渡す。

| 値 | 中身 | 使う所 |
| --- | --- | --- |
| 実行 ID と起こされ方 | `logic/run/ids.ts`（「[設定と記録][d02]」）と `logic/run/wake.ts` で決める | 記録、`status.json` |
| 設定 | 凍らせたマシンの設定とリポの設定と、リポの設定を読んだ base ブランチの SHA（「[設定と記録][d02]」の「設定を読む順」） | すべて |
| 期限 | 実行を始めた時刻に `run.maxMinutes` を足した時刻 | 順 9 |
| 費用の帳簿 | 「[AI のセッション][d07]」の「費用の取り置き」 | AI のセッション |
| 合図 | 最後に読んだ全体の止める合図と、全体の停止の Issue の有無（読んだ時刻と値） | 書き込み、重い工程 |
| 数え上げ | 着手した件数、下書きと書き直しの件数、`ready` の判定を書いた PR | `limits.startsPerRun`、`limits.intakePerRun`、[MERGE-5][merge] |
| 動いているセッション | 起こして、まだ終わりの行を書いていないセッション | 全体の停止の Issue を開いたときと、思わない失敗で止まったとき |
| 止められなかったセッションの作業の単位 | 順 2 で、前の実行が残したセッションを止められなかった作業の単位 | 順 8（次の工程の表より先に `session-not-stopped` にする） |
| 作業の単位の結果 | 作業の単位ごとの、進めた工程・結果・止めた理由・外れた着手の条件 | `runs.jsonl`、`status.json` |

- 値は 1 つの進行 Bot のプロセスの中にだけ持つ。実行の lock で、同じマシンで同じリポの実行は 1 つだけなので、ファイルで守る要は無い（「[AI のセッション][d07]」の「費用の取り置き」と同じ）
- 合図と全体の停止の Issue の値は、読み直すたびに入れ替える。全体の停止の Issue を開いたときは、読み直さずに「ある」に入れ替え、開いた後の重い工程を起こさない（「[状態遷移と判定][b02]」の「全体の停止の Issue」）

## 起こされ方と進む順

![1 回の実行の順と受け持つファイル][fig-run]

`logic/run/wake.ts` は、lock を取った後に読んだ 3 つの依頼のファイル（形が違うものは無いものとする。「[設定と記録][d02]」の「依頼と結果のファイル」）と今の時刻から、起こされ方と、依頼のファイルごとの扱い（残す・消す・見送って消す）を返す。決め方は「[導入と点検][b04]」の「見送った実行と、起動しなかった定期の実行」のとおり。`flow/run/run.ts` は、起こされ方で、次の順に進む。

| 順 | 受け持つファイル | 定期・手動 | 点検の依頼（待つ時間の内） | 受け入れの依頼 |
| --- | --- | --- | --- | --- |
| 1 | `flow/run/settings.ts`（マシンの設定） | 行う | 行う | 行う |
| 2 | `flow/run/lock.ts`、`logic/run/wake.ts`、`flow/session/leftovers.ts` | 行う | 行う | 行う |
| 3 | `flow/safety/signal.ts` | 行う | 行う | 行う |
| 4 | `flow/run/clone.ts`、`flow/run/settings.ts`（リポの設定） | 行う | 行う | 行う |
| 5 | `check.ts`（「[点検][d04]」） | 行う | 行う | 行う |
| 6・7 | `flow/run/gather.ts`、`logic/state/assemble.ts` | 行う | 行わない | 行わない |
| 8・9 | `flow/run/units.ts` | 行う | 行わない | 受付 AI を 1 つ起こす（`flow/run/intake.ts`） |
| 10 | `flow/run/requests.ts`、`flow/run/records.ts`、`flow/run/status.ts`、`flow/run/lock.ts` | 行う | 点検の結果を書く | 受け入れの結果を書く |

- 待つ時間を過ぎた点検の依頼は、順 5 の結果を書いて依頼のファイルを消してから、定期の実行として順 6 へ進む。待つ時間を過ぎた受け入れの依頼は、受付 AI を起こさずに消す（「[導入と点検][b04]」の「見送った実行と、起動しなかった定期の実行」）
- 順 3〜5 で止まるときも、順 10 を行う。順 10 は、点検の依頼のファイルを読み直し、あれば順 3 の合図を読み直して順 5 だけを行う（「[状態遷移と判定][b02]」の「1 回の実行の流れ」の順 10）
- 順 2〜4 の子プロセスと GitHub の呼び出しの時間の上限は、リポの設定を読むまで、プログラムに固定した 10 分と 10 秒を使う（「[設定と記録][d02]」の「プログラムに固定する値」）
- 起動しなかった定期の実行は、順 2 の後に、`logic/run/missed.ts` が `runs.jsonl` の行と、定期の実行の時刻（「[導入と点検][b04]」の「時刻」）と、今の時刻から決め、順 10 で記録に書く

## 終わり方と終了コード

`runs.jsonl` の終わり方（「[状態遷移と判定][b02]」の「手元の記録」）を、次の記号で書く。終わり方の記号と終了コードの対応は、`runs.jsonl` の行の形と一緒に `logic/run/records.ts`（「[設定と記録][d02]」の「手元の記録」）に置き、`main.ts` は、`flow/run/run.ts` が返した終わり方から、対応の終了コードで終える。

| 終わり方 | 記号 | 終了コード |
| --- | --- | --- |
| 作業した | `worked` | 0 |
| 点検だけ | `check-only` | 0 |
| 受け入れの依頼だけ | `intake-only` | 0 |
| 合図で止めた | `stop-signal` | 0 |
| lock で見送った | `lock-skipped` | 0 |
| 設定を読めずに止めた（順 4） | `settings-unreadable` | 1 |
| 点検の fail で止めた | `check-failed` | 1 |
| GitHub を読めずに止めた（順 7） | `github-unreadable` | 1 |
| 思わない失敗で止めた | `crashed` | 1 |

- 順 1 でマシンの設定を読めないときは、記録の置き場が分からないので `runs.jsonl` に書かず、標準エラー（定期の実行の `launchd.err.log`）に 1 行だけ書いて、終了コード 1 で終える。行には、読めなかったパスと理由の種類を書き、ファイルの中身を書かない
- 終了コードを 0 と 1 に分けるのは、人が `launchd.err.log` と定期の実行の登録から、手当てが要る終わり方かを見分けるため。合図で止めたときと lock で見送ったときは、決まりどおりの終わり方なので 0 にする
- 思わない失敗（`crashed`）は、基本設計の終わり方に無い。例外で止まったときも、下の順で後始末をし、記録と lock を残したままにしないために足した。`flow/run/run.ts` が例外を受け、動いているセッションを止める手順（「[AI のセッション][d07]」の「止める相手と前の実行の残り」）で止め、`runs.jsonl` の行と `status.json` を書き、lock を外す。止められなかったセッションは終わりの行を書かず、次の実行が「前の実行が残したセッション」で探す。後始末の途中でも例外が出たら、そこで終了コード 1 で終える（lock は、PID のプロセスが無いので、次の実行が古い lock として奪う。「[設定と記録][d02]」の「lock」）

## GitHub の事実の読み方

`flow/run/gather.ts` は、順 6 で基線の必須チェックを、順 7 で作業の単位ごとの事実（`logic/state/facts.ts` の型）を読む。読むものと使い方は「[状態遷移と判定][b02]」の「状態の組み立て」の表のとおりで、本節は読む形を決める。

| 読むもの | 呼び方 |
| --- | --- |
| 設定した Project の項目（当てる先のリポの Issue だけ。表示の欄の今の値も） | GraphQL の操作 `ProjectIssues` |
| 作業の単位の Issue（題名、本文、開閉、作者、マイルストーン、依存の相手、timeline のラベルと閉じた出来事、コメント） | GraphQL の操作 `WorkUnitIssue`。作業の単位ごとに 1 つ |
| 結びつく PR（作業ブランチの名前ごと。開閉、マージ、head の SHA、base、draft、コメント） | GraphQL の操作 `WorkUnitPullRequests`。作業の単位ごとに 1 つ |
| 依存の相手の Issue と、結びつく PR | `WorkUnitIssue` と `WorkUnitPullRequests` を、相手の番号で呼ぶ |
| 別のリポに移された Issue を探す、当てる先のリポの開いている PR | GraphQL の操作 `OpenPullRequests` と、印の Issue の REST（`repository_url`）。読み方は「状態の組み立て」の箇条（FACT-66） |
| 必須チェック（PR の head と、base ブランチの先端） | REST の `commits/{ref}/check-runs` と `actions/runs?head_sha=<SHA>` |
| 人の権限 | REST の `collaborators/{username}/permission`（[TEXT-10][text]） |

- コメントと timeline と PR は、100 件ずつのページで読み、続きは同じ名前の操作に、前のページの終わりの位置（`endCursor`）を渡して読む。途中のページで失敗したら、読めたページだけで判断しない（「[外との境][d01]」の「GitHub の呼び方」）
- 1 つの作業の単位でも読めなければ、読めた作業の単位だけで進めず、順 7 の「API が失敗したら、作業を増やさずに記録して止まる」に当てる（終わり方は `github-unreadable`）。読めない作業の単位を飛ばして進めると、着手の件数や進行中の作業の単位の数を少なく数えるため
- コメントの ID は、GraphQL の `fullDatabaseId`（文字列）を数にして持つ。REST の `id` と同じ値で、印の欄のコメントの ID（「[印][d03]」の「欄の値の形」）と突き合わせられる（[FACT-125][facts]）。`databaseId` は、GraphQL の Int の範囲を超える値を返すので使わない。数にした値が `Number.isSafeInteger` に収まらなければ、読めなかったものとする
- 投稿者とラベルの出来事の人の名前は、REST の `login` の形にそろえて持つ。GraphQL の `__typename` が `Bot` なら、`login` の後ろに `[bot]` を足す（GitHub App の名前は、REST では `<アプリの名前>[bot]`、GraphQL では `[bot]` が無い。[FACT-126][facts]）。`safety.botAccounts` は REST の形で書くので、そろえないと、進行 Bot の印を人のコメントと読み誤る。投稿者が無い（アカウントが消された）ときは、権限の無い人として読む。timeline の出来事の `actor` と、進行 Bot の App のコメントで同じ形になるかは、段階 3 の一度だけの試しで確かめる（「[導入と点検][b04]」の「GitHub App の権限」）
- 未編集かは、GraphQL の `lastEditedAt` が無いことで読む（[FACT-16][facts]）。人の見出しの最後に編集した人は `editor` で読み、名前を上と同じにそろえる
- 人の権限は、事実を読むときに 1 人ずつ引く。1 回の実行の中では同じ人を 1 回だけ引いて同じ実行の組み立てにだけ使い、実行をまたいで持たない。コラボレータの一覧は取らない（[TEXT-10][text]）。進行 Bot のアカウントの出来事（進行 Bot が外したラベル）は、権限を引かずに、行ったものとして読む
- コメントの印は、`logic/marker/read.ts` に通した結果（有効・無効・読まない。「[印][d03]」の「読み方」）を事実に入れる。印の中身の意味（数えるか、止める契機か）は、`logic/state/discount.ts` が決める
- 必須チェックは、チェックの実行にワークフローのパスが無いので、ワークフローの実行の `check_suite_id` とチェックの実行の `check_suite.id` を突き合わせてパスを付ける（[FACT-127][facts]）。GitHub Actions のほかの app のチェックは、パスを持たない（設定の `workflow` が `null` のもの。「[設定][b01]」の `checks.required`）
- 印の Issue を REST で引いたときに返る `301` は、`@octokit/core` がたどるかに依らず、返った Issue の `repository_url` で、当てる先のリポにあるかを見分ける（「[状態遷移と判定][b02]」の「状態の組み立て」）

## 状態と軸の記号

`logic/state/states.ts` に、主な流れの 17 の状態と、重ねる 3 本の軸の値の記号を持つ。記号は `status.json` と記録に書き、Project の表示の欄には、同じ表の [FLOW-39][flow] の名前で書く。

| 枠 | 状態 | 記号 |
| --- | --- | --- |
| 依頼 | 起票済み・検疫済み・差し戻し済み・直し済み・下書き済み | `filed`・`quarantined`・`returned`・`corrected`・`drafted` |
| 計画 | 決定済み・計画済み | `decided`・`planned` |
| 変更 | 変更済み・検査合格・検査不合格・レビュー不合格・レビュー合格・衝突あり | `changed`・`checks-passed`・`checks-failed`・`review-failed`・`review-passed`・`conflicted` |
| 終了 | マージ済み・閉鎖済み・完了・中止 | `merged`・`closed`・`done`・`cancelled` |

| 軸 | 値 | 記号 |
| --- | --- | --- |
| 止め方 | 停止中・引き取り中・一時停止中 | `stopped`・`manual`・`paused` |
| 先の依頼 | 未完了・中止あり | `unfinished`・`cancelled` |
| 安全 | 攻撃の疑い（AI の文）・攻撃の疑い（入口） | `ai-text`・`entry` |

- 状態の記号は、状態の名前と同じく、済ませた事実で付ける（次に動く担い手を名前に入れない）
- 軸の値の（なし）は、記号を持たず、値が無いことで表す。Project の表示の欄も、選択肢を選ばずに空にする（「[設定][b01]」の `project.fields.*`）
- `logic/state/assemble.ts` は、状態を決められない（読んだ事実の組み合わせが、どの表のどの行にも当たらない）ときに、状態の代わりに「決められない」を返す。`logic/state/next.ts` は、「決められない」を受けたら停止中（`unrecoverable`）にする（「[状態遷移と判定][b02]」の「次の工程」）

## 作業の単位の進め方

![作業の単位の見直し][fig-units]

`flow/run/units.ts` は、作業の単位を次の組の順に進める（「[状態遷移と判定][b02]」の「作業の単位を進める順」）。

| 組 | 進める工程 |
| --- | --- |
| 1（片付け） | 片付け |
| 2（進行中） | マージの判定のほかの工程 |
| 組 2 の後 | レビュー合格の PR の、マージの判定（PR の番号の小さい順） |
| 3（依頼） | 依頼の枠の工程 |
| 4（新しい着手） | 決定の印と計画の輪 |

作業の単位ごとに、次の見直しを繰り返す。

| 順 | すること |
| --- | --- |
| 1 | `logic/state/assemble.ts` で状態を組み立てる |
| 2 | `logic/state/next.ts` に、状態と、実行の全体の値（下の箇条）と、今の時刻を渡し、工程を決める |
| 3 | 下の「見直しを止める条件」に当たれば、作業の単位を終える |
| 4 | 工程のファイルを呼ぶ |
| 5 | 工程の結果の書いた印と GitHub の変化を、`logic/state/apply.ts` で事実に足し、順 1 に戻る |

| 見直しを止める条件 | 理由 |
| --- | --- |
| 何もしないか待つ行（止め方か安全の軸に値がある、待つ、人の番）に当たった | 「次の工程」の「先に見る」と「状態ごとの工程」 |
| 同じ実行で 2 つ目の重い工程に当たった | 重い工程は作業の単位ごとに 1 回の実行で 1 つ（[FLOW-4][flow]） |
| push した | push した作業の単位は次の実行まで進めない（[FLOW-4][flow]） |
| 工程の結果が `advanced` でない（止めた、攻撃の疑いにした、失敗した、起こさなかった） | 下の「工程の結果」 |
| 組 2 で、マージの判定に当たった | マージの判定は組 2 の後に回す（[MERGE-5][merge]） |
| 1 つの作業の単位で、見直しが 20 回を超えた | 足し方の誤りで同じ工程を繰り返し、実行が終わらない形を止めるため。実行の失敗として記録し、次の作業の単位へ進む |

- `logic/state/next.ts` に渡す実行の全体の値は、全体の止める合図、全体の停止の Issue の有無、基線が緑か、着手した件数、下書きと書き直しの件数、`ready` の判定を書いた PR、進行中の作業の単位の数（着手しようとする作業の単位自身を除く。`logic/state/order.ts` が、渡す時点のすべての作業の単位の状態から数える）、重い工程を起こしてよいか（下の「実行の失敗の数え方」）。どれも実行の中で変わるので、作業の単位ごとと見直しごとに渡し直す
- 足す時刻は、書いたときの GitHub の応答の時刻（コメントの `created_at` など）を使い、手元の時計を使わない。マスターと人の操作のように、時刻の前後で状態が決まるため
- 作業の単位の見直しを終えたら、`flow/steps/display.ts` で、Project の表示の欄と表示のラベル `security-suspected` を書く。`ProjectIssues` で読んだ今の値と違う欄とラベルだけを書く。毎回の実行で、変わっていない欄まで書き直さないため
- 作業の単位を 1 つ終えるたびに、期限（`RunContext`）を見る。過ぎていたら、残りの作業の単位を進めず、作業の単位の結果に「次の実行に回した」と書く（順 9）。組の境目でも同じに見る
- 前の実行が残したセッションを止められなかった作業の単位は、組を問わず、組 1 の前に停止中（`session-not-stopped`）にする。片付けの「先に見る」の順 2 の扱いは、書いた `stop` の印の時刻で決まる（「[状態遷移と判定][b02]」の「1 回の実行の流れ」の順 2）

## 工程

`logic/state/next.ts` は、次の記号の工程を返す。重さと線は「[状態遷移と判定][b02]」の「状態ごとの工程」と「先に見る」の表のとおり。

| 工程の記号 | 当たる行 | ファイル |
| --- | --- | --- |
| `entry-quarantine`・`draft`・`redraft`・`instruction-quarantine`・`approval-reset`・`ignored-instruction` | 起票済み・検疫済み・差し戻し済み・直し済み・下書き済み、決めた後の `## 作業指示` | `flow/steps/brief.ts` |
| `decision-marker`・`start` | 決定済み（(1) と (2)） | `flow/steps/decide.ts`（`start` は `flow/steps/plan.ts` を呼ぶ） |
| `create-change`・`create-pr-from-branch`・`fix-checks`・`fix-review`・`fix-conflict` | 計画済み・検査不合格・レビュー不合格・衝突あり | `flow/steps/change.ts` |
| `review`・`remark-review` | 検査合格（載せ直しの後に印を付け直すときは `remark-review`） | `flow/steps/review.ts` |
| `follow-up`・`ready`・`rebase`・`merge-judgment`・`remind` | レビュー合格の (1)〜(5) | `flow/steps/passed.ts` |
| `cleanup`・`close-old-pr` | マージ済み・閉鎖済み、「先に見る」の順 9 | `flow/steps/cleanup.ts` |
| `wait-marker`・`dependency-review` | 「先に見る」の順 13・14 | `flow/steps/dependency.ts` |
| `stop` | 停止中にする行（止めた理由の記号と、止めた先を添える） | `flow/steps/stop.ts` |
| `none` | 何もしないか待つ行 | — |

### 工程の結果

工程のファイルは、次の結果を返す。`flow/run/units.ts` は、結果で見直しを続けるかを決め、作業の単位の結果（`runs.jsonl`）に書く。

| 結果 | 中身 | 見直し |
| --- | --- | --- |
| `advanced` | 工程を終えた。書いた印と GitHub の変化を添える | 続ける（push したときと、重い工程の後に 2 つ目の重い工程に当たったときを除く） |
| `stopped` | 停止中にした（止めた理由の記号） | 止める |
| `suspected` | 攻撃の疑いにした（`security` の印の `where`） | 止める |
| `failed` | 実行の失敗（AI のセッションの終わり方の記号、push の失敗、書き込みの失敗など） | 止める |
| `deferred` | 合図・全体の停止・1 日か 1 回の実行の費用の上限で起こさなかったか、途中で止めた | 止める。実行の失敗に数えない |

- AI のセッションの終わり方の記号（「[AI のセッション][d07]」の「段階」）は、工程のファイルが結果に写す。`done` のほかは、`stop-signal` を `deferred` に、`not-started` を外れた順の扱い（`deferred`・`stopped`・`suspected`）に、`not-stopped` を `stopped`（`session-not-stopped`）に、ほかを `failed` にする
- 計画の輪とレビューのラウンドのように、1 つの工程が 2 つ以上のセッションを起こすときも、結果は工程に 1 つにする

## GitHub に書くとき

`flow/run/write.ts` は、GitHub に書くすべての処理（コメントと印、ラベル、Project の欄、PR の作成と閉じること、Issue を閉じること、ブランチを消すこと）の前と後に、次を行う。push は `flow/push.ts` が、同じ決まりで行う。

| 順 | すること |
| --- | --- |
| 1 | 全体の止める合図を読み直す（`flow/safety/signal.ts`）。`run` でなければ書かずに、工程を `deferred` で終え、実行を合図で止めた（`stop-signal`）として終える |
| 2 | 印を付けるコメントは、`marker.ts` と `logic/marker/sign.ts` で印の行を作り、`logic/text/comments.ts` で決まった部分と `CheckedText` の文につなぐ（「[文の検査と受け入れ][d06]」の「コメントの組み立て」） |
| 3 | 書く |
| 4 | 結果（書いた・HTTP の状態で断られた・時間の上限を過ぎて届いたか分からない）を `writes/<実行 ID>.jsonl` に 1 行足す（「[設定と記録][d02]」の「手元の記録」） |

- 書き込みが断られたときと、時間の上限を過ぎたときは、作業の単位を今の実行では進めず、工程を `failed` で終える。次の実行が GitHub から読み直して続ける（「[設定][b01]」の「子プロセスが時間の上限を過ぎたとき」）
- 書いた印は、コメントの ID と GitHub の応答の時刻と組で、工程の結果に添える（上の「作業の単位の進め方」の足す時刻）
- 「1 回だけ書く」印（「[状態遷移と判定][b02]」の「ほかの印」）を書くかは、`logic/state/next.ts` が、事実の有効な印から決める。`flow/run/write.ts` は書くかを判定しない
- 合図は、工程ごとにまとめて読まず、書き込みの呼び出しのたびに読む。工程の途中で人が合図を `stop` にした後の書き込みも止めるため（[ADR-20261017-007][adr-stop] の「GitHub に書き込む前に合図を読む」を、呼び出しの単位で行う）

## 実行の失敗の数え方

`logic/run/failures.ts` は、`runs.jsonl` の作業の単位の結果と、作業の単位の最新の `## 再開`（R）の時刻と、今の時刻から、作業の単位ごとに次を返す。数え方の決まりは「[AI のセッション][b03]」の「起動の前の確かめ」の表の下のとおり。

| 返すもの | 決め方 |
| --- | --- |
| 実行の失敗が続いた回数 | R より後の結果を新しい順に見て、結果が `failed` のものを数え、重い工程を終えた結果（`advanced` の重い工程）で数えるのをやめる。`deferred` と、作業の単位を進めなかった実行は、数えず、数えるのもやめない |
| 片付けの失敗が続いた回数 | 同じく、片付けの工程の `failed` だけを数える。片付けのほかの工程の結果では、数えるのをやめる |
| 重い工程を起こしてよいか | 最後に `failed` を書いた実行より後の、起こされ方が定期の実行の数（今の実行を含む）が、`limits.retry.intervalRuns` 以上か（初期値の 1 なら、次の定期の実行で起こしてよい） |

- 失敗が `limits.retry.count` 回目になったら、失敗した工程の後に、同じ実行で停止中にする（`retry-limit`、片付けは `cleanup-failed`）。`stop` の印を書けなかったときに備え、`logic/state/next.ts` も、回数が上限に達していて、R より新しい `stop` の印が無ければ、工程の代わりに停止中を返す
- 受け入れの依頼の実行のセッションは、作業の単位の結果に書かないので、続いた回数に入らない（「[AI のセッション][b03]」の「起動の前の確かめ」の表の下）

## push の手順

`flow/push.ts` は、「[状態遷移と判定][b02]」の「push の手順」の順 1〜8 を行い、次のどれかを返す。工程のファイルは、返った結果を工程の結果に写す。

| push の結果 | 中身 | 工程の結果 |
| --- | --- | --- |
| `pushed` | 送り、読み戻した（初回は PR と `link` の印まで） | `advanced` |
| `stopped` | `workspace-tampered`・`no-new-commit`・`protected-path`・`manual-push`・`not-fast-forward`・`unrecoverable` のどれかで止めた | `stopped` |
| `suspected` | 漏えいの検査（`where` が `leak`）か検疫（`where` が `text`、`manual-push` の差分は `code`）で見つけた | `suspected` |
| `deferred` | 順 6 で、合図が `run` でないか、全体の停止の Issue があった | `deferred` |
| `failed` | 出力契約を取り出せない、対応コメントを書けない、push が断られた、PR を作れない、時間の上限を過ぎた | `failed` |

- 範囲の commit の一覧（`git rev-list <控えた SHA> ^<base> ^<origin>`）、commit ごとの触れたパス（`git diff-tree -r --name-only -z`。merge の commit は `--cc`）、commit ごとの差分とメッセージと作者は、`io/git.ts` で読む。パスの照らしは `logic/push/paths.ts` が `logic/settings/paths.ts` を使って行い、push で効くものの当たりを、マージまで効かないものの当たりより先に返す
- PR の題名を規約（`conventions.*`）に照らす処理と、固定の形の題名と、本文の並べ方は、`logic/text/comments.ts` に置く（「[文の検査と受け入れ][d06]」の「コメントの組み立て」）
- 重なり（`overlap` の印）は、順 2 で取り込んだ後に、`logic/push/overlap.ts` が、PR の変更ファイルの一覧と、ほかの進行中の PR の変更ファイルの一覧（進行 Bot の clone の `git diff --name-only <base>...<ブランチ>`）を比べて決める
- 取り込みの前の後始末（git のプロセスが無いことの確かめ、`index.lock` を消す、`git merge --abort`、`git reset --hard` と `git clean -ffdx`）は、`flow/run/clone.ts` が行う。git のプロセスがあれば、取り込まずに `failed` にする
- 載せ直しと、載せ直しの後に印を付け直すときの差分の比べ方は、`git diff <base>...<head>` に `git patch-id --stable` を掛けた値にする。前の head と新しい head の両方で、同じ base の先端（進行 Bot の clone で fetch したもの）を使う

## 判定

`logic/judge/` の判定は、基本設計の「[状態遷移と判定][b02]」の「判定」のとおり。本節は、判定に渡す事実と、基本設計が決めていない細部を決める。

| ファイル | 渡す事実 | 決めた細部 |
| --- | --- | --- |
| `checks.ts` | `checks.required`、パスを付けたチェックの実行（名前、app、パス、状態、結果、始まりと終わりの時刻、ID）、今の時刻 | 「最新の 1 件」は、名前・app・パスが同じものの中で、始まりの時刻が新しいもの（同じなら ID の大きいもの）。`checks.required` が空なら緑と読まず、待つと返す（点検が fail にする。[IMPL-27][impl]） |
| `rounds.ts` | 数える `review` と `fix` の印 | `round` の欠けは、数える `review` の印の `round` の集合が 1 から最大まで欠けずにそろうかで見る。同じ `round` の印が 2 つ以上あっても欠けにしない（最新の 1 つを採る） |
| `convergence.ts` | 必須チェックの結果、PR の head、最新の `review` の印 | — |
| `repeat.ts` | ラウンドごとの `review` の印の `findings` と `head`、`fix` の印、最新のラウンドの head の形（親の SHA の並びと、2 つ目の親が base ブランチにあるか） | head の形は、工程のファイルが進行 Bot の clone で読む（`git rev-list --parents -n 1 <head>` と `git merge-base --is-ancestor <2 つ目の親> <base>`） |
| `merge.ts` | レビュー合格の判定、PR の向き先と `link` の印、`git diff --raw <base>...<head>` の一覧、最終の head の差分への漏えいの検査の結果、最新の `review` の印の `droppedHigh`、`merge.allowPaths` と `merge.policyPaths`、実行の全体の値の `ready` の判定を書いた PR、ほかの開いた PR の最新の `merge` の印 | `ready` の 1 件の上限は、今の実行で `ready` を書いた PR と、ほかの開いた PR のうち、最新の数える `merge` の印が `ready` で、`head` が PR の今の head と同じものを数える。head が進んだ PR は、判定し直すまで数えない |

- `reasons` の記号の一覧は `logic/judge/merge.ts` の定数にし、`marker.ts` が読む（「[印][d03]」の「欄の値の形」の表の下）

## 全体の停止と通知

| すること | ファイル | 決めたこと |
| --- | --- | --- |
| 全体の止める合図を読む | `flow/safety/signal.ts` | REST でリポの変数 `DEV_AUTOPILOT_RUN` を読む。値が `run` か、ほかの値か、読めないかを返す。読めないときは `run` でないものと扱う |
| 全体の停止の Issue を読む | `flow/safety/signal.ts`、`logic/state/halt.ts` | `safety.botAccounts` のアカウントごとに、起票した Issue を REST の `creator` で開閉を問わず一覧にし、`WorkUnitIssue` と同じ読み方でコメントを読む。止める間かは `logic/state/halt.ts` が決め、読めなければ「ある」とする |
| 攻撃の疑いにする | `flow/safety/suspect.ts` | `security` の印を、Issue と、PR が契機なら PR に書く。`where` が `text` なら `flow/safety/halt.ts` を呼ぶ。書けた印ごとに通知する |
| 全体の停止の Issue を開く | `flow/safety/halt.ts` | 開いた全体の停止の Issue があれば `halt` の印のコメントを足し、無ければ Issue を開いてから `halt` の印のコメントを書く。続けて、`RunContext` の動いているセッションを止める手順で止め、`RunContext` の全体の停止の Issue を「ある」にする |
| 通知する | `flow/safety/notify.ts` | 文は `logic/text/comments.ts` の固定の文と、決まった値（リポ、種類、Issue と PR の番号と URL）だけで作る。送り直しの回数と間隔は、プログラムに固定する値（「[設定と記録][d02]」の「プログラムに固定する値」）に置き、値は送り先のサービスを段階 3 で確かめるときに決める |

- 合図と全体の停止の Issue を読む時は、「[状態遷移と判定][b02]」の「1 回の実行の流れ」の表の下のとおり（順 3、重い工程の起動の前、書き込みの前（合図だけ）、セッションの見張り、push の手順の順 6、順 10）
- 送れなかった通知は、種類・Issue の番号・時刻・応答の状態を `runs.jsonl` の行に書き、点検の warn に出す。Webhook の URL は書かない（[ADR-20261101-002][adr-notify]）

## 止めた理由の記号

`logic/state/reasons.ts` に、止めた理由の記号の一覧を、14 個と実装の都合の記号の 2 つの組で定数として持つ。`marker.ts` は、`stop` の印の `reason` の値の形を、一覧で確かめる（「[印][d03]」の「欄の値の形」）。

- 実装の都合の記号は、基本設計の「止めた理由の記号」の表の名前を変えずに使う（基本設計が、名前を変えるかを詳細設計に任せた）。どれも英小文字を `-` でつないだ形で、14 個の記号と書き方がそろっており、要件・基本設計・図に同じ名前で載っている。名前を変えても動きは変わらず、文書と図を直す手間だけが増えるため
- 組み直す前の記号（`usage-limit` など）は、一覧に入れない。進行 Bot はまだ GitHub に印を書いたことが無く、読む古い印も無いため。古い記号の `stop` の印は、形が合わない無効な印になる
- 止めた理由のコメントの決まった文（人がすること）は、記号ごとに `logic/text/comments.ts` の言語ごとの表に持つ。記号ごとの差し込む値（当たったパス、PR へのリンク、消すフォルダの組と順など）は、`flow/steps/stop.ts` が工程の結果から受け取る

## 作業場所の残骸

`logic/run/residue.ts` は、置き場の `sessions/` のフォルダの名前と、短い一時フォルダの根の下の ID と、セッションのフォルダの ID のファイルの中身と、名前から引いた Issue と PR の事実を受け取り、フォルダごとに、扱い（「[状態遷移と判定][b02]」の「作業場所の残骸」の表の行）と扱う所（点検か次の工程か）を返す。

- 点検（順 5）は、名前から引いた Issue と PR を `flow/run/gather.ts` と同じ読み方で読み、扱う所が点検の行だけを結果にする（「[点検][d04]」）
- 次の工程は、`logic/state/next.ts` が、作業の単位の事実に入れた残骸の扱いで、「先に見る」の順 8・9・12 に当てる
- 名前から工程と番号を読む処理は、`logic/session/names.ts` と共有する（「[AI のセッション][d07]」の「ファイルの一覧」）

## 人が呼ぶ /dev-autopilot

`flow/skill/dev-autopilot.ts` は、「[導入と点検][b04]」の「状態の表示」を受け持つ。秘密の置き場を読まず、GitHub の API と `claude` を呼ばない（[README][readme] の「入口ごとの import の範囲」）。

| 引数 | すること |
| --- | --- |
| 表示 | マシンの設定から記録の置き場を見つけ、`status.json` を読んで表示する。順は、実行の時刻・起こされ方・終わり方、全体（合図・全体の停止の Issue・基線）、点検の fail と warn、人の番の作業の単位、ほかの作業の単位、見送った依頼。記録の目印のファイルを作った日が今日なら、`setup` をした日は AI のセッションを起動しないことを添える |
| 1 回実行 | `run.lock` があれば、依頼を置かずに、実行の最中であることと lock の起動の時刻を示して終える。無ければ、`run-request.json` を置くコマンドの文（書く先のパスと中身）を返す。Skill が人の許可を得て sandbox の外で動かし、続けて定期の実行と同じジョブを起こす（`-k` を付けない。FACT-80） |

- 表示は、状態と軸を、記号でなく [FLOW-39][flow] の名前で出す（`logic/state/states.ts` の表）。止めた理由は記号と、`logic/text/comments.ts` の決まった文で出す
- `status.json` の形が違うか読めなければ、表示できないことと、`launchd.err.log` を見ることを示す
- 依頼を置くコマンドの形と、人の許可の求め方は、点検の `check` と同じ（「[点検][d04]」の「人が呼ぶ check」）

## テスト

| ファイル | 単体テストで確かめること |
| --- | --- |
| `flow/run/run.ts` | 起こされ方ごとに、上の「起こされ方と進む順」の表のとおりに呼ぶこと。順 3〜5 で止まっても順 10 を行うこと。思わない失敗で、動いているセッションを止め、記録を書き、lock を外すこと |
| `logic/run/records.ts`（足す分） | 終わり方ごとの終了コード |
| `flow/run/gather.ts` | `fullDatabaseId` を数にすること、収まらない値を読めなかったものにすること。`__typename` が `Bot` の名前に `[bot]` を足すこと。途中のページの失敗と、1 つの作業の単位の失敗で全体を止めること。同じ人の権限を 1 回の実行で 1 回だけ引くこと。チェックの実行とワークフローの実行の突き合わせ |
| `logic/state/discount.ts`・`main-flow.ts`・`axes.ts` | 基本設計の表の行ごと。表の順で、進んだほうを先に見ること。どの行にも当たらないときに「決められない」を返すこと |
| `logic/state/next.ts` | 「先に見る」と「状態ごとの工程」の行ごと。実行の全体の値（全体の停止、基線、件数の上限、重い工程を起こしてよいか）ごとの扱い。続いた失敗が上限で、新しい `stop` の印が無いときに停止中を返すこと |
| `logic/state/apply.ts` | 工程ごとの書いた印と変化を足した後の組み立てが、GitHub から読み直したときと同じ状態になること |
| `logic/state/order.ts` | 組の順と組の中の順（マイルストーンの版、Issue と PR の番号）。進行中の作業の単位の数え方 |
| `logic/state/halt.ts` | 開いた Issue、`## 安全の確認` の無い閉じた Issue、権限の無い人の `## 安全の確認`、読めないときに止めること |
| `logic/state/reasons.ts` | 一覧が、基本設計の 2 つの表の記号と同じこと（期待値は基本設計から写す） |
| `flow/run/units.ts` | 見直しを止める条件の行ごと。組 2 でマージの判定を回さず、組 2 の後に PR の番号の順に回すこと。期限を過ぎたら残りを次の実行に回すこと |
| `flow/run/write.ts` | 書く前に合図を読み、`run` でなければ書かないこと。結果を `writes/` に書くこと。断られたときと時間の上限を過ぎたときに `failed` を返すこと |
| `flow/push.ts`・`logic/push/` | 基本設計の「push の手順」の順ごとの外れ方。範囲に、取り込んだ base の commit と GitHub にもうある commit を入れないこと。途中の commit で触れて戻したパスで止めること |
| `logic/judge/` | 基本設計の「判定」の表と例（ラウンドと往復の表、同じ指摘の再発の表）。最新のチェックの選び方。`checks.required` が空で緑にしないこと。`ready` の 1 件の上限の数え方 |
| `logic/run/wake.ts`・`missed.ts`・`failures.ts`・`residue.ts` | 依頼のファイルの組み合わせごとの起こされ方。眠っていた間の時刻の数え方。R と重い工程の成功で数え直すこと、`deferred` を数えないこと。残骸の表の行ごと |
| `flow/safety/` | 合図を読めないときに `run` でないと扱うこと。全体の停止の Issue を開いた後に、動いているセッションを止め、重い工程を起こさないこと。送り直しの後も送れないときに記録に書き、URL を書かないこと |
| `flow/skill/dev-autopilot.ts` | lock があるときに依頼を置かないこと。依頼を置くコマンドの文に、書く先のパスと中身が見えること。形の違う `status.json` を表示しないこと |

- 結合テストは、偽の GitHub（「[テスト][b05]」の「偽物」）の筋書きで、作業の単位を起票済みから完了まで、決まった実行の回数で進めることを確かめる。偽の GitHub は、GraphQL の操作の名前で答えを選ぶので、上の「GitHub の事実の読み方」の操作の名前を、偽物の筋書きと共有する
- 偽の GitHub の答えには、`fullDatabaseId` を 32 ビットの整数の上限より大きな値で、App の投稿者を `__typename` が `Bot` で `[bot]` の無い名前で入れる（[FACT-125・126][facts] の形）

[readme]: ./README.md
[d01]: ./01-io.md
[d02]: ./02-settings-and-records.md
[d03]: ./03-markers.md
[d04]: ./04-check.md
[d06]: ./06-text-and-intake.md
[d07]: ./07-sessions.md
[b01]: ../basic/01-settings.md
[b02]: ../basic/02-flow-and-judgment.md
[b03]: ../basic/03-ai-sessions.md
[b04]: ../basic/04-setup-and-check.md
[b05]: ../basic/05-tests.md
[flow]: ../../requirements/01-issue-flow.md
[impl]: ../../requirements/02-implement-review-fix.md
[merge]: ../../requirements/03-merge.md
[text]: ../../requirements/04-external-text-and-markers.md
[facts]: ../../requirements/10-verified-facts.md
[adr-stop]: ../../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-notify]: ../../adr/adr-20261101-002-notify-outside-github-by-webhook.md
[fig-run]: ./diagrams/08-run.drawio.svg
[fig-units]: ./diagrams/08-units.drawio.svg
