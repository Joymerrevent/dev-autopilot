# 詳細設計

基本設計（[設計の読み方][design]）が決めた部品を、`plugin/src/` のどのモジュールで、どう作るかを決める。
README は、詳細設計の全体に効く決まり（層と依存の向き、フォルダとモジュールの一覧、入口ごとの import の範囲、テストで外との境を差し替える方法、構成の検査に足すこと、TypeScript の書き方）と、部品ごとの文書の一覧を持つ。

## 読み方

- 詳細設計は基本設計に従う。食い違っていたら基本設計が正しく、詳細設計を直す。詳細設計を書く中で基本設計の誤りに気づいたら、基本設計を直す PR を別に出す（[設計の読み方][design]）
- 根拠は、基本設計の文書と節（例：「[状態遷移と判定][b02]」の「状態の組み立て」）、要件の番号、ADR の番号で示す。基本設計の表を写さない
- フォルダ・ファイル・関数・型の名前は英語で付け、説明は日本語で書く
- 節の図は、同じ節の表の写しで、正本は表。図は `detailed/diagrams/` に置き、文書の番号を頭に付ける（README の図は `00-`）。書き出しと後処理と色の決まりは、[要件の図の直し方][diagrams]と同じ

## 文書の一覧

部品ごとの文書は、導入と点検の実装（段階 3）で先に要るものから順に書く。書いたら、表の「状態」を直す。

| 文書 | 話題 | 状態 |
| --- | --- | --- |
| README.md | 層と依存の向き、フォルダとモジュールの一覧、入口ごとの import の範囲、テストで外との境を差し替える方法、構成の検査に足すこと、TypeScript の書き方 | あり |
| [01-io.md][d01] | 外との境（システム、子プロセスの時間の上限と止め方、GitHub の API・`git`・`claude` の呼び方、ファイル、秘密の置き場、HTTP、OS ごとの部分） | あり |
| [02-settings-and-records.md][d02] | 設定の読み込みと確かめ、手元の記録、lock、依頼と結果のファイル、`status.json` | あり |
| [03-markers.md][d03] | 印の形、署名に使う JSON の並べ方、鍵の読み方 | あり |
| [04-check.md][d04] | 点検（組ごとの処理と、setup の読み戻しと共有する比べる規則） | あり |
| [05-setup.md][d05] | setup | あり |
| [06-text-and-intake.md][d06] | 検疫、漏えいの検査、ハッシュ、コメントの組み立て、出力契約、見出しの比べ方、intake | あり |
| [07-sessions.md][d07] | AI のセッション | あり |
| [08-flow.md][d08] | 1 回の実行の流れ、状態の組み立て、次の工程、工程、push の手順、判定、全体の停止と通知、実装の都合で止める理由の記号 | あり |

## 層と依存の向き

`plugin/src/` のファイルを 4 つの層に分け、決まった向きにだけ import する。
外に触れる Node の機能（子プロセス、ファイル、時刻、乱数、待つこと、ネットワーク、環境変数、シグナル）を使えるのは入口の層だけにする。入口は、外に触れる Node の機能をまとめた値（システム。`System` 型）を作り、外との境に渡す。流れの層には、システムのうち時刻と待ちと乱数の 3 つ（`now`・`sleep`・`randomBytes`）だけを持つ値（`FlowSystem` 型）を渡す。

![層と依存の向き][fig-layers]

| 層 | 置き場 | 受け持つこと | import してよい先 | 外への触れ方 |
| --- | --- | --- | --- | --- |
| 入口 | `main.ts`、`entry/` | 引数を読み、システムを作り、外との境と `FlowSystem` を組み立てて、流れを呼ぶ。終了コードを決め、`process.exit` で終える（取り消し忘れのタイマーで終わりを遅らせないため） | すべての層 | 外に触れる Node の機能を使う。使ってよいのは入口の層だけ |
| 流れ | `flow/`、`check.ts` | 基本設計の表の順番どおりに、判断を呼んで次にすることを決め、外との境を呼ぶ | 流れ、判断。外との境は型だけ（`import type`） | 引数で受けた外との境と、`FlowSystem` を通してだけ |
| 外との境 | `io/` | 外のコマンド（`git`・`claude`・`lsof`・`launchctl`・`systemctl` など）と GitHub の API とファイルと OS を、決まった形で呼ぶ。子プロセスの時間の上限と止め方、秘密を子プロセスに渡す範囲を受け持つ | 外との境、判断 | 引数で受けたシステムを通してだけ |
| 判断 | `logic/`、`marker.ts` | 渡された値から結果を決める（状態、次の工程、判定、印の形と署名、検疫、出力契約、設定の確かめ） | 判断だけ | 触れない |

