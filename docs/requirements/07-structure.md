# 分離を見越した構成

dev-autopilot は、独立したリポ（Joymerrevent/dev-autopilot）で、ほかのリポにも当てられる Plugin として作る（[ADR-20261016-003][adr-move]）。
Plugin を入れたプロジェクトで、`setup` を実行すれば、進行 Bot を含めてすべて動く形にする（[ADR-20261017-003][adr-plugin]）。
ほかのリポに当てることと、ほかの人に配ることの時期は、「分離の時期の目安」のとおりマージ判定工程（段階 8）の後に決める。

## 構成

```text
（リポの root）
  plugin/                      配布物。Plugin として読み込む範囲（marketplace の `source` はここを向く。`setup` がマシンに写した写しを、進行 Bot が `--plugin-dir` で指す）
    .claude-plugin/plugin.json 名札
    skills/
      dev-autopilot/SKILL.md   人が呼ぶ入口（/dev-autopilot：状態の表示、手動の 1 回実行）
      setup/SKILL.md           導入（SETUP-1。足りない基盤を作り、Plugin の版をマシンに写し、定期の実行を登録する。冪等）
      check/SKILL.md           点検の入口（`plugin/src/check.ts` を呼ぶ包み。判定のロジックは持たない）
      review/SKILL.md          レビュー AI の手順（change-review v2 の正本。08-review-skill.md）
      review/scripts/per-commit-gates.ts  コミットごとのゲートの検査（TypeScript。単体導入でも Skill と一緒に写る）
      intake/SKILL.md          Issue の受け入れの手順（検疫、作業指示の下書き、依存とマイルストーンの候補）。人が `/verify-issue` などから呼ぶ入口。進行 Bot は agents/intake.md を起動する。人の操作は agent-ready だけ
    agents/
      planner.md               計画 AI（読むだけ。計画を出力契約で返す）
      worker.md                作業 AI（モデルは設定で）
      reviewer.md              レビュー AI（進行 Bot がレビューの軸ごとに別のセッションで起こす。Agent ツールは持たない。ADR-20261017-002）
      critic.md                批評 AI（レビューの指摘の再検証。進行 Bot が別に起こす独立セッション。IMPL-22）
      intake.md                受付 AI（道具なし。作業指示の下書きと計画の難しさの判定を出力契約で返す。検疫で見つかった Issue では起動しない。00-overview.md の「登場するもの」、TEXT-12）
    src/                       進行 Bot（TypeScript、Node 24 が直接実行。ビルドしない。bash を置かない）。Plugin に同封し、マシンの写しから定期の実行が動かす（ADR-20261017-003）
      main.ts                  入口
      select.ts                例：Issue の選定
      marker.ts                印の形の正本（1 定数）
      check.ts                 点検の正本（進行 Bot が毎回の最初に回す）
  test/                        テスト。Plugin に同封しないように `plugin/` の外に置く
    unit/                      単体テスト。`plugin/src/` と同じ形の路で置く（`plugin/src/select.ts` → `test/unit/select.test.ts`。STRUCT-4）
                               結合テスト（偽の gh を差し替えて進行 Bot を端から端まで回す）も test/ に置く。分け方は基本設計で決める
  fixtures/                    結合テストの前提データ（偽の gh の応答、使い捨てリポの雛形、Issue と PR の本文の例）
  docs/                        要件（requirements/）、設計判断（adr/）、構造仕様（design/）、図（diagrams/。要件の写し）
  README.md                    使い方。業務の固有名詞を書かない
  package.json                 検査のスクリプトと依存（STRUCT-4）
  .markdownlint-cli2.jsonc     Markdown の検査の設定（参照スタイルのリンク。STRUCT-5）
```

## 守ること

