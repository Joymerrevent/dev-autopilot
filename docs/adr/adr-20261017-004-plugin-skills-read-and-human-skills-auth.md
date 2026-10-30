---
status: accepted
created: 2026-10-17
updated: 2026-10-30
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# AI のセッションには写しの skills/ だけを読ませ、setup の GitHub の準備は人の gh で行い、check と /dev-autopilot は進行ボットのジョブを起こし、マシンの設定は OS ごとの決まった場所に当てる先のリポごとに置く

> **Amended by [ADR-20261020-002][adr-app]（2026-10-20）**: 決定 2 の「進行ボットの PAT」は、GitHub App に切り替えた後はアプリのトークンと読む。人の `setup` が人の `gh` で GitHub の準備をすること、進行ボットに管理の権限を持たせないことは据え置く。
>
> **Amended by [ADR-20261030-001][adr-20261030-001]（2026-10-30）**: 決定 2 の「GitHub の準備は人の `gh` のログインで行う」と「人の `gh` のログインを持たない専用マシンでは、GitHub の準備は人の手元のマシンで行う」を、`setup` 用の GitHub App の人の代わりのトークン（`setup` の実行ごとに人がブラウザで承認する）で行う形に改めた。`check` と `/dev-autopilot` が進行 Bot のジョブを起こし、GitHub の点検を進行 Bot の認証で行うことは据え置く。
>
> **［2026-10-30 追記］** 決定 2 の「進行ボットが自分の PAT で行う」と「進行ボットの PAT に、ruleset を作る管理の権限を持たせない」は、[ADR-20261030-002][adr-20261030-002] で進行 Bot が最初から GitHub App だけを使うことになったので、進行 Bot の GitHub App と読む。

要件の UNDECIDED-36 と UNDECIDED-37（[決めていないこと][undecided]の「安全と設定」）を決める。
問いは 3 つある：AI のセッションが Plugin の Skill の参照ファイルとスクリプトをどう使うか、人が呼ぶ `setup` と `check` がどの認証で GitHub を扱うか、人が呼ぶ Skill がマシンの設定をどう見つけるか。

改める決定は次の 1 つで、ほかの決定は据え置く。

- [ADR-20261015-001][adr-read] の決定 1 のうち、AI が読める範囲（sandbox の `allowRead` は自分のセッションのフォルダだけ、Read ツールは worktree の中だけ）に、進行ボットのプログラムの写しの `skills/` を足す（決定 1）。
  重ねる守りの `deny` の Read は、写しの全体から写しの `src/` に狭める。置き場を `/tmp/claude-<uid>/` の下に置かないことを足す。秘密を読めないこと、ホームと別のセッションを読めないことは据え置く

## 背景と課題（Context and Problem Statement）

問い 1（UNDECIDED-36）：AI のセッションが Plugin の Skill をどう使うか

- 進行ボットは、マシンの置き場の中の写し（`<置き場>/program/<版>/`）を `--plugin-dir` で AI のセッションに渡す（STRUCT-3、[ADR-20261017-003][adr-plugin] の決定 2）。
  一方、AI のセッションは置き場を読めない（sandbox の `denyRead`）。Read ツールは worktree の外を読めず、重ねる守りとして写しの Read を `deny` にしている（SAFE-2 の (e)、SAFE-10、[ADR-20261015-001][adr-read] の決定 1）
- レビュー AI の `review` Skill は、参照ファイル（壊れ方の表など）を読み、コミットごとのゲートの検査（`review/scripts/per-commit-gates.ts`。REVIEW-7、REVIEW-13）を sandbox の中の Bash で実行する
- 2026-10-17 に、今の要件のとおりの安全設定で実機で試した（補足情報）。エージェント定義の読み込みと Skill の呼び出し（`SKILL.md` の本文の読み込み）はできた。
  一方、Skill の参照ファイルの Read は `deny` で拒否され、Skill のスクリプトの実行は sandbox がファイルを隠して `Cannot find module` で失敗した。UNDECIDED-36 が「拒否されるはず」と書いたとおりで、`review` Skill は今の安全設定では動かない

問い 2（UNDECIDED-37）：人が呼ぶ `setup` と `check` の認証