- 外に触れる Node の機能を入口に集めるのは、単体テストで本物を使わず、外に触れるものを引数で渡す偽物にするため（「[テスト][b05]」の「テストの種類」）。入口のほかのファイルは、偽物を渡すだけで単体テストできる
- 外との境の層も、システムを引数で受けて単体テストする。止める手順（`SIGTERM` から `SIGKILL` までの猶予、プロセスグループ）や、子プロセスが時間の上限を過ぎたときの扱いのように、外との境の中にも間違えやすい処理があるため
- 流れの層が外との境を型だけで import するのは、外との境の実体を流れの中で作れなくするため。流れの単体テストは、外との境の型に合う偽物を渡す
- 流れの層が使ってよいシステムの道具は、時刻・待つこと・乱数（`FlowSystem`）だけ。実行の期限、結果を待つ間隔、ID の乱数は流れの順番そのものなので、外との境を挟まない。外の結果を読む道具（`kill`・`spawn`・`fs`・`fetch`・`env`）は流れに渡さず、外との境を通す（プロセスがあるかは、`io/processes.ts` の「プロセスがあるか」で読む。「[外との境][d01]」の「プロセスの一覧と開いているファイル」）
- 判断の層と流れの層が使ってよい Node の機能は、外に触れないもの（下の「構成の検査に足すこと」の許す一覧の、`node:path` と `node:crypto` の決めた関数）だけ。`node:crypto` の乱数は、外に触れるものに数える。乱数は区切りの文字列や ID に使い、単体テストで決まった値にするため
- 入口は薄くし、引数の読み方のほかの処理を持たない。システムを作る部分は、結合テスト（入口を子プロセスで起こす）で確かめる。ただし `entry/check-probe.ts` は、`node:` のほかを import できないので試しの処理を自分で持ち、外に触れる関数を引数で受ける（「[点検][d04]」の「ファイルの一覧」の下）
- 入口が動的な `import()` を使ってよいのは、`entry/setup.ts` が依存の照らし（`entry/deps-check.ts`）の後に本体を読み込むときだけ。読み込む先は相対パスの文字列の定数にする（「[setup][d05]」の「プログラムの写し」）。静的な import のままでは、依存が無いときに照らしの前に起動で止まり、止まる理由を示せないため（「[導入と点検][b04]」の「依存」）
- 層の決まりは、構成の検査（`test/repo/`）で確かめる（下の「構成の検査に足すこと」）

## フォルダとモジュールの一覧

名前が要件で決まっている `main.ts`（入口）、`marker.ts`（印の形の正本）、`check.ts`（点検の正本）は、`plugin/src/` の直下に置く（[分離を見越した構成][struct]）。
ほかのファイルは、層ごとのフォルダ（`entry/`・`flow/`・`io/`・`logic/`）の下に、話題ごとのフォルダを置いて分ける。フォルダの中のファイルの分け方は、部品ごとの文書で決める。