- STRUCT-1 **business-os の中身を import しない。** `plugin/hooks/lib` や `plugin/scripts/lib` の関数が要るなら、dev-autopilot 側に持つ。逆に business-os が dev-autopilot を import することもしない
- STRUCT-2 **設定の値はコードに書かず、リポの設定とマシンの設定の 2 つに出す。** リポの設定は business-os 側（例：`.claude/dev-autopilot.json`。commit する）で、マシンに依らない値を持つ。マシンの設定は commit せず、リポの外の同期されないフォルダに置き、下の一覧で「マシン」と書いた項目を持つ（[ADR-20261015-005][adr-settings]）。
  マシンの設定は、OS ごとの決まった場所に、当てる先のリポ（owner/repo）ごとに置く（例：macOS は `~/Library/Application Support/dev-autopilot/<owner>/<repo>/`、Linux は `$XDG_CONFIG_HOME/dev-autopilot/<owner>/<repo>/`。場所の値は構造仕様の表で決める）。
  人が呼ぶ Skill は、開いているプロジェクトの origin から owner/repo を引いて場所を決め、定期の実行の定義は同じ場所のパスを進行 Bot に渡す（`check` が一致を確かめる。SETUP-2。[ADR-20261017-004][adr-human] の決定 3）。
  項目の一覧は次のとおり（`check` はすべての項目の存在と型を確かめる。SETUP-2）。項目ごとの置き場・初期値・根拠の表は、構造仕様（`docs/design/`）に置く。
  要件が初期値を決めていない項目は、構造仕様の表が初期値と根拠の正本で、今の値の正本は設定ファイル。値は設定ファイルの PR で変える。安全の境界を緩める変更（AI の道具・読み書きの範囲・通信先を広げる、MERGE-4 の許可一覧にパスを足す、IMPL-15 の一覧からパスを外す、上限を外す）は、値の変更であっても ADR で決める（[ADR-20261015-005][adr-settings]）
  - リポと Project：owner / repo、base ブランチ（マシン。リポの設定を読む前に要るため。[ADR-20261017-007][adr-stop]）、Project の番号、Status と Agent 欄の ID と選択肢の ID（SETUP-5、SETUP-6）
  - ラベル名：`agent-ready` `needs-human` `from-review` `plan:fable` `plan:opus`（SETUP-8）。ブランチの型のラベル対応表（IMPL-2）
  - 固定の見出しの文字列：`## 作業指示` `## 却下 R<n>` `## 判定と違う判断` `## 異議`（「[印と固定の見出しの一覧][f04]」）
  - 進行 Bot の実行：`dryRun`（全体の止める合図は設定の項目でなく、当てる先のリポの変数に置く。「[運用の規則][f01]」、[ADR-20261017-007][adr-stop]）、`stage`（段階ごとの振る舞い。「[Issue の状態の遷移][f01]」）、`devAutopilotPath`（進行 Bot のプログラムの写しの絶対パス。STRUCT-3。マシン）、worktree の置き場（マシン）、記録と lock の置き場（マシン）、定期の実行の間隔（OS ごとの登録（macOS は launchd の plist）と二重になるので `check` がずれを検出する）
  - 上限：1 回の実行で着手する件数（FLOW-3）、WIP の上限（FLOW-3）、1 回の実行の所要時間（FLOW-4）、受け入れの件数（TEXT-12）、往復の回数（IMPL-24）、セッションの時間・ターン・費用（IMPL-13）、1 Issue の累計の利用枠と 1 日の利用枠（費用で数える。IMPL-14、SETUP-12）、AI のセッションを途中で止めるときの `SIGTERM` から `SIGKILL` までの猶予（IMPL-14。[ADR-20261018-001][adr-usage]）、実行の失敗の再試行の回数と間隔（IMPL-14）、催促までの日数（MERGE-3）、CI の結果が取り消しなどのまま変わらないときに待つ時間（IMPL-27）
  - モデル：役ごとのモデルと effort（「[モデルの流れ][f02]」）、IMPL-25 で上げる先、受付 AI の拒否後のモデル、批評 AI の確信度のしきい値（IMPL-22）。Agent ツールはどの役にも与えないので、設定の項目に持たない（[ADR-20261017-002][adr-review]）
  - 検査と判定：品質ゲートのコマンド（`pnpm check`。AI のセッションの中で使う）、必須チェックの名前と app とワークフローのパス（IMPL-27）、マージ判定の許可一覧（MERGE-4）、方針に係るパスの一覧（MERGE-4）、push しないパスの一覧（進行 Bot・次のセッション・CI が実行するか、規則として読むパス。規約の正典のパスを含む。IMPL-15）、漏えい検査の見本の置き場
  - 規約：コミットと PR の規約（言語、Conventional Commits、changeset の要否）
  - 安全：進行 Bot のアカウント名（IMPL-20）、sandbox の `--settings` の置き場（マシン）と `denyRead` `allowRead` `denyWrite` の一覧（`allowRead` に足すホームの中の道具のフォルダはマシン。「[隔離と鍵の分け方][f05]」）、外への通信の許可先、子プロセスに渡す環境変数の名前の一覧（SAFE-10 の git と pnpm のための環境変数と、`CLAUDE_CODE_TMPDIR`、claude.ai のコネクタを止める `ENABLE_CLAUDEAI_MCP_SERVERS`（SAFE-1。[ADR-20261017-005][adr-mcp]）を含む。「[安全側に倒す設計][f05]」）、
    セッションごとの短い一時フォルダの根（マシン）と、`<CLAUDE_CODE_TMPDIR>/claude-<uid>` の長さの上限（初期値 44。リポの設定かマシンの設定かは構造仕様の表で決める。SAFE-10。[ADR-20261017-006][adr-tmp]）、秘密の置き場（`paths.secrets`。SAFE-15。マシン）。PAT と署名の鍵の値は設定ファイルに書かず、秘密の置き場のファイルに置く
