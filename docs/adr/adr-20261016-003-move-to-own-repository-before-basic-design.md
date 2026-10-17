---
status: accepted
created: 2026-10-16
updated: 2026-10-17
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# dev-autopilot を基本設計の前に別のリポジトリへ移し、ほかのリポへの適用と配布は段階 8 の後に決める

dev-autopilot を、business-os の中のフォルダ（`dev-autopilot/`）から別のリポジトリへ移す時期を決める。要件の「[分離の時期の目安][structure]」（段階 8 の後に分離する）を改める。
business-os の ADR 20261008-01（[`dev-autopilot/` を business-os に置く橋渡しの ADR][bos-adr]）は、決定 7 で分離の時期と方法を dev-autopilot 側に任せているので、dev-autopilot 側の ADR で決める。
要件の UNDECIDED-42（`review` Skill の評価の置き場。[決めていないこと][undecided]の「基本設計の中で決めてよいもの」）も、同じ決定で答えが出る。

## 背景と課題（Context and Problem Statement）

要件の「分離の時期の目安」は、段階 8（進行ボットがマージの判定の印を書く段階）まで business-os で動かし、設定ファイルの項目だけで別のリポに適用できると確かめてから分離するとしている。
理由は「2 つ目の適用先ができる前に分離すると、固有の値がコードに残っていても気づけない」。business-os の ADR 20261008-01 も、最初から別のリポにする案を、同じ理由と「開発の初期に 2 つのリポを行き来する手間」で退けた。

2026-10-16 の時点の状態は次のとおり。

- `dev-autopilot/` の中身は `docs/`（要件・ADR・図）と `.markdownlint-cli2.jsonc` だけ。進行ボットのコード、pnpm の workspace のパッケージ、business-os の `check:dev-autopilot` はまだ無い（`pnpm-workspace.yaml` は単一パッケージのまま）
- dev-autopilot の文書から business-os のファイルへの相対リンクは無い。business-os 側で dev-autopilot に触れているのは、CLAUDE.md の 2 か所、構造仕様（`docs/design/architecture.md`）の 3 か所、business-os の ADR 3 本（20261008-01・20261009-01・20261009-02）と ADR の一覧
- 2026-10-08 から 2026-10-16 までの `develop` の commit 38 件のうち、27 件が dev-autopilot の文書の変更（題が `docs(dev-autopilot)` のもの）
- 要件の通し読みが済み、残る判断は「ADR で決めること」の 9 件（[決めていないこと][undecided]）。基本設計（構造仕様。`dev-autopilot/docs/design/`）はまだ書いていない

business-os の中に置いたままにすると、3 つの問題がある。

- 要件と基本設計が、置き場所の 2 通りの形を持つ。STRUCT-3（起動の仕方）・STRUCT-4（検査）・STRUCT-5（文書）・STRUCT-7（Skill の配り方）は、「分離前」と「分離後」を書き分けている。
  基本設計は、起動のパス、検査のつなぎ方、評価の置き場を書くので、2 通りを書くか、分離のときに書き直すことになる
- 進行ボットのコードと、AI のセッションに読ませるエージェント定義と Skill（`plugin/agents/` と `plugin/skills/`）が、AI が作業するリポの中にある。今は、IMPL-15（漏えいの検査と push）の push を止めるパスと、MERGE-4（マージを判定してよいパスの許可一覧）の「含めない」例（自己改変）に `dev-autopilot/**` を入れて止めている。
  IMPL-15 の一覧は設定に持つので、設定を誤ると、作業 AI が書き換えた進行ボットのコードが PR に載り、人のレビューだけが守りになる
- business-os の ADR 20261008-01 の決定 5 は、dev-autopilot を business-os の pnpm の workspace のパッケージにし、依存を business-os の root と共有するとしている。dev-autopilot が business-os の root の依存や設定に頼っていても、別のリポに移すまで気づけない

