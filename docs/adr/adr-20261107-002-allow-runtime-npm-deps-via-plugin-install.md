---
status: accepted
created: 2026-11-07
updated: 2026-11-08
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 進行 Bot は実行時に npm の依存を持ってよく、Claude Code の Plugin の依存の導入で入れる

進行 Bot（`plugin/src/`）が実行時に npm の依存を持つかと、持つならどう入れるかを決める。
[ADR-20261017-003][adr-plugin] の決定 1 は「持たないか、持つなら Claude Code の依存の導入で入る形にする。どちらにするかは基本設計で決める」とし、基本設計は「持たない」を選んだ（基本設計の「[導入と点検][design-setup]」の「依存」。2026-10-19 の PR #49）。本 ADR は、基本設計の選択を ADR で決め直す。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261017-003][adr-plugin]（進行 Bot を Plugin に同封し、写しから動かす） | 据え置く（決定 1 が基本設計に残した選択を、本 ADR で決める） | — | すべて（`plugin/src/` に置くこと、Node 24 が直接実行してビルドしないこと、写しから動かすこと） |
| [ADR-20261020-002][adr-app]（書き始める前に GitHub App に切り替える） | 据え置く | — | すべて。JWT を Node の標準の機能で作ることも変えない（Octokit などの部品で作るかは、下の「本 ADR で決めないこと」） |

## 背景と課題（Context and Problem Statement）

- 基本設計が「実行時に npm の依存を持たない」を選んだ理由は、写しに依存を入れる手順も、npm の登録所への通信も、lifecycle script（パッケージを入れるときに走るスクリプト）も要らないことだった（PR #49）。ADR で案を比べてはいない
- 依存を持たないので、GitHub の認証（JWT とインストールのトークン）、ページ送り、回数制限への対応、JSON の形の確かめを、`gh` と自分のコードで受け持っている。詳細設計の外との境を書く中で、`gh` では JWT を送れないと分かり（FACT-107）、UNDECIDED-95 と [ADR-20261107-001][adr-jwt]（`proposed`）で送り方を比べている。比べる中で、メンテナが GitHub の公式の SDK（Octokit）を使う案を挙げた
- Claude Code は、Plugin の root に `package.json` と npm か Bun のロックファイルがあるとき、Plugin を入れるか版を上げるときに、写した版のフォルダに依存を入れる。登録所の版を固定した依存だけで、lifecycle script は動かず、60 秒で打ち切られる（[ADR-20261017-003][adr-plugin] の補足情報。公式文書）。手元で試すと、npm のロックファイルの版では写しに `node_modules` が入り、pnpm のロックファイルだけの版では入らなかった（下の「補足情報」）
- `setup` は、入れた Plugin（`${CLAUDE_PLUGIN_ROOT}`）を置き場の `program/<版>/` に写し、中身のハッシュの一覧（manifest）を書く。毎回の点検は、写しの中身が一覧のとおりかを確かめる（「[導入と点検][design-setup]」の「プログラムの写し」）
- 進行 Bot のプロセスは、GitHub App の秘密鍵・deploy key・印の署名の鍵・Webhook の URL を持って動く（[SAFE-15][safety]）。依存のパッケージのコードも、同じプロセスで同じ権限で動く
- 開発の道具の依存は、リポの root の `package.json` と pnpm に置き、Dependabot が熟成期間（7 日）を置いて上げる。pnpm には、公開から 7 日たたない版を入れない守り（`minimumReleaseAge`）がある

課題：進行 Bot が実行時に npm の依存を持つか。持つなら、依存をどう写しに入れるか。

## 判断の決め手（Decision Drivers）

- 安全：秘密を持つ進行 Bot のプロセスで動くコードの出どころを絞り、取り込み経由で攻撃される入口を小さくする
- 変更のしやすさ：自分で書いて保つコード（GitHub の認証・ページ送り・回数制限・型）を減らす
- 動き続けること：配り方と写し方が、確かめた振る舞いと文書に書かれた振る舞いに頼る
- 運用のしやすさ：導入の手間、版の更新、パッケージマネージャーの数

## 検討した選択肢（Considered Options）

- 案 A：持たない（今の基本設計のまま。Node の標準の機能と `plugin/src/` の中のファイルだけ）
- 案 B：持つ。Claude Code の依存の導入で入れる（`plugin/` に `package.json` と npm のロックファイル）
- 案 C：持つ。ビルドで依存を 1 つのファイルにまとめて `plugin/` に入れる
- 案 D：持つ。`node_modules` を `plugin/` に入れて commit する

## 決定（Decision Outcome）

採用した選択肢:「案 B」。

