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

### 流れと段

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
  1 セッションに収まらない大きさの作業は、3 回とも止まって `needs-human` になる（IMPL-14）。引き継ぐ案として、止まったセッションでも AI が commit した分だけを進行役の clone へ取り込み、IMPL-15 の検査を通して作業ブランチへ push し、次のセッションを取り込んだ commit から始める形がある。
  IMPL-13 を変え、未完成の作業を GitHub に push することになる。段階 5 で上限に当たった回数を記録してから決める（[02-implement-review-fix.md][f02]）
- UNDECIDED-30 **計画のやり直しを同じ実行で行うか。** 計画役の `opus` が自己申告したら、進行役は 1 回だけ `fable` で計画し直す（IMPL-10）。同じ実行で行うと、重い段の計画で AI のセッションが 2 つになる。
  次の実行に回すなら、自己申告があったことと 1 回上げたことを GitHub に残す仕組み（`plan` の印の欄など）が要る。手元の記録は判定に使わない（[ADR-20261015-006][adr-flow]。[02-implement-review-fix.md][f02]）
- UNDECIDED-31 **作業 AI のモデルを上げた件数を、どの段階で数えるか。** IMPL-25 と段階の表は、段階 5（計画と実装を動かし、レビューは人が行う段階）で「`sonnet` で始めて `opus` で通った件数」を数えるとする。
  モデルを上げるのは修正のセッション（作業 AI の 2 ラウンド目以降）で、修正のセッションは段階 7（対応ループを足す段階）で足すので、段階 5 には上げる場面が無い（[02-implement-review-fix.md][f02]、[06-setup-and-stages.md][f06]）

### レビューとマージ

- UNDECIDED-9 **欠番**（[ADR-20261016-001][adr-pr] で決めた）：~~**pass なのに CI が緑でないとき。** pass なのに CI が赤・無い・保留のときの扱い。赤を修正に回して往復に数えるか、無い・保留は待つか（IMPL-27。[02-implement-review-fix.md][f02]）~~
- UNDECIDED-10 **欠番**（[ADR-20261016-001][adr-pr] で決めた）：~~**2 件目以降の `merge=ready`。** 1 回の実行で `merge=ready` 1 件を超えた PR の扱い（`human` にすると一致の記録が汚れる）（MERGE-5。[03-merge.md][f03]）~~
- UNDECIDED-32 **レビュー AI のサブエージェントと Agent ツール。** REVIEW-1 は 3 つの軸を並列のサブエージェントで見るとするが、「[登場するもの][f00]」のレビュー AI の道具に Agent が無い。
  [ADR-20261015-005][adr-settings] は、Agent ツールを許すことを、安全の境界を緩める変更として ADR で決めるとしている。設定の「Agent ツールを許すか」（IMPL-14、STRUCT-2）が役ごとかも決まっていない（[08-review-skill.md][f08]）
- UNDECIDED-33 **コラボレータの PR のコメントを AI に渡すか。** TEXT-1 と REVIEW-6 は、コラボレータのコメントは渡す、PR のコメントも同じ、とする。一方、渡す文の範囲は `agent-ready` の時点で確定し、後から足されたコメントは渡さない（TEXT-1）。
  再レビューの入力（IMPL-21）と修正の入力（IMPL-23）にも、コラボレータのコメントは無い。PR のコメントはすべて `agent-ready` の後に付くので、渡すのか、渡すならどの役のどの入力かが決まらない（[04-external-text-and-markers.md][f04]、[08-review-skill.md][f08]）
- UNDECIDED-34 **レビュー AI に渡す規約の正典を、base から取るか。** 「AI が読む文」の表は「規約の正典は base ブランチから渡す」とするが、レビュー AI の作業場所は PR の head で作る（IMPL-16）。
  作業 AI は head の `docs/adr/**` などを書き換えうる（IMPL-15 の push を止めるパスに入っていない。マージの判定では `merge=human`）。レビュー AI が書き換えられた規約を読まないようにするか、どう渡すかが決まっていない（[04-external-text-and-markers.md][f04]、[02-implement-review-fix.md][f02]）