「分離の時期の目安」の理由は、置き場所ではなく、当てる先の数の話として読める。固有の値に気づけるのは 2 つ目のリポに当てたときで、dev-autopilot のフォルダがどのリポにあるかでは変わらない。置き場所を移しても、段階 8 までは business-os だけに当てられる。

課題：dev-autopilot を別のリポへ移すのを、基本設計の前に行うか、今の目安どおり段階 8 の後まで待つか。

## 判断の決め手（Decision Drivers）

- 基本設計を、置き場所の 1 通りの形で書けること（書き直しを生まないこと）
- 進行ボットのコードを AI が書き換える経路を、設定の一覧ではなく置き場所で無くすこと。一覧の設定を誤っても、安全側に倒れること
- 「ほかのリポに当てても固有の値が残らない」ことを、2 つ目の適用先で確かめる機会を残すこと
- 移す手間と、移した後にリポをまたぐ手間が小さいこと
- 文書の参照（PR の番号、ファイルのパス）が、移した後も正しく読めること

## 検討した選択肢（Considered Options）

- 案 A：基本設計の前に移し、ほかのリポへの適用と配布は段階 8 の後に決める
- 案 B：今の目安どおり、段階 8 の後に移す
- 案 C：基本設計は business-os の中で書き、段階 3（`setup` と `check` を作る段階）の実装を始める前に移す

## 決定（Decision Outcome）

採用した選択肢:「案 A」。
理由: 文書しか無い今が、移す手間が最も小さい。基本設計を移した後の 1 通りの形で書け、進行ボットのコードを AI が書き換える経路が置き場所で無くなる。
2 つ目の適用先で固有の値を確かめる機会は、今の目安と同じ段階 8 の後に残る。

### 決定 1：基本設計の前に、別のリポジトリ `Joymerrevent/dev-autopilot` へ移す

- 移すもの：`dev-autopilot/` の中身すべて。フォルダの中身を新しいリポの root に置く（例：`dev-autopilot/docs/requirements/` → `docs/requirements/`）
- リポの名前は、Plugin の名前（`dev-autopilot`。STRUCT-3 の `dev-autopilot:worker`）と同じにする。owner は business-os と同じ `Joymerrevent`
- 公開の範囲は public にする。文書は今も public の business-os にあり、business-os の履歴にも残るので、private にしても隠せるものが無い
- 履歴は引き継がない。新しいリポの最初の commit は、business-os の `dev-autopilot/` の中身をそのまま写したものにし、本文に写した元の business-os の commit の SHA を書く。
  business-os の squash の commit の題には PR の番号（`(#120)` など）が付いていて、GitHub の commit の画面では、題の `#<番号>` が同じリポの Issue と PR へのリンクになる。履歴を引き継ぐと、移した先では別の Issue と PR を指す。
  移す前の経緯は business-os の履歴で読む（business-os の公開済みの履歴は書き換えない。business-os の CLAUDE.md の絶対ルール 6）
- 写した後は、business-os の `dev-autopilot/` を変更しない。消すまでの間に、同じ文書を 2 か所で直さないため
- 文書の参照を直す
  - business-os の PR と Issue の番号（2026-10-16 に数えて 9 ファイル・30 か所）は、business-os のものと分かる書き方にし、PR と Issue の URL へのリンクにする
  - `dev-autopilot/` で始まるパスは、新しいリポの root からのパスにする。承認済みの ADR は本文を書き換えず、[ADR の README][adr-readme] の決まりどおり、末尾の「パスの注記」に今の場所を書く
  - business-os のもの（`/verify-issue`、business-os の ADR、`pnpm eval:cases` など）を指す文は、business-os のものと分かる書き方にする

### 決定 2：ほかのリポへの適用と配布は、段階 8 の後に決める

- 「分離」を、置き場所を移すこと（決定 1）と、ほかのリポに当てること・ほかの人に配ること（marketplace への登録、skills CLI での `review` の単体導入。STRUCT-7）に分ける。
  後者の時期は今の目安のまま、段階 8 の出口を越え、設定ファイルの項目だけで別のリポに当てられると確かめてから決める
