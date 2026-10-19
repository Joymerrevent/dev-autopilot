# 決めていないこと

要件の中で、まだ決めていないことの一覧。決め方で 3 つに分ける。

- ADR で決めること：要件の中で答えが食い違っているか、答えが無い論点。どちらにするかを選ぶ判断なので、ADR を起票して決め、決まったら関係する要件のファイルを直す
- 基本設計の中で決めてよいもの：要件の形を変えない細かい選択
- 確かめること：判断ではなく、事実を確かめれば済むもの

## ADR で決めること

### AI の作業場所

- UNDECIDED-1 **欠番**（[ADR-20261013-001][adr-workspace] で決めた）：~~**AI の作業場所の clone に新しい commit が届かない。** 進行役は AI の作業場所の clone で worktree を作る・消すだけなので、
  clone に無い commit（`develop` の新しい commit、人の push、載せ直しの結果）から worktree を作れない（[05-safety.md][f05]）~~
- UNDECIDED-2 **欠番**（[ADR-20261013-001][adr-workspace] で決めた）：~~**ほかの Issue の作業場所を書き換えられる。** 「[隔離と鍵の分け方][f05]」は、書ける場所の一覧（「他の Issue の `.git/worktrees/**` は書けない」を含む）をやめ、sandbox の既定に任せた。
  同じ clone に複数の Issue の worktree があると、ある Issue の AI が別の Issue の `gitdir` や refs を書き換えられる（[05-safety.md][f05]）~~
- UNDECIDED-3 **欠番**（[ADR-20261013-001][adr-workspace] で決めた）：~~**AI が書き換えた objects がレビューに届く。** AI が clone の objects のファイルを書き換えると、同じ clone から作ったレビュー用の worktree に書き換えた中身がエラーなしで出て、
  `git fetch origin` を実行しても直らない（FACT-18。[10-verified-facts.md][f10]）。上の項目と一緒に解く~~
- UNDECIDED-4 **欠番**（[ADR-20261013-001][adr-workspace] で決めた）：~~**`commondir` を書き換えられるかが食い違っている。** 「[隔離と鍵の分け方][f05]」は「AI は自分の `commondir` を書き換えられる」を前提に、進行役が worktree と親 clone の中で git を実行しないと決めている。
  一方、FACT-9 では sandbox が `commondir` を保護していて書けなかった（[10-verified-facts.md][f10]）。守りを多重の守りとして残すなら、前提の書き方を直す~~

### 流れと工程

- UNDECIDED-5 **欠番**（[ADR-20261015-006][adr-flow] で決めた）：~~**1 回の実行で進める「段」の数え方。** FLOW-4 は段を 7 つ（受け入れ〜片付け）と定めて「1 回の実行で 1 段だけ」とするが、
  「[Issue の状態の遷移][f01]」の FLOW-18（承認の確定）・FLOW-23（収束）・FLOW-25（載せ直し）と、MERGE-3 の催促が 7 つのどれに当たるかが無い。FLOW-20 の「段階 5〜7 では計画と同じ実行で実装に入らない」は、
  段階 8 以降なら同じ実行で実装まで進むと読め、「1 段だけ」と食い違う。元の文は IMPL-10 の「段階 8 以降は同じ実行で続け」（[01-issue-flow.md][f01]、[02-implement-review-fix.md][f02]）
  数え方の案として、「AI のセッションか CI 待ちを伴う段だけを 1 回に 1 つとする」がある（催促は [03-merge.md][f03]）~~
- UNDECIDED-6 **欠番**（[ADR-20261015-006][adr-flow] で決めた）：~~**基線が赤のとき。** IMPL-9 は `needs-human`。進行中の Issue を続け、新規着手だけ止める案（IMPL-9、FLOW-29。[02-implement-review-fix.md][f02]、[01-issue-flow.md][f01]）~~
- UNDECIDED-7 **欠番**（[ADR-20261015-006][adr-flow] で決めた）：~~**段階 5 の Agent 欄。** FLOW-21 と「段階による違い」は「段階 5 では Agent 欄を `human` にする」とし、
  同じ節の決まりは「進行役は Agent 欄を `human` にしない」としている（[01-issue-flow.md][f01]）~~
- UNDECIDED-8 **欠番**（[ADR-20261015-006][adr-flow] で決めた）：~~**手元の記録を使うかどうか。** FLOW-4 は「手元の記録は使わない」、運用の規則は「手元の記録は補助にする」としている。
  止まる遷移 FLOW-29・FLOW-30 の「同じ実行の失敗が 3 回」「累計の利用枠」は実行をまたいで数えるので、手元の記録が要る（[01-issue-flow.md][f01]）
  実行をまたいで覚える値には、受け入れ AI の拒否の後のモデル（IMPL-29。[02-implement-review-fix.md][f02]）もある。
  記録を失ったとき（追跡しないフォルダの掃除、専用マシンへの移行など）に、失敗の回数と累計の利用枠をどう扱うかも決める（例：`needs-human` に倒す、回数を GitHub の印に残す）。数が 0 に戻ると、上限の `needs-human` が黙って効かなくなる~~
