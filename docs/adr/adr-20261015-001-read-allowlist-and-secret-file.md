---
status: accepted
created: 2026-10-15
updated: 2026-10-17
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# AI の読み取りは自分のセッションのフォルダに絞り、秘密は進行役だけが読むファイルに置き、手動実行は launchd のジョブを起こす

> **Amended by [ADR-20261016-003][adr-move]（2026-10-17）**: 重ねる守りの `deny` の Read に、進行ボットのプログラムの clone（`devAutopilotPath`）を足した。進行ボットのプログラムを、`<置き場>/controller/`（business-os の clone）とは別の clone に置くようになったため。ほかの決定は据え置く。
>
> **Amended by [ADR-20261017-003][adr-plugin]（2026-10-17）**: 決定 3 の「手動実行は launchd のジョブを起こす」を、OS ごとの定期の実行の仕組みで同じジョブを起こす形に改めた（macOS は `launchctl kickstart`（`-k` なし）、Linux は登録した仕組みで同じジョブを起こす操作）。
> 重ねる守りの `deny` の Read の「進行ボットのプログラムの clone」は、プログラムの写しを指すように改めた。人のセッションから進行ボットを直接起動しないことと、ほかの決定は据え置く。

要件の UNDECIDED-16（[決めていないこと][undecided]の「AI の読み取り範囲と秘密の置き場」）を決める。
問いは 3 つある：AI が読める範囲、秘密（PAT・印の署名の鍵・deploy key）の置き場、人が手で実行するときの起こし方。

## 背景と課題（Context and Problem Statement）

今の要件は、AI の読み取りを「全体を許し、秘密の場所を列挙して拒否する」形で書いている。

- Read ツール：`permissions.allow` に `Read`（全体）を入れ、`deny` に `Read(//<秘密のパス>/**)`（進行役の設定・記録・鍵・`~/.ssh` など）を入れる（SAFE-2 の (e)）
- sandbox：`denyRead` に `~/.ssh` `~/.config/gh` と進行役の設定・記録を入れる（SAFE-10）

列挙する形には、次の穴がある。

1. 一覧に無いものは読める。秘密を保管する場所（PAT と印の署名の鍵をどこに置くかは決まっていない）、別の案件の会話の記録（`~/.claude/projects/**`）、`~/.npmrc`、別のセッションの作業場所、進行役の clone が一覧に無い。
   Issue の文に紛れた指示で作業 AI が読めば、漏えいの検査（鍵の形式・メールアドレス・機械上のパス）が拾わない形で、公開のリポに出しうる
2. sandbox の `denyRead` は、パスの途中にあるシンボリックリンクを解決しない（[実機で確かめた事実][facts]の不具合）。手元の Mac では `~/.config` `~/.ssh` `~/.gitconfig` などが同期フォルダへのリンクで、一覧に書いたパスが効かないことがある
3. Read ツールには sandbox の `denyRead` が効かず、`allow` に `Read` があると一覧の外をすべて読める（FACT-13）

秘密の置き場も決まっていない。SAFE-15 は PAT を「進行役のプロセスの環境変数でだけ持つ」とし、IMPL-20 の (c) は印の署名の鍵を「PAT と同じ置き場」としているが、マシンのどこに保管し、どう進行役のプロセスに渡すかが無い。deploy key のファイルの置き場も無い。

手動実行（人が `/dev-autopilot` を呼ぶ。「[分離を見越した構成][struct]」）の起こし方も決まっていない。
人の Claude Code のセッションから進行役を直接起動すると、進行役は人のセッションの子プロセスになり、人の環境（SSH の agent、`gh` のログイン、セッションの設定）を受け継ぐ。定期の実行（launchd）と環境が変わり、判定を再現できなくなる（「[登場するもの][overview]」の進行役）。

2026-10-15 に、AI の読み取りを絞る形が動くかを実機で確かめた（補足情報の試行）。

- `denyRead` に `~/` と置き場を入れ、`allowRead` に自分のセッションのフォルダだけを入れると、狭い方の規則が勝ち、ホーム・進行役の clone・別のセッションは読めず、自分の clone と worktree は読めた
- 環境変数を足さないと、git と pnpm が動かない。git は個人設定（`~/.config/git/config`）、pnpm は `~/.npmrc` を読めずに止まる。
  個人設定を読まない環境変数を渡すと、`git commit`・`pnpm install`・`pnpm check:types`・`pnpm check:test` が通った
- Read ツールは、`allow` に `Read` を入れなければ、作業フォルダの外を `-p` で自動で拒否した

