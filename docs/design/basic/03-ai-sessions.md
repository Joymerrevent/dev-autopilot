# AI のセッション

進行 Bot が AI の 5 役（受付 AI・計画 AI・作業 AI・レビュー AI・批評 AI）のセッションを、どの作業場所で、どの引数と環境変数で起動するかを決める。

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
| 計画 | `plan-issue-<Issue の番号>-round-<ラウンド>` | `clone/`、`worktree/`、`tmp/` |
| 計画のレビュー | `plan-review-issue-<Issue の番号>-round-<ラウンド>` | `clone/`、`worktree/`、`tmp/` |
| 実装 | `work-issue-<Issue の番号>` | `clone/`、`worktree/`、`tmp/` |
| 修正 | `fix-pr-<PR の番号>-round-<ラウンド>` | `clone/`、`worktree/`、`tmp/` |
| 衝突の解消 | `rebase-pr-<PR の番号>` | `clone/`、`worktree/`、`tmp/` |
| レビュー | `review-pr-<PR の番号>-round-<ラウンド>-<軸>`（軸は `behavior`・`spec`・`conventions`・`gaps`。`gaps` は見落としの軸） | `clone/`、`worktree/`、`tmp/` |
| 批評 | `critic-pr-<PR の番号>-round-<ラウンド>` | `clone/`、`worktree/`、`tmp/` |
| 点検の試し | `check-<日時>` | `clone/`、`worktree/`、`tmp/` |

- `tmp/` は、セッションの一時フォルダ。XDG の 4 つ（`config/`・`cache/`・`data/`・`state/`）を置き、pnpm の store も `data/` の下にできる（[SAFE-10][safety]、[ADR-20261015-002][adr-store]）
- 修正のセッションの名前には、直す対象の `review` の印のラウンドを入れる。ラウンドごとに別のフォルダになり、記録と照らしやすい
- 計画と計画のレビューのセッションの名前には、計画の工程の中のラウンドを入れる。最初の計画が 1 で、計画のレビューの指摘で計画し直すたびに 1 足す（[状態遷移と判定][design-flow]の「計画の工程の中の順番」）。自己申告で `fable` で計画し直すときは、同じラウンドの名前のフォルダを作り直す（`opus` の計画は使わないため）
- 計画のレビューの指摘で計画し直すセッションには、渡す文（作業指示）、直す前の計画、指摘の一覧を、標準入力の指示文で渡す。指摘は GitHub に書かない（[ADR-20261019-003][adr-replan]）
- 同じ名前のフォルダがあれば、作り直す直前にフォルダごと消す（[ADR-20261013-001][adr-workspace]）
- 受け入れのセッションのフォルダ（`intake-issue-<Issue の番号>`）は、セッションが終わったら（途中で止めたときも、残ったコマンドが止まったと確かめた後に）消す。clone と worktree を持たず、人が調べる材料にならないため。残すと、着手の前に人が閉じた Issue が、作業場所の残骸として `needs-human` になる（[状態遷移と判定][design-flow]の「作業場所の残骸」）
- レビュー工程（段階 6）で比べるために回すレビュー（[IMPL-19][impl]）の名前と起こし方は、記録の欄と一緒に、レビュー工程（段階 6）に入る前に決める

## 起動の形

![起動の形][fig-launch]

進行 Bot は、作業場所の worktree（受付 AI はセッションの一時フォルダ）を作業フォルダにして、次の引数で `claude` を起動する。この形は、実機の試し（段階 2）と、その後の試験で使った形にそろえた（[実機で確かめた事実][facts]）。

| 引数 | 値 | 理由 |
| --- | --- | --- |
| `-p` | — | 無人で 1 回だけ動かす |
| `--setting-sources local` | — | ユーザー設定とプロジェクト設定（hook と `CLAUDE.md`）を読まない（[SAFE-2][safety] の (a)、FACT-11） |
| `--settings` | セッションごとの安全設定のファイル | sandbox と、道具の許可と禁止を渡す（[SAFE-10][safety]）。作り方は下の「セッションごとの安全設定」 |
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

## セッションごとの安全設定

![セッションごとの安全設定][fig-session-settings]

AI のセッションに `--settings` で渡す安全設定は、読み書きを許すパス（自分のセッションのフォルダ、worktree、短い一時フォルダ）がセッションごとに違う。
進行 Bot は、起動の直前にセッションごとのファイルを作り、AI が書けない場所に置いて、起動の前と後にハッシュを比べる（要件の UNDECIDED-40 を、この節で決めた）。

