---
status: proposed
created: 2026-10-30
updated: 2026-10-30
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# setup の GitHub の準備は setup 用の App で行って人の gh を使わず、2 つの App は setup がマニフェストで作る

メンテナの案（2026-10-30。人の `gh` のログインを最小限にし、GitHub の操作を App に寄せる）を決める。あわせて、[ADR-20261028-002][adr-app-secrets] の決定 2 の「人の `check` と `setup` は、人の `gh` のログインで同じことを確かめる」が、[ADR-20261017-004][adr-human] の決定 2（`check` は進行 Bot のジョブで動き、GitHub の点検は進行 Bot の認証で行う）と食い違うのを解く。
進行 Bot が最初から GitHub App だけを使い、PAT を使わないことは、ADR-20261030-002 で決める。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261017-004][adr-human]（setup の GitHub の準備は人の gh、check は進行 Bot のジョブ） | 一部を改める | 決定 2 の「GitHub の準備は人の `gh` のログインで行う」と「人の `gh` のログインを持たない専用マシンでは、GitHub の準備は人の手元のマシンで行う」を、本 ADR の決定 1 に改める | 決定 2 のほかの項目（`setup` を GitHub の準備とマシンの準備に分ける、`check` と `/dev-autopilot` は進行 Bot のジョブを起こして GitHub の点検は進行 Bot の認証で行う、進行 Bot に ruleset を作る権限を持たせない、deploy key は人が作る） |
| [ADR-20261020-002][adr-app]（GitHub App に切り替える） | 一部を改める | 決定 2 の「アプリの登録、リポへの導入、秘密鍵の作成は、人が行う」を、本 ADR の決定 2 に改める | 決定 2 の権限の表、決定 3〜5 |
| [ADR-20261028-002][adr-app-secrets]（進行 Bot の App に秘密の名前を読む権限を足す） | 一部を改める | 決定 2 の「人の `check` と `setup` は、人の `gh` のログインで同じことを確かめる」を、本 ADR の決定 1・3 に改める | 決定 1、決定 2 のほかの項目 |
| [ADR-20261028-004][adr-bypass-baseline]（ruleset の bypass を updated_at の控えで確かめる） | 一部を改める | 決定 1 の「`setup`（人の `gh` のログイン。組織の管理者）」の認証を、本 ADR の決定 1 の `setup` 用の App に改める | ほかのすべて |

## 背景と課題（Context and Problem Statement）

- `setup` の GitHub の準備（ラベル、Project の欄、ruleset、全体の止める合図と ruleset の控えの変数）は、人の `gh` のログインで行う（[ADR-20261017-004][adr-human] の決定 2）。人の `gh` のログインのトークンは、人が触れるすべてのリポ（本番の組織のほかのリポを含む）に届く
- 進行 Bot の App には、ruleset と変数を作る権限を持たせない。ruleset は作業 AI の push 先を限る柵で、止める合図の変数と ruleset の控えは進行 Bot を止めるためのものなので、進行 Bot が自分で外せてはいけない（[ADR-20261020-002][adr-app] の決定 2、[ADR-20261017-007][adr-stop]、[ADR-20261028-004][adr-bypass-baseline]）
- App は、マニフェストから作れる。手元で待ち受けて戻り先のコードを引き換えると、人が「Create」を押すだけで App の ID と秘密鍵を受け取れる（FACT-92）
- App の人の代わりのトークン（device flow）は、人が App の設定で Device Flow を有効にすると使える。トークンは 8 時間で切れ、更新のトークン（約 182 日）が付く（FACT-93）。`setup` 用の権限を持つ App の人の代わりのトークンで、ruleset・変数・ラベル・Project を作れ、`bypass_actors` も読め、App を入れていないリポとほかの組織には届かない（FACT-94）
- [ADR-20261028-002][adr-app-secrets] の決定 2 と、基本設計の「[導入と点検][design-setup]」の 2 か所（CI の秘密の置き方の点検、Status 欄に関わる組み込みのワークフローの点検）は、人の `check` が人の `gh` のログインで GitHub を読むと書いている。`check` は進行 Bot のジョブで動くので、食い違っている

課題：人の `gh` のログインを使わずに、進行 Bot に柵を外せる権限を渡さないまま、`setup` の GitHub の準備をどの認証で行うか。

## 判断の決め手（Decision Drivers）

