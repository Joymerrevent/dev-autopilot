# 導入と点検

人が呼ぶ `setup` と `check`、進行 Bot が毎回の実行の最初に回す点検、`/dev-autopilot` の状態の表示の、順番と仕組みを決める。
何を作り何を確かめるかは要件の [SETUP-1、SETUP-2][setup] が正本で、基本設計は順番・置き場・確かめ方を決める。

## setup の流れ

![setup の流れ][fig-setup-flow]

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 開いているプロジェクトの origin から owner/repo を引く。dev-autopilot 自身のリポなら、何も作らずに止まる（[STRUCT-3][struct]、[分離の時期の目安][struct]） | origin が無いか、GitHub のリポでなければ止まる |
| 2 | base ブランチの名前を決める。マシンの設定にあればその値、無ければ人に聞く（候補は origin の既定のブランチと `develop`）。次に、マシンの設定のフォルダ（[設定][design-settings]の「マシンの設定」）と、GitHub の今の状態を読み、足りないものの一覧を作る。base ブランチの先端のコミットで動いたチェックを、必須チェックの候補として一覧に入れる（下の「GitHub の準備」）。人の `gh` のログインが無ければ（専用マシン）、GitHub を読まず、マシンの準備だけを一覧にする | 人の `gh` のログインがあるのに GitHub を読めなければ止まる |
| 3 | 一覧を人に見せ、1 回だけ承認を受ける（[SETUP-1][setup]）。GitHub の準備と、このマシンで進行 Bot を動かすか（マシンの準備）を、分けて選べるようにする。人の手元のマシンで、専用マシンのために GitHub の準備だけを行うときは、マシンの準備を選ばない | 承認されなければ、何も作らない |
| 4 | GitHub の準備を、人の `gh` のログインで行い、作ったものを API で読み戻して確かめる（下の「GitHub の準備」）。人の `gh` のログインが無いか、3 で GitHub の準備を選ばなければとばす | 読み戻した値が違えば止まり、違いを示す |
| 5 | マシンの準備を、GitHub の認証を使わずに行う（下の「マシンの準備」）。秘密のファイルは作るだけで、中身を読まない。3 でマシンの準備を選ばなければとばす | 作れなければ止まり、作れたものを示す |
| 6 | 人がやることを、手順として示す（[SETUP-1][setup] の「人がやるもの」） | — |
| 7 | 定期の実行と同じジョブで点検だけを起こし、結果を示す（下の「点検だけで動かす仕組み」）。全体の止める合図が `stop` の間は、試しのセッションと書き込みの権限の試しを行わず、未確認の warn として示す。5 をとばしたときは行わない | fail があれば示す |
| 8 | 人が、点検の fail が無くなってから、全体の止める合図を `run` にする（[運用の規則][flow-rules]）。合図を `run` にした後の最初の実行が、作業の前に、試しのセッションと書き込みの権限の試しを行う（下の「組と回数」） | 試しが fail なら、最初の実行は作業に入らない |

- 何度実行しても同じ結果にする（冪等）。今あるものは作り直さず、設定ファイルの今の値は上書きしない。足りない項目だけを足す
- Plugin を更新した後に実行し直すと、プログラムの写しを新しい版で作る（下の「プログラムの写し」）
- 人の `gh` のログインが無いマシン（専用マシン）でも、順 1〜3（owner/repo を引く、足りないものの一覧、承認）は行う。dev-autopilot 自身のリポかを確かめ、マシンの準備の一覧に承認を受けるため（[SETUP-1][setup]）。順 2 で GitHub を読まず、順 4 の GitHub の準備をとばす。GitHub の準備は、人の手元のマシンで `setup` を実行し、マシンの準備を選ばずに行う（[ADR-20261017-004][adr-human] の決定 2）
- 専用マシンでも、`setup` を実行するために、当てる先のリポの作業コピー（Plugin を入れた `.claude/settings.json` を含む）を置き、そこで開いた Claude Code から呼ぶ（[STRUCT-3][struct]）。進行 Bot は作業コピーを読まない（[SAFE-2][safety]）
- 1 つの当てる先のリポで、定期の実行を登録するマシンは 1 台にする。lock はマシンの中の記録の置き場にあり、2 台が同じリポで動くと、重い工程・投稿・費用が重なるため（[運用の規則][flow-rules]）。動かすマシンを替えるときは、下の「動かすマシンを替えるとき」の順に、前のマシンの登録を外してから、新しいマシンで `setup` を実行する

### GitHub の準備

