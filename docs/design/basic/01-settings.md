# 設定

進行 Bot と、人が呼ぶ Skill（`setup`・`check`・`/dev-autopilot`）が読む設定の、ファイルの置き場・形・項目を決める。
設定は、リポの設定とマシンの設定の 2 つのファイルに分け、項目ごとに置き場をどちらか一方だけにする（STRUCT-2、[ADR-20261015-005][adr-settings] の決定 2）。

## どの項目がどちらにあるか

| 設定 | ファイル | 置く場所 | 持つ項目 |
| --- | --- | --- | --- |
| リポの設定 | `.claude/dev-autopilot.json` | 当てる先のリポの中（commit する） | 項目の大半：リポと Project、ラベル名と固定の見出し、進行 Bot の実行（`dryRun`・`stage`・定期の実行の間隔など）、上限、モデル、検査と判定、規約、安全（進行 Bot のアカウント名・通信の許可先・子プロセスに渡す環境変数の名前など） |
| マシンの設定 | `settings.json` | リポの外の、OS ごとの決まった場所。当てる先のリポごとにフォルダを分ける | マシンごとに違う 10 項目：base ブランチの名前、進行 Bot のプログラムの写しのパス（`devAutopilotPath`）、置き場のパス 4 つ（作業場所・記録・秘密・安全設定）、ホームの中の道具のフォルダ、短い一時フォルダの根・ID の長さ・長さの上限 |

マシンの設定をリポの中に置かない理由は 3 つある。

- マシンのパス（ユーザー名を含む）を公開のリポに入れない（[ADR-20261015-005][adr-settings] の決定 2）
- gitignore して人の作業コピーに置くと、人がふだん使う Claude Code が書き換えられる。`devAutopilotPath` を書き換えられると、PAT と deploy key を持つ定期の実行が別のプログラムを起こす。進行 Bot は人の作業コピーを読まない（[ADR-20261017-007][adr-stop] の決定 4、SAFE-2）
- 人の作業コピーに置くと、定期の実行が作業コピーの場所に縛られ、人が作業コピーを動かすか消すと止まる

## 値の決まり方

- 要件が初期値を決めている項目も、表に値を書き、出典の要件へリンクする。値の正本は要件で、食い違っていたら要件が正しい。要件の初期値を変える PR では、表も直す（[ADR-20261018-005][adr-table]）
- 要件が初期値を決めていない項目は、表に初期値と根拠を書き、「暫定」と添える。段階の出口の記録（FLOW-5、FLOW-13）で見直す（[ADR-20261015-005][adr-settings] の決定 1）
- 「メンテナが決める」と書いた項目は、要件にも ADR にも根拠になる値が無く、利用枠の使い方などメンテナの事情で決まる。値が空のままなら `check` は fail にし、進行 Bot を動かさない
- `setup` は、表の初期値で設定ファイルを作る。今の値の正本は設定ファイル。初期値の正本は、要件が決めた値は要件、それ以外は表
- リポの設定の値は PR で変える。マシンの設定の値は、そのマシンで人が書き換える。安全の境界を緩める変更は、値の変更でも ADR で決める（[設計を変えるとき][design-change]）

## リポの設定

- 当てる先のリポの `.claude/dev-autopilot.json`（commit する）。形は JSON
- 進行 Bot は、毎回の実行の最初に、進行 Bot の clone で fetch した base ブランチの先端から読む。読めないか形が違えば、作業に入らず止まる（[ADR-20261017-007][adr-stop] の決定 4、SAFE-4）
- マシンに依らない値だけを持ち、絶対パスを書かない（`check` が確かめる）
- AI のセッションは書き換えられない（`.claude/**` は IMPL-15 の push しないパス）

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

## マシンの中の置き場

`setup` は、次の既定のパスで置き場を作り、マシンの設定に書く。人が別の場所を指定したときは、指定した値を書く。

| 置き場 | キー | macOS | Linux | 中身 |
| --- | --- | --- | --- | --- |
| 置き場 | `paths.workspace` | `~/.local/share/dev-autopilot/<owner>/<repo>/` | `$XDG_DATA_HOME/dev-autopilot/<owner>/<repo>/`（`XDG_DATA_HOME` が無ければ `~/.local/share`） | 進行 Bot の clone（`controller/`）、セッションごとのフォルダ（`sessions/<セッションの名前>/`）、進行 Bot のプログラムの写し（`program/<版>/`）、安全設定のファイル（`sandbox-settings/`） |
| 記録 | `paths.records` | `<マシンの設定のフォルダ>/records/` | `$XDG_STATE_HOME/dev-autopilot/<owner>/<repo>/records/`（`XDG_STATE_HOME` が無ければ `~/.local/state`） | 実行の記録（FLOW-5）、lock、`setup` が置く印のファイル（「[運用の規則][flow-rules]」） |
| 秘密の置き場 | `paths.secrets` | `<マシンの設定のフォルダ>/secrets/` | `<マシンの設定のフォルダ>/secrets/` | PAT、deploy key、印の署名の鍵。フォルダの権限は 700、ファイルは 600（[ADR-20261015-001][adr-read] の決定 2） |
| 短い一時フォルダの根 | `sessionTmp.root` | `/tmp/dap` | `/tmp/dap` | セッションごとの短い一時フォルダ（[ADR-20261017-006][adr-tmp]） |

