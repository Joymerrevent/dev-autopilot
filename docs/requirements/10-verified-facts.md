# 実機で確かめた事実

## 段階 2 で実機で確かめた事実（2026-10-08〜11）

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
| FACT-12 | `commondir` を差し替えた worktree に対する進行ボットの操作 | **worktree の中で git を実行すると発火した**（`git -C <worktree> status` で差し替え先の `core.fsmonitor` が動いた）。進行ボットの clone から worktree のパスへの fetch、親 clone のパスへの fetch、親 clone での `git worktree list` `git worktree remove --force`、親 clone での `git log <ブランチ>` は**発火しなかった**。SAFE-2 の (b) のとおり進行ボットが worktree の中で git を実行しなければ届かない | 実行 |
| FACT-13 | sandbox の `denyRead` が Read ツールにも効くか | **効かない**（公式文書どおり。sandbox は Bash だけを覆う）。Read ツールは `denyRead` のパスを読めた。`permissions.deny` の `Read(//<絶対パス>/**)`（絶対パスは `//` 始まり。`/` 始まりはプロジェクト相対で一致しない）で止まった（`File is in a directory that is denied by your permission settings`）。Write・Edit も同じで、ホームの任意のパスに書けた。`permissions.allow` を `Write(//<worktree>/**)` `Edit(//<worktree>/**)` に絞ると、外への Write は「Path is outside allowed working directories」で拒否され、`deny` に入れた `.git/**` `.claude/**` `CLAUDE.md` への Edit も止まった。`.git/hooks/**` と `.mcp.json` は Write ツールが「sensitive file」として既定で拒む | 実行 |
| FACT-14 | 道具なし、`--json-schema`、強制終了時の費用 | 道具なしはエージェント定義の `tools: []` でも `--tools ""` でも成立（Read を頼んでも「道具が無い」と返す）。**`--json-schema` は `--agent` と併用すると `structured_output` が null になる**（道具の有無を問わず。`--agent` 無しなら `structured_output` に検証済みの JSON が入る）。進行ボットは `result` の文を自分の JSON スキーマで検証する（IMPL-5）。`--output-format stream-json` は `assistant` のイベントごとに `model` と `usage` を持つ~~ので、途中で止めても消費は集計できる~~（FACT-29 で、`usage` が同じメッセージで重複し、`output_tokens` が最後の値でないと分かった。途中で止めた回の数え方は UNDECIDED-47）。プロセスグループごとの kill（`setpgrp` で起動し `kill -TERM -- -<pgid>`）で孫プロセスも残らなかった（モデルが長いコマンドを実行しなかったため、`claude` 自体を途中で止める試験は未実施） | 実行（kill は model なし） |
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