| 作るもの | 作り方 | 確かめ方 |
| --- | --- | --- |
| ラベル 5 つ（`agent-ready`・`needs-human`・`from-review`・`plan:fable`・`plan:opus`） | 無いものだけ作る | 名前で読み戻す |
| Project の Status の選択肢（`In Review`・`Ready to Merge`・`Blocked`）と、単一選択の Agent 欄（`worker`・`reviewer`・`human`・`paused`） | Status は、今ある選択肢を `id` 付きで含めて丸ごと送る（`id` を落とすと、今ある選択肢と項目の値が消える。FACT-15）。Agent 欄は無ければ作り、あって選択肢が足りなければ、Status と同じく今ある選択肢を `id` 付きで含めて足す | 選択肢の名前と `id` を読み戻し、リポの設定に書く |
| base ブランチの ruleset（PR 必須、force push の禁止、必須チェックを strict（base ブランチと最新であることを求める設定）で登録、`require_extra_approval_for_unattributed_changes` は無効） | 無ければ作る。必須チェックは、順 2 で一覧に入れた候補のうち、人が承認したものを登録する。あれば違いを示し、上書きしない。ただし、リポの設定の `checks.required` にあって ruleset に無い必須チェックは、承認を受けて足す（守りを強める向きの変更だけ） | 読み戻して項目を比べる（FACT-7）。点検は、左の欄の項目を 1 つずつ比べ、守りが弱い向きの違い（PR 必須でない、force push を許す、必須チェックが足りないか strict でない）を fail にする |
| deploy key の push 先を `<型>/issue-*` のブランチに限り、タグの作成を禁じる ruleset | 無ければ作る。あれば違いを示し、上書きしない | 読み戻して項目を比べる（[IMPL-15][impl]）。点検は、対象のブランチの形が `<型>/issue-*` より広いとき、タグの作成を許すときに fail にする |
| 全体の止める合図（リポの変数 `DEV_AUTOPILOT_RUN`） | 無ければ値を `stop` で作る。あれば値を変えない | 読み戻す（[ADR-20261017-007][adr-stop]） |
| リポの設定（`.claude/dev-autopilot.json`） | 表の初期値と、API で引いた Project の ID と選択肢の ID と、順 2 で承認した必須チェックの候補（`checks.required`）で作り、新しいブランチから base ブランチへの PR にする（base ブランチは PR 必須のため）。人がマージする。前の `setup` が作った PR がまだ開いていれば、新しく作らずにその PR を示す | PR を示す。進行 Bot は base ブランチの先端から読むので、マージされるまで点検は fail になる |

- Project の自動追加（組み込みのワークフロー）は公開の API が無いので、人が有効にする（[SETUP-1][setup]）。点検は有効かを読むだけ
- 必須チェックの候補は、base ブランチの先端のコミットで動いたチェック（名前、出した app、ワークフローのパス）から作る。候補が無ければ（CI がまだ無い）、必須チェックを登録せずに ruleset を作り、`checks.required` を空のままにする。点検が fail にするので、人が CI を足して `checks.required` に書き、`setup` を実行し直すと、ruleset に足す（[SETUP-1][setup]、[IMPL-27][impl]）。`setup` が足すので、人が ruleset を手で直さずに済む（[SETUP-4][setup]）

### マシンの準備

| 作るもの | 置き場 | 中身 |
| --- | --- | --- |
| マシンの設定 | マシンの設定のフォルダの `settings.json` | マシンの 14 項目（[設定][design-settings]）。`repository.*` は開いているプロジェクトの origin から、`baseBranch` は順 2 で決めた値を書く。`githubApp.*` は `null` で作る（切り替えのときに人が書く）。`safety.sandbox.toolDirs` は、`node`・`pnpm`・`git`・`gh` の実体の場所から書く |
| 置き場 | `paths.workspace` | `sessions/`、`sandbox-settings/`（権限 700）、`program/`（下の「プログラムの写し」） |
| 記録と lock の置き場 | `paths.records` | 作った日時を書いた記録の目印のファイル（`records-created.json`。[運用の規則][flow-rules]、[状態遷移と判定][design-flow]の「手元の記録」） |
| 秘密の置き場 | `paths.secrets` | フォルダを権限 700 で作り、[設定][design-settings]の「秘密の置き場のファイル」の表のファイルを、`githubApp.*` の値に合わせて、空のまま権限 600 で作る。値は人が書く（[SAFE-15][safety]） |
| 短い一時フォルダの根 | `sessionTmp.root` | 権限 700 で作る（[ADR-20261017-006][adr-tmp]） |
| 定期の実行の登録 | OS ごと | 下の「定期の実行の登録」 |

- 進行 Bot の clone（`<置き場>/controller/`）は、`setup` では作らない。clone には deploy key が要り、`setup` のマシンの準備は秘密のファイルを読まないため。進行 Bot が最初の実行で作る

## プログラムの写し

![プログラムの写し][fig-program-copy]

`setup` は、インストールされた Plugin の版を、置き場の中の版ごとのフォルダに写す。定期の実行は写しから動き、AI のセッションにも写しを読ませる（[STRUCT-3][struct]、[ADR-20261017-003][adr-plugin] の決定 2）。

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 実行の lock を取る（写しを作り直すのは、進行 Bot が動いていない間に限る。[STRUCT-3][struct]） | 取れなければ、何もせずに、後で実行し直すように示す |
| 2 | インストールされた Plugin（`${CLAUDE_PLUGIN_ROOT}`）の版を、`plugin.json` の `version` から読む。`program/<版>/` がすでにあれば、中身のハッシュ（すべてのファイルのパスと中身の SHA-256）を比べる | 同じなら順 3（別の名前で写す）と順 4（名前を変える）をとばし、順 5 から行う。違えば止まる（同じ版で中身が違う Plugin を、黙って上書きしない） |
| 3 | `program/.new-<版>/` に写し、写した中身のハッシュを、Plugin の中身のハッシュと比べる | 違えば、写したフォルダを消して止まる |
| 4 | `program/<版>/` に名前を変え、ファイルの書き込みの権限を外す。写した中身の一覧（パスと SHA-256）を、写しの外の `program/<版>.manifest.json` に書く | — |
| 5 | マシンの設定の `devAutopilotPath` を、使う版のフォルダに向け、定期の実行を登録し直す（下の「登録し直すとき」） | — |
| 6 | 今使う版と、直前まで使っていた版のほかの `program/<版>/` を消し、lock を外す | — |