課題：AI が読める範囲を、列挙漏れが穴にならない形にできるか。秘密をどこに置き、どう進行役に渡すか。手動実行をどう起こすか。

## 判断の決め手（Decision Drivers）

- 列挙漏れが読み取りの穴にならないこと（許可を絞り、拒否は重ねる守りにする）
- 秘密の置き場が 1 か所に決まり、`check` が実際に試して確かめられること（SAFE-5）
- 手元の Mac で試す間も、人の鍵・会話の記録・別の案件を AI から隠すこと（SAFE-18）
- git・pnpm・`pnpm check` が、今と同じに動くこと
- 定期の実行と手動の実行が、同じ環境で動くこと

## 検討した選択肢（Considered Options）

問い 1：AI が読める範囲

- 案 1-A：自分のセッションのフォルダに絞る（ホームと置き場を読めなくし、自分のセッションのフォルダだけ開ける）
- 案 1-B：拒否を列挙する（今の要件の形のまま、一覧を増やす）
- 案 1-C：AI のセッションを別の OS ユーザーで動かす

問い 2：秘密の置き場

- 案 2-A：進行役だけが読むファイル（専用のフォルダ）
- 案 2-B：launchd の plist の環境変数（`EnvironmentVariables`）
- 案 2-C：macOS のキーチェーン
- 案 2-D：1Password などの秘密の管理サービス

問い 3：手動実行の起こし方

- 案 3-A：launchd のジョブを起こす（`launchctl kickstart`。`-k` を付けない）
- 案 3-B：人の Claude Code のセッションから進行役を直接起動する

## 決定（Decision Outcome）

採用した選択肢:「案 1-A」「案 2-A」「案 3-A」。
理由: 列挙漏れが穴にならず、2026-10-15 の試行で確かめた Claude Code の機能の範囲で動く。秘密は 1 か所にまとまり、`check` で確かめられる。定期の実行と手動の実行が同じ環境で動く。

### 決定 1：AI が読める範囲は、自分のセッションのフォルダに絞る

- **sandbox（Bash）**：`denyRead` に `~/`、置き場（[ADR-20261013-001][adr-workspace] の `<置き場>/`）、秘密の置き場（決定 2）を入れる。`allowRead` には自分のセッションのフォルダ（`<置き場>/sessions/<セッションの名前>/`）だけを入れる。
  範囲が重なるときは狭い方の規則が勝つので、ホーム・進行役の clone・別のセッション・秘密は読めず、自分の clone と worktree は読める（補足情報の試行）
  - 置き場と秘密の置き場は、ホームの中にあっても外にあっても `denyRead` に書く。置き場をホームの外（`/private/tmp`）に置いた試行では、`~/` の拒否だけでは進行役の clone と別のセッションを読めた
  - ホームの外の道具（Homebrew の `/opt/homebrew`、`/usr`）は読める。道具がホームの中にあるとき（nvm など）は、道具のフォルダを `allowRead` に足す。`setup` が `command -v` の実体のパスから決める
  - リンクの先がホームの中にあれば、`~/` の拒否で覆われる（試行では、`~/.config/gh` の下のファイルを、リンクの名前でも実体のパスでも読めなかった）。ホームの外を指すリンクは、今の規則（SAFE-5。実体に解決して書く）のまま
- **Read ツール**：`permissions.allow` から `Read` を外す。Read は作業フォルダ（worktree）の中だけになり、外を読もうとすると `-p` では自動で拒否され、`permission_denials` に記録される（補足情報の試行）。
  `deny` の `Read(//<秘密の置き場>/**)` と `Read(//<置き場>/controller/**)` は、重ねる守りとして残す
- **AI のセッションに渡す環境変数**：ホームを読めないと git と pnpm が個人設定を読めずに止まるので、進行役が次を渡す（子プロセスの環境変数は許可した名前だけで組み立てる。SAFE-1、FACT-17）
  - `GIT_CONFIG_GLOBAL=/dev/null`（git の個人設定を読まない）。commit の作者は `GIT_AUTHOR_NAME` などか `GIT_CONFIG_COUNT` で渡す
  - `NPM_CONFIG_USERCONFIG=/dev/null`（`~/.npmrc` を読まない）
  - `XDG_CONFIG_HOME` `XDG_CACHE_HOME` `XDG_DATA_HOME` `XDG_STATE_HOME` と `TMPDIR` を、セッションの一時フォルダ（`allowWrite` に入れる）に向ける
