# 導入・準備・段階

## 導入と点検（前提の基盤を Plugin が作る）

dev-autopilot は、Project の欄・ラベル・ブランチ保護・設定ファイル・定期の実行の登録が揃っていないと動かない。これらを人が手で揃える前提にすると、
分離して別のリポに入れるたびに手順書が要り、揃っているかも人の記憶に頼ることになる。business-os の `/onboard`（雛形を展開して会社を作る）と `/check`（重い点検）と同じ形で、
導入と点検を Plugin に含める。

- SETUP-1 **導入（`/dev-autopilot setup`）**：足りないものを作る。何度実行しても同じ結果になる（冪等）。まず「何を作るか」の一覧を出し、人が承認してから作る。
  外部に影響が出る操作（ラベル・Project の欄・ruleset の作成）は 1 回の承認でまとめて行い、作った後に実物を読み戻して確かめる。
  利用者は、当てる先のリポに Plugin をプロジェクトの範囲で入れ、当てる先のリポで開いた Claude Code で `setup` を実行する（STRUCT-3、[ADR-20261017-003][adr-plugin] の決定 4）。`setup` は dev-autopilot 自身のリポでは何も作らずに止まる。
  `setup` は GitHub の準備とマシンの準備に分ける（[ADR-20261017-004][adr-human] の決定 2）。GitHub の準備（ラベル、Project の欄、ruleset、リポの設定）は人の `gh` のログイン（人の権限）で行う。マシンの準備（下の一覧のそれ以外）は GitHub の認証を使わず、秘密のファイルを読まない。
  人の `gh` のログインを持たない専用マシンでは、GitHub の準備は人の手元のマシンで行い、専用マシンではマシンの準備だけを行う。進行ボットの PAT には、ruleset を作る管理の権限を持たせない。
  Plugin を更新した後に `setup` を実行し直すと、進行ボットのプログラムの写しを新しい版で作り直す（進行ボットが動いていない間に限る。実行の lock を取る）
  - Plugin が作れるもの：ラベル（`agent-ready` `needs-human` `from-review` `plan:fable` `plan:opus`）、Project の Status の選択肢（In Review・Ready to Merge・Blocked。`updateProjectV2Field` は選択肢を丸ごと置き換えるので、既存の選択肢を `id` 付きで含めて送る。`id` を落とすと既存の ID と項目の値が消える。FACT-15）と単一選択欄「Agent」、
    base ブランチの ruleset（PR 必須、force push 禁止、必須チェック（設定の一覧）を strict で登録して「base と最新であること」を要求、`require_extra_approval_for_unattributed_changes` は無効。FACT-7）、
    deploy key の push 先を `<型>/issue-*` のブランチに限りタグの作成を禁じる ruleset（IMPL-15）、設定ファイル（リポの設定 `.claude/dev-autopilot.json`。Project の ID や選択肢の ID は API で引いて書く。マシンの設定はマシンごとに、OS ごとの決まった場所に当てる先のリポ（owner/repo）ごとに作り（STRUCT-2）、OS ごとの起動の定義（macOS は launchd の plist）がマシンの設定のパスを進行ボットに渡す。[ADR-20261015-005][adr-settings]）、
    進行ボットのプログラムの写し（インストールされた Plugin の版を、置き場の中の版ごとのフォルダに写す。STRUCT-3）、
    定期の実行の登録（macOS は launchd の plist、Linux は systemd のユーザーのタイマーか cron。STRUCT-8）、worktree の置き場、記録と lock の置き場（記録のフォルダには、作った日時を書いた印のファイルを置く。「[運用の規則][f01]」）、秘密の置き場（フォルダとファイルを権限 700・600 で作る。値は人が書く。SAFE-15）
  - 人がやるもの（Plugin は手順を示し、済んだかを点検で確かめる）：SSH 鍵の作成と GitHub への登録、fine-grained PAT の発行と印の署名の鍵（IMPL-20）の作成、どちらも秘密の置き場への保管（SAFE-15）、専用マシンの用意、利用枠の確認、
    Project の自動追加（組み込みワークフロー「Auto-add to project」を有効にする。公開 API が無いので人が行い、`check` は有効かを読むだけ）、
    処理対象にする Issue の承認（受け入れの下書きを読んで `agent-ready` を付ける。TEXT-12）
