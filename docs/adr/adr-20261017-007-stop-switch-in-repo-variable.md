---
status: accepted
created: 2026-10-17
updated: 2026-10-20
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 全体を止める合図はリポの変数に置いて実行の途中でも読み、書かずに試す合図はリポの設定に残し、リポの設定は base ブランチの先端から読む

> **Amended by [ADR-20261020-002][adr-app]（2026-10-20）**: 決定 1 の「進行ボットの PAT には Variables の読み取りだけを与える」は、GitHub App に切り替えた後はアプリの権限（Variables の読み取りだけ）と読む。止める合図の置き場と読み方は据え置く。

要件の UNDECIDED-38（[決めていないこと][undecided]の「安全と設定」の「全体を止める合図と、書かずに試す合図の置き場」）を決める。

改める決定は次の 1 つで、ほかの決定は据え置く。

- [ADR-20261015-005][adr-settings] の決定 2 の割り振りのうち、止める合図（`enabled`）をリポの設定から外して当てる先のリポの変数に置き（決定 1）、base ブランチの名前をリポの設定からマシンの設定に移す（決定 4）。
  ほかの項目の割り振りと、項目ごとに置き場を 1 つにすることは据え置く

## 背景と課題（Context and Problem Statement）

- 止める合図（`enabled: false`）と書かずに試す合図（`dryRun: true`）は、設定ファイルに持ち、進行ボットが毎回の実行の最初に読む（[01-issue-flow.md][flow]の「運用の規則」）。Issue ごとに止める合図は、Project の Agent 欄の `paused`
- [ADR-20261015-005][adr-settings] の割り振りでは、どちらもリポの設定（当てる先のリポの `.claude/dev-autopilot.json`。commit する）になる。リポの設定は PR で変え、base ブランチは ruleset で PR が必須
- そのため、AI の振る舞いがおかしいと気づいて急いで止めたいときに、PR を作ってマージするまで止まらない。止める合図は実行の最初にしか読まないので、動いている実行（AI のセッションは 1 回 45 分まで。IMPL-13）も止まらない
- 進行ボットがどの版のリポの設定を読むか（人の作業コピー、既定のブランチ、base ブランチ）が書かれていない。当てる先の business-os は、既定のブランチ（`main`）と PR を向ける base ブランチ（`develop`）が違う
- 2026-10-17 に GitHub の公式文書を読んだ（補足情報）。リポの変数（Actions の変数）は、fine-grained PAT の「Variables」の読み取りの権限で読める。変えられるのは、org のリポなら write 以上の人、個人のリポならコラボレータで、GitHub の画面から PR なしで変えられる
- 同じ日に、試験用のリポで確かめた（補足情報）。Actions を止めたリポでも、変数を読めて変えられた。org のリポでは、変数を変えた人と日時が org の監査ログに残った。
  「Variables」の読み取りだけを与えた fine-grained PAT は、変数を読めて、作成・更新・削除は 403 で拒否された

課題：人が急いで止めたいときに、PR を待たず、動いている実行も含めて止まる合図をどこに置くか。書かずに試す合図をどこに置くか。進行ボットはどの版のリポの設定を読むか。

## 判断の決め手（Decision Drivers）

- 人が、PR を待たず、専用マシンに入らずに、全体を止められること（スマートフォンの GitHub の画面からも）
- 止める合図が読めないときは、動かない側に倒れること（フェイルセーフ。SAFE-4）
- 止める合図を、AI のセッションと外の人が変えられないこと。動かす向きに戻せる人が、リポの設定を変えられる人より広がらないこと
- 安全の境界を緩める向きの変更（実際に GitHub に書き始める）は、PR で記録に残ること（[ADR-20261015-005][adr-settings] の決定 1 と同じ考え方）
- 進行ボットが読む設定の版が、人の手元の状態に左右されず、決まること

## 検討した選択肢（Considered Options）

問い 1：全体を止める合図の置き場

- 案 1-A：当てる先のリポの変数（Actions の変数）に置き、進行ボットが実行の途中でも読む
- 案 1-B：マシンの設定（手元のファイル）に置く
- 案 1-C：今のまま、リポの設定に置く。急ぐときは OS の定期の実行の登録を外す
- 案 1-D：止めるための Issue のラベルに置く
- 案 1-E：リポの environment の変数に置く（変えられるのは admin だけ）

問い 2：書かずに試す合図の置き場

- 案 2-A：今のまま、リポの設定に置く
- 案 2-B：止める合図と同じリポの変数に、3 つの値（動かす・書かずに試す・止める）で置く

問い 3：進行ボットが読むリポの設定の版