- 別の名前で写してから名前を変えるのは、写している途中のフォルダを進行 Bot が使わないようにするため
- 直前まで使っていた版を残すのは、新しい版に問題があったときに、写し直さずに戻せるようにするため。`setup` を引数 `rollback` で実行すると、直前まで使っていた版の写しに戻す（順 1 の lock、順 5 の `devAutopilotPath` と定期の実行の登録を戻すこと、順 6 の片付けを行う）。入れた Plugin の版は新しいままなので、人が呼ぶ Skill は版の違いを warn に出す（作業は止めない）（[ADR-20261017-003][adr-plugin] が基本設計に残した、古い写しの片付け方）
- 人がマシンの設定の `devAutopilotPath` だけを戻しても、定期の実行の登録が新しい版のパスを持つので、動くプログラムは変わらない。点検が、登録のずれとして fail にする
- `claude plugin install` には、版を指定する引数が無い（Claude Code 2.1.285 の `claude plugin install --help` で確かめた）。入れた Plugin そのものを前の版に戻す手順は、配り方を決めるときに決める（[分離の時期の目安][struct]）。それまでは、上の `rollback` で写しだけを戻す
- 書き込みの権限を外すのは、人の誤った編集を防ぐため。AI は sandbox の中から、写しのどこにも書けない（[SAFE-16][safety]）
- 人が呼ぶ `setup`・`check`・`/dev-autopilot` は、入れた Plugin（`${CLAUDE_PLUGIN_ROOT}` の `plugin.json`）の版と写しの版が違えば warn に出す（[SETUP-2][setup]）。定期の実行で動く進行 Bot は、入れた Plugin の場所を知らないので、毎回の点検では、写しの中身が `program/<版>.manifest.json` の一覧とハッシュのとおりかを確かめ、違えば fail にする

### 依存

- 進行 Bot（`plugin/src/`）は、Node 24 の標準の機能（`node:` で始まるモジュール）と、`plugin/src/` の中のファイルだけを使い、実行時の npm の依存を持たない（[ADR-20261017-003][adr-plugin] の決定 1 が基本設計に残した選択）
- GitHub の読み書きは、子プロセスの `gh`（`gh api`）で行う。認証は、呼ぶたびに `GH_TOKEN` に、切り替えの前は PAT、切り替えた後はアプリのインストールのトークンを入れて渡す（[SAFE-15][safety]）。`gh` は人の手元に入れたものを使い、点検のマシンの組で版を確かめる。push と fetch は `git` で行う
- 写しに `node_modules` を作らず、`setup` は依存を入れない。npm の登録所への通信も、lifecycle script（パッケージを入れるときに走るスクリプト）も要らない
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
| `EnvironmentVariables` | `PATH`（`node`・`pnpm`・`git`・`gh`・`claude` の実体のフォルダと `/usr/bin:/bin:/usr/sbin:/sbin`）、`LANG` | launchd のジョブは人のシェルの設定を読まないので、道具の場所を渡す。秘密の値は書かない（点検が確かめる。[SETUP-2][setup]） |
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
- Linux では、sandbox の振る舞いを確かめ直すまで（UNDECIDED-49）、進行 Bot を動かさない（[STRUCT-8][struct]）。それまで `setup` は Linux で定期の実行を登録せず、`setup`・`check`・`/dev-autopilot` は、Linux では動かさないことを示して終わる。点検の OS ごとの部分は、動かしてよい OS（今は macOS）でなければ fail にするので、Linux で起こされた進行 Bot も作業に入らない（STRUCT-8 の「OS に依る処理は点検の OS ごとの部分に置く」）

### 登録し直すとき

- 写しの版が変わったとき（`devAutopilotPath` が変わる）と、`run.intervalHours` が変わったときは、`setup` で登録し直す
- 点検は、登録の中身（実行するファイルのパス、マシンの設定のパス、時刻）が、マシンの設定と `run.intervalHours` に合うかを確かめ、ずれていれば fail にする（[STRUCT-2][struct]、[設定][design-settings]）

### 動かすマシンを替えるとき

1 つの当てる先のリポで定期の実行を登録するマシンは 1 台なので（上の「setup の流れ」）、替えるときは次の順に行う。

| 順 | すること |
| --- | --- |
| 1 | 前のマシンで、`setup` を引数 `uninstall` で実行する。実行の lock を取り、定期の実行の登録を外す（macOS は `launchctl bootout`、Linux は `systemctl --user disable --now` でタイマーを止める）。写し・置き場・記録・秘密は消さない |
| 2 | 前のマシンの記録の置き場を、新しいマシンの同じキーの場所へ写す。記録は上限の数え上げに使うので、失ったものとして扱わない（[ADR-20261015-006][adr-flow] の決定 4 の「マシンを移すときは、記録のフォルダも移す」） |
| 3 | 新しいマシンで `setup` を実行する。秘密は、新しいマシンで人が置き直す（deploy key とアプリの秘密鍵）。`githubApp.*` もマシンの設定に書く |
| 4 | 新しいマシンで点検の fail が無いことを確かめる。前のマシンの置き場と秘密は、人が中身を確かめてから消す |

- 前のマシンの作業場所の残骸は、新しいマシンの点検には出ない（置き場はマシンごと）。前のマシンを使い続けないなら、人が消す

## 点検

![点検][fig-check]