- 進行ボットが当てる先は、段階 8 の後に決めるまで business-os だけにする
- dev-autopilot を、dev-autopilot 自身のリポに当てない。当てると、決定 1 で無くした経路（作業 AI が進行ボットのコードを書き換える）が戻る。当てるなら、別の ADR で決める
- 進行ボットは、Plugin を marketplace から入れず、dev-autopilot のリポの clone の `plugin/` を `--plugin-dir` で読み込む。進行ボット（`src/`）と、AI のセッションに読ませるエージェント定義と Skill（`plugin/`）を、同じ clone の同じ commit から使い、版がずれないようにする（STRUCT-7 で同封した理由と同じ）。
  STRUCT-3 の「分離したら `--plugin-dir` が Marketplace からの導入に変わる」は、ほかの人に配るときに決めることに改める

### 決定 3：進行ボットのプログラムの clone と、business-os の clone を分ける

- 今の要件では、「進行ボットの clone」（SAFE-16。[ADR-20261013-001][adr-workspace] の `<置き場>/controller/`）が 2 つの役を兼ねる。
  1 つは進行ボットのプログラム（`dev-autopilot/src/` と `dev-autopilot/plugin/`）を置いて実行する役、もう 1 つは business-os に対する git の操作（`origin/develop` の fetch、AI の commit の取り込み、merge、push）を行う役
- 移した後は、2 つの役を別の clone にする
  - `<置き場>/controller/`：business-os の clone。business-os に対する git の操作に使う（ADR-20261013-001 のまま）
  - 進行ボットのプログラムの clone：dev-autopilot のリポの clone。マシンの設定の `devAutopilotPath`（STRUCT-3、[ADR-20261015-005][adr-settings]）が指す。launchd のジョブは、プログラムの clone の `src/` を実行する
- どちらの clone も、進行ボットだけが書き、AI のセッションからは読めず書けない（SAFE-16、[ADR-20261015-001][adr-read]）。
  ADR-20261015-001 が重ねる守りとして `deny` の Read に書いた `<置き場>/controller/` に、プログラムの clone も足す。プログラムの clone の置き場の初期値は、構造仕様の設定の表で決める（ADR-20261015-005 の決定 1 と同じ扱い）
- UNDECIDED-36（AI のセッションが Plugin を読めるか）は、Plugin の場所が `<置き場>/controller/` の中からプログラムの clone の中に変わるだけで、論点は残る

### 決定 4：検査・評価・依存は dev-autopilot のリポが持つ

- 衛生の検査（書式、lint、漏えい、シェル、Markdown）と CI、commit の規約の検査は、project-recipes のレシピで dev-autopilot のリポに入れる（STRUCT-4 が分離の後の作業として書いたもの）。
  写した文書を直す PR より先に、検査と CI を入れる（business-os の検査の網から外れた文書を、検査なしで直さないため）
- 依存（`package.json` とロックファイル）は dev-autopilot のリポが持ち、business-os と共有しない。business-os の ADR 20261008-01 の決定 5（pnpm の workspace のパッケージにし、business-os の `pnpm check` から `check:dev-autopilot` で呼ぶ）は行わない
- `review` Skill の評価は、dev-autopilot のリポに置く（UNDECIDED-42 の答え）。
  business-os の `pnpm eval:cases` は、business-os の `plugin/` と `evals/` と `fixtures/` を一時フォルダに集めて Plugin を組み立てる仕組みで、別のリポの Skill を対象にするには、business-os が dev-autopilot のファイルを取り込むことになる（STRUCT-1 の「互いに import しない」に反する）。
  dev-autopilot の側にも、評価のケースと Skill を集めて Plugin を組み立てる入口を作る。business-os の仕組みの考え方は写してよいが、コードは import しない

### 決定 5：business-os 側の片付けと、リポをまたぐ結びつき