| 置き場 | 層 | 受け持つこと | 基本設計 | 詳細設計 |
| --- | --- | --- | --- | --- |
| `main.ts` | 入口 | 進行 Bot の入口。定期の実行が、写しの `src/main.ts` を、マシンの設定のパスを引数にして起こす | 「[導入と点検][b04]」の「macOS（launchd）」、「[状態遷移と判定][b02]」の「1 回の実行の流れ」 | `08-flow.md` |
| `entry/system.ts` | 入口 | システムの実体を作る。`main.ts` とほかの入口が使う | — | `01-io.md` |
| `entry/setup.ts` | 入口 | 人の `setup` の入口（引数の `rollback`・`upgrade`・`uninstall` を含む）。入れた Plugin の `src/` から動く。setup の本体（`run`）と、Skill が使う、`run` のコマンドの文と部屋のパスの組み立て（`begin`）、出来事の受け取り（`next`）、答えの書き込み（`answer`。人の許可を得て sandbox の外）。どの引数でも最初に依存を照らし（`entry/deps-check.ts`）、通ったら本体を `import()` で読み込む。処理は `flow/setup/` | 「[導入と点検][b04]」の「setup の流れ」「プログラムの写し」 | `05-setup.md` |
| `entry/deps-check.ts` | 入口 | 入れた Plugin の `plugin/package-lock.json` と `node_modules/<名前>/package.json` を読み、`logic/setup/manifest.ts` の照らしに渡す。`node:` で始まるモジュールと `logic/setup/manifest.ts` のほかを import しない。単体テストを持たない（下の「テストで外との境を差し替える方法」の表の下） | 「[導入と点検][b04]」の「依存」 | `05-setup.md` |
| `entry/check.ts` | 入口 | 人の `check` の入口。処理は `flow/skill/` | 「[導入と点検][b04]」の「点検だけで動かす仕組み」 | `04-check.md` |
| `entry/dev-autopilot.ts` | 入口 | `/dev-autopilot` の入口。処理は `flow/skill/` | 「[導入と点検][b04]」の「状態の表示」 | `08-flow.md` |
| `entry/check-probe.ts` | 入口 | 試しのスクリプト。試しのセッションの中で AI に 1 回だけ実行させ、読めたか・書けたかだけを出力する | 「[導入と点検][b04]」の「試しのセッション」 | `04-check.md` |
| `entry/intake.ts` | 入口 | intake の Skill の入口（`quarantine`・`draft`・`wait`。Skill の引数「検疫」「下書き」から呼ぶ）。処理は `flow/skill/` | 「[AI のセッション][b03]」の「人の Skill から受け入れを呼ぶ形」 | `06-text-and-intake.md` |
| `check.ts` | 流れ | 点検の正本。組の一覧と順番、組ごとの回数、結果の形（pass・warn・fail と、未確認の warn（`unverified`）） | 「[導入と点検][b04]」の「組と回数」 | `04-check.md` |
| `flow/check/` | 流れ | 点検の組ごとの処理（設定、GitHub、アプリの権限、マシン、子プロセスと clone、sandbox と AI のセッション） | 「[導入と点検][b04]」の「組と回数」「モデルを使わない試し」「試しのセッション」、「[設定][b01]」の「`check` が設定について確かめること」 | `04-check.md` |
| `flow/setup/` | 流れ | setup の手順（差分の一覧と承認、GitHub の準備と読み戻し、App の作成、マシンの準備、プログラムの写し、定期の実行の登録、動かすマシンを替えるとき） | 「[導入と点検][b04]」の「setup の流れ」から「動かすマシンを替えるとき」まで、「setup 用の App」「GitHub App を作る」 | `05-setup.md` |
| `flow/skill/` | 流れ | 人が呼ぶ `check`・`/dev-autopilot`・intake の Skill の処理。`check` は、依頼のファイルを置き、定期の実行と同じジョブを起こし、結果を待って表示し、消す（点検の処理は持たない）。`/dev-autopilot` は、`status.json` を読んで表示し、手動の 1 回実行は依頼のファイルを置いてジョブを起こす。intake は、引数「検疫」で検疫を動かして結果を出し、引数「下書き」で受け入れの依頼のファイルを置いて結果を待つ。3 つの Skill が共有する読み取りは `flow/skill/context.ts`（受け持ちとテストは `02-settings-and-records.md` の「ファイルの一覧」と「テスト」）に置き、`flow/run/` を import しない（intake の入口の範囲。下の「入口ごとの import の範囲」）。`run.lock` の中身は、`io/files.ts` で読み、`logic/run/lock.ts` の「lock の中身の形」で読む | 「[導入と点検][b04]」の「点検だけで動かす仕組み」「状態の表示」、「[AI のセッション][b03]」の「人の Skill から受け入れを呼ぶ形」 | `02-settings-and-records.md`、`04-check.md`、`06-text-and-intake.md`、`08-flow.md` |
| `flow/run/` | 流れ | 1 回の実行の順 1〜10、起こされ方と依頼のファイルの扱い、lock を取って外す、手元の記録と `status.json` を書く、見つかりの記録（`security-pending/`）のファイルを書く・読む・消す | 「[状態遷移と判定][b02]」の「1 回の実行の流れ」「手元の記録」、「[導入と点検][b04]」の「見送った実行と、起動しなかった定期の実行」 | `02-settings-and-records.md`、`08-flow.md` |
| `flow/steps/` | 流れ | 次の工程の表の工程ごとの処理（入口の検疫、下書き、決定、計画、変更の作成、レビュー、直し、レビュー合格の軽い工程、マージの判定、片付けなど） | 「[状態遷移と判定][b02]」の「次の工程」「計画の工程の中の順番」 | `08-flow.md` |
| `flow/push.ts` | 流れ | push の手順 | 「[状態遷移と判定][b02]」の「push の手順」 | `08-flow.md` |
| `flow/session/` | 流れ | AI のセッション（作業場所と短い一時フォルダと安全設定のファイルを作る、起動の前の確かめ、起動と監視、止める手順、前の実行が残したセッション、レビューの軸と批評 AI） | 「[AI のセッション][b03]」 | `07-sessions.md` |
| `flow/safety/` | 流れ | 全体の止める合図と全体の停止の Issue と見つかりの記録を読む（点検の試しと受け入れの依頼の実行の受付 AI の前は、作業の単位の Issue の印も読む）、攻撃の疑いにする（`where` が `text` なら、印より先に見つかりの記録を書く）、見つかりの記録から印を書き直す、全体の停止の Issue を開く、GitHub の外への通知。見つかりの記録のファイルの操作は `flow/run/` | 「[状態遷移と判定][b02]」の「全体の停止の Issue」「GitHub の外への通知」、「[設定][b01]」の「設定ファイルの外の合図」 | `08-flow.md` |
| `io/` | 外との境 | システムの型（流れに渡す `FlowSystem` の型を含む）、子プロセスの起動と時間の上限と出力の上限と止め方（出力の上限を超えたときの失敗の理由は `output-limit`）、GitHub の API（`@octokit/core` で REST と、名前を付けた GraphQL の操作）、JWT（`io/github-jwt.ts`）とインストールのトークン（点検のための、リポを絞らないトークンでの読み取り `readInstallationRepositories` を含む）、`git`（個人とシステムの設定と属性のファイルを読まない環境。`core.hooksPath` は置き場の空のフォルダ `empty-hooks/`）、`claude`（起動と `stream-json` の読み取り）、ファイル（権限、実体のパス、排他の作成、追記と書き出し、置き換え）、秘密の置き場を読む、秘密のファイルを置く（`io/secret-files.ts`）、HTTP（Webhook と、setup の手元の戻り先）、OS ごとの部分 | 「[設定][b01]」の「秘密の置き場のファイル」「子プロセスが時間の上限を過ぎたとき」、「[AI のセッション][b03]」の「起動の形」「止める手順」 | `01-io.md` |
| `marker.ts` | 判断 | 印の形の正本（種類と欄、組み立て、読み取り、AI の出力から印の形を取り除く規則） | 「[状態遷移と判定][b02]」の「印の種類」、要件の「[印と固定の見出しの一覧][markers]」 | `03-markers.md` |
| `logic/marker/` | 判断 | 印の署名（JSON の並べ方と HMAC-SHA256）、鍵のファイルの読み方、コメントの印の読み方（投稿者・未編集・印の数・形・署名） | 「[状態遷移と判定][b02]」の「印の署名」 | `03-markers.md` |
| `logic/setup/` | 判断 | setup の差分の一覧とハッシュ、dev-autopilot 自身のリポの見分け方、写しの中身の一覧、定期の実行の登録の中身、App のマニフェスト、版ごとの設定の表の控え（`settings-versions.ts`） | 「[導入と点検][b04]」の「setup の流れ」「プログラムの写し」「定期の実行の登録」「GitHub App を作る」 | `05-setup.md` |
| `logic/settings/` | 判断 | マシンの設定とリポの設定の形と確かめ、プログラムに固定する値（設定の項目に関わる値と、名指しのパス・環境変数の名前・`denyRead` の固定の分の一覧）、パスの照らし | 「[設定][b01]」 | `02-settings-and-records.md` |
| `logic/state/` | 判断 | GitHub の事実から作業の単位の状態を組み立て、次の工程を選び、進める順に並べる。AI のセッションの終わり方の記号を工程の結果に写す（`session-result.ts`） | 「[状態遷移と判定][b02]」の「状態の組み立て」「次の工程」「作業の単位を進める順」 | `08-flow.md` |
| `logic/judge/` | 判断 | レビュー合格とマージの判定、ラウンドと往復の数え方、同じ指摘の再発、必須チェックの読み方。マージの判定の `reasons` の記号の一覧（`merge-reasons.ts`。何も import しない） | 「[状態遷移と判定][b02]」の「判定」 | `08-flow.md` |
| `logic/push/` | 判断 | push の範囲の commit が触れたパスの照らしと止める理由、変更ファイルの一覧の重なり | 「[状態遷移と判定][b02]」の「push の手順」 | `08-flow.md` |
| `logic/run/` | 判断 | lock が古いかの判定、起こされ方の決定、起動しなかった定期の実行の時刻、実行の失敗が続いた回数、作業場所の残骸の見分け | 「[状態遷移と判定][b02]」の「1 回の実行の流れ」「作業場所の残骸」、「[導入と点検][b04]」の「見送った実行と、起動しなかった定期の実行」 | `02-settings-and-records.md`、`08-flow.md` |
| `logic/text/` | 判断 | Issue の文の組み立て（`issue-text.ts`）、検疫、漏えいの検査（git が中身を出さなかった差分を当てる `binary-diff` を含む）、見つかった種類の記号の一覧（`finding-kinds.ts`。何も import しない）、検査を通った文の 2 つの型、ハッシュ、コメントの組み立て、指摘の区切りの行（`dap-finding`）、出力契約のスキーマと取り出し、見出しの比べ方 | 「[AI のセッション][b03]」の「検疫が探す URL」「役ごとの出力契約」「コメントの形」「ハッシュの取り方」「レビューのコメントの形」 | `06-text-and-intake.md` |
| `logic/session/` | 判断 | セッションの名前、環境変数と安全設定と `claude` の引数の組み立て、費用の見積もりと数え方 | 「[AI のセッション][b03]」の「セッションの名前」「環境変数」「セッションごとの安全設定」「止め方と利用枠の数え方」 | `07-sessions.md` |
| `logic/check/` | 判断 | 点検の項目の ID と人に見せる決まった文（`items.ts`。点検、人の `check`、setup、`/dev-autopilot` の表示が共有する）、点検と setup の読み戻しが共有する比べる規則（ruleset の形、App の権限、Environment の規則の照らし（[FACT-101][facts]））、点検の結果の形、試しのセッションを省く判定と結果の決め方 | 「[導入と点検][b04]」の「ruleset の形と確かめ方」「GitHub App の権限」 | `04-check.md` |
| `logic/secret.ts` | 判断 | 秘密の値の型と、中身を取り出す関数（`revealSecret`）。何も import しない。`revealSecret` を import してよいファイルは、構成の検査の一覧で決める | 「[設定][b01]」の「秘密の置き場のファイル」 | `01-io.md` |

