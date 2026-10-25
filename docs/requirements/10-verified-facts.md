# 実機で確かめた事実

## 実機の試し（段階 2）で確かめた事実（2026-10-08〜11）

macOS、Claude Code 2.1.285、haiku、`claude -p --output-format json` で、business-os のローカル clone（未信頼の作業場所）から確かめた。入れ子の Plugin は `tmp/` に作った偽物（名札・agent 1 体・Skill 1 つ）。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-1 | 未信頼の作業場所で `.claude/settings.json` の `permissions.deny` が効くか | **効く**。deny に足した `Bash(echo DENIED_PROBE:*)` が拒否され、`permission_denials` に記録された。許可した方は通った | 実行 |
| FACT-2 | `claude -p --plugin-dir <サブフォルダ> --agent <plugin>:<agent>` が入れ子の Plugin で動くか。agent から Plugin の Skill を名前空間で呼べるか | **動く**。`dev-autopilot:worker` が `dev-autopilot:probe` を呼んで印を返した。外側の Plugin と同時に読み込んでも（`--plugin-dir` 2 つ）両方の Skill を呼べた。`claude plugin validate` は外側・入れ子の両方で通る（author の warn のみ） | 実行 |
| FACT-3 | `-p` のセッションからサブエージェントを起こせるか | **起こせる**。Agent ツールで haiku のサブエージェントが返事を返した | 実行 |
| FACT-4 | `--max-budget-usd` が定額のプランで効くか | **効く**。0.0001 ドルで 1 ターン後に `error_max_budget_usd` で止まり、`total_cost_usd` に見積もりの費用（0.0377）が入る | 実行 |
| FACT-5 | sandbox を掛けた `claude -p` で `pnpm check:types` と `git commit`（署名なし）が通るか | **通る**。`~/.ssh` の読み取りは `Operation not permitted` で止まった | 実行 |
| FACT-6 | fine-grained PAT で Organization 所有の Project 7 の欄を書けるか | **書ける**。Resource owner を Organization、Repository access を business-os だけ、Organization permissions に Projects の Read and write を付けた fine-grained PAT で、`gh api graphql` による Project 7 の読み取り、`addProjectV2DraftIssue`、`deleteProjectV2Item` が通った（2026-10-11、人が発行して実行）。代替（GitHub App、classic PAT）は不要 | 実行 |
| FACT-7 | ruleset を develop に写すときの承認の設定 | `protect-main` は `required_approving_review_count: 0`、`require_extra_approval_for_unattributed_changes: true`。develop の ruleset では後者を無効にする（自動化アカウントを足したときに自己承認できなくなる） | 読んだ（API） |
| FACT-8 | `claude -p --model haiku` の別名が最新の Haiku に解けるか。`--effort` が効くか。エージェント定義の `model` と `--model` の優先。`modelUsage` で実際のモデルが取れるか | **別名 `haiku` は Haiku 4.5（`claude-haiku-4-5-20251001`）に解けた。** 完全な ID `claude-haiku-5-5` は動く（`modelUsage` に `claude-haiku-5-5`。stderr に `unrecognized_model` の記録が出るが実行は成功）。`--model` はエージェント定義の `model` より優先。定義だけなら定義の値（`sonnet` → `claude-sonnet-5-5`）。`--effort low` は受理された。`modelUsage` は使われたモデル ID をキーに持つ（2026-10-11、Claude Code 2.1.285） | 実行 |
| FACT-9 | worktree で `git commit` が通るか。git が書く場所。親 `.git` の保護 | **通る**。作業フォルダが worktree のとき sandbox は親 `.git` への書き込みを既定で許し、`.git/config` `.git/hooks/**` `.git/worktrees/<名前>/commondir` `config.worktree` と worktree の `.claude/settings.json` は保護パスとして Bash から書けない（`Operation not permitted`）。**`.git/worktrees/<名前>/gitdir` と `.git/refs/**` は書けた**（gitdir は `denyWrite` に足す）。commit で新しく書かれたのは `objects/**` と `worktrees/<名前>/COMMIT_EDITMSG`（HEAD・index・logs は更新）。**Bash を `permissions.allow` に入れないと `git commit` は承認待ちになり `-p` では拒否される**（sandbox の自動許可の対象外。変数展開を含むコマンドも同じ）。allow に入れれば通る | 実行 |
| FACT-10 | sandbox の中で `pnpm install` と `pnpm check` 全体が通るか | **通る**。`network.allowedDomains` に `registry.npmjs.org` を入れた sandbox で `pnpm install --frozen-lockfile --prefer-offline` が成功（`prepare` の simple-git-hooks は親 `.git/hooks` に書けず EPERM で失敗するが install は exit 0。hook が仕込まれないので望ましい）。`pnpm check` は入れ子の `claude plugin validate`（check:plugin）と gitleaks（check:leak）を含め全項目 Done、vitest 441 件 pass。入れ子の `claude` からの datadoghq / api.anthropic.com への通信は sandbox が拒否した（動作に影響なし） | 実行 |
| FACT-11 | プロジェクト設定の hook と `CLAUDE.md` が読まれない起動の方法 | **既定の `-p` は未信頼の作業場所でもプロジェクトの SessionStart hook を実行し、`CLAUDE.md` を読む**（攻撃の経路が実在することを確認）。`--setting-sources user` でプロジェクトの hook と `CLAUDE.md` の両方が読まれなくなる。`--setting-sources local` ではユーザー設定の hook も消える（SessionStart の hook が 5 → 0。`--settings` のファイルは効く）。空文字は不正。`--bare` は `--plugin-dir` の agent を読まないので使えない | 実行 |
| FACT-12 | `commondir` を差し替えた worktree に対する進行 Bot の操作 | **worktree の中で git を実行すると発火した**（`git -C <worktree> status` で差し替え先の `core.fsmonitor` が動いた）。進行 Bot の clone から worktree のパスへの fetch、親 clone のパスへの fetch、親 clone での `git worktree list` `git worktree remove --force`、親 clone での `git log <ブランチ>` は**発火しなかった**。SAFE-2 の (b) のとおり進行 Bot が worktree の中で git を実行しなければ届かない | 実行 |
| FACT-13 | sandbox の `denyRead` が Read ツールにも効くか | **効かない**（公式文書どおり。sandbox は Bash だけを覆う）。Read ツールは `denyRead` のパスを読めた。`permissions.deny` の `Read(//<絶対パス>/**)`（絶対パスは `//` 始まり。`/` 始まりはプロジェクト相対で一致しない）で止まった（`File is in a directory that is denied by your permission settings`）。Write・Edit も同じで、ホームの任意のパスに書けた。`permissions.allow` を `Write(//<worktree>/**)` `Edit(//<worktree>/**)` に絞ると、外への Write は「Path is outside allowed working directories」で拒否され、`deny` に入れた `.git/**` `.claude/**` `CLAUDE.md` への Edit も止まった。`.git/hooks/**` と `.mcp.json` は Write ツールが「sensitive file」として既定で拒む | 実行 |
| FACT-14 | 道具なし、`--json-schema`、強制終了時の費用 | 道具なしはエージェント定義の `tools: []` でも `--tools ""` でも成立（Read を頼んでも「道具が無い」と返す）。**`--json-schema` は `--agent` と併用すると `structured_output` が null になる**（道具の有無を問わず。`--agent` 無しなら `structured_output` に検証済みの JSON が入る）。進行 Bot は `result` の文を自分の JSON スキーマで検証する（IMPL-5）。`--output-format stream-json` は `assistant` のイベントごとに `model` と `usage` を持つ~~ので、途中で止めても消費は集計できる~~（FACT-29 で、`usage` が同じメッセージで重複し、`output_tokens` が最後の値でないと分かった。途中で止めた回の数え方は UNDECIDED-47 で扱い、[ADR-20261018-001][adr-usage] で決めた）。プロセスグループごとの kill（`setpgrp` で起動し `kill -TERM -- -<pgid>`）で孫プロセスも残らなかった~~（モデルが長いコマンドを実行しなかったため、`claude` 自体を途中で止める試験は未実施）~~（FACT-38 で、作業中の `claude` を止める試験をした。Bash のコマンドは `claude` と別のプロセスグループで動き、`SIGTERM` を無視したコマンドと `SIGKILL` の後のコマンドは残る） | 実行（kill は model なし） |
| FACT-15 | Project v2 の Status に選択肢を足す GraphQL が、既存の選択肢の ID と項目の値を保つか | **`updateProjectV2Field` の `singleSelectOptions` に既存の選択肢の `id` を含めて渡せば、既存の ID と項目の値は保たれ、同じ入力を 2 回送っても ID は変わらない（冪等）。`id` を渡さないと既存の ID がすべて振り直され、項目の Status の値も消える**（2026-10-11、Organization に作った捨てる Project で確認し、削除した）。`setup` は必ず現在の選択肢を読んで `id` 付きで送り、送る前に件数と名前を読み戻して照合する | 実行 |
| FACT-16 | リアクションや非表示で `updated_at` が変わるか | **リアクションでは変わらない。非表示（`minimizeComment`）では `updated_at` が変わる**が `lastEditedAt` は null のまま、`userContentEdits.totalCount` は 0。IMPL-20 の (b) は `created_at` と `updated_at` の一致でなく、GraphQL の `lastEditedAt == null`（または `userContentEdits.totalCount == 0`）で判定する | 実行 |
| FACT-17 | 子プロセスの環境変数を絞る。`GIT_CONFIG_COUNT` | `env -i` に `HOME` `PATH` `USER` `LOGNAME` `TMPDIR` `TERM` `LANG` だけを渡せば `claude -p` はログイン状態を保って動き、渡さなかった変数は Bash から `unset` に見えた（`HOME` と `PATH` だけではログインが失われる）。`GIT_CONFIG_COUNT` と `GIT_CONFIG_KEY_n` / `GIT_CONFIG_VALUE_n` で `core.hooksPath` と `gc.auto` が効いた | 実行 |

