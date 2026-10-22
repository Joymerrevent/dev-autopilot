# テスト

進行 Bot と Skill を確かめるテストの、種類・置く場所・偽物・前提データ・回し方を決める。
置く場所の大枠（単体テストは `plugin/` の外の `test/unit/` に `plugin/src/` と同じ形のパスで置く、前提データは `fixtures/`）は STRUCT-4 と [ADR-20261017-003][adr-plugin] の決定 1 が正本で、基本設計は、STRUCT-4 と ADR-20261017-003 が基本設計に残した単体テストと結合テストの分け方と、偽物と前提データの形を決める。

## テストの種類

![テストの種類][fig-kinds]

| 種類 | 置く場所 | 確かめるもの | 本物を使うもの | 回すとき |
| --- | --- | --- | --- | --- |
| 単体テスト | `test/unit/`（`plugin/src/a/b.ts` の単体テストは `test/unit/a/b.test.ts`。[STRUCT-4][struct]） | 1 つのモジュールが、渡された値から結果を決める処理（次の工程の表、収束とマージの判定、印の署名と読み取り、指摘の ID、設定と出力契約の確かめ、安全設定と環境変数の組み立て、費用の数え方、検疫と漏えいの検査） | 無し。GitHub・git・`claude`・時刻・プロセスの操作（シグナルを送る、子孫・親・プロセスグループ・生まれた時刻を読む、`lsof`）・ファイルの情報（持ち主・権限・リンクか）は、引数で渡す偽物にする。ファイルは、テストごとの一時フォルダにだけ書く | CI、手元の `pnpm test` |
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
| 単体テスト | 1 つのモジュールの中で、入力の値から結果が決まるもの | 次の工程の表の各行、収束とマージの判定、印の署名、指摘の ID の計算（[ADR-20261019-001][adr-repeat]）、費用の数え方と試しのセッションの判定（実機の試しで得た `stream-json` の記録を前提データにする。[ADR-20261018-001][adr-usage]）、1 日の区切りや CI を待つ時間のような時刻で決まる判定、記録を失ったとみなす判定（記録の目印のファイルを作った時刻と、着手した Issue の最新の `intake-final` の印の時刻と push 以上の人が最後に `needs-human` を外した時刻の遅い方との比べ。「[AI のセッション][design-ai]」の「起動の前の確かめ」）、起動の前の確かめの順 9（worktree の中の `CLAUDE.local.md` は `local-settings` の `needs-human`、作業フォルダの上のフォルダの `CLAUDE.local.md` は Issue に `needs-human` を付けずに実行を終え、点検のマシンの組を fail にする。「[AI のセッション][design-ai]」の「起動の前の確かめ」）、止める相手の決まり（起動の時刻以後に生まれ、親をたどって起動より前のプロセスを通らないものだけ。「[AI のセッション][design-ai]」の「止める手順」）、レビューのコメントの組み立てと読み戻し（指摘の区切りの行、60,000 文字を超えたら 🟢 の失敗シナリオと推奨から省くこと、🟢 をすべて省いても超えるなら `review-too-long`。「[AI のセッション][design-ai]」の「レビューのコメントの形」）、点検の ruleset の比べ方（`enforcement`・対象・規則が表より守りの弱い向きなら fail、bypass にリポの役・deploy key・`push.rulesetBypassApps` の外のアプリ・進行 Bot のアプリがあれば fail、`push.rulesetBypassApps` のアプリは deploy key の push 先を限る 2 つの ruleset でだけ許す、読めない欄は fail。「[導入と点検][design-setup]」の「ruleset の形と確かめ方」、FACT-65、FACT-67）。ADR の確認方法が単体テストとしたもの：止める手順の順番（[ADR-20261018-001][adr-usage]）、短い一時フォルダの根の持ち主と権限の確かめ（[ADR-20261017-006][adr-tmp]）、計画の工程の中の順番と、計画を投稿しない条件（[ADR-20261018-002][adr-redo]、[ADR-20261018-004][adr-planrev]、[ADR-20261019-003][adr-replan]、[ADR-20261019-004][adr-policy]）、見落としの軸の結果を印に入れないこと（[ADR-20261019-002][adr-gaps]）、`review` Skill の本文の置き換えと、置き換えの書き方が残ったときにセッションを起こさないこと（[ADR-20261021-001][adr-review-skill]）、書かずに試す間の受け入れの相手と選定の結果、Issue の累計と失敗の回数から書かずに試した回を除くこと、`run.dryRun` と `run.stage` の組の確かめ（[ADR-20261021-002][adr-dry]）、承認のやり直しの条件と、最新の `intake-final` の印だけを読むこと、push 未満の人の作業指示コメントを読まずに止めないこと、作業指示コメントが 2 つ以上なら承認の確定・着手・対応ループで止めること（[ADR-20261021-003][adr-reapprove]）、リポの設定の `push.blockedPaths` が空でも、IMPL-15 が名指しするパスに触れる diff を push せず `needs-human` にすること、`push.blockedPaths` に足したパスに触れる diff も push せず `needs-human` にすること、計画が変える見込みのパスとの照らしと作業 AI の固定の指示文が、IMPL-15 が名指しするパスと `push.blockedPaths` を合わせた一覧を使うこと、プログラムに固定したパスの一覧が、IMPL-15 が名指しするパスと同じであること（テストの期待値に IMPL-15 の一覧を書く）（[ADR-20261021-005][adr-blocked]）、印の署名の鍵の一覧の先頭の鍵で書き、一覧のどれかの鍵で確かめ、一覧に無い鍵の印は署名の条件を満たさないこと、一覧が空なら点検が fail になること、先頭でない鍵でだけ確かめられた印が残る Issue と PR を warn に出し、残りが無くなったら前の鍵を外してよいことを warn に出すこと、`marker-invalid` の止めた理由の印に、無効だった印のコメントの ID が入り、署名の対象に入ること、数えない条件の組（push 以上の人が外したか、外す前の止めた理由の印に挙がっているか、外した後に編集されていないか）ごとに、止めるか数えないかが決まること、数えない印を無いものとして読み、`review` の印の `round` の欠けでは止めること（[ADR-20261021-004][adr-marker-keys]） |
| 結合テスト | 2 つ以上のモジュールをまたぎ、プロセス・ファイル・実行の回数をまたいで決まるもの | 1 回の実行で Issue ごとに進める重い工程が 1 つまでであること、2 回の実行で待ちのコメントが 1 件であること、lock、片付け、AI のセッションを起こした後の clone で git を実行しないこと（[ADR-20261013-001][adr-workspace]）、止める手順で残ったコマンドを止めること、書かずに試す実行を 2 回通して 2 回目が別の Issue を受け入れること、`run.dryRun` を `false` にした最初の実行が受け入れの印の無い Issue を受け入れ直すこと（[ADR-20261021-002][adr-dry]）、`brief-changed` で止まった Issue を、文を直して `Todo` に戻し、`agent-ready` を付け直すと、新しい `intake-final` の印を書いて計画し直すこと（[ADR-20261021-003][adr-reapprove]）、計画の印があり結びつく PR の無い Issue で、作業ブランチが bare の origin にあれば、実装せずに `needs-human`（`branch-without-pr`）を付け、作業ブランチの無い別の Issue では実装に進むこと（push の後、PR を作る前に切れた実行の続き。「[状態遷移と判定][design-flow]」の「次の工程」の順 13）、記録の目印のファイルの時刻が、着手した Issue の最新の `intake-final` の印より後なら、その Issue では AI のセッションを起動せずに `needs-human`（`records-missing`）を付け、同じ筋書きの、目印のファイルより後に `intake-final` の印を書いた別の Issue では起動すること（[ADR-20261015-006][adr-flow]）、人の `check` が秘密のファイルを読まないこと（下の「偽物」の Node の権限の仕組み。[ADR-20261017-004][adr-human]）、`push.blockedPaths` に `.github/**` が無いリポの設定で、作業 AI が `.github/workflows/` に足した diff を push しないこと（[ADR-20261021-005][adr-blocked]）、印の署名の鍵を替えた後の最初の実行で、前の鍵の印のある着手の前の Issue と作業中の PR が止まらず、新しい印を新しい鍵で書くこと、前の鍵を一覧から外すと、前の鍵の印が残る Issue が `marker-invalid` で止まり、push 以上の人が `needs-human` を外すと続くこと、進行 Bot のコメントを編集した Issue が止まり、push 以上の人が外すと続き、外した後にもう一度編集すると、また止まること（[ADR-20261021-004][adr-marker-keys]） |