- 置き場を macOS で `~/Library/Application Support/` の下に置かないのは、パスに空白が入るため。AI は置き場の中の worktree で品質ゲート（`pnpm check`）とパッケージのスクリプトを動かし、空白を含むパスで失敗する道具がある
- 記録と秘密の置き場は、AI が道具を動かさないので、macOS ではマシンの設定のフォルダの中に並べる（秘密の置き場をマシンの設定と並べてよいことは [ADR-20261015-005][adr-settings] の決定 2）。Linux の記録は、XDG の決まり（状態のファイルは `XDG_STATE_HOME`）に合わせる
- どの置き場も、`~/.claude/` と `/tmp/claude-<uid>/` の下に置かない（SAFE-10）。`check` は、実体のパスが同期フォルダの下に無いことを確かめる（[ADR-20261015-005][adr-settings] の決定 2）
- 短い一時フォルダの根 `/tmp/dap` は、[ADR-20261017-006][adr-tmp] の例のまま。`<根>/<ID>/claude-<uid>` を上限（`sessionTmp.maxBytes`）の 44 バイト以内に収めるため、根は短くする。ID を 8 文字にすると、uid は 19 桁まで収まる

## 設定ファイルの外の合図

- 全体を止める合図は、当てる先のリポの Actions の変数 `DEV_AUTOPILOT_RUN`。値が `run` のときだけ動き、ほかの値・変数が無い・読めないときは止まる（「[運用の規則][flow-rules]」、[ADR-20261017-007][adr-stop]）
- 変数の名前は設定に持たない。進行 Bot は、リポの設定を読む前の実行の最初に合図を読むため

## 項目

キーは、ファイルの中の JSON の位置を `.` でつないで書く（例：`limits.session.minutes`）。`check` は、初期値の欄の値の形（数・文字列・真偽・一覧・対応表）で型を確かめる（SETUP-2）。
要件が決めた初期値には、出典の要件へのリンクを添える。正本は要件で、食い違っていたら要件が正しい（[ADR-20261018-005][adr-table]）。

### リポの設定の項目

#### リポと Project

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `repository.owner`、`repository.name` | 当てる先のリポの owner と名前 | `setup` が origin から書く | 当てる先のリポを設定に持つ（[STRUCT-2][struct]）。`check` が、設定の値と origin が一致することを確かめる |
| `project.number` | Issue を並べる GitHub の Project の番号 | `setup` が使う Project の番号を書く | 進行 Bot は、設定した Project の Issue を読んで候補を選ぶ（[FLOW-1][flow]）。Status と Agent 欄は `setup` が足す（[SETUP-5、SETUP-6][setup]） |
| `project.statusField.id`、`project.statusField.options` | Project の Status の欄の ID と、選択肢（`todo` `inProgress` `inReview` `readyToMerge` `blocked` `done`）ごとの ID | `setup` が足した欄と選択肢の ID を書く | `setup` が Status に In Review・Ready to Merge・Blocked を足し、進行 Bot が Status を変える（[SETUP-5][setup]、[Issue の状態の遷移][flow]） |
| `project.agentField.id`、`project.agentField.options` | Project の Agent 欄の ID と、選択肢（`worker` `reviewer` `human` `paused`）ごとの ID | `setup` が足した欄と選択肢の ID を書く | AI を担当者にできないので、`setup` が Project に単一選択の Agent 欄を足し、進行 Bot が担い手を Agent 欄で示す（[SETUP-6][setup]） |

