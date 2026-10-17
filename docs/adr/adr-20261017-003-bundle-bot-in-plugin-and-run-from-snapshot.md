---
status: proposed
created: 2026-10-17
updated: 2026-10-17
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 進行ボットを Plugin に同封し、Plugin を入れたプロジェクトで setup が利用者のマシンに版を写して定期の実行を登録する

メンテナの要件「誰でも、Plugin をインストールしたプロジェクトで、dev-autopilot がすべて動く。Plugin を入れたら setup で GitHub の設定をして使えるようにする」（2026-10-17）を、配り方と動かし方の決定にする。

改める決定は次の 4 つで、ほかの決定は据え置く。

- [ADR-20261016-003][adr-move] の決定 2 のうち、「進行ボットは Plugin を marketplace から入れず、dev-autopilot のリポの clone の `plugin/` を `--plugin-dir` で読み込む」を、決定 1・2 で改める。
  進行ボットとエージェント定義・Skill を同じ版から使うこと、ほかのリポへの適用とほかの人への公開を段階 8 の後に決める時期、dev-autopilot 自身のリポに当てないことは据え置く
- ADR-20261016-003 の決定 3 のうち、「進行ボットのプログラムの clone（`devAutopilotPath`）」を、決定 2 の「進行ボットのプログラムの写し」に改める。business-os の clone（`<置き場>/controller/`）を分けることは据え置く
- [ADR-20261015-001][adr-read] の決定 3 の「手動実行は launchd のジョブを起こす」を、決定 3 で「OS ごとの定期の実行の仕組みで、同じジョブを起こす」に改める。人のセッションから進行ボットを直接起動しないことは据え置く
- [ADR-20261015-005][adr-settings] の決定 2 のうち、マシンの設定のパスを渡す「起動の定義（launchd の plist）」を、決定 3 で OS ごとの起動の定義に改める

## 背景と課題（Context and Problem Statement）

- 今の要件と ADR は、進行ボットを、dev-autopilot のリポを clone したマシンで動かす形にしている（STRUCT-3、ADR-20261016-003 の決定 2・3）。
  launchd が clone の `src/main.ts` を Node 24 で実行し、AI のセッションには同じ clone の `plugin/` を `--plugin-dir` で読ませる。marketplace で配るのは `plugin/` だけで、`src/` は配らない（[07-structure.md][struct] の構成）
- 今の形では、Plugin をインストールしただけの人は進行ボットを動かせない。人が呼ぶ `setup`・`check`・`/dev-autopilot` も、clone の `src/` を `devAutopilotPath` から探すので動かない（STRUCT-3）。単体で役に立つのは `review` だけ（STRUCT-7）
- メンテナの要件は、誰でも、Plugin を入れたプロジェクトで、進行ボットを含めてすべて動くこと（2026-10-17）。
  動かす場所は、今は利用者のマシン（macOS と Linux）で、将来は GitHub Actions などのクラウドと Windows でも動かしたい。
  Plugin で入れる形は基本設計から前提にする。当てる先を段階 8（進行ボットが PR にマージの判定の印を書き始める段階。「[段階][stages]」）の後まで business-os だけにすることと、ほかの人への公開を段階 8 の後にすることは変えない
- 2026-10-17 に Claude Code の公式文書（Plugin の読み込みと環境変数）を読んだ（補足情報）。
  marketplace から入れた Plugin は `~/.claude/plugins/cache/<marketplace>/<plugin>/<版>/` に写され、`${CLAUDE_PLUGIN_ROOT}` はそこを指し、版が上がると変わる。古い版のフォルダは 14 日後に消える。
  Plugin のフォルダの外のファイルは写されない。Plugin の `package.json` の依存は、npm か Bun のロックファイルがあるときだけ、写した版のフォルダに入る

課題：Plugin を入れたプロジェクトで進行ボットが動くために、進行ボットのコードを Plugin のどこに入れ、利用者のマシンでどこから動かし、定期の実行をどう登録し、どの順で準備するか。

## 判断の決め手（Decision Drivers）

