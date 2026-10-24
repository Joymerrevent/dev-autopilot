---
status: accepted
created: 2026-10-21
updated: 2026-10-24
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# レビュー AI に Skill の道具を持たせず、進行 Bot が review Skill の本文に引数を入れて渡す

> **［2026-10-24 追記］** 「レビュー工程（段階 6）に入る前の手動の確かめ」は、AI のレビューを初めて動かす前の手動の確かめと読む（[ADR-20261024-002][adr-20261024-002] の決定 8）。

要件の UNDECIDED-59（レビュー AI に `Skill` の道具を持たせるか）を決める。

改める決定と、据え置く決定は次のとおり。

- [ADR-20261017-002][adr-review]：決定 1 の「どの軸を見るかを、進行ボットが固定の指示文と `review` Skill の引数で渡す」のうち、引数の渡し方を改める。レビュー AI が `Skill` の道具で `review` を呼ぶのでなく、進行 Bot が `review` Skill の本文に引数を入れて、指示文として渡す。
  決定 1 の「軸ごとの手順は Skill の中に 1 回だけ書き、無人の入り口と人の入り口で同じものを使う」と、決定 1 のほかの項目、決定 2〜5 は据え置く。決定 2 の重ねる守り（`permissions.deny` の `Agent`）に、`Skill` を並べる
- [ADR-20261017-004][adr-human]：決定 1（AI のセッションには、写しの `skills/` だけを読ませる）は据え置く。レビュー AI は、今のとおり写しの `skills/review/` の参照ファイルを Read で読み、スクリプトを sandbox の中の Bash で動かす

## 背景と課題（Context and Problem Statement）

- IMPL-16 は、PR のレビューの軸のセッションで `review` Skill を手順として使い、見る軸を Skill の引数で渡すとする（REVIEW-5 の無人の入り口）。[ADR-20261017-002][adr-review] の決定 1 も同じ形で決めた
- レビュー AI（`change/reviewer`）の道具は Bash と Read（「登場するもの」と、基本設計の「AI のセッション」）。エージェント定義の `tools` に `Skill` が無いと、Skill を呼べない（FACT-55）
- `tools` に `Skill` を足すと、Plugin のほかの Skill（`setup`・`check`・`intake`・`dev-autopilot`）と、Claude Code に同梱の Skill（`update-config`・`schedule` など）も呼べるようになる（FACT-55）
- 2026-10-21 に、Skill を絞る書き方を実機で確かめた（FACT-58）
  - 安全設定の `permissions.deny` に `Skill(<名前>)` と書くと、名前ごとに拒めた。Claude Code に同梱の Skill も同じ書き方で拒めた
  - `deny` に `Skill(*)`、`allow` に `Skill(<名前>)` と書くと、道具の一覧から `Skill` が消え、どの Skill も呼べなかった。1 つだけ許す書き方はできない
  - エージェント定義の `skills:` に Skill の名前を書いても、`Skill` の道具が無いと、Skill は読み込まれなかった
  - 初期化の記録に並んだ Claude Code に同梱の Skill は 18 個で、Claude Code の版で変わりうる
- レビュー AI は、写しの `skills/` を Read で読め、sandbox の中の Bash でスクリプトを動かせる（SAFE-2 の (e)、SAFE-10、[ADR-20261017-004][adr-human] の決定 1）
- `review` Skill は、Plugin の中からも単体からも動くように書き、dev-autopilot 向けの部分（無人の入力、出力契約、エージェントの名前）は引数で渡されたときだけ使う（STRUCT-7）。同封したファイルは `${CLAUDE_SKILL_DIR}` で指す（[ADR-20261017-004][adr-human] の決定 1）

課題：レビュー AI に `review` Skill の手順をどう渡すか。`Skill` の道具を持たせるなら、`review` のほかの Skill をどう止めるか。

## 判断の決め手（Decision Drivers）

- AI の道具を、役に要る分より広げないこと（SAFE-2）
- Claude Code の版が上がって同梱の Skill が増えても、AI が呼べるものが黙って増えないこと。守りが足りないときは、呼べない側に倒れること（フェイルセーフ）
- 人が手で使う `review` と、進行 Bot が使う `review` の手順が 1 つであること（[ADR-20261017-002][adr-review] の決定 1、STRUCT-7）
- Skill を呼ぶかどうかを AI に任せず、渡す手順を進行 Bot が決めること
- 実機で確かめた事実（FACT-55、FACT-58）で決められること

## 検討した選択肢（Considered Options）

- 案 A：`change/reviewer` の `tools` に `Skill` を足し、`permissions.deny` で `review` のほかの Skill を名前ごとに拒む
- 案 B：エージェント定義の `skills:` で `review` を読み込ませ、`Skill` の道具は足さない
- 案 C：`Skill` の道具を足さず、進行 Bot が `review` Skill の本文に引数を入れて、指示文として渡す
- 案 D：`Skill` の道具を足さず、指示文で「写しの `skills/review/SKILL.md` を Read で読んで従う」と頼み、引数は指示文に書く