- UNDECIDED-26 **上限で止まった作業を引き継ぐか。** IMPL-13 は上限に当たったセッションの途中の状態を commit せず、[ADR-20261013-001][adr-workspace] では次のセッションを GitHub にある状態（push 済みの commit と計画）から始める。
  1 セッションに収まらない大きさの作業は、3 回とも止まって `needs-human` になる（IMPL-14）。引き継ぐ案として、止まったセッションでも AI が commit した分だけを進行 Bot の clone へ取り込み、IMPL-15 の検査を通して作業ブランチへ push し、次のセッションを取り込んだ commit から始める形がある。
  IMPL-13 を変え、未完成の作業を GitHub に push することになる。計画・実装工程（段階 5）で上限に当たった回数を記録してから決める（[02-implement-review-fix.md][f02]）
- UNDECIDED-30 **欠番**（[ADR-20261018-002][adr-redo] で決めた）：~~**計画のやり直しを同じ実行で行うか。** 計画 AI の `opus` が自己申告したら、進行ボットは 1 回だけ `fable` で計画し直す（IMPL-10）。同じ実行で行うと、計画の工程で AI のセッションが 2 つになる。
  次の実行に回すなら、自己申告があったことと 1 回上げたことを GitHub に残す仕組み（`plan` の印の欄など）が要る。手元の記録は判定に使わない（[ADR-20261015-006][adr-flow]。[02-implement-review-fix.md][f02]）~~
- UNDECIDED-31 **欠番**（[ADR-20261018-003][adr-model] で決めた）：~~**作業 AI のモデルを上げた件数を、どの段階で数えるか。** IMPL-25 と段階の表は、段階 5（計画と実装を動かし、レビューは人が行う段階）で「`sonnet` で始めて `opus` で通った件数」を数えるとする。
  モデルを上げるのは修正のセッション（作業 AI の 2 ラウンド目以降）で、修正のセッションは段階 7（対応ループを足す段階）で足すので、段階 5 には上げる場面が無い（[02-implement-review-fix.md][f02]、[06-setup-and-stages.md][f06]）~~

### レビューとマージ

- UNDECIDED-9 **欠番**（[ADR-20261016-001][adr-pr] で決めた）：~~**pass なのに CI が緑でないとき。** pass なのに CI が赤・無い・保留のときの扱い。赤を修正に回して往復に数えるか、無い・保留は待つか（IMPL-27。[02-implement-review-fix.md][f02]）~~
- UNDECIDED-10 **欠番**（[ADR-20261016-001][adr-pr] で決めた）：~~**2 件目以降の `merge=ready`。** 1 回の実行で `merge=ready` 1 件を超えた PR の扱い（`human` にすると一致の記録が汚れる）（MERGE-5。[03-merge.md][f03]）~~
- UNDECIDED-32 **欠番**（[ADR-20261017-002][adr-review] で、進行 Bot が軸ごとに別のセッションを起こし、Agent ツールはどの役にも与えず、設定の項目も無くすと決めた）：~~**レビュー AI のサブエージェントと Agent ツール。** REVIEW-1 は 3 つの軸を並列のサブエージェントで見るとするが、「[登場するもの][f00]」のレビュー AI の道具に Agent が無い。
  [ADR-20261015-005][adr-settings] は、Agent ツールを許すことを、安全の境界を緩める変更として ADR で決めるとしている。設定の「Agent ツールを許すか」（IMPL-14、STRUCT-2）が役ごとかも決まっていない（[08-review-skill.md][f08]）~~
- UNDECIDED-33 **欠番**（[ADR-20261017-002][adr-review] で、PR のコメントはどの役にも渡さないと決めた）：~~**コラボレータの PR のコメントを AI に渡すか。** TEXT-1 と REVIEW-6 は、コラボレータのコメントは渡す、PR のコメントも同じ、とする。一方、渡す文の範囲は `agent-ready` の時点で確定し、後から足されたコメントは渡さない（TEXT-1）。
  再レビューの入力（IMPL-21）と修正の入力（IMPL-23）にも、コラボレータのコメントは無い。PR のコメントはすべて `agent-ready` の後に付くので、渡すのか、渡すならどの役のどの入力かが決まらない（[04-external-text-and-markers.md][f04]、[08-review-skill.md][f08]）~~