- Plugin のインストールと `setup` だけで、進行ボットを含めてすべて動くこと（メンテナの要件）
- 進行ボットと、AI のセッションに読ませるエージェント定義・Skill を、同じ版から使うこと（出力契約や引数の食い違いで黙って壊れないため。STRUCT-7）
- GitHub に書く権限を持つ進行ボットのコードが、人の気づかないうちに入れ替わらないこと
- AI のセッションが、進行ボットのコードを読めず書けないこと（SAFE-16）
- 進行ボットの本体が macOS の仕組みに縛られず、Linux と、将来のクラウドと Windows に広げられること

## 検討した選択肢（Considered Options）

問い 1：進行ボットのコードを、Plugin のどこに入れるか

- 案 1-A：`src/` を `plugin/` の下に移す（`plugin/src/`）。Plugin の root は今のまま `plugin/`
- 案 1-B：リポの root を Plugin の root にする（marketplace の `source` を `./` にする）
- 案 1-C：構成は今のまま、配る前に `src/` を `plugin/` の下へ写す手順を持つ

問い 2：利用者のマシンで、進行ボットをどこから動かすか

- 案 2-A：`setup` が、インストールされた Plugin の版を、マシンの置き場の中の版ごとのフォルダに写し、定期の実行と `--plugin-dir` は写しを指す。版を上げるのは、人が `setup` を実行したときだけ
- 案 2-B：インストールされた Plugin の場所（`${CLAUDE_PLUGIN_ROOT}`）から直接動かす
- 案 2-C：`setup` が、dev-autopilot のリポを、入れた Plugin の版のタグで clone する

問い 3：定期の実行をどう登録するか

- 案 3-A：`setup` が OS ごとの仕組みで登録する（macOS は launchd、Linux は systemd のユーザーのタイマーか cron）。進行ボットの本体は登録の仕組みに依らない
- 案 3-B：どの OS でも cron で登録する
- 案 3-C：当面は macOS だけにする

問い 4：Plugin をどの範囲に入れ、どこで準備するか

- 案 4-A：当てる先のリポに、プロジェクトの範囲で入れる。当てる先のリポで開いた Claude Code で `setup` を実行する
- 案 4-B：ユーザーの範囲で入れ、`setup` に当てる先のリポを引数で渡す

## 決定（Decision Outcome）

採用した選択肢:「案 1-A」「案 2-A」「案 3-A」「案 4-A」。
理由: Plugin のインストールと `setup` だけで動き、配る範囲は `plugin/` に閉じたまま。進行ボットとエージェント定義・Skill は写した 1 つの版から使い、Plugin が更新されても、人が `setup` を実行するまで、動いている進行ボットのコードは変わらない。
進行ボットの本体は登録の仕組みに依らないので、Linux と、将来のクラウドと Windows に広げられる。

### 決定 1：進行ボットのコードを `plugin/` の下に移し、Plugin に同封する

- `src/`（進行ボット）を `plugin/src/` に移す。Plugin の root は今のまま `plugin/` で、marketplace の `source` はここを向く
- 単体テストは今のとおり対象のコードの隣に置く（STRUCT-4）ので、Plugin に同封される。結合テスト（`test/`）・前提データ（`fixtures/`）・文書（`docs/`）は `plugin/` の外のままで、配らない
- 進行ボットの実行に要る npm の依存は持たないか、持つなら Claude Code の依存の導入で入る形（`plugin/` に `package.json` と npm のロックファイルを置き、登録所の版を固定し、lifecycle script に頼らない）にする。どちらにするかは基本設計で決める。
  開発の道具（検査・テスト）の依存は、今のとおりリポの root の `package.json` と pnpm に置く
- 07-structure.md の「配布物は `plugin/` に閉じ、進行ボットの `src/`・テスト・前提データ・文書は配らない」を、「配布物は `plugin/` に閉じ、進行ボットも配る」に改める

### 決定 2：setup が Plugin の版をマシンに写し、進行ボットは写しから動かす

