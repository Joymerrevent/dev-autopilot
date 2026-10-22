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
- UNDECIDED-21 **クラウドの仕組みを、利用者のマシンの定期の実行の代わりか補助に使うか。** メンテナは将来、GitHub Actions などのクラウドでも動かしたい（2026-10-20 に「基本設計の中で決めてよいもの」から移した。[ADR-20261017-003][adr-plugin] の決定 3 が、広げるときは別の ADR で決めるとしていて、基本設計の中では決められないため）。
  広げるときは、進行 Bot の本体を変えずに、登録の仕組みと秘密の置き場を足す形にし、別の ADR で決める（[ADR-20261017-003][adr-plugin] の決定 3。STRUCT-8）。2026-10-07 の公式の仕様で分かっている範囲：
  - Routine（`/schedule`）：Anthropic のクラウドで動き、GitHub のリポを clone して作業できる。起動は時刻・GitHub の webhook・API。最短 1 時間間隔。Routine 同士の連鎖は未サポート。利用枠を消費する
  - GitHub Actions の claude-code-action：PR のイベントで起動でき、`/code-review --comment` で行にコメントを書ける。レビュー AI だけをここに寄せる案がある
  - 外部のコード実行 sandbox（E2B など）：AI のセッションを Claude Code の sandbox（「[隔離と鍵の分け方][f05]」）の代わりに、使い捨てのクラウドの sandbox で動かす案。
    鍵と認証情報をクラウドに置くことになり、今は利用者のマシンで動かす形（「[決まっていること][f00]」の配り方と実行環境）とは別の守りが要る。マージ判定工程（段階 8）の出口を越えて専用マシンへの移行を決めるときの候補に留める
  - 判断の軸：専用マシンの保守の手間、利用枠と課金、鍵と認証情報をどこに置くか、進行 Bot の判定を手元とクラウドで二重に持たないこと
- UNDECIDED-60 **欠番**（[ADR-20261021-002][adr-dry] で決めた。書かずに試すのは受け入れ工程（段階 4）だけ。書かずに試す間は、書くつもりだった受け入れの下書きを同じ文の間は書いたものとして扱い、選定の結果は人の承認の条件を除いて作り、試した回の費用と失敗は書き始めた後の上限に数えない）：~~**書かずに試す間（`dryRun`）に、何を確かめ、どう数えるか。** 「[運用の規則][f01]」は、`dryRun` の間は GitHub に書かず、受け入れ工程（段階 4）の最初と、新しい段階に入るときに使うとする。段階の表は、受け入れ工程（段階 4）の最初に、下書きと選定の結果を手元の記録にだけ書いて人が読むとする。
  一方、`dryRun` の間は印を書かないので、判定に手元の記録を使わない決まり（[ADR-20261015-006][adr-flow] の決定 4）のままだと、次のことが起きる（基本設計の再レビューで見つけた。2026-10-20）。
  受け入れの印も `intake-final` も書かないので、選定（候補の条件を満たすか）まで進まず、選定の結果が出ない。毎回の実行で、Issue の番号の小さい `limits.intakePerRun` 件を受け入れ直し、ほかの Issue には下書きが出ない。受け入れ直した回の費用が Issue の累計に入り、書き始めた後に、一度も投稿していない Issue が 1 Issue の累計の上限（`usage-limit`）で止まりうる。人が手元の記録の下書きを読んで `agent-ready` を付けると、受け入れの印が無いので、書き始めた最初の実行で `approved-before-intake` で止まる。計画・実装工程（段階 5）より後の段階で `dryRun` にすると、計画の印を書かないので、毎回同じ Issue の計画をやり直し、実装に届かない。
  決めること：`dryRun` の間の受け入れの相手の選び方、選定の結果の作り方、書かずに試した回の費用を Issue の累計に数えるか、新しい段階に入るときに `dryRun` を使うか（印を書いて記録だけに残す形にするか）。受け入れ工程（段階 4）で `dryRun` を使う前に決める（[01-issue-flow.md][f01]、[06-setup-and-stages.md][f06]）~~

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
- UNDECIDED-59 **欠番**（[ADR-20261021-001][adr-review-skill] で決めた。どの役にも `Skill` の道具を持たせず、進行 Bot が `review` Skill の本文に引数を入れて渡す。安全設定の `permissions.deny` に `Skill` を入れる）：~~**レビュー AI に `Skill` の道具を持たせるか。** IMPL-16 は、PR のレビューの 3 つの軸のセッションで、`review` Skill を手順として使い、見る軸を Skill の引数で渡すとする（REVIEW-5）。一方、「[登場するもの][f00]」と基本設計の「[AI のセッション][design-ai]」は、レビュー AI（`change/reviewer`）の道具を Bash と Read にしている。エージェント定義の `tools` に `Skill` が無いと、Skill を呼べない（FACT-55）。
  `tools` に `Skill` を足すと、AI の道具を広げる変更になり、Claude Code に同梱の Skill（`update-config`・`schedule` など）も呼べるようになる（FACT-55）。
  候補：(A) `change/reviewer` の `tools` に `Skill` を足し、`permissions` で `review` のほかの Skill を拒む（拒めるかを先に確かめる）。(B) エージェント定義の `skills:` で `review` Skill を先に読み込ませ、`Skill` の道具は足さない（効くかを先に確かめる）。(C) 進行 Bot が `review` Skill の本文を固定の指示文に写して渡し、Skill の引数の入口（REVIEW-5）は使わない。レビュー工程（段階 6）に入る前に決める（[02-implement-review-fix.md][f02]、[08-review-skill.md][f08]、[00-overview.md][f00]）~~
