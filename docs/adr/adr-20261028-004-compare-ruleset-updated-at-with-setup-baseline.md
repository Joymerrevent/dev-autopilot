---
status: proposed
created: 2026-10-28
updated: 2026-10-28
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 進行 Bot の毎回の点検は、setup が bypass を確かめたときの ruleset の updated_at を控えと比べ、変わっていれば止める

要件の UNDECIDED-94（GitHub App のトークンで `bypass_actors` を読めないとき、毎回の点検で ruleset の bypass をどう確かめるか。[決めていないこと][undecided]の「ADR で決めること」の「安全と設定」）を決める。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261017-004][adr-human]（setup の GitHub の準備は人の gh、check は進行 Bot のジョブ） | 据え置く | — | 決定 2（`setup` の GitHub の準備は人の `gh` のログインで行い、GitHub の点検は進行 Bot が自分の認証で行う）。本 ADR は、bypass を `setup` で確かめ、進行 Bot は変わっていないことを確かめる形を、決定 2 の分担の中で決める |
| [ADR-20261020-002][adr-app]（GitHub に書き始める前に GitHub App に切り替える） | 据え置く | — | 決定 2 の権限の表（Administration を付けない） |

## 背景と課題（Context and Problem Statement）

- 要件の SETUP-2 は、進行 Bot が毎回の実行の最初に、自分の認証で ruleset を確かめ、bypass にリポの役・deploy key・一覧の外の GitHub App・進行 Bot のアプリが入っていれば fail にするとする。[ADR-20261017-004][adr-human] の決定 2 は、GitHub の点検を進行 Bot が自分の認証で行い、人の `check` は進行 Bot のジョブを起こして結果を表示するとした。`setup` の GitHub の準備は、人の `gh` のログイン（組織の管理者）で行う
- Administration の権限の無い GitHub App のインストールのトークンでは、ruleset の `enforcement`・対象・規則・`current_user_can_bypass`・`updated_at` と、base ブランチに効く規則は読めるが、`bypass_actors` は返らない。ruleset の変更の履歴は `administration=write` を求める（FACT-88）。進行 Bot の App には Administration を付けない（[ADR-20261020-002][adr-app] の決定 2）
- bypass だけを変えても、App から見える `updated_at` は変わる（FACT-89）
- 2026-10-28 に、試し用の App を ruleset の bypass に入れると、App から見える `current_user_can_bypass` が `never` から `always` に変わり、外すと `never` に戻った（補足情報。承認の後の反映で FACT に載せる）
- 基本設計の「[導入と点検][design-setup]」は、読めない欄があれば fail にする。このままでは、GitHub App に切り替えた後の毎回の点検が fail になり、進行 Bot が当てる先のリポに書き始められない。PAT の間は、組織の管理者の fine-grained PAT で `bypass_actors` を読める（FACT-65）ので、困るのは GitHub App に切り替えた後だけ
- UNDECIDED-94 の候補 (A) は「人の `setup` と `check`（人の `gh` のログイン）」が控えると書いていたが、[ADR-20261017-004][adr-human] の決定 2 では `check` は進行 Bot のジョブで動く。人の `gh` のログインで bypass を読めるのは `setup` だけ

課題：GitHub App に切り替えた後、`bypass_actors` を読めない進行 Bot の毎回の点検で、ruleset の bypass が守りを弱める向きに変わっていないことを、どう確かめるか。

## 判断の決め手（Decision Drivers）

- 守りが崩れたら、止まる側に倒れる
- 進行 Bot に、ruleset を書き換えられる権限を渡さない
- GitHub App に切り替えた後も、進行 Bot が当てる先のリポに書き始められる
- 人の手間は、ruleset が変わったときだけにする

## 検討した選択肢（Considered Options）

- 案 A：`setup` が `bypass_actors` を確かめたときに、3 つの ruleset の ID と `updated_at` を控える。進行 Bot の毎回の点検は、App で読める欄に加えて、`updated_at` が控えと同じことと、`current_user_can_bypass` が `never` であることを確かめ、違えば fail にする
- 案 B：進行 Bot の GitHub App に Administration の権限を足し、`bypass_actors` を毎回読む
- 案 C：毎回の点検では bypass を確かめず、人の `setup` に任せる
- 案 D：今のまま、読めない欄を fail にする

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: bypass が変われば `updated_at` が変わる（FACT-89）ので、変わったときに止まる側に倒れる。進行 Bot のアプリ自身が bypass に入ったことは、`current_user_can_bypass` で直接分かる。進行 Bot に ruleset を書き換えられる権限を渡さずに済み、人の手間は ruleset が変わったときだけになる。

### 決定 1：setup が bypass を確かめたときに、3 つの ruleset の ID と updated_at を控える

- `setup`（人の `gh` のログイン。組織の管理者）は、GitHub の準備で 3 つの ruleset を作った後と、すでにあったときに、今のとおり `bypass_actors` を含めて読み戻して確かめる。守りが弱い向きの違いが無ければ、3 つの ruleset の ID と `updated_at` を控える。違いがあれば控えず、違いを示す（今のとおり）
- 控えを書くのは人の `setup` だけにする。進行 Bot と AI は、控えを書き換えない。`updated_at` が変わっていても、進行 Bot は控えを新しくしない（`bypass_actors` を読めない進行 Bot が、変わった ruleset を認めることになるため）
- 控えの置き場と形は、基本設計で決める（候補：マシンの設定のフォルダか記録の置き場、または管理者だけが書けるリポの変数）

