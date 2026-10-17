---
status: accepted
created: 2026-10-17
updated: 2026-10-17
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# AI のセッションの一時フォルダは、セッションごとの短いパスを CLAUDE_CODE_TMPDIR で渡し、共有の一時フォルダとほかのセッションの一時フォルダを読めなくする

要件の UNDECIDED-50（[決めていないこと][undecided]の「安全と設定」の「sandbox の中の一時フォルダが、セッションの間で共有される」）を決める。

改める決定は次の 1 つで、ほかの決定は据え置く。

- [ADR-20261015-001][adr-read] の決定 1 のうち、`TMPDIR` をセッションの一時フォルダに向けるところを、セッションごとの短いパスの一時フォルダを `CLAUDE_CODE_TMPDIR` と `TMPDIR` で渡す形に改める（決定 1）。
  あわせて、AI が読める範囲から、Claude Code のユーザーごとの一時フォルダとほかのセッションの一時フォルダを外す（決定 2）。XDG の 4 つをセッションの一時フォルダに向けること、ホームと秘密を読めないことは据え置く

## 背景と課題（Context and Problem Statement）

- SAFE-10 と [ADR-20261015-001][adr-read] の決定 1 は、AI のセッションの読み取りを自分のセッションのフォルダに絞り、`TMPDIR` と XDG の 4 つをセッションの一時フォルダに向けるとする
- 一方、Claude Code は sandbox の中のコマンドの `$TMPDIR` を、ユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）に置き換え、そこは sandbox の中のコマンドが既定で書ける（FACT-34）。
  同じ OS ユーザーで動くすべての AI のセッションが、同じ一時フォルダを使う。手元の Mac で試す間（SAFE-18）は、人の Claude Code のセッションも同じ一時フォルダを使う
- 2026-10-17 に、今の要件の安全設定（`denyRead` は `~/` と置き場）で試した（補足情報）。sandbox の中のコマンドは、ユーザーごとの一時フォルダに書けたうえ、中のファイルを読めて一覧も出せた。
  ユーザーごとの一時フォルダには、人の Claude Code のセッションの作業ファイルとコマンドの出力のファイルもある。`denyRead` の外に置いたほかのセッションの一時フォルダも読めた
- 公式文書は、Claude Code の一時フォルダの場所を `CLAUDE_CODE_TMPDIR` で変えられるとする。Claude Code は後ろに `/claude-<uid>/` を足し、パスが長いと sandbox の中のコマンドには既定の場所の下の短い `$TMPDIR` を渡す。
  `CLAUDE_CODE_TMPDIR` はシェル（環境変数）・ユーザー設定・managed の設定で効き、project と local の設定では無視される
- 同じ日の試験で、`CLAUDE_CODE_TMPDIR` に `/claude-501` を足した長さが 44 文字までなら、sandbox の中の `$TMPDIR` は渡した場所になり、ユーザーごとの一時フォルダには書けなかった。
  45 文字からは、`$TMPDIR` はユーザーごとの一時フォルダに戻り、渡した場所とユーザーごとの一時フォルダの両方に書けた。置き場の中のセッションのフォルダ（試験では 62 文字）は、長いパスとして扱われた

課題：AI のセッションごとに、ほかのセッションと人の Claude Code と共有しない一時フォルダを、どう持たせるか。

## 判断の決め手（Decision Drivers）

- AI のセッションの間で、ファイルを受け渡す経路を作らないこと（objects と依存の store と同じ種類の論点。[ADR-20261013-001][adr-workspace]、[ADR-20261015-002][adr-store]）
- AI のセッションが、人の Claude Code のセッションの作業ファイルを読めないこと（SAFE-10 の「読み取りは自分のセッションのフォルダだけ」）
- Claude Code の振る舞い（長さの境目、置き換えの規則）が版で変わっても、黙って共有の一時フォルダに戻らず、止まるか気づけること（フェイルセーフ）
- 一時フォルダを使う道具（`mktemp`、Node の `os.tmpdir()`、pnpm など）が、そのまま動くこと
- `setup` に OS ユーザーの作成のような重い準備を足さないこと

## 検討した選択肢（Considered Options）

- 案 A：セッションごとの短いパスの一時フォルダを `CLAUDE_CODE_TMPDIR` で渡し、一時フォルダの根とユーザーごとの一時フォルダを `denyRead` にする。長すぎるパスでは起動しない
- 案 B：置き場の中のセッションのフォルダの下を、`CLAUDE_CODE_TMPDIR` で渡す
- 案 C：ユーザーごとの一時フォルダのまま、安全設定の `denyWrite` か `denyRead` で止める
- 案 D：ユーザーごとの一時フォルダを共有するまま受け入れる（今の要件の形）
- 案 E：AI のセッションを、人と別の OS ユーザーで動かす