- **pnpm の store**：ホームの中の共有の store は読めなくなる。試行ではセッションの一時フォルダに空の store を作り、sandbox の中の `pnpm install` は 9 秒だった。
  共有の store を開けるかは UNDECIDED-25 で決める（開けるなら store を `allowRead` と `allowWrite` に足す）
- **置き場の場所**：置き場は `~/.claude/` の下に置かない。試行では、`~/.claude/` の下に置いた worktree に、`allowWrite` に足しても何も書けなかった
- `setup` が安全設定を書き、`check` が sandbox の中と Read ツールの両方から実際に試す（SAFE-5）

### 決定 2：秘密は進行役だけが読むファイルに置く

- PAT、印の署名の鍵（IMPL-20 の (c)）、deploy key を、秘密の置き場（設定の `paths.secrets`）のファイルに置く。フォルダの権限は 700、ファイルは 600
- 秘密の置き場は、置き場（`<置き場>/`）の外の、同期されないフォルダにする。手元の Mac では `~/.config` と `~/.ssh` が同期フォルダへのリンクなので、`~/.config` の下には置かない。
  `check` は、秘密の置き場の実体のパスが同期フォルダ（`~/Library/CloudStorage/`、`~/Library/Mobile Documents/`）の下に無いことを確かめる。既定のパスの値は UNDECIDED-17 で決める
- 進行役は起動したときに秘密のファイルを読み、値を自分のプロセスの中だけで持つ。`gh` を呼ぶときだけ `GH_TOKEN` を渡し、push と fetch のときだけ deploy key を `GIT_SSH_COMMAND` で渡す。AI のセッションには渡さない（SAFE-1）
- launchd の plist には秘密の値を書かない。plist は `setup` が書き `check` が読むジョブの定義で、秘密の値を混ぜると `setup` と `check` が秘密を扱うことになる
- 秘密の値は人が書く（SETUP-1 の「人がやるもの」）。`setup` はフォルダとファイルを権限付きで作り、書く欄を示す

### 決定 3：手動実行は launchd のジョブを起こす

- `/dev-autopilot` の Skill（人が呼ぶ入口）は、進行役を直接起動しない。`launchctl kickstart gui/<uid>/<ジョブの名前>` で定期の実行と同じジョブを起こし、実行の記録を読んで結果を表示する
- `-k` は付けない。`-k` は動いている進行役を殺してから起動し直すので、作業の途中のセッションを止めてしまう。`-k` なしなら、動いている間は何もしない。多重起動は lock でも防ぐ（「[運用の規則][flow]」）
- 人の Claude Code のセッションは秘密を扱わず、進行役の親にもならない。進行役は定期の実行と同じ環境で動く

### 影響（Consequences）

- 良い点: 列挙漏れが読み取りの穴にならない。新しい秘密や記録がホームに増えても、AI からは読めない
- 良い点: AI は別のセッションの作業場所も読めない。ADR-20261013-001 の書き込みの分離に、読み取りの分離が加わる
- 良い点: ホームの中を指すリンクも覆うので、`denyRead` がリンクを解決しない不具合の影響を受けにくい
- 良い点: 秘密は 1 か所にあり、`check` で権限・同期の有無・AI から読めないことを確かめられる
- 悪い点: AI のセッションに渡す環境変数が増える。新しく足した道具がホームの中のファイルを読もうとすると、EPERM で止まる（読めてしまうのではなく、止まって見える側に倒れる）。止まったら環境変数か `allowRead` を足す
- 悪い点: 共有の pnpm の store を開けない限り、セッションごとに依存を取り直す（sandbox の中で約 9 秒）
- 悪い点: 秘密のファイルは、同じ OS ユーザーのプロセスなら読める。sandbox の掛かっていない人の Claude Code のセッションからも読める。防ぐには案 1-C（別の OS ユーザー）が要る
- 悪い点: 手動実行の結果は、実行の記録を読んで知る形になる

### 確認方法（Confirmation）

- 段階 3（`setup` と `check` を作る段階）の `check`（SETUP-2）で、次を実際に試す
  - sandbox の中から、ホームのファイル・`<置き場>/controller/`・別のセッションのフォルダ・秘密の置き場を読めないこと。自分のセッションの clone と worktree は読めること
  - sandbox の中で、決定 1 の環境変数を渡して `git commit` と `pnpm install` が通ること
  - Read ツールで worktree の外を読もうとすると拒否され、`permission_denials` に記録されること
  - 秘密の置き場の権限（700・600）と、実体のパスが同期フォルダの下に無いこと。launchd の plist に秘密の値が無いこと
