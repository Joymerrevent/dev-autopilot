---
status: accepted
created: 2026-10-28
updated: 2026-10-30
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 当てる先のリポの CI の秘密は base ブランチだけが使える Environment に置き、CI の定義は今のまま人が push する

要件の UNDECIDED-89（AI の作業ブランチの CI を、秘密を持たない形で動かすか。秘密を持たない形にしたら、push で効く組を作業 AI に変えさせてよいか。[決めていないこと][undecided]の「ADR で決めること」の「安全と設定」）を決める。
本 ADR は、[ADR-20261024-004][adr-master] の決定 9 の最後の箇条（AI の作業ブランチの CI を秘密を持たない形で動かすかは別の ADR で決め、決めた後に push で効く組の扱いを緩めるかを決め直す）で、別に決めるとしたことを決める。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261024-004][adr-master]（マスターの下書き、検疫、変えてはいけないパス） | 据え置く（決定 9 の最後の箇条で別に決めるとしたことを、本 ADR で決める） | — | 決定 9 の push で効く組の扱い（認めていても AI の変更を push せず、人が push する）を、緩めずに据え置く |

## 背景と課題（Context and Problem Statement）

- 同じリポのブランチから出した PR の CI は、PR の中の CI の定義で、リポの秘密を持って動く（FACT-75）。CI の定義を変えなくても、作業 AI が書いたコード（テストなど）は、今の CI の定義のまま、秘密を渡されたステップで動きうる
- 作業ブランチで足したワークフローは、リポのトークンの既定が読むだけでも、`permissions: contents: write` を求めれば、PR の CI で書き込みのトークンを得る（FACT-78）。CI の定義を変えられれば、秘密が無くても、作業ブランチの CI からリポに書き込める
- Environment の秘密は、その Environment を使うジョブだけが読め、使えるブランチの規則は実行の `GITHUB_REF` と照らす。PR の CI の `GITHUB_REF` は `refs/pull/<番号>/merge` なので、使えるブランチを「選んだブランチ」で base ブランチだけにすれば、PR の CI からは読めない見込み。「Protected branches only」はブランチ保護の規則を見るので、ruleset だけで守るリポでは絞れない。無料のプランでは、非公開のリポで Environment を使えない（FACT-79。公式文書で確かめ、実機では確かめていない）
- 2026-10-28 の時点で、当てる先の business-os（公開のリポ）の CI は秘密を使っていない。リポと組織の秘密が届かず、Environment も無く、リポのトークンの既定は読むだけ

課題：当てる先のリポの CI の秘密を、作業 AI のコードから届かない形にするか。そうしたら、push で効く組（CI の定義など）を作業 AI に変えさせてよいか。

## 判断の決め手（Decision Drivers）

- 作業 AI が書いたコードが、CI で秘密に届かない
- 後から秘密が足されたときに、人の記憶でなく点検で気づける
- CI の定義を作業 AI に変えさせない理由は、秘密だけではない（書き込みのトークン。FACT-78）
- 当てる先のリポの手間を増やしすぎない（今の business-os は秘密を使っていない）

## 検討した選択肢（Considered Options）

- 案 A：秘密は、使えるブランチを base ブランチだけにした Environment に置く。点検で、リポと組織の秘密が届いていないことと、Environment の使えるブランチを確かめる。push で効く組は、今のまま人が push する
- 案 B：秘密の置き方は決めず、今のまま push で効く組を人が push することだけで守る
- 案 C：案 A に加えて、push で効く組の CI の定義を作業 AI に変えさせる
- 案 D：CI の定義の中で、作業ブランチ（`<型>/issue-*`）の実行には秘密を渡さない条件（`if:`）を書く

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: 秘密を base ブランチだけが使える Environment に置けば、作業 AI のコードは、CI の定義を変えずとも変えても、PR の CI で秘密に届かない。点検で確かめるので、後から秘密が足されても気づける。CI の定義は、秘密が無くても書き込みのトークンを求められる（FACT-78）ので、作業 AI に変えさせない。今の business-os は秘密を使っていないので、すぐの手間は無い。

### 決定 1：秘密は、base ブランチだけが使える Environment に置く

- 当てる先のリポの CI が秘密を使うときは、Environment に置き、Environment の使えるブランチを「選んだブランチ」（Selected branches and tags）で、base ブランチと、人が決めたブランチ（`main` など）だけにする。作業ブランチの形（`<型>/issue-*`）に当たる規則は入れない。「No restriction」と「Protected branches only」は使わない
- リポの秘密と組織の秘密は、当てる先のリポに届かないようにする（リポの秘密を置かず、組織の秘密の届くリポから外す）
- 非公開のリポで秘密が要るときは、Environment に有料のプランが要る（FACT-79）。無料のプランの非公開のリポでは、CI で秘密を使わない

  > **［2026-10-30 追記］** 無料のプランの組織の非公開のリポでも、Environment の秘密は CI に渡り、「選んだブランチ」の規則も効いた（[FACT-102][facts]）。公式文書は、GitHub Free では Environment の秘密は公開のリポだけ、使えるブランチの規則は非公開のリポでは有料のプランだけ、と書いていて、実機と食い違う。決定は変わらない。

### 決定 2：点検で、秘密の置き方を確かめる

- 点検は、リポの秘密が無いこと、組織の秘密が当てる先のリポに届いていないこと、Environment があれば使えるブランチが「選んだブランチ」で、作業ブランチの形（`<型>/issue-*`）と PR（`refs/pull/*`）に当たらないことを確かめ、外れていれば fail にする
- 確かめる人と権限（人の `check` の `gh` のログインで読むか、進行 Bot の GitHub App に秘密の名前を読む権限を足すか）と、`setup` が人に示す手順は、基本設計で決める