- 安全：`setup` に、人のすべてのリポに届く広いトークンを使わない
- 安全：柵を外せる権限（ruleset と変数の書き込み）を、進行 Bot に渡さない。手元に、長く使える管理の鍵を残さない
- 運用のしやすさ：人がする操作と、要る道具（`gh` の導入とログイン）を減らす
- 確かめやすさ：点検の結果の出どころを、進行 Bot の点検の 1 つに保つ

## 検討した選択肢（Considered Options）

- 案 A：`setup` 用の App を分け、`setup` の実行ごとに人が承認する人の代わりのトークン（device flow）で GitHub の準備をする。トークンも更新のトークンも、`setup` 用の App の秘密鍵も、手元に残さない
- 案 B：今のまま、人の `gh` のログインで GitHub の準備をする
- 案 C：進行 Bot の App に、ruleset と変数を作る権限を足し、`setup` も進行 Bot の App で GitHub の準備をする
- 案 D：`setup` 用の App を分け、`setup` 用の App の秘密鍵（インストールのトークン）で GitHub の準備をする
- 案 E：案 A で、更新のトークンを秘密の置き場に置き、2 回目からは承認を省く

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: `setup` が使うトークンは、`setup` 用の App を入れたリポと組織の Project にしか届かず（FACT-94）、人のほかのリポに届かない。柵を外せる権限は、人が承認した `setup` の実行の 8 時間の間だけあり、進行 Bot も手元のファイルも持たない。人の `gh` のログインは要らなくなる。

### 決定 1：setup の GitHub の準備は、setup 用の App の人の代わりのトークンで行う

- `setup` の GitHub の準備と、`setup` が GitHub について確かめること（ruleset の `bypass_actors` を含む）は、`setup` 用の App の人の代わりのトークン（device flow）で行う。人の `gh` のログインは使わない
- トークンは `setup` の実行の間だけメモリに持ち、ファイルに書かない。更新のトークンも持たない。`setup` を実行するたびに、人がブラウザで承認のコードを入れる
- `setup` 用の App の権限は、GitHub の準備と `setup` の確かめに要るものだけにする（ruleset を作る Administration の書き込み、変数を作る Variables の書き込み、ラベルを作る Issues の書き込み、Project の欄を作る Projects（組織）の書き込み、確かめに要る読み取り）。権限の表は基本設計で決める。当てる先のリポだけに入れる
- 人の代わりのトークンは、App の権限と、承認した人の権限の重なりだけを持つ。ruleset を作るには、組織の管理者が承認する

### 決定 2：2 つの App は、setup がマニフェストで作る

- `setup` は、進行 Bot の App（[ADR-20261020-002][adr-app] の決定 2 の権限）と `setup` 用の App を、マニフェストで作る。人は、それぞれの「Create」と、当てる先のリポへの導入（Install）をブラウザで押す。`setup` 用の App は、人が設定で Device Flow を 1 回だけ有効にする（FACT-93）
- 進行 Bot の App の秘密鍵は、受け取ったら画面に出さずに秘密の置き場に置く（[ADR-20261020-002][adr-app] の決定 3）。`setup` 用の App の秘密鍵は、置かずに捨てる（人の代わりのトークンだけを使うため）。`client_secret` と Webhook の秘密は、どちらの App も置かない
- すでに App があるときは作らず、App の権限と導入先を読んで、違いを示す（何度実行しても同じ結果。SETUP-1）

### 決定 3：check は、進行 Bot の App で確かめる

- `check` は、[ADR-20261017-004][adr-human] の決定 2 のとおり進行 Bot のジョブを起こし、進行 Bot の App で確かめた結果を表示する。人の `gh` のログインでも `setup` 用の App でも読まない

### 影響（Consequences）

- 良い点（安全）: `setup` のトークンは、人のほかのリポと、ほかの組織に届かない（FACT-94）
- 良い点（安全）: 柵を外せる権限が、承認した `setup` の実行の間だけあり、手元のファイルにも進行 Bot にも残らない
- 良い点（運用）: `gh` の導入とログインが要らない。App の登録の手順を、人が読みながら行わずに済む（`setup` が作り、人は押すだけ）
- 悪い点（運用）: App が 2 つになり、最初に作成 2 回・導入 2 回・Device Flow の有効化 1 回を人が押す
- 悪い点（運用）: `setup` を実行するたびに、人がブラウザで承認する
- 中立: `check` の動きは変わらない。ruleset を作れるのは、今と同じく組織の管理者だけ

### 確認方法（Confirmation）

