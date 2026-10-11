---
status: proposed
created: 2026-10-11
updated: 2026-10-11
source: ./dev-autopilot-requirements.md
---

# dev-autopilot 要件定義書

dev-autopilot（自律開発ループ）の要件を、番号付きで 1 件ずつ書いた文書。[要件メモ][memo]（2026-10-07〜11 の検討の記録。2026-10-11 に凍結）から、経緯と日付を落として「いまの判断」だけを写した。

- 番号（R・I・S・P と節の番号）は要件メモから引き継ぐ。基本設計と ADR はこの番号で根拠を示す
- 本書の要件を変えるときは、ADR を `proposed` で起票して人が `accepted` にし、その後に本書を直す。要件メモは直さない
- 設定に持つ値（上限、初期値、しきい値）は 8 節に集める。要件メモが初期値を決めていたものは 8 節に写し、決めていないものは「未定」と書く。設計書は 8 節の項目名で参照する
- 用語：**進行役**＝AI ではない Node のスクリプト。**段**＝1 回の実行で進める単位。**段階**＝導入の順（0〜6）。**印**＝進行役がコメントに書く機械で読む 1 行。**渡す文**＝AI に作業の内容として渡す文

## 1. 目的

Project に登録された Issue のうち、人が「処理対象」と判断したものを、AI が worktree で実装して PR にし、別セッションの AI がレビューし、指摘が無くなるまで作業 AI が直す。
開発の PR は、条件を満たしたら進行役が「マージしてよい」と判定して人に渡し、マージの実行は人が行う。方針に係る PR（ADR など）は判定の対象にせず、マージできる状態で止めて人に渡す。
マージの実行を進行役に移すかは、判定の信頼度を測ってから別に決める（R7-6）。

## 2. 前提と制約

| 項目 | 要件 |
| --- | --- |
| 置き場所 | business-os の開発専用フォルダ `dev-autopilot/` に閉じて作る。business-os の配布物（`plugin/` の下）には入れないので利用者には配られない。業務の固有名詞と秘密を置かない（`check:leak` の対象）。将来は別リポジトリに分離する（9 節） |
| 実行環境 | 手元の cron（macOS は launchd）で始め、将来は専用マシンで同じものを回す。クラウドの仕組みは土台にしない |
| 人の関所 | 受け入れ（検疫・作業指示の下書き・依存とマイルストーンの候補）は進行役が自動で用意し、人は下書きを読んで `agent-ready` を付ける 1 操作で処理対象にする。PR のマージの実行は人だけが行う |
| マイルストーン | リリースごとにマイルストーンを組む。ループはマイルストーンの中の Issue を対象にする |
| 間隔 | 最初は 1 日 1 回。安定したら狭める（R6-2） |
| 鍵 | 人の鍵と自動化の鍵を分ける。人は手元の SSH 鍵で署名する。dev-autopilot のマシンには人の鍵も `gh` のログインも置かず、business-os に絞った deploy key（push 用）と fine-grained PAT（進行役用）だけを置く |
| 隔離 | AI の 5 役すべての `claude -p` に、進行役が `--settings` で渡す安全設定で Claude Code の sandbox を掛ける。進行役・次のセッション・CI は、AI が書いた worktree の中身（設定・gitdir・CI 定義・スクリプト）を信頼して実行しない（5 節） |
| モデル | 役ごとに別名で書き版番号を書かない（`fable` `opus` `sonnet`）。`haiku` だけは別名が旧版に解けるので完全な ID `claude-haiku-5-5` を書く。effort も役ごとに設定に持つ。作業 AI に `fable` を使わない。どの役にも `fable` を既定にしない（4.11 節） |
| エージェント | 計画用・作業用・レビュー用・批評者・受け入れ用の 5 つの定義を `dev-autopilot/plugin/agents/` に置く |

## 3. 登場するもの

AI の 5 役すべてに共通：`claude -p` を進行役が子プロセスとして起動し、sandbox を掛け、`gh` の認証・`GH_TOKEN`・進行役の鍵を渡さず（環境変数は許可した名前だけで組み立てる）、ユーザー設定とプロジェクト設定から切り離して起動する（`--setting-sources local`。Plugin は進行役専用 clone の絶対パスで渡す）。道具はエージェント定義の `tools` と `permissions` の allow・deny で絞る。GitHub への書き込みは一切できない。

| 役 | 実体と道具 | セッション |
| --- | --- | --- |
| 進行役 | cron が起動する Node 24 の TypeScript。AI と共有しない専用 clone から実行する。判定は決定的な処理で行う | AI ではない |
| 計画役 | `claude -p --agent <計画用>` を worktree で、読むだけの道具（Read・Grep・Glob）で起動 | Issue ごと。計画だけを出力契約で返す |
| 作業 AI | `claude -p --agent <作業用>` を worktree で起動。Bash・Read・Write・Edit（WebFetch・WebSearch・Agent なし） | Issue ごと、ラウンドごと。計画は進行役が固定の指示文で渡す |
| レビュー AI | `claude -p --agent <レビュー用>` を、PR の head で作る別の worktree で起動。Bash・Read、Write は worktree の中だけ | PR ごと、ラウンドごと。作業 AI と文脈を共有しない。モデルは設定で分けられる |
| 批評者 | `claude -p --agent <批評者用>` を、レビュー AI と同じ worktree で、レビュー AI の後に進行役が別に起動する独立セッション。Bash・Read | PR ごと、ラウンドごと。入力はレビュー AI の指摘と差分だけ |
| 受け入れ AI | `claude -p --agent <受け入れ用>` を道具なしで起動 | Issue ごと（著者を問わず走る）。読めるのはその Issue の文だけ |
| 人 | メンテナ | 処理対象の承認、PR のマージの実行、止まったときの対処 |

AI が担うのは「受け入れの下書きを書く」「計画を書く」「実装する」「レビューする」「指摘を再検証する」「直す」の 6 つだけ。どの Issue を選ぶか、待つか、止めるかは進行役が決定的に判定する。

## 4. 要件

### 4.1 棚卸し（進行役）

| 番号 | 要件 |
| --- | --- |
| R1-1 | 次をすべて満たす Issue だけを候補にする：`agent-ready` があり付けた人が push 以上（I10）、マイルストーンがある（版の小さい順）、Status が `Todo`、依存（blocked by）の相手がすべて閉じている、`needs-human` が無く Agent 欄が `human` でない、受け入れの下書きコメント（進行役の投稿、`intake` の印。4.12 節）がある。本文を渡すか下書きだけを渡すかは、印の値でなく渡す直前に著者の権限（I10）から決める |
| R1-2 | 着手前の予測で候補を待たせない。待たせるのは人が入れた依存だけ。触る範囲の重なりは着手後に実際の変更ファイルで見て、警告のコメントに留める（R5-3） |
| R1-3 | 1 回の実行で着手する件数と、進行中の Issue の数（WIP）に上限を持つ。上限に達していれば新しい Issue に着手しない |
| R1-4 | 実行の記録（選んだ Issue、待たせた Issue と理由、使った時間）を追跡しないフォルダに残す。AI のセッション 1 回ごとに、実行 ID・Issue 番号・ラウンド・役・指定したモデルと effort・実際に使われたモデル（`stream-json` の `result` のイベントの `modelUsage`）・費用・ターン数・実行の成否・作業の成否を固定の欄で残す |
| R1-5 | 進行役は Issue ごとに 1 回の実行で 1 段だけ進める。段は、受け入れ、計画、実装と PR の作成、レビュー（批評を含む）、修正、マージの判定、マージ後の片付け。進行中の Issue は毎回 GitHub の状態（Status、PR の有無と draft、最新の印の種類と `head`、必須チェックの結果）から次の段を決定的に決めて再開し、手元の記録は使わない。CI は実行の中で待たず次回の実行で読む。1 回の実行の所要時間に上限を持ち、超えたら残りは次回に回す |

