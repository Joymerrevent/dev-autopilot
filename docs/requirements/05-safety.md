# 安全側に倒す設計（隔離と鍵）

## 安全側に倒す設計

- SAFE-1 AI のセッション（5 役。「[登場するもの][f00]」）は GitHub に書けない。`gh` の認証と `GH_TOKEN` と進行ボットの鍵を渡さず（子プロセスの環境変数は許可した名前だけで組み立て、`check` が「子プロセスから PAT と鍵が見えない」ことを試す）、道具はエージェント定義で絞る（SETUP-18、IMPL-5）。
  push・PR・コメント・Project の更新は進行ボットだけが行う。進行ボットも `gh pr merge` を呼ばず、`develop` と `main` への直 push と force push はしない（ruleset でも止める。SETUP-9、MERGE-5）
- SAFE-2 **進行ボット・次のセッション・CI は、AI が書いた worktree の中身を信頼して実行しない（「[2026-10-11 のレビューで決めたことと、採らなかった案][f11]」）**。5 つの経路を塞ぐ：
  (a) AI のセッションはユーザー設定とプロジェクト設定から切り離して起動する（`--setting-sources local`。段階 2 で、既定の `-p` はプロジェクトの hook を実行し `CLAUDE.md` を読むこと、`local` ならユーザーの hook もプロジェクトの hook も `CLAUDE.md` も読まれないことを確認した。`--settings` の安全設定は効く）。Plugin は進行ボットのプログラムの clone の絶対パスで渡す（AI が改変した agent 定義と Skill を読まない）。
  (b) 進行ボットが AI の作業場所（親の clone と worktree）で git を実行するのは、AI のセッションを起動する前の clone と worktree の作成だけ。起動した後は、進行ボットの clone へ取り込む fetch の取り出し元にするだけで、消すときは git を使わずフォルダごと消す（SAFE-12）。
  (c) `.claude/**` `CLAUDE.md` `.mcp.json` `.github/**` `dev-autopilot/**` に触れる diff は push しない（IMPL-15。CI が AI の書いたワークフローを secrets 付きで実行しない）。必須チェックは名前に加え app と base 側のワークフローのパスで照合する（IMPL-27）。
  (d) 進行ボットはリポのコードとパッケージの lifecycle script を sandbox の外で実行しない（IMPL-1、IMPL-9）。漏えい検査は dev-autopilot 自身の `src/` で行う（IMPL-15）。
  (e) sandbox は Bash だけを覆い、Read・Write・Edit の各ツールは覆わない（段階 2 で確認）。ファイルの道具は `permissions` で絞る：`allow` は `Bash`（sandbox が境界。allow に入れないと `git commit` が承認待ちで拒否される）、`Write(//<worktree>/**)`、`Edit(//<worktree>/**)` だけで、`Read` は入れない（入れると worktree の外をすべて読める。FACT-13）。Read ツールは worktree の中だけを読め、外を読もうとすると `-p` では自動で拒否される（FACT-22。[ADR-20261015-001][adr-read]）。計画 AI が使う Grep と Glob も同じ（FACT-26）。`deny` に、重ねる守りとして `Read(//<秘密の置き場>/**)` と `Read(//<置き場>/controller/**)` と `Read(//<進行ボットのプログラムの clone>/**)`、`Write` と `Edit` の `//<親 clone>/.git/**` `//<worktree>/.claude/**` `//<worktree>/CLAUDE.md` `//<worktree>/.mcp.json` を入れる。絶対パスは `//` 始まりで書く（`/` 始まりはプロジェクト相対）
  sandbox の `--settings` のファイルは AI が書けない場所に置き、起動の前にハッシュを確かめる。一時フォルダはセッションごとに別のものを作る
