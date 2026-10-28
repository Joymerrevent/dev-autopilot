# 設定

進行 Bot と、人が呼ぶ Skill（`setup`・`check`・`/dev-autopilot`・`intake`。`intake` は `/verify-issue` などから呼ぶ受け入れの Skill）が読む設定の、ファイルの置き場・形・項目を決める。
設定は、リポの設定とマシンの設定の 2 つのファイルに分け、項目ごとに置き場をどちらか一方だけにする（STRUCT-2、[ADR-20261015-005][adr-settings] の決定 2）。

## どの項目がどちらにあるか

| 設定 | ファイル | 置く場所 | 持つ項目 |
| --- | --- | --- | --- |
| リポの設定 | `.claude/dev-autopilot.json` | 当てる先のリポの中（commit する） | 項目の大半：Project、ラベル名と固定の見出し、進行 Bot の実行（定期の実行の間隔など）、上限、モデル、検査と判定、規約、安全（印を書いたと認めるアカウント名の一覧・通信の許可先・子プロセスに渡す環境変数の名前など） |
| マシンの設定 | `settings.json` | リポの外の、OS ごとの決まった場所。当てる先のリポごとにフォルダを分ける | 17 項目。リポの設定を読む前に要る 5 項目（当てる先のリポの owner と名前、base ブランチの名前、GitHub App の ID とインストールの ID）と、マシンごとに違う 12 項目（進行 Bot のプログラムの写しのパス（`devAutopilotPath`）、直前の版と版を固定したか（`program.*`）、フォルダのパス 4 つ（置き場・記録の置き場・秘密の置き場・安全設定の置き場）、ホームの中の道具のフォルダ、子プロセスに足す環境変数の値、短い一時フォルダの根・ID の長さ・長さの上限） |

マシンの設定をリポの中に置かない理由は 3 つある。

- マシンのパス（ユーザー名を含む）を公開のリポに入れない（[ADR-20261015-005][adr-settings] の決定 2）
- gitignore して人の作業コピーに置くと、人がふだん使う Claude Code が書き換えられる。`devAutopilotPath` を書き換えられると、PAT（GitHub App に切り替えた後はアプリの秘密鍵）と deploy key を持つ定期の実行が別のプログラムを起こす。進行 Bot は人の作業コピーを読まない（[ADR-20261017-007][adr-stop] の決定 4、SAFE-2）
- 人の作業コピーに置くと、定期の実行が作業コピーの場所に縛られ、人が作業コピーを動かすか消すと止まる

## 値の決まり方

- 要件が初期値を決めている項目も、表に値を書き、出典の要件へリンクする。値の正本は要件で、食い違っていたら要件が正しい。要件の初期値を変える PR では、表も直す（[ADR-20261018-005][adr-table]）
- 要件が初期値を決めていない項目は、表に初期値と根拠を書き、「暫定」と添える。運用の記録（FLOW-5、FLOW-13。「[状態遷移と判定][design-flow]」の「運用の記録の数え方」）で見直す（[ADR-20261015-005][adr-settings] の決定 1）
- `setup` は、表の初期値のうち当てる先のリポに依らない値と、人に聞いた規約（言語・Conventional Commits・changeset・規約の正典のパス）で設定ファイルを作る。表に business-os の値として書いた初期値（許可一覧と方針のパス）は、`setup` のプログラムに持たず（[STRUCT-2][struct]）、人が `setup` の作るリポの設定の PR に書き足す。今の値の正本は設定ファイル。初期値の正本は、要件が決めた値は要件、それ以外は表
- リポの設定の値は PR で変える。マシンの設定の値は、そのマシンで人が書き換える。安全の境界を緩める変更は、値の変更でも ADR で決める（[設計を変えるとき][design-change]）

## リポの設定

- 当てる先のリポの `.claude/dev-autopilot.json`（commit する）。形は JSON
- 進行 Bot は、毎回の実行の最初に、進行 Bot の clone で fetch した base ブランチの先端から読む。読めないか形が違えば、作業に入らず止まる（[ADR-20261017-007][adr-stop] の決定 4、SAFE-4）
- マシンに依らない値だけを持ち、絶対パスを書かない（`check` が確かめる）
- AI のセッションは、人が `## 仕組みの変更の承認` で認めなければ書き換えられない。`.claude/**` は IMPL-15 の変えてはいけないパスのうち、マージまで効かない組で、書き換えても人がマージするまで効かない（進行 Bot は base ブランチの先端から読む）。認めても、`.claude/**` は作業 AI に書かせず、人が変える（Claude Code 自身も作業 AI に書かせない。FACT-77、[ADR-20261027-003][adr-claude-dir]）

## マシンの設定

- マシンの設定のフォルダの `settings.json`（commit しない）。形は JSON
- マシンの設定のフォルダは、OS ごとの決まった場所に、当てる先のリポ（owner/repo）ごとに置く（[ADR-20261017-004][adr-human] の決定 3）

  | OS | マシンの設定のフォルダ |
  | --- | --- |
  | macOS | `~/Library/Application Support/dev-autopilot/<owner>/<repo>/` |
  | Linux | `$XDG_CONFIG_HOME/dev-autopilot/<owner>/<repo>/`（`XDG_CONFIG_HOME` が無ければ `~/.config`） |

- macOS で `~/.config` を使わないのは、手元の Mac では `~/.config` が同期フォルダへのリンクのため（[ADR-20261015-001][adr-read]）。Linux でも実体のパスが同期フォルダの下なら、`check` が fail にする
- 人が呼ぶ Skill は、開いているプロジェクトの origin から owner/repo を引いて、マシンの設定のフォルダを見つける。定期の実行の定義（macOS は launchd の plist）は、同じ `settings.json` のパスを進行 Bot に渡す。`check` が 2 つの一致を確かめる（STRUCT-2）
- AI のセッションの `denyRead` に入れる（SAFE-10）

## マシンの中のフォルダ

`setup` は、次の既定のパスでフォルダを作り、マシンの設定に書く。人が別の場所を指定したときは、指定した値を書く。

基本設計で「置き場」とだけ書くときは、`paths.workspace` を指す（[ADR-20261013-001][adr-workspace] の `<置き場>`）。ほかのフォルダは、「記録の置き場」「秘密の置き場」のように、何の置き場かを添えて呼ぶ。

| フォルダ | キー | macOS | Linux | 中身 |
| --- | --- | --- | --- | --- |
| 置き場 | `paths.workspace` | `~/.local/share/dev-autopilot/<owner>/<repo>/` | `$XDG_DATA_HOME/dev-autopilot/<owner>/<repo>/`（`XDG_DATA_HOME` が無ければ `~/.local/share`） | 進行 Bot の clone（`controller/`）、セッションごとのフォルダ（`sessions/<セッションの名前>/`）、進行 Bot のプログラムの写し（`program/<版>/`）、GitHub の SSH のホストの鍵（`known_hosts`。下の「秘密の置き場のファイル」）、安全設定の置き場の既定の場所（`sandbox-settings/`） |
| 記録の置き場 | `paths.records` | `<マシンの設定のフォルダ>/records/` | `$XDG_STATE_HOME/dev-autopilot/<owner>/<repo>/records/`（`XDG_STATE_HOME` が無ければ `~/.local/state`） | 実行の記録（FLOW-5）、lock、`setup` が置く記録の目印のファイル（`records-created.json`）。ファイルの一覧は「[状態遷移と判定][design-flow]」の「手元の記録」（「[運用の規則][flow-rules]」） |
| 秘密の置き場 | `paths.secrets` | `<マシンの設定のフォルダ>/secrets/` | `<マシンの設定のフォルダ>/secrets/` | 下の「秘密の置き場のファイル」。フォルダの権限は 700、ファイルは 600（[ADR-20261015-001][adr-read] の決定 2、[ADR-20261020-002][adr-app]） |
| 短い一時フォルダの根 | `sessionTmp.root` | `/tmp/dap` | `/tmp/dap` | セッションごとの短い一時フォルダ（[ADR-20261017-006][adr-tmp]） |
| 安全設定の置き場 | `paths.sandboxSettings` | `<paths.workspace>/sandbox-settings/` | `<paths.workspace>/sandbox-settings/` | AI のセッションごとの安全設定のファイル（[AI のセッション][design-ai]の「セッションごとの安全設定」）。フォルダの権限は 700 |

表のパスは、どれも要件に値が無いので暫定（[ADR-20261015-005][adr-settings] の補足情報）。

- 置き場を macOS で `~/Library/Application Support/` の下に置かないのは、パスに空白が入るため。AI は置き場の中の worktree で品質ゲート（`pnpm check`）とパッケージのスクリプトを動かし、空白を含むパスで失敗する道具がある
- 記録と秘密の置き場は、AI が道具を動かさないので、macOS ではマシンの設定のフォルダの中に並べる（秘密の置き場をマシンの設定と並べてよいことは [ADR-20261015-005][adr-settings] の決定 2）。Linux の記録は、XDG の決まり（状態のファイルは `XDG_STATE_HOME`）に合わせる
- どのフォルダも、`~/.claude/` と `/tmp/claude-<uid>/` の下に置かない（SAFE-10）。置き場のほかのフォルダは、AI が書ける場所（置き場の `sessions/` の下と、短い一時フォルダの根の下）にも置かない。`check` は、実体のパスがどれにも当たらず、同期フォルダの下にも無いことを確かめる（[ADR-20261015-005][adr-settings] の決定 2。下の「`check` が設定について確かめること」）
- 短い一時フォルダの根 `/tmp/dap` は、[ADR-20261017-006][adr-tmp] の例のまま。`<根>/<ID>/claude-<uid>` を上限（`sessionTmp.maxBytes`）の 44 バイト以内に収めるため、根は短くする。長さは、環境変数に渡す実体のパスで数える（[AI のセッション][design-ai]の「短い一時フォルダ」）。macOS の `/tmp` は `/private/tmp` へのリンクなので、実体のパスは `/private/tmp/dap/<ID>/claude-<uid>`（ID が 8 文字なら 33 バイトと uid の桁数）で、uid は 11 桁まで収まる。Linux の `/tmp/dap` なら 19 桁まで

### 秘密の置き場のファイル

`setup` は、次のファイルを、`githubApp.*` の値に合わせて空で作る（権限 600）。値は人が書く（[SAFE-15][safety]）。進行 Bot と点検は、空のファイルを「無い」と読む。