## 決定（Decision Outcome）

採用した選択肢:「案 A」。
理由: 2026-10-17 の試験で、セッションごとの短いパスを渡すと、sandbox の中の `$TMPDIR` と Node の `os.tmpdir()` が渡した場所になり、ユーザーごとの一時フォルダとほかのセッションの一時フォルダは読めず書けなかった。
`setup` に重い準備を足さずに済み、長さの境目が版で変わっても、起動の前の確かめと `check` で止まる。

### 決定 1：進行ボットは、AI のセッションごとに短いパスの一時フォルダを作って渡す

- 進行ボットは、AI のセッションを起動する前に、セッションごとの一時フォルダを、dev-autopilot 専用の短い根の下に作る（例：`/tmp/dap/<セッションの短い ID>`）。根と ID の長さの値は、構造仕様の設定の表で決める（マシンの設定。[ADR-20261015-005][adr-settings]）
- 根は、ホームの外、置き場の外、Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）の外に置く
- 進行ボットは、セッションごとの一時フォルダのパスを、子プロセスの環境変数の `CLAUDE_CODE_TMPDIR` と `TMPDIR` で渡す（許可した名前だけで組み立てる。SAFE-1）。Claude Code は後ろに `/claude-<uid>/` を足し、sandbox の中のコマンドの `$TMPDIR` にする
- `/tmp` はマシンのすべての OS ユーザーが書ける。進行ボットは、根が無ければ権限 700 で作り、根が自分の持ち物で、シンボリックリンクでなく、権限が 700 であることを毎回確かめる。外れたら AI のセッションを起動しない（SAFE-4）
- セッションごとの一時フォルダは、毎回新しく作る（すでにあれば起動しない）。消すときは、AI の作業場所と一緒に、git を使わずフォルダごと消す（SAFE-12）
- XDG の 4 つと pnpm の store は、今のとおり置き場の中のセッションの一時フォルダに置く（[ADR-20261015-002][adr-store]）。パスの長さの制約は `TMPDIR` だけにかかる

### 決定 2：安全設定で、一時フォルダの根とユーザーごとの一時フォルダを読めなくする

- 安全設定の sandbox の `denyRead` に、一時フォルダの根と、Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）を足す。`allowRead` に、自分のセッションの一時フォルダを足す（狭い方の規則が勝つ。FACT-22）
- パスは `setup` が実体に解決して書く（SAFE-5。macOS の `/tmp` は `/private/tmp` へのリンク）
- 書き込みの許可は足さない。Claude Code が、渡した場所の下の `claude-<uid>/` を書けるようにする
- 重ねる守りとして効く：長いパスで置き換えが起きても、ユーザーごとの一時フォルダの中は読めず一覧も出せない（試験で確かめた。書き込みは止まらない）

### 決定 3：長すぎるパスでは、AI のセッションを起動しない

- 進行ボットは、起動の前に `<CLAUDE_CODE_TMPDIR>/claude-<uid>` の長さを数え、設定の上限（初期値は 44。2026-10-17 の試験の値）を超えたら起動しない（SAFE-4）。Claude Code の置き換えに任せて、共有の一時フォルダで動かさない
- 上限は Claude Code の振る舞いで、公式文書に値が無く、版で変わりうる。値が合っているかは `check` が試して確かめる（決定 4）

### 決定 4：check は、sandbox の中の一時フォルダを試して確かめる

- `check`（SETUP-2）は、AI のセッションと同じ起動の形（安全設定・環境変数・起動の引数）で短いセッションを起こし、sandbox の中で次を確かめる。外れたら fail にする
  - `$TMPDIR` が、渡した場所の下の `claude-<uid>` であること。書けること
  - ユーザーごとの一時フォルダが、読めず、一覧も出せず、書けないこと
  - ほかのセッションの一時フォルダ（`check` が根の下に作った試験用のフォルダ）が、読めず書けないこと
- 確かめる回数は、[ADR-20261017-005][adr-mcp] の決定 4（MCP の道具が無いことの確かめ）と同じ短いセッションでまとめて行い、基本設計で決める

### 影響（Consequences）

