# 安全側に倒す設計（隔離と鍵）

## 安全側に倒す設計

- SAFE-1 AI のセッション（5 役。「[登場するもの][f00]」）は GitHub に書けない。`gh` の認証と `GH_TOKEN` と進行 Bot の鍵を渡さず（子プロセスの環境変数は許可した名前だけで組み立て、`check` が「子プロセスから PAT と鍵が見えない」ことを試す）、道具はエージェント定義で絞る（SETUP-18、IMPL-5）。
  push・PR・コメント・Project の更新は進行 Bot だけが行う。進行 Bot も `gh pr merge` を呼ばず、`develop` と `main` への直 push と force push はしない（ruleset でも止める。SETUP-9、MERGE-5）。
  AI のセッションは MCP のサーバー（ログインしている claude.ai のアカウントのコネクタを含む）を 1 つも読み込まない。進行 Bot は、安全設定の `disableClaudeAiConnectors: true`、子プロセスの環境変数の `ENABLE_CLAUDEAI_MCP_SERVERS=false`、`--strict-mcp-config`（`--mcp-config` は渡さない）の 3 つを重ね、
  重ねる守りとして安全設定の `permissions.deny` に `mcp__*` を入れる。`check` は、AI のセッションと同じ起動の形で、道具の一覧に MCP の道具が無いことを試す（SETUP-2。FACT-35。[ADR-20261017-005][adr-mcp]）。AI の役に MCP の道具を与えるのは、安全の境界を緩める変更として ADR で決める