### 作り方

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 進行 Bot のプログラムに持つ雛形に、マシンの設定のパスと、リポの設定の足す分を当てはめて、セッションの安全設定を組み立てる。パスは実体に解決する（シンボリックリンクをたどる。まだ無いパスは書いたまま入れる。[SAFE-5][safety]） | リポの設定の足す分に、決まった記号（下の「リポの設定に書く足す分」）で始まらないパスがあれば、起動しない（`check` も fail にする） |
| 2 | `<paths.sandboxSettings>/<セッションの名前>.json` に書き（フォルダの権限は 700、ファイルは 600）、中身の SHA-256 を手元に持つ | 同じ名前のファイルがあれば、消してから書く |
| 3 | 起動の直前に、ファイルを読み直してハッシュを比べる | 違えば起動せず、`needs-human` を付ける（stop の理由は `settings-tampered`。[状態遷移と判定][design-flow]） |
| 4 | `claude -p --settings <ファイル>` で起動する（「起動の形」） | — |
| 5 | セッションが終わったら（途中で止めたときも）、もう一度ハッシュを比べ、実行の記録にハッシュを残してから、ファイルを消す | 違えば、そのセッションの結果を使わず、`needs-human` を付ける（`settings-tampered`） |

- 雛形はファイルにせず、進行 Bot のプログラム（`plugin/src/`）の中の 1 か所に持つ。ファイルにすると、雛形の置き場とハッシュの確かめがもう 1 つ要る。プログラムの写しは AI が書けない（[STRUCT-3][struct]、[SAFE-16][safety]）
- 安全設定のファイルは、セッションの途中で書き換えると、動いているセッションにすぐ効く（[公式文書][doc-sandboxing]）。AI が書けない場所に置き、セッションの後にもハッシュを比べるのは、このため
- `sandbox-settings/` は置き場の中にある。置き場は AI の `denyRead` に入り、書き込みも許していないので、AI からは読めず書けない
- 実行の最初に、`sandbox-settings/` に残っているファイルを消す（進行 Bot が途中で止まったときの残り。実行の lock があるので、ほかの実行のファイルではない）
- `check` は、本物のセッションと同じ組み立て方で安全設定を作り、実際に読めない・書けないことを試す（[SAFE-5][safety]。試し方は「導入と点検」の文書で決める）

### 中身

| キー | 値 | 理由 |
| --- | --- | --- |
| `sandbox.enabled` | `true` | AI の Bash を sandbox の中で動かす（[SAFE-10][safety]） |
| `sandbox.failIfUnavailable` | `true` | sandbox が起動できない（依存が無い、OS が対応していない）とき、Claude Code は既定では sandbox なしでコマンドを動かす。`true` にすると、起動の時点で止まり、実行の失敗になる（[公式文書][doc-sandboxing]。安全側に倒す。[SAFE-4][safety]） |
| `sandbox.allowUnsandboxedCommands` | `false` | sandbox の中で失敗したコマンドを、sandbox の外で動かし直させない（[SAFE-10][safety]） |
| `sandbox.filesystem.denyRead` | `~/`、置き場（`paths.workspace`）、秘密の置き場（`paths.secrets`）、マシンの設定のフォルダ、短い一時フォルダの根（`sessionTmp.root`）、`/tmp/claude-<uid>/`、リポの設定の `safety.sandbox.denyRead` | 読み取りを、自分のセッションのフォルダだけにする（[SAFE-10][safety]、[ADR-20261015-001][adr-read]、[ADR-20261017-006][adr-tmp]） |
| `sandbox.filesystem.allowRead` | 自分のセッションのフォルダ（`<置き場>/sessions/<セッションの名前>/`）、自分の短い一時フォルダ、写しの `skills/`、マシンの設定の `safety.sandbox.toolDirs` | 狭い方の規則が勝つので、`denyRead` の中を開け直す（[SAFE-10][safety]、FACT-22、FACT-32） |
| `sandbox.filesystem.allowWrite` | 自分のセッションのフォルダの `tmp/`（XDG の 4 つと pnpm の store） | sandbox が既定で書けるのは、作業フォルダ（worktree）と、ユーザーごとの一時フォルダ（短い一時フォルダの下の `claude-<uid>/`）と、worktree の親 clone の `.git` だけで（FACT-9、FACT-34）、`tmp/` は入らない。`allowWrite` に入れると、sandbox の中から書ける（FACT-41。[SAFE-10][safety]） |
| `sandbox.filesystem.denyWrite` | 親 clone の `.git/worktrees/<名前>/gitdir`、リポの設定の `safety.sandbox.denyWrite` | 既定で書ける親 clone の `.git` のうち、`gitdir` は書けてしまうので止める（FACT-9、[SAFE-12][safety]） |
| `sandbox.network.allowedDomains` | リポの設定の `safety.network.allowedDomains` | 外への通信は、許可した先だけにする（[SAFE-10][safety]） |
| `disableClaudeAiConnectors` | `true` | claude.ai のコネクタを読み込ませない（[SAFE-1][safety]、[ADR-20261017-005][adr-mcp]） |
| `permissions.allow` | `Bash`、`Write(//<worktree>/**)`、`Edit(//<worktree>/**)`、`Read(//<写し>/skills/**)` | ほかの `Read` を入れると worktree の外をすべて読めるので入れない（[SAFE-2][safety] の (e)、FACT-13）。Skill を呼ぶ `Skill` の道具は、入れなくても使える（FACT-42） |
| `permissions.deny` | `Agent`、`mcp__*`、`Read(//<秘密の置き場>/**)`、`Read(//<置き場>/controller/**)`、`Read(//<写し>/src/**)`、`Write` と `Edit` の `//<親 clone>/.git/**`・`//<worktree>/.claude/**`・`//<worktree>/CLAUDE.md`・`//<worktree>/.mcp.json` | 重ねる守り（[SAFE-2][safety] の (e)、[ADR-20261017-002][adr-review]、[ADR-20261017-005][adr-mcp]） |

