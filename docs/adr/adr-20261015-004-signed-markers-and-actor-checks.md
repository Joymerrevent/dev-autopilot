---
status: accepted
created: 2026-10-15
updated: 2026-10-20
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 進行役の投稿にはすべて署名付きの印を付け、人の操作は操作した人の権限と時刻を確かめてから従う

> **Amended by [ADR-20261020-001][adr-bots]（2026-10-20）**: 決定 1 の「進行役のアカウント」を、1 つのアカウントから、設定に並べたアカウントの一覧（`safety.botAccounts`）のどれかに改めた。印を読む条件の投稿者は一覧のどれかと読む。署名と未編集の条件、決定 2 は据え置く。
>
> **Amended by [ADR-20261016-002][adr-draft]（2026-10-16）**: 決定 2 の時刻の規則の対象から、Issue の本文を外した（Issue の本文は、著者を問わず AI に渡さなくなったため）。時刻の規則の対象は、作業指示コメントと直した下書きの 2 つになる。ほかの決定は据え置く。
>
> **［2026-10-17 追記］** 本文の PR と Issue の番号（`#<番号>`）は、business-os（<https://github.com/Joymerrevent/business-os>）のもの。ADR-20261015-004 は business-os の中で起票し、ADR-20261016-003 と ADR-20261017-001 の決定で Joymerrevent/dev-autopilot に移した。決定は変わらない。

要件の UNDECIDED-11 と UNDECIDED-13（[決めていないこと][undecided]の「受け入れと印」）を決める。
どちらも「GitHub の上の合図を、どの条件で信じるか」の論点で、問いは 2 つある：進行役の投稿の印をどこまで広げるか、人の操作のどこまでを権限で確かめるか。

## 背景と課題（Context and Problem Statement）

進行役は、状態の正を GitHub に置く（FLOW-4、IMPL-20）。GitHub の上の合図は 2 種類ある。

- 進行役の印：進行役が投稿するコメントの、署名付きの 1 行（「[印と固定の見出しの一覧][markers]」）。IMPL-20 の 3 条件（投稿者が進行役のアカウント、未編集、署名が検証できる）を満たす印だけを読む
- 人の操作：`agent-ready` を付ける、`needs-human` を外す、`plan:*` を付ける、`## 作業指示` や `## 却下 R<n>` を書く、下書きを直す、Status や Agent 欄を戻す

問い 1（UNDECIDED-11）：印の範囲

- 印は 6 種類（`intake` `intake-final` `plan` `review` `fix` `merge`）で、どれも「段を進めた結果」に付く。進行役は 6 種類のほかに、依存の待ち（FLOW-10）、触る範囲の重なりの警告（FLOW-8）、催促（MERGE-3）、止めた理由（`needs-human` を付けたとき）のコメントも書くが、印が無い
- FLOW-10・FLOW-8・MERGE-3 は、これらのコメントを「1 回だけ」書くとしている。印が無いと、すでに書いたかを GitHub から判定できない（手元の記録を使うかは UNDECIDED-8 で決めていない）
- 進行役は、当面メンテナと同じ GitHub アカウントで投稿する（SETUP-13）。投稿者では人と進行役を見分けられないので、見分けは印の有無で行う（IMPL-20）。FLOW-14 の「`## 却下 R<n>`」のように、人の見出しは「印を持たないコメント」で読む。
  印の無い進行役の投稿は、人の投稿と区別できない（進行役が外の人の本文を引用したコメントに `##` の見出しが含まれると、人の見出しとして読まれうる）
- 署名の対象は「印の `sig` 以外のすべての欄」で、印の種類・リポ・Issue や PR の番号は対象に無い。別の PR に投稿された印を写しても署名が通る
- 進行役の印のコメントが編集されると、IMPL-20 の (b) を満たさず「無いもの」として読まれる。`review` の `verdict=fix` や `dropped_high` のような、止める側の印が無いものになると、判定が通す側に倒れうる

問い 2（UNDECIDED-13）：人の操作の権限

- TEXT-10 が権限（push 以上）を確かめる相手は、Issue の著者、作業指示の投稿者、`agent-ready` を付けた人、渡す文を編集した人だけ。`needs-human` を外す人、`plan:*` を付ける人、Status と Agent 欄を戻す人は対象に無い。triage 権限でもラベルは付け外しできる
- `intake-final` の印は、`agent-ready` の後の次の実行の時点の渡す文のハッシュを取る（TEXT-12）。人が下書き X を読んで `agent-ready` を付けた後に、別の人が下書きを Y に直すと、人が読んでいない Y で確定する
- 操作した人を GitHub から取れるかを、GraphQL の型の定義で確かめた（2026-10-15。補足情報）。ラベルの付け外しは `LabeledEvent` `UnlabeledEvent` の `actor`、コメントの編集は `UserContentEdit` の `editor` で取れる。
  Project の欄の値（`ProjectV2ItemFieldSingleSelectValue`）は `creator`（「項目を作った人」）と `updatedAt` しか持たず、値を最後に変えた人を指すかは型の定義から分からない