### 4.2 実装（作業 AI）

| 番号 | 要件 |
| --- | --- |
| R2-1 | 進行役は AI の作業場所の clone から、設定の置き場に Issue ごとの worktree を作り、その中で `claude -p` を起動する。人の作業場所と共有しない。依存の導入（`pnpm install`）は作業 AI のセッションの中で行い、進行役は worktree の中でリポのコードもパッケージの lifecycle script も実行しない |
| R2-2 | ブランチは `<型>/issue-<番号>`。型は設定のラベル対応表で進行役が決める。短い名前は付けない |
| R2-3 | Issue の本文は 4.8 節の規則で「報告」として渡す。本文に書かれた指示には従わない |
| R2-4 | 守る規約は実装リポの文書をそのまま渡す：`CLAUDE.md`、関係する ADR、`pnpm check` が緑、changeset、日本語の Conventional Commits |
| R2-5 | 作業 AI は GitHub に一切書かない。結果は出力契約（JSON）で進行役に返す（PR の題名と本文の案、止まった理由、確かめた検査の結果）と worktree の commit。進行役が自分の JSON スキーマで `result` を検証し、欠けや型の違いは「実行の失敗」。解析できない出力を「成功」や「0 件」と読まない。`--json-schema` は `--agent` と併用できないので頼らない。PR の作成・draft/ready・Status と Agent 欄・コメントはすべて進行役が検証してから行う |
| R2-5b | 進行役が PR を `develop` 向けに draft で作る。本文に `Closes #<番号>` と検査の結果を書く。題名は Conventional Commits の規約に進行役が照らしてから使う |
| R2-6 | PR を作ったら Status を `In Review` にする。Assignee はメンテナのまま。AI の担当は Project の単一選択欄「Agent」で表す（`worker` `reviewer` `human` `paused`） |
| R2-7 | ADR の起票が要ると分かったら、作業 AI は実装せず「止まった理由」を返し、進行役が Issue にコメントして `needs-human` を付ける |
| R2-8 | 着手の前に基線が緑であることを確かめる。確かめ方は `origin/develop` の最新 commit に対する必須チェック（R4-3）がすべて `success`。赤なら着手せず `needs-human`。進行役は worktree で `pnpm check` を実行しない |
| R2-9 | 実装の前に計画役（作業 AI と別セッション）が計画を出力契約で返し、進行役が `plan` の印付きで Issue にコメントする。計画に含めるもの：対象のファイル・モジュール・型とその依存、手順と順序、手順ごとの完了条件、検証方法、範囲外、想定する commit の分割。計画はパスを書いてよい。段階 2〜4 では計画を投稿した実行では実装に入らず次回の実行で入る（人が止められる。止める合図は Agent 欄 `paused` か `needs-human`）。段階 5 以降は同じ実行で続ける。作業 AI は計画を固定の指示文の中で「作業の内容」として受け取る。次回に計画を読み戻すときは R3-5 の 3 条件を満たす印のコメントだけを使い、渡す直前に検疫（I11）を通す。計画役のモデルは受け入れの印 `plan=` と人のラベル（優先）で決め、既定は `opus`。`opus` が「設計判断・アーキテクチャの選択・原因不明の障害解析を含む」と自己申告したら 1 回だけ `fable` で計画し直す（2 回目の自己申告は無視して記録に残す） |
| R2-10 | PR は draft で作り、収束して `Ready to Merge` になったときに進行役が ready にする |
| R2-11 | テストを消す・skip する・期待値を緩める・lint を抑止する変更は、作業指示が明示していない限り禁止。レビュー AI は 🔴 として探す |
| R2-12 | 1 回のセッションの上限（時間・ターン・費用）を設定に持つ。時間の上限は進行役が子プロセスをプロセスグループごと止める。上限に当たったら途中の状態を commit せず、worktree を残す。扱いは R2-13 の表 |
| R2-13 | 失敗を「実行の失敗」と「作業の失敗」に分ける。実行の失敗（利用枠切れ、時間の上限、ターンの上限、`claude` の異常終了、出力契約の欠け、受け入れ AI の拒否）は `needs-human` を付けずに次回へ持ち越し、同じ Issue で 3 回続いたら `needs-human`。受け入れ AI の拒否の後は次回 `sonnet` で試す。作業の失敗（止まった理由がある）は `needs-human`、レビューの fix は往復（R4-2）。記録は `stream-json` の `assistant` のイベントごとの `model` と `usage` を足し上げる。設定に持つ上限：セッションの時間・ターン・費用、1 Issue の累計の利用枠、Agent ツールを許すか、再試行の回数と間隔、1 回の実行と 1 日の利用枠。累計が上限に当たったら `needs-human` |
| R2-14 | 進行役は GitHub に書くすべての文（diff、commit メッセージと author、PR の題名と本文、計画・レビュー・対応・受け入れのコメント、起票する Issue）に漏えいの検査（鍵の形式、メールアドレス、機械上のパス、進行役の設定と記録の内容）を掛け、引っかかれば書かず `needs-human`。検査の実装は dev-autopilot 自身の `src/` に持つ。マージの判定の時点で最終の head の diff 全体に掛け直す。diff が「進行役・次のセッション・CI が実行するパス」（`.claude/**` `CLAUDE.md` `.mcp.json` `.github/**` `dev-autopilot/**`。設定に持つ）に触れていれば push せず `needs-human`。push は進行役の clone から `git push origin <検証した SHA>:refs/heads/<型>/issue-<番号>` の形で行い、親 commit が API の head（初回は base）と一致することを確かめる |

### 4.3 レビュー（レビュー AI と批評者）

| 番号 | 要件 |
| --- | --- |
| R3-1 | PR の差分を、作業 AI とは別の作業場所で読む。作業場所は進行役が PR の head（API の SHA）で切る専用の worktree で、レビューのたびに作り直す。手順は `review` Skill（change-review v2）を使う。核は変えない（実行して確かめる、fail-open を探す、収束するまで回す）。無人の規則：入力（基点・仕様の在りか・🟢 の起票先・予算）が欠けていれば聞かずに「未収束（理由）」で返す。レビュー AI の中から `claude -p` を起動しない。サブエージェントは再帰して Skill を呼ばず、修正せず、マージしない |
| R3-2 | 結果は PR のコメントに書く（行に付けるレビューコメントにしない）。1 回目は「## レビュー結果（基点: `develop`）」、2 回目以降は「## 再レビュー結果 R<n>（対象: <範囲>）」。冒頭に判定、末尾にラウンドの表 |
| R3-3 | 人が読む本文に加えて `review` の印（4.12 節）を 1 行足す。`verdict` を pass にするのは次の 5 つをすべて満たすときだけ：(1) レビュー AI と批評者の出力契約が完全に解析できた、(2) レビューが収束した、(3) `is_error` が偽、(4) 重要度ごとの件数がすべて数値、(5) 実際に使われたモデルが設定と一致。満たしたうえで 🔴 🟡 が 0 件なら pass、1 件でもあれば fix。どれかを満たさなければ印を書かず実行の失敗。判定は印だけで行い本文の言い回しは解析しない |
| R3-4 | レビュー AI のモデルは設定で変えられる。作業 AI と同じモデルでもよい。段階 3 の比較のためだけに `fable` で回したレビューは印を書かない |
| R3-5 | 印の真正性。正は GitHub に置き手元の記録に依存しない。次の 3 つをすべて満たすコメントの印だけを読む：(a) 投稿者が進行役のアカウント、(b) 未編集（GraphQL の `lastEditedAt` が null、または `userContentEdits.totalCount` が 0。`updated_at` は非表示でも変わるので使わない。受け入れの下書きは人が編集する前提なので掛けない）、(c) 印に進行役の署名 `sig`（`sig` 以外のすべての欄に対する HMAC）があり検証できる。鍵は進行役のプロセスの環境変数に置き AI には渡さない。AI の出力を投稿する前に `<!-- dev-autopilot:` で始まる文字列を取り除く。進行役が投稿した文は、投稿者がコラボレータであっても常に AI の出力として扱う |
| R3-6 | 再レビューの入力は「前回の印と指摘」「修正コミットの範囲（SHA）」「差分」だけ。作業 AI の対応コメントは渡さない。前のラウンドで解決した指摘は、新しい失敗シナリオが無ければ再提起しない |
| R3-7 | 批評者。レビュー AI の 🔴 と 🟡 を別セッション（新しい文脈）に差分とともに渡し、指摘ごとに「反対する前提」で再現を試みさせる。返すのは、再現できたか（実行済み / 読んだだけ）、確信度（0〜100）、推奨する重要度。統合：🔴 は 🟡 より下げない（再現できなくても 🟡 のまま）、🟡 は確信度がしきい値未満なら 🟢 に下げる、🟢 は渡さない。🔴 を「再現できない」と返した PR は、その後 pass になっても `merge=human`。批評者の出力契約が解析できなければ実行の失敗 |