## 決定（Decision Outcome）

採用した選択肢:「案 C」。
理由: レビュー AI の道具は Bash と Read のままで、Claude Code の版で同梱の Skill が増えても呼べない。手順の正本は `review` Skill の本文 1 つのままで、人の入り口と無人の入り口で同じものを使う。渡す文は進行 Bot が作るので、Skill の呼び忘れや別の Skill の呼び出しが起きない。
案 A は、1 つだけ許す書き方が無く（FACT-58）、拒む名前を Claude Code の版ごとに保つことになる。案 B は効かない（FACT-58）。

### 決定 1：AI の 5 役に Skill の道具を持たせない

- `change/reviewer` の `tools` は Bash と Read のままにする。AI の 5 役のどのエージェント定義の `tools` にも、`Skill` を書かない
- 重ねる守りとして、進行 Bot が渡す安全設定の `permissions.deny` に `Skill` を入れる（[ADR-20261017-002][adr-review] の決定 2 の `Agent` と同じ書き方）。エージェント定義に誤って `Skill` が足されても、道具の一覧から消える
  - FACT-58 で道具の一覧から消えたのは `Skill(*)` の書き方。道具の名前だけの書き方（`Skill`）でも消えるかを実装の前に確かめ、消えなければ `Skill(*)` と書く
- AI の 5 役に `Skill` の道具を与えるのは、新しい ADR で決める

> **［2026-10-21 追記］** 道具の名前だけの `Skill` でも、エージェント定義の `tools` に `Skill` を書いたエージェントから、道具の一覧が消えることを、反映の前に実機で確かめた（FACT-59）。安全設定には `Skill` と書く。決定は変わらない。

### 決定 2：進行 Bot が review Skill の本文に引数を入れて渡す

- 進行 Bot は、軸のセッション（3 つの軸と見落としの軸）を起こすたびに、写しの `skills/review/SKILL.md` を読み、frontmatter を除いた本文に次の 2 つの置き換えをする
  - 引数：REVIEW-5 の無人の入り口の引数（軸・基点・仕様の在りか・🟢 の起票先・予算）。形は、人が `/review <引数>` で渡すときと同じにする
  - `${CLAUDE_SKILL_DIR}`：写しの `skills/review/` の絶対パス
- 置き換えた本文を、固定の指示文に続けて、セッションの最初の指示として渡す。固定の指示文と本文の並べ方は、基本設計で決める
- `review` Skill の本文は、引数を `$ARGUMENTS` の 1 か所で受ける形に書き、Claude Code が Skill を呼ぶときに本文を書き換える機能は、上の 2 つの置き換えのほかに使わない。
  進行 Bot は、置き換えた後の本文に置き換えの書き方（`$ARGUMENTS` や `${CLAUDE_` で始まる変数）が残っていたら、セッションを起こさずに実行の失敗（IMPL-14）にする
- レビュー AI は、本文が `${CLAUDE_SKILL_DIR}` で指す参照ファイルを Read で読み、スクリプト（`scripts/per-commit-gates.ts`）を sandbox の中の Bash で動かす。読める範囲は今のまま（写しの `skills/`）で、広げない
- 人の入り口は変えない。人は `review` を Skill として呼び（`/review <引数>`）、Claude Code が同じ本文に同じ置き換えをする

> **［2026-10-22 追記］** REVIEW-5 の無人の入り口の引数に、品質ゲートを回すかを足した。進行 Bot は、CI の結果をレビュー AI に渡すので「回さない」を入れて渡し、`review` Skill の本文は、品質ゲートの全体とコミットごとのゲートの検査（REVIEW-7）を飛ばして壊れ方の表から始める。本文の手順（品質ゲートから始める）と固定の指示文（品質ゲートの全体を回さない）が、1 つのセッションの中で食い違わないようにするため（基本設計の「[AI のセッション][design-ai]」の「レビューの軸の起こし方」）。進行 Bot が本文に引数を入れて渡すという決定は変わらない。

### 影響（Consequences）

- 良い点: レビュー AI の道具は Bash と Read のまま。Claude Code の版で同梱の Skill が増えても、AI は呼べない
- 良い点: 拒む Skill の名前の一覧を、Claude Code の版ごとに保たなくてよい
- 良い点: Skill を呼ぶかを AI に任せないので、手順の呼び忘れや、別の Skill の呼び出しが起きない
- 悪い点: Claude Code が Skill を呼ぶときの置き換えを、進行 Bot が真似る。Claude Code の Skill の仕組みが変わると、人の入り口と無人の入り口で、本文の読まれ方がずれうる。置き換えを 2 つに絞り、置き換えの書き方が残っていたら起動しないことで、ずれを見える形にする
- 悪い点: `review` Skill の本文の書き方に、置き換えを 2 つに絞る制約が付く
- 中立: REVIEW-5 の引数の形は変わらない。変わるのは、引数を本文に入れる者（Claude Code の `Skill` の道具から、進行 Bot へ）
- 中立: Skill の道具で呼んでも本文は文脈に入るので、本文を最初の指示に入れても、文脈に入る量はほぼ変わらない

