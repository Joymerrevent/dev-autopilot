# 実機で確かめた事実

## 段階 2 で実機で確かめた事実（2026-10-08〜11）

macOS、Claude Code 2.1.285、haiku、`claude -p --output-format json` で、business-os のローカル clone（未信頼の作業場所）から確かめた。入れ子の Plugin は `tmp/` に作った偽物（名札・agent 1 体・Skill 1 つ）。

| # | 確かめたこと | 結果 | 証拠 |
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
| FACT-12 | `commondir` を差し替えた worktree に対する進行役の操作 | **worktree の中で git を実行すると発火した**（`git -C <worktree> status` で差し替え先の `core.fsmonitor` が動いた）。進行役の clone から worktree のパスへの fetch、親 clone のパスへの fetch、親 clone での `git worktree list` `git worktree remove --force`、親 clone での `git log <ブランチ>` は**発火しなかった**。SAFE-2 の (b) のとおり進行役が worktree の中で git を実行しなければ届かない | 実行 |
| FACT-13 | sandbox の `denyRead` が Read ツールにも効くか | **効かない**（公式文書どおり。sandbox は Bash だけを覆う）。Read ツールは `denyRead` のパスを読めた。`permissions.deny` の `Read(//<絶対パス>/**)`（絶対パスは `//` 始まり。`/` 始まりはプロジェクト相対で一致しない）で止まった（`File is in a directory that is denied by your permission settings`）。Write・Edit も同じで、ホームの任意のパスに書けた。`permissions.allow` を `Write(//<worktree>/**)` `Edit(//<worktree>/**)` に絞ると、外への Write は「Path is outside allowed working directories」で拒否され、`deny` に入れた `.git/**` `.claude/**` `CLAUDE.md` への Edit も止まった。`.git/hooks/**` と `.mcp.json` は Write ツールが「sensitive file」として既定で拒む | 実行 |
| FACT-14 | 道具なし、`--json-schema`、強制終了時の費用 | 道具なしはエージェント定義の `tools: []` でも `--tools ""` でも成立（Read を頼んでも「道具が無い」と返す）。**`--json-schema` は `--agent` と併用すると `structured_output` が null になる**（道具の有無を問わず。`--agent` 無しなら `structured_output` に検証済みの JSON が入る）。進行役は `result` の文を自分の JSON スキーマで検証する（IMPL-5）。`--output-format stream-json` は `assistant` のイベントごとに `model` と `usage` を持つので、途中で止めても消費は集計できる。プロセスグループごとの kill（`setpgrp` で起動し `kill -TERM -- -<pgid>`）で孫プロセスも残らなかった（モデルが長いコマンドを実行しなかったため、`claude` 自体を途中で止める試験は未実施） | 実行（kill は model なし） |
| FACT-15 | Project v2 の Status に選択肢を足す GraphQL が、既存の選択肢の ID と項目の値を保つか | **`updateProjectV2Field` の `singleSelectOptions` に既存の選択肢の `id` を含めて渡せば、既存の ID と項目の値は保たれ、同じ入力を 2 回送っても ID は変わらない（冪等）。`id` を渡さないと既存の ID がすべて振り直され、項目の Status の値も消える**（2026-10-11、Organization に作った捨てる Project で確認し、削除した）。`setup` は必ず現在の選択肢を読んで `id` 付きで送り、送る前に件数と名前を読み戻して照合する | 実行 |
| FACT-16 | リアクションや非表示で `updated_at` が変わるか | **リアクションでは変わらない。非表示（`minimizeComment`）では `updated_at` が変わる**が `lastEditedAt` は null のまま、`userContentEdits.totalCount` は 0。IMPL-20 の (b) は `created_at` と `updated_at` の一致でなく、GraphQL の `lastEditedAt == null`（または `userContentEdits.totalCount == 0`）で判定する | 実行 |
| FACT-17 | 子プロセスの環境変数を絞る。`GIT_CONFIG_COUNT` | `env -i` に `HOME` `PATH` `USER` `LOGNAME` `TMPDIR` `TERM` `LANG` だけを渡せば `claude -p` はログイン状態を保って動き、渡さなかった変数は Bash から `unset` に見えた（`HOME` と `PATH` だけではログインが失われる）。`GIT_CONFIG_COUNT` と `GIT_CONFIG_KEY_n` / `GIT_CONFIG_VALUE_n` で `core.hooksPath` と `gc.auto` が効いた | 実行 |

