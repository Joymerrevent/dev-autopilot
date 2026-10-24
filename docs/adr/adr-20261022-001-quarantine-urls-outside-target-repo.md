---
status: accepted
created: 2026-10-22
updated: 2026-10-24
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 検疫は、当てる先のリポの GitHub の URL のほかの URL を探して止め、AI には GitHub に投稿する文にほかの URL を書かせない

> **Amended by [ADR-20261024-005][adr-20261024-005]（2026-10-24）**: 決定 1・3 の「止めて `needs-human`」を、安全の軸に値を入れる形に改めた。AI に渡す前の文で見つかれば攻撃の疑い（入口）、AI が書いた文で見つかれば攻撃の疑い（AI の文）で、全体も止める（決定 1・3）。検疫が探す URL の範囲と、AI に外の URL を書かせない指示は据え置く。
>
> **［2026-10-24 追記］** 段階 4〜8 に触れる文は、[ADR-20261024-002][adr-20261024-002] の決定 8 で読む（段階 4〜8 と段階の出口の数え上げを持たない）。

要件の UNDECIDED-68（検疫が探す URL の範囲を、どこまでにするか）を決める。

## 背景と課題（Context and Problem Statement）

- TEXT-9 の「指示の混入の検査」は、題名・本文・コメント・添付のリンク先の文言に、AI への指示の形や隠した文と並べて「外部へ取りに行かせる URL」が無いかを探すとする
- TEXT-11 は、AI に渡す文（受付 AI に渡す Issue の文、作業指示コメント、承認された下書き、計画コメント、再レビューの入力の前回の指摘）を、渡す直前に毎回、TEXT-9 と同じ TypeScript のモジュールで検査し、「URL」を探して、1 つでも見つかれば渡さずに止めて `needs-human` にするとする（fail-closed）
- モジュールは 1 つなので、探す範囲も 1 つでなければならない。今は要件の中で 2 通りに書かれている
  - 「URL」をすべて止めると、計画コメント（IMPL-10）や前回の指摘（TEXT-1）のように、AI が書いて次の AI に渡す文に、当てる先のリポの Issue・PR・ファイルの URL があるだけで、Issue が `needs-human` で止まる
  - 「外部へ取りに行かせる URL」は、取りに行かせる意図を機械で見分けられないので、機械の検査の範囲として決まらない
- AI は URL を取りに行けない。作業 AI とレビュー AI には、文中の URL を取りに行く手段（WebFetch・WebSearch・外への curl）を与えず（TEXT-5）、AI のセッションの外への通信は、許可した先（npm のレジストリなど）だけに限る（SAFE-10）
- AI が取りに行けなくても、外の URL は危うい。AI が外の URL を、依存のパッケージの取得先やスクリプトに書き写すと、CI か人の手元が、外の URL から取ってきたものを動かしうる。外の人が Issue に書いた URL が、作業指示を経て AI に届く道を、検疫で止めたい
- 基本設計の 2 回目の再レビュー（2026-10-21）の直しで、GitHub に投稿する AI の文も、投稿の前に検疫に通すことにした（基本設計の「[AI のセッション][design-ai]」の「役ごとの出力契約」の「共通の決まり」）。AI が書いた URL でも、投稿の前と、次の AI に渡す前に止まる
- UNDECIDED-68 は、受け入れ工程（段階 4）で GitHub に書き始める前に決めるとしている

課題：検疫が探して止める URL の範囲を、TEXT-9 と TEXT-11 で 1 つに決める。

## 判断の決め手（Decision Drivers）

- 外の人の文にある外の URL を、AI に渡す前に止めること（fail-closed を保つ）
- 機械で決まる範囲であること（意図を推し量る形にしない）
- 当てる先のリポの中を指す URL だけで、Issue が止まり続けないこと
- AI 自身が書いた文で、次の AI に渡すときに止まる形を減らすこと
- 範囲を後から広げたり狭めたりするときに、どこを直せばよいかが分かること

## 検討した選択肢（Considered Options）