#### ラベル名と固定の見出し

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `labels.agentReady` | 人が処理対象にした Issue に付けるラベル | `agent-ready`（[SETUP-8][setup]） | 人がこのラベルを付ける 1 操作が、Issue を処理対象にする判断になる（[TEXT-12][text]）。`setup` が足し、名前は設定に持つ（[SETUP-8][setup]） |
| `labels.needsHuman` | 人の判断が要る Issue に付けるラベル | `needs-human`（[SETUP-8][setup]） | 人が引き取るまで進行 Bot が触らない印（[ラベルの一覧][flow]）。`setup` が足し、名前は設定に持つ（[SETUP-8][setup]） |
| `labels.fromReview` | 🟢 の指摘から起票した Issue に付けるラベル | `from-review`（[SETUP-8][setup]） | 進行 Bot が 🟢 の指摘から起票した Issue に付け、人がマイルストーンを付けるまで受け入れの対象から外す（[IMPL-26][impl]、[TEXT-12][text]）。`setup` が足す（[SETUP-8][setup]） |
| `labels.planFable`、`labels.planOpus` | 人が計画のモデルを指定するラベル | `plan:fable`、`plan:opus`（[SETUP-8][setup]） | 人がラベルで計画 AI のモデルを上書きする。ラベルは受け入れの判定より優先する（[IMPL-30][impl]）。`setup` が足す（[SETUP-8][setup]） |
| `labels.branchTypes` | Issue のラベルから作業ブランチの型を決める対応表 | `bug` → `fix`（暫定） | 作業ブランチは `<型>/issue-<番号>` で、型は設定のラベル対応表から進行 Bot が決める。初期値は要件の例（[IMPL-2][impl]） |
| `labels.defaultBranchType` | 対応表に無いラベルの Issue の、作業ブランチの型 | `feat`（暫定） | 要件の例の「それ以外 → `feat`」（[IMPL-2][impl]） |
| `headings.workInstruction` | コラボレータが作業指示を書くコメントの見出し | `## 作業指示`（[印と固定の見出しの一覧][markers]） | コラボレータがこの見出しで書いたコメントは、受付 AI の下書きより優先して作業 AI に渡す（[TEXT-1、TEXT-12][markers]） |
| `headings.rejection` | 人がレビューの指摘を却下するコメントの見出し（`<n>` はラウンド） | `## 却下 R<n>`（[印と固定の見出しの一覧][markers]） | 進行 Bot は、push 以上の権限を持つ人がこの見出しで書いたコメントだけを、却下した指摘として数える（誤検知率の記録。[FLOW-14][flow]） |
| `headings.differentJudgment` | 人が `merge=ready` の PR をマージしない理由を書くコメントの見出し | `## 判定と違う判断`（[印と固定の見出しの一覧][markers]） | 人がこの見出しで書いたら、進行 Bot の判定と人の判断の不一致に数える（判定の信頼度の記録。[MERGE-3][merge]） |
| `headings.objection` | 人が選定・待機・下書き・計画に異議を書くコメントの見出し | `## 異議`（[印と固定の見出しの一覧][markers]） | 受け入れ工程（段階 4）と計画・実装工程（段階 5）の出口で、人の異議の回数を数える（[段階の表][stages]、[印と固定の見出しの一覧][markers]） |

#### 進行 Bot の実行

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `run.dryRun` | GitHub に書かずに試すか（`true` なら、書くつもりだった内容を手元の記録にだけ残す） | `true`（受け入れ工程（段階 4）の最初。[運用の規則][flow]） | GitHub への書き込みをすべて手元の記録に残して試す合図で、受け入れ工程（段階 4）の最初と新しい段階に入るときに使う（[運用の規則][flow]、[ADR-20261017-007][adr-stop]）。`false` にする（書き始める）のは、安全の境界を緩める向きの変更なので PR に残す |
| `run.stage` | 今の段階の番号（段階ごとの振る舞いを切り替える） | `4`（受け入れ工程。暫定） | 段階ごとの振る舞い（計画・実装工程（段階 5）では Agent 欄が空、修正工程（段階 7）では判定の印を書かない、など）を、設定の `stage` で切り替える（[Issue の状態の遷移][flow]）。Issue の流れは受け入れ工程（段階 4）から動く（[段階の表][stages]） |
| `run.intervalHours` | 定期の実行の間隔 | 24 時間（1 日 1 回。[FLOW-12][flow]） | 間隔と件数の上限は、リポの設定 1 つで変える（[FLOW-11][flow]）。間隔は OS ごとの定期の実行の登録にも書くので、`check` がずれを確かめる（[STRUCT-2][struct]） |
| `run.maxMinutes` | 1 回の実行の所要時間の上限。超えたら残りの Issue は次の実行に回す | 180 分（暫定） | 1 回の実行の所要時間の上限を設定に持ち、超えたら残りの Issue は次の実行に回す。lock を長く持たないため（[FLOW-4][flow]）。180 分は、重い工程 1 つ（1 セッションは最大 45 分）の後にも、軽い工程とほかの Issue の重い工程を進める余地を持たせる値で、[ADR-20261015-005][adr-settings] の補足情報が候補に挙げた値 |
| `run.stopSignalPollSeconds` | AI のセッションが動いている間に、全体を止める合図を読む間隔 | 60 秒（暫定） | AI のセッションが動いている間も、全体を止める合図を一定の間隔で読む。間隔は基本設計で決める（[運用の規則][flow]）。60 秒にすると、止めると決めてから AI のセッションが止まるまでが 1 分ほどになる。1 セッションの間に読む回数は数十回で、GitHub の API の上限（1 時間に 5,000 回）に比べて少ない |

