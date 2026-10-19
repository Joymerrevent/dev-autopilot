# 導入と点検

人が呼ぶ `setup` と `check`、進行 Bot が毎回の実行の最初に回す点検の、順番と仕組みを決める。
何を作り何を確かめるかは要件の [SETUP-1、SETUP-2][setup] が正本で、基本設計は順番・置き場・確かめ方を決める。
定期の実行の登録と、点検の項目と仕組みは、この文書に後で足す。

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
| 7 | 定期の実行と同じジョブで点検だけを起こし、結果を示す（仕組みは、この文書に後で足す） | fail があれば示す |
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
| 定期の実行の登録 | OS ごと | この文書に後で足す |

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
