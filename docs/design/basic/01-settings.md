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
| `repository.owner`、`repository.name` | 当てる先のリポの owner と名前 | `setup` が origin から書く | [STRUCT-2][struct]。`check` が origin と一致することを確かめる |
| `project.number` | Issue を並べる GitHub の Project の番号 | `setup` が使う Project の番号を書く | [SETUP-5、SETUP-6][setup] |
| `project.statusField.id`、`project.statusField.options` | Project の Status の欄の ID と、選択肢（`todo` `inProgress` `inReview` `readyToMerge` `blocked` `done`）ごとの ID | `setup` が足した欄と選択肢の ID を書く | [SETUP-5][setup] |
| `project.agentField.id`、`project.agentField.options` | Project の Agent 欄の ID と、選択肢（`worker` `reviewer` `human` `paused`）ごとの ID | `setup` が足した欄と選択肢の ID を書く | [SETUP-6][setup] |

#### ラベル名と固定の見出し

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `labels.agentReady` | 人が処理対象にした Issue に付けるラベル | `agent-ready`（[SETUP-8][setup]） | [SETUP-8][setup] |
| `labels.needsHuman` | 人の判断が要る Issue に付けるラベル | `needs-human`（[SETUP-8][setup]） | [SETUP-8][setup] |
| `labels.fromReview` | 🟢 の指摘から起票した Issue に付けるラベル | `from-review`（[SETUP-8][setup]） | [SETUP-8][setup] |
| `labels.planFable`、`labels.planOpus` | 人が計画のモデルを指定するラベル | `plan:fable`、`plan:opus`（[SETUP-8][setup]） | [SETUP-8][setup]、[IMPL-30][impl] |
| `labels.branchTypes` | Issue のラベルから作業ブランチの型を決める対応表 | `bug` → `fix`（暫定） | [IMPL-2][impl] の例 |
| `labels.defaultBranchType` | 対応表に無いラベルの Issue の、作業ブランチの型 | `feat`（暫定） | [IMPL-2][impl] の例 |
| `headings.workInstruction` | コラボレータが作業指示を書くコメントの見出し | `## 作業指示`（[印と固定の見出しの一覧][markers]） | [TEXT-1、TEXT-12][markers] |
| `headings.rejection` | 人がレビューの指摘を却下するコメントの見出し（`<n>` はラウンド） | `## 却下 R<n>`（[印と固定の見出しの一覧][markers]） | [FLOW-14][flow] |
| `headings.differentJudgment` | 人が `merge=ready` の PR をマージしない理由を書くコメントの見出し | `## 判定と違う判断`（[印と固定の見出しの一覧][markers]） | [MERGE-3][merge] |
| `headings.objection` | 人が選定・待機・下書き・計画に異議を書くコメントの見出し | `## 異議`（[印と固定の見出しの一覧][markers]） | [段階の表][stages]の出口 |

#### 進行 Bot の実行

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `run.dryRun` | GitHub に書かずに試すか（`true` なら、書くつもりだった内容を手元の記録にだけ残す） | `true`（受け入れ工程（段階 4）の最初。[運用の規則][flow]） | [運用の規則][flow]、[ADR-20261017-007][adr-stop]。`false` にする（書き始める）のは PR で行う |
| `run.stage` | 今の段階の番号（段階ごとの振る舞いを切り替える） | `4`（受け入れ工程。暫定） | [Issue の状態の遷移][flow]の段階による違い。Issue の流れは受け入れ工程（段階 4）から動く（[段階の表][stages]） |
| `run.intervalHours` | 定期の実行の間隔 | 24 時間（1 日 1 回。[FLOW-12][flow]） | [FLOW-11][flow]。OS ごとの登録と二重になるので、`check` がずれを確かめる（[STRUCT-2][struct]） |
| `run.maxMinutes` | 1 回の実行の所要時間の上限。超えたら残りの Issue は次の実行に回す | 180 分（暫定） | [FLOW-4][flow]。重い工程 1 つ（1 セッションは最大 45 分）の後にも、軽い工程とほかの Issue の重い工程を進める余地を持たせる。[ADR-20261015-005][adr-settings] の補足情報の候補の値 |
| `run.stopSignalPollSeconds` | AI のセッションが動いている間に、全体を止める合図を読む間隔 | 60 秒（暫定） | [運用の規則][flow]（間隔は基本設計で決める）。止めると決めてから AI のセッションが止まるまでを 1 分ほどに抑える。1 セッションの間に読む回数は数十回で、GitHub の API の上限（1 時間に 5,000 回）に比べて少ない |

