# 分離を見越した構成

将来は別リポジトリ（独立した Plugin）で管理する前提で、いまから 1 つのフォルダに閉じて作る。

## 構成

```text
dev-autopilot/
  plugin/                      配布物。Plugin として読み込む範囲（`--plugin-dir` はここを指す。分離後は marketplace の `source` をここへ向ける）
    .claude-plugin/plugin.json 名札
    skills/
      dev-autopilot/SKILL.md   人が呼ぶ入口（/dev-autopilot：状態の表示、手動の 1 回実行）
      setup/SKILL.md           導入（SETUP-1。足りない基盤を作る。冪等）
      check/SKILL.md           点検の入口（`src/check.ts` を呼ぶ包み。判定のロジックは持たない）
      review/SKILL.md          レビュー AI の手順（change-review v2 の正本。08-review-skill.md）
      review/scripts/per-commit-gates.ts  コミットごとのゲートの検査（TypeScript。単体導入でも Skill と一緒に写る）
      intake/SKILL.md          Issue の受け入れの手順（検疫、作業指示の下書き、依存とマイルストーンの候補）。人が `/verify-issue` などから呼ぶ入口。進行役は agents/intake.md を起動する。人の操作は agent-ready だけ
    agents/
      planner.md               計画役（読むだけ。計画を出力契約で返す）
      worker.md                作業 AI（モデルは設定で）
      reviewer.md              レビュー AI
      critic.md                批評者（レビューの指摘の再検証。進行役が別に起こす独立セッション。IMPL-22）
      intake.md                受け入れ AI（道具なし。作業指示の下書き（外の人の Issue だけ）と計画の難しさの判定を出力契約で返す。検疫で見つかった Issue では起動しない。00-overview.md の「登場するもの」、TEXT-12）
  src/                         進行役（TypeScript、Node 24 が直接実行。bash を置かない）。Plugin としては配らず、cron が直接動かす
    main.ts                    入口
    select.ts                  例：Issue の選定
    select.test.ts             その単体テスト。テストは対象のコードの隣に置く（xxx.ts → xxx.test.ts）
    marker.ts                  印の形の正本（1 定数）
    check.ts                   点検の正本（進行役が毎回の最初に回す）
  fixtures/                    結合テストの前提データ（偽の gh の応答、使い捨てリポの雛形、Issue と PR の本文の例）
  test/                        結合テストだけ（偽の gh を差し替えて進行役を端から端まで回す）。単体テストは置かない
  docs/                        要件（requirements/）、設計判断（adr/）、構造仕様（design/）
  README.md                    使い方。業務の固有名詞を書かない
  .markdownlint-cli2.jsonc     参照スタイルのリンクを許す入れ子の設定
```

## 守ること

- STRUCT-1 **business-os の中身を import しない。** `plugin/hooks/lib` や `plugin/scripts/lib` の関数が要るなら、dev-autopilot 側に持つ。逆に business-os が dev-autopilot を import することもしない
- STRUCT-2 **設定の値はコードに書かず、リポの設定とマシンの設定の 2 つに出す。** リポの設定は business-os 側（例：`.claude/dev-autopilot.json`。commit する）で、マシンに依らない値を持つ。マシンの設定は commit せず、リポの外の同期されないフォルダに置き、下の一覧で「マシン」と書いた項目を持つ（[ADR-20261015-005][adr-settings]）。
  項目の一覧は次のとおり（`check` はすべての項目の存在と型を確かめる。SETUP-2）。項目ごとの置き場・初期値・根拠の表は、構造仕様（`dev-autopilot/docs/design/`）に置く。
  要件が初期値を決めていない項目は、構造仕様の表が初期値と根拠の正本で、今の値の正本は設定ファイル。値は設定ファイルの PR で変える。安全の境界を緩める変更（AI の道具・読み書きの範囲・通信先を広げる、MERGE-4 の許可一覧にパスを足す、IMPL-15 の一覧からパスを外す、上限を外す）は、値の変更であっても ADR で決める（[ADR-20261015-005][adr-settings]）
  - リポと Project：owner / repo、base ブランチ、Project の番号、Status と Agent 欄の ID と選択肢の ID（SETUP-5、SETUP-6）
  - ラベル名：`agent-ready` `needs-human` `from-review` `plan:fable` `plan:opus`（SETUP-8）。ブランチの型のラベル対応表（IMPL-2）
  - 固定の見出しの文字列：`## 作業指示` `## 却下 R<n>` `## 判定と違う判断` `## 異議`（「[印と固定の見出しの一覧][f04]」）
  - 進行役の実行：`enabled`、`dryRun`、`stage`（段階ごとの振る舞い。「[Issue の状態の遷移][f01]」）、`devAutopilotPath`（絶対パス。STRUCT-3。マシン）、worktree の置き場（マシン）、記録と lock の置き場（マシン）、cron の間隔（launchd の plist と二重になるので `check` がずれを検出する）
  - 上限：1 回の実行で着手する件数（FLOW-3）、WIP の上限（FLOW-3）、1 回の実行の所要時間（FLOW-4）、受け入れの件数（TEXT-12）、往復の回数（IMPL-24）、セッションの時間・ターン・費用（IMPL-13）、1 Issue の累計の利用枠と 1 日の利用枠（IMPL-14、SETUP-12）、実行の失敗の再試行の回数と間隔（IMPL-14）、催促までの日数（MERGE-3）、CI の結果が取り消しなどのまま変わらないときに待つ時間（IMPL-27）
  - モデル：役ごとのモデルと effort（「[モデルの流れ][f02]」）、IMPL-25 で上げる先、受け入れ AI の拒否後のモデル、Agent ツールを許すか（IMPL-14）、批評者の確信度のしきい値（IMPL-22）
  - 検査と判定：品質ゲートのコマンド（`pnpm check`。AI のセッションの中で使う）、必須チェックの名前と app とワークフローのパス（IMPL-27）、マージ判定の許可一覧（MERGE-4）、方針に係るパスの一覧（MERGE-4）、進行役が実行するパスの一覧（IMPL-15）、漏えい検査の見本の置き場
  - 規約：コミットと PR の規約（言語、Conventional Commits、changeset の要否）
  - 安全：進行役のアカウント名（IMPL-20）、sandbox の `--settings` の置き場（マシン）と `denyRead` `allowRead` `denyWrite` の一覧（`allowRead` に足すホームの中の道具のフォルダはマシン。「[隔離と鍵の分け方][f05]」）、外への通信の許可先、子プロセスに渡す環境変数の名前の一覧（SAFE-10 の git と pnpm のための環境変数を含む。「[安全側に倒す設計][f05]」）、秘密の置き場（`paths.secrets`。SAFE-15。マシン）。PAT と署名の鍵の値は設定ファイルに書かず、秘密の置き場のファイルに置く