理由: [ADR-20261017-003][adr-plugin] の決定 1 が認めた入れ方のまま、ビルドをせずに、部品で書けるコードを部品に任せられる。依存は Claude Code が登録所の固定の版で入れ、lifecycle script は動かず、入った依存は `setup` の写しのハッシュの一覧と毎回の点検で確かめられる。手元の試しで、写しに依存が入ることを確かめた。

- 進行 Bot は、実行時に npm の依存を持ってよい。依存は `plugin/package.json` に版を固定して書き（範囲の指定をしない）、npm のロックファイル（`plugin/package-lock.json`）を commit する
- 依存は、Claude Code が Plugin を入れるときと版を上げるときに入れる。`setup` は、今のとおり入れた Plugin を写し、写しのハッシュの一覧に `node_modules` も含める。毎回の点検は、今のとおり写しの中身を一覧と照らす（依存のファイルが書き換えられても止まる）
- 依存を足すときは、足す PR に、足すパッケージと、下の依存まで含めた数と、lifecycle script の有無を書く。下の依存の少ない部品を選ぶ（例：全部入りの `octokit` でなく、要る部品だけ）
- Dependabot に `plugin/` の npm を足し、熟成期間をリポの root と同じにする
- 開発の道具の依存は、今のとおりリポの root の `package.json` と pnpm に置く
- どの依存を持つか（Octokit を使うか、`gh` をやめるか）は、本 ADR で決めない（下の「補足情報」）

### 影響（Consequences）

- 良い点: GitHub の認証やページ送りのように、部品に任せられる処理を自分で書かずに済む（変更のしやすさが上がる）
- 良い点: ビルドをせず、Node 24 が `plugin/src/` を直接動かす今の形を保つ（案 C より変更のしやすさが上がる）
- 良い点: 入った依存も写しのハッシュの一覧に入り、書き換えられれば毎回の点検で止まる（安全）
- 悪い点: 秘密を持つ進行 Bot のプロセスで、dev-autopilot のリポの外で書かれたコードが動く。依存が乗っ取られたときの害は、ふつうのアプリより大きい（安全が下がる）
- 悪い点: 人のマシンで Plugin を入れるときに、npm の登録所から依存を取る。pnpm の `minimumReleaseAge` は `plugin/` の npm には効かず、熟成期間は Dependabot の側だけで守る（安全が下がる）
- 悪い点: パッケージマネージャーが、リポの root の pnpm と `plugin/` の npm の 2 つになる（運用のしやすさが下がる）
- 悪い点: Claude Code の依存の導入の決まり（60 秒で打ち切る、npm か Bun のロックファイル）に頼る（動き続けることが下がる）

### 確認方法（Confirmation）

- 構成の検査で、`plugin/package.json` の依存の版がすべて固定（範囲の記号が無い）であることと、`plugin/package-lock.json` があることを確かめる
- 構成の検査で、`plugin/src/` の import が、`node:` で始まるもの・相対パス・`plugin/package.json` の依存のどれかであることを確かめる（今の「`node:` か相対パスだけ」を改める）
- 点検で、写しの中身（`node_modules` を含む）が、`setup` の書いた一覧とハッシュのとおりであることを確かめる（今の確かめのまま）

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：持たない

- 良い点: 進行 Bot のプロセスで動くコードは、dev-autopilot のリポで書いてレビューしたものだけ（安全が上がる）
- 良い点: パッケージマネージャーが、リポの root の pnpm の 1 つのまま（案 B より運用のしやすさが上がる）
- 中立: 人のマシンで依存を取らない（案 C・D も同じ）
- 悪い点: GitHub の認証・ページ送り・回数制限・JSON の形の確かめを、`gh` と自分のコードで受け持ち、保つ。`gh` で足りないところ（JWT）は、別の道具や手当てで埋める（変更のしやすさが下がる）

### 案 B：Claude Code の依存の導入で入れる

- 良い点: ビルドをせず、[ADR-20261017-003][adr-plugin] の決定 1 が認めた入れ方のまま、部品を使える（案 C より変更のしやすさが上がる）
- 良い点: 依存を入れるときに lifecycle script が動かない（Claude Code の決まり。安全）
- 中立: 写しのハッシュの一覧で、入った依存の書き換えに気づける（案 C・D も同じ）
- 悪い点: 人のマシンで Plugin を入れるときに、登録所から依存を取る（案 C より安全が下がる）
- 悪い点: `plugin/` に npm のロックファイルを置き、パッケージマネージャーが 2 つになる（運用のしやすさが下がる）
- 悪い点: Claude Code の依存の導入の決まりに頼る（動き続けることが下がる）