#### 上限

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `limits.startsPerRun` | 1 回の実行で新しく着手する Issue の数の上限 | 1 件（[FLOW-3][flow]） | 進行 Bot は、1 回の実行で着手する件数に上限を持つ（[FLOW-3][flow]） |
| `limits.wip` | 進行中の Issue（`In Progress` から `Done` の前まで）の数の上限 | 1 件（マージ判定工程（段階 8）まで。[FLOW-3][flow]） | 進行中の Issue の数にも上限を持ち、上限に達していれば新しい Issue に着手しない（[FLOW-3][flow]） |
| `limits.intakePerRun` | 1 回の実行で受け入れる Issue の数の上限 | 5 件（暫定） | 1 回の実行で受け入れる件数に上限を持つ。初回の実行で未処理の Issue すべてに走らないためと、外の人が Issue を大量に立てたときの利用枠の歯止め（[TEXT-12][text]）。5 件は [ADR-20261015-005][adr-settings] の補足情報が候補に挙げた値 |
| `limits.roundTrips` | レビューと修正の往復の回数の上限。超えたら `needs-human` | 3 回（[IMPL-24][impl]） | 往復の回数に上限を持ち、超えたら Status を Blocked にして `needs-human` を付ける。同じ指摘が再び出たとき（同じ指摘の再発）も同じ扱い（[IMPL-24][impl]） |
| `limits.planRevisions` | 計画のレビューの指摘で計画し直す回数の、計画の工程ごとの上限。上限まで計画し直しても `fix` なら、マージ判定工程（段階 8）の前は最後の計画を投稿する | 1 回（[IMPL-10][impl]） | 計画は実行して確かめられないので、レビューのたびに直すところが出て、ループが終わらないおそれがある。1 回なら、計画の工程は最大 5 つのセッション（1 セッションの上限 45 分なら 225 分）で終わる（[IMPL-10][impl]、[ADR-20261019-003][adr-replan]） |
| `limits.session.minutes` | AI のセッション 1 回の時間の上限。超えたら進行 Bot が止める | 45 分（[IMPL-13][impl]） | `claude` には時間の上限が無いので、進行 Bot が時間で止める（[IMPL-13][impl]）。止めた回は実行の失敗として次回に持ち越す（[IMPL-14][impl]） |
| `limits.session.turns` | AI のセッション 1 回のターンの上限（`--max-turns`） | 200 ターン（[IMPL-13][impl]） | ターンの上限は `--max-turns` で掛ける（[IMPL-13][impl]。GitHub Agentic Workflows の既定は 500 ターン） |
| `limits.session.costUsd` | AI のセッション 1 回の費用の上限（`--max-budget-usd`）。途中で止めた回は、この値をその回の費用とみなす | メンテナが決める（USD） | 費用の上限を `--max-budget-usd` でも掛ける（[IMPL-14][impl]）。途中で止めた回は費用が分からないので、この値をその回の費用とみなす（[ADR-20261018-001][adr-usage]） |
| `limits.issueCostUsd` | 1 Issue の累計の費用の上限（往復を含む）。起動の前に、累計にこの回の費用の上限を足して超えるなら、起動せずに `needs-human` | メンテナが決める（USD） | 1 Issue の累計の利用枠（往復を含む）に上限を持ち、費用で数える。当たったら `needs-human`（[IMPL-14][impl]、[ADR-20261018-001][adr-usage]） |
| `limits.runCostUsd` | 1 回の実行の費用の上限。起動の前に、その回の累計にこの回の費用の上限を足して超えるなら、起動せずに次の実行で続ける | メンテナが決める（USD） | `claude -p` の実行は利用枠を消費するので、1 回の実行の上限を持つ（[SETUP-12][setup]、[IMPL-14][impl]） |
| `limits.dailyCostUsd` | 1 日（マシンの時刻の日付）の費用の上限。起動の前に、その日の累計にこの回の費用の上限を足して超えるなら、起動せずに次の日の実行で続ける | メンテナが決める（USD） | 1 日の利用枠にも上限を持ち、費用で数える（[SETUP-12][setup]、[IMPL-14][impl]） |
| `limits.stopGraceSeconds` | AI のセッションを途中で止めるとき、`SIGTERM` を送ってから `SIGKILL` を送るまでの猶予 | 10 秒（暫定） | 途中で止めるときは `SIGTERM` を送り、猶予の間に終わらなければ `SIGKILL` を送る。猶予は設定に持つ（[IMPL-14][impl]）。試験では `SIGTERM` の後 1 秒ほどで終わった（[ADR-20261018-001][adr-usage]）ので、10 倍の余裕を見る |
| `limits.retry.count` | 同じ Issue で実行の失敗が続いたとき、`needs-human` にするまでの回数 | 3 回（[IMPL-14][impl]） | 実行の失敗（利用枠切れ・時間の上限など、作業の中身と無関係の失敗）は次回に持ち越し、同じ Issue で続いたら `needs-human` にする（[IMPL-14][impl]） |
| `limits.retry.intervalRuns` | 実行の失敗の後、何回目の定期の実行で再試行するか | 1（次の定期の実行。暫定） | 実行の失敗の再試行の間隔を設定に持つ（[IMPL-14][impl]）。実行の中で待つと lock を長く持つので、次の定期の実行で再試行する（[FLOW-4][flow]）。[ADR-20261015-005][adr-settings] の補足情報が挙げた候補（再試行は次の実行） |
| `limits.remindDays` | `merge=ready` の PR が、マージも「判定と違う判断」のコメントも無いまま経ったとき、進行 Bot が 1 回だけ催促するまでの日数 | 7 日（[MERGE-3][merge]） | 人が動かない PR は、判定の信頼度の記録（マージ判定工程（段階 8）の 10 件）に数えられないので、1 回だけ催促する（[MERGE-3][merge]） |
| `limits.ciStaleHours` | CI の結果が `cancelled` `skipped` `neutral` `stale` のまま変わらないとき、`needs-human` にするまで待つ時間 | 48 時間（暫定） | CI の結果が `cancelled` などのまま、完了から設定の時間が経っても変わらなければ `needs-human` にする（[IMPL-27][impl]）。48 時間は、1 日 1 回の間隔（[FLOW-12][flow]）で、人が再実行する機会を間隔 1 回分より長く取る値 |