- `accepted` の ADR の確認方法が単体テストか結合テストとしたものは、上の表の例に無いものも含めて、すべてテストに書く。単体テストか結合テストかは、ADR の書き方に従う
- ADR の確認方法を確かめるテストは、テストの名前（vitest の `it` に渡す名前）の先頭に、ADR の完全な番号（`ADR-20261015-004` の形）を入れる。1 つのテストが 2 つ以上の ADR を確かめるときは、番号を並べる。ADR の確認方法の項目とテストの名前を照らして、漏れを数えられるようにするため
- 迷ったら単体テストに書き、結合テストは流れの確かめに絞る。結合テストは時間がかかり、落ちたときに原因の場所が分かりにくいため
- 結合テストでは時刻を差し替えない。進行 Bot のプログラムに、テストのためだけの切り替え（時刻を変える環境変数など）を持たないため。時刻で決まる判定は、単体テストで確かめる

## 偽物

![結合テストの組み立て][fig-integration]

結合テストは、テストごとに一時フォルダを作り、偽物を `PATH` の先頭に置いて、進行 Bot の入口を子プロセスで起こす。進行 Bot のプログラムは本番と同じものを使い、テストのためだけの切り替えを持たない。偽物は、テストの中で一度に呼ばれうるものをすべて答えられるように、筋書きごとの最初の状態を前提データに持つ。