### 受け入れと印

- UNDECIDED-11 **欠番**（[ADR-20261015-004][adr-markers] で決めた）：~~**印の体系の拡張。** 進行役の投稿すべてに印を付けるか（待ち・催促・警告・停止の理由）、署名の対象に種類・リポ・番号を含めるか、進行役の印が無効化されていたら `needs-human` にするか（「[印と固定の見出しの一覧][f04]」。[04-external-text-and-markers.md][f04]）~~
- UNDECIDED-12 **欠番**（[ADR-20261016-001][adr-pr] で決めた）：~~**PR と Issue の結びつけ。** `Closes #n` でなく head のブランチ名・同じリポ・作者・印の番号で引くか（IMPL-6。[02-implement-review-fix.md][f02]）~~
- UNDECIDED-13 **欠番**（[ADR-20261015-004][adr-markers] で決めた）：~~**権限確認の範囲。** `needs-human` を外す・Status を戻す・`plan:*` を付ける操作の人と、渡す文の編集者。渡す文の最終編集が `agent-ready` より後なら `needs-human` にするか（TEXT-10、「[Issue の状態の遷移][f01]」の戻す手順。[04-external-text-and-markers.md][f04]、[01-issue-flow.md][f01]）~~
- UNDECIDED-14 **欠番**（[ADR-20261015-003][adr-intake] で決めた）：~~**検疫で見つかったときの印。** TEXT-9 は「見つかったら下書きを作らない」と書いた直後に「下書きコメントに『混入の検査：有り（種類）』の行を必ず書く」としている。
  TEXT-12 も「有りなら下書きを作らない」。一方、「[印と固定の見出しの一覧][f04]」の `intake` の印は `quarantine=found` を持ち、下書きを作らないなら印を載せる場所が無い（[04-external-text-and-markers.md][f04]）~~
- UNDECIDED-15 **欠番**（[ADR-20261015-003][adr-intake] で決めた）：~~**受け入れの対象とマイルストーンの候補。** TEXT-12 は受け入れの対象を「マイルストーンが付いている Issue」に限りながら、マイルストーンの候補を出すことになっている。
  「[決まっていること][f00]」の「人の関所」と FLOW-15〜FLOW-16 は、起票されたら受け入れる流れで、マイルストーンを条件にしていない（[04-external-text-and-markers.md][f04]、[00-overview.md][f00]、[01-issue-flow.md][f01]）~~
- UNDECIDED-35 **進行役が起票した Issue の本文の見分け方。** 🟢 から起票した Issue（IMPL-26）の本文は AI の出力として扱う（TEXT-1）が、印の一覧に、Issue の本文に付ける印が無い。
  進行役は当面メンテナと同じアカウントで投稿する（SETUP-13）ので、著者のアカウントでは見分けられず、本文がコラボレータの文（`body=pass`）として作業 AI に届きうる。
  [ADR-20261015-004][adr-markers] は、印の無い進行役のアカウントの投稿を人の投稿として読むとしている（[04-external-text-and-markers.md][f04]、[02-implement-review-fix.md][f02]）

### 安全と設定

- UNDECIDED-16 **欠番**（[ADR-20261015-001][adr-read] で決めた）：~~**AI の読み取り範囲と秘密の置き場。** Read の許可を worktree に絞りホーム全体を `denyRead` にするか。PAT・鍵・deploy key の置き場。手動実行を `launchctl kickstart` にするか（SAFE-2 の (e)。[05-safety.md][f05]）~~
- UNDECIDED-17 **欠番**（[ADR-20261015-005][adr-settings] で決めた）：~~**まだ値が決まっていない設定。** 実行時間、受け入れの件数、費用の上限、再試行の間隔、Agent ツール、確信度のしきい値、置き場の値をどこで決めるか（設定の項目の一覧は STRUCT-2。[07-structure.md][f07]）~~
- UNDECIDED-25 **欠番**（[ADR-20261015-002][adr-store] で決めた）：~~**依存の store をセッションの間で共有するか。** AI のセッションは作業場所の中で依存を導入する（IMPL-1）。pnpm の store をセッションの間で共有すると、前のセッションが細工したものを次のセッションが読む経路になりうる（objects と同じ種類の論点。[ADR-20261013-001][adr-workspace]）。
  手元の Mac（APFS）では store と `node_modules` のファイルを共有しなかったが、専用マシンのファイルシステムでは確かめていない。セッションごとに空の store を使うと、`pnpm install` は 1.1 秒から 3.0 秒になった（FACT-21。[10-verified-facts.md][f10]）~~
