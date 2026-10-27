# 安全側に倒す設計（隔離と鍵）

## 安全側に倒す設計

- SAFE-1 AI のセッション（5 役。「[登場するもの][f00]」）は GitHub に書けない。`gh` の認証と `GH_TOKEN` と進行 Bot の鍵を渡さず（子プロセスの環境変数は許可した名前だけで組み立て、`check` が「子プロセスから PAT と鍵が見えない」ことを試す）、道具はエージェント定義で絞る（SETUP-18、IMPL-5）。
  push・PR・コメント・Project の更新は進行 Bot だけが行う。進行 Bot も `gh pr merge` を呼ばず、`develop` と `main` への直 push と force push はしない（ruleset でも止める。SETUP-9、MERGE-5）。
  AI のセッションは MCP のサーバー（ログインしている claude.ai のアカウントのコネクタを含む）を 1 つも読み込まない。進行 Bot は、安全設定の `disableClaudeAiConnectors: true`、子プロセスの環境変数の `ENABLE_CLAUDEAI_MCP_SERVERS=false`、`--strict-mcp-config`（`--mcp-config` は渡さない）の 3 つを重ね、
  重ねる守りとして安全設定の `permissions.deny` に `mcp__*` を入れる。`check` は、AI のセッションと同じ起動の形で、道具の一覧に MCP の道具が無いことを試す（SETUP-2。FACT-35。[ADR-20261017-005][adr-mcp]）。AI の役に MCP の道具を与えるのは、安全の境界を緩める変更として ADR で決める
