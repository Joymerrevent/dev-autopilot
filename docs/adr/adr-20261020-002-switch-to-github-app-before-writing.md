---
status: accepted
created: 2026-10-20
updated: 2026-10-24
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 進行 Bot は、GitHub に書き始める前に GitHub App に切り替え、自動化用のアカウントを分ける

> **Amended by [ADR-20261024-002][adr-20261024-002]（2026-10-24）**: 決定 1 の「受け入れ工程（段階 4）で `run.dryRun` を `false` にする（GitHub に書き始める）前に、GitHub App に切り替える」を、「進行 Bot が当てる先のリポに初めて書く前に切り替える」に改めた。書かずに試す間に PAT で読む文は、書かずに試すことを持たないので要らなくなった（決定 2）。決定 1 のほかの項目と、決定 2〜5 は据え置く。
>
> **［2026-10-24 追記］** 権限の表の「Project の Status と Agent 欄」は、[ADR-20261024-001][adr-20261024-001] の決定 5 の表示の欄（主な流れの枠・状態・止め方・先の依頼・安全）と読む。

ADR「自動化用の GitHub アカウントはいまは分けず、印を書いたと認めるアカウントを設定の一覧に持つ」（[ADR-20261020-001][adr-bots]）の決定 1 が「分ける要望が出たら、別の ADR で決める」とした論点を決める。あわせて、非公開のリポで CI のチェックの実行を読む手段（要件の UNDECIDED-57）を決める。

改める決定と、据え置く決定は次のとおり。

- [ADR-20261020-001][adr-bots]：決定 1（いまは分けない）を改める。決定 2 の「投稿に使うアカウントは PAT の持ち主」と、点検の「PAT の持ち主が一覧にある」を、アプリの投稿者の名前に読み替える。決定 2 の一覧の形、決定 3、決定 4 は据え置く
- [ADR-20261017-004][adr-human]：決定 2 の「進行 Bot の PAT」を、切り替えの後はアプリのトークンに読み替える。人の `setup` が人の `gh` のログインで GitHub の準備をすること、進行 Bot に管理の権限を持たせないことは据え置く
- [ADR-20261017-007][adr-stop]：決定 1 の「進行 Bot の PAT には Variables の読み取りだけを与える」を、切り替えの後はアプリの権限に読み替える。止める合図の置き場と読み方は据え置く
- [ADR-20261015-001][adr-read]：決定 2 の秘密の置き場に置くもののうち、PAT を、切り替えの後はアプリの秘密鍵にする。置き場と権限（700・600）は据え置く

## 背景と課題（Context and Problem Statement）

- 進行 Bot は、メンテナのアカウントで発行した fine-grained PAT で GitHub を読み書きする前提になっている（SAFE-15、SETUP-13、[ADR-20261017-004][adr-human]、[ADR-20261017-007][adr-stop]）。[ADR-20261020-001][adr-bots] で、自動化用のアカウントはいまは分けないと決めた
- fine-grained PAT には Checks の権限が無い。公開のリポのチェックの実行は権限なしで読めたが、非公開のリポでは 403 で読めなかった（FACT-52、FACT-54）。GitHub App は、リポの権限に Checks があり、チェックの実行の一覧は読み取りで足りる（GitHub の公式文書）
- GitHub App は、登録した人とは別の身元（`<アプリの名前>[bot]`）で投稿する。API を呼ぶトークンは、アプリの秘密鍵から作る、1 時間で切れるインストールのトークン
- アカウントを替えると、替える前のアカウントで書いた印は、投稿者の条件を満たさなくなる（[ADR-20261020-001][adr-bots]）。GitHub に印を書く前に替えれば、替える前のアカウントで書いた印は 1 つも無い
- 受け入れ工程（段階 4）の最初は、書かずに試す合図（`run.dryRun`）が `true` で、進行 Bot は GitHub に書かない（[段階の表][stages]）
- 2026-10-20 に、メンテナが「いまは PAT で進め、早い段階で GitHub App に切り替える。時期は GitHub に書き始める前」と決めた

課題：自動化用のアカウントをいつ、どの形で分けるか。非公開のリポのチェックの実行をどう読むか。

## 判断の決め手（Decision Drivers）

- 人と進行 Bot の投稿を、印に加えて投稿者でも見分けられること（監査。メンテナが PR を GitHub の approve でレビューできること）
- 非公開のリポに当てるときも、必須チェックの結果を読めること（IMPL-27）
- 切り替えで、作業中の Issue が止まったり、印の読み方の手当てが要ったりしないこと
- 長く持つ秘密を減らし、漏れたときの害を小さくすること（フェイルセーフ）
- 今の進み（PAT を前提にした要件と基本設計）を止めないこと

## 検討した選択肢（Considered Options）

- 案 A：GitHub に書き始める前（受け入れ工程（段階 4）で `run.dryRun` を `false` にする前）に GitHub App に切り替える。切り替えるまでは PAT で読むだけ
- 案 B：導入と点検の実装（段階 3）から GitHub App で作る。PAT は今の試しだけに使う
- 案 C：マージ判定工程（段階 8）の前に GitHub App に切り替える。切り替えるまでは PAT で書き、[ADR-20261020-001][adr-bots] の一覧で古い印も読む
- 案 D：分けない（[ADR-20261020-001][adr-bots] の決定 1 のまま）
- 案 E：GitHub App でなく、machine user（自動化用の人のアカウント）に分ける