- `plugin/skills/review/scripts/per-commit-gates.ts` は `plugin/src/` の外に置き、`review` Skill と一緒に単体でも写る（[分離を見越した構成][struct]）。単体で写したときに `plugin/src/` が無いので、`plugin/src/` を import しない
- 検疫（`logic/text/` の検疫のモジュール）は、入口の検疫、人の見出しの検疫、AI に渡す直前、AI の文を GitHub に書く前、intake の Skill から呼ばれ（「[AI のセッション][b03]」の「検疫が探す URL」）、見つけたときの扱い（安全の軸の値、全体を止めるか）が呼ぶ側で違う（「[状態遷移と判定][b02]」の「計画の工程の中の順番」「push の手順」など）。検疫のモジュールは見つけたものを返すだけにし、扱いは呼ぶ側の流れが決める

## 入口ごとの import の範囲

入口ごとに、動く所と権限が違うので、import してはいけないものを決める。範囲は、入口のファイルからたどれる実行時の import（数え方は下の「構成の検査に足すこと」）のすべてで見る。

![入口ごとの import の範囲][fig-entries]

| 入口 | 動く所 | import してはいけないもの | 理由 |
| --- | --- | --- | --- |
| `main.ts` | 写し。定期の実行が sandbox の外で起こす | 無し | 進行 Bot の全体 |
| `entry/setup.ts` | 入れた Plugin。人の Claude Code（作業フォルダの外に書くコマンドは、人の許可を得て sandbox の外。`begin` と `next` は sandbox の中、`run` と `answer` は人の許可を得て sandbox の外） | 秘密の置き場を読むモジュール（`io/` の中）、`check.ts` と `flow/check/` | 秘密のファイルは作るか書くだけで、中身を読まない。点検は、人の `check` と同じくジョブを起こして進行 Bot に任せる（「[導入と点検][b04]」の「setup の流れ」の順 7） |
| `entry/check.ts` | 写し。人の Claude Code（依頼を置くコマンドと結果を消すコマンドだけ、人の許可を得て sandbox の外） | 秘密の置き場を読むモジュール、`check.ts` と `flow/check/`、GitHub の API と `claude` を呼ぶモジュール | 判定のロジックを持たず、点検は進行 Bot の 1 か所で行う（[STRUCT-3][struct]） |
| `entry/dev-autopilot.ts` | 写し。人の Claude Code | 秘密の置き場を読むモジュール、GitHub の API と `claude` を呼ぶモジュール、`flow/steps/` | `status.json` を読んで表示し、手動の 1 回実行は依頼を置いてジョブを起こすだけ |
| `entry/intake.ts` | 写し。人の Claude Code の sandbox の中（記録の置き場に書くコマンドだけ、人の許可を得て外） | 秘密の置き場を読むモジュール、GitHub の API と `claude` を呼ぶモジュール、lock を取るモジュール（`flow/run/` の中） | 引数「検疫」は、AI のセッションも秘密も使わない。引数「下書き」は、lock を取らず、`claude` を起こさない（[STRUCT-3][struct]、[ADR-20261022-004][adr-intake-req]） |
| `entry/check-probe.ts` | 試しのセッションのフォルダに写したもの。AI が sandbox の中で `node` で動かす | `node:` で始まるモジュールのほかすべて | 写した先に `plugin/src/` も `node_modules` も無い。秘密も GitHub も使わず、読めたか・書けたかだけを出す |