- UNDECIDED-34 **欠番**（[ADR-20261017-002][adr-review] で、規約の正典に触れる diff を push せず、レビュー AI は作業場所の head の正典を読むと決めた）：~~**レビュー AI に渡す規約の正典を、base から取るか。** 「AI が読む文」の表は「規約の正典は base ブランチから渡す」とするが、レビュー AI の作業場所は PR の head で作る（IMPL-16）。
  作業 AI は head の `docs/adr/**` などを書き換えうる（IMPL-15 の push を止めるパスに入っていない。マージの判定では `merge=human`）。レビュー AI が書き換えられた規約を読まないようにするか、どう渡すかが決まっていない（[04-external-text-and-markers.md][f04]、[02-implement-review-fix.md][f02]）~~
- UNDECIDED-51 **欠番**（[ADR-20261018-004][adr-planrev] で決めた）：~~**計画をレビューするか（誰が、いつ）。** 計画 AI の計画は、進行ボットが印を付けて Issue にコメントするだけで、AI はレビューしない（IMPL-10。レビュー AI が見るのは PR の差分だけ。REVIEW-1）。
  段階 5〜7（計画と実装を AI に任せ、レビュー AI と対応ループを順に足す段階）では、人が計画を読んで止めることができるが、読むのは決まりではない。段階 8（進行ボットがマージしてよいかを判定し始める段階）以降は、計画は記録として残るだけになる（IMPL-10、FLOW-20）。
  計画が誤っていると、実装と修正の手直しが大きくなり、往復の回数と利用枠が増える。PR の出来が悪いときに、原因が作業 AI のモデルなのか計画なのかを分けられず、最初の実装に使うモデルの比べ方（[ADR-20261018-003][adr-model]）も狂う。
  計画を誰がレビューするか（AI、人、両方）、いつ行うか（計画を投稿する前か、段階 5〜7 だけか、段階 8 以降もか）、レビューで問題が見つかったときにどうするか（計画し直す回数）を決める（[02-implement-review-fix.md][f02]、[01-issue-flow.md][f01]）~~

### 受け入れと印

- UNDECIDED-11 **欠番**（[ADR-20261015-004][adr-markers] で決めた）：~~**印の体系の拡張。** 進行役の投稿すべてに印を付けるか（待ち・催促・警告・停止の理由）、署名の対象に種類・リポ・番号を含めるか、進行役の印が無効化されていたら `needs-human` にするか（「[印と固定の見出しの一覧][f04]」。[04-external-text-and-markers.md][f04]）~~
- UNDECIDED-12 **欠番**（[ADR-20261016-001][adr-pr] で決めた）：~~**PR と Issue の結びつけ。** `Closes #n` でなく head のブランチ名・同じリポ・作者・印の番号で引くか（IMPL-6。[02-implement-review-fix.md][f02]）~~
- UNDECIDED-13 **欠番**（[ADR-20261015-004][adr-markers] で決めた）：~~**権限確認の範囲。** `needs-human` を外す・Status を戻す・`plan:*` を付ける操作の人と、渡す文の編集者。渡す文の最終編集が `agent-ready` より後なら `needs-human` にするか（TEXT-10、「[Issue の状態の遷移][f01]」の戻す手順。[04-external-text-and-markers.md][f04]、[01-issue-flow.md][f01]）~~
- UNDECIDED-14 **欠番**（[ADR-20261015-003][adr-intake] で決めた）：~~**検疫で見つかったときの印。** TEXT-9 は「見つかったら下書きを作らない」と書いた直後に「下書きコメントに『混入の検査：有り（種類）』の行を必ず書く」としている。
  TEXT-12 も「有りなら下書きを作らない」。一方、「[印と固定の見出しの一覧][f04]」の `intake` の印は `quarantine=found` を持ち、下書きを作らないなら印を載せる場所が無い（[04-external-text-and-markers.md][f04]）~~
- UNDECIDED-15 **欠番**（[ADR-20261015-003][adr-intake] で決めた）：~~**受け入れの対象とマイルストーンの候補。** TEXT-12 は受け入れの対象を「マイルストーンが付いている Issue」に限りながら、マイルストーンの候補を出すことになっている。
  「[決まっていること][f00]」の「人の関所」と FLOW-15〜FLOW-16 は、起票されたら受け入れる流れで、マイルストーンを条件にしていない（[04-external-text-and-markers.md][f04]、[00-overview.md][f00]、[01-issue-flow.md][f01]）~~