課題：進行役の投稿のどこまでに印を付け、署名の対象に何を含め、印が無効になったらどうするか。人の操作のどこまでを、操作した人の権限と時刻で確かめるか。

## 判断の決め手（Decision Drivers）

- 状態の正を GitHub に保ち、手元の記録に頼らないこと（FLOW-4、IMPL-20）
- 人と進行役の投稿の見分けを、1 つの規則（印の有無）で済ませること（SETUP-13 の 1 つのアカウントでも成り立つ）
- 止める側の合図が消えたり偽られたりしても、通す側に倒れないこと（SAFE-4）
- 人の関所（下書きを読んで `agent-ready` を付ける）が、人が読んだ文で確定すること
- 確かめていない API の振る舞いを前提にしないこと

## 検討した選択肢（Considered Options）

問い 1：印の範囲

- 案 1-A：進行役の投稿すべてに署名付きの印を付け、署名の対象に種類・リポ・番号を含め、印が無効になっていたら止める
- 案 1-B：印は今の 6 種類のまま。「1 回だけ」は手元の記録で判定する
- 案 1-C：署名の対象だけ広げ、印の種類は増やさない

問い 2：人の操作の権限

- 案 2-A：ラベルの付け外しと渡す文の編集は、操作した人の権限を確かめ、`agent-ready` より後に直された文では確定しない。Status と Agent 欄は、操作した人に頼らない形で扱う
- 案 2-B：GitHub 側の権限の設定（triage の人を置かない、Project の書き込みを絞る）で守り、進行役は確かめない
- 案 2-C：今のまま（TEXT-10 の範囲）

## 決定（Decision Outcome）

採用した選択肢:「案 1-A」「案 2-A」。
理由: 判定の正を GitHub に保ったまま、1 つのアカウントでも人と進行役の投稿を見分けられる。止める側の合図が消えたり、権限の無い人が再開の合図を出したりしても、止まる側に倒れる。操作した人を取れると確かめた範囲（ラベルとコメントの編集）だけに頼る。

### 決定 1：進行役の投稿にはすべて署名付きの印を付ける

- 印の種類に `note` を足す。欄は `kind=wait|overlap|remind|stop` と、必要なら `reason=<短い記号>`。置き場は Issue か PR のコメント。人の編集は許さない。読むのは進行役
  - `wait`：依存の待ち（FLOW-10）、`overlap`：触る範囲の重なりの警告（FLOW-8）、`remind`：催促（MERGE-3）、`stop`：`needs-human` を付けた理由
  - 「1 回だけ書く」は、同じ Issue か PR に、同じ `kind` と `reason` の有効な `note` の印があるかで判定する
- 進行役が投稿するコメントは、人向けの本文の有無を問わず、必ず署名付きの印を 1 つ持つ。印の無い進行役のアカウントの投稿は、人の投稿として読む（FLOW-14 の読み方のまま）
- 署名の対象を、印の `sig` 以外のすべての欄に加えて、印の種類、リポ（`owner/repo`）、置き場の Issue か PR の番号とする。種類・リポ・番号は欄としては書かず、署名の計算にだけ入れる。別の Issue や PR に写した印は、署名が通らない
- 印が無効になっていたら止める：同じ Issue か PR の中で、進行役のアカウントが投稿した印のうち、IMPL-20 の (b)（未編集）か (c)（署名）を満たさないものが 1 つでもあれば、印の置き場の Issue か PR を `needs-human` にし、`note kind=stop reason=marker-invalid` を投稿する。
  受け入れの下書き（`intake`）は人が直す前提なので、(b) を掛けない（IMPL-20 のまま）
- 印のコメントの削除は、GitHub の API から検出できない。`review` の印の `round` は 1 から欠けずに続くことを判定の条件に足し、欠けていたら `needs-human` にする。`dropped_high` は PR ごとの累計を書く（途中の印が消えても、最新の印に累計が残る）
- メンテナが進行役のコメントの誤りを見つけても、編集せず、新しいコメントで補う（編集すると印が無効になり `needs-human` になる）。運用の文書に書く
- AI の出力を投稿する前に印の形の文字列を取り除く処理（IMPL-20）は、印を読む処理（`src/marker.ts`）と同じ規則（空白の揺れを含む）で行う。AI のセッションに前回の印を渡すときは、進行役が読み取った値（ラウンド・件数・判定）だけを渡し、印の 1 行を手を加えずに渡すことはしない