- `setup` は、インストールされた Plugin の版（`${CLAUDE_PLUGIN_ROOT}` の中身）を、マシンの置き場の中の版ごとのフォルダ（例：`<置き場>/program/<版>/`）に写す。写しを「進行ボットのプログラムの写し」と呼ぶ。マシンの設定の `devAutopilotPath` は写しを指す
- 定期の実行は写しの `src/main.ts` を Node 24 で実行し、進行ボットは AI のセッションに写しを `--plugin-dir` で読ませる。STRUCT-3 の「同じ clone の同じ commit から使う」を「同じ写しから使う」に改める
- 写しは進行ボットと `setup` だけが書き、AI のセッションは読めず書けない（置き場は AI のセッションの `denyRead` に入っている。SAFE-10、SAFE-16、[ADR-20261015-001][adr-read]）
- 版を上げるのは、人が `setup` を実行したときだけ。Plugin が更新されても（自動の更新を含む）、写しは変わらない。`check` は、インストールされた Plugin の版と写しの版が違えば warn にする
- 写しを作り直すのは、進行ボットが動いていない間に限る（実行の lock を取る）。古い版の写しの片付け方は、基本設計で決める

### 決定 3：定期の実行は setup が OS ごとの仕組みで登録し、進行ボットの本体は登録の仕組みに依らない

- 今は macOS と Linux に対応する。macOS は launchd、Linux は systemd のユーザーのタイマーか cron で登録する（Linux でどちらにするかは基本設計で決める）
- 進行ボットの本体（`plugin/src/`）は、どの仕組みから起こされても同じに動く。OS に依る処理（登録、手動実行の起こし方、秘密のファイルの権限の確かめ）は、`setup`・`check`・`/dev-autopilot` の中の OS ごとの部分に閉じる
- マシンの設定のパスは、OS ごとの起動の定義（macOS は launchd の plist）から進行ボットに渡す（ADR-20261015-005 の決定 2 の「起動の定義（launchd の plist）」を改める）
- 手動実行（`/dev-autopilot`）は、進行ボットを直接起動せず、定期の実行と同じジョブを起こす（macOS は `launchctl kickstart`（`-k` なし）。Linux は、登録した仕組みで同じジョブを起こす操作）。ADR-20261015-001 の決定 3 を、OS ごとに同じジョブを起こす形に改める
- Windows と、GitHub Actions などのクラウドで動かすことは、今は対応しない。広げるときは、進行ボットの本体を変えずに、登録の仕組みと秘密の置き場を足す形にし、別の ADR で決める（クラウドの論点は UNDECIDED-21）
- sandbox の振る舞いを確かめた事実（FACT-9、FACT-13、FACT-22 など）は、macOS で確かめたもの。Linux では確かめ直し、確かめるまで Linux で進行ボットを動かさない

### 決定 4：Plugin は当てる先のリポにプロジェクトの範囲で入れ、当てる先のリポで setup を実行する

- 利用者は、当てる先のリポで Plugin をプロジェクトの範囲で入れる（入れる設定は当てる先のリポの `.claude/settings.json` に書かれ、commit される）。当てる先のリポで開いた Claude Code で `setup` を実行する
- `setup` は SETUP-1 のとおり、作るものの一覧を出して人の承認を得てから、次を作る：GitHub の設定（ラベル、Project の欄、ruleset）、リポの設定（`.claude/dev-autopilot.json`。STRUCT-2）、マシンの設定と置き場と秘密の置き場、プログラムの写し（決定 2）、定期の実行の登録（決定 3）
- PAT と deploy key の発行は人が行う。`setup` と `check` がどの認証で GitHub を操作するかは、UNDECIDED-37 のまま別に決める
- `setup` は、dev-autopilot 自身のリポでは何も作らずに止まる（ADR-20261016-003 の決定 2）
- AI のセッションは今のとおり `--setting-sources local` で起動するので、当てる先のリポの `.claude/settings.json` の Plugin の設定は読まない。AI のセッションが読む Plugin は、進行ボットが `--plugin-dir` で渡す写しだけ
- 1 台のマシンで複数のリポに当てる形は、段階 8 の後に決める（段階 8 の後までは、当てる先は business-os だけ）