- 利用者は、当てる先のリポに Plugin を入れ、当てる先のリポで開いた Claude Code で `setup` を実行する（[ADR-20261017-003][adr-plugin] の決定 4）。`setup` は GitHub の設定（ラベル、Project の欄、ruleset）を作り、`check` は PAT の権限を確かめる（SETUP-1、SETUP-2）
- ruleset を作るには、リポの管理の権限が要る。進行ボットの fine-grained PAT は、business-os に絞って発行する（SAFE-15）
- 人のセッションに秘密（進行ボットの PAT など）を扱わせない（[ADR-20261015-001][adr-read] の決定 2）。人のセッションから進行ボットを直接起動せず、定期の実行と同じジョブを起こす（同じ ADR の決定 3。[ADR-20261017-003][adr-plugin] の決定 3）
- 専用マシンには人の `gh` のログインを置かない（「[決まっていること][overview]」のコミットの署名と鍵）。手元の Mac には人の `gh` のログインがある

問い 3（UNDECIDED-37）：人が呼ぶ Skill がマシンの設定をどう見つけるか

- マシンの設定（`devAutopilotPath`、置き場、記録と lock の置き場など）は、リポの外の同期されないフォルダに置く。進行ボットは、起動の定義から渡されるパスでマシンの設定を読む。既定のパスの値は構造仕様で決める（[ADR-20261015-005][adr-settings] の決定 1・2）
- 人が呼ぶ Skill（`setup`・`check`・`/dev-autopilot`）は、入れた Plugin（`~/.claude/plugins/cache/` の下）から動き、マシンの設定の場所を知る手段が無い。人にパスを打たせると、打ち間違いで別のマシンの設定を読み書きする

課題：AI のセッションが Skill の参照ファイルとスクリプトを、書き換えずに使えるようにするには、読み取りの範囲をどう広げるか。人が呼ぶ `setup` と `check` は、どの認証で GitHub を扱うか。人が呼ぶ Skill は、マシンの設定をどう見つけるか。

## 判断の決め手（Decision Drivers）

- レビュー AI が `review` Skill の参照ファイルとスクリプトを使えること（REVIEW-7）
- AI が、次のセッションが読み込む Plugin（エージェント定義と Skill）を書き換えられないこと（SAFE-2 の (a)）
- AI が読める範囲を、要るところだけ広げること。進行ボットのコードと秘密は読めないままにすること
- 人のセッションが、進行ボットの秘密を読まないこと（[ADR-20261015-001][adr-read] の決定 2）
- 進行ボットの PAT に、毎回の実行で使わない管理の権限を持たせないこと
- 人が呼ぶ Skill が、人にパスを打たせずにマシンの設定を見つけられること。リポにマシンのパスを書かないこと（[ADR-20261015-005][adr-settings]）

## 検討した選択肢（Considered Options）

問い 1：AI のセッションが Plugin の Skill をどう使うか

- 案 1-A：写しの `skills/` だけを、AI のセッションの読み取りの許可に足す（sandbox の `allowRead` と、`permissions.allow` の `Read`）。書き込みは足さない。写しの `src/` は読めないまま
- 案 1-B：進行ボットが、AI のセッションごとに Plugin をセッションのフォルダに写し、`--plugin-dir` はセッションの写しを指す
- 案 1-C：Skill を `SKILL.md` 1 つにまとめ、参照ファイルとスクリプトを持たない

問い 2：人が呼ぶ `setup` と `check` の認証

- 案 2-A：`setup` の GitHub の準備は、人の `gh` のログインで行う。マシンの準備は GitHub の認証を使わない。`check` と `/dev-autopilot` は、定期の実行と同じジョブを起こし、GitHub は進行ボットが自分の PAT で読む
- 案 2-B：`setup` も `check` も進行ボットのジョブに頼み、進行ボットの PAT で行う（PAT に管理の権限を持たせる）
- 案 2-C：`setup` のときだけ、人が管理の権限を持つトークンを環境変数で渡す

問い 3：人が呼ぶ Skill がマシンの設定をどう見つけるか

- 案 3-A：マシンの設定を、OS ごとの決まった場所に、当てる先のリポ（owner/repo）ごとに置く。人が呼ぶ Skill は、開いているプロジェクトの origin から owner/repo を引いて場所を決める
- 案 3-B：マシンの設定を、Plugin の持続するフォルダ（`${CLAUDE_PLUGIN_DATA}`。`~/.claude/plugins/data/<id>/`）に置く
- 案 3-C：人が Skill を呼ぶたびにパスを渡す（引数か環境変数）

