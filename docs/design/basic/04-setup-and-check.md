# 導入と点検

人が呼ぶ `setup` と `check`、進行 Bot が毎回の実行の最初に回す点検の、順番と仕組みを決める。
何を作り何を確かめるかは要件の [SETUP-1、SETUP-2][setup] が正本で、基本設計は順番・置き場・確かめ方を決める。

## setup の流れ

![setup の流れ][fig-setup-flow]

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 開いているプロジェクトの origin から owner/repo を引く。dev-autopilot 自身のリポなら、何も作らずに止まる（[STRUCT-3][struct]、[分離の時期の目安][struct]） | origin が無いか、GitHub のリポでなければ止まる |
| 2 | マシンの設定のフォルダ（[設定][design-settings]の「マシンの設定」）と、GitHub の今の状態を読み、足りないものの一覧を作る。人の `gh` のログインが無ければ、GitHub の準備を一覧から外す | GitHub を読めなければ止まる |
| 3 | 一覧を人に見せ、1 回だけ承認を受ける（[SETUP-1][setup]） | 承認されなければ、何も作らない |
| 4 | GitHub の準備を、人の `gh` のログインで行い、作ったものを API で読み戻して確かめる（下の「GitHub の準備」） | 読み戻した値が違えば止まり、違いを示す |
| 5 | マシンの準備を、GitHub の認証を使わずに行う（下の「マシンの準備」）。秘密のファイルは作るだけで、中身を読まない | 作れなければ止まり、作れたものを示す |
| 6 | 人がやることを、手順として示す（[SETUP-1][setup] の「人がやるもの」） | — |
| 7 | 定期の実行と同じジョブで点検だけを起こし、結果を示す（下の「点検だけで動かす仕組み」） | fail があれば示す |
| 8 | 人が、点検が pass になってから、全体の止める合図を `run` にする（[運用の規則][flow-rules]） | — |

- 何度実行しても同じ結果にする（冪等）。今あるものは作り直さず、設定ファイルの今の値は上書きしない。足りない項目だけを足す
- Plugin を更新した後に実行し直すと、プログラムの写しを新しい版で作る（下の「プログラムの写し」）
- 人の `gh` のログインが無いマシン（専用マシン）では、5 から後だけを行う。GitHub の準備は、人の手元のマシンで `setup` を実行して行う（[ADR-20261017-004][adr-human] の決定 2）

### GitHub の準備

| 作るもの | 作り方 | 確かめ方 |
| --- | --- | --- |
| ラベル 5 つ（`agent-ready`・`needs-human`・`from-review`・`plan:fable`・`plan:opus`） | 無いものだけ作る | 名前で読み戻す |
| Project の Status の選択肢（`In Review`・`Ready to Merge`・`Blocked`）と、単一選択の Agent 欄（`worker`・`reviewer`・`human`・`paused`） | Status は、今ある選択肢を `id` 付きで含めて丸ごと送る（`id` を落とすと、今ある選択肢と項目の値が消える。FACT-15）。Agent 欄は無ければ作る | 選択肢の名前と `id` を読み戻し、リポの設定に書く |
| base ブランチの ruleset（PR 必須、force push の禁止、必須チェックを strict で登録、`require_extra_approval_for_unattributed_changes` は無効） | 無ければ作る。あれば違いを示し、上書きしない | 読み戻して項目を比べる（FACT-7） |
| deploy key の push 先を `<型>/issue-*` のブランチに限り、タグの作成を禁じる ruleset | 無ければ作る。あれば違いを示し、上書きしない | 読み戻して項目を比べる（[IMPL-15][impl]） |
| 全体の止める合図（リポの変数 `DEV_AUTOPILOT_RUN`） | 無ければ値を `stop` で作る。あれば値を変えない | 読み戻す（[ADR-20261017-007][adr-stop]） |
| リポの設定（`.claude/dev-autopilot.json`） | 表の初期値と、API で引いた Project の ID と選択肢の ID で作り、新しいブランチから base ブランチへの PR にする（base ブランチは PR 必須のため）。人がマージする | PR を示す。進行 Bot は base ブランチの先端から読むので、マージされるまで点検は fail になる |

- Project の自動追加（組み込みのワークフロー）は公開の API が無いので、人が有効にする（[SETUP-1][setup]）。点検は有効かを読むだけ

### マシンの準備