### 決定 2：人の操作は、操作した人の権限と時刻を確かめてから従う

- 権限（push 以上。TEXT-10 の確かめ方）を確かめる相手に、次を足す
  - `needs-human` を外した人（`UnlabeledEvent` の `actor`）。push 未満なら、`needs-human` を付け直し、`note kind=stop reason=unauthorized-resume` を投稿する
  - `plan:fable` / `plan:opus` を付けた人（`LabeledEvent` の `actor`）。push 未満なら、ラベルを無いものとして扱い、受け入れの印の `plan=` を使う
- 時刻の規則：渡す文（本文、作業指示コメント、直した下書き）の投稿か最後の編集が、`agent-ready` を付けた時刻（`LabeledEvent` の `createdAt`）より後なら、`intake-final` を投稿せず `needs-human` にする。
  人は、文を直してから `agent-ready` を付け直す（付け直した時刻より前の文なら確定する）。`intake-final` の後の変化は、今のまま TEXT-4 のハッシュの照合で止める
- Status と Agent 欄は、操作した人を確かめない（値を最後に変えた人を API で取れるか確かめていない。UNDECIDED-29）。代わりに次の形で扱う
  - Status：進行役は、印と PR の状態から決まる Status と、Project の Status を比べる。違っていれば、人が戻す手順（「[止まった状態から戻す手順][flow]」：`needs-human` を外して `Todo` か `In Review` に戻す）に当たるときだけ従い、`needs-human` を外した人の権限で確かめる。ほかの違いは、想定外の Status として `needs-human`（SAFE-4）
  - Agent 欄：止める向き（`human`・`paused` にする）は、操作した人を問わず従う。再開の向き（`paused` から戻す、`human` から `worker` に戻す）は今のまま従う。Project に書ける人がリポの push 以上の人に限られることを、運用の前提にする（UNDECIDED-29 で確かめる）

  > **［2026-10-16 追記］** UNDECIDED-29 を確かめた。Project の欄の値の `creator` は値を最初に入れた人で、値を変えても変わらない。Issue のタイムラインの Status の変更のイベントは変えた人を持つが、Agent 欄の変更では出ず、Status でも抜けがある（[FACT-27 と FACT-28][facts]）。
  > 値を最後に変えた人は API で取れないため、Status と Agent 欄を操作した人を確かめない形と、Project に書ける人をリポの push 以上の人に限る運用の前提は変わらない。

### 影響（Consequences）

- 良い点: 1 回だけ書くコメントの重複を、GitHub だけで止められる
- 良い点: 1 つのアカウントでも、人と進行役の投稿を印の有無で見分けられる。進行役が引用した見出しを、人の見出しとして読まない
- 良い点: 写した印と、編集や削除で消えた止める側の印が、通す側に倒れない
- 良い点: push 未満の人は、止まった Issue を再開できず、計画役のモデルも変えられない。人の関所は、人が読んだ文で確定する
- 悪い点: 印の種類が 7 つになり、`src/marker.ts` と「印と固定の見出しの一覧」が増える
- 悪い点: メンテナが進行役のコメントを編集すると `needs-human` になる。直すときは新しいコメントで補う
- 悪い点: 人が下書きを直した後に、`agent-ready` を付け直す手間がある（直してから付ける順を運用の文書に書く）
- 悪い点: Agent 欄の再開の向きは、操作した人を確かめられないまま従う

### 確認方法（Confirmation）

- `src/marker.ts` の単体テストで、種類・リポ・番号を変えた印の署名が通らないこと、空白を変えた印の形の文字列が AI の出力から取り除かれることを検査する
- 結合テスト（偽の `gh`）で、次を検査する
  - 依存の待ちのコメントが、2 回の実行で 1 件しか付かないこと
  - 編集された `review` の印、`round` の欠けがあると `needs-human` になること
  - push 未満の人が外した `needs-human` が付け直され、push 未満の人が付けた `plan:*` が無視されること
  - `agent-ready` の後に直された下書きでは、`intake-final` を投稿せずに `needs-human` になること
  - 印から決まる Status と Project の Status が違うとき、人が戻す手順に当たらなければ `needs-human` になること

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 1-A：すべての投稿に印を付け、署名の対象を広げ、無効化で止める

- 良い点: 判定の正が GitHub に残り、1 つのアカウントでも人と進行役の投稿を見分けられる
- 良い点: 写した印と消えた印で通す側に倒れない
- 悪い点: 印の種類が増える。進行役のコメントを人が編集できなくなる

### 案 1-B：印は 6 種類のまま、「1 回だけ」は手元の記録で判定する

