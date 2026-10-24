---
status: accepted
created: 2026-10-18
updated: 2026-10-24
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# AI のセッションは SIGTERM で止めて残ったコマンドも止め、途中で止めた回の費用は 1 セッションの費用の上限とみなし、利用枠は費用で数える

> **［2026-10-24 追記］** `needs-human`・Status・Agent 欄・`note` の印の書き方は、[ADR-20261024-001][adr-20261024-001] の決定 11 で読む（`needs-human` を付けるのは停止中にすること、外すのは `## 再開 <戻す先>` を書くこと、Agent 欄の `paused`・`human` は `agent-paused`・`agent-manual` のラベル）。

要件の UNDECIDED-47（[決めていないこと][undecided]の「安全と設定」の「途中で止めた回の消費の数え方」）を決める。
あわせて、確かめることの UNDECIDED-44（作業中の `claude` を途中で止めたときに、孫プロセスと消費の記録がどうなるか）を実機で確かめた結果から、止め方を決める。

## 背景と課題（Context and Problem Statement）

- 進行ボットは、時間の上限（IMPL-13。初期値 45 分）と全体の止める合図（[ADR-20261017-007][adr-stop]）で、AI のセッションを途中で止める。IMPL-14 は、子プロセスを別のプロセスグループに起こし、グループごと `SIGTERM` で止めるとする
- IMPL-14 は、1 Issue の累計の利用枠と 1 日の利用枠を、記録の `usage` と `modelUsage` から数え、途中で止めた回も `stream-json` の `assistant` のイベントの `usage` を足し上げて数えるとする。
  一方、`assistant` のイベントの `usage` は同じメッセージで重複し、`output_tokens` は最後の値でない（FACT-29）。そのまま足すと出力を少なく数え、累計の上限が黙って効かなくなる
- 2026-10-18 に、sandbox を掛けた `claude -p` を途中で止める試験をした（補足情報）。分かったことは次のとおり
  - Bash のコマンドは、`claude` と別のプロセスグループで動く
  - `SIGTERM` を受けた `claude` は、動いているコマンドのプロセスグループを止め、1 秒ほどで終了コード 143 で終わる。`result` は出ない
  - コマンドが `SIGTERM` を無視すると（`trap '' TERM`）、`claude` が終わった後もコマンドが残って動き続ける。`SIGKILL` で `claude` のプロセスグループを止めたときも、コマンドが残る。残ったコマンドは sandbox の中のままで、外には書けなかった
  - `SIGINT` を受けた `claude` は `result` を出して終わる。ただし、書いている途中のメッセージの消費を数えず、途中で止めた回の `total_cost_usd` は 0 だった
  - `--include-partial-messages` を付けると、書き終えたメッセージごとに `message_delta` が出て、最後のトークン数を持つ。書いている途中で止めたメッセージは、`message_start`（入力の側のトークン数）だけで、出力のトークン数は分からない
  - `--max-budget-usd` はメッセージの間で確かめるので、1 セッションの費用は上限をメッセージ 1 つ分まで超えうる（上限 0.01 で 0.0769 使った）
- 要件は、利用枠をトークンで数えるか費用で数えるかを決めていない。`stream-json` のイベントは費用を持たず、費用は `result` の `total_cost_usd` と `modelUsage` の `costUSD` にだけある

課題：AI のセッションを途中で止めるとき、コマンドを残さずに止めるにはどうするか。途中で止めた回の消費を、少なく数えずにどう数えるか。利用枠は何で数えるか。

## 判断の決め手（Decision Drivers）

- 止めた後に、AI のコマンドが動き続けないこと（作業場所を作り直したり消したりする前に止まっていること）
- 消費を少なく数えないこと。少なく数えると、累計の上限と 1 日の上限が黙って効かなくなる（フェイルセーフ。SAFE-4）
- 進行ボットが、モデルの単価の表のような、外の値の写しを持たずに済むこと（写しが古くなると、少なく数える）
- モデルの違う消費（計画 AI の `fable`、作業 AI の `sonnet` など）を、同じ物差しで足せること

## 検討した選択肢（Considered Options）

問い 1：途中で止めるときの止め方