## 決定（Decision Outcome）

採用した選択肢:「案 1-A」「案 2-A」「案 3-A」。
理由: AI が読める範囲は Skill に要るところだけ広がり、書き込みと進行ボットのコードと秘密は閉じたまま。人のセッションは進行ボットの秘密を読まず、進行ボットの PAT は管理の権限を持たない。
人が呼ぶ Skill は、開いているプロジェクトからマシンの設定を見つけられ、パスを打つ必要もリポにパスを書く必要も無い。

### 決定 1：AI のセッションには、写しの `skills/` だけを読ませる

- AI のセッションの安全設定（SAFE-2 の (e)、SAFE-10）に、写しの `skills/` の読み取りを足す：sandbox の `allowRead` に `<写し>/skills`、`permissions.allow` に `Read(//<写し>/skills/**)`
- 重ねる守りの `deny` の Read は、写しの全体（`Read(//<写し>/**)`）から、写しの `src/`（`Read(//<写し>/src/**)`）に狭める（`deny` は `allow` より強いので、写しの全体を `deny` に残すと `skills/` も読めない）。
  写しの中の `skills/` 以外の場所（`src/`、エージェント定義など）は、今のとおり sandbox の `denyRead` で隠れ、Read ツールでは作業フォルダの外として自動で拒否される
- 書き込みは足さない。写しは sandbox の書ける場所（worktree と一時フォルダ）の外にあり、Write と Edit の許可は worktree の中だけ（SAFE-2 の (e)）。
  2026-10-17 の試験で、写しの `skills/` への Bash の追記は `operation not permitted`、Write と Edit は拒否された（補足情報）
- 置き場（写しを含む）は、Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）の下に置かない。sandbox の中のコマンドは、ユーザーごとの一時フォルダに既定で書ける（試験で、一時フォルダの下に置いた写しには Bash で追記できた）。`check` は置き場の実体のパスで確かめる
- Skill は、同封したファイルを `${CLAUDE_SKILL_DIR}` で指す（Claude Code が Skill の本文の中で、`SKILL.md` のあるフォルダに置き換える）。Plugin の中からも、skills CLI で単体で入れたときも同じに動く（STRUCT-7）

### 決定 2：setup の GitHub の準備は人の gh で行い、check と /dev-autopilot は進行ボットのジョブを起こす

- `setup` は、GitHub の準備とマシンの準備に分ける。どちらも、作るものの一覧を出して人の承認を得てから作り、何度実行しても同じ結果になる（SETUP-1）
  - GitHub の準備：ラベル、Project の欄、ruleset。人の `gh` のログイン（人の権限。リポの管理と Project の書き込み）で行う
  - マシンの準備：プログラムの写し、定期の実行の登録、マシンの設定、置き場、秘密の置き場のフォルダ（権限 700・600。値は人が書く）。GitHub の認証を使わず、秘密のファイルを読まない
- 人の `gh` のログインを持たない専用マシンでは、GitHub の準備は人の手元のマシンで行い、専用マシンではマシンの準備だけを行う（「[決まっていること][overview]」のコミットの署名と鍵のまま）
- `check` と `/dev-autopilot` は、進行ボットを直接起動せず、定期の実行と同じジョブを起こす。GitHub の点検（ラベル、Project の欄、ruleset、PAT の権限）は、進行ボットが自分の PAT で行う。人のセッションは、実行の記録を読んで表示する。
  `check` を点検だけで動かす仕組み（別のジョブか、ジョブへの依頼のファイルか）は、基本設計で決める
- 進行ボットの PAT に、ruleset を作る管理の権限を持たせない。deploy key の作成と登録は、今のとおり人が行う（SETUP-1 の「人がやるもの」）。GitHub の点検に要る読み取りの権限は、基本設計で決める

### 決定 3：マシンの設定は OS ごとの決まった場所に、当てる先のリポごとに置く