点検の項目は、要件の [SETUP-2][setup] が正本。基本設計は、項目を組に分け、いつ・どう確かめるかを決める。結果は fail / warn / pass で、fail が 1 つでもあれば、進行 Bot は作業に入らない（[SETUP-2][setup]）。

### 組と回数

| 組 | 主な項目（[SETUP-2][setup]） | 確かめ方 | 毎回の実行の最初 | `setup`・人の `check` |
| --- | --- | --- | --- | --- |
| 設定 | 設定ファイルの項目と型、リポの設定に絶対パスが無いこと、項目が一方にだけあること、マシンの設定の場所の一致と、`repository.*` とマシンの設定のフォルダのパスの一致、`run.intervalHours` が 24 の約数であること、`safety.env.names` に渡さない名前と固定した名前が無いこと、作業 AI のモデルに `fable` が無いこと、`safety.sandbox.toolDirs` が人の store を含まないこと（一覧は[設定][design-settings]の「`check` が設定について確かめること」） | ファイルを読む | 確かめる | 確かめる |
| GitHub | ラベル、Project の欄と自動追加、リポの設定に書いた Project と選択肢の ID が実在すること、ruleset（上の「GitHub の準備」の表の項目）、必須チェックの一覧、全体の止める合図の変数、origin の base ブランチと、その先端のリポの設定、base ブランチの先端に `.claude/settings.local.json` が無いこと（FACT-56）、基線の必須チェック（warn） | 進行 Bot の GitHub の認証（切り替えの前は PAT、後はアプリのトークン）で読む。base ブランチの中身は進行 Bot の clone で読む | 確かめる | 確かめる |
| PAT かアプリの権限 | 下の「PAT の権限」か「GitHub App の権限」の読み取りが通ること。書き込みの権限が無いこと（リポの変数を作れないこと。切り替えの前は、ラベルも作れないこと）。投稿に使うアカウントが `safety.botAccounts` にあること（[ADR-20261020-001][adr-bots]）。`run.dryRun` が `false` なら、アプリに切り替えてあり、PAT のファイルが無いこと（[ADR-20261020-002][adr-app]） | 読み取りと、書き込みの権限の試し（下の「PAT の権限」。403 で断られれば pass） | 確かめる | 読み取りは確かめる。書き込みの権限の試しは、合図が `run` のときだけ |
| マシン | 動かしてよい OS か（今は macOS。UNDECIDED-49）、`claude` と Node 24 と `gh` の版、写しの中身が写したときの一覧とハッシュのとおりか（人が呼ぶ Skill では、入れた Plugin と写しの版の違いも warn）、写しと `controller/` が別のフォルダか、置き場と秘密の置き場の権限と実体のパス、短い一時フォルダの根の持ち主・リンクでないこと・権限 700（[AI のセッション][design-ai]の「起動の前の確かめ」の順 5）、記録と記録の目印のファイル、lock、見送った実行と起動しなかった定期の実行（warn）、作業場所の残骸、定期の実行の登録の中身と秘密の値が無いこと、進行 Bot の clone の objects のファイルのリンク数が 1 であること（ハードリンクで共有していない。FACT-19）、組み立てた安全設定の `allowRead` と `allowWrite` に共有の store が無いこと（[ADR-20261015-002][adr-store]） | ファイルとプロセスを読む。OS の部分は、点検の OS ごとの部分（[STRUCT-8][struct]） | 確かめる | 確かめる |
| 子プロセスと clone | 子プロセスに渡す環境変数に PAT・鍵・`GH_TOKEN`・`PNPM_HOME` が無いこと、objects を書き換えた clone からの fetch が失敗すること | 下の「モデルを使わない試し」 | 確かめる | 確かめる |
| sandbox と AI のセッション | 下の「試しのセッション」の「確かめる項目」の表（読めない・書けないこと、`git commit` と `pnpm install` が通ること、pnpm の store、安全設定の `env` が効くこと、Read ツールの拒否と写しの `skills/` の読み取り、worktree の `gitdir` のファイルに書けないこと、道具の一覧に MCP の道具と `Agent`・`Task`・`Skill` が無いこと、短い一時フォルダ） | 下の「試しのセッション」 | 下の契機に当たるときだけ | 合図が `run` のときだけ |

- 試しのセッションは利用枠を使うので、毎回は回さない。前の試しのセッションの記録が、次のすべてに当たるときだけ省き、前の結果（pass）をそのラウンドの sandbox の組の結果にする。1 つでも外れれば、作業の前に回す（[ADR-20261017-005][adr-mcp] と [ADR-20261017-006][adr-tmp] が基本設計に残した、確かめる回数）
  - 前の試しのセッションの結果が pass。fail・未確認・起動できなかった回の後は、毎回回し直す。前の結果を引き継いで fail のまま止まり続けず、fail を pass と読んで作業にも入らないため
  - `claude --version` の値が同じ。Claude Code の更新で、sandbox や道具の振る舞いが変わりうるため
  - 試しのセッションに渡す安全設定と環境変数の名前（リポの設定とマシンの設定の `safety.*` から組み立てたもの。セッションごとに変わるパスを除く）のハッシュが同じ。リポの設定の PR で、安全設定が変わりうるため
  - 写しの版が同じ。エージェント定義の道具の一覧、安全設定の雛形、試しのスクリプトが変わりうるため
  - OS の版（macOS は `sw_vers -productVersion`）が同じ。sandbox は OS の仕組み（Seatbelt）なので、OS の更新で振る舞いが変わりうるため
