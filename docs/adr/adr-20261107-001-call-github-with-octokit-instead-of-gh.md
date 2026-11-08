---
status: accepted
created: 2026-11-07
updated: 2026-11-08
author: Claude Code（詳細設計を進める依頼の中で起票）
decision-makers: メンテナ
---

# 進行 Bot と setup は、GitHub を Octokit で呼び、gh を使わない

要件の UNDECIDED-95（進行 Bot が GitHub App の JWT で GitHub を呼ぶ手段）と UNDECIDED-96（進行 Bot が GitHub を呼ぶ道具を `gh` のままにするか、Octokit に替えるか）を、1 つの決定として決める。どちらも「`plugin/src/` のプログラムが GitHub の API を何で呼ぶか」で、UNDECIDED-96 の答えで UNDECIDED-95 の問いが残るかが決まるため（[決めていないこと][undecided]の「ADR で決めること」の「安全と設定」）。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261015-001][adr-read]（読める場所の許可と秘密のファイル） | 一部を改める | 決定 2 の「`gh` を呼ぶときだけ `GH_TOKEN` を渡す」。GitHub のトークンを子プロセスに渡さず、進行 Bot のプロセスの中でだけ使う | 決定 2 のほか（秘密の置き場、起動したときに読んでプロセスの中だけで持つこと、deploy key を `GIT_SSH_COMMAND` で渡すこと、AI のセッションに渡さないこと）と、ほかの決定すべて |
| [ADR-20261020-002][adr-app]（書き始める前に GitHub App に切り替える） | 据え置く | — | すべて。JWT を Node の標準の機能（`node:crypto`）で作り、インストールのトークンに引き換えることも変えない |
| [ADR-20261107-002][adr-deps]（実行時に npm の依存を持ってよい） | 据え置く（どの依存を持つかを、本 ADR で 1 つ決める） | — | すべて |

本 ADR の「JWT」は、進行 Bot が GitHub App の秘密鍵で署名して作る、アプリを証明する短い文字列を指す。JWT があれば、アプリが入ったリポのインストールのトークンを作れる。

> **［2026-11-08 書き直し］** 2026-11-07 に、UNDECIDED-95 だけを決める ADR（推奨は JWT を `curl` の標準入力から渡す案）として起票した。メンテナが Octokit を使う案を挙げ、[ADR-20261107-002][adr-deps] で実行時の npm の依存を持ってよいと決まったので、ADR の規則（同じ論点の `proposed` の ADR には追記する）に従い、UNDECIDED-96 も含めた ADR に書き直した。前の推奨は、下の「案 B」に残した。

## 背景と課題（Context and Problem Statement）

- 基本設計の「[導入と点検][design-setup]」の「依存」は、GitHub の読み書きを子プロセスの `gh` で行い、呼ぶたびに `GH_TOKEN` でインストールのトークンを渡すとする（[SAFE-15][safety]）。「GitHub App の権限」は、進行 Bot が実行のたびに JWT を作ってインストールのトークンに引き換え、点検はインストールの情報を JWT で読むとする
- GitHub は JWT を `Authorization: Bearer` の形でだけ受け、`gh` は `GH_TOKEN` を `token` の形で送るので、`gh` では JWT で呼ぶ 2 つの API を呼べない（FACT-107）。`gh api -H` で見出しを上書きすれば通るが、JWT が `gh` のプロセスの引数に載り、同じユーザーのほかのプロセスから `ps` で読める（FACT-108・110）。`curl -H @-` なら、JWT を標準入力だけで渡せる（FACT-109）
- 基本設計の「[テスト][design-tests]」は、結合テストで外のコマンドを名前で差し替え、GitHub への呼び出しをすべて偽の `gh` が受けるとし、進行 Bot のプログラムにテストのためだけの切り替えを持たないとする
- 詳細設計の「[外との境][d-io]」は、`gh` の出力（`gh api -i` の状態の行）の読み方、`Link` をたどるページ送り、人の `gh` の設定を読ませない空の `GH_CONFIG_DIR`、`gh` の版の確かめを決めた。`setup` の device flow と App の作成の引き換え、GitHub の外への通知は、すでにシステムの `fetch` で HTTPS を呼ぶ。結合テストの通知の偽の送り先は、テスト用の CA の証明書と `NODE_EXTRA_CA_CERTS` で立てる（FACT-114）
- [ADR-20261107-002][adr-deps] で、進行 Bot は実行時に npm の依存を持ってよくなった。依存は Claude Code が Plugin を入れるときに入れ、`setup` の写しのハッシュの一覧で確かめる。どの依存を持つかは決めていない
- Octokit（GitHub の公式の JavaScript の SDK）の `@octokit/core` は、`request.fetch` で渡した fetch だけで通信し（FACT-121）、`node:crypto` で作った JWT を `auth` に渡すと `bearer` の形で送り、引き換えの本文で権限とリポを絞れる（FACT-122）。試し用 App で、本物の GitHub にも通った（FACT-124）。App の認証の部品（`@octokit/auth-app`）は、Octokit に組み込む形では引き換えで権限とリポを絞らない（FACT-122）
- 子プロセスの環境変数（`NODE_USE_ENV_PROXY=1`・`HTTPS_PROXY`・`NODE_EXTRA_CA_CERTS`）だけで、宛先の `https://api.github.com` を変えずに、Node の fetch と Octokit を偽の送り先に向けられる。`NODE_USE_ENV_PROXY` は Node の文書で v24.0.0 から入り、安定度は「1.1 - Active Development」（FACT-123）
- 進行 Bot のコードはまだ無い。`gh` を使う決まりは、要件と基本設計と詳細設計の文書の中だけにある

