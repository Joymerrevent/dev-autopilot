---
status: accepted
created: 2026-10-31
updated: 2026-10-31
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 🟢 の指摘と同じ見出しの Issue が開いていれば、PR の番号を足すコメントにも follow-up の印を書く

要件の UNDECIDED-92（🟢 の重複の Issue に PR の番号を足すコメント（IMPL-26）に付ける印の名前。[決めていないこと][undecided]の「基本設計の中で決めてよいもの」）を決める。
[ADR-20261024-001][adr-model-states] の決定 9 は、印を 21 種類に組み直したときに、組み直す前の `note kind=duplicate` の扱いを決めずに残した。本 ADR は、その扱いを決める。
UNDECIDED-92 は基本設計の中で決めてよいものに並ぶが、決め方が承認済みの決定 9 の印の表（`follow-up` の行）を改めるので、ADR で決める（[ADR の規則][adr-readme]の「決定の一部だけを改める」）。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261024-001][adr-model-states]（作業の単位の主な流れと重ねる軸） | 一部を改める | 決定 9 の印の表の `follow-up` の行の「書くとき」（🟢 を新しい Issue にした）と「置き場」（新しい Issue）を、本 ADR の決定に改める | 決定 9 の印の種類の数（21）、`follow-up` の欄（`pr`、`finding`）、決定 9 のほかの行、ほかの決定 |

## 背景と課題（Context and Problem Statement）

- IMPL-26（🟢 の指摘の起票）：🟢 は進行 Bot が 1 件 1 Issue にする。同じ見出しの open な Issue があれば、作らずにコメントで PR の番号を足す（重複を防ぐ）。Issue を作って（`follow-up` の印）から、PR を ready にする
- [ADR-20261024-001][adr-model-states] の決定 9 は、組み直す前の `note` の `kind` のうち `duplicate` の扱いを決めなかった。基本設計は、決めるときは `stop` と `link` に寄せないとしている。`stop` に寄せると止まったと読まれ、`link` に寄せると PR と Issue の結びつきを誤るため（基本設計の「[状態遷移と判定][design-flow]」の「印の種類」）
- `follow-up` の印は、新しい Issue を開いた後のコメントに書く（印の署名に Issue の番号が要るため）。基本設計は、進行 Bot のアカウントが起票した Issue を、`follow-up` の印があるときだけ作業の単位にする。`halt` の印も `follow-up` の印も無いもの（Issue を開いた後、印を書く前に実行が切れたもの）は、作業の単位にせず、点検の warn に出す（基本設計の「[状態遷移と判定][design-flow]」の「全体の停止の Issue」）
- レビュー合格に入るとき（T11）、進行 Bot は、PR のどのラウンドの 🟢 も、まだ Issue にしていなければ新しい Issue にし、その後で PR を ready にする
- 全体の停止の Issue は、開いた Issue があれば、新しく開かずに `halt` の印のコメントを足す。新しく開いたときも、開いた後のコメントに同じ `halt` の印を書く
- 進行 Bot のコード（`plugin/`）はまだ無い（2026-10-31 に確かめた）。印を読むものも書くものも無い

課題：🟢 の指摘と同じ見出しの開いた Issue に PR の番号を足すコメントに、どの印を付けるか。

## 判断の決め手（Decision Drivers）

- 確かめやすさ：PR を ready にする前の「どの 🟢 も Issue に記録したか」の確かめを、少ない分かれ道で書き、テストできる
- 読み違えにくさ：印の名前から、起きたことを誤って読まない（`stop` と `link` に寄せない理由と同じ）
- 安全：作業の単位の判定（`follow-up` の印があるときだけ作業の単位にする）を、全体の停止の Issue と、印を書く前に実行が切れた Issue に広げない
- 変更のしやすさ：印の種類（印の一覧と、その実装の正本 `plugin/src/marker.ts`）を増やさない
- 人の読む量：同じ指摘の Issue を並べない（IMPL-26 が重複を防ぐ目的）

## 検討した選択肢（Considered Options）

- 案 A：`follow-up` の印を書く。足す先は、有効な `follow-up` の印を持つ、同じ見出しの開いた Issue に限る
- 案 B：新しい種類 `duplicate`（欄は `pr`、`finding`）を足し、同じ見出しの開いた Issue に書く

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: PR を ready にする前の確かめを、`follow-up` の印 1 種類で書ける。新しい Issue の `follow-up` の印も開いた後のコメントに書くので、同じ見出しの Issue に足すコメントと形が同じになる。足す先を有効な `follow-up` の印を持つ Issue に限れば、作業の単位の判定は今のまま変わらない。

- 🟢 の指摘を記録するとき、有効な `follow-up` の印を持ち、題名が指摘の見出しと同じ開いた Issue があれば、新しい Issue を作らずに、その Issue に PR の番号を足すコメントを書き、`follow-up` の印（`pr` に PR の番号、`finding` に指摘の ID）を付ける。無ければ、今のとおり新しい Issue を作り、開いた後のコメントに `follow-up` の印を書く
- 印の表の `follow-up` の行は、次のとおりにする（欄は変えない）

  | 種類（`type`） | 書くとき | 置き場 | 欄（`type` と `sig` のほか） |
  | --- | --- | --- | --- |
  | `follow-up` | 🟢 を Issue に記録した（新しい Issue にしたか、同じ見出しの開いた Issue に足した） | 🟢 を記録した Issue | `pr`、`finding`（指摘の ID） |

