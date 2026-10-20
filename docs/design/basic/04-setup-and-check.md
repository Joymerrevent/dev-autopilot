# 導入と点検

人が呼ぶ `setup` と `check`、進行 Bot が毎回の実行の最初に回す点検、`/dev-autopilot` の状態の表示の、順番と仕組みを決める。
何を作り何を確かめるかは要件の [SETUP-1、SETUP-2][setup] が正本で、基本設計は順番・置き場・確かめ方を決める。

## setup の流れ

![setup の流れ][fig-setup-flow]

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 開いているプロジェクトの origin から owner/repo を引く。dev-autopilot 自身のリポなら、何も作らずに止まる（[STRUCT-3][struct]、[分離の時期の目安][struct]） | origin が無いか、GitHub のリポでなければ止まる |
| 2 | マシンの設定のフォルダ（[設定][design-settings]の「マシンの設定」）と、GitHub の今の状態を読み、足りないものの一覧を作る。base ブランチの先端のコミットで動いたチェックを、必須チェックの候補として一覧に入れる（下の「GitHub の準備」）。人の `gh` のログインが無ければ（専用マシン）、GitHub を読まず、マシンの準備だけを一覧にする | 人の `gh` のログインがあるのに GitHub を読めなければ止まる |
| 3 | 一覧を人に見せ、1 回だけ承認を受ける（[SETUP-1][setup]） | 承認されなければ、何も作らない |
| 4 | GitHub の準備を、人の `gh` のログインで行い、作ったものを API で読み戻して確かめる（下の「GitHub の準備」）。人の `gh` のログインが無ければとばす | 読み戻した値が違えば止まり、違いを示す |
| 5 | マシンの準備を、GitHub の認証を使わずに行う（下の「マシンの準備」）。秘密のファイルは作るだけで、中身を読まない | 作れなければ止まり、作れたものを示す |
| 6 | 人がやることを、手順として示す（[SETUP-1][setup] の「人がやるもの」） | — |
| 7 | 定期の実行と同じジョブで点検だけを起こし、結果を示す（下の「点検だけで動かす仕組み」）。全体の止める合図が `stop` の間は、試しのセッションと PAT の書き込みの試しを行わず、未確認の warn として示す | fail があれば示す |
| 8 | 人が、点検の fail が無くなってから、全体の止める合図を `run` にする（[運用の規則][flow-rules]）。合図を `run` にした後の最初の実行が、作業の前に、試しのセッションと PAT の書き込みの試しを行う（下の「組と回数」） | 試しが fail なら、最初の実行は作業に入らない |

- 何度実行しても同じ結果にする（冪等）。今あるものは作り直さず、設定ファイルの今の値は上書きしない。足りない項目だけを足す
- Plugin を更新した後に実行し直すと、プログラムの写しを新しい版で作る（下の「プログラムの写し」）
- 人の `gh` のログインが無いマシン（専用マシン）でも、1〜3 は行う。dev-autopilot 自身のリポかを確かめ、マシンの準備の一覧に承認を受けるため（[SETUP-1][setup]）。2 で GitHub を読まず、4 をとばす。GitHub の準備は、人の手元のマシンで `setup` を実行して行う（[ADR-20261017-004][adr-human] の決定 2）

### GitHub の準備

