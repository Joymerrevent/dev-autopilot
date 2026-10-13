# dev-autopilot 要件

dev-autopilot（自律開発ループ）の要件を、話題ごとのファイルに分けて書く。

## ファイル

| ファイル | 話題 |
| --- | --- |
| [00-overview.md][f00] | 目的・前提・登場するもの |
| [01-issue-flow.md][f01] | Issue の流れと運用（棚卸し・並列・間隔・状態の遷移・運用の規則） |
| [02-implement-review-fix.md][f02] | 実装・レビュー・対応ループ・モデル |
| [03-merge.md][f03] | マージ |
| [04-external-text-and-markers.md][f04] | 外から入る文と印 |
| [05-safety.md][f05] | 安全側に倒す設計（隔離と鍵） |
| [06-setup-and-stages.md][f06] | 導入・準備・段階 |
| [07-structure.md][f07] | 分離を見越した構成 |
| [08-review-skill.md][f08] | レビューの Skill（change-review v2） |
| [09-undecided.md][f09] | 決めていないこと |
| [10-verified-facts.md][f10] | 実機で確かめた事実 |
| [11-records.md][f11] | 調査とレビューの記録（要件ではない） |

## 読み方

- 参照は、要件の番号（FLOW-1 など。下の索引）か、見出しの名前（「隔離と鍵の分け方」など）で書く。別のファイルを指す見出しの名前は、参照先のファイルへのリンクにしてある
- 用語：**進行役**＝AI ではない Node のスクリプト。**段**＝1 回の実行で進める単位（FLOW-4）。**段階**＝導入の順（1〜9。「[段階][f06]」）。**印**＝進行役がコメントに書く、機械で読む 1 行（「[印と固定の見出しの一覧][f04]」）。**渡す文**＝AI に作業の内容として渡す文（TEXT-1）
- まだ決めていないことは [09-undecided.md][f09] に集めてある。ADR で決めること、基本設計の中で決めてよいもの、確かめることに分けてある
- [11-records.md][f11] は調査とレビューの記録で、要件ではない

## 要件の番号の索引

要件の番号は、ファイル名の話題を頭にして、ファイルの中で 1 から順に振る。

| 番号 | 中身 | ファイル |
| --- | --- | --- |
| FLOW-n | 棚卸し、並列と競合、間隔の運用、Issue の状態の遷移（正常な順と止まる遷移） | [01-issue-flow.md][f01] |
| IMPL-n | 実装、レビュー、対応ループ、モデルの流れ | [02-implement-review-fix.md][f02] |
| MERGE-n | マージ | [03-merge.md][f03] |
| TEXT-n | 外から入る文の扱い | [04-external-text-and-markers.md][f04] |
| SAFE-n | 安全側に倒す設計と、隔離と鍵の分け方 | [05-safety.md][f05] |
| SETUP-n | 導入と点検、準備の作業 | [06-setup-and-stages.md][f06] |
| STRUCT-n | 分離を見越した構成の守ること | [07-structure.md][f07] |
| REVIEW-n | change-review を v2 に組み直す方針と、無人で使うには足りない点 | [08-review-skill.md][f08] |
| UNDECIDED-n | 決めていないこと（ADR で決めること、基本設計の中で決めてよいもの、確かめること） | [09-undecided.md][f09] |
| FACT-n | 実機で確かめた事実 | [10-verified-facts.md][f10] |

## 要件を変えるとき

- 選ぶ判断が要るもの（[09-undecided.md][f09] の「ADR で決めること」など）は、[ADR][adr] を `proposed` で起票し、メンテナが `accepted` にしてから、関係するファイルを直す
- 記述の誤りや言い回しは、PR で直接直す
- 要件の番号は振り直さない。ADR や基本設計は番号で根拠を示すので、振り直すと参照が壊れずに別の要件を指す。
  要件を消すときは、番号の行（箇条か表の行）を残して「欠番」と書き、消した理由か決めた ADR を添える（例：`- UNDECIDED-24 欠番（ADR-<番号> で決めた）`）。
  足すときは、欠番を含めてファイルの中でいちばん大きい番号の次を使う（節の途中に置くときも、番号は変えない）。欠番の番号は、別の要件に使い回さない

## 元の文書

要件は、1 つのファイルだった[要件メモ][memo]（約 900 行。2026-10-07〜11 の検討の記録）と、要件メモから写した[要件定義書][req]（2026-10-11）に書かれていた。
2026-10-12〜13 に要件メモを話題ごとのファイルに分け、古い文と経緯の注記を消して整理し、2 つの文書は消した。2 つの文書へのリンクは、消す前の commit に固定してある。

[f00]: ./00-overview.md
[f01]: ./01-issue-flow.md
[f02]: ./02-implement-review-fix.md
[f03]: ./03-merge.md
[f04]: ./04-external-text-and-markers.md
[f05]: ./05-safety.md
[f06]: ./06-setup-and-stages.md
[f07]: ./07-structure.md
[f08]: ./08-review-skill.md
[f09]: ./09-undecided.md
[f10]: ./10-verified-facts.md
[f11]: ./11-records.md
[adr]: ../adr/README.md
[memo]: https://github.com/Joymerrevent/business-os/blob/b8bc13eef06d430e24483168cfa0ff8edc1987e0/dev-autopilot/docs/dev-autopilot-requirements.md
[req]: https://github.com/Joymerrevent/business-os/blob/b8bc13eef06d430e24483168cfa0ff8edc1987e0/dev-autopilot/docs/requirements.md