| ファイル | 中身 | `setup` が空で作るとき | 動かすマシンを替えるとき |
| --- | --- | --- | --- |
| `github-pat` | 進行 Bot の fine-grained PAT（GitHub App に切り替えるまで。読むだけ。「[導入と点検][design-setup]」の「PAT の権限」） | `githubApp.appId` と `githubApp.installationId` が空のときだけ。切り替えた後は作らない（作ると、点検の「PAT のファイルが無いこと」が fail になるため） | 新しく作る（人が新しい PAT を発行して書く）。前のマシンの PAT は、新しいマシンの最初の実行が通った後に GitHub で消す |
| `github-app-key.pem` | GitHub App の秘密鍵（切り替えた後。[ADR-20261020-002][adr-app]） | `githubApp.appId` と `githubApp.installationId` があるときだけ | 新しく作る（人がアプリの新しい秘密鍵を作って書く）。前のマシンの秘密鍵は、新しいマシンの最初の実行が通った後に GitHub で取り消す |
| `deploy-key` | 作業ブランチに push する deploy key の秘密鍵（[IMPL-15][impl]） | いつも | 新しく作る（人が新しい鍵を作り、deploy key として GitHub に登録する）。前のマシンの deploy key は、新しいマシンの最初の実行が通った後に GitHub から消す |
| `marker-signing-keys` | 印の署名の鍵の一覧（[IMPL-20][impl]。1 行に 1 つ。印は先頭の行の鍵で署名し、どの行の鍵でも確かめる。形と読み方と替え方は「[状態遷移と判定][design-flow]」の「印の署名」） | いつも | 前のマシンから一覧を写す（行の順も変えない）。新しいマシンで鍵を作るときは、鍵を替える手順（「印の署名」）のとおり、新しい鍵を先頭の行に足し、前の鍵を残す。前の鍵を外すと、前の鍵で書いた印が署名の条件を満たさなくなり、開いている Issue と PR が `marker-invalid` で止まるため（[ADR-20261021-004][adr-marker-keys] の決定 2） |

- 秘密の値を、設定ファイル・定期の実行の定義・記録に書かない。進行 Bot は起動したときに読み、`gh` を呼ぶときだけ `GH_TOKEN`、push と fetch のときだけ `GIT_SSH_COMMAND` で渡す（[SAFE-15][safety]）
- `setup` が示す秘密の作り方（印の署名の鍵なら `openssl rand -base64 32` など）は、初めて導入するときのもの。`setup` は、印の署名の鍵の一覧を「初めて導入するときは鍵を 1 つ作って 1 行目に書き、動かすマシンを替えるときは前のマシンから一覧を写す」と分けて示す（上の表の右の欄。「[導入と点検][design-setup]」の「動かすマシンを替えるとき」）
- `GIT_SSH_COMMAND` は進行 Bot のプログラムに固定し、設定に持たない。`ssh` を名前で呼び（定期の実行の定義の `PATH` から見つける）、`-F /dev/null`（人の `~/.ssh/config` を読まない）、`-i <秘密の置き場>/deploy-key`、`-o IdentitiesOnly=yes`（渡した鍵のほかを出さない）、`-o IdentityAgent=none`（ssh-agent の鍵を使わない）、`-o UserKnownHostsFile=<置き場>/known_hosts`、`-o StrictHostKeyChecking=yes` を付ける。人の鍵で push して、deploy key の push 先を限る ruleset（[IMPL-15][impl]）を外れないため。`GIT_SSH_COMMAND` と同じ形の ssh で、deploy key の push 先が ruleset で限られることを確かめた（FACT-65）
- `<置き場>/known_hosts` は、進行 Bot が、ファイルが無ければ最初の push か fetch の前に、GitHub の API の `meta` が返す SSH のホストの鍵から書く。`meta` は、ほかの GitHub の読み取りと同じく、子プロセスの `gh api meta` に進行 Bot の認証（`GH_TOKEN`）を渡して読む（「[導入と点検][design-setup]」の「依存」）。`setup` のマシンの準備は GitHub の認証を使わず、`gh` は認証が無いと公開の API も読まないので、進行 Bot の clone と同じく `setup` では書かない。結合テストでは、偽の `gh` が `meta` に答えるので、本物の GitHub に触れない。書いた後は、進行 Bot は書き直さない。ファイルが GitHub のホストの鍵と合わなければ、`ssh` が止まり、push と fetch は実行の失敗になる（止まる側に倒れる）。GitHub がホストの鍵を替えたときは、人がファイルを消し、進行 Bot が次の実行で書き直す

## 設定ファイルの外の合図

- 全体の止める合図は、当てる先のリポの Actions の変数 `DEV_AUTOPILOT_RUN`。値が `run` のときだけ動き、ほかの値・変数が無い・読めないときは止まる（「[運用の規則][flow-rules]」、[ADR-20261017-007][adr-stop]）
- 変数の名前は設定に持たない。進行 Bot は、リポの設定を読む前の実行の最初に合図を読むため。合図を読む先（当てる先のリポの owner と名前）と、読むときの認証（GitHub App の 2 つの ID）も、同じ理由でマシンの設定に置く（[STRUCT-2][struct]、[ADR-20261021-006][adr-repo]）

## 項目

キーは、ファイルの中の JSON の位置を `.` でつないで書く（例：`limits.session.minutes`）。`check` は、形の欄の型（数・文字列・真偽・一覧・対応表）と範囲で値を確かめ、表に無いキーがあれば fail にする（SETUP-2。下の「[`check` が設定について確かめること][check-settings]」）。
要件が決めた初期値には、出典の要件へのリンクを添える。正本は要件で、食い違っていたら要件が正しい（[ADR-20261018-005][adr-table]）。

### リポの設定の項目

#### Project

| キー | 中身 | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- | --- |
| `project.number` | Issue を並べる GitHub の Project の番号 | 数（1 以上の整数） | `setup` が使う Project の番号を書く | 進行 Bot は、設定した Project の Issue を読んで作業の単位にする（[FLOW-1][flow]）。表示の欄は `setup` が足す（[SETUP-5][setup]） |
| `project.fields.frame`、`project.fields.state`、`project.fields.stop`、`project.fields.dependency`、`project.fields.safety` | Project の表示の欄（主な流れの枠・状態・止め方・先の依頼・安全）ごとの、欄の ID と選択肢の ID | 対応表。欄ごとに `id`（欄の ID の文字列）と `options`（キーは選択肢の名前、値は選択肢の ID の文字列）。選択肢の名前は [FLOW-39][flow] の表の値で、（なし）の選択肢は持たない | `setup` が足した欄と選択肢の ID を書く | 進行 Bot は、主な流れと軸ごとに 1 つの欄を Project に書く。表示だけで、判定には読まない（[FLOW-39][flow]、[SETUP-5][setup]）。状態の欄（`project.fields.state`）は、もともとある Status 欄を名前を変えずに使う。主な流れの枠の欄（`project.fields.frame`）は、`setup` が単一選択の欄として新しく作る（Status 欄の名前は API で変えられない。FACT-74、[ADR-20261026-001][adr-status]）。軸の値が（なし）のときは、選択肢を選ばずに欄を空にする。人は、ビューで欄が空でない作業の単位に絞って、自分の番を探せる |

#### ラベル名と固定の見出し

| キー | 中身 | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- | --- |
| `labels.agentReady` | 人が決めるときに付けるラベル | 文字列（空でない） | `agent-ready`（[SETUP-8][setup]） | push 以上の人が付け、最新のマスターより後に付けたときだけ有効（[FLOW-36][flow]、[FLOW-38][flow]）。`setup` が足し、名前は設定に持つ（[SETUP-8][setup]） |
| `labels.agentPaused` | 人が一時止めるときに付けるラベル | 文字列（空でない） | `agent-paused`（[SETUP-8][setup]） | push 以上の人が付けると、止め方が一時停止中になり、外すと続ける（[FLOW-38][flow] の S1・S2）。`setup` が足し、名前は設定に持つ（[SETUP-8][setup]） |
| `labels.agentManual` | 人が引き取るときに付けるラベル | 文字列（空でない） | `agent-manual`（[SETUP-8][setup]） | push 以上の人が付けると、止め方が引き取り中になり、外すと続ける（[FLOW-38][flow] の S3・S4）。`setup` が足し、名前は設定に持つ（[SETUP-8][setup]） |
| `labels.securitySuspected` | 攻撃の疑いがある間、進行 Bot が付ける表示のラベル | 文字列（空でない） | `security-suspected`（[SETUP-8][setup]） | 安全の軸に値がある間だけ付け、値が外れたら外す。表示だけで、判定には読まない（[SAFE-19][safety]）。`setup` が足し、名前は設定に持つ（[SETUP-8][setup]） |
| `labels.fromReview` | 🟢 の指摘から起票した Issue に付けるラベル | 文字列（空でない） | `from-review`（[SETUP-8][setup]） | 進行 Bot が 🟢 の指摘から起票した Issue に付け、人がマイルストーンを付けるまで、入口の検疫と下書きの対象から外す（[IMPL-26][impl]、[TEXT-12][text]）。`setup` が足す（[SETUP-8][setup]） |
| `labels.branchTypes` | Issue のラベルから作業ブランチの型を決める対応表（一覧で書き、前の要素ほど優先する） | 一覧。要素は `{ "label": <ラベルの名前>, "type": <型> }`。型は Conventional Commits の型（`feat`・`fix` など） | `bug` → `fix`（暫定） | 作業ブランチは `<型>/issue-<番号>` で、型は設定のラベル対応表から進行 Bot が決める。初期値は要件の例（[IMPL-2][impl]）。型は最初の実装のときだけラベルから決め、作業ブランチか PR があれば、ブランチ名から読む（着手の後にラベルが変わっても、push の先と PR の結びつきをずらさないため）。当たるラベルが 2 つ以上なら、前の要素の型にする。型は、規約から外れたときの PR の題名にも使うので（「[状態遷移と判定][design-flow]」の「push の手順」）、Conventional Commits の型に限る。型を足す PR をマージしたら、人が `setup` を実行し直して（GitHub の準備）、deploy key の push 先を限る ruleset の除外に足す（「[導入と点検][design-setup]」の「ruleset の形と確かめ方」）。足すまでは、新しい型の作業ブランチへの push が ruleset で断られる。型を外すか `labels.defaultBranchType` を替えて、前の型が設定のどこにも無くなるときは、前の型の作業ブランチが origin から無くなってから、人が `setup` を実行し直して除外を外す。前の型の作業ブランチが残るうちに除外を外すと、前の型で作った作業ブランチへの push が断られるため。`setup` は、設定に無い型の除外を、外す型の作業ブランチが origin に無いときだけ、承認を受けて外す。残っていれば外さずに、残っている作業ブランチを示す。外すまでの間も、前の型の作業ブランチの Issue は、型をブランチ名から読むので push を続けられる。外すまでの間、点検は、設定に無い型の除外のうち `refs/heads/<名前>/issue-*` の形で名前に `*`・`?`・`[` を含まないものを warn にする（作業ブランチの形なので、deploy key が作業ブランチの形のほかへ push できる道にはならない。名前に `*`・`?`・`[` を含む除外は、作業ブランチの形より広く当たりうるので fail のまま。「[導入と点検][design-setup]」の「ruleset の形と確かめ方」） |
| `labels.defaultBranchType` | 対応表に無いラベルの Issue の、作業ブランチの型 | 文字列（Conventional Commits の型） | `feat`（暫定） | 要件の例の「それ以外 → `feat`」（[IMPL-2][impl]） |
| `headings.workInstruction` | push 以上の人が、マスターを写して直した全文を書くコメントの見出し | 文字列（`##` と空白で始まる 1 行） | `## 作業指示`（[印と固定の見出しの一覧][markers]） | 直し済みで検疫してから、全文をマスターに写す。決めた後は読まない（[TEXT-1][text]、[FLOW-36][flow]） |
| `headings.sendBack` | push 以上の人が、マスターを直してほしい点を書くコメントの見出し | 文字列（`##` と空白で始まる 1 行） | `## 差し戻し`（[印と固定の見出しの一覧][markers]） | 受付 AI が、頼みとマスターと Issue の文から下書きを書き直す（[FLOW-36][flow]） |
| `headings.resume` | push 以上の人が、止まった作業の単位を戻すコメントの見出しの前半。見出しの行は、前半の後に空白と戻す先（`続きから`・`下書きから`・`計画から`）を続ける | 文字列（`##` と空白で始まる 1 行。戻す先の語を含めない） | `## 再開`（[印と固定の見出しの一覧][markers]） | 停止中を終え、戻す先を選ぶ。一時停止中と引き取り中は、ラベルを外す前に書けば戻す先になる（[FLOW-40][flow]）。戻す先の語は、戻した後の読み方（「[状態遷移と判定][design-flow]」の「戻した後の読み方」）と結びつくので、設定に持たず、進行 Bot のプログラムに固定する |
| `headings.securityConfirmed` | maintain 以上の人が、攻撃の疑いを確かめたことを書くコメントの見出し | 文字列（`##` と空白で始まる 1 行） | `## 安全の確認`（[印と固定の見出しの一覧][markers]） | 安全の軸の値を外す（Q3・Q4）。全体の停止の Issue に書いて閉じると、全体の停止が解ける（[FLOW-38][flow]、[SAFE-19][safety]） |
| `headings.changeApproval` | maintain 以上の人が、変えてはいけないパスのうち変えてよいパスを書くコメントの見出し | 文字列（`##` と空白で始まる 1 行） | `## 仕組みの変更の承認`（[印と固定の見出しの一覧][markers]） | 認めたパスは、マージまで効かない組なら AI が変えて push してよく、push で効く組なら人が push する（[IMPL-15][impl]、下の `protectedPaths.onPush` と `protectedPaths.onMerge`） |
| `headings.rejection` | 人がレビューの指摘を却下するコメントの見出し（`<n>` はラウンド） | 文字列（`##` と空白で始まる 1 行。`<n>` を 1 つ含む） | `## 却下 R<n>`（[印と固定の見出しの一覧][markers]） | 進行 Bot は、push 以上の権限を持つ人が `## 却下 R<n>` の見出しで書いたコメントだけを、却下した指摘として数える（誤検知率の記録。[FLOW-14][flow]） |
| `headings.differentJudgment` | 人が、`result` が `ready` の `merge` の印の PR をマージしない理由を書くコメントの見出し | 文字列（`##` と空白で始まる 1 行） | `## 判定と違う判断`（[印と固定の見出しの一覧][markers]） | 人が `## 判定と違う判断` の見出しで書いたら、進行 Bot の判定と人の判断の不一致に数える（判定の信頼度の記録。[MERGE-3][merge]） |