- 案 3-A：進行ボットの clone で fetch した base ブランチの先端を読む。base ブランチの名前はマシンの設定に移す
- 案 3-B：既定のブランチのリポの設定から base ブランチの名前を知り、base ブランチの先端を読む
- 案 3-C：既定のブランチの先端を読む
- 案 3-D：人の作業コピー（当てる先のリポを開いているフォルダ）を読む

## 決定（Decision Outcome）

採用した選択肢:「案 1-A」「案 2-A」「案 3-A」。
理由: 止める合図は PR も専用マシンへの接続も要らずに変えられ、読めなければ止まる。動かす向きに戻せる人は、Agent 欄を戻せる人と同じ範囲（push 以上）に収まる。
書き込みを始める向きの変更は PR に残る。進行ボットが読む設定は、レビューを通った base ブランチの先端に決まり、最初の導入でも既定のブランチへのリリースを待たない。

### 決定 1：全体を止める合図は、当てる先のリポの変数に置く

- 合図は、当てる先のリポの変数（Actions の変数。名前は構造仕様で決める。例：`DEV_AUTOPILOT_RUN`）に置く。値が `run` のときだけ動き、ほかの値・変数が無い・読めない（API の失敗、権限の不足）ときは止まる
- `setup` の GitHub の準備（人の `gh`。[ADR-20261017-004][adr-human] の決定 2）が、値を `stop` にして作る。人は `check` が pass になった後に `run` に変える（動き出すのは人が決めたときだけ）
- 人は、GitHub の画面（スマートフォンを含む）か人の `gh` で変える。PR は要らない。変えられるのは、org のリポなら write 以上の人、個人のリポならコラボレータ。Agent 欄を戻せる人を push 以上の人に限る運用の前提（[ADR-20261015-004][adr-markers]）と同じ範囲
- 進行ボットの PAT には、変数の読み取りの権限（Variables: read）だけを与え、書き込みは与えない。AI のセッションは GitHub に書けない（SAFE-1）
- リポの設定の `enabled` は無くす。止める合図を 1 か所にする（[ADR-20261015-005][adr-settings] の決定 2 の「項目ごとに置き場を 1 つ」）
- Issue ごとに止める合図（Agent 欄の `paused`）は今のまま

### 決定 2：進行ボットは、止める合図を実行の途中でも読む

- 進行ボットは、実行の最初、AI のセッションを起動する前、GitHub に書き込む前に合図を読む。`run` を読めなければ、AI のセッションを起動せず、書き込まない
- AI のセッションが動いている間も、一定の間隔で合図を読む（間隔は基本設計で決める）。`run` 以外の値を読んだら、AI のセッションを時間の上限と同じやり方（プロセスグループごと。IMPL-13）で止める。
  間の読み取りの失敗では止めない（GitHub に書き込む前に、もう一度 `run` を読めなければ書かないため）
- 合図で止めた実行は、手元の記録に残し、lock を外して終わる。その回は実行の失敗に数えない（人が止めたので、FLOW-29 の 3 回に入れない）。消費は IMPL-14 のとおり記録に残す（途中で止めた回の数え方は UNDECIDED-47）
- 次の実行は、合図が `run` に戻った後、GitHub の状態から続ける（「運用の規則」の復旧）

### 決定 3：書かずに試す合図は、リポの設定に残す

- `dryRun` は、今のままリポの設定に置き、PR で変える。`dryRun` を切る（実際に GitHub に書き始める）のは安全の境界を緩める向きの変更なので、PR で記録に残す
- 書き込みを急いで止めたいときは、決定 1 の止める合図を使う

### 決定 4：進行ボットは、base ブランチの先端のリポの設定を読む

- 進行ボットは、毎回の実行の最初に、進行ボットの clone（`<置き場>/controller/`）で fetch した base ブランチの先端から、リポの設定を読む。1 回の実行は 1 つの版の設定で動き、途中で読み直さない（止める合図は除く）
- 人の作業コピー、AI の worktree、Plugin の写しの中のリポの設定は読まない（SAFE-2）
- base ブランチの名前は、リポの設定を読む前に要るので、リポの設定からマシンの設定に移す。`setup` のマシンの準備で書く。`check` は、origin に base ブランチがあること、base ブランチの先端にリポの設定があること、ruleset が base ブランチを守っていることを確かめる
- 設定の変更は、base ブランチにマージされた後の次の実行から効く。base ブランチの先端のリポの設定を読めない・形が違うときは、作業に入らず止まる（SAFE-4）

### 影響（Consequences）