- SAFE-2 **進行 Bot・次のセッション・CI は、AI が書いた worktree の中身を信頼して実行しない（「[2026-10-11 のレビューで決めたことと、採らなかった案][f11]」）**。5 つの経路を塞ぐ：
  (a) AI のセッションはユーザー設定とプロジェクト設定から切り離して起動する（`--setting-sources local`。実機の試し（段階 2）で、既定の `-p` はプロジェクトの hook を実行し `CLAUDE.md` を読むこと、`local` ならユーザーの hook もプロジェクトの hook も `CLAUDE.md` も読まれないことを確認した。`--settings` の安全設定は効く）。ただし、作業フォルダの `.claude/settings.local.json`（local の設定）は `local` でも読まれ、hook が sandbox の外で動く（FACT-56）。sandbox の中の Bash からは書けないが、当てる先のリポに commit されていれば clone に入るので、進行 Bot は起動の前に worktree に無いことを確かめ、あれば起動しない。`CLAUDE.local.md` も `local` で読まれ、作業フォルダとその上のフォルダのものは起動の時点で、下のフォルダのものはそのフォルダのファイルを読んだときに、指示として AI の文脈に入る（下のフォルダの `CLAUDE.md` と、作業フォルダの `AGENTS.md` は読まれない。FACT-60）。上のフォルダ（置き場や、その上の人のフォルダ）に人が置いた `CLAUDE.local.md` も入るので、進行 Bot は起動の前に、worktree の中と上のフォルダに `CLAUDE.local.md` が無いことを確かめ、あれば起動しない。作業 AI が足した `CLAUDE.local.md` は、(c) のとおり push しない。Plugin は進行 Bot のプログラムの写し（SAFE-16）の絶対パスで渡す（AI が改変した agent 定義と Skill を読まない）。
  (b) 進行 Bot が AI の作業場所（親の clone と worktree）で git を実行するのは、AI のセッションを起動する前の clone と worktree の作成だけ。起動した後は、進行 Bot の clone へ取り込む fetch の取り出し元にするだけで、消すときは git を使わずフォルダごと消す（SAFE-12）。
  (c) `.claude/**` `CLAUDE.md` `**/CLAUDE.local.md` `.mcp.json` `.github/**` `dev-autopilot/**` と規約の正典のパス（business-os なら `docs/adr/**` `docs/design/**`）に触れる diff は push しない
  （IMPL-15。CI が AI の書いたワークフローを secrets 付きで実行しない。次のセッションが、作業 AI の書き換えた正典を規則として読まない。[ADR-20261017-002][adr-review]）。`.claude/**` から `dev-autopilot/**` までの名指しした 6 つのパスは、進行 Bot のプログラムに固定し、リポの設定の PR で外せないようにする（リポの設定には、規約の正典のパスなどの足す分だけを持つ。[ADR-20261021-005][adr-blocked]）。必須チェックは名前に加え app と base 側のワークフローのパスで照合する（IMPL-27）。
  (d) 進行 Bot はリポのコードとパッケージの lifecycle script を sandbox の外で実行しない（IMPL-1、IMPL-9）。漏えい検査は dev-autopilot 自身の `plugin/src/` で行う（IMPL-15）。
  (e) sandbox は Bash だけを覆い、Read・Write・Edit の各ツールは覆わない（実機の試し（段階 2）で確認）。ファイルの道具は `permissions` で絞る：`allow` は `Bash`（sandbox が境界。allow に入れないと `git commit` が承認待ちで拒否される）、`Write(//<worktree>/**)`、`Edit(//<worktree>/**)` と、写しの Skill を読む `Read(//<進行 Bot のプログラムの写し>/skills/**)` だけで、ほかの `Read` は入れない（入れると worktree の外をすべて読める。FACT-13）。Skill を呼ぶ `Skill` の道具は、どの役にも持たせない（エージェント定義の `tools` に `Skill` を書いた役だけが使え、`allow` に入れなくても使える。FACT-42、FACT-55）。レビュー AI が使う `review` Skill の手順は、進行 Bot が本文に引数を入れて渡し、参照ファイルとスクリプトは写しの `skills/` から読む（[ADR-20261021-001][adr-review-skill]）。
  Read ツールは worktree の中と写しの `skills/` だけを読め、外を読もうとすると `-p` では自動で拒否される（FACT-22、FACT-32。[ADR-20261015-001][adr-read]、[ADR-20261017-004][adr-human]）。計画 AI が使う Grep と Glob も同じ（FACT-26）。`deny` に、重ねる守りとして `Read(//<秘密の置き場>/**)` と `Read(//<置き場>/controller/**)` と `Read(//<進行 Bot のプログラムの写し>/src/**)`（写しの全体を `deny` にすると、`deny` は `allow` より強いので `skills/` も読めない）、`Write` と `Edit` の `//<親 clone>/.git/**` `//<worktree>/.claude/**` `//<worktree>/CLAUDE.md` `//<worktree>/.mcp.json` を入れる。絶対パスは `//` 始まりで書く（`/` 始まりはプロジェクト相対）。
  `deny` には `Agent` も入れる（Agent ツールはどの役にも与えず、エージェント定義の `tools` にも書かない。道具の名前だけを書いた deny は、道具をモデルの文脈から取り除く。[ADR-20261017-002][adr-review]）。MCP の道具を取り除く `mcp__*` も入れる（SAFE-1）。Skill の道具を取り除く `Skill` も入れる（道具の名前だけの `Skill` で、`tools` に `Skill` を書いたエージェントからも道具が消える。FACT-59。[ADR-20261021-001][adr-review-skill]）
  sandbox の `--settings` のファイルは、進行 Bot がセッションごとに作って AI が書けない場所に置き、起動の前とセッションの後にハッシュを確かめる（安全設定のファイルは、セッションの途中で書き換えるとそのセッションに効くため。作り方は構造仕様の「AI のセッション」）。一時フォルダはセッションごとに別のものを作る
- SAFE-3 `--max-turns` と時間の上限と `--max-budget-usd`（IMPL-13）で 1 回の作業の長さと費用を止める
- SAFE-4 判定できない状態（API の失敗、想定外の Status、作業場所の残骸。見分け方は基本設計の「[状態遷移と判定][design-flow]」の「作業場所の残骸」）では、作業を増やさずに記録して止める（fail-closed）
- SAFE-5 sandbox の安全設定は、書いてあることではなく「実際に読めない・書けない」ことを `check` が試して確かめる。`denyRead` `denyWrite` のパスは、進行 Bot がセッションごとの安全設定を作るときに実体に解決する（シンボリックリンクを解決しない不具合が「[実機で確かめた事実][f10]」で見つかった）。ホームの中を指すリンクは、`denyRead` の `~/`（SAFE-10）で覆われる（FACT-22）
- SAFE-6 作業 AI・計画 AI・レビュー AI・批評 AI に渡すのはコラボレータが書いた文だけ。外の人の文は渡さない。受付 AI だけは、道具を持たずに外の文を読み、下書きを返す（TEXT-12）。渡す文も作業の内容であって、動き方を変える指示にはしない（「[Issue と PR の本文の扱い][f04]」）
- SAFE-7 AI に渡す文は、書いた人を問わず、渡す直前に毎回、進行 Bot の検疫を通す。検疫の限界と、検疫の代わりに頼る主の防衛は TEXT-11 に書く
- SAFE-8 AI の出力が公開の場に書かれる前に、進行 Bot が漏えいの検査を掛ける（diff だけでなく、すべてのコメント・PR の本文・commit メッセージ・起票する Issue。IMPL-15）
- SAFE-9 処理対象の判断は人。AI が書けるのは worktree の中と、親 clone の `.git` の一部（「隔離と鍵の分け方」）だけで、GitHub には書けない。マージの実行は人だけが行い、進行 Bot は「[マージ][f03]」の判定を印として人に渡す。方針に係る PR は判定の対象外