#### 進行 Bot の実行

| キー | 中身 | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- | --- |
| `run.intervalHours` | 定期の実行の間隔。24 の約数（1・2・3・4・6・8・12・24）に限る（[導入と点検][design-setup]の「定期の実行の登録」） | 数（24 の約数） | 24 時間（1 日 1 回。[FLOW-12][flow]） | 間隔と件数の上限は、リポの設定 1 つで変える（[FLOW-11][flow]）。間隔は OS ごとの定期の実行の登録にも書くので、`check` がずれを確かめる（[STRUCT-2][struct]）。間隔を変える PR がマージされたら、動かすマシンで `setup` を実行し直して登録し直す。登録し直すまで、点検がずれを fail にして作業に入らない |
| `run.maxMinutes` | 1 回の実行の所要時間の上限。超えたら残りの作業の単位は次の実行に回す | 数（1 以上の整数。分） | 180 分（暫定） | 1 回の実行の所要時間の上限を設定に持ち、超えたら残りの作業の単位は次の実行に回す。lock を長く持たないため（[FLOW-4][flow]）。上限は作業の単位の合間に見るので、動いている工程は止めない（計画の工程は、計画し直しを含めて最大 225 分かかりうる。`limits.planRevisions`）。180 分は、1 セッションで終わる重い工程（最大 45 分）なら、ほかの作業の単位の工程も進める余地を持たせる値で、[ADR-20261015-005][adr-settings] の補足情報が候補に挙げた値 |
| `run.stopSignalPollSeconds` | AI のセッションが動いている間に、全体の止める合図を読む間隔 | 数（1 以上の整数。秒） | 60 秒（暫定） | AI のセッションが動いている間も、全体を止める合図を一定の間隔で読む。間隔は基本設計で決める（[運用の規則][flow]）。60 秒にすると、止めると決めてから AI のセッションが止まるまでが 1 分ほどになる。1 セッションの間に読む回数は数十回で、GitHub の API の上限（1 時間に 5,000 回）に比べて少ない |
| `run.commandMinutes` | 進行 Bot が起こす子プロセス（`gh`・`git`・`lsof` など。AI のセッションと、リポの設定を読む前の子プロセスを除く）1 つの時間の上限 | 数（1 以上の整数。分） | 10 分（暫定） | ネットワークで止まった子プロセスを待ち続けると、lock を持ったまま実行が終わらず、後の定期の実行がすべて見送りになる（[STRUCT-2][struct]、[SAFE-4][safety]）。超えたら、AI のセッションと同じ手順で止める（`SIGTERM` から `SIGKILL` までの猶予は `limits.stopGraceSeconds`）。止めた後の扱いは、子プロセスの種類ごとに下の「[子プロセスが時間の上限を過ぎたとき][command-timeout]」で決める。10 分は、当てる先のリポの clone（business-os で 1 分ほど）に余裕を見た値 |
| `run.checkWaitMinutes` | 人が呼ぶ `check` が、点検の結果のファイルを待つ時間の上限 | 数（1 以上の整数。分） | 15 分（暫定） | 点検だけの実行は、試しのセッションを含めても 10 分ほどで終わる（試しのセッションの時間の上限が 10 分）。超えたら待つのをやめ、実行中か、ジョブが起きなかったかを示して終わる（「[導入と点検][design-setup]」の「点検だけで動かす仕組み」）。進行 Bot の実行の最中なら、その実行の終わりに点検の依頼を見る。人の `check` は、待つ時間を点検の依頼のファイルに書き、進行 Bot は、置いた時刻から書いた待つ時間を過ぎた点検の依頼で、点検の結果を書いた後に作業を続ける（「[導入と点検][design-setup]」の「見送った実行と、起動しなかった定期の実行」。[ADR-20261022-004][adr-intake-req] の決定 5） |
| `run.intakeWaitMinutes` | 人が `/verify-issue` などから呼ぶ intake の Skill が、受け入れの依頼の結果のファイルを待つ時間の上限 | 数（1 以上の整数。分） | 60 分（暫定） | 受け入れの依頼の実行は、実行の最初の点検（点検だけの実行は、試しのセッションを含めて 10 分ほど。上の `run.checkWaitMinutes`）の後に、受付 AI のセッション（時間の上限は `limits.session.minutes` の 45 分）を 1 つ起こすので、2 つを足した長さより長くする（[ADR-20261022-004][adr-intake-req] の決定 1）。`limits.session.minutes` を上げる PR では、この値も見直す。超えたら待つのをやめ、実行中か、ジョブが起きなかったかを示して終わる。intake の Skill は、待つ時間を受け入れの依頼のファイルに書き、進行 Bot は、置いた時刻から書いた待つ時間を過ぎた依頼を、受付 AI を起こさずに消す（「[AI のセッション][design-ai]」の「人の Skill から受け入れを呼ぶ形」、[ADR-20261022-004][adr-intake-req] の決定 5） |

##### 子プロセスが時間の上限を過ぎたとき

`run.commandMinutes` を過ぎて止めた子プロセスの扱い（[SAFE-4][safety]、[IMPL-14][impl]）。「[状態遷移と判定][design-flow]」の「1 回の実行の流れ」も、下の表に従う。

リポの設定を読む前の子プロセス（「1 回の実行の流れ」の順 2〜4 の、前の実行が残したセッションを探す `lsof`、全体の止める合図を読む `gh api`、進行 Bot の clone の `git fetch`）は、`run.commandMinutes` をまだ読んでいないので、進行 Bot のプログラムに固定した 10 分（`run.commandMinutes` の初期値と同じ）を時間の上限にする。止めるときの `SIGTERM` から `SIGKILL` までの猶予も、`limits.stopGraceSeconds` をまだ読んでいないので、進行 Bot のプログラムに固定した 10 秒（`limits.stopGraceSeconds` の初期値と同じ）にする。過ぎたときの扱いは、`run.commandMinutes` を過ぎたときと同じく下の表に従う。

| 子プロセス | 過ぎたとき | 理由 |
| --- | --- | --- |
| GitHub を読む処理（`gh api` の読み取り、進行 Bot の clone の `git fetch`） | 作業を増やさずに記録して、実行を終える | 読めない状態で判断して書き込まないため |
| GitHub への書き込み（`gh api` のコメントと印、ラベル、Project の欄、PR の作成） | 書き込みが止まった作業の単位を今の実行で進めず、次の実行で GitHub から読み直して続ける。実行の失敗に数える | 止めた時点で GitHub に届いたかが分からないので、読み戻して確かめる前に次の書き込みをしない |
| push と取り込み（`git push`、push の手順の `git fetch` と `git merge`） | 実行の失敗として、次の実行に持ち越す | lock を持ったまま待ち続けず、push の手順を次の実行でやり直すため |
| 止める手順と、残りのプロセスの探し方（前の実行が残したセッションと、作業場所を消す前。`lsof`、`lsof +D`） | 残りを確かめられなかったとして、作業場所を消さずに停止中（`session-not-stopped`。[ADR-20261027-001][adr-leftover]） | 残りが無いと読んで作業場所を消すと、残ったコマンドがあるまま消すことになる（[AI のセッション][design-ai]の「止める手順」） |