- 良い点: 人は、PR を待たず、専用マシンに入らずに、スマートフォンからでも全体を止められる。動いている AI のセッションも止まる
- 良い点: 合図が無い・読めない・値が違うときは止まる。`setup` の直後も、人が `run` にするまで動かない
- 良い点: 書き込みを始める向きの変更（`dryRun` を切る）は PR に残る
- 良い点: 進行ボットが読む設定の版が、base ブランチの先端に決まる。最初の導入でも、既定のブランチへのリリースを待たない
- 悪い点: 止める合図はリポの設定と別の場所にあり、PR の記録には残らない。誰がいつ変えたかは、org のリポなら org の監査ログ（owner だけが見られる。180 日）に残るが、個人のリポでは変数の `updated_at` しか残らない
- 悪い点: 合図を読む API の呼び出しが、実行の途中にも増える
- 悪い点: base ブランチの名前が、マシンごとの設定になる。手元の Mac から専用マシンに移るときに、もう一度書く（`check` が確かめる）

### 確認方法（Confirmation）

- 構造仕様を書くときに、変数の名前と値、PAT の権限（Variables: read）、設定の表（`enabled` が無く、base ブランチがマシンの設定にある）が、決定のとおりかをレビューで確かめる
- 進行ボットの単体テストで、合図が `run` 以外・無い・読めないときに AI のセッションを起動せず書き込まないこと、AI のセッションの間に `stop` を読むと止めること、止めた回を実行の失敗に数えないことを検査する
- 段階 3（`setup` と `check` を作る段階）で、`setup` が変数を `stop` で作ること、進行ボットの PAT で変数を読めること、変数を変えられないことを実機で試す

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 1-A：リポの変数に置き、実行の途中でも読む

- 良い点: GitHub の画面から PR なしで変えられ、専用マシンに入らなくてよい。変えられる人は push 以上の人で、Agent 欄と同じ前提に収まる
- 良い点: 進行ボットの PAT は読み取りだけで済み、AI のセッションは GitHub に書けない
- 良い点: Actions を止めたリポでも読める（2026-10-17 に確かめた）。org のリポでは、誰が変えたかが org の監査ログに残る
- 悪い点: PR の記録に残らず、個人のリポでは誰が変えたかが残らない。実行の途中の API の呼び出しが増える

### 案 1-B：マシンの設定に置く

- 良い点: GitHub が止まっていても変えられる（ただし GitHub が止まっていれば、進行ボットも作業できない）
- 悪い点: 専用マシンでは、人がマシンに入らないと変えられない。手元から見えない

### 案 1-C：リポの設定のまま。急ぐときは定期の実行の登録を外す

- 良い点: 置き場が増えない
- 悪い点: 設定での止め方は PR を待つ。登録を外すのはマシンに入る操作で、動いている実行は別に止める必要がある

### 案 1-D：止めるための Issue のラベルに置く

- 良い点: GitHub の画面から変えられ、Issue のタイムラインに誰が変えたかが残る
- 悪い点: triage の権限でもラベルを付け外しできるので、動かす向きに戻せる人が push 以上の人より広がる（[ADR-20261015-004][adr-markers] の背景）。止めるための Issue を消したり閉じたりしたときの扱いが要る

### 案 1-E：environment の変数に置く

- 良い点: 変えられるのが admin だけに絞られる
- 悪い点: environment を作る準備が要り、Actions の環境の仕組みを止める合図のためだけに使う。push 以上の人が止められない

### 案 2-A：書かずに試す合図はリポの設定のまま

- 良い点: 実際に書き始める変更が PR に残り、レビューを通る
- 悪い点: 書かずに試す側へ戻すにも PR が要る（急ぐときは止める合図で代わりにする）

### 案 2-B：止める合図と同じ変数に 3 つの値で置く

- 良い点: 1 か所で、書かずに試す側へも急いで戻せる
- 悪い点: 実際に書き始める変更が PR を通らず、レビューされない

### 案 3-A：base ブランチの先端を読み、base ブランチの名前はマシンの設定に移す

- 良い点: レビューを通った最新の設定で動き、人の手元の状態に左右されない。最初の導入で、リポの設定を base ブランチに入れればすぐ動ける
- 悪い点: base ブランチの名前がマシンごとの設定になる

### 案 3-B：既定のブランチの設定から base を知り、base の先端を読む

- 良い点: base ブランチの名前をリポの設定に残せる
- 悪い点: 既定のブランチにリポの設定が入るまで動けない。business-os では `main` へのリリースを待つ。base ブランチを変えるときに 2 つのブランチの設定が食い違う

### 案 3-C：既定のブランチの先端を読む

- 良い点: 読む場所が 1 つで分かりやすい
- 悪い点: business-os では、設定の変更が `main` へのリリースまで効かない。作業は `develop` の上で進むので、設定とコードの版がずれる

### 案 3-D：人の作業コピーを読む

- 良い点: 手元で直した設定がすぐ効く
- 悪い点: 作業コピーはどのブランチでも、commit していない変更があってもよく、版が決まらない。専用マシンには作業コピーが無い

