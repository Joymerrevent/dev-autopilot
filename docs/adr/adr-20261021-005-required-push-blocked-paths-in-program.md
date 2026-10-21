---
status: proposed
created: 2026-10-21
updated: 2026-10-21
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# push しないパスのうち要件が名指しするものは進行 Bot のプログラムに固定し、リポの設定には足す分だけを持つ

要件の UNDECIDED-64（IMPL-15 が名指しする push しないパスを、設定の PR で外せない形にするか）を決める。

改める決定と、据え置く決定は次のとおり。

- [ADR-20261015-005][adr-settings]：決定 1 の「安全の境界を緩める変更」のうち、「進行役が実行するパスの一覧（IMPL-15）からパスを外す」を、設定の値の変更として ADR で決める形を改める。
  IMPL-15 が名指しするパスは設定の値でなくなり、外すのは要件と進行 Bot のプログラムの変更として ADR で決める（本 ADR の決定 1 と決定 3）。足す分（規約の正典のパスなど）から外すのを ADR で決めることと、決定 1 のほかの項目と、決定 2 は据え置く
- [ADR-20261017-002][adr-review]：決定 5 の「一覧はリポの設定に持ち（STRUCT-2）、初期値は構造仕様の設定の表で決める」を、名指しのパスはプログラムに固定し、規約の正典のパスはリポの設定の足す分に持つ形に改める（本 ADR の決定 2）。
  規約の正典に触れる diff を push しないこと、正典のパスの初期値を構造仕様の表で決めること、正典のパスを外すのを ADR で決めることと、決定 1〜4 は据え置く
- [ADR-20261016-003][adr-move]：決定 5 の、business-os から `dev-autopilot/` を消した後も `dev-autopilot/**` を push しないパスに残すことは据え置く。`dev-autopilot/**` は名指しのパスに入るので、プログラムに固定される
- [ADR-20261019-004][adr-policy]：決定 2 の、計画が変える見込みのパスを push しないパスと照らして止めることは据え置く。照らす一覧は、本 ADR の決定 2 の合わせた一覧にする

## 背景と課題（Context and Problem Statement）

- IMPL-15 は、diff が「進行 Bot・次のセッション・CI が実行するか、規則として読むパス」に触れていれば、push せずに `needs-human` にするとし、パスを名指しする：`.claude/**` `CLAUDE.md` `**/CLAUDE.local.md` `.mcp.json` `.github/**` `dev-autopilot/**` と、規約の正典のパス（business-os なら `docs/adr/**` `docs/design/**`）。一覧は設定に持つ。
  `**/CLAUDE.local.md` は、基本設計の 2 回目の再レビュー（2026-10-21）の直しで足した（FACT-60）
- 名指しのパスで止めたいものは 3 つある
  - CI が、作業 AI の書いたワークフローを secrets 付きで実行すること（`.github/**`。IMPL-15）
  - 進行 Bot と次のセッションが、作業 AI の書いた設定と指示を規則として読むこと（`.claude/**` `CLAUDE.md` `**/CLAUDE.local.md` `.mcp.json`。SAFE-2 の (a) と (c)）
  - 作業 AI が進行 Bot のコードを書き換えること（`dev-autopilot/**`。[ADR-20261016-003][adr-move] の決定 5）
- STRUCT-2 と [ADR-20261015-005][adr-settings] の決定 1 は、IMPL-15 の一覧からパスを外すのを、安全の境界を緩める値の変更として ADR で決めるとする。
  基本設計の「[設定][design-settings]」の `push.blockedPaths` の行も、初期値に名指しのパスを書き、外すのは ADR で決めるとする
- 点検（SETUP-2）は、`push.blockedPaths` の中身を見ない。人の誤りの PR 1 本で `.github/**` が外れてマージされると、次の実行から、作業 AI の書いたワークフローが push されうる。守りは、PR のレビューと「ADR で決める」の運用だけになる
  - `.claude/**` が外れると、作業 AI の diff がリポの設定のファイル（`.claude/dev-autopilot.json`）に触れても push される。基本設計の「[設定][design-settings]」は、AI のセッションがリポの設定を書き換えられない理由を、`.claude/**` が push しないパスであることに置いている