### 4.4 対応ループ

| 番号 | 要件 |
| --- | --- |
| R4-1 | 判定が fix なら作業 AI を同じ worktree で起動し、最新のラウンドの 🔴 と 🟡 だけに対応させる。push は進行役が漏えいの検査を通してから行う。🟢 はこの PR では直さない。対応の結果は「## R<n> の指摘への対応（<コミット>）」のコメントに書き、`fix` の印を付ける |
| R4-1b | 🟢 は進行役が Issue にする。1 件 1 Issue、題名は指摘の見出し、本文に PR 番号・ラウンド・失敗シナリオ・推奨、ラベル `from-review`。本文は AI の出力として扱う（受け入れでは `replace`）。マイルストーンは付けず、人が付けるまで受け入れも走らせない。同じ見出しの open な Issue があれば作らずコメントで PR 番号を足す。Issue を作ってから PR を `Ready to Merge` にする |
| R4-2 | 往復の回数に上限を持つ。超えたら Status を `Blocked`、`needs-human`、経緯を Issue にコメント。同じ指摘が 2 ラウンド続けて出たら振動として同じ扱い |
| R4-2b | 作業 AI が `sonnet` のとき、2 ラウンド目の修正セッションから `opus` に 1 段上げる。`fable` には上げない。再計画は人が指示する（自動で計画役に戻さない）。上げたことは記録と Issue のコメントに残す |
| R4-3 | 終了条件：CI が緑、かつ最新のラウンドの印が `verdict=pass`（🟢 は残ってよい）。CI が緑＝設定に列挙した必須チェックのすべてが PR の head に対して `success`。照合はチェック名に加え、出した app と base 側のワークフローのパスでも行う。必須チェックの一覧が空なら点検を fail にする。進行役は実行の中で CI を待たない。満たしたら Status を `Ready to Merge` にする |
| R4-4 | `Ready to Merge` の PR は進行役が 4.7 節の条件でマージの可否を判定し、印として PR に書く。マージの実行は人 |

### 4.5 並列と競合

| 番号 | 要件 |
| --- | --- |
| R5-1 | Issue ごとに worktree を分け、同時に複数の Issue を進められる |
| R5-2 | 競合の正は人が入れる Issue 依存（blocked by）。進行役はそれに従う |
| R5-3 | 競合の予測は AI にさせない。作業 AI の commit が出た時点で、進行役がその PR の変更ファイルを他の進行中の PR の変更ファイルと機械で比べ、重なりがあれば Issue に警告のコメントを 1 回書く |
| R5-4 | 先行の PR がマージされたら、依存していた PR を `develop` に載せ直してから続ける（R7-3c） |
| R5-5 | 依存で待つ Issue は Status・Agent 欄・ラベルを変えない。待たせた理由を記録し、Issue には 1 回だけコメントする。先行が閉じたら次回の実行で自動で続ける。作業中に人が依存を足したときも同じで、レビューと対応ループを止めて待ち、worktree と PR は残す |

### 4.6 間隔と指標

| 番号 | 要件 |
| --- | --- |
| R6-1 | 間隔と件数の上限は設定ファイル 1 つで変える |
| R6-2 | 間隔を狭める条件：連続 7 回の実行で `needs-human` が 0 件、利用枠の消費が上限内。1 段ずつ狭める |
| R6-3 | 指標を記録する：マージ率、往復の回数、人が却下した指摘の割合（誤検知率）、マージ後の revert 率、1 Issue あたりの利用枠。revert 率は進行役のマージ判定を信頼してよいかの判断に使う |
| R6-4 | 指標の記録は GitHub に置く。却下は人が PR に「## 却下 R<n>」の見出しでコメントし、却下する指摘の見出しを列挙する。進行役は push 以上の権限を持つ人が書き、進行役の印を持たないコメントだけを数える。手元の記録は集計の写し |

### 4.7 マージ

| 番号 | 要件 |
| --- | --- |
| R7-1 | マージの実行は人だけが行う。進行役は可否を判定して人に渡すところまで。AI にも進行役にも `gh pr merge` を許さない |
| R7-2 | 「マージしてよい」の条件（すべて満たす）：dev-autopilot が作った PR（進行役の署名付きの印で判定。作者のアカウントでは判定しない）で向き先が `develop`、R4-3 の終了条件（印の `head` が PR の head と一致、CI 緑）、漏えいの検査を最終の head の diff 全体に掛け直して通る、批評者が 🔴 を落としたラウンドが無い、変更ファイルがすべて許可一覧（R7-3）にあり状態が追加（A）か変更（M）で mode が `100644`、方針に係るパスに 1 つも触れない。変更ファイルの一覧は API でなく進行役の clone の `git diff --raw <base>...<head>` で作る |
| R7-2b | 判定の結果を `merge` の印（`merge=ready` か `human`、満たさない条件、head）として PR に書く。`ready` なら `Ready to Merge` にして人に渡す。`human` なら理由を添えて `Ready to Merge` のまま人に渡す。一致の数え方：`merge=human` の PR が人にマージされたら不一致、`merge=ready` の PR に「## 判定と違う判断」のコメントがあれば不一致、マージされたら一致。どちらも無く 7 日経った PR には 1 回だけ催促し、人が動くまで数えない |
| R7-3 | 許可一覧（拒否一覧でなく許可一覧）。初期値は `docs/usage/**` と `.changeset/*.md`。`plugin/skills/**` `plugin/scripts/**`（配布物）、`test/**` `evals/**` `fixtures/**`（テストの弱体化を止められない）は入れず、段階 5 の後に別の判断で広げる。削除（D）・改名（R）・種類の変更（T）・実行属性の変更は `merge=human`。常に `merge=human` の例：`docs/adr/**` `docs/design/**` `ROADMAP.md` `CLAUDE.md` `.claude/**` `dev-autopilot/**` `.claude-plugin/**` `.github/**` `plugin/hooks/**` `plugin/agents/**` `plugin/templates/**`、`plugin/scripts/check-repo.ts` など `check:*` の実装、`package.json` `pnpm-lock.yaml` `pnpm-workspace.yaml` `.node-version` `tsconfig.json` `vitest.config.ts` `eslint.config.*` `commitlint.config.*` `.gitleaks.toml` `.gitignore` `.markdownlint*` `.changeset/config.json`、`release/*` と `main` 向けの PR |
| R7-3b | 人のマージは `gh pr merge --squash --match-head-commit <印の head>`。1 回の実行で `merge=ready` にするのは 1 件まで。ruleset に必須チェックの strict（base と最新であること）を入れる。載せ直しは進行役が自分の clone で `git merge origin/develop` を行い結果の SHA を push する（force push を使わない） |
| R7-3c | 載せ直しで head が変わったとき、`develop` に対する差分の patch-id（`git patch-id --stable --verbatim`）が前後で同一なら、進行役が同じ内容の印を新しい head に付け直す。再レビューはせず往復にも数えない。CI は新しい head で改めて緑を要求する。patch-id が変わったときだけ再レビューを行い往復に数える。衝突の解消は作業 AI の修正セッションで行う |
| R7-4 | マージ後（次回の実行で検出）：Issue を閉じる、worktree を消す、依存していた PR を載せ直す |
| R7-5 | 利用者の全体ルール「PR のマージは Claude が実行しない」の例外は要らない。判定の実装は vitest で検査する |
| R7-6 | マージの実行を進行役に移すかは将来の選択肢。段階 5 の出口を満たしても自動では移さない。移すなら ADR で決める。揃える条件：向き先が `develop`、方針に係るパスの除外、revert 率の監視、人の判断と判定の一致、全体ルールの例外の扱い |