- 良い点: 印の仕組みを変えずに済む
- 悪い点: 手元の記録を失うと、同じコメントを何度も書く（手元の記録の扱いは UNDECIDED-8 で決めていない）
- 悪い点: 印の無い進行役の投稿が人の投稿と区別できない問題が残る

### 案 1-C：署名の対象だけ広げる

- 良い点: 写した印は塞がる。変更が小さい
- 悪い点: 「1 回だけ」の判定と、印の無い投稿と人の投稿の区別の問題が残る

### 案 2-A：ラベルと編集は操作した人を確かめ、Status と Agent 欄は操作した人に頼らない

- 良い点: 確かめられる範囲（ラベル、コメントの編集）で、権限の無い再開と、読んでいない文での確定を止める
- 良い点: 確かめていない API の振る舞いに頼らない
- 悪い点: Agent 欄の再開の向きは、運用の前提に頼る

### 案 2-B：GitHub 側の権限の設定で守る

- 良い点: 進行役の仕組みが増えない
- 悪い点: 設定は分離した後の別のリポごとに違いうる。設定が緩んでも気づけない（fail-open）
- 悪い点: 読んでいない文での確定は、権限の設定では防げない

### 案 2-C：今のまま

- 悪い点: triage 権限の人が `needs-human` を外すと再開する。人が読んでいない文で確定しうる
- 却下

## 補足情報（More Information）

- 関連する要件：IMPL-10、IMPL-18、IMPL-20、IMPL-22（[02-implement-review-fix.md][impl]）、TEXT-4、TEXT-10、TEXT-12、「印と固定の見出しの一覧」（[04-external-text-and-markers.md][markers]）、
  FLOW-4、FLOW-8、FLOW-10、FLOW-14、「止まった状態から戻す手順」（[01-issue-flow.md][flow]）、MERGE-3（[03-merge.md][merge]）、SAFE-4（[05-safety.md][safety]）、SETUP-13（[06-setup-and-stages.md][setup]）、UNDECIDED-8
- 2026-10-15 の確認（GitHub の GraphQL の型の定義を読んだ。データは読んでいない）：`LabeledEvent` は `actor` と `createdAt` を持つ。`UserContentEdit` は `editor` と `editedAt` を持つ。
  `ProjectV2ItemFieldSingleSelectValue` は `creator`（説明は「The actor who created the item.」）、`createdAt`、`updatedAt` を持ち、値を最後に変えた人の欄は無い
- 前の版：2026-10-11 に、同じ論点を ADR-20261011-007 と ADR-20261011-010 として起票した（閉じた PR #95）。決定の向きは同じ。本 ADR は今の要件の番号で書き直した。
  検疫で見つかったときの印は ADR-20261015-003 で決めたので外した。1 つのアカウントで人と進行役を見分ける論点（SETUP-13）を足した。Status を戻した人を API で取れるかを確かめていなかったので、頼らない形に改めた
- 範囲外：Project の欄の値の `creator` が、値を最後に変えた人を指すか。捨てる Project で値を書き換えて確かめる（FACT-15 と同じやり方）。要件に反映するときに、[決めていないこと][undecided]の「確かめること」に UNDECIDED-29 として足す
- 要件への反映（承認の後に、別の PR で行う）
  - 「印と固定の見出しの一覧」：`note` の行を足す。署名の対象に種類・リポ・番号を含めることを書き足す
  - IMPL-20：署名の対象、印が無効になっていたら止めること、印を取り除く規則と AI に渡す値を書き足す
  - IMPL-18：`review` の `round` が欠けずに続くことと、`dropped_high` が累計であることを書き足す
  - FLOW-8、FLOW-10、MERGE-3：「1 回だけ」を `note` の印で判定することを書き足す
  - TEXT-10：権限を確かめる相手に `needs-human` を外した人と `plan:*` を付けた人を足し、時刻の規則を書き足す
  - 「止まった状態から戻す手順」：Status と Agent 欄の扱い（決定 2）と、進行役のコメントを編集しない運用を書き足す
  - 09-undecided.md：UNDECIDED-11 と UNDECIDED-13 を欠番にする（元の文を取り消し線で残す）。UNDECIDED-29（Project の欄の値の `creator`）を「確かめること」に足す

[undecided]: ../requirements/09-undecided.md
[markers]: ../requirements/04-external-text-and-markers.md
[flow]: ../requirements/01-issue-flow.md
[impl]: ../requirements/02-implement-review-fix.md
[merge]: ../requirements/03-merge.md
[safety]: ../requirements/05-safety.md
[setup]: ../requirements/06-setup-and-stages.md
[facts]: ../requirements/10-verified-facts.md
[adr-draft]: ./adr-20261016-002-intake-ai-drafts-every-issue.md
[adr-bots]: ./adr-20261020-001-bot-accounts-as-list.md