- マシンの設定は、OS ごとの決まった場所に、当てる先のリポ（owner/repo）ごとに置く。場所の値は、構造仕様の設定の表で決める（[ADR-20261015-005][adr-settings] の決定 1。例：macOS は `~/Library/Application Support/dev-autopilot/<owner>/<repo>/`、Linux は `$XDG_CONFIG_HOME/dev-autopilot/<owner>/<repo>/`）
- 人が呼ぶ Skill は、開いているプロジェクトの origin から owner/repo を引き、決まった場所のマシンの設定を読む。マシンの設定から、進行ボットのプログラムの写し（`devAutopilotPath`）と記録の置き場を知る
- 定期の実行の定義（OS ごと）は、同じ場所のパスを進行ボットに渡す（ADR-20261015-005 の決定 2）。`check` は、人が呼ぶ Skill が引いた場所と、定期の実行の定義が渡す場所が同じことを確かめる
- 場所の実体のパスが同期フォルダの下に無いことは、今のとおり `check` が確かめる（ADR-20261015-005 の決定 2）。`~/.config` を同期フォルダへのリンクにしているマシンがある（[10-verified-facts.md][facts] の段階 2 の「見つかった不具合」）ので、実体に解決してから確かめる

### 影響（Consequences）

- 良い点: `review` Skill が、参照ファイルとスクリプトを使って動く。AI は写しを書き換えられず、進行ボットのコードと秘密も読めないまま
- 良い点: 人のセッションは進行ボットの秘密を読まず、進行ボットの PAT は毎回の実行に要る権限に絞ったままにできる
- 良い点: 人が呼ぶ Skill は、開いているプロジェクトからマシンの設定を見つける。リポにマシンのパスを書かず、人にパスを打たせない
- 悪い点: AI のセッションは、写しの `skills/` の中身（Skill の手順と参照ファイル）を読める。中身は公開のリポにあるものと同じで、秘密は無い
- 悪い点: `setup` が 2 つの準備に分かれ、専用マシンでは 2 台のマシンで実行する
- 悪い点: `check` を人から呼ぶと、ジョブが動き終わるまで結果が出ない
- 悪い点: マシンの設定の場所が OS ごとに 2 通りある

### 確認方法（Confirmation）

- 構造仕様を書くときに、安全設定の雛形（`allowRead`・`allow`・`deny`）、`setup` の 2 つの準備、マシンの設定の場所の表が、決定 1〜3 のとおりかをレビューで確かめる
- 段階 3（`setup` と `check` を作る段階）の `check`（SETUP-2、SAFE-5）で、sandbox の中と Read ツールの両方から、写しの `skills/` を読めて書けないこと、写しの `src/` を読めないこと、置き場が `/tmp/claude-<uid>/` の下に無いことを試す
- 結合テスト（偽の `gh`）で、`setup` のマシンの準備が GitHub に何も書かないこと、`check` が人のセッションから秘密のファイルを読まないことを検査する

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 1-A：写しの `skills/` だけを読み取りの許可に足す

- 良い点: 安全設定に 2 行足し、`deny` を 1 行狭めるだけで済む。写しは 1 つのまま
- 良い点: 2026-10-17 の試験で、参照ファイルの Read とスクリプトの実行が通り、`src/` は読めず、`skills/` には書けなかった
- 悪い点: AI のセッションが、全セッションで共有する写しの `skills/` を読む（書けないので、ほかのセッションへの経路にはならない）

### 案 1-B：セッションごとに Plugin をセッションのフォルダに写す

- 良い点: AI が読むのは自分のセッションの写しだけで、置き場の写しを開けない
- 悪い点: セッションごとに Plugin を写す処理と時間が増える。写しが 2 種類になり、`check` が確かめる場所が増える

### 案 1-C：Skill を `SKILL.md` 1 つにまとめる

- 良い点: 安全設定を変えずに済む（`SKILL.md` の本文は Claude Code が読み込む）
- 悪い点: コミットごとのゲートの検査（REVIEW-13 で TypeScript のスクリプトにした）を実行できない。参照ファイルを本文に入れると、本文が長くなる

### 案 2-A：GitHub の準備は人の gh、check はジョブを起こして進行ボットの PAT で読む

- 良い点: 人のセッションは進行ボットの秘密を読まない。進行ボットの PAT に管理の権限が要らない
- 悪い点: 専用マシンでは、GitHub の準備を別のマシンで行う。`check` の結果を待つ時間が要る

