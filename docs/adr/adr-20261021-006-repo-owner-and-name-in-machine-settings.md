---
status: proposed
created: 2026-10-21
updated: 2026-10-21
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 当てる先のリポの owner と名前は、リポの設定を読む前に要る項目として、base ブランチの名前と GitHub App の 2 つの ID と一緒にマシンの設定に置く

要件の UNDECIDED-65（当てる先のリポの owner と名前をマシンの設定に置くことを、ADR でどう残すか）を決める。

改める決定と、据え置く決定は次のとおり。

- [ADR-20261015-005][adr-settings]：決定 2 の割り振りのうち、リポの設定の「リポと Project の ID」から当てる先のリポの owner と名前を外し、マシンの設定に移す。Project の ID（Project の番号、Status と Agent 欄の ID と選択肢の ID）はリポの設定に据え置く。
  ほかの項目の割り振り、項目ごとに置き場を 1 つにすること、決定 1 は据え置く
- [ADR-20261017-007][adr-stop]：決定 4（base ブランチの名前をマシンの設定に置き、進行 Bot は base ブランチの先端からリポの設定を読む）と、決定 1・2（全体の止める合図を当てる先のリポの変数に置き、実行の最初から読む）は据え置く
- [ADR-20261020-002][adr-app]：決定 3（GitHub App の ID とインストールの ID を設定に持ち、どちらの設定に置くかは基本設計で決める）は据え置く。基本設計の「[設定][design-settings]」はマシンの設定に置いた。本 ADR は、owner と名前と同じ理由の項目として、2 つの ID の置き場も決定に含める

## 背景と課題（Context and Problem Statement）

- [ADR-20261015-005][adr-settings] の決定 2 は、設定をリポの設定（当てる先のリポの `.claude/dev-autopilot.json`。commit し、PR で変える）とマシンの設定（commit しない）に分け、「リポと Project の ID」をリポの設定に置く。
  [ADR-20261017-007][adr-stop] は、base ブランチの名前だけをマシンの設定に移し、ほかの項目の割り振りは据え置くとした
- 進行 Bot は、1 回の実行の最初に、当てる先のリポの変数から全体の止める合図を読み（[ADR-20261017-007][adr-stop] の決定 1・2）、続けて進行 Bot の clone（`<置き場>/controller/`）で fetch した base ブランチの先端からリポの設定を読む（同じ ADR の決定 4）。
  基本設計の「[状態遷移と判定][design-flow]」の「1 回の実行の流れ」では、順 3 で合図を読み、順 4 でリポの設定を読む
- 合図を読むには、読む先のリポ（owner と名前）と認証が要る。GitHub App に切り替えた後の認証には、アプリの ID とインストールの ID が要る（[ADR-20261020-002][adr-app] の決定 3）。
  進行 Bot の clone は、`setup` では作らず、進行 Bot が最初の実行で作る（基本設計の「[導入と点検][design-setup]」）ので、clone を作る先の owner と名前も、リポの設定を読む前に要る
- マシンの設定のフォルダは、OS ごとの決まった場所に、当てる先のリポ（owner/repo）ごとに置く（[ADR-20261017-004][adr-human] の決定 3）。人が呼ぶ Skill は、開いているプロジェクトの origin から owner/repo を引いてフォルダを見つける
- 基本設計の「設定」の最初の版は、owner と名前（`repository.owner`、`repository.name`）をリポの設定に置いていた。
  基本設計の再レビュー（2026-10-20）で、GitHub App に切り替えた後は合図を読む認証が無く、owner と名前もリポの設定にしか無いと分かった。直しで、要件の STRUCT-2 と基本設計の「設定」は、owner と名前、base ブランチの名前、GitHub App の 2 つの ID をマシンの設定に置く形になった
- 基本設計の「設定」は、owner と名前をマシンの設定に置く根拠に [ADR-20261017-007][adr-stop] の決定 4 を「base ブランチと同じ理由」として挙げる。決定 4 が移したのは base ブランチの名前だけで、owner と名前の置き場はどの ADR も決めていない。
  基本設計の 2 回目の再レビュー（2026-10-21）で、ADR だけを読む人は owner と名前をリポの設定に置くと分かった