## 決定（Decision Outcome）

採用した選択肢:「案 A」。
理由: 印を書く前に切り替えるので、替える前のアカウントの印が 1 つも無く、切り替えの手当てが要らない。GitHub App は Checks の読み取りを持てるので、非公開のリポにも当てられる。PAT を使うのは、書かずに試す間の読み取りだけになる。

### 決定 1：切り替える時期

- 受け入れ工程（段階 4）で `run.dryRun` を `false` にする（GitHub に書き始める）前に、GitHub App に切り替える
- 切り替えるまでは、今の要件と基本設計のとおり、メンテナのアカウントの fine-grained PAT で GitHub を読む。`run.dryRun` が `true` の間、進行 Bot は GitHub に書かない
- 切り替えた後は PAT を使わない。人が GitHub で PAT を削除し、秘密の置き場の PAT のファイルも消す

### 決定 2：GitHub App の形

- 当てる先のリポの持ち主（business-os なら組織 Joymerrevent）に 1 つ登録し、当てる先のリポだけに入れる
- 権限は、毎回の実行に要るものだけにする

| 権限 | 読み取りか書き込みか | 使うところ |
| --- | --- | --- |
| Metadata（リポ） | 読み取り | ruleset などを読む |
| Issues（リポ） | 書き込み | ラベル、印とコメント、🟢 から Issue を作る |
| Pull requests（リポ） | 書き込み | PR を作る、ready にする、PR にコメントする |
| Checks（リポ） | 読み取り | CI のチェックの実行（非公開のリポを含む。IMPL-27） |
| Actions（リポ） | 読み取り | ワークフローの実行（IMPL-27） |
| Commit statuses（リポ） | 読み取り | コミットの状態 |
| Variables（リポ） | 読み取り | 全体の止める合図を読む。書き込みは付けない（[ADR-20261017-007][adr-stop]） |
| Projects（組織） | 書き込み | Project の Status と Agent 欄 |

- Contents の書き込みと Administration は付けない。push は今のとおり deploy key で行い（IMPL-15）、ruleset は人の `setup` が作る（[ADR-20261017-004][adr-human]）
- Webhook は使わない。進行 Bot は定期の実行で GitHub を読む
- アプリの登録、リポへの導入、秘密鍵の作成は、人が行う。`setup` は手順を示し、点検が確かめる（SETUP-1 の「人がやるもの」）

### 決定 3：認証

- アプリの秘密鍵のファイル（`.pem`）は、秘密の置き場に置く（[ADR-20261015-001][adr-read] の決定 2 の PAT の代わり。権限は 600）。人のセッションと AI のセッションは読まない
- 進行 Bot は、実行のたびに秘密鍵で JWT を作り、インストールのトークン（1 時間で切れる）に引き換えて使う。切れる前に作り直す。npm の依存を足さず、Node の標準の機能で作る（基本設計の「導入と点検」の「依存」）
- アプリの ID とインストールの ID は秘密でないので、設定に持つ。どちらの設定に置くかは基本設計で決める

### 決定 4：投稿者と一覧

- `safety.botAccounts`（[ADR-20261020-001][adr-bots]）には、アプリの投稿者の名前（`<アプリの名前>[bot]`）を入れる。切り替えのときに、メンテナのアカウントを一覧から外す（PAT で印を書いていないので、読み続ける印は無い）
- 点検の「PAT の持ち主が一覧にある」は、「アプリの投稿者の名前が一覧にある」と読み替える。投稿者の名前の取り方は、試してから基本設計で決める
- AI のセッションに渡す commit の作者は、アプリの投稿者の名前と、GitHub の noreply のアドレスにする

### 決定 5：非公開のリポのチェックの実行

- 非公開のリポのチェックの実行は、アプリの Checks の読み取りで読む。IMPL-27 の読み方（名前・app・ワークフローのパスで照らす）は変えない。要件の UNDECIDED-57 は、本 ADR で決めたとする

### 影響（Consequences）

- 良い点: 人と進行 Bot の投稿を、投稿者でも見分けられる。PR の作者がアプリになるので、メンテナが GitHub の approve でレビューできる
- 良い点: 非公開のリポでも、必須チェックの結果を読める
- 良い点: GitHub に書く前に替えるので、古いアカウントの印を読む手当てが要らない
- 良い点: API のトークンは 1 時間で切れる。長く持つ秘密は、PAT から秘密鍵に替わる
- 悪い点: 進行 Bot に、PAT の認証と、アプリの認証（JWT とトークンの引き換え）の 2 つを書く。PAT の認証は、書かずに試す間だけ使う
- 悪い点: アプリの登録と導入という、人の手作業が 1 つ増える
- 悪い点: アプリで Project の欄を書き換えられるかなど、PAT で確かめたことの一部を、アプリで確かめ直す
- 中立: 秘密鍵が漏れたときの害は、PAT が漏れたときと同じく大きい。秘密の置き場の守り（[ADR-20261015-001][adr-read]）が守りの本体であることは変わらない