#### 上限

| キー | 中身 | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- | --- |
| `limits.startsPerRun` | 1 回の実行で新しく着手する作業の単位の数の上限 | 数（1 以上の整数。件） | 1 件（[FLOW-3][flow]） | 進行 Bot は、1 回の実行で着手する件数に上限を持つ（[FLOW-3][flow]） |
| `limits.wip` | 進行中の作業の単位の数の上限（数え方は「[状態遷移と判定][design-flow]」の「進行中の作業の単位の数え方」） | 数（1 以上の整数。件） | 1 件（並列化（段階 9）に入るまで。[FLOW-3][flow]） | 進行中の作業の単位の数にも上限を持ち、上限に達していれば新しく着手しない（[FLOW-3][flow]） |
| `limits.intakePerRun` | 1 回の実行で、受付 AI を起こす下書きと書き直しの件数の上限 | 数（1 以上の整数。件） | 5 件（暫定） | 1 回の実行で受付 AI を起こす件数に上限を持つ。初回の実行で未処理の Issue すべてに走らないためと、外の人が Issue を大量に立てたときの利用枠の歯止め（[TEXT-12][text]）。5 件は [ADR-20261015-005][adr-settings] の補足情報が候補に挙げた値 |
| `limits.roundTrips` | 直しの往復の回数の上限。検査不合格・レビュー不合格・衝突ありで、往復の回数がすでに上限に達していれば、作業 AI を起こさずに停止中（`round-limit`）にする（数え方は「[状態遷移と判定][design-flow]」の「ラウンドと往復の数え方」。[ADR-20261022-003][adr-rounds]） | 数（1 以上の整数。回） | 3 回（[IMPL-24][impl]） | 往復の回数に上限を持ち、超えたら停止中（`round-limit`）にする。同じ指摘が再び出たとき（同じ指摘の再発）は、停止中（`repeated-finding`）にする（[IMPL-24][impl]） |
| `limits.planRevisions` | 計画の確かめの指摘で計画し直す回数の、計画の工程ごとの上限。上限まで計画し直しても直す点が残れば、計画を投稿せずに停止中（`plan-limit`）にする | 数（1 以上の整数。回） | 1 回（[IMPL-10][impl]） | 計画は実行して確かめられないので、確かめるたびに直すところが出て、ループが終わらないおそれがある。1 回なら、計画の工程は最大 5 つのセッション（1 セッションの上限 45 分なら 225 分）で終わる（[IMPL-10][impl]、[ADR-20261019-003][adr-replan]、[ADR-20261024-002][adr-no-stages] の決定 3） |
| `limits.session.minutes` | AI のセッション 1 回の時間の上限。超えたら進行 Bot が止める | 数（1 以上の整数。分） | 45 分（[IMPL-13][impl]） | `claude` には時間の上限が無いので、進行 Bot が時間で止める（[IMPL-13][impl]）。止めた回は実行の失敗として次回に持ち越す（[IMPL-14][impl]） |
| `limits.session.turns` | AI のセッション 1 回のターンの上限（`--max-turns`） | 数（1 以上の整数。ターン） | 200 ターン（[IMPL-13][impl]） | ターンの上限は `--max-turns` で掛ける（[IMPL-13][impl]。GitHub Agentic Workflows の既定は 500 ターン） |
| `limits.session.costUsd` | AI のセッション 1 回の費用の上限（`--max-budget-usd`）。途中で止めた回は、起動したときに `--max-budget-usd` に渡した値を、止めた回の費用とみなす（前の実行が残したセッションも、実行の記録の始まりの行に書いた値で数え、設定を変えた後の値で数えない） | 数（0 より大きい。USD） | 10 USD（暫定） | 費用の上限を `--max-budget-usd` でも掛ける（[IMPL-14][impl]）。途中で止めた回は費用が分からないので、この値をその回の費用とみなす（[ADR-20261018-001][adr-usage]）。値の決め方は下の「[費用の上限の値][cost-values]」 |
| `limits.issueCostUsd` | 1 作業の単位の累計の費用の上限（往復を含む）。起動の前に、累計に、動いているセッションと起動するセッションの費用の上限を足して超えるなら、起動せずに停止中（`cost-limit`）にする（比べ方は「[AI のセッション][design-ai]」の「起動の前の確かめ」） | 数（0 より大きい。USD） | 100 USD（暫定） | 1 Issue の累計の利用枠（往復を含む）に上限を持ち、費用で数える。当たったら停止中（`cost-limit`）にする（[IMPL-14][impl]、[ADR-20261018-001][adr-usage]）。値の決め方は下の「[費用の上限の値][cost-values]」 |
| `limits.runCostUsd` | 1 回の実行の費用の上限。起動の前に、その実行の累計に、動いているセッションと起動するセッションの費用の上限を足して超えるなら、起動せずに次の実行で続ける | 数（0 より大きい。USD） | 60 USD（暫定） | `claude -p` の実行は利用枠を消費するので、1 回の実行の上限を持つ（[SETUP-12][setup]、[IMPL-14][impl]）。値の決め方は下の「[費用の上限の値][cost-values]」 |
| `limits.dailyCostUsd` | 1 日（マシンの時刻の日付）の費用の上限。起動の前に、その日の累計に、動いているセッションと起動するセッションの費用の上限を足して超えるなら、起動せずに次の日の実行で続ける | 数（0 より大きい。USD） | 120 USD（暫定） | 1 日の利用枠にも上限を持ち、費用で数える（[SETUP-12][setup]、[IMPL-14][impl]）。値の決め方は下の「[費用の上限の値][cost-values]」 |
| `limits.stopGraceSeconds` | AI のセッションを途中で止めるとき、`SIGTERM` を送ってから `SIGKILL` を送るまでの猶予。前の実行が残したセッションは、起動したときの値（実行の記録の始まりの行に書く）で止める | 数（1 以上の整数。秒） | 10 秒（暫定） | 途中で止めるときは `SIGTERM` を送り、猶予の間に終わらなければ `SIGKILL` を送る。猶予は設定に持つ（[IMPL-14][impl]）。試験では `SIGTERM` の後 1 秒ほどで終わった（[ADR-20261018-001][adr-usage]）ので、10 倍の余裕を見る |
| `limits.retry.count` | 同じ作業の単位で実行の失敗が続いたとき、停止中（`retry-limit`）にするまでの回数。片付けの失敗が続いたときに、停止中（`cleanup-failed`）にするまでの回数にも使う | 数（1 以上の整数。回） | 3 回（[IMPL-14][impl]） | 実行の失敗（利用枠切れ・時間の上限など、作業の中身と無関係の失敗）は次回に持ち越し、同じ作業の単位で続いたら停止中にする（[IMPL-14][impl]）。片付けの失敗も、実行の失敗の上限と同じ考え方で数える（[ADR-20261024-006][adr-cleanup] の決定 1） |
| `limits.retry.intervalRuns` | 実行の失敗の後、何回目の定期の実行で再試行するか | 数（1 以上の整数。回） | 1（次の定期の実行。暫定） | 実行の失敗の再試行の間隔を設定に持つ（[IMPL-14][impl]）。実行の中で待つと lock を長く持つので、次の定期の実行で再試行する（[FLOW-4][flow]）。[ADR-20261015-005][adr-settings] の補足情報が挙げた候補（再試行は次の実行） |
| `limits.remindDays` | `result` が `ready` の `merge` の印の PR が、マージも「判定と違う判断」のコメントも無いまま経ったとき、進行 Bot が 1 回だけ催促するまでの日数 | 数（1 以上の整数。日） | 7 日（[MERGE-3][merge]） | 人が動かない PR は、マージの判定と人の判断の一致の記録に数えられないので、1 回だけ催促する（[MERGE-3][merge]） |
| `limits.ciStaleHours` | CI の結果が `cancelled` `skipped` `neutral` `stale` のまま変わらないとき、停止中（`ci-stale`）にするまで待つ時間 | 数（1 以上の整数。時間） | 48 時間（暫定） | CI の結果が `cancelled` などのまま、完了から設定の時間が経っても変わらなければ停止中（`ci-stale`）にする（[IMPL-27][impl]）。48 時間は、1 日 1 回の間隔（[FLOW-12][flow]）で、人が再実行する機会を間隔 1 回分より長く取る値 |

##### 費用の上限の値

費用の上限 4 つ（`limits.session.costUsd`・`limits.issueCostUsd`・`limits.runCostUsd`・`limits.dailyCostUsd`）の値と決め方。値は USD で、Claude Code が API の定価で換算した費用（`total_cost_usd`）と比べる（[AI のセッション][design-ai]の「費用の数え方」）。

| 上限 | 初期値 | 決め方 |
| --- | --- | --- |
| `limits.session.costUsd` | 10 USD | `opus` のセッションを止めない値にする。Opus 5.5 の短い会話（やりとり 33〜71 回）は、API の定価で 1.7〜3.3 USD だった（2026-10-20 に、メンテナの手元の会話の記録から数えた）。ターンの上限（200 ターン）まで動いたセッションを止めない値にする。受付 AI（`claude-haiku-5-5`）のセッションも同じ上限で起こす。Claude Code 2.1.285 は `claude-haiku-5-5` の単価を知らず、Opus 5.5 の単価で数える（FACT-48）が、上限に早く当たる向き（安全側）なので、値は変えない |
| `limits.issueCostUsd` | 100 USD | 1 セッションの上限の 10 回分。人が `/verify-issue` などから intake の Skill で頼んだ受付 AI のセッションも、作業の単位の累計に入る（[ADR-20261022-004][adr-intake-req] の決定 4）。超える作業の単位は停止中（`cost-limit`）にして人が見る。往復を 3 回使う作業の単位は 30 セッションほど起こし（要件の「利用枠の見方」）、実費しだいで往復の上限より先に当たるので、運用の記録で決め直す |
| `limits.runCostUsd` | 60 USD | 1 つの重い工程が同じ実行で起こしうるセッションの数（計画の工程は最大 5、レビューの 1 ラウンドは 3 つの軸・見落としの軸・批評 AI の 5）の分に、1 セッションの上限 1 つ分を足した値（1 セッションの上限の 6 倍）。重い工程を始める前にその分を比べ、工程の途中で止まらないようにする。同じ実行で先に試しのセッションか下書きが使っても、重い工程を始められるように、1 セッションの上限 1 つ分を足す（下の関係） |
| `limits.dailyCostUsd` | 120 USD | 定期の実行 1 回（[FLOW-12][flow]）と、人が手で起こす実行 1 回の分（1 回の実行の上限の 2 倍） |

