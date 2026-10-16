---
status: accepted
created: 2026-10-17
updated: 2026-10-17
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# dev-autopilot の履歴を新しいリポへ引き継ぎ、PR の番号を business-os の参照に書き換えて、メンテナの鍵で署名を付け直す

[ADR-20261016-003][adr-move]（基本設計の前に別のリポへ移す）の決定 1 のうち、「履歴は引き継がない」を改める。
ADR-20261016-003 のほかの決定（移す時期、リポの名前と公開の範囲、文書の参照の直し方、決定 2〜6）は据え置く。

## 背景と課題（Context and Problem Statement）

ADR-20261016-003 の決定 1 は、新しいリポ `Joymerrevent/dev-autopilot` の最初の commit を、business-os の `dev-autopilot/` の中身をそのまま写したものにし、履歴は引き継がないとした。
理由は、business-os の squash の commit の題に PR の番号（`(#120)` など）が付いていて、GitHub の commit の画面では題の `#<番号>` が同じリポの Issue と PR へのリンクになるため。
履歴を引き継ぐと、移した先では別の Issue と PR を指す。移す前の経緯は business-os の履歴で読むとした。

2026-10-17 に、メンテナが `dev-autopilot/` の commit の履歴も新しいリポへ持ってきたいとした。2026-10-17 の時点の状態は次のとおり。

- 新しいリポは作ってあり、まだ commit が無い。手元で ADR-20261016-003 の決定 1 のとおりに作った最初の commit は、push していない
- business-os の `develop`（d311493）で `dev-autopilot/` に触れた commit は 28 件。merge の commit は無く、一直線に並んでいる
- 28 件のメッセージには、PR の番号の参照（`#` と 2〜3 桁の数字）が 50 か所ある。28 件すべての題に `(#<番号>)` がある
- 28 件はすべて、committer が GitHub の squash マージの commit で、署名が付いている。GitHub の API で d311493 の署名を確かめると `verified` だった
- 手元には `git filter-repo` が入っている。メンテナの git は、commit に SSH の鍵で署名する設定（`commit.gpgsign` が true、`gpg.format` が ssh）

決定 1 の理由にした PR の番号の問題は、取り出すときにメッセージを書き換えれば起きない。一方、メッセージや親の commit を書き換えると、GitHub が付けた署名は無効になる。

課題：`dev-autopilot/` の履歴を新しいリポへ引き継ぐか。引き継ぐなら、PR の番号と署名をどう扱うか。

## 判断の決め手（Decision Drivers）

- 新しいリポで、文書の経緯（`git log`・`git blame`）を読めること
- commit のメッセージの PR の番号が、移した先で別の Issue と PR を指さないこと
- 署名の無い commit を、新しいリポに入れないこと
- business-os の公開済みの履歴を書き換えないこと（business-os の CLAUDE.md の絶対ルール 6）
- 引き継いだ中身が business-os の中身と同じだと、機械で確かめられること

## 検討した選択肢（Considered Options）

- 案 A：`git filter-repo` で `dev-autopilot/` の commit だけを取り出し、メッセージの PR の番号を business-os の参照に書き換え、メンテナの鍵で署名を付け直す
- 案 B：`git filter-repo` で取り出し、メッセージを書き換えるが、署名は付け直さない
- 案 C：`git subtree split` で取り出し、メッセージは書き換えない
- 案 D：ADR-20261016-003 の決定 1 のまま、履歴を引き継がない

## 決定（Decision Outcome）

採用した選択肢:「案 A」。
理由: 新しいリポで文書の経緯を読めるようになり、PR の番号は business-os の PR を指したまま残り、すべての commit に署名が付く。business-os の履歴には手を入れない。

### 決定 1：`dev-autopilot/` の commit だけを取り出す

- 取り出す元は、この ADR の PR をマージした後の business-os の `develop` の先頭にする。この ADR とこの ADR の一覧の 1 行も、引き継ぐ履歴に入る
- business-os を origin から一時的なフォルダに clone し直し、その clone で `git filter-repo --subdirectory-filter dev-autopilot` を実行する。`dev-autopilot/` に触れた commit だけが残り、フォルダの中身がリポの root に移る
- 書き換えるのは一時的なフォルダの clone だけで、business-os の clone と GitHub の business-os には push しない。取り出した clone は、origin を新しいリポに向けてから push する
- 手元で ADR-20261016-003 の決定 1 のとおりに作った最初の commit（push していないもの）は使わない
- 取り出した後は、business-os の `dev-autopilot/` を変更しない（ADR-20261016-003 の決定 1 のまま）

### 決定 2：メッセージの PR の番号を、business-os の参照に書き換える

- メッセージの中の `#<数字>` を、`Joymerrevent/business-os#<数字>` に書き換える（例：`(#120)` → `(Joymerrevent/business-os#120)`）。GitHub は `<owner>/<repo>#<数字>` の形を、そのリポの Issue と PR へのリンクにする
- 書き換えるのは、`#` の直前が英数字と `/` 以外で、`#` の後が数字だけの参照に限る。2026-10-17 の時点で当てはまるのは 50 か所で、すべて PR の番号
- 番号の参照のほかは、メッセージを変えない。author と、author の日時も変えない