- 回すべきときに、試しのセッションを起動できない（費用の上限に当たった、短い一時フォルダの根の異常など）ときは、sandbox の組を fail にし、作業に入らない。合図が `stop` の間だけは、下のとおり未確認の warn にする
- 全体の止める合図が `stop` の間は、試しのセッションと書き込みの権限の試しを行わず、未確認として warn に出す。`run` を読めない間は、AI のセッションを起動せず、GitHub に書き込まないため（[ADR-20261017-007][adr-stop] の決定 2）。人は、fail が無くなってから合図を `run` にする。[ADR-20261017-007][adr-stop] の決定 1 は「`check` が pass になった後に」とするが、`stop` の間は 2 つの試しが未確認の warn になり pass にならないので、2 つを除いて fail が無いことを条件に読む。合図を `run` にした後の最初の実行が、作業の前に 2 つを確かめ、fail なら作業に入らない。受け入れ工程（段階 4）の最初は `run.dryRun` が `true` なので、最初の実行の作業も GitHub に書かない（[設定][design-settings]）
- 作業場所の残骸は、Issue に紐づく残骸を warn に出して残骸の Issue だけ `needs-human` にし、どの Issue か分からない残骸を fail にする（[SETUP-2][setup]）。見分け方は [状態遷移と判定][design-flow]の「作業場所の残骸」
- 基線が緑でなければ、緑でない必須チェックの名前を warn に出す。新しい着手・最初の実装・載せ直しは止まるが、Issue に `needs-human` を付けないので、人が気づく場所に出す（[IMPL-9][impl]、[ADR-20261015-006][adr-flow] の決定 2）

### 見送った実行と、起動しなかった定期の実行

lock で見送った実行と、マシンのスリープや停止で起動しなかった定期の実行を、記録に残して warn に出す（[運用の規則][flow-rules]）。受け入れ工程（段階 4）の出口の「7 回の実行」は、2 つを除き、定期の実行で起こされて作業に入った実行を数える（[段階の表][setup]、[状態遷移と判定][design-flow]の「段階の出口の数え方」）。

- lock で見送った実行は、lock を取れなかった実行が、見送ったことを記録に残す（[状態遷移と判定][design-flow]の「1 回の実行の流れ」の順 2）
- 実行の記録に、起こされ方（定期・手動・点検の依頼）を残す。人が呼ぶ `/dev-autopilot` の手動の 1 回実行は、ジョブを起こす前に記録の置き場に `run-request.json` を置き、`check` は `check-request.json` を置く。進行 Bot は、lock を取った後に 2 つのファイルを見て起こされ方を決める。どちらも無ければ定期。`run-request.json` は読んだら消し、`check-request.json` は下の「点検だけで動かす仕組み」のとおり、結果を書いた後に消す
- 起動しなかった定期の実行は、進行 Bot が、lock を取った後に、前の実行の起動時刻から今までの定期の実行の時刻（上の「定期の実行の登録」の「時刻」）を並べて見つける。時刻ごとに、次の時刻までの間に、起こされ方が定期の実行が始まっていなければ、起動しなかった時刻として記録に残す（同じ間に人が手で起こした実行があっても、定期の実行の代わりに数えない）。眠っていた間の分を起きたときに走らせた実行は、起きた時刻の直前の時刻の分に数える。眠っていた間のほかの時刻は、起動しなかった時刻になる
- 点検は、直近 7 日の分を warn に出す。7 日は、受け入れ工程（段階 4）の出口が 7 回の実行を数えるため

### 点検だけで動かす仕組み

人が呼ぶ `check` は、進行 Bot を直接起動せず、定期の実行と同じジョブを起こす。GitHub の点検は進行 Bot の GitHub の認証で行い、人のセッションは結果を読んで表示する（[SETUP-2][setup]、[ADR-20261017-004][adr-human] の決定 2）。点検だけで動かすことは、記録の置き場に置く依頼のファイルで伝える（[ADR-20261017-004][adr-human] が基本設計に残した仕組み）。

| 順 | すること |
| --- | --- |
| 1 | 人の `check` が、origin からマシンの設定を見つけ、記録の置き場に依頼のファイル（`check-request.json`。依頼の ID と時刻。形は「[状態遷移と判定][design-flow]」の「手元の記録」）を置く |
| 2 | 人の `check` が、定期の実行と同じジョブを起こす（上の「定期の実行の登録」の手動の 1 回実行） |
| 3 | 進行 Bot が、実行の lock を取った後、依頼のファイルがあれば、点検だけを行い、作業に入らない。全体の止める合図が `stop` でも、AI のセッションを起動せず GitHub に書き込まない項目は行う（人は点検の fail が無くなってから合図を `run` にするため）。試しのセッションと書き込みの権限の試しは、合図が `run` のときだけ行い、`stop` の間は未確認の warn として結果に書く（[ADR-20261017-007][adr-stop] の決定 2）。リポの設定を読めないなど、点検の途中で止まったときも、止まった項目を fail として結果に書く |
| 4 | 進行 Bot が、結果を `check-result-<依頼の ID>.json` に書き、依頼のファイルを消して終わる |
| 5 | 人の `check` が、結果のファイルを `run.checkWaitMinutes`（15 分）まで待って表示する。来なければ待つのをやめ、実行中か（lock を見る）、ジョブが起きなかったか（定期の実行の登録と `launchd.err.log`）を示して終わる |

