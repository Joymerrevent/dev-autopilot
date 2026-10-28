---
status: accepted
created: 2026-10-28
updated: 2026-10-28
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 進行 Bot の GitHub App に秘密の名前を読む権限を足し、毎回の点検で当てる先のリポの秘密の置き方を確かめる

[ADR-20261028-001][adr-ci-secrets] の決定 2（点検で当てる先のリポの秘密の置き方を確かめる。読む権限は基本設計で決める）を、毎回の実行の点検で行うために、[ADR-20261020-002][adr-app] の決定 2 の GitHub App の権限の表を改める。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261020-002][adr-app]（GitHub App への切り替え） | 一部を改める | 決定 2 の権限の表に、Secrets（リポ）の読み取りを足す（本 ADR の決定 1） | 決定 2 のほかの権限と、「権限は、毎回の実行に要るものだけにする」こと。決定 1・3〜5 |
| [ADR-20261028-001][adr-ci-secrets]（CI の秘密の置き方） | 据え置く（決定 2 で基本設計に任せた読む権限のうち、App の権限を本 ADR で決める） | — | 決定 1〜3 |

## 背景と課題（Context and Problem Statement）

- [ADR-20261028-001][adr-ci-secrets] は、当てる先のリポの CI の秘密を、base ブランチだけが使える Environment に置き、点検で、リポの秘密が無いこと、組織の秘密が当てる先のリポに届いていないこと、Environment の使えるブランチを確かめ、外れていれば fail にすると決めた。読む権限は基本設計に任せた
- メンテナは、人の `check` だけでなく、毎回の実行の点検で確かめる形を選んだ（2026-10-28）。後から秘密が足されたときに、人が `check` を回すのを待たずに、次の実行で止めるため
- GitHub App のインストールのトークンで読むときに要る権限は、GitHub の公式文書の元データ（2026-10-28 に読んだ）で次のとおり
  - リポの秘密の一覧（`GET /repos/{owner}/{repo}/actions/secrets`）と、リポに届く組織の秘密の一覧（`GET /repos/{owner}/{repo}/actions/organization-secrets`）：Secrets の読み取り
  - Environment の一覧と中身（`GET /repos/{owner}/{repo}/environments`、`…/environments/{environment_name}`）と、使えるブランチの規則（`…/deployment-branch-policies`）：Actions の読み取り
- [ADR-20261020-002][adr-app] の決定 2 の権限の表は、Actions の読み取りを持ち、Secrets の読み取りを持たない。秘密の一覧の API は、名前と日時だけを返し、値を返さない

課題：毎回の実行の点検で秘密の置き方を確かめるために、GitHub App の権限をどう足すか。

## 判断の決め手（Decision Drivers）

- 後から秘密が足されたときに、人の記憶でなく、次の実行で止まる（止める側に倒す）
- GitHub App の権限は、毎回の実行に要るものだけにする（[ADR-20261020-002][adr-app] の決定 2）
- 秘密の値は、進行 Bot にも読ませない

## 検討した選択肢（Considered Options）

- 案 A：GitHub App に Secrets（リポ）の読み取りを足し、毎回の実行の点検で確かめる
- 案 B：GitHub App の権限は足さず、人の `check` と `setup` だけが、人の `gh` のログインで確かめる
- 案 C：GitHub App に Administration（リポ）の読み取りを足す（リポの設定をまとめて読めるようにする）

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: 秘密の一覧の API は名前だけを返すので、Secrets の読み取りを足しても、進行 Bot は秘密の値を読めない。Environment と使えるブランチの規則は、今ある Actions の読み取りで読める。足すのは 1 つの読み取りの権限だけで、後から秘密が足されたときに次の実行で止まる。

### 決定 1：GitHub App の権限に、Secrets（リポ）の読み取りを足す