- STRUCT-3 **Plugin を入れたプロジェクトで、写しから動かす。** 利用者は、当てる先のリポに Plugin をプロジェクトの範囲で入れ（入れる設定は当てる先のリポの `.claude/settings.json` に commit される）、当てる先のリポで開いた Claude Code で `setup` を実行する（SETUP-1）。
  `setup` は、インストールされた Plugin の版（`${CLAUDE_PLUGIN_ROOT}` の中身）を、マシンの置き場の中の版ごとのフォルダ（例：`<置き場>/program/<版>/`）に写す。写しを「進行 Bot のプログラムの写し」と呼び、マシンの設定の `devAutopilotPath` は写しを指す（[ADR-20261017-003][adr-plugin] の決定 2・4）。
  進行 Bot は、写しの `src/main.ts` を定期の実行から起こされ、`claude -p --plugin-dir <写しの絶対パス> --agent dev-autopilot:worker` のように、写しを Plugin として読み込み、自分の Skill と agent を名前空間で呼ぶ。
  `--plugin-dir` は AI の worktree の中ではなく、写し（AI が書けず、読めるのは `skills/` だけ。SAFE-16）の絶対パスを指す（AI が worktree の中の agent 定義や Skill を書き換えて次のセッションに読ませる経路を塞ぐ。「[安全側に倒す設計][f05]」）。
  進行 Bot（`plugin/src/`）と、AI のセッションに読ませるエージェント定義と Skill は、同じ写しから使う（版がずれないようにするため。STRUCT-7 で同封した理由と同じ）。
  版を上げるのは、人が `setup` を実行したときだけ。Plugin が更新されても（自動の更新を含む）、写しは変わらない。写しを作り直すのは、進行 Bot が動いていない間に限る（実行の lock を取る）。
  配布物は `plugin/` に閉じ（business-os の ADR 20261009-02 と同じ考え方）、進行 Bot も配る。テスト・前提データ・文書は配らない。
  `/dev-autopilot` の手動の 1 回実行は、進行 Bot を直接起動せず、定期の実行と同じジョブを起こす（macOS は `launchctl kickstart`（`-k` なし）。Linux は登録した仕組みで同じジョブを起こす操作。[ADR-20261015-001][adr-read]、[ADR-20261017-003][adr-plugin] の決定 3）。
  人が呼ぶ Skill（`dev-autopilot` `setup` `check`）が進行 Bot のコードを使うときは、マシンの設定（[ADR-20261015-005][adr-settings]）の `devAutopilotPath`（写しの絶対パス）から `src/` の場所を知る。マシンの設定は、開いているプロジェクトの origin から引いた場所で見つける（STRUCT-2）。Plugin のパスを手順に埋め込まない。
  `setup` は、dev-autopilot 自身のリポでは何も作らずに止まる（「分離の時期の目安」）
- STRUCT-8 **定期の実行は、OS ごとの仕組みで登録し、進行 Bot の本体は登録の仕組みに依らない。** 今は macOS と Linux に対応する。macOS は launchd、Linux は systemd のユーザーのタイマーか cron で、`setup` が登録する（Linux でどちらにするかは基本設計で決める。[ADR-20261017-003][adr-plugin] の決定 3）。
  進行 Bot の本体（`plugin/src/`）は、どの仕組みから起こされても同じに動く。OS に依る処理（登録、手動実行の起こし方、秘密のファイルの権限の確かめ）は、`setup`・`check`・`/dev-autopilot` の中の OS ごとの部分に閉じる。
  Windows と、GitHub Actions などのクラウドで動かすことは、今は対応しない。広げるときは、進行 Bot の本体を変えずに、登録の仕組みと秘密の置き場を足す形にし、別の ADR で決める（クラウドの論点は UNDECIDED-21）。
  Linux の sandbox の振る舞いは確かめ直すまで（UNDECIDED-49）、Linux で進行 Bot を動かさない