- ジョブが動いている最中なら、手動の 1 回実行は新しく起こさない（`kickstart` に `-k` を付けない）。進行 Bot は、実行の終わり（「[状態遷移と判定][design-flow]」の「1 回の実行の流れ」の順 10）にも依頼のファイルを見て、あれば点検だけを行う。人の `check` は、実行中であることを示して待ち、`run.checkWaitMinutes` を過ぎたら、終わった後に `check` を実行し直すか、`/dev-autopilot` で結果を見るように示す
- 進行 Bot がマシンの設定を読めずに止まったときは、記録の置き場が分からず、結果を書けない。人の `check` は、待つ時間を過ぎたら、`launchd.err.log` を見るように示す
- 依頼と結果のファイルは記録の置き場にあり、AI は sandbox の中から書けない

### モデルを使わない試し

毎回の実行の最初に、進行 Bot が、AI のセッションを起動せずに試す（[SETUP-2][setup] の「進行 Bot は毎回の実行の最初に同じ点検を回し」）。リポの設定の PR で環境変数の名前が変わっても、次の実行で確かめるため。

- 子プロセスの環境変数：AI のセッションを起動するときと同じ処理で環境変数を組み立てる。名前が、決めた名前と `safety.env.names` の足す名前だけであること、`GH_TOKEN` と `PNPM_HOME` が無いこと、値に PAT かアプリの秘密鍵とほかの鍵の中身と、秘密の置き場のパスが無いことを確かめる（[SAFE-1][safety]、[SAFE-10][safety]、[AI のセッション][design-ai]の「環境変数」）
- objects を書き換えた clone からの fetch：一時フォルダに、試しのためだけの小さなリポ（commit 1 つ）と、小さなリポの clone を作り、clone の objects のファイルを 1 つ書き換える。小さなリポに、進行 Bot が取り込み（「[状態遷移と判定][design-flow]」の「push の手順」の順 2）で使うのと同じ形（`git fetch <パス> <ブランチ>`）と、同じ環境変数（`GIT_CONFIG_GLOBAL` などの git の設定）と `core.hooksPath` で、clone から fetch し、失敗することを確かめる（[ADR-20261013-001][adr-workspace] の確認方法。ローカルのパスからの fetch でも失敗することは FACT-57）。一時フォルダは試しの後に消す
  - 進行 Bot の clone を試しに使わない。進行 Bot の clone から clone すると objects のファイルがハードリンクで共有され、書き換えが進行 Bot の clone に及ぶうえ、試しが通ってしまったときに、書き換えた objects を取り込むため
- sandbox が読み書きを止めることと、道具の一覧は、モデルを使わずには試せないので、試しのセッションで確かめる

### 試しのセッション

- AI のセッションと同じ起動の形（安全設定、環境変数、引数。[AI のセッション][design-ai]）で、短いセッションを 1 つ起こす。名前は `check-<日時>`（[AI のセッション][design-ai]の「セッションの名前」）
- エージェント定義は、作業 AI（`dev-autopilot:change:worker`）を使う。道具がいちばん広い役で、境界を試すため
- モデルは `claude-haiku-5-5`（`haiku` の別名は旧版に解けるので、完全な ID で書く。[IMPL-29][impl]）、ターンの上限は 10、時間の上限は 10 分、費用の上限は `limits.session.costUsd`（試しのスクリプトの実行と 2 つの読み取りは、4 ターン・7 秒で終わった。FACT-49）。モデルとターンと時間の上限は、試しの中身と一緒に Plugin の版で変えるので、設定に持たず、進行 Bot のプログラムに書く
- 作業場所は、ほかの役と同じく、当てる先のリポを origin から clone し、進行 Bot の clone で控えた base ブランチの先端の SHA で worktree を切る（作業 AI と同じ。[AI のセッション][design-ai]の「役とエージェント定義」）。`pnpm install` も当てる先のリポのロックファイルで試す。本物のセッションと同じ条件で確かめるため
- 進行 Bot は、セッションを起こす前に、ほかのセッションの代わりの試験用のフォルダを 2 つ作る（`<置き場>/sessions/check-other-<日時>/` と `<sessionTmp.root>/check-other-<日時>/`。中身は意味の無い文字列のファイル 1 つ）。名前と ID の組を記録に残してから作り、試しの後に消す（残骸と見分けるため）
- 進行 Bot が、試しのスクリプト（Node で動く。`plugin/src/` の中に持つ）をセッションのフォルダに置き、AI に 1 回だけ実行させる。スクリプトは、下の表の項目を順に試し、結果をセッションのフォルダに書く。結果には、読めたかどうかだけを書き、読めたファイルの中身を書かない
- Read ツールの試しは、AI が道具を呼ばないと行えないので、AI に、決めたパスを Read ツールで 1 つずつ読ませる。パスは、進行 Bot がセッションのフォルダの外（`<置き場>/check-probe.txt`。中身は意味の無い文字列）に置いた試験用のファイルと、写しの `skills/` の中のファイル。本物の秘密のファイルは Read ツールで読ませない（拒否されなかったとき、中身が AI と記録に渡るため）
- 確かめる項目と、期待する結果と、判定の元

  | 項目 | 期待する結果 | 判定の元 |
  | --- | --- | --- |
  | ホームのファイル、`<置き場>/controller/`、写しの `src/`、試験用のほかのセッションのフォルダ 2 つ、秘密の置き場、記録の置き場を Bash で読む | すべて読めない | スクリプトの結果 |
  | 自分の clone と worktree、写しの `skills/` を Bash で読む | 読める | スクリプトの結果 |
  | 写し、`<置き場>/controller/`、worktree の `gitdir` のファイル、ホームに Bash で書く | すべて書けない（ファイルができない） | スクリプトの結果と、進行 Bot がディスクで確かめた有無 |
  | `$TMPDIR` | `<短い一時フォルダ>/claude-<uid>` で、書ける | スクリプトの結果 |
  | `/tmp/claude-<uid>/` の一覧を出す | 出せない | スクリプトの結果 |
  | worktree で `git commit --allow-empty`、`pnpm install --frozen-lockfile`、`pnpm store path` | commit と install が通り、store のパスがセッションの `tmp/` の下 | スクリプトの結果 |
  | 安全設定の `env`（`NODE_USE_ENV_PROXY`、`npm config get cache`） | `1` と、`$TMPDIR` の下（FACT-47） | スクリプトの結果 |
  | 試験用のファイルと、写しの `skills/` の中のファイルを Read ツールで読む | 前者は拒否、後者は読める | `stream-json` の Read の結果と `permission_denials` |
  | 道具の一覧と MCP のサーバーの一覧 | `mcp__` の道具と `Agent`・`Task`・`Skill` が無く、サーバーが空 | `init` のイベント |