- 案 1-A：`claude` のプロセスグループに `SIGTERM` を送り、猶予の間に終わらなければ `SIGKILL` を送る。`claude` が終わった後、セッションのコマンドの残りを止める
- 案 1-B：今の要件のまま、プロセスグループに `SIGTERM` を送るだけにする
- 案 1-C：`SIGINT` で止め、`claude` が出す `result` を使う

問い 2：途中で止めた回の消費の数え方

- 案 2-A：費用は、その回の 1 セッションの費用の上限（`--max-budget-usd`）とみなす。トークンは `message_delta` からメッセージごとに数えて記録する
- 案 2-B：`message_delta` のトークンを、進行ボットが持つモデルごとの単価の表で費用に直す。書いている途中のメッセージは、出力を 1 回の応答の上限まで使ったとみなす
- 案 2-C：今の要件のまま、`assistant` のイベントの `usage` を足し上げる

問い 3：累計の利用枠と 1 日の利用枠の物差し

- 案 3-A：費用（Claude Code が出す USD 換算の値）で数える
- 案 3-B：モデルごとのトークンで数える

## 決定（Decision Outcome）

採用した選択肢:「案 1-A」「案 2-A」「案 3-A」。
理由: 止めた後にコマンドが残らないことを、`claude` の後片付けに頼らずに進行ボットが確かめられる。途中で止めた回の費用は、単価の表を持たずに、ほとんどの場合で多めに数えられる。費用で数えれば、モデルの違う消費を同じ物差しで足せる。

### 決定 1：途中で止めるときは、SIGTERM の後に残りを止める

- 進行ボットは、AI のセッションを途中で止めるとき（時間の上限、全体の止める合図）、`claude` のプロセスグループに `SIGTERM` を送る。`claude` は、動いているコマンドのプロセスグループも止めて終わる
- 猶予（値は基本設計で決める。試験では 1 秒ほどで終わった）の間に終わらなければ、`SIGKILL` を送る
- `claude` が終わった後、そのセッションの Bash のコマンドのプロセスグループが残っていないかを確かめ、残っていれば `SIGKILL` で止める。コマンドは `claude` と別のプロセスグループで動くので、`SIGTERM` を無視したコマンドと、`SIGKILL` の後のコマンドは残る。
  残りの見つけ方（止める前にセッションの子孫のプロセスグループを控える、など）は基本設計で決める
- 残りが止まったと確かめられなければ、作業場所を消さず、そのセッションの次の AI のセッションを起動せず、`needs-human` にする（SAFE-4）。残ったコマンドは sandbox の中のままで、書けるのは自分のセッションの作業場所と一時フォルダだけ（試験で確かめた）
- `SIGINT` は使わない（書いている途中のメッセージの消費を 0 と数える `result` を出すため）

### 決定 2：途中で止めた回の費用は、1 セッションの費用の上限とみなす

- 進行ボットは、AI のセッションを `--include-partial-messages` 付きで起動し、メッセージごとの `message_delta` の `usage`（最後のトークン数）を記録に残す。`assistant` のイベントの `usage` は足し上げに使わない（FACT-29）
- 正常に終わった回（`result` があり、`terminal_reason` が中断（`aborted_` で始まる値）でない）は、`result` の `total_cost_usd` を費用とし、`modelUsage` をモデルごとのトークンとして記録する
- 途中で止めた回（`result` が無いか、`terminal_reason` が中断）は、費用をその回の 1 セッションの費用の上限（`--max-budget-usd`）とみなす。トークンは、書き終えたメッセージの `message_delta` と、書いている途中のメッセージの `message_start` から数え、出力が分からないメッセージがあったことを記録に残す
- 上限とみなすと、ほとんどの場合は多めに数える（止めた時点では費用の上限に届いていないため）。ただし、書いている途中で止めたメッセージ 1 つ分だけ、少なく数えうる（`--max-budget-usd` がメッセージの間で確かめるのと同じ幅）

### 決定 3：累計の利用枠と 1 日の利用枠は、費用で数える

- 1 Issue の累計の利用枠と 1 日の利用枠（IMPL-14、SETUP-12）は、費用（Claude Code が出す USD 換算の値）で数える。上限の値は設定に持つ（初期値は構造仕様で決める）
- トークンは、モデルごとに記録に残し、`/dev-autopilot` の表示と、段階の出口の見直し（モデルの振り分け）に使う

