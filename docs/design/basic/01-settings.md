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

- 要件が初期値を決めている項目は、表に値を写さず、要件の番号を書く（[ADR-20261015-005][adr-settings] の決定 1）
- 要件が初期値を決めていない項目は、表に初期値と根拠を書き、「暫定」と添える。段階の出口の記録（FLOW-5、FLOW-13）で見直す
- 「メンテナが決める」と書いた項目は、要件にも ADR にも根拠になる値が無く、利用枠の使い方などメンテナの事情で決まる。値が空のままなら `check` は fail にし、進行 Bot を動かさない
- `setup` は、表の初期値で設定ファイルを作る。今の値の正本は設定ファイルで、表は初期値と根拠の正本
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

キーは、ファイルの中の JSON の位置を `.` でつないで書く（例：`limits.session.minutes`）。「形」は `check` が確かめる型（SETUP-2）。

### リポの設定の項目

#### リポと Project

| キー | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `repository.owner`、`repository.name` | 文字列 | `setup` が origin から書く | STRUCT-2。`check` が origin と一致することを確かめる |
| `project.number` | 数 | `setup` が使う Project の番号 | SETUP-5、SETUP-6 |
| `project.statusField.id`、`project.statusField.options`（`todo` `inProgress` `inReview` `readyToMerge` `blocked` `done` から選択肢の ID への対応表） | 文字列、対応表 | `setup` が足した欄と選択肢の ID | SETUP-5 |
| `project.agentField.id`、`project.agentField.options`（`worker` `reviewer` `human` `paused` から選択肢の ID への対応表） | 文字列、対応表 | `setup` が足した欄と選択肢の ID | SETUP-6 |

#### ラベル名と固定の見出し

| キー | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `labels.agentReady`、`labels.needsHuman`、`labels.fromReview`、`labels.planFable`、`labels.planOpus` | 文字列 | SETUP-8 | SETUP-8 |
| `labels.branchTypes` | 対応表（ラベル → ブランチの型） | 暫定：`bug` → `fix` | IMPL-2 の例 |
| `labels.defaultBranchType` | 文字列 | 暫定：`feat` | IMPL-2 の例（対応表に無いラベルの型） |
| `headings.workInstruction`、`headings.rejection`、`headings.differentJudgment`、`headings.objection` | 文字列 | 「[印と固定の見出しの一覧][markers]」 | 「[印と固定の見出しの一覧][markers]」 |

#### 進行 Bot の実行

| キー | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `run.dryRun` | 真偽 | 「[運用の規則][flow-rules]」と受け入れ工程（段階 4） | 「[運用の規則][flow-rules]」、[ADR-20261017-007][adr-stop]。`false` にする（書き始める）のは PR で行う |
| `run.stage` | 数（1〜9） | 暫定：4 | 「[Issue の状態の遷移][flow]」の段階による違い。Issue の流れは受け入れ工程（段階 4）から動く（「[段階][stages]」） |
| `run.intervalHours` | 数 | FLOW-12 | FLOW-11。OS ごとの登録と二重になるので、`check` がずれを確かめる（STRUCT-2） |
| `run.maxMinutes` | 数 | 暫定：180 | FLOW-4。重い工程 1 つ（1 セッションの上限は IMPL-13）の後にも、軽い工程とほかの Issue の重い工程を進める余地を持たせる。[ADR-20261015-005][adr-settings] の補足情報の候補の値 |
| `run.stopSignalPollSeconds` | 数 | 暫定：60 | 「[運用の規則][flow-rules]」（AI のセッションが動いている間に合図を読む間隔）。止めると決めてから AI のセッションが止まるまでを 1 分ほどに抑える。1 セッションの上限の間に読む回数は数十回で、GitHub の API の上限（1 時間に 5,000 回）に比べて少ない |

#### 上限

