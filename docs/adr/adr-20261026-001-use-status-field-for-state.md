---
status: accepted
created: 2026-10-26
updated: 2026-10-26
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# Project にもともとある Status 欄は名前を変えずに状態の欄として使い、主な流れの枠は新しい欄で作る

GitHub の API では Project にもともとある Status 欄の名前を変えられないと確かめた（FACT-74。[実機で確かめた事実][facts]）。本 ADR は、Status 欄を表示の欄のどれに使うかを決め直す。
本 ADR は、[ADR-20261024-001][adr-model-states]（作業の単位の状態を主な流れと重ねる 3 本の軸で表し、Project の欄は表示だけにする）を前提にする。

改める決定と、据え置く決定は次のとおり。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261024-001][adr-model-states]（作業の単位の模型と表示） | 一部を改める | 決定 5 の「主な流れの枠には、Project にもともとある Status 欄を使い、名前を『主な流れの枠』に変える。Status 欄の名前を変えられなければ、同じ働きの欄を新しく作る」を、本 ADR の決定 1・2 に改める | 決定 5 のほかの項目（表示の欄 5 つと中身、どれも表示だけで判定に読まないこと、「次の番」の欄を持たないこと、軸ごとのビュー）。決定 1〜4 と 6 以降 |

## 背景と課題（Context and Problem Statement）

- [ADR-20261024-001][adr-model-states] の決定 5 は、進行 Bot が Project に書く表示の欄を 5 つ（主な流れの枠・状態・止め方・先の依頼・安全）とした。主な流れの枠には、Project にもともとある Status 欄の名前を変えて使い（ボードの列が 4 つで済む）、名前を変えられなければ、同じ働きの欄を新しく作るとした
- 2026-10-26 に、試し専用の組織に作った Project で確かめた（FACT-74）
  - Status 欄の名前は、GraphQL の `updateProjectV2Field` で変えられない。mutation はエラーにならずに、名前が変わらない。自分で作った単一選択の欄なら、名前が変わる
  - Status 欄の選択肢は、API で置き換えられる
  - ビューは API で作れ、名前・形・見せる欄・絞り込みを指定できる。ボードの列とグループ分けにする欄は、指定できない
- 決定 5 の形のまま新しい欄を作ると、もともとの Status 欄は使われずに残る（選択肢は Todo・In Progress・Done のまま）。API で作ったボードは、列を Status 欄で分けるので、使われない欄で列を分けることになる
- Project には、項目が閉じられたときと PR がマージされたときに Status 欄を Done にする組み込みのワークフローがあり、新しい Project では最初から有効になっている。Status 欄の値が変わったときに Issue を閉じるワークフローもある（[公式文書][doc-workflows]）。進行 Bot が書く欄に Status 欄を使うと、組み込みのワークフローが上書きしうる。試しの Project では、Status 欄の選択肢を置き換えた後に読むと、どちらも無効になっていた（最初から無効だったのか、置き換えで無効になったのかは確かめていない）

課題：Status 欄の名前を変えられないとき、Status 欄を表示の欄のどれに使い、主な流れの枠をどの欄に書くか。

## 判断の決め手（Decision Drivers）

- `setup` が API だけで欄・選択肢・ビューを作れる（人が画面で名前を変える手順を増やさない）
- 欄の名前と中身が合う（「Status」の欄に、状態でないものを入れない）
- 使われない欄を残さない
- API で作るボードは列を Status 欄で分けるので、Status 欄の列が人の役に立つ
- 進行 Bot が書く欄を、組み込みのワークフローが書き換えない

## 検討した選択肢（Considered Options）

- 案 A：主な流れの枠は単一選択の欄として新しく作り、Status 欄は名前を変えずに「状態」の欄として使う
- 案 B：決定 5 の形のまま、主な流れの枠と状態を新しい欄で作り、Status 欄は使わない
- 案 C：`setup` が手順を示し、人が画面で Status 欄の名前を「主な流れの枠」に変える
- 案 D：Status 欄を、名前を変えずに主な流れの枠として使う（選択肢を依頼・計画・変更・終了にする）

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: 「Status」は「状態」の英語なので、名前を変えられなくても、欄の名前と中身が合う。使われない欄が残らず、`setup` は API だけで作れる。API で作るボードが状態ごとの列になり、人の番の状態（下書き済み・レビュー合格）が列で見える。

### 決定 1：主な流れの枠は、単一選択の欄として新しく作る

- `setup` は、主な流れの枠を、単一選択の欄として新しく作る。選択肢は依頼・計画・変更・終了（[ADR-20261024-001][adr-model-states] の決定 5 の表のまま）

### 決定 2：Status 欄は、名前を変えずに状態の欄として使う

- 状態の欄には、Project にもともとある Status 欄を使う。名前は「Status」のまま変えず、選択肢を主な流れの 17 の状態の名前にする
- `setup` は、Status 欄の今の選択肢を 17 の状態に置き換える。置き換えると、Project の項目の今の Status の値は失われる（[ADR-20261024-001][adr-model-states] の決定 5 の、名前を変えて選択肢を置き換える形でも同じ）。`setup` は、作るものの一覧（SETUP-1）に置き換えることと値が失われることを出し、人の承認を受けてから置き換える
- 文書では、状態の欄を「状態（Status 欄）」と書く

### 決定 3：Status 欄に書き込む組み込みのワークフローは止める

