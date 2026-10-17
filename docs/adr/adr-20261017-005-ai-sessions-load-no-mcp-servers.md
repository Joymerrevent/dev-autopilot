---
status: accepted
created: 2026-10-17
updated: 2026-10-17
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# AI のセッションは MCP のサーバーを読み込まず、claude.ai のコネクタを設定・環境変数・起動の引数と道具の deny で重ねて止める

要件の UNDECIDED-48（[決めていないこと][undecided]の「安全と設定」の「AI のセッションに claude.ai のコネクタを読み込ませないか」）を決める。

## 背景と課題（Context and Problem Statement）

- AI の 5 役のセッションは GitHub に書けず、道具はエージェント定義の `tools` と `permissions` で絞る（SAFE-1、SAFE-2 の (e)。「[登場するもの][overview]」）。AI の出力は、進行ボットが漏えいの検査を掛けてから公開の場に書く（SAFE-8、IMPL-15）。
  要件と安全設定は、MCP のサーバー（claude.ai のコネクタを含む）に触れていない
- `--setting-sources local` で起動したセッションにも、ログインしている claude.ai のアカウントのコネクタが入る回があった（FACT-30）。試験の親の道具はエージェント定義の `tools` で絞っていたので、コネクタの道具は道具の一覧に無かった
- 2026-10-17 に、エージェント定義に `tools` を書かないエージェントで試した（補足情報）。2 回とも、道具の一覧に claude.ai のコネクタの道具が 38 個入った。
  メールのコネクタの道具 30 個の中には、送信（`send_message`）・転送（`forward`）・返信（`reply`）・ゴミ箱への移動（`trash_message`）がある。今の守りは、エージェント定義に `tools` を書くことだけで、`tools` を書き漏らした定義や Claude Code の版の変化で、AI のセッションがメールを送る道具を持ちうる
- 誰でも Plugin を入れて使う形（[ADR-20261017-003][adr-plugin]）では、利用者のアカウントにどのコネクタがつながっているかを dev-autopilot は選べない
- 公式文書は、Claude Code が claude.ai から取ってくるコネクタを止める手段として、設定の `disableClaudeAiConnectors`、環境変数 `ENABLE_CLAUDEAI_MCP_SERVERS=false`、`--strict-mcp-config` を挙げている。`permissions.deny` に道具の名前の形（`mcp__*`）を書くと、当たる道具はモデルの文脈から取り除かれる

課題：AI のセッションが claude.ai のコネクタ（と、ほかの MCP のサーバー）の道具を持たないように、どの手段で止めるか。

## 判断の決め手（Decision Drivers）

- AI のセッションが、進行ボットを通さずに外へ書けないこと（SAFE-1、SAFE-8）
- 守りを 1 つの仕組みに頼らないこと。Claude Code の版で 1 つが効かなくなっても、ほかで止まること（フェイルセーフ）
- 止めたことを、`check` が実際に試して確かめられること（SAFE-5 と同じ考え方）
- 利用者のアカウントの設定（どのコネクタをつないでいるか）に頼らないこと
- 人のセッション（人が呼ぶ `setup`・`check`）のコネクタには影響しないこと

## 検討した選択肢（Considered Options）

- 案 A：起動の形で止める 3 つ（安全設定の `disableClaudeAiConnectors: true`、環境変数 `ENABLE_CLAUDEAI_MCP_SERVERS=false`、`--strict-mcp-config`）と、重ねる守りの `permissions.deny` の `mcp__*` をすべて使う。`check` が道具の一覧で確かめる
- 案 B：今のまま、エージェント定義の `tools` で道具を絞るだけにする
- 案 C：起動の形で止める仕組みを 1 つだけ足す（例：`--strict-mcp-config`）
- 案 D：進行ボットが使う `claude` のログインに、コネクタをつないでいないアカウントを使う（運用で守る）

## 決定（Decision Outcome）

採用した選択肢:「案 A」。
理由: 2026-10-17 の試験で 4 つとも効いた。重ねておけば、1 つが版の変化で効かなくなっても、ほかで止まる。利用者のアカウントの設定に頼らず、`check` が実際の道具の一覧で確かめられる。

### 決定 1：AI のセッションは MCP のサーバーを 1 つも使わない

