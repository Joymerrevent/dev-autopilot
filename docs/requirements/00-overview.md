# 概要（目的・前提・登場するもの）

## 目的

Project に登録された Issue のうち、人が「処理対象」と判断したものを、AI が worktree で実装して PR にし、
別セッションの AI がレビューし、指摘が無くなるまで作業 AI が直す。開発の PR は、条件を満たしたら進行 Bot が「マージしてよい」と判定して人に渡し、マージの実行は人が行う。
方針に係る PR（ADR など）は判定の対象にせず、マージできる状態で止めて人に渡す。マージの実行を進行 Bot に移すかは、判定の信頼度を測ってから別に決める（「[マージ][f03]」）。

## 決まっていること

| 項目 | 決定 |
|---|---|
| 置き場所 | 独立したリポ（Joymerrevent/dev-autopilot）で作る（[ADR-20261016-003][adr-move]）。進行 Bot が当てる先は、マージ判定工程（段階 8）の後に決めるまで business-os だけ（「[分離を見越した構成][f07]」）。文書とコードには業務の固有名詞と秘密を置かない（秘密は gitleaks で検査する。固有名詞の混入を探す検査は無い。UNDECIDED-46） |
| 配り方と実行環境 | 誰でも、Plugin を当てる先のリポに入れて `setup` を実行すれば、進行 Bot を含めてすべて動く。進行 Bot は Plugin に同封し、`setup` が Plugin の版を利用者のマシンに写し、OS ごとの定期の実行が写しから動かす（[ADR-20261017-003][adr-plugin]、「[分離を見越した構成][f07]」の STRUCT-3・8）。今は macOS（launchd）と Linux（systemd のユーザーのタイマー）。手元の Mac で始め、将来は専用マシンでも同じものを回す。GitHub Actions などのクラウドと Windows でも動かしたいが、今は対応せず、広げるときに別の ADR で決める（クラウドの論点は「[決めていないこと][f09]」の UNDECIDED-21） |
| 人の関所 | Issue が起票されたら、受け入れ（検疫・作業指示の下書き・依存とマイルストーンの候補）は進行 Bot が自動で用意し、人は下書きを読んで `agent-ready` を付ける 1 操作で処理対象にする（TEXT-12）。PR のマージの実行は人だけが行う。開発の PR は進行 Bot がマージの可否を判定して人に渡す。方針に係る PR（「[マージ][f03]」の一覧）は判定の対象外。実行を進行 Bot に移すかは、判定の信頼度を測ってから別に決める |
| マイルストーン | リリースごとにマイルストーンを組む。着手する Issue はマイルストーンの中から選ぶ（FLOW-1）。受け入れはマイルストーンの有無を問わない（TEXT-12） |
| 名前 | dev-autopilot。自動で動くのは「開発」。人が行き先（処理対象の Issue）を決め、AI が操縦し、異常時は人が操縦を取る。呼び方は `/dev-autopilot` |
| 棚卸しの間隔 | 最初は 1 日 1 回。仕組みが安定したら徐々に狭める |
| コミットの署名と鍵 | 人の鍵と自動化の鍵を分ける。人は手元の SSH 鍵で署名する（1Password の agent は使わない）。dev-autopilot のマシンには人の鍵も gh のログインも置かず、business-os に絞った deploy key（push 用）、fine-grained PAT（進行 Bot 用）、印の署名の鍵（IMPL-20）だけを置く（「[隔離と鍵の分け方][f05]」） |
| AI のセッションの隔離 | AI の 5 役（「登場するもの」）すべての `claude -p` に、dev-autopilot が `--settings` で渡す安全設定で Claude Code の sandbox を掛ける。進行 Bot・次のセッション・CI は、AI が書いた worktree の中身（設定・gitdir・CI 定義・スクリプト）を信頼して実行しない（「[安全側に倒す設計][f05]」、「[2026-10-11 のレビューで決めたことと、採らなかった案][f11]」）。詳細は「[隔離と鍵の分け方][f05]」 |
| モデル | 役ごとに別名で書き、版番号を書かない（`fable` `opus` `sonnet`。CLI の `--model` が別名を受け、最新の版に解ける。`haiku` だけは完全な ID（「[モデルの流れ][f02]」））。effort も役ごとに設定に持つ（`--effort`）。工程ごとのモデルと effort と切り替えの条件は「[モデルの流れ][f02]」の表が正本。要点は、作業 AI に `fable` を使わない、どの役にも `fable` を既定にしない、モデルを上げるのは計画 AI の自己申告と作業 AI の 2 ラウンド目の 2 か所だけ |
| エージェント | AI の 5 役のエージェント定義を、`plugin/agents/` のループごとのフォルダ（`brief` `plan` `change`）に、役の名前（`triager` `planner` `worker` `reviewer` `verifier`）で 6 つ置く（「[分離を見越した構成][f07]」の STRUCT-9）。business-os の配布物 `plugin/agents/` には入れない |

## 登場するもの