- SETUP-2 **点検（`/dev-autopilot check`）**：前提がすべて揃っているかを機械で確かめ、fail / warn / pass で返す。進行ボットは毎回の実行の最初に同じ点検を回し、
  fail が 1 つでもあれば作業に入らず止まる（fail-closed）。人が呼ぶ `check` は進行ボットを直接起動せず、定期の実行と同じジョブを起こす。GitHub の点検（ラベル、Project の欄、ruleset、PAT の権限）は進行ボットが自分の PAT で行い、人のセッションは実行の記録を読んで表示する（点検だけで動かす仕組みは基本設計で決める。[ADR-20261017-004][adr-human] の決定 2）。
  見るもの：ラベルと Project の欄の存在、ruleset、設定ファイルの項目と ID の実在と型（要件が「設定に持つ」と書いた項目がすべてあること。STRUCT-2）、必須チェックの一覧が空でないこと（IMPL-27）、進行ボットの PAT の権限、
  `claude` と Node の版（Node 24 が要る）、インストールされた Plugin の版と進行ボットのプログラムの写しの版の一致（違えば warn。STRUCT-3）、Skill の有無、前回の実行の記録と記録の印のファイル（無ければ fail。「[運用の規則][f01]」）、lock の状態（古い lock。「[運用の規則][f01]」）、子プロセスに PAT と署名の鍵が見えないこと（実際に試す）、sandbox が実際に読めない・書けないこと（「[安全側に倒す設計][f05]」）。
  作業場所の残骸は warn にし、Issue に紐づく残骸はその Issue だけ `needs-human`、どの Issue か分からない残骸だけ fail（何を残骸とみなすかは UNDECIDED-27）。
  AI の作業場所も実際に試す（[ADR-20261013-001][adr-workspace] の確認方法）：進行ボットの clone と AI の作業場所の clone に、リンク数が 2 以上の objects のファイルが無いこと、`.git/worktrees/<名前>/gitdir` が sandbox の中から書けないこと、objects を書き換えた clone から進行ボットの clone へ fetch すると失敗すること。
  AI の読み取りと秘密の置き場も実際に試す（[ADR-20261015-001][adr-read] の確認方法）：sandbox の中からホームのファイル・`<置き場>/controller/`・進行ボットのプログラムの写しの `src/`・別のセッション・秘密の置き場を読めず、自分のセッションの clone と worktree は読めること、
  写しの `skills/` は sandbox の中と Read ツールの両方から読めて書けないこと、置き場の実体のパスが `/tmp/claude-<uid>/` の下に無いこと（[ADR-20261017-004][adr-human] の確認方法）、SAFE-10 の環境変数で `git commit` と `pnpm install` が通ること、Read ツールで worktree の外を読むと拒否されること、秘密の置き場の権限（700・600）と実体のパスが同期フォルダの下に無いこと、OS ごとの起動の定義（macOS は launchd の plist）に秘密の値が無いこと。
  依存の store も試す（[ADR-20261015-002][adr-store] の確認方法）：sandbox の中の `pnpm store path` がセッションの一時フォルダの下を返すこと、安全設定の `allowRead` と `allowWrite` に共有の store が無いこと、子プロセスに渡す環境変数の名前に `PNPM_HOME` が無いこと。
  設定も確かめる（[ADR-20261015-005][adr-settings] の確認方法）：すべての項目がリポの設定かマシンの設定の一方にだけあること、リポの設定に絶対パス（`/Users/` `/home/` などで始まる値）が無いこと、マシンの設定の実体のパスが同期フォルダの下に無いこと、人が呼ぶ Skill が origin から引いたマシンの設定の場所と、定期の実行の定義が渡す場所が同じこと（[ADR-20261017-004][adr-human] の決定 3）。
  進行ボットのプログラムの写しと clone も試す（[ADR-20261016-003][adr-move]、[ADR-20261017-003][adr-plugin] の確認方法）：進行ボットのプログラムの写しと `<置き場>/controller/` が別のフォルダで、`<置き場>/controller/` は sandbox の中から読めず書けず、写しは書けず `src/` を読めないこと、
  定期の実行が写しから動くこと、Plugin を更新しても写しが変わらないこと。
  MCP の道具が無いことも試す（[ADR-20261017-005][adr-mcp] の決定 4）：AI のセッションと同じ起動の形（安全設定・環境変数・起動の引数）で短いセッションを起こし、初期化の記録の道具の一覧に `mcp__` で始まる道具が無く、MCP のサーバーの一覧が空であること（外れたら fail。確かめる回数は利用枠との兼ね合いで基本設計で決める）。
  同じ短いセッションで、sandbox の中の一時フォルダも試す（[ADR-20261017-006][adr-tmp] の決定 4）：`$TMPDIR` が渡した短い一時フォルダの下の `claude-<uid>` で書けること、Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）が読めず一覧も出せず書けないこと、
  `check` が短い一時フォルダの根の下に作った試験用のフォルダ（ほかのセッションの一時フォルダの代わり）が読めず書けないこと（外れたら fail）
