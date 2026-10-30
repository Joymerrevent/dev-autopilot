# 導入・準備・段階

## 導入と点検（前提の基盤を Plugin が作る）

dev-autopilot は、Project の欄・ラベル・ブランチ保護・設定ファイル・定期の実行の登録が揃っていないと動かない。これらを人が手で揃える前提にすると、
分離して別のリポに入れるたびに手順書が要り、揃っているかも人の記憶に頼ることになる。business-os の `/onboard`（雛形を展開して会社を作る）と `/check`（重い点検）と同じ形で、
導入と点検を Plugin に含める。

- SETUP-1 **導入（`/dev-autopilot setup`）**：足りないものを作る。何度実行しても同じ結果になる（冪等）。まず「何を作るか」の一覧を出し、人が承認してから作る。
  外部に影響が出る操作（ラベル・Project の欄・ruleset の作成）は 1 回の承認でまとめて行い、作った後に実物を読み戻して確かめる。
  利用者は、当てる先のリポに Plugin をプロジェクトの範囲で入れ、当てる先のリポで開いた Claude Code で `setup` を実行する（STRUCT-3、[ADR-20261017-003][adr-plugin] の決定 4）。`setup` は dev-autopilot 自身のリポでは何も作らずに止まる。
  `setup` は GitHub の準備とマシンの準備に分ける（[ADR-20261017-004][adr-human] の決定 2）。GitHub の準備（ラベル、Project の欄、ruleset、リポの設定、全体の止める合図の変数、ruleset の控えの変数（[ADR-20261028-004][adr-bypass-baseline]））は、`setup` 用の GitHub App の人の代わりのトークン（`setup` の実行ごとに人がブラウザで承認する。トークンはファイルに書かない）で行い、人の `gh` のログインは使わない（[ADR-20261030-001][adr-setup-app]）。マシンの準備（下の一覧のそれ以外）は GitHub の認証を使わず、秘密のファイルを読まない。
  App を作る手順は、マニフェストの戻り先が手元なので、人のブラウザと同じマシンで行う。進行 Bot の GitHub App には、ruleset と変数を作る権限を持たせない。
  Plugin を更新した後に `setup` を実行し直すと、進行 Bot のプログラムの写しを新しい版で作り直す（進行 Bot が動いていない間に限る。実行の lock を取る）
  - Plugin が作れるもの：ラベル（人が付ける `agent-ready` `agent-paused` `agent-manual`、進行 Bot が付ける表示の `security-suspected` と `from-review`。SETUP-8）、Project の表示の欄（主な流れの枠・状態・止め方・先の依頼・安全の単一選択欄と選択肢。状態は、もともとある Status 欄を名前を変えずに使い、選択肢を 17 の状態に置き換える。置き換えると今の選択肢と項目の Status の値が失われるので、作るものの一覧に出して人の承認を受ける。ほかの 4 つは単一選択の欄として新しく作る。FLOW-39、[ADR-20261026-001][adr-status]）と、軸ごとのビュー（FLOW-39）。`updateProjectV2Field` は選択肢を丸ごと置き換えるので、既存の選択肢を `id` 付きで含めて送る。`id` を落とすと既存の ID と項目の値が消える（FACT-15）、
    base ブランチの ruleset（PR 必須、force push 禁止、必須チェック（設定の一覧）を strict で登録して「base と最新であること」を要求、`require_extra_approval_for_unattributed_changes` は無効。FACT-7）、
    deploy key の push 先を `<型>/issue-*` のブランチに限りタグの作成を禁じる ruleset（IMPL-15。2 つの ruleset とも、bypass は組織の管理者と、リポの設定の一覧で人が許したアプリで作る。bypass に入れてよいのは組織の管理者とチームと一覧のアプリだけで、リポの役を入れると、書き込みのある deploy key も規則を素通りする。FACT-65、FACT-67）、全体の止める合図のリポの変数（値は `stop`。人が `check` の後に `run` にする。「[運用の規則][f01]」、[ADR-20261017-007][adr-stop]）、設定ファイル（リポの設定 `.claude/dev-autopilot.json`。Project の ID や選択肢の ID は API で引いて書く。マシンの設定はマシンごとに、OS ごとの決まった場所に当てる先のリポ（owner/repo）ごとに作り（base ブランチの名前もここに書く。STRUCT-2）、OS ごとの起動の定義（macOS は launchd の plist）がマシンの設定のパスを進行 Bot に渡す。[ADR-20261015-005][adr-settings]）、
    進行 Bot のプログラムの写し（インストールされた Plugin の版を、置き場の中の版ごとのフォルダに写す。STRUCT-3）、
    定期の実行の登録（macOS は launchd の plist、Linux は systemd のユーザーのタイマー。STRUCT-8）、worktree の置き場、記録と lock の置き場（記録のフォルダには、作った日時を書いた記録の目印のファイルを置く。「[運用の規則][f01]」）、秘密の置き場（フォルダとファイルを権限 700・600 で作る。値は人が書く。SAFE-15）
  - 人がやるもの（Plugin は手順を示し、済んだかを点検で確かめる）：SSH 鍵の作成と GitHub への登録、印の署名の鍵の一覧（IMPL-20）の作成と秘密の置き場への保管（SAFE-15）、2 つの GitHub App（進行 Bot 用と `setup` 用）の作成と導入の画面での操作（`setup` が作り、人は「Create」と導入を押す。`setup` 用の App は、作る前に Device Flow を有効にし、作った後に `client_id` を 1 回 `setup` に渡す。[ADR-20261020-002][adr-app]、[ADR-20261030-001][adr-setup-app]）、`setup` のたびの承認のコードの入力、専用マシンの用意、利用枠の確認、
    Project の自動追加（組み込みワークフロー「Auto-add to project」を有効にする。公開 API が無いので人が行い、`check` は有効かを読むだけ）、CI が秘密を使うなら、Environment に置き、使えるブランチを base ブランチと人が決めたブランチだけにすること（SAFE-20）、
    処理対象にする Issue を決めること（マスターの下書きを読み、要れば直すか差し戻してから、`agent-ready` を付ける。TEXT-12）