#### モデル

役ごとのモデルと effort は、`model` と `effort` の 2 つの欄を持つ（例：`models.plan.model`、`models.plan.effort`）。モデルは別名で書く（`haiku` だけは別名が旧版に解けるので完全な ID。[モデルの流れ][impl]）。

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `models.intake` | 受付 AI（受け入れと計画の難しさの判定） | `claude-haiku-5-5` / medium（[IMPL-29][impl]） | 役ごとのモデルの正本は要件の「モデルの流れ」の表（[IMPL-29][impl]）。`haiku` は別名が Haiku 4.5 に解けるので、完全な ID で書く |
| `models.intakeAfterRefusal` | 受付 AI が拒否（`stop_reason: refusal`）で止まった Issue の、次の回の受付 AI | `sonnet`（[IMPL-29][impl]） / medium（effort は暫定。`models.intake` と同じ） | 受付 AI が拒否で止まったら実行の失敗として扱い、次の回は `sonnet` で試す（[IMPL-14、IMPL-29][impl]）。拒否の記録が無いときは `models.intake` で起動する（[運用の規則][flow]） |
| `models.plan` | 計画 AI（既定。計画のレビューの指摘で計画し直すときは、直す前の計画と同じ値） | `opus` / xhigh（[IMPL-31][impl]） | 計画 AI は既定で `opus`。受け入れの印かラベルが `fable` なら `models.planHard` で計画する（[IMPL-31][impl]） |
| `models.planHard` | 計画 AI（受け入れの判定かラベルで `fable` にしたとき、計画 AI が自己申告して計画し直すとき） | `fable` / high（[IMPL-31][impl]） | `opus` が設計判断などを含むと自己申告したら、同じ実行で 1 回だけ `fable` で計画し直す（[IMPL-31][impl]、[ADR-20261018-002][adr-redo]） |
| `models.work` | 作業 AI の最初の実装 | `opus` / high（[IMPL-32][impl]） | 作業 AI は `opus` で始め、計画・実装工程（段階 5）で人が手直しせずにマージできた種類から `sonnet` に仮に下げる（[IMPL-25、IMPL-32][impl]） |
| `models.workByType` | Issue の種類ごとの、作業 AI の最初の実装 | 空（Issue の種類の分け方を決めるまで。UNDECIDED-52） | 最初の実装に使うモデルは、Issue の種類ごとに決める（[IMPL-25][impl]、[ADR-20261018-003][adr-model]）。種類の分け方が決まるまで空 |
| `models.workRaised` | 最初の実装が `sonnet` だった Issue で、修正のときに上げる先 | `opus` / high（[IMPL-35][impl]） | 最初の実装が `sonnet` なら、修正のときに `opus` に 1 段上げる。`fable` には上げない（[IMPL-32、IMPL-35][impl]） |
| `models.review` | レビュー AI（3 つの軸と見落としの軸とも同じ値） | `opus` / xhigh（[IMPL-33][impl]） | 3 つの軸と見落としの軸とも同じ値で始め（[ADR-20261019-002][adr-gaps]）、軸ごとに分けるかはレビュー工程（段階 6）で決める（[IMPL-33][impl]）。モデルは設定で変えられるようにする（[IMPL-19][impl]） |
| `models.reviewComparison` | レビュー工程（段階 6）で比べるためだけに回すレビュー AI（印を書かない） | `fable` / high（[IMPL-33][impl]） | レビュー工程（段階 6）で、同じ PR を `fable` でも回して比べ、既定のモデルを決める。比較の実行は印を書かない（[IMPL-19、IMPL-33][impl]、[段階の表][stages]） |
| `models.critic` | 批評 AI | `opus` / high（[IMPL-34][impl]） | 批評 AI は、🔴 と 🟡 を別のセッションで再検証する。値は変えない（[IMPL-22、IMPL-34][impl]） |
| `models.planReview` | 計画をレビューするレビュー AI | `opus` / xhigh（`models.review` と同じ。暫定） | 計画のレビューのモデルと effort は、基本設計で決める（[IMPL-37][impl]）。同じレビュー AI の役で、計画の誤りを見つけるには PR のレビューと同じ深さが要るので、`models.review` と同じ値で始める。修正工程（段階 7）を終えるときに、人の判断と比べて見直す（[IMPL-10][impl]、[ADR-20261018-004][adr-planrev]） |
| `review.criticConfidenceThreshold` | 批評 AI が付けた確信度が、この値より低い 🟡 を 🟢 に下げる | 80（0〜100。暫定） | 批評 AI が付けた確信度がしきい値（設定）未満の 🟡 は、🟢 に下げる（[IMPL-22][impl]）。80 は、Anthropic 公式 Plugin が確信度 80 未満の指摘を捨てる線（[レビューの Skill][review-skill]）で、[ADR-20261015-005][adr-settings] の補足情報が候補に挙げた値 |

