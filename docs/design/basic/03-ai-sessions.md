# AI のセッション

進行 Bot が AI の 5 役（受付 AI・計画 AI・作業 AI・レビュー AI・批評 AI）のセッションを、どの作業場所で、どの引数と環境変数で起動するかを決める。
セッションごとの安全設定、止め方と利用枠の数え方、役ごとの出力契約は、この文書に後で足す。

## 役とエージェント定義

![役と作業場所][fig-roles]

| 役 | エージェント定義（`--agent`） | 道具 | 作業場所 | 起動する工程 |
| --- | --- | --- | --- | --- |
| 受付 AI | `dev-autopilot:brief:triager` | なし | セッションのフォルダの一時フォルダだけ。clone しない | 受け入れ |
| 計画 AI | `dev-autopilot:plan:planner` | Read・Grep・Glob | base ブランチの先端で切った worktree | 計画 |
| レビュー AI（計画のレビュー） | `dev-autopilot:plan:reviewer` | Read・Grep・Glob | base ブランチの先端で切った worktree（計画とは別に作る） | 計画のレビュー |
| 作業 AI | `dev-autopilot:change:worker` | Bash・Read・Write・Edit | 実装は base ブランチの先端、修正と衝突の解消は PR の head で切った worktree（作業ブランチを切る） | 実装、修正、衝突の解消 |
| レビュー AI | `dev-autopilot:change:reviewer` | Bash・Read（Write は worktree の中だけ） | PR の head で切った worktree（軸ごとに別に作る） | レビュー |
| 批評 AI | `dev-autopilot:change:verifier` | Bash・Read | PR の head で切った worktree（レビューとは別に作る） | 批評 |

- 道具は、エージェント定義の `tools` で決まる（[登場するもの][overview]、[ADR-20261017-002][adr-review] の決定 2）。どの役にも Agent ツールを与えない
- エージェント定義は、ループのフォルダ（`brief` `plan` `change`）に、役の名前で置く。`--agent` の値は `dev-autopilot:<ループ>:<役>` になる（[STRUCT-9][struct]）
- 計画のレビューは、PR のレビューと別のエージェント定義（`plan/reviewer.md`）で起こす。計画のレビューは計画 AI と同じく読むだけの道具で動かす決まりで（[IMPL-10][impl]、[ADR-20261018-004][adr-planrev]）、PR のレビューの定義（`change/reviewer.md`）には Bash があるため。役はレビュー AI のまま（新しい役は足さない）
- 作業場所は、AI のセッションごとに origin から clone し、決まった SHA で worktree を切る（[ADR-20261013-001][adr-workspace]）。作業 AI だけが作業ブランチ（`<型>/issue-<番号>`）を切り、ほかの役は `--detach` で切る

## セッションの名前

セッションのフォルダ（`<置き場>/sessions/<セッションの名前>/`）の名前は、役と、Issue か PR の番号を略さずに書く（[ADR-20261013-001][adr-workspace]）。進行 Bot は名前から Issue か PR を引き、作業場所の残骸を見分ける（[状態遷移と判定][design-flow]の「作業場所の残骸」）。

| 工程 | セッションの名前 | フォルダの中 |
| --- | --- | --- |
| 受け入れ | `intake-issue-<Issue の番号>` | `tmp/` |
| 計画 | `plan-issue-<Issue の番号>` | `clone/`、`worktree/`、`tmp/` |
| 計画のレビュー | `plan-review-issue-<Issue の番号>` | `clone/`、`worktree/`、`tmp/` |
| 実装 | `work-issue-<Issue の番号>` | `clone/`、`worktree/`、`tmp/` |
| 修正 | `fix-pr-<PR の番号>-round-<ラウンド>` | `clone/`、`worktree/`、`tmp/` |
| 衝突の解消 | `rebase-pr-<PR の番号>` | `clone/`、`worktree/`、`tmp/` |
| レビュー | `review-pr-<PR の番号>-round-<ラウンド>-<軸>`（軸は `behavior`・`spec`・`conventions`） | `clone/`、`worktree/`、`tmp/` |
| 批評 | `critic-pr-<PR の番号>-round-<ラウンド>` | `clone/`、`worktree/`、`tmp/` |