- 値は、プランの利用枠の何割かを表すものではない。Pro と Max の利用枠は、5 時間ごとの上限と 1 週間の上限で、チャットと Claude Code で共有し、量は USD で公開されていない（[Claude Code の費用の文書][doc-costs]、[Claude の料金とプラン][doc-plans]）
- 並列化（段階 9）に入るまでは、定期の実行が 1 日 1 回で（[FLOW-12][flow]）、進行中の作業の単位は 1 件なので（[FLOW-3][flow]）、4 つの値はプランに依らない。プランで変わるのは、動かしてよいかだけ

  | プラン | 目安 |
  | --- | --- |
  | Pro | 勧めない。`opus` のセッションは 1 つで数 USD かかり、Pro の利用枠（Max 5x の 5 分の 1。[Claude の料金とプラン][doc-plans]）では、人の作業と利用枠を取り合う（推定） |
  | Max 5x、Max 20x | 上の表の値で動かす。Max 5x で、人の作業が週の上限に近いときは、`limits.dailyCostUsd` を 60 USD（定期の実行 1 回分）にする |

- 動かし始めて最初の 5 件の作業の単位の記録（セッションごとの費用。[FLOW-5][flow]）を見て、4 つの値を決め直す
- 並列化（段階 9。実行の間隔を狭める段階）では、`limits.dailyCostUsd` を「1 回の実行の上限 × 1 日の実行の回数」で決め直す。Max 20x と Max 5x の違いは、ここで値に出る
- 値を決め直すときも、次の関係を保つ。重い工程を始める前に「累計 ＋ その工程が起こしうるセッションの数 × 1 セッションの上限」で比べるので（[AI のセッション][design-ai]の「起動の前の確かめ」）、外れるとその工程を 1 度も始められない。起動しなかった回は費用を使わず、作業の単位の累計の上限にも当たらないので、黙って止まり続ける。`check` は、外れていれば fail にする
  - `limits.issueCostUsd` は、`limits.session.costUsd` × 1 つの重い工程が起こしうるセッションの数の最大（計画の工程の `3 + 2 × limits.planRevisions` と、レビューの 1 ラウンドの 5 の、大きいほう）以上
  - `limits.runCostUsd` と `limits.dailyCostUsd` は、`limits.session.costUsd` ×（1 つの重い工程が起こしうるセッションの数の最大 ＋ 1）以上。重い工程 1 つ分に、1 セッションの上限 1 つ分を足す。同じ実行と同じ日に、先に試しのセッションか下書きが使っても、重い工程を始められるようにするため（試しのセッションの費用も、1 回の実行と 1 日の累計に数える。「[導入と点検][design-setup]」の「試しのセッション」）
  - `limits.dailyCostUsd` は、`limits.runCostUsd` 以上

#### モデル

役ごとのモデルと effort は、`model` と `effort` の 2 つの欄を持つ（例：`models.plan.model`、`models.plan.effort`）。モデルは別名で書く（`haiku` だけは別名が旧版に解けるので完全な ID。[モデルの流れ][impl]）。

| キー | 中身 | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- | --- |
| `models.intake` | 受付 AI（マスターの下書きと書き直し） | 対応表（`model` と `effort`。どちらも文字列） | `claude-haiku-5-5` / medium（[IMPL-29][impl]） | 役ごとのモデルの正本は要件の「モデルの流れ」の表（[IMPL-29][impl]）。`haiku` は別名が Haiku 4.5 に解けるので、完全な ID で書く |
| `models.intakeAfterRefusal` | 受付 AI が拒否（`stop_reason: refusal`）で止まった Issue の、次の回の受付 AI | 対応表（`model` と `effort`。どちらも文字列） | `sonnet`（[IMPL-29][impl]） / medium（effort は暫定。`models.intake` と同じ） | 受付 AI が拒否で止まったら実行の失敗として扱い、次の回は `sonnet` で試す（[IMPL-14、IMPL-29][impl]）。拒否の記録が無いときは `models.intake` で起動する（[運用の規則][flow]） |
| `models.plan` | 計画 AI（既定。計画の確かめの指摘で計画し直すときは、直す前の計画と同じ値） | 対応表（`model` と `effort`。どちらも文字列） | `opus` / xhigh（[IMPL-31][impl]） | 計画 AI は、いつも `models.plan` で始める。受付 AI の計画の難しさの判定と、人のラベル `plan:*` は、作業ごとに使うモデルの決め方を決めるまで使わない（[IMPL-30、IMPL-31][impl]、[ADR-20261024-004][adr-master] の決定 8） |
| `models.planHard` | 計画 AI（計画 AI が自己申告して計画し直すとき） | 対応表（`model` と `effort`。どちらも文字列） | `fable` / high（[IMPL-31][impl]） | `opus` が設計判断などを含むと自己申告したら、同じ実行で 1 回だけ `fable` で計画し直す（[IMPL-31][impl]、[ADR-20261018-002][adr-redo]） |
| `models.work` | 作業 AI の最初の実装 | 対応表（`model` と `effort`。どちらも文字列） | `opus` / high（[IMPL-32][impl]） | 作業 AI は `models.work` で始める。種類ごとに変える決め方は別に決め、決めるまではこの値を使い、変えるときはリポの設定の PR で変える（[IMPL-25、IMPL-32][impl]、[ADR-20261024-002][adr-no-stages] の決定 7）。作業 AI には `fable` を使わない（[IMPL-32][impl]。`check` が、`models.planHard` のほかの `models.*` に `fable` があれば fail にする） |
| `models.workRaised` | 最初の実装が `sonnet` だった作業の単位で、修正のときに上げる先 | 対応表（`model` と `effort`。どちらも文字列） | `opus` / high（[IMPL-35][impl]） | 最初の実装が `sonnet` なら、修正のときに `opus` に 1 段上げる。`fable` には上げない（[IMPL-32、IMPL-35][impl]）。最初の実装のモデルは、修正のときに、今の設定の `models.work` から決める。手元の記録は判定に使わないため（[ADR-20261015-006][adr-flow] の決定 4）。PR を作った後に設定を変えると、変えた後の値で決まる。衝突の直しのセッション（[MERGE-6][merge]。修正と同じ形）も、修正と同じ決め方で上げる |
| `models.review` | レビュー AI（3 つの軸と見落としの軸。4 つの軸とも同じ値） | 対応表（`model` と `effort`。どちらも文字列） | `opus` / xhigh（[IMPL-33][impl]） | 4 つの軸とも同じ値で始め、軸ごとに分けるかは基本設計の設定の値で決める（[IMPL-33][impl]、[ADR-20261019-002][adr-gaps]）。今は分けずに 1 つの値で持つ。分けるときは、運用の記録を見て、軸ごとのキーを足す（[ADR-20261024-002][adr-no-stages] の決定 8）。モデルは設定で変えられるようにする（[IMPL-19][impl]） |
| `models.critic` | 批評 AI | 対応表（`model` と `effort`。どちらも文字列） | `opus` / high（[IMPL-34][impl]） | 批評 AI は、🔴 と 🟡 を別のセッションで再検証する。値は変えない（[IMPL-22、IMPL-34][impl]） |
| `models.planReview` | 計画を確かめるレビュー AI | 対応表（`model` と `effort`。どちらも文字列） | `opus` / xhigh（`models.review` と同じ。暫定） | 計画の確かめのモデルと effort は、基本設計で決める（[IMPL-37][impl]）。同じレビュー AI の役で、計画の誤りを見つけるには PR のレビューと同じ深さが要るので、`models.review` と同じ値で始める。運用の記録で見直す（[IMPL-10][impl]、[ADR-20261018-004][adr-planrev]） |
| `review.criticConfidenceThreshold` | 批評 AI が付けた確信度が、しきい値より低い 🟡 を 🟢 に下げる | 数（0〜100） | 80（0〜100。暫定） | 批評 AI が付けた確信度がしきい値（設定）未満の 🟡 は、🟢 に下げる（[IMPL-22][impl]）。80 は、Anthropic 公式 Plugin が確信度 80 未満の指摘を捨てる線（[レビューの Skill][review-skill]）で、[ADR-20261015-005][adr-settings] の補足情報が候補に挙げた値 |

#### 検査と判定