- SETUP-2 **点検（`/dev-autopilot check`）**：前提がすべて揃っているかを機械で確かめ、fail / warn / pass で返す。進行 Bot は毎回の実行の最初に同じ点検を回し、
  fail が 1 つでもあれば作業に入らず止まる（fail-closed）。人が呼ぶ `check` は進行 Bot を直接起動せず、定期の実行と同じジョブを起こす。GitHub の点検（ラベル、Project の欄、ruleset、App の権限）は進行 Bot が自分の GitHub App で行い、人のセッションは実行の記録を読んで表示する（点検だけで動かす仕組みは、基本設計の「[導入と点検][design-setup]」の「点検だけで動かす仕組み」で決めた。[ADR-20261017-004][adr-human] の決定 2）。
  見るもの：ラベルと Project の欄の存在、ruleset（bypass にリポの役・deploy key・一覧の外の GitHub App・進行 Bot のアプリが入っていれば fail。入れてよいのは組織の管理者とチームと、deploy key の push 先を限る ruleset ではリポの設定の一覧で人が許したアプリだけ。FACT-65、FACT-67。進行 Bot の GitHub App のトークンでは `bypass_actors` がいつも返らない（FACT-88・91）ので、`setup` が bypass を確かめたときに控えた ruleset の `updated_at` と比べ、App 自身が bypass に入っていないこと（`current_user_can_bypass`。FACT-90）を確かめる（[ADR-20261028-004][adr-bypass-baseline]）。確かめ方は、基本設計の「[導入と点検][design-setup]」）、設定ファイルの項目と ID の実在と型（要件が「設定に持つ」と書いた項目がすべてあること。STRUCT-2）、必須チェックの一覧が空でないこと（IMPL-27）、進行 Bot の GitHub App の権限（全体の止める合図の変数は読めて、変えられないこと）、マシンの設定に GitHub App があり、秘密の置き場に PAT のファイルが無いこと（[ADR-20261020-002][adr-app]、[ADR-20261030-002][adr-no-pat]）、
  印の署名の鍵の一覧に 2 つ以上の鍵があるときは、先頭でない鍵でだけ確かめられた印が残る開いた Issue と PR の番号と、残りが無くなったら前の鍵を一覧から外してよいこと（warn。[ADR-20261021-004][adr-marker-keys]）、
  全体の止める合図の変数があって読めること、origin に base ブランチがあり、base ブランチの先端にリポの設定があり、ruleset が base ブランチを守っていること（[ADR-20261017-007][adr-stop]）、当てる先のリポの CI の秘密の置き方（リポの秘密が無く、組織の秘密が届いておらず、Environment の使えるブランチが作業ブランチと PR に当たらないこと。SAFE-20、[ADR-20261028-001][adr-ci-secrets]。外れていれば fail）、
  `claude` と Node の版（Node 24 が要る）、インストールされた Plugin の版と進行 Bot のプログラムの写しの版の一致（違えば warn。STRUCT-3）、Skill の有無、前回の実行の記録と記録の目印のファイル（無ければ fail。「[運用の規則][f01]」）、lock の状態（古い lock。「[運用の規則][f01]」）、子プロセスに GitHub App の秘密鍵とトークンと印の署名の鍵の一覧が見えないこと（実際に試す）、sandbox が実際に読めない・書けないこと（「[安全側に倒す設計][f05]」）。
  作業場所の残骸は warn にし、Issue に紐づく残骸はその Issue だけ停止中、どの Issue か分からない残骸だけ fail（何を残骸とみなすかは、基本設計の「[状態遷移と判定][design-flow]」の「作業場所の残骸」で決めた）。
  AI の作業場所も実際に試す（[ADR-20261013-001][adr-workspace] の確認方法）：進行 Bot の clone と AI の作業場所の clone に、リンク数が 2 以上の objects のファイルが無いこと、`.git/worktrees/<名前>/gitdir` が sandbox の中から書けないこと、objects を書き換えた clone から、進行 Bot の clone と同じ取り込みの設定（git に渡す環境変数と設定）で fetch すると失敗すること（取り込む先は、進行 Bot の clone を汚さないように、試しのためだけに作る小さなリポにする。基本設計の「[導入と点検][design-setup]」の「モデルを使わない試し」）。
  AI の読み取りと秘密の置き場も実際に試す（[ADR-20261015-001][adr-read] の確認方法）：sandbox の中からホームのファイル・`<置き場>/controller/`・進行 Bot のプログラムの写しの `src/`・別のセッション・秘密の置き場を読めず、自分のセッションの clone と worktree は読めること、
  写しの `skills/` は sandbox の中と Read ツールの両方から読めて書けないこと、置き場の実体のパスが `/tmp/claude-<uid>/` の下に無いこと（[ADR-20261017-004][adr-human] の確認方法）、SAFE-10 の環境変数で `git commit` と `pnpm install` が通ること、Read ツールで worktree の外を読むと拒否されること、秘密の置き場の権限（700・600）と実体のパスが同期フォルダの下に無いこと、OS ごとの起動の定義（macOS は launchd の plist）に秘密の値が無いこと。
  依存の store も試す（[ADR-20261015-002][adr-store] の確認方法）：sandbox の中の `pnpm store path` がセッションの一時フォルダの下を返すこと、安全設定の `allowRead` と `allowWrite` に共有の store が無いこと、子プロセスに渡す環境変数の名前に `PNPM_HOME` が無いこと。
  設定も確かめる（[ADR-20261015-005][adr-settings] の確認方法）：すべての項目がリポの設定かマシンの設定の一方にだけあること、リポの設定に絶対パス（`/Users/` `/home/` などで始まる値）が無いこと、マシンの設定の実体のパスが同期フォルダの下に無いこと、人が呼ぶ Skill が origin から引いたマシンの設定の場所と、定期の実行の定義が渡す場所が同じこと（[ADR-20261017-004][adr-human] の決定 3）。
  進行 Bot のプログラムの写しと clone も試す（[ADR-20261016-003][adr-move]、[ADR-20261017-003][adr-plugin] の確認方法）：進行 Bot のプログラムの写しと `<置き場>/controller/` が別のフォルダで、`<置き場>/controller/` は sandbox の中から読めず書けず、写しは書けず `src/` を読めないこと、
  定期の実行が写しから動くこと、Plugin を更新しても写しが変わらないこと。
  MCP の道具が無いことも試す（[ADR-20261017-005][adr-mcp] の決定 4）：AI のセッションと同じ起動の形（安全設定・環境変数・起動の引数）で短いセッションを起こし、初期化の記録の道具の一覧に `mcp__` で始まる道具と、`Agent`（初期化の記録では `Task` の名前で出るので、両方を探す。[ADR-20261017-002][adr-review] の確認方法）が無く、MCP のサーバーの一覧が空であること（外れたら fail。確かめる回数は、利用枠との兼ね合いで、基本設計の「[導入と点検][design-setup]」の「組と回数」で決めた）。
  同じ短いセッションで、sandbox の中の一時フォルダも試す（[ADR-20261017-006][adr-tmp] の決定 4）：`$TMPDIR` が渡した短い一時フォルダの下の `claude-<uid>` で書けること、Claude Code のユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）が読めず一覧も出せず書けないこと、
  `check` が短い一時フォルダの根の下に作った試験用のフォルダ（ほかのセッションの一時フォルダの代わり）が読めず書けないこと（外れたら fail）