FACT-6〜17 は 2026-10-11 に、手で書いた安全設定と偽の入れ子 Plugin（scratchpad の実験場。親 clone と進行ボットの clone と worktree 2 つ）と捨てる Project と、人が発行した PAT で実機確認した。段階 2 の未確認は残っていない。
「`setup` が書いた安全設定で実体のパスの解決と読み取りの検査が動くこと」は段階 3 の出口。

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
| FACT-22 | AI の読み取りを、自分のセッションのフォルダに絞れるか | **絞れる。** sandbox の `denyRead` に `~/` と置き場の全体、`allowRead` に自分のセッションのフォルダを入れると、狭い方の規則が勝った。ホームのファイル（`.zshrc`、`.gitconfig`、`gh` の設定（リンクの名前と実体のパスの両方）、`.claude.json`、会話の記録、LaunchAgents）、進行ボットの偽の秘密、別のセッション、セッションの一覧は読めず、自分の clone と worktree は読めた。置き場がホームの外（`/private/tmp`）にあると、`~/` の拒否だけでは進行ボットの偽の秘密と別のセッションを読めた。**環境変数を渡さないと git と pnpm が止まる**：git は `~/.config/git/config` を読めず終了コード 128、pnpm は `~/.npmrc` を読めず終了コード 255。`GIT_CONFIG_GLOBAL=/dev/null`、`NPM_CONFIG_USERCONFIG=/dev/null`、XDG の 4 つと `TMPDIR` をセッションの一時フォルダに向けると、`git commit`、`pnpm install`（空の store で 9 秒）、`pnpm check:types`、`pnpm check:test`（441 件）が通った。親 clone の `.git/hooks` と `.git/config` には書けなかった。Read ツールは、`permissions.allow` に `Read` が無いと、worktree の外のファイルを「Claude requested permissions to read from this path, but you haven't granted it yet」で拒否し、`permission_denials` に記録した。worktree の中は読めた | 実行 |
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
| FACT-29 | `stream-json` の `assistant` のイベントの `usage` を足し上げて、セッションの消費を数えられるか | **数えられない。** 1 つのメッセージ（同じ `message.id`）が中身の区切り（thinking・text）ごとに別のイベントに分かれ、どのイベントも同じ `usage` を持つ（そのまま足すと、入力とキャッシュを重ねて数える）。`usage.output_tokens` はストリームの始めの値（`stop_reason` が null）で、最後の値ではない。3 回目の親のメッセージ 2 つでは、イベントの `output_tokens` の和が 25、最後の `result` の `modelUsage` が 485 だった。`--include-partial-messages` を付けると `stream_event` の `message_delta` が出て、親の `output_tokens` の和は 485 で `modelUsage` と一致した。サブエージェントのメッセージは `stream_event` として出なかった。最後の `result` の `modelUsage` と `total_cost_usd` は、サブエージェントの消費を含めたセッション全体の値だった。途中で止めたときに `message_delta` が出るかは確かめていない（UNDECIDED-44） | 実行 |
| FACT-30 | `--setting-sources local` で起動したセッションに、ログインしている claude.ai のアカウントのコネクタが入るか | **入った回があった。** 3 回のうち 2 回（1 回目と 2 回目）、初期化の記録の `mcp_servers` に claude.ai のコネクタが 4 つ出た（`source` は `claudeai`。メールと文書の 2 つが `connected`、残りの 2 つが `needs-auth`）。3 回目は `mcp_servers` が空だった。親の道具はエージェント定義の `tools` で `Task` と `Read` に絞っていたので、どの回もコネクタの道具は道具の一覧に無かった。認証が要るコネクタの案内の文が、2 回（2 回目と 3 回目）、親の最後の返事に混ざった。公式文書（[MCP][doc-mcp]）は、Claude Code が claude.ai から取ってくるコネクタを止める手段として、設定の `disableClaudeAiConnectors`、環境変数 `ENABLE_CLAUDEAI_MCP_SERVERS=false`、`--strict-mcp-config` を挙げている。~~dev-autopilot の起動の形（`--setting-sources local` と `--settings` の安全設定、許可した名前だけの環境変数）で効くかは確かめていない（UNDECIDED-48）~~（FACT-35 で、dev-autopilot の起動の形で 3 つとも効くと確かめた） | 実行 |
| FACT-31 | サブエージェントの消費とモデルが、`stream-json` の記録にどう出るか。サブエージェントはどう動くか | **最後の `result` にだけ揃って出る。** `modelUsage` は、サブエージェントのモデルが親と違えば別の項目に出た（3 回目：親の `claude-haiku-5-5` とサブエージェントの `claude-sonnet-5-5`）。サブエージェントの `model` に親と同じ系統の別名（`haiku`）を書くと、親のモデル（`claude-haiku-5-5`）で動き、同じ項目に足された（公式文書のモデルの決まり方のとおり）。途中の記録では数え切れない：サブエージェントの `assistant` のイベントは、バックグラウンドで動いた回（1 回目と 2 回目）はすべてのメッセージが `parent_tool_use_id` 付きで出たが、前で動いた回（3 回目。`--forward-subagent-text` なし）は 2 つのメッセージのうち道具を呼んだ 1 つだけが出た。`task_progress` と `task_notification` は `total_tokens`・`tool_uses`・`duration_ms` だけを持ち、モデルと入力・出力の内訳を持たない。サブエージェントのメッセージは `stream_event` として出ない（FACT-29）。サブエージェントをバックグラウンドで動かすかは、同じ指示でも回ごとに違った。バックグラウンドの回は、親がサブエージェントの答えを待たずにターンを終え、`result` が 2 つ出た（1 つ目は「サブエージェントはまだ動いている」という返事）。プロセスは 2 つ目の `result` の後に終わった。初期化の記録の道具の一覧では、Agent ツールは `Task` という名前で出た | 実行 |

## 写しの Plugin の読み取りとコネクタの止め方を確かめた事実（2026-10-17）