- sandbox のパスは、`/` 始まりが絶対パス、`~/` 始まりがホームの中。`permissions` のパスは、`//` 始まりが絶対パス（`/` 始まりはプロジェクトの中。[公式文書][doc-sandboxing]、FACT-13）
- 受付 AI は worktree を作らない（「役とエージェント定義」）ので、worktree と親 clone のパスを含む項目（`permissions.allow` の `Write` と `Edit`、`denyWrite` の `gitdir`、`deny` の `Write` と `Edit`）を入れない
- 役ごとの道具の違いは、エージェント定義の `tools` で決める。安全設定は役を問わず同じ形にする

### リポの設定に書く足す分

- リポの設定の `safety.sandbox.denyRead` と `safety.sandbox.denyWrite` には、絶対パスを書かない（[ADR-20261015-005][adr-settings]）。パスは `${home}`（ホーム）か `${worktree}`（セッションの worktree）で始めて書き、進行 Bot が組み立てるときに実体のパスに置き換える
- 読み取りと書き込みを許す側（`allowRead`・`allowWrite`・`permissions.allow`）は、設定で足せない。許す範囲を広げるのは、安全の境界を緩める変更なので、ADR で決める（[設計を変えるとき][design-change]）

## 止め方と利用枠の数え方

![止め方と利用枠の数え方][fig-stop-usage]

AI のセッションの上限、起動の前の利用枠の確かめ方、途中で止める手順、止めた後に残ったコマンドの探し方、費用の数え方を決める。
止める手順と費用の数え方は [ADR-20261018-001][adr-usage] が決めていて、基本設計で決めることになっていた「残りの探し方」を、実機で確かめてここで決めた（FACT-43）。

### 上限

| 上限 | 掛け方 | 当たったとき |
| --- | --- | --- |
| 時間（`limits.session.minutes`。45 分） | `claude` に時間の上限は無いので、進行 Bot が起動からの時間を数え、超えたら下の「止める手順」で止める | 実行の失敗。次の実行に持ち越す（[IMPL-13、IMPL-14][impl]） |
| ターン（`limits.session.turns`。200 ターン） | `--max-turns` | 実行の失敗。次の実行に持ち越す（[IMPL-13、IMPL-14][impl]） |
| 費用（`limits.session.costUsd`） | `--max-budget-usd`。Claude Code はメッセージの間で確かめるので、上限をメッセージ 1 つ分超えうる（FACT-39） | `result` の `subtype` が `error_max_budget_usd` になり、出力契約が欠けるので、実行の失敗として持ち越す（[IMPL-14][impl]） |

### 起動の前の確かめ