## 補足情報（More Information）

- 関連する要件：「運用の規則」の止める合図と書かずに試す合図（[01-issue-flow.md][flow]）、IMPL-13、IMPL-14（[02-implement-review-fix.md][impl]）、SAFE-1、SAFE-2、SAFE-4（[05-safety.md][safety]）、SETUP-1、SETUP-2（[06-setup-and-stages.md][stages]）、STRUCT-2（[07-structure.md][struct]）、
  FLOW-29、UNDECIDED-38、UNDECIDED-47（[09-undecided.md][undecided]）、[ADR-20261015-004][adr-markers]、[ADR-20261015-005][adr-settings]、[ADR-20261017-004][adr-human]
- 2026-10-17 に GitHub の公式文書を読んだ
  - [リポの変数の REST API][gh-rest-variables]：変数の読み取り（`GET /repos/{owner}/{repo}/actions/variables/{name}`）は、リポのコラボレータが行える
  - [fine-grained PAT に要る権限][gh-fg-perms]：変数の読み取りは「Variables」の read、作成・更新・削除は write
  - [変数の使い方][gh-variables]：org のリポの変数を作るには write の権限が要り、個人のリポではコラボレータであることが要る。リポの Settings の「Secrets and variables」の「Actions」から変えられる。environment の変数は、org のリポでは admin の権限が要る
- 2026-10-17 に、試験用のリポ（private、中身なし）で確かめた。メンテナの `gh`（OAuth のトークン）を使った
  - 変数を作り（`POST /repos/{owner}/{repo}/actions/variables`）、読めた。リポの Actions を止めても（`PUT /repos/{owner}/{repo}/actions/permissions` の `enabled=false`）、変数を読めて変えられた。試験の後に Actions の設定を元に戻し、変数を消した
  - org の監査ログ（GitHub の画面）に、`repo.update_actions_variable` のイベントが、変えた人と「via the API」と一緒に出た。変えた後の値は出ない。監査ログの API は GitHub Enterprise Cloud だけで、Team のプランでは 404 だった
- 公式文書（[org の監査ログの見方][gh-audit-review]、[org の監査ログのイベント][gh-audit]）：監査ログは過去 180 日のイベントを持ち、見られるのは org の owner だけ。リポの変数の作成・更新・削除は `repo.create_actions_variable`・`repo.update_actions_variable`・`repo.remove_actions_variable` のイベントになる
  - メンテナが作った fine-grained PAT（対象は試験用のリポだけ、権限は Variables の読み取りだけ、期限 1 日）で試した。変数の値と一覧を読めた。変数の更新・作成・削除は、どれも 403（`Resource not accessible by personal access token`）で拒否され、値は変わらなかった。対象に入れていない別のリポの変数の一覧も 403 だった
- 確かめていないこと：write の権限だけの人が変数を変えられるか（公式文書に書いてある。試すには別のアカウントが要る）
- 急いで止める手段の最後の手段（人がマシンで、定期の実行の登録を外し、動いている進行ボットを止める）は、今のまま残る
- 要件への反映（承認の後に、別の PR で行う）
  - 01-issue-flow.md：「運用の規則」の止める合図（リポの変数、途中でも読む、止めた回を数えない）と、書かずに試す合図（リポの設定のまま）
  - 02-implement-review-fix.md：IMPL-14 の表に、止める合図で止めた回を足す
  - 06-setup-and-stages.md：SETUP-1（GitHub の準備で変数を `stop` で作る。マシンの準備で base ブランチの名前を書く）、SETUP-2（変数を読めること、base ブランチとリポの設定と ruleset の確かめ）
  - 07-structure.md：STRUCT-2（`enabled` を無くし、base ブランチをマシンの設定に移す。リポの設定は base ブランチの先端から読む）
  - 09-undecided.md：UNDECIDED-38 を欠番にする（元の文を取り消し線で残す）
  - ADR-20261015-005：タイトルの下に、決定 2 を本 ADR で改めた旨の注記を足す

[flow]: ../requirements/01-issue-flow.md
[impl]: ../requirements/02-implement-review-fix.md
[safety]: ../requirements/05-safety.md
[stages]: ../requirements/06-setup-and-stages.md
[struct]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[adr-markers]: ./adr-20261015-004-signed-markers-and-actor-checks.md
[adr-settings]: ./adr-20261015-005-settings-values-and-machine-settings.md
[adr-human]: ./adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[gh-rest-variables]: https://docs.github.com/en/rest/actions/variables
[gh-fg-perms]: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
[gh-variables]: https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-variables
[gh-audit-review]: https://docs.github.com/en/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/reviewing-the-audit-log-for-your-organization
[gh-audit]: https://docs.github.com/en/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/audit-log-events-for-your-organization
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