- Project の組み込みのワークフローのうち、Status 欄に値を書くもの（項目が閉じられた、PR がマージされた、など）と、Status 欄の値で Issue を閉じるものは、止めておく。進行 Bot が書いた状態を書き換えず、状態の値で Issue を閉じないため
- 止める手段（人が画面で切るか、API で消すか）と、点検での確かめ方（有効なら fail か warn か）は、基本設計で決める。組み込みのワークフローの有効か無効かは API で読める。有効と無効を切り替える mutation は無く、消す mutation（`deleteProjectV2Workflow`）はある（2026-10-26 に GraphQL のスキーマと試しの Project で確かめた。消す mutation は試していない）

### 影響（Consequences）

- 良い点: `setup` が、欄・選択肢・ビューを API だけで作れる。人が画面で欄の名前を変える手順が要らない
- 良い点: 使われない Status 欄が残らない
- 良い点: API で作るボードが状態ごとの列になり、人の番の状態が列で見える
- 悪い点: ボードの列が 17 になる。決定 5 の「ボードの列が 4 つで済む」は得られない。枠ごとのボードが要るときは、人が画面でボードの列の欄を主な流れの枠に変えるか、主な流れの枠で絞った表のビュー（`主な流れの枠:変更` など）を使う
- 悪い点: 組み込みのワークフローを止める手間がかかる。切り替える API が無い
- 中立: 状態の欄だけ、名前が英語（Status）になる

### 確認方法（Confirmation）

- `setup` の結合テスト（偽の `gh`）で、主な流れの枠を単一選択の欄として作り、Status 欄の名前を変えずに選択肢を 17 の状態に置き換え、置き換える前に人の承認を受けることを確かめる
- 点検の結合テストで、Status 欄に書き込む組み込みのワークフローが有効なときに、基本設計で決めた結果（fail か warn）になることを確かめる
- 試しの Project で、組み込みのワークフローを止める手段（画面で切る、API で消す）を確かめてから、基本設計に書く

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：主な流れの枠は新しい欄、Status 欄は状態の欄

- 良い点: 名前と中身が合い、使われない欄が残らず、`setup` が API だけで作れる
- 良い点: API で作るボードが状態ごとの列になる
- 悪い点: ボードの列が 17 になる
- 悪い点: 組み込みのワークフローを止める手間がかかる

### 案 B：主な流れの枠と状態を新しい欄で作り、Status 欄は使わない

- 良い点: [ADR-20261024-001][adr-model-states] の決定 5 の、名前を変えられないときの形のままで、ADR を改めなくてよい
- 悪い点: Todo・In Progress・Done のままの Status 欄が残り、人が読み違えうる
- 悪い点: API で作るボードが、使われない Status 欄で列を分ける
- 悪い点: 組み込みのワークフローが使われない Status 欄に書き続ける（進行 Bot の欄は書き換えないが、紛らわしい）

### 案 C：人が画面で Status 欄の名前を変える

- 良い点: 決定 5 の形（ボードの列が 4 つ）のままにできる
- 悪い点: `setup` だけで導入を終えられず、人の画面の操作が増える。点検で名前が合うかを確かめる必要がある
- 悪い点: 組み込みのワークフローを止める手間は、案 A と同じにかかる

### 案 D：Status 欄を、名前を変えずに主な流れの枠として使う

- 良い点: API で作るボードが枠ごとの 4 つの列になる
- 悪い点: 「Status」の欄に枠（依頼・計画・変更・終了）が入り、別に「状態」の欄があるので、名前と中身が合わず、人が読み違えうる
- 悪い点: 組み込みのワークフローを止める手間は、案 A と同じにかかる

## 補足情報（More Information）

- 2026-10-26 に、要件の UNDECIDED-85 を確かめ、Status 欄の名前を API で変えられないと分かった（FACT-74。#152）。名前を変えられないときの扱いが、要件の SETUP-1 と基本設計 04（同じ働きの欄を新しく作る）と、UNDECIDED-85 の続きの行（`setup` が人の画面の手順を示す）で違っていた。メンテナが、主な流れの枠は新しい欄で作り、Status 欄は状態の欄として使う形（案 A）を出し、本 ADR で決めることにした
- 本 ADR で決めないこと
  - 組み込みのワークフローを止める手段と、点検での確かめ方（基本設計）
  - 軸ごとのビューの名前と絞り込みの書き方（基本設計。[ADR-20261024-001][adr-model-states] の決定 5 のビューの例のまま）
- 関連する要件：FLOW-39（[01-issue-flow.md][flow]）、SETUP-1、SETUP-5（[06-setup-and-stages.md][setup]）、UNDECIDED-85（[09-undecided.md][undecided]）、FACT-15、FACT-74（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261024-001][adr-model-states]
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 01-issue-flow.md：FLOW-39 の表の、主な流れの枠と状態の行
  - 06-setup-and-stages.md：SETUP-1 の「Plugin が作れるもの」と、SETUP-5
  - 基本設計の「[設定][design-settings]」：`project.fields.frame` と `project.fields.state` の説明
  - 基本設計の「[導入と点検][design-setup]」：`setup` が作るものの表の Project の行、組み込みのワークフローの扱いと点検
  - [ADR-20261024-001][adr-model-states]：タイトルの下に、本 ADR で決定 5 の一部を改めた旨の注記を足す

[adr-model-states]: ./adr-20261024-001-work-unit-main-flow-and-overlay-axes.md
[flow]: ../requirements/01-issue-flow.md
[setup]: ../requirements/06-setup-and-stages.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-settings]: ../design/basic/01-settings.md
[design-setup]: ../design/basic/04-setup-and-check.md
[doc-workflows]: https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project/using-the-built-in-automations
