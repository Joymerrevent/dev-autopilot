---
status: superseded by ADR-20261024-001
created: 2026-10-22
updated: 2026-10-24
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 人が戻した時刻より前に閉じた PR は止めずに結びつきから外し、止まった状態から戻す手順はどれも needs-human を付けたまま行って最後に外す

要件の UNDECIDED-74（人が戻す前に閉じた PR を結びつきから外す例外と、止まった状態から戻す手順で `needs-human` を外す位置。[決めていないこと][undecided]の「ADR で決めること」の「流れと工程」）を決める。

改める決定と、据え置く決定は次のとおり。改める文と、改めた後の読み方は、決定 3 に並べる。

- [ADR-20261016-001][adr-pr]：決定 3 の「マージされずに閉じられていれば `needs-human`（人が閉じた意図を確かめる）」と、結びつきの印が無い PR で `needs-human` にする文に、人が戻した時刻より前に閉じた PR は止めずに結びつきから外す例外（本 ADR の決定 1）を足す。
  結びつけの 3 つの条件、人が戻した時刻より後に閉じた PR と開いている印の無い PR で止めること、fork からの PR を扱わないこと、`Closes #<番号>` を結びつけに使わないこと、Issue 1 件に結びつく PR は 1 件であること、結びついた PR がマージされていれば片付けへ進むことと、決定 1・2 は据え置く
- [ADR-20261021-003][adr-reapprove]：決定 2 の承認のやり直しの順（`needs-human` があれば外し、Status を `Todo` に戻す → `agent-ready` を外して付け直す）を、`needs-human` を付けたまま行って最後に外す順に改め、`needs-human` の無い Issue では最初に付けることを足す（本 ADR の決定 2）。
  文を直すのは `agent-ready` を付け直す前にすること、PR がある Issue では PR を閉じて作業ブランチを消すこと、計画コメントが渡す直前の検疫で見つかったときの扱いと、決定 1・3・4 は据え置く
- [ADR-20261021-004][adr-marker-keys]：決定 5 の、どの場合も `needs-human` を最後に外すことは据え置く（本 ADR の決定 2 と同じ向き）。決定 5 の「進行 Bot は、閉じた PR を結びつきから外し、計画からやり直す」は、本 ADR の決定 1 の例外によると読む
- [ADR-20261015-004][adr-markers]：決定 2 の、人が戻す手順に当たるときだけ Status の違いに従い、`needs-human` を外した人の権限で確かめることは据え置く。決定 2 が括弧で写した手順（`needs-human` を外して `Todo` か `In Review` に戻す）の順は、本 ADR の決定 2 の順で読む
- [ADR-20261015-003][adr-intake]：決定 2 の、検疫で見つかった Issue の人への案内（作業指示を一から書く）は据え置く。案内に書く戻す順は、本 ADR の決定 2 による

## 背景と課題（Context and Problem Statement）

- [ADR-20261016-001][adr-pr] の決定 3 は、Issue 1 件に結びつく PR を 1 件とし、結びついた PR が「マージされずに閉じられていれば `needs-human`（人が閉じた意図を確かめる）」とする。ブランチ名が一致するのに結びつきの印が無い PR があれば、`needs-human` にする（`unlinked-pr`）。
  どちらにも例外は無い。IMPL-6 も同じで、例外が無い
- 一方、人が止まった Issue を戻す手順のいくつかは、PR を閉じてから戻す
  - 承認のやり直しで、PR がある Issue の渡す文を直すとき（[ADR-20261021-003][adr-reapprove] の決定 2）
  - 途中のラウンドの `review` の印が無効なときと `round` が欠けるとき、印の署名の鍵が漏れたとき（[ADR-20261021-004][adr-marker-keys] の決定 5）
  - `unlinked-pr` と `pr-closed` で止まったとき（「[止まった状態から戻す手順][flow]」）

  閉じた PR は、閉じた後もブランチ名と結びつきの印で結びつく PR のまま残る。決定 3 のとおりに作ると、人が戻すたびに、同じ閉じた PR で止まる