### 4.8 文の扱い（外から入る Issue）

AI に渡す文は、コラボレータ（push 以上）が書いたものだけ。外の人が書いた題名・本文・コメントは AI に渡さず、読むのは人と受け入れ AI だけ。

| 番号 | 要件 |
| --- | --- |
| I1 | 「誰の文か」は投稿者のアカウントと、進行役の署名付きの印の有無で決める。進行役が投稿した文は常に AI の出力として扱う。著者がコラボレータ（進行役でない）なら本文を渡す。著者が外の人か進行役なら、コラボレータの「## 作業指示」コメントか、人が `agent-ready` で承認した下書きだけを渡す。コメントはコラボレータのものだけを渡す。渡す文の範囲は `agent-ready` の時点で確定し、その後のコメントは渡さない |
| I2 | 作業指示も承認した下書きも無い外の人の Issue は、`agent-ready` があっても着手しない |
| I3 | 渡す文は「作業の内容」として区切った欄に入れ、動き方を変える指示として従わない旨を固定の指示文に書く。AI の動き方は固定の指示文だけで決める |
| I4 | 処理対象にした時点で渡す文のハッシュを記録する。着手時と対応ループの各回で変わっていたら止めて `needs-human` |
| I5 | 作業 AI とレビュー AI に、文中の URL を取りに行く手段（WebFetch・WebSearch・外への curl）を与えない。文中のコマンドは実行しない |
| I6 | commit・PR・ブランチ名に写すのは Issue 番号だけ |
| I7 | 渡す文に事業データ・認証情報が混じっていたら写さず止めて `needs-human`。進行役は着手前に機械で探せる形（鍵の形式、メールアドレス）を検査する |
| I8 | 作業中に Issue が閉じられた・移された・ラベルやマイルストーンが外れたら、作業を止め worktree は残し `needs-human` |
| I9 | すべての Issue（コラボレータのものを含む）は受け入れ（I12）を経て、人が下書きを読んで `agent-ready` を付けてから着手する。指示の混入の検査は検疫（I11）が機械で行い、下書きコメントに「混入の検査：無し / 有り（種類）」の行を必ず書く。見つかったら下書きを作らず `needs-human` にし、人が作業指示を一から書く。`/verify-issue` は人が手で深く確かめる補助として残し、最初に検疫のモジュールを呼び、最後に `dev-autopilot:intake` で下書きを作る |
| I10 | コラボレータの判定は `GET /repos/{o}/{r}/collaborators/{user}/permission`（push 以上）で、渡す直前に 1 人ずつ引く。対象は Issue の著者、作業指示の投稿者、`agent-ready` を付けた人（timeline）、コメントを編集した人（`userContentEdits`）。満たさなければそのラベルと編集は無効として扱い `needs-human`。進行役のアカウントの投稿は常に AI の出力 |
| I11 | 渡す直前の検疫。AI に渡す文（本文、作業指示、承認された下書き、計画、PR のコメント、再レビューの入力）を、書いた人を問わず渡す直前に毎回、機械の検査に通す。探すもの：見えない文字（ゼロ幅・制御文字）、HTML コメント、長い base64 や難読化、URL、指示の形の語、欄の区切りを模した文字列。1 つでも見つかれば渡さず `needs-human`。検疫は網であって壁ではなく、主の防衛は AI の権限の制限・方針に係るパスの除外・マージの実行を人に残すこと。LLM に判定させる方法は主の検査にしない。見本を `dev-autopilot/test/` に持ち vitest で検査する |
| I12 | 受け入れの自動化。`agent-ready` も受け入れの印も無く、マイルストーンがある Issue に対し毎回：検疫を題名・本文・コメントに通す（有りなら `needs-human`）、作業指示の下書き（著者がコラボレータなら検疫を通した本文、外の人なら受け入れ AI が書く）、計画の難しさの判定（すべての Issue。基準：複数リポにまたがる、設計判断、原因不明の障害解析、大規模な計画 → `plan=fable`、他は `opus`）、依存の候補（本文の `#n` だけ。機械）、マイルストーンの候補（open な 0.x のうち版が最小のもの。人が先に付けていればそれ）を「受け入れの下書き」コメントに `intake` の印付きで投稿する。下書きは agent brief の形（ファイルパスと行番号を書かない、手順でなく振る舞いを書く、受け入れ条件を 1 つずつ検証できる形で書く、範囲外を明記する）。1 回の実行で受け入れる件数に上限。受け入れ AI は道具なし、モデルは `claude-haiku-5-5`。人は下書きを編集するか `## 作業指示` を書き（優先）、`agent-ready` を付ける。進行役は次回に付けた人の権限を確かめ、渡す文のハッシュを取り、`intake-final` の印を新しいコメントで投稿し、依存とマイルストーンを確定する。下書きを読まずに `agent-ready` を付けると関所が無くなる |

AI が読む文と出どころ：

| AI が読む文 | 出どころ | 扱い |
| --- | --- | --- |
| Issue の本文・作業指示 | コラボレータ | 検疫を通し、作業の内容の欄に入れて渡す |
| Issue の題名・本文・コメント（受け入れ） | すべての Issue | 受け入れ AI だけが読む。作業 AI・レビュー AI には渡さない |
| 計画コメント | 進行役が投稿した計画役の出力 | `plan` の印を確かめ、検疫を通して作業 AI に渡す |
| PR のコメント | 進行役とコラボレータ | 進行役の投稿は R3-5 の 3 条件、コラボレータの文は検疫 |
| 差分、CI の結果、変更ファイルの一覧 | 作業 AI の出力と GitHub の API | 進行役が機械で付けた事実として渡す。作業 AI の説明文は渡さない |
| リポジトリのファイル | `develop` の内容 | 規約の文書は base ブランチから渡す。外の人の PR は扱わない |
| テストとコマンドの出力 | 作業 AI のコード | 出力を指示として扱わない旨を固定の指示文に書く |
| 依存パッケージ | npm | 熟成期間と lockfile。通信は許可した先だけ |
| Web | — | 渡さない |

### 4.9 導入と点検

