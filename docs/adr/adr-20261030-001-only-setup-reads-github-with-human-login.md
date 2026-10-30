---
status: proposed
created: 2026-10-30
updated: 2026-10-30
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 人の gh のログインで GitHub を確かめるのは setup だけにし、check は進行 Bot の認証で確かめる

[ADR-20261028-002][adr-app-secrets] の決定 2 の 1 項目「人の `check` と `setup` は、人の `gh` のログインで同じことを確かめる」は、[ADR-20261017-004][adr-human] の決定 2（`check` は進行 Bot のジョブを起こし、GitHub の点検は進行 Bot の認証で行う）と食い違う。本 ADR で改める。
基本設計の「[導入と点検][design-setup]」の同じ食い違い 2 か所も、本 ADR に合わせて直す。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261028-002][adr-app-secrets]（進行 Bot の App に秘密の名前を読む権限を足す） | 一部を改める | 決定 2 の「人の `check` と `setup` は、人の `gh` のログインで同じことを確かめる」を、本 ADR の決定 1・2 に改める | 決定 1（App に Secrets の読み取りを足す）、決定 2 のほかの項目（GitHub App に切り替えた後は毎回の点検で確かめる、PAT の間は進行 Bot は確かめない） |
| [ADR-20261017-004][adr-human]（setup の GitHub の準備は人の gh、check は進行 Bot のジョブ） | 据え置く | — | 決定 2 |

## 背景と課題（Context and Problem Statement）

- [ADR-20261017-004][adr-human] の決定 2 は、`setup` の GitHub の準備を人の `gh` のログイン（組織の管理者）で行うとした。`check` と `/dev-autopilot` は、進行 Bot を直接起動せず、定期の実行と同じジョブを起こす。GitHub の点検は進行 Bot が自分の認証で行い、人のセッションは実行の記録を読んで表示する
- [ADR-20261028-002][adr-app-secrets] の決定 2 の最後の項目は、人の `check` が人の `gh` のログインで秘密の置き方を確かめるとした。起票のときに、`check` が人の `gh` のログインで動くと取り違えた
- 基本設計の「導入と点検」に、同じ食い違いが 2 か所ある
  - CI の秘密の置き方の点検：「人の `check` と `setup` は、人の `gh` のログインで同じことを確かめる」（[ADR-20261028-002][adr-app-secrets] を写した文）
  - Status 欄に関わる組み込みのワークフローの点検：「人の `setup` と `check` は、人の `gh` のログイン（`project` の権限）で読む」（[ADR-20261026-001][adr-status] の反映で書いた文）
- 進行 Bot の PAT は、Secrets の読み取りが無く、Projects は読み取りだけ（組み込みのワークフローを読むには Projects の書き込みが要る。FACT-76）なので、PAT の間は、進行 Bot はどちらも確かめない。GitHub App に切り替えた後は、App の Secrets の読み取り（[ADR-20261028-002][adr-app-secrets]）と Projects の書き込みで読める
- PAT の間は、進行 Bot は作業に入らない（基本設計の「導入と点検」の「PAT の権限」）

課題：人の `gh` のログインで GitHub を確かめるのを、`setup` だけにするか、`check` にも持たせるか。

## 判断の決め手（Decision Drivers）

- 点検の結果の出どころを 1 つにする（人の `check` は、進行 Bot の点検の結果を表示する。[ADR-20261017-004][adr-human] の決定 2）
- 人の Claude Code のセッションに、GitHub を読む別の道を増やさない
- 守りが崩れたときに、進行 Bot が GitHub に書く前に止まる

## 検討した選択肢（Considered Options）

- 案 A：人の `gh` のログインで確かめるのは `setup` だけにする。`check` は進行 Bot の認証で確かめ、PAT の間は確かめない項目として結果に示す
- 案 B：`check` の Skill にも、人の `gh` のログインで GitHub を読む部分を足す（[ADR-20261017-004][adr-human] の決定 2 を改める）
- 案 C：進行 Bot の PAT に Secrets の読み取りと Projects の書き込みを足し、PAT の間も `check` で確かめる

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: [ADR-20261017-004][adr-human] の決定 2 のとおり、点検の結果の出どころが進行 Bot の点検の 1 つに保たれる。PAT の間は進行 Bot が作業に入らないので、確かめない間に守りが崩れていても、AI が書いたものは GitHub に出ない。GitHub App に切り替えた後は、毎回の点検で確かめる。