- UNDECIDED-66 **マージ判定工程（段階 8）以降の計画の工程で、上限まで計画し直しても `fix` の計画を止めるか。計画コメントに計画のレビューの結果を書くか。** IMPL-10 と段階の表は、マージ判定工程（段階 8）の前は、上限まで計画し直しても `fix` なら最後の計画を投稿し、レビューの結果も計画し直したことも Issue に書かないとする。
  マージ判定工程（段階 8）以降に `fix` の計画を止めるか（`needs-human` にするか）は、修正工程（段階 7）を終えるときの比べの結果を見て、マージ判定工程（段階 8）に進む前に別の ADR で決めるとしている（[ADR-20261018-004][adr-planrev]、[ADR-20261019-003][adr-replan]）。マージ判定工程（段階 8）以降に計画コメントにレビューの結果を書くかは、先送りの項目に名指しされておらず、決める場所が無い。
  どちらも、決めるまでマージ判定工程（段階 8）に進まないので、今の流れは止まらない。マージ判定工程（段階 8）に進む前に、2 つを ADR で決める（[02-implement-review-fix.md][f02]、[06-setup-and-stages.md][f06]）
- UNDECIDED-67 **欠番**（[ADR-20261021-007][adr-compare] で決めた。比べるためのレビューは、本番のラウンドの後の実行に別の重い工程として起こし、済んだかを手元の記録で見分ける。比べのセッションの失敗は実行の失敗に数えず、費用は Issue の累計に数えない）：~~**比べるためのレビューを、どの実行で起こし、済んだかを何で見分けるか。** レビュー工程（段階 6）では、レビュー AI を `opus` で回した同じ PR を `fable` でも回して比べ、比べのためのレビューは印を書かない（IMPL-19、段階の表）。基本設計の「[AI のセッション][design-ai]」は、本番のラウンドの `review` の印を書いた後の次の実行に、別の重い工程として 3 つの軸を同じ形で起こし、結果を手元の記録にだけ残すとする。
  一方、基本設計の「[状態遷移と判定][design-flow]」の作業中の次の工程の表には、比べるためのレビューに進む行が無い。印を書かないので、済んだかは手元の記録でしか分からず、判定に手元の記録を使わない決まり（[ADR-20261015-006][adr-flow] の決定 4）の例外は、書かずに試す間の受け入れだけ（[ADR-20261021-002][adr-dry]）。記録から読めば決まりに反し、読まなければ毎回の実行で起こし直す。比べのセッションの失敗を実行の失敗に数えるか（数えると、印を書かない比べの失敗で本番の Issue が `retry-limit` で止まる）、Issue の累計に数えるか（数えると `usage-limit` で本番の流れが止まりうる）も決まっていない。
  候補：(a) 本番のラウンドと同じ重い工程の中で続けて起こす（済んだかの見分けが要らないが、1 ラウンドのセッションが 5 から 8 になり、レビュー工程（段階 6）の 1 回の実行の上限と費用の上限の値も直す）。(b) 済んだことを署名付きの `note` の印で PR に書き、次の工程の表に行を足す（IMPL-19 の「印を書かない」に反し、印の一覧も直す）。(c) 済んだかを手元の記録で見ることを、書かずに試す間の受け入れと同じく [ADR-20261015-006][adr-flow] の決定 4 の例外とし、比べのセッションの失敗は実行の失敗に数えず、Issue の累計にも数えない（記録を失ったら比べを回し直すだけで、本番の流れに影響しない）。レビュー工程（段階 6）に進む前に決める（[02-implement-review-fix.md][f02]、[06-setup-and-stages.md][f06]）~~