| キー | 中身 | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- | --- |
| `checks.qualityGate` | AI が作業場所で実行する品質ゲートのコマンド | 文字列（コマンド） | `pnpm check`（[STRUCT-2][struct]） | 作業 AI は、CI のログを受け取らず、自分の作業場所で品質ゲートを実行して確かめる（[IMPL-27][impl]）。守る規約に「品質ゲートが緑」を含む（[IMPL-4][impl]） |
| `checks.required` | CI が緑かを判定する必須チェックの一覧（チェックの名前、チェックを出した app、base 側のワークフローのパス） | 一覧。要素は `{ "name": <チェックの名前>, "app": <チェックを出した app の slug>, "workflow": <base 側のワークフローのパス> }`（例：`{ "name": "check", "app": "github-actions", "workflow": ".github/workflows/ci.yml" }`）。GitHub Actions のほかの app のチェックは、`workflow` を `null` にする | 空。`setup` が、base ブランチの先端のコミットで動いたチェックから候補を出し、人が承認したものをリポの設定の PR に書く（「[導入と点検][design-setup]」の「GitHub の準備」）。候補が無ければ空のままで、`check` が fail にするので、人が当てる先のリポの必須チェックを書く（business-os の例：`check (ubuntu-latest)` `check (windows-latest)` `commitlint`。[IMPL-27][impl]） | CI が緑かは、設定に列挙した必須チェックがすべて `success` かで判定する。一覧が空なら点検を fail にし、空を「すべて success」と読まない（[IMPL-27][impl]） |
| `merge.allowPaths` | 進行 Bot が「マージしてよい」（`ready`）と判定してよいパスの許可一覧。一覧に無いパスを 1 つでも触れば `human` | 一覧（パスの文字列。git の pathspec の `:(glob)` の決まりで書く。「[検査と判定][path-glob]」の表の下） | 空（`setup` が書く値。すべて `human`）。business-os では、人が `setup` の作るリポの設定の PR に `docs/usage/**` `.changeset/*.md` を書き足す（[MERGE-4][merge]） | 進行 Bot が「マージしてよい」と判定してよいパスは、拒否一覧ではなく許可一覧で設定に持つ（[MERGE-4][merge]）。パスを足すのは、安全の境界を緩める変更なので ADR で決める（[ADR-20261015-005][adr-settings]） |
| `merge.policyPaths` | 方針に係るパスの一覧。1 つでも触れば判定の対象外（`human`） | 一覧（パスの文字列。git の pathspec の `:(glob)` の決まりで書く。「[検査と判定][path-glob]」の表の下） | 空（`setup` が書く値。ほかのリポは、許可一覧にパスを足すときに一緒に決める）。business-os では、人が `setup` の作るリポの設定の PR に、要件が「許可一覧に含めない」の例として挙げたパスを書き足す（[MERGE-4][merge]）：`docs/adr/**` `docs/design/**` `ROADMAP.md` `CLAUDE.md` `.claude/**` `dev-autopilot/**` `.claude-plugin/**` `.github/**` `plugin/hooks/**` `plugin/agents/**` `plugin/templates/**`、`check:*` の実装のファイル（business-os の `package.json` の `check:*` が呼ぶファイルを 1 つずつ並べる。要件の例は `plugin/scripts/check-repo.ts`）、`package.json` `pnpm-lock.yaml` `pnpm-workspace.yaml` `.node-version` `tsconfig.json` `vitest.config.ts` `eslint.config.*` `commitlint.config.*` `.gitleaks.toml` `.gitignore` `.markdownlint*`。向き先（`release/*` ブランチと `main` 向けの PR）は一覧に書かず、マージの判定の `base`（「[状態遷移と判定][design-flow]」の「マージの判定」）で止める | 方針に係るパスに触れる PR は、進行 Bot の判定の対象外（[MERGE-2][merge]、[SAFE-9][safety]）。一覧は設定に持つ（[STRUCT-2][struct]） |
| `protectedPaths.onPush` | 変えてはいけないパスのうち、push で効くものの足す分（CI が信じて動かすスクリプトなど、当てる先のリポごとに違うパス） | 一覧（パスの文字列。git の pathspec の `:(glob)` の決まりで書く。「[検査と判定][path-glob]」の表の下） | 空（`setup` が人に聞いて書く） | push の前に、diff が push で効くパスに触れていれば、`## 仕組みの変更の承認` で認めていても AI の変更を push せず、差分を Issue に載せて停止中（`manual-push`）にし、人が中身を確かめて自分で push する。認めていなければ停止中（`protected-path`）にする。計画の後に、計画が変える見込みのパスとも照らす（[IMPL-10][impl]、[IMPL-15][impl]、[ADR-20261024-004][adr-master] の決定 9）。名指しのパスのうち push で効くもの（`.github/**`、`**/CLAUDE.local.md`、`.claude/settings.local.json`）は、進行 Bot のプログラムに固定し、設定には足す分だけを持つ（[ADR-20261021-005][adr-blocked]） |
| `protectedPaths.onMerge` | 変えてはいけないパスのうち、マージまで効かないものの足す分（規約の正典のパスなど、当てる先のリポごとに違うパス） | 一覧（パスの文字列。git の pathspec の `:(glob)` の決まりで書く。「[検査と判定][path-glob]」の表の下） | `setup` が人に聞いた当てる先のリポの規約の正典のパス（business-os は `docs/adr/**` `docs/design/**`。[IMPL-15][impl]）。規約の正典のうち `CLAUDE.md` は、名指しのパスに入るので、書かなくても当たる | 計画の後と push の前に、計画が変える見込みのパスか diff が、maintain 以上の人が `## 仕組みの変更の承認` で認めていないパスに触れていれば、停止中（`protected-path`）にする。認めたパスは AI が変えて push してよく、マージの判定はいつも `human` にする（[IMPL-10][impl]、[IMPL-15][impl]、[ADR-20261024-004][adr-master] の決定 9）。名指しのパスのうちマージまで効かないもの（ほかの `.claude/**`、`CLAUDE.md`、`.mcp.json`、`dev-autopilot/**`）は、進行 Bot のプログラムに固定し、設定には足す分だけを持つ（[ADR-20261021-005][adr-blocked]） |
| `push.rulesetBypassApps` | deploy key の push 先を限る 2 つの ruleset の bypass に入れる GitHub App の一覧（例：Dependabot）。進行 Bot のアプリは入れられない | 一覧（GitHub App の名前（slug）の文字列。例：`dependabot`） | 空（`setup` が人に聞いて書く） | deploy key の push 先を限る ruleset の bypass には、人がリポの設定の一覧で許したアプリも入れてよい（[IMPL-15][impl]）。ブランチの ruleset は bypass の外のアプリのブランチの作成も止めるので、Dependabot などを使うリポは一覧に足す（FACT-67）。一覧に足しても、deploy key の push 先の制限は変わらない（AI の変更が GitHub に届く道は deploy key だけ）。base ブランチの ruleset にはアプリを入れないので、base ブランチは PR 必須のまま。足したアプリは、dev-autopilot の ruleset を作る前にできていたこと（ブランチの作成など）を取り戻すだけで、安全の境界を緩める変更に当たらないので、ADR で決めずに、人がリポの設定の PR で足す（[IMPL-15][impl]。一覧は設定に持つ。[STRUCT-2][struct]）。リポの設定のファイルは変えてはいけないパス（`.claude/**`。マージまで効かない組）にあるので、AI は、人が認めなければ変えられず、変えても人がマージするまで効かない（進行 Bot は base ブランチの先端から読む）。足し方と点検は「[導入と点検][design-setup]」の「ruleset の形と確かめ方」 |

- 変えてはいけないパスの一覧は、名指しのパスと、`protectedPaths.onPush`・`protectedPaths.onMerge` の足す分を合わせたもの。2 つの組の両方に当たるパスは、push で効く組として扱う（止める側に倒す）。足す分からパスを外すのは、安全の境界を緩める変更なので ADR で決める（[ADR-20261015-005][adr-settings]）。名指しのパスを外すのは、IMPL-15 と進行 Bot のプログラムの変更として ADR で決める（[ADR-20261021-005][adr-blocked] の決定 3）
- パスの一覧（`merge.allowPaths`・`merge.policyPaths`・`protectedPaths.onPush`・`protectedPaths.onMerge`）は、git の pathspec の `:(glob)` の決まりで書く。進行 Bot は、IMPL-15 が名指しするパスも含めて、git の pathspec で照らす。照らし方を道具の既定に任せず、許可一覧が広く当たる形を防ぐため（[MERGE-4][merge]、[IMPL-15][impl]）
  - `*` は `/` を越えない（`.changeset/*.md` は `.changeset/a/b.md` に当たらない）。`**/` は 0 個以上のフォルダに当たり（`**/CLAUDE.local.md` は根の `CLAUDE.local.md` にも当たる）、`/**` はフォルダの中のすべてに当たる
  - `.` で始まる名前にも `*` が当たる
  - 大文字と小文字は、止める側の一覧と許す側の一覧で分けて扱う。止める側の一覧（名指しのパス、`protectedPaths.onPush`、`protectedPaths.onMerge`、`merge.policyPaths`。計画が変える見込みのパス（`plan.paths`）を変えてはいけないパスの一覧と照らすときも）は、`:(glob,icase)` で大文字と小文字を区別せずに照らす（`.Claude/settings.local.json` は `.claude/**` に、`sub/claude.local.md` は `**/CLAUDE.local.md` に当たる）。許す側の一覧（`merge.allowPaths` と、人が `## 仕組みの変更の承認` に書いたパス）は、`:(glob)` で大文字と小文字を区別して照らす（`Docs/usage/a.md` は `docs/usage/**` に当たらない）
  - 分けるのは、大文字と小文字を区別しないボリューム（手元の Mac の作業場所）では、綴りの大小を変えた名前も同じファイルとして読まれるため。作業 AI が `.Claude/settings.local.json` や `sub/claude.local.md` を commit すると、`:(glob)` の照らしには当たらず、次のセッションの worktree では `.Claude/settings.local.json` が `.claude/` の中に置かれ、`claude.local.md` も指示として読まれる（FACT-68）。sandbox と Write の道具は大文字の違う `.claude` のフォルダへの書き込みを止めるが、ファイルを書かずに commit は作れる（FACT-70）。止める側を広く、許す側を狭く当てれば、どちらの誤りも止まる側に倒れ、大文字と小文字を区別するファイルシステム（Linux）では余計に止めるだけで済む
  - 当たる例と当たらない例は、単体テストの前提データに置く。大文字と小文字を変えた例（止める側の一覧では当たり、許す側の一覧では当たらない）も入れる

#### 規約

| キー | 中身 | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- | --- |
| `conventions.language` | commit メッセージと PR の言語 | 文字列（言語のタグ。例：`ja`） | `setup` が人に聞いて書く（business-os：`ja`（日本語）。[IMPL-4][impl]） | 作業 AI に守らせる規約は、当てる先のリポの正典をそのまま渡す（[IMPL-4][impl]）。規約は設定に持つ（[STRUCT-2][struct]） |
| `conventions.conventionalCommits` | commit メッセージを Conventional Commits で書くか | 真偽 | `setup` が人に聞いて書く（business-os：`true`。[IMPL-4][impl]） | 作業 AI に守らせる規約（[IMPL-4][impl]） |
| `conventions.changeset` | 変更に changeset を付けるか | 真偽 | `setup` が人に聞いて書く（business-os：`true`。[IMPL-4][impl]） | 作業 AI に守らせる規約（[IMPL-4][impl]） |

#### 安全