- 良い点: AI のセッションは、ほかのセッションと人の Claude Code の一時フォルダを読めず書けない。`$TMPDIR` を使う道具はそのまま動く
- 良い点: 長さの境目が版で変わっても、起動の前の確かめで止まるか、`check` で気づける。置き換えが起きても、ユーザーごとの一時フォルダの中は読めない
- 悪い点: セッションごとに、置き場の外にもう 1 つフォルダを作って消す。消し損ねた残骸が `/tmp` の下に残りうる（何を残骸とみなすかは UNDECIDED-27）
- 悪い点: 公式文書に値の無い長さの上限に頼る。上限を設定に持ち、`check` で確かめる手間が要る
- 悪い点: パスの長さの制約で、セッションの ID を短くする（`claude-<uid>` の長さも上限に入る）
- 悪い点: Linux で同じに振る舞うかは確かめていない（UNDECIDED-49 で確かめ直す）

### 確認方法（Confirmation）

- 構造仕様を書くときに、設定の表に一時フォルダの根と長さの上限があること、安全設定の雛形の `denyRead` と `allowRead` が決定 2 のとおりかを、レビューで確かめる
- 進行ボットの単体テストで、起動の環境変数に `CLAUDE_CODE_TMPDIR` と `TMPDIR` があり長さが上限以内であること、上限を超えると起動しないこと、根の持ち物・リンク・権限が外れると起動しないことを検査する
- 段階 3（`setup` と `check` を作る段階）で、決定 4 の確かめを実機で試す

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：短いパスを渡し、根とユーザーごとの一時フォルダを読めなくする

- 良い点: 2026-10-17 の試験で、`$TMPDIR` と `os.tmpdir()` が渡した場所になり、ユーザーごとの一時フォルダとほかのセッションの一時フォルダは読めず書けなかった。Bash は普通に動いた
- 良い点: `setup` の準備は根を作るだけで、OS ユーザーを足さない
- 悪い点: 公式文書に値の無い長さの上限に頼る（起動の前の確かめと `check` で補う）。置き場の外にセッションのフォルダが 1 つ増える

### 案 B：置き場の中のセッションのフォルダの下を渡す

- 良い点: セッションのものが置き場の中にまとまり、消すのも一緒にできる
- 悪い点: 置き場のパスは長く、試験では 62 文字で置き換えが起き、`$TMPDIR` は共有の一時フォルダに戻った。渡した場所とユーザーごとの一時フォルダの両方に書けた

### 案 C：ユーザーごとの一時フォルダのまま、denyWrite か denyRead で止める

- 良い点: パスの長さに頼らない
- 悪い点: `denyWrite` を掛けると、Claude Code が Bash のたびに `$TMPDIR` に書く作業ファイルが書けず、すべてのコマンドが終了コード 1 になった（試験で確かめた）。
  `denyRead` だけでは、ほかのセッションと人の Claude Code へ向けた書き込みが止まらない。`$TMPDIR` の中が読めなくなるので、一時ファイルを読み戻す道具も動かない（試験では、書いたファイルを消すこともできなかった）

### 案 D：共有するまま受け入れる

- 良い点: 何も足さずに済む
- 悪い点: AI のセッションが、ほかのセッションの一時ファイルを読み書きでき、人の Claude Code のセッションの作業ファイルとコマンドの出力も読める。SAFE-10 の「読み取りは自分のセッションのフォルダだけ」に反する

### 案 E：AI のセッションを別の OS ユーザーで動かす

- 良い点: 人の Claude Code と、一時フォルダもホームも分かれる
- 悪い点: AI のセッションどうしは、同じ OS ユーザーで一時フォルダを共有したまま。`setup` で OS ユーザーと権限を作る準備が重く、Plugin を入れて使う形（[ADR-20261017-003][adr-plugin]）に合わない

## 補足情報（More Information）

- 関連する要件：SAFE-1、SAFE-4、SAFE-5、SAFE-10、SAFE-12、SAFE-18（[05-safety.md][safety]）、SETUP-2（[06-setup-and-stages.md][stages]）、STRUCT-2（[07-structure.md][struct]）、FACT-22、FACT-34（[10-verified-facts.md][facts]）、
  UNDECIDED-27、UNDECIDED-49、UNDECIDED-50（[09-undecided.md][undecided]）、[ADR-20261013-001][adr-workspace]、[ADR-20261015-001][adr-read]、[ADR-20261015-002][adr-store]、[ADR-20261015-005][adr-settings]、[ADR-20261017-005][adr-mcp]