| 作るもの | 作り方 | 確かめ方 |
| --- | --- | --- |
| ラベル 5 つ（`agent-ready`・`needs-human`・`from-review`・`plan:fable`・`plan:opus`） | 無いものだけ作る | 名前で読み戻す |
| Project の Status の選択肢（`In Review`・`Ready to Merge`・`Blocked`）と、単一選択の Agent 欄（`worker`・`reviewer`・`human`・`paused`） | Status は、今ある選択肢を `id` 付きで含めて丸ごと送る（`id` を落とすと、今ある選択肢と項目の値が消える。FACT-15）。Agent 欄は無ければ作る | 選択肢の名前と `id` を読み戻し、リポの設定に書く |
| base ブランチの ruleset（PR 必須、force push の禁止、必須チェックを strict で登録、`require_extra_approval_for_unattributed_changes` は無効） | 無ければ作る。必須チェックは、2 で一覧に入れた候補のうち、人が承認したものを登録する。あれば違いを示し、上書きしない。ただし、リポの設定の `checks.required` にあって ruleset に無い必須チェックは、承認を受けて足す（守りを強める向きの変更だけ） | 読み戻して項目を比べる（FACT-7） |
| deploy key の push 先を `<型>/issue-*` のブランチに限り、タグの作成を禁じる ruleset | 無ければ作る。あれば違いを示し、上書きしない | 読み戻して項目を比べる（[IMPL-15][impl]） |
| 全体の止める合図（リポの変数 `DEV_AUTOPILOT_RUN`） | 無ければ値を `stop` で作る。あれば値を変えない | 読み戻す（[ADR-20261017-007][adr-stop]） |
| リポの設定（`.claude/dev-autopilot.json`） | 表の初期値と、API で引いた Project の ID と選択肢の ID と、2 で承認した必須チェックの候補（`checks.required`）で作り、新しいブランチから base ブランチへの PR にする（base ブランチは PR 必須のため）。人がマージする | PR を示す。進行 Bot は base ブランチの先端から読むので、マージされるまで点検は fail になる |

- Project の自動追加（組み込みのワークフロー）は公開の API が無いので、人が有効にする（[SETUP-1][setup]）。点検は有効かを読むだけ
- 必須チェックの候補は、base ブランチの先端のコミットで動いたチェック（名前、出した app、ワークフローのパス）から作る。候補が無ければ（CI がまだ無い）、必須チェックを登録せずに ruleset を作り、`checks.required` を空のままにする。点検が fail にするので、人が CI を足して `checks.required` に書き、`setup` を実行し直すと、ruleset に足す（[SETUP-1][setup]、[IMPL-27][impl]）。`setup` が足すので、人が ruleset を手で直さずに済む（[SETUP-4][setup]）

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
| 2 | インストールされた Plugin（`${CLAUDE_PLUGIN_ROOT}`）の版を、`plugin.json` の `version` から読む。`program/<版>/` がすでにあれば、中身のハッシュ（すべてのファイルのパスと中身の SHA-256）を比べる | 同じなら 3・4 をとばし、5 から行う。違えば止まる（同じ版で中身が違う Plugin を、黙って上書きしない） |
| 3 | `program/.new-<版>/` に写し、写した中身のハッシュを、Plugin の中身のハッシュと比べる | 違えば、写したフォルダを消して止まる |
| 4 | `program/<版>/` に名前を変え、ファイルの書き込みの権限を外す | — |
| 5 | マシンの設定の `devAutopilotPath` を、使う版のフォルダに向け、定期の実行を登録し直す（下の「登録し直すとき」） | — |
| 6 | 今使う版と、直前まで使っていた版のほかの `program/<版>/` を消し、lock を外す | — |

- 別の名前で写してから名前を変えるのは、写している途中のフォルダを進行 Bot が使わないようにするため
- 直前まで使っていた版を残すのは、新しい版に問題があったときに、Plugin を前の版に戻して `setup` を実行し直せば、写し直さずに戻せるようにするため。2 で中身が同じなら写さず、5 で `devAutopilotPath` と定期の実行の登録を戻す（[ADR-20261017-003][adr-plugin] が基本設計に残した、古い写しの片付け方）
- 人がマシンの設定の `devAutopilotPath` だけを戻しても、定期の実行の登録が新しい版のパスを持つので、動くプログラムは変わらない。点検が、登録のずれとして fail にする
- `claude plugin install` には、版を指定する引数が無い（Claude Code 2.1.285 の `claude plugin install --help` で確かめた）。Plugin を前の版に戻す手順は、配り方を決めるときに決める（[分離の時期の目安][struct]）
- 書き込みの権限を外すのは、人の誤った編集を防ぐため。AI は sandbox の中から、写しのどこにも書けない（[SAFE-16][safety]）
- 点検は、インストールされた Plugin の版と写しの版が違えば warn にする（[SETUP-2][setup]）

### 依存