#### 検査と判定

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `checks.qualityGate` | AI が作業場所で実行する品質ゲートのコマンド | `pnpm check`（[STRUCT-2][struct]） | 作業 AI は、CI のログを受け取らず、自分の作業場所で品質ゲートを実行して確かめる（[IMPL-27][impl]）。守る規約に「品質ゲートが緑」を含む（[IMPL-4][impl]） |
| `checks.required` | CI が緑かを判定する必須チェックの一覧（チェックの名前、チェックを出した app、base 側のワークフローのパス） | 空。`check` が fail にするので、人が当てる先のリポの必須チェックを書く（business-os の例：`check (ubuntu-latest)` `check (windows-latest)` `commitlint`。[IMPL-27][impl]） | CI が緑かは、設定に列挙した必須チェックがすべて `success` かで判定する。一覧が空なら点検を fail にし、空を「すべて success」と読まない（[IMPL-27][impl]） |
| `merge.allowPaths` | 進行 Bot が `merge=ready` と判定してよいパスの許可一覧。一覧に無いパスを 1 つでも触れば `merge=human` | business-os：`docs/usage/**`、`.changeset/*.md`（[MERGE-4][merge]）。ほかのリポは空（すべて `merge=human`） | 進行 Bot が「マージしてよい」と判定してよいパスは、拒否一覧ではなく許可一覧で設定に持つ（[MERGE-4][merge]）。パスを足すのは、安全の境界を緩める変更なので ADR で決める（[ADR-20261015-005][adr-settings]） |
| `merge.policyPaths` | 方針に係るパスの一覧。1 つでも触れば判定の対象外（`merge=human`） | business-os：`docs/adr/**` `docs/design/**` `ROADMAP.md` `CLAUDE.md` `.claude/**` `dev-autopilot/**` `.claude-plugin/**` `.github/**` `plugin/hooks/**` `plugin/agents/**` `plugin/templates/**`、`check:*` の実装（`plugin/scripts/check-repo.ts` など）、`package.json` `pnpm-lock.yaml` `pnpm-workspace.yaml` `.node-version` `tsconfig.json` `vitest.config.ts` `eslint.config.*` `commitlint.config.*` `.gitleaks.toml` `.gitignore` `.markdownlint*`（要件が「許可一覧に含めない」の例として挙げた一覧。[MERGE-4][merge]。パスのほかに、`release/*` ブランチと `main` 向けの PR も対象外）。ほかのリポは空（許可一覧にパスを足すときに一緒に決める） | 方針に係るパスに触れる PR は、進行 Bot の判定の対象外（[MERGE-2][merge]、[SAFE-9][safety]）。一覧は設定に持つ（[STRUCT-2][struct]） |
| `push.blockedPaths` | diff が触れていたら push せず `needs-human` にするパスの一覧（進行 Bot・次のセッション・CI が実行するか、規則として読むパス） | `.claude/**` `CLAUDE.md` `.mcp.json` `.github/**` `dev-autopilot/**` と、当てる先のリポの規約の正典のパス（business-os は `docs/adr/**` `docs/design/**`）（[IMPL-15][impl]） | push の前に、diff がこの一覧のパスに触れていれば、push せず `needs-human` にする。一覧は設定に持つ（[IMPL-15][impl]）。パスを外すのは、安全の境界を緩める変更なので ADR で決める（[ADR-20261015-005][adr-settings]） |
| `leakCheck.samplesPath` | 漏えい検査の見本の置き場 | 未定 | 設定の項目の一覧にある（[STRUCT-2][struct]）が、要件は「見本」が何か（実行の中で読むものか、テストの前提データか）を書いていない。確かめてから決める |