| 偽物 | 置く場所 | 振る舞い |
| --- | --- | --- |
| 偽の `gh` | `test/integration/fakes/gh.ts` | GitHub の状態を、テストの一時フォルダの JSON に持つ。持つもの：Issue、PR、コメントとその編集の記録（`userContentEdits`）、ラベル、timeline のイベント、依存（blocked by）、マイルストーン、Project の欄と自動追加の状態、リポの変数、ruleset（base ブランチに効く規則（`GET /repos/{owner}/{repo}/rules/branches/{branch}`）も、持っている ruleset から答える）、コラボレータの権限、認証の主（`GET /user`）、アプリのトークンの引き換えとインストールの情報（権限と、入っているリポ）、SSH のホストの鍵（`GET /meta`。テストの bare の origin への ssh は偽の `ssh` が受けるので、決まった値を返す）、必須チェックの結果、ワークフローの実行とジョブ。最初の状態は `fixtures/gh/` から読む。書き込みは状態を変え、読み取りは状態から答え、呼ばれた引数をすべて記録する。PR の head と base、ブランチの先端、チェックの SHA は、JSON に持たず、呼ばれたときに bare の origin（下の箇条）から引く。`fixtures/gh/` の必須チェックの結果は、SHA でなく、ブランチ名と何回目の push か（例：`feat/issue-1` の 2 回目の push は緑）で書く。テストの中でできる commit の SHA は、時刻が入るので前もって書けないため。点検の書き込みの権限の試しの名前（変数 `DEV_AUTOPILOT_PROBE` とラベル `dev-autopilot-probe`）を作る呼び出しには、必ず 403 を返す（進行 Bot の認証で作れないことを模す。`setup` は点検用の名前を作らないので、呼ぶ認証で答えを分けない）。状態を使わずに答える呼び方：`gh --version`（決めた版）、`gh auth status`（筋書きごとに、人の `gh` のログインがあるか無いか）。GraphQL は操作の名前で見分け、名前の無い GraphQL は知らない呼び方にする（進行 Bot は、GraphQL の操作に必ず名前を付ける）。知らない呼び方は、終了コード 1 で終わって記録に残す（黙って成功しない）。アプリの投稿者の名前の答え方は、GitHub App に切り替える前の試し（「[導入と点検][design-setup]」の「GitHub App の権限」の切り替えの順 2）で取り方を決めた後に足す。それまでは、`run.dryRun` が `false` の筋書き（アプリのトークンで書くもの。点検が、投稿に使うアカウントが `safety.botAccounts` にあることを確かめる）を書けない |
| 偽の `claude` | `test/integration/fakes/claude.ts` | 引数・環境変数の名前・標準入力を記録し、`fixtures/claude/` の筋書きに従って動く。`--version` には決めた版を返す。筋書きは、`--agent` の値と、作業フォルダ（セッションの名前）で選ぶ。同時に起こす 3 つの軸の起動の順は毎回変わるので、何回目の起動かでは選ばない。筋書きの動きは、`stream-json` を書いて終わる、worktree に commit を作る（本物の git を絶対パスで呼び、下の git の記録の包みを通さない）、費用の上限に当たる、拒否で止まる、壊れた出力契約を返す、返事をせずに待つ、`SIGTERM` を無視するコマンドを裏に残す、試しのセッションの結果を書く |
| 偽の `ssh` | `test/integration/fakes/ssh.ts` | 進行 Bot が `GIT_SSH_COMMAND` で git に渡す `ssh` を差し替える。`GIT_SSH_COMMAND` は `ssh` を名前で呼ぶ（「[設定][design-settings]」の「秘密の置き場のファイル」）ので、`PATH` の先頭の偽物が呼ばれる。付いたオプションを記録し、`git-upload-pack` と `git-receive-pack` を、テストの一時フォルダの bare のリポ（作業フォルダを持たない、受け取り専用のリポ）に対して手元で動かす。進行 Bot の clone の URL は本番と同じ形のままで、テストのためだけの切り替えを持たない。pack でやりとりするので、objects をハードリンクで共有しない（点検のリンク数の確かめが通る。FACT-19） |
| 偽の `launchctl`・`systemctl` | `test/integration/fakes/launchctl.ts`、`test/integration/fakes/systemctl.ts` | 引数を記録する（`kickstart` に `-k` を付けないことを確かめる。[ADR-20261015-001][adr-read]）。`kickstart` と `start` を受けたら、登録した定義と同じ引数と環境変数（定義の `PATH`・`LANG`・`safety.env.values`）に、テストの `HOME` と XDG の 4 つを足して、進行 Bot の入口を子プロセスで起こす（人の `check` が結果のファイルを待つ流れを確かめるため。`HOME` が無いと、Node は OS のユーザーのホームを使い、人のホームに書くため） |
| git の記録の包み（[ADR-20261013-001][adr-workspace] の「偽の git」） | `test/integration/fakes/git.ts` | 呼ばれた引数と作業フォルダを記録してから、本物の git に渡す。AI のセッションを起こした後の clone で、取り込みの fetch のほかに git を実行しないことを確かめる（[ADR-20261013-001][adr-workspace]） |