- FACT-92〜94 で、マニフェストでの作成、device flow、人の代わりのトークンで GitHub の準備ができることと、届く範囲を確かめた
- `setup` の実装で、トークンと更新のトークンと `setup` 用の App の秘密鍵がファイルに書かれないことを、単体テストで確かめる
- `setup` の実装の後に、試し専用の組織で、`setup` が 2 つの App を作り、`setup` 用の App で GitHub の準備を終えることを確かめる
- 反映の後に、要件と基本設計に、人の `gh` のログインで GitHub の準備や点検をする書き方が残っていないことを grep で確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：setup 用の App を分け、実行ごとに承認する

- 良い点: トークンの届く範囲が、入れたリポと組織の Project に絞られる。長く使える管理の鍵が手元に残らない
- 悪い点: App が 2 つになる。`setup` のたびに承認が要る

### 案 B：今のまま人の gh のログイン

- 良い点: App は 1 つで済み、承認の操作が要らない
- 悪い点: `setup` に、人のすべてのリポに届くトークンを使う。人のマシンに `gh` の導入とログインが要る

### 案 C：進行 Bot の App に setup の権限を足す

- 良い点: App は 1 つで、承認の操作も要らない
- 悪い点: 進行 Bot が、作業 AI の push 先を限る ruleset を消し、止める合図を `run` に戻し、ruleset の控えを書き換えられる。進行 Bot の誤りや Plugin の版の不具合で、柵が外れる

### 案 D：setup 用の App の秘密鍵で準備する

- 良い点: 承認の操作が要らない
- 悪い点: ruleset と変数を作れる鍵が、手元に長く残る。進行 Bot と同じマシンに置くと、進行 Bot のプロセスから読めてしまい、案 C に近くなる

### 案 E：案 A で更新のトークンを置く

- 良い点: 2 回目から承認が要らない
- 悪い点: ruleset と変数を作れる更新のトークン（約 182 日）が、手元に長く残る（FACT-93）

## 補足情報（More Information）

- 2026-10-30 に、メンテナが「人の `setup` は App の作成だけを人の `gh` で行い、ほかは App で」と提案した。進行 Bot の App に `setup` の権限を持たせると柵を外せるので、`setup` 用の App を分ける形にし、メンテナが了承した。試し専用の組織で確かめた事実は FACT-92〜94
- 本 ADR は、元は「人の `gh` のログインで GitHub を確かめるのは `setup` だけにする」として起票した（同じ番号）。メンテナの案に合わせて書き直した
- 本 ADR で決めないこと
  - `setup` 用の App の権限の表、2 つの App の名前の付け方、Device Flow を有効にする手順の示し方（基本設計）
  - 承認する人が組織の管理者でないときの扱い（基本設計。ruleset を作れないので止まる）
- 関連する要件：SETUP-1、SETUP-2（[06-setup-and-stages.md][setup]）、SAFE-15、SAFE-20（[05-safety.md][safety]）、FACT-92〜94（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261017-004][adr-human]、[ADR-20261017-007][adr-stop]、[ADR-20261020-002][adr-app]、[ADR-20261028-002][adr-app-secrets]、[ADR-20261028-004][adr-bypass-baseline]、ADR-20261030-002
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 06-setup-and-stages.md：SETUP-1 の人がやるもの（App の登録と導入を `setup` が行い、人は押す）と、GitHub の準備の認証
  - 05-safety.md：SAFE-15 の鍵の系統に、`setup` 用の App の鍵を持たないことを足す
  - 基本設計の「[導入と点検][design-setup]」：`setup` の流れ、GitHub の準備の認証、`setup` 用の App の権限の表、CI の秘密の置き方と組み込みのワークフローの点検の文、ruleset の控えを書く認証
  - [ADR-20261017-004][adr-human]・[ADR-20261020-002][adr-app]・[ADR-20261028-002][adr-app-secrets]・[ADR-20261028-004][adr-bypass-baseline]：タイトルの下に、改めた旨の注記を足す

[adr-human]: ./adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-stop]: ./adr-20261017-007-stop-switch-in-repo-variable.md
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-app-secrets]: ./adr-20261028-002-app-reads-secret-names-for-checks.md
[adr-bypass-baseline]: ./adr-20261028-004-compare-ruleset-updated-at-with-setup-baseline.md
[setup]: ../requirements/06-setup-and-stages.md
[safety]: ../requirements/05-safety.md
[facts]: ../requirements/10-verified-facts.md
[design-setup]: ../design/basic/04-setup-and-check.md