#### 上限

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `limits.startsPerRun` | 1 回の実行で新しく着手する Issue の数の上限 | 1 件（[FLOW-3][flow]） | [FLOW-3][flow] |
| `limits.wip` | 進行中の Issue（`In Progress` から `Done` の前まで）の数の上限 | 1 件（マージ判定工程（段階 8）まで。[FLOW-3][flow]） | [FLOW-3][flow] |
| `limits.intakePerRun` | 1 回の実行で受け入れる Issue の数の上限 | 5 件（暫定） | [TEXT-12][text]（初回の実行で未処理の Issue すべてに走らない。外の人が Issue を大量に立てたときの歯止め）。[ADR-20261015-005][adr-settings] の補足情報の候補の値 |
| `limits.roundTrips` | レビューと修正の往復の回数の上限。超えたら `needs-human` | 3 回（[IMPL-24][impl]） | [IMPL-24][impl] |
| `limits.session.minutes` | AI のセッション 1 回の時間の上限。超えたら進行 Bot が止める | 45 分（[IMPL-13][impl]） | [IMPL-13][impl] |
| `limits.session.turns` | AI のセッション 1 回のターンの上限（`--max-turns`） | 200 ターン（[IMPL-13][impl]） | [IMPL-13][impl] |
| `limits.session.costUsd` | AI のセッション 1 回の費用の上限（`--max-budget-usd`）。途中で止めた回は、この値をその回の費用とみなす | メンテナが決める（USD） | [IMPL-13、IMPL-14][impl] |
| `limits.issueCostUsd` | 1 Issue の累計の費用の上限（往復を含む）。超えたら `needs-human` | メンテナが決める（USD） | [IMPL-14][impl] |
| `limits.runCostUsd` | 1 回の実行の費用の上限 | メンテナが決める（USD） | [IMPL-14][impl]、[SETUP-12][setup] |
| `limits.dailyCostUsd` | 1 日の費用の上限 | メンテナが決める（USD） | [IMPL-14][impl]、[SETUP-12][setup] |
| `limits.stopGraceSeconds` | AI のセッションを途中で止めるとき、`SIGTERM` を送ってから `SIGKILL` を送るまでの猶予 | 10 秒（暫定） | [IMPL-14][impl]。試験では `SIGTERM` の後 1 秒ほどで終わった（[ADR-20261018-001][adr-usage]）ので、10 倍の余裕を見る |
| `limits.retry.count` | 同じ Issue で実行の失敗が続いたとき、`needs-human` にするまでの回数 | 3 回（[IMPL-14][impl]） | [IMPL-14][impl] |
| `limits.retry.intervalRuns` | 実行の失敗の後、何回目の定期の実行で再試行するか | 1（次の定期の実行。暫定） | [IMPL-14][impl]。実行の中で待たない（lock を長く持たない。[FLOW-4][flow]）。[ADR-20261015-005][adr-settings] の補足情報の候補（再試行は次の実行） |
| `limits.remindDays` | `merge=ready` の PR が、マージも「判定と違う判断」のコメントも無いまま経ったとき、進行 Bot が 1 回だけ催促するまでの日数 | 7 日（[MERGE-3][merge]） | [MERGE-3][merge] |
| `limits.ciStaleHours` | CI の結果が `cancelled` `skipped` `neutral` `stale` のまま変わらないとき、`needs-human` にするまで待つ時間 | 48 時間（暫定） | [IMPL-27][impl]。1 日 1 回の間隔（[FLOW-12][flow]）で、人が再実行する機会を、間隔 1 回分より長く取る |

#### モデル