- AI の 5 役（受付・計画・作業・レビュー・批評）のどれも、MCP の道具を使わない。claude.ai のコネクタも、ほかの MCP のサーバーも読み込ませない
- AI の役に MCP の道具を与えるのは、安全の境界を緩める変更として、新しい ADR で決める（[ADR-20261015-005][adr-settings] の決定 1）

### 決定 2：進行ボットは、起動の形でコネクタを止める仕組みを 3 つ重ねる

進行ボットが AI のセッションを起動するとき、次の 3 つをすべて使う。どれも 2026-10-17 の試験で、コネクタが読み込まれず、道具の一覧に MCP の道具が無くなった。

- 安全設定（`--settings` で渡すファイル）に `disableClaudeAiConnectors: true` を書く
- 子プロセスに渡す環境変数（許可した名前だけで組み立てる。SAFE-1、SAFE-10）に、`ENABLE_CLAUDEAI_MCP_SERVERS=false` を足す
- `--strict-mcp-config` を付け、`--mcp-config` は渡さない（MCP のサーバーを 1 つも読み込まない）

### 決定 3：重ねる守りとして、安全設定の `permissions.deny` に `mcp__*` を入れる

- MCP のサーバーがつながっても、MCP の道具がモデルの文脈に出ないようにする（試験では、コネクタがつながったまま、道具の一覧から MCP の道具が消えた）
- エージェント定義には、今のとおり `tools` を必ず書く（「[登場するもの][overview]」）

### 決定 4：check は、AI のセッションと同じ起動の形で、MCP の道具が無いことを確かめる

- `check`（SETUP-2）は、AI のセッションと同じ起動の形（安全設定・環境変数・起動の引数）で短いセッションを起こし、初期化の記録の道具の一覧に `mcp__` で始まる道具が無いこと、MCP のサーバーの一覧が空であることを確かめる。外れたら fail にする
- 確かめる回数（`setup` と人が呼ぶ `check` だけか、毎回の実行の最初か）は、利用枠との兼ね合いで基本設計で決める

### 影響（Consequences）

- 良い点: AI のセッションは、エージェント定義の `tools` を書き漏らしても、claude.ai のコネクタの道具（メールの送信など）を持たない。4 つのどれか 1 つが効けば止まる
- 良い点: 利用者のアカウントにどのコネクタがつながっていても、同じに止まる。人のセッションのコネクタには影響しない
- 良い点: `check` が実際の道具の一覧で確かめるので、Claude Code の版で止め方が効かなくなったら気づける
- 悪い点: AI のセッションは、どの MCP のサーバーも使えない。要るようになったら、ADR で決める
- 悪い点: 止める仕組みを 4 か所で持つ。Claude Code が設定や環境変数の名前を変えると、直す箇所が増える（`check` が気づく）
- 悪い点: `check` の確かめのために、短い AI のセッションを起こす利用枠が要る

### 確認方法（Confirmation）

- 構造仕様を書くときに、安全設定の雛形（`disableClaudeAiConnectors`、`deny` の `mcp__*`）、子プロセスに渡す環境変数の一覧、起動の引数が、決定 2・3 のとおりかをレビューで確かめる
- 進行ボットの単体テストで、AI のセッションの起動の引数に `--strict-mcp-config` があり `--mcp-config` が無いこと、環境変数に `ENABLE_CLAUDEAI_MCP_SERVERS=false` があることを検査する
- 段階 3（`setup` と `check` を作る段階）で、`check` の決定 4 の確かめを、コネクタをつないだアカウントで試す

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：起動の形で止める 3 つと、道具の deny を重ねる

- 良い点: 2026-10-17 の試験で、4 つとも効いた。1 つが効かなくなっても、ほかで止まる
- 悪い点: 4 か所で持ち、`check` の確かめに利用枠が要る

### 案 B：エージェント定義の `tools` で絞るだけ

- 良い点: 今のままで、追加の設定が要らない
- 悪い点: `tools` を書き漏らした定義が 1 つあれば、コネクタの道具（メールの送信など）が入る（2026-10-17 の試験で、38 個入った）。Claude Code の版の変化にも弱い

### 案 C：起動の形で止める仕組みを 1 つだけ足す

- 良い点: 持つ箇所が少ない
- 悪い点: 足した 1 つが版の変化で効かなくなると、`tools` の守りだけに戻る

