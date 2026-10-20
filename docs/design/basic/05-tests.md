# テスト

進行 Bot と Skill を確かめるテストの、種類・置き場・偽物・前提データ・回し方を決める。
置き場の大枠（単体テストは `plugin/` の外の `test/unit/` に `plugin/src/` と同じ形の路で置く、前提データは `fixtures/`）は STRUCT-4 と [ADR-20261017-003][adr-plugin] の決定 1 が正本で、基本設計は、STRUCT-4 と ADR-20261017-003 が基本設計に残した単体テストと結合テストの分け方と、偽物と前提データの形を決める。

## テストの種類

![テストの種類][fig-kinds]

| 種類 | 置き場 | 確かめるもの | 本物を使うもの | 回すとき |
| --- | --- | --- | --- | --- |
| 単体テスト | `test/unit/`（`plugin/src/a/b.ts` の単体テストは `test/unit/a/b.test.ts`。[STRUCT-4][struct]） | 1 つのモジュールが、渡された値から結果を決める処理（次の工程の表、収束とマージの判定、印の署名と読み取り、指摘の ID、設定と出力契約の確かめ、安全設定と環境変数の組み立て、費用の数え方、検疫と漏えいの検査） | 無し。GitHub・git・`claude`・時刻は、引数で渡す偽物にする。ファイルは、テストごとの一時フォルダにだけ書く | CI、手元の `pnpm test` |
| 結合テスト | `test/integration/` | 進行 Bot の入口（`plugin/src/main.ts`）から 1 回以上の実行を通したときの、工程の順番、1 回の実行で進める数、lock、置き場のファイル、GitHub に書く内容、起こすプロセスと引数 | Node と git。`gh`・`claude`・`launchctl`・`systemctl` は偽物（下の「偽物」） | CI、手元の `pnpm test` |
| 構成の検査 | `test/repo/` | `plugin/src/` のファイルごとに単体テストがあること、`plugin/` の中にテストのファイルが無いこと（[STRUCT-4][struct]）、`plugin/src/` の import が `node:` で始まるか相対パスだけであること（「[導入と点検][design-setup]」の「依存」） | リポのファイルの一覧 | CI、手元の `pnpm test` |
| 評価 | `evals/` | `review` Skill（`change-review` の v2）が、レビュー済みの PR で、人が出した 🔴 🟡 を出すか（一致率と増加率。「[レビューの Skill][review-skill]」） | `claude`（利用枠を使う） | 人が手で回す。CI では回さない |

- 単体テスト・結合テスト・構成の検査は、本物の GitHub・`claude`・npm のレジストリに触れない。利用枠を使わず、ネットワークが無くても通る
- sandbox が読み書きを止めること、MCP の道具が無いことなど、Claude Code の振る舞いは、テストでなく点検の試しのセッション（「[導入と点検][design-setup]」の「試しのセッション」）で確かめる。偽の `claude` は sandbox を持たないため
- 評価をテストに入れないのは、結果がモデルの返事で揺れ、利用枠を使うため。数は、レビュー工程（段階 6。レビュー AI と批評 AI を足す段階）に入る前の確かめに使う（「[レビューの Skill][review-skill]」）

## 単体テストと結合テストの分け方

| 書く先 | 当てはまるもの | 例 |
| --- | --- | --- |
| 単体テスト | 1 つのモジュールの中で、入力の値から結果が決まるもの | 次の工程の表の各行、収束とマージの判定、印の署名、指摘の ID の計算（[ADR-20261019-001][adr-repeat]）、費用の数え方（実機の試しで得た `stream-json` の記録を前提データにする。[ADR-20261018-001][adr-usage]）、1 日の区切りや CI を待つ時間のような時刻で決まる判定 |
| 結合テスト | 2 つ以上のモジュールをまたぎ、プロセス・ファイル・実行の回数をまたいで決まるもの | 1 回の実行で Issue ごとに進める重い工程が 1 つまでであること、2 回の実行で待ちのコメントが 1 件であること、lock、片付け、AI のセッションを起こした後の clone で git を実行しないこと（[ADR-20261013-001][adr-workspace]）、止める手順で残ったコマンドを止めること |

- ADR の確認の方法が、単体テストか結合テストかを書いているものは、ADR の書き方に従う
- 迷ったら単体テストに書き、結合テストは流れの確かめに絞る。結合テストは時間がかかり、落ちたときに原因の場所が分かりにくいため
- 結合テストでは時刻を差し替えない。進行 Bot のプログラムに、テストのためだけの切り替え（時刻を変える環境変数など）を持たないため。時刻で決まる判定は、単体テストで確かめる

## 偽物

![結合テストの組み立て][fig-integration]

結合テストは、テストごとに一時フォルダを作り、偽物を `PATH` の先頭に置いて、進行 Bot の入口を子プロセスで起こす。進行 Bot のプログラムは本番と同じものを使い、テストのためだけの切り替えを持たない。