- STRUCT-4 **検査は dev-autopilot のリポが持つ。** dev-autopilot のリポが、自分の `package.json`（検査のスクリプト）・依存とロックファイル・`.markdownlint-cli2.jsonc` と、コードを足すときの `tsconfig.json`・`vitest.config.ts` を持ち、business-os と共有しない（[ADR-20261016-003][adr-move] の決定 4）。
  衛生の検査（書式、lint、秘密の検査、Markdown）と CI と commit の規約の検査は、project-recipes のレシピで入れる
  - 単体テストは、Plugin に同封しないように `plugin/` の外の `test/unit/` に、`plugin/src/` と同じ形の路で置く（`plugin/src/select.ts` → `test/unit/select.test.ts`。[ADR-20261017-003][adr-plugin] の決定 1）。
    Claude Code の Plugin には、写すときにファイルを除く設定が無いため。`tsconfig.json` と `vitest.config.ts` は、`plugin/src/` と `test/` を対象にする
  - 単体テストが対象のコードの隣でなくなるので、CI で次の 2 つを確かめ、どちらかが外れたら CI を落とす（TypeScript の検査。検査そのものも vitest で検査する）
    - `plugin/src/` のコードのファイルごとに、`test/unit/` に同じ路の単体テストがあること。単体テストを持たないファイル（型の定義だけのファイルなど）は、理由を添えた例外の一覧に書いたものだけを認める
    - `plugin/` の中に、テストのファイル（`*.test.ts` など）が無いこと
  - 結合テスト（偽の `gh` と使い捨てのリポで進行 Bot を端から端まで回す）も `test/` に置く（単体テストとの分け方は基本設計で決める）。前提データは `fixtures/`
  - `review` Skill の評価も dev-autopilot のリポが持つ（「[レビューの Skill][f08]」。[ADR-20261016-003][adr-move] の決定 4）
- STRUCT-5 **文書は dev-autopilot のリポに置く。** dev-autopilot の要件、設計判断（ADR）、構造仕様は `docs/` に、使い方は `README.md` に置き、business-os の `docs/` には書かない
  - **ADR の規則は、利用者の全体設定の新しい規則（`project-recipes` の `adr-docs` レシピ）に従う。** business-os の旧規則（`YYYYMMDD-nn-<slug>.md`、4 日付欄）ではなく、dev-autopilot のリポだけで成り立つ規則を持つ
    - 導入：レシピの 3 ファイル `README.md`（運用ルール）・`index.md`（一覧）・`adr-template.md`（MADR 4.0.0 の日本語版）を `docs/adr/` に写して置く（済み）。初回だけ `~/.claude` のレシピから取り、以後は `~/.claude` に依存しない
    - ファイル名は `adr-<yyyymmdd>-<nnn>-<title>.md`。frontmatter の欄は `docs/adr/adr-template.md` に従う
    - 起票は `documentation-and-adrs` スキルで行い、`docs/adr/README.md` の規則に従わせる。AI は `proposed` 以外の status を付けない
    - Markdown のリンクは参照スタイル（レシピの規則）。business-os の規約（inline）とは逆になる
  - `docs/design/`：進行 Bot の構造仕様（状態遷移、判定の表、設定ファイルの項目）
  - `README.md`：使い方
  - **Markdown の規則は、リポの root の `.markdownlint-cli2.jsonc` に持つ**（`md-lint-format` レシピ。MD054 は参照スタイル）。Markdown は Prettier で整形しない（`.prettierignore`）
  - business-os 側の ADR は、dev-autopilot を business-os に置いた橋渡しの ADR 20261008-01 と、それを置き換える「分離した」ADR（[ADR-20261016-003][adr-move] の決定 5）だけにし、dev-autopilot の中身の判断は dev-autopilot の ADR に書く
- STRUCT-6 **`/verify-issue` との分担。** 汎用の部分（検疫、作業指示の下書き、依存とマイルストーンの候補）は `dev-autopilot:intake` に置き、進行 Bot が自動で呼ぶ（人の操作は `agent-ready` だけ。TEXT-12）。
  business-os の `/verify-issue` は、business-os 固有の確かめ方（hook・雛形・第三者のツールの再現）を残し、最初に検疫のモジュール（TEXT-11）を呼び、最後に `dev-autopilot:intake` で下書きを作る（TEXT-9、SETUP-17 と同じ順）。
  business-os からはリポをまたいだ呼び出しになる。呼び方は決めていない（UNDECIDED-43）