課題：`plugin/src/` のプログラム（進行 Bot と `setup`）が GitHub の API（REST と GraphQL。JWT で呼ぶ 2 つの API を含む）を、何で呼ぶか。

## 判断の決め手（Decision Drivers）

- 安全：JWT とトークンを、進行 Bot のプロセスの外（子プロセスの引数と環境変数）に出さない（[SAFE-1、SAFE-15][safety]）。トークンは表の権限と当てる先のリポだけに絞る。秘密を持つプロセスで動く外のコードを少なくする
- 確かめやすさ：単体テストで外との境を偽物にでき、結合テストで GitHub を偽物に差し替えられる。進行 Bot のプログラムに、テストのためだけの切り替えを持たない
- 変更のしやすさ：GitHub を呼ぶ手段の数と、自分で書いて保つ処理（出力の読み方、ページ送り、認証）が少ない
- 動き続けること：頼る振る舞いが、道具の文書に書かれている
- 運用のしやすさ：人のマシンに入れる道具と、版の確かめと更新の手間が少ない

## 検討した選択肢（Considered Options）

- 案 A：GitHub の読み書きは `gh` のまま。JWT で呼ぶ 2 つも `gh api -H 'Authorization: Bearer <JWT>'` で呼ぶ（`GH_TOKEN` に意味の無い値を入れる）
- 案 B：GitHub の読み書きは `gh` のまま。JWT で呼ぶ 2 つだけ、`curl -q -H @-` で JWT を含む見出しを標準入力から渡す
- 案 C：GitHub の読み書きは `gh` のまま。JWT で呼ぶ 2 つだけ、進行 Bot が Node の `fetch` で呼ぶ
- 案 D：GitHub の読み書きは `gh` のまま。JWT で呼ぶ 2 つだけ、Octokit の App の認証の部品（`@octokit/auth-app`）で呼ぶ
- 案 E：GitHub の読み書きを、JWT で呼ぶ 2 つも含めてすべて Octokit の `@octokit/core` で呼び、`gh` を使わない。JWT は今のとおり `node:crypto` で作る

## 決定（Decision Outcome）

採用した選択肢:「案 E」。

理由: JWT もインストールのトークンも進行 Bot のプロセスの外に出ず、GitHub を呼ぶ手段が 1 つになる。足す依存は `@octokit/core` だけで、JWT の作り方（[ADR-20261020-002][adr-app]）を変えずに、権限とリポを絞った引き換えを呼べる。結合テストは、進行 Bot のプログラムを変えずに、子プロセスの環境変数で GitHub を偽物に差し替えられる。案 A〜D は `gh` を残すので、`gh` の出力の読み方と版の確かめを持ち続け、案 B〜D はさらに GitHub を呼ぶ手段が 2 つになる。