### 決定 1：人の gh のログインで GitHub を確かめるのは setup だけにする

- `setup`（GitHub の準備）は、今のとおり人の `gh` のログインで、CI の秘密の置き方と、Status 欄に関わる組み込みのワークフローも確かめ、外れていれば示す
- `check` は、[ADR-20261017-004][adr-human] の決定 2 のとおり進行 Bot のジョブを起こし、進行 Bot の認証で確かめた結果を表示する。人の `gh` のログインでは読まない

### 決定 2：PAT の間は、確かめない項目として check の結果に示す

- PAT の間、進行 Bot の点検は、CI の秘密の置き方と組み込みのワークフローを確かめず、結果に確かめていない項目として示す（warn）。人は `setup` で確かめる
- GitHub App に切り替えた後は、毎回の点検で確かめる（[ADR-20261028-002][adr-app-secrets] の決定 2 のまま）

### 影響（Consequences）

- 良い点: 点検の結果の出どころが 1 つのまま。人のセッションが GitHub を読む道を増やさない
- 良い点: PAT の間に確かめていない項目が、`check` の結果で人に見える
- 悪い点: PAT の間は、CI の秘密の置き方と組み込みのワークフローを、人が `setup` を実行したときにしか確かめない
- 中立: PAT の間は進行 Bot が作業に入らないので、確かめない間に崩れていても、AI が書いたものは GitHub に出ない

### 確認方法（Confirmation）

- 反映の後に、要件と基本設計に、人の `check` が人の `gh` のログインで GitHub を読む書き方が残っていないことを grep で確かめる
- 点検の単体テストで、PAT の間は、CI の秘密の置き方と組み込みのワークフローが、確かめていない項目（warn）として結果に出ることを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：人の gh のログインで確かめるのは setup だけ

- 良い点: 決まっている分担（[ADR-20261017-004][adr-human] の決定 2）のまま
- 悪い点: PAT の間は、`setup` を実行したときにしか確かめない

### 案 B：check の Skill にも人の gh のログインで読む部分を足す

- 良い点: PAT の間も、`check` を実行すれば確かめられる
- 悪い点: 点検の結果の出どころが、進行 Bot と人のセッションの 2 つに分かれる。人のセッションに GitHub を読むコマンドが増え、sandbox を有効にした人の Claude Code では、通信の許可が要りうる

### 案 C：進行 Bot の PAT に権限を足す

- 良い点: PAT の間も、毎回の点検で確かめられる
- 悪い点: PAT の間に使わない権限（Projects の書き込み）を PAT に持たせる。PAT は読み取りだけにする決まり（[ADR-20261020-002][adr-app] の決定 1）を改めることになる

## 補足情報（More Information）

- 2026-10-28 に、ADR-20261028-004（ruleset の bypass を `updated_at` の控えで確かめる）を起票する途中で見つけた。ADR-20261028-004 は、控えを書くのを `setup` だけにしている
- 関連する要件：SETUP-1、SETUP-2（[06-setup-and-stages.md][setup]）、SAFE-20（[05-safety.md][safety]）、FACT-76（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261017-004][adr-human]、[ADR-20261020-002][adr-app]、[ADR-20261026-001][adr-status]、[ADR-20261028-002][adr-app-secrets]
- 要件と設計への反映（承認の後に、別の PR で 1 本にまとめて行う）
  - 基本設計の「[導入と点検][design-setup]」：CI の秘密の置き方の点検と、組み込みのワークフローの点検の文を直し、PAT の間は確かめない項目として結果に示すことを書く
  - [ADR-20261028-002][adr-app-secrets]：タイトルの下に、改めた旨の注記を足す

[adr-app-secrets]: ./adr-20261028-002-app-reads-secret-names-for-checks.md
[adr-human]: ./adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-status]: ./adr-20261026-001-use-status-field-for-state.md
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[setup]: ../requirements/06-setup-and-stages.md
[safety]: ../requirements/05-safety.md
[facts]: ../requirements/10-verified-facts.md
[design-setup]: ../design/basic/04-setup-and-check.md