### 影響（Consequences）

- 良い点: Plugin を入れたプロジェクトで `setup` を実行すれば、進行ボットを含めて動く（メンテナの要件）
- 良い点: 配る範囲は `plugin/` に閉じたまま。進行ボットとエージェント定義・Skill は同じ写しから使うので、版がずれない
- 良い点: Plugin の更新で、動いている進行ボットのコードが人の気づかないうちに入れ替わらない。版を上げる操作が `setup` の 1 つにまとまる
- 良い点: 進行ボットの本体が登録の仕組みに依らないので、Linux・クラウド・Windows に広げるときに本体を書き換えない
- 悪い点: 利用者のマシンに Node 24 が要る。`setup` と `check` で確かめる
- 悪い点: Plugin を更新しても、`setup` を実行するまで進行ボットは古い版で動く。人が呼ぶ Skill（インストールされた版）と進行ボット（写しの版）がずれる間がある（`check` の warn で知らせる）
- 悪い点: 単体テストのファイルが Plugin に同封される
- 悪い点: OS ごとの登録と手動実行の処理を 2 通り持つ。Linux の sandbox の振る舞いは確かめ直しが要る
- 悪い点: 要件と図の書き直しが広い（要件への反映の一覧）

### 確認方法（Confirmation）

- 構造仕様を書くときに、構成（`plugin/src/`）、プログラムの写しの場所と作り方、OS ごとの登録が、決定 1〜4 のとおりかをレビューで確かめる
- 段階 3（`setup` と `check` を作る段階）で、次を確かめる
  - marketplace から入れた Plugin に `plugin/src/` が写り、`setup` が写しを作って、定期の実行が写しから動くこと（macOS と Linux）
  - Plugin を更新しても写しが変わらず、`check` が版のずれを warn にすること
  - AI のセッションから写しを読めないこと（SAFE-5 の、読み取りを試す確かめ）
  - Linux で、sandbox の振る舞い（FACT-9、FACT-13、FACT-22 に当たるもの）
  - `setup` が dev-autopilot 自身のリポで止まること

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 1-A：`src/` を `plugin/` の下に移す

- 良い点: 配る範囲を `plugin/` に閉じたまま、進行ボットも配れる（business-os の ADR 20261009-02 と同じ考え方を保つ）
- 悪い点: `src/` を移し、検査の設定（tsconfig、vitest）の対象を直す
- 悪い点: 単体テストのファイルも同封される

### 案 1-B：リポの root を Plugin の root にする

- 良い点: 構成を変えずに済む
- 悪い点: `docs/`・`test/`・`fixtures/`・`.github/`、開発用の `.claude/` と `CLAUDE.md` まで Plugin に写る
- 悪い点: リポの root の `package.json`（開発の道具の依存と pnpm のロックファイル）が Plugin の `package.json` になり、Claude Code の依存の導入の条件（npm か Bun のロックファイル）に合わない

### 案 1-C：配る前に `src/` を `plugin/` の下へ写す

- 良い点: 開発の構成を変えずに済む
- 悪い点: 写す手順が要り、写し忘れや食い違いで、配った版と検査した版がずれうる

### 案 2-A：setup が版をマシンに写し、写しから動かす

- 良い点: 写しの場所は版ごとに決まり、Plugin の更新で変わらない。進行ボットとエージェント定義・Skill が同じ写しにある
- 良い点: 写しは置き場の中にあり、AI のセッションから読めない（今の置き場の規則のまま）
- 悪い点: 写すぶんディスクを使う。版を上げるのに `setup` の実行が要る

### 案 2-B：インストールされた Plugin の場所から直接動かす

- 良い点: 写さずに済む
- 悪い点: `${CLAUDE_PLUGIN_ROOT}` は版が上がると変わり、古い版のフォルダは 14 日後に消える。定期の実行の定義が古い場所を指したまま、更新の 14 日後に動かなくなる
- 悪い点: Plugin の更新（自動の更新を含む）で、GitHub に書く権限を持つ進行ボットのコードが、人の確認なしに入れ替わる