- 秘密の置き場を読むモジュールは、`io/` の中で 1 つに分ける。`setup` が空の秘密のファイルを作るときは、ファイルの操作のモジュール（`io/files.ts`）を使い、秘密鍵を置くときは `io/secret-files.ts`（秘密を書くだけで、読まない）を使う。どちらも秘密を読むモジュールを使わない。秘密鍵の中身を書くのは `io/secret-files.ts` だけ
- 秘密の値の型（`logic/secret.ts`）は、何も import しない判断の層のファイルに置く。入口と判断の層と外との境が型を使っても、秘密の置き場を読むモジュールに届かない（「[外との境][d01]」の「秘密の置き場」）
- 3 つの人の Skill が共有する読み取り（上の「フォルダとモジュールの一覧」の `flow/skill/` の行）は `flow/skill/context.ts` に置き、lock を取るモジュール（`flow/run/` の中）を import しない
- 検疫のモジュールは判断の層なので、秘密・GitHub・記録・lock に依存しない。当てる先のリポの owner と名前は引数で受ける（「[AI のセッション][b03]」の「人の Skill から受け入れを呼ぶ形」）
- テストは、`check`・`/dev-autopilot`・`intake` の入口を `node --permission` で起こし、秘密の置き場の読み込みを許さない（「[テスト][b05]」の「偽物」）。import の範囲の確かめは、秘密の置き場を読むコードが入口から届かないことを、動かす前に構成の検査で確かめる

