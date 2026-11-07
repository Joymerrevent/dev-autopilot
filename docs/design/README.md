# dev-autopilot 設計

dev-autopilot の進行 Bot と、進行 Bot が起動する AI のセッションを、どう作るかを書く。
[要件][req]が「何をするか」を、[ADR][adr] が「どちらを選んだか」を決め、設計は「どう作るか」を決める。設計は、基本設計と詳細設計の 2 つに分ける。

| 設計 | 決めること | 置き場 |
| --- | --- | --- |
| 基本設計（構造仕様） | どの部品が、どの値で、どう動くか（設定の項目、状態遷移と判定の表、AI のセッションの起動の形、導入と点検の順番、テストの分け方） | `basic/`（読み方と文書の一覧は [basic/README.md][b00]） |
| 詳細設計 | 部品の中をどう作るか（`plugin/src/` のモジュールの分け方） | `detailed/`（書いている途中。読み方と文書の一覧は [detailed/README.md][d00]） |

要件と ADR が「構造仕様（`docs/design/`）」と書いているのは、基本設計のこと。

## 読み方

- 根拠は、要件の番号（FLOW-1 など）と ADR の番号（ADR-20261015-005 など）で示す。用語は[要件の読み方][req-read]と同じ（進行 Bot、AI の 5 役、工程、段階、印、渡す文）
- 設計は、要件と ADR に従う。食い違っていたら要件と ADR が正しく、設計を直す。詳細設計は基本設計に従う
- 節の図は、同じ節の表の写しで、正本は表。図は draw.io の `.drawio.svg` で `basic/diagrams/`（詳細設計は `detailed/diagrams/`）に置き、文書の名前の番号を頭に付ける（例：`02-run-flow.drawio.svg`）。書き出しと後処理と色の決まりは、[要件の図の直し方][diagrams]と同じ

## 設計を変えるとき

- 要件の形を変えない細かい選択（[決めていないこと][undecided]の「基本設計の中で決めてよいもの」）は、設計の PR で決める。
  決めた項目は、同じ PR で要件の番号を欠番にし、どの文書で決めたかを添える（例：`- UNDECIDED-19 **欠番**（基本設計の「設定」で決めた）：~~元の文~~`。書き方は[要件を変えるとき][req-change]と同じ）
- 要件の形を変える判断と、安全の境界を緩める変更（AI の道具・読み書きの範囲・通信先を広げる、MERGE-4 の許可一覧にパスを足す、IMPL-15 が名指しする変えてはいけないパスを外す（要件と進行 Bot のプログラムの変更）、変えてはいけないパスの足す分からパスを外す、上限を外す）は、設計の中で決めない。ADR を `proposed` で起票し、メンテナが `accepted` にしてから設計を直す（STRUCT-2、ADR-20261015-005、ADR-20261021-005）
- 設計を書く中で、要件の誤りや足りない点に気づいたら、要件を直す PR を別に出す。設計に要件の文を写さず、番号で指す
- 表を直したら、同じ PR で同じ節の図も直す

[req]: ../requirements/README.md
[req-read]: ../requirements/README.md#読み方
[req-change]: ../requirements/README.md#要件を変えるとき
[adr]: ../adr/README.md
[undecided]: ../requirements/09-undecided.md
[diagrams]: ../requirements/diagrams/README.md
[b00]: ./basic/README.md
[d00]: ./detailed/README.md