- UNDECIDED-35 **欠番**（[ADR-20261016-002][adr-draft] で、Issue の本文を著者を問わず作業 AI に渡さない形に決め、見分ける必要が無くなった）：~~**進行ボットが起票した Issue の本文の見分け方。** 🟢 から起票した Issue（IMPL-26）の本文は AI の出力として扱う（TEXT-1）が、印の一覧に、Issue の本文に付ける印が無い。
  進行ボットは当面メンテナと同じアカウントで投稿する（SETUP-13）ので、著者のアカウントでは見分けられず、本文がコラボレータの文（`body=pass`）として作業 AI に届きうる。
  [ADR-20261015-004][adr-markers] は、印の無い進行ボットのアカウントの投稿を人の投稿として読むとしている（[04-external-text-and-markers.md][f04]、[02-implement-review-fix.md][f02]）~~

### 安全と設定

- UNDECIDED-16 **欠番**（[ADR-20261015-001][adr-read] で決めた）：~~**AI の読み取り範囲と秘密の置き場。** Read の許可を worktree に絞りホーム全体を `denyRead` にするか。PAT・鍵・deploy key の置き場。手動実行を `launchctl kickstart` にするか（SAFE-2 の (e)。[05-safety.md][f05]）~~
- UNDECIDED-17 **欠番**（[ADR-20261015-005][adr-settings] で決めた）：~~**まだ値が決まっていない設定。** 実行時間、受け入れの件数、費用の上限、再試行の間隔、Agent ツール、確信度のしきい値、置き場の値をどこで決めるか（設定の項目の一覧は STRUCT-2。[07-structure.md][f07]）~~
- UNDECIDED-25 **欠番**（[ADR-20261015-002][adr-store] で決めた）：~~**依存の store をセッションの間で共有するか。** AI のセッションは作業場所の中で依存を導入する（IMPL-1）。pnpm の store をセッションの間で共有すると、前のセッションが細工したものを次のセッションが読む経路になりうる（objects と同じ種類の論点。[ADR-20261013-001][adr-workspace]）。
  手元の Mac（APFS）では store と `node_modules` のファイルを共有しなかったが、専用マシンのファイルシステムでは確かめていない。セッションごとに空の store を使うと、`pnpm install` は 1.1 秒から 3.0 秒になった（FACT-21。[10-verified-facts.md][f10]）~~
- UNDECIDED-36 **欠番**（[ADR-20261017-004][adr-human] で決めた）：~~**AI のセッションが Plugin を読めるか。** Plugin は、進行ボットのプログラムの写し（`devAutopilotPath`。置き場の中。[ADR-20261017-003][adr-plugin] の決定 2）を `--plugin-dir` で指す（STRUCT-3）。一方、[ADR-20261015-001][adr-read] で、AI のセッションは置き場を読めなくした。
  エージェント定義の読み込みは確かめた（FACT-2、FACT-26）が、レビュー AI が Skill の参照ファイルを Read で読むことと、`review/scripts/` を sandbox の中の Bash で実行することは、拒否されるはず（確かめていない）。Plugin の置き場か、読み取りの許可を決める（[07-structure.md][f07]、[05-safety.md][f05]）~~
- UNDECIDED-37 **欠番**（[ADR-20261017-004][adr-human] で決めた）：~~**人が呼ぶ `setup` と `check` の認証と、マシンの設定の見つけ方。** `setup` と `check` は人の Claude Code のセッションから動き、GitHub への書き込み（ラベル・Project の欄・ruleset）と、PAT の権限の確かめを行う。
  ADR-20261015-001 は人のセッションに秘密を扱わせないとし、専用マシンには人の `gh` のログインを置かない（「[決まっていること][f00]」）。
  どの認証で動くかと、人が呼ぶ Skill がマシンの設定（`devAutopilotPath` や記録の置き場。[ADR-20261015-005][adr-settings]）をどう見つけるかが決まっていない（[06-setup-and-stages.md][f06]、[07-structure.md][f07]）~~
- UNDECIDED-38 **欠番**（[ADR-20261017-007][adr-stop] で決めた）：~~**全体を止める合図と、書かずに試す合図の置き場。** 止める合図（`enabled: false`）と書かずに試す合図（`dryRun: true`）は設定ファイルに持つ（「運用の規則」）。
  [ADR-20261015-005][adr-settings] の割り振りでは、どちらもリポの設定になり、変えるのに PR が要る。急いで止めたいときに PR を待つことになり、進行ボットがどの版のリポの設定を読むかも書かれていない（[01-issue-flow.md][f01]、[07-structure.md][f07]）~~