**見つかった不具合（🔴）：sandbox の `denyRead` はシンボリックリンクを解決しない。**
この Mac では `~/.config` が 同期フォルダ上の dotfiles へのシンボリックリンクで、`denyRead` に `~/.config/gh/**` と書いても `~/.config/gh/hosts.yml` が読めた（`~/.config/gh` と glob なしでも同じ）。
実体のパス（`<同期フォルダ>/dotfiles/.config/gh/**`）を書くと止まった。`~/.ssh` も同じ dotfiles へのリンクだが、こちらは `~/.ssh/**` で止まった（リンクがパターンの先頭にあるときは解決されるように見える。途中にあると解決されない）。

- dev-autopilot への反映：`setup` が安全設定を書くとき、`denyRead` と `denyWrite` の各パスを実体に解決してから書く（リンクの場合は両方のパスを書く）。
  `check` は「設定に書いてある」ではなく「sandbox の中で実際に読めないこと」を、読み取りを試して確かめる（fail-open を機械で検出する）
- business-os への反映：`plugin/templates/settings.json.tmpl` の `denyRead` の `~/.config/gh/**` `~/.aws/**` が、`~/.config` をリンクにしている利用者では効かない。
  利用者向けの不具合なので Issue #80 に起票した（2026-10-08）
- 人への注意：`~/.ssh` の実体がクラウドの同期フォルダにある。dev-autopilot 用の鍵も人の署名鍵も、同期されないパスに置く

FACT-6〜17 は 2026-10-11 に、手で書いた安全設定と偽の入れ子 Plugin（scratchpad の実験場。親 clone と進行役の clone と worktree 2 つ）と捨てる Project と、人が発行した PAT で実機確認した。段階 2 の未確認は残っていない。
「`setup` が書いた安全設定で実体のパスの解決と読み取りの検査が動くこと」は段階 3 の出口。

## 要件の見直しの途中で確かめた事実（2026-10-12）

使い捨てのリポジトリ（macOS、git 2.50.1）で確かめた。Claude Code と sandbox は使わず、AI による書き換えは、人の手で objects のファイルを書き換えて再現した。
sandbox の中の AI が既存の objects のファイルを書き換えられるかは確かめていない（FACT-9 では、commit で新しい objects を書けた）。

| # | 確かめたこと | 結果 | 証拠 |
|---|---|---|---|
| FACT-18 | clone の objects のファイルを書き換えたとき、同じ clone から作る worktree、`git fetch origin`、別の clone への送り出しがどうなるか | **worktree には書き換えた中身がエラーなしで出た**（`git worktree add --detach <SHA>` が終了コード 0）。`git fsck` は hash-path mismatch を出した。同じ clone で `git fetch --no-tags origin` を実行しても直らなかった（手元にある objects は取り直さない）。origin から新しく clone して同じ commit で worktree を作ると、正しい中身が出た。書き換えた clone から `file://` で別の場所へ clone すると、受け取り側の検査で `did not receive expected object` になり失敗した | 実行（sandbox なし） |
| FACT-19 | 手元のパスから `git clone` したとき、objects のファイルを共有するか | **パスから clone すると、ハードリンクで共有した**（objects のファイルのリンク数 2）。`--no-hardlinks` を付けたときと、`file://` の URL から clone したときは共有しなかった（リンク数 1） | 実行（sandbox なし） |