- リポの設定は、進行 Bot が base ブランチの先端から読む（[ADR-20261017-007][adr-stop] の決定 4）。作業 AI は `.claude/**` に触れる diff を push できないので、一覧からパスを外せるのは人の PR だけで、心配は人の誤り
- 一方、sandbox の `denyRead` と、子プロセスに渡す環境変数の名前は、要件と ADR が決めた分を進行 Bot のプログラムに固定し、設定には足す分だけを持つ（STRUCT-2、基本設計の「[設定][design-settings]」と「[AI のセッション][design-ai]」）。設定の PR で消せないようにするため。
  push しないパスだけが形が違い、違う理由は要件にも設計にも無い
- UNDECIDED-64 は、計画・実装工程（段階 5）で作業 AI の変更を push し始める前に決めるとしている

課題：IMPL-15 が名指しする push しないパスを、リポの設定の PR で外せる形のままにするか。外せない形にするなら、どの形にするか。

## 判断の決め手（Decision Drivers）

- 人の誤りの PR 1 本で、名指しのパスの守りが外れないこと（フェイルセーフ）
- 要件が決めた守りを設定の PR で消せない形を、`denyRead` と環境変数の名前と 1 つにそろえ、読み方を 1 つにすること
- 当てる先のリポごとに違うパス（規約の正典のパス）を、リポの設定で足せること
- 名指しのパスを外す道は残し、外すときは ADR を通ること
- PR のレビューや点検のように、人や後の検査が気づくことに頼らないこと

## 検討した選択肢（Considered Options）

- 案 A：名指しのパスを進行 Bot のプログラムに固定し、リポの設定には足す分（規約の正典のパスなど）だけを持つ（UNDECIDED-64 の候補 (a)）
- 案 B：今の一覧の形のまま、点検で名指しのパスが一覧にあることを確かめ、無ければ fail にする（候補 (b)）
- 案 C：今の形を保つ（候補 (c)）

## 決定（Decision Outcome）

採用した選択肢:「案 A」。
理由: リポの設定の PR で名指しのパスを外す道が無くなり、人の誤りの PR がマージされても、作業 AI の書いたワークフローと指示は push されない。`denyRead` と環境変数の名前がすでに使っている「要件の分はプログラムに固定し、設定は足す分だけ」の形にそろい、読み方が 1 つで済む。点検に項目を足さずに済む。

### 決定 1：要件が名指しする push しないパスは、進行 Bot のプログラムに固定する

- 進行 Bot は、`.claude/**` `CLAUDE.md` `**/CLAUDE.local.md` `.mcp.json` `.github/**` `dev-autopilot/**`（IMPL-15 が名指しするパス。以下「名指しのパス」）を、リポの設定の値に関わらず、push しないパスに必ず入れる
- 名指しのパスの一覧は、dev-autopilot のリポの進行 Bot のコード（`plugin/src/`）に持つ。当てる先のリポの設定の PR では変わらない。
  進行 Bot はプログラムの写しから動き、AI のセッションは写しを書けない（STRUCT-3、SAFE-16）。dev-autopilot は dev-autopilot 自身のリポに当てない（[07-structure.md][struct] の「分離の時期の目安」）
- sandbox の `denyRead` と、子プロセスに渡す環境変数の名前と同じ形（要件の分はプログラムに固定し、設定には足す分だけを持つ）にそろえる

### 決定 2：リポの設定の `push.blockedPaths` は、足す分だけを持つ

- `push.blockedPaths` には、当てる先のリポごとに違う push しないパスだけを書く。今あるのは規約の正典のパスで、business-os は `docs/adr/**` `docs/design/**`。
  規約の正典のうち `CLAUDE.md` は名指しのパスに入るので、足す分に書かなくても止まる
- 進行 Bot は、push しないパスを使うところのすべてで、名指しのパスと足す分を合わせた一覧を使う
  - push の前の検査（IMPL-15）
  - 計画が変える見込みのパスとの照らし（IMPL-10、[ADR-20261019-004][adr-policy] の決定 2）
  - 作業 AI の固定の指示文に書く push しないパス
- 足す分に名指しのパスと同じパスがあっても、合わせた一覧は変わらないので、点検で fail にしない。点検（SETUP-2）に、一覧の中身を確かめる項目は足さない
- 足す分にパスを足すのは、止める側に広げる変更なので、リポの設定の PR で行う。足す分からパスを外すのは、今のとおり安全の境界を緩める変更として ADR で決める（[ADR-20261015-005][adr-settings] の決定 1）