### 決定 3：CI の定義は、今のまま人が push する

- [ADR-20261024-004][adr-master] の決定 9 の push で効く組（CI の定義、CI が信じて動かすスクリプト、AI のセッションが作業場所から読み込む設定）は、秘密を Environment に置いた後も、緩めない。作業 AI の変更を push せず、人が push する

### 影響（Consequences）

- 良い点: 作業 AI のコードが、PR の CI で秘密に届かない
- 良い点: 後から秘密が足されたときに、点検で気づける
- 良い点: 今の business-os には、すぐの手間が無い
- 悪い点: CI の定義の変更は、今のまま人が push する手間が残る
- 悪い点: ほかのリポに広げるとき（UNDECIDED-82）、非公開のリポで秘密が要るなら、有料のプランが要る
- 悪い点: FACT-79 は実機で確かめていない。決定 1 の守りが効くことは、公開の試し用リポか有料のプランの組織で、実機で確かめる（確認方法）

  > **［2026-10-30 追記］** 試し専用の組織の公開の試し用リポで、実機で確かめた（[FACT-99〜101][facts]）。結果は確認方法の追記にある。

### 確認方法（Confirmation）

- 実機の試し（公開の試し用リポか、有料のプランの組織）で、使えるブランチを `main` だけにした Environment の秘密が、作業ブランチの PR の CI から読めず、`main` への push の CI からは読めることを確かめ、FACT に載せる

  > **［2026-10-30 追記］** 試し専用の組織の公開と非公開の試し用リポで確かめ、FACT-99〜102 に載せた（[確かめた事実][facts]）。使えるブランチを `main` だけにした Environment の秘密は、`main` への push の CI だけが読め、作業ブランチへの push の CI と PR の CI では、Environment を使うジョブが手順を 1 つも動かさずに失敗した（FACT-99）。PR の CI は、規則を `refs/pull/<番号>/merge` と照らし、head のブランチとは照らさない（FACT-99）。「Protected branches only」は、ruleset だけで守るリポでは作業ブランチへの push の CI と PR の CI にも秘密を渡し、従来のブランチ保護が `main` にあれば拒んだ（FACT-100）。規則の書き方は Ruby の `File.fnmatch`（`File::FNM_PATHNAME`）と同じに当たり、`refs/pull/*/merge` と `**/*` の規則は PR の CI に秘密を渡した（FACT-101）。決定 1 の守りは効く。決定は変わらない。

- 点検の単体テストで、リポの秘密があるとき、組織の秘密が届くとき、Environment の使えるブランチが「No restriction」か「Protected branches only」か、作業ブランチの形に当たる規則があるときに、fail になることを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：base ブランチだけの Environment に置き、点検で確かめる

- 良い点: 作業 AI のコードが秘密に届かず、点検で守りが崩れていないかを確かめられる
- 悪い点: 非公開のリポでは有料のプランが要る

### 案 B：秘密の置き方を決めない

- 良い点: 決まりが増えない
- 悪い点: 後から CI に秘密を足すと、作業 AI のコードが PR の CI で秘密に届く。気づく仕組みが無い

### 案 C：案 A に加えて、CI の定義を作業 AI に変えさせる

- 良い点: CI の定義の変更を、人が push しなくて済む
- 悪い点: CI の定義を変えれば、秘密が無くても書き込みのトークンを求められる（FACT-78）。作業ブランチの CI からリポに書き込める

### 案 D：CI の定義の中で、作業ブランチの実行に秘密を渡さない

- 良い点: Environment が要らず、無料のプランの非公開のリポでも使える
- 悪い点: ワークフローごとに条件を書く必要があり、書き忘れを点検で見つけにくい。条件の書き方を誤ると、黙って秘密が渡る

## 補足情報（More Information）

- 2026-10-28 に、business-os の CI の秘密の使い方を読み、試し専用の組織の非公開のリポで、書き込みのトークンを確かめた（FACT-78）。Environment の秘密は、公開の試し用リポを作れなかったので、公式文書で確かめた（FACT-79）。UNDECIDED-89 と FACT-78・79 は #166 で足した
- 本 ADR で決めないこと
  - 点検で秘密の置き方を読む権限と、`setup` が人に示す手順（基本設計）
  - 自前の実行環境（self-hosted runner）。Environment の秘密も守られない（公式文書）。使うなら別に決める
- 関連する要件：IMPL-15（[02-implement-review-fix.md][impl]）、SAFE-8（[05-safety.md][safety]）、SETUP-1、SETUP-2（[06-setup-and-stages.md][setup]）、UNDECIDED-82、UNDECIDED-89（[09-undecided.md][undecided]）、FACT-75、FACT-78、FACT-79（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261024-004][adr-master]
- 要件と設計への反映（承認の後に、別の PR で 1 本にまとめて行う）
  - 05-safety.md：CI の秘密の置き方の要件を足す
  - 06-setup-and-stages.md：SETUP-1（人がやるもの）と SETUP-2（点検の項目）
  - 09-undecided.md：UNDECIDED-89 を欠番にする（元の文を取り消し線で残す）
  - 基本設計の「[導入と点検][design-setup]」：点検の GitHub の組、`setup` が示す手順
  - [ADR-20261024-004][adr-master]：決定 9 の最後の箇条に、本 ADR で決めた旨の追記を足す

[adr-master]: ./adr-20261024-004-master-draft-quarantine-and-protected-paths.md
[impl]: ../requirements/02-implement-review-fix.md
[safety]: ../requirements/05-safety.md
[setup]: ../requirements/06-setup-and-stages.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-setup]: ../design/basic/04-setup-and-check.md