- SETUP-3 **設定の正本は項目ごとに 1 つ**（リポの設定かマシンの設定のどちらか。STRUCT-2）。導入が書き、点検が読み、進行ボットが使う。人が直接編集してもよいが、点検が整合を確かめる（[ADR-20261015-005][adr-settings]）
- SETUP-4 **business-os への導入は、この仕組みの最初の利用者として `setup` で行う**。手で作らない。手で作ると、ほかのリポに当てるときに `setup` が動く保証が無くなる

## いまのリポジトリに無いもの（準備の作業）

SETUP-5・SETUP-6・SETUP-8・SETUP-9 は、「導入と点検」の `setup` が作る。SETUP-7 は運用の中で進行ボットが入れる。人が手で作るのは SETUP-10（鍵）と、deploy key と PAT と印の署名の鍵（「[隔離と鍵の分け方][f05]」）だけ。

準備が済んでも、行は消さない。「対応」の欄には、済んだ後も有効な要件（例：SETUP-14 の、Issue のフォームに欄を足さないこと）があり、行の番号はほかのファイルから参照されるため。
済んだ行は、「足りないもの」の欄の頭に `済み（YYYY-MM-DD に確認）：` と書き添える。

| 番号 | 足りないもの | 対応 |
|---|---|---|
| SETUP-5 | Project の Status が Todo / In Progress / Done の 3 つ | In Review・Ready to Merge・Blocked を足す。更新は GraphQL の `updateProjectV2ItemFieldValue` で行える（公式文書で確認） |
| SETUP-6 | AI を担当者にできない（コラボレータは 1 人） | Project に単一選択欄「Agent」を足す。Assignee は人のまま |
| SETUP-7 | Issue 依存（blocked by）が 1 件も入っていない | 受け入れが候補（本文の `#n` の参照）を出し、人が下書きで直し、`agent-ready` の後に進行ボットが確定して入れる（TEXT-12）。読み書きは REST の `/issues/{n}/dependencies/blocked_by` で行える（公式文書で確認） |
| SETUP-8 | 処理対象を示すラベルが無い | `agent-ready` `needs-human` `from-review` `plan:fable` `plan:opus` を `setup` が足す。名前は設定ファイルに持ち、ここに書いた名前は初期値 |
| SETUP-9 | `develop` にブランチ保護が無い | ruleset で PR 必須にする（自律セッションの直 push を仕組みで止める） |
| SETUP-10 | 済み（2026-10-16 に確認）：コミットの署名が 1Password のソケット頼み | 人の署名は手元の SSH 鍵に移す（`gpg.format ssh`、`user.signingkey` を鍵ファイルに）。dev-autopilot は人の鍵を使わない。作業 AI の commit は署名なし、`develop` への squash マージの commit は GitHub が署名する（SAFE-14）。business-os の雛形（ADR 20261003-11）は利用者向けなので変えない |
| SETUP-11 | 開発専用の TypeScript の置き場が無い（tsconfig は `evals/` `plugin/hooks/` `plugin/scripts/` `test/` だけ） | ~~開発物のフォルダ `dev-autopilot/` を 1 つ足し、構造仕様 3.1 節の表に加える。~~ dev-autopilot は別のリポに移した（[ADR-20261016-003][adr-move]）ので、そのリポの root に `src/` と `tsconfig.json` を置く。中身は「[分離を見越した構成][f07]」の構成 |
| SETUP-12 | `claude -p` の実行は利用枠を消費する | 1 回の実行・1 日あたりの上限を設定に持つ |
| SETUP-13 | 作業 AI とレビュー AI が同じ GitHub アカウント（メンテナ）で投稿することになる | GitHub は PR の作者自身の approve と request changes を受け付けない。いまの運用も GitHub のレビュー状態を使わず、PR のコメントで判定している。dev-autopilot も同じで、判定は PR のコメントの固定の印（IMPL-18）で行う。アカウントは最初はメンテナの 1 つで始め、無人のマシンにはこのリポだけに絞った fine-grained PAT を置く。人と AI の投稿は印で見分ける。監査のために分けたくなったら、自動化用のアカウントをコラボレータに足す（判定の仕組みは変えずに済む） |
| SETUP-14 | Issue のフォームに受け入れ条件の欄が無い | 外から入る Issue は報告・提案の形で、作業の定義ではない。コラボレータの作業指示コメント（TEXT-1）で補う。フォームには足さない（外の人が書く欄にしない） |
| SETUP-15 | 渡す文が後から編集されうる | 処理対象にした時点のハッシュで検出する（TEXT-4）。外の人の文は渡さないので、外の人の編集は影響しない |
| SETUP-16 | Issue を確かめる手順が dev-autopilot と結びついていない | 受け入れ（TEXT-12）を進行ボットが自動で行い、人の操作は `agent-ready` だけにする。順「受け入れの下書き（自動）→ `agent-ready`（人）」を利用の手順に書く。コラボレータの Issue も対象（TEXT-9）。`/verify-issue` は手動の補助 |
| SETUP-17 | 指示の混入の検査と作業指示の下書きを行うものが無い。`/verify-issue` は外の人の不具合の報告・改善の提案を人が確かめる手順として書かれている | 検査と下書きは進行ボットの受け入れ（TEXT-12。`intake` エージェントと検疫のモジュール）が自動で行う。機械で探せる検査（見えない文字、HTML コメント、base64、URL、指示の形の語）は TypeScript にして vitest で検査する。`/verify-issue` は手動の補助として残し、最初の手順で同じ検疫のモジュールを呼ぶ。着手の条件は dev-autopilot の ADR に書く |
| SETUP-18 | レビュー AI は PR のコードを実行して確かめる。その環境に GitHub のトークンや鍵があると、作業 AI が書いたコードから読み取れてしまう。開発リポは sandbox を使わないので、同じ OS ユーザーで動く限り `~/.ssh` と `~/.config/gh` は Bash から読める（`Read(~/.ssh/**)` の deny は Read ツールにしか効かない） | AI のセッションには `GH_TOKEN` を渡さず、`gh` を認証しない。GitHub への書き込みはすべて進行ボットが行う（IMPL-5）。AI のセッションに sandbox を掛け、鍵を分ける（「[隔離と鍵の分け方][f05]」） |
| SETUP-19 | `change-review` スキルが利用者の全体設定（`~/.claude/skills/`）にあり、dev-autopilot の専用マシンでは無いかもしれない | レビューの Skill の正本を dev-autopilot の中（`plugin/skills/review/`）に置き、dev-autopilot が自分の Skill として読み込む。`~/.claude` には依存しない（「[分離を見越した構成][f07]」） |