- 案 A：URL をすべて探して止める（TEXT-11 のまま。TEXT-9 の書き方を直す。UNDECIDED-68 の候補 (a)）
- 案 B：当てる先のリポの GitHub の URL のほかの URL を探して止める（候補 (b)）
- 案 C：URL は探さず、指示の形の語の検査に任せる（候補 (c)）

## 決定（Decision Outcome）

採用した選択肢:「案 B」。
理由: 外の URL は今までどおり止め、当てる先のリポの中を指す URL だけを通すので、fail-closed を保ったまま、計画や前回の指摘のような AI が書いた文で止まり続ける形を減らせる。範囲は「URL の始まりが、当てる先のリポの GitHub の URL か」で機械で決まり、意図を推し量らない。

### 決定 1：検疫は、当てる先のリポの GitHub の URL のほかの URL を探して止める

- 検疫のモジュール（TEXT-9、TEXT-11）は、文の中の URL を探し、当てる先のリポの GitHub の URL のほかが 1 つでもあれば、今のとおり渡さずに止めて `needs-human` にする（fail-closed）
- 当てる先のリポの GitHub の URL は、`https://github.com/<owner>/<name>` で始まり、名前の後が文字列の終わりか `/`・`#`・`?` のものとする。`<owner>` と `<name>` は、マシンの設定の `repository.owner` と `repository.name`（[ADR-20261021-006][adr-repo]）で、大文字と小文字は区別しない。
  `https://github.com/<owner>/<name>.evil.example/` のように、名前の後に別の文字が続くものは、ほかの URL として止める
- 何を URL として探すか（スキームの形、`www.` で始まる文字列、Markdown のリンクと自動リンク、% エンコードや見えない文字で崩した形）は、基本設計で決め、検出の見本（`fixtures/quarantine/`）に持つ。崩した形は、見つけたら止める側に読む
- 当てる先のリポの GitHub の URL も、ほかの検査（見えない文字、指示の形の語、欄の区切りを模した文字列）からは外さない

### 決定 2：TEXT-9 と TEXT-11 の書き方を、決定 1 の範囲にそろえる

- TEXT-9 の「外部へ取りに行かせる URL」と、TEXT-11 の「URL」を、どちらも「当てる先のリポの GitHub の URL のほかの URL」にする
- 範囲を変えるとき（たとえば、公式文書の URL を通すとき）は、安全の境界を変える変更として、新しい ADR で決める。リポの設定に、通す URL の一覧は持たない

### 決定 3：AI には、GitHub に投稿する文に、当てる先のリポの GitHub の URL のほかの URL を書かせない

- 受付 AI・計画 AI・作業 AI・レビュー AI・批評 AI の固定の指示文に、GitHub に投稿する文（下書き、計画、PR の題名と本文、対応、指摘）に、当てる先のリポの GitHub の URL のほかの URL を書かないことを入れる。
  公式文書などを示したいときは、URL でなく、文書の名前と節の名前で書く
- 書いてしまったときは、投稿の前の検疫で止まり、`needs-human` になる（止める側に倒れる）。指示文は止まる回数を減らすためのもので、守りは検疫に置く

### 影響（Consequences）

- 良い点: 外の人の文にある外の URL は、今までどおり AI に渡る前に止まる
- 良い点: 当てる先のリポの Issue・PR・ファイルを指す URL だけで、Issue が止まり続けない
- 良い点: 範囲が機械で決まり、TEXT-9 と TEXT-11 の書き方が 1 つになる
- 悪い点: 外の人が当てる先のリポの Issue に、正当な外の URL（再現の手順のリンクなど）を書くと、受け入れで止まる。人が作業指示を一から書く（TEXT-9 の、見つかったときの扱い）
- 悪い点: 公式文書の URL を通したくなったら、新しい ADR が要る。受け入れ工程（段階 4）と計画・実装工程（段階 5）で、URL で止まった回数を数え、多ければ起票する
- 中立: 当てる先のリポの GitHub の URL の先にある文（Issue のコメントなど）は、AI が取りに行けない（TEXT-5、SAFE-10）ので、通しても AI の文脈には入らない

### 確認方法（Confirmation）