- 進行 Bot は、AI の返事ではなく、次で判定する
  - スクリプトが書いた結果のファイルと、書けないはずの場所にファイルができていないこと
  - Read ツール：`stream-json` の Read の呼び出しと結果を照らし、試験用のファイルが `result` の `permission_denials` にあること、写しの `skills/` の中のファイルの結果がエラーでないこと（[ADR-20261015-001][adr-read] の確認方法、[SETUP-2][setup]。拒否が `permission_denials` に載ることは FACT-49 で確かめた）
  - 道具の一覧：初期化の記録（`init` のイベント）の一覧に、`mcp__` で始まる道具と `Agent`・`Task`・`Skill` が無く、MCP のサーバーの一覧が空であること（[ADR-20261017-005][adr-mcp] の決定 4。Agent は初期化の記録で `Task` の名前で出るので、両方を探す。[ADR-20261017-002][adr-review] と [ADR-20261021-001][adr-review-skill] の確認方法）。道具の名前は、名前の全体が一致するかで探す。一覧には `TaskStop` という別の道具があり、名前の前の部分で探すと取り違えるため（FACT-49）
- AI が決めたパスを読まなかったとき、ターンか時間か費用の上限に当たったとき、出力が解析できないときは fail にする。確かめられなかったものを pass と読まないため
- 終わったら、「止め方と利用枠の数え方」の手順で残りを確かめ（[AI のセッション][design-ai]）、セッションのフォルダと試験用のファイルを消す。費用は実行の記録に残し、1 日の利用枠に数える。結果は、Claude Code の版と、安全設定と環境変数の名前のハッシュと一緒に記録に残す（上の「組と回数」の契機に使う）
- 試しのセッションのフォルダが消せずに残ったら（途中で落ちた、残りが止まったと確かめられない）、名前から Issue か PR を引けない残骸として扱い、点検を fail にする。人が中身を確かめて消す（[状態遷移と判定][design-flow]の「作業場所の残骸」）

### PAT の権限

GitHub App に切り替えるまで（受け入れ工程（段階 4）で GitHub に書き始めるまで）、進行 Bot は fine-grained PAT で GitHub を読む。切り替えた後は、下の「GitHub App の権限」による（[ADR-20261020-002][adr-app]）。

進行 Bot の fine-grained PAT は、当てる先のリポだけに絞り、次の権限だけを付ける（[ADR-20261017-004][adr-human] が基本設計に残した、GitHub の点検に要る読み取りの権限を含む）。権限の名前は [GitHub の公式文書][doc-pat-permissions] による。

| 権限 | 読み取りか書き込みか | 使うところ |
| --- | --- | --- |
| Metadata（リポ） | 読み取り | ruleset を読む（`GET /repos/{owner}/{repo}/rulesets` は Metadata の読み取りで通る） |
| Issues（リポ） | 読み取り | Issue、ラベル、コメント、timeline を読む。書くのはアプリに切り替えた後 |
| Pull requests（リポ） | 読み取り | PR と、PR のコメントを読む |
| Commit statuses（リポ） | 読み取り | CI の結果（コミットの状態）。結合した状態（`/status`）は、状態が 0 件でも `pending` を返すので、CI が緑かの判定には使わず、必須チェックを名前で照らす（FACT-53） |
| Actions（リポ） | 読み取り | ワークフローの実行（人の承認が要るものを見つける。[IMPL-27][impl]） |
| Variables（リポ） | 読み取り | 全体の止める合図を読む。書き込みは付けない（[ADR-20261017-007][adr-stop]） |
| Projects（Organization） | 読み取り | Project の Status と Agent 欄を読む（FACT-6） |

