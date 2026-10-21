# テスト

進行 Bot と Skill を確かめるテストの、種類・置く場所・偽物・前提データ・回し方を決める。
置く場所の大枠（単体テストは `plugin/` の外の `test/unit/` に `plugin/src/` と同じ形のパスで置く、前提データは `fixtures/`）は STRUCT-4 と [ADR-20261017-003][adr-plugin] の決定 1 が正本で、基本設計は、STRUCT-4 と ADR-20261017-003 が基本設計に残した単体テストと結合テストの分け方と、偽物と前提データの形を決める。

## テストの種類

![テストの種類][fig-kinds]

| 種類 | 置く場所 | 確かめるもの | 本物を使うもの | 回すとき |
| --- | --- | --- | --- | --- |
| 単体テスト | `test/unit/`（`plugin/src/a/b.ts` の単体テストは `test/unit/a/b.test.ts`。[STRUCT-4][struct]） | 1 つのモジュールが、渡された値から結果を決める処理（次の工程の表、収束とマージの判定、印の署名と読み取り、指摘の ID、設定と出力契約の確かめ、安全設定と環境変数の組み立て、費用の数え方、検疫と漏えいの検査） | 無し。GitHub・git・`claude`・時刻・プロセスの操作（シグナルを送る、子孫とプロセスグループを読む、`lsof`）・ファイルの情報（持ち主・権限・リンクか）は、引数で渡す偽物にする。ファイルは、テストごとの一時フォルダにだけ書く | CI、手元の `pnpm test` |
| 結合テスト | `test/integration/` | 進行 Bot の入口（`plugin/src/main.ts`）と、人が呼ぶ Skill の処理（`setup`・`check`・`/dev-autopilot`。`plugin/src/` の中のモジュール）から通したときの、工程の順番、1 回の実行で進める数、lock、置き場と記録の置き場のファイル、GitHub に書く内容、起こすプロセスと引数 | Node と git。`gh`・`claude`・`ssh`・`launchctl`・`systemctl` は偽物（下の「偽物」） | CI の macOS のランナー、手元の `pnpm test` |
| 構成の検査 | `test/repo/` | `plugin/src/` のファイルごとに単体テストがあること、`plugin/` の中にテストのファイルが無いこと（[STRUCT-4][struct]）、`plugin/src/` の import が `node:` で始まるか相対パスだけであること（「[導入と点検][design-setup]」の「依存」） | リポのファイルの一覧 | CI、手元の `pnpm test` |
| 評価 | `evals/` | `review` Skill（business-os の `change-review` を組み直した v2）が、レビュー済みの PR で、人が出した 🔴 🟡 を出すか（一致率と増加率。「[レビューの Skill][review-skill]」） | `claude`（利用枠を使う） | 人が手で回す。CI では回さない |

- 単体テスト・結合テスト・構成の検査は、本物の GitHub・`claude`・npm のレジストリに触れない。利用枠を使わず、ネットワークが無くても通る
- 結合テストは、CI の macOS のランナーで回す。点検の OS ごとの部分は、動かしてよい OS（今は macOS）でなければ fail にする（「[導入と点検][design-setup]」の「組と回数」）ので、Linux のランナーでは進行 Bot が作業に入らず、流れを確かめられないため。テストのためだけの切り替え（OS の確かめを外す環境変数など）は持たない。リポは公開なので、macOS のランナーにも費用は掛からない。単体テストと構成の検査は、OS に依らないので Linux のランナーでも回す。CI の設定（`.github/workflows/`）は、テストのコードを入れる PR で分ける
- sandbox が読み書きを止めること、MCP の道具が無いことなど、Claude Code の振る舞いは、テストでなく点検の試しのセッション（「[導入と点検][design-setup]」の「試しのセッション」）で確かめる。偽の `claude` は sandbox を持たないため
- 評価をテストに入れないのは、結果がモデルの返事で揺れ、利用枠を使うため。数は、レビュー工程（段階 6。レビュー AI と批評 AI を足す段階）に入る前の確かめに使う（「[レビューの Skill][review-skill]」）

## 単体テストと結合テストの分け方