AI のセッションを起動する前に、毎回、次の順に確かめる。セッションは費用の上限まで使いうるので、累計にこの回の費用の上限（`limits.session.costUsd`）を足して比べる。足しても超えないときだけ起動すれば、上限を超えない。

| 順 | 確かめること | 外れたとき |
| --- | --- | --- |
| 1 | 全体の止める合図が `run` である | 起動しない。実行の失敗に数えず、合図が戻った後の実行で続ける（[運用の規則][flow-rules]、[ADR-20261017-007][adr-stop]） |
| 2 | その日の累計 ＋ この回の費用の上限が、`limits.dailyCostUsd` を超えない | 起動しない。その日は、ほかの AI のセッションも起動しない。実行の失敗に数えず、次の日の実行で続ける |
| 3 | その回の実行の累計 ＋ この回の費用の上限が、`limits.runCostUsd` を超えない | 起動しない。実行の失敗に数えず、次の実行で続ける |
| 4 | その Issue の累計（往復を含む）＋ この回の費用の上限が、`limits.issueCostUsd` を超えない | 起動せず、`needs-human` を付ける（stop の理由は `usage-limit`。[IMPL-14][impl]） |

- 1 日は、マシンの時刻の日付で区切る（0 時で切り替える）
- 累計は、手元の記録から足し上げる（[状態遷移と判定][design-flow]の「状態の組み立て」）

### 止める手順

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | セッションの間、2 秒ごとに `claude` の子孫をたどり、子孫のプロセスグループを控える（残りの探し方 1）。Bash のコマンドは `claude` と別のプロセスグループで動く（FACT-38） | — |
| 2 | 止めるとき（時間の上限、全体の止める合図）は、`claude` のプロセスグループに `SIGTERM` を送る。`SIGINT` は使わない（書いている途中のメッセージの費用を 0 と数える `result` を出すため。FACT-38） | — |
| 3 | `limits.stopGraceSeconds`（10 秒）の間に終わらなければ、`claude` のプロセスグループに `SIGKILL` を送る | — |
| 4 | `claude` が終わったら（止めずに終わったときも）、残りを 2 つの探し方で探す。探し方 1 は、控えたプロセスグループにまだプロセスがあるか。探し方 2 は、作業フォルダ（cwd）がセッションのフォルダの中にあるプロセスがあるか（`lsof`）。見つかれば、プロセスグループごと `SIGKILL` を送る | — |
| 5 | 3 秒待って、もう一度 2 つの探し方で探す | 残っていれば、作業場所を消さず、そのセッションの次の AI のセッションを起動せず、`needs-human` を付ける（`session-not-stopped`。[IMPL-14][impl]） |

- 探し方 1 は、`SIGTERM` を無視して残り、親が PID 1 に替わったコマンドも、プロセスグループで見つけられる。探し方 2 は、控える間隔（2 秒）の間に生まれて親が替わったものを拾う。どちらでも見つけて止められることを確かめた（FACT-43）
- 止めずに終わったときも探すのは、`&` で裏に回したコマンドが、`claude` が終わった後も残りうるため
- 環境変数の印と、セッションの ID では探さない。macOS の `ps` で取れなかった（FACT-44）
- 残ったコマンドは sandbox の中のままで、書けるのは自分のセッションの作業場所と一時フォルダだけ（FACT-38）。Linux で同じに振る舞うかは確かめていない（UNDECIDED-49）

### 費用の数え方

| 回の終わり方 | 費用 | トークン |
| --- | --- | --- |
| 正常に終わった（`result` があり、`terminal_reason` が `aborted_` で始まらない。費用の上限に当たった回も含む） | `result` の `total_cost_usd` | `result` の `modelUsage` |
| 途中で止めた（`result` が無いか、`terminal_reason` が `aborted_` で始まる） | `limits.session.costUsd`（`--max-budget-usd` に渡した値）とみなす | 書き終えたメッセージの `message_delta` と、書いている途中のメッセージの `message_start` から数え、出力が分からないメッセージがあったことを記録する |

- 費用とトークンは、セッションごとに手元の記録（[FLOW-5][flow]）に残す（セッションの名前、Issue か PR の番号、モデル、終わり方、費用、トークン）
- `assistant` のイベントの `usage` は、同じメッセージで重複するので足し上げに使わない（FACT-29）
- 費用は Claude Code が出す USD 換算の値で、請求の額とは限らない。上限と比べる物差しとしてだけ使う（[ADR-20261018-001][adr-usage]）

