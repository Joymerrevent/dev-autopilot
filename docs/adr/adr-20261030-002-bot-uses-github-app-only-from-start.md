---
status: proposed
created: 2026-10-30
updated: 2026-10-30
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 進行 Bot は最初から GitHub App だけを使い、PAT を使わない

メンテナの案（2026-10-30。最初から GitHub App で始める）を決める。`setup` が App を作ることと、`setup` の GitHub の準備の認証は、ADR-20261030-001 で決める。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261020-002][adr-app]（GitHub に書き始める前に GitHub App に切り替える） | 一部を改める | 決定 1（切り替えるまでメンテナのアカウントの fine-grained PAT で GitHub を読み、書き始める前に切り替え、切り替えた後に PAT を消す）を、本 ADR の決定 1 に改める。決定 4 の「切り替えのときに、メンテナのアカウントを一覧から外す」と「点検の『PAT の持ち主が一覧にある』は『アプリの投稿者の名前が一覧にある』と読み替える」は、最初からアプリの投稿者の名前を一覧に入れる、と読む | 決定 2〜3、決定 5、決定 4 のほかの項目 |
| [ADR-20261024-002][adr-no-stages]（段階 4〜8 と書かずに試すことを持たない） | 一部を改める | 決定 2 の「進行 Bot の GitHub の認証は、当てる先のリポに初めて書く前に GitHub App に切り替える」を、本 ADR の決定 1 に改める | 決定 2 のほかの項目、ほかの決定 |
| [ADR-20261028-002][adr-app-secrets]（進行 Bot の App に秘密の名前を読む権限を足す） | 一部を改める | 決定 2 の「切り替える前の PAT の間は、PAT に秘密を読む権限を足さず、確かめない」を外す（本 ADR の決定 2） | 決定 1、決定 2 のほかの項目 |
| [ADR-20261028-004][adr-bypass-baseline]（ruleset の bypass を updated_at の控えで確かめる） | 一部を改める | 決定 2 の「PAT の間は、今のとおり `bypass_actors` を読んで確かめる」を外す（本 ADR の決定 2） | ほかのすべて |
| [ADR-20261017-004][adr-human]（setup の GitHub の準備は人の gh、check は進行 Bot のジョブ） | 読み替える | 決定 2 の「GitHub の点検は、進行ボットが自分の PAT で行う」と「進行ボットの PAT に、ruleset を作る管理の権限を持たせない」を、進行 Bot の GitHub App と読む | 決定のすべて |

## 背景と課題（Context and Problem Statement）

- [ADR-20261020-002][adr-app] の決定 1（2026-10-20）は、進行 Bot がメンテナのアカウントの fine-grained PAT で始め、GitHub に書き始める前に GitHub App に切り替えるとした。受け入れ工程（段階 4）の最初に、GitHub に書かずに試す間（`run.dryRun`）があり、その間は読むだけの PAT で足りた。App の登録は人の操作なので、後に回せた
- [ADR-20261024-002][adr-no-stages] で、段階 4〜8 と書かずに試すことを持たなくなり、切り替えの時期は「当てる先のリポに初めて書く前」になった（決定 2）
- 今の基本設計では、PAT の間、進行 Bot は作業に入らない。PAT は読むだけで、App に切り替えていなければ点検が fail になる（基本設計の「[導入と点検][design-setup]」の「PAT の権限」と「GitHub App の権限」）。PAT の間に進行 Bot がすることは、点検の一部だけ
- PAT の間だけの決まりが、要件と基本設計に残っている。PAT の権限の表と秘密の置き場の PAT のファイル、切り替えの手順（マシンの設定の `githubApp.*` を書く、`safety.botAccounts` を替える PR、PAT の削除、PAT のファイルが残っていれば fail）、PAT の間は確かめない項目（CI の秘密の置き方、組み込みのワークフロー）、PAT の間は ruleset の `bypass_actors` を読み、App に切り替えた後は控えで確かめる分かれ道
- ADR-20261030-001（`proposed`）は、`setup` がマニフェストで進行 Bot の App を作り、人は作成と導入を押すだけにする。決まれば、App の登録を後に回す理由も無くなる

課題：進行 Bot の GitHub の認証を、最初から GitHub App にするか、今のとおり書き始める前まで PAT を使うか。

## 判断の決め手（Decision Drivers）

- 安全：マシンに置く、長く使える秘密を減らす。印を書く身元を最初から 1 つにする
- 変更のしやすさ・確かめやすさ：認証によって分かれる決まりと点検を減らす
- 運用のしやすさ：人の手順（PAT の発行・保管・削除、切り替え）を減らす

## 検討した選択肢（Considered Options）

- 案 A：進行 Bot は最初から GitHub App だけを使い、PAT を使わない
- 案 B：今のとおり、当てる先のリポに初めて書く前まで PAT を使い、その前に GitHub App に切り替える

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: PAT の間に進行 Bot がすることは点検の一部だけで、PAT で始める理由（書かずに試す間と、App の登録を後に回すこと）は残っていない。PAT を使わなければ、PAT の間だけの決まりと切り替えの手順が要らなくなり、マシンに置く秘密も 1 つ減る。