| 書く先 | 当てはまるもの | 例 |
| --- | --- | --- |
| 単体テスト | 1 つのモジュールの中で、入力の値から結果が決まるもの | 次の工程の表の各行、収束とマージの判定、印の署名、指摘の ID の計算（[ADR-20261019-001][adr-repeat]）、費用の数え方と試しのセッションの判定（実機の試しで得た `stream-json` の記録を前提データにする。[ADR-20261018-001][adr-usage]）、1 日の区切りや CI を待つ時間のような時刻で決まる判定。ADR の確認方法が単体テストとしたもの：止める手順の順番（[ADR-20261018-001][adr-usage]）、短い一時フォルダの根の持ち主と権限の確かめ（[ADR-20261017-006][adr-tmp]）、計画の工程の中の順番と、計画を投稿しない条件（[ADR-20261018-002][adr-redo]、[ADR-20261018-004][adr-planrev]、[ADR-20261019-003][adr-replan]、[ADR-20261019-004][adr-policy]）、見落としの軸の結果を印に入れないこと（[ADR-20261019-002][adr-gaps]）、`review` Skill の本文の置き換えと、置き換えの書き方が残ったときにセッションを起こさないこと（[ADR-20261021-001][adr-review-skill]）、書かずに試す間の受け入れの相手と選定の結果、Issue の累計と失敗の回数から書かずに試した回を除くこと、`run.dryRun` と `run.stage` の組の確かめ（[ADR-20261021-002][adr-dry]）、承認のやり直しの条件と、最新の `intake-final` の印だけを読むこと、push 未満の人の作業指示コメントを読まずに止めないこと、作業指示コメントが 2 つ以上なら承認の確定・着手・対応ループで止めること（[ADR-20261021-003][adr-reapprove]） |
| 結合テスト | 2 つ以上のモジュールをまたぎ、プロセス・ファイル・実行の回数をまたいで決まるもの | 1 回の実行で Issue ごとに進める重い工程が 1 つまでであること、2 回の実行で待ちのコメントが 1 件であること、lock、片付け、AI のセッションを起こした後の clone で git を実行しないこと（[ADR-20261013-001][adr-workspace]）、止める手順で残ったコマンドを止めること、書かずに試す実行を 2 回通して 2 回目が別の Issue を受け入れること、`run.dryRun` を `false` にした最初の実行が受け入れの印の無い Issue を受け入れ直すこと（[ADR-20261021-002][adr-dry]）、`brief-changed` で止まった Issue を、文を直して `Todo` に戻し、`agent-ready` を付け直すと、新しい `intake-final` の印を書いて計画し直すこと（[ADR-20261021-003][adr-reapprove]） |

- ADR の確認の方法が、単体テストか結合テストかを書いているものは、ADR の書き方に従う
- 迷ったら単体テストに書き、結合テストは流れの確かめに絞る。結合テストは時間がかかり、落ちたときに原因の場所が分かりにくいため
- 結合テストでは時刻を差し替えない。進行 Bot のプログラムに、テストのためだけの切り替え（時刻を変える環境変数など）を持たないため。時刻で決まる判定は、単体テストで確かめる

## 偽物

![結合テストの組み立て][fig-integration]

結合テストは、テストごとに一時フォルダを作り、偽物を `PATH` の先頭に置いて、進行 Bot の入口を子プロセスで起こす。進行 Bot のプログラムは本番と同じものを使い、テストのためだけの切り替えを持たない。偽物は、テストの中で一度に呼ばれうるものをすべて答えられるように、筋書きごとの最初の状態を前提データに持つ。

| 偽物 | 置く場所 | 振る舞い |
| --- | --- | --- |
| 偽の `gh` | `test/integration/fakes/gh.ts` | GitHub の状態を、テストの一時フォルダの JSON に持つ。持つもの：Issue、PR、コメントとその編集の記録（`userContentEdits`）、ラベル、timeline のイベント、依存（blocked by）、マイルストーン、Project の欄と自動追加の状態、リポの変数（作る試しには 403 を返す）、ruleset、コラボレータの権限、認証の主（`GET /user`）、アプリのトークンの引き換え、必須チェックの結果、ワークフローの実行とジョブ。最初の状態は `fixtures/gh/` から読む。書き込みは状態を変え、読み取りは状態から答え、呼ばれた引数をすべて記録する。知らない呼び方は、終了コード 1 で終わって記録に残す（黙って成功しない） |
| 偽の `claude` | `test/integration/fakes/claude.ts` | 引数・環境変数の名前・標準入力を記録し、`fixtures/claude/` の筋書きに従って動く。`--version` には決めた版を返す。筋書きは、`--agent` の値と、作業フォルダ（セッションの名前）で選ぶ。同時に起こす 3 つの軸の起動の順は毎回変わるので、何回目の起動かでは選ばない。筋書きの動きは、`stream-json` を書いて終わる、worktree に commit を作る（本物の git を絶対パスで呼び、下の git の記録の包みを通さない）、費用の上限に当たる、拒否で止まる、壊れた出力契約を返す、返事をせずに待つ、`SIGTERM` を無視するコマンドを裏に残す、試しのセッションの結果を書く |
| 偽の `ssh` | `test/integration/fakes/ssh.ts` | 進行 Bot が `GIT_SSH_COMMAND` で git に渡す `ssh` を差し替え、`git-upload-pack` と `git-receive-pack` を、テストの一時フォルダの bare のリポ（作業フォルダを持たない、受け取り専用のリポ）に対して手元で動かす。進行 Bot の clone の URL は本番と同じ形のままで、テストのためだけの切り替えを持たない。pack でやりとりするので、objects をハードリンクで共有しない（点検のリンク数の確かめが通る。FACT-19） |
| 偽の `launchctl`・`systemctl` | `test/integration/fakes/launchctl.ts`、`test/integration/fakes/systemctl.ts` | 引数を記録する（`kickstart` に `-k` を付けないことを確かめる。[ADR-20261015-001][adr-read]）。`kickstart` と `start` を受けたら、登録した定義と同じ引数で、進行 Bot の入口を子プロセスで起こす（人の `check` が結果のファイルを待つ流れを確かめるため） |
| git の記録の包み（[ADR-20261013-001][adr-workspace] の「偽の git」） | `test/integration/fakes/git.ts` | 呼ばれた引数と作業フォルダを記録してから、本物の git に渡す。AI のセッションを起こした後の clone で、取り込みの fetch のほかに git を実行しないことを確かめる（[ADR-20261013-001][adr-workspace]） |