## レビューの軸の起こし方

![レビューの軸の起こし方][fig-review-axes]

1 ラウンドのレビューは、軸ごとの 3 つのセッション（動作・仕様・規約）、その後の見落としの軸のセッション、その後の批評 AI のセッションで行う（[ADR-20261017-002][adr-review] の決定 1、[ADR-20261019-002][adr-gaps] の決定 1）。基本設計で決めることになっていた、3 つを並べて起こすか、順に起こすかを決める。

- 3 つの軸のセッションは、並べて同時に起こす。3 つが終わってから見落としの軸のセッションを起こし、見落としの軸が終わってから批評 AI のセッションを起こす
- 理由：3 つの軸を順に起こすと、1 ラウンドのレビューが、軸の 3 つと見落としと批評で最大 5 セッション分（1 セッションの上限 45 分なら 225 分）かかり、1 回の実行の所要時間の上限（`run.maxMinutes` の初期値 180 分）を超える。並べると、最大 3 セッション分（135 分）で済む
- 見落としの軸には、3 つの軸が出した指摘（🔴 🟡 🟢 の見出しとファイルのパス）を渡す。結果は、修正工程（段階 7）に進む前に使うかを決めるまで手元の記録に残すだけで、批評 AI に渡さず、`review` の印にも入れない（[ADR-20261019-002][adr-gaps] の決定 2）。見落としの軸のセッションが失敗しても、そのラウンドのレビューは続け、失敗を記録に残す
- 3 つの軸のうち 1 つでも失敗したら（出力契約が解析できない、時間かターンか費用の上限など）、そのラウンドのレビューは印を書かず、実行の失敗として次の実行に持ち越す（[IMPL-18][impl]、[ADR-20261017-002][adr-review] の決定 1）。次の実行では、3 つとも起こし直す
- 並べて起こすと、利用枠を短い時間にまとめて使う。利用枠切れ（HTTP 429）も実行の失敗として扱い、次の実行に持ち越す（[IMPL-14][impl]）
- レビュー AI（3 つの軸と見落としの軸）と批評 AI は、品質ゲート（`checks.qualityGate`。初期値は `pnpm check`）の全体を回さない。CI が緑かは、収束の判定で必須チェックの結果から決める（[状態遷移と判定][design-flow]の「収束の判定」、[IMPL-27][impl]）。指摘を確かめるための実行（テストの一部を回す、再現の手順を動かす）は行ってよく、確かめた指摘は証拠レベルを `executed` にする（UNDECIDED-18 を、ここで決めた）
- 品質ゲートの全体を回さないのは、CI と同じ確かめを重ねても判定が変わらず、同時に起こす 3 つの軸が、それぞれ 1 セッションの時間の上限（45 分）の中で、全体の検査に時間を取られるため

## 役ごとの出力契約

![役ごとの出力契約][fig-contracts]

AI のセッションは、作業の結果を GitHub に書かず、出力契約で進行 Bot に返す（[IMPL-5][impl]、[SAFE-1][safety]）。役と工程ごとに、返す値の形を決める。

### 共通の決まり

- 最後の返事（`stream-json` の最後の `result` の文）の全体を、JSON の値 1 つにする。前後に文を書かない。全体が 1 つのコードブロック（言語名 `json`）のときは、ブロックの区切りの行を外して読む。ほかの形は、取り出せないとして扱う
- 進行 Bot は、役と工程ごとの JSON スキーマで確かめる。欠け・型の違い・決めていないキーがあれば、実行の失敗にして次の実行に持ち越す。解析できない出力を「成功」や「0 件」と読まない（[IMPL-5、IMPL-14][impl]）
- `--json-schema` は使わない。`--agent` と一緒に使うと結果が空になる（FACT-14）。形は、エージェント定義の指示と、進行 Bot が標準入力で渡す指示文で伝える
- JSON スキーマは、進行 Bot のプログラム（`plugin/src/`）の中の 1 か所に持つ（印の形の正本と同じ）。エージェント定義と進行 Bot は同じ写しから使うので、版はずれない（[STRUCT-3][struct]、[STRUCT-7][struct]）
- 決まった値から選ぶ欄（下の表で `"…"` の並びを書いた欄）は、ほかの値を許さない。進行 Bot が Issue や PR に書き添えるときは、AI の文でなく、値ごとに決めた固定の文で書く（[ADR-20261018-002][adr-redo] の決定 2）
- 文の欄は AI が書いた文で、GitHub に書く前に漏えいの検査を掛ける（[IMPL-15][impl]）。コメントに載せる文の長さは、GitHub のコメントの上限（65,536 文字）に収まるように、スキーマで上限を持つ