### 確認方法（Confirmation）

- 実装の前に、安全設定の `permissions.deny` に道具の名前だけの `Skill` を書いたセッションで、初期化の記録の道具の一覧に `Skill` が無いことを実機で確かめて、FACT に残す
- 進行 Bot の単体テストで、2 つの置き換えと、置き換えの書き方が残ったときにセッションを起こさないことを検査する
- 毎回の点検の試しのセッション（基本設計の「導入と点検」）で、道具の一覧に `Skill` が無いことを確かめる（`Agent`・`Task` と同じく、名前の全体が一致するかで探す）
- レビュー工程（段階 6）に入る前の手動の確かめ（「レビューの Skill」の、直したスキルを 1 件の PR で確かめる手順）で、人の `/review <引数>` と、進行 Bot が渡す文で、同じ PR に同じ形の出力が出ることを比べる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：Skill の道具を足し、ほかの Skill を名前ごとに拒む

- 良い点: 人と同じく `Skill` の道具で呼ぶので、置き換えを真似なくてよい
- 悪い点: 1 つだけ許す書き方が無い（FACT-58）。拒む Skill の名前（Plugin の 4 つと、同梱の 18 個）を全部並べる
- 悪い点: 同梱の Skill は Claude Code の版で変わり、新しい Skill は拒む一覧に無いまま呼べる（開いた側に倒れる）。初期化の記録に知らない Skill があれば止める仕組みを足せば閉じられるが、一覧を版ごとに保つ
- 悪い点: AI の道具が 1 つ広がる

### 案 B：エージェント定義の skills: で読み込ませる

- 良い点: 道具を広げない
- 悪い点: `Skill` の道具が無いと読み込まれない（FACT-58）。採れない

### 案 C：進行 Bot が本文に引数を入れて渡す

- 良い点: 道具を広げず、同梱の Skill が増えても呼べない
- 良い点: 渡す手順を進行 Bot が決める
- 悪い点: 置き換えを真似る。本文の書き方に制約が付く

### 案 D：写しの SKILL.md を Read で読ませる

- 良い点: 道具を広げず、進行 Bot が置き換えを真似なくてよい
- 悪い点: 読むか、どこまで読むかを AI に任せる。読み飛ばしても進行 Bot は気づけない
- 悪い点: `${CLAUDE_SKILL_DIR}` と引数を、AI が自分で読み替える

## 補足情報（More Information）

- 関連する要件：IMPL-16（[02-implement-review-fix.md][impl]）、REVIEW-1、REVIEW-5（[08-review-skill.md][review]）、STRUCT-7（[07-structure.md][structure]）、SAFE-2、SAFE-10（[05-safety.md][safety]）、UNDECIDED-59（[09-undecided.md][undecided]）、FACT-55、FACT-58（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261017-002][adr-review]、[ADR-20261017-004][adr-human]
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 02-implement-review-fix.md：IMPL-16 の「固定の指示文と `review` Skill の引数で渡す」を、進行 Bot が `review` Skill の本文に引数を入れて、固定の指示文に続けて渡す形に直す
  - 08-review-skill.md：REVIEW-5 に、無人の入り口では進行 Bot が本文に引数を入れて渡すことと、本文が使ってよい置き換え（`$ARGUMENTS` と `${CLAUDE_SKILL_DIR}`）を足す
  - 05-safety.md：SAFE-2 の (e) の `Skill` の道具の記述を、どの役にも持たせず、`permissions.deny` に入れることに直す
  - 09-undecided.md：UNDECIDED-59 を欠番にする
  - 基本設計の「AI のセッション」：セッションごとの安全設定の `permissions.deny` に `Skill` を足し、`permissions.allow` の行の「`Skill` の道具は、入れなくても使える」を直す。レビュー AI の起動の形に、本文の置き換えと渡し方を足す
  - 基本設計の「導入と点検」：試しのセッションの道具の一覧の確かめに、`Skill` が無いことを足す
  - [ADR-20261017-002][adr-review]：タイトルの下に、本 ADR で決定 1 の一部を改め、決定 2 の重ねる守りに `Skill` を並べた旨の注記を足す

[adr-review]: ./adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-human]: ./adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[impl]: ../requirements/02-implement-review-fix.md
[review]: ../requirements/08-review-skill.md
[structure]: ../requirements/07-structure.md
[safety]: ../requirements/05-safety.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-ai]: ../design/basic/03-ai-sessions.md
[adr-20261024-002]: ./adr-20261024-002-no-stages-4-to-8-and-no-trials.md