### 決定 3：署名を、メンテナの鍵で付け直す

- 書き換えた commit には、GitHub が付けた署名が残らない。署名の無い commit を残さないよう、取り出して書き換えた後に、すべての commit にメンテナの SSH の鍵で署名を付け直す（例：`git rebase --root --gpg-sign`）
- 付け直した commit の committer はメンテナになる。committer の日時は、author の日時にそろえる（例：`--committer-date-is-author-date`）
- 新しいリポの commit の SHA は、business-os の commit の SHA と変わる。文書の中にある business-os の SHA を固定したリンクは、business-os を指したままなので壊れない

### 決定 4：最初の push

- 取り出した履歴を、1 回の `git push` でまとめて新しいリポの `main` に送る。送る commit は、business-os の 28 件と、ADR-20261017-001 の PR の squash の commit 1 件の、合わせて 29 件の見込み。
  ADR-20261017-001 の PR をマージした後、取り出す前に `dev-autopilot/` の変更が `develop` に入れば、その分だけ増える
- 空のリポには元になるブランチが無く、PR を作れないので、`main` へ直接 push するのはこの 1 回だけにする。push の前にメンテナに確かめる
- 2 回目以降の変更は、ADR-20261016-003 の決定 6 のとおり、新しいリポの PR で行う

### 影響（Consequences）

- 良い点: 新しいリポで、要件メモを足したとき（business-os の PR #81）からの経緯を `git log` と `git blame` で読める
- 良い点: メッセージの PR の番号は、business-os の PR を指したまま残る。新しいリポの Issue と PR が増えても、別のものを指さない
- 良い点: 新しいリポのすべての commit に署名が付く
- 悪い点: 署名は GitHub の署名からメンテナの署名に変わり、committer もメンテナに変わる。元の squash マージを GitHub が行ったことは、新しいリポの commit からは読めない（business-os の履歴で読む）
- 悪い点: メッセージの 50 か所を書き換えるので、新しいリポのメッセージは business-os の元のメッセージと文字の上で一致しない
- 悪い点: 取り出し・書き換え・署名の付け直しの手順を間違えると、中身の違う履歴ができうる。確認方法の tree のハッシュで、push の前に止める
- 中立: business-os の履歴は変わらない

### 確認方法（Confirmation）

新しいリポの `main` に push する前に、取り出した clone で次を確かめる。1 つでも外れたら push しない。

- 最後の commit の tree のハッシュが、business-os の `<取り出した元の commit>:dev-autopilot` の tree のハッシュと同じ
- commit の数が、business-os で `git log --oneline <取り出した元の commit> -- dev-autopilot` が返す数と同じ
- どの commit のメッセージにも、`Joymerrevent/business-os` の付かない `#<数字>` の参照が無い
- どの commit にも署名（`gpgsig`）がある
- author の名前・メールアドレス・日時が、business-os の対応する commit と同じ
- 取り出した clone の remote に、business-os が無い

push した後に、新しいリポで次を確かめる。

- GitHub の API で、すべての commit の署名の `verified` が true（メンテナの SSH の鍵が、GitHub に署名の鍵として登録されていること）

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：filter-repo で取り出し、PR の番号を書き換え、署名を付け直す

- 良い点: 経緯を読め、PR の番号が正しく business-os を指し、すべての commit に署名が付く
- 悪い点: 署名と committer がメンテナに変わる
- 悪い点: 取り出しの後に、署名を付け直す手順が 1 つ増える

### 案 B：filter-repo で取り出し、署名は付け直さない

- 良い点: committer と、committer の日時が元のまま残る
- 悪い点: 新しいリポの 28 件以上の commit に署名が無い。メンテナは commit に署名する設定にしていて、署名の無い commit を混ぜないことを決め手にした

### 案 C：`git subtree split` で取り出し、メッセージは書き換えない

- 良い点: git に入っている道具だけで済む
- 悪い点: メッセージの PR の番号が、新しいリポの Issue と PR を指す。ADR-20261016-003 が履歴を引き継がない理由にした問題が残る
- 中立: 取り出した commit に署名がどう付くかは確かめていない

### 案 D：履歴を引き継がない

- 良い点: 手順が最も少なく、ADR を改めずに済む
- 悪い点: 新しいリポの `git log` と `git blame` が、写した commit から始まる。それより前の経緯は business-os の履歴で読むことになる

## 補足情報（More Information）

- 関連する ADR：[ADR-20261016-003][adr-move]（基本設計の前に別のリポへ移す）
- 要件と ADR への反映（承認の後に、新しいリポの PR で行う）
  - ADR-20261016-003：タイトルの下に、決定 1 の「履歴は引き継がない」をこの ADR で改めた旨の注記（Amended by）を足す。
    あわせて、確認方法の「最初の commit の本文に、写した元の business-os の commit の SHA がある」と、影響の「新しいリポの `git log` と `git blame` は、写した commit から始まる」が、この ADR の決定で当てはまらなくなったことを注記に書く
- 範囲外：新しいリポのブランチの運用、ライセンス、入れる検査（ADR-20261016-003 の決定 6 の 3 番目で決める）

[adr-move]: ./adr-20261016-003-move-to-own-repository-before-basic-design.md