- 基本設計の「[状態遷移と判定][design-flow]」の次の工程の順 4 と「戻した Issue のやり直し」、要件の「止まった状態から戻す手順」の `unlinked-pr` と `pr-closed` は、人が戻した時刻より前に閉じた PR（印の無い PR を含む）を、止めずに結びつきから外し、計画からやり直す。
  人が戻した時刻は、push 以上の人が最後に `needs-human` を外した時刻、最新の `intake-final` の印の時刻、承認のやり直しに当たる `agent-ready` を付けた時刻（新しい `intake-final` の印を書くまでの間）の、最も遅いもの（基本設計の「戻した Issue のやり直し」）。
  基本設計の 3 回目の再レビュー（2026-10-22）の H-5 の直し（PR #100）で、外す条件を、Status を問わず、着手の後も同じ閉じた PR を外し続ける形に広げた
- [ADR-20261021-003][adr-reapprove] と [ADR-20261021-004][adr-marker-keys] は、閉じた PR を外す形を前提にしているが、冒頭の改める決定に [ADR-20261016-001][adr-pr] を挙げていない。
  基本設計の 4 回目の再レビュー（2026-10-22）の H-4 は、例外が承認済みの決定 3 を注記も新しい ADR も無いまま改めていて、正典の ADR と IMPL-6 のとおりに作ると、人が PR を閉じて戻した Issue が止まり続けると指摘した
- 戻す手順で `needs-human` を外す位置が 2 通りある（4 回目の再レビューの M-20）
  - 要件の「止まった状態から戻す手順」の一般の手順と、[ADR-20261021-004][adr-marker-keys] の決定 5 は、`needs-human` を最後に外す。`needs-human` がある間は、進行 Bot は Issue に触らない（次の工程の順 2）
  - 承認のやり直し（要件の同じ節、[ADR-20261021-003][adr-reapprove] の決定 2、基本設計の「承認のやり直し」）、`unlinked-pr`（要件の同じ節）、検疫で止まったときの人への案内（TEXT-12、基本設計の受け入れの検疫の行、「[AI のセッション][design-ai]」の固定の文）、要件の図 `issue-flow` の `p8`、基本設計の図 `02-next-step-before-start` の `s1` は、`needs-human` を先に外す
- 承認のやり直しで `needs-human` を先に外すと、`agent-ready` を付け直すまで、承認のやり直しの条件（push 以上の人が `agent-ready` を付けた最新の時刻が、最新の `intake-final` の印より後）に当たらない。
  間に定期か手動の実行が入ると、次のどちらかで止まり直す（4 回目の再レビューの M-20。表をたどった結果で、実機では確かめていない）
  - PR があるとき：PR を閉じた時刻が、人が戻した時刻（`agent-ready` を付け直す前は、`needs-human` を外した時刻）より後なので、順 4 で `pr-closed` になる
  - PR が無いとき：Status を `Todo` に戻した後、前の `agent-ready` が付いたままなので、着手（順 11）の前の照らしで、直した文のハッシュが `intake-final` の印と違い、`brief-changed` になる

  `needs-human` の無い Issue の承認のやり直しも、人が操作を始めてから `agent-ready` を付け直すまでの間に、同じ窓がある

課題：人が戻した時刻より前に閉じた PR を、[ADR-20261016-001][adr-pr] の決定 3 の例外として結びつきから外すか。外すなら、人が戻した時刻をどう決め、Status と着手の前後でどう扱うか。
戻す手順で `needs-human` をどこで外し、`needs-human` の無い Issue の承認のやり直しで、途中の実行に触らせないにはどうするか。

## 判断の決め手（Decision Drivers）

- 人が手順どおりに戻した Issue が、閉じた PR で止まり続けないこと（3 回目の再レビュー（2026-10-22）の H-5 の形を作らない）
- 人が戻す手順を経ずに閉じられた PR では、今のとおり止まり、人が閉じた意図を確かめること（決定 3 の目的。止める側に倒す）
- 人が戻す操作の途中に実行が入っても、止まり直さないこと
- 戻す手順を 1 通りにし、人が覚える順と選ぶ値を増やさないこと
- 外すかを、GitHub に残る記録（timeline と印）だけから決め、着手の後と次の実行で読み方が変わらないこと（[ADR-20261015-006][adr-flow] の決定 4）
- 正典の ADR と、要件と設計の文が食い違わないこと。直し方を、[ADR の README][adr-readme] の基準で選ぶこと