### 受付 AI（`brief/triager`。受け入れ）

| キー | 型 | 中身 |
| --- | --- | --- |
| `brief.problem` | 文 | 困っていること |
| `brief.behavior` | 文 | 望む振る舞い |
| `brief.acceptance` | 文の一覧（1 件以上） | 受け入れ条件。1 件ずつ検証できる形で書く |
| `brief.outOfScope` | 文の一覧 | 範囲外 |
| `plan` | `"opus"` か `"fable"` | 計画の難しさの判定（[TEXT-12][text]） |
| `planReason` | 文（1 行） | 判定の根拠 |
| `policy` | 真偽 | 方針に係る要望か（当てる先のリポの方針の文書を変えてほしいという要望か。[ADR-20261019-004][adr-policy]） |
| `policyReason` | 文（1 行） | 方針に係る要望かの判定の根拠 |

- 下書きの形は「[作業指示（agent brief）の形][review-skill]」に従う。ファイルのパスと行番号を書かず、手順でなく振る舞いを書く
- 進行 Bot は、4 つの欄を決まった見出しの下に並べて、「受け入れの下書き」コメントにする（[TEXT-12][text]）
- `policy` が真なら、進行 Bot は判定と `policyReason` を決まった見出しの下に書き添える。判定で Issue を止めない（[ADR-20261019-004][adr-policy]）
- 拒否（`stop_reason: refusal`）で止まったときは、出力契約を読まずに実行の失敗にする（次の回は `sonnet`。[TEXT-12][text]）

### 計画 AI（`plan/planner`。計画、計画し直し）

| キー | 型 | 中身 |
| --- | --- | --- |
| `plan.targets` | 文の一覧（1 件以上） | 対象のファイル・モジュール・型定義と、その依存関係 |
| `plan.steps` | 一覧（1 件以上）。1 件は `do`（文）と `done`（文） | 作業の手順と順序。`done` は手順ごとの完了条件 |
| `plan.verification` | 文の一覧（1 件以上） | 検証方法（どのテストとコマンドで確かめるか） |
| `plan.outOfScope` | 文の一覧 | 範囲外とするもの |
| `plan.commits` | 文の一覧（1 件以上） | 想定する commit の分割 |
| `plan.paths` | 文の一覧（1 件以上） | 変える見込みのファイルの、リポの中の相対パス。進行 Bot が push しないパスの一覧と照らす（[ADR-20261019-004][adr-policy]） |
| `selfReport` | `"design-decision"`・`"architecture-choice"`・`"unknown-failure"` から選んだ一覧（空でよい） | 自己申告。設計判断、アーキテクチャの選択、原因不明の障害解析を含むか（[IMPL-10][impl]） |

- `plan.paths` のほかの 6 つの欄は、計画に必ず含める項目（[IMPL-10][impl]）。`plan.paths` は計画コメントに載せず、照らすためだけに使う。進行 Bot は、決まった見出しの下に並べて計画コメントにし、`plan` の印のハッシュを取る
- 計画し直すセッションも同じ形で返す。計画し直したセッションの `selfReport` は記録に残すだけで、使わない（[ADR-20261019-003][adr-replan]）

### レビュー AI の計画のレビュー（`plan/reviewer`）

| キー | 型 | 中身 |
| --- | --- | --- |
| `verdict` | `"ok"` か `"fix"` | 問題が無いか、直すべき点があるか |
| `findings` | 一覧。1 件は `heading`（文）と `detail`（文） | 指摘の一覧。`fix` なら 1 件以上、`ok` なら 0 件 |

- 見るのは、渡す文に沿っているか、範囲外を含まないか、計画に必ず含める項目がそろっているか、手順と検証方法がリポの中身と食い違わないか（[IMPL-10][impl]）
- `findings` は、計画し直すセッションに渡すだけで、GitHub に書かない（マージ判定工程（段階 8）の前。[ADR-20261019-003][adr-replan]）

### 作業 AI（`change/worker`。実装、修正、衝突の解消）

