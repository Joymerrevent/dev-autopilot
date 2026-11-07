---
status: proposed
created: 2026-11-07
updated: 2026-11-07
author: Claude Code（詳細設計を進める依頼の中で起票）
decision-makers: メンテナ
---

# 進行 Bot は、GitHub App の JWT で呼ぶ 2 つの API を、JWT を標準入力から渡した curl で呼ぶ

要件の UNDECIDED-95（進行 Bot が GitHub App の JWT で GitHub を呼ぶ手段。[決めていないこと][undecided]の「ADR で決めること」の「安全と設定」）を決める。
本 ADR は、[ADR-20261020-002][adr-app] の「進行 Bot は、実行のたびに秘密鍵で JWT を作り、インストールのトークンに引き換えて使う」を、どの手段で GitHub に送るかを決める。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261020-002][adr-app]（書き始める前に GitHub App に切り替える） | 据え置く（JWT を GitHub に送る手段を、本 ADR で決める） | — | すべて（JWT を Node の標準の機能で作ること、インストールのトークンに引き換えること、npm の依存を足さないこと） |

本 ADR の「JWT」は、進行 Bot が GitHub App の秘密鍵で署名して作る、アプリを証明する短い文字列を指す。JWT があれば、アプリが入ったリポのインストールのトークンを作れる。

## 背景と課題（Context and Problem Statement）

- 基本設計の「[導入と点検][design-setup]」の「GitHub App の権限」は、進行 Bot が実行のたびに JWT を作り、インストールのトークン（1 時間で切れる）に引き換えるとする（`POST /app/installations/{installation_id}/access_tokens`）。点検は、インストールの情報（`GET /app/installations/{installation_id}`）を JWT で読み、権限を確かめる
- 同じ「依存」は、GitHub の読み書きを子プロセスの `gh` で行い、呼ぶたびに `GH_TOKEN` でインストールのトークンを渡すとする（[SAFE-15][safety]）。基本設計の「[テスト][design-tests]」は、結合テストで外のコマンドを名前で差し替え、GitHub への呼び出しをすべて偽の `gh` が受けるとし、テストのためだけの切り替えを持たないとする
- GitHub は JWT を `Authorization: Bearer` の形でだけ受け、`gh` は `GH_TOKEN` を `token` の形で送る。`GH_TOKEN` に JWT を入れた `gh api` は 401 で断られた（FACT-107）。基本設計の書き方のままでは、JWT で呼ぶ 2 つの API を呼べない
- `gh api -H 'Authorization: Bearer <JWT>'` は、`GH_TOKEN` に何かの値があれば通るが、JWT が `gh` のプロセスの引数に載る（FACT-108）。同じユーザーのほかのプロセスの引数は、`ps` で読めた（FACT-110。AI のセッションの sandbox の中から読めるかは確かめていない）。環境変数は `ps` で読めなかった（FACT-44）
- `curl -H @-` は、見出しを標準入力から読む。JWT を引数にも環境変数にも置かずに、2 つの API を呼べた（FACT-109）。`-H @-` は curl 7.55.0 からある（curl の説明書）
- 進行 Bot は、`gh` を呼ぶ前にインストールのトークンの残りが 10 分を切っていれば作り直す。AI のセッションの間も、全体の止める合図を `gh` で読むので、作り直しは AI のコマンドが動いている間にも起こる
- AI のセッションの外への通信は、許可した先（初期値は npm のレジストリだけ）に限る（基本設計の「[設定][design-settings]」の `safety.network.allowedDomains`）

課題：JWT で呼ぶ 2 つの API（インストールのトークンへの引き換えと、点検のインストールの情報の読み取り）を、どの手段で GitHub に送るか。

## 判断の決め手（Decision Drivers）

- 安全：JWT は、秘密鍵と同じくインストールのトークンを作れる。進行 Bot の外のプロセスから読める所（プロセスの引数）に置かない（[SAFE-1、SAFE-15][safety]）
- 確かめやすさ：結合テストで、外のコマンドを名前で差し替える今の偽物の形のまま受けられ、テストのためだけの切り替えを持たない
- 動き続けること：道具の文書に書かれた使い方に頼り、版が上がっても振る舞いが変わりにくい
- 運用のしやすさ：道具を増やさないか、増やすなら OS に付いているものにし、点検で版を確かめられる