課題：当てる先のリポの owner と名前をどちらの設定に置くか。置き場を、ADR でどう残すか。

## 判断の決め手（Decision Drivers）

- 進行 Bot が、リポの設定を読む前に、合図を読む先と認証と clone を作る先を決められること
- 全体の止める合図を実行の最初に読む順（[ADR-20261017-007][adr-stop] の決定 2）を変えないこと
- 項目ごとに置き場を 1 つにし（[ADR-20261015-005][adr-settings] の決定 2）、`check` が確かめられること（SETUP-2）
- PR を通らない置き場の値が食い違ったときに、作業に入らず止まる側に倒れること（フェイルセーフ。SAFE-4）
- ADR だけを読む人が、要件と基本設計と同じ置き場を読めること（ADR が設計判断の正典）

## 検討した選択肢（Considered Options）

問い 1：owner と名前の置き場

- 案 1-A：マシンの設定に、base ブランチの名前と GitHub App の 2 つの ID と一緒に置く
- 案 1-B：リポの設定に置き（[ADR-20261015-005][adr-settings] の決定 2 のまま）、合図を読む前に進行 Bot の clone で fetch してリポの設定を読む
- 案 1-C：項目に持たず、マシンの設定のフォルダのパス（`<owner>/<repo>`）から引く

問い 2：置き場を ADR でどう残すか

- 案 2-A：新しい ADR（本 ADR）で決め、承認の後に [ADR-20261015-005][adr-settings] へ「Amended by」の注記を足す
- 案 2-B：[ADR-20261015-005][adr-settings] の決定 2 の「リポの ID」に owner と名前は入らないと読み、決定 2 に訂正の注記を足す
- 案 2-C：ADR に残さず、要件の STRUCT-2 と基本設計の「設定」だけに書いておく

## 決定（Decision Outcome）

採用した選択肢:「案 1-A」「案 2-A」。
理由: リポの設定を読む前に要る値を、読む前に読める置き場に持てるので、合図を実行の最初に読む順を変えずに済む。base ブランチの名前と同じ理由の項目が 1 か所にそろう。
置き場を変える判断は、ADR の README の基準（注記を読んだ人が、決定が変わったと思うか）では決定の変更に当たるので、訂正の注記でなく新しい ADR で残す。

### 決定 1：リポの設定を読む前に要る 5 項目を、マシンの設定に置く

- 次の 5 項目をマシンの設定に置く。キーは基本設計の「[設定][design-settings]」のもの
  - 当てる先のリポの owner（`repository.owner`）と名前（`repository.name`）：進行 Bot の clone を作る先と fetch する先、全体の止める合図を読む先
  - base ブランチの名前（`baseBranch`）：リポの設定を読むブランチ（[ADR-20261017-007][adr-stop] の決定 4 のまま）
  - GitHub App の ID（`githubApp.appId`）とインストールの ID（`githubApp.installationId`）：合図を読む前の認証（[ADR-20261020-002][adr-app] の決定 3 が任せた置き場を、基本設計の「設定」のとおりマシンの設定にする）
- 5 項目はどれも秘密でない。PAT とアプリの秘密鍵は、今のとおり秘密の置き場のファイルに置く（[ADR-20261015-001][adr-read] の決定 2、[ADR-20261020-002][adr-app] の決定 3）
- 本 ADR でリポの設定から移すのは、owner と名前だけ。Project の ID はリポの設定に残す。リポの設定は当てる先のリポの中にあるので、どのリポの設定かはファイルの場所で決まり、リポの設定に owner と名前は要らない

### 決定 2：owner と名前は setup が origin から書き、点検でマシンの設定のフォルダのパスと照らす