- `tmp/` は、セッションの一時フォルダ。XDG の 4 つ（`config/`・`cache/`・`data/`・`state/`）を置き、pnpm の store も `data/` の下にできる（[SAFE-10][safety]、[ADR-20261015-002][adr-store]）
- 修正のセッションの名前には、直す対象の `review` の印のラウンドを入れる。ラウンドごとに別のフォルダになり、記録と照らしやすい
- 同じ名前のフォルダがあれば、作り直す直前にフォルダごと消す（[ADR-20261013-001][adr-workspace]）
- レビュー工程（段階 6）で比べるために回すレビュー（[IMPL-19][impl]）の名前と起こし方は、記録の欄と一緒に、レビュー工程（段階 6）に入る前に決める

## 起動の形

![起動の形][fig-launch]

進行 Bot は、作業場所の worktree（受付 AI はセッションの一時フォルダ）を作業フォルダにして、次の引数で `claude` を起動する。この形は、実機の試し（段階 2）と、その後の試験で使った形にそろえた（[実機で確かめた事実][facts]）。

| 引数 | 値 | 理由 |
| --- | --- | --- |
| `-p` | — | 無人で 1 回だけ動かす |
| `--setting-sources local` | — | ユーザー設定とプロジェクト設定（hook と `CLAUDE.md`）を読まない（[SAFE-2][safety] の (a)、FACT-11） |
| `--settings` | セッションごとの安全設定のファイル | sandbox と、道具の許可と禁止を渡す（[SAFE-10][safety]）。作り方は、この文書に後で足す（UNDECIDED-40） |
| `--strict-mcp-config` | — | MCP のサーバーを読み込まない。`--mcp-config` は渡さない（[SAFE-1][safety]、[ADR-20261017-005][adr-mcp]） |
| `--plugin-dir` | 進行 Bot のプログラムの写し（`devAutopilotPath`） | AI が書けない写しから、エージェント定義と Skill を読む（[STRUCT-3][struct]） |
| `--agent` | 役の表の `dev-autopilot:<ループ>:<役>` | 役ごとのエージェント定義で動かす（[STRUCT-3、STRUCT-9][struct]、FACT-2、FACT-40） |
| `--model` | 設定の `models.*` の `model` | エージェント定義の `model` より優先される（FACT-8、[モデルの流れ][impl]） |
| `--effort` | 設定の `models.*` の `effort` | 役ごとの effort を明示する（[モデルの流れ][impl]） |
| `--max-turns` | `limits.session.turns` | ターンの上限（[IMPL-13][impl]） |
| `--max-budget-usd` | `limits.session.costUsd` | 費用の上限（[IMPL-14][impl]） |
| `--output-format stream-json`、`--verbose`、`--include-partial-messages` | — | 費用とトークンを数える（[IMPL-14][impl]、FACT-29、FACT-39）。試験と同じ形にそろえる |

- 指示文と、渡す文などの値は、標準入力で渡す。引数に置くと、同じマシンのほかのユーザーから、プロセスの一覧で読めるため
- 使わない引数：`--json-schema`（`--agent` と併用すると結果が空になる。[IMPL-5][impl]、FACT-14）。sandbox を外す引数と、許可の確かめを省く引数
- 時間の上限（`limits.session.minutes`）は `claude` に無いので、進行 Bot が数えて止める（[IMPL-13][impl]）。止め方は、この文書に後で足す

## 環境変数

進行 Bot は、子プロセスの環境変数を、次の名前だけで組み立てる（[SAFE-1、SAFE-10][safety]）。ここに無い名前は渡さない。