- UNDECIDED-70 **欠番**（[ADR-20261022-002][adr-critic-skip] で決めた。3 つの軸がすべて IMPL-18 の 5 つの条件を満たし、🔴 🟡 が合わせて 0 件のラウンドは、批評 AI のセッションを起こさず、5 つの条件はラウンドで起こしたセッションに掛ける。見落としの軸だけが 🔴 🟡 を出したラウンドも、批評 AI を起こさない。IMPL-16 と「[登場するもの][f00]」の批評 AI の行をそろえた）：~~**3 つの軸の 🔴 🟡 が 0 件のラウンドで、批評 AI のセッションを起こすか。** IMPL-18 と基本設計の「[AI のセッション][design-ai]」の「レビューの軸の起こし方」は、3 つの軸の 🔴 🟡 が合わせて 0 件のラウンドでは、批評 AI に渡す指摘が無いので批評 AI のセッションを起こさず、IMPL-18 の 5 つの条件を 3 つの軸のセッションだけに掛けるとする（基本設計の 2 回目の再レビューの直しで足した。確かめる指摘が無いラウンドで、1 セッション分の利用枠と失敗の機会を使わないため）。
  一方、承認済みの [ADR-20261017-002][adr-review] の決定 1（1 ラウンドのレビューは、3 つの軸のセッションと、3 つの軸の後に起こす批評 AI のセッションで行う）と確認方法（5 つの条件をすべてのセッションに掛ける）と、[ADR-20261019-002][adr-gaps] の決定 1（見落としの軸が終わってから、批評 AI のセッションを起こす）は、0 件のラウンドの例外を書いていない。IMPL-16 と「[登場するもの][f00]」の批評 AI の行も、例外を書かない。ADR の決定 1 を読んで作る人は、0 件のラウンドでも批評 AI を起こす（基本設計の 3 回目の再レビューで見つけた。2026-10-22）。
  決めること：0 件のラウンドで批評 AI を起こすか。5 つの条件を掛けるセッションの範囲。[ADR-20261017-002][adr-review] の決定 1 と確認方法、[ADR-20261019-002][adr-gaps] の決定 1 のどこを改めるか。見落としの軸を起こすラウンドで、3 つの軸が 0 件で見落としの軸だけが指摘を出したときに、どう扱うか（見落としの軸の結果は、記録だけの間は判定に使わず、批評 AI に渡さない。IMPL-16）。
  IMPL-18 と基本設計の文は、起こさない形に先に直してある。ADR で決めるまで今の文のまま残し、決まったら IMPL-16 と「[登場するもの][f00]」の批評 AI の行をそろえる。レビュー工程（段階 6）でレビュー AI と批評 AI を動かす前に決める（[02-implement-review-fix.md][f02]、[00-overview.md][f00]）~~

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
- UNDECIDED-53 **欠番**（[ADR-20261019-004][adr-policy] で決めた。受付 AI が判定して下書きに書き添え、計画が push しないパスに触れるなら計画を投稿せずに人に渡す）：~~**方針に係る要望の Issue を、受け入れで見分けるか。** 当てる先のリポの方針の文書（規約の正典。business-os なら `docs/adr/**` `docs/design/**`。IMPL-15）を変えてほしいという Issue でも、受け入れでは、受付 AI がほかの Issue と同じく作業指示の下書きと計画の難しさの判定を返すだけで、方針に係る要望かは見分けない（TEXT-12）。
  人が下書きを読んで `agent-ready` を付けなければ処理対象にならないので、今は人が見分けている。人が気づかずに `agent-ready` を付けると、計画と実装に進み、作業 AI が ADR の起票が要ると分かったとき（IMPL-8）か、正典に触れる diff を push する前（IMPL-15）に初めて止まり、それまでのセッションの利用枠を使う。
  受付 AI が方針に係る要望かを判定して出力契約で返し、下書きに書き添えるか、進行 Bot が計画の前に止めるか。Issue の種類の分け方（UNDECIDED-52）と合わせて決めるか。
  AI に方針の文書の下書きまで書かせるか（正典のパスを push しない決まりを緩めるので、安全の境界を変える）は、この論点に含めない（[04-external-text-and-markers.md][f04]、[02-implement-review-fix.md][f02]）~~
- UNDECIDED-61 **欠番**（[ADR-20261021-003][adr-reapprove] で決めた。最新の `intake-final` の印より後に push 以上の人が付け直した `agent-ready` を承認のやり直しとして扱う。push 未満の人の作業指示コメントは読まない。push 以上の人の作業指示コメントが 2 つ以上なら止める）：~~**承認した後に渡す文を直したいとき、承認をやり直す手順をどうするか。外の人の `## 作業指示` のコメントをどう扱うか。** TEXT-4 は、承認した時点の渡す文のハッシュを `intake-final` の印に書き、着手時と対応ループの各回で変わっていたら `needs-human` にする。人が文を直して `needs-human` を外しても、`intake-final` の印は残るので、同じハッシュの照らしで止まり続ける。承認の確定は `intake-final` の印が無いときだけ行うので、承認をやり直す手順が無い（「[止まった状態から戻す手順][f01]」にも無い）。計画コメントが渡す直前の検疫（TEXT-11）で見つかったときも、人は計画コメントを直せない（未編集の条件。IMPL-20）。
  また、TEXT-10 は、`## 作業指示` のコメントを書いた人が push 未満なら `needs-human` にするとし、「[印と固定の見出しの一覧][f04]」は、固定の見出しは push 以上の人が書いたものだけを読むとする。公開のリポでは、外の人が `## 作業指示` のコメントを書くだけで、どの Issue の承認も止められる。作業指示のコメントが 2 つ以上あるときに、どれを渡すかも決まっていない。
  候補の例：`intake-final` の印より後に push 以上の人が付け直した `agent-ready` を、承認のやり直しとして扱い、新しい `intake-final` の印を書く。push 未満の人の `## 作業指示` は読まない（止めない）。作業指示のコメントが 2 つ以上なら、止めて人に選ばせる。受け入れ工程（段階 4）で GitHub に書き始める前に決める（[04-external-text-and-markers.md][f04]、[01-issue-flow.md][f01]）~~