- UNDECIDED-47 **欠番**（[ADR-20261018-001][adr-usage] で決めた）：~~**途中で止めた回の消費の数え方。** IMPL-14 は、`stream-json` の `assistant` のイベントごとの `model` と `usage` を足し上げ、進行ボットが途中で止めた回も、そこまでの消費を記録に残すとする。
  一方、`assistant` のイベントは 1 つのメッセージで同じ `usage` を繰り返して持ち、`output_tokens` はストリームの始めの値なので、そのまま足すと入力を多く、出力を少なく数える（FACT-29）。
  出力を少なく数えると、1 Issue の累計の利用枠が上限（IMPL-14）を超えても `needs-human` にならず、上限が黙って効かなくなる。
  正しい値は最後の `result` の `modelUsage` にあるが、時間の上限で止めた回には最後の `result` が出ない。`--include-partial-messages` の `message_delta` は、親のメッセージの出力の最後の値を持つ（FACT-29）が、途中で止めたときに出るかは確かめていない（UNDECIDED-44）。
  数え方（`message_delta` を使う、最後の `result` が無い回は 1 セッションの費用の上限（`--max-budget-usd`）まで使ったとみなす、など）を決める（[02-implement-review-fix.md][f02]、[10-verified-facts.md][f10]）~~
- UNDECIDED-48 **欠番**（[ADR-20261017-005][adr-mcp] で決めた）：~~**AI のセッションに claude.ai のコネクタを読み込ませないか。** AI の 5 役は GitHub に書けない（SAFE-1）。
  一方、`--setting-sources local` で起動したセッションにも、ログインしている claude.ai のアカウントのコネクタ（メールの送信などの道具を持つもの）が入った回があった（FACT-30）。
  試験では、エージェント定義の `tools` で道具を絞っていたので、コネクタの道具は道具の一覧に無かった。要件と安全設定（SAFE-2 の (e)、SAFE-10）は MCP のコネクタに触れておらず、エージェント定義の道具の絞り込みだけに頼っている。
  AI のセッションでコネクタを読み込ませないか、読み込ませないならどの手段で止めるか（設定の `disableClaudeAiConnectors`、環境変数 `ENABLE_CLAUDEAI_MCP_SERVERS=false`、`--strict-mcp-config`、安全設定の `permissions.deny` に `mcp__*` を入れる、専用マシンの `claude` にコネクタの無いアカウントでログインする、など）を決める。
  手段ごとに、dev-autopilot の起動の形で効くかを先に確かめる（[05-safety.md][f05]、[10-verified-facts.md][f10]）~~
- UNDECIDED-50 **欠番**（[ADR-20261017-006][adr-tmp] で決めた）：~~**sandbox の中の一時フォルダが、セッションの間で共有される。** SAFE-10 と [ADR-20261015-001][adr-read] の決定 1 は、AI のセッションの `TMPDIR` をセッションの一時フォルダに向けるとする。
  一方、Claude Code は sandbox の中のコマンドの `$TMPDIR` をユーザーごとの一時フォルダ（`/tmp/claude-<uid>/`）に置き換え、sandbox の中のコマンドはユーザーごとの一時フォルダに既定で書ける（FACT-34）。
  同じユーザーで動くすべての AI のセッションから書けるので、前のセッションが置いたファイルを次のセッションが読む経路になりうる（objects と依存の store と同じ種類の論点。[ADR-20261013-001][adr-workspace]、[ADR-20261015-002][adr-store]）。
  手元の Mac で試す間（SAFE-18）は、人の Claude Code のセッションの作業用のフォルダも同じ一時フォルダの下にある。
  セッションごとに一時フォルダを分けるか、分けるならどの手段で分けるか（進行ボットがセッションごとに `CLAUDE_CODE_TMPDIR` を渡す、など）を決める。
  `CLAUDE_CODE_TMPDIR` はパスが長いと既定の場所の下の短いパスに置き換わる（FACT-34）ので、dev-autopilot の置き場のパスで効くかを先に確かめる（[05-safety.md][f05]、[10-verified-facts.md][f10]）~~

## 基本設計の中で決めてよいもの