**見つかった不具合（🔴）：sandbox の `denyRead` はシンボリックリンクを解決しない。**
この Mac では `~/.config` が 同期フォルダ上の dotfiles へのシンボリックリンクで、`denyRead` に `~/.config/gh/**` と書いても `~/.config/gh/hosts.yml` が読めた（`~/.config/gh` と glob なしでも同じ）。
実体のパス（`<同期フォルダ>/dotfiles/.config/gh/**`）を書くと止まった。`~/.ssh` も同じ dotfiles へのリンクだが、こちらは `~/.ssh/**` で止まった（リンクがパターンの先頭にあるときは解決されるように見える。途中にあると解決されない）。

- dev-autopilot への反映：`setup` が安全設定を書くとき、`denyRead` と `denyWrite` の各パスを実体に解決してから書く（リンクの場合は両方のパスを書く）。
  `check` は「設定に書いてある」ではなく「sandbox の中で実際に読めないこと」を、読み取りを試して確かめる（fail-open を機械で検出する）
- business-os への反映：`plugin/templates/settings.json.tmpl` の `denyRead` の `~/.config/gh/**` `~/.aws/**` が、`~/.config` をリンクにしている利用者では効かない。
  利用者向けの不具合なので business-os の Issue [#80][bos-issue-80] に起票した（2026-10-08）
- 人への注意：`~/.ssh` の実体がクラウドの同期フォルダにある。dev-autopilot 用の鍵も人の署名鍵も、同期されないパスに置く

FACT-6〜17 は 2026-10-11 に、手で書いた安全設定と偽の入れ子 Plugin（scratchpad の実験場。親 clone と進行 Bot の clone と worktree 2 つ）と捨てる Project と、人が発行した PAT で実機確認した。実機の試し（段階 2）の未確認は残っていない。
「`setup` が書いた安全設定で実体のパスの解決と読み取りの検査が動くこと」は導入と点検の実装（段階 3）の出口。

## 要件の見直しの途中で確かめた事実（2026-10-12）

使い捨てのリポジトリ（macOS、git 2.50.1）で確かめた。Claude Code と sandbox は使わず、AI による書き換えは、人の手で objects のファイルを書き換えて再現した。
sandbox の中の AI が既存の objects のファイルを書き換えられるかは確かめていない（FACT-9 では、commit で新しい objects を書けた）。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-18 | clone の objects のファイルを書き換えたとき、同じ clone から作る worktree、`git fetch origin`、別の clone への送り出しがどうなるか | **worktree には書き換えた中身がエラーなしで出た**（`git worktree add --detach <SHA>` が終了コード 0）。`git fsck` は hash-path mismatch を出した。同じ clone で `git fetch --no-tags origin` を実行しても直らなかった（手元にある objects は取り直さない）。origin から新しく clone して同じ commit で worktree を作ると、正しい中身が出た。書き換えた clone から `file://` で別の場所へ clone すると、受け取り側の検査で `did not receive expected object` になり失敗した | 実行（sandbox なし） |
| FACT-19 | 手元のパスから `git clone` したとき、objects のファイルを共有するか | **パスから clone すると、ハードリンクで共有した**（objects のファイルのリンク数 2）。`--no-hardlinks` を付けたときと、`file://` の URL から clone したときは共有しなかった（リンク数 1） | 実行（sandbox なし） |

## 作業場所の ADR を起票する途中で確かめた事実（2026-10-13）

使い捨てのリポジトリと business-os の clone（macOS、git 2.50.1、pnpm 10.29.3）で確かめた。Claude Code と sandbox は使わず、AI による書き換えは、人の手で再現した。
sandbox の中の AI が親 clone の `.git/commondir` を新しく作れるかは確かめていない（FACT-9 で保護を確かめたのは worktree の `.git/worktrees/<名前>/commondir`）。
2 つの事実は、[ADR-20261013-001][adr-workspace]（AI の作業場所）の材料にした。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-20 | 普通の clone（worktree でない）の `.git/` に `commondir` のファイルを置いたとき、git がどう振る舞うか | **git は `commondir` が指すフォルダを共通のフォルダとして使い、指す先の `config` を読む。** `.git/commondir` に別のフォルダのパスを書き、別のフォルダの `config` に `core.fsmonitor` のコマンドを書くと、clone で `git fetch origin` を実行したとき（終了コード 0）にコマンドが実行された。`git rev-parse --git-common-dir` は別のフォルダを返した | 実行（sandbox なし） |
| FACT-21 | AI のセッションごとに作業場所を作り直す費用 | business-os を GitHub の URL から `git clone --no-checkout --no-tags` で約 3.0 秒、`git worktree add --detach` で 0.04 秒。`pnpm install --frozen-lockfile --ignore-scripts` は、手元の共有の store で 1.1 秒、空の store で 3.0 秒（`node_modules` は 137 MB）。clone の objects にリンク数 2 以上のファイルは無かった。手元の Mac（APFS）では、store と `node_modules` のファイルは共有しなかった（リンク数 1）。sandbox の中（通信は許可した先だけ）では計測していない | 実行（sandbox なし） |

## 読み取りの範囲の ADR を起票する途中で確かめた事実（2026-10-15）

macOS、Claude Code 2.1.285、haiku、git 2.50.1、pnpm 10.29.3 で確かめた。[ADR-20261013-001][adr-workspace] の形（`sessions/<名前>/clone` と `worktree`）を使い捨ての場所に作り、偽の秘密を置いた。
`claude -p --setting-sources local --settings <安全設定>` を、許可した名前の環境変数だけで起動した。3 つの事実は、[ADR-20261015-001][adr-read]（AI の読み取り範囲と秘密の置き場）の材料にした。
sandbox の中からキーチェーンの項目を読めるか、launchd の plist の環境変数が `launchctl print` で見えるかは、試験の操作を作業の環境の自動の許可判定が止めたため、確かめていない。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-22 | AI の読み取りを、自分のセッションのフォルダに絞れるか | **絞れる。** sandbox の `denyRead` に `~/` と置き場の全体、`allowRead` に自分のセッションのフォルダを入れると、狭い方の規則が勝った。ホームのファイル（`.zshrc`、`.gitconfig`、`gh` の設定（リンクの名前と実体のパスの両方）、`.claude.json`、会話の記録、LaunchAgents）、進行 Bot の偽の秘密、別のセッション、セッションの一覧は読めず、自分の clone と worktree は読めた。置き場がホームの外（`/private/tmp`）にあると、`~/` の拒否だけでは進行 Bot の偽の秘密と別のセッションを読めた。**環境変数を渡さないと git と pnpm が止まる**：git は `~/.config/git/config` を読めず終了コード 128、pnpm は `~/.npmrc` を読めず終了コード 255。`GIT_CONFIG_GLOBAL=/dev/null`、`NPM_CONFIG_USERCONFIG=/dev/null`、XDG の 4 つと `TMPDIR` をセッションの一時フォルダに向けると、`git commit`、`pnpm install`（空の store で 9 秒）、`pnpm check:types`、`pnpm check:test`（441 件）が通った。親 clone の `.git/hooks` と `.git/config` には書けなかった。Read ツールは、`permissions.allow` に `Read` が無いと、worktree の外のファイルを「Claude requested permissions to read from this path, but you haven't granted it yet」で拒否し、`permission_denials` に記録した。worktree の中は読めた | 実行 |
| FACT-23 | `~/.claude/` の下に置いた作業場所に、sandbox の中から書けるか | **書けない。** `~/.claude/` の下に置いた worktree・`node_modules`・親 clone の `.git` に、`allowWrite` に足しても、ホームの読み取りの拒否を外しても、Operation not permitted で書けなかった。同じ形を `/private/tmp` に移すと書けた | 実行 |
| FACT-24 | Grep と Glob の道具が `-p` のセッションで使えるか | **道具の一覧に無かった。** 道具を絞らない `claude -p` のセッションの道具の一覧に、Grep と Glob が無かった（Read・Write・Edit・Bash などはあった）。~~エージェント定義の `tools` に書いたときに使えるかは確かめていない~~ FACT-26 で確かめた（使える） | 実行 |

## 依存の store の ADR を起票する途中で確かめた事実（2026-10-15）

macOS、pnpm 10.29.3 で確かめた。sandbox とモデルは使っていない。[ADR-20261015-002][adr-store]（依存の store を共有しない）の材料にした。
pnpm の公式文書（[Store の設定][pnpm-store]）は、store の既定の場所を、`$PNPM_HOME` があれば `$PNPM_HOME/store`、`$XDG_DATA_HOME` があれば `$XDG_DATA_HOME/pnpm/store` と書いている。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-25 | AI のセッションと同じ環境変数で、pnpm の store がどこになるか | **セッションの一時フォルダの下になる。** 許可した名前の環境変数だけで、`XDG_DATA_HOME` を一時フォルダに向けて `pnpm store path` を実行すると、`<一時フォルダ>/pnpm/store/v10` を返した。`PNPM_HOME` を足すと `<PNPM_HOME>/store/v10` を返した（`PNPM_HOME` が優先される）。人の既定の store（`~/Library/pnpm/store/v10`）は変わらなかった | 実行（sandbox とモデルなし） |

## 確かめることを片付ける途中で確かめた事実（2026-10-16）

macOS、Claude Code 2.1.285、haiku で確かめた。試験用の Plugin（エージェント 1 つ、`tools: Read, Grep, Glob`）を `--plugin-dir` と `--agent` で起動し、`claude -p --setting-sources local --settings <安全設定>` を、許可した名前の環境変数だけで起動した。`permissions.allow` に `Read` は入れていない。UNDECIDED-28 を確かめるために行った。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-26 | エージェント定義の `tools` に Grep と Glob を書いたときに使えるか。作業フォルダの外を読めるか | **使える。** `--output-format stream-json` の初期化の記録で、セッションの道具の一覧が `Read` `Grep` `Glob` になり、3 つとも作業フォルダの中で使えた。FACT-24 で道具の一覧に無かったのは、道具を指定しない既定のセッションの一覧。**作業フォルダの外は、Read と同じく `-p` で自動で拒否された**：Grep と Glob で、作業フォルダの外のフォルダ、ホーム、`~/.zshrc` を試すと、「Claude requested permissions to read from …, but you haven't granted it」で拒否され、`permission_denials` に記録された | 実行 |

## Project の欄の値を確かめた事実（2026-10-16）

Organization に作った非公開の捨てるリポ（Issue 2 件）と、ロードマップの Project を `copyProjectV2` で複製した捨てる Project（組み込みのワークフローも複製された）で確かめた。
値を書き換えたのは、人のトークンの GraphQL（`updateProjectV2ItemFieldValue` と `clearProjectV2ItemFieldValue`）と、Project の組み込みのワークフロー（Item added to project が `Todo`、Item closed が `Done` を入れる）の 2 者。
UNDECIDED-29 を確かめるために行い、試験の後に捨てる Project を削除した（捨てるリポの削除はメンテナが行う）。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-27 | Project の単一選択の欄の値（`ProjectV2ItemFieldSingleSelectValue`）の `creator` が、値を最後に変えた人を指すか | **指さない。`creator` は値を最初に入れた人。** ワークフローが入れた Status を人が変えても、値の `id`・`creator`・`createdAt` は変わらず、`updatedAt` だけが進んだ。人が入れた値をワークフローが `Done` に変えても、`creator` は人のままだった。値を消すと値そのものが消え、入れ直すと新しい `id` の値ができて、`creator` は入れ直した人になった。値を最後に変えた人は、値からは取れない | 実行 |
| FACT-28 | Issue のタイムラインの Status の変更のイベント（`ProjectV2ItemStatusChangedEvent`）で、値を最後に変えた人を取れるか | **取れるのは Status だけで、抜けがあるため正本にできない。** イベントは `actor`・`previousStatus`・`status` を持ち、試験では Status の 4 回の変更すべてに正しい `actor` で、1〜3 秒遅れて出た。一方、独自の単一選択の欄（Agent 欄に当たる）を変えてもイベントは出なかった。ロードマップの Project（19 項目）を読むと、値の `updatedAt` が進んだのにイベントが 1 件も無い項目があった（business-os の [#37][bos-issue-37]・[#44][bos-issue-44]・[#45][bos-pr-45] など）。ワークフローによる変更でも `wasAutomated` は `false` だった | 実行 |

## サブエージェントの消費の出方を調べる途中で確かめた事実（2026-10-17）

macOS、Claude Code 2.1.285 で確かめた。[ADR-20261017-002][adr-review]（レビュー AI の入力と道具）の材料として、サブエージェントの消費が `stream-json` の記録にどう出るかを調べた。
試験用の Plugin（親のエージェントは `tools: Agent, Read` で、サブエージェントを 1 つ起こす。サブエージェントは `tools: Read` でファイルを 1 つ読む）を、
`claude -p --setting-sources local --plugin-dir <Plugin> --agent <親> --model claude-haiku-5-5 --output-format stream-json --verbose` で 3 回起動した。2 回目は `--forward-subagent-text`、3 回目は `--include-partial-messages` を足した。
サブエージェントの `model` は、1 回目と 2 回目が `haiku`、3 回目が `sonnet`。sandbox は掛けず、環境変数は絞っていない。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-29 | `stream-json` の `assistant` のイベントの `usage` を足し上げて、セッションの消費を数えられるか | **数えられない。** 1 つのメッセージ（同じ `message.id`）が中身の区切り（thinking・text）ごとに別のイベントに分かれ、どのイベントも同じ `usage` を持つ（そのまま足すと、入力とキャッシュを重ねて数える）。`usage.output_tokens` はストリームの始めの値（`stop_reason` が null）で、最後の値ではない。3 回目の親のメッセージ 2 つでは、イベントの `output_tokens` の和が 25、最後の `result` の `modelUsage` が 485 だった。`--include-partial-messages` を付けると `stream_event` の `message_delta` が出て、親の `output_tokens` の和は 485 で `modelUsage` と一致した。サブエージェントのメッセージは `stream_event` として出なかった。最後の `result` の `modelUsage` と `total_cost_usd` は、サブエージェントの消費を含めたセッション全体の値だった。~~途中で止めたときに `message_delta` が出るかは確かめていない（UNDECIDED-44）~~（FACT-39 で確かめた。書き終えたメッセージには出て、書いている途中のメッセージには出ない） | 実行 |
| FACT-30 | `--setting-sources local` で起動したセッションに、ログインしている claude.ai のアカウントのコネクタが入るか | **入った回があった。** 3 回のうち 2 回（1 回目と 2 回目）、初期化の記録の `mcp_servers` に claude.ai のコネクタが 4 つ出た（`source` は `claudeai`。メールと文書の 2 つが `connected`、残りの 2 つが `needs-auth`）。3 回目は `mcp_servers` が空だった。親の道具はエージェント定義の `tools` で `Task` と `Read` に絞っていたので、どの回もコネクタの道具は道具の一覧に無かった。認証が要るコネクタの案内の文が、2 回（2 回目と 3 回目）、親の最後の返事に混ざった。公式文書（[MCP][doc-mcp]）は、Claude Code が claude.ai から取ってくるコネクタを止める手段として、設定の `disableClaudeAiConnectors`、環境変数 `ENABLE_CLAUDEAI_MCP_SERVERS=false`、`--strict-mcp-config` を挙げている。~~dev-autopilot の起動の形（`--setting-sources local` と `--settings` の安全設定、許可した名前だけの環境変数）で効くかは確かめていない（UNDECIDED-48）~~（FACT-35 で、dev-autopilot の起動の形で 3 つとも効くと確かめた） | 実行 |
| FACT-31 | サブエージェントの消費とモデルが、`stream-json` の記録にどう出るか。サブエージェントはどう動くか | **最後の `result` にだけ揃って出る。** `modelUsage` は、サブエージェントのモデルが親と違えば別の項目に出た（3 回目：親の `claude-haiku-5-5` とサブエージェントの `claude-sonnet-5-5`）。サブエージェントの `model` に親と同じ系統の別名（`haiku`）を書くと、親のモデル（`claude-haiku-5-5`）で動き、同じ項目に足された（公式文書のモデルの決まり方のとおり）。途中の記録では数え切れない：サブエージェントの `assistant` のイベントは、バックグラウンドで動いた回（1 回目と 2 回目）はすべてのメッセージが `parent_tool_use_id` 付きで出たが、前で動いた回（3 回目。`--forward-subagent-text` なし）は 2 つのメッセージのうち道具を呼んだ 1 つだけが出た。`task_progress` と `task_notification` は `total_tokens`・`tool_uses`・`duration_ms` だけを持ち、モデルと入力・出力の内訳を持たない。サブエージェントのメッセージは `stream_event` として出ない（FACT-29）。サブエージェントをバックグラウンドで動かすかは、同じ指示でも回ごとに違った。バックグラウンドの回は、親がサブエージェントの答えを待たずにターンを終え、`result` が 2 つ出た（1 つ目は「サブエージェントはまだ動いている」という返事）。プロセスは 2 つ目の `result` の後に終わった。初期化の記録の道具の一覧では、Agent ツールは `Task` という名前で出た | 実行 |

## 写しの Plugin の読み取りとコネクタの止め方を確かめた事実（2026-10-17）

macOS、Claude Code 2.1.285、`claude-haiku-5-5` で確かめた。[ADR-20261017-004][adr-human]（写しの Plugin の読み取りと人が呼ぶ Skill の認証）と [ADR-20261017-005][adr-mcp]（AI のセッションに MCP のサーバーを読み込ませない）の材料として調べた。
AI のセッションと同じ起動の形（`claude -p --setting-sources local --settings <安全設定（sandbox を有効）> --plugin-dir <Plugin> --agent <試験用> --output-format stream-json`。環境変数は許可した名前だけで組み立てた）で、worktree から起動した。
試験の場所は、断りが無ければ Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）の外。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-32 | 進行 Bot のプログラムの写しを `--plugin-dir` で渡したとき、AI のセッションが写しの Skill の参照ファイルとスクリプトを使えるか | **今の要件の安全設定では使えず、写しの `skills/` だけを読み取りの許可に足すと使える。** 置き場の中に写し（`program/0.0.1/`。エージェント定義、Skill、Skill の参照ファイルとスクリプト、`src/`）と、セッションのフォルダの worktree を作り、試験の Skill は参照ファイルとスクリプトを `${CLAUDE_SKILL_DIR}` で指した。今の要件の安全設定（`denyRead` に `~/` と置き場、`allowRead` に自分のセッションのフォルダ、`deny` に写しの Read）では、エージェント定義は読み込まれ、Skill を呼べた（`SKILL.md` の本文は Claude Code が読み込んだ）。一方、参照ファイルの Read は「File is in a directory that is denied by your permission settings」で拒否され、`node <写し>/skills/.../scripts/probe.mjs` は `Cannot find module` で失敗した。`allowRead` と `permissions.allow` の Read に写しの `skills/` を足し、`deny` の Read を写しの `src/` に狭めると、参照ファイルを Read で読め、スクリプトを Bash で実行できた。どちらの設定でも、`src/` は Bash でも Read でも読めなかった。写しの `skills/` への Bash の追記は `operation not permitted`、Write と Edit は「sensitive file」として拒否された。最初に試験の場所を `/tmp/claude-501/` の下に置いた回は、写しの `skills/` への Bash の追記が通った（Write と Edit は拒否された） | 実行 |
| FACT-33 | skills CLI が `plugin/skills/` の下の Skill を見つけるか | **見つける。** skills CLI 1.7.2 の `skills add <フォルダ> --list` は、試験用のフォルダの `plugin/skills/review/` と `plugin/skills/setup/` の 2 つを見つけた。`skills add anthropics/claude-code --list` は、リポの中の `SKILL.md` 10 個（`plugins/<名前>/skills/<名前>/SKILL.md` の深さを含む）をすべて見つけた。dev-autopilot のリポを指して試すのは、`plugin/skills/` を作った後になる | 実行 |
| FACT-34 | sandbox の中のコマンドの `$TMPDIR` は、進行 Bot が渡した値になるか | **ならない。** FACT-32 の試験で、子プロセスの環境変数の `TMPDIR` にセッションの一時フォルダを渡したが、sandbox の中のコマンドの `$TMPDIR` は `/tmp/claude-501` だった。公式文書（[sandbox][doc-sandboxing]、[環境変数][doc-env]）は、sandbox の中のコマンドが作業フォルダとユーザーごとの一時フォルダに既定で書けること、Claude Code が sandbox の中のコマンドの `$TMPDIR` をユーザーごとの一時フォルダに置き換えることを書いている。ユーザーごとの一時フォルダは同じユーザーで動くすべての AI のセッションから書ける（FACT-32 の `/tmp/claude-501/` の下の写しへの追記）。一時フォルダの場所は `CLAUDE_CODE_TMPDIR` で変えられ、Claude Code が後ろに `/claude-<uid>/` を足す（パスが長いと、sandbox の中のコマンドには既定の場所の下の短いパスが渡る）。~~セッションごとに `CLAUDE_CODE_TMPDIR` を変えたときの振る舞いは確かめていない（UNDECIDED-50）~~（FACT-37 で確かめた。`<渡した場所>/claude-<uid>` が 44 文字までなら、渡した場所が使われる） | 実行と公式文書 |
| FACT-35 | claude.ai のコネクタが AI のセッションの道具に入るか。止め方が効くか | **止める仕組みが無いと入り、4 つの止め方はどれも効いた。** `tools` を書かないエージェント定義の試験用の Plugin を `--verbose` を足して起動し、初期化の記録の道具の一覧と MCP のサーバーの一覧を読んだ（道具は呼ばせていない）。止める仕組みなし（2 回）：MCP のサーバーは claude.ai の文書とメールが `connected`、ほかの 2 つが `needs-auth`。道具 63 個のうち 38 個が MCP の道具で、メールの道具 30 個に送信（`send_message`）・転送（`forward`）・返信（`reply`）・ゴミ箱への移動（`trash_message`）などがあった。安全設定の `disableClaudeAiConnectors: true`（2 回）、環境変数 `ENABLE_CLAUDEAI_MCP_SERVERS=false`（2 回）、`--strict-mcp-config`（2 回）：どれも MCP のサーバーの一覧が空で、MCP の道具は 0 個（道具 25 個）。安全設定の `permissions.deny` に `mcp__*`（1 回）：MCP のサーバーは `connected` のままで、MCP の道具は 0 個（道具 25 個）。MCP の道具を、`-p` で許可の規則に無いまま呼んだときの扱いは、メールを送る道具を呼ぶことになるので試していない | 実行 |

## 一時フォルダの ADR を起票する途中で確かめた事実（2026-10-17）

macOS、Claude Code 2.1.285、`claude-haiku-5-5`、uid 501 で確かめた。[ADR-20261017-006][adr-tmp]（セッションごとの短い一時フォルダ）の材料として調べた。
AI のセッションと同じ起動の形（`claude -p --setting-sources local --settings <安全設定（sandbox を有効、allowUnsandboxedCommands は false）> --strict-mcp-config --output-format stream-json`。環境変数は許可した名前だけで組み立てた）で起動し、
sandbox の中の Bash で、`$TMPDIR` と、各フォルダへの書き込み・読み取り・一覧を試すスクリプトを 1 回実行させた。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-36 | 今の要件の安全設定（`denyRead` は `~/` と置き場）で、AI のセッションから Claude Code のユーザーごとの一時フォルダとほかのセッションの一時フォルダをどう扱えるか | **ユーザーごとの一時フォルダは読み書きでき、一覧も出せた。** `$TMPDIR` と Node の `os.tmpdir()` は `/tmp/claude-501` で、そこに書けて、中に置いたファイルを読めて一覧も出せた。ユーザーごとの一時フォルダには、人の Claude Code のセッションの作業ファイルとコマンドの出力のファイルもある。`denyRead` の外に置いたほかのセッションの一時フォルダは、読めて一覧も出せた（書けなかった）。安全設定の `denyWrite` にユーザーごとの一時フォルダを入れると、Claude Code が Bash のたびに書く作業ファイル（`/tmp/claude-501/cwd-…`）を書けず、Bash のコマンドが終了コード 1 になった。`mktemp -d`（引数なし）は、`$TMPDIR` でなく macOS のユーザーの一時フォルダ（`/var/folders/…`）に作ろうとして、sandbox に止められた | 実行 |
| FACT-37 | `CLAUDE_CODE_TMPDIR` で、sandbox の中の `$TMPDIR` をセッションごとの場所にできるか | **`<渡した場所>/claude-501` が 44 文字までならできる。** 28・29・36・44 文字では、`$TMPDIR` と `os.tmpdir()` が渡した場所の下の `claude-501` になり、ユーザーごとの一時フォルダには書けなかった。45・46・48・49・50・52・58・62 文字では、`$TMPDIR` が `/tmp/claude-501` に戻り、渡した場所とユーザーごとの一時フォルダの両方に書けた。長さは渡した文字列で数える（`/tmp/…` で 44 文字、実体の `/private/tmp/…` で 52 文字の場所が使われた）。`denyRead` に短い一時フォルダの根とユーザーごとの一時フォルダ、`allowRead` に自分の短い一時フォルダを入れると、ユーザーごとの一時フォルダとほかのセッションの一時フォルダは、読めず、一覧も出せず、書けなかった。Bash は普通に動いた。同じ安全設定で 50 文字の場所を渡すと、`$TMPDIR` は `/tmp/claude-501` に戻ってファイルを作れたが、中は読めず一覧も出せなかった。公式文書（[環境変数][doc-env]）は、長いパスでは既定の場所の下の短い `$TMPDIR` を渡すと書くが、長いとみなす値は書いていない。~~上限が `CLAUDE_CODE_TMPDIR` だけの長さか、`/claude-<uid>` を足した長さかは、uid が 501 だけの試験では区別できない~~（2026-10-18 に Claude Code 2.1.285 の本体を読んで確かめた。上限は `<CLAUDE_CODE_TMPDIR>/claude-<uid>` のバイト数で、44 バイトを超えると `/tmp/claude-<uid>` を使う。あわせて Claude Code は、一時フォルダがシンボリックリンクか、ほかの OS ユーザーの持ち物なら使うのを拒み、権限が 700 でなければ 700 に直す） | 実行・公式文書・Claude Code の本体 |

## 途中で止める ADR を起票する途中で確かめた事実（2026-10-18）

macOS、Claude Code 2.1.285、`claude-haiku-5-5` で確かめた。[ADR-20261018-001][adr-usage]（途中で止めた回の止め方と消費の数え方）の材料として調べた。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外。
AI のセッションと同じ起動の形（`claude -p --setting-sources local --settings <安全設定（sandbox を有効、allowUnsandboxedCommands は false）> --strict-mcp-config --output-format stream-json --verbose --include-partial-messages`。環境変数は許可した名前だけ、一時フォルダはセッションごとの短いパス）で、別のプロセスグループに起動し、途中でシグナルを送った。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-38 | 作業中の `claude` を途中で止めたときに、AI のコマンド（孫プロセス）がどうなるか | **Bash のコマンドは `claude` と別のプロセスグループで動き、止め方によっては残る。** モデルが Bash で長いコマンド（`sh` が `sleep` を子に持ち、1 秒ごとに印のファイルを書く）を実行している間に止めた。`claude` のプロセスグループか `claude` のプロセスだけに `SIGTERM` を送ると、`claude` はコマンドを止めて 1 秒ほどで終了コード 143 で終わり、3 秒後にコマンドは残っていなかった。コマンドが `trap '' TERM HUP INT` で `SIGTERM` を無視すると、`claude` は終了コード 143 で終わったが、コマンドは親が PID 1 になって残り、印のファイルを書き続けた。プロセスグループに `SIGKILL` を送ったときも、コマンドが残った。残ったコマンドは sandbox の中のままで、`claude` が終わった後に作業場所の外へ書こうとすると止められた。`claude` のプロセスだけに `SIGINT` を送ると、`claude` は終了コード 0 で終わり、コマンドは残らず、`result` が出た（`subtype` は `error_during_execution`、`terminal_reason` は `aborted_tools`、`modelUsage` は書き終えたメッセージの `message_delta` と同じ値）。ただし、モデルが文を書いている途中に `SIGINT` を送ると、`result` の `terminal_reason` は `aborted_streaming`、`total_cost_usd` は 0、`usage` のトークンもすべて 0 だった | 実行 |
| FACT-39 | 途中で止めた回の `stream-json` に、消費がどこまで残るか | **書き終えたメッセージは `message_delta` に最後のトークン数が残り、書いている途中のメッセージは入力の側しか残らない。** コマンドの実行中に `SIGTERM` で止めた回は、道具を呼んだメッセージの `message_start` と `message_delta`（出力 114。`stop_reason` は `tool_use`）が出ていて、`result` は無かった。文を書いている途中に `SIGTERM` で止めた回は、`message_start`（入力の側のトークン数。出力は 2）と文の差分（`text_delta`）だけで、`message_delta` も `result` も無かった。`--max-budget-usd 0.01` で短い指示を実行すると、`result` は `subtype` が `error_max_budget_usd`、`terminal_reason` が `budget_exhausted`、`total_cost_usd` が 0.0769 で、上限をメッセージ 1 つ分超えた（上限はメッセージの間で確かめる）。正常に終わった回の `terminal_reason` は `completed`（FACT-31 の試験の記録） | 実行 |

## エージェント定義の置き方を決める途中で確かめた事実（2026-10-19）

macOS、Claude Code 2.1.285、haiku（`claude-haiku-4-5-20251001` に解けた）で確かめた。scratchpad に、名札とエージェント定義 2 つ（`agents/plan/reviewer.md` と `agents/change/verifier.md`。frontmatter の `name` はファイルの名前と同じ）だけを持つ試しの Plugin（名前は `probe`）を作った。
`claude -p --setting-sources local --strict-mcp-config --plugin-dir <試しの Plugin> --agent probe:plan:reviewer --max-turns 1 --max-budget-usd 0.05 --output-format stream-json --verbose` を 1 回実行した。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-40 | `agents/` の下のフォルダに置いたエージェント定義を、コロンが 2 つ付く名前（`<Plugin の名前>:<フォルダ>:<名前>`）の `--agent` で起動できるか | **できる。** `init` のイベントの `agents` に `probe:change:verifier` と `probe:plan:reviewer` が出た。`--agent probe:plan:reviewer` で起動したセッションは、その定義の指示どおりの語だけを返した（`result` は `success`、費用は 0.001 USD）。STRUCT-9 の名前（`dev-autopilot:<ループ>:<役>`）で起動できる | 実行 |

## セッションごとの安全設定を決める途中で確かめた事実（2026-10-19）

macOS、Claude Code 2.1.285、haiku（`claude-haiku-4-5-20251001` に解けた）で確かめた。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外（`/private/tmp/dapprobe/`）。
どちらも `claude -p --setting-sources local --strict-mcp-config --settings <安全設定> --output-format stream-json --verbose` を 1 回ずつ実行した（費用は 0.025 USD と 0.007 USD）。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-41 | sandbox の中のコマンドが、`allowWrite` に入れたセッションのフォルダの `tmp/` に書けるか。入れていないフォルダには書けないか | **`tmp/` には書け、入れていない隣のフォルダには書けない。** 置き場を `denyRead`、自分のセッションのフォルダを `allowRead`、`<セッションのフォルダ>/tmp` を `allowWrite` にし、作業フォルダを worktree にして、1 つの `sh -c` で 3 か所に書かせた。`tmp/` は終了コード 0 でファイルができ、`allowWrite` に入れていない `<セッションのフォルダ>/other/` は `Operation not permitted`（終了コード 1）でファイルができず、作業フォルダは終了コード 0 でファイルができた。ファイルの有無はディスクで確かめた | 実行 |
| FACT-42 | `permissions.allow` に `Skill` を入れずに、エージェントが Plugin の Skill を呼べるか | **呼べる。** `allow` が `Bash` だけの安全設定で、`tools` に `Skill` だけを書いたエージェント（`--agent probe:change:reviewer`）を起動すると、`Skill` の道具で `probe:hello` を呼び、Skill の本文の指示どおりの語を返した。`permission_denials` は空で、`result` は `success` だった | 実行 |

## 止め方と利用枠の数え方を決める途中で確かめた事実（2026-10-19）

macOS、Claude Code 2.1.285、haiku（`claude-haiku-4-5-20251001` に解けた）で確かめた。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外（`/private/tmp/dap-stop-probe/`）。
FACT-43 と FACT-45 は、AI のセッションと同じ起動の形（`--setting-sources local`、`--settings` の sandbox、許可した名前だけの環境変数、`--include-partial-messages`）の `claude -p` を 1 回実行して確かめた。FACT-44 は、モデルを使わずに、この会話の Bash から確かめた。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-43 | 止めた後に残ったコマンドを、止める前に控えたプロセスグループと、作業フォルダ（cwd）で見つけて止められるか | **どちらでも見つけて止められる。** `SIGTERM` を無視するコマンド（`sh` が `sleep` を子に持ち、1 秒ごとに印のファイルを書く）を Bash で動かしている間に、1 秒ごとに `claude` の子孫をたどってプロセスグループを控えた（始めてから 4.2 秒で控えられた）。`claude` のプロセスグループに `SIGTERM` を送ると、`claude` は 0.5 秒で終了コード 143 で終わり、`sh`（親が PID 1 に替わった）と `sleep` 2 つが残った。3 つとも控えたプロセスグループの中にあり、`lsof` で作業フォルダがセッションのフォルダの中にあるプロセスを探しても、同じ 3 つが見つかった。控えたプロセスグループに `SIGKILL` を送ると、3 秒後にはどちらの探し方でも何も見つからず、印のファイルの更新も止まった | 実行 |
| FACT-44 | macOS の `ps` で、セッションの ID と、ほかのプロセスの環境変数を取れるか | **この会話の Bash からは取れなかった。** `ps -o sess` は 0 を出した。`ps -ww -E` は、子のプロセスにも自分のシェルにも、環境変数を出さなかった。進行 Bot と同じ条件（Claude Code の外の定期の実行）では確かめていない | 実行（モデルなし） |
| FACT-45 | AI のセッションの Bash のコマンドは、どのシェルで動くか | **人のログインシェル（zsh）で、~~Claude Code がホームの `.claude/shell-snapshots/` に作ったシェルのスナップショットを読み込んで動いた~~ Claude Code がホームの `.claude/shell-snapshots/` に作ったシェルのスナップショットを読み込む形のコマンドで動いた（読み込みは sandbox に止められる。FACT-46）。** FACT-43 の試験で控えたコマンドは `/bin/zsh -c source <ホーム>/.claude/shell-snapshots/snapshot-zsh-…` の形だった（子プロセスに `SHELL` は渡していない）。同じフォルダの最新のスナップショットには、人のシェルの関数 13・別名 2・`PATH` の export 1 行が入っていた（値は見ていない）。AI のセッションから `shell-snapshots/` を読めるか、何が入りうるかは確かめていない（UNDECIDED-54） | 実行 |

## 人のシェルのスナップショットを確かめた事実（2026-10-19）

macOS、Claude Code 2.1.285、haiku（`claude-haiku-4-5-20251001` に解けた）で確かめた。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外（`/private/tmp/dap-snap-probe/`）。
AI のセッションと同じ安全設定（`denyRead` に `~/` と置き場）と、許可した名前だけの環境変数で `claude -p` を 1 回実行し（費用は 0.046 USD）、Bash で確かめた。値は見ず、数と可否だけを出した。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-46 | sandbox の中の AI のコマンドが、人のシェルのスナップショットと、その置き場をどう扱えるか | **読めず、人の関数も当たらない。** `~/.claude/shell-snapshots/` の一覧は出せず、ほかのセッションのスナップショットは `operation not permitted` で読めなかった。人のシェルの関数は 0 個（スナップショットには 15 個ある）、別名は zsh にもともとある 2 つ（`run-help`、`which-command`）だけで、`PATH` は進行 Bot が渡した値のままだった。人の `~/.zshrc` には `export` の行が 3 つあったが、スナップショットの `export` は `PATH` の 1 行だけだった。公式文書は、セッションの始めにシェルの設定ファイルを読み込み、別名・関数・シェルのオプションを取り出して Bash のコマンドに当てるとする（[tools-reference][doc-tools]） | 実行 |

## 安全設定の env と費用の数え方を確かめた事実（2026-10-20）

macOS、Claude Code 2.1.285、`claude-haiku-5-5`（完全な ID で渡した）、Node 24.3.0、npm 11.4.2 で確かめた。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外（`/private/tmp/dapv-<時刻>/`）。
基本設計の「[AI のセッション][design-ai]」の安全設定と同じ形（sandbox、`denyRead` に `~/` と置き場、通信の許可先は `registry.npmjs.org` だけ、`env`、`permissions`）の設定ファイルと、基本設計の環境変数の表の名前だけで `claude -p` を 1 回実行した（`--agent` は付けない。表示の費用は 0.249 USD。FACT-48）。Bash で試しのスクリプトを 1 回動かし、Read ツールでファイルを 2 つ読ませた。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-47 | 安全設定の `env` に書いた値が、sandbox の中の Bash に届くか。npm のキャッシュと Node の `fetch` に効くか | **届き、効く。** `NODE_USE_ENV_PROXY` は `1`、`npm_config_cache` は `${TMPDIR}/npm-cache` のまま届いた（Claude Code は展開しない）。`npm config get cache` は `<短い一時フォルダ>/claude-<uid>/npm-cache` を返し（展開したのは npm）、`npm view` が通って、キャッシュも同じフォルダにできた。許可した通信先への Node の `fetch` は、`NODE_USE_ENV_PROXY` があれば 200 で、外すと `ENOTFOUND` で届かなかった。許可していない通信先は sandbox が止めた。sandbox の中の `$TMPDIR` は、渡した短い一時フォルダの下の `claude-<uid>` だった | 実行 |
| FACT-48 | Claude Code が `claude-haiku-5-5` の費用をどう数えるか | **単価を知らず、Opus 5.5 の単価で数える。** 起動のときに `unrecognized_model` の警告が出て、`result` の `modelUsage` の `costBasis` は `unknown` だった。モデルは API が受け付けて動いた（`modelUsage` のキーは `claude-haiku-5-5`）。`total_cost_usd` の 0.2489072 USD は、トークン数（入力 4、出力 965、キャッシュの読み取り 27,276、1 時間のキャッシュの書き込み 28,017）を Opus 5.5 の単価（100 万トークンあたり、入力 4 USD、出力 20 USD、キャッシュの読み取り 0.20 USD、1 時間のキャッシュの書き込み 8 USD）で数えた値と一致した。FACT-40 から FACT-46 の費用は、`haiku` の別名（Haiku 4.5 に解けた）で起動した値で、Claude Code が単価を知っているモデルの値 | 実行 |
| FACT-49 | Read ツールの拒否が記録に載るか。写しの `skills/` を読めるか。道具の一覧に `Agent` と `Task` が出るか。短い試しが何ターンで終わるか | **拒否は `permission_denials` に載り、`Agent` も `Task` も一覧に出ない。** 作業フォルダの外のファイルを Read ツールで読ませると、結果が `is_error` になり、`result` の `permission_denials` に道具の名前とファイルのパスが載った。`permissions.allow` の `Read(//<写し>/skills/**)` に当たるファイルは読めた。`permissions.deny` に `Agent` を入れると、初期化の記録の道具の一覧に `Agent` も `Task` も無かった。一覧には、別の道具の `TaskStop` があった。試しのスクリプトの実行と 2 つの読み取りは、4 ターン・7 秒で終わった | 実行 |
| FACT-50 | AI のセッションが、短い一時フォルダの外と人のホームに何を残すか | **短い一時フォルダの直下と、人のホームの会話の記録に残す。** 短い一時フォルダ（`CLAUDE_CODE_TMPDIR`）の直下に、`claude-<uid>` のほかに `cc-socks`、`node-compile-cache`、`npm-cache` ができた。人のホームの `~/.claude/projects/` の下に、作業フォルダのパスから名前を付けたフォルダができ、会話の記録（`.jsonl`）が残った | 実行 |

## 会話の記録を残さない引数を確かめた事実（2026-10-20）

macOS、Claude Code 2.1.285、haiku の別名で確かめた。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外（`/private/tmp/dapnp-<時刻>/wt`）。`claude -p --no-session-persistence --setting-sources local --strict-mcp-config --max-turns 1` を 1 回実行した（表示の費用は 0.016 USD）。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-51 | `--no-session-persistence` を付けると、会話の記録が人のホームに残るか | **会話の記録は残らないが、空のフォルダは残る。** 実行の前に無かった `~/.claude/projects/<作業フォルダのパスから付けた名前>/` ができ、中は空の `memory/` だけで、会話の記録（`.jsonl`）は無かった（FACT-50 では、付けずに起動して `.jsonl` が残った）。`claude --help` は、`--no-session-persistence` は `--print` のときだけ効き、会話を保存せず再開できなくなると書く | 実行 |

## fine-grained PAT でチェックの実行を読めるかを確かめた事実（2026-10-20）

メンテナが発行した fine-grained PAT（対象は Joymerrevent/business-os だけ、権限は Actions・Commit statuses の読み取りと、自動で付く Metadata の読み取り、期限 1 日）で、business-os の main の先端の commit に対して、REST API を読んだ。トークンの値は画面に出さず、応答の状態と件数、応答の見出し（`X-Accepted-GitHub-Permissions` と `X-RateLimit-Limit`）だけを控えた。試しの後に、トークンを保存したファイルを消した。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-52 | fine-grained PAT で、CI のチェックの実行（check runs）を読めるか | **公開のリポなら読める。** `GET /repos/{owner}/{repo}/commits/{ref}/check-runs` は 200 で、6 件のチェックの実行の名前・結果・app（`github-actions`）が読めた。利用の上限は 5,000 で、PAT の認証が効いていた。応答の見出しは、要る権限を `checks=read` と返した。fine-grained PAT の権限の一覧には Checks が無く（GitHub の公式文書）、PAT にも付けていないので、公開のリポのリソースを権限なしで読めた形。check suites も同じく読めた。認証なし（上限 60）と、PAT の対象に入れていない公開のリポでも読めた。非公開のリポでは読めなかった（FACT-54） | 実行 |
| FACT-53 | 同じ PAT で、ワークフローの実行、コミットの状態、PAT の持ち主を読めるか | **読める。ただし、コミットの結合した状態は、状態が無くても `pending` を返す。** `GET .../actions/runs?head_sha=` は 200（要る権限は `actions=read`）で、ワークフローの名前・パス（`.github/workflows/ci.yml` など）・結果が読めた。`GET .../commits/{ref}/status` は 200（`statuses=read`）だったが、状態の件数が 0 なのに `state` は `pending` だった。`GET /user` は 200（`allows_permissionless_access=true`）で、PAT の持ち主のログイン名が読めた | 実行 |

## 非公開のリポでチェックの実行を読めるかを確かめた事実（2026-10-20）

試しのためだけに、組織 Joymerrevent に非公開のリポ（`dap-checks-probe`）を作り、`echo ok` だけのワークフローを 1 回動かして、チェックの実行を 1 件作った。メンテナが発行した fine-grained PAT（対象は試しのリポだけ、権限は Actions・Commit statuses の読み取りと、自動で付く Metadata の読み取り、期限 1 日）で、FACT-52・53 と同じ API と、ワークフローのジョブを読んだ。ジョブの名前とチェックの実行の名前の対応は、business-os（公開）の main の先端の commit で、認証なしで比べた。トークンの値は画面に出さず、試しの後にトークンを保存したファイルを消した。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-54 | 非公開のリポで、fine-grained PAT でチェックの実行を読めるか。読めないなら、ほかの API で必須チェックの結果を読めるか | **チェックの実行は読めず、Actions のジョブからなら読める。** `commits/{ref}/check-runs` と `check-suites` は 403（`Resource not accessible by personal access token`。要る権限は `checks=read`）だった。`actions/runs?head_sha=` と `actions/runs/{id}/jobs` は 200（要る権限は `actions=read`）で、ワークフローのパスと、ジョブの名前・結果が読めた。ジョブには、対応するチェックの実行の URL があった。business-os では、ジョブの名前（マトリクスの `check (ubuntu-latest)` などを含む）が、チェックの実行の名前と同じだった。コミットの結合した状態は、状態が 0 件で `pending` だった（FACT-53 と同じ）。認証なしでは 404 だった | 実行 |

## エージェント定義の道具と、作業フォルダの local の設定と、取り込みの検査を確かめた事実（2026-10-20）

基本設計の再レビューで確かめた。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外（`/private/tmp/` の下に試験ごとのフォルダを作った）。FACT-57 は git だけで試した。FACT-55・56 は、macOS、Claude Code 2.1.285、`claude-haiku-4-5-20251001` で、どれも `claude -p --setting-sources local --no-session-persistence --output-format stream-json --verbose` で起動し、終わった後に試験のフォルダと、人のホームに残った空のフォルダ（FACT-51）を消した。費用は合わせて 0.083 USD。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-55 | エージェント定義の `tools` に `Skill` が無いとき、エージェントが Plugin の Skill を呼べるか | **呼べない。** `tools: Bash, Read` のエージェント（`--agent probe:change:reviewer`）は、初期化の記録の道具の一覧が `["Bash","Read"]` で、Skill を呼ぶよう頼むと、Skill の道具が無いと返した。同じ Plugin の `tools: Bash, Read, Skill` のエージェントは、道具の一覧に `Skill` があり、`probe:hello` を呼んで、Skill の本文の指示どおりの語を返した。どちらも、初期化の記録の Skill の一覧には、Plugin の Skill と、Claude Code に同梱の Skill（`update-config`・`schedule`・`loop` など 18 個）が並んだ。FACT-42 の「`allow` に入れなくても使える」は、`tools` に `Skill` を書いたエージェントのときに当たる | 実行 |
| FACT-56 | 作業フォルダの `.claude/settings.local.json` を、sandbox の中の Bash から書けるか。`--setting-sources local` で起動すると読まれるか | **Bash からは書けないが、置いてあれば読まれ、hook が sandbox の外で動く。** sandbox を有効にし、`permissions.allow` を `Bash` だけにした安全設定で、作業フォルダ（git のリポ）で 3 つの書き込みを実行させた。`.claude/settings.local.json` と `.claude/settings.json` への書き込みは `operation not permitted`（終了コード 1）でファイルができず、同じフォルダの `.claude/other.txt` は終了コード 0 でできた。別の試験で、作業フォルダに SessionStart の hook を書いた `.claude/settings.local.json` を置いて起動すると、hook が動き、作業フォルダの外（sandbox の書き込みの範囲の外）にファイルを作った。当てる先のリポに `.claude/settings.local.json` が commit されていれば、clone に入って読まれる | 実行 |
| FACT-57 | objects のファイルを書き換えた clone から、ローカルのパス（`file://` でない）で `git fetch` したとき、書き換えた objects を取り込むか | **取り込まず、失敗する。** 進行 Bot の取り込み（push の手順の 2。`git fetch <パス> <ブランチ>`）と同じ形で試した。AI の作業場所に見立てた clone で commit を 1 つ作り、その blob の objects のファイルを別の中身に書き換えた。別の clone から、パスと `file://` の URL のそれぞれで fetch すると、どちらも `missing blob object` と `did not send all necessary objects` で終了コード 1 になり、`FETCH_HEAD` はできなかった。FACT-18 は `file://` の clone で試したもので、パスからの fetch でも同じに止まる。git 2.50.1、`GIT_CONFIG_GLOBAL=/dev/null`、sandbox なし。モデルは使っていない | 実行（sandbox なし） |

## エージェントの Skill を絞る書き方を確かめた事実（2026-10-21）

UNDECIDED-59（レビュー AI に `Skill` の道具を持たせるか）の ADR の前に確かめた。macOS、Claude Code 2.1.285、`claude-haiku-4-5-20251001` で、試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外（`/private/tmp/` の下に試験のフォルダを作った）。Skill を 2 つ（`probe:hello`、`probe:other`）持つ試験用の Plugin を作り、`claude -p --plugin-dir <Plugin> --agent <エージェント> --setting-sources local --no-session-persistence --output-format stream-json --verbose` で 3 回起動した。終わった後に試験のフォルダを消した。費用は合わせて 0.046 USD。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-58 | `Skill` の道具を持つエージェントで、呼べる Skill を絞れるか。`Skill` の道具を持たずに、エージェント定義の `skills:` で Skill を読み込ませられるか | **名前ごとに拒めるが、「1 つだけ許す」書き方はできない。`skills:` では読み込まれない。** `tools: Bash, Read, Skill` のエージェントに、安全設定の `permissions.deny` で `Skill(probe:other)` と `Skill(update-config)` を拒むと、2 つの呼び出しは「Skill execution blocked by permission rules」で拒まれて `permission_denials` に載り、`probe:hello` は呼べた。Claude Code に同梱の Skill（`update-config`）も同じ書き方で拒めた。`permissions.deny` に `Skill(*)`、`allow` に `Skill(probe:hello)` を書くと、初期化の記録の道具の一覧から `Skill` が消え、どの Skill も呼べなかった（拒む規則が許す規則より強い）。`tools: Bash, Read` で、エージェント定義の frontmatter に `skills:` で `probe:hello` を書いたエージェントは、Skill の本文の中身を知らなかった。初期化の記録の Skill の一覧に並んだ Claude Code に同梱の Skill は 18 個（`deep-research`・`design`・`design-sync`・`dataviz`・`update-config`・`verify`・`debug`・`code-review`・`simplify`・`batch`・`fewer-permission-prompts`・`doctor`・`loop`・`schedule`・`claude-api`・`workflow-authoring`・`run`・`run-skill-generator`）で、Claude Code の版で変わりうる | 実行 |

## Skill の道具を名前だけで拒む書き方を確かめた事実（2026-10-21）

ADR-20261021-001（レビュー AI に `Skill` の道具を持たせない）の確認方法に従って、要件と基本設計に反映する前に確かめた。macOS、Claude Code 2.1.285、`claude-haiku-4-5-20251001` で、試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外（`/private/tmp/` の下に試験のフォルダを作った）。Skill を 1 つ（`probe:hello`）と、`tools: Bash, Read, Skill` のエージェント（`probe:change:withskill`）を持つ試験用の Plugin を作り、安全設定に `permissions.deny` の `Skill` だけを書いて、`claude -p --plugin-dir <Plugin> --agent probe:change:withskill --setting-sources local --settings <安全設定> --no-session-persistence --output-format stream-json --verbose --max-turns 4` で 1 回起動した。終わった後に試験のフォルダを消した。費用は 0.012 USD。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-59 | 安全設定の `permissions.deny` に道具の名前だけの `Skill` を書くと、エージェント定義の `tools` に `Skill` を書いたエージェントからも、Skill の道具が消えるか | **消える。** 初期化の記録の道具の一覧は `["Bash","Read"]` で、`Skill` が無かった。Skill を呼ぶよう頼むと、Skill の道具が無いと返し、Skill を呼ばなかった（`permission_denials` は空で、拒まれたのでなく道具が無かった）。初期化の記録の Skill の一覧には、Plugin の `probe:hello` と Claude Code に同梱の Skill が、合わせて 19 個並んだ。FACT-58 の `Skill(*)` と同じく、道具ごと取り除かれる | 実行 |

## 基本設計の 2 回目の再レビューで確かめた事実（2026-10-21）

基本設計の 2 回目の再レビュー（2026-10-21）の指摘を直す前に確かめた。macOS、Claude Code 2.1.285、git 2.50.1（Apple Git-155）、Node v24.3.0。Claude Code の試しは `claude-haiku-4-5-20251001` で、どれも `claude -p --setting-sources local --no-session-persistence --output-format stream-json --verbose` で起動した。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外（`/private/tmp/` の下に試験ごとのフォルダを作った。FACT-62 だけは、確かめる対象が人のホームの中にあるので、`~/.claude/projects/` の下に試験のフォルダを作った）。終わった後に、試験のフォルダとファイルを消した。Claude Code の試しの費用は合わせて 0.21 USD。
GitHub の試し（FACT-65〜67）は、Joymerrevent に非公開の試しのリポ 2 つと試しの Project と試しのチームを作り、人の `gh` のログイン（組織の管理者）で行った。FACT-65 の読み取りの一部は、人が作った fine-grained PAT（試しのリポだけ、読むだけ）でも行い、試しの後に PAT を消した。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-60 | `--setting-sources local` で起動したとき、作業フォルダとその上と下のフォルダの `CLAUDE.local.md`、下のフォルダの `CLAUDE.md`、作業フォルダの `AGENTS.md` が指示として読まれるか | **`CLAUDE.local.md` は読まれる。** 作業フォルダと、作業フォルダの上のフォルダの `CLAUDE.local.md` に書いた合言葉は、起動の時点で読まれた（道具を拒んだセッションが合言葉を返した）。下のフォルダ（`sub/`）の `CLAUDE.local.md` は、起動の時点では読まれず、`sub/` のファイルを Read で読んだ後に差し込まれた（`stream-json` の道具の結果には出ず、AI は読まずに合言葉を返した。`CLAUDE.local.md` 自体の Read を `deny` にしても差し込まれた）。下のフォルダの `CLAUDE.md` と、作業フォルダの `AGENTS.md` は読まれなかった。費用は合わせて 0.08 USD | 実行 |
| FACT-61 | objects を書き換えた clone からの fetch の試しが、clone に commit を足さないときも失敗するか | **失敗しない。** 小さなリポ（commit 1 つ）を `--no-hardlinks` で clone し、clone の blob の loose object を書き換えてから、小さなリポに `git fetch <clone のパス> main` をすると、終了コード 0 で通った（取り込む objects が無いため）。clone に commit を 1 つ足し、足した commit の blob を書き換えると、終了コード 128 で失敗し、`FETCH_HEAD` は中身が空（0 バイト）のまま残った。FACT-57 は clone で commit を作ってから書き換えた形で、終了コードは書き換え方（別の中身の object か、壊れた中身か）で変わりうる。モデルは使っていない | 実行 |
| FACT-62 | `-p` と `--setting-sources local` の起動で、自動のメモリが読まれるか。止める書き方 | **読まれる。止められる。** 作業フォルダに対応する `~/.claude/projects/<作業フォルダのパスの / と . を - にしたもの>/memory/MEMORY.md` に書いた合言葉を、道具を拒んだセッションが返した。安全設定に `"autoMemoryEnabled": false` を書くと読まれず、環境変数 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` でも読まれなかった。費用は合わせて 0.03 USD | 実行 |
| FACT-63 | sandbox の中から、ホームの外の一時フォルダと共有のフォルダを読めるか。`denyRead` に足しても道具が動くか | **`~/` の拒否だけでは読める。足すと読めず、道具は動く。** `denyRead` が `~/` だけのとき、`getconf DARWIN_USER_TEMP_DIR`（`/var/folders/…/T`）と `DARWIN_USER_CACHE_DIR`（`…/C`）と `/Users/Shared` のファイルを読め、`/Volumes` の一覧を出せた。4 つを `denyRead` に足すと、どれも読めず一覧も出せず、`node`・`pnpm`（`/opt/homebrew`）・`git`（`/usr/bin/git`）は動いた。`git` は、`GIT_CONFIG_GLOBAL=/dev/null` と `XDG_CONFIG_HOME` をセッションの中に向けないと、`~/.config/git/config` を読めずに終了コード 128 で止まる（設計どおりの環境変数なら動く）。sandbox は有効・`failIfUnavailable`・`allowUnsandboxedCommands` は `false`。試しのスクリプトが作業フォルダに書いた結果で判定した。費用は合わせて 0.10 USD | 実行 |
| FACT-64 | Node 24 の権限モデル（`--permission`）で、許していないファイルの読み込みと、子プロセスの起動が止まるか | **止まる。** `node --permission --allow-fs-read=<フォルダ>/*` で、許していないファイルの `readFileSync` は `ERR_ACCESS_DENIED` になった。`--allow-child-process` が無いと、`spawnSync` が `ERR_ACCESS_DENIED` を投げた。付けると子プロセスを起こせたが、Node は「権限の守りを無効にしうる」と `SecurityWarning` を出した。モデルも GitHub も使っていない | 実行 |
| FACT-65 | 書き込みのある deploy key の push 先を ruleset で限れるか。ruleset の bypass の種類で変わるか | **限れる。ただし bypass にリポの役を入れると素通りする。** 組織のプランは team。ブランチの ruleset（`~ALL` から `refs/heads/*/issue-*` を除き、creation・update・deletion）とタグの ruleset（creation）を `enforcement` が `active` で作り、deploy key で push した。ssh は `-F /dev/null`・`-i <deploy key>`・`-o IdentitiesOnly=yes`・`-o IdentityAgent=none`・GitHub の API の `meta` から作った known_hosts と `StrictHostKeyChecking=yes` にした。bypass がリポの役（admin・maintain・write のどれか）のときは、`main` の更新・ほかのブランチの作成・タグの作成・削除まで全部通った。bypass が空か、組織の管理者（`OrganizationAdmin`）か、チーム（`Team`）のときは、`feat/issue-2` は通り、`main`・ほかのブランチ・タグは `GH013: Repository rule violations` で断られた。bypass が組織の管理者かチームのとき、組織の管理者でチームに入った人（`gh` の認証）はほかのブランチを作れた。bypass が空のときは人も断られた。base ブランチに効く規則は `GET /repos/{owner}/{repo}/rules/branches/main` で読めた。人の fine-grained PAT（持ち主は組織の管理者。試しのリポだけ。Administration の権限なし）でも、ruleset の一覧と、各 ruleset の `conditions`・`rules`・`bypass_actors`、`rules/branches/main` を読めた（値は `gh` のログインで読んだものと同じ）。同じ PAT で、deploy key の一覧（Administration の読み）、同じ中身での ruleset の書き換え（Administration の書き）、組織の ruleset の一覧は 403 だった。持ち主が組織の管理者でない PAT と、GitHub App のインストールのトークンで `bypass_actors` が返るかは確かめていない | 実行 |
| FACT-66 | Issue を別のリポに移したとき、元の番号を REST と GraphQL で読むとどうなるか。Project の項目はどうなるか | **REST は移した先へ 301、GraphQL は見つからない、Project の項目は移した先を指す。** 元の番号の REST は `301 Moved Permanently` で、移した先の Issue の API の URL を返した（`gh api` は黙ってたどり、移した先の Issue を返す。`repository_url` で見分けられる）。GraphQL の元の番号は `NOT_FOUND`。Project の項目は残り、移した先の Issue を指した。移した先の timeline に `transferred` が載った | 実行 |
| FACT-67 | ruleset の bypass に Dependabot を足せるか。足さないと、Dependabot のブランチの作成が止まるか | **足さないと止まる。足せるのは、Dependabot がリポで動いた後。** 試しのリポに新しいブランチを作って既定のブランチにし、`package.json`（古い版の依存 1 つ）と `.github/dependabot.yml`（npm、毎日）を置いた。FACT-65 のブランチの ruleset（bypass はチームだけ）が効いている間は、Dependabot の実行（GitHub Actions の `dynamic`）は成功で終わり、ログは更新を見つけて PR を作る手前（`Submitting … pull request for creation`）まで進んだが、`dependabot/` のブランチも PR もできなかった。Dependabot が動く前は、画面の bypass の候補に Dependabot が出ず、API で `Integration`（`GET /apps/dependabot` の ID の 29110）を足すと、422（`must be part of the ruleset source or owner organization`）で断られた。`github-actions`（15368）も同じ 422 だった。組織に入れたアプリ（`Integration`）は足せた。Dependabot が 1 回動いた後は、画面の候補に出て足せ、API で読むと `actor_type` が `Integration`、`actor_id` が 29110 だった。足した後に動かし直すと、Dependabot がブランチと PR を作れた。bypass を Dependabot だけにすると、組織の管理者（bypass の外）が既定のブランチを書き換えるのは 409（`Repository rule violations found` / `Cannot update this protected ref.`）で断られ、Dependabot の PR は `mergeable_state` が `blocked` になった（マージそのものは試していない） | 実行 |

## 基本設計の 3 回目の再レビューで確かめた事実（2026-10-22）

基本設計の 3 回目の再レビュー（2026-10-22）の指摘を直す前に確かめた。macOS（起動のボリュームは大文字と小文字を区別しない APFS）、Claude Code 2.1.285、git 2.50.1（Apple Git-155）。Claude Code の試しは `claude-haiku-4-5-20251001` で、どれも `claude -p --setting-sources local --no-session-persistence` で起動した。試験の場所は `/private/tmp/` の下に試験ごとのフォルダを作り（`/tmp/claude-<uid>/` と `~/.claude/` の外）、終わった後に消した。Claude Code の試しの費用は合わせて 0.09 USD。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-68 | 大文字と小文字を変えた `.claude` と `CLAUDE.local.md` のパスが、git の pathspec の照らしと、macOS の clone と、Claude Code の読み込みでどう扱われるか | **pathspec をすり抜け、clone では同じ場所に置かれ、Claude Code は読む。** `GIT_CONFIG_GLOBAL=/dev/null` の使い捨てのリポ（`core.ignorecase` は `true`）で、base に `.claude/dev-autopilot.json`、作業ブランチに `.Claude/settings.local.json`（`git update-index --cacheinfo` で足した）を置くと、`git diff --name-only <base>..<作業ブランチ> -- ':(glob).claude/**'` は何も返さず、`':(glob,icase).claude/**'` は返した。作業ブランチを clone すると、`.claude/` の中に `dev-autopilot.json` と `settings.local.json` の両方があった。作業フォルダに小文字の `claude.local.md` だけを置いて起動し、道具を拒んで合言葉を聞くと、`claude.local.md` の合言葉を返した（大文字の `CLAUDE.local.md` と同じ。費用は合わせて 0.03 USD） | 実行 |
| FACT-69 | GitHub の Issue に添付した画像などの URL の形 | **`https://github.com/user-attachments/assets/<ID>`。** 公開のリポの Issue 3 件の本文で、添付の URL はどれもこの形だった（読むだけ。モデルは使っていない） | 実行 |
| FACT-70 | sandbox の中の Bash と Write の道具が、大文字と小文字を変えた `.claude` のフォルダに書けるか | **どちらも書けない。** sandbox を有効にし（`failIfUnavailable`、`allowUnsandboxedCommands: false`）、`permissions.allow` を `Bash` にした安全設定で、`.Claude/settings.local.json` と `.CLAUDE/settings.local.json` への書き込みは `operation not permitted`（終了コード 1）でファイルができなかった（同じ設定で `.claude/other2.txt` は書けた。費用 0.02 USD）。`permissions.deny` に `Write(//<リポ>/.claude/**)` と `Edit(//<リポ>/.claude/**)` を書いた安全設定で、Write の道具で `.Claude/settings.local.json`・`.CLAUDE/notes.txt`・`.Claude/settings.json`・`.claude/notes3.txt` に書かせると、どれも「File is in a directory that is denied by your permission settings.」で拒まれ、ファイルはできなかった（費用 0.04 USD）。git の objects は書けるので、ファイルを書かずに大文字を混ぜたパスの commit は作れる（FACT-68） | 実行 |

## 基本設計の 4 回目の再レビューで確かめた事実（2026-10-22）

基本設計の 4 回目の再レビュー（2026-10-22）の指摘を直す前に確かめた。macOS、Claude Code 2.1.285。Claude Code の試しは `claude-haiku-4-5-20251001` で、`claude -p --setting-sources local --no-session-persistence` で起動した。試験の場所は `/private/tmp/` の下とホームの下に試験のフォルダを作り（`/tmp/claude-<uid>/` と `~/.claude/` の外）、終わった後に消した。費用は 0.06 USD。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-71 | sandbox を有効にした Claude Code の Bash から、作業フォルダの外のフォルダ（記録の置き場に見立てたもの）にファイルを書けるか | **書けない。** sandbox を有効にし（`failIfUnavailable`、`allowUnsandboxedCommands: false`）、`permissions.allow` を `Bash` にした安全設定で、作業フォルダ（`/private/tmp/` の下の試しの git のリポ）の外の、ホームの下の試しのフォルダと `/private/tmp/` の下の試しのフォルダに `mkdir -p` とファイルの書き込みをさせると、どちらも `Operation not permitted`（終了コード 1）で、ファイルはできなかった。人が sandbox を有効にした Claude Code で呼ぶ Skill が、作業フォルダの外の記録の置き場に依頼のファイルを書くときは、sandbox の外でコマンドを動かす許可が要る | 実行 |

## 作業の単位の状態遷移の ADR を起票する途中で確かめた事実（2026-10-24）

ADR-20261024-004 と ADR-20261024-006 の前提を確かめた。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-72 | 進行 Bot の認証（GitHub App）で、進行 Bot が投稿したコメントを「古い（outdated）」として畳めるか（前の版のマスターの下書きを畳む前提。[ADR-20261024-004][adr-master] の決定 1） | **畳める。** メンテナが、GitHub App の認証で、App が投稿したコメントを畳めることを確かめた | メンテナの答え（2026-10-24、ADR-20261024-004 の起票のとき）。2026-10-25 に、確かめた認証が GitHub App のトークンだったことをメンテナに確かめた。確かめた手順と日時の記録は無い |
| FACT-73 | push の権限を持たない人が、Issue を閉じられるか（push 以上の人が閉じたときだけ中止にする前提。[ADR-20261024-006][adr-cleanup] の決定 2） | **閉じられる。** Read の役でも、自分が開いた Issue を閉じられ、自分が閉じた Issue を開き直せる。Triage の役は、すべての Issue と PR を閉じ、開き直せる（Write・Maintain・Admin も同じ） | GitHub の公式文書の、組織のリポの役の表（「Close issues they opened themselves」「Close, reopen, and assign all issues and pull requests」の行。[公式文書][doc-repo-roles]）を、2026-10-24 に読み、2026-10-25 に読み直して同じ記述を確かめた |

[adr-workspace]: ../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-store]: ../adr/adr-20261015-002-no-shared-dependency-store.md
[pnpm-store]: https://pnpm.io/settings/store
[bos-issue-37]: https://github.com/Joymerrevent/business-os/issues/37
[bos-issue-44]: https://github.com/Joymerrevent/business-os/issues/44
[bos-pr-45]: https://github.com/Joymerrevent/business-os/pull/45
[bos-issue-80]: https://github.com/Joymerrevent/business-os/issues/80
[doc-mcp]: https://code.claude.com/docs/en/mcp
[adr-review]: ../adr/adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-human]: ../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-mcp]: ../adr/adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[adr-tmp]: ../adr/adr-20261017-006-per-session-short-tmpdir.md
[adr-usage]: ../adr/adr-20261018-001-stop-session-and-count-usage.md
[doc-sandboxing]: https://code.claude.com/docs/en/sandboxing
[doc-env]: https://code.claude.com/docs/en/env-vars
[doc-tools]: https://code.claude.com/docs/en/tools-reference#bash-tool-behavior
[design-ai]: ../design/basic/03-ai-sessions.md
[adr-master]: ../adr/adr-20261024-004-master-draft-quarantine-and-protected-paths.md
[adr-cleanup]: ../adr/adr-20261024-006-cleanup-states-and-dependencies.md
[doc-repo-roles]: https://docs.github.com/en/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/repository-roles-for-an-organization