- SAFE-3 `--max-turns` と時間の上限と `--max-budget-usd`（IMPL-13）で 1 回の作業の長さと費用を止める
- SAFE-4 判定できない状態（API の失敗、想定外の Status、作業場所の残骸。UNDECIDED-27）では、作業を増やさずに記録して止める（fail-closed）
- SAFE-5 sandbox の安全設定は、書いてあることではなく「実際に読めない・書けない」ことを `check` が試して確かめる。`denyRead` `denyWrite` のパスは `setup` が実体に解決して書く（シンボリックリンクを解決しない不具合が「[実機で確かめた事実][f10]」で見つかった）。ホームの中を指すリンクは、`denyRead` の `~/`（SAFE-10）で覆われる（FACT-22）
- SAFE-6 作業 AI・計画 AI・レビュー AI・批評 AI に渡すのはコラボレータが書いた文だけ。外の人の文は渡さない。受付 AI だけは、道具を持たずに外の文を読み、下書きを返す（TEXT-12）。渡す文も作業の内容であって、動き方を変える指示にはしない（「[Issue と PR の本文の扱い][f04]」）
- SAFE-7 AI に渡す文は、書いた人を問わず、渡す直前に毎回、進行ボットの検疫を通す。検疫の限界と、検疫の代わりに頼る主の防衛は TEXT-11 に書く
- SAFE-8 AI の出力が公開の場に書かれる前に、進行ボットが漏えいの検査を掛ける（diff だけでなく、すべてのコメント・PR の本文・commit メッセージ・起票する Issue。IMPL-15）
- SAFE-9 処理対象の判断は人。AI が書けるのは worktree の中と、親 clone の `.git` の一部（「隔離と鍵の分け方」）だけで、GitHub には書けない。マージの実行は人だけが行い、進行ボットは「[マージ][f03]」の判定を印として人に渡す。方針に係る PR は判定の対象外

## 隔離と鍵の分け方

開発リポは sandbox を使わないので、同じ OS ユーザーで動く作業 AI は `~/.ssh` の鍵と `gh` の設定を Bash から読め、worktree が共有する `.git` の hook を書き換えて進行ボットの push 時に実行させることもできる。
さらに GitHub の deploy key は認証専用で署名の検証に使えないので、「リポに絞った鍵 1 本で push も署名も」は成り立たない。隔離と鍵は次のように分ける。

- SAFE-10 AI の 5 役（「[登場するもの][f00]」）は、dev-autopilot が `--settings` で渡す安全設定で動かす：sandbox を有効、読み取りは自分のセッションのフォルダだけ（`denyRead` に `~/`・置き場・秘密の置き場・マシンの設定の置き場（ADR-20261015-005）、`allowRead` に `<置き場>/sessions/<セッションの名前>/` と、ホームの中にある道具のフォルダ。狭い方の規則が勝つ。FACT-22。[ADR-20261015-001][adr-read]）、
  書き込みは自分のセッションの worktree と一時フォルダと、親 clone の `.git`（sandbox が既定で書けるようにする範囲。`gitdir` は除く）だけ、`allowUnsandboxedCommands: false`、外への通信は許可した先（npm のレジストリなど）だけ。
  雛形は business-os の `plugin/templates/settings.json.tmpl` の sandbox の節を元にする（利用者向けの四重防衛と同じ考え方）。npm のキャッシュとプロキシは ADR 20261006-01 の設定を写す。
  ホームを読めないと git と pnpm が個人設定を読めずに止まるので、進行ボットは AI のセッションに `GIT_CONFIG_GLOBAL=/dev/null` と `NPM_CONFIG_USERCONFIG=/dev/null` を渡し、XDG の 4 つ（`XDG_CONFIG_HOME` `XDG_CACHE_HOME` `XDG_DATA_HOME` `XDG_STATE_HOME`）と `TMPDIR` をセッションの一時フォルダに向ける。commit の作者も環境変数で渡す（FACT-22）。置き場は `~/.claude/` の下に置かない（sandbox が書き込みを止める。FACT-23）。
  pnpm の store はセッションの一時フォルダの下（`$XDG_DATA_HOME/pnpm/store`）に置き、セッションの間で共有しない。共有の store（人の store も、dev-autopilot 用に別に作る store も）を `allowRead` と `allowWrite` に足さず、`PNPM_HOME` は渡さない（渡すと store の場所が変わる。FACT-25。[ADR-20261015-002][adr-store]）
- SAFE-11 AI は worktree で作業する。worktree の gitdir と objects は親 clone の `.git/` にあり、worktree のフォルダだけでは `git commit` が通らない。
  作業フォルダが worktree のとき sandbox は親 `.git` への書き込みを既定で許すので、`allowWrite` の列挙は要らない（段階 2 で `git commit` が通ることを確認）。
  `.git/worktrees/<名前>/gitdir` は `denyWrite` にする（段階 2 で、sandbox は `commondir` `config.worktree` と親の `config` `hooks/**` を既定で保護するが `gitdir` は書けたため）。
  AI のセッションの git では自動 gc を止め、書く場所を増やさない