- UNDECIDED-63 **欠番**（[ADR-20261021-004][adr-marker-keys] で決めた。印の署名を確かめる鍵を一覧にし、先頭の鍵で書き、進行中の Issue があっても替えられる。人が `needs-human` を外す前に、進行 Bot が止めた理由の印で知らせた無効な印は、外した後は数えない）：~~**無効な印で止めた Issue をどう戻すか。印の署名の鍵をどう替えるか。** IMPL-20 は、同じ Issue か PR の中に、進行 Bot のアカウントが投稿した印のうち未編集か署名の条件を満たさないものが 1 つでもあれば、`needs-human`（`marker-invalid`）にする。人が編集した進行 Bot のコメントは編集の記録（`lastEditedAt`）が残り続けるので、人が `needs-human` を外しても、次の実行でまた止まる。抜け道はコメントを消すことだけで、`review` の印を消すと `round` の欠けで止まる（IMPL-18）。「[止まった状態から戻す手順][f01]」にも、`marker-invalid` からの戻し方が無い。
  また、印の署名の鍵を替えると、前の鍵で署名した印がすべて無効になる。基本設計の「[状態遷移と判定][design-flow]」の「印の署名」は、鍵を替えるのを進行中の Issue が無いときにするとしているが、着手の前の Issue（下書きを人が読む番の Issue、承認の確定の後に着手を待つ Issue）の `intake` と `intake-final` の印も無効になり、そのすべてが止まって戻せない。鍵が漏れたときは、替えるしかない。
  候補：(a) 署名を確かめる鍵を一覧にし、新しい鍵で書き、前の鍵は確かめにだけ使い、前の鍵の印が残る Issue が閉じたら一覧から外す。(b) `marker-invalid` の戻し方を決め、「[止まった状態から戻す手順][f01]」に足す（例：人が `needs-human` を外した時刻より前に無効になった印は数えない）。(c) 鍵を替える前に、印のある開いた Issue を閉じる手順を基本設計に書く（ADR は要らないが、編集で止まった Issue は戻らず、鍵が漏れたときは印のある開いた Issue をすべて閉じることになる）。
  (a) と (b) は読む印の範囲（安全の境界）を変えるので、ADR で決める。鍵を替えてよい条件と替える手順も、同じ ADR で決める。受け入れ工程（段階 4）で GitHub に書き始める前に決める（[02-implement-review-fix.md][f02]、[01-issue-flow.md][f01]、[ADR-20261015-004][adr-markers]）~~
- UNDECIDED-68 **欠番**（[ADR-20261022-001][adr-url] で決めた。検疫は、当てる先のリポの GitHub の URL のほかの URL を探して止める。AI には、GitHub に投稿する文にほかの URL を書かせない）：~~**検疫が探す URL の範囲を、どこまでにするか。** TEXT-9 の「指示の混入の検査」は、題名・本文・コメント・添付のリンク先の文言に「外部へ取りに行かせる URL」が無いかを探すとする。TEXT-11 は、同じ TypeScript のモジュールで「URL」を探し、1 つでも見つかれば渡さずに止めて `needs-human` にするとする（fail-closed）。モジュールは 1 つなので、探す範囲も 1 つに決める必要がある。
  「URL」をすべて止めると、計画コメント（IMPL-10）と再レビューの入力の前回の指摘（TEXT-1）のように、AI が書いて次の AI に渡す文に、当てる先のリポの Issue・PR・ファイルや公式文書の URL があるだけで、Issue が `needs-human` で止まる。「外部へ取りに行かせる URL」は、取りに行かせる意図を機械で見分けられないので、機械の検査の範囲として決まらない。AI のセッションの外への通信は許可した先だけに限っている（SAFE-10）。
  候補：(a) URL をすべて探して止める（TEXT-11 のまま。TEXT-9 の書き方を直す）。(b) 当てる先のリポの GitHub の URL（`https://github.com/<owner>/<repo>/` で始まるもの）のほかの URL を探して止める。(c) URL は探さず、指示の形の語の検査に任せる。検疫の範囲（安全の境界）を変えるので、ADR で決める。受け入れ工程（段階 4）で GitHub に書き始める前に決める（[04-external-text-and-markers.md][f04]、[05-safety.md][f05]）~~

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
- UNDECIDED-57 **欠番**（[ADR-20261020-002][adr-app] で決めた。書き始める前に GitHub App に切り替え、非公開のリポのチェックの実行もアプリの Checks の読み取りで読む）：~~**非公開のリポに当てるときに、必須チェックの結果をどう読むか。** IMPL-27 は、必須チェックを名前・app・ワークフローのパスで照らし、チェックの実行の結果を読む。fine-grained PAT には Checks の権限が無く、公開のリポでは権限なしで読めたが（FACT-52）、非公開のリポでは 403 で読めなかった（FACT-54）。GitHub Actions のチェックは、ワークフローのジョブ（Actions の読み取り）から名前と結果を読めた（FACT-54）。
  非公開のリポに当てると決めるとき（ほかのリポに当てる時期は、マージ判定工程（段階 8）の後に決める。[07-structure.md][f07]）に、ジョブから読むか（GitHub Actions 以外の app のチェックは読めない）、Checks の権限を持てる GitHub App にするか、などを決める（[02-implement-review-fix.md][f02]）。
  GitHub の公式文書では、GitHub App のリポの権限に「Checks」があり、チェックの実行の一覧は読み取りで足りる（[GitHub App の権限の一覧][doc-app-permissions]）。入れたリポだけに絞れる。classic PAT も `repo` のスコープで読めるが（[チェックの実行の API][doc-check-runs]）、持ち主が触れるすべての非公開のリポを読み書きできるので、進行 Bot の PAT を当てる先のリポだけに絞る決まり（SAFE-15）に合わず、候補から外す。GitHub App にすると、投稿者は App の名前のアカウントになるので、UNDECIDED-20 の「アカウントを分ける」とあわせて決める（[ADR-20261020-001][adr-bots] の決定 1）~~