| キー | 中身 | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- | --- |
| `safety.botAccounts` | 印を書いたと認める GitHub のアカウント名の一覧。投稿に使うアカウントは、GitHub App に切り替える前は PAT の持ち主、切り替えた後はアプリの投稿者の名前（`<アプリの名前>[bot]`）で決まり、一覧では決めない | 一覧（GitHub のアカウント名の文字列。1 つ以上） | `setup` が人の `gh` のログイン名を 1 つ書く（最初はメンテナのアカウント。[SETUP-13][setup]）。GitHub App に切り替えるときに、アプリの投稿者の名前に替える（[ADR-20261020-002][adr-app]） | 進行 Bot は、一覧のどれかが投稿し、未編集で、署名が通るコメントの印だけを読む（[IMPL-20][impl]）。当てる先のリポに初めて書く前に GitHub App に切り替え（[SETUP-13][setup]、[ADR-20261024-002][adr-no-stages] の決定 2）、切り替えのときにメンテナのアカウントを一覧から外す。PAT では印を書いていないので、読み続ける印は無い（[ADR-20261020-002][adr-app] の決定 4）。その後にアカウントを替えるときは、新しいアカウントを足し、前のアカウントの印が残る Issue と PR が閉じたら前のアカウントを外す。足すのは認める投稿者を増やす向きなので、替えると決めた ADR を添えて PR に残す（[ADR-20261020-001][adr-bots] の決定 4） |
| `safety.network.allowedDomains` | AI のセッションの sandbox から通信してよい先 | 一覧（ドメイン名の文字列） | `registry.npmjs.org`（暫定） | AI のセッションの外への通信は、npm のレジストリなどの許可した先だけにする（[SAFE-10][safety]）。先を足すのは、安全の境界を緩める変更なので ADR で決める（[ADR-20261015-005][adr-settings]） |
| `safety.env.names` | 進行 Bot が AI のセッションに渡す環境変数の名前のうち、足す名前（ほかの名前は渡さない） | 一覧（環境変数の名前の文字列） | 空（暫定。決めた名前は進行 Bot のプログラムに固定する。名前と値と理由は「[AI のセッション][design-ai]」の「環境変数」。決めた名前のうち、`TMPDIR`・`CLAUDE_CODE_TMPDIR`・`XDG_*` の 4 つ・`GIT_CONFIG_GLOBAL`・`NPM_CONFIG_USERCONFIG`・commit の作者・`ENABLE_CLAUDEAI_MCP_SERVERS` は [SAFE-10][safety] と [SAFE-1][safety] が決めた名前） | 子プロセスの環境変数は許可した名前だけで組み立て、`gh` の認証と `GH_TOKEN` を渡さない（[SAFE-1][safety]）。決めた名前は、リポの設定の PR で消せないようにプログラムに固定し、設定には足す名前だけを持つ（`safety.sandbox.denyRead` と同じ形）。足す名前の値は、マシンの設定の `safety.env.values` に書き、`setup` が定期の実行の定義の環境変数に写す。進行 Bot は、自分の環境変数の値をそのまま渡す。`safety.env.names` の名前が進行 Bot の環境変数に無ければ、AI のセッションを起動せず、`check` も fail にする（黙って渡さずに進まないため）。名前を足すのは ADR で決める（迷ったら ADR。[ADR-20261015-005][adr-settings] の決定 1） |
| `safety.sandbox.denyRead` | AI のセッションの sandbox で読ませないパスのうち、足す分。`${home}` か `${worktree}` で始めて書き、絶対パスは書かない | 一覧（`${home}` か `${worktree}` で始まるパスの文字列） | 空（暫定） | 読み取りは、ホーム・置き場・秘密の置き場・macOS の人の一時フォルダとキャッシュ（`/var/folders/<…>/T` と `…/C`）・`/Users/Shared`・`/Volumes` などを読ませない形にし（ホームの外の道具は読める）、`denyRead` に入れるパスは要件が決めている（[SAFE-10][safety]、FACT-63。進行 Bot が入れるパスの一覧は「[AI のセッション][design-ai]」の「セッションごとの安全設定」の「中身」）。進行 Bot は要件が決めたパスをマシンの中のフォルダ（[マシンの中のフォルダ][places]）から組み立てて必ず入れ、設定には足す分だけを持つ（設定で消せないようにするため） |
| `safety.sandbox.denyWrite` | AI のセッションの sandbox で書かせないパスのうち、足す分。`${home}` か `${worktree}` で始めて書き、絶対パスは書かない | 一覧（`${home}` か `${worktree}` で始まるパスの文字列） | 空（暫定） | 書き込みは、自分のセッションの worktree と一時フォルダなどに限る（[SAFE-10][safety]）。`denyRead` と同じく、設定には足す分だけを持つ |

### マシンの設定の項目

| キー | 中身 | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- | --- |
| `repository.owner`、`repository.name` | 当てる先のリポの owner と名前 | 文字列 | `setup` が、開いているプロジェクトの origin から書く | 当てる先のリポを設定に持つ（[STRUCT-2][struct]）。進行 Bot の clone を作る先と、全体の止める合図を読む先なので、リポの設定を読む前に要る。そのためマシンの設定に置く（[ADR-20261021-006][adr-repo] の決定 1。base ブランチと同じ理由） |
| `githubApp.appId`、`githubApp.installationId` | 進行 Bot が使う GitHub App の ID と、当てる先のリポへのインストールの ID | 数か `null` | `null`（数か `null`。切り替えのときに人が書く。`null` の間は PAT で読む） | 当てる先のリポに初めて書く前に、進行 Bot を GitHub App に切り替える（[ADR-20261020-002][adr-app]、[ADR-20261024-002][adr-no-stages] の決定 2）。全体の止める合図を読む前の認証に使うので、リポの設定を読む前に要る。そのためマシンの設定に置く（[ADR-20261021-006][adr-repo] の決定 1。[ADR-20261020-002][adr-app] の決定 3 は、置き場を基本設計に任せた）。どちらも秘密でない。秘密鍵は秘密の置き場に置く。進行 Bot を動かすマシンを替えたら、新しいマシンにも書く |
| `baseBranch` | PR を向ける base ブランチの名前（進行 Bot はこのブランチの先端からリポの設定を読む） | 文字列 | 無し。`setup` のマシンの準備で人が指定する（business-os は `develop`。[IMPL-6][impl]、[MERGE-2][merge]） | 進行 Bot は base ブランチの先端からリポの設定を読むので、base ブランチの名前はリポの設定を読む前に要る。そのためマシンの設定に置く（[ADR-20261017-007][adr-stop] の決定 4） |
| `devAutopilotPath` | 進行 Bot のプログラムの写しのフォルダ | 文字列（絶対パス） | `setup` が `<paths.workspace>/program/<版>/` を書く（暫定） | `setup` は、入れた Plugin の版を置き場の中の版ごとのフォルダに写し、定期の実行は写しから進行 Bot を起こす（[STRUCT-3][struct]） |
| `program.previousVersion` | 直前まで使っていた進行 Bot のプログラムの版（`setup` の `rollback` の戻す先で、写しの片付けで残す版） | 文字列か `null` | `null`（`setup` が版を上げるときに、上げる前の版を書く） | 直前の版を持たないと、同じ版で `setup` を実行し直したときに戻す先の写しを消し、`rollback` が戻す先を決められない（「[導入と点検][design-setup]」の「プログラムの写し」） |
| `program.pinned` | `rollback` で版を固定しているか。`true` の間の `setup` は版を上げない | 真偽 | `false` | `rollback` の後に、登録し直しなどで `setup` を実行し直すと、入れた Plugin の新しい版に黙って戻るため。版を上げるのは、人が `setup` を引数 `upgrade` で実行したときだけにする（[STRUCT-3][struct] の「版を上げるのは、人が `setup` を実行したときだけ」） |
| `paths.workspace` | 置き場（進行 Bot の clone、セッションごとのフォルダ、プログラムの写し、GitHub の SSH のホストの鍵、安全設定の置き場の既定の場所） | 文字列（絶対パス） | 「[マシンの中のフォルダ][places]」 | AI のセッションごとに origin から clone し直す。進行 Bot の clone は `controller/`、セッションのフォルダは `sessions/<セッションの名前>/` に置く（[ADR-20261013-001][adr-workspace]） |
| `paths.records` | 実行の記録と lock の置き場 | 文字列（絶対パス） | 「[マシンの中のフォルダ][places]」 | 実行の記録を追跡しないフォルダに残す（[FLOW-5][flow]）。記録と lock の置き場はマシンの設定に置く（[ADR-20261015-005][adr-settings]） |
| `paths.secrets` | 秘密の置き場（PAT かアプリの秘密鍵、deploy key、印の署名の鍵） | 文字列（絶対パス） | 「[マシンの中のフォルダ][places]」 | PAT・印の署名の鍵・deploy key を、秘密の置き場のファイルに置く（[ADR-20261015-001][adr-read] の決定 2）。GitHub App に切り替えた後は、PAT の代わりにアプリの秘密鍵を置き、PAT のファイルは消す（[ADR-20261020-002][adr-app] の決定 1・3） |
| `paths.sandboxSettings` | AI のセッションに `--settings` で渡す安全設定のファイルの置き場 | 文字列（絶対パス） | `<paths.workspace>/sandbox-settings/`（暫定） | 安全設定のファイルは AI が書けない場所に置き、起動の前にハッシュを確かめる（[SAFE-2][safety] の (e)）。進行 Bot がセッションごとに作り、終わった後にもハッシュを比べて消す（[AI のセッション][design-ai]の「セッションごとの安全設定」） |
| `safety.sandbox.toolDirs` | sandbox の `allowRead` に足す、ホームの中にある道具のフォルダ | 一覧（絶対パスの文字列） | `setup` が、`node` `pnpm` `git` の実体のうちホームの中にあるものの、実体のファイルがあるいちばん狭いフォルダを書く。人の pnpm の store（人の環境の `pnpm store path`）を含むフォルダは書かない。`gh` は AI のセッションで使わないので書かない（暫定） | sandbox はホーム（`~/`）を読ませないので、ホームの中にある道具のフォルダを `allowRead` に足す（[SAFE-10][safety]）。人の store を含むフォルダを足すと、共有の store を読めてしまう（[SAFE-10][safety]、[ADR-20261015-002][adr-store]）。マシンの設定は PR を通らないので、`check` が、各フォルダがホーム（`~/`）自体でなく、`node`・`pnpm`・`git` の実体のどれかを含み、人の pnpm の store・`~/.ssh`・`~/.config/gh`・`~/.claude/`・マシンの設定のフォルダ・マシンの中のフォルダを含まないことを確かめ、外れれば fail にする（`allowRead` は狭い方の規則が勝つので、広いフォルダを書くとホームの中の秘密を読めるため。FACT-22） |
| `safety.env.values` | リポの設定の `safety.env.names` に足した名前の、環境変数の値 | 対応表（キーは環境変数の名前、値は文字列） | 空（暫定） | 定期の実行は人のシェルの環境を持たないので、足した名前の値を、マシンの設定から定期の実行の定義に写して進行 Bot に届ける（「[導入と点検][design-setup]」の「定期の実行の登録」）。プロキシや証明書の場所のように、マシンごとに違いうる値なので、マシンの設定に置く。値に秘密を書かない（`check` が、秘密の形が無いことを確かめる。[SAFE-1][safety]） |
| `sessionTmp.root` | セッションごとの短い一時フォルダを作る根 | 文字列（絶対パス） | 「[マシンの中のフォルダ][places]」 | セッションごとの短い一時フォルダを、ホーム・置き場・`/tmp/claude-<uid>/` の外の、dev-autopilot 専用の短い根の下に作る（[ADR-20261017-006][adr-tmp]、[SAFE-10][safety]） |
| `sessionTmp.idLength` | 短い一時フォルダの名前（セッションの短い ID）の文字数 | 数（1 以上の整数。文字） | 8 文字（暫定） | 根と ID の長さの値は、設定の表で決める（[ADR-20261017-006][adr-tmp]）。8 文字にすると、`<根>/<ID>/claude-<uid>` が 44 バイト以内に収まる（実体のパスで数え、macOS は uid が 11 桁まで、Linux は 19 桁まで。上の「[マシンの中のフォルダ][places]」） |
| `sessionTmp.maxBytes` | `<CLAUDE_CODE_TMPDIR>/claude-<uid>` の長さの上限。超えたら AI のセッションを起動しない | 数（1〜44 の整数。バイト） | 44 バイト（[SAFE-10][safety]） | `<CLAUDE_CODE_TMPDIR>/claude-<uid>` が長いと、Claude Code は黙ってユーザーごとの一時フォルダに戻すので、進行 Bot は起動の前に長さを数え、上限を超えたら起動しない（[SAFE-10][safety]、[ADR-20261017-006][adr-tmp]）。Claude Code の振る舞いで決まる値で、Claude Code はマシンごとに入れるので、マシンの設定に置く |