- 進行 Bot（`plugin/src/`）は、Node 24 の標準の機能（`node:` で始まるモジュール）と、`plugin/src/` の中のファイルだけを使い、実行時の npm の依存を持たない（[ADR-20261017-003][adr-plugin] の決定 1 が基本設計に残した選択）
- 写しに `node_modules` を作らず、`setup` は依存を入れない。npm の登録所への通信も、lifecycle script も要らない
- JSON スキーマで確かめる処理（出力契約、設定）も、外の部品を使わずに `plugin/src/` の中に書く
- リポの CI で、`plugin/src/` の import が `node:` で始まるか相対パスだけであることを確かめる
- 開発の道具（検査・テスト）の依存は、リポの root の `package.json` と pnpm に置き、配らない（[STRUCT-4][struct]）

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

- cron でなく systemd のユーザーのタイマーにする。`OnCalendar` は眠っていた間の分を起きたときに走らせ、`Persistent=true` は電源が切れていた間の分を、次にタイマーが有効になったときに走らせる（[systemd.timer][doc-systemd-timer]）。Linux の cron が眠っていた間の分をどう扱うかは確かめていないので、根拠にしない
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

| 組 | 主な項目（[SETUP-2][setup]） | 確かめ方 | 毎回の実行の最初 | `setup`・人の `check` |
| --- | --- | --- | --- | --- |
| 設定 | 設定ファイルの項目と型、リポの設定に絶対パスが無いこと、項目が一方にだけあること、マシンの設定の場所の一致、`run.intervalHours` が 24 の約数であること、`safety.env.names` に渡さない名前が無いこと | ファイルを読む | 確かめる | 確かめる |
| GitHub | ラベル、Project の欄と自動追加、リポの設定に書いた Project と選択肢の ID が実在すること、ruleset（必須チェックに `checks.required` がすべて strict で入っていること）、必須チェックの一覧、全体の止める合図の変数、origin の base ブランチと、その先端のリポの設定、基線の必須チェック（warn） | 進行 Bot の PAT で読む | 確かめる | 確かめる |
| PAT かアプリの権限 | 下の「PAT の権限」か「GitHub App の権限」の読み取りが通ること。リポの変数を作れないこと。投稿に使うアカウントが `safety.botAccounts` にあること（[ADR-20261020-001][adr-bots]）。`run.dryRun` が `false` なら、アプリに切り替えてあり、PAT のファイルが無いこと（[ADR-20261020-002][adr-app]） | 読み取りと、点検用の変数を作る試し（403 で断られれば pass） | 確かめる | 読み取りは確かめる。作る試しは、合図が `run` のときだけ |
| マシン | `claude` と Node 24 の版、Plugin と写しの版、写しの Skill とエージェント定義が Plugin の中の一覧どおりにあること、置き場と秘密の置き場の権限と実体のパス、記録と印のファイル、lock、見送った実行と起動しなかった定期の実行（warn）、作業場所の残骸、定期の実行の登録の中身と秘密の値が無いこと、objects のリンク数 | ファイルとプロセスを読む | 確かめる | 確かめる |
| 子プロセスと clone | 子プロセスに渡す環境変数に PAT・鍵・`GH_TOKEN`・`PNPM_HOME` が無いこと、objects を書き換えた clone からの fetch が失敗すること | 下の「モデルを使わない試し」 | 確かめる | 確かめる |
| sandbox と AI のセッション | 読めない・書けないこと、`git commit` と `pnpm install` が通ること、pnpm の store、安全設定の `env` が効くこと、Read ツールの拒否と写しの `skills/` の読み取り、子プロセスから秘密の置き場を読めないこと、gitdir、道具の一覧に MCP の道具と `Agent`・`Task` が無いこと、一時フォルダ | 下の「試しのセッション」 | 下の契機に当たるときだけ | 合図が `run` のときだけ |

- 試しのセッションは利用枠を使うので、毎回は回さない。毎回の実行の最初に、次のどれかに当たるときだけ、作業の前に回す（[ADR-20261017-005][adr-mcp] と [ADR-20261017-006][adr-tmp] が基本設計に残した、確かめる回数）
  - 試しのセッションの記録が無い（合図を `run` にした後の最初の実行）
  - `claude --version` の値が、前の試しのセッションの記録と違う。Claude Code の更新で、sandbox や道具の振る舞いが変わりうるため
  - 試しのセッションに渡す安全設定と環境変数の名前（リポの設定とマシンの設定の `safety.*` から組み立てたもの。セッションごとに変わるパスを除く）のハッシュが、前の試しのセッションの記録と違う。リポの設定の PR で、安全設定が変わりうるため