## 検討した選択肢（Considered Options）

問い 1：人が戻す前に閉じた PR

- 案 1-A：新しい ADR で、[ADR-20261016-001][adr-pr] の決定 3 に例外を足す。人が戻した時刻より前に閉じた PR（結びつきの印の無い PR を含む）は、Status を問わず、止めずに結びつきから外す。IMPL-6 にも書く（直しの計画の案 C1）
- 案 1-B：[ADR-20261021-003][adr-reapprove] の決定 2 と [ADR-20261021-004][adr-marker-keys] の決定 5 が決定 3 を改めたと読み、2 本の冒頭の改める決定に ADR-20261016-001 を足す訂正の注記と、ADR-20261016-001 の注記を足す（直しの計画の案 C2）
- 案 1-C：例外を外し、決定 3 のとおり、閉じた PR はいつも `needs-human` にする（直しの計画の案 C3）

問い 2：戻す手順で `needs-human` を外す位置

- 案 2-A：どの戻す手順も、`needs-human` を付けたまま行い、最後に外す。`needs-human` の無い Issue で承認をやり直すときは、人が最初に `needs-human` を付ける（直しの計画の (i)）
- 案 2-B：`needs-human` のある Issue は案 2-A と同じく最後に外し、`needs-human` の無い Issue では、最初に Agent 欄を `paused` にし、最後に空に戻す（FLOW-32 の一時停止を使う。直しの計画の (ii)）
- 案 2-C：順を変えず、要件の一般の手順の「最後に」を外し、承認のやり直しでは PR を閉じてから `needs-human` を外すことを書き添える（直しの計画の (iii)）

## 決定（Decision Outcome）

> **［2026-10-24 追記・supersede］** 本 ADR の決定 1〜3 は、[ADR-20261024-001][adr-20261024-001] で置き換えた。人が止まった作業を戻す操作は `## 再開 <戻す先>` の見出しになり、`needs-human` を外す順の決まりは無くなった。閉じた PR は、戻す先が「下書きから」か「計画から」なら `## 再開` より前の PR を数えず、開いている前の PR は進行 Bot が閉じる。PR だけが閉じられたら停止中（`pr-closed`）にする（[ADR-20261024-001][adr-20261024-001] の決定 7）。以下の記述は当時の決定の記録として残す。

採用した選択肢:「案 1-A」「案 2-A」。
理由: 例外を承認済みの ADR の決定として正典に置き、[ADR-20261021-003][adr-reapprove] と [ADR-20261021-004][adr-marker-keys] が前提にした形を追認する。人が戻した時刻は GitHub の印とラベルの記録だけから決まり、着手の後も同じ値になる。
人が戻した時刻より後に閉じた PR は今のとおり止まり、人が閉じた意図を確かめる決定 3 の目的を保つ。
戻す手順は、`needs-human` を付けたまま行えば、人の操作の途中に実行が入っても進行 Bot が触らない（次の工程の順 2）。手順が 1 通りになり、[ADR-20261021-004][adr-marker-keys] の決定 5 と要件の一般の手順にそろう。
`needs-human` を外す時刻が手順の中で最も遅くなるので、手順どおりに閉じた PR は、いつも人が戻した時刻より前に閉じたことになり、案 1-A の例外に当たる。

### 決定 1：人が戻した時刻より前に閉じた PR は、止めずに結びつきから外す