| 番号 | 要件 |
| --- | --- |
| S1 | 導入（`/dev-autopilot setup`）は足りないものを作り、冪等。まず一覧を出し人が承認してから作り、作った後に読み戻して確かめる。作れるもの：ラベル、Project の Status の選択肢（`updateProjectV2Field` に既存の選択肢を `id` 付きで含めて送る。落とすと ID と値が消える）と Agent 欄、base ブランチの ruleset（PR 必須、force push 禁止、必須チェックの strict、`require_extra_approval_for_unattributed_changes` 無効）、deploy key の push 先を `<型>/issue-*` に限りタグを禁じる ruleset、設定ファイル、cron（launchd の plist）、worktree・記録・lock の置き場。人がやるもの：SSH 鍵、PAT、専用マシン、利用枠、Project の自動追加の有効化、Issue の承認 |
| S2 | 点検（`/dev-autopilot check`）は前提を機械で確かめ fail / warn / pass で返す。進行役は毎回の最初に回し、fail が 1 つでもあれば止まる。見るもの：ラベルと Project の欄、ruleset、設定ファイルの項目と ID の実在と型、必須チェックの一覧が空でないこと、`gh` の権限、SSH 鍵で署名できるか、`claude` と Node の版、Skill の有無、前回の記録、lock の状態（古い lock、見送った実行、起動されなかった日は warn）、起動の間隔と launchd の plist のずれ（warn）、子プロセスに PAT と鍵が見えないこと（実際に試す）、sandbox が実際に読めない・書けないこと。worktree の残骸は warn にし、Issue に紐づく残骸はその Issue だけ `needs-human`、どの Issue か分からない残骸だけ fail |
| S3 | 設定ファイルは 1 つ。導入が書き、点検が読み、進行役が使う。人が直接編集してもよい |
| S4 | business-os への導入は最初の利用者として `setup` で行い、手で作らない |

### 4.10 Issue の状態の遷移

状態は Project の Status（`Todo` → `In Progress` → `In Review` → `Ready to Merge` → `Done`、止まったら `Blocked`）、Agent 欄（空 / `worker` / `reviewer` / `human` / `paused`）、ラベル（`agent-ready` `needs-human` `from-review` `plan:fable` `plan:opus`）の 3 つで表し、正は GitHub に置く。

正常な順：

| # | 契機 | 変える人 | Status | Agent 欄 | ラベル |
| --- | --- | --- | --- | --- | --- |
| 1 | Issue が起票され Project に入る | 自動 | `Todo` | 空 | 無し |
| 2a | 受け入れの下書きを投稿する | 進行役 | 変えない | 変えない | 変えない |
| 2b | 下書きを読み、処理対象にする | 人 | 変えない | 変えない | `agent-ready` |
| 2c | `agent-ready` を検出し、渡す文を確定し、依存とマイルストーンを確定する | 進行役 | 変えない | 変えない | 変えない |
| 3 | 棚卸しで選び、基線が緑なら着手する | 進行役 | `In Progress` | `worker` | 変えない |
| 4 | 計画を `plan` の印付きで投稿する（段階 2〜4 は次回まで人が止められる） | 進行役 | 変えない | 変えない | 変えない |
| 5 | draft PR を作る | 進行役 | `In Review` | `reviewer`（段階 2 は人がレビューする） | 変えない |
| 6 | 対応ループ。載せ直しで差分が変わったときもここに戻る | 進行役 | 変えない | `worker` → `reviewer` | 変えない |
| 7 | 収束。🟢 を Issue にしてから PR を ready にする | 進行役 | `Ready to Merge` | 空 | 変えない |
| 8 | マージの可否を判定し印を書く（段階 4 は記録だけ） | 進行役 | 変えない | 変えない | 変えない |
| 8b | base が進んだので載せ直す。差分が同じなら印を付け直し、変わったら 6 に戻す | 進行役 | `Ready to Merge` か `In Review` | 空か `reviewer` | 変えない |
| 9 | 印を見てマージする | 人 | 変えない | 変えない | 変えない |
| 10 | マージを検出し Issue を閉じ worktree を消す | 進行役 | `Done` | 空 | 変えない |

止まる遷移：

| # | 契機 | 変える人 | Status | Agent 欄 | ラベル |
| --- | --- | --- | --- | --- | --- |
| B1 | 往復の上限か振動 | 進行役 | `Blocked` | 変えない | `needs-human` |
| B2 | ADR が要る、基線が赤、実行の失敗が 3 回、累計の利用枠の上限、漏えいの検査、進行役が実行するパスへの変更、渡す文の変更、検疫、Issue の状態の変化、権限不足、復旧できない状態、Issue に紐づく残骸 | 進行役 | `Blocked` | 変えない | `needs-human` |
| B3 | 実行の失敗（3 回未満） | 進行役 | 変えない | 変えない | 変えない |
| B4 | 人が引き取る | 人 | 変えない | `human` | 変えない |
| B5 | 人が一時停止する | 人 | 変えない | `paused` | 変えない |
| B6 | 着手前に依存の相手が閉じていない | 進行役（判定だけ） | 変えない | 変えない | 変えない |
| B7 | 作業中に人が依存を足した | 進行役（判定だけ） | 変えない | 変えない | 変えない |

決まり：`needs-human` を付けるときはどの契機でも Status を `Blocked` にする。進行役は Agent 欄を `human` と `paused` にしない。`from-review` の Issue は `Todo` で Project に入れ、人がマイルストーンを付けたら受け入れが走る。Status と Agent 欄の更新は進行役だけが行い冪等。段階ごとの振る舞いは設定の `stage` で切り替える。

戻す手順（人だけが行う）：`needs-human` は原因を直してからラベルを外し、Status を戻す（PR が無ければ `Todo`、PR があれば `In Review`。Agent 欄が `human` なら `worker` に戻す）。`human` を返すときは Agent 欄を `worker` に戻し、Status は触った内容に合わせて人が直す。`paused` を再開するときは Agent 欄を空か `worker` に戻す。依存の待ち（B6・B7）は人の操作が要らず、待ちを解きたければ依存を外す。

### 4.11 モデルの流れ

| 順 | 工程 | 役 | モデル / effort | 切り替わる条件 |
| --- | --- | --- | --- | --- |
| 1 | 受け入れ | 受け入れ AI | `claude-haiku-5-5` / medium | 拒否で止まったら実行の失敗。次回は `sonnet` |
| 2 | 計画の難しさの判定 | 受け入れ AI | 同上 | 印に `plan=opus` か `fable`。人のラベルが優先 |
| 3 | 計画 | 計画役 | 既定 `opus` / xhigh。印かラベルが `fable` なら `fable` / high | `opus` の自己申告で 1 回だけ `fable` で計画し直す |
| 4 | 実装 | 作業 AI | `opus` / high から始め、段階 2 で種類ごとに `sonnet` / high へ下げる | 上限は `opus` |
| 5 | レビュー | レビュー AI | `opus` / xhigh（段階 3 で `fable` と比べる） | 設定で変える。往復の途中では変えない |
| 6 | 批評 | 批評者 | `opus` / high | 変えない |
| 7 | 修正（2 ラウンド目以降） | 作業 AI | `sonnet` なら `opus` / high に 1 段上げる | 往復 3 回で `needs-human` |
| 8 | 再レビューと再批評 | レビュー AI、批評者 | 5・6 と同じ | 載せ直しで差分が変わらなければ回さない |

決まり：

- モデルを上げるのは 3（自己申告）と 7（2 ラウンド目）の 2 か所だけで、1 段・1 回まで。受け入れの拒否後の `sonnet` は回復であって上げる扱いではない
- モデルは `--model` で渡し（エージェント定義の `model` より優先）、effort は `--effort` で渡す。`opus` の既定の effort は medium なので表の値を明示する（`haiku` の既定は medium、`sonnet` と `fable` は high）
- どの役にも `fable` を既定にしない。`fable` が走るのは、受け入れの判定か人のラベルか自己申告で選ばれた計画と、段階 3 の比較のレビューだけ
- `fable` を使う役のエージェント定義には、無人で動かすときのクセを抑える指示を入れる：途中でターンを終えず最後まで実行する、依存しないツールの呼び出しはまとめて要求する、頼まれていない修正を足さず「見つけたこと」として返す、差分で編集する、区切りごとに出力する。自己検証の指示は削らない
- 利用枠の見方：1 Issue で走るセッションは、受け入れ 1 回、計画 1〜2 回、実装 1 回、レビューと批評が往復ごとに 1 組（2 セッション）、修正が往復ごとに 1 回。R2-13 の累計の上限はこの合計に掛かる