macOS、Claude Code 2.1.285、`claude-haiku-5-5` で確かめた。[ADR-20261017-004][adr-human]（写しの Plugin の読み取りと人が呼ぶ Skill の認証）と [ADR-20261017-005][adr-mcp]（AI のセッションに MCP のサーバーを読み込ませない）の材料として調べた。
AI のセッションと同じ起動の形（`claude -p --setting-sources local --settings <安全設定（sandbox を有効）> --plugin-dir <Plugin> --agent <試験用> --output-format stream-json`。環境変数は許可した名前だけで組み立てた）で、worktree から起動した。
試験の場所は、断りが無ければ Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）の外。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-32 | 進行ボットのプログラムの写しを `--plugin-dir` で渡したとき、AI のセッションが写しの Skill の参照ファイルとスクリプトを使えるか | **今の要件の安全設定では使えず、写しの `skills/` だけを読み取りの許可に足すと使える。** 置き場の中に写し（`program/0.0.1/`。エージェント定義、Skill、Skill の参照ファイルとスクリプト、`src/`）と、セッションのフォルダの worktree を作り、試験の Skill は参照ファイルとスクリプトを `${CLAUDE_SKILL_DIR}` で指した。今の要件の安全設定（`denyRead` に `~/` と置き場、`allowRead` に自分のセッションのフォルダ、`deny` に写しの Read）では、エージェント定義は読み込まれ、Skill を呼べた（`SKILL.md` の本文は Claude Code が読み込んだ）。一方、参照ファイルの Read は「File is in a directory that is denied by your permission settings」で拒否され、`node <写し>/skills/.../scripts/probe.mjs` は `Cannot find module` で失敗した。`allowRead` と `permissions.allow` の Read に写しの `skills/` を足し、`deny` の Read を写しの `src/` に狭めると、参照ファイルを Read で読め、スクリプトを Bash で実行できた。どちらの設定でも、`src/` は Bash でも Read でも読めなかった。写しの `skills/` への Bash の追記は `operation not permitted`、Write と Edit は「sensitive file」として拒否された。最初に試験の場所を `/tmp/claude-501/` の下に置いた回は、写しの `skills/` への Bash の追記が通った（Write と Edit は拒否された） | 実行 |
| FACT-33 | skills CLI が `plugin/skills/` の下の Skill を見つけるか | **見つける。** skills CLI 1.7.2 の `skills add <フォルダ> --list` は、試験用のフォルダの `plugin/skills/review/` と `plugin/skills/setup/` の 2 つを見つけた。`skills add anthropics/claude-code --list` は、リポの中の `SKILL.md` 10 個（`plugins/<名前>/skills/<名前>/SKILL.md` の深さを含む）をすべて見つけた。dev-autopilot のリポを指して試すのは、`plugin/skills/` を作った後になる | 実行 |
| FACT-34 | sandbox の中のコマンドの `$TMPDIR` は、進行ボットが渡した値になるか | **ならない。** FACT-32 の試験で、子プロセスの環境変数の `TMPDIR` にセッションの一時フォルダを渡したが、sandbox の中のコマンドの `$TMPDIR` は `/tmp/claude-501` だった。公式文書（[sandbox][doc-sandboxing]、[環境変数][doc-env]）は、sandbox の中のコマンドが作業フォルダとユーザーごとの一時フォルダに既定で書けること、Claude Code が sandbox の中のコマンドの `$TMPDIR` をユーザーごとの一時フォルダに置き換えることを書いている。ユーザーごとの一時フォルダは同じユーザーで動くすべての AI のセッションから書ける（FACT-32 の `/tmp/claude-501/` の下の写しへの追記）。一時フォルダの場所は `CLAUDE_CODE_TMPDIR` で変えられ、Claude Code が後ろに `/claude-<uid>/` を足す（パスが長いと、sandbox の中のコマンドには既定の場所の下の短いパスが渡る）。~~セッションごとに `CLAUDE_CODE_TMPDIR` を変えたときの振る舞いは確かめていない（UNDECIDED-50）~~（FACT-37 で確かめた。`<渡した場所>/claude-<uid>` が 44 文字までなら、渡した場所が使われる） | 実行と公式文書 |
| FACT-35 | claude.ai のコネクタが AI のセッションの道具に入るか。止め方が効くか | **止める仕組みが無いと入り、4 つの止め方はどれも効いた。** `tools` を書かないエージェント定義の試験用の Plugin を `--verbose` を足して起動し、初期化の記録の道具の一覧と MCP のサーバーの一覧を読んだ（道具は呼ばせていない）。止める仕組みなし（2 回）：MCP のサーバーは claude.ai の文書とメールが `connected`、ほかの 2 つが `needs-auth`。道具 63 個のうち 38 個が MCP の道具で、メールの道具 30 個に送信（`send_message`）・転送（`forward`）・返信（`reply`）・ゴミ箱への移動（`trash_message`）などがあった。安全設定の `disableClaudeAiConnectors: true`（2 回）、環境変数 `ENABLE_CLAUDEAI_MCP_SERVERS=false`（2 回）、`--strict-mcp-config`（2 回）：どれも MCP のサーバーの一覧が空で、MCP の道具は 0 個（道具 25 個）。安全設定の `permissions.deny` に `mcp__*`（1 回）：MCP のサーバーは `connected` のままで、MCP の道具は 0 個（道具 25 個）。MCP の道具を、`-p` で許可の規則に無いまま呼んだときの扱いは、メールを送る道具を呼ぶことになるので試していない | 実行 |