- 進行 Bot は、マージされずに閉じた PR が、Issue を人が戻した時刻より前に閉じられていれば、`needs-human` にせず、閉じた PR を結びつきから外す。ほかに結びつく PR が無ければ、計画からやり直す（計画の印の読み方は、基本設計の「戻した Issue のやり直し」のまま）
- 外す PR には、結びつく PR（3 つの条件を満たす PR）と、ブランチ名が一致するのに結びつきの印が無い PR の両方を含む。結びつきの印を数えない印として無いものと読んだ PR（[ADR-20261021-004][adr-marker-keys] の決定 4）も、印の無い PR に当たる
- 人が戻した時刻は、次の 3 つのうち、最も遅いもの。どれも GitHub の timeline と印から読む
  - push 以上の人が最後に `needs-human` を外した時刻（timeline の `UnlabeledEvent`。外した人の権限は TEXT-10 で確かめる）
  - 最新の `intake-final` の印の投稿の時刻
  - 承認のやり直し（[ADR-20261021-003][adr-reapprove] の決定 1）に当たる `agent-ready` を、push 以上の人が付けた最新の時刻（timeline の `LabeledEvent`）。新しい `intake-final` の印を書くまでの間だけ使い、書いた後は、より遅い `intake-final` の印の時刻に含まれる
- 外すのは、Status を問わない。人が戻した時刻は着手の後も GitHub から同じ値に決まるので、着手の後の見直しと次の実行でも、同じ閉じた PR を外し続ける。外したことは、GitHub にも手元の記録にも書かない
- 人が戻した時刻より後に閉じた PR は、今のとおり止める。結びつく PR なら `pr-closed`、印の無い PR なら `unlinked-pr` にする。人が戻した後に、戻す手順を経ずに閉じられた PR で、人が閉じた意図を確かめるため（決定 3 の目的）
- 開いている印の無い PR は、閉じた時刻が無いので、今のとおり `unlinked-pr` で止める
- 結びつきから外した PR は、Issue 1 件に結びつく PR は 1 件の数に入れない。計画からやり直して作った新しい PR が、結びつく PR になる
- 外した PR の番号の名前のセッションのフォルダは、片付けと同じに消す（基本設計の「作業場所の残骸」のまま）

### 決定 2：止まった状態から戻す手順は、どれも needs-human を付けたまま行い、最後に外す

- 人は、どの戻す手順も、Issue に `needs-human` が付いた状態で行い、手順の最後に `needs-human` を外す。`needs-human` がある間は、進行 Bot は Issue に触らない（次の工程の順 2）ので、手順の途中に実行が入っても止まり直さない
- 承認のやり直しは、次の順に行う。文を直すのは、`agent-ready` を付け直す前にする（時刻の規則）
  1. `needs-human` が無ければ付ける
  2. 作業指示コメントか下書きを直す
  3. PR があれば閉じ、作業ブランチを消す
  4. Status を `Todo` に戻す
  5. `agent-ready` を外して付け直す
  6. 最後に `needs-human` を外す
- 承認のやり直しの 1 で `needs-human` を付けるときは、Status を変えなくてよい。`needs-human` がある間は、進行 Bot は Status を問わず Issue に触らず、Status は 4 で `Todo` に戻す
- `unlinked-pr`：PR の中身を確かめてから、PR を閉じ、作業ブランチを消し、Status を `Todo` に戻してから、最後に `needs-human` を外す
- `pr-closed`：今のとおり、作業ブランチの中身を確かめてから消し、Status を `Todo` に戻してから、最後に `needs-human` を外す
- 検疫で止まったとき（`quarantine`）の人への案内：処理対象にするなら、`## 作業指示` のコメントを一から書き、マイルストーン（要るなら依存も）を Issue に付け、Status を `Todo` に戻し、`agent-ready` を付けてから、最後に `needs-human` を外す
- `marker-invalid`（[ADR-20261021-004][adr-marker-keys] の決定 5）と `usage-limit` は、今のとおり最後に外す
- 最後に `needs-human` を外すのは、今のとおり push 以上の人。push 未満の人が外したときは、進行 Bot が `needs-human` を付け直す（`unauthorized-resume`。[ADR-20261015-004][adr-markers] の決定 2）

### 決定 3：改める文を、次のとおりに読み替える

- [ADR-20261016-001][adr-pr] の決定 3 の「結びついた PR がマージされていれば片付け（MERGE-7）へ進み、マージされずに閉じられていれば `needs-human`（人が閉じた意図を確かめる）」は、
  マージされずに閉じた PR のうち、人が戻した時刻より前に閉じた PR は止めずに結びつきから外し、人が戻した時刻より後に閉じた PR は `needs-human`（`pr-closed`）にすると読む（決定 1）