- 全体の止める合図が `stop` の間は、試しのセッションと PAT の書き込みの試しを行わず、未確認として warn に出す。`run` を読めない間は、AI のセッションを起動せず、GitHub に書き込まないため（[ADR-20261017-007][adr-stop] の決定 2）。人は、fail が無くなってから合図を `run` にする。合図を `run` にした後の最初の実行が、作業の前に 2 つを確かめ、fail なら作業に入らない。受け入れ工程（段階 4）の最初は `run.dryRun` が `true` なので、最初の実行の作業も GitHub に書かない（[設定][design-settings]）
- 作業場所の残骸は、Issue に紐づく残骸を warn に出して残骸の Issue だけ `needs-human` にし、どの Issue か分からない残骸を fail にする（[SETUP-2][setup]）。見分け方は [状態遷移と判定][design-flow]の「作業場所の残骸」
- 基線が緑でなければ、緑でない必須チェックの名前を warn に出す。新しい着手・最初の実装・載せ直しは止まるが、Issue に `needs-human` を付けないので、人が気づく場所に出す（[IMPL-9][impl]、[ADR-20261015-006][adr-flow] の決定 2）

### 見送った実行と、起動しなかった定期の実行

lock で見送った実行と、マシンのスリープや停止で起動しなかった定期の実行を、記録に残して warn に出す（[運用の規則][flow-rules]）。受け入れ工程（段階 4）の出口の「7 回の実行」は、2 つを除いて数える（[段階の表][setup]）。

- lock で見送った実行は、lock を取れなかった実行が、見送ったことを記録に残す（[状態遷移と判定][design-flow]の「1 回の実行の流れ」の順 2）
- 起動しなかった定期の実行は、進行 Bot が、lock を取った後に、前の実行の起動時刻から今までの定期の実行の時刻（上の「定期の実行の登録」の「時刻」）を並べて見つける。時刻ごとに、次の時刻までの間に始まった実行が無ければ、起動しなかった時刻として記録に残す。眠っていた間の分を起きたときに走らせた実行は、起きた時刻の直前の時刻の分に数える。眠っていた間のほかの時刻は、起動しなかった時刻になる
- 点検は、直近 7 日の分を warn に出す。7 日は、受け入れ工程（段階 4）の出口が 7 回の実行を数えるため

### 点検だけで動かす仕組み

人が呼ぶ `check` は、進行 Bot を直接起動せず、定期の実行と同じジョブを起こす。GitHub の点検は進行 Bot の PAT で行い、人のセッションは結果を読んで表示する（[SETUP-2][setup]、[ADR-20261017-004][adr-human] の決定 2）。点検だけで動かすことは、記録の置き場に置く依頼のファイルで伝える（[ADR-20261017-004][adr-human] が基本設計に残した仕組み）。

| 順 | すること |
| --- | --- |
| 1 | 人の `check` が、origin からマシンの設定を見つけ、記録の置き場に依頼のファイル（`check-request.json`。依頼の ID と時刻）を置く |
| 2 | 人の `check` が、定期の実行と同じジョブを起こす（上の「定期の実行の登録」の手動の 1 回実行） |
| 3 | 進行 Bot が、実行の lock を取った後、依頼のファイルがあれば、点検だけを行い、作業に入らない。全体の止める合図が `stop` でも、AI のセッションを起動せず GitHub に書き込まない項目は行う（人は点検の fail が無くなってから合図を `run` にするため）。試しのセッションと PAT の書き込みの試しは、合図が `run` のときだけ行い、`stop` の間は未確認の warn として結果に書く（[ADR-20261017-007][adr-stop] の決定 2） |
| 4 | 進行 Bot が、結果を `check-result-<依頼の ID>.json` に書き、依頼のファイルを消して終わる |
| 5 | 人の `check` が、結果のファイルを待って表示する |

- ジョブが動いている最中なら、手動の 1 回実行は新しく起こさない（`kickstart` に `-k` を付けない）。進行 Bot は、実行の終わりにも依頼のファイルを見て、あれば点検だけを行う。人の `check` は、実行中であることを示して待つ
- 依頼と結果のファイルは記録の置き場にあり、AI は sandbox の中から書けない