#### 規約

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `conventions.language` | commit メッセージと PR の言語 | business-os：日本語（[IMPL-4][impl]）。ほかのリポは、当てる先の規約に合わせて PR で直す | 作業 AI に守らせる規約は、当てる先のリポの正典をそのまま渡す（[IMPL-4][impl]）。規約は設定に持つ（[STRUCT-2][struct]） |
| `conventions.conventionalCommits` | commit メッセージを Conventional Commits で書くか | business-os：`true`（[IMPL-4][impl]） | 作業 AI に守らせる規約（[IMPL-4][impl]） |
| `conventions.changeset` | 変更に changeset を付けるか | business-os：`true`（[IMPL-4][impl]） | 作業 AI に守らせる規約（[IMPL-4][impl]） |

#### 安全

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `safety.botAccount` | 進行 Bot が投稿に使う GitHub のアカウント名。印はこのアカウントの投稿だけを読む | `setup` が人の `gh` のログイン名を書く（最初はメンテナのアカウント。[SETUP-13][setup]） | 進行 Bot は、このアカウントが投稿し、未編集で、署名が通るコメントの印だけを読む（[IMPL-20][impl]）。アカウントは最初はメンテナの 1 つで始める（[SETUP-13][setup]） |
| `safety.network.allowedDomains` | AI のセッションの sandbox から通信してよい先 | `registry.npmjs.org`（暫定） | AI のセッションの外への通信は、npm のレジストリなどの許可した先だけにする（[SAFE-10][safety]）。先を足すのは、安全の境界を緩める変更なので ADR で決める（[ADR-20261015-005][adr-settings]） |
| `safety.env.names` | 進行 Bot が AI のセッションに渡す環境変数の名前（ほかの名前は渡さない） | `HOME` `PATH` `USER` `LOGNAME` `TERM` `LANG` `TMPDIR` `CLAUDE_CODE_TMPDIR` `XDG_CONFIG_HOME` `XDG_CACHE_HOME` `XDG_DATA_HOME` `XDG_STATE_HOME` `GIT_CONFIG_GLOBAL` `NPM_CONFIG_USERCONFIG` `GIT_CONFIG_COUNT` `GIT_CONFIG_KEY_0` `GIT_CONFIG_VALUE_0` `GIT_AUTHOR_NAME` `GIT_AUTHOR_EMAIL` `GIT_COMMITTER_NAME` `GIT_COMMITTER_EMAIL` `ENABLE_CLAUDEAI_MCP_SERVERS`。それぞれの値と理由は「[AI のセッション][design-ai]」の「環境変数」 | 子プロセスの環境変数は許可した名前だけで組み立て、`gh` の認証と `GH_TOKEN` を渡さない（[SAFE-1][safety]）。渡す名前は要件が挙げている（[SAFE-10][safety]）。名前を足すのは ADR で決める（迷ったら ADR。[ADR-20261015-005][adr-settings] の決定 1） |
| `safety.sandbox.denyRead` | AI のセッションの sandbox で読ませないパスのうち、足す分。`${home}` か `${worktree}` で始めて書き、絶対パスは書かない | 空 | 読み取りは自分のセッションのフォルダだけにし、`denyRead` に入れるパスは要件が決めている（[SAFE-10][safety]）。進行 Bot は要件が決めたパスをマシンの置き場から組み立てて必ず入れ、設定には足す分だけを持つ（設定で消せないようにするため） |
| `safety.sandbox.denyWrite` | AI のセッションの sandbox で書かせないパスのうち、足す分。`${home}` か `${worktree}` で始めて書き、絶対パスは書かない | 空 | 書き込みは、自分のセッションの worktree と一時フォルダなどに限る（[SAFE-10][safety]）。`denyRead` と同じく、設定には足す分だけを持つ |