- SETUP-3 **設定の正本は項目ごとに 1 つ**（リポの設定かマシンの設定のどちらか。STRUCT-2）。導入が書き、点検が読み、進行 Bot が使う。人が直接編集してもよいが、点検が整合を確かめる（[ADR-20261015-005][adr-settings]）
- SETUP-4 **business-os への導入は、この仕組みの最初の利用者として `setup` で行う**。手で作らない。手で作ると、ほかのリポに当てるときに `setup` が動く保証が無くなる

## いまのリポジトリに無いもの（準備の作業）

SETUP-5・SETUP-6・SETUP-8・SETUP-9 は、「導入と点検」の `setup` が作る。SETUP-7 は運用の中で進行 Bot が入れる。人が手で作るのは SETUP-10（鍵）と、deploy key と印の署名の鍵の一覧（「[隔離と鍵の分け方][f05]」）だけ。

準備が済んでも、行は消さない。「対応」の欄には、済んだ後も有効な要件（例：SETUP-14 の、Issue のフォームに欄を足さないこと）があり、行の番号はほかのファイルから参照されるため。
済んだ行は、「足りないもの」の欄の頭に `済み（YYYY-MM-DD に確認）：` と書き添える。

| 番号 | 足りないもの | 対応 |
|---|---|---|
| SETUP-5 | Project の Status が Todo / In Progress / Done の 3 つ | ~~In Review・Ready to Merge・Blocked を足す。~~ Status 欄は名前を変えずに状態の欄として使い、選択肢を主な流れの 17 の状態に置き換える（Status 欄の名前は API で変えられない。FACT-74）。あわせて、主な流れの枠・止め方・先の依頼・安全の単一選択欄を足す。Status 欄に値を書く組み込みのワークフロー（閉じたら Done、マージしたら Done など）と、Status 欄の値で Issue を閉じるワークフローは止める（[ADR-20261026-001][adr-status]）。どれも進行 Bot が書く表示で、判定には読まない（FLOW-39、[ADR-20261024-001][adr-model-states]）。更新は GraphQL の `updateProjectV2ItemFieldValue` で行える（公式文書で確認） |
| SETUP-6 | AI を担当者にできない（コラボレータは 1 人） | ~~Project に単一選択欄「Agent」を足す。~~ Agent 欄は足さない。次に動く担い手は表示の欄に出さず、人は軸ごとのビューで自分の番を探す。一時止める・引き取るは、人が付けるラベル（`agent-paused`・`agent-manual`）で受ける（FLOW-38、FLOW-39、[ADR-20261024-001][adr-model-states]）。Assignee は人のまま |
| SETUP-7 | Issue 依存（blocked by）が 1 件も入っていない | 受付 AI がマスターの下書きに候補（本文の `#n` の参照）を書き、人が `## 作業指示` で直し、人が決めたとき（`decision` の印を書くとき）に進行 Bot が入れる（TEXT-12、FLOW-36）。読み書きは REST の `/issues/{n}/dependencies/blocked_by` で行える（公式文書で確認） |
| SETUP-8 | 処理対象を示すラベルが無い | ~~`agent-ready` `needs-human` `from-review` `plan:fable` `plan:opus` を `setup` が足す。~~ 人が付ける `agent-ready`（決める）・`agent-paused`（一時止める）・`agent-manual`（引き取る）と、進行 Bot が付ける表示の `security-suspected`（攻撃の疑い）・`from-review`（🟢 を起票した Issue）を `setup` が足す。`needs-human` は使わない（止めるのは `stop` の印。FLOW-40）。`plan:fable`・`plan:opus` は、作業ごとに使うモデルの決め方を決めるまで作らない（[ADR-20261024-004][adr-master] の決定 8）。名前は設定ファイルに持ち、ここに書いた名前は初期値 |
| SETUP-9 | `develop` にブランチ保護が無い | ruleset で PR 必須にする（自律セッションの直 push を仕組みで止める） |
| SETUP-10 | 済み（2026-10-16 に確認）：コミットの署名が 1Password のソケット頼み | 人の署名は手元の SSH 鍵に移す（`gpg.format ssh`、`user.signingkey` を鍵ファイルに）。dev-autopilot は人の鍵を使わない。作業 AI の commit は署名なし、`develop` への squash マージの commit は GitHub が署名する（SAFE-14）。business-os の雛形（ADR 20261003-11）は利用者向けなので変えない |
| SETUP-11 | 開発専用の TypeScript の置き場が無い（tsconfig は `evals/` `plugin/hooks/` `plugin/scripts/` `test/` だけ） | ~~開発物のフォルダ `dev-autopilot/` を 1 つ足し、構造仕様 3.1 節の表に加える。~~ dev-autopilot は別のリポに移した（[ADR-20261016-003][adr-move]）ので、そのリポの root に `src/` と `tsconfig.json` を置く。中身は「[分離を見越した構成][f07]」の構成 |
| SETUP-12 | `claude -p` の実行は利用枠を消費する | 1 回の実行・1 日あたりの上限を設定に持つ |
| SETUP-13 | 作業 AI とレビュー AI が同じ GitHub アカウント（メンテナ）で投稿することになる | GitHub は PR の作者自身の approve と request changes を受け付けない。いまの運用も GitHub のレビュー状態を使わず、PR のコメントで判定している。dev-autopilot も同じで、判定は PR のコメントの固定の印（IMPL-18）で行う。進行 Bot は最初から GitHub App で投稿し、人のアカウントと分ける（[ADR-20261030-002][adr-no-pat]）。投稿者はアプリの名前（`<アプリの名前>[bot]`）になり、印を書いたと認めるアカウントの一覧（設定）もアプリの名前にする。最初から App で書くので、ほかのアカウントの印を読む手当ては要らない（[ADR-20261020-001][adr-bots]、[ADR-20261020-002][adr-app]） |
| SETUP-14 | Issue のフォームに受け入れ条件の欄が無い | 外から入る Issue は報告・提案の形で、作業の定義ではない。コラボレータの作業指示コメント（TEXT-1）で補う。フォームには足さない（外の人が書く欄にしない） |
| SETUP-15 | 渡す文が後から編集されうる | ~~処理対象にした時点のハッシュで検出する（TEXT-4）。~~ 人が決めた時点のマスターの下書きのハッシュを `decision` の印に書き、決めた後の `## 作業指示` と Issue の本文の変更は読まない（TEXT-4、FLOW-36）。マスターは進行 Bot の署名付きの印のコメントなので、書き換えられれば印の書き換えとして止まる。外の人の文は渡さないので、外の人の編集は影響しない |
| SETUP-16 | Issue を確かめる手順が dev-autopilot と結びついていない | 入口の検疫と下書き（TEXT-12）を進行 Bot と受付 AI が自動で行い、人の操作は、マスターの下書きを直す（`## 作業指示`）か差し戻す（`## 差し戻し`）ことと、決める（`agent-ready`）ことにする（FLOW-36）。順「入口の検疫と下書き（自動）→ 直す・差し戻す（人。要るときだけ）→ `agent-ready`（人）」を利用の手順に書く。コラボレータの Issue も対象（TEXT-9）。`/verify-issue` は手動の補助 |
| SETUP-17 | 指示の混入の検査と作業指示の下書きを行うものが無い。`/verify-issue` は外の人の不具合の報告・改善の提案を人が確かめる手順として書かれている | 検査と下書きは進行 Bot の受け入れ（TEXT-12。`dev-autopilot:brief:triager` と検疫のモジュール）が自動で行う。機械で探せる検査（見えない文字、HTML コメント、base64、当てる先のリポの GitHub の URL のほかの URL、指示の形の語）は TypeScript にして vitest で検査する。`/verify-issue` は手動の補助として残し、最初の手順で同じ検疫のモジュールを呼ぶ。着手の条件は dev-autopilot の ADR に書く |
| SETUP-18 | レビュー AI は PR のコードを実行して確かめる。その環境に GitHub のトークンや鍵があると、作業 AI が書いたコードから読み取れてしまう。開発リポは sandbox を使わないので、同じ OS ユーザーで動く限り `~/.ssh` と `~/.config/gh` は Bash から読める（`Read(~/.ssh/**)` の deny は Read ツールにしか効かない） | AI のセッションには `GH_TOKEN` を渡さず、`gh` を認証しない。GitHub への書き込みはすべて進行 Bot が行う（IMPL-5）。AI のセッションに sandbox を掛け、鍵を分ける（「[隔離と鍵の分け方][f05]」） |
| SETUP-19 | `change-review` スキルが利用者の全体設定（`~/.claude/skills/`）にあり、dev-autopilot の専用マシンでは無いかもしれない | レビューの Skill の正本を dev-autopilot の中（`plugin/skills/review/`）に置き、dev-autopilot が自分の Skill として読み込む。`~/.claude` には依存しない（「[分離を見越した構成][f07]」） |