- business-os 側は、business-os の ADR 20261008-01 の決定 7 のとおり、ADR 20261008-01 を「分離した」ADR で置き換え、`dev-autopilot/` を消し、CLAUDE.md と構造仕様の dev-autopilot の行を消す。business-os の ADR の規則に従い、business-os の PR で行う
- IMPL-15 の push を止めるパスの初期値にある `dev-autopilot/**` は、business-os から `dev-autopilot/` を消した後も残す。
  外すのは安全の境界を緩める変更（STRUCT-2）にあたり、残しておけば、同じ名前のフォルダが business-os に戻ったときにも止まる。MERGE-4 は許可一覧なので、`dev-autopilot/**` は何もしなくても `merge=human` のまま
- business-os の `/verify-issue` から dev-autopilot の検疫と `dev-autopilot:intake` を呼ぶ形（STRUCT-6、UNDECIDED-43）は、ここでは決めない。
  移した後はリポをまたぐ呼び出しになるが、選択肢（business-os の開発の環境に dev-autopilot の Plugin を入れて Skill を呼ぶ、`devAutopilotPath` の `src/` のコマンドを実行する）は変わらないので、基本設計で決める
- dev-autopilot 自身の作業（要件、ADR、実装）の Issue と PR は、dev-autopilot のリポに置く。進行ボットが処理する対象（business-os の Issue と Project）は変わらない

### 決定 6：進める順

1. ADR の承認
2. 新しいリポを作り、`dev-autopilot/` の中身を写す（リポの作成は、メンテナに確かめてから行う）
3. 新しいリポに、検査と CI を入れる（決定 4）
4. 新しいリポで、文書の参照を直し（決定 1）、要件と図に決定を反映する（補足情報の一覧）
5. business-os で、ADR 20261008-01 を置き換える ADR を起票し、承認の後に `dev-autopilot/` を消す（決定 5）
6. 残っている「ADR で決めること」（UNDECIDED-26・30〜34・36〜38）を新しいリポで決め、その後に基本設計を書く

### 影響（Consequences）

- 良い点: 基本設計を、置き場所の 1 通りの形で書ける。要件の「分離前」と「分離後」の書き分け（STRUCT-3・4・5・7）が 1 通りになる
- 良い点: 進行ボットのコードと、AI のセッションに読ませるエージェント定義と Skill が、AI が作業する business-os から無くなる。IMPL-15 の一覧の設定を誤っても、作業 AI は進行ボットのコードもエージェント定義と Skill も書き換えられない
- 良い点: 依存と検査の設定を business-os と共有しないので、business-os に頼っている箇所が、移した時点で見つかる
- 良い点: business-os の履歴・PR の一覧・CI が、business-os の変更だけになる。CLAUDE.md の絶対ルール 3 の例外と、ADR の規則が 1 つのリポに 2 組並ぶ状態が無くなる
- 悪い点: 新しいリポに、検査・CI・commit の規約・依存の設定を入れる手間がかかる。入れるまでの間、文書は検査の網の外にある（決定 4 と決定 6 の順で、検査を入れる前に文書を直さない）
- 悪い点: `review` Skill の評価の入口（評価のケースと Skill を集めて Plugin を組み立てる仕組み）を、dev-autopilot の側にも作る
- 悪い点: 段階 3 以降、進行ボットの変更（dev-autopilot のリポ）と、business-os 側の変更（リポの設定 `.claude/dev-autopilot.json` など）が、別のリポの PR になる。`/verify-issue` からの呼び出しも、リポをまたぐ
- 悪い点: 新しいリポの `git log` と `git blame` は、写した commit から始まる。それより前の経緯は business-os の履歴で読む
- 中立: 文書の公開の範囲は変わらない（今も public の business-os にある）
- 中立: 進行ボットが当てる先は、段階 8 の後まで business-os のまま

### 確認方法（Confirmation）

- 新しいリポで、次を確かめる
  - 最初の commit の本文に、写した元の business-os の commit の SHA がある。最初の commit の中身が、その commit の `dev-autopilot/` の中身と一致する（`diff -r` で比べる）
  - 検査と CI を入れた後のすべての PR で、CI の検査が終了コード 0 で通る
  - Markdown のファイルを `dev-autopilot/` で grep すると、残るのは承認済みの ADR の本文（パスの注記で今の場所を書いたもの）と、business-os のパスを指す文だけ
  - Markdown のファイルを `#<数字>` で grep すると、見つかる PR と Issue の番号に、どのリポのものかが分かる書き方とリンクがある