### マシンの設定の項目

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `baseBranch` | PR を向ける base ブランチの名前（進行 Bot はこのブランチの先端からリポの設定を読む） | 無し。`setup` のマシンの準備で人が指定する | 進行 Bot は base ブランチの先端からリポの設定を読むので、base ブランチの名前はリポの設定を読む前に要る。そのためマシンの設定に置く（[ADR-20261017-007][adr-stop] の決定 4） |
| `devAutopilotPath` | 進行 Bot のプログラムの写しのフォルダ | `setup` が `<paths.workspace>/program/<版>/` を書く | `setup` は、入れた Plugin の版を置き場の中の版ごとのフォルダに写し、定期の実行は写しから進行 Bot を起こす（[STRUCT-3][struct]） |
| `paths.workspace` | 置き場（進行 Bot の clone、セッションごとのフォルダ、プログラムの写し、安全設定のファイル） | 「[マシンの中の置き場][places]」 | AI のセッションごとに origin から clone し直す。進行 Bot の clone は `controller/`、セッションのフォルダは `sessions/<セッションの名前>/` に置く（[ADR-20261013-001][adr-workspace]） |
| `paths.records` | 実行の記録と lock の置き場 | 「[マシンの中の置き場][places]」 | 実行の記録を追跡しないフォルダに残す（[FLOW-5][flow]）。記録と lock の置き場はマシンの設定に置く（[ADR-20261015-005][adr-settings]） |
| `paths.secrets` | 秘密の置き場（PAT、deploy key、印の署名の鍵） | 「[マシンの中の置き場][places]」 | PAT・印の署名の鍵・deploy key を、秘密の置き場のファイルに置く（[ADR-20261015-001][adr-read] の決定 2） |
| `paths.sandboxSettings` | AI のセッションに `--settings` で渡す安全設定のファイルの置き場 | `<paths.workspace>/sandbox-settings/` | 安全設定のファイルは AI が書けない場所に置き、起動の前にハッシュを確かめる（[SAFE-2][safety] の (e)）。進行 Bot がセッションごとに作り、終わった後にもハッシュを比べて消す（[AI のセッション][design-ai]の「セッションごとの安全設定」） |
| `safety.sandbox.toolDirs` | sandbox の `allowRead` に足す、ホームの中にある道具のフォルダ | `setup` が、`node` `pnpm` `git` の実体のうちホームの中にあるもののフォルダを書く（暫定） | sandbox はホーム（`~/`）を読ませないので、ホームの中にある道具のフォルダを `allowRead` に足す（[SAFE-10][safety]） |
| `sessionTmp.root` | セッションごとの短い一時フォルダを作る根 | 「[マシンの中の置き場][places]」 | セッションごとの短い一時フォルダを、ホーム・置き場・`/tmp/claude-<uid>/` の外の、dev-autopilot 専用の短い根の下に作る（[ADR-20261017-006][adr-tmp]、[SAFE-10][safety]） |
| `sessionTmp.idLength` | 短い一時フォルダの名前（セッションの短い ID）の文字数 | 8 文字（暫定） | 根と ID の長さの値は、設定の表で決める（[ADR-20261017-006][adr-tmp]）。8 文字にすると、`<根>/<ID>/claude-<uid>` が 44 バイト以内に収まる（uid は 19 桁まで） |
| `sessionTmp.maxBytes` | `<CLAUDE_CODE_TMPDIR>/claude-<uid>` の長さの上限。超えたら AI のセッションを起動しない | 44 バイト（[SAFE-10][safety]） | `<CLAUDE_CODE_TMPDIR>/claude-<uid>` が長いと、Claude Code は黙ってユーザーごとの一時フォルダに戻すので、進行 Bot は起動の前に長さを数え、上限を超えたら起動しない（[SAFE-10][safety]、[ADR-20261017-006][adr-tmp]）。Claude Code の振る舞いで決まる値で、Claude Code はマシンごとに入れるので、マシンの設定に置く |

## `check` が設定について確かめること

SETUP-2 の点検のうち、設定に係るものをまとめる。

- すべての項目が、リポの設定かマシンの設定のどちらか一方にだけあり、形が表のとおりであること
- リポの設定に絶対パス（`/Users/` `/home/` などで始まる値）が無いこと
- マシンの設定の置き場の実体のパスが、同期フォルダの下・`~/.claude/` の下・`/tmp/claude-<uid>/` の下に無いこと
- 「メンテナが決める」の項目と `checks.required` が空でないこと
- 定期の実行の定義が渡すマシンの設定のパスと、人が呼ぶ Skill が origin から見つけるパスが一致すること。定期の実行の間隔が `run.intervalHours` と一致すること

[adr-settings]: ../../adr/adr-20261015-005-settings-values-and-machine-settings.md
[adr-stop]: ../../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-human]: ../../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-read]: ../../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-tmp]: ../../adr/adr-20261017-006-per-session-short-tmpdir.md
[adr-usage]: ../../adr/adr-20261018-001-stop-session-and-count-usage.md
[adr-model]: ../../adr/adr-20261018-003-count-model-raise-at-stage-7.md
[adr-planrev]: ../../adr/adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[adr-workspace]: ../../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[design-change]: ../README.md#設計を変えるとき
[places]: #マシンの中の置き場
[flow]: ../../requirements/01-issue-flow.md
[flow-rules]: ../../requirements/01-issue-flow.md
[markers]: ../../requirements/04-external-text-and-markers.md
[review-skill]: ../../requirements/08-review-skill.md
[stages]: ../../requirements/06-setup-and-stages.md
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