- `plugin/src/` のプログラム（進行 Bot と `setup`）は、GitHub の API（REST と GraphQL）を `@octokit/core` で呼ぶ。`gh` を使わず、人のマシンに `gh` を求めない。push と fetch は今のとおり `git` で行う
- `@octokit/core` には、入口が作るシステムの `fetch` を `request.fetch` で渡す（FACT-121）。`@octokit/core` を import するのは外との境の層だけにする（詳細設計の README の「構成の検査に足すこと」の依存ごとの層）
- JWT は今のとおり `node:crypto` で作り、`@octokit/core` の `auth` に渡して、インストールのトークンへの引き換えと、点検のインストールの情報の読み取りを呼ぶ（`bearer` の形。FACT-122）。引き換えの本文に、基本設計の表の権限と当てる先のリポを入れる。App の認証の部品（`@octokit/auth-app`）は入れない
- JWT とインストールのトークン（`setup` では `setup` 用の App の人の代わりのトークン）は、進行 Bot のプロセスの中でだけ持ち、子プロセスの引数と環境変数に入れない
- 結合テストは、偽の `gh` の代わりに、偽の GitHub（CONNECT を受ける proxy と、テスト用の CA の証明書で `api.github.com` として答える HTTPS の送り先）を立て、進行 Bot の入口を `NODE_USE_ENV_PROXY=1`・`HTTPS_PROXY`・`NODE_EXTRA_CA_CERTS` を足した環境で起こす（FACT-123）。宛先と証明書の確かめ方は本番と同じで、進行 Bot のプログラムにテストのためだけの切り替えを持たない
- 定期の実行の登録（plist と systemd の unit）には、proxy の環境変数・`NODE_USE_ENV_PROXY`・`NODE_EXTRA_CA_CERTS` を書かない。点検は、進行 Bot の環境に proxy の環境変数・`NODE_USE_ENV_PROXY`・`NODE_EXTRA_CA_CERTS` が無いことを確かめ、あれば fail にする（GitHub への通信の行き先を、環境変数で変えられないようにする）
- 足す実行時の依存は `@octokit/core` だけにする（下の依存まで 12 個。試した版では、lifecycle script と `bin` を持つものは無い。FACT-121 の節）。ページ送りなどの部品（plugin）を足すかは、足す PR で [ADR-20261107-002][adr-deps] の決まりに従って決める

### 影響（Consequences）

- 良い点: JWT とトークンが、子プロセスの引数にも環境変数にも出ない。案 A〜D は、インストールのトークンを `GH_TOKEN` で `gh` の子プロセスに渡す（環境変数は `ps` では読めない。FACT-44）（安全が上がる）
- 良い点: GitHub を呼ぶ手段が `@octokit/core` の 1 つになり、`setup` の device flow と通知と同じシステムの `fetch` の上に載る。`gh` の出力の読み方、空の `GH_CONFIG_DIR`、`gh` の版の確かめを持たない（変更のしやすさが上がる）
- 良い点: 人のマシンに `gh` を入れて版を合わせなくてよい（運用のしやすさが上がる）
- 良い点: 偽の GitHub は、`setup` が HTTPS で呼ぶ GitHub の API の結合テストにも使える。今は単体テストだけで確かめている（「[外との境][d-io]」の「HTTPS」）（確かめやすさが上がる）
- 悪い点: 秘密を持つ進行 Bot のプロセスで、npm の依存 12 個のコードが動く（安全が下がる。依存の入れ方と確かめ方は [ADR-20261107-002][adr-deps]）
- 悪い点: 結合テストの差し替えは、Node の `NODE_USE_ENV_PROXY`（安定度 1.1）に頼る。振る舞いが変われば結合テストが通らなくなる。本番は proxy の環境変数を使わないので、本番の動きは変わらない（確かめやすさと動き続けることが下がる）
- 悪い点: 偽の `gh`（コマンドの名前で差し替え、GitHub の状態から答える）を、偽の GitHub（HTTPS の送り先）に作り替える。偽物の作りは、`gh` の引数でなく HTTP の要求で答えを選ぶ形になる（確かめやすさのための作業が増える）
- 中立: 要件と設計の `gh` の前提を直す。直す先は下の「補足情報」の「要件と設計への反映」。進行 Bot のコードはまだ無いので、直すのは文書だけ

### 確認方法（Confirmation）

- 単体テストで、外との境の GitHub のモジュールが `@octokit/core` にシステムの `fetch` を渡し、グローバルの `fetch` を使わないことを確かめる（グローバルの `fetch` を、呼ばれたら投げるものに替える）
- 単体テストで、引き換えの要求に表の権限と当てる先のリポが入ること、JWT を `bearer` の形で送ること、子プロセスの組み立てに JWT とトークンと `GH_TOKEN` が無いことを確かめる
- 結合テストで、偽の GitHub が `api.github.com` の名前で要求を受け、引き換えたトークンで続く呼び出しを受けることを確かめる
- 構成の検査で、`@octokit/core` を外との境の層だけが import していることを確かめる
- 単体テストで、進行 Bot の環境に proxy の環境変数・`NODE_USE_ENV_PROXY`・`NODE_EXTRA_CA_CERTS` があるときに、点検が fail になることを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：`gh api -H` で JWT を送る