| 偽物 | 置き場 | 振る舞い |
| --- | --- | --- |
| 偽の `gh` | `test/integration/fakes/gh.ts` | GitHub の状態（Issue、PR、コメント、ラベル、timeline のイベント、Project の欄、リポの変数、必須チェックの結果）を、テストの一時フォルダの JSON に持つ。最初の状態は `fixtures/gh/` から読む。書き込みは状態を変え、読み取りは状態から答え、呼ばれた引数をすべて記録する。知らない呼び方は、終了コード 1 で終わって記録に残す（黙って成功しない） |
| 偽の `claude` | `test/integration/fakes/claude.ts` | 引数・環境変数の名前・標準入力を記録し、`fixtures/claude/` の筋書きに従って動く。筋書きは、`--agent` の値と、同じテストの中で何回目の起動かで選ぶ。筋書きの動きは、`stream-json` を書いて終わる、worktree に commit を作る、費用の上限に当たる、拒否で止まる、壊れた出力契約を返す、返事をせずに待つ、`SIGTERM` を無視するコマンドを裏に残す |
| 偽の `launchctl`・`systemctl` | `test/integration/fakes/launchctl.ts`、`test/integration/fakes/systemctl.ts` | 引数を記録して終わる（`kickstart` に `-k` を付けないことを確かめる。[ADR-20261015-001][adr-read]） |
| git の記録の包み | `test/integration/fakes/git.ts` | 呼ばれた引数と作業フォルダを記録してから、本物の git に渡す。AI のセッションを起こした後の clone で、取り込みの fetch のほかに git を実行しないことを確かめる（[ADR-20261013-001][adr-workspace]） |

- テストの準備で、一時フォルダの `bin/` に、`gh` などの名前の起動の包みを作る。包みは、`fakes/` の TypeScript を読み込むだけの JavaScript にする（拡張子の無いファイルは TypeScript として読まれないため）。偽物も bash で書かない（[07-structure.md][struct] の決まり）
- origin は、テストの一時フォルダに作る bare のリポで、`fixtures/repos/` の雛形から作る。clone と push は、ファイルのパスの URL で行い、SSH と deploy key を使わない
- 秘密の置き場には、テストの中で作る使い捨ての値（意味の無い PAT の文字列と、印の署名の鍵）を置く。本物の秘密を前提データに入れない
- `HOME` と XDG の 4 つ（`XDG_CONFIG_HOME` `XDG_CACHE_HOME` `XDG_DATA_HOME` `XDG_STATE_HOME`）を、テストの一時フォルダに向ける。マシンの設定は、一時フォルダの `settings.json` のパスを入口の引数で渡す（「[導入と点検][design-setup]」の「定期の実行の登録」と同じ形）。テストが、人のホームと Claude Code の設定に書かないため

## 前提データ

| フォルダ | 中身 | 使うテスト |
| --- | --- | --- |
| `fixtures/gh/` | 偽の `gh` の最初の状態（筋書きごとの JSON） | 結合テスト |
| `fixtures/claude/` | 偽の `claude` の筋書きと、実機の試しで得た `stream-json` の記録 | 単体テスト（費用の数え方、出力契約の取り出し）、結合テスト |
| `fixtures/repos/` | 使い捨てのリポの雛形（ファイルの木） | 結合テスト |
| `fixtures/texts/` | Issue と PR の本文の例 | 単体テスト、結合テスト |
| `fixtures/quarantine/` | 検疫の見本（指示の文例、区切りを模した文、見えない文字。[TEXT-11][text]）。見つけるべき文と、見つけてはいけない文の組 | 単体テスト |
| `fixtures/leak/` | 漏えいの検査の見本（メールアドレス、機械上のパス、進行 Bot の設定と記録の内容の形。[IMPL-15][impl]）。見つけるべき文と、見つけてはいけない文の組 | 単体テスト |
| `fixtures/repo-layout/` | 構成の検査を確かめるファイルの木（単体テストが欠けた木、`plugin/` にテストがある木、例外の一覧に書いたファイルがある木） | 構成の検査 |

- 検疫の見本は `fixtures/quarantine/` に置く（UNDECIDED-22 を、ここで決めた）。STRUCT-4 と構成の図が、前提データを `fixtures/` に置くのに合わせる
- 鍵の形の見本は、ファイルに書かず、テストの中で部品をつないで作る。ファイルに書くと gitleaks が commit を止め、止めないためには gitleaks の許す一覧を広げることになるため
- メールアドレスの見本は、例のために予約されたドメイン（`example.com`、`example.invalid` など）で書く

## 回し方

- vitest の project を、単体テスト・結合テスト・構成の検査の 3 つに分ける。`pnpm test` で 3 つとも回し、CI は `pnpm test` を回す。`tsconfig.json` と `vitest.config.ts` は、`plugin/src/` と `test/` を対象にする（[STRUCT-4][struct]）
- 構成の検査は、進行 Bot のコードを書き始める PR で入れる（[ADR-20261017-003][adr-plugin] の確認の方法）。単体テストは、対象のコードと同じ PR で足す（構成の検査が、単体テストの無いファイルを止めるため）
- 評価は、`evals/` の中の入口（TypeScript）から回す。入口は、`plugin/` と評価のケースを一時フォルダに集めて Plugin を組み立て、`claude plugin eval` を呼ぶ（business-os の `pnpm eval:cases` と同じ考え方で、business-os のコードは import しない。「[レビューの Skill][review-skill]」、[ADR-20261016-003][adr-move] の決定 4）
- 評価のケース（PR の差分と、期待する指摘の一覧）は `evals/review/` に置く。入口は、結果で Plugin が読み込まれたかを確かめ、読み込まれていなければ不合格にする（Plugin が読み込まれずに素のモデルで走っても、指摘の数だけでは見分けにくいため）

## 単体テストを持たないファイル

構成の検査の「`plugin/src/` のファイルごとに単体テストがあること」の例外（型の定義だけのファイルなど）は、理由を添えた一覧に書いたものだけを認める（[STRUCT-4][struct]）。

- 置き場は `test/repo/no-unit-test.json`。1 件は、`path`（`plugin/src/` からの路）と `reason`（理由）を持つ
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