### モデルを使わない試し

毎回の実行の最初に、進行 Bot が、AI のセッションを起動せずに試す（[SETUP-2][setup] の「進行 Bot は毎回の実行の最初に同じ点検を回し」）。リポの設定の PR で環境変数の名前が変わっても、次の実行で確かめるため。

- 子プロセスの環境変数：AI のセッションを起動するときと同じ処理で環境変数を組み立てる。名前が、決めた名前と `safety.env.names` の足す名前だけであること、`GH_TOKEN` と `PNPM_HOME` が無いこと、値に PAT かアプリの秘密鍵とほかの鍵の中身と、秘密の置き場のパスが無いことを確かめる（[SAFE-1][safety]、[SAFE-10][safety]、[AI のセッション][design-ai]の「環境変数」）
- objects を書き換えた clone からの fetch：一時フォルダに、試しのためだけの小さなリポ（commit 1 つ）と、小さなリポの clone を作り、clone の objects のファイルを 1 つ書き換える。小さなリポに、進行 Bot の clone と同じ fetch の設定で clone から fetch し、失敗することを確かめる（[ADR-20261013-001][adr-workspace] の確認方法）。一時フォルダは試しの後に消す
  - 進行 Bot の clone を試しに使わない。進行 Bot の clone から clone すると objects のファイルがハードリンクで共有され、書き換えが進行 Bot の clone に及ぶうえ、試しが通ってしまったときに、書き換えた objects を取り込むため
- sandbox が読み書きを止めることと、道具の一覧は、モデルを使わずには試せないので、試しのセッションで確かめる

### 試しのセッション

- AI のセッションと同じ起動の形（安全設定、環境変数、引数。[AI のセッション][design-ai]）で、短いセッションを 1 つ起こす。名前は `check-<日時>`（[AI のセッション][design-ai]の「セッションの名前」）
- エージェント定義は、作業 AI（`dev-autopilot:change:worker`）を使う。道具がいちばん広い役で、境界を試すため
- モデルは `claude-haiku-5-5`（`haiku` の別名は旧版に解けるので、完全な ID で書く。[IMPL-29][impl]）、ターンの上限は 10、時間の上限は 10 分、費用の上限は `limits.session.costUsd`（試しのスクリプトの実行と 2 つの読み取りは、4 ターン・7 秒で終わった。FACT-49）。モデルとターンと時間の上限は、試しの中身と一緒に Plugin の版で変えるので、設定に持たず、進行 Bot のプログラムに書く
- 進行 Bot が、試しのスクリプト（Node で動く。`plugin/src/` の中に持つ）をセッションのフォルダに置き、AI に 1 回だけ実行させる。スクリプトは、読めないはずの場所を読み、書けないはずの場所に書き、安全設定の `env` が効くこと（`NODE_USE_ENV_PROXY` が `1` であることと、`npm config get cache` が `$TMPDIR` の下を返すこと。FACT-47）を確かめ、結果をセッションのフォルダに書く。結果には、読めたかどうかだけを書き、読めたファイルの中身を書かない
- Read ツールの試しは、AI が道具を呼ばないと行えないので、AI に、決めたパスを Read ツールで 1 つずつ読ませる。パスは、進行 Bot がセッションのフォルダの外（`<置き場>/check-probe.txt`。中身は意味の無い文字列）に置いた試験用のファイルと、写しの `skills/` の中のファイル。本物の秘密のファイルは Read ツールで読ませない（拒否されなかったとき、中身が AI と記録に渡るため）
- 進行 Bot は、AI の返事ではなく、次で判定する
  - スクリプトが書いた結果のファイルと、書けないはずの場所にファイルができていないこと
  - Read ツール：`stream-json` の Read の呼び出しと結果を照らし、試験用のファイルが `result` の `permission_denials` にあること、写しの `skills/` の中のファイルの結果がエラーでないこと（[ADR-20261015-001][adr-read] の確認方法、[SETUP-2][setup]。拒否が `permission_denials` に載ることは FACT-49 で確かめた）
  - 道具の一覧：初期化の記録（`init` のイベント）の一覧に、`mcp__` で始まる道具と `Agent`・`Task` が無く、MCP のサーバーの一覧が空であること（[ADR-20261017-005][adr-mcp] の決定 4。Agent は初期化の記録で `Task` の名前で出るので、両方を探す。[ADR-20261017-002][adr-review] の確認方法）。道具の名前は、名前の全体が一致するかで探す。一覧には `TaskStop` という別の道具があり、名前の前の部分で探すと取り違えるため（FACT-49）