### 確認方法（Confirmation）

- 受け入れ工程（段階 4）で `run.dryRun` を `false` にする前に、登録したアプリで、次を実機で確かめて FACT に残す
  - Issue と PR のコメント、ラベル、Project の欄（Status と Agent 欄）を書けること
  - 止める合図の変数を読めて、作成・更新・削除が断られること
  - 公開のリポと非公開のリポで、チェックの実行を読めること
  - コメントの投稿者の名前の形（`<アプリの名前>[bot]`）と、進行 Bot が投稿者の名前を取る方法
  - インストールのトークンが 1 時間で切れ、作り直せること
- 進行 Bot の単体テストで、JWT の作り方と、トークンが切れる前に作り直すことを検査する
- 点検で、PAT のファイルが秘密の置き場に残っていないこと（切り替えの後）を確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：GitHub に書き始める前に切り替える

- 良い点: 古いアカウントの印が無いので、切り替えの手当てが要らない
- 良い点: 今の要件と基本設計（PAT）のまま、段階 3 と段階 4 の最初を進められる
- 悪い点: PAT とアプリの認証を両方書く

### 案 B：段階 3 から GitHub App で作る

- 良い点: PAT の認証を書かずに済む
- 悪い点: 要件と基本設計の PAT の記述を、いますぐ書き直す。段階 3 の前にアプリの登録が要る

### 案 C：段階 8 の前に切り替える

- 良い点: 段階 7 まで PAT だけで済む
- 悪い点: 段階 4〜7 の印がメンテナのアカウントで残り、切り替えのときに一覧で古い印も読む手当てが要る。段階 4〜7 の間、メンテナは自分の PR を approve できない

### 案 D：分けない

- 良い点: 仕組みが増えない
- 悪い点: 非公開のリポのチェックの実行を読めない。人と進行 Bot の投稿を、投稿者では見分けられない

### 案 E：machine user に分ける

- 良い点: 認証は PAT のままでよい
- 悪い点: 人のアカウントを 1 つ増やして管理する（メール、2 要素認証、組織の席）。fine-grained PAT のままなので、非公開のリポのチェックの実行を読めない

## 補足情報（More Information）

- 関連する要件：SAFE-15（[05-safety.md][safety]）、SETUP-1、SETUP-2、SETUP-13（[06-setup-and-stages.md][stages]）、IMPL-20、IMPL-27（[02-implement-review-fix.md][impl]）、STRUCT-2（[07-structure.md][structure]）、UNDECIDED-57（[09-undecided.md][undecided]）、FACT-52、FACT-54（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261020-001][adr-bots]、[ADR-20261017-004][adr-human]、[ADR-20261017-007][adr-stop]、[ADR-20261015-001][adr-read]
- GitHub の公式文書：[GitHub App の権限の一覧][doc-app-permissions]（リポの権限に Checks がある）
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 05-safety.md：SAFE-15 に、書き始める前に GitHub App に切り替え、秘密鍵を秘密の置き場に置くことを足す
  - 06-setup-and-stages.md：SETUP-1 の「人がやるもの」にアプリの登録と導入と秘密鍵を足す。SETUP-2 の点検に、アプリの権限と、切り替えの後に PAT が残っていないことを足す。SETUP-13 を、書き始める前にアプリに分けることに直す。段階の表の受け入れ工程（段階 4）に、`dryRun` を切る前にアプリに切り替えることを足す
  - 02-implement-review-fix.md：IMPL-27 に、非公開のリポのチェックの実行はアプリの Checks の読み取りで読むことを足す
  - 09-undecided.md：UNDECIDED-57 を欠番にする
  - 基本設計の「設定」：アプリの ID とインストールの ID の置き場、`safety.botAccounts` の切り替え
  - 基本設計の「導入と点検」：PAT の権限の節を、切り替えの前の PAT と、切り替えの後のアプリの権限に分ける。`setup` の人がやることと、点検の組を直す
  - 基本設計の「AI のセッション」：commit の作者を、切り替えの後はアプリの投稿者の名前にする
  - [ADR-20261020-001][adr-bots]、[ADR-20261017-004][adr-human]、[ADR-20261017-007][adr-stop]、[ADR-20261015-001][adr-read]：タイトルの下に、本 ADR で一部を改めた旨の注記を足す

[adr-bots]: ./adr-20261020-001-bot-accounts-as-list.md
[adr-human]: ./adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-stop]: ./adr-20261017-007-stop-switch-in-repo-variable.md
[adr-read]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[safety]: ../requirements/05-safety.md
[stages]: ../requirements/06-setup-and-stages.md
[impl]: ../requirements/02-implement-review-fix.md
[structure]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[doc-app-permissions]: https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps
[adr-20261024-001]: ./adr-20261024-001-work-unit-main-flow-and-overlay-axes.md
[adr-20261024-002]: ./adr-20261024-002-no-stages-4-to-8-and-no-trials.md