- STRUCT-3 **起動は Plugin として。** 進行役は `claude -p --plugin-dir <進行役専用 clone の絶対パス>/dev-autopilot/plugin --agent dev-autopilot:worker` のように、`plugin/` を Plugin として読み込み、自分の Skill と agent を名前空間で呼ぶ。
  `--plugin-dir` は AI の worktree の中ではなく、進行役が実行している専用 clone（AI が書けない）の絶対パスを指す（AI が worktree の中の agent 定義や Skill を書き換えて次のセッションに読ませる経路を塞ぐ。「[安全側に倒す設計][f05]」）。
  分離したら `--plugin-dir` が Marketplace からの導入（`source` は `./plugin`）に変わるだけで、呼び方は変わらない。
  配布物は `plugin/` に閉じ（business-os の ADR 20261009-02 と同じ考え方）、進行役の `src/`・テスト・前提データ・文書は配らない。
  `/dev-autopilot` の手動の 1 回実行は、進行役を直接起動せず、`launchctl kickstart`（`-k` なし）で定期の実行と同じ launchd のジョブを起こす（[ADR-20261015-001][adr-read]）。
  人が呼ぶ Skill（`dev-autopilot` `setup` `check`）が進行役のコードを使うときは、マシンの設定（[ADR-20261015-005][adr-settings]）の `devAutopilotPath`（dev-autopilot のフォルダの絶対パス。分離前は進行役専用 clone の中の `dev-autopilot/`、分離後は専用マシンの clone）から `src/` の場所を知る。Plugin のパスを手順に埋め込まない
- STRUCT-4 **検査は dev-autopilot 自身が持ち、business-os はそれを呼ぶだけ。** `dev-autopilot/` を pnpm の workspace のパッケージにし、自分の `package.json`（`check` スクリプト）・`tsconfig.json`・`vitest.config.ts`・`.markdownlint-cli2.jsonc` を持つ。
  型検査・vitest・markdownlint（参照スタイルのリンク）は dev-autopilot の `check` が回す。business-os の `package.json` には `check:dev-autopilot`（`pnpm --filter dev-autopilot check`）を 1 つ足すだけで、
  business-os の tsconfig と vitest の include には `dev-autopilot/` を足さない。分離するときは `check:dev-autopilot` の 1 行を消すだけで済む（橋渡し ADR 20261008-01 の決定 5 と同じ）
  - 単体テストは対象のコードの隣（`xxx.ts` → `xxx.test.ts`）。配布への影響は無い（配布物は `plugin/` に閉じ、`src/` は配られない）
  - 結合テスト（偽の `gh` と使い捨てのリポで進行役を端から端まで回す）だけ `test/` に置く。前提データは `fixtures/`
  - business-os の `check:format` `check:lint` `check:leak` `check:shell` はリポジトリ全体の衛生の検査なので、root のまま `dev-autopilot/` にも効く。分離後は dev-autopilot 側が project-recipes で同じものを入れる