役ごとのモデルと effort は、`model` と `effort` の 2 つの欄を持つ（例：`models.plan.model`、`models.plan.effort`）。モデルは別名で書く（`haiku` だけは別名が旧版に解けるので完全な ID。[モデルの流れ][impl]）。

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `models.intake` | 受付 AI（受け入れと計画の難しさの判定） | `claude-haiku-5-5` / medium（[IMPL-29][impl]） | [モデルの流れ][impl] |
| `models.intakeAfterRefusal` | 受付 AI が拒否（`stop_reason: refusal`）で止まった Issue の、次の回の受付 AI | `sonnet`（[IMPL-29][impl]） / medium（effort は暫定。`models.intake` と同じ） | [IMPL-14、IMPL-29][impl] |
| `models.plan` | 計画 AI（既定） | `opus` / xhigh（[IMPL-31][impl]） | [IMPL-10、IMPL-31][impl] |
| `models.planHard` | 計画 AI（受け入れの判定かラベルで `fable` にしたとき、計画 AI が自己申告して計画し直すとき） | `fable` / high（[IMPL-31][impl]） | [IMPL-31][impl]、[ADR-20261018-002][adr-redo] |
| `models.work` | 作業 AI の最初の実装 | `opus` / high（[IMPL-32][impl]） | [IMPL-25、IMPL-32][impl] |
| `models.workByType` | Issue の種類ごとの、作業 AI の最初の実装 | 空（種類の分け方は UNDECIDED-52 で決める） | [IMPL-25][impl]、[ADR-20261018-003][adr-model] |
| `models.workRaised` | 最初の実装が `sonnet` だった Issue で、修正のときに上げる先 | `opus` / high（[IMPL-35][impl]） | [IMPL-25、IMPL-35][impl] |
| `models.review` | レビュー AI（3 つの軸とも同じ値） | `opus` / xhigh（[IMPL-33][impl]） | [IMPL-19、IMPL-33][impl] |
| `models.reviewComparison` | レビュー工程（段階 6）で比べるためだけに回すレビュー AI（印を書かない） | `fable` / high（[IMPL-33][impl]） | [IMPL-19][impl] |
| `models.critic` | 批評 AI | `opus` / high（[IMPL-34][impl]） | [IMPL-22、IMPL-34][impl] |
| `models.planReview` | 計画をレビューするレビュー AI | `opus` / xhigh（`models.review` と同じ。暫定） | [IMPL-37][impl]（基本設計で決める）。同じレビュー AI の役で、計画の誤りを見つけるには PR のレビューと同じ深さが要る。修正工程（段階 7）を終えるときに、人の判断と比べて見直す（[IMPL-10][impl]、[ADR-20261018-004][adr-planrev]） |
| `review.criticConfidenceThreshold` | 批評 AI が付けた確信度が、この値より低い 🟡 を 🟢 に下げる | 80（0〜100。暫定） | [IMPL-22][impl]。Anthropic 公式 Plugin が確信度 80 未満の指摘を捨てる線（[レビューの Skill][review-skill]）。[ADR-20261015-005][adr-settings] の補足情報の候補の値 |

#### 検査と判定

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `checks.qualityGate` | AI が作業場所で実行する品質ゲートのコマンド | `pnpm check`（[STRUCT-2][struct]） | [IMPL-4][impl]、[STRUCT-2][struct] |
| `checks.required` | CI が緑かを判定する必須チェックの一覧（チェックの名前、チェックを出した app、base 側のワークフローのパス） | 空。`check` が fail にするので、人が当てる先のリポの必須チェックを書く（business-os の例：`check (ubuntu-latest)` `check (windows-latest)` `commitlint`。[IMPL-27][impl]） | [IMPL-27][impl]（空を「すべて success」と読まない） |
| `merge.allowPaths` | 進行 Bot が `merge=ready` と判定してよいパスの許可一覧。一覧に無いパスを 1 つでも触れば `merge=human` | business-os：`docs/usage/**`、`.changeset/*.md`（[MERGE-4][merge]）。ほかのリポは空（すべて `merge=human`） | [MERGE-4][merge]。パスを足すのは ADR |
| `merge.policyPaths` | 方針に係るパスの一覧。1 つでも触れば判定の対象外（`merge=human`） | business-os：`docs/adr/**` `docs/design/**` `ROADMAP.md` `CLAUDE.md` `.claude/**` `dev-autopilot/**` `.claude-plugin/**` `.github/**` `plugin/hooks/**` `plugin/agents/**` `plugin/templates/**`、`check:*` の実装（`plugin/scripts/check-repo.ts` など）、`package.json` `pnpm-lock.yaml` `pnpm-workspace.yaml` `.node-version` `tsconfig.json` `vitest.config.ts` `eslint.config.*` `commitlint.config.*` `.gitleaks.toml` `.gitignore` `.markdownlint*`（[MERGE-4][merge] の「許可一覧に含めない」の例。パスのほかに、`release/*` ブランチと `main` 向けの PR も対象外）。ほかのリポは空（許可一覧にパスを足すときに一緒に決める） | [MERGE-2、MERGE-4][merge] |
| `push.blockedPaths` | diff が触れていたら push せず `needs-human` にするパスの一覧（進行 Bot・次のセッション・CI が実行するか、規則として読むパス） | `.claude/**` `CLAUDE.md` `.mcp.json` `.github/**` `dev-autopilot/**` と、当てる先のリポの規約の正典のパス（business-os は `docs/adr/**` `docs/design/**`）（[IMPL-15][impl]） | [IMPL-15][impl]。パスを外すのは ADR |
| `leakCheck.samplesPath` | 漏えい検査の見本の置き場 | 未定 | [STRUCT-2][struct] の一覧にある。要件が何を指すか（実行の中で読むものか、テストの前提データか）を確かめてから決める |