| 作るもの | 置き場 | 中身 |
| --- | --- | --- |
| マシンの設定 | マシンの設定のフォルダの `settings.json` | マシンの 10 項目（[設定][design-settings]）。`safety.sandbox.toolDirs` は、`node`・`pnpm`・`git` の実体の場所から書く |
| 置き場 | `paths.workspace` | `sessions/`、`sandbox-settings/`（権限 700）、`program/`（下の「プログラムの写し」） |
| 記録と lock の置き場 | `paths.records` | 作った日時を書いた印のファイル（[運用の規則][flow-rules]） |
| 秘密の置き場 | `paths.secrets` | フォルダを権限 700、空のファイルを権限 600 で作る。値は人が書く（[SAFE-15][safety]） |
| 短い一時フォルダの根 | `sessionTmp.root` | 権限 700 で作る（[ADR-20261017-006][adr-tmp]） |
| 定期の実行の登録 | OS ごと | 下の「定期の実行の登録」 |

- 進行 Bot の clone（`<置き場>/controller/`）は、`setup` では作らない。clone には deploy key が要り、`setup` のマシンの準備は秘密のファイルを読まないため。進行 Bot が最初の実行で作る

## プログラムの写し

![プログラムの写し][fig-program-copy]

`setup` は、インストールされた Plugin の版を、置き場の中の版ごとのフォルダに写す。定期の実行は写しから動き、AI のセッションにも写しを読ませる（[STRUCT-3][struct]、[ADR-20261017-003][adr-plugin] の決定 2）。

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 実行の lock を取る（写しを作り直すのは、進行 Bot が動いていない間に限る。[STRUCT-3][struct]） | 取れなければ、何もせずに、後で実行し直すように示す |
| 2 | インストールされた Plugin（`${CLAUDE_PLUGIN_ROOT}`）の版を、`plugin.json` の `version` から読む。`program/<版>/` がすでにあれば、中身のハッシュ（すべてのファイルのパスと中身の SHA-256）を比べる | 同じなら 3〜5 をとばす。違えば止まる（同じ版で中身が違う Plugin を、黙って上書きしない） |
| 3 | `program/.new-<版>/` に写し、写した中身のハッシュを、Plugin の中身のハッシュと比べる | 違えば、写したフォルダを消して止まる |
| 4 | `program/<版>/` に名前を変え、ファイルの書き込みの権限を外す | — |
| 5 | マシンの設定の `devAutopilotPath` を、新しい版のフォルダに向ける | — |
| 6 | 今使う版と 1 つ前の版のほかの `program/<版>/` を消し、lock を外す | — |

- 別の名前で写してから名前を変えるのは、写している途中のフォルダを進行 Bot が使わないようにするため
- 1 つ前の版を残すのは、新しい版に問題があったとき、人が `devAutopilotPath` を戻して動かせるようにするため（[ADR-20261017-003][adr-plugin] が基本設計に残した、古い写しの片付け方）
- 書き込みの権限を外すのは、人の誤った編集を防ぐため。AI は sandbox の中から、写しのどこにも書けない（[SAFE-16][safety]）
- 点検は、インストールされた Plugin の版と写しの版が違えば warn にする（[SETUP-2][setup]）

### 依存

- 進行 Bot（`plugin/src/`）は、Node 24 の標準の機能（`node:` で始まるモジュール）と、`plugin/src/` の中のファイルだけを使い、実行時の npm の依存を持たない（[ADR-20261017-003][adr-plugin] の決定 1 が基本設計に残した選択）
- 写しに `node_modules` を作らず、`setup` は依存を入れない。npm の登録所への通信も、lifecycle script も要らない
- JSON スキーマで確かめる処理（出力契約、設定）も、外の部品を使わずに `plugin/src/` の中に書く
- リポの CI で、`plugin/src/` の import が `node:` で始まるか相対パスだけであることを確かめる
- 開発の道具（検査・テスト）の依存は、今のとおりリポの root の `package.json` と pnpm に置き、配らない（[STRUCT-4][struct]）

## 定期の実行の登録

![定期の実行の登録][fig-schedule]

`setup` のマシンの準備で、OS ごとの仕組みに定期の実行を登録する。進行 Bot の本体は、どの仕組みから起こされても同じに動く（[STRUCT-8][struct]）。

### 時刻

