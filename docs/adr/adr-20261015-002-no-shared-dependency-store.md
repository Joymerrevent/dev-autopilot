---
status: accepted
created: 2026-10-15
updated: 2026-10-15
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# AI のセッションは依存の store を共有せず、セッションの一時フォルダの store で依存を導入する

要件の UNDECIDED-25（[決めていないこと][undecided]の「依存の store をセッションの間で共有するか」）を決める。

## 背景と課題（Context and Problem Statement）

AI のセッションは、作業場所の中（sandbox の中）で `pnpm install` を実行して依存を導入する（IMPL-1）。pnpm は、取ってきた package を store（パッケージの置き場）に保管し、`node_modules` へ写す。
store をセッションの間で共有すると、ある AI のセッションが store のファイルを書き換えたときに、次のセッション（とくにレビュー AI と批評者）が書き換えた package で `pnpm check` を実行することになる。
[ADR-20261013-001][adr-workspace] が git の objects で塞いだ「前のセッションが細工したものを次のセッションが読む」経路と、同じ種類の論点である。

前提になる事実：

- [ADR-20261015-001][adr-read] で、AI のセッションはホームを読めなくした。人の共有の store（macOS では `~/Library/pnpm/store`）は、開けなければ読めない
- 同じ ADR で、AI のセッションには `XDG_DATA_HOME` をセッションの一時フォルダに向けて渡すと決めた。pnpm の store の既定の場所は、`XDG_DATA_HOME` があれば `$XDG_DATA_HOME/pnpm/store` になる（pnpm の公式文書）。
  手元で、AI のセッションと同じ環境変数の組み立てで `pnpm store path` を実行すると、セッションの一時フォルダの下を返した。`PNPM_HOME` を渡すと、`$PNPM_HOME/store` が優先された
- pnpm の公式文書は、store のファイルが変わったら使う前に中身を確かめる設定（`verifyStoreIntegrity`、既定は有効）について、「偶発的な破損を検出するもので、信頼できない利用者が書ける store を守るものではない」と書いている
- 費用：sandbox の中で空の store から `pnpm install` すると 9 秒（FACT-22）。sandbox の外では、共有の store で 1.1 秒、空の store で 3.0 秒（FACT-21）。取ってくる量は約 133 MB

課題：AI のセッションの間で、依存の store を共有するか。

## 判断の決め手（Decision Drivers）

- 前のセッションの AI が細工したものを、次のセッション（とくにレビュー AI と批評者）が読まないこと
- 人の store を、AI のセッションから書き換えられないこと
- 進行役が sandbox の外でリポやパッケージに由来するものを動かさないこと（SAFE-2 の (d)）
- 手順と設定が少ないこと

## 検討した選択肢（Considered Options）

- 案A：共有しない。セッションの一時フォルダの store で依存を導入する
- 案B：共有の store を、AI のセッションから読み書きできるように開ける
- 案C：共有の store を読むだけで開け、足りない package は進行役が sandbox の外で取ってくる

## 決定（Decision Outcome）

採用した選択肢:「案A：共有しない。セッションの一時フォルダの store で依存を導入する」。
理由: 前のセッションの細工が届かない形が、設定を足さずに成り立つ（ADR-20261015-001 の環境変数で、store は自然にセッションの一時フォルダの下になる）。費用はセッションごとに 10 秒ほどで済む。

中身：

- AI のセッションの pnpm の store は、セッションの一時フォルダの下（`$XDG_DATA_HOME/pnpm/store`）に置く。ADR-20261015-001 のとおり `XDG_DATA_HOME` をセッションの一時フォルダに向けて渡せば、`--store-dir` を指定しなくてもそこになる
- `PNPM_HOME` は AI のセッションに渡さない（渡すと store の場所が `$PNPM_HOME/store` に変わる）。子プロセスの環境変数は許可した名前だけで組み立てるので（SAFE-1、FACT-17）、許可する名前に入れない
- 共有の store（人の store も、dev-autopilot 用に別に作る store も）を、`allowRead` と `allowWrite` に足さない
- store は、セッションのフォルダを消すとき（ADR-20261013-001）に一緒に消える
- npm のキャッシュは、今の要件のまま一時フォルダに置く（ADR 20261006-01 の `npm_config_cache` の設定。SAFE-10）
- npm のレジストリにつながらず `pnpm install` が失敗したら、実行の失敗（IMPL-14）として次回に持ち越す
- 見直す時期：段階 9（並列と間隔の短縮）で、導入の時間や通信量が問題になったら、案C を確かめてから決め直す

### 影響（Consequences）