### 決定 3：名指しのパスを変えるのは、要件と進行 Bot のプログラムの変更として行う

- 名指しのパスを外すのは、設定の値の変更でなく、IMPL-15 と進行 Bot のプログラムを変えることにする。安全の境界を緩める変更なので、新しい ADR で決め、承認の後に要件とプログラムを直す
- 名指しのパスを足すのは、止める側に広げる変更なので、要件の直し（要件の README の「要件を変えるとき」）で IMPL-15 に足し、プログラムの一覧を IMPL-15 に合わせる。`**/CLAUDE.local.md` を足したときと同じ
- 名指しのパスは、当てる先のリポすべてに同じく掛かる。リポごとに名指しのパスを外す形は持たない。要るときは、新しい ADR で決める

### 影響（Consequences）

- 良い点: リポの設定の PR で名指しのパスを外せない。人の誤りの PR がマージされても、作業 AI の書いたワークフローは push されず、CI が secrets 付きで実行する道が開かない
- 良い点: `.claude/**` が外れないので、作業 AI の diff がリポの設定のファイルに触れれば、いつも push しない
- 良い点: `denyRead` と環境変数の名前と同じ形になり、要件の分と設定の足す分を、1 つの読み方で読める
- 良い点: 点検に項目を足さずに済む
- 悪い点: 名指しのパスを変えるには、dev-autopilot のリポでプログラムを直して新しい版を出し、当てる先のリポで `setup` を回して写しを替える（STRUCT-3）。設定の PR より時間がかかる
- 悪い点: リポの設定のファイルを読むだけでは、push しないパスの全体が分からない。基本設計の設定の表に、名指しのパスを書いて補う（`safety.env.names` の行と同じ）
- 中立: 名指しのパスは、当てる先のリポすべてに同じく掛かる。`dev-autopilot/**` は business-os のための名指しだが、ほかのリポでも止める側に働く。ほかのリポへの適用は、マージ判定工程（段階 8）の後に決める（[ADR-20261016-003][adr-move]）
- 中立: 急いで止める側に広げたいときは、リポの設定の足す分に足せる

### 確認方法（Confirmation）

- 進行 Bot の単体テストで、次を検査する
  - リポの設定の `push.blockedPaths` が空でも、名指しのパスに触れる diff を push せず、`needs-human` にすること
  - 足す分のパスに触れる diff も、push せず `needs-human` にすること
  - 計画が変える見込みのパスとの照らしと、作業 AI の固定の指示文が、名指しのパスと足す分を合わせた一覧を使うこと
  - プログラムの名指しのパスの一覧が、IMPL-15 の名指しのパスと同じであること（テストの期待値に IMPL-15 の一覧を書く）
- 結合テストで、`push.blockedPaths` に `.github/**` が無いリポの設定で、作業 AI が `.github/workflows/` に足した diff を push しないことを確かめる
- 基本設計のレビューで、設定の表の `push.blockedPaths` の初期値に名指しのパスが無く、名指しのパスをプログラムに固定することが行の中に書いてあることを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：名指しのパスをプログラムに固定し、設定は足す分だけ

- 良い点: リポの設定の PR で名指しのパスを外せない
- 良い点: `denyRead` と環境変数の名前と同じ形にそろう
- 良い点: 点検に項目を足さずに済む
- 悪い点: 名指しのパスを変えるには、プログラムの新しい版と、当てる先のリポでの `setup` が要る
- 悪い点: リポの設定のファイルだけでは、push しないパスの全体が分からない

### 案 B：点検で名指しのパスが一覧にあることを確かめる

- 良い点: リポの設定のファイルに、push しないパスの全体が残る
- 良い点: 名指しのパスを外す PR がマージされても、次の実行の最初の点検で fail になり、作業に入らない（SETUP-2 は毎回の実行の最初に回す）
- 悪い点: 点検のために、名指しのパスの一覧をプログラムにも持つ。同じ一覧がプログラムと設定の 2 か所になり、外すときは両方を直す
- 悪い点: 守りが点検の項目に頼る。点検の項目が漏れると、push の前の検査は設定の一覧だけで照らす
- 悪い点: `denyRead` と環境変数の名前と形が違ったまま
- 中立: 外す PR がマージされると、人が戻すまで、push しないパスに関係の無い Issue も止まる（止める側に倒れる）