### 案 2-C：setup が dev-autopilot のリポを版のタグで clone する

- 良い点: 写しの出どころを、git の履歴と版のタグで確かめられる
- 悪い点: Plugin とは別の取り方になり、Plugin の版と clone の版がずれうる。git とネットワークと、リポを読める認証が要る

### 案 3-A：OS ごとの仕組みで登録する

- 良い点: 手動実行で、定期の実行と同じジョブを起こせる（ADR-20261015-001 の決定 3 の、人のセッションの環境を受け継がない理由を保てる）
- 悪い点: 登録と手動実行の処理を OS ごとに持つ

### 案 3-B：どの OS でも cron で登録する

- 良い点: 登録の処理が 1 通りで済む
- 悪い点: cron には、登録したジョブを手で起こす操作が無い。手動実行で進行ボットを直接起動すると、人のセッションの環境を受け継ぐ（ADR-20261015-001 の決定 3 が避けたこと）
- 悪い点: cron は、マシンがスリープしている間の実行を飛ばす。launchd（`StartCalendarInterval`）は、次に起きたときに 1 回にまとめて実行する（補足情報）。
  棚卸しは最初は 1 日 1 回（「[決まっていること][overview]」の棚卸しの間隔）なので、閉じていることがあるノート PC で cron を使うと、その日の実行がまるごと飛ぶ

### 案 3-C：当面は macOS だけにする

- 良い点: 今の要件と事実（macOS で確かめた）のまま進められる
- 悪い点: メンテナの要件（今は macOS と Linux）に合わない

### 案 4-A：当てる先のリポにプロジェクトの範囲で入れる

- 良い点: 「Plugin を入れたプロジェクトで動く」という要件の形のまま。人が呼ぶ Skill は、当てる先のリポで開いたセッションにだけ出る
- 悪い点: 当てる先のリポに、Plugin を入れる設定が commit される

### 案 4-B：ユーザーの範囲で入れ、当てる先を引数で渡す

- 良い点: 当てる先のリポに設定を commit しない
- 悪い点: どのプロジェクトのセッションにも dev-autopilot の Skill が出る。`setup` が当てる先を取り違えうる

## 補足情報（More Information）

- 関連する要件：STRUCT-1〜7（[07-structure.md][struct]）、SAFE-5、SAFE-10、SAFE-16（[05-safety.md][safety]）、SETUP-1、SETUP-2（[06-setup-and-stages.md][stages]）、「決まっていること」の実行環境（[00-overview.md][overview]）、
  UNDECIDED-21、UNDECIDED-36、UNDECIDED-37（[09-undecided.md][undecided]）、[ADR-20261015-001][adr-read]、[ADR-20261015-005][adr-settings]、[ADR-20261016-003][adr-move]
- 2026-10-17 に、Claude Code の公式文書（[Plugin の読み込み][doc-loading]、[Plugin の参照][doc-plugins-ref]、[Plugin を探して入れる][doc-discover]）を読んだ。手元の Claude Code は 2.1.285。読んだこと：
  - marketplace から入れた Plugin は、`~/.claude/plugins/cache/<marketplace>/<plugin>/<版>/` に写されて読み込まれる。`${CLAUDE_PLUGIN_ROOT}` は入れた版の場所を指し、版が上がると変わる
  - Plugin を更新するか外すと、前の版のフォルダに印が付き、14 日後に消える（すでに古い版を読み込んだセッションは動き続ける）
  - Plugin のフォルダの外のファイルは写されない。Plugin の部品は、Plugin のフォルダの外を指せない
  - Plugin の `package.json` の依存は、Plugin の root に `package.json` と npm か Bun のロックファイルがあるときだけ、写した版のフォルダに入る（pnpm のロックファイルでは入らない）。登録所の版を固定した依存だけで、lifecycle script は動かず、60 秒で打ち切られる
  - `${CLAUDE_PLUGIN_DATA}`（`~/.claude/plugins/data/<id>/`）は版をまたいで残るが、Plugin を最後の場所から外すと既定で消える
  - 自動の更新は、Anthropic 以外の marketplace では既定で切れていて、設定か `/plugin` の切り替えで入れられる。`claude plugin update` でも版が上がる
  - プロジェクトの範囲で入れると、入れる設定はリポの `.claude/settings.json` に書かれ、commit される
