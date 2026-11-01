---
status: proposed
created: 2026-11-01
updated: 2026-11-01
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 攻撃の疑いと全体の停止は、人が選んだ Slack 互換の Webhook に決まった文を送って GitHub の外に知らせる

要件の UNDECIDED-91（攻撃の疑いと全体の停止を、GitHub の外に知らせる仕組みと鍵。[決めていないこと][undecided]の「ADR で決めること」の「安全と設定」）を決める。
本 ADR は、[ADR-20261024-005][adr-security] の決定 4 の「GitHub の外への通知：送る仕組みと鍵は、後で決める」と、[ADR-20261027-002][adr-leak] で同じく後で決めるとしたことを決める。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261024-005][adr-security]（安全の軸と全体の停止の Issue） | 据え置く（決定 4 で後で決めるとした、GitHub の外への通知の仕組みと鍵を、本 ADR で決める） | — | 決定 4 の、攻撃の疑いと全体の停止を GitHub の外にも知らせること。ほかの決定 |
| [ADR-20261027-002][adr-leak]（漏えいの検査で見つかったら安全の軸に入れる） | 据え置く（補足情報で後で決めるとした通知を、本 ADR で決める） | — | すべて |

本 ADR の「Webhook」は、進行 Bot が送り先に文を送る口（送り先のサービスが発行する URL）を指す。GitHub App が GitHub から受け取る Webhook（[ADR-20261020-002][adr-app] で使わないと決めた）とは別のもの。

## 背景と課題（Context and Problem Statement）

- SAFE-19（安全の軸と全体の停止）と [ADR-20261024-005][adr-security] の決定 4 は、攻撃の疑いと全体の停止を、表示のラベル `security-suspected`・`security` の印のコメント・Project の安全の欄に加えて、GitHub の外にも知らせるとした。漏えいの検査で見つかったときも同じ（[ADR-20261027-002][adr-leak]）。基本設計には、まだ GitHub の外への通知が無い
- 攻撃の疑いと全体の停止の正本は、GitHub の印（`security`・`halt`）と `## 安全の確認` にある。AI のセッションを起こさない働きは、通知が届かなくても変わらない。GitHub の外への通知は、人に早く気づかせるためのもの
- 進行 Bot は、人の Mac（いずれは専用マシン。UNDECIDED-71）の定期の実行（launchd）で動き、人がその場にいるとは限らない。Linux に広げるかは後で決める（UNDECIDED-93）
- 秘密は、進行 Bot だけが読む秘密の置き場（`paths.secrets`。フォルダ 700・ファイル 600）のファイルに置く。AI のセッションには渡さない（[ADR-20261015-001][adr-read] の決定 2、基本設計の「[設定][design-settings]」の「秘密の置き場のファイル」）
- 進行 Bot は、lock で見送った実行などを手元の記録に残し、点検の warn に出している（基本設計の「[導入と点検][design-setup]」）。手元の記録は、作業の単位の状態の判定には使わない（[ADR-20261024-002][adr-no-stages] の決定 5）
- 当てる先の Issue の題名と本文、見つかった文、AI が書いた文は、外から入る文か、操られたかもしれない文。`security` の印のコメントにも写さない（[ADR-20261024-005][adr-security] の決定 4）

課題：攻撃の疑いと全体の停止を、どの仕組みで GitHub の外に知らせ、送るための鍵をどこに置き、送れなかったときにどう扱い、何を書くか。

## 判断の決め手（Decision Drivers）

- 気づきやすさ：人が動かすマシンの前にいなくても、人の手元（スマホなど）に届く
- 安全：送るための鍵が漏れても、できることが狭い。外から入る文と、操られたかもしれない文を、送る文に入れない
- 確かめやすさ：届いたかを進行 Bot が確かめられ、作りが小さく、テストしやすい
- 動かす場所によらないこと：専用マシンや Linux に移しても、同じ仕組みで送れる
- 運用のしやすさ：導入のときに人が用意するものを増やしすぎない

## 検討した選択肢（Considered Options）

- 案 A：人が選んだ Slack 互換の Webhook の URL に、決まった文を送る
- 案 B：動かす Mac の通知センターに出す（`osascript`）
- 案 C：メール（SMTP）で送る
- 案 D：GitHub の通知（Watch とメール）に任せ、進行 Bot からは GitHub の外へ送らない

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: 送るための鍵（Webhook の URL）でできるのは、決まった 1 つの宛先に書くことだけで、漏れても読めるものが無い。人のスマホのアプリに届き、動かす場所によらない。進行 Bot は HTTP の POST を 1 つ持てばよく、届いたかを応答の状態で確かめられる。