## 隔離と鍵の分け方

開発リポは sandbox を使わないので、同じ OS ユーザーで動く作業 AI は `~/.ssh` の鍵と `gh` の設定を Bash から読め、worktree が共有する `.git` の hook を書き換えて進行 Bot の push 時に実行させることもできる。
さらに GitHub の deploy key は認証専用で署名の検証に使えないので、「リポに絞った鍵 1 本で push も署名も」は成り立たない。隔離と鍵は次のように分ける。

- SAFE-10 AI の 5 役（「[登場するもの][f00]」）は、dev-autopilot が `--settings` で渡す安全設定で動かす：sandbox を有効（sandbox が起動できなければ、sandbox なしで動かさずに Claude Code が止まる `failIfUnavailable`）、読み取りは、ホーム・置き場・秘密などを読ませない（`denyRead` に `~/`・置き場・秘密の置き場・マシンの設定の置き場（ADR-20261015-005）・短い一時フォルダの根・Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`。[ADR-20261017-006][adr-tmp]）・macOS のユーザーごとの一時フォルダとキャッシュのフォルダ（`getconf DARWIN_USER_TEMP_DIR` と `DARWIN_USER_CACHE_DIR` が返す `/var/folders/<…>/T` と `…/C`）・`/Users/Shared`・`/Volumes`（`~/` の拒否だけでは読める。足しても `node`・`pnpm`・`git` は動く。FACT-63）、`allowRead` に `<置き場>/sessions/<セッションの名前>/` と、自分の短い一時フォルダと、写しの `skills/`（Skill の参照ファイルとスクリプトのため。FACT-32。[ADR-20261017-004][adr-human]）と、ホームの中にある道具のフォルダ。狭い方の規則が勝つ。FACT-22。sandbox の読み取りは既定で開いていて、`denyRead` に並べた場所だけが閉じるので、ホームの外の道具（`/opt/homebrew`、`/usr`）は読める。[ADR-20261015-001][adr-read]）、
  書き込みは自分のセッションの worktree と一時フォルダ（置き場の中の一時フォルダと、短い一時フォルダの下の `claude-<uid>/`）と、親 clone の `.git`（sandbox が既定で書けるようにする範囲。`gitdir` は除く）だけ、`allowUnsandboxedCommands: false`、外への通信は許可した先（npm のレジストリなど）だけ。
  雛形は business-os の `plugin/templates/settings.json.tmpl` の sandbox の節を元にする（利用者向けの四重防衛と同じ考え方）。npm のキャッシュとプロキシは ADR 20261006-01 の設定を写す。
  ホームを読めないと git と pnpm が個人設定を読めずに止まるので、進行 Bot は AI のセッションに `GIT_CONFIG_GLOBAL=/dev/null` と `NPM_CONFIG_USERCONFIG=/dev/null` を渡し、XDG の 4 つ（`XDG_CONFIG_HOME` `XDG_CACHE_HOME` `XDG_DATA_HOME` `XDG_STATE_HOME`）を置き場の中のセッションの一時フォルダに向ける。commit の作者も環境変数で渡す（FACT-22）。claude.ai のコネクタを止める `ENABLE_CLAUDEAI_MCP_SERVERS=false` も渡す（SAFE-1）。
  置き場は `~/.claude/` の下に置かない（sandbox が書き込みを止める。FACT-23）。Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）の下にも置かない（sandbox の中のコマンドが既定で書ける。FACT-32。[ADR-20261017-004][adr-human]）。
  `TMPDIR` は、セッションごとの短い一時フォルダ（例：`/tmp/dap/<セッションの短い ID>`。根は dev-autopilot 専用で、ホーム・置き場・`/tmp/claude-<uid>/` の外）に向け、`CLAUDE_CODE_TMPDIR` にも同じパスを渡す。
  Claude Code は後ろに `/claude-<uid>/` を足して、sandbox の中のコマンドの `$TMPDIR` にする（`TMPDIR` だけを渡しても、Claude Code はユーザーごとの一時フォルダに置き換える。FACT-34）。
  `<CLAUDE_CODE_TMPDIR>/claude-<uid>` が長いと、Claude Code は黙ってユーザーごとの一時フォルダに戻すので（FACT-37）、進行 Bot は起動の前に長さ（バイト数）を数え、設定の上限（初期値 44。STRUCT-2）を超えたら起動しない。
  短い一時フォルダは毎回新しく作り（すでにあれば起動しない）、根は無ければ権限 700 で作る。根が自分の持ち物で、シンボリックリンクでなく、権限が 700 であることを毎回確かめ、外れたら起動しない（`/tmp` はすべての OS ユーザーが書けるため。SAFE-4。[ADR-20261017-006][adr-tmp]）。
  pnpm の store はセッションの一時フォルダの下（`$XDG_DATA_HOME/pnpm/store`）に置き、セッションの間で共有しない。共有の store（人の store も、dev-autopilot 用に別に作る store も）を `allowRead` と `allowWrite` に足さず、`PNPM_HOME` は渡さない（渡すと store の場所が変わる。FACT-25。[ADR-20261015-002][adr-store]）
- SAFE-11 AI は worktree で作業する。worktree の gitdir と objects は親 clone の `.git/` にあり、worktree のフォルダだけでは `git commit` が通らない。
  作業フォルダが worktree のとき sandbox は親 `.git` への書き込みを既定で許すので、`allowWrite` の列挙は要らない（実機の試し（段階 2）で `git commit` が通ることを確認）。
  `.git/worktrees/<名前>/gitdir` は `denyWrite` にする（実機の試し（段階 2）で、sandbox は `commondir` `config.worktree` と親の `config` `hooks/**` を既定で保護するが `gitdir` は書けたため）。
  AI のセッションの git では自動 gc を止め、書く場所を増やさない
- SAFE-12 git は、gitdir に `commondir` のファイルがあると、指す先のフォルダの `config`（`core.fsmonitor` `core.sshCommand` `diff.external` など）を読む。普通の clone の `.git/commondir` でも、進行 Bot の `git fetch` が指す先の `core.fsmonitor` を実行した（FACT-20）。
  sandbox は worktree の `commondir`（`.git/worktrees/<名前>/commondir`）を保護する（FACT-9）が、保護の一覧は Claude Code の版で変わりうるうえ、親 clone の `.git/commondir` の作成のように確かめていない経路がある。
  sandbox の保護に重ねる守りとして、進行 Bot は AI のセッションを起動した後の作業場所（親の clone と worktree）で git を実行しない。進行 Bot は、AI の worktree のブランチを「進行 Bot の clone」（下）へ fetch してから merge・patch-id・diff・漏えい検査・push を行う（SAFE-2 の (b)。「[2026-10-11 のレビューで決めたことと、採らなかった案][f11]」）。
  AI の作業場所は AI のセッションごとに作り直し、消すときは git を使わずフォルダごと消す。セッションごとの短い一時フォルダ（SAFE-10）も、作業場所と一緒にフォルダごと消す（[ADR-20261017-006][adr-tmp]）。消す前に `commondir` と `gitdir` をファイルとして読み、作ったときの中身と違えば、消さずに `needs-human` にする（[ADR-20261013-001][adr-workspace]）。
  実機の試し（段階 2）の攻撃の再現では、進行 Bot の clone からの fetch と親 clone での `git worktree list` / `remove` は差し替え先の config を読まなかった。worktree の中で git を実行したときだけ発火した
- SAFE-13 **進行 Bot は AI の作業場所の clone の refs を使わない。** refs と packed-refs を AI が書けるので、base は進行 Bot の clone で fetch した `origin/develop`、PR の head は GitHub の API が返す SHA だけを基準にする。載せ直し（MERGE-6）と push は fetch の直後に行い、API の head（初回は、作業場所を作ったときに進行 Bot の clone で控えた base の SHA）が push する SHA の祖先であること（fast-forward で届くこと）を確かめる（自分の merge の結果も、API の head から fast-forward で届く）。初回に今の base の先端と比べないのは、作業の間に人が base にマージすると、履歴の書き換えが無くても祖先でなくなるため
- SAFE-14 作業 AI の commit は署名なし（セッションの git 設定で `commit.gpgsign=false`）。`develop` への squash マージの commit は GitHub が作って署名するので Verified になる。
  作業ブランチの commit はマージ後に消えるため、再署名の仕組みは持たない。`develop` の ruleset に「署名必須」は入れない（入れると作業 AI の push が通らない）
- SAFE-15 鍵は 2 系統：人の署名鍵（手元の SSH 鍵。dev-autopilot は触らない）と、dev-autopilot のマシン専用の deploy key（business-os だけに書き込み可。進行 Bot だけが使う）。
  進行 Bot の `gh` は business-os に絞った fine-grained PAT を、進行 Bot のプロセスの中でだけ持つ（`gh auth login` でマシンに残さない）。
  受け入れ工程（段階 4）で GitHub に書き始める前に、PAT から GitHub App に切り替える。切り替えの後は、アプリの秘密鍵から作る 1 時間で切れるトークンを使い、PAT を消す。要件の中の「進行 Bot の PAT」は、切り替えの後はアプリのトークンと読む（[ADR-20261020-002][adr-app]）。
  PAT（切り替えの後はアプリの秘密鍵）、印の署名の鍵（IMPL-20 の (c)）、deploy key は、秘密の置き場（設定の `paths.secrets`。置き場の外の、同期されないフォルダ。権限はフォルダ 700・ファイル 600）のファイルに置く。進行 Bot は起動したときに読み、`gh` を呼ぶときだけ `GH_TOKEN`、push と fetch のときだけ `GIT_SSH_COMMAND` で渡す。launchd の plist には書かない（[ADR-20261015-001][adr-read]）
- SAFE-16 進行 Bot は、専用の clone を 2 種類と、プログラムの写しを持つ：AI の worktree を切る「AI の作業場所」の clone、進行 Bot だけが書き、当てる先のリポ（business-os）の fetch・merge・push を行う「進行 Bot の clone」（`<置き場>/controller/`）、
  進行 Bot と `setup` だけが書き、`src/` を実行して Plugin として読み込ませる「進行 Bot のプログラムの写し」（`setup` が Plugin の版を置き場に写したもの。`devAutopilotPath`。[ADR-20261017-003][adr-plugin] の決定 2。STRUCT-3）。進行 Bot の clone は、AI のセッションから読めず書けない。
  プログラムの写しは、AI のセッションから書けず、読めるのは `skills/`（Skill の参照ファイルとスクリプト）だけで、`src/`（進行 Bot のコード）は読めない（SAFE-2 の (e)、SAFE-10。FACT-32。[ADR-20261017-004][adr-human]）。
  AI の作業場所の clone は AI のセッションごとに origin の URL から作り、手元の clone のパスからは作らない（パスから作ると objects のファイルをハードリンクで共有する。FACT-19。[ADR-20261013-001][adr-workspace]）。進行 Bot が git を動かすときは `core.hooksPath` を空のフォルダに固定し、git の設定は環境変数（`GIT_CONFIG_COUNT` 等）で渡す
- SAFE-17 `develop` にも ruleset（PR 必須、force push 禁止、base と最新であること）を掛け、deploy key で直接 push できるのは作業ブランチだけにする
- SAFE-18 手元の Mac で試す間（専用マシンの前）も同じ設定で動かす。sandbox が人の鍵と gh の設定を AI から隠すので、試験の段階で普段の鍵が読まれることも防げる

[f00]: ./00-overview.md
[f03]: ./03-merge.md
[f04]: ./04-external-text-and-markers.md
[f10]: ./10-verified-facts.md
[f11]: ./11-records.md
[adr-workspace]: ../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-store]: ../adr/adr-20261015-002-no-shared-dependency-store.md
[adr-plugin]: ../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-review]: ../adr/adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-human]: ../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-mcp]: ../adr/adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[adr-tmp]: ../adr/adr-20261017-006-per-session-short-tmpdir.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
[adr-app]: ../adr/adr-20261020-002-switch-to-github-app-before-writing.md
[adr-review-skill]: ../adr/adr-20261021-001-review-skill-text-without-skill-tool.md
[adr-blocked]: ../adr/adr-20261021-005-required-push-blocked-paths-in-program.md