- SAFE-12 git は、gitdir に `commondir` のファイルがあると、指す先のフォルダの `config`（`core.fsmonitor` `core.sshCommand` `diff.external` など）を読む。普通の clone の `.git/commondir` でも、進行ボットの `git fetch` が指す先の `core.fsmonitor` を実行した（FACT-20）。
  sandbox は worktree の `commondir`（`.git/worktrees/<名前>/commondir`）を保護する（FACT-9）が、保護の一覧は Claude Code の版で変わりうるうえ、親 clone の `.git/commondir` の作成のように確かめていない経路がある。
  sandbox の保護に重ねる守りとして、進行ボットは AI のセッションを起動した後の作業場所（親の clone と worktree）で git を実行しない。進行ボットは、AI の worktree のブランチを「進行ボットの clone」（下）へ fetch してから merge・patch-id・diff・漏えい検査・push を行う（SAFE-2 の (b)。「[2026-10-11 のレビューで決めたことと、採らなかった案][f11]」）。
  AI の作業場所は AI のセッションごとに作り直し、消すときは git を使わずフォルダごと消す。消す前に `commondir` と `gitdir` をファイルとして読み、作ったときの中身と違えば、消さずに `needs-human` にする（[ADR-20261013-001][adr-workspace]）。
  段階 2 の攻撃の再現では、進行ボットの clone からの fetch と親 clone での `git worktree list` / `remove` は差し替え先の config を読まなかった。worktree の中で git を実行したときだけ発火した
- SAFE-13 **進行ボットは AI の作業場所の clone の refs を使わない。** refs と packed-refs を AI が書けるので、base は進行ボットの clone で fetch した `origin/develop`、PR の head は GitHub の API が返す SHA だけを基準にする。載せ直し（MERGE-5）と push は fetch の直後に行い、API の head（初回は base）が push する SHA の祖先であること（fast-forward で届くこと）を確かめる（自分の merge の結果も、API の head から fast-forward で届く）
- SAFE-14 作業 AI の commit は署名なし（セッションの git 設定で `commit.gpgsign=false`）。`develop` への squash マージの commit は GitHub が作って署名するので Verified になる。
  作業ブランチの commit はマージ後に消えるため、再署名の仕組みは持たない。`develop` の ruleset に「署名必須」は入れない（入れると作業 AI の push が通らない）
- SAFE-15 鍵は 2 系統：人の署名鍵（手元の SSH 鍵。dev-autopilot は触らない）と、dev-autopilot のマシン専用の deploy key（business-os だけに書き込み可。進行ボットだけが使う）。
  進行ボットの `gh` は business-os に絞った fine-grained PAT を、進行ボットのプロセスの中でだけ持つ（`gh auth login` でマシンに残さない）。
  PAT、印の署名の鍵（IMPL-20 の (c)）、deploy key は、秘密の置き場（設定の `paths.secrets`。置き場の外の、同期されないフォルダ。権限はフォルダ 700・ファイル 600）のファイルに置く。進行ボットは起動したときに読み、`gh` を呼ぶときだけ `GH_TOKEN`、push と fetch のときだけ `GIT_SSH_COMMAND` で渡す。launchd の plist には書かない（[ADR-20261015-001][adr-read]）
- SAFE-16 進行ボットは専用の clone を 3 種類持つ：AI の worktree を切る「AI の作業場所」の clone、進行ボットだけが書き、当てる先のリポ（business-os）の fetch・merge・push を行う「進行ボットの clone」（`<置き場>/controller/`）、進行ボットだけが書き、`src/` を実行して `plugin/` を読み込ませる「進行ボットのプログラムの clone」（dev-autopilot のリポの clone。`devAutopilotPath`。[ADR-20261016-003][adr-move] の決定 3）。2 つの進行ボットの clone は、AI のセッションから読めず書けない。
  AI の作業場所の clone は AI のセッションごとに origin の URL から作り、手元の clone のパスからは作らない（パスから作ると objects のファイルをハードリンクで共有する。FACT-19。[ADR-20261013-001][adr-workspace]）。進行ボットが git を動かすときは `core.hooksPath` を空のフォルダに固定し、git の設定は環境変数（`GIT_CONFIG_COUNT` 等）で渡す
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
[adr-move]: ../adr/adr-20261016-003-move-to-own-repository-before-basic-design.md