- 時刻は、0 時から `run.intervalHours` ごと（分は 0）。初期値の 24 時間なら、毎日 0 時
- `run.intervalHours` は、24 の約数（1・2・3・4・6・8・12・24）に限る。毎日同じ時刻に並べられるため。ほかの値は点検が fail にする
- マシンが眠っていた間の時刻の分は、起きたときに 1 回だけ走らせる。launchd の `StartCalendarInterval` と systemd の `OnCalendar` は、眠っていた間の分を起きたときに走らせる（`man launchd.plist`、[systemd.timer][doc-systemd-timer]）。launchd の `StartInterval` と cron は、眠っていた間の分を逃すので使わない（`man launchd.plist`）
- 実行が重ならないように、進行 Bot は実行の lock を取り、取れなければ何もせずに終わる（[運用の規則][flow-rules]）

### macOS（launchd）

| キー | 値 | 理由 |
| --- | --- | --- |
| `Label` | `dev-autopilot.<owner>.<repo>` | 当てる先のリポごとに 1 つ登録する |
| `ProgramArguments` | Node 24 の実体の絶対パス、`<devAutopilotPath>/src/main.ts`、マシンの設定の `settings.json` の絶対パス | 写しから動かす（[STRUCT-3][struct]）。マシンの設定のパスを渡す（[ADR-20261017-004][adr-human] の決定 3） |
| `StartCalendarInterval` | 上の「時刻」の一覧（`Hour` と `Minute` の組） | 眠っていた間の時刻の分を、起きたときに走らせる |
| `EnvironmentVariables` | `PATH`（`node`・`pnpm`・`git`・`claude` の実体のフォルダと `/usr/bin:/bin:/usr/sbin:/sbin`）、`LANG` | launchd のジョブは人のシェルの設定を読まないので、道具の場所を渡す。秘密の値は書かない（点検が確かめる。[SETUP-2][setup]） |
| `StandardOutPath`・`StandardErrorPath` | `<paths.records>/launchd.out.log`・`<paths.records>/launchd.err.log` | 進行 Bot が実行の記録を書く前に止まったときの出力を残す |
| `RunAtLoad` | `false` | 登録したときには走らせない |

- 置き場は `~/Library/LaunchAgents/<Label>.plist`。`launchctl bootstrap gui/<uid> <plist>` で登録し、登録し直すときは `launchctl bootout gui/<uid>/<Label>` の後に登録する
- 手動の 1 回実行と点検は、`launchctl kickstart gui/<uid>/<Label>` で同じジョブを起こす。`-k` を付けない（動いている最中のジョブを止めずにそのままにする。`man launchctl`、[ADR-20261015-001][adr-read] の決定 3）

### Linux（systemd のユーザーのタイマー）

| ファイル | 中身 |
| --- | --- |
| `$XDG_CONFIG_HOME/systemd/user/dev-autopilot-<owner>-<repo>.service` | `Type=oneshot`。`ExecStart=` は macOS の `ProgramArguments` と同じ 3 つ。`Environment=` は `PATH` と `LANG` |
| `$XDG_CONFIG_HOME/systemd/user/dev-autopilot-<owner>-<repo>.timer` | `OnCalendar=` は上の「時刻」（24 時間なら `*-*-* 00:00:00`、6 時間なら `*-*-* 00/6:00:00`）。`Persistent=true` |

- cron でなく systemd のユーザーのタイマーにする。`OnCalendar` は眠っていた間の分を起きたときに走らせ、`Persistent=true` は電源が切れていた間の分を、次にタイマーが有効になったときに走らせる（[systemd.timer][doc-systemd-timer]）。cron は眠っていた間の分を逃す（`man launchd.plist`）
- ログインしていない間も動かすには、ユーザーの lingering（`loginctl enable-linger`）が要る（[loginctl][doc-loginctl]）。`setup` は、人がやることとして示す
- 手動の 1 回実行と点検は、`systemctl --user start dev-autopilot-<owner>-<repo>.service` で同じジョブを起こす
- Linux では、sandbox の振る舞いを確かめ直すまで（UNDECIDED-49）、進行 Bot を動かさない（[STRUCT-8][struct]）。それまで `setup` は Linux で定期の実行を登録せず、そのことを示す。進行 Bot も、Linux で起こされたら作業に入らずに止まる

### 登録し直すとき