## テストで外との境を差し替える方法

層ごとに、単体テストで差し替えるものを決める。結合テストは、テストのためだけの切り替えを持たない（「[テスト][b05]」の「偽物」）。

![テストで外との境を差し替える方法][fig-tests]

| テスト | 差し替えるもの | 差し替え方 |
| --- | --- | --- |
| 判断の層の単体テスト | 無し | 値を渡して、返った結果を比べる |
| 流れの層の単体テスト | 外との境と `FlowSystem` | 外との境の型に合う偽物を、テストの中で作って引数で渡す。偽物は、呼ばれた順と引数を記録し、筋書きで決めた答えを返す。時刻と待ちと乱数は、`FlowSystem` の偽物（偽の時刻と待ち、決まった乱数）を引数で渡す |
| 外との境の層の単体テスト | システム | システムの型に合う偽物（偽の子プロセス、偽の時刻と待ち、決まった乱数、テストの一時フォルダの中だけを指すファイルの操作、受けた要求を記録して決めた答えを返す `fetch`、テストが要求を送り込める `listen`、送ったシグナルを記録して決めた結果を返す `kill`、決めた値の `env`・`platform`・`uid`・`pid`・`hostname`）を引数で渡す。偽物は、本物と同じ `code` の失敗を投げる（「[外との境][d01]」の「システム」）。子プロセスが時間の上限を過ぎたときの扱いと止める猶予を、偽の時刻で確かめる |
| 入口の単体テスト | 無し | 引数の読み方を確かめる。`entry/check-probe.ts` は、外に触れる関数の偽物を渡して試しの処理も確かめる。システムの実体を作る `entry/system.ts` は単体テストを持たず、結合テストで確かめる。`entry/system.ts` の `listen` は、結合テストの中の小さなテスト（`entry/system.ts` でシステムを作って `listen` で待ち、テストの側から `http://127.0.0.1:<ポート>/` と `http://[::1]:<ポート>/` に送って、両方が届くこと）で確かめる |
| 結合テスト | 外のコマンド | 入口を子プロセスで起こし、本物のシステムで動かす。`claude`・`ssh`・`launchctl`・`systemctl` を `PATH` の先頭の偽物にし、GitHub の API を、proxy の環境変数とテスト用の CA で偽の GitHub に向ける（FACT-123）。`git` を、呼ばれた引数と作業フォルダを記録して本物に渡す包み（「[テスト][b05]」の「偽物」の git の記録の包み）にし、AI のセッションを起こした後の clone で、取り込みの fetch のほかに `git` を実行しないことを確かめる（[ADR-20261013-001][adr-workspace] の確認方法）。時刻は差し替えない（「[テスト][b05]」の「偽物」） |