### 4.12 印と固定の見出し

印は 1 行の HTML コメント `<!-- dev-autopilot: <種類> <欄>=<値> ... sig=<HMAC> -->`。`sig` は `sig` 以外のすべての欄を進行役の鍵で署名したもの。真正性は R3-5 の 3 条件で確かめる。実装は `src/marker.ts` が持ち、本節と一致させる。

| 種類 | 置き場 | 欄 | 人の編集 | 読む人 |
| --- | --- | --- | --- | --- |
| `intake` | Issue の受け入れの下書き | `body=pass\|replace`（参考）、`brief`、`plan=opus\|fable`、`quarantine=none\|found` | 許す | 人。進行役は `plan` と `quarantine` だけ |
| `intake-final` | `agent-ready` の後に進行役が新しく投稿 | `brief`（渡す文のハッシュ）、`source=body\|instruction\|draft`、`plan` | 許さない | 進行役 |
| `plan` | Issue の計画コメント | `head`（base の SHA）、`plan`（計画のハッシュ）、`model` | 許さない | 進行役 |
| `review` | PR のレビュー結果 | `round`、`head`、`high`、`medium`、`low`、`verdict=pass\|fix`、`dropped_high` | 許さない | 進行役 |
| `fix` | PR の対応コメント | `round`、`head` | 許さない | 進行役 |
| `merge` | PR のマージ判定 | `head`、`merge=ready\|human`、`reasons` | 許さない | 人、進行役 |

人が書く固定の見出し（push 以上の人が書き、印を持たないコメントだけを読む）：

| 見出し | 置き場 | 意味 |
| --- | --- | --- |
| `## 作業指示` | Issue | 外の人の Issue に対するコラボレータの作業指示。下書きより優先 |
| `## 却下 R<n>` | PR | ラウンド n の指摘のうち人が却下したもの |
| `## 判定と違う判断` | PR | `merge=ready` の PR をマージしない理由 |
| `## 異議` | Issue、PR | 段階 1〜2 の出口で数える、選定・待機・下書き・計画への人の異議 |

進行役が書く人向けの見出し（`## レビュー結果（基点: ...）`、`## 再レビュー結果 R<n>（対象: ...）`、`## R<n> の指摘への対応（<コミット>）`、`## 受け入れの下書き`、`## 計画`、`## マージの判定`）は本文の形であって判定には使わない。進行役が投稿するコメントの先頭には「AI が生成した」の 1 行を付ける。

## 5. 安全側の規則

| 番号 | 要件 |
| --- | --- |
| SF-1 | AI の 5 役は GitHub に書けない。`gh` の認証・`GH_TOKEN`・進行役の鍵を渡さず、子プロセスの環境変数は許可した名前だけで組み立てる。push・PR・コメント・Project の更新は進行役だけが行う。進行役も `gh pr merge` を呼ばず、`develop` と `main` への直 push と force push はしない（ruleset でも止める） |
| SF-2 | 進行役・次のセッション・CI は、AI が書いた worktree の中身を信頼して実行しない。(a) AI のセッションは `--setting-sources local` で起動し、Plugin は進行役専用 clone の絶対パスで渡す。(b) 進行役は worktree の中でも AI の作業場所の親 clone の中でも git を実行せず、worktree のブランチを自分の clone に fetch してから merge・patch-id・diff・漏えい検査・push を行う。親 clone に対しては worktree の作成と削除だけを行う（`commondir` が差し替えられた worktree には行わず `needs-human`）。(c) `.claude/**` `CLAUDE.md` `.mcp.json` `.github/**` `dev-autopilot/**` に触れる diff は push しない。必須チェックは名前に加え app とワークフローのパスで照合する。(d) 進行役はリポのコードとパッケージの lifecycle script を sandbox の外で実行しない。(e) sandbox は Bash だけを覆うので、Read・Write・Edit は `permissions` で絞る：allow は `Bash` `Read` `Write(//<worktree>/**)` `Edit(//<worktree>/**)`、deny は `Read(//<秘密のパス>/**)` と、Write・Edit の `//<親 clone>/.git/**` `//<worktree>/.claude/**` `//<worktree>/CLAUDE.md` `//<worktree>/.mcp.json`。`--settings` のファイルは AI が書けない場所に置き起動前にハッシュを確かめる。一時フォルダはセッションごとに別にする |
| SF-3 | sandbox の設定：有効、`allowUnsandboxedCommands: false`、`~/.ssh` `~/.config/gh` と進行役の設定・記録の読み取り禁止、外への通信は許可した先（npm のレジストリ）だけ。`denyRead` `denyWrite` のパスは `setup` が実体に解決して書く。`.git/worktrees/<名前>/gitdir` を `denyWrite` に足す（sandbox が既定で守らない）。`check` は「設定に書いてある」でなく「実際に読めない・書けない」ことを試して確かめる |
| SF-4 | 作業 AI の commit は署名なし。`develop` への squash マージの commit は GitHub が署名する。再署名の仕組みは持たない。`develop` の ruleset に署名必須は入れない |
| SF-5 | 進行役は専用の clone を 2 つ持つ：AI の worktree を切る「AI の作業場所」の clone と、進行役だけが書き `src/` を実行し fetch・merge・push を行う「進行役の clone」。進行役は共有の clone の refs を信頼せず、使う前に必ず `origin` から fetch し、base は `origin/develop`、PR の head は GitHub の API が返す SHA だけを基準にする。git の設定は環境変数（`GIT_CONFIG_COUNT`）で渡し、`core.hooksPath` を空のフォルダに固定する。AI のセッションの git では自動 gc を止める |
| SF-6 | `--max-turns`、時間の上限、`--max-budget-usd` で 1 回の作業の長さと費用を止める |
| SF-7 | 判定できない状態（API の失敗、想定外の Status、残骸）では作業を増やさず記録して止める |
| SF-8 | AI に渡す文は書いた人を問わず渡す直前に毎回検疫を通す。AI の出力が公開の場に書かれる前に漏えいの検査を掛ける。処理対象の判断は人。マージの実行は人 |
| SF-9 | 状態の正は GitHub に置く（PR の有無と draft、印、Status、Agent 欄、ラベル）。進行役は毎回そこから状態を組み立て、手元の記録は補助にする。各手順は「すでに済んでいれば何もしない」（PR があれば作らない、コメントがあれば足さない）。進行役が途中で死んだら次回の実行が GitHub の状態から復旧し、復旧できない組み合わせ（PR はあるが worktree が無い、など）はその Issue を `needs-human` にする |
| SF-10 | 多重起動を lockfile で防ぐ。launchd の自動実行と人の `/dev-autopilot` の手動実行も同じ lock を取る。lock には PID・ホスト名・起動時刻を書き、PID が生きていなければ古い lock として奪う。lock で見送った実行と launchd が起動しなかった日は記録に残し、`check` が warn に出す |
| SF-11 | 止める合図は設定の `enabled: false`（全体）と Agent 欄 `paused`（その Issue）。書かずに試す合図は `dryRun: true` で、GitHub への書き込みをすべて手元の記録に「書くつもりだった内容」として残して実際には書かず、読むことと AI のセッションは行う。Agent 欄が `human` の Issue と PR には進行役は触らない |

## 6. 段階と出口