- 検疫のモジュールの単体テストで、次を検査する（`fixtures/quarantine/` の前提データ）
  - 当てる先のリポの GitHub の URL（Issue・PR・ファイル・`#` と `?` の付いたもの、owner と名前の大文字と小文字が違うもの）は、URL としては止めないこと
  - ほかのリポの GitHub の URL、外のドメインの URL、`https://github.com/<owner>/<name>.evil.example/` のように名前の後に別の文字が続く URL は、止めること
  - 当てる先のリポの GitHub の URL の中に見えない文字や指示の形の語があれば、ほかの検査で止めること
- 結合テストで、計画コメントに当てる先のリポの PR の URL がある Issue が止まらずに進み、外の URL がある計画は投稿の前に止まることを確かめる
- 受け入れ工程（段階 4）の出口で、URL で止まった回数を数える（段階の表の記録）

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：URL をすべて探して止める

- 良い点: 範囲がいちばん簡単で、通す URL の判定の誤りが起きない
- 悪い点: 当てる先のリポを指す URL だけで、計画や前回の指摘を渡すときに止まり続ける。AI に URL を書かせない指示だけでは、人が書いた作業指示の URL も止まる
- 悪い点: TEXT-9 の「外部へ取りに行かせる」の言い方を消すことになり、要件が本当に止めたかったもの（外の URL）より広く止める

### 案 B：当てる先のリポの GitHub の URL のほかを探して止める

- 良い点: 外の URL は止め、当てる先のリポの中を指す URL は通す
- 良い点: 範囲が機械で決まる（マシンの設定の owner と名前で照らす）
- 悪い点: 通す URL の判定（名前の後の文字の扱いなど）を誤ると、外の URL が通る。単体テストで境目を押さえる
- 悪い点: 公式文書などの正当な外の URL も止まる

### 案 C：URL を探さない

- 良い点: URL で止まることが無くなる
- 悪い点: 外の人の文にある外の URL が、作業指示を経て AI に届き、依存の取得先やスクリプトに書き写されうる。守りが PR のレビューとマージの判定だけになる
- 悪い点: TEXT-9 と TEXT-11 が URL を探すとしている理由（外の文の取り込みを止める）を外すことになる

## 補足情報（More Information）

- 基本設計の 2 回目の再レビュー（2026-10-21）の直しの中で、テストの設計を直す役が見つけた食い違い（PR #96 の「範囲の外で見つけた残り」）から起票した
- 関連する要件：TEXT-1、TEXT-5、TEXT-9、TEXT-11（[04-external-text-and-markers.md][text]）、SAFE-10（[05-safety.md][safety]）、IMPL-10（[02-implement-review-fix.md][impl]）、UNDECIDED-68（[09-undecided.md][undecided]）
- 関連する ADR：[ADR-20261015-003][adr-intake]（受け入れの範囲と検疫の結果の書き方）、[ADR-20261021-006][adr-repo]（当てる先のリポの owner と名前をマシンの設定に置く）
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 04-external-text-and-markers.md：TEXT-9 の「外部へ取りに行かせる URL」と TEXT-11 の「URL」を、当てる先のリポの GitHub の URL のほかの URL にそろえる。範囲を変えるのは ADR で決めることを書く
  - 09-undecided.md：UNDECIDED-68 を欠番にする（元の文を取り消し線で残す）
  - 基本設計の「[AI のセッション][design-ai]」：検疫が探す URL の形と、当てる先のリポの GitHub の URL の照らし方を書く。5 役の固定の指示文に、ほかの URL を書かないことを足す
  - 基本設計の「テスト」：検出の見本に、通す URL と止める URL の境目を足す

[adr-intake]: ./adr-20261015-003-intake-scope-and-quarantine-report.md
[adr-repo]: ./adr-20261021-006-repo-owner-and-name-in-machine-settings.md
[text]: ../requirements/04-external-text-and-markers.md
[safety]: ../requirements/05-safety.md
[impl]: ../requirements/02-implement-review-fix.md
[undecided]: ../requirements/09-undecided.md
[design-ai]: ../design/basic/03-ai-sessions.md
[adr-20261024-002]: ./adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[adr-20261024-005]: ./adr-20261024-005-security-axis-and-halt-issue.md