### 案 D：コネクタをつないでいないアカウントを使う

- 良い点: Claude Code の設定に頼らない
- 悪い点: 誰でも Plugin を入れて使う形（ADR-20261017-003）では、利用者のアカウントを dev-autopilot が選べない。人がアカウントにコネクタをつなぐと、黙って守りが外れる

## 補足情報（More Information）

- 関連する要件：SAFE-1、SAFE-2、SAFE-5、SAFE-8、SAFE-10（[05-safety.md][safety]）、SETUP-2（[06-setup-and-stages.md][stages]）、STRUCT-2（[07-structure.md][struct]）、FACT-30（[10-verified-facts.md][facts]）、「登場するもの」（[00-overview.md][overview]）、
  [ADR-20261015-005][adr-settings]、[ADR-20261017-003][adr-plugin]
- 2026-10-17 に、claude.ai のコネクタが AI のセッションの道具に入るかと、止め方が効くかを実機で確かめた（macOS、Claude Code 2.1.285、`claude-haiku-5-5`。試験の場所は `/tmp/claude-<uid>/` の外）。
  `tools` を書かないエージェント定義の試験用の Plugin を、`claude -p --setting-sources local --settings <安全設定（sandbox を有効）> --plugin-dir <Plugin> --agent <試験用> --output-format stream-json --verbose` で、許可した名前だけの環境変数で起動し、初期化の記録の道具の一覧と MCP のサーバーの一覧を読んだ。道具は呼ばせていない
  - 止める仕組みなし（2 回）：MCP のサーバーは claude.ai の文書とメールが `connected`、ほかの 2 つが `needs-auth`。道具 63 個のうち 38 個が MCP の道具で、メールの道具 30 個に `send_message`・`forward`・`reply`・`trash_message` などがあった
  - 安全設定に `disableClaudeAiConnectors: true`（2 回）、環境変数 `ENABLE_CLAUDEAI_MCP_SERVERS=false`（2 回）、`--strict-mcp-config`（2 回）：どれも MCP のサーバーの一覧が空で、MCP の道具は 0 個（道具 25 個）
  - 安全設定の `permissions.deny` に `mcp__*`（1 回）：MCP のサーバーは `connected` のままで、MCP の道具は 0 個（道具 25 個）
  - MCP の道具を、`-p` で許可の規則に無いまま呼ぶとどうなるか（確認待ちで拒否されるか）は、メールを送る道具を呼ぶことになるので試していない
- 公式文書（[MCP][doc-mcp]、[権限][doc-permissions]）：`disableClaudeAiConnectors` は、どの設定の出どころでも `true` が勝つ。`ENABLE_CLAUDEAI_MCP_SERVERS=false` は同じ働き。`--strict-mcp-config` は `--mcp-config` で渡したサーバーだけを使う。
  `permissions.deny` の道具の名前の位置のグロブ（`mcp__*`）は、当たる道具をモデルの文脈から取り除く
- 要件への反映（承認の後に、別の PR で行う）
  - 05-safety.md：SAFE-1 か SAFE-2 に、AI のセッションは MCP のサーバーを読み込まないこと（決定 1〜3）を書く。SAFE-10 の環境変数に `ENABLE_CLAUDEAI_MCP_SERVERS=false` を足す
  - 00-overview.md：「登場するもの」の 5 役に共通の説明に、MCP のサーバーを読み込まないことを書く
  - 06-setup-and-stages.md：SETUP-2 に、決定 4 の確かめを足す
  - 07-structure.md：STRUCT-2 の「子プロセスに渡す環境変数の名前の一覧」に、`ENABLE_CLAUDEAI_MCP_SERVERS` を足す
  - 10-verified-facts.md：上の試験の事実を FACT として足す
  - 09-undecided.md：UNDECIDED-48 を欠番にする（元の文を取り消し線で残す）

[overview]: ../requirements/00-overview.md
[safety]: ../requirements/05-safety.md
[stages]: ../requirements/06-setup-and-stages.md
[struct]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[adr-settings]: ./adr-20261015-005-settings-values-and-machine-settings.md
[adr-plugin]: ./adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[doc-mcp]: https://code.claude.com/docs/en/mcp
[doc-permissions]: https://code.claude.com/docs/en/permissions