- STRUCT-5 **文書は dev-autopilot のフォルダに閉じる。** dev-autopilot の要件、設計判断（ADR）、構造仕様、使い方は `dev-autopilot/docs/` に置き、business-os の `docs/` には書かない。
  分離するときはフォルダごと移るので、写しも supersede も要らない
  - **ADR の規則は、利用者の全体設定の新しい規則（`project-recipes` の `adr-docs` レシピ）に従う。** business-os の旧規則（`YYYYMMDD-nn-<slug>.md`、4 日付欄）ではなく、
    分離後も単独で成り立つ規則を dev-autopilot 側に持つ
    - 導入：レシピの 3 ファイル `README.md`（運用ルール）・`index.md`（一覧）・`adr-template.md`（MADR 4.0.0 の日本語版）を `dev-autopilot/docs/adr/` に写して置く（済み）。初回だけ `~/.claude` のレシピから取り、以後は `~/.claude` に依存しない
    - ファイル名は `adr-<yyyymmdd>-<nnn>-<title>.md`。frontmatter の欄は `dev-autopilot/docs/adr/adr-template.md` に従う
    - 起票は `documentation-and-adrs` スキルで行い、`dev-autopilot/docs/adr/README.md` の規則に従わせる。AI は `proposed` 以外の status を付けない
    - Markdown のリンクは参照スタイル（レシピの規則）。business-os の規約（inline）とは逆になる
  - `dev-autopilot/docs/design/`：進行役の構造仕様（状態遷移、判定の表、設定ファイルの項目）
  - `dev-autopilot/README.md`：使い方
  - **dev-autopilot の Markdown の規則は、入れ子の設定で dev-autopilot 側に持つ。** business-os の `.markdownlint-cli2.jsonc` は `**/*.md` に MD054 の inline を強制しており、
    参照スタイルの dev-autopilot の文書と衝突する。dev-autopilot には自分の markdownlint の設定（入れ子の `dev-autopilot/.markdownlint-cli2.jsonc`。`md-lint-format` レシピ、MD054 は参照スタイル）を置く（橋渡し ADR 20261008-01 の決定 5）。markdownlint-cli2 は下位フォルダの設定をそのフォルダ以下に適用する（`check:md` と pre-commit で確認）。
    `check:adr` `check:docs` も `docs/` しか見ないので同じ扱い。これは STRUCT-4 の具体化
  - business-os 側の ADR は橋渡しの **1 本だけ**（ADR 20261008-01。承認済み）にし、dev-autopilot の中身の判断は dev-autopilot 側の ADR に書く
- STRUCT-6 **`/verify-issue` との分担。** 汎用の部分（検疫、作業指示の下書き、依存とマイルストーンの候補）は `dev-autopilot:intake` に置き、進行役が自動で呼ぶ（人の操作は `agent-ready` だけ。TEXT-12）。
  business-os の `/verify-issue` は、business-os 固有の確かめ方（hook・雛形・第三者のツールの再現）を残し、最初に検疫のモジュール（TEXT-11）を呼び、最後に `dev-autopilot:intake` で下書きを作る（TEXT-9、SETUP-17 と同じ順）

- STRUCT-7 **Skill は Plugin に同封し、分離後は同じフォルダから単体でも入れられるようにする。** 別に公開して組み合わせる形は採らない
  - 同封する理由：進行役・agent・Skill は出力契約や引数で結びついていて、版がずれると黙って壊れる。Plugin の 1 つの版で一緒に検査し、一緒に配る
  - 単体で使う価値があるのは `review`（人が手でレビューするときの change-review v2）だけ。`setup` `check` `intake` `dev-autopilot` は進行役と設定ファイルが無いと意味を持たない
  - skills CLI は、リポの `skills/<名前>/` を `<owner>/<repo>@<名前>` で単体導入できる。分離した dev-autopilot のリポの `plugin/skills/review/` を指せるので（skills CLI が `plugin/` の下の Skill を見つけるかは確かめていない。UNDECIDED-45）、
    単体で配るために別の場所へ写す必要は無い（正本は 1 つのまま）
  - そのため `review` は、Plugin の中からも単体からも動くように書く：手順の本体は単体で成り立たせ、dev-autopilot 向けの部分（無人の入力、出力契約、agent の名前）は引数で渡されたときだけ使う。印は進行役が付け、Skill は印を出さない（REVIEW-3）。
    `${CLAUDE_PLUGIN_ROOT}` や agent の名前を手順の本体に埋め込まない
  - 分離前（business-os の中にある間）は、`~/.claude/skills/change-review` を今の版のまま残して手動のレビューに使う。v2 を手でも使いたくなったら、
    business-os の `dev-autopilot/plugin/skills/review/` を skills CLI（`--copy` 付き）で入れる

## 分離の時期の目安

段階 8（進行役のマージの判定）まで business-os で動かし、設定ファイルの項目だけで別のリポに適用できると確かめられたら分離する。
2 つ目の適用先ができる前に分離すると、固有の値がコードに残っていても気づけない。

[f01]: ./01-issue-flow.md
[f02]: ./02-implement-review-fix.md
[f04]: ./04-external-text-and-markers.md
[f05]: ./05-safety.md
[adr-read]: ../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-settings]: ../adr/adr-20261015-005-settings-values-and-machine-settings.md