- business-os で、置き換えの ADR が承認された後に次を確かめる
  - `dev-autopilot/` が無く、CLAUDE.md と `docs/design/architecture.md` に dev-autopilot の行が無い。`pnpm check` が終了コード 0
- 段階 3（`setup` と `check` を作る段階）の `check`（SETUP-2）で、次を試す
  - 進行ボットのプログラムの clone と `<置き場>/controller/` が別のフォルダで、どちらも sandbox の中から読めず書けないこと

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：基本設計の前に移し、ほかのリポへの適用と配布は段階 8 の後に決める

- 良い点: 文書しか無いので、移す手間が最も小さい。写すのはフォルダ 1 つで、直す参照は番号の 30 か所とパス
- 良い点: 基本設計を 1 通りの形で書ける
- 良い点: 進行ボットのコードを AI が書き換える経路が、置き場所で無くなる
- 中立: 2 つ目の適用先で固有の値を確かめる機会は、今の目安と同じ段階 8 の後に残る
- 悪い点: 基本設計に入る前に、新しいリポの検査と CI を整える手間が入る

### 案 B：今の目安どおり、段階 8 の後に移す

- 良い点: 段階 8 まで 1 つのリポで済み、business-os の `pnpm check` 1 回で dev-autopilot も検査される（business-os の ADR 20261008-01 の決定 5）
- 悪い点: 基本設計と実装が「分離前」の形で作られる。段階 8 までに進行ボット・Skill・評価・結合テストができているので、移すときにコード・前提データ・検査のつなぎ方・評価の置き場を一緒に直す
- 悪い点: 段階 8 まで、進行ボットのコードの守りが IMPL-15 の一覧の設定に頼る
- 悪い点: business-os の履歴と CI に、dev-autopilot の変更が混ざり続ける

### 案 C：基本設計は business-os の中で書き、段階 3 の実装の前に移す

- 良い点: 移す作業を待たずに基本設計に入れる
- 良い点: コードを書く前に移すので、移す手間は案 A に近い
- 悪い点: 基本設計を「分離前」の形で書き、移すときに起動のパス・検査のつなぎ方・評価の置き場の記述を書き直す。案 A との違いは移す時期だけで、書き直しの分だけ手間が大きい

## 補足情報（More Information）

- 関連する要件：「[分離を見越した構成][structure]」の STRUCT-1〜7 と「分離の時期の目安」、「[決まっていること][overview]」の置き場所の行、SETUP-2・SETUP-4・SETUP-11（[06-setup-and-stages.md][setup]）、
  IMPL-15（[02-implement-review-fix.md][impl]）、MERGE-4（[03-merge.md][merge]）、SAFE-16（[05-safety.md][safety]）、「[レビューの Skill][review]」の評価の置き場、UNDECIDED-36・42・43・45（[決めていないこと][undecided]）
- 関連する ADR：business-os の ADR 20261008-01（[`dev-autopilot/` を business-os に置く橋渡しの ADR][bos-adr]）、[ADR-20261013-001][adr-workspace]（AI の作業場所と進行役の clone）、
  [ADR-20261015-001][adr-read]（AI の読み取りの範囲と秘密の置き場）、[ADR-20261015-005][adr-settings]（設定の値とマシンの設定）
- 範囲外
  - 新しいリポのブランチの運用（`develop` を置くか）とリリースの手順。移す作業の中で、project-recipes のレシピに合わせて決める
  - 進行ボットのプログラムの clone を新しい版に更新する手順（基本設計）
  - `/verify-issue` からの呼び方（UNDECIDED-43）と、Plugin を AI のセッションが読めるか（UNDECIDED-36）
  - 2 つ目の適用先の選び方（段階 8 の後）