- AI が決めたパスを読まなかったとき、ターンか時間か費用の上限に当たったとき、出力が解析できないときは fail にする。確かめられなかったものを pass と読まないため
- 終わったら、「止め方と利用枠の数え方」の手順で残りを確かめ（[AI のセッション][design-ai]）、セッションのフォルダと試験用のファイルを消す。費用は実行の記録に残し、1 日の利用枠に数える。結果は、Claude Code の版と、安全設定と環境変数の名前のハッシュと一緒に記録に残す（上の「組と回数」の契機に使う）
- 試しのセッションのフォルダが消せずに残ったら（途中で落ちた、残りが止まったと確かめられない）、名前から Issue か PR を引けない残骸として扱い、点検を fail にする。人が中身を確かめて消す（[状態遷移と判定][design-flow]の「作業場所の残骸」）

### PAT の権限

GitHub App に切り替えるまで（受け入れ工程（段階 4）で GitHub に書き始めるまで）、進行 Bot は fine-grained PAT で GitHub を読む。切り替えた後は、下の「GitHub App の権限」による（[ADR-20261020-002][adr-app]）。

進行 Bot の fine-grained PAT は、当てる先のリポだけに絞り、次の権限だけを付ける（[ADR-20261017-004][adr-human] が基本設計に残した、GitHub の点検に要る読み取りの権限を含む）。権限の名前は [GitHub の公式文書][doc-pat-permissions] による。

| 権限 | 読み取りか書き込みか | 使うところ |
| --- | --- | --- |
| Metadata（リポ） | 読み取り | ruleset を読む（`GET /repos/{owner}/{repo}/rulesets` は Metadata の読み取りで通る） |
| Issues（リポ） | 書き込み | ラベル、印とコメント、🟢 から Issue を作る |
| Pull requests（リポ） | 書き込み | PR を作る、ready にする、PR にコメントする |
| Commit statuses（リポ） | 読み取り | CI の結果（コミットの状態）。結合した状態（`/status`）は、状態が 0 件でも `pending` を返すので、CI が緑かの判定には使わず、必須チェックを名前で照らす（FACT-53） |
| Actions（リポ） | 読み取り | ワークフローの実行（人の承認が要るものを見つける。[IMPL-27][impl]） |
| Variables（リポ） | 読み取り | 全体の止める合図を読む。書き込みは付けない（[ADR-20261017-007][adr-stop]） |
| Projects（Organization） | 書き込み | Project の Status と Agent 欄（FACT-6） |

- Administration（ruleset を作る管理の権限）と Contents は付けない。ruleset は人の `setup` が作り（[SETUP-1][setup]）、push は deploy key で行う（[SAFE-15][safety]）
- CI のチェックの実行（check runs）は、fine-grained PAT に Checks の権限が無くても、公開のリポなら読める（FACT-52）。非公開のリポでは読めない（403。FACT-54）。非公開のリポは、GitHub App に切り替えた後に、アプリの Checks の読み取りで読む（[ADR-20261020-002][adr-app]）
- 書き込みを付けていないことは、点検用の名前の変数（`DEV_AUTOPILOT_PROBE`）を作る試しで確かめる。403 で断られれば pass。作れてしまったら fail にし、作った変数を消す。全体の止める合図の変数には書き込まない（読んだ後に人が `stop` に変えると、書き戻しで `run` に戻してしまうため）

### GitHub App の権限

受け入れ工程（段階 4）で `run.dryRun` を `false` にする（GitHub に書き始める）前に、進行 Bot を GitHub App に切り替える。印を書く前に替えるので、PAT のアカウントで書いた印は無く、読み続ける手当ては要らない（[ADR-20261020-002][adr-app]、[ADR-20261020-001][adr-bots]）。