| 段階 | 内容 | 出口 |
| --- | --- | --- |
| 0a | 橋渡し ADR（business-os 側） | accepted（済み） |
| 0b | 手で書いた安全設定で `claude -p` を動かし、未確認を実機で潰す | 済み（付録 A） |
| 0c | `setup` と `check` の実装と準備（P1・P2・P4・P5）、deploy key と PAT、`change-review` v2、受け入れ、`/verify-issue` の拡張。Project の選択肢の追加は捨てる Project で先に試す | `check` が business-os で全項目 pass（`setup` が書いた安全設定で実体のパスの解決と読み取りの検査が動くことを含む）。v2 は手動のレビュー 1 件で今の PR の形が出る |
| 1 | 棚卸しと受け入れ（実装はしない）。最初は `dryRun` で下書きと選定を記録にだけ書き、異議が無ければ切る | 7 回の実行（lock で見送った実行と起動されなかった日は数えない）で `## 異議` 0 回（dry-run の間は記録ファイルへの追記）、うち投稿ありが 3 回以上 |
| 2 | 計画役と実装。レビューとマージは人。作業 AI は `opus` で始め種類ごとに `sonnet` を試し（R4-2b で上げた件数を数える）、計画役の `plan=` の判定を人が妥当と見たかを記録する | 5 件のうち 4 件以上を手直しなし（人が PR に commit を足していない、close して作り直していない）でマージ。`plan=` の判定への `## 異議` の割合を記録する |
| 3 | レビュー AI と批評者。`opus` を既定に、同じ PR を `fable` でも回して比べる（比較は印を書かない）。人の二重レビューは 5 件に限る | 5 件で、一致率（人の 🔴 🟡 のうち AI も出した割合）80% 以上、却下率（AI だけが出した 🔴 🟡 のうち `## 却下` に書かれた割合）15% 以下。`opus` と `fable` の一致率・却下率・利用枠を並べて既定のモデルを決める |
| 4 | 対応ループ。マージ判定は記録だけ（shadow） | 5 件のうち 4 件以上が往復 3 回以内で `Ready to Merge` |
| 5 | マージ判定の印を書く。実行は人 | 10 件で判定と人の判断が全件一致（R7-2b の数え方）、revert 0 件（revert＝マージから 14 日以内に `develop` にその commit を revert する commit か、本文でその PR 番号を「戻す」と言及する fix の PR がマージされたもの）。満たしても実行は人のまま |
| 6 | 並列と間隔の短縮 | R6-2 の条件 |

## 7. 準備（リポジトリに足りないもの）

| 番号 | 対応 |
| --- | --- |
| P1 | Project の Status に `In Review` `Ready to Merge` `Blocked` を足す（`setup`） |
| P2 | Project に単一選択欄「Agent」を足す。Assignee は人のまま（`setup`） |
| P3 | Issue 依存は受け入れが候補を出し、`agent-ready` の後に進行役が確定して入れる |
| P4 | ラベルを `setup` が足す。名前は設定に持つ |
| P5 | `develop` に ruleset を掛ける（`setup`） |
| P6 | 人の署名は手元の SSH 鍵に移す。dev-autopilot は人の鍵を使わない |
| P7 | `dev-autopilot/` を pnpm の workspace のパッケージにする（9 節） |
| P8 | 利用枠の上限を設定に持つ |
| P9 | アカウントは最初はメンテナの 1 つ。人と AI の投稿は印で見分ける。分けたくなったら自動化用のアカウントを足す（判定の仕組みは変えない） |
| P10 | Issue のフォームに受け入れ条件の欄は足さない。作業の定義はコラボレータの文にある |
| P11 | 渡す文の編集はハッシュで検出する |
| P12・P15 | 受け入れは進行役が自動で行う。検疫は TypeScript で vitest で検査する。`/verify-issue` は手動の補助 |
| P13 | AI のセッションには認証を渡さない。sandbox を掛け、鍵を分ける |
| P14 | レビューの Skill は `dev-autopilot/plugin/skills/review/` に持ち `~/.claude` に依存しない |

## 8. 設定に持つ値

設定ファイル（適用先のリポの `.claude/dev-autopilot.json`。S1）に持つ項目と、要件が決めた初期値。「未定」は要件メモが決めていない値で、設計か ADR で決める。変えるときは設定だけを変える。

| 項目 | 初期値 | 根拠 |
| --- | --- | --- |
| リポと Project：owner / repo、base ブランチ、Project の番号、Status と Agent 欄の ID と選択肢の ID | business-os、`develop`、Project 7。ID は `setup` が API で引く | R1-1、P1、P2 |
| ラベル名 | `agent-ready` `needs-human` `from-review` `plan:fable` `plan:opus` | P4 |
| Issue のラベルからブランチの型への対応表 | 未定（例：`bug` → `fix`、他 → `feat`） | R2-2 |
| 固定の見出しの文字列 | 4.12 節のとおり | 4.12 節 |
| 進行役の実行：`enabled`、`dryRun`、`stage`、dev-autopilot のフォルダの絶対パス、worktree・記録・lock の置き場 | `dryRun` は段階 1 の最初は `true`。置き場は未定 | SF-11、R1-4、C-4 |
| 起動の間隔 | 1 日 1 回。狭めるときは 12 時間 → 6 時間 | 2 節、R6-2 |
| 1 回の実行で着手する件数 | 1 | R1-3 |
| WIP（進行中の Issue）の上限 | 1（段階 5 まで） | R1-3、6 節の段階 6 |
| 1 回の実行の所要時間の上限 | 未定 | R1-5 |
| 1 回の実行で受け入れる件数 | 未定 | I12 |
| 往復の回数の上限 | 3 | R4-2 |
| セッションの時間・ターン・費用の上限 | 45 分・200 ターン。費用は未定 | R2-12 |
| 1 Issue の累計、1 回の実行、1 日の利用枠 | 未定 | R2-13、P8 |
| 実行の失敗の再試行の回数と間隔 | 3 回。間隔は未定 | R2-13 |
| 催促までの日数 | 7 | R7-2b |
| 間隔を狭める条件の実行回数 | 連続 7 回 | R6-2 |
| 役ごとのモデルと effort、2 ラウンド目に上げる先、受け入れの拒否後のモデル | 4.11 節のとおり。上げる先は `opus`、拒否後は `sonnet` | 4.11 節 |
| Agent ツールを許すか | 未定（既定は許さない想定） | R2-13 |
| 批評者の確信度のしきい値 | 未定（参考値 80） | R3-7 |
| 品質ゲートのコマンド | `pnpm check` | R2-4 |
| 必須チェック（名前・app・ワークフローのパス） | `check (ubuntu-latest)` `check (windows-latest)` `commitlint` | R4-3 |
| マージ判定の許可一覧 | `docs/usage/**` `.changeset/*.md` | R7-3 |
| 方針に係るパス | R7-3 の一覧 | R7-3 |
| 進行役・次のセッション・CI が実行するパス | `.claude/**` `CLAUDE.md` `.mcp.json` `.github/**` `dev-autopilot/**` | R2-14 |
| 漏えい検査の見本の置き場、検疫の見本の置き場 | `dev-autopilot/test/` の前提データ | I11 |
| 規約：言語、Conventional Commits、changeset の要否 | 日本語、要、要 | R2-4 |
| 依存パッケージの熟成期間 | 7 日 | 4.8 節の表 |
| 進行役のアカウント名 | メンテナのログイン | R3-5、P9 |
| 安全設定の置き場、`allowWrite` と `denyWrite` の一覧、通信の許可先 | 通信は npm のレジストリだけ。`allowWrite` の扱いは D-13 | SF-2、SF-3 |
| 子プロセスに渡す環境変数の名前 | `HOME` `PATH` `USER` `LOGNAME` `TMPDIR` `TERM` `LANG` | 付録 A |

PAT と署名の鍵と deploy key は設定に書かず、進行役のプロセスの環境変数で渡す。