- 2026-10-17 に、手元の macOS の man ページ（`crontab(1)`・`crontab(5)`・`cron(8)`・`launchd.plist(5)`）を読んだ。読んだこと：
  - cron と crontab は macOS（Darwin）で正式に動くが、役割は launchd に取り込まれていて、launchd の方が柔軟だと注記されている（`crontab(1)` と `crontab(5)`）
  - 「cron はコンピュータがスリープしている間の実行を飛ばすが、launchd は次に起きたときにジョブを実行する。複数回ぶんを逃していたら、起きたときに 1 回にまとめる」（`launchd.plist(5)` の `StartCalendarInterval`）。
    一方、`StartInterval`（N 秒ごと）は、スリープしている間の発火を逃す
  - Linux の systemd のタイマーと cron のスリープの扱いは、手元に Linux の man ページが無いので確かめていない。Linux でどちらにするかを基本設計で決めるときに確かめる
- 写しの置き場を `${CLAUDE_PLUGIN_DATA}` にする形も考えた。Plugin を外すと写しも消え、置き場の規則（ADR-20261015-001、SAFE-10）とも別の場所になるので、マシンの置き場の中にした
- 本 ADR で決めずに残すもの
  - UNDECIDED-36（AI のセッションが Plugin を読めるか）：Plugin の場所が写しに変わるだけで、写しも置き場の中にあり、論点は残る
  - UNDECIDED-37（人が呼ぶ `setup` と `check` の認証と、マシンの設定の見つけ方）：Plugin がプロジェクトに入るので、人が呼ぶ Skill はプロジェクトのセッションに出るが、認証と見つけ方は残る
  - UNDECIDED-21（クラウドの仕組み）と Windows：決定 3 のとおり、広げるときに別の ADR で決める
- 要件への反映（承認の後に、別の PR で行う）
  - 07-structure.md：構成（`src/` を `plugin/src/` に）、STRUCT-3（写しから起動し、`--plugin-dir` は写しを指す）、STRUCT-4（`tsconfig` と `vitest` の対象）、STRUCT-7（進行ボットも同封する）、「分離の時期の目安」の言い方
  - 05-safety.md：SAFE-16 の「進行ボットのプログラムの clone」を写しに改める。SAFE-2 の (a) の Plugin の渡し方
  - 06-setup-and-stages.md：SETUP-1（Plugin をプロジェクトの範囲で入れてから `setup` を実行する流れ、写しと OS ごとの登録を作ること）、SETUP-2（Node 24、版のずれ、写しを読めないことの確かめ）
  - 00-overview.md：「決まっていること」の実行環境（今は macOS と Linux の利用者のマシン、将来はクラウドと Windows）と、「登場するもの」の進行ボットの行
  - 09-undecided.md：Linux での sandbox の振る舞いを確かめることを、「確かめること」に足す
  - 図：構成の図（`plugin/src/`、写し、OS ごとの登録）と、全体の構成・隔離と鍵の図の「進行ボットのプログラムの clone」を直す
  - ADR-20261016-003、ADR-20261015-001、ADR-20261015-005：タイトルの下に、本 ADR で改めた旨の注記を足す

[overview]: ../requirements/00-overview.md
[safety]: ../requirements/05-safety.md
[stages]: ../requirements/06-setup-and-stages.md
[struct]: ../requirements/07-structure.md
[undecided]: ../requirements/09-undecided.md
[adr-read]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[adr-settings]: ./adr-20261015-005-settings-values-and-machine-settings.md
[adr-move]: ./adr-20261016-003-move-to-own-repository-before-basic-design.md
[doc-loading]: https://code.claude.com/docs/en/plugins/loading
[doc-plugins-ref]: https://code.claude.com/docs/en/plugins-reference
[doc-discover]: https://code.claude.com/docs/en/discover-plugins