- テストの準備で、一時フォルダの `bin/` に、`gh` などの名前の起動の包み（偽物を呼ぶための小さな起動用のファイル）を作る。包みは、`fakes/` の TypeScript を読み込むだけの JavaScript にする（拡張子の無いファイルは TypeScript として読まれないため）。偽物も bash で書かない（[07-structure.md][struct] の決まり）
- origin は、テストの一時フォルダに作る bare のリポで、`fixtures/repos/` の雛形から作る。clone と push は、上の偽の `ssh` を通して行う。deploy key は、テストの中で作る使い捨ての鍵にする
- 秘密の置き場には、テストの中で作る使い捨ての値（意味の無い PAT の文字列、テストの中で作るアプリの秘密鍵、印の署名の鍵）を置く。本物の秘密を前提データに入れない
- 短い一時フォルダの根は、テストごとに `/tmp` の直下に短い名前のフォルダ（`/tmp/dapt-<英小文字と数字の 4 文字>`）を作って、マシンの設定の `sessionTmp.root` に書き、テストの後に消す。本物の根（`/tmp/dap`）を使わないのは、同じ Mac で動く本物の進行 Bot とテストが、互いのフォルダを残骸として fail にしないため。実体のパス（`/private/tmp/dapt-…/<ID>/claude-<uid>`）が 44 バイトに収まる長さにする
- テストの準備で、記録の置き場（`records-created.json` と、前の試しのセッションの pass の記録）、プログラムの写しと manifest、定期の実行の定義のファイル（一時フォルダの `HOME` の下）を作る。毎回の実行の最初の点検が、作業に入る前に fail にしないため。試しのセッションを確かめる筋書きでは、前の記録を置かない
- `HOME` と XDG の 4 つ（`XDG_CONFIG_HOME` `XDG_CACHE_HOME` `XDG_DATA_HOME` `XDG_STATE_HOME`）を、テストの一時フォルダに向ける。マシンの設定は、一時フォルダの `settings.json` のパスを入口の引数で渡す（「[導入と点検][design-setup]」の「定期の実行の登録」と同じ形）。テストが、人のホームと Claude Code の設定に書かないため

## 前提データ

| フォルダ | 中身 | 使うテスト |
| --- | --- | --- |
| `fixtures/gh/` | 偽の `gh` の最初の状態（筋書きごとの JSON） | 結合テスト |
| `fixtures/claude/` | 偽の `claude` の筋書きと、実機の試しで得た `stream-json` の記録 | 単体テスト（費用の数え方、出力契約の取り出し、試しのセッションの判定）、結合テスト |
| `fixtures/repos/` | 使い捨てのリポの雛形（ファイルの木） | 結合テスト |
| `fixtures/texts/` | Issue と PR の本文の例 | 単体テスト、結合テスト |
| `fixtures/quarantine/` | 検疫の見本（指示の文例、区切りを模した文、見えない文字。[TEXT-11][text]）。見つけるべき文と、見つけてはいけない文の組 | 単体テスト |
| `fixtures/leak/` | 漏えいの検査の見本（メールアドレス、機械上のパス、進行 Bot の設定と記録の内容の形。[IMPL-15][impl]）。見つけるべき文と、見つけてはいけない文の組 | 単体テスト |
| `fixtures/repo-layout/` | 構成の検査を確かめるファイルの木（単体テストが欠けた木、`plugin/` にテストがある木、例外の一覧に書いたファイルがある木、`plugin/src/` で npm のパッケージを import した木、実在しないファイルを例外の一覧に書いた木、理由が空の行がある木） | 構成の検査 |