| キー | 型 | 中身 |
| --- | --- | --- |
| `outcome` | `"done"` か `"stopped"` | 作業を終えたか、止まったか |
| `stopReason` | `"adr-required"` か `"canon-change"` | `stopped` のときだけ。ADR の起票が要る、規約の正典を変える必要がある（[IMPL-8][impl]） |
| `stopDetail` | 文 | `stopped` のときだけ。止まった理由の説明 |
| `pr.title`・`pr.body` | 文 | 実装で `done` のときだけ。PR の題名と本文の案（[IMPL-5][impl]） |
| `responses` | 一覧。1 件は `id`（進行 Bot が渡した指摘の ID）、`action`（`"fixed"` か `"unable"`）、`check`（文） | 修正で `done` のときだけ。渡した 🔴 🟡 の指摘ごとの対応と確認。渡した ID をすべて含む（[IMPL-23][impl]） |
| `checks` | 一覧。1 件は `command`（文）、`result`（`"pass"`・`"fail"`・`"skipped"`）、`note`（文） | `done` のとき。確かめた検査とその結果（[IMPL-5][impl]） |

- 工程（実装・修正・衝突の解消）は進行 Bot が指示文で渡し、工程ごとに要る欄をスキーマで分ける。衝突の解消は `outcome` と `checks` だけを返す
- 作業の結果は、worktree の中の commit で返す。出力契約に diff を入れない（[IMPL-5][impl]）
- `stopped` なら、進行 Bot は理由を Issue にコメントし、`needs-human` を付ける（stop の理由は `adr-required`。[状態遷移と判定][design-flow]）
- `action` が `"unable"` の指摘は、次のラウンドのレビューで再び出れば、同じ指摘の再発として人に渡る（[IMPL-24][impl]）

### レビュー AI の PR のレビュー（`change/reviewer`。軸ごと）

| キー | 型 | 中身 |
| --- | --- | --- |
| `axis` | `"behavior"`・`"spec"`・`"conventions"`・`"gaps"` | 見た軸。進行 Bot が渡した軸と同じ |
| `converged` | 真偽 | レビューが収束したか。偽なら「未収束」 |
| `unconvergedReason` | 文 | `converged` が偽のときだけ（[REVIEW-5][review-skill]） |
| `body` | 文 | 人が読む本文（[REVIEW-3][review-skill]） |
| `counts.high`・`counts.medium`・`counts.low` | 0 以上の整数 | 重要度ごとの件数。`findings` の重要度ごとの件数と同じ |
| `findings` | 一覧。1 件は下の 7 つ | 指摘の一覧 |
| `findings[].severity` | `"high"`・`"medium"`・`"low"` | 重要度（🔴 🟡 🟢） |
| `findings[].heading` | 文（1 行） | 見出し。指摘の ID と、🟢 から作る Issue の題名に使う（[ADR-20261019-001][adr-repeat]、[IMPL-26][impl]） |
| `findings[].path` | 文 | 主なファイルのリポの中のパス。ファイルに結びつかない指摘は空の文 |
| `findings[].confidence` | 0〜100 の整数 | 確信度（[REVIEW-3][review-skill]） |
| `findings[].scenario` | 文 | 失敗シナリオ |
| `findings[].evidence` | `"executed"` か `"read"` | 証拠レベル（実行で確かめた、読んだだけ） |
| `findings[].recommendation` | 文 | 推奨 |

- 3 つの軸と見落としの軸は、同じ形で返す。見落としの軸の結果は、記録だけの間は印に入れない（[IMPL-16][impl]）
- 進行 Bot は、軸ごとの `body` を仕様 → 動作 → 規約の順に並べて 1 つのコメントにし、`counts` と批評 AI の結果から `review` の印を書く。指摘の ID は、`axis`・`heading`・`path` から計算する（[IMPL-18][impl]、[状態遷移と判定][design-flow]）
- `counts` と `findings` の件数が合わなければ、出力契約が解析できないとして扱う（IMPL-18 の (1) と (4)）

### 批評 AI（`change/verifier`）

| キー | 型 | 中身 |
| --- | --- | --- |
| `results` | 一覧。1 件は下の 5 つ。渡した指摘をすべて含む | 指摘ごとの再検証 |
| `results[].id` | 文 | 進行 Bot が渡した指摘の ID |
| `results[].reproduced` | `"executed"` か `"read"` | 再現できたか（実行済み、読んだだけ） |
| `results[].confidence` | 0〜100 の整数 | 確信度 |
| `results[].severity` | `"high"`・`"medium"`・`"low"` | 推奨する重要度 |
| `results[].note` | 文 | 再検証の要点 |