- UNDECIDED-58 **進行 Bot と AI のセッションを Docker のコンテナの中で動かし、OS への依存を減らすか。** 今は、AI のセッションの境界を Claude Code の sandbox（macOS は Seatbelt、Linux は bubblewrap）に置き、確かめた事実も macOS のもの（UNDECIDED-49）。Claude Code の sandbox は macOS・Linux・WSL2 で動き、Windows ではコマンドを sandbox なしで動かす（`failIfUnavailable` なら起動のときに止まる）。権限の無いコンテナの中では bubblewrap が `/proc` を作れず、特権のコンテナにするか、ほかのプロセスの情報が見える弱めた sandbox（`enableWeakerNestedSandbox`）にするかになる（[公式文書][doc-sandboxing]）。
  入れ方の候補：(A) 進行 Bot と AI を 1 つのコンテナに入れ、中の bubblewrap を境界にする（特権のコンテナが要る。弱めた sandbox では、AI から進行 Bot のプロセスの情報が見え、秘密が漏れる経路になりうる）。(B) 進行 Bot はマシンで動かし、AI のセッションを 1 回ごとに新しいコンテナで起こし、コンテナを境界にする（秘密は進行 Bot の側に残る。通信の許可先を絞る仕組みをコンテナの側で用意する）。(C) 進行 Bot もコンテナに入れ、そこから AI のコンテナを起こす（進行 Bot に Docker を操作させると、マシンの管理者に近い権限を渡す）。
  良くなる見込み：道具の版を OS によらずそろえられる。コンテナを消せば残ったコマンドも消え、止める手順（IMPL-14）が簡単になる。AI が触れるのは渡したフォルダだけになる。前提を変える承認済みの ADR：[ADR-20261013-001][adr-workspace]（作業場所）、[ADR-20261015-001][adr-read]（読み取りの範囲）、[ADR-20261017-006][adr-tmp]（一時フォルダ）、[ADR-20261018-001][adr-usage]（止め方）など。決めるのは、Linux か Windows のマシンに広げるとき、またはほかの人に配るとき（マージ判定工程（段階 8）の後。[07-structure.md][f07]）を目安にする。後で替えやすいように、詳細設計で AI のセッションを起こす処理を 1 つの部品にまとめておく（[05-safety.md][f05]）
- UNDECIDED-64 **欠番**（[ADR-20261021-005][adr-blocked] で決めた。名指しのパスを進行 Bot のプログラムに固定し、リポの設定には足す分だけを持つ）：~~**IMPL-15 が名指しする push しないパスを、設定の PR で外せない形にするか。** IMPL-15 は push しないパスの一覧を設定に持つとし、STRUCT-2 と [ADR-20261015-005][adr-settings] の決定 1 は、IMPL-15 の一覧からパスを外すのを、安全の境界を緩める値の変更として ADR で決めるとしている。点検（SETUP-2）は一覧の中身を見ないので、人の誤りの PR 1 本で `.github/**` が外れると、作業 AI の書いたワークフローが push され、同じリポの PR の CI が secrets 付きで動く（IMPL-15 が止めたい経路）。守りは PR のレビューと「ADR で決める」の運用だけになる。
  一方、sandbox の `denyRead` と、子プロセスに渡す環境変数の名前は、要件と ADR が決めた分を進行 Bot のプログラムに固定し、設定には足す分だけを持つ（STRUCT-2）。
  候補：(a) IMPL-15 が名指しするパス（`.claude/**` `CLAUDE.md` `**/CLAUDE.local.md` `.mcp.json` `.github/**` `dev-autopilot/**`）を進行 Bot のプログラムに固定し、設定には足す分（規約の正典のパスなど）だけを持つ。`denyRead` と環境変数の名前と同じ形にそろい、読み方が 1 つで済む。(b) 点検で、名指しのパスが一覧にあることを確かめ、無ければ fail にする。(c) 今の形を保つ。(a) と (b) は [ADR-20261015-005][adr-settings] の決定 1 を改めるので、ADR で決める。計画・実装工程（段階 5）で作業 AI の変更を push し始める前に決める（[02-implement-review-fix.md][f02]、[07-structure.md][f07]）~~
- UNDECIDED-65 **欠番**（[ADR-20261021-006][adr-repo] で決めた。owner と名前は、base ブランチの名前と GitHub App の 2 つの ID と一緒にマシンの設定に置き、`setup` が origin から書き、点検でマシンの設定のフォルダのパスと照らす）：~~**当てる先のリポの owner と名前をマシンの設定に置くことを、ADR でどう残すか。** STRUCT-2 は、owner と名前を、base ブランチと GitHub App の 2 つの ID と一緒にマシンの設定に置くとする（リポの設定を読む前に、進行 Bot の clone を作る先と全体の止める合図を読む先に要るため）。一方、[ADR-20261015-005][adr-settings] の決定 2 は「リポと Project の ID」をリポの設定に置き、[ADR-20261017-007][adr-stop] の決定 4 は base ブランチの名前だけをマシンの設定に移して、ほかの項目の割り振りは据え置くとしている。owner と名前をマシンの設定に移した ADR が無いので、ADR だけを読む人は owner と名前をリポの設定に置く（GitHub App の 2 つの ID は、[ADR-20261020-002][adr-app] の決定 3 が置き場を基本設計に任せている）。
  候補：(a) owner と名前をマシンの設定に置くことを新しい ADR で決め、承認の後に [ADR-20261015-005][adr-settings] へ Amended by を足し、基本設計の「[設定][design-settings]」の根拠を直す。(b) 「リポの ID」に owner と名前は入らないと読み、[ADR-20261015-005][adr-settings] の決定 2 に訂正の注記を足す（注記で済ませると、読んだ人は決定が変わったと読む。ADR の README の判断の基準）。導入と点検の実装（段階 3）で `setup` を作る前に決める（[07-structure.md][f07]）~~