#### 規約

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `conventions.language` | commit メッセージと PR の言語 | business-os：日本語（[IMPL-4][impl]）。ほかのリポは、当てる先の規約に合わせて PR で直す | [IMPL-4][impl]、[STRUCT-2][struct] |
| `conventions.conventionalCommits` | commit メッセージを Conventional Commits で書くか | business-os：`true`（[IMPL-4][impl]） | [IMPL-4][impl] |
| `conventions.changeset` | 変更に changeset を付けるか | business-os：`true`（[IMPL-4][impl]） | [IMPL-4][impl] |

#### 安全

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `safety.botAccount` | 進行 Bot が投稿に使う GitHub のアカウント名。印はこのアカウントの投稿だけを読む | `setup` が人の `gh` のログイン名を書く（最初はメンテナのアカウント。[SETUP-13][setup]） | [IMPL-20][impl]、[SETUP-13][setup] |
| `safety.network.allowedDomains` | AI のセッションの sandbox から通信してよい先 | `registry.npmjs.org`（暫定） | [SAFE-10][safety]（外への通信は npm のレジストリなどの許可した先だけ）。先を足すのは ADR |
| `safety.env.names` | 進行 Bot が AI のセッションに渡す環境変数の名前（ほかの名前は渡さない） | `GIT_CONFIG_GLOBAL` `NPM_CONFIG_USERCONFIG` `XDG_CONFIG_HOME` `XDG_CACHE_HOME` `XDG_DATA_HOME` `XDG_STATE_HOME` `TMPDIR` `CLAUDE_CODE_TMPDIR` `ENABLE_CLAUDEAI_MCP_SERVERS`（[SAFE-10][safety]）。commit の作者を渡す名前と、起動に要るほかの名前は、AI のセッションの文書で決める | [SAFE-1、SAFE-10][safety]。名前を足すのは ADR（迷ったら ADR。[ADR-20261015-005][adr-settings] の決定 1） |
| `safety.sandbox.denyRead` | AI のセッションの sandbox で読ませないパスのうち、足す分 | 空 | [SAFE-10][safety]。SAFE-10 が決めたパスは、進行 Bot がマシンの置き場から組み立てて必ず入れる。設定は足す分だけを持つ |
| `safety.sandbox.denyWrite` | AI のセッションの sandbox で書かせないパスのうち、足す分 | 空 | [SAFE-10][safety]。`denyRead` と同じ |

### マシンの設定の項目

| キー | 中身 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `baseBranch` | PR を向ける base ブランチの名前（進行 Bot はこのブランチの先端からリポの設定を読む） | 無し。`setup` のマシンの準備で人が指定する | [ADR-20261017-007][adr-stop] の決定 4（リポの設定を読む前に要る） |
| `devAutopilotPath` | 進行 Bot のプログラムの写しのフォルダ | `setup` が `<paths.workspace>/program/<版>/` を書く | [STRUCT-3][struct] |
| `paths.workspace` | 置き場（進行 Bot の clone、セッションごとのフォルダ、プログラムの写し、安全設定のファイル） | 「[マシンの中の置き場][places]」 | [ADR-20261013-001][adr-workspace] |
| `paths.records` | 実行の記録と lock の置き場 | 「[マシンの中の置き場][places]」 | [FLOW-5][flow] |
| `paths.secrets` | 秘密の置き場（PAT、deploy key、印の署名の鍵） | 「[マシンの中の置き場][places]」 | [ADR-20261015-001][adr-read] |
| `paths.sandboxSettings` | AI のセッションに `--settings` で渡す安全設定のファイルの置き場 | `<paths.workspace>/sandbox-settings/` | [SAFE-10][safety]（安全設定のファイルは AI が書けない場所に置く）。ファイルの作り方は AI のセッションの文書で決める（UNDECIDED-40） |
| `safety.sandbox.toolDirs` | sandbox の `allowRead` に足す、ホームの中にある道具のフォルダ | `setup` が、`node` `pnpm` `git` の実体のうちホームの中にあるもののフォルダを書く（暫定） | [SAFE-10][safety] |
| `sessionTmp.root` | セッションごとの短い一時フォルダを作る根 | 「[マシンの中の置き場][places]」 | [ADR-20261017-006][adr-tmp] |
| `sessionTmp.idLength` | 短い一時フォルダの名前（セッションの短い ID）の文字数 | 8 文字（暫定） | [ADR-20261017-006][adr-tmp]（根と ID の長さは設定の表で決める）。`<根>/<ID>/claude-<uid>` を 44 バイト以内に収める |
| `sessionTmp.maxBytes` | `<CLAUDE_CODE_TMPDIR>/claude-<uid>` の長さの上限。超えたら AI のセッションを起動しない | 44 バイト（[SAFE-10][safety]） | [SAFE-10][safety]、[ADR-20261017-006][adr-tmp]。Claude Code の振る舞いで決まる値で、Claude Code はマシンごとに入れるので、マシンの設定に置く |

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