## 段階

段階の表はここが唯一の正本。出口は数で書き、満たしたかは記録（FLOW-5）から数える。

| 段階 | 内容 | 出口 |
|---|---|---|
| 1 | 橋渡し ADR 20261008-01（business-os 側）を proposed で起票 | accepted（済み） |
| 2 | 最初の薄切り：進行ボットなし・GitHub 書き込みなしで、手で書いた安全設定（`setup` はまだ無い）を `--settings` で渡した `claude -p` を未信頼の worktree の中で動かし、「[実機で確かめた事実][f10]」の表の未確認を実機で潰す。攻撃の再現（project の hook が走らないこと、`commondir` の差し替えが進行ボットの git に届かないこと）を含む | 「[実機で確かめた事実][f10]」の表の未確認がすべて「実行」で埋まる（済み） |
| 3 | `setup` と `check` の実装（「導入と点検」）と、それによる準備（SETUP-5・SETUP-6・SETUP-8・SETUP-9）。deploy key と PAT と印の署名の鍵の用意（「[隔離と鍵の分け方][f05]」）。`change-review` スキルの v2（「[レビューの Skill][f08]」）。受け入れ（TEXT-12）と `/verify-issue` の拡張 | `check` が business-os で全項目 pass（`setup` が書いた安全設定で実体のパスの解決と読み取りの検査が動くことを含む）。スキルの v2 は手動のレビュー 1 件で今の PR の形が出ることを確かめる |
| 4 | 棚卸しと受け入れ（TEXT-12）を動かす（選ぶ・待たせる・下書きを書く・記録する。実装はしない）。最初は設定 `dryRun: true` で、下書きと選定の結果を手元の記録にだけ書いて人が読む。異議が無ければ `dryRun` を切り、残りの週は Issue に実際に投稿する | 7 回の実行（lock で見送った実行と起動されなかった日は数えない。「[運用の規則][f01]」）で、人が `## 異議` の見出しで書いた回数（dry-run の間は記録ファイルへの追記）0。うち投稿ありの実行が 3 回以上 |
| 5 | 計画 AI と実装 → draft PR を 1 件ずつ。計画（IMPL-10）は人が見る。レビューは人が行い、PR のマージも人。モデルの比較もここで行う：作業 AI は `opus` で始め、種類ごとに `sonnet` を試す（IMPL-25 の上げた件数を数える）。計画 AI は受け入れの `plan=` の判定を人が妥当と見たか、`fable` にした計画が `opus` と比べて違ったかを記録する | 5 件の PR のうち 4 件以上を「手直しなし」で人がマージ（手直し＝人が PR に commit を足した、または close して作り直した）。`plan=` の判定への `## 異議` の割合を記録する |
| 6 | レビュー AI と批評 AI を足す（`review` Skill を `claude -p` で動かし、印を足す）。レビュー AI は `opus`（xhigh）を既定に、同じ PR を `fable` でも回して比べる（比較の実行は印を書かない。IMPL-19） | 5 件の PR で、人が同じ PR を `change-review` した結果（人の 🔴 🟡 の見出しの一覧）と照合し、一致率（人の 🔴 🟡 のうち AI も出した割合）が 80% 以上、却下した指摘の割合（AI だけが出した 🔴 🟡 のうち `## 却下` に書かれた割合）が 15% 以下。`opus` と `fable` の一致率・却下率・利用枠を並べ、既定のモデルを決める。人の二重レビューはこの 5 件に限る |
| 7 | 対応ループを足す。マージは人。進行ボットは「[マージ][f03]」の判定を記録に残すだけで印は書かない（shadow 判定） | 5 件のうち 4 件以上が往復 3 回以内で Ready to Merge |
| 8 | 進行ボットのマージの判定（「[マージ][f03]」）の印を書く。マージの実行は人のまま | 10 件で、人のマージの判断と進行ボットの判定（`merge=ready` / `merge=human`）が全件一致（MERGE-3 の数え方）。revert 0 件（revert＝マージから 14 日以内に、`develop` にその squash commit を revert する commit か、本文でその PR 番号を「戻す」と言及する fix の PR がマージされたもの）。満たしても実行は人のまま。実行を進行ボットに移すかは別の ADR で決める（MERGE-9） |
| 9 | 並列（複数 worktree。WIP の上限を 1 から上げる）と間隔の短縮 | FLOW-12 の条件（連続 7 回の実行で `needs-human` 0 件、利用枠が上限内）で判断 |

[f01]: ./01-issue-flow.md
[f03]: ./03-merge.md
[f05]: ./05-safety.md
[f07]: ./07-structure.md
[f08]: ./08-review-skill.md
[f10]: ./10-verified-facts.md
[adr-workspace]: ../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-store]: ../adr/adr-20261015-002-no-shared-dependency-store.md
[adr-settings]: ../adr/adr-20261015-005-settings-values-and-machine-settings.md
[adr-move]: ../adr/adr-20261016-003-move-to-own-repository-before-basic-design.md
[adr-plugin]: ../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-human]: ../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-mcp]: ../adr/adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[adr-tmp]: ../adr/adr-20261017-006-per-session-short-tmpdir.md