- SAFE-2 **進行 Bot・次のセッション・CI は、AI が書いた worktree の中身を信頼して実行しない（「[2026-10-11 のレビューで決めたことと、採らなかった案][f11]」）**。5 つの経路を塞ぐ：
  (a) AI のセッションはユーザー設定とプロジェクト設定から切り離して起動する（`--setting-sources local`。実機の試し（段階 2）で、既定の `-p` はプロジェクトの hook を実行し `CLAUDE.md` を読むこと、`local` ならユーザーの hook もプロジェクトの hook も `CLAUDE.md` も読まれないことを確認した。`--settings` の安全設定は効く）。ただし、作業フォルダの `.claude/settings.local.json`（local の設定）は `local` でも読まれ、hook が sandbox の外で動く（FACT-56）。sandbox の中の Bash からは書けないが、当てる先のリポに commit されていれば clone に入るので、進行 Bot は起動の前に worktree に無いことを確かめ、あれば起動しない。`CLAUDE.local.md` も `local` で読まれ、作業フォルダとその上のフォルダのものは起動の時点で、下のフォルダのものはそのフォルダのファイルを読んだときに、指示として AI の文脈に入る（下のフォルダの `CLAUDE.md` と、作業フォルダの `AGENTS.md` は読まれない。FACT-60）。上のフォルダ（置き場や、その上の人のフォルダ）に人が置いた `CLAUDE.local.md` も入るので、進行 Bot は起動の前に、worktree の中と上のフォルダに `CLAUDE.local.md` が無いことを確かめ、あれば起動しない。作業 AI が足した `CLAUDE.local.md` は、(c) のとおり push しない。Plugin は進行 Bot のプログラムの写し（SAFE-16）の絶対パスで渡す（AI が改変した agent 定義と Skill を読まない）。
  (b) 進行 Bot が AI の作業場所（親の clone と worktree）で git を実行するのは、AI のセッションを起動する前の clone と worktree の作成だけ。起動した後は、進行 Bot の clone へ取り込む fetch の取り出し元にするだけで、消すときは git を使わずフォルダごと消す（SAFE-12）。
  (c) 変えてはいけないパス（`.claude/**` `CLAUDE.md` `**/CLAUDE.local.md` `.mcp.json` `.github/**` `dev-autopilot/**` と規約の正典のパス（business-os なら `docs/adr/**` `docs/design/**`））に触れる diff は、maintain 以上の人が `## 仕組みの変更の承認` で認めていなければ push しない（IMPL-15）。
  push で効くもの（CI の定義、CI が信じて動かすスクリプト、AI のセッションが作業場所から読み込む設定）は、認めていても AI の変更を push せず、人が差分を確かめて push する（CI が AI の書いたワークフローを secrets 付きで実行しないため）。規約の正典などのマージまで効かないものは、認めれば AI が変えてよく、次のセッションは正典を base ブランチから読む（作業 AI の書き換えた正典を、人がマージする前に規則として読まないため。[ADR-20261017-002][adr-review]、[ADR-20261024-004][adr-master] の決定 9）。`.claude/**` から `dev-autopilot/**` までの名指しした 6 つのパスは、進行 Bot のプログラムに固定し、リポの設定の PR で外せないようにする（リポの設定には、規約の正典のパスなどの足す分だけを持つ。[ADR-20261021-005][adr-blocked]）。必須チェックは名前に加え app と base 側のワークフローのパスで照合する（IMPL-27）。
  (d) 進行 Bot はリポのコードとパッケージの lifecycle script を sandbox の外で実行しない（IMPL-1、IMPL-9）。漏えい検査は dev-autopilot 自身の `plugin/src/` で行う（IMPL-15）。
  (e) sandbox は Bash だけを覆い、Read・Write・Edit の各ツールは覆わない（実機の試し（段階 2）で確認）。ファイルの道具は `permissions` で絞る：`allow` は `Bash`（sandbox が境界。allow に入れないと `git commit` が承認待ちで拒否される）、`Write(//<worktree>/**)`、`Edit(//<worktree>/**)` と、写しの Skill を読む `Read(//<進行 Bot のプログラムの写し>/skills/**)` だけで、ほかの `Read` は入れない（入れると worktree の外をすべて読める。FACT-13）。Skill を呼ぶ `Skill` の道具は、どの役にも持たせない（エージェント定義の `tools` に `Skill` を書いた役だけが使え、`allow` に入れなくても使える。FACT-42、FACT-55）。レビュー AI が使う `review` Skill の手順は、進行 Bot が本文に引数を入れて渡し、参照ファイルとスクリプトは写しの `skills/` から読む（[ADR-20261021-001][adr-review-skill]）。
  Read ツールは worktree の中と写しの `skills/` だけを読め、外を読もうとすると `-p` では自動で拒否される（FACT-22、FACT-32。[ADR-20261015-001][adr-read]、[ADR-20261017-004][adr-human]）。計画 AI が使う Grep と Glob も同じ（FACT-26）。`deny` に、重ねる守りとして `Read(//<秘密の置き場>/**)` と `Read(//<置き場>/controller/**)` と `Read(//<進行 Bot のプログラムの写し>/src/**)`（写しの全体を `deny` にすると、`deny` は `allow` より強いので `skills/` も読めない）、`Write` と `Edit` の `//<親 clone>/.git/**` `//<worktree>/.claude/**` `//<worktree>/CLAUDE.md` `//<worktree>/.mcp.json` を入れる。絶対パスは `//` 始まりで書く（`/` 始まりはプロジェクト相対）。
  `deny` には `Agent` も入れる（Agent ツールはどの役にも与えず、エージェント定義の `tools` にも書かない。道具の名前だけを書いた deny は、道具をモデルの文脈から取り除く。[ADR-20261017-002][adr-review]）。MCP の道具を取り除く `mcp__*` も入れる（SAFE-1）。Skill の道具を取り除く `Skill` も入れる（道具の名前だけの `Skill` で、`tools` に `Skill` を書いたエージェントからも道具が消える。FACT-59。[ADR-20261021-001][adr-review-skill]）
  sandbox の `--settings` のファイルは、進行 Bot がセッションごとに作って AI が書けない場所に置き、起動の前とセッションの後にハッシュを確かめる（安全設定のファイルは、セッションの途中で書き換えるとそのセッションに効くため。作り方は構造仕様の「AI のセッション」）。一時フォルダはセッションごとに別のものを作る