## 一時フォルダの ADR を起票する途中で確かめた事実（2026-10-17）

macOS、Claude Code 2.1.285、`claude-haiku-5-5`、uid 501 で確かめた。[ADR-20261017-006][adr-tmp]（セッションごとの短い一時フォルダ）の材料として調べた。
AI のセッションと同じ起動の形（`claude -p --setting-sources local --settings <安全設定（sandbox を有効、allowUnsandboxedCommands は false）> --strict-mcp-config --output-format stream-json`。環境変数は許可した名前だけで組み立てた）で起動し、
sandbox の中の Bash で、`$TMPDIR` と、各フォルダへの書き込み・読み取り・一覧を試すスクリプトを 1 回実行させた。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外。

| 番号 | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-36 | 今の要件の安全設定（`denyRead` は `~/` と置き場）で、AI のセッションから Claude Code のユーザーごとの一時フォルダとほかのセッションの一時フォルダをどう扱えるか | **ユーザーごとの一時フォルダは読み書きでき、一覧も出せた。** `$TMPDIR` と Node の `os.tmpdir()` は `/tmp/claude-501` で、そこに書けて、中に置いたファイルを読めて一覧も出せた。ユーザーごとの一時フォルダには、人の Claude Code のセッションの作業ファイルとコマンドの出力のファイルもある。`denyRead` の外に置いたほかのセッションの一時フォルダは、読めて一覧も出せた（書けなかった）。安全設定の `denyWrite` にユーザーごとの一時フォルダを入れると、Claude Code が Bash のたびに書く作業ファイル（`/tmp/claude-501/cwd-…`）を書けず、Bash のコマンドが終了コード 1 になった。`mktemp -d`（引数なし）は、`$TMPDIR` でなく macOS のユーザーの一時フォルダ（`/var/folders/…`）に作ろうとして、sandbox に止められた | 実行 |
| FACT-37 | `CLAUDE_CODE_TMPDIR` で、sandbox の中の `$TMPDIR` をセッションごとの場所にできるか | **`<渡した場所>/claude-501` が 44 文字までならできる。** 28・29・36・44 文字では、`$TMPDIR` と `os.tmpdir()` が渡した場所の下の `claude-501` になり、ユーザーごとの一時フォルダには書けなかった。45・46・48・49・50・52・58・62 文字では、`$TMPDIR` が `/tmp/claude-501` に戻り、渡した場所とユーザーごとの一時フォルダの両方に書けた。長さは渡した文字列で数える（`/tmp/…` で 44 文字、実体の `/private/tmp/…` で 52 文字の場所が使われた）。`denyRead` に短い一時フォルダの根とユーザーごとの一時フォルダ、`allowRead` に自分の短い一時フォルダを入れると、ユーザーごとの一時フォルダとほかのセッションの一時フォルダは、読めず、一覧も出せず、書けなかった。Bash は普通に動いた。同じ安全設定で 50 文字の場所を渡すと、`$TMPDIR` は `/tmp/claude-501` に戻ってファイルを作れたが、中は読めず一覧も出せなかった。公式文書（[環境変数][doc-env]）は、長いパスでは既定の場所の下の短い `$TMPDIR` を渡すと書くが、長いとみなす値は書いていない。上限が `CLAUDE_CODE_TMPDIR` だけの長さか、`/claude-<uid>` を足した長さかは、uid が 501 だけの試験では区別できない | 実行と公式文書 |

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
[doc-sandboxing]: https://code.claude.com/docs/en/sandboxing
[doc-env]: https://code.claude.com/docs/en/env-vars