| 権限 | 読み取りか書き込みか | 使うところ |
| --- | --- | --- |
| Metadata（リポ） | 読み取り | ruleset などを読む |
| Issues（リポ） | 書き込み | ラベル、印とコメント、🟢 から Issue を作る |
| Pull requests（リポ） | 書き込み | PR を作る、ready にする、PR にコメントする |
| Checks（リポ） | 読み取り | CI のチェックの実行（非公開のリポを含む。[IMPL-27][impl]） |
| Actions（リポ） | 読み取り | ワークフローの実行（[IMPL-27][impl]） |
| Commit statuses（リポ） | 読み取り | コミットの状態（結合した状態は判定に使わない。FACT-53） |
| Variables（リポ） | 読み取り | 全体の止める合図を読む。書き込みは付けない（[ADR-20261017-007][adr-stop]） |
| Projects（組織） | 書き込み | Project の Status と Agent 欄 |

- Contents の書き込みと Administration は付けない。push は deploy key で行い（[IMPL-15][impl]）、ruleset は人の `setup` が作る（[SETUP-1][setup]）。Webhook は使わない
- アプリの登録、当てる先のリポへの導入、秘密鍵の作成は、人が行う。`setup` は手順を示し、点検が確かめる（[SETUP-1][setup] の「人がやるもの」）
- 秘密鍵のファイルは秘密の置き場に置く（権限 600）。アプリの ID とインストールの ID は、リポの設定の `githubApp.appId` と `githubApp.installationId`（[設定][design-settings]）
- 進行 Bot は、実行のたびに秘密鍵で JWT を作り、1 時間で切れるインストールのトークンに引き換えて使う。切れる前に作り直す。Node の標準の機能（`node:crypto`）で作り、npm の依存を足さない（上の「依存」）
- 切り替えの順：人がアプリを登録して導入し、秘密鍵を秘密の置き場に置く → `githubApp.*` と `safety.botAccounts`（アプリの投稿者の名前）を、リポの設定の PR で直す → `dryRun` のまま、アプリで書けることと読めることを確かめる（[ADR-20261020-002][adr-app] の確認方法） → 人が GitHub で PAT を削除し、秘密の置き場の PAT のファイルを消す → `run.dryRun` を `false` にする PR をマージする
- 点検は、`run.dryRun` が `false` なのにアプリに切り替えていない（`githubApp.*` が空か、秘密鍵が無いか、PAT のファイルが残っている）とき fail にする。書き始める前に替える決まりを、人の記憶でなく点検で守るため

## 状態の表示

人が呼ぶ `/dev-autopilot` は、状態の表示と、手動の 1 回実行を行う（[STRUCT-3][struct] の構成）。進行 Bot を直接起動せず、GitHub は進行 Bot が自分の PAT で読む（[ADR-20261017-004][adr-human] の決定 2）。

- 進行 Bot は、毎回の実行の終わりに、記録の置き場の `status.json` を書き直す。中身は、実行の時刻、全体の止める合図と `run.dryRun`・`run.stage` の値、点検の fail と warn、基線の結果（緑でない必須チェックの名前）、Issue ごとの Status・Agent 欄・次の工程・止めた理由（stop の理由の記号）。点検だけの実行は、点検の結果だけを書き直す
- 計画・実装工程（段階 5）で Status が `In Review` の Issue は、Agent 欄が空なので、「人の番」と出す（[ADR-20261015-006][adr-flow] の決定 3）
- `/dev-autopilot` は、マシンの設定（[設定][design-settings]の「マシンの設定」）から記録の置き場を見つけ、`status.json` を読んで表示する。GitHub を読まないので、表示は最新の実行の時点のもので、実行の時刻を添える
- 手動の 1 回実行は、定期の実行と同じジョブを起こす（上の「定期の実行の登録」の手動の 1 回実行。`-k` を付けない）
- `status.json` は記録の置き場にあり、AI は sandbox の中から読めず、書けない

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
[adr-flow]: ../../adr/adr-20261015-006-run-stages-baseline-agent-field-and-local-records.md
[adr-workspace]: ../../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-review]: ../../adr/adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-bots]: ../../adr/adr-20261020-001-bot-accounts-as-list.md
[adr-app]: ../../adr/adr-20261020-002-switch-to-github-app-before-writing.md