### 影響（Consequences）

- 良い点: 止めた後に AI のコマンドが動き続けない。止まったと確かめられないときは、作業場所を消さずに人に渡す
- 良い点: 途中で止めた回を 0 や少なめに数えることがなくなり、累計の上限と 1 日の上限が効く
- 良い点: 単価の表を持たずに済み、モデルの違う消費を同じ物差しで足せる
- 悪い点: 途中で止めた回は、実際より多く数えることが多い。止める回が続くと、累計の上限に早く当たる（人が `needs-human` を見て判断する）
- 悪い点: 書いている途中で止めたメッセージ 1 つ分は、少なく数えうる
- 悪い点: 費用は Claude Code の計算に頼る。契約の形（従量か定額か）によっては、実際の請求と合わない（物差しとしてだけ使う）
- 悪い点: Linux で同じに振る舞うかは確かめていない（UNDECIDED-49 で確かめ直す）

### 確認方法（Confirmation）

- 進行ボットの単体テストで、止める手順（`SIGTERM`、猶予、`SIGKILL`、残りの確かめ）と、止まったと確かめられないときに `needs-human` にすることを検査する。記録の数え方（正常な回と止めた回）も、試験で得た `stream-json` の記録を前提データにして検査する
- 段階 3（`setup` と `check` を作る段階）で、`SIGTERM` を無視するコマンドを実行中のセッションを止め、残りが止まることを実機で試す

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 1-A：SIGTERM、猶予、SIGKILL、残りを止める

- 良い点: `claude` が後片付けをしないとき（コマンドが `SIGTERM` を無視した、`claude` が固まった）でも、コマンドが残らない
- 悪い点: 残りを見つける仕組みが要る

### 案 1-B：SIGTERM を送るだけ

- 良い点: 今の要件のままで、仕組みが少ない
- 悪い点: `SIGTERM` を無視するコマンドが残る（試験で、`claude` が終わった後も動き続けた）。AI がそういうコマンドを書けば、止めた後も動き続ける

### 案 1-C：SIGINT で止めて result を使う

- 良い点: `result` が出て、書き終えたメッセージの消費は `modelUsage` に入る
- 悪い点: 書いている途中で止めると、`total_cost_usd` が 0 になる（試験で確かめた）。少なく数える

### 案 2-A：費用は上限とみなし、トークンは message_delta で数える

- 良い点: 単価の表が要らない。ほとんどの場合で多めに数える
- 悪い点: 実際より多く数えることが多い。書いている途中のメッセージ 1 つ分は少なく数えうる

### 案 2-B：トークンを単価の表で費用に直す

- 良い点: 実際に近い値になる
- 悪い点: モデルごとの単価の表を写して持ち、値上げや新しいモデルに合わせて直す必要がある。直し忘れると少なく数える。書いている途中のメッセージを 1 回の応答の上限まで使ったとみなすと、多く数え過ぎる

### 案 2-C：assistant のイベントの usage を足し上げる

- 良い点: 今の要件のまま
- 悪い点: 出力を少なく数え、累計の上限が黙って効かなくなる（FACT-29）

### 案 3-A：費用で数える

- 良い点: モデルの違う消費を同じ物差しで足せる。1 セッションの費用の上限（`--max-budget-usd`）と同じ物差しになる
- 悪い点: Claude Code の計算に頼る

### 案 3-B：トークンで数える

- 良い点: 請求の形に依らない
- 悪い点: モデルごとに重みが違い、足しても意味のある値にならない。モデルごとに上限を持つと、設定の項目が増える

## 補足情報（More Information）

- 関連する要件：IMPL-13、IMPL-14（[02-implement-review-fix.md][impl]）、SAFE-4（[05-safety.md][safety]）、SETUP-12（[06-setup-and-stages.md][stages]）、STRUCT-2（[07-structure.md][struct]）、FACT-14、FACT-29、FACT-31（[10-verified-facts.md][facts]）、
  UNDECIDED-44、UNDECIDED-47、UNDECIDED-49（[09-undecided.md][undecided]）、[ADR-20261017-006][adr-tmp]、[ADR-20261017-007][adr-stop]