- 結合テストで、偽の `launchctl` を使い、`/dev-autopilot` が `kickstart` を `-k` なしで呼ぶことを検査する

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 1-A：自分のセッションのフォルダに絞る

- 良い点: 列挙漏れが穴にならない（読めないのが既定）
- 良い点: 別のセッションの作業場所も読めない
- 悪い点: git と pnpm の個人設定を読まない環境変数と、ホームの中の道具の `allowRead` が要る

### 案 1-B：拒否を列挙する

- 良い点: 今の要件のまま。git と pnpm は個人設定を読めるので、環境変数を足さずに動く
- 悪い点: 一覧に無いもの（秘密の置き場、別の案件の会話の記録、`~/.npmrc`、別のセッション）は読める。秘密や記録が増えるたびに一覧を直す
- 悪い点: パスの途中のリンクを解決しない不具合で、一覧に書いたパスが効かないことがある

### 案 1-C：AI のセッションを別の OS ユーザーで動かす

- 良い点: OS の権限で隔離するので最も強い。人の Claude Code のセッションからも秘密を隠せる
- 悪い点: 手元の Mac で運用しにくい（「[採らなかった案][records]」の隔離の案 B）。専用マシンに移るときの候補に残す

### 案 2-A：進行役だけが読むファイル

- 良い点: 置き場が 1 か所で、権限と同期の有無を `check` で確かめられる
- 良い点: 手元の Mac でも専用マシンでも同じ形で動く
- 悪い点: 同じ OS ユーザーのプロセスからは読める。AI のセッションは決定 1 で止める

### 案 2-B：launchd の plist の環境変数

- 良い点: 進行役が起動した時点で環境変数に入っている
- 悪い点: ジョブの定義と秘密が混ざり、`setup` と `check` が秘密の値を扱う
- 悪い点: `launchctl print` で同じユーザーのプロセスから見えるかもしれない（確かめていない）

### 案 2-C：macOS のキーチェーン

- 良い点: 暗号化して保管される
- 悪い点: sandbox の中の Bash から `security` コマンドで読めるかを確かめていない
- 悪い点: 無人の専用マシンでは、キーチェーンの鍵を開けたままにする運用が要る
- 見送り。専用マシンに移るときに確かめ直す

### 案 2-D：1Password などの秘密の管理サービス

- 悪い点: サービスに入るための鍵がマシンに要り、置き場の問いが残る
- 悪い点: 外への通信に頼る。人の署名も、1Password の agent から手元の鍵に移した（SETUP-10）
- 却下

### 案 3-A：launchd のジョブを起こす

- 良い点: 定期の実行と同じ環境で動く。人のセッションは秘密を扱わない
- 悪い点: 結果は実行の記録を読んで知る

### 案 3-B：人の Claude Code のセッションから直接起動する

- 良い点: 進行役の出力を実行中に見られる
- 悪い点: 進行役が人の環境（SSH の agent、`gh` のログイン）と人のセッションの設定（sandbox、権限）を受け継ぎ、定期の実行と振る舞いが変わる

## 補足情報（More Information）

- 関連する要件：SAFE-1、SAFE-2 の (e)、SAFE-5、SAFE-10、SAFE-15、SAFE-18（[05-safety.md][safety]）、IMPL-20 の (c)、STRUCT-2（[07-structure.md][struct]）、SETUP-1、SETUP-2、
  「[運用の規則][flow]」、FACT-13、FACT-17（[10-verified-facts.md][facts]）、UNDECIDED-17、UNDECIDED-25（[09-undecided.md][undecided]）、[ADR-20261013-001][adr-workspace]