- 良い点: 結合テストは、今の偽の `gh` だけで受けられる。偽物も道具も依存も増えない（確かめやすさと運用のしやすさが上がる）
- 悪い点: JWT が `gh` のプロセスの引数に載り、同じユーザーのほかのプロセスから `ps` で読める（FACT-108・110）。トークンの作り直しは AI のセッションの間にも起こるので、AI のコマンドが動いている間に載りうる（安全が下がる）
- 悪い点: `GH_TOKEN` に意味の無い値を入れないと `gh` が送る前に止まり、渡した見出しが `gh` の認証の見出しより先に使われることに頼る（FACT-108）。どちらも `gh` の文書で確かめていない振る舞い（動き続けることが下がる）

### 案 B：`curl -q -H @-` で JWT を送る

- 良い点: JWT を標準入力から渡し、引数にも環境変数にも載せない（案 A より安全が上がる）
- 良い点: 偽物をコマンドの名前で差し替える今の結合テストの形のまま、偽の `curl` で受けられる。頼る使い方（`-H @-`、`-q`）は curl の説明書に書かれている（案 C・D より確かめやすさが上がり、案 A より動き続けることが上がる）
- 悪い点: 道具に `curl` が増え、点検で有無と版を確かめ、結合テストに偽物が 1 つ増える（運用のしやすさが下がる）
- 悪い点: GitHub を呼ぶ手段が `gh` と `curl` の 2 つになり、失敗の読み方を 2 通り持つ（変更のしやすさが下がる）

### 案 C：JWT で呼ぶ 2 つだけ Node の `fetch` で呼ぶ

- 良い点: JWT が進行 Bot のプロセスの外に出ない。道具も依存も増えない（案 B より安全と運用のしやすさが上がる）
- 悪い点: 結合テストに、偽の `gh` と偽の GitHub（環境変数で向ける HTTPS の送り先。FACT-123）の 2 つの差し替えが要る（確かめやすさが下がる）
- 悪い点: GitHub を呼ぶ手段が `gh` と `fetch` の 2 つになり、失敗の読み方を 2 通り持つ（変更のしやすさが下がる）

### 案 D：JWT で呼ぶ 2 つだけ Octokit の App の認証の部品で呼ぶ

- 良い点: JWT を作り、引き換え、トークンを持ち回る処理を部品に任せられる（案 C より変更のしやすさが上がる）
- 悪い点: 部品を Octokit に組み込む形では、引き換えで権限とリポを絞らない（FACT-122）。絞るには部品を直接呼ぶので、持ち回りの処理は使えず、任せられる処理が JWT を作ることだけになる（変更のしやすさが下がる）
- 悪い点: 2 か所のために依存 16 個（`@octokit/auth-app` の下の依存まで。FACT-120）を足し、JWT を Node の標準の機能で作る [ADR-20261020-002][adr-app] の決定も改める（安全が下がる）
- 悪い点: 結合テストに 2 つの差し替えが要り、GitHub を呼ぶ手段が 2 つになる（案 C と同じ。確かめやすさと変更のしやすさが下がる）

### 案 E：すべて `@octokit/core` で呼び、`gh` を使わない

- 良い点: JWT もトークンも子プロセスに渡さない（案 A〜D より安全が上がる）
- 良い点: GitHub を呼ぶ手段が 1 つになり、`gh` の出力の読み方・`GH_CONFIG_DIR`・版の確かめを持たない。人のマシンに `gh` が要らない（変更のしやすさと運用のしやすさが上がる）
- 良い点: 結合テストの差し替えが偽の GitHub の 1 つになり、`setup` の GitHub の呼び出しにも使える（案 C・D より確かめやすさが上がる）
- 悪い点: 秘密を持つプロセスで依存 12 個が動く（案 A〜C より安全が下がる）
- 悪い点: 結合テストの差し替えが、安定度 1.1 の `NODE_USE_ENV_PROXY` に頼る（案 A・B より動き続けることが下がる）
- 悪い点: 偽の `gh` を偽の GitHub に作り替え、書いた詳細設計の `gh` の節を書き直す（確かめやすさのための作業が増える）

## 補足情報（More Information）