- テストの準備で、一時フォルダの `bin/` に、`gh` などの名前の起動の包み（偽物を呼ぶための小さな起動用のファイル）を作る。包みは、`fakes/` の TypeScript を読み込むだけの JavaScript にする（拡張子の無いファイルは TypeScript として読まれないため）。偽物も bash で書かない（[07-structure.md][struct] の決まり）
- origin は、テストの一時フォルダに作る bare のリポで、`fixtures/repos/` の雛形から作る。clone と push は、上の偽の `ssh` を通して行う。deploy key は、テストの中で作る使い捨ての鍵にする
- 秘密の置き場には、テストの中で作る使い捨ての値（意味の無い PAT の文字列、テストの中で作るアプリの秘密鍵、印の署名の鍵）を置く。本物の秘密を前提データに入れない。印の署名の鍵の一覧は、本番と同じ形（1 行に 1 つ、32 バイトの乱数を base64 で書いたもの。「[状態遷移と判定][design-flow]」の「印の署名」）で作る。点検が鍵の形を確かめるため。鍵を替える筋書きでは、2 つの鍵を並べる
- `fixtures/gh/` の印は、`sig` の欄を空にした雛形で書く。テストの準備で、テストの回の印の署名の鍵の一覧のうち筋書きが決めた行の鍵（鍵を替えた後の筋書きでは、前の鍵の行）と、`plugin/src/` の署名の関数で `sig` を埋める。前もって署名した値は、テストごとに作る鍵では通らないため。印のコメントの投稿者、リポの設定の `safety.botAccounts`、偽の `gh` の認証の主（`GET /user`）は、結合テストの補助の 1 か所の定数から作る。3 つの前提データが食い違うと、進行 Bot が印を読まずに止まるため
- 短い一時フォルダの根は、テストごとに `/tmp` の直下に短い名前のフォルダ（`/tmp/dapt-<英小文字と数字の 4 文字>`）を権限 700 で作って、マシンの設定の `sessionTmp.root` に書き、テストの後に消す。本物の根（`/tmp/dap`）を使わないのは、同じ Mac で動く本物の進行 Bot とテストが、互いのフォルダを残骸として fail にしないため。実体のパス（`/private/tmp/dapt-…/<ID>/claude-<uid>`）が 44 バイトに収まる長さにする
- テストの準備で、マシンの中のフォルダ（置き場・秘密の置き場・安全設定の置き場・記録の置き場）を `setup` のマシンの準備と同じ中身と権限で作り（「[導入と点検][design-setup]」の「マシンの準備」。記録の目印のファイルの時刻は下の箇条）、プログラムの写しと manifest、定期の実行の定義のファイル（一時フォルダの `HOME` の下）を作る。毎回の実行の最初の点検が、作業に入る前に fail にしないため
- 記録の目印のファイル（`records-created.json`）の作った時刻は、前提データのどのコメントとイベントの時刻よりも前にする。前提データの時刻は、テストの日より前の決まった日時で書く。本番の 2 つの決まりで止めないため（「[AI のセッション][design-ai]」の「起動の前の確かめ」）：目印のファイルを作った日が今日なら、その日は AI のセッションを起動しない。目印のファイルの時刻が、着手した Issue の最新の `intake-final` の印の時刻と、push 以上の人が最後に `needs-human` を外した時刻の遅い方より後なら、記録を失ったとみなして止める。記録を失ったときを確かめる筋書きでは、目印のファイルの時刻を、記録を失ったとみなす Issue の印とイベントより後で、テストの日より前にする
- 途中から始める筋書き（印のある Issue と PR を前提データに置くもの）で、Issue の累計を確かめるとき（費用の上限に当たる筋書きなど）は、同じ Issue の `sessions.jsonl` の行を前提データに置く。始まりの行と終わりの行を組で置く。終わりの行の無いセッションは、前の実行が残したセッションとして扱われるため
- 前の試しのセッションの pass の記録は、テストの準備で `plugin/src/` のハッシュの関数を呼んで作り、試しを省く条件（「[導入と点検][design-setup]」の「組と回数」）に当たる値（安全設定と環境変数の名前と `sessionTmp.*` のハッシュ、偽の `claude` が返す版、写しの版、テストを回すマシンの OS の版）で書く。試しのセッションを確かめる筋書きでは、前の記録を置かない
- `HOME` と XDG の 4 つ（`XDG_CONFIG_HOME` `XDG_CACHE_HOME` `XDG_DATA_HOME` `XDG_STATE_HOME`）を、テストの一時フォルダに向ける。テストが、人のホームと Claude Code の設定に書かないため。マシンの設定（`settings.json`）は、一時フォルダの `HOME` の下の、OS ごとの決まった場所（「[設定][design-settings]」の「マシンの設定」の場所の表。macOS は `$HOME/Library/Application Support/dev-autopilot/<owner>/<repo>/`）に置き、そのパスを入口の引数で渡す（「[導入と点検][design-setup]」の「定期の実行の登録」と同じ形）。点検が、`repository.*` とマシンの設定のフォルダのパスが合うかを確かめるため
- 進行 Bot と人の Skill の処理は、決めた名前だけの環境変数（`HOME`・XDG の 4 つ・`PATH`・`LANG` と、筋書きが足す名前）で起こす。テストを回す人と CI の環境の `SSH_AUTH_SOCK`・`GH_TOKEN`・`GITHUB_TOKEN`・`ANTHROPIC_API_KEY`・`CLAUDE_CODE_OAUTH_TOKEN` を渡さない。`PATH` は、一時フォルダの `bin/` と `/usr/bin:/bin:/usr/sbin:/sbin` だけにする。Node は、テストを回している `node` の実体へのリンクを `bin/` に置いて呼ぶ。Node のフォルダには、本物の `gh` と `claude` が並ぶことがあるため（Homebrew など）。テストの準備で、`PATH` のどのフォルダにも、偽物のほかの `gh` と `claude` が無いことを確かめ、あれば止める。偽物が外れたときに、本物の GitHub と `claude` へ流れないため（`PATH` の順に頼らない 2 つ目の守り）
- 前提データとマシンの設定の owner/repo は、試験用に決めた名前（本物の当てる先とも、dev-autopilot のリポとも違う名前）にする。テストの準備で、マシンの設定と `fixtures/` の owner/repo が試験用の名前でなければ止める。偽の `ssh` が外れても、本物のリポに push しないため。dev-autopilot のリポの名前にしないのは、点検が作業に入る前に止めるため（「[導入と点検][design-setup]」の「setup の流れ」の順 1）
- 人の `check` と `/dev-autopilot` の処理は、Node の権限の仕組み（`node --permission`）で起こす。読み込みを許すパス（`--allow-fs-read`）と書き込みを許すパス（`--allow-fs-write`）は、処理が読み書きするものだけにし、秘密の置き場を入れない。秘密の置き場はマシンの設定のフォルダの中にあるので、マシンの設定のフォルダはフォルダごと許さず、`settings.json` と記録の置き場だけを許す（「[設定][design-settings]」の「マシンの中のフォルダ」）。秘密のファイルを読めば `ERR_ACCESS_DENIED` で落ちる（FACT-64）。テストは、終了コードと、出力に `ERR_ACCESS_DENIED` が無いことを確かめる。処理が読み込みの失敗を受け止めて続けても、見逃さないため。偽の `launchctl` を起こすので `--allow-child-process` を付ける（Node は `SecurityWarning` を出す。FACT-64）。[ADR-20261017-004][adr-human] の確認方法の「`check` が人のセッションから秘密のファイルを読まない」を、進行 Bot のプログラムを変えずに確かめるため。子プロセス（偽の `launchctl` と、偽の `launchctl` が起こす進行 Bot）に権限の制限が引き継がれるかは確かめていない。引き継がれて進行 Bot が秘密を読めなければ、テストは落ちる（通ってしまう側には倒れない）
- 結合テストは、共通の後始末で、偽物の知らない呼び方の記録が空であることと、`runs.jsonl` の終わり方（「[状態遷移と判定][design-flow]」の「手元の記録」）が筋書きの期待と同じであることを、テストごとに確かめる。進行 Bot が、偽物の答えられない呼び方か点検の fail で早く止まったときに、「〜しない」を確かめるテストが、何も確かめずに通らないため
- 「〜しない」を確かめる結合テスト（起動しない、書かない、git を実行しない）は、同じ筋書きの中に、同じ経路を通ったことを示す肯定の対照を組にする（例：git の記録の包みの記録に取り込みの `git fetch` があること、検疫に当たらない別の Issue では受付 AI を起動したこと）