## 検討した選択肢（Considered Options）

- 案 A：`gh api -H 'Authorization: Bearer <JWT>'` で呼ぶ（`GH_TOKEN` に意味の無い値を入れる）
- 案 B：`curl -q -H @-` で呼び、JWT を含む見出しを標準入力から渡す
- 案 C：進行 Bot が Node の HTTPS（`fetch`）で直接呼ぶ

## 決定（Decision Outcome）

採用した選択肢:「案 B」。

理由: JWT が引数にも環境変数にも載らず、`curl` の標準入力だけに渡る。外のコマンドを名前で差し替える今の結合テストの形のまま、偽の `curl` で受けられる。頼るのは curl の説明書に書かれた使い方（`-H @-` と `-q`）だけ。

- JWT で呼ぶのは 2 つだけにする：インストールのトークンへの引き換え（要求の `permissions` と `repositories` で絞る）と、点検のインストールの情報の読み取り。ほかの GitHub の読み書きは、今のとおり、インストールのトークンを `GH_TOKEN` で渡した `gh` で行う
- 呼び方：最初の引数に `-q`（人の `~/.curlrc` を読まない）、`-sS`、`-H @-`（JWT を含む `Authorization: Bearer`・`Accept`・`X-GitHub-Api-Version` の見出しを標準入力から読む）を付ける。要求の JSON（秘密を含まない）は `--data-binary` で渡す。宛先は `https://api.github.com` の決まったパスに固定する
- 応答：標準出力で受ける。インストールのトークンは、今のとおり進行 Bot のプロセスの中でだけ持ち、`gh` を呼ぶときだけ `GH_TOKEN` で渡す（[SAFE-15][safety]）
- 点検：マシンの組で、`curl` があることと、版が 7.55.0 以降であることを確かめる。無いか古ければ fail にする
- 結合テスト：偽の `curl` を足す。偽の `gh` と同じ GitHub の状態から、引き換えとインストールの情報に答え、受けた引数と環境変数と標準入力を記録する

### 影響（Consequences）

- 良い点: JWT が、`curl` の標準入力のほかに出ない。プロセスの一覧と環境変数のどちらからも読めない（安全が上がる）
- 良い点: 結合テストは、外のコマンドを名前で差し替える今の形のまま、偽の `curl` で引き換えを受けられる（確かめやすさを保つ）
- 悪い点: 道具に `curl` が増える。点検で有無と版を確かめ、結合テストに偽物が 1 つ増える（運用のしやすさが下がる）
- 悪い点: GitHub を呼ぶ手段が `gh` と `curl` の 2 つになり、失敗の読み方（終了コード、HTTP の状態）を 2 通り持つ（変更のしやすさが下がる）
- 中立: macOS は OS に付いた `curl` を使う。Linux に広げるときは、`curl` の有無を確かめる（UNDECIDED-93 の時期）

### 確認方法（Confirmation）

- 単体テストで、`curl` の引数と標準入力を組み立てる処理が、JWT を標準入力の見出しにだけ入れ、引数と環境変数に入れないこと、最初の引数が `-q` であることを確かめる
- 結合テストで、偽の `curl` が記録した引数と環境変数に JWT が無いことと、引き換えたトークンで偽の `gh` を呼ぶことを確かめる
- 単体テストで、`curl` が無いときと 7.55.0 より古いときに、点検のマシンの組が fail になることを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：`gh api -H` で呼ぶ

