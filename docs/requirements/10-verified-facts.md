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
| FACT-14 | 道具なし、`--json-schema`、強制終了時の費用 | 道具なしはエージェント定義の `tools: []` でも `--tools ""` でも成立（Read を頼んでも「道具が無い」と返す）。**`--json-schema` は `--agent` と併用すると `structured_output` が null になる**（道具の有無を問わず。`--agent` 無しなら `structured_output` に検証済みの JSON が入る）。進行ボットは `result` の文を自分の JSON スキーマで検証する（IMPL-5）。`--output-format stream-json` は `assistant` のイベントごとに `model` と `usage` を持つので、途中で止めても消費は集計できる。プロセスグループごとの kill（`setpgrp` で起動し `kill -TERM -- -<pgid>`）で孫プロセスも残らなかった（モデルが長いコマンドを実行しなかったため、`claude` 自体を途中で止める試験は未実施） | 実行（kill は model なし） |
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

[adr-workspace]: ../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-store]: ../adr/adr-20261015-002-no-shared-dependency-store.md
[pnpm-store]: https://pnpm.io/settings/store
[bos-issue-37]: https://github.com/Joymerrevent/business-os/issues/37
[bos-issue-44]: https://github.com/Joymerrevent/business-os/issues/44
[bos-pr-45]: https://github.com/Joymerrevent/business-os/pull/45
[bos-issue-80]: https://github.com/Joymerrevent/business-os/issues/80