- 写しの版が変わったとき（`devAutopilotPath` が変わる）と、`run.intervalHours` が変わったときは、`setup` で登録し直す
- 点検は、登録の中身（実行するファイルのパス、マシンの設定のパス、時刻）が、マシンの設定と `run.intervalHours` に合うかを確かめ、ずれていれば fail にする（[STRUCT-2][struct]、[設定][design-settings]）

## 点検

![点検][fig-check]

点検の項目は、要件の [SETUP-2][setup] が正本。基本設計は、項目を組に分け、いつ・どう確かめるかを決める。結果は fail / warn / pass で、fail が 1 つでもあれば、進行 Bot は作業に入らない（[SETUP-2][setup]）。

### 組と回数

| 組 | 主な項目（[SETUP-2][setup]） | 確かめ方 | 毎回の実行の最初 | `setup`・人の `check`・Claude Code の版が変わったとき |
| --- | --- | --- | --- | --- |
| 設定 | 設定ファイルの項目・型・ID、リポの設定に絶対パスが無いこと、項目が一方にだけあること、マシンの設定の場所の一致、`run.intervalHours` が 24 の約数であること | ファイルを読む | 確かめる | 確かめる |
| GitHub | ラベル、Project の欄と自動追加、ruleset、必須チェックの一覧、全体の止める合図の変数、origin の base ブランチと、その先端のリポの設定 | 進行 Bot の PAT で読む | 確かめる | 確かめる |
| PAT の権限 | 下の「PAT の権限」の読み取りが通ること。全体の止める合図の変数を変えられないこと | 読み取りと、変数を今と同じ値で書き戻す試し（403 で断られれば pass） | 確かめる | 確かめる |
| マシン | `claude` と Node 24 の版、Plugin と写しの版、置き場と秘密の置き場の権限と実体のパス、記録と印のファイル、lock、作業場所の残骸、定期の実行の登録の中身と秘密の値が無いこと、objects のリンク数 | ファイルとプロセスを読む | 確かめる | 確かめる |
| sandbox と AI のセッション | 読めない・書けないこと、`git commit` と `pnpm install` が通ること、pnpm の store、Read ツールの拒否、子プロセスから PAT と鍵が見えないこと、gitdir、objects を書き換えた clone からの fetch の失敗、MCP の道具が無いこと、一時フォルダ | 下の「試しのセッション」 | — | 確かめる |

- 毎回の実行の最初は、モデルを使わない組だけを確かめる。試しのセッションは利用枠を使うので、`setup` と人の `check` のとき、Claude Code の版が変わったときに回す（[ADR-20261017-005][adr-mcp] と [ADR-20261017-006][adr-tmp] が基本設計に残した、確かめる回数）
- Claude Code の版は、毎回の実行の最初に `claude --version` で読み、前の試しのセッションの記録と比べる。違えば、作業の前に試しのセッションを回す。Claude Code の更新で sandbox や道具の振る舞いが変わりうるため
- 作業場所の残骸は warn にし、扱いは [状態遷移と判定][design-flow]の「作業場所の残骸」に従う

### 点検だけで動かす仕組み

人が呼ぶ `check` は、進行 Bot を直接起動せず、定期の実行と同じジョブを起こす。GitHub の点検は進行 Bot の PAT で行い、人のセッションは結果を読んで表示する（[SETUP-2][setup]、[ADR-20261017-004][adr-human] の決定 2）。点検だけで動かすことは、記録の置き場に置く依頼のファイルで伝える（[ADR-20261017-004][adr-human] が基本設計に残した仕組み）。

| 順 | すること |
| --- | --- |
| 1 | 人の `check` が、origin からマシンの設定を見つけ、記録の置き場に依頼のファイル（`check-request.json`。依頼の ID と時刻）を置く |
| 2 | 人の `check` が、定期の実行と同じジョブを起こす（上の「定期の実行の登録」の手動の 1 回実行） |
| 3 | 進行 Bot が、実行の lock を取った後、依頼のファイルがあれば、点検だけを行う。全体の止める合図が `stop` でも行う（人は点検が pass になってから合図を `run` にするため） |
| 4 | 進行 Bot が、結果を `check-result-<依頼の ID>.json` に書き、依頼のファイルを消して終わる |
| 5 | 人の `check` が、結果のファイルを待って表示する |

- ジョブが動いている最中なら、手動の 1 回実行は新しく起こさない（`kickstart` に `-k` を付けない）。進行 Bot は、実行の終わりにも依頼のファイルを見て、あれば点検だけを行う。人の `check` は、実行中であることを示して待つ
- 依頼と結果のファイルは記録の置き場にあり、AI は sandbox の中から書けない