- 同じ決定 3 の「同じリポに、ブランチ名が一致するのに結びつきの印が無い PR があれば、Issue を `needs-human` にし、`note kind=stop reason=unlinked-pr` を投稿する」は、
  印の無い PR のうち、人が戻した時刻より前に閉じた PR は止めずに外し、開いている PR と、人が戻した時刻より後に閉じた PR は `unlinked-pr` にすると読む（決定 1）
- [ADR-20261021-003][adr-reapprove] の決定 2 の「作業指示コメントか下書きを直す → `needs-human` があれば外し、Status を `Todo` に戻す → `agent-ready` を外して付け直す」は、決定 2 の承認のやり直しの 6 つの順と読む。
  同じ決定 2 の「PR がある Issue の渡す文を直すときは、PR を閉じ、作業ブランチを消して `Todo` に戻してから付け直す」は、6 つの順の 3 と読む
- [ADR-20261021-004][adr-marker-keys] の決定 5 の「進行 Bot は、閉じた PR を結びつきから外し、計画からやり直す」は、人が戻す手順の中で閉じた PR が人が戻した時刻より前に閉じられたので、決定 1 の例外で外すと読む
- [ADR-20261015-004][adr-markers] の決定 2 の括弧の「`needs-human` を外して `Todo` か `In Review` に戻す」は、`Todo` か `In Review` に戻してから、最後に `needs-human` を外すと読む
- 決定を改める 2 本の ADR（[ADR-20261016-001][adr-pr]、[ADR-20261021-003][adr-reapprove]）の本文は書き換えず、承認の後に、タイトルの下に改めた旨の注記を足す（ADR の README の「決定の一部だけを改める」）。
  決定を変えない 2 本の ADR（[ADR-20261021-004][adr-marker-keys]、[ADR-20261015-004][adr-markers]）には、読み替える文のすぐ後ろに追記を足す

### 影響（Consequences）

- 良い点: 正典の ADR が、要件と設計の閉じた PR の扱いに追いつく。ADR と IMPL-6 のとおりに作っても、人が PR を閉じて戻した Issue が止まり続けない
- 良い点: 人が戻した時刻より後に閉じた PR では今のとおり止まり、人が閉じた意図を確かめる
- 良い点: 承認のやり直しの途中に実行が入っても、`pr-closed` か `brief-changed` で止まり直さない。着手した Issue で `agent-ready` を外しても、`needs-human` があるので `issue-changed` で止まらない（[ADR-20261021-003][adr-reapprove] の案 1-A の悪い点が無くなる）
- 良い点: 戻す手順が 1 通りになり、要件の一般の手順、[ADR-20261021-004][adr-marker-keys] の決定 5、要件の図 `issue-flow` の `p9` とそろう
- 悪い点: `needs-human` の無い Issue の承認のやり直しで、人の操作が 2 つ増える（最初に付け、最後に外す）
- 悪い点: 人が順を守らずに `needs-human` を先に外すと、途中の実行で `pr-closed` か `brief-changed` で止まる。止める側なので壊れはしないが、人は戻す操作をもう 1 回する
- 悪い点: 承認の後に、4 本の ADR への注記と追記と、要件・設計・図の反映の PR が要る。反映までの間は、要件と設計の戻す順が本 ADR より古いまま残る（UNDECIDED-74 に書いて示す）
- 中立: 結びつきから外したことは、GitHub にも手元の記録にも残らない。外すかは、実行のたびに、閉じた時刻と人が戻した時刻から決め直す
- 中立: 本 ADR の順は、`needs-human` を付けた後に始まる実行の判定を止める。`needs-human` を付ける前に始まっていた工程の扱いは変えない

### 確認方法（Confirmation）