### 案 C：ビルドで 1 つのファイルにまとめる

- 良い点: 依存は CI のビルドのときだけ取り、人のマシンでは登録所に通信しない（案 B より安全が上がる）
- 悪い点: ビルドが要り、[ADR-20261017-003][adr-plugin] の決定 1 の「ビルドしない」と開発ガイドの決まり 3 を改める。ビルドの成果物を commit すると差分を人が読めず、配る前に作るなら、配るものとリポの中身を照らす手順が要る（変更のしやすさと安全が下がる）
- 悪い点: 型を取り除くだけで動く書き方（FACT-103〜106）のほかに、ビルドの道具の決まりが加わる（変更のしやすさが下がる）

### 案 D：`node_modules` を commit する

- 良い点: ビルドも、人のマシンでの登録所への通信も要らない（案 B より安全が上がる）
- 悪い点: 依存のファイルがリポに入り、版を上げる PR の差分を人が読めない（安全と運用のしやすさが下がる）
- 悪い点: どのパッケージマネージャーの置き方で commit するかを別に決め、Dependabot の上げ方と合わせる手間が要る（運用のしやすさが下がる）

## 補足情報（More Information）

- 2026-11-07 に、メンテナが「npm 依存があってもいいと思います」と言い、CC が理由を調べた。基本設計の選択の理由は PR #49 の本文にだけあり、ADR で比べていなかった。メンテナの依頼で、推奨案を案 B にして `proposed` で起票した。同じ日に、メンテナが推奨案で決定した
- 2026-11-07 に、手元で確かめた（macOS、Claude Code 2.1.285。`CLAUDE_CONFIG_DIR` を一時の場所に切り替え、人の `~/.claude` に触れずに試した。記録は本体の `.tmp/probe-plugin-deps-20261107/`）。承認されたら、[実機で確かめた事実][facts]に載せる
  - 手元の marketplace に置いた試し用の Plugin（`package.json` と `package-lock.json`）を `claude plugin install` で入れると、写した版のフォルダ（`plugins/cache/<marketplace>/<plugin>/<版>/`）に `node_modules` が入った（`toad-cache`）。`@octokit/core` と `@octokit/auth-app` に替えて `claude plugin update` で版を上げると、新しい版のフォルダの `node_modules/@octokit/` に 14 個が入った
  - `pnpm-lock.yaml` だけの版では、`node_modules` は作られなかった
  - 依存の数（下の依存まで含める）：全部入りの `octokit` 5.0.5 は 63 個、`@octokit/auth-app` 8.3.1 は 16 個、`@octokit/core` 7.0.6 は 12 個。`octokit` の 63 個に、lifecycle script を持つものは無く、ライセンスは MIT・Apache-2.0・ISC だけ
- 本 ADR で決めないこと
  - どの依存を持つか。GitHub を `gh` でなく Octokit で呼ぶか（`gh` をやめるか）は、要件の SAFE-15 と基本設計の「依存」が前提にしている `gh` を改める判断なので、本 ADR の承認の後に別の ADR で比べる。[ADR-20261107-001][adr-jwt]（JWT を送る手段）は、別の ADR の結果で見直す
  - Dependabot の `plugin/` の行の細かい形（反映の PR で決める）
- 関連する要件：SAFE-15（[05-safety.md][safety]）、STRUCT-3・STRUCT-4（[07-structure.md][struct]）、UNDECIDED-95（[09-undecided.md][undecided]）、FACT-103〜107（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261017-003][adr-plugin]、[ADR-20261020-002][adr-app]、[ADR-20261107-001][adr-jwt]
- 要件と設計への反映（承認の後に、別の PR で行う）
  - 07-structure.md：構成の図と木に `plugin/package.json` と `plugin/package-lock.json` を足す。STRUCT-4 の import の決まり
  - 10-verified-facts.md：上の試しを FACT に載せる
  - 基本設計の「[導入と点検][design-setup]」の「依存」と「プログラムの写し」（写しのハッシュの一覧に `node_modules` を含める）、「[テスト][design-tests]」の構成の検査
  - 詳細設計の [README][detailed] の「構成の検査に足すこと」と「TypeScript の書き方」（依存の import）
  - `.github/dependabot.yml` に `plugin/` の npm を足す（反映の PR）

[adr-plugin]: ./adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-jwt]: ./adr-20261107-001-call-github-with-octokit-instead-of-gh.md
[safety]: ../requirements/05-safety.md
[struct]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-setup]: ../design/basic/04-setup-and-check.md
[design-tests]: ../design/basic/05-tests.md
[detailed]: ../design/detailed/README.md