- 検疫の見本は `fixtures/quarantine/` に置く（UNDECIDED-22 を、ここで決めた）。STRUCT-4 と構成の図が、前提データを `fixtures/` に置くのに合わせる
- 鍵の形の見本は、ファイルに書かず、テストの中で部品をつないで作る。ファイルに書くと gitleaks が commit を止め、止めないためには gitleaks の許す一覧を広げることになるため
- メールアドレスの見本は、例のために予約されたドメインのうち、`.gitleaks.toml` が許す 4 つ（`example.com`・`example.org`・`example.net`・`example.invalid`）で書く。ほかのドメインは gitleaks が commit を止める
- 実機の `stream-json` の記録と、business-os の PR の差分と本文を前提データに置くときは、人のホームのパス・ユーザー名・`session_id`・業務の固有名詞を置き換えてから置く。置く前に、gitleaks と目で確かめる（固有名詞は gitleaks が見つけられないため）

## 回し方

- vitest の project を、単体テスト・結合テスト・構成の検査の 3 つに分ける。`pnpm test` で 3 つとも回す。CI は、Linux のランナーで単体テストと構成の検査を、macOS のランナーで結合テストを回す（上の「テストの種類」）。`tsconfig.json` は、`plugin/src/`・`plugin/skills/**/scripts/`・`test/`・`evals/` の TypeScript を型の検査の対象にする。`vitest.config.ts` は `test/` を対象にする（[STRUCT-4][struct]）
- 構成の検査の「ファイルごとに単体テストがあること」は、`plugin/src/` だけに掛ける。`plugin/skills/**/scripts/` の検査の単体テストは `test/unit/skills/` に置き、`evals/` は評価の入口なので単体テストを求めない
- 構成の検査は、進行 Bot のコードを書き始める PR で入れる（[ADR-20261017-003][adr-plugin] の確認の方法）。単体テストは、対象のコードと同じ PR で足す（構成の検査が、単体テストの無いファイルを止めるため）
- 評価は、`evals/` の中の入口（TypeScript）から回す。評価する `review` Skill は、business-os の `change-review` の Skill を組み直した版（v2。「[レビューの Skill][review-skill]」）。入口は、`plugin/` と評価のケースを一時フォルダに集めて Plugin を組み立て、`claude plugin eval` を呼ぶ（business-os の `pnpm eval:cases` と同じ考え方で、business-os のコードは import しない。「[レビューの Skill][review-skill]」、[ADR-20261016-003][adr-move] の決定 4）
- 評価のケース（PR の差分と、期待する指摘の一覧）は `evals/review/` に置く。入口は、結果で Plugin が読み込まれたかを確かめ、読み込まれていなければ不合格にする（Plugin が読み込まれずに素のモデルで走っても、指摘の数だけでは見分けにくいため）

## 単体テストを持たないファイル

構成の検査の「`plugin/src/` のファイルごとに単体テストがあること」の例外（型の定義だけのファイルなど）は、理由を添えた一覧に書いたものだけを認める（[STRUCT-4][struct]）。

- 一覧の場所は `test/repo/no-unit-test.json`。1 件は、`path`（`plugin/src/` からのパス）と `reason`（理由）を持つ
- 構成の検査は、一覧のファイルが実在すること、理由が空でないことも確かめる。消したファイルの行と、理由の無い行が残らないようにするため

[adr-plugin]: ../../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-repeat]: ../../adr/adr-20261019-001-repeated-findings-by-ids.md
[adr-usage]: ../../adr/adr-20261018-001-stop-session-and-count-usage.md
[adr-workspace]: ../../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ../../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-move]: ../../adr/adr-20261016-003-move-to-own-repository-before-basic-design.md
[struct]: ../../requirements/07-structure.md
[text]: ../../requirements/04-external-text-and-markers.md
[impl]: ../../requirements/02-implement-review-fix.md
[review-skill]: ../../requirements/08-review-skill.md
[design-setup]: ./04-setup-and-check.md
[fig-kinds]: ./diagrams/05-test-kinds.drawio.svg
[fig-integration]: ./diagrams/05-integration.drawio.svg
[adr-tmp]: ../../adr/adr-20261017-006-per-session-short-tmpdir.md
[adr-redo]: ../../adr/adr-20261018-002-plan-redo-in-same-run.md
[adr-planrev]: ../../adr/adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[adr-replan]: ../../adr/adr-20261019-003-replan-on-plan-review-from-stage-5.md
[adr-policy]: ../../adr/adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths.md
[adr-gaps]: ../../adr/adr-20261019-002-review-gaps-axis-in-shadow-first.md
[adr-review-skill]: ../../adr/adr-20261021-001-review-skill-text-without-skill-tool.md
[adr-dry]: ../../adr/adr-20261021-002-dry-run-only-at-intake-stage.md
[adr-reapprove]: ../../adr/adr-20261021-003-reapprove-by-relabel-and-ignore-outside-instructions.md