- `setup` は、開いているプロジェクトの origin から owner と名前を引き、マシンの設定に書く。マシンの設定のフォルダを見つけるときと同じ origin を使う（[ADR-20261017-004][adr-human] の決定 3）
- 点検（SETUP-2）は、マシンの設定の owner と名前が、マシンの設定のフォルダのパス（`<owner>/<repo>`）と一致することを確かめる。人が呼ぶ `check` では、開いているプロジェクトの origin とも一致することを確かめる。食い違えば fail にし、作業に入らない
- 点検は、owner と名前がリポの設定にもあれば、項目が一方にだけあることの確かめで fail にする（[ADR-20261015-005][adr-settings] の決定 2）

### 決定 3：ADR-20261015-005 の決定 2 を、訂正の注記でなく本 ADR で改める

- 承認の後に、[ADR-20261015-005][adr-settings] のタイトルの下に、決定 2 の割り振りのうち当てる先のリポの owner と名前を本 ADR でマシンの設定に移した旨の「Amended by」の注記を足す
- 要件の STRUCT-2 と、基本設計の「設定」の 5 項目の根拠は、本 ADR を指す。base ブランチの名前は [ADR-20261017-007][adr-stop] の決定 4 も、GitHub App の 2 つの ID は [ADR-20261020-002][adr-app] の決定 3 も、今のとおり指す
- ほかの項目をリポの設定とマシンの設定の間で移すときも、今のとおり ADR で決める

### 影響（Consequences）

- 良い点: ADR だけを読む人も、owner と名前をマシンの設定に置くと読める。要件の STRUCT-2、基本設計、ADR が同じ置き場を指す
- 良い点: 進行 Bot は、リポの設定を読む前に、合図を読む先と認証と clone を作る先を決められる。合図を実行の最初に読む順は変わらない
- 良い点: リポの設定を読む前に要る項目が、同じ理由でマシンの設定にそろう
- 悪い点: owner と名前は PR を通らない置き場にあり、人がマシンで書き換えられ、書き換えた記録が git に残らない。点検がマシンの設定のフォルダのパスと照らし（決定 2）、PAT は当てる先のリポだけに絞り（基本設計の「[導入と点検][design-setup]」の「PAT の権限」）、GitHub App も当てる先のリポだけに入れる（[ADR-20261020-002][adr-app] の決定 2）ことで補う
- 中立: 進行 Bot を動かすマシンを替えるときは、新しいマシンで `setup` を実行して owner と名前を書く。マシンの設定を `setup` で作り直すこと（[ADR-20261015-005][adr-settings] の影響）は今と同じ

### 確認方法（Confirmation）

- 承認の後の反映の PR のレビューで、[ADR-20261015-005][adr-settings] の注記、STRUCT-2、基本設計の「設定」の 5 項目の根拠が本 ADR を指すことを確かめる
- 導入と点検の実装（段階 3。`setup` と `check` を作る段階）の単体テストで、次を検査する
  - owner と名前が、マシンの設定のフォルダのパスと食い違えば fail にすること。人が呼ぶ `check` では、origin と食い違っても fail にすること
  - owner と名前がリポの設定にもあれば fail にすること
- 結合テストで、進行 Bot の clone が無い状態の最初の実行が、マシンの設定の owner と名前で合図を読み、clone を作ってからリポの設定を読むことを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 1-A：マシンの設定に、base ブランチの名前と GitHub App の 2 つの ID と一緒に置く

- 良い点: リポの設定を読む前に要る値を、読む前に読める。合図を実行の最初に読む順を変えない
- 良い点: base ブランチの名前と同じ理由・同じ置き場で、読む前に要る項目がそろう
- 悪い点: 値が PR を通らない。点検でマシンの設定のフォルダのパスと照らして補う

### 案 1-B：リポの設定に置き、合図を読む前に fetch してリポの設定を読む

- 良い点: owner と名前が PR を通る置き場に残る。[ADR-20261015-005][adr-settings] の割り振りを変えない
- 悪い点: 最初の実行には進行 Bot の clone が無く、clone を作る先をリポの設定から知れない。別の出どころが要り、同じ値を 2 か所に持つ
- 悪い点: 合図を読む前に、deploy key で fetch する。合図を実行の最初に読む順（[ADR-20261017-007][adr-stop] の決定 2）を改める