- 進行 Bot は、渡した 🔴 🟡 に ID を付けて渡す。統合の規則（🔴 は 🟡 より下げない、🟡 は確信度がしきい値未満なら 🟢 に下げる）は進行 Bot が当てる（[IMPL-22][impl]、`review.criticConfidenceThreshold`）
- 渡した ID が欠けていれば、出力契約が解析できないとして扱う（[IMPL-22][impl]）

## 人の Skill から受け入れを呼ぶ形

![人の Skill から受け入れを呼ぶ形][fig-human-intake]

business-os の `/verify-issue`（人が手で深く確かめたいときに呼ぶ Skill）が、dev-autopilot の検疫と受付 AI を使う形を決める（UNDECIDED-43 を、ここで決めた）。分担は STRUCT-6 のとおりで、`/verify-issue` は、最初に検疫を呼び、business-os 固有の確かめの後に、受付 AI で下書きを作る。

| 順 | すること |
| --- | --- |
| 1 | `/verify-issue` が、`dev-autopilot:intake` の Skill を、引数「検疫」で呼ぶ。intake の Skill は、インストールした Plugin の中の検疫のモジュール（[TEXT-11][text]）を Node で動かし、結果（無し、または見つかった種類と場所。見つかった文の中身は写さない）を返す。見つかったら、`/verify-issue` はそこで止まり、下書きを作らない |
| 2 | `/verify-issue` が、business-os 固有の確かめ（hook・雛形・第三者のツールの再現）を行う |
| 3 | `/verify-issue` が、`dev-autopilot:intake` の Skill を、引数「下書き」で呼ぶ。intake の Skill は、受付 AI の定義（`dev-autopilot:brief:triager`）をサブエージェントとして起こし、受付 AI の出力契約と同じ形の下書きを返す |
| 4 | 人が、確かめの結果と下書きを読んで判断する。下書きを使うなら、Issue に `## 作業指示` のコメントとして書く |

- 呼び出しは許す。Skill を呼ぶことは、business-os が dev-autopilot のファイルを取り込む（import する）ことでなく、Plugin の入口を使うことなので、STRUCT-1 に反しない
- `/verify-issue` は、dev-autopilot の `plugin/src/` を直接実行しない。Plugin の中のファイルの路は版で変わりうるので、約束の境目を Skill の名前と引数にする
- 人が呼んだ intake の Skill は、GitHub に印を書かない。印の署名の鍵は進行 Bot だけが読み、人のセッションは秘密を読まない（[ADR-20261015-001][adr-read] の決定 2、[ADR-20261017-004][adr-human]）。進行 Bot の受け入れ（[状態遷移と判定][design-flow]の次の工程の順 9）とは別に、人の確かめの助けとして動く
- business-os の `/verify-issue` を直すのは、business-os の PR で行う（導入と点検の実装の段階 3 の「`/verify-issue` の拡張」。「[段階][stages]」の表）

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
[adr-gaps]: ../../adr/adr-20261019-002-review-gaps-axis-in-shadow-first.md
[adr-replan]: ../../adr/adr-20261019-003-replan-on-plan-review-from-stage-5.md
[adr-read]: ../../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-settings]: ../../adr/adr-20261015-005-settings-values-and-machine-settings.md
[design-change]: ../README.md#設計を変えるとき
[doc-sandboxing]: https://code.claude.com/docs/en/sandboxing
[fig-session-settings]: ./diagrams/03-session-settings.drawio.svg
[adr-usage]: ../../adr/adr-20261018-001-stop-session-and-count-usage.md
[adr-stop]: ../../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[flow]: ../../requirements/01-issue-flow.md
[flow-rules]: ../../requirements/01-issue-flow.md
[fig-stop-usage]: ./diagrams/03-stop-and-usage.drawio.svg
[text]: ../../requirements/04-external-text-and-markers.md
[review-skill]: ../../requirements/08-review-skill.md
[adr-redo]: ../../adr/adr-20261018-002-plan-redo-in-same-run.md
[adr-repeat]: ../../adr/adr-20261019-001-repeated-findings-by-ids.md
[fig-contracts]: ./diagrams/03-output-contracts.drawio.svg
[adr-policy]: ../../adr/adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths.md
[fig-human-intake]: ./diagrams/03-human-intake.drawio.svg
[adr-human]: ../../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[stages]: ../../requirements/06-setup-and-stages.md