## 基本設計の中で決めてよいもの

- UNDECIDED-18 **欠番**（基本設計の「[AI のセッション][design-ai]」の「レビューの軸の起こし方」で決めた。レビュー AI と批評 AI は品質ゲートの全体を回さず、CI の結果で判定する。指摘を確かめるための実行は行ってよい）：~~レビュー側で `pnpm check` を実行するか（CI の結果を待つだけにするか）~~
- UNDECIDED-19 **欠番**（基本設計の「[設定][design-settings]」の「マシンの中のフォルダ」で決めた）：~~実行の記録の置き場（記録と lock の置き場はマシンの設定の項目になった（[ADR-20261015-005][adr-settings]）ので、既定のパスを構造仕様の設定の表で決める。`setup` が作る「記録のフォルダ」の場所。SETUP-1）~~
- UNDECIDED-20 **欠番**（[ADR-20261020-001][adr-bots] で決めた。いまは分けない。分ける要望が出たら、machine user か GitHub App かを別の ADR で決め、印を書いたと認めるアカウントの一覧に足して切り替える）：~~自動化用の GitHub アカウントを分けるか（SETUP-13）。分けるなら machine user と GitHub App のどちらか。レビュー工程（段階 6。レビュー AI と批評 AI を足す段階）で、1 つのアカウントで印の判定が動くこと（GitHub のレビュー状態は使わない。IMPL-18）を実機で確かめてから決める~~
- UNDECIDED-22 **欠番**（基本設計の「[テスト][design-tests]」の「前提データ」で決めた。検疫の見本は `fixtures/quarantine/` に置く）：~~検疫の見本（指示の文例、区切りを模した文、見えない文字）の置き場。TEXT-11 は `test/` の前提データとし、STRUCT-4 と「[分離を見越した構成][f07]」の構成の図は、前提データを `fixtures/` に置き、`test/` には結合テストだけを置くとしている。要件メモの時点からある食い違い（[04-external-text-and-markers.md][f04]、[07-structure.md][f07]）~~
- UNDECIDED-27 **欠番**（基本設計の「[状態遷移と判定][design-flow]」の「作業場所の残骸」で決めた）：~~作業場所の残骸の決め方。[ADR-20261013-001][adr-workspace] では、セッションのフォルダ（`sessions/<セッションの名前>/`）は同じ名前で作り直すか片付ける（FLOW-27）まで残るので、作業中の Issue のフォルダがあるのは普通の状態になる。
  どのフォルダを残骸とみなすか（例：片付けの後に残ったもの、名前から Issue か PR を引けないもの）を決め、SETUP-2 の warn・Issue ごとの `needs-human`・fail に当てはめる（FLOW-29、SAFE-4、「運用の規則」）。
  消す前に読んだ `commondir` か `gitdir` が作ったときと違う作業場所は、ADR のとおり消さずに `needs-human` にし、`check` で warn にする~~
- UNDECIDED-39 **欠番**（基本設計の「[状態遷移と判定][design-flow]」の「印の種類」で決めた。`raise` と `duplicate` を足した）：~~進行 Bot のほかのコメントに付ける印の種類。モデルを上げたことを残すコメント（IMPL-25）と、🟢 の重複のときに PR 番号を足すコメント（IMPL-26）に当てはまる `note` の種類が無い（今は `wait` `overlap` `remind` `stop` `link`）。種類を足すか、既存の種類に寄せるかを決める（[ADR-20261015-004][adr-markers]）~~
- UNDECIDED-40 **欠番**（基本設計の「[AI のセッション][design-ai]」の「セッションごとの安全設定」で決めた。進行 Bot がプログラムの中の雛形からセッションごとに作り、起動の前と後にハッシュを比べる。リポの設定の足す分は `${home}` か `${worktree}` で始めて書く）：~~AI のセッションごとの安全設定の作り方。`allowRead`（自分のセッションのフォルダ）と `Write(//<worktree>/**)` はセッションごとに違うパスなのに、SAFE-2 の (e) と SAFE-5 は、`setup` が書く固定のファイルとハッシュの確かめを前提にしている。
  進行 Bot が雛形からセッションごとに作るか、雛形をどこに置きハッシュをどう確かめるか、リポの設定の `denyRead` `allowRead` の一覧にマシンのパスをどう書くか（記号で書いて起動の前に解決するなど。リポの設定には絶対パスを置かない。ADR-20261015-005）を決める~~