- UNDECIDED-18 レビュー側で `pnpm check` を実行するか（CI の結果を待つだけにするか）
- UNDECIDED-19 **欠番**（基本設計の「[設定][design-settings]」の「マシンの中の置き場」で決めた）：~~実行の記録の置き場（記録と lock の置き場はマシンの設定の項目になった（[ADR-20261015-005][adr-settings]）ので、既定のパスを構造仕様の設定の表で決める。`setup` が作る「記録のフォルダ」の場所。SETUP-1）~~
- UNDECIDED-20 自動化用の GitHub アカウントを分けるか（SETUP-13）。分けるなら machine user と GitHub App のどちらか。レビュー工程（段階 6。レビュー AI と批評 AI を足す段階）で、1 つのアカウントで印の判定が動くこと（GitHub のレビュー状態は使わない。IMPL-18）を実機で確かめてから決める
- UNDECIDED-21 クラウドの仕組みを、利用者のマシンの定期の実行の代わりか補助に使うか。メンテナは将来、GitHub Actions などのクラウドでも動かしたい。
  広げるときは、進行 Bot の本体を変えずに、登録の仕組みと秘密の置き場を足す形にし、別の ADR で決める（[ADR-20261017-003][adr-plugin] の決定 3。STRUCT-8）。2026-10-07 の公式の仕様で分かっている範囲：
  - Routine（`/schedule`）：Anthropic のクラウドで動き、GitHub のリポを clone して作業できる。起動は時刻・GitHub の webhook・API。最短 1 時間間隔。Routine 同士の連鎖は未サポート。利用枠を消費する
  - GitHub Actions の claude-code-action：PR のイベントで起動でき、`/code-review --comment` で行にコメントを書ける。レビュー AI だけをここに寄せる案がある
  - 外部のコード実行 sandbox（E2B など）：AI のセッションを Claude Code の sandbox（「[隔離と鍵の分け方][f05]」）の代わりに、使い捨てのクラウドの sandbox で動かす案。
    鍵と認証情報をクラウドに置くことになり、今は利用者のマシンで動かす形（「[決まっていること][f00]」の配り方と実行環境）とは別の守りが要る。マージ判定工程（段階 8）の出口を越えて専用マシンへの移行を決めるときの候補に留める
  - 判断の軸：専用マシンの保守の手間、利用枠と課金、鍵と認証情報をどこに置くか、進行 Bot の判定を手元とクラウドで二重に持たないこと
- UNDECIDED-22 検疫の見本（指示の文例、区切りを模した文、見えない文字）の置き場。TEXT-11 は `test/` の前提データとし、STRUCT-4 と「[分離を見越した構成][f07]」の構成の図は、前提データを `fixtures/` に置き、`test/` には結合テストだけを置くとしている。要件メモの時点からある食い違い（[04-external-text-and-markers.md][f04]、[07-structure.md][f07]）
- UNDECIDED-27 **欠番**（基本設計の「[状態遷移と判定][design-flow]」の「作業場所の残骸」で決めた）：~~作業場所の残骸の決め方。[ADR-20261013-001][adr-workspace] では、セッションのフォルダ（`sessions/<セッションの名前>/`）は同じ名前で作り直すか片付ける（FLOW-27）まで残るので、作業中の Issue のフォルダがあるのは普通の状態になる。
  どのフォルダを残骸とみなすか（例：片付けの後に残ったもの、名前から Issue か PR を引けないもの）を決め、SETUP-2 の warn・Issue ごとの `needs-human`・fail に当てはめる（FLOW-29、SAFE-4、「運用の規則」）。
  消す前に読んだ `commondir` か `gitdir` が作ったときと違う作業場所は、ADR のとおり消さずに `needs-human` にし、`check` で warn にする~~
- UNDECIDED-39 **欠番**（基本設計の「[状態遷移と判定][design-flow]」の「印の種類」で決めた。`raise` と `duplicate` を足した）：~~進行 Bot のほかのコメントに付ける印の種類。モデルを上げたことを残すコメント（IMPL-25）と、🟢 の重複のときに PR 番号を足すコメント（IMPL-26）に当てはまる `note` の種類が無い（今は `wait` `overlap` `remind` `stop` `link`）。種類を足すか、既存の種類に寄せるかを決める（[ADR-20261015-004][adr-markers]）~~
- UNDECIDED-40 AI のセッションごとの安全設定の作り方。`allowRead`（自分のセッションのフォルダ）と `Write(//<worktree>/**)` はセッションごとに違うパスなのに、SAFE-2 の (e) と SAFE-5 は、`setup` が書く固定のファイルとハッシュの確かめを前提にしている。
  進行 Bot が雛形からセッションごとに作るか、雛形をどこに置きハッシュをどう確かめるか、リポの設定の `denyRead` `allowRead` の一覧にマシンのパスをどう書くか（記号で書いて起動の前に解決するなど。リポの設定には絶対パスを置かない。ADR-20261015-005）を決める