### 決定 2：進行 Bot の毎回の点検は、updated_at と current_user_can_bypass で確かめる

- GitHub App に切り替えた後の毎回の点検は、控えた ID の 3 つの ruleset を読み、次のどれかに当たれば fail にする
  - 控えが無い
  - 控えた ID の ruleset が無い
  - `enforcement`・対象・規則が、基本設計の表より守りが弱い（今のとおり）
  - `updated_at` が控えと違う
  - `current_user_can_bypass` が `never` でない（進行 Bot のアプリが bypass に入っている）
- base ブランチに効く規則（`GET /repos/{owner}/{repo}/rules/branches/{baseBranch}`）も、今のとおり確かめる
- fail のときは、人の手元のマシンで `setup` を実行し直す（GitHub の準備）ように示す。`setup` が `bypass_actors` を確かめ直し、違いが無ければ控えを新しくする
- PAT の間は、今のとおり `bypass_actors` を読んで確かめる（FACT-65）

### 影響（Consequences）

- 良い点: bypass が変われば、次の実行の最初の点検で止まる
- 良い点: 進行 Bot のアプリに Administration を付けずに済む
- 良い点: 進行 Bot のアプリ自身が bypass に入ったことは、控えによらず `current_user_can_bypass` で分かる
- 悪い点: 守りを強める変更や、GitHub の側の都合で `updated_at` が変わったときも止まり、人が `setup` を実行し直す手間がかかる
- 悪い点: `setup` の GitHub の準備は人の `gh` のログインのある手元のマシンで行うので、専用マシンに移した後も、ruleset が変わるたびに人の手元のマシンで `setup` を実行する
- 中立: 実行の途中で bypass が変わったときは、今のとおり次の実行の最初の点検で気づく

### 確認方法（Confirmation）

- 点検の単体テストで、控えが無いとき、控えた ID の ruleset が無いとき、`updated_at` が控えと違うとき、`current_user_can_bypass` が `never` でないときに、fail になることを確かめる
- GitHub App に切り替えるときの試し（基本設計の「導入と点検」の「GitHub App の権限」の切り替えの順 2）で、本物のアプリと試しのリポで、`setup` が控えを書き、bypass を変えると次の点検が fail になることを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：setup が updated_at を控え、毎回の点検で比べる

- 良い点: 変わったときに止まる。進行 Bot に強い権限を渡さない
- 悪い点: bypass と関係の無い変更でも止まる

### 案 B：進行 Bot の App に Administration を足す

- 良い点: 毎回 `bypass_actors` を読める
- 悪い点: `bypass_actors` は ruleset に書き込める認証にだけ返り（[公式文書][doc-rules]）、変更の履歴も `administration=write` を求めた（FACT-88）。進行 Bot に ruleset を書き換えられる権限を渡すことになりうる。[ADR-20261020-002][adr-app] の決定 2 を改めることになる

### 案 C：毎回の点検では bypass を確かめない

- 良い点: 決まりが増えない
- 悪い点: bypass が変わっても、人が `setup` を実行するまで気づかない。守りが崩れたまま進行 Bot が書き続ける

### 案 D：今のまま読めない欄を fail にする

- 良い点: 決まりを変えない
- 悪い点: GitHub App に切り替えた後、毎回の点検が fail になり、進行 Bot が書き始められない

## 補足情報（More Information）

- 2026-10-28 に、試し専用の組織の公開の試し用リポ `probe-ruleset-read-20261028` で、試し用の App（Administration なし）を、deploy key の push 先を限るブランチの ruleset の bypass に `Integration` として入れた。App のトークンで読む `current_user_can_bypass` は、入れる前が `never`、入れた後が `always`、外した後が `never` だった。どの読み取りでも `bypass_actors` は返らず、`updated_at` は入れたときと外したときに変わった
- 本 ADR で決めないこと
  - 控えの置き場と形、`setup` が控えを書く手順、fail のときの人への案内の文（基本設計）
  - 組織の ruleset（当てる先のリポの ruleset でなく、組織に置く ruleset）の確かめ方。今の設計は当てる先のリポの 3 つの ruleset だけを確かめる
- 関連する要件：SETUP-1、SETUP-2（[06-setup-and-stages.md][setup]）、UNDECIDED-94（[09-undecided.md][undecided]）、FACT-65、FACT-88、FACT-89（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261017-004][adr-human]、[ADR-20261020-002][adr-app]
- 要件と設計への反映（承認の後に、別の PR で 1 本にまとめて行う）
  - 06-setup-and-stages.md：SETUP-2 の ruleset の点検の文
  - 09-undecided.md：UNDECIDED-94 を欠番にする（元の文を取り消し線で残す）
  - 10-verified-facts.md：補足情報の試しを FACT として足す
  - 基本設計の「[導入と点検][design-setup]」：「ruleset の形と確かめ方」の確かめ方、`setup` の GitHub の準備、点検の GitHub の組の ruleset の行、「GitHub App の権限」の切り替えの順 2

[adr-human]: ./adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[setup]: ../requirements/06-setup-and-stages.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-setup]: ../design/basic/04-setup-and-check.md
[doc-rules]: https://docs.github.com/en/rest/repos/rules