- SAFE-3 `--max-turns` と時間の上限と `--max-budget-usd`（IMPL-13）で 1 回の作業の長さと費用を止める
- SAFE-4 判定できない状態（API の失敗、GitHub の状態から次の工程を決められない組み合わせ、作業場所の残骸。見分け方は基本設計の「[状態遷移と判定][design-flow]」の「作業場所の残骸」）では、作業を増やさずに記録して止める（fail-closed）
- SAFE-5 sandbox の安全設定は、書いてあることではなく「実際に読めない・書けない」ことを `check` が試して確かめる。`denyRead` `denyWrite` のパスは、進行 Bot がセッションごとの安全設定を作るときに実体に解決する（シンボリックリンクを解決しない不具合が「[実機で確かめた事実][f10]」で見つかった）。ホームの中を指すリンクは、`denyRead` の `~/`（SAFE-10）で覆われる（FACT-22）
- SAFE-6 作業 AI・計画 AI・レビュー AI・批評 AI に渡す依頼の文は、人が決めたマスターの下書き（人の作業指示は、検疫の後にマスターに写したもの）だけ（TEXT-1、FLOW-36）。外の人の文は渡さない。受付 AI だけは、道具を持たずに外の文を読み、下書きを返す（TEXT-12）。渡す文も作業の内容であって、動き方を変える指示にはしない（「[Issue と PR の本文の扱い][f04]」）
- SAFE-7 AI に渡す文と、AI が GitHub に書く文は、書いた人を問わず、渡す前と書く前に毎回、進行 Bot の検疫を通す。見つかったら安全の軸に値を入れる（SAFE-19）。検疫の限界と、検疫の代わりに頼る主の防衛は TEXT-11 に書く
- SAFE-8 AI の出力が公開の場に書かれる前に、進行 Bot が漏えいの検査を掛ける（diff だけでなく、すべてのコメント・PR の本文・commit メッセージ・起票する Issue。IMPL-15）
- SAFE-9 処理対象の判断は人。AI が書けるのは worktree の中と、親 clone の `.git` の一部（「隔離と鍵の分け方」）だけで、GitHub には書けない。マージの実行は人だけが行い、進行 Bot は「[マージ][f03]」の判定を印として人に渡す。方針に係る PR は判定の対象外
- SAFE-19 **安全の軸と全体の停止**（[ADR-20261024-005][adr-security]、[ADR-20261027-002][adr-leak]）。検疫（TEXT-11）で見つかったら、止め方（停止中）でなく、作業の単位の安全の軸に値を入れる（FLOW-38）。AI に渡す前の文（Issue の文、人の見出しのコメント）で見つかれば攻撃の疑い（入口）、AI が書いた文か PR の差分（人の push を含む）で見つかれば攻撃の疑い（AI の文）。GitHub に書く文の漏えいの検査（IMPL-15）で見つかったときも、止め方でなく、攻撃の疑い（AI の文）にする。値がある間、AI はその作業の単位を進めない。
  値を外すのは、maintain 以上の人が見つかった文を直してから書く `## 安全の確認` だけ（`## 再開` とラベルを外す操作では外れない）。外した後も検疫は毎回行い、見つかった文が残っていれば、また値が入る。
  AI が書いた文（下書き、計画、PR の題名と本文、指摘、直しの返事など。GitHub に書く前）で見つかったら、AI が操られた疑いがあり、同じ手口がほかの作業に届いているかもしれないので、進行 Bot が全体の停止の Issue を開いて `halt` の印を書き（開いている全体の停止の Issue があれば、そこに足す）、どの作業の単位でも AI のセッションを起こさない。開いた実行で動いている AI のセッションは、全体の止める合図と同じ手順で止める。AI を使わない進行 Bot の工程（検疫、表示の欄、片付けなど）は続ける。
  全体の停止は、maintain 以上の人が、全体の停止の Issue に最新の `halt` の印より後に `## 安全の確認` を書き、Issue を閉じたときに解ける。確かめずに閉じたときと、確かめても閉じていないときは、止めたままにする。全体の停止を解いても、見つけた作業の単位の値は残り、作業の単位の Issue の `## 安全の確認` で外す。PR の差分（コード）で見つかったときは、全体を止めず、見つかった作業の単位だけを止める（コードの中の文字列やテストのデータで誤って反応しやすいため）。漏えいの検査で見つかったときも、全体を止めない（機械上のパスやメールアドレスで誤って反応しやすいため）。
  人が全体を止めるリポの変数（「[運用の規則][f01]」の止める合図）は据え置く。進行 Bot は、リポの変数が `run` で、全体の停止の Issue が無いときだけ AI のセッションを起こし、リポの変数を書く権限を持たない。
  攻撃の疑いは、進行 Bot が付ける表示のラベル `security-suspected`（値が外れたら外す。判定には読まない）、見つかった種類と場所を書いたコメント（`security` の印。見つかった文は写さない）、Project の安全の欄、GitHub の外への通知（仕組みと鍵は後で決める）で知らせる

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
  AI の作業場所は AI のセッションごとに作り直し、消すときは git を使わずフォルダごと消す。セッションごとの短い一時フォルダ（SAFE-10）も、作業場所と一緒にフォルダごと消す（[ADR-20261017-006][adr-tmp]）。消す前に `commondir` と `gitdir` をファイルとして読み、作ったときの中身と違えば、消さずに停止中にする（止めた理由の記号は、実装の都合の記号として詳細設計で足す。[ADR-20261013-001][adr-workspace]）。
  実機の試し（段階 2）の攻撃の再現では、進行 Bot の clone からの fetch と親 clone での `git worktree list` / `remove` は差し替え先の config を読まなかった。worktree の中で git を実行したときだけ発火した