- PAT を使うのは、書かずに試す間（`run.dryRun` が `true`）の読み取りだけなので、書き込みの権限を付けない（[ADR-20261020-002][adr-app] の決定 1）。`run.dryRun` の判定に誤りがあっても、メンテナのアカウントで印を書けない。書き込みを付けるのは、アプリの権限（下の「GitHub App の権限」）だけ
- Administration（ruleset を作る管理の権限）と Contents は付けない。ruleset は人の `setup` が作り（[SETUP-1][setup]）、push は deploy key で行う（[SAFE-15][safety]）
- CI のチェックの実行（check runs）は、fine-grained PAT に Checks の権限が無くても、公開のリポなら読める（FACT-52）。非公開のリポでは読めない（403。FACT-54）。非公開のリポは、GitHub App に切り替えた後に、アプリの Checks の読み取りで読む（[ADR-20261020-002][adr-app]）
- 書き込みの権限の試し：点検用の名前の変数（`DEV_AUTOPILOT_PROBE`）を作る試しと、切り替えの前は、点検用の名前のラベル（`dev-autopilot-probe`）を作る試しを行う。403 で断られれば pass。作れてしまったら fail にし、作った変数かラベルを消す。全体の止める合図の変数には書き込まない（読んだ後に人が `stop` に変えると、書き戻しで `run` に戻してしまうため）
- 書き込みの権限の試しは、書けないことを確かめる試しなので、`run.dryRun` が `true` の間も行う（書くつもりの内容を記録に回す決まりの例外）。アプリはラベルを作れるので、切り替えた後は変数の試しだけを行う

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
- 秘密鍵のファイル（`github-app-key.pem`）は秘密の置き場に置く（権限 600）。アプリの ID とインストールの ID は、マシンの設定の `githubApp.appId` と `githubApp.installationId`（[設定][design-settings]。全体の止める合図を読む前に要るため）
- 進行 Bot は、実行のたびに秘密鍵で JWT（アプリが自分を証明する、署名付きの短い文字列）を作り、1 時間で切れるインストールのトークンに引き換えて使う。`gh` を呼ぶ前に残りの時間を見て、10 分を切っていれば作り直す（AI のセッションの時間の上限 45 分の間に読む全体の止める合図の読み取りでも切れないように、余裕を持たせる）。10 分は GitHub のトークンの有効期間で決まる値なので、設定に持たず、プログラムに書く。Node の標準の機能（`node:crypto`）で作り、npm の依存を足さない（上の「依存」）
- アプリの投稿者の名前は、REST の `login` では `<アプリの名前>[bot]` になり、GraphQL では形が違うことがある（確かめていない）。取り方と比べ方は、切り替えの前の試し（下の切り替えの順の 3）で決め、点検と印を読む処理で同じ形にそろえる（[ADR-20261020-002][adr-app] の決定 4 と確認方法）
- 切り替えの順

  | 順 | すること |
  | --- | --- |
  | 1 | 人がアプリを登録して当てる先のリポに導入し、秘密鍵を秘密の置き場の `github-app-key.pem` に置く |
  | 2 | 人が、進行 Bot を動かすマシンの設定に `githubApp.*` を書く。`safety.botAccounts` をアプリの投稿者の名前にする PR を、リポの設定に出してマージする |
  | 3 | 一度だけ、試しのためだけのリポと試しの鍵で、アプリで書けることを確かめ（コメント・ラベル・Project の欄・PR）、投稿者の名前の形と、トークンを作り直す処理を確かめる。結果は FACT に残す（[ADR-20261020-002][adr-app] の確認方法） |
  | 4 | 当てる先のリポでは、`dryRun` のまま、点検の読み取りと書き込みの権限の試し（変数を作れないこと）を通す。書けることは、`dryRun` を切った最初の実行の書き込みと読み戻し（「[状態遷移と判定][design-flow]」の「push の手順」の順 8 など）で確かめる |
  | 5 | 人が GitHub で PAT を削除し、秘密の置き場の `github-pat` を消す |
  | 6 | `run.dryRun` を `false` にする PR をマージする |

- 点検は、`run.dryRun` が `false` なのにアプリに切り替えていない（`githubApp.*` が `null` か、秘密鍵が無いか、PAT のファイルが残っている）とき fail にする。書き始める前に替える決まりを、人の記憶でなく点検で守るため

## 状態の表示

人が呼ぶ `/dev-autopilot` は、状態の表示と、手動の 1 回実行を行う（[STRUCT-3][struct] の構成）。進行 Bot を直接起動せず（手動の 1 回実行は、`run-request.json` を置いてから同じジョブを起こす）、GitHub は進行 Bot が自分の GitHub の認証で読む（[ADR-20261017-004][adr-human] の決定 2）。

- 進行 Bot は、毎回の実行の終わりに、記録の置き場の `status.json` を書き直す。中身は、実行の時刻、全体の止める合図と `run.dryRun`・`run.stage` の値、点検の fail と warn、基線の結果（緑でない必須チェックの名前）、Issue ごとの Status・Agent 欄・次の工程・止めた理由（stop の理由の記号）。点検だけの実行は、点検の結果だけを書き直す
- 計画・実装工程（段階 5）で Status が `In Review` の Issue と、レビュー工程（段階 6）で AI のレビューが直すべき点を返した Issue は、Agent 欄が空なので、「人の番」と出す（[ADR-20261015-006][adr-flow] の決定 3、[Issue の状態の遷移][flow-rules]の「段階による違い」）
- 書かずに試す間（`run.dryRun` が `true`）は、最新の実行の記録の、書くつもりだった内容（受け入れの下書きなど）の一覧も出す。人が JSON Lines を直接読まずに済むようにするため
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
[adr-review-skill]: ../../adr/adr-20261021-001-review-skill-text-without-skill-tool.md
[adr-store]: ../../adr/adr-20261015-002-no-shared-dependency-store.md