- 単体テストを持たないファイル（`entry/system.ts`、`entry/deps-check.ts`、実行時の値を export せず型だけを書いたファイル（今は `io/system.ts` と `logic/state/facts.ts`））は、理由を添えて `test/repo/no-unit-test.json` に書く（「[テスト][b05]」の「単体テストを持たないファイル」）。`entry/deps-check.ts` は 2 つのファイルを読んで渡すだけで、照らしは `logic/setup/manifest.ts` の単体テストで、読む処理は `setup` の結合テストで確かめる。型だけのファイルに関数か値を足したら、一覧から外して単体テストを足す
- GitHub の外への通知の送り先（Webhook）の偽物は、基本設計の「[テスト][b05]」の偽物の表に無い。置き場と、URL が `https://` で始まることを点検が確かめる決まり（「[設定][b01]」の「`check` が設定について確かめること」）との合わせ方は、`01-io.md` で決める

## 構成の検査に足すこと

基本設計の「[テスト][b05]」の構成の検査（`plugin/src/` のファイルごとの単体テスト、`plugin/` の中にテストが無いこと、import が `node:` で始まるもの・相対パス・`plugin/package.json` の依存のどれかであること、`plugin/package.json` の依存の形）に、次を足す。どれも外れたら構成の検査を落とす。使ってよいものと使ってはいけないものの一覧は、構成の検査の 1 か所に持つ。

| 確かめること | 理由 |
| --- | --- |
| 入口のほかの層が import してよい `node:` のモジュールは、`node:path`（文字列の操作だけ）と `node:crypto` の決めた関数（`createHash`・`createHmac`・`timingSafeEqual`・`createPrivateKey`・`sign`）だけであること。ほかの `node:` のモジュールと、`node:crypto` のほかの関数（`randomBytes`・`randomUUID`・`randomInt`・`randomFill`・`generateKeyPair` などを含む）の import を落とす | 単体テストで、外に触れるものをすべて偽物にするため（上の「層と依存の向き」）。許す一覧にして、一覧に無いものが黙って通らないようにするため |
| 入口のほかの層で、次の全体の名前を使わないこと：`process`、`globalThis`、`require`、`Date.now`、`Date` を関数として呼ぶこと、引数の無い `new Date`、タイマー（`setTimeout`・`setInterval`・`setImmediate`）、`fetch`、`WebSocket`、`EventSource`、`Math.random`、全体の `crypto`（Web Crypto）、`performance`、`AbortSignal.timeout`、`navigator` | import の無いまま外に触れる機能を止めるため。`globalThis` を止めるのは、名前を組み立てて呼ぶ書き方で一覧を外れないため |
| 判断の層が判断の層だけを、外との境の層が外との境の層と判断の層だけを import していること | 判断の層の結果が、渡された値だけで決まるようにするため |
| 流れの層が、流れの層と判断の層だけを import し、外との境の層は `import type` の文だけで import していること（入口の層は import しない。`import { type … }` は実行時の import に数える） | 外との境とシステムの実体を、流れの中で作れなくするため |
| 入口からたどれる実行時の import（`import()` の先を含む）に、上の「入口ごとの import の範囲」の import してはいけないものが無いこと | 秘密の置き場を読むコードと点検の処理が、人の Skill の入口から届かないようにするため |
| `entry/deps-check.ts` からたどれる import が、`node:` で始まるものと `plugin/src/` の相対パスだけで、`plugin/package.json` の依存に届かないこと | 依存が無いときにも照らしが動き、止まる理由を示すため（「[導入と点検][b04]」の「依存」） |
| `logic/secret.ts` の `revealSecret` を import するのが、構成の検査の一覧のファイルだけであること。`logic/secret.ts` のものを export しなおさないこと | 秘密の中身を取り出す所を、決めたファイルに限るため（「[外との境][d01]」の「秘密の置き場」） |
| `plugin/src/` の中の実行時の import に輪が無いこと | ES モジュールは輪の中で先に入ったほうを後で評価するので、入口ごとに import の順が変わり、ある入口だけが起動の時に初期化の前の参照で止まるため |
| `plugin/skills/` の TypeScript が `plugin/src/` を import していないこと | `review` Skill を単体で写したときにも動くようにするため |
| `plugin/package.json` の依存が、構成の検査の一覧で依存ごとに決めた層のほかで import されていないこと。一覧に無い依存の import も落とす | どの層で使うかを、依存を足す PR のレビューで決めるため。外に触れる依存は外との境の層だけで使い、入口が作るシステムの `fetch` を渡して呼ぶ（依存が自分で外に触れると、単体テストで偽物にできないため）。今の一覧は、`@octokit/core` を外との境の層だけに許す（[ADR-20261107-001][adr-github]） |