### 案 C：今の形を保つ

- 良い点: 要件と設計を変えずに済む。一覧が設定の 1 か所にある
- 悪い点: 人の誤りの PR 1 本で名指しのパスが外れる。守りは PR のレビューと「ADR で決める」の運用だけ
- 悪い点: `denyRead` と環境変数の名前と形が違ったまま、違う理由も無い

## 補足情報（More Information）

- 基本設計の 2 回目の再レビュー（2026-10-21）の指摘から起票した。メンテナが、直しの計画の推奨どおり候補 (a) を選んだ
- 関連する要件：IMPL-10、IMPL-15（[02-implement-review-fix.md][impl]）、SAFE-2 の (a) と (c)、SAFE-16（[05-safety.md][safety]）、SETUP-2（[06-setup-and-stages.md][setup]）、STRUCT-2、STRUCT-3 と「分離の時期の目安」（[07-structure.md][struct]）、UNDECIDED-64（[09-undecided.md][undecided]）、FACT-60（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261015-005][adr-settings]、[ADR-20261017-002][adr-review]、[ADR-20261016-003][adr-move]、[ADR-20261019-004][adr-policy]、[ADR-20261017-007][adr-stop]
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 02-implement-review-fix.md：IMPL-15 の「設定に持つ」を、名指しのパスは進行 Bot のプログラムに固定し、設定には足す分（規約の正典のパスなど）だけを持つ、に直す。名指しのパスを外すのは要件とプログラムの変更として ADR で決めることを書く
  - 05-safety.md：SAFE-2 の (c) に、名指しのパスを進行 Bot のプログラムに固定することを書き添える
  - 07-structure.md：STRUCT-2 の安全の境界を緩める変更の「IMPL-15 の一覧からパスを外す」を、名指しのパスを外すこと（要件とプログラムの変更）と、push しないパスの足す分からパスを外すことに分けて書く。
    「検査と判定」の push しないパスの一覧を、足す分の一覧に直し、名指しのパスは進行 Bot が必ず入れることを書く（`denyRead` の書き方と同じ）
  - 09-undecided.md：UNDECIDED-64 を欠番にする（元の文を取り消し線で残す）
  - 設計の README の「設計を変えるとき」：安全の境界を緩める変更の「IMPL-15 の一覧からパスを外す」を、07-structure.md と同じ言い方に直す
  - 基本設計の「[設定][design-settings]」：`push.blockedPaths` の行の中身を足す分にし、初期値を規約の正典のパスだけ（business-os は `docs/adr/**` `docs/design/**`）にし、名指しのパスはプログラムに固定することを書く（`safety.env.names` の行と同じ形）
  - 基本設計の「[状態遷移と判定][design-flow]」：push の前の検査の順 3 と、計画の工程の順 3 の照らす相手を、名指しのパスと `push.blockedPaths` を合わせた一覧にする。図 `02-push` の「`push.blockedPaths` と照らす」も直す
  - 基本設計の「[AI のセッション][design-ai]」：作業 AI の固定の指示文に書く push しないパスを、合わせた一覧にする
  - 基本設計の「テスト」：確認方法の単体テストと結合テストを足す
  - [ADR-20261015-005][adr-settings]：タイトルの下に、本 ADR で決定 1 の「IMPL-15 の一覧からパスを外す」の形を改めた旨の注記を足す
  - [ADR-20261017-002][adr-review]：タイトルの下に、本 ADR で決定 5 の一覧の置き場を改めた旨の注記を足す

[adr-settings]: ./adr-20261015-005-settings-values-and-machine-settings.md
[adr-review]: ./adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-move]: ./adr-20261016-003-move-to-own-repository-before-basic-design.md
[adr-policy]: ./adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths.md
[adr-stop]: ./adr-20261017-007-stop-switch-in-repo-variable.md
[impl]: ../requirements/02-implement-review-fix.md
[safety]: ../requirements/05-safety.md
[setup]: ../requirements/06-setup-and-stages.md
[struct]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-settings]: ../design/basic/01-settings.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
[design-ai]: ../design/basic/03-ai-sessions.md