### 試しのセッション

- AI のセッションと同じ起動の形（安全設定、環境変数、引数。[AI のセッション][design-ai]）で、短いセッションを 1 つ起こす。名前は `check-<日時>`（[AI のセッション][design-ai]の「セッションの名前」）
- エージェント定義は、作業 AI（`dev-autopilot:change:worker`）を使う。道具がいちばん広い役で、境界を試すため。モデルは `haiku` で、ターンの上限は小さくする
- 進行 Bot が、試しのスクリプト（Node で動く。`plugin/src/` の中に持つ）をセッションのフォルダに置き、AI に 1 回だけ実行させる。スクリプトは、読めないはずの場所を読み、書けないはずの場所に書き、結果をセッションのフォルダに書く
- 進行 Bot は、AI の返事ではなく、スクリプトが書いた結果のファイルと、書けないはずの場所にファイルができていないことで判定する。MCP の道具とサーバーは、初期化の記録（`init` のイベント）の一覧で確かめる（[ADR-20261017-005][adr-mcp] の決定 4）
- 終わったら、「止め方と利用枠の数え方」の手順で残りを確かめ（[AI のセッション][design-ai]）、セッションのフォルダを消す。費用は実行の記録に残し、1 日の利用枠に数える
- 試しのセッションのフォルダが消せずに残ったら（途中で落ちた、残りが止まったと確かめられない）、名前から Issue か PR を引けない残骸として扱い、点検を fail にする。人が中身を確かめて消す（[状態遷移と判定][design-flow]の「作業場所の残骸」）

### PAT の権限

進行 Bot の fine-grained PAT は、当てる先のリポだけに絞り、次の権限だけを付ける（[ADR-20261017-004][adr-human] が基本設計に残した、GitHub の点検に要る読み取りの権限を含む）。権限の名前は [GitHub の公式文書][doc-pat-permissions] による。

| 権限 | 段 | 使うところ |
| --- | --- | --- |
| Metadata（リポ） | 読み取り | ruleset を読む（`GET /repos/{owner}/{repo}/rulesets` は Metadata の読み取りで通る） |
| Issues（リポ） | 書き込み | ラベル、印とコメント、🟢 から Issue を作る |
| Pull requests（リポ） | 書き込み | PR を作る、ready にする、PR にコメントする |
| Commit statuses（リポ） | 読み取り | CI の結果（コミットの状態） |
| Actions（リポ） | 読み取り | ワークフローの実行（人の承認が要るものを見つける。[IMPL-27][impl]） |
| Variables（リポ） | 読み取り | 全体の止める合図を読む。書き込みは付けない（[ADR-20261017-007][adr-stop]） |
| Projects（Organization） | 書き込み | Project の Status と Agent 欄（FACT-6） |

- Administration（ruleset を作る管理の権限）と Contents は付けない。ruleset は人の `setup` が作り（[SETUP-1][setup]）、push は deploy key で行う（[SAFE-15][safety]）
- CI のチェックの実行（check runs）を fine-grained PAT で読めるかは、確かめていない（UNDECIDED-55）

[setup]: ../../requirements/06-setup-and-stages.md
[struct]: ../../requirements/07-structure.md
[impl]: ../../requirements/02-implement-review-fix.md
[safety]: ../../requirements/05-safety.md
[flow-rules]: ../../requirements/01-issue-flow.md
[design-settings]: ./01-settings.md
[adr-human]: ../../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-plugin]: ../../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-stop]: ../../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-tmp]: ../../adr/adr-20261017-006-per-session-short-tmpdir.md
[fig-setup-flow]: ./diagrams/04-setup-flow.drawio.svg
[fig-program-copy]: ./diagrams/04-program-copy.drawio.svg
[adr-read]: ../../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[doc-systemd-timer]: https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html
[doc-loginctl]: https://www.freedesktop.org/software/systemd/man/latest/loginctl.html
[fig-schedule]: ./diagrams/04-schedule.drawio.svg
[adr-mcp]: ../../adr/adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[design-flow]: ./02-flow-and-judgment.md
[design-ai]: ./03-ai-sessions.md
[doc-pat-permissions]: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
[fig-check]: ./diagrams/04-check.drawio.svg