- 送る仕組み：進行 Bot は、JSON の `text` の欄に文を入れて POST すると、その文を投稿する Webhook（Slack の Incoming Webhook と同じ形。以下「Slack 互換の Webhook」）に送る。送り先のサービスは人が選ぶ（Slack、Google Chat、Mattermost、Discord の Slack 互換の URL など）。URL は `https://` で始まるものだけを使う
- 送るとき：`security` の印を書いたとき（攻撃の疑いを見つけたとき。入口・AI の文・コード・漏えいのどれでも）と、`halt` の印を書いたとき（全体を止めたとき）。印を GitHub に書いた後に送る。印を書けなかったときは送らない（正本の無い知らせを出さないため）
- 鍵の置き場：Webhook の URL を、秘密の置き場の新しいファイル（例：`notify-webhook-url`）に人が書く。`setup` は空のファイルを作る（権限 600）。進行 Bot だけが読み、設定ファイル・記録・ログに書かず、AI のセッションに渡さない。動かすマシンを替えるときは、前のマシンからファイルを写すか、送り先で新しい URL を作って書く
- 点検：URL のファイルが空か、`https://` で始まる 1 行でなければ、点検を fail にする（送り先が無いまま動かさない）
- 送れなかったとき：攻撃の疑いと全体の停止の扱いは変えない（正本は GitHub の印）。同じ実行の中で、決まった回数だけ送り直す（回数と間隔は基本設計）。送り直しても送れなければ、送れなかったこと（種類・Issue の番号・時刻・応答の状態。URL は書かない）を手元の記録に残し、点検の warn に出す。次の実行には持ち越さない（手元の記録で送るかを決めないため）
- 送る文：進行 Bot が組み立てる決まった文だけにする。書くのは、リポ（`owner/repo`）、種類（攻撃の疑い（入口・AI の文・コード・漏えいのどれか）か全体の停止か。`security` の印の `where` と `finding` の決まった値）、Issue の番号と URL（PR が契機なら PR の番号と URL も）、人がすること（Issue を開いて確かめる）。Issue の題名と本文、見つかった文、AI が書いた文、機械上のパスは書かない

### 影響（Consequences）

- 良い点: 人が動かすマシンの前にいなくても、攻撃の疑いと全体の停止にすぐ気づける（気づきやすさ）
- 良い点: 鍵が漏れても、送り先の 1 つの宛先に書けるだけで、GitHub にも、メールの受信箱にも届かない（安全）
- 良い点: 送る文が決まった文だけなので、見つかった文や操られた文が、通知を通って広まらない（安全）
- 良い点: 届いたかを応答の状態で確かめ、送れなかったことを点検の warn に出せる（確かめやすさ）
- 悪い点: 人が送り先のサービスを用意し、Webhook の URL を作って書く手間が増える。書くまでは、点検が fail になり、進行 Bot が動かない
- 悪い点: 送る文（リポの名前と Issue の番号）が、送り先のサービスに残る。非公開のリポに広げるときは、リポの名前を外のサービスに出してよいかを確かめる
- 悪い点: 送り直しても送れなかった知らせは、次の実行で送り直さない。人が気づくのは、点検の warn か GitHub の表示になる

### 確認方法（Confirmation）

- 単体テストで、送る文に決まった項目だけが入り、Issue の題名と本文・見つかった文・AI が書いた文・機械上のパスが入らないことを確かめる（見本は、題名に指示の形の文を入れた Issue）
- 単体テストで、URL のファイルが空のとき、`https://` で始まらないとき、2 行以上のときに、点検が fail になることを確かめる
- 結合テスト（偽の Webhook の送り先を手元で立てる）で、`security` と `halt` の印を書いた後に 1 回ずつ送ることと、送り先が失敗を返すと決まった回数だけ送り直し、それでも送れなければ記録に残して点検の warn に出し、攻撃の疑いと全体の停止の扱いが変わらないことを確かめる
- 漏えいの検査の見本で、記録とログに URL が出ないことを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：Slack 互換の Webhook に送る