### 案 2-B：setup も check も進行ボットの PAT で行う

- 良い点: 認証が 1 つで済み、どのマシンでも同じ手順になる
- 悪い点: 進行ボットの PAT が、毎回の実行で使わない管理の権限（ruleset を作る）を持ち続ける。PAT が漏れたときの害が大きくなる

### 案 2-C：setup のときだけ、管理の権限を持つトークンを環境変数で渡す

- 良い点: 進行ボットの PAT を広げずに済む
- 悪い点: 人のセッションがトークン（秘密）を扱う（ADR-20261015-001 の決定 2 が避けたこと）。会話や記録にトークンが残りうる

### 案 3-A：OS ごとの決まった場所に、リポごとに置く

- 良い点: 人が呼ぶ Skill と進行ボットが、同じ規則で同じ場所を見つける。2 つ目の当てる先ができても、リポごとに分かれている
- 悪い点: 場所の規則を OS ごとに持つ

### 案 3-B：`${CLAUDE_PLUGIN_DATA}` に置く

- 良い点: Claude Code が Plugin ごとに用意し、版をまたいで残る。Skill の本文で `${CLAUDE_PLUGIN_DATA}` と書ける
- 悪い点: `~/.claude/` の下にある。`~/.claude/` を git などで同期している利用者では、同期されるフォルダの下にマシンの設定を置くことになりうる（ADR-20261015-005 の条件に反する）。Plugin を外すと消える

### 案 3-C：人が毎回パスを渡す

- 良い点: 決まった場所の規則が要らない
- 悪い点: 打ち間違いで、別のリポか別のマシンの設定を読み書きする。人の操作が増える

## 補足情報（More Information）

- 関連する要件：SAFE-2、SAFE-5、SAFE-10、SAFE-15、SAFE-16（[05-safety.md][safety]）、SETUP-1、SETUP-2（[06-setup-and-stages.md][stages]）、STRUCT-2、STRUCT-3、STRUCT-7、STRUCT-8（[07-structure.md][struct]）、REVIEW-7、REVIEW-13（[08-review-skill.md][review]）、
  「決まっていること」のコミットの署名と鍵（[00-overview.md][overview]）、UNDECIDED-45（[09-undecided.md][undecided]）、[ADR-20261015-001][adr-read]、[ADR-20261015-005][adr-settings]、[ADR-20261017-003][adr-plugin]