- UNDECIDED-41 **欠番**（[ADR-20261019-001][adr-repeat] で決めた）：~~同じ指摘の再発（修正の後のラウンドで、前のラウンドと同じ指摘がもう一度出ること）の数え方。IMPL-24 は 2 ラウンド続けて出たとき、「[レビューの Skill][f08]」の方針は 2 回出たときに人に渡すとする。間を空けて再び出た指摘も数えるかを決める~~
- UNDECIDED-42 **欠番**（[ADR-20261016-003][adr-move] で、評価を dev-autopilot のリポに置くと決めた）：~~`review` Skill の評価の置き場。「[レビューの Skill][f08]」は評価を business-os の `pnpm eval:cases` の仕組みの上に置くとし、STRUCT-4 は検査を dev-autopilot 自身が持つとする。分離のときに評価も移すかを決める~~
- UNDECIDED-43 **欠番**（基本設計の「[AI のセッション][design-ai]」の「人の Skill から受け入れを呼ぶ形」で決めた。`/verify-issue` は `dev-autopilot:intake` の Skill を呼び、`plugin/src/` を直接実行しない）：~~`/verify-issue` から dev-autopilot を呼ぶ形。STRUCT-6 は business-os の `/verify-issue` が検疫のモジュールと `dev-autopilot:brief:triager` を呼ぶとし、「[分離を見越した構成][f07]」は business-os が dev-autopilot を import しないとする。呼び出しなら許すのか、どう呼ぶか（Skill を呼ぶ、`plugin/src/` のコマンドを実行する）を決める~~
- UNDECIDED-46 **欠番**（`.gitleaks.toml` に規則を足して決めた。メールアドレス・電話番号・法人格・円の金額を、commit のフックと CI の全履歴の検査で止める。決まった形の無い固有名詞は見つけられない）：~~業務の固有名詞の混入を探す検査。business-os の中にあった間は、business-os の `check:leak`（メールアドレス・電話番号・法人格・金額・許可リストに無いドメインを探す）が dev-autopilot の文書も見ていた。dev-autopilot のリポの gitleaks は秘密の値だけを探すので、固有名詞の混入を探す検査が無い（「[決まっていること][f00]」の置き場所の行）。検査を足すか、足すならどう足すか（gitleaks の独自の規則、TypeScript の検査など）を決める~~
- UNDECIDED-52 **欠番**（基本設計の「[設定][design-settings]」の `models.workByType` で決めた。Issue の種類は作業ブランチの型にする）：~~Issue の種類の分け方。最初の実装で使うモデルは Issue の種類ごとに決める（IMPL-25、[ADR-20261018-003][adr-model]）が、要件は Issue をどう種類に分けるか（ブランチの型（IMPL-2）か、別の分け方か）を書いていない。計画・実装工程（段階 5）の記録の欄を決めるときに決める~~
- UNDECIDED-56 **欠番**（基本設計の「[AI のセッション][design-ai]」の「起動の形」で決めた。`--no-session-persistence` で会話の記録を人のホームに残さず、進行 Bot が受け取った `stream-json` を記録の置き場に残す。FACT-51）：~~AI のセッションの会話の記録の扱い。`claude -p` は、作業フォルダのパスから名前を付けたフォルダに、会話の記録を人のホームの `~/.claude/projects/` の下に残す（FACT-50）。進行 Bot が動かすと、セッションごとに記録がたまる。消すか、残すならどこにいつまで残すかを決める~~
- UNDECIDED-71 **専用マシンで進行 Bot を動かすときに、人の手元のマシンから受け入れの Skill（`intake`。`/verify-issue` から呼ぶもの）を使う形。** 基本設計の「[AI のセッション][design-ai]」の「人の Skill から受け入れを呼ぶ形」は、`intake` の Skill を、マシンの設定がある、進行 Bot を動かすマシンでだけ動かし、実行の lock を取って、進行 Bot と同じ置き場・安全設定・止める手順と記録で、受付 AI のセッションを 1 つ起こすとした（STRUCT-3。今は手元の Mac で進行 Bot を動かすので、`/verify-issue` も手元の Mac で呼べる。SAFE-18）。
  専用マシンに移すと、人の手元のマシンにはマシンの設定も進行 Bot のプログラムの写しも無いので、手元から `/verify-issue` で `intake` の Skill を使えない。手元のマシンで使う形（入れた Plugin のコードで動かし、セッションの置き場・安全設定・止める手順を手元に用意する、など）を、専用マシンに移すときに決める（[07-structure.md][f07]）

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
- UNDECIDED-54 **欠番**（FACT-46 で確かめた。AI のセッションの安全設定では、sandbox の読み取りの拒否（`~/`）がスナップショットの読み込みを止め、人の関数は当たらず、AI はスナップショットの置き場を読めない。`PATH` は進行 Bot が渡した値のまま）：~~**AI の Bash が、人のシェルのスナップショットを読み込む。** AI のセッションの Bash のコマンドは、Claude Code がホームの `.claude/shell-snapshots/` に作る人のシェルのスナップショット（関数・別名・`PATH`）を読み込んで動く（FACT-45）。
  SAFE-2 の (a) はユーザー設定から切り離すとし、SAFE-10 は読み取りを自分のセッションのフォルダだけにするとしている。スナップショットに何が入りうるか（環境変数が入ることがあるか）、AI のセッションから `shell-snapshots/` を読めるか、スナップショットを読み込ませない手段があるかを確かめる（[05-safety.md][f05]、[10-verified-facts.md][f10]）~~