- 良い点: 鍵でできるのは 1 つの宛先に書くことだけで、漏れても読めるものが無い（案 C より安全が上がる）
- 良い点: HTTP の POST を 1 つ持てばよく、SMTP より作りが小さい。届いたかを応答の状態で確かめられる（確かめやすさが上がる）
- 中立: 人のスマホに届く（案 C も同じ）。macOS でも Linux でも同じ仕組みで送れる（案 C・D も同じ）
- 悪い点: 人が送り先のサービスを用意し、URL を作って書く（運用のしやすさが下がる）
- 悪い点: 送る文（リポの名前と Issue の番号）が、外のサービスに残る（案 B より安全が下がる）

### 案 B：Mac の通知センターに出す

- 良い点: 送る文が外のサービスに出ない（安全が上がる）。鍵も送り先も要らない（運用のしやすさが上がる）
- 悪い点: 動かす Mac の前に人がいないと気づけない。専用マシンに移すと、まず見ない（気づきやすさが下がる）
- 悪い点: 定期の実行（launchd のジョブ）から通知センターに出せるかと、出たかを進行 Bot が確かめられるかを、確かめていない（確かめやすさが下がる）
- 悪い点: Linux では同じ仕組みが無い（動かす場所によらないことが下がる）

### 案 C：メールで送る

- 良い点: 送り先のサービスを用意しなくても、人のメールに届く（運用のしやすさが上がる）
- 悪い点: 送るための鍵（メールのアカウントのパスワードなど）は、送るほかに、受信箱を読める権限を持つことが多い（安全が下がる）
- 悪い点: SMTP の送り方と、届いたかの確かめが、Webhook より大きい（確かめやすさが下がる）

### 案 D：GitHub の通知に任せる

- 良い点: 鍵も、進行 Bot の新しい作りも要らない（運用のしやすさが上がる）
- 悪い点: 届くかは、人の GitHub の通知の設定（Watch とメール）しだいで、進行 Bot から確かめられない（確かめやすさと気づきやすさが下がる）
- 悪い点: [ADR-20261024-005][adr-security] の決定 4（GitHub の外にも知らせる）を改めることになる

## 補足情報（More Information）

- 2026-11-01 に、CC が案 A〜D をメンテナに示し、メンテナの依頼で、推奨案を案 A にして `proposed` で起票した
- 本 ADR で決めないこと
  - 秘密の置き場のファイルの名前（基本設計。例として `notify-webhook-url`）
  - 送り直しの回数と間隔、送る文の言い回し（基本設計）
  - 送り先のサービス（人が選ぶ）
- 確かめること（確かめたら[実機で確かめた事実][facts]に載せる。段階 3 の実装のとき）：使う送り先のサービスが、`text` の欄の JSON の POST を受け、成功と失敗を HTTP の状態で返すか
- 関連する要件：SAFE-15、SAFE-19（[05-safety.md][safety]）、SETUP-1、SETUP-2（[06-setup-and-stages.md][setup]）、UNDECIDED-71、UNDECIDED-91、UNDECIDED-93（[09-undecided.md][undecided]）
- 関連する ADR：[ADR-20261015-001][adr-read]、[ADR-20261020-002][adr-app]、[ADR-20261024-002][adr-no-stages]、[ADR-20261024-005][adr-security]、[ADR-20261027-002][adr-leak]
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 05-safety.md：SAFE-19 の GitHub の外への通知、SAFE-15 の秘密に Webhook の URL を足す
  - 06-setup-and-stages.md：SETUP-1（人がやるもの：送り先の Webhook を作って URL を書く）と SETUP-2（点検の項目）
  - 09-undecided.md：UNDECIDED-91 を欠番にする（元の文を取り消し線で残す）
  - 基本設計の「[設定][design-settings]」の「秘密の置き場のファイル」、「[状態遷移と判定][design-flow]」の `security` と `halt` の印を書くところ、「[導入と点検][design-setup]」の点検の項目と `setup` が作るもの、「[テスト][design-tests]」の確認方法のテスト
  - [ADR-20261024-005][adr-security] の決定 4 と、[ADR-20261027-002][adr-leak] の補足情報：本 ADR で決めた旨の追記を足す

[adr-read]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-no-stages]: ./adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[adr-security]: ./adr-20261024-005-security-axis-and-halt-issue.md
[adr-leak]: ./adr-20261027-002-leak-findings-go-to-security-axis.md
[safety]: ../requirements/05-safety.md
[setup]: ../requirements/06-setup-and-stages.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-settings]: ../design/basic/01-settings.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
[design-setup]: ../design/basic/04-setup-and-check.md
[design-tests]: ../design/basic/05-tests.md