- UNDECIDED-41 **欠番**（[ADR-20261019-001][adr-repeat] で決めた）：~~同じ指摘の再発（修正の後のラウンドで、前のラウンドと同じ指摘がもう一度出ること）の数え方。IMPL-24 は 2 ラウンド続けて出たとき、「[レビューの Skill][f08]」の方針は 2 回出たときに人に渡すとする。間を空けて再び出た指摘も数えるかを決める~~
- UNDECIDED-42 **欠番**（[ADR-20261016-003][adr-move] で、評価を dev-autopilot のリポに置くと決めた）：~~`review` Skill の評価の置き場。「[レビューの Skill][f08]」は評価を business-os の `pnpm eval:cases` の仕組みの上に置くとし、STRUCT-4 は検査を dev-autopilot 自身が持つとする。分離のときに評価も移すかを決める~~
- UNDECIDED-43 `/verify-issue` から dev-autopilot を呼ぶ形。STRUCT-6 は business-os の `/verify-issue` が検疫のモジュールと `dev-autopilot:intake` を呼ぶとし、「[分離を見越した構成][f07]」は business-os が dev-autopilot を import しないとする。呼び出しなら許すのか、どう呼ぶか（Skill を呼ぶ、`plugin/src/` のコマンドを実行する）を決める
- UNDECIDED-46 業務の固有名詞の混入を探す検査。business-os の中にあった間は、business-os の `check:leak`（メールアドレス・電話番号・法人格・金額・許可リストに無いドメインを探す）が dev-autopilot の文書も見ていた。dev-autopilot のリポの gitleaks は秘密の値だけを探すので、固有名詞の混入を探す検査が無い（「[決まっていること][f00]」の置き場所の行）。検査を足すか、足すならどう足すか（gitleaks の独自の規則、TypeScript の検査など）を決める
- UNDECIDED-52 Issue の種類の分け方。最初の実装で使うモデルは Issue の種類ごとに決める（IMPL-25、[ADR-20261018-003][adr-model]）が、要件は Issue をどう種類に分けるか（ブランチの型（IMPL-2）か、別の分け方か）を書いていない。計画・実装工程（段階 5）の記録の欄を決めるときに決める

## 確かめること

- UNDECIDED-23 **欠番**（2026-10-16 に確かめた。済んでいたのは SETUP-10 だけで、「[いまのリポジトリに無いもの][f06]」の表に書き添えた。SETUP-11 は `dev-autopilot/` と構造仕様の表は済み、TypeScript の置き場（`src/` と tsconfig）はまだ無い。ほかの行は導入と点検の実装（段階 3。`setup` と `check` を作る段階）以降で作る）：~~**準備の作業（「[いまのリポジトリに無いもの][f06]」）が今どこまで済んでいるか。** 「[いまのリポジトリに無いもの][f06]」は 2026-10-07 時点の「足りないもの」の一覧で、その後どこまで済んだかを確かめた記録が無い。
  SETUP-11 は、`dev-autopilot/` フォルダと構造仕様（`docs/design/architecture.md`）の表は済んでいるが、TypeScript の置き場（`src/` と tsconfig）はまだ無い（2026-10-12 に確認）。
  SETUP-9（`develop` の ruleset）や SETUP-10（署名の移行）なども 1 つずつ確かめ、済んだものは「[いまのリポジトリに無いもの][f06]」の冒頭の規則に従って、行を残したまま済んだことを書き添える（[06-setup-and-stages.md][f06]）~~
- UNDECIDED-24 **欠番**（[ADR-20261013-001][adr-workspace] で、進行 Bot が AI の触った clone で fetch しない形に決め、確かめる必要が無くなった）：~~**AI が触った clone の中で、進行役が fetch を実行しても安全か。** AI の作業場所の案のうち、Issue ごとの clone を使い続けて新しい commit を fetch で届ける案は、この確認が前提になる。
  段階 2 で確かめたのは、別の clone からの fetch と、親 clone での `worktree list` と `remove` だけ（FACT-12。[10-verified-facts.md][f10]）。段階 3 の点検で確かめる~~
- UNDECIDED-28 **欠番**（FACT-26 で確かめた。エージェント定義の `tools` に書けば使え、作業フォルダの外は Read と同じく拒否される）：~~**計画役の道具の Grep と Glob が使えるか。** 計画役は読むだけの道具（Read・Grep・Glob）で動く前提（「[登場するもの][f00]」）だが、Claude Code 2.1.285 の `-p` のセッションの道具の一覧に Grep と Glob が無かった（FACT-24。[10-verified-facts.md][f10]）。
  エージェント定義の `tools` に書いたときに使えるかを、段階 3 で確かめる。使えなければ、計画役がファイルを探す手段を決める~~
- UNDECIDED-29 **欠番**（FACT-27 と FACT-28 で確かめた。値の `creator` は値を最初に入れた人で、Agent 欄の変更はタイムラインにも出ないため、値を最後に変えた人は API で取れない。Status と Agent 欄を変えた人を確かめない形（[ADR-20261015-004][adr-markers]）のままにする）：~~**Project の欄の値を最後に変えた人を API で取れるか。** GraphQL の `ProjectV2ItemFieldSingleSelectValue` は `creator`（「項目を作った人」）と `updatedAt` しか持たず、値を最後に変えた人を指すかが型の定義から分からない（[ADR-20261015-004][adr-markers]）。
  捨てる Project で、別の人が値を書き換えたときに `creator` が変わるかを確かめる（FACT-15 と同じやり方）。取れるなら、Status と Agent 欄を戻した人の権限も確かめる形にできる~~