## 9. 構成と分離

| 番号 | 要件 |
| --- | --- |
| C-1 | `dev-autopilot/plugin/`（配布物：名札、Skill の `dev-autopilot` `setup` `check` `review` `intake`、agent の 5 定義）、`src/`（進行役。配らない）、`fixtures/`、`test/`（結合テストだけ）、`docs/`（要件、ADR、設計）、`README.md`、`.markdownlint-cli2.jsonc` の構成。bash は置かない |
| C-2 | business-os の中身を import しない。逆もしない |
| C-3 | リポ固有の値はすべて設定ファイル 1 つに出す（8 節） |
| C-4 | 起動は Plugin として（`--plugin-dir <進行役専用 clone>/dev-autopilot/plugin`）。人が呼ぶ Skill は設定の絶対パスから `src/` を知る |
| C-5 | 検査は dev-autopilot 自身が持つ（pnpm の workspace のパッケージ。型検査・vitest・markdownlint）。business-os は `check:dev-autopilot` で呼ぶだけ |
| C-6 | 文書は `dev-autopilot/docs/` に閉じる。ADR の規則は `docs/adr/README.md`（MADR、参照スタイルのリンク、AI は `proposed` 以外を付けない） |
| C-7 | `review` Skill は Plugin の中からも単体でも動くように書く。印の 1 行・無人の入力・agent の名前は引数で渡されたときだけ使う |
| C-8 | 分離は段階 5 まで business-os で動かし、設定の項目だけで別のリポに適用できると確かめてから |

## 10. 決めていないこと

基本設計のレビュー（2026-10-11）で、要件に無い判断か要件の内部の矛盾として挙がったもの。ADR で決める。

| 番号 | 論点 |
| --- | --- |
| D-1 | レビュー用 worktree の作り元と objects の供給。R3-1 の「自分の clone から」と SF-5（進行役の clone は進行役だけが書く）が矛盾する。AI の作業場所の clone に作り、`git fetch origin` だけを許すか |
| D-2 | 1 回の実行で進める段の数え方。R1-5 の 7 段に、承認の確定・収束・載せ直し・催促をどう含めるか。AI のセッションか CI 待ちを伴う段だけを 1 回に 1 つとするか |
| D-3 | pass なのに CI が赤・無い・保留のときの扱い。赤を修正に回して往復に数えるか、無い・保留は待つか |
| D-4 | 1 回の実行で `merge=ready` 1 件を超えた PR の扱い（`human` にすると一致の記録が汚れる） |
| D-5 | 基線が赤のとき。R2-8 は `needs-human`。進行中の Issue を続け、新規着手だけ止める案 |
| D-6 | 段階 2 の Agent 欄。4.10 節の遷移 5（`human`）と決まり（進行役は `human` にしない）が矛盾する |
| D-7 | 手元の記録を判定に使う例外（実行の失敗の回数、拒否後のモデル、累計の利用枠）と、記録を失ったときの扱い |
| D-8 | 印の体系の拡張。進行役の投稿すべてに印を付けるか（待ち・催促・警告・停止の理由）、署名の対象に種類・リポ・番号を含めるか、進行役の印が無効化されていたら `needs-human` にするか |
| D-9 | PR と Issue の結びつけ。`Closes #n` でなく head のブランチ名・同じリポ・作者・印の番号で引くか |
| D-10 | AI の読み取り範囲と秘密の置き場。Read の許可を worktree に絞りホーム全体を `denyRead` にするか。PAT・鍵・deploy key の置き場。手動実行を `launchctl kickstart` にするか |
| D-11 | 権限確認の範囲。`needs-human` を外す・Status を戻す・`plan:*` を付ける操作の人と、渡す文の編集者。渡す文の最終編集が `agent-ready` より後なら `needs-human` にするか |
| D-12 | 要件メモ 8 節から引き継ぐもの：レビュー側で `pnpm check` を実行するか、記録の置き場、自動化用アカウントの分離、クラウドの仕組みの利用 |
| D-13 | 親 `.git` の書ける範囲。要件メモの判断 1 は「他の Issue の `.git/worktrees/**` は書けない。書く場所の一覧を設定に持つ」としたが、同じ判断の訂正は「sandbox が既定で許すので `allowWrite` の列挙は要らない」とした。並列（段階 6）で他の Issue の `gitdir` や refs を書ける状態になる |
| D-14 | 検疫で見つかったときの印。I9・I12 は「下書きを作らず `needs-human`」、4.12 節の `intake` の印は `quarantine=found` を持つ。下書きを作らないなら印を載せる場所が無い |
| D-15 | I12 のマイルストーンの候補。受け入れの対象を「マイルストーンがある Issue」に限りながら、候補を出すことになっている |
| D-16 | 8 節で「未定」の値（実行時間、受け入れの件数、費用の上限、再試行の間隔、Agent ツール、確信度のしきい値、置き場）をどこで決めるか |

## 付録 A：実機で確かめた事実（2026-10-08〜11、Claude Code 2.1.285）

要件はこれらの事実に依存する。環境が変わったら確かめ直す。

| 事実 |
| --- |
| 未信頼の作業場所でも `--settings` の `permissions.deny` は効く |
| `claude -p --plugin-dir <入れ子>` と `--agent <plugin>:<agent>` は動き、agent から Plugin の Skill を呼べる |
| `-p` からサブエージェントを起こせる。`--max-budget-usd` は定額のプランでも効く |
| sandbox の中で `pnpm install`（レジストリを許可）と `pnpm check` 全体が通る。`prepare` の simple-git-hooks は hooks に書けず失敗するが install は成功する |
| fine-grained PAT（Organization、business-os だけ、Projects の読み書き）で Project 7 の読み取りと書き込みができる |
| 別名 `haiku` は Haiku 4.5 に解ける。完全な ID `claude-haiku-5-5` は動く。`--model` はエージェント定義の `model` より優先 |
| worktree で `git commit` が通る。sandbox は親 `.git` の `config` `hooks/**` `commondir` `config.worktree` と worktree の `.claude/settings.json` を Bash から守るが、`gitdir` と `refs/**` は書ける。Bash を `permissions.allow` に入れないと `git commit` は承認待ちで拒否される |
| 既定の `-p` は未信頼の作業場所でもプロジェクトの hook を実行し `CLAUDE.md` を読む。`--setting-sources local` でユーザーとプロジェクトの hook と `CLAUDE.md` が読まれなくなる。`--bare` は Plugin の agent を読まない |
| `commondir` を差し替えた worktree は、その中で git を実行したときだけ発火する。別の clone からの fetch と親での `worktree list/remove` は発火しない |
| sandbox の `denyRead` は Read ツールに効かない。`permissions.deny` の `Read(//絶対パス/**)` で止まる。Write・Edit の allow を `//<worktree>/**` に絞ると外への書き込みは拒否される |
| `tools: []` と `--tools ""` で道具なしになる。`--json-schema` は `--agent` と併用すると `structured_output` が空 |
| `stream-json` は `assistant` のイベントごとに `model` と `usage` を持つ。プロセスグループの kill で孫プロセスは残らない |
| `updateProjectV2Field` は既存の選択肢の `id` を含めれば ID と値を保ち冪等。落とすと ID が振り直され値も消える |
| リアクションでは `updated_at` は変わらない。非表示では変わるが `lastEditedAt` は null のまま |
| `env -i` に `HOME` `PATH` `USER` `LOGNAME` `TMPDIR` `TERM` `LANG` を渡せばログイン状態を保つ。`GIT_CONFIG_COUNT` で git の設定を渡せる |
| sandbox の `denyRead` はパスの途中のシンボリックリンクを解決しない。実体のパスを書く必要がある |

[memo]: ./dev-autopilot-requirements.md