- STRUCT-7 **Skill と進行 Bot は Plugin に同封し、ほかの人に配るときは `review` だけ同じフォルダから単体でも入れられるようにする。** 別に公開して組み合わせる形は採らない
  - 同封する理由：進行 Bot・agent・Skill は出力契約や引数で結びついていて、版がずれると黙って壊れる。Plugin の 1 つの版で一緒に検査し、一緒に配る（進行 Bot の同封は [ADR-20261017-003][adr-plugin] の決定 1）
  - 単体で使う価値があるのは `review`（人が手でレビューするときの change-review v2）だけ。`setup` `check` `intake` `dev-autopilot` は進行 Bot と設定ファイルが無いと意味を持たない
  - skills CLI は、リポの `skills/<名前>/` を `<owner>/<repo>@<名前>` で単体導入できる。dev-autopilot のリポの `plugin/skills/review/` を指せるので（skills CLI は `plugin/skills/` の下の Skill を見つける。FACT-33）、
    単体で配るために別の場所へ写す必要は無い（正本は 1 つのまま）
  - そのため `review` は、Plugin の中からも単体からも動くように書く：手順の本体は単体で成り立たせ、dev-autopilot 向けの部分（無人の入力、出力契約、agent の名前）は引数で渡されたときだけ使う。印は進行 Bot が付け、Skill は印を出さない（REVIEW-3）。
    `${CLAUDE_PLUGIN_ROOT}` や agent の名前を手順の本体に埋め込まない。同封したファイル（参照ファイルとスクリプト）は `${CLAUDE_SKILL_DIR}` で指す（`SKILL.md` のあるフォルダに置き換わるので、Plugin の中からも単体でも同じに動く。[ADR-20261017-004][adr-human] の決定 1）
  - 配るまでの間は、`~/.claude/skills/change-review` を今の版のまま残して手動のレビューに使う。v2 を手でも使いたくなったら、
    dev-autopilot のリポの `plugin/skills/review/` を skills CLI（`--copy` 付き）で入れる

## 分離の時期の目安

置き場所は、基本設計の前に別のリポへ移した（2026-10-17。[ADR-20261016-003][adr-move]、[ADR-20261017-001][adr-history]）。
ほかのリポに当てることと、ほかの人に配ること（marketplace への登録、skills CLI での `review` の単体導入。STRUCT-7）は、マージ判定工程（段階 8。進行 Bot がマージの判定の印を書く段階）の出口を越え、設定ファイルの項目だけで別のリポに当てられると確かめてから決める。
2 つ目の適用先ができる前は、固有の値がコードに残っていても気づけないため。

- それまで、進行 Bot が当てる先は business-os だけにする。1 台のマシンで複数のリポに当てる形も、マージ判定工程（段階 8）の後に決める
- 形は、マージ判定工程（段階 8）の前から、Plugin を入れたプロジェクトで動く形（STRUCT-3）で作る。マージ判定工程（段階 8）の後に決めるのは、当てる先を広げることと、ほかの人への公開（marketplace への登録）の時期だけ（[ADR-20261017-003][adr-plugin]）
- dev-autopilot を、dev-autopilot 自身のリポに当てない。当てると、作業 AI が進行 Bot のコードを書き換える経路が戻る。`setup` は dev-autopilot 自身のリポでは止まる。当てるなら、別の ADR で決める

[f01]: ./01-issue-flow.md
[f02]: ./02-implement-review-fix.md
[f04]: ./04-external-text-and-markers.md
[f05]: ./05-safety.md
[f08]: ./08-review-skill.md
[adr-read]: ../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-settings]: ../adr/adr-20261015-005-settings-values-and-machine-settings.md
[adr-move]: ../adr/adr-20261016-003-move-to-own-repository-before-basic-design.md
[adr-history]: ../adr/adr-20261017-001-carry-history-into-own-repository.md
[adr-review]: ../adr/adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-plugin]: ../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-human]: ../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-mcp]: ../adr/adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[adr-tmp]: ../adr/adr-20261017-006-per-session-short-tmpdir.md
[adr-stop]: ../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-usage]: ../adr/adr-20261018-001-stop-session-and-count-usage.md