- 有効な `follow-up` の印を持たない Issue（人が起票した Issue、全体の停止の Issue、進行 Bot が開いた後に印を書く前に実行が切れた Issue）は、題名が同じでも足す先にしない。これらの Issue には `follow-up` の印を書かない
- PR を ready にする前の確かめ（T11）は、PR のどのラウンドの 🟢 にも、`pr` がその PR で `finding` がその指摘の ID の、有効な `follow-up` の印があるかで見る。印が新しい Issue のコメントにあるか、足したコメントにあるかは問わない

### 影響（Consequences）

- 良い点: PR を ready にする前の確かめが 1 種類の印で書け、印の種類ごとのテストの組が増えない（確かめやすさ）
- 良い点: 印の種類は 21 のままで、印の実装と一覧で扱う種類が増えない（変更のしやすさ）
- 中立: 印の一覧は `follow-up` の行の「書くとき」と「置き場」を直す。基本設計の図（印が付く順、進行中の次の工程）は、`follow-up` の名前と「🟢 を Issue に」だけを描き、置き場を描いていないので、直さない
- 悪い点: 題名が同じでも、人が起票した Issue には足さない。人の Issue と 🟢 の Issue が、同じ見出しで並ぶことがある（人の読む量が増える。人が読んで片方を閉じる）
- 悪い点: `follow-up` の印が、Issue を起こした指摘だけを指さなくなる。Issue を起こした指摘は、Issue の本文（PR の番号とラウンド。IMPL-26）で見分ける

### 確認方法（Confirmation）

- 単体テストで、次の 3 つを確かめる
  - 同じ見出しの、有効な `follow-up` の印を持つ開いた Issue があれば、新しい Issue を作らずに、コメントと `follow-up` の印を足す
  - 同じ見出しでも、有効な `follow-up` の印を持たない Issue（人の Issue、全体の停止の Issue、印を書く前に実行が切れた Issue）には足さず、新しい Issue を作る
  - PR を ready にする前の確かめが、足したコメントの `follow-up` の印でも通る
- 反映の後に、`duplicate` の名前が要件と基本設計に残っていないことを grep で確かめる（承認済みの ADR と、取り消し線の文を除く）

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：`follow-up` の印を書き、足す先を有効な `follow-up` の印を持つ Issue に限る

- 良い点: PR を ready にする前の確かめで読む印が 1 種類で済む（確かめやすさが上がる）
- 良い点: 新しい Issue にしたときと、足したときが、同じ印の同じ形のコメントになる。全体の停止の Issue の `halt` の印（開いていれば印のコメントを足す）と同じ書き方になる（読み違えにくさが上がる）
- 悪い点: `follow-up` の印は作業の単位の判定に使うので、足す先を有効な `follow-up` の印を持つ Issue に限る。人が起票した同じ見出しの Issue には足せず、重複を防げる範囲が狭まる（人の読む量が増える）

### 案 B：新しい種類 `duplicate` を足す

- 良い点: `duplicate` の印は作業の単位の判定に使わないので、人が起票した同じ見出しの Issue にも PR の番号を足せる（重複を防げる範囲が広く、人の読む量が減る）
- 良い点: Issue を起こした指摘（`follow-up`）と、足した指摘（`duplicate`）を、印の種類で見分けられる（読み違えにくさが上がる）
- 中立: 全体の停止の Issue と、印を書く前に実行が切れた Issue は、案 B でも足す先から外すことになる（足すと、人が読み違える）
- 悪い点: PR を ready にする前の確かめで 2 種類の印を読む（確かめやすさが下がり、テストの組が増える）
- 悪い点: 印が 22 種類になり、印の実装と一覧と図（「ほかの印」）で扱う種類が 1 つ増える（変更のしやすさが下がる）

## 補足情報（More Information）

- 2026-10-30 に CC が推奨案（`follow-up` の印を使う）を示し、2026-10-31 にメンテナが推奨案での起票を依頼した。起票の途中で、`follow-up` の印が作業の単位の判定に使われていることに気づき、足す先を有効な `follow-up` の印を持つ Issue に限る項目を足した
- 本 ADR で決めないこと
  - 同じ見出しの、有効な `follow-up` の印を持つ開いた Issue が 2 つ以上あるときに、どれに足すか（基本設計）
  - 見出しが同じかの比べ方（前後の空白の扱いなど。詳細設計）
  - 足すコメントの文（PR の番号のほかに何を書くか。基本設計）
- 関連する要件：IMPL-26（[02-implement-review-fix.md][impl]）、「印と固定の見出しの一覧」（[04-external-text-and-markers.md][text]）、UNDECIDED-92（[09-undecided.md][undecided]）
- 関連する ADR：[ADR-20261024-001][adr-model-states]
- 要件と設計への反映（承認の後に、別の PR で 1 本にまとめて行う）
  - 02-implement-review-fix.md：IMPL-26 の重複のコメントの印と、足す先
  - 04-external-text-and-markers.md：印の表の `follow-up` の行と、表の下の `note kind=duplicate` の文
  - 09-undecided.md：UNDECIDED-92 を欠番にする（元の文を取り消し線で残す）
  - 基本設計の「[状態遷移と判定][design-flow]」：レビュー合格の行の 🟢 の起票、「印が付く順」の `follow-up` の行、「ほかの印」の下の `note kind=duplicate` の文
  - [ADR-20261024-001][adr-model-states]：タイトルの下に、改めた旨の注記を足す

[adr-model-states]: ./adr-20261024-001-work-unit-main-flow-and-overlay-axes.md
[adr-readme]: ./README.md
[impl]: ../requirements/02-implement-review-fix.md
[text]: ../requirements/04-external-text-and-markers.md
[undecided]: ../requirements/09-undecided.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