- 2026-11-07 に、詳細設計の外との境を書き始めたところで、`gh` では JWT を送れないと分かり（FACT-107〜110、UNDECIDED-95。#200）、UNDECIDED-95 だけを決める ADR として起票した（推奨は案 B）。同じ日に、メンテナの提案で案 D を足した
- 2026-11-07 に、メンテナが「npm 依存があってもいいと思います」と言い、[ADR-20261107-002][adr-deps] で実行時の npm の依存を持ってよいと決まった。どの依存を持つか（`gh` か Octokit か）は UNDECIDED-96 に足した（#205）
- 2026-11-08 に、メンテナが「#201 を作り直して、gh か Octokit かも含めて 1 本の ADR で決める」進め方を了承した。FACT-121〜124 を確かめ（試しの記録は本体の `.tmp/probe-octokit-20261108/`）、推奨を案 E にして書き直した。同じ日に、メンテナが推奨案（案 E）で決定した
- 決定が触れるもの（`plugin/src/` の GitHub の呼び方、人のマシンに求める道具、定期の実行の環境変数）を使うのは、dev-autopilot のリポの中の文書だけ（進行 Bot のコードはまだ無い）。人が自分の作業で使う `gh` と、AI のセッションに `gh` の認証を渡さない決まり（[SAFE-1][safety]）は変わらない
- 本 ADR で決めないこと
  - ページ送りの部品（`@octokit/plugin-paginate-rest` など）を足すか、`Link` を自分でたどるか（詳細設計の外との境。足すなら [ADR-20261107-002][adr-deps] の決まりで）
  - GitHub の呼び出しの時間の上限と、上限を過ぎたときの扱い（今の子プロセスの `run.commandMinutes` と、リポの設定を読む前の固定の 10 分に合わせるか。基本設計の「[設定][design-settings]」の「子プロセスが時間の上限を過ぎたとき」）
  - JWT の期限の長さ、失敗の分け方と記録に書く欄（詳細設計の外との境）
- 関連する要件：SAFE-1、SAFE-15（[05-safety.md][safety]）、SETUP-2（[06-setup-and-stages.md][setup]）、STRUCT-4（[07-structure.md][struct]）、UNDECIDED-95・96（[09-undecided.md][undecided]）、FACT-44、FACT-107〜110、FACT-114、FACT-121〜124（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261015-001][adr-read]、[ADR-20261020-002][adr-app]、[ADR-20261030-001][adr-setup-app]（`setup` 用の App）、[ADR-20261107-002][adr-deps]
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 05-safety.md：SAFE-15 の `gh` と `GH_TOKEN` の文を、トークンを進行 Bot のプロセスの中でだけ使う形に直す
  - 06-setup-and-stages.md：SETUP-2 の点検の項目から `gh` の版を除き、進行 Bot の環境に proxy の環境変数・`NODE_USE_ENV_PROXY`・`NODE_EXTRA_CA_CERTS` が無いことを足す
  - 07-structure.md：構成の木の結合テストの「偽の `gh`」を、偽の GitHub に直す
  - 09-undecided.md：UNDECIDED-95・96 を欠番にする（元の文を取り消し線で残す）
  - 基本設計の「[設定][design-settings]」（`gh-config/`、子プロセスの時間の上限の表）、「[状態遷移と判定][design-flow]」と「[AI のセッション][design-ai]」の `gh` の文、「[導入と点検][design-setup]」（依存、plist の `PATH`、点検のマシンの組、`known_hosts` を書くときの `meta` の読み方、GitHub App の権限）、「[テスト][design-tests]」（偽の `gh` を偽の GitHub に）
  - 詳細設計の README（モジュールの一覧、依存ごとの層の一覧に `@octokit/core`）と「[外との境][d-io]」（`gh` の呼び方の節を Octokit の呼び方に、JWT で呼ぶ 2 つの API の節）
  - [ADR-20261015-001][adr-read]：決定 2 を本 ADR で改めた旨の注記を足す。[ADR-20261107-002][adr-deps] と「[外との境][d-io]」の `[adr-jwt]` の参照定義を、本 ADR の新しいファイル名に直す（2026-11-08 にファイル名を変えたため）

[adr-read]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-setup-app]: ./adr-20261030-001-setup-uses-setup-app-instead-of-human-gh.md
[adr-deps]: ./adr-20261107-002-allow-runtime-npm-deps-via-plugin-install.md
[safety]: ../requirements/05-safety.md
[setup]: ../requirements/06-setup-and-stages.md
[struct]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-settings]: ../design/basic/01-settings.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
[design-ai]: ../design/basic/03-ai-sessions.md
[design-setup]: ../design/basic/04-setup-and-check.md
[design-tests]: ../design/basic/05-tests.md
[d-io]: ../design/detailed/01-io.md