- UNDECIDED-36 **AI のセッションが Plugin を読めるか。** Plugin は、進行役の clone（`<置き場>/controller/`）の中を `--plugin-dir` で指す（STRUCT-3）。一方、[ADR-20261015-001][adr-read] で、AI のセッションは置き場を読めなくした。
  エージェント定義の読み込みは確かめた（FACT-2、FACT-26）が、レビュー AI が Skill の参照ファイルを Read で読むことと、`review/scripts/` を sandbox の中の Bash で実行することは、拒否されるはず（確かめていない）。Plugin の置き場か、読み取りの許可を決める（[07-structure.md][f07]、[05-safety.md][f05]）
- UNDECIDED-37 **人が呼ぶ `setup` と `check` の認証と、マシンの設定の見つけ方。** `setup` と `check` は人の Claude Code のセッションから動き、GitHub への書き込み（ラベル・Project の欄・ruleset）と、PAT の権限の確かめを行う。
  ADR-20261015-001 は人のセッションに秘密を扱わせないとし、専用マシンには人の `gh` のログインを置かない（「[決まっていること][f00]」）。
  どの認証で動くかと、人が呼ぶ Skill がマシンの設定（`devAutopilotPath` や記録の置き場。[ADR-20261015-005][adr-settings]）をどう見つけるかが決まっていない（[06-setup-and-stages.md][f06]、[07-structure.md][f07]）
- UNDECIDED-38 **全体を止める合図と、書かずに試す合図の置き場。** 止める合図（`enabled: false`）と書かずに試す合図（`dryRun: true`）は設定ファイルに持つ（「運用の規則」）。
  [ADR-20261015-005][adr-settings] の割り振りでは、どちらもリポの設定になり、変えるのに PR が要る。急いで止めたいときに PR を待つことになり、進行役がどの版のリポの設定を読むかも書かれていない（[01-issue-flow.md][f01]、[07-structure.md][f07]）

## 基本設計の中で決めてよいもの

- UNDECIDED-18 レビュー側で `pnpm check` を実行するか（CI の結果を待つだけにするか）
- UNDECIDED-19 実行の記録の置き場（記録と lock の置き場はマシンの設定の項目になった（[ADR-20261015-005][adr-settings]）ので、既定のパスを構造仕様の設定の表で決める。`setup` が作る「記録のフォルダ」の場所。SETUP-1）
- UNDECIDED-20 自動化用の GitHub アカウントを分けるか（SETUP-13）。分けるなら machine user と GitHub App のどちらか。段階 6（レビュー AI と批評者を足す段階）で、1 つのアカウントで印の判定が動くこと（GitHub のレビュー状態は使わない。IMPL-18）を実機で確かめてから決める
- UNDECIDED-21 クラウドの仕組みを、手元の cron の代わりか補助に使うか。2026-10-07 の公式の仕様で分かっている範囲：
  - Routine（`/schedule`）：Anthropic のクラウドで動き、GitHub のリポを clone して作業できる。起動は時刻・GitHub の webhook・API。最短 1 時間間隔。Routine 同士の連鎖は未サポート。利用枠を消費する
  - GitHub Actions の claude-code-action：PR のイベントで起動でき、`/code-review --comment` で行にコメントを書ける。レビュー AI だけをここに寄せる案がある
  - 外部のコード実行 sandbox（E2B など）：AI のセッションを Claude Code の sandbox（「[隔離と鍵の分け方][f05]」）の代わりに、使い捨てのクラウドの sandbox で動かす案。
    鍵と認証情報をクラウドに置くことになり、「手元の cron で始め、クラウドの仕組みを土台にしない」（「[決まっていること][f00]」）と衝突する。段階 8 の出口を越えて専用マシンへの移行を決めるときの候補に留める
  - 判断の軸：専用マシンの保守の手間、利用枠と課金、鍵と認証情報をどこに置くか、進行役の判定を手元とクラウドで二重に持たないこと