- 進行 Bot の単体テストで、次を検査する
  - マージされずに閉じた結びつく PR が、人が戻した時刻より前に閉じられていれば、Status が `Todo`・`In Progress`・`In Review` のどれでも止めずに外し、後に閉じられていれば `pr-closed` にすること
  - 印の無い PR が、人が戻した時刻より前に閉じられていれば外し、開いているか、後に閉じられていれば `unlinked-pr` にすること
  - 人が戻した時刻を 3 つの時刻の最も遅いものとして決め、push 未満の人が外した `needs-human` と、承認のやり直しに当たらない `agent-ready` を数えないこと
- 結合テスト（偽の `gh`）で、次を確かめる
  - `needs-human` の無い Issue で、決定 2 の順で承認をやり直し、1〜6 の各手順の後に実行を入れても止まらず（`pr-closed`・`brief-changed`・`issue-changed`・`unexpected-status` にならず）、`needs-human` を外した後の実行が新しい `intake-final` の印を書いて計画し直すこと。PR がある筋書きと無い筋書きの 2 つで確かめる
  - 人が戻した後に 2 回以上の実行を通して、計画・実装・PR の作成まで止まらず、着手の後の見直しと次の実行で同じ閉じた PR に当たり直さないこと（`pr-closed`、`unlinked-pr`、承認のやり直しの 3 つの筋書き。承認のやり直しは決定 2 の順で行う）
  - `needs-human` を先に外してから PR を閉じた筋書きでは、間に入った実行が `pr-closed` で止まること（例外が、人が戻した時刻より後に閉じた PR に広がっていないこと）
- 基本設計のレビューで、IMPL-6、要件の「止まった状態から戻す手順」と決まり、TEXT-12 の人への案内、基本設計の次の工程の順 4・「戻した Issue のやり直し」・「承認のやり直し」・受け入れの検疫の行・stop の理由の記号の `pr-closed` と `unlinked-pr`、
  「[AI のセッション][design-ai]」の固定の文、図 `issue-flow` の `p8`、`02-next-step-before-start` の `s1`、`02-next-step-stops` の `o3b` が、本 ADR の決定と同じことを書いていることを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 1-A：新しい ADR で、決定 3 に人が戻した時刻より前に閉じた PR の例外を足す

- 良い点: 例外を正典の決定として置き、[ADR-20261021-003][adr-reapprove] と [ADR-20261021-004][adr-marker-keys] が前提にした形を追認する。ADR の README の基準（直した後の注記を読んで、決定が変わったと思う人がいるか）に沿う
- 良い点: 人が戻した時刻を GitHub の記録だけから決め、着手の後も同じ値になる。3 回目の再レビュー（2026-10-22）の H-5 の直し（Status を問わず、着手の後も外し続ける）を、決定として残せる
- 良い点: 人が戻した時刻より後に閉じた PR では止まり、決定 3 の目的を保つ
- 悪い点: ADR 1 本と、承認の後の反映の PR が要る。反映までの間、IMPL-6 が ADR より古いまま残る

### 案 1-B：2 本の ADR が決定 3 を改めたと読み、訂正の注記を足す

- 良い点: ADR を起票せず、注記を足す PR で済む
- 悪い点: PR #100 で広げた「Status を問わず」「着手の後も外し続ける」は、2 本の ADR の決定に無いので、注記では覆えない
- 悪い点: 人が戻した時刻の決め方と、人が戻した時刻より後に閉じた PR で止めることが、どの ADR の決定にも無いまま残る
- 悪い点: 訂正の注記は、事実の誤りか古くなった記述を直すもので、ADR の README の基準では、決定の変更に当たる直しに使えない

### 案 1-C：例外を外し、閉じた PR はいつも needs-human にする

- 良い点: [ADR-20261016-001][adr-pr] の決定 3 を変えずに済み、閉じた PR ではいつも人が確かめる
- 悪い点: 人が PR を閉じて戻す手順（[ADR-20261021-003][adr-reapprove] の決定 2、[ADR-20261021-004][adr-marker-keys] の決定 5、`unlinked-pr` と `pr-closed` の戻し方）が、閉じた PR で止まり続け、2 本の ADR と食い違う
- 悪い点: 閉じた PR が残る Issue を進めるには、人が Issue を閉じて作り直すしかなく、経緯が 2 つの Issue に分かれる

### 案 2-A：どの戻す手順も needs-human を付けたまま行い、最後に外す