| キー | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `limits.startsPerRun` | 数 | FLOW-3 | FLOW-3 |
| `limits.wip` | 数 | FLOW-3 | FLOW-3 |
| `limits.intakePerRun` | 数 | 暫定：5 | TEXT-12（初回の実行で未処理の Issue すべてに走らない。外の人が Issue を大量に立てたときの歯止め）。[ADR-20261015-005][adr-settings] の補足情報の候補の値 |
| `limits.roundTrips` | 数 | IMPL-24 | IMPL-24 |
| `limits.session.minutes`、`limits.session.turns` | 数 | IMPL-13 | IMPL-13 |
| `limits.session.costUsd` | 数 | メンテナが決める | IMPL-13、IMPL-14（`--max-budget-usd`。途中で止めた回はこの値を費用とみなす） |
| `limits.issueCostUsd` | 数 | メンテナが決める | IMPL-14（1 Issue の累計。往復を含む） |
| `limits.runCostUsd` | 数 | メンテナが決める | IMPL-14、SETUP-12 |
| `limits.dailyCostUsd` | 数 | メンテナが決める | IMPL-14、SETUP-12 |
| `limits.stopGraceSeconds` | 数 | 暫定：10 | IMPL-14（`SIGTERM` から `SIGKILL` までの猶予）。試験では `SIGTERM` の後 1 秒ほどで終わった（[ADR-20261018-001][adr-usage]）ので、10 倍の余裕を見る |
| `limits.retry.count` | 数 | IMPL-14 | IMPL-14 |
| `limits.retry.intervalRuns` | 数 | 暫定：1（次の定期の実行で再試行する） | IMPL-14。実行の中で待たない（lock を長く持たない。FLOW-4）。[ADR-20261015-005][adr-settings] の補足情報の候補（再試行は次の実行） |
| `limits.remindDays` | 数 | MERGE-3 | MERGE-3 |
| `limits.ciStaleHours` | 数 | 暫定：48 | IMPL-27（`cancelled` などのまま変わらないときに待つ時間）。1 日 1 回の間隔（FLOW-12）で、人が再実行する機会を、間隔 1 回分より長く取る |

#### モデル

役ごとのモデルと effort は、`model` と `effort` の 2 つの欄を持つ（例：`models.plan.model`、`models.plan.effort`）。

| キー | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `models.intake` | モデルと effort | IMPL-29 | 「[モデルの流れ][models]」 |
| `models.intakeAfterRefusal` | モデルと effort | IMPL-29（拒否の次回） | IMPL-14、IMPL-29 |
| `models.plan`、`models.planHard` | モデルと effort | IMPL-31（既定と、`fable` にしたとき） | IMPL-10、IMPL-31 |
| `models.work` | モデルと effort | IMPL-32 | IMPL-25、IMPL-32 |
| `models.workByType` | 対応表（Issue の種類 → モデルと effort） | 空（種類の分け方は UNDECIDED-52 で決める） | IMPL-25、[ADR-20261018-003][adr-model] |
| `models.workRaised` | モデルと effort | IMPL-35 | IMPL-25、IMPL-35 |
| `models.review` | モデルと effort | IMPL-33 | IMPL-19、IMPL-33 |
| `models.reviewComparison` | モデルと effort | IMPL-33（レビュー工程（段階 6）で比べる側） | IMPL-19（比較の実行は印を書かない） |
| `models.critic` | モデルと effort | IMPL-34 | IMPL-22、IMPL-34 |
| `models.planReview` | モデルと effort | 暫定：`models.review` と同じ値 | IMPL-37（基本設計で決める）。同じレビュー AI の役で、計画の誤りを見つけるには PR のレビューと同じ深さが要る。修正工程（段階 7）を終えるときに、人の判断と比べて見直す（IMPL-10、[ADR-20261018-004][adr-planrev]） |
| `review.criticConfidenceThreshold` | 数（0〜100） | 暫定：80 | IMPL-22。Anthropic 公式 Plugin が確信度 80 未満の指摘を捨てる線（「[レビューの Skill][review-skill]」）。[ADR-20261015-005][adr-settings] の補足情報の候補の値 |

#### 検査と判定