## 段階

段階の表はここが唯一の正本。出口は数で書き、満たしたかは記録（FLOW-5）から数える。
段階は、番号だけでなく「導入と点検の実装（段階 3）」のように名前と番号の組で呼ぶ。段階 1〜3 は準備で、Issue の流れは導入と点検の実装（段階 3）の後に、すべての工程を最初から決まった担い手で動かす。
段階 4〜8（Issue の流れで AI が担う工程を段階ごとに広げる段階）は、[ADR-20261024-002][adr-no-stages] で持たなくなったので、番号を振り直さずに欠番にした。段階 4〜8 を指す文は、[ADR-20261024-002][adr-no-stages] の決定 8 で読む。段階 9 は、導入と点検の実装（段階 3）の後、FLOW-12 の条件を満たしたら入る

| 段階 | 名前 | 内容 | 出口 |
|---|---|---|---|
| 1 | 橋渡し ADR | 橋渡し ADR 20261008-01（business-os 側）を proposed で起票 | accepted（済み） |
| 2 | 実機の試し | 最初の薄切り：進行 Bot なし・GitHub 書き込みなしで、手で書いた安全設定（`setup` はまだ無い）を `--settings` で渡した `claude -p` を未信頼の worktree の中で動かし、「[実機で確かめた事実][f10]」の表の未確認を実機で潰す。攻撃の再現（project の hook が走らないこと、`commondir` の差し替えが進行 Bot の git に届かないこと）を含む | 「[実機で確かめた事実][f10]」の表の未確認がすべて「実行」で埋まる（済み） |
| 3 | 導入と点検の実装 | `setup` と `check` の実装（「導入と点検」）と、それによる準備（SETUP-5・SETUP-6・SETUP-8・SETUP-9）。deploy key と印の署名の鍵の一覧の用意（「[隔離と鍵の分け方][f05]」）と、2 つの GitHub App の作成と導入（`setup` が作り、人が押す）。`change-review` スキルの v2（「[レビューの Skill][f08]」）。受け入れ（TEXT-12）と `/verify-issue` の拡張 | `check` が business-os で全項目 pass（進行 Bot がセッションごとに作る安全設定で、実体のパスの解決と読み取りの検査が動くことを含む。試しのセッションと書き込みの権限の試し（進行 Bot の GitHub の認証で、リポの変数を作れないこと）は、全体の止める合図を `run` にした後の最初の実行で pass。基本設計の「[導入と点検][design-setup]」の「組と回数」）。スキルの v2 は手動のレビュー 1 件で今の PR の形が出ることを確かめる |
| 4 | **欠番**（[ADR-20261024-002][adr-no-stages] で、段階 4〜8 を持たなくなった）：~~受け入れ工程~~ | ~~棚卸しと受け入れ（TEXT-12）を動かす（選ぶ・待たせる・下書きを書く・記録する。実装はしない）。最初は設定 `dryRun: true` で、下書きと選定の結果を手元の記録にだけ書いて人が読む。書かずに試す間は、下書きを作った Issue を文が変わるまで受け入れ直さず、選定の結果は候補の条件から人の承認にかかわる条件を除いて作る。人は、書き始めるまで `agent-ready` を付けない（[ADR-20261021-002][adr-dry]）。異議が無ければ、進行 Bot を GitHub App に切り替えてから（[ADR-20261020-002][adr-app]）`dryRun` を切り、残りの週は Issue に実際に投稿する。受付 AI の方針に係る要望かの判定を記録し、人が `agent-ready` を付けたかと比べる（[ADR-20261019-004][adr-policy]）。検疫で URL によって止まった回数を、GitHub の添付の URL（TEXT-9）で止まった回数を分けて記録し、通す URL の範囲を見直す材料にする（[ADR-20261022-001][adr-url]）~~ | ~~7 回の実行（lock で見送った実行と起動されなかった日は数えない。「[運用の規則][f01]」）で、人が `## 異議` の見出しで書いた回数（dry-run の間は記録ファイルへの追記）0。うち投稿ありの実行が 3 回以上~~ |
| 5 | **欠番**（[ADR-20261024-002][adr-no-stages] で、段階 4〜8 を持たなくなった）：~~計画・実装工程~~ | ~~計画 AI と実装 → draft PR を 1 件ずつ。計画（IMPL-10）は人がすべて読み、誤っていれば止める。レビュー AI による計画のレビューで直すべき点が見つかったら、計画 AI に計画し直させる（上限は設定。初期値 1 回）。レビューの結果と計画し直したことは Issue に書かず、記録に残して人の判断と比べる（[ADR-20261018-004][adr-planrev]、[ADR-20261019-003][adr-replan]）。レビューは人が行い、PR のマージも人。モデルの比較もここで行う：作業 AI は最初の実装を `opus` で行い、Issue の種類ごとに `sonnet` も試す。`sonnet` と `opus` の PR を、人が手直しせずにマージできたかで比べ、最初の実装に使うモデルを仮に決める（IMPL-25、[ADR-20261018-003][adr-model]）。計画 AI は受け入れの `plan=` の判定を人が妥当と見たか、`fable` にした計画が `opus` と比べて違ったかを記録する。受け入れ工程（段階 4）と同じく、検疫で URL によって止まった回数（受け入れと、計画と PR の文の検疫。GitHub の添付の URL で止まった回数を分けて）を記録する（[ADR-20261022-001][adr-url]）~~ | ~~5 件の PR のうち 4 件以上を「手直しなし」で人がマージ（手直し＝人が PR に commit を足した、または close して作り直した）。`plan=` の判定への `## 異議` の割合を記録する~~ |
| 6 | **欠番**（[ADR-20261024-002][adr-no-stages] で、段階 4〜8 を持たなくなった）：~~レビュー工程~~ | ~~レビュー AI と批評 AI を足す（`review` Skill を `claude -p` で動かし、印を足す）。レビュー AI は `opus`（xhigh）を既定に、同じ PR を `fable` でも回して比べる（比較の実行は印を書かない。IMPL-19）。比べるためのレビューは、本番のラウンドの `review` の印を書いた後の実行に、別の重い工程としてラウンドごとに 1 回起こす。比べのセッションの費用は 1 回の実行と 1 日の上限に数え、Issue の累計には数えない。比べのセッションの失敗は実行の失敗に数えず、やり直さない（[ADR-20261021-007][adr-compare]）。作業 AI の最初の実装に使ったモデルごとに、最初の AI のレビューで直すように言われたか（`review` の印の `verdict`）と、重さごとの指摘の件数を記録する（IMPL-25）。見落としの軸（IMPL-16）の結果は記録だけに残し、人が同じ PR をレビューする 5 件で、人の 🔴 🟡 のうち 3 つの軸が見落として見落としの軸が見つけたものの数と、見落としの軸だけが出して人が出さなかった 🔴 🟡 の数を記録する（[ADR-20261019-002][adr-gaps]）~~ | ~~5 件の PR で、人が同じ PR を `change-review` した結果（人の 🔴 🟡 の見出しの一覧）と照合し、一致率（人の 🔴 🟡 のうち AI も出した割合）が 80% 以上、却下した指摘の割合（AI だけが出した 🔴 🟡 のうち `## 却下` に書かれた割合）が 15% 以下。`opus` と `fable` の一致率・却下率・利用枠を並べ、既定のモデルを決める。人の二重レビューはこの 5 件に限る~~ |
| 7 | **欠番**（[ADR-20261024-002][adr-no-stages] で、段階 4〜8 を持たなくなった）：~~修正工程~~ | ~~修正工程（段階 7）に進む前に、レビュー工程（段階 6）で記録した見落としの軸の 2 つの数を見て、見落としの軸の指摘を批評 AI に通し `review` の印に入れるかを、ADR で決める（IMPL-16、[ADR-20261019-002][adr-gaps]）。対応ループを足す。マージは人。進行 Bot は「[マージ][f03]」の判定を記録に残すだけで印は書かない（shadow 判定）。計画・実装工程（段階 5）と同じく、人はすべての計画を読み、計画のレビューの結果は Issue に書かない（直すべき点があれば計画し直す）。修正工程（段階 7）を終えるときに、計画のレビューの結果と人の判断を比べ、計画し直した計画ごとに、直す前と後の計画を人が見比べる（IMPL-10）。作業 AI の最初の実装に使ったモデルごとに、AI が直した回数（往復の回数）と、修正で `opus` に切り替えた件数と、切り替えた後に Ready to Merge まで進んだ件数を記録し、修正工程（段階 7）を終えるときに、最初の実装に使うモデルを Issue の種類ごとに見直す（IMPL-25）~~ | ~~5 件のうち 4 件以上が往復 3 回以内で Ready to Merge~~ |
| 8 | **欠番**（[ADR-20261024-002][adr-no-stages] で、段階 4〜8 を持たなくなった）：~~マージ判定工程~~ | ~~進行 Bot のマージの判定（「[マージ][f03]」）の印を書く。マージの実行は人のまま。マージ判定工程（段階 8）に進む前に、レビュー AI による計画のレビューで計画を止めるかを、ADR で決める（IMPL-10、[ADR-20261018-004][adr-planrev]）~~ | ~~10 件で、人のマージの判断と進行 Bot の判定（`merge=ready` / `merge=human`）が全件一致（MERGE-3 の数え方）。revert 0 件（revert＝マージから 14 日以内に、`develop` にその squash commit を revert する commit か、本文でその PR 番号を「戻す」と言及する fix の PR がマージされたもの）。満たしても実行は人のまま。実行を進行 Bot に移すかは別の ADR で決める（MERGE-9）~~ |
| 9 | 並列化 | 並列（複数 worktree。WIP の上限を 1 から上げる）と間隔の短縮 | FLOW-12 の条件（連続 7 回の実行で、停止中か攻撃の疑いになった作業の単位が 0 件、利用枠が上限内）で判断 |

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
[adr-stop]: ../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-model]: ../adr/adr-20261018-003-count-model-raise-at-stage-7.md
[adr-planrev]: ../adr/adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[adr-gaps]: ../adr/adr-20261019-002-review-gaps-axis-in-shadow-first.md
[adr-replan]: ../adr/adr-20261019-003-replan-on-plan-review-from-stage-5.md
[adr-policy]: ../adr/adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths.md
[design-setup]: ../design/basic/04-setup-and-check.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
[adr-review]: ../adr/adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-bots]: ../adr/adr-20261020-001-bot-accounts-as-list.md
[adr-app]: ../adr/adr-20261020-002-switch-to-github-app-before-writing.md
[adr-dry]: ../adr/adr-20261021-002-dry-run-only-at-intake-stage.md
[adr-marker-keys]: ../adr/adr-20261021-004-marker-key-list-and-resume-after-invalid-markers.md
[adr-compare]: ../adr/adr-20261021-007-compare-review-done-from-local-records.md
[adr-url]: ../adr/adr-20261022-001-quarantine-urls-outside-target-repo.md
[adr-model-states]: ../adr/adr-20261024-001-work-unit-main-flow-and-overlay-axes.md
[adr-no-stages]: ../adr/adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[adr-master]: ../adr/adr-20261024-004-master-draft-quarantine-and-protected-paths.md
[adr-status]: ../adr/adr-20261026-001-use-status-field-for-state.md
[adr-ci-secrets]: ../adr/adr-20261028-001-ci-secrets-only-in-base-branch-environments.md
[adr-bypass-baseline]: ../adr/adr-20261028-004-compare-ruleset-updated-at-with-setup-baseline.md
[adr-no-pat]: ../adr/adr-20261030-002-bot-uses-github-app-only-from-start.md
[adr-setup-app]: ../adr/adr-20261030-001-setup-uses-setup-app-instead-of-human-gh.md