- UNDECIDED-45（skills CLI が `plugin/skills/` の下の Skill を単体で入れられるか）は、新しいリポができると、そのリポに対して確かめられる
- 選んでいない形：履歴を引き継ぐ（`git filter-repo --subdirectory-filter dev-autopilot` など）。`git blame` が続く一方、commit の題の PR の番号が、移した先の別の Issue と PR を指す
- 要件と ADR への反映（承認の後に、新しいリポの PR で行う。ファイルの名前は移す前のもの）
  - 07-structure.md：「分離の時期の目安」を決定 2 の形に改める。構成の図の root を新しいリポの root にする。
    STRUCT-3 の `--plugin-dir` をプログラムの clone の `plugin/` に、「分離したら Marketplace からの導入」をほかの人に配るときに決めることに改め、`devAutopilotPath` の「分離前」の記述を外す。
    STRUCT-4 を、自分のリポの検査と依存の形に改める（workspace のパッケージと `check:dev-autopilot` を外す）。STRUCT-5 と STRUCT-7 の「分離前」と「分離後」の書き分けを 1 通りにする
  - 00-overview.md：「決まっていること」の置き場所の行と、エージェントの行のパス
  - 05-safety.md：SAFE-16 の「進行ボットの clone」を、プログラムの clone と business-os の clone の 2 つに分ける（決定 3）
  - 06-setup-and-stages.md：SETUP-4 の「分離後」の書き方を直す。SETUP-11 に、移した旨を書き添える。SETUP-2 の点検に、決定 3 の 2 つの clone を試す項目を足す
  - 02-implement-review-fix.md：IMPL-15 の `dev-autopilot/**` に、business-os から消した後も残す理由を書き添える
  - 08-review-skill.md：評価の置き場を、dev-autopilot のリポの評価の仕組みに改める
  - 09-undecided.md：UNDECIDED-42 を欠番にする（元の文を取り消し線で残す）。UNDECIDED-36 の Plugin の場所を、プログラムの clone の中に直す
  - [図][diagrams]：「全体の構成」「隔離と鍵」「フォルダ構成と起動」の進行ボットの clone を、決定 3 の 2 つに分ける。「フォルダ構成と起動」の root を新しいリポの root にする
  - [ADR-20261015-001][adr-read]：重ねる守りの `deny` の Read にプログラムの clone を足した旨の注記（Amended by）を、タイトルの下に足す
  - [ADR-20261015-005][adr-settings] と [ADR 一覧][adr-index]：本文の `dev-autopilot/` のパスに、パスの注記を足す
  - business-os 側（business-os の PR）：ADR 20261008-01 を置き換える ADR、CLAUDE.md、`docs/design/architecture.md`、business-os の ADR の一覧

## パスの注記

- `dev-autopilot/docs/design/` → `docs/design/`、`dev-autopilot/src/` → `src/`、`dev-autopilot/plugin/` → `plugin/`（2026-10-17・決定 1 の移動による）。
  本文のほかの `dev-autopilot/` は、business-os の中のフォルダを指すので、business-os のパスのまま
- ADR 一覧（`docs/adr/index.md`）の `dev-autopilot/` は、business-os の commit に固定したリンクの中だけにあり、注記は要らなかった

[structure]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[overview]: ../requirements/00-overview.md
[setup]: ../requirements/06-setup-and-stages.md
[impl]: ../requirements/02-implement-review-fix.md
[merge]: ../requirements/03-merge.md
[safety]: ../requirements/05-safety.md
[review]: ../requirements/08-review-skill.md
[diagrams]: ../diagrams/README.md
[adr-readme]: ./README.md
[adr-index]: ./index.md
[adr-workspace]: ./adr-20261013-001-fresh-clone-per-ai-session.md
[adr-read]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[adr-settings]: ./adr-20261015-005-settings-values-and-machine-settings.md
[bos-adr]: https://github.com/Joymerrevent/business-os/blob/9d37a83bd3b44b059ad3478ce8797254d4c45e72/docs/adr/20261008-01-dev-autopilot-folder.md
