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
- 色の使い分け：黄はメンテナ（人の操作と人の鍵）、青は進行 Bot（AI ではない決まった処理のプログラムと、進行 Bot のコード）、緑は AI（受付 AI・計画 AI・作業 AI・レビュー AI・批評 AI のセッションと、Plugin の `agents/` と `skills/`）、紫は GitHub、赤は秘密と、人の対処が要る停止（Blocked）、灰は起票する人（外の人を含む）とファイル・設定などの仕組み。赤の破線と × は「できない」を表す
- 背景は白に固定してある（閲覧側がダークモードでも線と文字が見えるように）
- 2026-10-16 に draw.io 32.4.1 の CLI で書き出したとき、ラベルごとに代替の PNG が埋め込まれ、1 枚が 1 MB を超えた。代替の PNG はラベルの文字だけの代替に置き換えてある（ブラウザ・GitHub・VS Code のプレビューは HTML のラベルを表示するので見え方は変わらない）。draw.io で保存し直すと代替の PNG が戻ることがある
- draw.io で書き出し直したり保存し直したりしたら、コミットの前に次の 2 つを直す。どちらも書き出すたびに戻り、見つける検査は無い
  - 色：`light-dark(<明るい側の色>, <暗い側の色>)` を明るい側の色に置き換え、`color-scheme: light dark` を `color-scheme: light` にする（背景を白に固定する）
  - 代替の PNG：ラベルごとの `<image>` を、ラベルの文字だけを書いた `<text>` に置き換える
  - 直したかは、`grep -c -e 'light-dark' -e '<image ' *.drawio.svg` の結果がすべて 0 になることで確かめる

[req]: ../requirements/README.md
[f00]: ../requirements/00-overview.md
[f01]: ../requirements/01-issue-flow.md
[f03]: ../requirements/03-merge.md
[f04]: ../requirements/04-external-text-and-markers.md
[f05]: ../requirements/05-safety.md
[f06]: ../requirements/06-setup-and-stages.md
[f07]: ../requirements/07-structure.md
[overview]: ./overview.drawio.svg
[issue-flow]: ./issue-flow.drawio.svg
[isolation]: ./isolation-and-keys.drawio.svg
[structure]: ./structure.drawio.svg
