---
status: accepted
created: 2026-11-01
updated: 2026-11-01
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 作業 AI のモデルを上げたことは、fix の印の model の欄と決まった文で残す

要件の UNDECIDED-90（作業ごとに使うモデルの決め方と、モデルを上げたことを残す印。[決めていないこと][undecided]の「ADR で決めること」の「流れと工程」）のうち、モデルを上げたことを残す印を決める。
作業ごとに使うモデルの決め方は、[ADR-20261031-002][adr-same-model]（作業ごとに使うモデルを変えず、役ごとに設定の値を使う）で決めた。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261024-001][adr-model-states]（作業の単位の主な流れと重ねる軸） | 一部を改める | 決定 9 の印の表の `fix` の行の欄（`round`、`head`、`cause`）に `model` を足す。決定 9 の「今の `note kind=raise` は、作業ごとに使うモデルの決め方を決めるときに一緒に決める」を、本 ADR の決定とする | 決定 9 の印の種類の数（21）、`fix` のほかの欄と書くとき、決定 9 のほかの行と、ほかの決定 |
| [ADR-20261024-002][adr-no-stages]（段階 4〜8 と書かずに試すことを持たない） | 据え置く（決定 7 で別に決めるとしたことのうち、モデルを上げたことの印を、本 ADR で決める） | — | 決定 7 のすべて |

## 背景と課題（Context and Problem Statement）

- IMPL-25（詰まったときの上げ方）：最初の実装が `sonnet` だった作業は、作業 AI の 2 ラウンド目（最初の修正のセッション）から `opus` に 1 段上げる。上げたことは記録と Issue のコメントに残すとし、印の名前は決めていない
- 上げるのは、リポの設定で `models.work` を `sonnet` にしたときだけ（初期値は `opus`。[ADR-20261031-002][adr-same-model]）。上げるかは、修正のときに、今の設定の `models.work` から進行 Bot が決め、手元の記録は判定に使わない（基本設計の「[設定][design-settings]」の `models.workRaised`）。衝突の直しのセッションも、修正と同じ決め方で上げる
- 手元の記録（FLOW-5）は、AI のセッションごとに、役、ラウンド、指定したモデルと effort、実際に使われたモデルを残す
- GitHub の印では、計画の `plan` の印とレビューの `review` の印が `model` の欄を持つ。`review` の印の `model` には、そのラウンドを起こしたモデル（`--model` に渡した値）を書く。計画 AI を `fable` に上げたことは、`plan` の印の `model` が `fable` であることで分かる（IMPL-10）
- 作業 AI の修正は、進行 Bot が push の前に PR に書く `fix` の印（欄は `round`、`head`、`cause`）を持つ。進行 Bot が投稿するコメントは、すべて署名付きの印を 1 つ持つ

課題：作業 AI のモデルを上げたことを、GitHub の上でどう残すか。

## 判断の決め手（Decision Drivers）

- 読みやすさ：GitHub の上で、どの AI のセッションがどのモデルで動いたかを、同じ形で読める
- 変更のしやすさ：印の種類（印の一覧と、その実装の正本 `plugin/src/marker.ts`）を増やさない
- 記録が欠けたときの強さ：手元の記録を失っても、どのモデルで直したかを GitHub から分かる

## 検討した選択肢（Considered Options）

- 案 A：`fix` の印に `model` の欄を足し、上げた修正のコメントに決まった文を書く
- 案 B：新しい種類 `raise`（Issue に書く。欄はラウンドとモデル）を足し、上げたときに 1 回だけ書く
- 案 C：印の欄も種類も足さない。手元の記録と、`fix` のコメントの決まった文だけに残す

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: `plan` と `review` の印と同じ `model` の欄を `fix` にも持てば、計画・レビュー・修正のどの AI のセッションのモデルも、GitHub の上で同じ形で読める。印の種類は増えず、上げたことは上げた修正の `fix` の印のコメントで読める。

- `fix` の印に `model` の欄を足す。値は、その修正（`cause` が `check`・`review`・`conflict` のどれでも）で作業 AI を起こしたモデル（`--model` に渡した値）
- 最初の実装が `sonnet` で、修正を `models.workRaised` で起こしたときは、その `fix` の印のコメントに、最初の実装のモデルから上げたことを決まった文で書く（AI が書いた文は書き添えない）
- 上げたことを知らせる別のコメントは、Issue に書かない。手元の記録（FLOW-5）は今のまま
- 組み直す前の `note kind=raise` は、`fix` の印の `model` の欄と、決まった文に置き換える

### 影響（Consequences）