- UNDECIDED-55 **欠番**（FACT-52・54 で確かめた。fine-grained PAT に Checks の権限は無い。当てる先の business-os は公開のリポなので、権限なしでチェックの実行を読めた。IMPL-27 の読み方は改めない。非公開のリポでは読めず、Actions のジョブからなら読めた。非公開のリポに当てるときの読み方は UNDECIDED-57）：~~**fine-grained PAT で、CI のチェックの実行（check runs）を読めるか。** IMPL-27 は、必須チェックを名前・app・ワークフローのパスで照らし、PR の head に対する結果を読む。GitHub の公式文書の fine-grained PAT の権限の一覧には、Commit statuses と Actions はあるが、チェックの実行の権限が見当たらない（基本設計の「導入と点検」の「PAT の権限」）。
  人が発行した PAT で、チェックの実行の一覧を読めるかを確かめる。読めなければ、必須チェックの結果を GitHub Actions のワークフローのジョブ（Actions の読み取り）から読む形に、IMPL-27 の読み方を改めるかを決める（[02-implement-review-fix.md][f02]）~~
- UNDECIDED-62 **sandbox の中から、macOS のキーチェーンを読めるか。** SAFE-18 は、手元の Mac で動かす間、sandbox が人の鍵と `gh` の設定を AI から隠すとする。`gh` のトークンと Claude Code の認証は、キーチェーンに入りうる。キーチェーンは、ホームの読み取りの拒否（`denyRead` の `~/`）で覆えるかが分からず、[ADR-20261015-001][adr-read] も確かめていない。
  試験用のキーチェーンの項目を作る操作は、Claude の自動の許可判定に止められたので、人が試す（試し方：試験用の項目を作り、AI のセッションと同じ安全設定の sandbox の中の Bash から `security find-generic-password` で読めないことを確かめて、項目を消す）。受け入れ工程（段階 4）で、手元の Mac で進行 Bot を動かす前に確かめる（[05-safety.md][f05]）
- UNDECIDED-69 **Administration の権限の無い GitHub App のインストールのトークンで、3 つの ruleset の `bypass_actors` と、base ブランチに効く規則（`GET /repos/{owner}/{repo}/rules/branches/{baseBranch}`）を読めるか。** SETUP-2 は、進行 Bot が自分の認証で ruleset を確かめ、bypass にリポの役・deploy key・一覧の外の GitHub App・進行 Bot のアプリが入っていれば fail にするとする。基本設計の「[導入と点検][design-setup]」は、毎回の点検で、3 つの ruleset の `bypass_actors` と、base ブランチに効く規則を読み、読めない欄があれば fail にする。
  [GitHub の公式文書][doc-rules]は、`bypass_actors` は ruleset に書き込める認証にだけ返るとする。組織の管理者の fine-grained PAT（Administration の権限なし）では、`bypass_actors` も base ブランチに効く規則も読めた（FACT-65）が、進行 Bot のアプリには Administration を付けない（[ADR-20261020-002][adr-app] の決定 2）ので、アプリのトークンでどちらも読めるかは確かめていない。読めなければ、GitHub App に切り替えた後の毎回の点検が fail になり、受け入れ工程（段階 4）で `dryRun` を切れない。
  確かめるにはアプリの登録（人の操作）が要るので、GitHub App に切り替えるときの試し（基本設計の「[導入と点検][design-setup]」の「GitHub App の権限」の切り替えの順 2。試しのリポに、当てる先のリポと同じ形の ruleset を作っておく）で、3 つの ruleset の `bypass_actors` と、base ブランチに効く規則を読めるかを確かめ、結果を FACT に残す。読めなかったときの扱いは、確かめたときに ADR で決める。ADR で決めるまでは、読めない欄を fail にする（[06-setup-and-stages.md][f06]）

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
[design-ai]: ../design/basic/03-ai-sessions.md
[adr-policy]: ../adr/adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths.md
[design-tests]: ../design/basic/05-tests.md
[adr-bots]: ../adr/adr-20261020-001-bot-accounts-as-list.md
[doc-app-permissions]: https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps
[doc-check-runs]: https://docs.github.com/en/rest/checks/runs
[adr-app]: ../adr/adr-20261020-002-switch-to-github-app-before-writing.md
[doc-sandboxing]: https://code.claude.com/docs/en/sandboxing
[adr-review-skill]: ../adr/adr-20261021-001-review-skill-text-without-skill-tool.md
[adr-dry]: ../adr/adr-20261021-002-dry-run-only-at-intake-stage.md
[adr-reapprove]: ../adr/adr-20261021-003-reapprove-by-relabel-and-ignore-outside-instructions.md
[adr-marker-keys]: ../adr/adr-20261021-004-marker-key-list-and-resume-after-invalid-markers.md
[adr-replan]: ../adr/adr-20261019-003-replan-on-plan-review-from-stage-5.md
[adr-repo]: ../adr/adr-20261021-006-repo-owner-and-name-in-machine-settings.md
[adr-blocked]: ../adr/adr-20261021-005-required-push-blocked-paths-in-program.md
[adr-url]: ../adr/adr-20261022-001-quarantine-urls-outside-target-repo.md
[adr-compare]: ../adr/adr-20261021-007-compare-review-done-from-local-records.md
[adr-gaps]: ../adr/adr-20261019-002-review-gaps-axis-in-shadow-first.md
[design-setup]: ../design/basic/04-setup-and-check.md
[doc-rules]: https://docs.github.com/en/rest/repos/rules
[adr-critic-skip]: ../adr/adr-20261022-002-skip-critic-when-no-high-or-medium-findings.md