- 良い点: `needs-human` があれば進行 Bot は触らないので、人の操作の途中に実行が入っても止まり直さない。`needs-human` の無い Issue の承認のやり直しの窓も閉じる
- 良い点: 手順が 1 通りで、[ADR-20261021-004][adr-marker-keys] の決定 5 と要件の一般の手順にそろう
- 良い点: `needs-human` を外す時刻が手順の中で最も遅くなるので、手順どおりに閉じた PR は、いつも案 1-A の例外に当たる
- 悪い点: `needs-human` の無い Issue では、人の操作が 2 つ増える

### 案 2-B：needs-human の無い Issue では Agent 欄を paused にする

- 良い点: `needs-human` の無い Issue で、止めた理由のラベルを人が付けずに済む
- 悪い点: 手順が `needs-human` の有無で 2 通りになる。最後に Agent 欄を戻す値（空か `worker`）を、人が選ぶ
- 悪い点: Agent 欄の変更は Issue の timeline に出ない（FACT-27、FACT-28）。`paused` から戻した時刻と人は、人が戻した時刻にも権限の確かめにも使えず、手順の終わりを GitHub の記録で決められない

### 案 2-C：順を変えず、承認のやり直しの書き添えだけにする

- 良い点: 要件と設計の手順の順を、書き添えのほかは変えずに済む
- 悪い点: `needs-human` の無い Issue の承認のやり直しで、途中の実行の窓が残る（`brief-changed` か `pr-closed` で止まり直す）
- 悪い点: 手順ごとに `needs-human` を外す位置が違ったまま残り、人が覚える順が増える

## 補足情報（More Information）

- 基本設計の 4 回目の再レビュー（2026-10-22）の H-4 と M-20 から起票した。メンテナが、直しの計画の推奨どおり、案 C1（本 ADR の案 1-A）と (i)（本 ADR の案 2-A）を選び、`proposed` で起票して、承認は PR ごとに決めるとした
- 3 回目の再レビュー（2026-10-22）では、[ADR-20261016-001][adr-pr] の決定 3 に注記が無いことは低の指摘（R8-15）だった。PR #100 で例外を広げたので、4 回目の再レビューで高（H-4）になった
- 決定 1 の中心の文（人が戻した時刻より前に閉じた PR を、Status を問わず外し続ける）は、PR #100 までの直しで、基本設計の順 4 と「戻した Issue のやり直し」、要件の `unlinked-pr` と `pr-closed` の戻し方に先に書いてある。
  UNDECIDED-74 は、4 回目の再レビューの直しの PR で[決めていないこと][undecided]に足す