| 権限 | 読み取りか書き込みか | 使うところ |
| --- | --- | --- |
| Secrets（リポ） | 読み取り | 当てる先のリポの秘密と、リポに届く組織の秘密の名前を読み、点検で秘密の置き方を確かめる（[ADR-20261028-001][adr-ci-secrets] の決定 2）。値は読めない |

- Secrets の書き込みは付けない。秘密を置くのも消すのも、人が行う
- Environment と使えるブランチの規則は、今ある Actions の読み取りで読む

### 決定 2：毎回の実行の点検で確かめる

- 進行 Bot は、GitHub App に切り替えた後、毎回の実行の最初の点検（GitHub の組）で、秘密の置き方を確かめ、外れていれば fail にして作業に入らない
- 切り替える前の PAT の間は、PAT に秘密を読む権限を足さず、確かめない（PAT の間は作業に入らない）
- 人の `check` と `setup` は、人の `gh` のログインで同じことを確かめる

### 影響（Consequences）

- 良い点: 後から秘密が足されても、次の実行で止まり、作業 AI のコードが秘密に届く前に気づける
- 良い点: 足す権限は読み取り 1 つで、値は読めない
- 悪い点: GitHub App の権限が 1 つ増え、点検の「アプリの権限が表のとおりか」の比べる相手が増える。既に入れたアプリは、人が権限を足して、組織の持ち主が承認する手間がかかる
- 中立: 秘密の名前（どんな秘密があるか）は、進行 Bot の点検の記録に残りうる。名前だけで、値は残らない

### 確認方法（Confirmation）

- 試し専用の組織の試し用の GitHub App に Secrets の読み取りを足し、インストールのトークンで、リポの秘密と組織の秘密の名前を読めること、秘密を置く・消す操作が断られること、Environment の使えるブランチの規則を Actions の読み取りで読めることを確かめ、FACT に載せる
- 点検の単体テストで、[ADR-20261028-001][adr-ci-secrets] の確認方法の fail の組み合わせを、GitHub App の読み取りの結果から判定できることを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：Secrets の読み取りを足し、毎回確かめる

- 良い点: 次の実行で止まり、値は読めない
- 悪い点: 権限が 1 つ増える

### 案 B：人の check だけで確かめる

- 良い点: GitHub App の権限が増えない
- 悪い点: 後から秘密が足されたことに、人が `check` を回すまで気づかない。その間、作業 AI のコードが秘密に届きうる

### 案 C：Administration の読み取りを足す

- 良い点: リポの設定をまとめて読める
- 悪い点: 要るのは秘密の名前だけで、Administration は読める範囲が広すぎる。「権限は、毎回の実行に要るものだけにする」に合わない

## 補足情報（More Information）

- 2026-10-28 に、[ADR-20261028-001][adr-ci-secrets] の反映の前に、点検で読む権限をメンテナに聞き、毎回の実行の点検で確かめる形（案 A）が選ばれた
- 要る権限は、GitHub の公式文書の元データ（`github/docs` の `src/github-apps/data/fpt-2022-11-28/server-to-server-permissions.json`）で確かめた。実機では確かめていない（確認方法）
- 関連する要件：SETUP-2（[06-setup-and-stages.md][setup]）
- 関連する ADR：[ADR-20261020-002][adr-app]、[ADR-20261028-001][adr-ci-secrets]
- 要件と設計への反映（承認の後に、[ADR-20261028-001][adr-ci-secrets] の反映と 1 本の PR にまとめて行う）
  - 基本設計の「[導入と点検][design-setup]」：GitHub App の権限の表、点検の GitHub の組、アプリの権限を表と比べる点検
  - [ADR-20261020-002][adr-app]：タイトルの下に、本 ADR で決定 2 の一部を改めた旨の注記を足す

[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-ci-secrets]: ./adr-20261028-001-ci-secrets-only-in-base-branch-environments.md
[setup]: ../requirements/06-setup-and-stages.md
[design-setup]: ../design/basic/04-setup-and-check.md