- 2026-10-18 に、途中で止める試験をした（macOS、Claude Code 2.1.285、`claude-haiku-5-5`。試験の場所は `/tmp/claude-<uid>/` と `~/.claude/` の外）。
  `claude -p --setting-sources local --settings <安全設定（sandbox を有効、allowUnsandboxedCommands は false）> --strict-mcp-config --output-format stream-json --verbose --include-partial-messages` を、許可した名前だけの環境変数とセッションごとの短い一時フォルダ（[ADR-20261017-006][adr-tmp]）で、別のプロセスグループに起動した
  - コマンドの実行中（`sh long.sh` が `sleep` を子に持ち、1 秒ごとに印のファイルを書く）に止めた。コマンドのプロセスグループは `claude` のものと違った
    - プロセスグループに `SIGTERM`：`claude` は 1 秒ほどで終了コード 143。3 秒後にコマンドは残らず、印のファイルの更新も止まった。`result` は無く、書き終えたメッセージの `message_start` と `message_delta`（出力 114）が出ていた
    - `claude` のプロセスだけに `SIGTERM`：同じく、コマンドは残らなかった
    - プロセスグループに `SIGKILL`：`claude` はすぐに終わったが、コマンド（`sh` と `sleep`）は親が PID 1 になって残り、印のファイルを書き続けた
    - コマンドが `trap '' TERM HUP INT` で `SIGTERM` を無視するとき、プロセスグループに `SIGTERM`：`claude` は終了コード 143 で終わったが、コマンドは残って動き続けた。`claude` が終わった後に、コマンドが作業場所の外に書こうとすると、sandbox が止めた
    - `claude` のプロセスだけに `SIGINT`：`claude` は終了コード 0 で終わり、コマンドは残らなかった。`result` は `subtype` が `error_during_execution`、`terminal_reason` が `aborted_tools` で、`modelUsage` は書き終えたメッセージの `message_delta` と同じ値だった
  - 長い文を書いている途中に止めた
    - プロセスグループに `SIGTERM`：`message_start`（入力の側のトークン数。出力は 2）と、文の差分（`text_delta`）だけが出て、`message_delta` も `result` も無かった
    - `claude` のプロセスだけに `SIGINT`：`result` は出たが、`terminal_reason` が `aborted_streaming`、`total_cost_usd` が 0、`usage` のトークンもすべて 0 だった
  - 試験で残ったコマンドは、試験のスクリプトが `SIGKILL` で止め、試験のフォルダは消した
- 同じ日に、`--max-budget-usd 0.01` で 1 つの短い指示を実行した。`result` は `subtype` が `error_max_budget_usd`、`terminal_reason` が `budget_exhausted`、`total_cost_usd` が 0.0769 だった（上限をメッセージ 1 つ分超えた）
- 正常に終わった回の `result` の `terminal_reason` は `completed` だった（2026-10-17 のサブエージェントの試験の記録。FACT-31）
- 要件への反映（承認の後に、別の PR で行う）
  - 02-implement-review-fix.md：IMPL-13 と IMPL-14 の止め方（`SIGTERM`、猶予、`SIGKILL`、残りを止める、`SIGINT` を使わない）、IMPL-14 の数え方（`--include-partial-messages`、正常な回は `result`、止めた回は費用を上限とみなす、利用枠は費用で数える）
  - 07-structure.md：STRUCT-2（止めるときの猶予）
  - 10-verified-facts.md：上の試験の事実を FACT として足し、FACT-14 の「`claude` 自体を途中で止める試験は未実施」を直す
  - 09-undecided.md：UNDECIDED-47 と UNDECIDED-44 を欠番にする（元の文を取り消し線で残す）。UNDECIDED-49 に、Linux で止め方と残りの振る舞いを確かめ直すことを足す

[impl]: ../requirements/02-implement-review-fix.md
[safety]: ../requirements/05-safety.md
[stages]: ../requirements/06-setup-and-stages.md
[struct]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[adr-tmp]: ./adr-20261017-006-per-session-short-tmpdir.md
[adr-stop]: ./adr-20261017-007-stop-switch-in-repo-variable.md
[adr-20261024-001]: ./adr-20261024-001-work-unit-main-flow-and-overlay-axes.md