- 関連する要件：IMPL-2、IMPL-6（[02-implement-review-fix.md][impl]）、FLOW-1、FLOW-29、FLOW-32、「止まった状態から戻す手順」と決まり（[01-issue-flow.md][flow]）、TEXT-4、TEXT-10、TEXT-12（[04-external-text-and-markers.md][text]）、UNDECIDED-74（[09-undecided.md][undecided]）、FACT-27、FACT-28（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261016-001][adr-pr]、[ADR-20261021-003][adr-reapprove]、[ADR-20261021-004][adr-marker-keys]、[ADR-20261015-004][adr-markers]、[ADR-20261015-003][adr-intake]、[ADR-20261015-006][adr-flow]
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 02-implement-review-fix.md：IMPL-6 の「マージされずに閉じられていれば `needs-human`」と `unlinked-pr` の文に、決定 1 の例外（人が戻した時刻より前に閉じた PR は、Status を問わず、止めずに結びつきから外し、ほかに結びつく PR が無ければ計画からやり直す）と、本 ADR へのリンクを足す。人が戻した時刻の決め方は、基本設計の「戻した Issue のやり直し」を指す
  - 01-issue-flow.md：「止まった状態から戻す手順」を、次のように直す
    - 一般の手順（`needs-human`）に、どの手順も `needs-human` を付けたまま行うことを書き添える
    - `unlinked-pr` の手順を決定 2 の順にし、`unlinked-pr` と `pr-closed` の進行 Bot の扱いに本 ADR へのリンクを足す
    - 承認のやり直しを、決定 2 の 6 つの順にする
    - 検疫で止まったとき（`quarantine`）の戻し方を、決定 2 の順で足し、TEXT-12 を指す。止めた理由ごとの手順の並びに `quarantine` を足す
    - FLOW-29 の「マージされずに閉じられた PR（IMPL-6）」に、人が戻した時刻より前に閉じた PR を除くことを書き添える
    - 決まりの「`needs-human` を付けるときは、どの契機でも Status を `Blocked` にする」に、人が戻す手順の最初に付けるときは Status を変えなくてよいことを書き添える
  - 04-external-text-and-markers.md：TEXT-12 の、検疫で見つかったときの人への案内の順を、決定 2 の順にする
  - 09-undecided.md：UNDECIDED-74 を欠番にする（元の文を取り消し線で残す）
  - 要件の図 `issue-flow`：`p8` を決定 2 の承認のやり直しの順にする
  - 基本設計の「[状態遷移と判定][design-flow]」：次のように直す
    - 次の工程の順 4 と「戻した Issue のやり直し」の根拠に、本 ADR を足す
    - 「承認のやり直し」の冒頭の人の手順を決定 2 の順にし、順 4 と順 8 で止めない理由の文（`needs-human` の無い Issue を手順どおりに戻したとき）を、決定 2 の順に合わせて直す
    - 受け入れの検疫の行（順 9）の人への案内を、決定 2 の順にする
    - stop の理由の記号の `pr-closed` と `unlinked-pr` の行に、人がすることを決定 2 の順で書く
  - 基本設計の「[AI のセッション][design-ai]」：受け入れの検疫で止まったときの人への案内の固定の文を、決定 2 の順にする
  - 基本設計の「[テスト][design-tests]」：結合テストの、人が戻した後の 3 つの筋書きのうち承認のやり直しを、決定 2 の順で途中に実行を入れても止まらない筋書きに直す。確認方法の単体テストと、`needs-human` を先に外した筋書きを足す
  - 図 `02-next-step-before-start`：`s1` を決定 2 の承認のやり直しの順にし、指す先を要件の「止まった状態から戻す手順」にする
  - 図 `02-next-step-stops`：`o3b` が決定 1 のとおり（人が戻した時刻より前に閉じた PR だけを外す）かを、図の要素で確かめる
  - [ADR-20261016-001][adr-pr]：タイトルの下に、本 ADR で決定 3 に人が戻した時刻より前に閉じた PR の例外を足した旨の注記を足す
  - [ADR-20261021-003][adr-reapprove]：タイトルの下に、本 ADR で決定 2 の承認のやり直しの順を改めた旨の注記を足す
  - [ADR-20261021-004][adr-marker-keys]：決定 5 の「閉じた PR を結びつきから外し、計画からやり直す」のすぐ後ろに、本 ADR の決定 1 の例外によることの追記を足す
  - [ADR-20261015-004][adr-markers]：決定 2 の Status の扱いの括弧のすぐ後ろに、戻す順は本 ADR の決定 2 によることの追記を足す

[adr-pr]: ./adr-20261016-001-ci-states-merge-ready-limit-and-pr-issue-link.md
[adr-reapprove]: ./adr-20261021-003-reapprove-by-relabel-and-ignore-outside-instructions.md
[adr-marker-keys]: ./adr-20261021-004-marker-key-list-and-resume-after-invalid-markers.md
[adr-markers]: ./adr-20261015-004-signed-markers-and-actor-checks.md
[adr-intake]: ./adr-20261015-003-intake-scope-and-quarantine-report.md
[adr-flow]: ./adr-20261015-006-run-stages-baseline-agent-field-and-local-records.md
[adr-readme]: ./README.md
[flow]: ../requirements/01-issue-flow.md
[impl]: ../requirements/02-implement-review-fix.md
[text]: ../requirements/04-external-text-and-markers.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
[design-ai]: ../design/basic/03-ai-sessions.md
[design-tests]: ../design/basic/05-tests.md
[adr-20261024-001]: ./adr-20261024-001-work-unit-main-flow-and-overlay-axes.md