## `check` が設定について確かめること

SETUP-2 の点検のうち、設定に係るものをまとめる。点検の組と回数（いつ・どの認証で確かめるか）は「[導入と点検][design-setup]」の「組と回数」が正本で、ここは設定の値から見た一覧。

- すべての項目が、リポの設定かマシンの設定のどちらか一方にだけあり、型と範囲が表の形の欄のとおりであること（範囲の外の値で守りが黙って緩む形を止める）
- 表に無いキーが無いこと。綴りの誤りで、足したつもりの守り（`safety.sandbox.denyRead` など）が黙って効かない形を止めるため。Plugin の版で項目を消したときは、`setup` を実行し直すと、リポの設定からキーを消す PR を出し、マシンの設定からキーを消す（どちらも `setup` の承認の一覧に出す）
- リポの設定に絶対パス（`/Users/` `/home/` などで始まる値）が無いこと
- マシンの設定のフォルダと、上の「[マシンの中のフォルダ][places]」の表のフォルダすべて（`paths.sandboxSettings` を含む）の実体のパスが、同期フォルダの下・`~/.claude/` の下・`/tmp/claude-<uid>/` の下に無いこと。置き場のほかのフォルダは、置き場の `sessions/` の下と短い一時フォルダの根の下（AI が書ける場所）にも無いこと。記録や安全設定を AI が書き換えられる場所に置くと、上限の数え上げと安全設定のハッシュの確かめが効かなくなるため
- マシンの設定の `repository.owner` と `repository.name` が、マシンの設定のフォルダのパス（`<owner>/<repo>`）と一致すること。人が呼ぶ `check` では、開いているプロジェクトの origin とも一致すること
- origin に `baseBranch` があり、その先端にリポの設定があること。リポの設定に書いた Project と選択肢の ID が実在すること（[ADR-20261017-007][adr-stop] の決定 4）
- base ブランチの先端に `.claude/settings.local.json` が無く、どのフォルダにも `CLAUDE.local.md` が無いこと。base ブランチの先端は AI のセッションの worktree に入り、`.claude/settings.local.json` は `--setting-sources local` でも読まれて hook が sandbox の外で動き、`CLAUDE.local.md` は指示として AI の文脈に入るため（FACT-56、FACT-60、[SAFE-2][safety] の (a)）。点検で止めないと、起動の前の確かめ（[AI のセッション][design-ai]の「起動の前の確かめ」の順 9）が、作業の単位ごとに停止中（`local-settings`）にしていく。名前は、base ブランチの先端の git の木（進行 Bot の clone で読む）のすべてのフォルダをたどり、英字の大文字と小文字を区別せずに比べる（`.Claude/settings.local.json` と `sub/claude.local.md` も当てる。FACT-68）。綴りどおりに比べるか、ファイルシステムに名前で問うだけにすると、大文字と小文字を変えた名前か、下のフォルダの `CLAUDE.local.md` を見落とすため。fail のときは、試しのセッションも起こさない（試しのセッションの worktree も base ブランチの先端で切るため。「[導入と点検][design-setup]」の「試しのセッション」）
- `checks.required` が空でないこと
- `run.intervalHours` が 24 の約数であること
- `safety.botAccounts` が空でなく、投稿に使うアカウントが入っていること。GitHub App に切り替える前は PAT の持ち主（`GET /user` の `login`）、切り替えた後はアプリの投稿者の名前（取り方は「[導入と点検][design-setup]」の「GitHub App の権限」）（[ADR-20261020-001][adr-bots]、[ADR-20261020-002][adr-app]）。切り替えた後に、一覧にほかの名前が残っていれば warn にする（外し忘れを人に知らせる。前のアカウントの印が残る間は残してよい。[ADR-20261020-001][adr-bots] の決定 4）
- `githubApp.appId` と `githubApp.installationId` があり、秘密の置き場にアプリの秘密鍵があって、PAT のファイルが無いこと。GitHub App に切り替える前（PAT のまま）は fail にする。進行 Bot は当てる先のリポに初めて書く前に GitHub App に切り替える（[SETUP-13][setup]、[ADR-20261020-002][adr-app]、[ADR-20261024-002][adr-no-stages] の決定 2）ので、PAT のまま作業に入らないため
- `models.planHard` のほかの `models.*` の `model` に `fable` が無いこと（[モデルの流れ][impl]、[IMPL-32][impl]）
- `safety.env.names` に、AI のセッションに渡さない名前（`GH_TOKEN`、`PNPM_HOME` など。「[AI のセッション][design-ai]」の「環境変数」）と、プログラムに固定した名前が無いこと。`safety.env.names` の名前がすべて進行 Bot の環境変数にあること（値はマシンの設定の `safety.env.values` から定期の実行の定義に写す）。`safety.env.values` の値に秘密の形（漏えいの検査の鍵の形式など。[IMPL-15][impl]）が無いこと
- `safety.sandbox.toolDirs` の各フォルダが、ホーム（`~/`）自体でなく、`node`・`pnpm`・`git` の実体のどれかを含み、人の pnpm の store・`~/.ssh`・`~/.config/gh`・`~/.claude/`・マシンの設定のフォルダ・マシンの中のフォルダを含まないこと（[ADR-20261015-002][adr-store]、[SAFE-10][safety]）
- `<sessionTmp.root の実体のパス>/<sessionTmp.idLength の文字数の ID>/claude-<uid>` の長さが `sessionTmp.maxBytes` 以内で、`sessionTmp.maxBytes` が 44 以下であること（[SAFE-10][safety]、FACT-37）。根を長くすると、起動の前の確かめで AI のセッションを黙って起こさなくなり、上限を 45 以上にすると、Claude Code がユーザーごとの一時フォルダに戻したまま起動するため
- 秘密の置き場の `marker-signing-keys` に鍵の行が 1 つ以上あり、どの行も、末尾の空白と改行を除いて base64 として解け、解いた値が 32 バイトであること（読み方は「[状態遷移と判定][design-flow]」の「印の署名」）。鍵の行が 1 つも無ければ fail にする（空のファイルを「無い」と読む。[ADR-20261021-004][adr-marker-keys] の決定 1）
- 投稿に使うアカウントが当てる先のリポに書いた、最新の未編集の印の署名が、`marker-signing-keys` のどれかの鍵で通ること。通らなければ warn にする。動かすマシンを替えたときの鍵の取り違えを、人に知らせるため（印が 1 つも無いときは、比べる印が無いので確かめない）。fail にしないのは、鍵が漏れて一覧から外したときに、進行 Bot が次の実行で、漏れた鍵の印のある開いた Issue と PR を `marker-invalid` で止めるため（fail にすると、進行 Bot が作業に入らず、止められない。[ADR-20261021-004][adr-marker-keys] の決定 3）。取り違えても、前の鍵の印のある Issue と PR は `marker-invalid` で止まり、先へは進まない
- 費用の上限 4 つが、上の「[費用の上限の値][cost-values]」の関係を保つこと
- 定期の実行の定義が渡すマシンの設定のパスと、人が呼ぶ Skill が origin から見つけるパスが一致すること。定期の実行の間隔が `run.intervalHours` と一致すること

[adr-settings]: ../../adr/adr-20261015-005-settings-values-and-machine-settings.md
[adr-stop]: ../../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-human]: ../../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-read]: ../../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-tmp]: ../../adr/adr-20261017-006-per-session-short-tmpdir.md
[adr-usage]: ../../adr/adr-20261018-001-stop-session-and-count-usage.md
[adr-planrev]: ../../adr/adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[adr-workspace]: ../../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[design-change]: ../README.md#設計を変えるとき
[places]: #マシンの中のフォルダ
[flow]: ../../requirements/01-issue-flow.md
[flow-rules]: ../../requirements/01-issue-flow.md
[markers]: ../../requirements/04-external-text-and-markers.md
[review-skill]: ../../requirements/08-review-skill.md
[adr-table]: ../../adr/adr-20261018-005-settings-table-shows-required-values.md
[adr-redo]: ../../adr/adr-20261018-002-plan-redo-in-same-run.md
[impl]: ../../requirements/02-implement-review-fix.md
[merge]: ../../requirements/03-merge.md
[text]: ../../requirements/04-external-text-and-markers.md
[safety]: ../../requirements/05-safety.md
[setup]: ../../requirements/06-setup-and-stages.md
[struct]: ../../requirements/07-structure.md
[design-ai]: ./03-ai-sessions.md
[adr-gaps]: ../../adr/adr-20261019-002-review-gaps-axis-in-shadow-first.md
[adr-replan]: ../../adr/adr-20261019-003-replan-on-plan-review-from-stage-5.md
[design-setup]: ./04-setup-and-check.md
[cost-values]: #費用の上限の値
[doc-costs]: https://code.claude.com/docs/en/costs
[doc-plans]: https://claude.com/pricing
[design-flow]: ./02-flow-and-judgment.md
[check-settings]: #check-が設定について確かめること
[path-glob]: #検査と判定
[command-timeout]: #子プロセスが時間の上限を過ぎたとき
[adr-flow]: ../../adr/adr-20261015-006-run-stages-baseline-agent-field-and-local-records.md
[adr-bots]: ../../adr/adr-20261020-001-bot-accounts-as-list.md
[adr-app]: ../../adr/adr-20261020-002-switch-to-github-app-before-writing.md
[adr-store]: ../../adr/adr-20261015-002-no-shared-dependency-store.md
[adr-repo]: ../../adr/adr-20261021-006-repo-owner-and-name-in-machine-settings.md
[adr-blocked]: ../../adr/adr-20261021-005-required-push-blocked-paths-in-program.md
[adr-marker-keys]: ../../adr/adr-20261021-004-marker-key-list-and-resume-after-invalid-markers.md
[adr-rounds]: ../../adr/adr-20261022-003-count-round-trips-by-ai-pushes.md
[adr-intake-req]: ../../adr/adr-20261022-004-intake-skill-via-request-file.md
[adr-no-stages]: ../../adr/adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[adr-master]: ../../adr/adr-20261024-004-master-draft-quarantine-and-protected-paths.md
[adr-cleanup]: ../../adr/adr-20261024-006-cleanup-states-and-dependencies.md
[adr-status]: ../../adr/adr-20261026-001-use-status-field-for-state.md
[adr-leftover]: ../../adr/adr-20261027-001-find-leftover-processes-before-deleting-workspaces.md
[adr-claude-dir]: ../../adr/adr-20261027-003-approved-claude-md-by-ai-and-claude-dir-by-humans.md