- 2026-10-17 に実機で確かめた（macOS、Claude Code 2.1.285、`claude-haiku-5-5`、uid 501。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外）。
  `claude -p --setting-sources local --settings <安全設定（sandbox を有効、allowUnsandboxedCommands は false）> --strict-mcp-config --output-format stream-json` を、許可した名前だけの環境変数で起動し、
  sandbox の中の Bash で `$TMPDIR` と、各フォルダへの書き込み・読み取り・一覧を試すスクリプトを 1 回実行させた
  - `CLAUDE_CODE_TMPDIR` なし（今の要件の形）：`$TMPDIR` と `os.tmpdir()` は `/tmp/claude-501`。ユーザーごとの一時フォルダに書けて、中のファイルを読めて一覧も出せた。`denyRead` の外に置いたほかのセッションの一時フォルダは、読めた（書けなかった）
  - `CLAUDE_CODE_TMPDIR` の長さ（後ろの `/claude-501` を含む）：28・29・36・44 文字では `$TMPDIR` が渡した場所になり、ユーザーごとの一時フォルダには書けなかった。45・46・48・49・50・52・58・62 文字では `$TMPDIR` が `/tmp/claude-501` に戻り、渡した場所とユーザーごとの一時フォルダの両方に書けた
  - 長さは渡した文字列で数える：`/tmp/<名前>/claude-501` で 44 文字（実体の `/private/tmp/…` では 52 文字）を渡すと、渡した場所が使われた
  - 長いパスのまま、安全設定の `denyWrite` にユーザーごとの一時フォルダを入れると、Claude Code が作業ファイル（`/tmp/claude-501/cwd-…`）を書けず、Bash のコマンドが終了コード 1 になった
  - 案 A の形（`CLAUDE_CODE_TMPDIR` と `TMPDIR` に、`/claude-501` を含めて 28 文字になる場所、`denyRead` に根とユーザーごとの一時フォルダ、`allowRead` に自分の一時フォルダ）：`$TMPDIR` と `os.tmpdir()` は渡した場所の下の `claude-501`。ユーザーごとの一時フォルダとほかのセッションの一時フォルダは、読めず、一覧も出せず、書けなかった。Bash は普通に動いた
  - 案 A の安全設定のまま長いパス（50 文字）を渡すと、`$TMPDIR` は `/tmp/claude-501` に戻ってファイルを作れたが、ユーザーごとの一時フォルダの中は読めず一覧も出せなかった
  - `mktemp -d`（引数なし）は、どの形でも `$TMPDIR` でなく macOS のユーザーの一時フォルダ（`/var/folders/…`）に作ろうとして、sandbox に止められた
- 公式文書（[環境変数][doc-env]の `CLAUDE_CODE_TMPDIR`、[sandbox][doc-sandboxing]の「Temporary directories」）：Claude Code は後ろに `/claude-{uid}/` を足す。既定は macOS で `/tmp`、Linux で `os.tmpdir()`。
  パスが長いと、sandbox の中の Bash には既定の場所の下の短い `$TMPDIR` を渡す（長いとみなす値は書かれていない）。Claude Code 自身の一時ファイルは常に渡した場所を使う。シェル・ユーザー設定・managed の設定で効き、project と local の設定では無視される
- 要件への反映（承認の後に、別の PR で行う）
  - 05-safety.md：SAFE-10（`TMPDIR` の向け先を、セッションごとの短いパスの一時フォルダと `CLAUDE_CODE_TMPDIR` に。`denyRead` に一時フォルダの根とユーザーごとの一時フォルダ、`allowRead` に自分の一時フォルダ）、SAFE-12（セッションの一時フォルダも一緒に消す）
  - 06-setup-and-stages.md：SETUP-2（決定 4 の確かめ）
  - 07-structure.md：STRUCT-2（設定の項目に一時フォルダの根と長さの上限、子プロセスに渡す環境変数に `CLAUDE_CODE_TMPDIR`）
  - 10-verified-facts.md：上の試験の事実を FACT として足す
  - 09-undecided.md：UNDECIDED-50 を欠番にする（元の文を取り消し線で残す）。UNDECIDED-49 に、Linux で一時フォルダの振る舞いを確かめ直すことを足す
  - 図：隔離と鍵の図の一時フォルダの箱
  - ADR-20261015-001：タイトルの下に、決定 1 を本 ADR で改めた旨の注記を足す

[safety]: ../requirements/05-safety.md
[stages]: ../requirements/06-setup-and-stages.md
[struct]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[adr-workspace]: ./adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[adr-store]: ./adr-20261015-002-no-shared-dependency-store.md
[adr-settings]: ./adr-20261015-005-settings-values-and-machine-settings.md
[adr-plugin]: ./adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-mcp]: ./adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[doc-env]: https://code.claude.com/docs/en/env-vars
[doc-sandboxing]: https://code.claude.com/docs/en/sandboxing