### 決定 1：進行 Bot は、最初から GitHub App だけを使う

- 進行 Bot の GitHub の認証は、最初の `setup` で App を作って当てる先のリポに入れた後から、GitHub App のインストールのトークンだけにする（[ADR-20261020-002][adr-app] の決定 3）。fine-grained PAT は発行せず、秘密の置き場に置かない
- マシンの設定の `githubApp.*` か、App の秘密鍵が無ければ、点検は fail にし、進行 Bot は GitHub を読まない。秘密の置き場に PAT のファイルがあれば、点検は fail にする（今のとおり）
- `safety.botAccounts` には、最初から App の投稿者の名前を入れる。メンテナのアカウントは入れない

### 決定 2：PAT の間だけの決まりを外す

- PAT の権限の表、PAT から App への切り替えの手順、PAT の間は確かめない項目（CI の秘密の置き方、組み込みのワークフロー）、PAT の間は ruleset の `bypass_actors` を直接読む決まりを外す。どの点検も、App で確かめる形の 1 つにする
- App の権限と投稿者の名前の取り方を試す一度だけの試し（今の切り替えの順 2）は、導入と点検の実装（段階 3）のときに、試し専用の組織で行う

### 影響（Consequences）

- 良い点（安全）: メンテナの身元で長く使える PAT を、マシンに置かない。印を書く身元が、最初から App の 1 つになる
- 良い点（変更のしやすさ・確かめやすさ）: PAT の間と App の後で分かれていた点検が 1 つの形になり、切り替えの手順と、その手順を守らせる点検が要らなくなる
- 良い点（運用）: PAT の発行・保管・削除の手順が無くなる
- 悪い点（運用）: 最初の `setup` で App を作って入れるまで、進行 Bot は GitHub の点検もできない（PAT で先に点検する道が無くなる）
- 中立: 書き始めた後の形は、今の決まり（書き始める前に App に切り替える）と同じ

### 確認方法（Confirmation）

- 反映の後に、要件と基本設計に PAT の間の決まりが残っていないことを grep で確かめる（確かめた事実の表は経緯なので除く）
- `setup` の実装の後に、試し専用の組織で、PAT を使わずに `setup` と `check` が通ることを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：最初から App だけ

- 良い点: 秘密が 1 つ減り、認証で分かれる決まりと切り替えの手順が無くなる
- 悪い点: App を作るまで、進行 Bot は何も確かめられない

### 案 B：書き始める前まで PAT

- 良い点: App を作る前から、進行 Bot の点検の一部（読むだけの項目）を動かせる
- 悪い点: PAT の間だけの決まりと切り替えの手順を持ち続ける。PAT の間に進行 Bot がすることは点検の一部だけで、作業には入らない

## 補足情報（More Information）

- 2026-10-30 に、メンテナが「いっそのこと、最初から GitHub App で始めるのはどうか」と提案した。PAT で始めた理由が、段階 4〜8 を持たなくなって残っていないことを確かめて起票した
- PAT の記述は、要件 5 本（17 か所）と基本設計 7 本（44 か所）にある（2026-10-30 に数えた。確かめた事実の表を除く）
- 関連する要件：SAFE-15（[05-safety.md][safety]）、SETUP-1、SETUP-2、SETUP-13（[06-setup-and-stages.md][setup]）
- 関連する ADR：[ADR-20261017-004][adr-human]、[ADR-20261020-002][adr-app]、[ADR-20261024-002][adr-no-stages]、[ADR-20261028-002][adr-app-secrets]、[ADR-20261028-004][adr-bypass-baseline]、ADR-20261030-001
- 要件と設計への反映（承認の後に、別の PR で行う。量があるので、ファイルごとに分けることも考える）
  - 05-safety.md と 06-setup-and-stages.md：SAFE-15 の鍵、SETUP-1 の人がやるもの（PAT の発行と保管を外す）、SETUP-2 の点検
  - 基本設計：「[導入と点検][design-setup]」の PAT の権限の表、切り替えの手順、PAT の間の点検の分かれ道。設定の `github-pat`。状態遷移と判定、AI のセッション、テストの PAT の記述
  - [ADR-20261020-002][adr-app]・[ADR-20261024-002][adr-no-stages]・[ADR-20261028-002][adr-app-secrets]・[ADR-20261028-004][adr-bypass-baseline]：タイトルの下に、改めた旨の注記を足す。[ADR-20261017-004][adr-human]：読み替えの注記を足す

[adr-human]: ./adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-no-stages]: ./adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[adr-app-secrets]: ./adr-20261028-002-app-reads-secret-names-for-checks.md
[adr-bypass-baseline]: ./adr-20261028-004-compare-ruleset-updated-at-with-setup-baseline.md
[safety]: ../requirements/05-safety.md
[setup]: ../requirements/06-setup-and-stages.md
[design-setup]: ../design/basic/04-setup-and-check.md