| 名前 | 値 | 理由 |
| --- | --- | --- |
| `HOME` `PATH` `USER` `LOGNAME` `TERM` `LANG` | 進行 Bot の値をそのまま | `claude -p` がログインを保って動くのに要る。この 6 つと `TMPDIR` だけで動くことを確かめた（FACT-17） |
| `TMPDIR`、`CLAUDE_CODE_TMPDIR` | セッションごとの短い一時フォルダ（`<sessionTmp.root>/<ID>`） | sandbox の中の `$TMPDIR` をセッションごとに分ける（[SAFE-10][safety]、[ADR-20261017-006][adr-tmp]） |
| `XDG_CONFIG_HOME` `XDG_CACHE_HOME` `XDG_DATA_HOME` `XDG_STATE_HOME` | セッションのフォルダの `tmp/` の下の `config/`・`cache/`・`data/`・`state/` | ホームを読めないので、道具の設定とキャッシュをセッションの中に向ける（[SAFE-10][safety]、FACT-22） |
| `GIT_CONFIG_GLOBAL`、`NPM_CONFIG_USERCONFIG` | `/dev/null` | git と pnpm が、ホームの個人設定を読まずに動く（[SAFE-10][safety]、FACT-22） |
| `GIT_CONFIG_COUNT`、`GIT_CONFIG_KEY_0`、`GIT_CONFIG_VALUE_0` | `1`、`commit.gpgsign`、`false` | 作業 AI の commit は署名しない（[SAFE-14][safety]）。環境変数で git の設定を渡せることを確かめた（FACT-17） |
| `GIT_AUTHOR_NAME` `GIT_AUTHOR_EMAIL` `GIT_COMMITTER_NAME` `GIT_COMMITTER_EMAIL` | 進行 Bot のアカウント（`safety.botAccount`）の名前と、GitHub の noreply のアドレス | commit の作者を環境変数で渡す（[SAFE-10][safety]）。noreply のアドレスは、漏えいの検査が止めないアドレスにする（[IMPL-15][impl]） |
| `ENABLE_CLAUDEAI_MCP_SERVERS` | `false` | claude.ai のコネクタを読み込ませない（[SAFE-1][safety]） |

- 渡さないもの：`gh` の認証、`GH_TOKEN`、PAT と鍵、`PNPM_HOME`（[SAFE-1、SAFE-10][safety]）
- 名前の一覧は、設定の `safety.env.names` の初期値にする（[設定][design-settings]）。名前を足すのは ADR で決める

## レビューの 3 つの軸の起こし方

![レビューの 3 つの軸の起こし方][fig-review-axes]

1 ラウンドのレビューは、軸ごとの 3 つのセッション（動作・仕様・規約）と、その後の批評 AI のセッションで行う（[ADR-20261017-002][adr-review] の決定 1）。基本設計で決めることになっていた、3 つを並べて起こすか、順に起こすかを決める。

- 3 つの軸のセッションは、並べて同時に起こす。3 つが終わってから、批評 AI のセッションを起こす
- 理由：順に起こすと、1 ラウンドのレビューが、軸の 3 つと批評で最大 4 セッション分（1 セッションの上限 45 分なら 180 分）かかり、1 回の実行の所要時間の上限（`run.maxMinutes` の初期値 180 分）に届く。並べると、最大 2 セッション分（90 分）で済む
- 3 つのうち 1 つでも失敗したら（出力契約が解析できない、時間かターンか費用の上限など）、そのラウンドのレビューは印を書かず、実行の失敗として次の実行に持ち越す（[IMPL-18][impl]、[ADR-20261017-002][adr-review] の決定 1）。次の実行では、3 つとも起こし直す
- 並べて起こすと、利用枠を短い時間にまとめて使う。利用枠切れ（HTTP 429）も実行の失敗として扱い、次の実行に持ち越す（[IMPL-14][impl]）

[adr-review]: ../../adr/adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-planrev]: ../../adr/adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[adr-workspace]: ../../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-store]: ../../adr/adr-20261015-002-no-shared-dependency-store.md
[adr-mcp]: ../../adr/adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[adr-tmp]: ../../adr/adr-20261017-006-per-session-short-tmpdir.md
[overview]: ../../requirements/00-overview.md
[impl]: ../../requirements/02-implement-review-fix.md
[safety]: ../../requirements/05-safety.md
[struct]: ../../requirements/07-structure.md
[facts]: ../../requirements/10-verified-facts.md
[design-flow]: ./02-flow-and-judgment.md
[design-settings]: ./01-settings.md
[fig-roles]: ./diagrams/03-roles-and-workspaces.drawio.svg
[fig-launch]: ./diagrams/03-launch.drawio.svg
[fig-review-axes]: ./diagrams/03-review-axes.drawio.svg