- 良い点: 結合テストは、今の偽の `gh` だけで受けられ、偽物が増えない（確かめやすさが上がる）
- 中立: 道具が増えない（案 C も同じ）
- 悪い点: JWT が `gh` のプロセスの引数に載り、同じユーザーのほかのプロセスから `ps` で読める（FACT-110）。作り直しは AI のセッションの間にも起こるので、AI のコマンドが動いている間に載りうる。sandbox の中から読めるかは確かめていない。JWT の期限を短くすれば載る時間は縮むが、なくならない（安全が下がる）
- 悪い点: `GH_TOKEN` に意味の無い値を入れないと `gh` が送る前に止まり、渡した見出しが `gh` の認証の見出しより先に使われることに頼る（FACT-108）。どちらも `gh` の文書では確かめていない振る舞いで、版が上がると変わりうる（動き続けることが下がる）

### 案 B：`curl -q -H @-` で呼ぶ

- 良い点: JWT を標準入力から渡し、引数にも環境変数にも載せない（案 A より安全が上がる）
- 良い点: 偽物をコマンドの名前で差し替える今の結合テストの形のまま受けられる（案 C より確かめやすさが上がる）
- 良い点: 頼る使い方（`-H @-`、`-q`）は curl の説明書に書かれている（案 A より動き続けることが上がる）
- 悪い点: 道具に `curl` が増え、点検と偽物が増える（運用のしやすさが下がる）
- 悪い点: GitHub を呼ぶ手段が 2 つになる（案 C も同じ。変更のしやすさが下がる）

### 案 C：Node の HTTPS で直接呼ぶ

- 良い点: JWT が進行 Bot のプロセスの外に出ない（案 B より安全が上がる）
- 中立: 道具が増えない（案 A も同じ）
- 悪い点: 結合テストの偽の `gh` で受けられない。ネットワークの無い結合テストで引き換えを通すには、Node の外への通信の通り道とテスト用の証明書を差し替える仕組みが要る。Node 24 で使えるかは確かめていない。API の宛先を変える設定を持てば作りは小さくなるが、テストのためだけの切り替えになり、基本設計の「テスト」の決まりに反する（確かめやすさが下がる）
- 悪い点: GitHub を呼ぶ手段が `gh` と Node の 2 つになる（案 B も同じ。変更のしやすさが下がる）

## 補足情報（More Information）

- 2026-11-07 に、詳細設計の外との境（`docs/design/detailed/01-io.md`）を書き始めたところで、基本設計の書き方のままでは JWT を送れないことを見つけた。FACT-107〜110 を確かめ、UNDECIDED-95 に足した（#200）。推奨案を案 B にして `proposed` で起票した
- 本 ADR で決めないこと
  - JWT の期限の長さ、`curl` の子プロセスに渡す環境変数、時間の上限を過ぎたときの扱い（詳細設計の外との境）
  - `curl` の応答の読み方と、失敗の分け方（詳細設計の外との境）
- 関連する要件：SAFE-1、SAFE-15（[05-safety.md][safety]）、SETUP-2（[06-setup-and-stages.md][setup]）、UNDECIDED-93、UNDECIDED-95（[09-undecided.md][undecided]）、FACT-44、FACT-107〜110（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261017-003][adr-plugin]（npm の依存を持たない）、[ADR-20261020-002][adr-app]、[ADR-20261030-002][adr-no-pat]
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 05-safety.md：SAFE-15 に、JWT は `curl` の標準入力でだけ渡すことを足す
  - 06-setup-and-stages.md：SETUP-2 の点検の項目に、`curl` の有無と版を足す
  - 09-undecided.md：UNDECIDED-95 を欠番にする（元の文を取り消し線で残す）
  - 基本設計の「[導入と点検][design-setup]」の「依存」と「GitHub App の権限」と点検のマシンの組、「[テスト][design-tests]」の偽物の表（偽の `curl`）と確認方法のテスト
  - [ADR-20261020-002][adr-app]：JWT を送る手段を本 ADR で決めた旨の追記を足す

[adr-plugin]: ./adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-no-pat]: ./adr-20261030-002-bot-uses-github-app-only-from-start.md
[safety]: ../requirements/05-safety.md
[setup]: ../requirements/06-setup-and-stages.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-settings]: ../design/basic/01-settings.md
[design-setup]: ../design/basic/04-setup-and-check.md
[design-tests]: ../design/basic/05-tests.md