- UNDECIDED-44 **欠番**（FACT-38 と FACT-39 で確かめ、止め方を [ADR-20261018-001][adr-usage] で決めた）：~~**作業中の `claude` を途中で止めたときに、孫プロセスと消費の記録がどうなるか。** IMPL-14 は、時間の上限でプロセスグループごと止め、そこまでの消費を `stream-json` から数えるとする。
  段階 2 で確かめたのは、モデルが長いコマンドを実行していない状態でのプロセスグループの停止だけ（FACT-14。[10-verified-facts.md][f10]）。作業中の `claude` を止める試験を、段階 3（`setup` と `check` を作る段階）で行う。
  同じ試験で、止めたときに `--include-partial-messages` の `message_delta` がどこまで出るかも確かめる（消費の数え方は UNDECIDED-47）~~
- UNDECIDED-45 **欠番**（FACT-33 で確かめた。skills CLI は `plugin/skills/` の下の Skill を見つける）：~~**skills CLI が `plugin/skills/` の下の Skill を単体で入れられるか。** STRUCT-7 は、dev-autopilot のリポの `review` Skill を、skills CLI で単体導入できるとする。
  Skill の置き場は `plugin/skills/review/`（配布物を `plugin/` に集めたため）で、skills CLI が `plugin/` の下を見つけるかを確かめていない（[07-structure.md][f07]）~~
- UNDECIDED-49 **Linux で、Claude Code の sandbox が macOS と同じに振る舞うか。** [ADR-20261017-003][adr-plugin] で、今は macOS と Linux に対応すると決めた（STRUCT-8）。
  sandbox の振る舞いを確かめた事実（FACT-9、FACT-13、FACT-22 など）は、macOS で確かめたもの。Linux で、読み取りを自分のセッションのフォルダに絞れること（`denyRead` と `allowRead`。FACT-22）、Read ツールが作業フォルダの外を拒否すること（FACT-13、FACT-22）、
  worktree で `git commit` が通り親 clone の `.git` の保護が効くこと（FACT-9）、SAFE-10 の環境変数で git と pnpm が動くこと（FACT-22）、`CLAUDE_CODE_TMPDIR` で sandbox の中の `$TMPDIR` をセッションごとの短い一時フォルダにでき、長さの上限と戻り方が macOS と同じこと（FACT-37。[ADR-20261017-006][adr-tmp]）、途中で止めたときの AI のコマンドの残り方と、残ったコマンドが sandbox の中のままであること（FACT-38。[ADR-20261018-001][adr-usage]）を確かめ直す。確かめるまで、Linux で進行 Bot を動かさない（[07-structure.md][f07]、[05-safety.md][f05]）

[f00]: ./00-overview.md
[f01]: ./01-issue-flow.md
[f02]: ./02-implement-review-fix.md
[f03]: ./03-merge.md
[f04]: ./04-external-text-and-markers.md
[f05]: ./05-safety.md
[f06]: ./06-setup-and-stages.md
[f07]: ./07-structure.md
[f10]: ./10-verified-facts.md
[adr-workspace]: ../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[adr-store]: ../adr/adr-20261015-002-no-shared-dependency-store.md
[adr-intake]: ../adr/adr-20261015-003-intake-scope-and-quarantine-report.md
[adr-markers]: ../adr/adr-20261015-004-signed-markers-and-actor-checks.md
[adr-settings]: ../adr/adr-20261015-005-settings-values-and-machine-settings.md
[adr-flow]: ../adr/adr-20261015-006-run-stages-baseline-agent-field-and-local-records.md
[adr-pr]: ../adr/adr-20261016-001-ci-states-merge-ready-limit-and-pr-issue-link.md
[adr-draft]: ../adr/adr-20261016-002-intake-ai-drafts-every-issue.md
[f08]: ./08-review-skill.md
[adr-move]: ../adr/adr-20261016-003-move-to-own-repository-before-basic-design.md
[adr-review]: ../adr/adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-plugin]: ../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-human]: ../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-mcp]: ../adr/adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[adr-tmp]: ../adr/adr-20261017-006-per-session-short-tmpdir.md
[adr-stop]: ../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-usage]: ../adr/adr-20261018-001-stop-session-and-count-usage.md
[adr-redo]: ../adr/adr-20261018-002-plan-redo-in-same-run.md
[adr-model]: ../adr/adr-20261018-003-count-model-raise-at-stage-7.md
[adr-planrev]: ../adr/adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[design-settings]: ../design/basic/01-settings.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
[adr-repeat]: ../adr/adr-20261019-001-repeated-findings-by-ids.md