- 構成の検査は、実行時に読み込まれる import を数える。`import type …` の文と `export type … from` の文だけを型の import とする。名前ごとに `type` を付けた `import { type … }` は、Node が読み込んで中のコードを動かすので、実行時の import に数える
- 入口ごとの import の範囲と import の輪は、実行時の import と、`export … from` と、文字列で書いた動的な `import()` をたどって見る。型の import は、読み込まれないので範囲に数えない
- 動的な `import()` は、`entry/setup.ts` の中で、引数を文字列の定数で書いたときだけ使ってよい（上の「層と依存の向き」）。構成の検査は、ほかの所の `import()` と、文字列でない `import()` を落とす
- `revealSecret` を import してよいファイルの今の一覧は、`io/github.ts`、`io/github-jwt.ts`、`io/http.ts`、`io/secret-files.ts`、`logic/marker/keys.ts`、`logic/marker/sign.ts`、`logic/text/leak.ts`（「[外との境][d01]」の「秘密の置き場」）
- `fixtures/repo-layout/` に、流れが外との境を `import { type … }` で読む木、流れが入口を import する木、`entry/setup.ts` のほかで `import()` を使う木、文字列でない `import()` を使う木、import の輪がある木、上の止める全体の名前と許す一覧に無い `node:` のモジュールと関数を 1 つずつ使った木を置き、どれも落ちることを確かめる

## TypeScript の書き方

進行 Bot は、Node 24 が型を取り除いて直接動かす（ビルドしない。[ADR-20261017-003][adr-plugin]）。型を取り除くだけで動く書き方に限り、外れた書き方を型の検査で止める。

| 決まり | 理由 | 型の検査で止める設定 |
| --- | --- | --- |
| 相対の import は、拡張子の `.ts` まで書く | 書かないと、起動の時に止まる（[FACT-103][facts]） | `module` と `moduleResolution` を `nodenext` にし、`allowImportingTsExtensions` を付ける |
| `enum` やコンストラクタの引数に書くプロパティなど、型を取り除くだけでは消えない書き方を使わない | 起動の時に止まる（[FACT-104][facts]） | `erasableSyntaxOnly` |
| 型だけの名前は、`import type` か、名前に `type` を付けて import する。流れの層が外との境の型を読むときは、`import type` の文に限る（上の「構成の検査に足すこと」） | 付けないと、起動の時に止まる（[FACT-105][facts]）。名前ごとに `type` を付けた文は、型しか使わなくてもモジュールを読み込む | `verbatimModuleSyntax` |

- `plugin/package.json` は、実行時の依存を足す PR で作り、`"type": "module"` を書く（「[導入と点検][b04]」の「依存」）。`plugin/package.json` が無いうちも、import と export を持つ `.ts` は ES モジュールとして動く（[FACT-106][facts]）
- 依存は、`plugin/package.json` に書いた名前で import し、依存のパッケージが持つ型を使う。型の検査は `plugin/node_modules` の型を読むので、型の検査の前に `plugin/` で依存を入れる（「[テスト][b05]」の「回し方」）
- 上の設定で止まることは、TypeScript 5.9.3 で確かめた（[FACT-103〜105][facts]）。`tsconfig.json` は、コードを入れる PR で足す（[STRUCT-4][struct]）

[design]: ../README.md
[diagrams]: ../../requirements/diagrams/README.md
[b01]: ../basic/01-settings.md
[b02]: ../basic/02-flow-and-judgment.md
[b03]: ../basic/03-ai-sessions.md
[b04]: ../basic/04-setup-and-check.md
[b05]: ../basic/05-tests.md
[struct]: ../../requirements/07-structure.md
[markers]: ../../requirements/04-external-text-and-markers.md
[facts]: ../../requirements/10-verified-facts.md
[adr-plugin]: ../../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-intake-req]: ../../adr/adr-20261022-004-intake-skill-via-request-file.md
[adr-workspace]: ../../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[d01]: ./01-io.md
[d02]: ./02-settings-and-records.md
[d03]: ./03-markers.md
[d04]: ./04-check.md
[d05]: ./05-setup.md
[d06]: ./06-text-and-intake.md
[d07]: ./07-sessions.md
[d08]: ./08-flow.md
[fig-layers]: ./diagrams/00-layers.drawio.svg
[fig-entries]: ./diagrams/00-entries.drawio.svg
[fig-tests]: ./diagrams/00-tests.drawio.svg
[adr-github]: ../../adr/adr-20261107-001-call-github-with-octokit-instead-of-gh.md
