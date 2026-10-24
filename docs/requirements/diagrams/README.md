# dev-autopilot の図

dev-autopilot の要件のうち、関係を図にした方が読みやすいものを draw.io で描いた図。
図は要件の写しで、正本は[要件][req]。要件を変えたら、対応する図も同じ PR で直す。

| 図 | 何を表すか | 元にした要件 |
| --- | --- | --- |
| [全体の構成][overview] | 進行 Bot・AI の 5 役・作業場所・GitHub・人（メンテナと外の人）の関係。誰が何に書けるか | [00-overview.md][f00]、[01-issue-flow.md][f01]、[04-external-text-and-markers.md][f04] の TEXT-12 と印の一覧、[05-safety.md][f05] |
| [Issue の流れ][issue-flow] | Issue の起票（外の人でもコラボレータでも）からマージ後の片付けまでの正常な順と、止まる・待つ・戻す遷移。各工程の担い手と Project の Status。各工程が動き始める段階 | [01-issue-flow.md][f01] の FLOW-15〜34、[03-merge.md][f03] の MERGE-5・6、[04-external-text-and-markers.md][f04] の TEXT-12、[06-setup-and-stages.md][f06] の段階の表 |
| [隔離と鍵][isolation] | sandbox の境界、秘密の置き場、進行 Bot の clone（business-os の clone）とプログラムの写し、GitHub への書き込みの経路 | [05-safety.md][f05] の SAFE-1〜18 |
| [フォルダ構成と起動][structure] | dev-autopilot のリポの構成（予定）と、Plugin の入れ方・写し・定期の実行・進行 Bot・設定の呼び出しの関係 | [07-structure.md][f07] の STRUCT-1〜9 |

## 全体の構成

![dev-autopilot の全体の構成][overview]

## Issue の流れ

![Issue の流れ][issue-flow]

## 隔離と鍵

![隔離と鍵][isolation]

## フォルダ構成と起動

![フォルダ構成と起動の関係][structure]

## 図を直すとき

- ファイルは `.drawio.svg`（図のデータを埋め込んだ SVG）。draw.io のデスクトップ版か、VS Code の draw.io の拡張でそのまま開いて直せる
- 色は、下の「色の決まり」にそろえる
- 背景は白に固定してある（閲覧側がダークモードでも線と文字が見えるように）
- 2026-10-16 に draw.io 32.4.1 の CLI で書き出したとき、ラベルごとに代替の PNG が埋め込まれ、1 枚が 1 MB を超えた。代替の PNG はラベルの文字だけの代替に置き換えてある（ブラウザ・GitHub・VS Code のプレビューは HTML のラベルを表示するので見え方は変わらない）。draw.io で保存し直すと代替の PNG が戻ることがある
- draw.io で書き出し直したり保存し直したりしたら、コミットの前に次の 2 つを直す。どちらも書き出すたびに戻り、見つける検査は無い
  - 色：`light-dark(<明るい側の色>, <暗い側の色>)` を明るい側の色に置き換え、`color-scheme: light dark` を `color-scheme: light` にする（背景を白に固定する）
  - 代替の PNG：ラベルごとの `<image>` を、ラベルの文字だけを書いた `<text>` に置き換える
  - 直したかは、`grep -c -e 'light-dark' -e '<image ' *.drawio.svg` の結果がすべて 0 になることで確かめる

## 色の決まり

要件の図と基本設計の図で、箱の色を次の表にそろえる。凡例には表の「凡例の言葉」を使い、図ごとの補足は括弧で添える（例：「進行 Bot（軽い工程）」）。

| 色 | 塗り / 枠 | 意味 | 凡例の言葉 |
| --- | --- | --- | --- |
| 黄 | `#fff2cc` / `#d6b656` | 人（メンテナとコラボレータ）の操作と、人の鍵。人が呼ぶ Skill（`setup`・`check`・`/dev-autopilot`、人が `/verify-issue` から呼ぶ `intake`）も、人の Claude Code で人の権限で動くので黄 | 人、人が呼ぶ Skill |
| 青 | `#dae8fc` / `#6c8ebf` | 進行 Bot（AI ではない決まった処理のプログラム）の工程と、進行 Bot のコード・写し・clone | 進行 Bot |
| 緑 | `#d5e8d4` / `#82b366` | AI（受付 AI・計画 AI・作業 AI・レビュー AI・批評 AI）のセッションと、AI が読むエージェント定義と Skill（`review`） | AI のセッション |
| 紫 | `#e1d5e7` / `#9673a6` | GitHub（Issue・PR・Project とその欄・CI・ruleset・リポの変数） | GitHub |
| 赤 | `#f8cecc` / `#b85450` | 止める・止まる（`needs-human` と Blocked、起動しない、push しない、実行の失敗）と、秘密。赤の破線と × は「できない」 | 止める |
| 橙 | `#ffe6cc` / `#d79b00` | 一部だけ止めて、ほかは続ける | 一部だけ止める |
| 灰 | `#f5f5f5` / `#666666` | ファイル・設定・記録・OS の仕組み、起票する人（外の人を含む）、待つ・何もしない・残す | ファイル・仕組み、待つ |
| 白 | `#ffffff` / `#666666` | 問い（分かれ道）と、箇条の注記 | 問い |

- 1 つの箱に、色の違う結果を 2 つ入れない。分けて描く
- フォルダの箱は、中身を動かす主の色にする（`src/` は青、`agents/` は緑、人が呼ぶ Skill は黄）。主が違うものは箱を分ける
- 図を足すときと直すときに、この表と凡例の言葉で確かめる

[req]: ../README.md
[f00]: ../00-overview.md
[f01]: ../01-issue-flow.md
[f03]: ../03-merge.md
[f04]: ../04-external-text-and-markers.md
[f05]: ../05-safety.md
[f06]: ../06-setup-and-stages.md
[f07]: ../07-structure.md
[overview]: ./overview.drawio.svg
[issue-flow]: ./issue-flow.drawio.svg
[isolation]: ./isolation-and-keys.drawio.svg
[structure]: ./structure.drawio.svg