- UNDECIDED-22 検疫の見本（指示の文例、区切りを模した文、見えない文字）の置き場。TEXT-11 は `dev-autopilot/test/` の前提データとし、STRUCT-4 と「[分離を見越した構成][f07]」の構成の図は、前提データを `fixtures/` に置き、`test/` には結合テストだけを置くとしている。要件メモの時点からある食い違い（[04-external-text-and-markers.md][f04]、[07-structure.md][f07]）
- UNDECIDED-27 作業場所の残骸の決め方。[ADR-20261013-001][adr-workspace] では、セッションのフォルダ（`sessions/<セッションの名前>/`）は同じ名前で作り直すか片付ける（FLOW-27）まで残るので、作業中の Issue のフォルダがあるのは普通の状態になる。
  どのフォルダを残骸とみなすか（例：片付けの後に残ったもの、名前から Issue か PR を引けないもの）を決め、SETUP-2 の warn・Issue ごとの `needs-human`・fail に当てはめる（FLOW-29、SAFE-4、「運用の規則」）。
  消す前に読んだ `commondir` か `gitdir` が作ったときと違う作業場所は、ADR のとおり消さずに `needs-human` にし、`check` で warn にする
- UNDECIDED-39 進行役のほかのコメントに付ける印の種類。モデルを上げたことを残すコメント（IMPL-25）と、🟢 の重複のときに PR 番号を足すコメント（IMPL-26）に当てはまる `note` の種類が無い（今は `wait` `overlap` `remind` `stop` `link`）。種類を足すか、既存の種類に寄せるかを決める（[ADR-20261015-004][adr-markers]）
- UNDECIDED-40 AI のセッションごとの安全設定の作り方。`allowRead`（自分のセッションのフォルダ）と `Write(//<worktree>/**)` はセッションごとに違うパスなのに、SAFE-2 の (e) と SAFE-5 は、`setup` が書く固定のファイルとハッシュの確かめを前提にしている。
  進行役が雛形からセッションごとに作るか、雛形をどこに置きハッシュをどう確かめるか、リポの設定の `denyRead` `allowRead` の一覧にマシンのパスをどう書くか（記号で書いて起動の前に解決するなど。リポの設定には絶対パスを置かない。ADR-20261015-005）を決める
- UNDECIDED-41 振動の数え方。IMPL-24 は「同じ指摘が 2 ラウンド続けて出たら振動」、「[レビューの Skill][f08]」の方針は「同じ指摘が 2 回出たら振動」とする。間を空けて再び出た指摘を振動に数えるかを決める
- UNDECIDED-42 `review` Skill の評価の置き場。「[レビューの Skill][f08]」は評価を business-os の `pnpm eval:cases` の仕組みの上に置くとし、STRUCT-4 は検査を dev-autopilot 自身が持つとする。分離のときに評価も移すかを決める
- UNDECIDED-43 `/verify-issue` から dev-autopilot を呼ぶ形。STRUCT-6 は business-os の `/verify-issue` が検疫のモジュールと `dev-autopilot:intake` を呼ぶとし、「[分離を見越した構成][f07]」は business-os が dev-autopilot を import しないとする。呼び出しなら許すのか、どう呼ぶか（Skill を呼ぶ、`src/` のコマンドを実行する）を決める

## 確かめること