AI の役は 5 つで、一覧はこの表が正本。5 役すべてに共通：`claude -p --output-format stream-json` を進行 Bot が子プロセスとして起動し（途中で止めた回の消費も数えるため。IMPL-14）、「[隔離と鍵の分け方][f05]」の sandbox を掛け、`gh` の認証・`GH_TOKEN`・進行 Bot の鍵を渡さず（環境変数は許可した名前だけで組み立てる）、
ユーザー設定とプロジェクト設定から切り離して起動する（`--setting-sources local`。Plugin は進行 Bot のプログラムの写し（[ADR-20261017-003][adr-plugin]）の絶対パスで渡す。「[安全側に倒す設計][f05]」）。道具の制限はエージェント定義の `tools`（道具なしは `tools: []`。実機の試し（段階 2）で確認）と `permissions` の allow と deny（SAFE-2 の (e)）で行い、GitHub への書き込みは一切できない。
サブエージェントを起こす Agent ツールは、どの役にも与えない（エージェント定義の `tools` に書かず、`permissions` の deny にも入れる。[ADR-20261017-002][adr-review]）。
MCP のサーバー（ログインしている claude.ai のアカウントのコネクタを含む）は、どの役にも読み込ませない（SAFE-1。[ADR-20261017-005][adr-mcp]）。

| 役割 | 実体と道具 | セッション |
|---|---|---|
| 進行 Bot | 定期の実行（macOS は launchd、Linux は systemd のユーザーのタイマー。STRUCT-8）が起動する Node のスクリプト（TypeScript、Node 24 が直接実行）。Plugin に同封し、`setup` がマシンの置き場に写した、AI と共有しない進行 Bot のプログラムの写し（SAFE-16。STRUCT-3）から実行する | AI ではない。判定は決定的な処理で行う |
| 計画 AI | `claude -p --agent dev-autopilot:plan:planner` を worktree の中で、読むだけの道具（Read・Grep・Glob。Bash・Write・Edit なし）で起動する | Issue ごと、計画し直すごとに新しいセッション。計画だけを出力契約で返す（IMPL-10）。モデルは受け入れの印とラベルで決まり、自己申告で 1 回だけ `fable` に上がる。計画のレビューで直すべき点が見つかったら、計画し直す（上限は設定。[ADR-20261019-003][adr-replan]） |
| 作業 AI | `claude -p --agent dev-autopilot:change:worker` を worktree の中で起動する。Bash・Read・Write・Edit（WebFetch・WebSearch・Agent なし） | Issue ごと、ラウンドごとに新しいセッション。計画は進行 Bot が固定の指示文の中で渡す（思考の引き継ぎに頼らない） |
| レビュー AI | `claude -p --agent dev-autopilot:change:reviewer` を、レビューの 3 つの軸（動作・仕様・規約。REVIEW-1）ごとに、進行 Bot が PR の head で作る別の worktree（作業 AI の worktree と同じ書き込みの規則）で起動する。Bash・Read（実行して確かめるため）、Write は worktree の中だけ | PR ごと、ラウンドごと、軸ごとに新しいセッション（[ADR-20261017-002][adr-review]）。3 つの軸の後に、どの軸も挙げていない問題を探す見落としの軸のセッションも起こす（結果は、修正工程（段階 7）に進む前に使うかを決めるまで記録だけ。[ADR-20261019-002][adr-gaps]）。計画を投稿する前の計画のレビューも行う（`dev-autopilot:plan:reviewer`。計画ごとに新しいセッション。計画 AI と同じく読むだけの道具。IMPL-10、[ADR-20261018-004][adr-planrev]）。作業 AI と文脈を共有しない。モデルは設定で分けられる（既定は「[モデルの流れ][f02]」） |
| 批評 AI | `claude -p --agent dev-autopilot:change:verifier` を、レビュー AI と同じ PR の head で別に作った作業場所（[ADR-20261013-001][adr-workspace]）で、レビュー AI の後に進行 Bot が別に起動する（レビュー AI のサブエージェントでなく独立セッション）。Bash・Read | PR ごと、ラウンドごとに新しいセッション。入力は 3 つの軸のレビュー AI の指摘と差分だけ。指摘ごとの再検証の結果を出力契約で返し、進行 Bot がレビューの印に統合する（IMPL-22） |
| 受付 AI | `claude -p --agent dev-autopilot:brief:triager` を道具なしで起動する | Issue ごとに新しいセッション（コラボレータの Issue でも走る。検疫で見つかった Issue では起動しない）。読めるのは Issue の文だけ。作業指示の下書きと、計画の難しさと方針に係る要望かの判定を出力契約で返す（TEXT-12） |
| 人 | メンテナ | 処理対象の承認（受け入れの下書きを読んで `agent-ready` を付ける）、PR のマージの実行（進行 Bot の判定を見て行う）、ループが止まったときの対処 |

進行 Bot を AI にしないのは、「どの Issue を選ぶか」「待つか」「止めるか」を再現できる判定にするため。
AI は「受け入れの下書きを書く」「計画を書く」「実装する」「レビューする」「指摘を再検証する」「直す」の 6 つの作業だけを担う。
モデルの振り分け（どの Issue の計画を `fable` にするか）も AI の提案と人の上書きで決め、進行 Bot は記録された値を読むだけにする（TEXT-12、IMPL-10）。

[f02]: ./02-implement-review-fix.md
[f03]: ./03-merge.md
[f05]: ./05-safety.md
[f07]: ./07-structure.md
[f09]: ./09-undecided.md
[f11]: ./11-records.md
[adr-workspace]: ../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-move]: ../adr/adr-20261016-003-move-to-own-repository-before-basic-design.md
[adr-review]: ../adr/adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-plugin]: ../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-mcp]: ../adr/adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[adr-planrev]: ../adr/adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[adr-gaps]: ../adr/adr-20261019-002-review-gaps-axis-in-shadow-first.md
[adr-replan]: ../adr/adr-20261019-003-replan-on-plan-review-from-stage-5.md