- SAFE-13 **進行 Bot は AI の作業場所の clone の refs を使わない。** refs と packed-refs を AI が書けるので、base は進行 Bot の clone で fetch した `origin/develop`、PR の head は GitHub の API が返す SHA だけを基準にする。載せ直し（MERGE-6）と push は fetch の直後に行い、API の head（初回は、作業場所を作ったときに進行 Bot の clone で控えた base の SHA）が push する SHA の祖先であること（fast-forward で届くこと）を確かめる（自分の merge の結果も、API の head から fast-forward で届く）。初回に今の base の先端と比べないのは、作業の間に人が base にマージすると、履歴の書き換えが無くても祖先でなくなるため
- SAFE-14 作業 AI の commit は署名なし（セッションの git 設定で `commit.gpgsign=false`）。`develop` への squash マージの commit は GitHub が作って署名するので Verified になる。
  作業ブランチの commit はマージ後に消えるため、再署名の仕組みは持たない。`develop` の ruleset に「署名必須」は入れない（入れると作業 AI の push が通らない）
- SAFE-15 鍵は 2 系統：人の署名鍵（手元の SSH 鍵。dev-autopilot は触らない）と、dev-autopilot のマシン専用の deploy key（business-os だけに書き込み可。進行 Bot だけが使う）。
  進行 Bot の `gh` は business-os に絞った fine-grained PAT を、進行 Bot のプロセスの中でだけ持つ（`gh auth login` でマシンに残さない）。
  進行 Bot が当てる先のリポに初めて書く前に、PAT から GitHub App に切り替える（書かずに試すことは持たない。[ADR-20261024-002][adr-no-stages]）。切り替えの後は、アプリの秘密鍵から作る 1 時間で切れるトークンを使い、PAT を消す。要件の中の「進行 Bot の PAT」は、切り替えの後はアプリのトークンと読む（[ADR-20261020-002][adr-app]）。
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
[adr-no-stages]: ../adr/adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[adr-master]: ../adr/adr-20261024-004-master-draft-quarantine-and-protected-paths.md
[adr-security]: ../adr/adr-20261024-005-security-axis-and-halt-issue.md
[f01]: ./01-issue-flow.md
[adr-leak]: ../adr/adr-20261027-002-leak-findings-go-to-security-axis.md
