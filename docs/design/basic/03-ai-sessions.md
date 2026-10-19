# AI のセッション

進行 Bot が AI の 5 役（受付 AI・計画 AI・作業 AI・レビュー AI・批評 AI）のセッションを、どの作業場所で、どの引数と環境変数で起動するかを決める。
止め方と利用枠の数え方、役ごとの出力契約は、この文書に後で足す。

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

- `tmp/` は、セッションの一時フォルダ。XDG の 4 つ（`config/`・`cache/`・`data/`・`state/`）を置き、pnpm の store も `data/` の下にできる（[SAFE-10][safety]、[ADR-20261015-002][adr-store]）
- 修正のセッションの名前には、直す対象の `review` の印のラウンドを入れる。ラウンドごとに別のフォルダになり、記録と照らしやすい
- 計画と計画のレビューのセッションの名前には、計画の工程の中のラウンドを入れる。最初の計画が 1 で、計画のレビューの指摘で計画し直すたびに 1 足す（[状態遷移と判定][design-flow]の「計画の工程の中の順番」）。自己申告で `fable` で計画し直すときは、同じラウンドの名前のフォルダを作り直す（`opus` の計画は使わないため）
- 計画のレビューの指摘で計画し直すセッションには、渡す文（作業指示）、直す前の計画、指摘の一覧を、標準入力の指示文で渡す。指摘は GitHub に書かない（[ADR-20261019-003][adr-replan]）
- 同じ名前のフォルダがあれば、作り直す直前にフォルダごと消す（[ADR-20261013-001][adr-workspace]）
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
| `sandbox.filesystem.allowWrite` | 自分のセッションのフォルダの `tmp/`（XDG の 4 つと pnpm の store） | sandbox が既定で書けるのは、作業フォルダ（worktree）と、ユーザーごとの一時フォルダ（短い一時フォルダの下の `claude-<uid>/`）と、worktree の親 clone の `.git` だけで（FACT-9、FACT-34）、`tmp/` は入らない（[SAFE-10][safety]） |
| `sandbox.filesystem.denyWrite` | 親 clone の `.git/worktrees/<名前>/gitdir`、リポの設定の `safety.sandbox.denyWrite` | 既定で書ける親 clone の `.git` のうち、`gitdir` は書けてしまうので止める（FACT-9、[SAFE-12][safety]） |
| `sandbox.network.allowedDomains` | リポの設定の `safety.network.allowedDomains` | 外への通信は、許可した先だけにする（[SAFE-10][safety]） |
| `disableClaudeAiConnectors` | `true` | claude.ai のコネクタを読み込ませない（[SAFE-1][safety]、[ADR-20261017-005][adr-mcp]） |
| `permissions.allow` | `Bash`、`Skill`、`Write(//<worktree>/**)`、`Edit(//<worktree>/**)`、`Read(//<写し>/skills/**)` | ほかの `Read` を入れると worktree の外をすべて読めるので入れない（[SAFE-2][safety] の (e)、FACT-13）。`Skill` は、レビュー AI が `review` Skill を呼ぶため（FACT-32 の試しは `Skill` を入れて行った） |
| `permissions.deny` | `Agent`、`mcp__*`、`Read(//<秘密の置き場>/**)`、`Read(//<置き場>/controller/**)`、`Read(//<写し>/src/**)`、`Write` と `Edit` の `//<親 clone>/.git/**`・`//<worktree>/.claude/**`・`//<worktree>/CLAUDE.md`・`//<worktree>/.mcp.json` | 重ねる守り（[SAFE-2][safety] の (e)、[ADR-20261017-002][adr-review]、[ADR-20261017-005][adr-mcp]） |

- sandbox のパスは、`/` 始まりが絶対パス、`~/` 始まりがホームの中。`permissions` のパスは、`//` 始まりが絶対パス（`/` 始まりはプロジェクトの中。[公式文書][doc-sandboxing]、FACT-13）
- 受付 AI は worktree を作らない（「役とエージェント定義」）ので、worktree と親 clone のパスを含む項目（`permissions.allow` の `Write` と `Edit`、`denyWrite` の `gitdir`、`deny` の `Write` と `Edit`）を入れない
- 役ごとの道具の違いは、エージェント定義の `tools` で決める。安全設定は役を問わず同じ形にする

### リポの設定に書く足す分

- リポの設定の `safety.sandbox.denyRead` と `safety.sandbox.denyWrite` には、絶対パスを書かない（[ADR-20261015-005][adr-settings]）。パスは `${home}`（ホーム）か `${worktree}`（セッションの worktree）で始めて書き、進行 Bot が組み立てるときに実体のパスに置き換える
- 読み取りと書き込みを許す側（`allowRead`・`allowWrite`・`permissions.allow`）は、設定で足せない。許す範囲を広げるのは、安全の境界を緩める変更なので、ADR で決める（[設計を変えるとき][design-change]）

## レビューの軸の起こし方

![レビューの軸の起こし方][fig-review-axes]

1 ラウンドのレビューは、軸ごとの 3 つのセッション（動作・仕様・規約）、その後の見落としの軸のセッション、その後の批評 AI のセッションで行う（[ADR-20261017-002][adr-review] の決定 1、[ADR-20261019-002][adr-gaps] の決定 1）。基本設計で決めることになっていた、3 つを並べて起こすか、順に起こすかを決める。

- 3 つの軸のセッションは、並べて同時に起こす。3 つが終わってから見落としの軸のセッションを起こし、見落としの軸が終わってから批評 AI のセッションを起こす
- 理由：3 つの軸を順に起こすと、1 ラウンドのレビューが、軸の 3 つと見落としと批評で最大 5 セッション分（1 セッションの上限 45 分なら 225 分）かかり、1 回の実行の所要時間の上限（`run.maxMinutes` の初期値 180 分）を超える。並べると、最大 3 セッション分（135 分）で済む
- 見落としの軸には、3 つの軸が出した指摘（🔴 🟡 🟢 の見出しとファイルのパス）を渡す。結果は、修正工程（段階 7）に進む前に使うかを決めるまで手元の記録に残すだけで、批評 AI に渡さず、`review` の印にも入れない（[ADR-20261019-002][adr-gaps] の決定 2）。見落としの軸のセッションが失敗しても、そのラウンドのレビューは続け、失敗を記録に残す
- 3 つの軸のうち 1 つでも失敗したら（出力契約が解析できない、時間かターンか費用の上限など）、そのラウンドのレビューは印を書かず、実行の失敗として次の実行に持ち越す（[IMPL-18][impl]、[ADR-20261017-002][adr-review] の決定 1）。次の実行では、3 つとも起こし直す
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
[adr-gaps]: ../../adr/adr-20261019-002-review-gaps-axis-in-shadow-first.md
[adr-replan]: ../../adr/adr-20261019-003-replan-on-plan-review-from-stage-5.md
[adr-read]: ../../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-settings]: ../../adr/adr-20261015-005-settings-values-and-machine-settings.md
[design-change]: ../README.md#設計を変えるとき
[doc-sandboxing]: https://code.claude.com/docs/en/sandboxing
[fig-session-settings]: ./diagrams/03-session-settings.drawio.svg