- UNDECIDED-23 **欠番**（2026-10-16 に確かめた。済んでいたのは SETUP-10 だけで、「[いまのリポジトリに無いもの][f06]」の表に書き添えた。SETUP-11 は `dev-autopilot/` と構造仕様の表は済み、TypeScript の置き場（`src/` と tsconfig）はまだ無い。ほかの行は段階 3（`setup` と `check` を作る段階）以降で作る）：~~**準備の作業（「[いまのリポジトリに無いもの][f06]」）が今どこまで済んでいるか。** 「[いまのリポジトリに無いもの][f06]」は 2026-10-07 時点の「足りないもの」の一覧で、その後どこまで済んだかを確かめた記録が無い。
  SETUP-11 は、`dev-autopilot/` フォルダと構造仕様（`docs/design/architecture.md`）の表は済んでいるが、TypeScript の置き場（`src/` と tsconfig）はまだ無い（2026-10-12 に確認）。
  SETUP-9（`develop` の ruleset）や SETUP-10（署名の移行）なども 1 つずつ確かめ、済んだものは「[いまのリポジトリに無いもの][f06]」の冒頭の規則に従って、行を残したまま済んだことを書き添える（[06-setup-and-stages.md][f06]）~~
- UNDECIDED-24 **欠番**（[ADR-20261013-001][adr-workspace] で、進行役が AI の触った clone で fetch しない形に決め、確かめる必要が無くなった）：~~**AI が触った clone の中で、進行役が fetch を実行しても安全か。** AI の作業場所の案のうち、Issue ごとの clone を使い続けて新しい commit を fetch で届ける案は、この確認が前提になる。
  段階 2 で確かめたのは、別の clone からの fetch と、親 clone での `worktree list` と `remove` だけ（FACT-12。[10-verified-facts.md][f10]）。段階 3 の点検で確かめる~~
- UNDECIDED-28 **欠番**（FACT-26 で確かめた。エージェント定義の `tools` に書けば使え、作業フォルダの外は Read と同じく拒否される）：~~**計画役の道具の Grep と Glob が使えるか。** 計画役は読むだけの道具（Read・Grep・Glob）で動く前提（「[登場するもの][f00]」）だが、Claude Code 2.1.285 の `-p` のセッションの道具の一覧に Grep と Glob が無かった（FACT-24。[10-verified-facts.md][f10]）。
  エージェント定義の `tools` に書いたときに使えるかを、段階 3 で確かめる。使えなければ、計画役がファイルを探す手段を決める~~
- UNDECIDED-29 **Project の欄の値を最後に変えた人を API で取れるか。** GraphQL の `ProjectV2ItemFieldSingleSelectValue` は `creator`（「項目を作った人」）と `updatedAt` しか持たず、値を最後に変えた人を指すかが型の定義から分からない（[ADR-20261015-004][adr-markers]）。
  捨てる Project で、別の人が値を書き換えたときに `creator` が変わるかを確かめる（FACT-15 と同じやり方）。取れるなら、Status と Agent 欄を戻した人の権限も確かめる形にできる
- UNDECIDED-44 **作業中の `claude` を途中で止めたときに、孫プロセスと消費の記録がどうなるか。** IMPL-14 は、時間の上限でプロセスグループごと止め、そこまでの消費を `stream-json` から数えるとする。
  段階 2 で確かめたのは、モデルが長いコマンドを実行していない状態でのプロセスグループの停止だけ（FACT-14。[10-verified-facts.md][f10]）。作業中の `claude` を止める試験を、段階 3（`setup` と `check` を作る段階）で行う
- UNDECIDED-45 **skills CLI が `plugin/skills/` の下の Skill を単体で入れられるか。** STRUCT-7 は、分離した dev-autopilot のリポの `review` Skill を、skills CLI で単体導入できるとする。
  Skill の置き場は `plugin/skills/review/`（配布物を `plugin/` に集めたため）で、skills CLI が `plugin/` の下を見つけるかを確かめていない（[07-structure.md][f07]）

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
[f08]: ./08-review-skill.md