### 案 1-C：項目に持たず、マシンの設定のフォルダのパスから引く

- 良い点: 項目が 2 つ減り、項目とパスの食い違いが起きない
- 悪い点: どのリポに当てるかが、定期の実行の定義が渡すパスの読み方で決まる。パスの決まり（[ADR-20261017-004][adr-human] の決定 3。場所の値は構造仕様で決める）を変えると、当てる先の読み方も変わる
- 悪い点: 項目とフォルダのパスを照らして、2 つの出どころの食い違いで止める点検ができない

### 案 2-A：本 ADR で決め、ADR-20261015-005 に「Amended by」の注記を足す

- 良い点: ADR だけを読む人も、置き場と理由と検討した選択肢を読める。ADR の README の「決定の一部だけを改める」の直し方に合う
- 悪い点: ADR が 1 本増える

### 案 2-B：ADR-20261015-005 の決定 2 に訂正の注記を足す

- 良い点: ADR を増やさない
- 悪い点: [ADR-20261015-005][adr-settings] の決定 2 は「リポと Project の ID」をリポの設定に置き、[ADR-20261017-007][adr-stop] は base ブランチの名前だけを移して、ほかの項目の割り振りは据え置くとしている。
  注記を読んだ人は決定が変わったと読むので、ADR の README の基準では、訂正でなく決定の変更に当たる
- 悪い点: 選んだ理由と、検討した選択肢が残らない

### 案 2-C：ADR に残さない

- 良い点: 手間が無い
- 悪い点: ADR だけを読む人は、owner と名前をリポの設定に置く。設計判断の正典である ADR と、要件と基本設計が食い違ったまま残る

## 補足情報（More Information）

- 関連する要件：STRUCT-2（[07-structure.md][struct]）、SETUP-1、SETUP-2（[06-setup-and-stages.md][stages]）、「運用の規則」の止める合図（[01-issue-flow.md][flow]）、SAFE-4（[05-safety.md][safety]）、UNDECIDED-65（[09-undecided.md][undecided]）
- 関連する ADR：[ADR-20261015-005][adr-settings]、[ADR-20261017-004][adr-human]、[ADR-20261017-007][adr-stop]、[ADR-20261020-002][adr-app]、[ADR-20261015-001][adr-read]
- 関連する基本設計：「[設定][design-settings]」の「マシンの設定の項目」と「`check` が設定について確かめること」、「[状態遷移と判定][design-flow]」の「1 回の実行の流れ」の順 1〜4、「[導入と点検][design-setup]」の進行 Bot の clone を最初の実行で作ること
- 要件と設計への反映（承認の後に、別の PR で行う）
  - [ADR-20261015-005][adr-settings]：タイトルの下に、決定 2 の割り振りのうち当てる先のリポの owner と名前を本 ADR でマシンの設定に移した旨の注記を足す（決定 3）
  - 07-structure.md：STRUCT-2 の「リポと Project」の行で、マシンに置く理由の出典に本 ADR を足す
  - 09-undecided.md：UNDECIDED-65 を欠番にする（元の文を取り消し線で残す）
  - 基本設計の「設定」：`repository.*` の根拠を、[ADR-20261017-007][adr-stop] の決定 4 を「同じ理由」として挙げる形から、本 ADR を指す形に直す。`githubApp.*` の根拠と「設定ファイルの外の合図」の節にも本 ADR を足す
  - [ADR-20261017-007][adr-stop] と [ADR-20261020-002][adr-app] には注記を足さない（決定 4 と決定 3 は据え置く）

[adr-settings]: ./adr-20261015-005-settings-values-and-machine-settings.md
[adr-stop]: ./adr-20261017-007-stop-switch-in-repo-variable.md
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-human]: ./adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-read]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[design-settings]: ../design/basic/01-settings.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
[design-setup]: ../design/basic/04-setup-and-check.md
[struct]: ../requirements/07-structure.md
[stages]: ../requirements/06-setup-and-stages.md
[flow]: ../requirements/01-issue-flow.md
[safety]: ../requirements/05-safety.md
[undecided]: ../requirements/09-undecided.md