## 前提データ

| フォルダ | 中身 | 使うテスト |
| --- | --- | --- |
| `fixtures/gh/` | 偽の `gh` の最初の状態（筋書きごとの JSON） | 結合テスト |
| `fixtures/claude/` | 偽の `claude` の筋書きと、実機の試しで得た `stream-json` の記録 | 単体テスト（費用の数え方、出力契約の取り出し、試しのセッションの判定）、結合テスト |
| `fixtures/repos/` | 使い捨てのリポの雛形（ファイルの木） | 結合テスト |
| `fixtures/texts/` | Issue と PR の本文の例 | 単体テスト、結合テスト |
| `fixtures/quarantine/` | 検疫の見本（指示の文例、区切りを模した文、見えない文字。[TEXT-11][text]）。見つけるべき文と、見つけてはいけない文の組。見つけてはいけない文には、ふつうの計画と指摘の見本（「〜を実行する」の形の手順、コマンドとファイルのパスを書いた指摘など）も入れる。URL は、通す URL と止める URL の境目を組にする（下の箇条） | 単体テスト |
| `fixtures/leak/` | 漏えいの検査の見本（メールアドレス、機械上のパス、進行 Bot の設定と記録の内容の形。[IMPL-15][impl]）。見つけるべき文と、見つけてはいけない文の組 | 単体テスト |
| `fixtures/repo-layout/` | 構成の検査を確かめるファイルの木（単体テストが欠けた木、`plugin/` にテストがある木、例外の一覧に書いたファイルがある木、`plugin/src/` で npm のパッケージを import した木、実在しないファイルを例外の一覧に書いた木、理由が空の行がある木） | 構成の検査 |