- 2026-10-17 に、AI のセッションが写しの Plugin をどう使えるかを実機で確かめた（macOS、Claude Code 2.1.285、`claude-haiku-5-5`）。
  置き場の中に写し（`program/0.0.1/`。エージェント定義、Skill、Skill の参照ファイルとスクリプト、`src/`）と、セッションのフォルダの worktree を作り、
  `claude -p --setting-sources local --settings <安全設定> --plugin-dir <写し> --agent <レビュー用> --output-format stream-json` を、許可した名前だけの環境変数で worktree から起動した。
  試験の Skill は、参照ファイルとスクリプトを `${CLAUDE_SKILL_DIR}` で指した
  - 今の要件の安全設定（`denyRead` に `~/` と置き場、`allowRead` に自分のセッションのフォルダ、`deny` に写しの Read）：エージェント定義は読み込まれ、Skill を呼べた（`SKILL.md` の本文は Claude Code が読み込んだ）。
    参照ファイルの Read は「File is in a directory that is denied by your permission settings」で拒否され、`node <写し>/skills/.../scripts/probe.mjs` は `Cannot find module` で失敗し、`src/` は Bash でも Read でも読めなかった
  - 決定 1 の安全設定（`allowRead` と `allow` の Read に写しの `skills/`、`deny` は写しの `src/` だけ）：参照ファイルを Read で読め、スクリプトを Bash で実行でき、`src/` は Bash でも Read でも読めなかった。
    写しの `skills/` への Bash の追記は `operation not permitted`、Write と Edit は「sensitive file」として拒否された
  - 最初は試験の場所を Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-501/`）の下に置いた。一時フォルダの下に置いた写しでは、`skills/` への Bash の追記が通った（Write と Edit は拒否された）。場所を `/private/tmp/` の別の名前のフォルダに移すと止まった
- 2026-10-17 に、skills CLI（1.7.2）で、`plugin/skills/` の下の Skill を見つけるかを確かめた（UNDECIDED-45）。`skills add <フォルダ> --list` は、試験用のフォルダの `plugin/skills/review/` と `plugin/skills/setup/` の 2 つを見つけた。
  `skills add anthropics/claude-code --list` は、リポの中の `SKILL.md` 10 個（`plugins/<名前>/skills/<名前>/SKILL.md` の深さを含む）をすべて見つけた。dev-autopilot のリポを指して試すのは、`plugin/skills/` を作った後になる
- 2026-10-17 に、Claude Code の公式文書（[Skill][doc-skills]、[sandbox][doc-sandboxing]、[環境変数][doc-env]）を読んだ。読んだこと：
  - Skill の本文の中の `${CLAUDE_SKILL_DIR}` は、`SKILL.md` のあるフォルダに置き換わる。Plugin の Skill では、Plugin の中の Skill のフォルダになる
  - sandbox の中のコマンドは、作業フォルダとユーザーごとの一時フォルダに既定で書ける。Claude Code は、sandbox の中のコマンドの `$TMPDIR` を、ユーザーごとの一時フォルダに置き換える。
    一時フォルダの場所は `CLAUDE_CODE_TMPDIR` で変えられ、Claude Code が後ろに `/claude-<uid>/` を足す（パスが長いと、sandbox の中のコマンドには既定の場所の下の短いパスが渡る）
- 本 ADR の決定と関わらない発見（要件の直し方は、本 ADR と別に決める）：上の試験で、sandbox の中の `$TMPDIR` は、進行ボットが渡した値（セッションの一時フォルダ）でなく `/tmp/claude-501` だった。
  [ADR-20261015-001][adr-read] の決定 1 と SAFE-10 は、`TMPDIR` をセッションの一時フォルダに向けるとしている。`/tmp/claude-<uid>/` は同じユーザーで動くすべての AI のセッションから書けるので、セッションの間でファイルを受け渡す経路になりうる
- 要件への反映（承認の後に、別の PR で行う）
  - 05-safety.md：SAFE-2 の (e)（`allow` の Read に写しの `skills/`、`deny` の Read を写しの `src/` に）、SAFE-10（`allowRead` に写しの `skills/`、置き場を `/tmp/claude-<uid>/` の下に置かない）、SAFE-16（AI は写しの `skills/` を読めるが書けない）
  - 06-setup-and-stages.md：SETUP-1（GitHub の準備とマシンの準備に分け、GitHub の準備は人の `gh` で行う。専用マシンではマシンの準備だけ）、SETUP-2（`check` はジョブを起こして進行ボットの PAT で点検する、写しの読み書きと置き場の場所の確かめ、マシンの設定の場所の一致）
  - 07-structure.md：STRUCT-2 か STRUCT-3（マシンの設定の場所の規則と、人が呼ぶ Skill の見つけ方）、STRUCT-7（Skill は同封したファイルを `${CLAUDE_SKILL_DIR}` で指す）
  - 10-verified-facts.md：上の試験の事実を FACT として足す（写しの Plugin の読み取り、skills CLI、sandbox の `$TMPDIR`）
  - 09-undecided.md：UNDECIDED-36、UNDECIDED-37、UNDECIDED-45 を欠番にする（元の文を取り消し線で残す）。sandbox の `$TMPDIR` の発見を、決めていないことに足す
  - ADR-20261015-001：タイトルの下に、決定 1 を本 ADR で改めた旨の注記を足す

[overview]: ../requirements/00-overview.md
[safety]: ../requirements/05-safety.md
[stages]: ../requirements/06-setup-and-stages.md
[struct]: ../requirements/07-structure.md
[review]: ../requirements/08-review-skill.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[adr-read]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[adr-settings]: ./adr-20261015-005-settings-values-and-machine-settings.md
[adr-plugin]: ./adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[doc-skills]: https://code.claude.com/docs/en/skills
[doc-sandboxing]: https://code.claude.com/docs/en/sandboxing
[doc-env]: https://code.claude.com/docs/en/env-vars
[adr-app]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-20261030-001]: ./adr-20261030-001-setup-uses-setup-app-instead-of-human-gh.md
[adr-20261030-002]: ./adr-20261030-002-bot-uses-github-app-only-from-start.md