| キー | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `checks.qualityGate` | 文字列 | STRUCT-2 | IMPL-4、STRUCT-2 |
| `checks.required` | 一覧（チェックの名前・app・ワークフローのパス） | 空。`check` が fail にするので、人が当てる先のリポの必須チェックを書く | IMPL-27（空を「すべて success」と読まない） |
| `merge.allowPaths` | 一覧（パスの glob） | business-os は MERGE-4。ほかのリポは空（すべて `merge=human`） | MERGE-4。パスを足すのは ADR |
| `merge.policyPaths` | 一覧（パスの glob） | business-os は MERGE-4。ほかのリポは空（許可一覧にパスを足すときに一緒に決める） | MERGE-2、MERGE-4 |
| `push.blockedPaths` | 一覧（パスの glob） | IMPL-15（規約の正典のパスは当てる先のリポごと。business-os は IMPL-15 の例） | IMPL-15。パスを外すのは ADR |
| `leakCheck.samplesPath` | — | 未定 | STRUCT-2 の「漏えい検査の見本の置き場」。要件が何を指すか（実行の中で読むものか、テストの前提データか）を確かめてから決める |

#### 規約

| キー | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `conventions.language`、`conventions.conventionalCommits`、`conventions.changeset` | 文字列、真偽 | business-os は IMPL-4。ほかのリポは、当てる先の規約に合わせて PR で直す | IMPL-4、STRUCT-2 |

#### 安全

| キー | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `safety.botAccount` | 文字列 | `setup` が人の `gh` のログイン名を書く（最初はメンテナのアカウント） | IMPL-20、SETUP-13 |
| `safety.network.allowedDomains` | 一覧 | 暫定：`registry.npmjs.org` | SAFE-10（外への通信は npm のレジストリなどの許可した先だけ）。先を足すのは ADR |
| `safety.env.names` | 一覧 | SAFE-10 が挙げる名前。起動に要るほかの名前は、AI のセッションの文書で決める | SAFE-1、SAFE-10。名前を足すのは ADR（迷ったら ADR。[ADR-20261015-005][adr-settings] の決定 1） |
| `safety.sandbox.denyRead`、`safety.sandbox.denyWrite` | 一覧 | 空 | SAFE-10。SAFE-10 が決めたパスは、進行 Bot がマシンの置き場から組み立てて必ず入れる。設定は足す分だけを持つ |

### マシンの設定の項目

| キー | 形 | 初期値 | 根拠 |
| --- | --- | --- | --- |
| `baseBranch` | 文字列 | 無し。`setup` のマシンの準備で人が指定する | [ADR-20261017-007][adr-stop] の決定 4（リポの設定を読む前に要る） |
| `devAutopilotPath` | 絶対パス | `setup` が `<paths.workspace>/program/<版>/` を書く | STRUCT-3 |
| `paths.workspace`、`paths.records`、`paths.secrets` | 絶対パス | 「[マシンの中の置き場][places]」 | [ADR-20261013-001][adr-workspace]、FLOW-5、[ADR-20261015-001][adr-read] |
| `paths.sandboxSettings` | 絶対パス | `<paths.workspace>/sandbox-settings/` | SAFE-10（安全設定のファイルは AI が書けない場所に置く）。ファイルの作り方は AI のセッションの文書で決める（UNDECIDED-40） |
| `safety.sandbox.toolDirs` | 一覧（絶対パス） | 暫定：`setup` が、`node` `pnpm` `git` の実体のうちホームの中にあるもののフォルダを書く | SAFE-10（`allowRead` に足すホームの中の道具のフォルダ） |
| `sessionTmp.root` | 絶対パス | 「[マシンの中の置き場][places]」 | [ADR-20261017-006][adr-tmp] |
| `sessionTmp.idLength` | 数 | 暫定：8 | [ADR-20261017-006][adr-tmp]（根と ID の長さは設定の表で決める）。`<根>/<ID>/claude-<uid>` を 44 バイト以内に収める |
| `sessionTmp.maxBytes` | 数 | SAFE-10 | SAFE-10、[ADR-20261017-006][adr-tmp]。Claude Code の振る舞いで決まる値で、Claude Code はマシンごとに入れるので、マシンの設定に置く |

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
[models]: ../../requirements/02-implement-review-fix.md
[review-skill]: ../../requirements/08-review-skill.md
[stages]: ../../requirements/06-setup-and-stages.md