- 検疫の見本は `fixtures/quarantine/` に置く（UNDECIDED-22 を、ここで決めた）。STRUCT-4 と構成の図が、前提データを `fixtures/` に置くのに合わせる
- 見つけてはいけない文に、ふつうの計画と指摘の見本を入れるのは、AI が書いた計画と指摘も、渡す前に外の文と同じ検疫に通す（[IMPL-10][impl]、[TEXT-11][text]）ため。検証の手順を書いた計画と、コマンドを写した指摘で止まる形を、テストで先に見つける
- 検疫の見本の URL は、通す URL と止める URL の境目を組にする（[ADR-20261022-001][adr-url] の確認方法、「[AI のセッション][design-ai]」の「検疫が探す URL」）
  - 通す側：当てる先のリポの Issue・PR・ファイルの URL、`#` と `?` の付いたもの、owner と名前の大文字と小文字が違うもの、Markdown のリンクの先と自動リンクに書いたもの
  - 止める側：ほかのリポの GitHub の URL、外のドメインの URL、`www.` で始まる文字列、名前の後に別の文字が続く URL（`https://github.com/<owner>/<name>.evil.example/` など）、`.` か `..` の区切りでほかのリポを指す URL、崩した形（`%` の符号、`\`、全角の文字、見えない文字で書いたもの）
  - 当てる先のリポの GitHub の URL の中に見えない文字か指示の形の語がある文は、ほかの検査で止まる見本にする
  - 当てる先のリポの owner と名前は、上の「偽物」と同じ試験用の名前で書き、単体テストが検疫のモジュールに引数で渡す
- 鍵の形の見本は、ファイルに書かず、テストの中で部品をつないで作る。ファイルに書くと gitleaks が commit を止め、止めないためには gitleaks の許す一覧を広げることになるため
- メールアドレスの見本は、例のために予約されたドメインのうち、`.gitleaks.toml` が許す 4 つ（`example.com`・`example.org`・`example.net`・`example.invalid`）で書く。ほかのドメインは gitleaks が commit を止める
- 実機の `stream-json` の記録と、business-os の PR の差分と本文を前提データに置くときは、人のホームのパス・ユーザー名・`session_id`・業務の固有名詞を置き換えてから置く。置く前に、gitleaks と目で確かめる（固有名詞は gitleaks が見つけられないため）

## 回し方

- vitest の project を、単体テスト・結合テスト・構成の検査の 3 つに分ける。`pnpm test` で 3 つとも回す。CI は、Linux のランナーで単体テストと構成の検査を、macOS のランナーで結合テストを回す（上の「テストの種類」）。`tsconfig.json` は、`plugin/src/`・`plugin/skills/**/scripts/`・`test/`・`evals/` の TypeScript を型の検査の対象にする。`vitest.config.ts` は `test/` を対象にする（[STRUCT-4][struct]）
- 構成の検査の「ファイルごとに単体テストがあること」は、`plugin/src/` だけに掛ける。`plugin/skills/**/scripts/` の検査の単体テストは `test/unit/skills/` に置き、`evals/` は評価の入口なので単体テストを求めない
- 構成の検査は、進行 Bot のコードを書き始める PR で入れる（[ADR-20261017-003][adr-plugin] の確認方法）。単体テストは、対象のコードと同じ PR で足す（構成の検査が、単体テストの無いファイルを止めるため）
- 評価は、`evals/` の中の入口（TypeScript）から回す。評価する `review` Skill は、business-os の `change-review` の Skill を組み直した版（v2。「[レビューの Skill][review-skill]」）。入口は、`plugin/` と評価のケースを一時フォルダに集めて Plugin を組み立て、`claude plugin eval` を呼ぶ（business-os の `pnpm eval:cases` と同じ考え方で、business-os のコードは import しない。「[レビューの Skill][review-skill]」、[ADR-20261016-003][adr-move] の決定 4）
- 評価のケース（PR の差分と、期待する指摘の一覧）は `evals/review/` に置く。入口は、結果で Plugin が読み込まれたかを確かめ、読み込まれていなければ不合格にする（Plugin が読み込まれずに素のモデルで走っても、指摘の数だけでは見分けにくいため）
- 評価の入口は、`review` Skill を、レビュー工程（段階 6）で進行 Bot が使うのと同じ無人の入り口の引数（[REVIEW-5][review-skill]）で呼ぶ。引数を渡さない人の入り口で数えると、3 つの軸をサブエージェントで見る形になり、レビュー工程（段階 6）の経路と違う形の数になるため。ケースごとに、基点（PR の base）、仕様の在りか、🟢 の起票先、予算（ターンと時間）、見る軸、品質ゲートを回すかを決めて渡す。見る軸は、進行 Bot が軸ごとに別のセッションを起こすのと同じく、1 回の呼び出しに 1 つの軸にする。🟢 の起票先は進行 Bot と同じく Issue（起票は呼ぶ側が行う）、品質ゲートは回さないと渡す（「[AI のセッション][design-ai]」の「役ごとの要素」のレビュー AI の行）。評価の入口は 🟢 を起票せず、本物の GitHub に書かない

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
[adr-flow]: ../../adr/adr-20261015-006-run-stages-baseline-agent-field-and-local-records.md
[adr-human]: ../../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-marker-keys]: ../../adr/adr-20261021-004-marker-key-list-and-resume-after-invalid-markers.md
[design-settings]: ./01-settings.md
[design-flow]: ./02-flow-and-judgment.md
[design-ai]: ./03-ai-sessions.md
[adr-blocked]: ../../adr/adr-20261021-005-required-push-blocked-paths-in-program.md
[adr-url]: ../../adr/adr-20261022-001-quarantine-urls-outside-target-repo.md