- 良い点: AI のセッションが書いた package のファイルは、次のセッションに届かない。レビュー AI と批評者は、レジストリから取ってきたままの依存で `pnpm check` を実行する
- 良い点: 人の store は AI から読めず、書き換えられない
- 良い点: 設定を足さずに済む（ADR-20261015-001 の環境変数のまま）
- 悪い点: セッションごとにレジストリから依存を取り直す（sandbox の中で約 9 秒、約 133 MB）。計画役（Bash なし）は導入しないので、取り直すのは作業 AI・レビュー AI・批評者のセッション
- 悪い点: レジストリにつながらない間は、AI のセッションが `pnpm check` を実行できない

### 確認方法（Confirmation）

- 段階 3 の `check`（SETUP-2）で、sandbox の中の `pnpm store path` がセッションの一時フォルダの下を返すことを確かめる
- `check` で、安全設定の `allowRead` と `allowWrite` に共有の store が無いことと、子プロセスに渡す環境変数の名前に `PNPM_HOME` が無いことを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案A：共有しない

- 良い点: 前のセッションの細工が届かない
- 良い点: 設定を足さずに成り立つ
- 悪い点: セッションごとに依存を取り直す（約 9 秒）

### 案B：共有の store を読み書きできるように開ける

- 良い点: 速い（sandbox の外で 1.1 秒）
- 悪い点: AI が store のファイルを書き換えると、次のセッションが書き換えた package を使う。pnpm の `verifyStoreIntegrity` は、信頼できない利用者が書ける store を守るものではないと公式文書に書かれている
- 悪い点: 人の store を開けると、人の手元の別のプロジェクトも書き換えた package を使いうる
- 却下

### 案C：共有の store を読むだけで開け、足りない package は進行役が取ってくる

- 良い点: AI は store に書けず、速い
- 悪い点: 進行役が sandbox の外で `pnpm` を動かし、AI が書き換えうる lockfile を読んで外へ通信する。lifecycle script は動かさない（`--ignore-scripts`）が、SAFE-2 の (d) の境界に近づく
- 悪い点: 読むだけの store で `pnpm install` が通るかを確かめていない。仕組みが増える
- 見送り。段階 9 で導入の時間が問題になったら確かめる

## 補足情報（More Information）

- 関連する要件：IMPL-1、IMPL-14（[02-implement-review-fix.md][impl]）、SAFE-1、SAFE-2 の (d)、SAFE-10（[05-safety.md][safety]）、SETUP-2（[06-setup-and-stages.md][setup]）、FACT-17、FACT-21、FACT-22（[10-verified-facts.md][facts]）、[ADR-20261013-001][adr-workspace]、[ADR-20261015-001][adr-read]
- pnpm の公式文書（[Store の設定][pnpm-store]）：`storeDir` の既定は、`$PNPM_HOME` があれば `$PNPM_HOME/store`、`$XDG_DATA_HOME` があれば `$XDG_DATA_HOME/pnpm/store`、macOS では `~/Library/pnpm/store`。
  `verifyStoreIntegrity`（既定は有効）は「偶発的な破損を検出するが、信頼できない利用者が書ける store を守るものではない」（2026-10-15 に読んだ）
- 2026-10-15 の確認（macOS、pnpm 10.29.3。sandbox とモデルは使わない）：許可した名前の環境変数だけで、`XDG_DATA_HOME` を一時フォルダに向けて `pnpm store path` を実行すると、`<一時フォルダ>/pnpm/store/v10` を返した。`PNPM_HOME` を足すと `<PNPM_HOME>/store/v10` を返した
- 要件への反映（承認の後に、別の PR で行う）
  - SAFE-10：AI のセッションの pnpm の store はセッションの一時フォルダの下に置き、共有の store を `allowRead` と `allowWrite` に足さないこと、`PNPM_HOME` を渡さないことを書き足す
  - IMPL-1：依存の導入はセッションの一時フォルダの store で行い、レジストリにつながらない失敗は実行の失敗（IMPL-14）として持ち越すことを書き足す
  - SETUP-2：決定の確認方法の検査を足す
  - 10-verified-facts.md：2026-10-15 の確認を FACT-25 として足す
  - 09-undecided.md：UNDECIDED-25 を欠番にする（元の文を取り消し線で残す）

[undecided]: ../requirements/09-undecided.md
[impl]: ../requirements/02-implement-review-fix.md
[safety]: ../requirements/05-safety.md
[setup]: ../requirements/06-setup-and-stages.md
[facts]: ../requirements/10-verified-facts.md
[adr-workspace]: ./adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[pnpm-store]: https://pnpm.io/settings/store