- 良い点: 計画・レビュー・修正のモデルを、`plan`・`review`・`fix` の印の同じ `model` の欄で読める（読みやすさ）
- 良い点: 印の種類は 21 のまま（変更のしやすさ）
- 良い点: 手元の記録を失っても、どのモデルで直したかが PR の `fix` の印に残る（記録が欠けたときの強さ）
- 悪い点: 上げない修正（設定の初期値のとき、すべての修正）でも `model` を書き、`fix` の印が 1 欄長くなる
- 悪い点: Issue だけを見る人は、上げたことに気づかない。PR の `fix` のコメントを読む

### 確認方法（Confirmation）

- 単体テストで、`fix` の印の `model` が、その修正で `--model` に渡した値と同じことを確かめる（`cause` の 3 つのどれでも）
- 単体テストで、`models.work` が `sonnet` のときに、修正を `models.workRaised` で起こし、その `fix` の印のコメントに決まった文があり、Issue に別のコメントを書かないことを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：`fix` の印に `model` の欄を足す

- 良い点: 計画・レビュー・修正のモデルが、同じ欄の名前で GitHub に残る（読みやすさが上がる）
- 良い点: 上げなかった修正のモデルも残るので、手元の記録を失っても、どの修正をどのモデルで直したかが分かる（記録が欠けたときの強さが上がる）
- 悪い点: `fix` の印の欄が 1 つ増える（変更のしやすさが少し下がる）

### 案 B：新しい種類 `raise` を Issue に書く

- 良い点: Issue だけを見る人にも、上げたことが伝わる（Issue で作業を追う人の読みやすさが上がる）
- 悪い点: 印が 22 種類になり、印の実装と一覧で扱う種類が増える（変更のしやすさが下がる）
- 悪い点: 同じ作業で何度も書かないための「1 回だけ書く」判定が要る
- 悪い点: 上げたときのモデルしか GitHub に残らず、上げなかった修正のモデルは分からない

### 案 C：印を変えず、手元の記録と決まった文だけに残す

- 良い点: 印の欄も種類も増えず、印の実装で扱う欄が最も少ない（変更のしやすさが上がる）
- 悪い点: GitHub の上では、どのモデルで直したかを決まった文でしか読めず、機械で読めない（読みやすさが下がる）
- 悪い点: 手元の記録を失うと、上げなかった修正のモデルが分からない（記録が欠けたときの強さが下がる）

## 補足情報（More Information）

- 2026-10-31 に、CC が UNDECIDED-90 を 2 つに分け、1 つ目を [ADR-20261031-002][adr-same-model] で決めた後、2 つ目について案 A〜C を示した。2026-11-01 にメンテナが推奨案（案 A）で決めた
- 本 ADR で決めないこと
  - 上げた修正のコメントの決まった文の言い回し（基本設計）
  - 印の署名に使う JSON の並べ方（詳細設計）
- 関連する要件：IMPL-10、IMPL-25、IMPL-35、「モデルの流れ」（[02-implement-review-fix.md][impl]）、「印と固定の見出しの一覧」（[04-external-text-and-markers.md][text]）、FLOW-5（[01-issue-flow.md][flow]）、UNDECIDED-90（[09-undecided.md][undecided]）
- 関連する ADR：[ADR-20261024-001][adr-model-states]、[ADR-20261024-002][adr-no-stages]、[ADR-20261031-002][adr-same-model]
- 要件と設計への反映（承認の後に、[ADR-20261031-002][adr-same-model] の反映と 1 本の PR にまとめて行う）
  - 02-implement-review-fix.md：IMPL-25 の上げたことの残し方と、「モデルの流れ」の表の IMPL-35 の行
  - 04-external-text-and-markers.md：印の表の `fix` の行と、表の下の `note kind=raise` の文
  - 09-undecided.md：UNDECIDED-90 を欠番にする（元の文を取り消し線で残す）
  - 基本設計の「[状態遷移と判定][design-flow]」：「ほかの印」の下の `note kind=raise` の文。`fix` の印を書く所
  - [ADR-20261024-001][adr-model-states]：タイトルの下に、改めた旨の注記を足す。[ADR-20261024-002][adr-no-stages] の決定 7：本 ADR と [ADR-20261031-002][adr-same-model] で決めた旨の追記を足す

[adr-model-states]: ./adr-20261024-001-work-unit-main-flow-and-overlay-axes.md
[adr-no-stages]: ./adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[adr-same-model]: ./adr-20261031-002-same-model-per-role-for-all-tasks.md
[flow]: ../requirements/01-issue-flow.md
[impl]: ../requirements/02-implement-review-fix.md
[text]: ../requirements/04-external-text-and-markers.md
[undecided]: ../requirements/09-undecided.md
[design-settings]: ../design/basic/01-settings.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