- 2026-10-15 の試行（macOS、Claude Code 2.1.285、haiku、git 2.50.1、pnpm 10.29.3）。ADR-20261013-001 の形（`sessions/<名前>/clone` と `worktree`）を使い捨ての場所に作り、偽の秘密を置いて確かめた。
  `claude -p --setting-sources local --settings <安全設定>` を、許可した名前の環境変数だけで起動した。8 回で、見積もりの費用は計約 0.2 ドル
  - `denyRead: ["~/"]`、`allowRead` に自分のセッションのフォルダ：ホームのファイル（`.zshrc`、`.gitconfig`、`gh` の設定（リンクの名前と実体のパスの両方）、`.claude.json`、会話の記録、LaunchAgents）は読めなかった。自分の clone と worktree は読めた
  - 置き場がホームの外（`/private/tmp`）にあると、`~/` の拒否だけでは進行役の偽の秘密と別のセッションを読めた。`denyRead` に置き場の全体を足すと、進行役の偽の秘密、別のセッション、セッションの一覧は読めず、自分の clone と worktree は読め、`git commit` と `pnpm check:types` が通った
  - 環境変数を渡さないと、git は `~/.config/git/config` を読めず終了コード 128、pnpm は `~/.npmrc` を読めず終了コード 255 で止まった。
    `GIT_CONFIG_GLOBAL=/dev/null`、`NPM_CONFIG_USERCONFIG=/dev/null`、XDG の 4 つと `TMPDIR` をセッションの一時フォルダに向けると、`git commit`、`pnpm install`（空の store で 9 秒）、`pnpm check:types`、`pnpm check:test`（441 件）が通った。親 clone の `.git/hooks` と `.git/config` には書けなかった（FACT-9 と同じ）
  - 試験の場所を `~/.claude/` の下に置くと、`allowWrite` に足しても、worktree・`node_modules`・親 clone の `.git` に書けなかった（Operation not permitted）。ホームの読み取りの拒否を外しても同じだった。場所を `/private/tmp` に移すと書けた
  - Read ツール：`allow` に `Read` が無いと、worktree の外のファイル（偽の秘密、`~/.zshrc`）は「Claude requested permissions to read from this path, but you haven't granted it yet」で拒否され、`permission_denials` に記録された。worktree の中は読めた
  - Grep と Glob の道具は、2.1.285 の `-p` のセッションの道具の一覧に無かった
- 確かめていないこと：sandbox の中からキーチェーンの項目を読めるか、plist の環境変数が `launchctl print` で見えるか。どちらも、試験の操作（試験用のキーチェーンの項目を作る、`launchctl print` で環境変数の欄を探す）を作業の環境の自動の許可判定が止めたため、試していない。どちらの案も採らないので、決定には影響しない
- 範囲外：Grep と Glob が 2.1.285 の道具の一覧に無い。計画役は「Read・Grep・Glob」（「[登場するもの][overview]」）で動く前提なので、エージェント定義の `tools` に書いたときに使えるかを確かめる。要件に反映するときに、[決めていないこと][undecided]の「確かめること」に足す
- 範囲外：共有の pnpm の store を開けるか（UNDECIDED-25）
- 要件への反映（承認の後に、別の PR で行う）
  - SAFE-2 の (e)：`allow` から `Read` を外し、Read は worktree の中だけにする。`deny` の Read は、重ねる守りとして秘密の置き場と `<置き場>/controller/` に書く
  - SAFE-10：`denyRead` を「`~/`・置き場・秘密の置き場」、`allowRead` を「自分のセッションのフォルダ（と、ホームの中の道具）」に改める。AI のセッションに渡す環境変数（決定 1）と、置き場を `~/.claude/` の下に置かないことを書き足す
  - SAFE-5：ホームの中を指すリンクは `~/` の拒否で覆われることを書き足す
  - SAFE-15、IMPL-20 の (c)：秘密の置き場（決定 2）を書き足す
  - STRUCT-2：設定の項目に `paths.secrets`、`allowRead` に足す道具のフォルダ、子プロセスに渡す環境変数の名前を足す
  - SETUP-1、SETUP-2：`setup` が秘密の置き場を権限付きで作ること、`check` が決定 1 と決定 2 を試すことを足す
  - 「運用の規則」と「分離を見越した構成」の `/dev-autopilot`：手動実行は `launchctl kickstart`（`-k` なし）で起こすことを書き足す
  - 10-verified-facts.md：2026-10-15 の試行を FACT-22（読み取りの絞り込みと環境変数）、FACT-23（`~/.claude/` の下には書けない）、FACT-24（Grep と Glob が道具の一覧に無い）として足す
  - 09-undecided.md：UNDECIDED-16 を欠番にする（元の文を取り消し線で残す）。Grep と Glob の論点を「確かめること」に足す

[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[struct]: ../requirements/07-structure.md
[overview]: ../requirements/00-overview.md
[flow]: ../requirements/01-issue-flow.md
[records]: ../requirements/11-records.md
[safety]: ../requirements/05-safety.md
[adr-workspace]: ./adr-20261013-001-fresh-clone-per-ai-session.md
[adr-move]: ./adr-20261016-003-move-to-own-repository-before-basic-design.md
[adr-plugin]: ./adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
