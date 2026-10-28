# 導入と点検

人が呼ぶ `setup` と `check`、進行 Bot が毎回の実行の最初に回す点検、`/dev-autopilot` の状態の表示の、順番と仕組みを決める。
何を作り何を確かめるかは要件の [SETUP-1、SETUP-2][setup] が正本で、基本設計は順番・置き場・確かめ方を決める。

## setup の流れ

![setup の流れ][fig-setup-flow]

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 開いているプロジェクトの origin から owner/repo を引く。dev-autopilot 自身のリポなら、何も作らずに止まる（[STRUCT-3][struct]、[分離の時期の目安][struct]）。dev-autopilot 自身のリポかは、origin の owner/repo が dev-autopilot のリポ（owner/repo を進行 Bot のプログラムに持つ）か、開いているプロジェクトに名前（`name`）が `dev-autopilot` の `plugin/.claude-plugin/plugin.json` があるかで見分ける（名前を変えたリポと fork も止めるため）。毎回の点検も、`repository.*` と、base ブランチの先端の `plugin/.claude-plugin/plugin.json` で同じように確かめる（下の「組と回数」の GitHub の組） | origin が無いか、GitHub のリポでなければ止まる |
| 2 | base ブランチの名前を決める。マシンの設定にあればその値、無ければ人に聞く（候補は origin の既定のブランチと `develop`）。次に、マシンの設定のフォルダ（[設定][design-settings]の「マシンの設定」）と、GitHub の今の状態を読み、足りないものと消すもの（Plugin の版で消した項目の設定のキーと、設定に無い型の ruleset の除外。下の「GitHub の準備」）の一覧を作る。base ブランチの先端のコミットで動いたチェックを、必須チェックの候補として一覧に入れる（下の「GitHub の準備」）。人の `gh` のログインが無ければ（専用マシン）、GitHub を読まず、マシンの準備だけを一覧にする | 人の `gh` のログインがあるのに GitHub を読めなければ止まる |
| 3 | 一覧を人に見せ、1 回だけ承認を受ける（[SETUP-1][setup]）。GitHub の準備と、このマシンで進行 Bot を動かすか（マシンの準備）を、分けて選べるようにする。人の手元のマシンで、専用マシンのために GitHub の準備だけを行うときは、マシンの準備を選ばない | 承認されなければ、何も作らない |
| 4 | GitHub の準備を、人の `gh` のログインで行い、作ったものを API で読み戻して確かめる（下の「GitHub の準備」）。人の `gh` のログインが無いか、3 で GitHub の準備を選ばなければとばす | 読み戻した値が違えば止まり、違いを示す |
| 5 | マシンの準備を、GitHub の認証を使わずに行う（下の「マシンの準備」）。秘密のファイルは作るだけで、中身を読まない。作業フォルダの外に書くコマンドは、sandbox の外で動かす許可を人に求めてから動かす（表の下）。3 でマシンの準備を選ばなければとばす | sandbox の外で動かせなければ、作業フォルダの外に何も作らずに、理由を示して止まる（表の下）。作れなければ止まり、作れたものを示す |
| 6 | 人がやることを、手順として示す（[SETUP-1][setup] の「人がやるもの」） | — |
| 7 | 定期の実行と同じジョブで点検だけを起こし、結果を示す（下の「点検だけで動かす仕組み」）。全体の止める合図が `stop` の間は、試しのセッションと書き込みの権限の試しを行わず、未確認の warn として示す。5 をとばしたときは行わない | fail があれば示す |
| 8 | 人が、点検の fail が無くなってから、全体の止める合図を `run` にする（[運用の規則][flow-rules]）。合図を `run` にした後の最初の実行が、作業の前に、試しのセッションと書き込みの権限の試しを行う（下の「組と回数」）。`setup` をした日（記録の目印のファイルを作った日）は AI のセッションを起動しない（[AI のセッション][design-ai]の「起動の前の確かめ」）ので、試しのセッションを行う最初の実行は次の日からになる。`setup` は、試しのセッションが次の日からになることを、順 7 の結果と一緒に示す | 試しが fail なら、最初の実行は作業に入らない |

- 何度実行しても同じ結果にする（冪等）。今あるものは作り直さず、設定ファイルの今の値は上書きしない。設定ファイルには、足りない項目を足し、Plugin の版で消した項目のキーを消す（どちらも順 3 の承認の一覧に出す。リポの設定は PR で、マシンの設定はファイルを書き直して行う。下の「GitHub の準備」と「マシンの準備」）。消さずに残すと、表に無いキーとして点検が fail にし続けるため（[設定][design-settings]の「`check` が設定について確かめること」）。リポの設定の PR がマージされるまでは、新しい版の点検は fail になる（下の「GitHub の準備」の表のリポの設定の行）。`setup` を引数 `rollback` で実行して前の版に戻すときは、逆向きに、戻す先の版の表に合わせて足し引きする（下の「プログラムの写し」）
- Plugin を更新した後に実行し直すと、プログラムの写しを新しい版で作る（下の「プログラムの写し」）。`rollback` で版を固定している間は、引数 `upgrade` のときだけ
- 人の `gh` のログインが無いマシン（専用マシン）でも、順 1〜3（owner/repo を引く、足りないものの一覧、承認）は行う。dev-autopilot 自身のリポかを確かめ、マシンの準備の一覧に承認を受けるため（[SETUP-1][setup]）。順 2 で GitHub を読まず、順 4 の GitHub の準備をとばす。GitHub の準備は、人の手元のマシンで `setup` を実行し、マシンの準備を選ばずに行う（[ADR-20261017-004][adr-human] の決定 2）
- 専用マシンでも、`setup` を実行するために、当てる先のリポの作業コピー（Plugin を入れた `.claude/settings.json` を含む）を置き、そこで開いた Claude Code から呼ぶ（[STRUCT-3][struct]）。進行 Bot は作業コピーを読まない（[SAFE-2][safety]）
- 1 つの当てる先のリポで、定期の実行を登録するマシンは 1 台にする。lock はマシンの中の記録の置き場にあり、2 台が同じリポで動くと、重い工程・投稿・費用が重なるため（[運用の規則][flow-rules]）。動かすマシンを替えるときは、下の「動かすマシンを替えるとき」の順に、前のマシンの登録を外してから、新しいマシンで `setup` を実行する
- 人の Claude Code が sandbox を有効にしていると、Bash の道具から作業フォルダの外に書けない（FACT-71）。`setup`（引数の `rollback`・`upgrade`・`uninstall` を含む）は、作業フォルダの外に書くコマンド（下の「マシンの準備」の表のフォルダとファイル、プログラムの写し、定期の実行の登録、記録の置き場の lock と点検の依頼のファイル）を、sandbox の外で動かす許可を人に求めてから動かす。人の設定が sandbox の外でコマンドを動かすことを許していない（`sandbox.allowUnsandboxedCommands` が `false`）か、人が許可しなければ、作業フォルダの外に何も作らずに、sandbox の中からは作業フォルダの外に書けないことを理由として示して止まる

### GitHub の準備

| 作るもの | 作り方 | 確かめ方 |
| --- | --- | --- |
| ラベル 5 つ（人が付ける `agent-ready`・`agent-paused`・`agent-manual`、進行 Bot が付ける表示の `security-suspected`・`from-review`。名前は設定の `labels.*`） | 無いものだけ作る | 名前で読み戻す |
| Project の表示の欄 5 つ（主な流れの枠・状態・止め方・先の依頼・安全）と、選択肢と、軸ごとのビュー（[FLOW-39][flow-rules]） | 状態は、もともとある Status 欄を名前を変えずに使い、選択肢を主な流れの 17 の状態に置き換える（Status 欄の名前は API で変えられない。FACT-74、[ADR-20261026-001][adr-status]）。置き換えると、今の選択肢と項目の Status の値は失われるので、作るものの一覧に出し、人の承認を受けてから置き換える。主な流れの枠・止め方・先の依頼・安全の 4 つは、無ければ単一選択の欄として作る。選択肢は、[FLOW-39][flow-rules] の表の値で作り、（なし）の選択肢は作らない（値が無いときは欄を空にする。[設定][design-settings]の `project.fields.*`）。選択肢を足すときは、今ある選択肢を `id` 付きで含めて丸ごと送る（`id` を落とすと、今ある選択肢と項目の値が消える。FACT-15）。ビューは、軸ごとに、人の番を探す条件（[FLOW-39][flow-rules] の例）で、`createProjectV2View` と `updateProjectV2View` で作る（名前・形・見せる欄・絞り込み。FACT-74）。絞り込みを並べると「かつ」になるので、軸ごとに 1 つのビューにする。ボードの列にする欄は API で指定できず、Status 欄（状態）で分かれる | 欄と選択肢の名前と `id` を読み戻し、リポの設定の `project.fields.*` に書く |
| base ブランチの ruleset（PR 必須、force push の禁止、必須チェックを strict（base ブランチと最新であることを求める設定）で登録、`require_extra_approval_for_unattributed_changes` は無効） | 無ければ、bypass を組織の管理者だけにして作る（下の「ruleset の形と確かめ方」）。必須チェックは、順 2 で一覧に入れた候補のうち、人が承認したものを登録する。あれば違いを示し、上書きしない。ただし、リポの設定の `checks.required` にあって ruleset に無い必須チェックは、承認を受けて足す（守りを強める向きの変更だけ） | 読み戻して項目を比べる（FACT-7）。点検は、左の欄の項目と、`enforcement`・対象・bypass を比べ、守りが弱い向きの違い（PR 必須でない、force push を許す、必須チェックが足りないか strict でない、`active` でない、base ブランチが対象に無い、bypass に組織の管理者とチームのほかがある）を fail にする（下の「ruleset の形と確かめ方」） |
| deploy key の push 先を `<型>/issue-*` のブランチに限り、タグの作成を禁じる ruleset（ブランチの ruleset とタグの ruleset の 2 つ） | 無ければ、bypass を組織の管理者と `push.rulesetBypassApps` のアプリにして作る（下の「ruleset の形と確かめ方」）。あれば違いを示し、上書きしない。ただし、`labels.branchTypes` と `labels.defaultBranchType` の型の除外が足りないか、`push.rulesetBypassApps` のアプリが bypass に無ければ、承認を受けて足す。設定に無い型の除外（型を外したか、既定の型を替えた後に残るもの）は、外す型の作業ブランチ（`<型>/issue-*`）が origin に無いときだけ、承認を受けて外す。あれば外さず、残っている作業ブランチを示す（下の「ruleset の形と確かめ方」） | 読み戻して項目を比べる（[IMPL-15][impl]、FACT-65）。点検は、deploy key が push できるブランチが `<型>/issue-*` より広いとき（除外が型ごとの `refs/heads/<型>/issue-*` より広い。設定に無い型の除外でも、`refs/heads/<名前>/issue-*` の形で名前に `*`・`?`・`[` を含まないものは、`setup` が外すまでの間 fail にせず warn にする）、作成・更新・削除のどれかを止めていないとき、タグの作成を許すとき、`active` でないとき、bypass に組織の管理者とチームと `push.rulesetBypassApps` のアプリのほかがあるとき（進行 Bot のアプリは、一覧にあっても fail）に fail にする |
| 全体の止める合図（リポの変数 `DEV_AUTOPILOT_RUN`） | 無ければ値を `stop` で作る。あれば値を変えない | 読み戻す（[ADR-20261017-007][adr-stop]） |
| リポの設定（`.claude/dev-autopilot.json`） | 表の初期値のうち当てる先に依らない値（許可一覧と方針のパスは空）と、API で引いた Project の ID と選択肢の ID と、順 2 で承認した必須チェックの候補（`checks.required`）と、人に聞いた規約（言語・Conventional Commits・changeset）と、人に聞いた変えてはいけないパスの足す分（規約の正典のパスを `protectedPaths.onMerge` に、CI が信じて動かすスクリプトのパスを `protectedPaths.onPush` に）と、人に聞いた ruleset の bypass に入れるアプリ（`push.rulesetBypassApps`。Dependabot を使うリポなら `dependabot`）で作り、新しいブランチから base ブランチへの PR にする（base ブランチは PR 必須のため）。表に business-os の値として書いた初期値は、人が同じ PR に書き足す（[設定][design-settings]の「値の決まり方」）。人がマージする。前の `setup` が作った PR がまだ開いていれば、新しく作らずにその PR を示す。リポの設定がすでにあれば、足りない項目を足し、Plugin の版で消した項目のキーを消す PR を、同じく新しいブランチから出す（今の値は書き換えない） | PR を示す。進行 Bot は base ブランチの先端から読むので、マージされるまで点検は fail になる |

- Project の自動追加（組み込みのワークフロー）は公開の API が無いので、人が有効にする（[SETUP-1][setup]）。点検は有効かを読むだけ
- 当てる先のリポの CI が秘密を使うときは、Environment に置き、使えるブランチを「選んだブランチ」（Selected branches and tags）で、base ブランチと人が決めたブランチだけにする（[SAFE-20][safety]、[ADR-20261028-001][adr-ci-secrets]）。`setup` は、リポの秘密か、リポに届く組織の秘密があれば、Environment を作って秘密を移し、使えるブランチを選ぶ手順を人に示す（秘密を置くのも移すのも人が行う）。
  点検は、リポの秘密の一覧（`GET /repos/{owner}/{repo}/actions/secrets`）と、リポに届く組織の秘密の一覧（`…/actions/organization-secrets`）が空であることを、GitHub App の Secrets の読み取りで確かめる（名前だけで、値は読めない。[ADR-20261028-002][adr-app-secrets]）。Environment があれば（`…/environments`）、どれも使えるブランチの規則が「選んだブランチ」（`deployment_branch_policy` の `custom_branch_policies` が `true`）で、規則（`…/deployment-branch-policies`）のどれも、作業ブランチの名前（`labels.branchTypes` の型ごとの `<型>/issue-1`）と `refs/pull/1/merge` に当たらないことを、Actions の読み取りで確かめる。「No restriction」と「Protected branches only」は fail にする（ruleset だけで守るリポでは、Protected branches only が絞らないため。FACT-79）。外れていれば fail。PAT の間は確かめない（PAT の間は作業に入らない）。人の `check` と `setup` は、人の `gh` のログインで同じことを確かめる
- Status 欄に関わる組み込みのワークフロー 5 つ（Item closed・Pull request merged・Item added to project・Pull request linked to issue・Auto-close issue）は、`setup` で消さない。Status 欄の選択肢を 17 の状態に置き換えると、5 つは無効になる（FACT-76）。点検は、5 つのどれかが有効なら fail にし、人が Project の Workflows の画面で切る手順を示す。Status に値を書く 4 つは、進行 Bot が書いた状態を上書きし、Auto-close issue は Status の値で Issue を閉じて作業の単位を止めるため（[ADR-20261026-001][adr-status] の決定 3）。API で消しても、人は画面のひな形から作り直せる（FACT-76）ので、消す操作は足さず、有効かを読んで確かめる
- 組み込みのワークフローが有効かを読むには、Projects の書き込みの権限が要る（FACT-76）。進行 Bot の点検は、GitHub App に切り替えた後に、アプリの Projects（組織）の書き込みで読む（下の「GitHub App の権限」）。切り替えの前の PAT は Projects の読み取りだけなので、PAT の間は確かめない（PAT の間は作業に入らない。下の「PAT の権限」）。人の `setup` と `check` は、人の `gh` のログイン（`project` の権限）で読む
- 必須チェックの候補は、base ブランチの先端のコミットで動いたチェック（名前、出した app、ワークフローのパス）から作る。候補が無ければ（CI がまだ無い）、必須チェックを登録せずに ruleset を作り、`checks.required` を空のままにする。点検が fail にするので、人が CI を足して `checks.required` に書き、`setup` を実行し直すと、ruleset に足す（[SETUP-1][setup]、[IMPL-27][impl]）。`setup` が足すので、人が ruleset を手で直さずに済む（[SETUP-4][setup]）

#### ruleset の形と確かめ方

`setup` が作る ruleset は 3 つ。deploy key の push 先を限る ruleset は、ruleset の対象がブランチかタグの一方なので、ブランチの ruleset とタグの ruleset に分けて作る（[SETUP-1][setup]、[IMPL-15][impl]、FACT-65）。

| ruleset | `target` | 対象（`conditions.ref_name`） | 規則（`rules`） |
| --- | --- | --- | --- |
| base ブランチを守る | `branch` | include は `refs/heads/<baseBranch>` | PR 必須（`require_extra_approval_for_unattributed_changes` は無効）、force push の禁止、必須チェック（`checks.required` を strict で） |
| deploy key の push 先を限る（ブランチ） | `branch` | include は `~ALL`、exclude は型ごとの `refs/heads/<型>/issue-*`（型は `labels.branchTypes` と `labels.defaultBranchType` の値） | 作成・更新・削除（`creation`・`update`・`deletion`） |
| deploy key の push 先を限る（タグ） | `tag` | include は `~ALL` | 作成（`creation`） |

- 3 つとも、`enforcement` は `active` にする（`evaluate` と `disabled` は止めない）
- bypass（`bypass_actors`）に入れてよいのは、組織の管理者（`OrganizationAdmin`）とチーム（`Team`）だけ（deploy key の push 先を限る 2 つの ruleset では、下の `push.rulesetBypassApps` のアプリも）。`setup` は、人では組織の管理者だけを入れて作る（[IMPL-15][impl]、[SETUP-1][setup]）。リポの役（`RepositoryRole`。admin・maintain・write）を入れると、書き込みのある deploy key も、リポの役を bypass に入れた ruleset を素通りし、base ブランチの更新もほかのブランチとタグの作成も通る（FACT-65）。deploy key（`DeployKey`）も入れない
- GitHub App（`Integration`）は、base ブランチの ruleset には入れない。deploy key の push 先を限る 2 つの ruleset には、リポの設定の `push.rulesetBypassApps` のアプリだけを入れる。進行 Bot のアプリ（マシンの設定の `githubApp.appId`）は、一覧にあっても入れない。進行 Bot のアプリに書き込みの権限を足したときに、作業ブランチのほかへ書ける経路にしないため
  - `setup` は、一覧のアプリの名前を `GET /apps/<名前>` で ID に直し、`actor_type` を `Integration` にして入れる
  - GitHub は、リポで使われていないアプリを bypass に入れるのを 422 で断る（Dependabot は、リポで 1 回動く前は、画面の候補にも出ない。FACT-67）。断られたら、`setup` はアプリを入れずに ruleset を作り、アプリをリポで使える状態にしてから `setup` をやり直すよう示す。Dependabot なら、`.github/dependabot.yml` を置いて 1 回動かす（1 回目は、ruleset に止められてブランチを作れない）
- ブランチの ruleset は作業ブランチのほかのすべてのブランチを対象にするので、bypass に入っていない人とアプリは、作業ブランチのほかのブランチを作れず、base ブランチも更新できない（FACT-65、FACT-67）。bypass の外の人には、PR が `blocked` と出る（FACT-67）。そのため、PR をマージする人は bypass（組織の管理者か、bypass に入れたチーム）に入れる。`setup` は、終わりの案内でこの決まりを示す
- 人が `labels.branchTypes` に型を足したら、人の手元のマシンで `setup` を実行し直し（GitHub の準備）、ブランチの ruleset の exclude に足す。足すまでは、新しい型の作業ブランチへの push が断られる（止まる側に倒れる）
- 人が `labels.branchTypes` から型を外すか、`labels.defaultBranchType` を替えたら、前の型の作業ブランチが origin から無くなってから、人の手元のマシンで `setup` を実行し直し（GitHub の準備）、ブランチの ruleset の exclude から前の型を外す。`setup` は、前の型の作業ブランチが origin に残っていれば exclude から外さず、残っている作業ブランチを示す。作業ブランチの型はブランチ名から読む（[設定][design-settings]の `labels.branchTypes`）ので、前の型の作業ブランチが残る Issue は、外すまでの間も push を続けられる。外すまでの間、点検は設定に無い型の除外を warn に出す。除外は `<型>/issue-*` の形なので、deploy key が作業ブランチの形のほかへ push できる道にはならない（点検の fail は、作業ブランチの形より広い除外に限る）
- 確かめ方
  - `setup`（人の `gh` のログイン。組織の管理者）は、作った後と、すでにあったときに、3 つの ruleset の `enforcement`・`conditions`・`rules`・`bypass_actors` を読み戻して上の表と比べ、違いを示す。bypass の読み方の正本は、`setup` の読み戻し（[GitHub の公式文書][doc-rules]は、`bypass_actors` は ruleset に書き込める認証にだけ返るとする。Administration の権限の無い、組織の管理者の fine-grained PAT には返った。FACT-65）
  - 毎回の点検は、進行 Bot の GitHub の認証で、base ブランチに効く規則（`GET /repos/{owner}/{repo}/rules/branches/{baseBranch}`。FACT-65）を読み、PR 必須・force push の禁止・必須チェック（strict）が効いていることを確かめる。続けて、3 つの ruleset の `enforcement`・`conditions`・`rules`・`bypass_actors` を読み、上の表より守りが弱い向きの違いを fail にする（上の「GitHub の準備」の表の確かめ方の欄）
  - 進行 Bot の認証のうち、組織の管理者の fine-grained PAT（Administration の権限なし）では、上の欄をすべて読めた（FACT-65）。組織の管理者でない人の PAT で `bypass_actors` を読めるかは確かめていない。アプリのインストールのトークン（Administration の権限なし）で、3 つの ruleset の `bypass_actors` と base ブランチに効く規則を読めるかは、GitHub App に切り替えるときの試し（下の「GitHub App の権限」の切り替えの順 2）で確かめる（要件の UNDECIDED-69）。読めなかったときの扱いは、確かめたときに ADR で決める。ADR で決めるまでは、読めない欄があれば fail にし、読めない欄を確かめたものとして扱わない

### マシンの準備

| 作るもの | 置き場 | 中身 |
| --- | --- | --- |
| マシンの設定 | マシンの設定のフォルダの `settings.json` | マシンの 17 項目（[設定][design-settings]）。`repository.*` は開いているプロジェクトの origin から、`baseBranch` は順 2 で決めた値を書く。`githubApp.*` は `null` で作る（切り替えのときに人が書く）。`program.previousVersion` は `null`、`program.pinned` は `false`、`safety.env.values` は空で作る。`safety.sandbox.toolDirs` は、`node`・`pnpm`・`git` の実体の場所から書く。すでにあれば、足りない項目を足し、Plugin の版で消した項目のキーを消す（今の値は書き換えない） |
| 置き場 | `paths.workspace` | `sessions/`、`program/`（下の「プログラムの写し」） |
| 安全設定の置き場 | `paths.sandboxSettings` | 権限 700 で作る（既定は置き場の中の `sandbox-settings/`。[設定][design-settings]の「マシンの中のフォルダ」） |
| 記録と lock の置き場 | `paths.records` | 作った日時を書いた記録の目印のファイル（`records-created.json`。[運用の規則][flow-rules]、[状態遷移と判定][design-flow]の「手元の記録」） |
| 秘密の置き場 | `paths.secrets` | フォルダを権限 700 で作り、[設定][design-settings]の「秘密の置き場のファイル」の表のファイルを、`githubApp.*` の値に合わせて、空のまま権限 600 で作る。値は人が書く（[SAFE-15][safety]） |
| 短い一時フォルダの根 | `sessionTmp.root` | 権限 700 で作る（[ADR-20261017-006][adr-tmp]） |
| 定期の実行の登録 | OS ごと | 下の「定期の実行の登録」 |

- 進行 Bot の clone（`<置き場>/controller/`）は、`setup` では作らない。clone には deploy key が要り、`setup` のマシンの準備は秘密のファイルを読まないため。進行 Bot が最初の実行で作る
- SSH のホストの鍵（`<置き場>/known_hosts`）も、`setup` では書かない。GitHub の API の `meta` を `gh` で読むには認証が要り、`setup` のマシンの準備は GitHub の認証を使わないため。進行 Bot が、最初の push か fetch の前に、自分の認証の `gh api meta` から書く（[設定][design-settings]の「秘密の置き場のファイル」の `GIT_SSH_COMMAND`）

## プログラムの写し

![プログラムの写し][fig-program-copy]

`setup` は、インストールされた Plugin の版を、置き場の中の版ごとのフォルダに写す。定期の実行は写しから動き、AI のセッションにも写しを読ませる（[STRUCT-3][struct]、[ADR-20261017-003][adr-plugin] の決定 2）。
`setup`（引数の `rollback`・`upgrade`・`uninstall` を含む）は、入れた Plugin の `${CLAUDE_PLUGIN_ROOT}/src/` のコードで動く。初めての `setup` にはマシンの設定も写しも無く、Plugin を更新した後の `setup` で新しい版の準備を 1 回で入れるため。`check`・`/dev-autopilot`・`intake` は、写しの `src/` で動く（[STRUCT-3][struct]）。

| 順 | すること | うまくいかないとき |
| --- | --- | --- |
| 1 | 実行の lock を取る（写しを作り直すのは、進行 Bot が動いていない間に限る。[STRUCT-3][struct]） | 取れなければ、何もせずに、後で実行し直すように示す |
| 2 | インストールされた Plugin（`${CLAUDE_PLUGIN_ROOT}`）の版を、`plugin.json` の `version` から読む。マシンの設定の `program.pinned` が `true`（`rollback` で版を固定している）なら、引数 `upgrade` のときのほかは版を上げず、今の版のまま順 5 から行う。`program/<版>/` がすでにあれば、中身のハッシュ（すべてのファイルのパスと中身の SHA-256）を比べる | 固定しているときは、版を上げないことと上げ方（`setup upgrade`）を示す。中身が同じなら順 3（別の名前で写す）と順 4（名前を変える）をとばし、順 5 から行う。違えば止まる（同じ版で中身が違う Plugin を、黙って上書きしない） |
| 3 | 前の `setup` が途中で止まって残した `program/.new-*` を消してから、`program/.new-<版>/` に写す。写した中身のハッシュを Plugin の中身のハッシュと比べ、ファイルの書き込みの権限を外し、写した中身の一覧（パスと SHA-256）を、写しの外の `program/<版>.manifest.json` に書く | ハッシュが違えば、写したフォルダを消して止まる |
| 4 | `program/<版>/` に名前を変える。名前を変えるのを最後の 1 手にし、名前を変える前に止まっても、`setup` を実行し直せば順 3 から同じ結果になる（冪等） | — |
| 5 | マシンの設定の `devAutopilotPath` を、使う版のフォルダに向ける。版が変わるときは、変わる前の版を `program.previousVersion` に書く。定期の実行を登録し直す（下の「登録し直すとき」） | — |
| 6 | 今使う版と、`program.previousVersion` の版のほかの `program/<版>/` と manifest を消し、lock を外す | — |

- 別の名前で写してから名前を変えるのは、写している途中のフォルダを進行 Bot が使わないようにするため
- 直前の版（`program.previousVersion`）を残すのは、新しい版に問題があったときに、写し直さずに戻せるようにするため。直前の版は版が変わるときだけ書き換えるので、同じ版で `setup` を実行し直しても、戻す先の写しを消さない。`setup` を引数 `rollback` で実行すると、`program.previousVersion` の版の写しに戻し（順 1 の lock、順 5 の `devAutopilotPath` と定期の実行の登録を戻すこと、順 6 の片付けを行う）、戻す前の版を `program.previousVersion` に書いて、`program.pinned` を `true` にする。入れた Plugin の版は新しいままなので、人が呼ぶ Skill は版の違いを warn に出す（作業は止めない）（[ADR-20261017-003][adr-plugin] が基本設計に残した、古い写しの片付け方）
- `rollback` は、設定も戻す先の版の表に合わせる。マシンの設定は、戻す先の版の表に無いキー（新しい版で足した項目）を消し、戻す先の版の表にあって無い項目（新しい版で消した項目）を表の初期値で足して書き直す。リポの設定は、同じ足し引きをする PR（新しい版の `setup` が出した PR を戻すもの）を、上の「GitHub の準備」の表のリポの設定の行と同じく新しいブランチから出す。足すキーと消すキーは、上の「setup の流れ」の順 3 と同じく承認の一覧に出し、承認を受けてから行う。PR がマージされるまで、戻した版の点検は fail のままで、作業に入らない。表と違う設定のまま前の版を動かさないため
- 固定している間（`program.pinned` が `true`）は、登録し直しや GitHub の準備のために `setup` を実行し直しても、版を上げない。戻した版が、人の気づかないうちに新しい版に戻らないため。版を上げるのは、人が `setup` を引数 `upgrade` で実行したときだけで、`program.pinned` を `false` に戻す
- 人がマシンの設定の `devAutopilotPath` だけを戻しても、定期の実行の登録が新しい版のパスを持つので、動くプログラムは変わらない。点検が、登録のずれとして fail にする
- `claude plugin install` には、版を指定する引数が無い（Claude Code 2.1.285 の `claude plugin install --help` で確かめた）。入れた Plugin そのものを前の版に戻す手順は、配り方を決めるときに決める（[分離の時期の目安][struct]）。それまでは、上の `rollback` で写しだけを戻す
- 書き込みの権限を外すのは、人の誤った編集を防ぐため。AI は sandbox の中から、写しのどこにも書けない（[SAFE-16][safety]）
- 人が呼ぶ `setup`・`check`・`/dev-autopilot`・`intake` は、入れた Plugin（`${CLAUDE_PLUGIN_ROOT}` の `plugin.json`）の版と写しの版が違えば warn に出す（[SETUP-2][setup]）。定期の実行で動く進行 Bot は、入れた Plugin の場所を知らないので、毎回の点検では、写しの中身が `program/<版>.manifest.json` の一覧とハッシュのとおりかを確かめ、違えば fail にする

### 依存

- 進行 Bot（`plugin/src/`）は、Node 24 の標準の機能（`node:` で始まるモジュール）と、`plugin/src/` の中のファイルだけを使い、実行時の npm の依存を持たない（[ADR-20261017-003][adr-plugin] の決定 1 が基本設計に残した選択）
- GitHub の読み書きは、子プロセスの `gh`（`gh api`）で行う。認証は、呼ぶたびに `GH_TOKEN` に、切り替えの前は PAT、切り替えた後はアプリのインストールのトークンを入れて渡す（[SAFE-15][safety]）。`gh` は人の手元に入れたものを使い、点検のマシンの組で版を確かめる。push と fetch は `git` で行う
- 写しに `node_modules` を作らず、`setup` は依存を入れない。npm の登録所への通信も、lifecycle script（パッケージを入れるときに走るスクリプト）も要らない
- JSON スキーマで確かめる処理（出力契約、設定）も、外の部品を使わずに `plugin/src/` の中に書く
- リポの CI で、`plugin/src/` の import が `node:` で始まるか相対パスだけであることを確かめる
- 開発の道具（検査・テスト）の依存は、リポの root の `package.json` と pnpm に置き、配らない（[STRUCT-4][struct]）

## 定期の実行の登録

![定期の実行の登録][fig-schedule]

`setup` のマシンの準備で、OS ごとの仕組みに定期の実行を登録する。進行 Bot の本体は、どの仕組みから起こされても同じに動く（[STRUCT-8][struct]）。

### 時刻

- 時刻は、0 時から `run.intervalHours` ごと（分は 0）。初期値の 24 時間なら、毎日 0 時
- `run.intervalHours` は、24 の約数（1・2・3・4・6・8・12・24）に限る。毎日同じ時刻に並べられるため。ほかの値は点検が fail にする
- マシンが眠っていた間の時刻の分は、起きたときに 1 回だけ走らせる。launchd の `StartCalendarInterval` と systemd の `OnCalendar` は、眠っていた間の分を起きたときに走らせる（`man launchd.plist`、[systemd.timer][doc-systemd-timer]）。launchd の `StartInterval` と cron は、眠っていた間の分を逃すので使わない（`man launchd.plist`）
- 実行が重ならないように、進行 Bot は実行の lock を取り、取れなければ何もせずに終わる（[運用の規則][flow-rules]）

### macOS（launchd）

| キー | 値 | 理由 |
| --- | --- | --- |
| `Label` | `dev-autopilot.<owner>.<repo>` | 当てる先のリポごとに 1 つ登録する |
| `ProgramArguments` | Node 24 の実体の絶対パス、`<devAutopilotPath>/src/main.ts`、マシンの設定の `settings.json` の絶対パス | 写しから動かす（[STRUCT-3][struct]）。マシンの設定のパスを渡す（[ADR-20261017-004][adr-human] の決定 3） |
| `StartCalendarInterval` | 上の「時刻」の一覧（`Hour` と `Minute` の組） | 眠っていた間の時刻の分を、起きたときに走らせる |
| `EnvironmentVariables` | `PATH`（`node`・`pnpm`・`git`・`gh`・`claude` の実体のフォルダと `/usr/bin:/bin:/usr/sbin:/sbin`）、`LANG`、マシンの設定の `safety.env.values` の名前と値 | launchd のジョブは人のシェルの設定を読まないので、道具の場所と、リポの設定の `safety.env.names` に足した名前の値を渡す（[設定][design-settings]の `safety.env.values`）。秘密の値は書かない（点検が確かめる。[SETUP-2][setup]） |
| `StandardOutPath`・`StandardErrorPath` | `<paths.records>/launchd.out.log`・`<paths.records>/launchd.err.log` | 進行 Bot が実行の記録を書く前に止まったときの出力を残す |
| `RunAtLoad` | `false` | 登録したときには走らせない |

- 置き場は `~/Library/LaunchAgents/<Label>.plist`。`launchctl bootstrap gui/<uid> <plist>` で登録し、登録し直すときは `launchctl bootout gui/<uid>/<Label>` の後に登録する
- 手動の 1 回実行と点検は、`launchctl kickstart gui/<uid>/<Label>` で同じジョブを起こす。`-k` を付けない（動いている最中のジョブを止めずにそのままにする。`man launchctl`、[ADR-20261015-001][adr-read] の決定 3）

### Linux（systemd のユーザーのタイマー）

| ファイル | 中身 |
| --- | --- |
| `$XDG_CONFIG_HOME/systemd/user/dev-autopilot-<owner>-<repo>.service` | `Type=oneshot`。`ExecStart=` は macOS の `ProgramArguments` と同じ 3 つ。`Environment=` は `PATH` と `LANG` と、マシンの設定の `safety.env.values` の名前と値 |
| `$XDG_CONFIG_HOME/systemd/user/dev-autopilot-<owner>-<repo>.timer` | `OnCalendar=` は上の「時刻」（24 時間なら `*-*-* 00:00:00`、6 時間なら `*-*-* 00/6:00:00`）。`Persistent=true` |

- cron でなく systemd のユーザーのタイマーにする。`OnCalendar` は眠っていた間の分を起きたときに走らせ、`Persistent=true` は電源が切れていた間の分を、次にタイマーが有効になったときに走らせる（[systemd.timer][doc-systemd-timer]）。Linux の cron が眠っていた間の分をどう扱うかは確かめていないので、根拠にしない
- ログインしていない間も動かすには、ユーザーの lingering（`loginctl enable-linger`）が要る（[loginctl][doc-loginctl]）。`setup` は、人がやることとして示す
- 手動の 1 回実行と点検は、`systemctl --user start dev-autopilot-<owner>-<repo>.service` で同じジョブを起こす
- Linux では、sandbox の振る舞いを確かめ直すまで（UNDECIDED-49）、進行 Bot を動かさない（[STRUCT-8][struct]）。それまで `setup` は Linux で定期の実行を登録せず、`setup`・`check`・`/dev-autopilot` は、Linux では動かさないことを示して終わる。点検の OS ごとの部分は、動かしてよい OS（今は macOS）でなければ fail にするので、Linux で起こされた進行 Bot も作業に入らない（STRUCT-8 の「OS に依る処理は点検の OS ごとの部分に置く」）

### 登録し直すとき

- 写しの版が変わったとき（`devAutopilotPath` が変わる）と、`run.intervalHours` か、マシンの設定の `safety.env.values` が変わったときは、`setup` で登録し直す
- 点検は、登録の中身（実行するファイルのパス、マシンの設定のパス、時刻、環境変数）が、マシンの設定と `run.intervalHours` に合うかを確かめ、ずれていれば fail にする（[STRUCT-2][struct]、[設定][design-settings]）

### 動かすマシンを替えるとき

1 つの当てる先のリポで定期の実行を登録するマシンは 1 台なので（上の「setup の流れ」）、替えるときは次の順に行う。

| 順 | すること |
| --- | --- |
| 1 | 前のマシンで、`setup` を引数 `uninstall` で実行する。実行の lock を取り、定期の実行の登録を外す。macOS は `launchctl bootout` の後に `~/Library/LaunchAgents/<Label>.plist` を消し、`launchctl print gui/<uid>/<Label>` で登録が無いことと、ファイルが無いことを読み戻す（ファイルが残ると、ログインし直したときに launchd が読み込み直しうるため。`man launchd`。実機では確かめていない）。Linux は `systemctl --user disable --now` でタイマーを止める。最後に lock を外して終わる。写し・置き場・記録・秘密は消さない |
| 2 | 前のマシンの記録の置き場を、新しいマシンの同じキーの場所へ写す。記録は上限の数え上げに使うので、失ったものとして扱わない（[ADR-20261015-006][adr-flow] の決定 4 の「マシンを移すときは、記録のフォルダも移す」）。実行の lock（`run.lock`）と依頼のファイル（`run-request.json`・`check-request.json`・`check-result-*.json`・`intake-request.json`・`intake-result-*.json`）は写さない。前のマシンの lock と依頼を、新しいマシンの実行が読まないため（[状態遷移と判定][design-flow]の「手元の記録」） |
| 3 | 新しいマシンで `setup` を実行する。秘密は、[設定][design-settings]の「秘密の置き場のファイル」の表の「動かすマシンを替えるとき」の欄のとおりに置く。印の署名の鍵の一覧（`marker-signing-keys`）は、前のマシンから一覧ごと写す（行の順も変えない）。新しいマシンで鍵を作るときは、[状態遷移と判定][design-flow]の「印の署名」の「鍵を替えるとき」のとおり、新しい鍵を先頭の行に足し、前の鍵を残す（[ADR-20261021-004][adr-marker-keys] の決定 2）。deploy key とアプリの秘密鍵（切り替えの前は PAT）は新しく作り、新しい deploy key は GitHub に登録する。`githubApp.*` もマシンの設定に書く |
| 4 | 新しいマシンで点検の fail が無いことと、鍵の取り違えの warn が無いことを確かめる（点検は、最新の印の署名が、写した一覧のどれかの鍵で通るかを確かめ、通らなければ warn にする）。新しいマシンの最初の実行で印が通った（`marker-invalid` で止まらなかった）ことを確かめた後に、前のマシンの deploy key を GitHub から消し、前のマシンのアプリの秘密鍵（切り替えの前は PAT）を GitHub で取り消す。前のマシンが戻っても、GitHub を読めず push もできずに止まる側に倒れるため。前のマシンの置き場と秘密は、人が中身を確かめてから消す |

- 前のマシンの作業場所の残骸は、新しいマシンの点検には出ない（置き場はマシンごと）。前のマシンを使い続けないなら、人が消す

## 点検

![点検][fig-check]

点検の項目は、要件の [SETUP-2][setup] が正本。基本設計は、項目を組に分け、いつ・どう確かめるかを決める。結果は fail / warn / pass で、fail が 1 つでもあれば、進行 Bot は作業に入らない（[SETUP-2][setup]）。

### 組と回数

| 組 | 主な項目（[SETUP-2][setup]） | 確かめ方 | 毎回の実行の最初 | `setup`・人の `check` |
| --- | --- | --- | --- | --- |
| 設定 | 設定ファイルの項目と型と範囲、表に無いキーが無いこと、リポの設定に絶対パスが無いこと、項目が一方にだけあること、マシンの設定の場所の一致と、`repository.*` とマシンの設定のフォルダのパスの一致、`run.intervalHours` が 24 の約数であること、`safety.env.names` に渡さない名前と固定した名前が無く、名前がすべて進行 Bot の環境変数にあること、`safety.env.values` に秘密の形が無いこと、`models.planHard` のほかのモデルに `fable` が無いこと、`safety.sandbox.toolDirs` の中身、短い一時フォルダの長さと `sessionTmp.maxBytes`、印の署名の鍵の一覧の形と、鍵の行が 1 つ以上あること（一覧は[設定][design-settings]の「`check` が設定について確かめること」） | ファイルを読む | 確かめる | 確かめる |
| GitHub | ラベル、Project の欄と自動追加、Status 欄に関わる組み込みのワークフロー 5 つが無効なこと（GitHub App に切り替えた後。上の「GitHub の準備」の表の下）、当てる先のリポの CI の秘密の置き方（GitHub App に切り替えた後。上の「GitHub の準備」の表の下）、リポの設定に書いた Project と選択肢の ID が実在すること、ruleset（上の「ruleset の形と確かめ方」。読めない欄は fail）、必須チェックの一覧、全体の止める合図の変数、origin の base ブランチと、その先端のリポの設定、base ブランチの先端に `.claude/settings.local.json` が無く、どのフォルダにも `CLAUDE.local.md` が無いこと（git の木のフォルダをたどり、名前を英字の大小を区別せずに比べる。`.Claude/settings.local.json` と `claude.local.md` にも当てる。FACT-56、FACT-60、FACT-68）、dev-autopilot 自身のリポでないこと（上の「setup の流れ」の順 1 の見分け方を、`repository.*` と base ブランチの先端の `plugin/.claude-plugin/plugin.json` で確かめる）、投稿に使うアカウントが書いた最新の未編集の印の署名が、印の署名の鍵の一覧のどれかの鍵で通ること（通らなければ warn）、一覧に 2 つ以上の鍵があるときの、先頭でない鍵でだけ確かめられた印が残る開いた Issue と PR（warn。下の箇条）、基線の必須チェック（warn） | 進行 Bot の GitHub の認証（切り替えの前は PAT、後はアプリのトークン）で読む。base ブランチの中身は進行 Bot の clone で読む | 確かめる | 確かめる |
| PAT かアプリの権限 | 下の「PAT の権限」か「GitHub App の権限」の読み取りが通ること。書き込みの権限が無いこと（リポの変数を作れないこと。切り替えの前は、ラベルも作れないこと）。切り替えた後は、アプリの権限が表のとおりで、当てる先のリポだけに入っていること（下の「GitHub App の権限」）。投稿に使うアカウントが `safety.botAccounts` にあること（[ADR-20261020-001][adr-bots]）。アプリに切り替えてあり、PAT のファイルが無いこと（切り替える前は fail。[ADR-20261020-002][adr-app]、[設定][design-settings]の「`check` が設定について確かめること」） | 読み取りと、書き込みの権限の試し（下の「PAT の権限」。403 で断られたときだけ pass） | 確かめる | 読み取りは確かめる。書き込みの権限の試しは、合図が `run` のときだけ |
| マシン | 動かしてよい OS か（今は macOS。UNDECIDED-49）、`claude` と Node 24 と `gh` の版、写しの中身が写したときの一覧とハッシュのとおりか（人が呼ぶ Skill では、入れた Plugin と写しの版の違いも warn）、写しと `controller/` が別のフォルダか、マシンの設定のフォルダとマシンの中のフォルダの実体のパス（一覧は[設定][design-settings]の「`check` が設定について確かめること」）、置き場と秘密の置き場と安全設定の置き場の権限、短い一時フォルダの根の持ち主・リンクでないこと・権限 700（根が無ければ権限 700 で作ってから確かめる。[AI のセッション][design-ai]の「短い一時フォルダ」の順 2 と「起動の前の確かめ」の順 5）、置き場の `sessions/`・置き場・置き場の上の根（`/`）までのフォルダに `CLAUDE.local.md` が無いこと（名前は英字の大小を区別せずに比べる。[AI のセッション][design-ai]の「起動の前の確かめ」の順 9、FACT-60、FACT-68）、前の実行が残したセッションを探したときに止めなかったプロセスが無いこと（`session-not-stopped` で人に渡したセッションのフォルダと短い一時フォルダの中のものは warn。下の箇条。[AI のセッション][design-ai]の「前の実行が残したセッション」の順 3）、記録と記録の目印のファイル、lock、見送った実行と起動しなかった定期の実行（warn）、作業場所の残骸、定期の実行の登録の中身と秘密の値が無いこと、進行 Bot の clone と、置き場の `sessions/` に残っている作業中のセッションの clone の objects のファイルのリンク数が 1 であること（`find <clone>/.git/objects -type f -links +1` の出力が空。ハードリンクで共有していない。FACT-19、[ADR-20261013-001][adr-workspace] の確認方法）、組み立てた安全設定の `allowRead` と `allowWrite` に共有の store が無いこと（[ADR-20261015-002][adr-store]） | ファイルとプロセスを読む。OS の部分は、点検の OS ごとの部分（[STRUCT-8][struct]） | 確かめる | 確かめる |
| 子プロセスと clone | 子プロセスに渡す環境変数に PAT・鍵・`GH_TOKEN`・`PNPM_HOME` が無いこと、objects を書き換えた clone からの fetch が失敗し、書き換えていない clone からの fetch は通ること | 下の「モデルを使わない試し」 | 確かめる | 確かめる |
| sandbox と AI のセッション | 下の「試しのセッション」の「確かめる項目」の表（読めない・書けないこと（`/tmp/claude-<uid>/` とほかのセッションのフォルダを含む）、macOS の人の一時フォルダとキャッシュと `/Users/Shared` を読めず `/Volumes` の一覧を出せないこと、自動のメモリが読み込まれないこと、`git commit` と `pnpm install` が通ること、pnpm の store、安全設定の `env` が効くこと、Read ツールの拒否（写しの `src/` を含む）と写しの `skills/` の読み取り、Write ツールで写しの `skills/` に書けないこと、Write ツールと Bash で worktree の `.claude/**` と `.mcp.json` に書けないこと、worktree の `gitdir` のファイルに書けないこと、道具の一覧に MCP の道具と `Agent`・`Task`・`Skill` が無いこと、短い一時フォルダ、試しのセッションの clone のリンク数） | 下の「試しのセッション」 | 下の契機に当たるときだけ | 合図が `run` のときだけ |

- 試しのセッションは利用枠を使うので、毎回は回さない。前の試しのセッションの記録が、次のすべてに当たるときだけ省き、前の結果（pass）をそのラウンドの sandbox の組の結果にする。1 つでも外れれば、作業の前に回す（[ADR-20261017-005][adr-mcp] と [ADR-20261017-006][adr-tmp] が基本設計に残した、確かめる回数）
  - 前の試しのセッションの結果が pass。fail・未確認・起動できなかった回の後は、毎回回し直す。前の結果を引き継いで fail のまま止まり続けず、fail を pass と読んで作業にも入らないため
  - `claude --version` の値が同じ。Claude Code の更新で、sandbox や道具の振る舞いが変わりうるため
  - 試しのセッションに渡す安全設定と環境変数の名前（リポの設定とマシンの設定の `safety.*` から組み立てたもの。セッションごとに変わるパスを除く）と、マシンの設定の `sessionTmp.*` の値のハッシュが同じ。リポの設定の PR で安全設定が変わりうり、マシンで `sessionTmp.*` を変えると一時フォルダの振る舞いが変わりうるため
  - 写しの版が同じ。エージェント定義の道具の一覧、安全設定の雛形、試しのスクリプトが変わりうるため
  - OS の版（macOS は `sw_vers -productVersion`）が同じ。sandbox は OS の仕組み（Seatbelt）なので、OS の更新で振る舞いが変わりうるため
- 回すべきときに、試しのセッションを起動できない（費用の上限に当たった、短い一時フォルダの根の異常など）ときは、sandbox の組を fail にし、作業に入らない。合図が `stop` の間だけは、下のとおり未確認の warn にする
- 記録の目印のファイル（`records-created.json`）を作った日（`setup` をした日）は、AI のセッションを起動しない決まり（[AI のセッション][design-ai]の「起動の前の確かめ」）で試しのセッションを起こせないので、fail でなく、理由を添えた未確認の warn にする（「`setup` をした日は AI のセッションを起動しない。最初の実行は次の日から」）。`setup` をした日は、ほかの AI のセッションも起動しないので、試しを済ませないまま AI のセッションが動くことは無い。次の日の最初の実行が、作業の前に試しを回す（前の結果が pass でないため）
- 全体の止める合図が `stop` の間は、試しのセッションと書き込みの権限の試しを行わず、未確認として warn に出す。`run` を読めない間は、AI のセッションを起動せず、GitHub に書き込まないため（[ADR-20261017-007][adr-stop] の決定 2）。人は、fail が無くなってから合図を `run` にする。[ADR-20261017-007][adr-stop] の決定 1 は「`check` が pass になった後に」とするが、`stop` の間は 2 つの試しが未確認の warn になり pass にならないので、2 つを除いて fail が無いことを条件に読む。合図を `run` にした後の最初の実行が、作業の前に 2 つを確かめ、fail なら作業に入らない。書かずに試すことは持たないので、最初の実行から GitHub に書く（[ADR-20261024-002][adr-no-stages] の決定 2）。設定と導入を確かめ終わるまでは、合図を `stop` にしておく
- 作業場所の残骸は、作業の単位に紐づく残骸を、紐づく作業の単位だけの停止中（`workspace-residue`）か、Project に無い Issue の `stop` の印にし、作業の単位に紐づかない残骸と、片付けの後に残った残骸を fail にする（[SETUP-2][setup]）。見分け方は [状態遷移と判定][design-flow]の「作業場所の残骸」
- 前の実行が残したセッションの順 3（置き場の `sessions/` の下と、短い一時フォルダの根の下の全体の探し方）で見つけたプロセスのうち、`session-not-stopped` で人に渡したセッション（実行の中の止める手順か、前の実行が残したセッションの順 2 で止められず、作業の単位を停止中にしたことを終わりの行に書いたもの）のセッションのフォルダか短い一時フォルダの中のものは、warn に出す。止められなかったセッションの作業の単位は停止中で止まり、ほかの作業の単位は続ける。ほかのプロセスは、どの作業の単位のものか分からないので fail にする（[SETUP-2][setup]、[IMPL-14][impl]）。受け入れの依頼の実行のセッション（人が `/verify-issue` から呼んだ intake の Skill の依頼で起こしたもの）の残りは、作業の単位を停止中にせず、終わりの行も書かないので、人が残りを止めるまで fail にする（[ADR-20261022-004][adr-intake-req] の決定 2）。探し方と記録は [AI のセッション][design-ai]の「前の実行が残したセッション」
- 印の署名の鍵の一覧に 2 つ以上の鍵があるとき（鍵を替える途中。[状態遷移と判定][design-flow]の「印の署名」の「鍵を替えるとき」）は、当てる先のリポの開いている Issue と PR の、`safety.botAccounts` のアカウントが書いた印を一覧の鍵で確かめ、先頭でない鍵でだけ確かめられた印が残る Issue と PR の番号を、鍵の行の番号ごとに warn に出す。鍵は行の番号で示し、鍵の値は出さない。先頭でない鍵のうち、印が残る Issue と PR が無くなった鍵は、一覧から外してよいことを warn に出す。外し忘れて、印を作れる鍵が多いまま残らないため（[ADR-20261021-004][adr-marker-keys] の決定 2）。一覧に鍵が 1 つのときは、読まない
- 基線が緑でなければ、緑でない必須チェックの名前を warn に出す。新しい着手（計画の輪）・最初の変更の作成・載せ直しは止まるが、停止中にしないので、人が気づく場所に出す（[IMPL-9][impl]、[ADR-20261015-006][adr-flow] の決定 2）

### 見送った実行と、起動しなかった定期の実行

lock で見送った実行と、マシンのスリープや停止で起動しなかった定期の実行を、記録に残して warn に出す（[運用の規則][flow-rules]）。並列化（段階 9）に入る条件の「連続 7 回の実行」（[FLOW-12][flow-rules]）は、2 つと、受け入れの依頼で起こされた実行を除いて数える（[運用の規則][flow-rules]、[状態遷移と判定][design-flow]の「運用の記録の数え方」）。

- lock で見送った実行は、lock を取れなかった実行が、見送ったことを記録に残す（[状態遷移と判定][design-flow]の「1 回の実行の流れ」の順 2）
- 実行の記録に、起こされ方（定期・手動・点検の依頼・受け入れの依頼）を残す。人が呼ぶ `/dev-autopilot` の手動の 1 回実行は、ジョブを起こす前に記録の置き場に `run-request.json` を置き、`check` は `check-request.json` を置き、`/verify-issue` などから呼ぶ intake の Skill の下書きは `intake-request.json` を置く。`/dev-autopilot` は、lock を見て実行の最中なら、依頼を置かずに、実行中であることを示して終わる（動いている実行は依頼のファイルを読み終えているので、置いた依頼が次の定期の実行に読まれ、定期の実行を「手動」と記録するため。下の「状態の表示」）。intake の Skill も、lock を見て進行 Bot の実行か `setup` の最中なら、依頼を置かずに、最中であることを示して止まる。待つ時間の内の受け入れの依頼がすでにあれば、置かずに、ほかの依頼を待っていることを示して止まる（置ける受け入れの依頼は 1 つ。[ADR-20261022-004][adr-intake-req] の決定 1）
- 進行 Bot は、lock を取った後に 3 つの依頼のファイルを見て起こされ方を決める。どれも無ければ定期。点検の依頼があれば、点検の依頼を先にして点検だけを行う。手動の依頼が点検の依頼と重なれば、手動の依頼は消して、見送ったことを記録と状態の表示に出す（人は `/dev-autopilot` で頼み直す）。受け入れの依頼が、点検の依頼か手動の依頼と重なれば、受け入れの依頼は、受付 AI を起こさずに消し、見送った理由を結果のファイル（`intake-result-<依頼の ID>.json`）に書く（人は intake の Skill を呼び直す）。受け入れの依頼だけなら、受け入れの依頼の実行として、受付 AI のセッションを 1 つ起こす（[AI のセッション][design-ai]の「人の Skill から受け入れを呼ぶ形」、[ADR-20261022-004][adr-intake-req] の決定 2）。`run-request.json` は読んだら消し、`check-request.json` は下の「点検だけで動かす仕組み」のとおり、`intake-request.json` は受け入れの依頼の結果を書いた後に消す。どれも、読んだ依頼と ID が同じときだけ消す（読んだ後に人が置き直した依頼を消さないため）
- 進行 Bot は、lock を取った後に見つけた点検の依頼と受け入れの依頼のうち、置いた時刻から、依頼のファイルに書いた待つ時間（人の Skill が待つ時間。`run.checkWaitMinutes` か `run.intakeWaitMinutes` の値）を過ぎたもの（人の Skill が待つのをやめた後に残ったもの）を、次のとおりに扱う。実行の起こされ方は定期とし、並列化に入る条件の実行の回数に数える（[ADR-20261022-004][adr-intake-req] の決定 5）
  - 受け入れの依頼：受付 AI を起こさずに消し、消したことを実行の記録に残す。人は待つのをやめていて、結果を読まないため
  - 点検の依頼：点検の結果を書き、依頼のファイルを消してから、定期の実行として作業を続ける。点検の依頼があれば点検だけを行って作業に入らない決まりは、待つ時間の内の依頼に限る。進行 Bot の実行か `setup` が lock を持つ間に人が `check` を呼んで残った点検の依頼で、次の定期の実行が 1 回分の作業を失わないため
  - 待つ時間を依頼のファイルに書くのは、起こされ方を、リポの設定を読む（「[状態遷移と判定][design-flow]」の「1 回の実行の流れ」の順 4）前の順 2 で決めるため
- 起動しなかった定期の実行は、進行 Bot が、lock を取った後に、前の実行の起動時刻から今までの定期の実行の時刻（上の「定期の実行の登録」の「時刻」）を並べて見つける。時刻ごとに、次の時刻までの間に、起こされ方が定期の実行が始まっていなければ、起動しなかった時刻として記録に残す（同じ間に人が手で起こした実行があっても、定期の実行の代わりに数えない）。眠っていた間の分を起きたときに走らせた実行は、起きた時刻の直前の時刻の分に数える。眠っていた間のほかの時刻は、起動しなかった時刻になる
- 点検は、直近 7 日の分を warn に出す。7 日は、並列化（段階 9）に入る条件が連続 7 回の実行を数えるため（[FLOW-12][flow-rules]）

### 点検だけで動かす仕組み

人が呼ぶ `check` は、進行 Bot を直接起動せず、定期の実行と同じジョブを起こす。GitHub の点検は進行 Bot の GitHub の認証で行い、人のセッションは結果を読んで表示する（[SETUP-2][setup]、[ADR-20261017-004][adr-human] の決定 2）。点検だけで動かすことは、記録の置き場に置く依頼のファイルで伝える（[ADR-20261017-004][adr-human] が基本設計に残した仕組み）。

| 順 | すること |
| --- | --- |
| 1 | 人の `check` が、origin からマシンの設定を見つけ、記録の置き場に依頼のファイル（`check-request.json`。依頼の ID、置いた時刻、待つ時間（`run.checkWaitMinutes` の値）。形は「[状態遷移と判定][design-flow]」の「手元の記録」）を置く。依頼のファイルを書くコマンドは、sandbox の外で動かす（表の下） |
| 2 | 人の `check` が、定期の実行と同じジョブを起こす（上の「定期の実行の登録」の手動の 1 回実行） |
| 3 | 進行 Bot が、実行の lock を取った後、待つ時間の内の依頼のファイルがあれば、点検だけを行い、作業に入らない（待つ時間を過ぎた依頼は、上の「見送った実行と、起動しなかった定期の実行」）。全体の止める合図が `stop` でも、AI のセッションを起動せず GitHub に書き込まない項目は行う（人は点検の fail が無くなってから合図を `run` にするため）。試しのセッションと書き込みの権限の試しは、合図が `run` のときだけ行い、`stop` の間は未確認の warn として結果に書く（[ADR-20261017-007][adr-stop] の決定 2）。リポの設定を読めないなど、点検の途中で止まったときも、止まった項目を fail として結果に書く |
| 4 | 進行 Bot が、結果を `check-result-<依頼の ID>.json` に書き、依頼のファイルを消して終わる（読んだ依頼と ID が同じときだけ消す） |
| 5 | 人の `check` が、結果のファイルを `run.checkWaitMinutes`（15 分）まで待って表示し、表示した後に結果のファイルを消す（消すコマンドは sandbox の外で動かす。表の下）。来なければ待つのをやめ、実行中か（lock を見る）、ジョブが起きなかったか（定期の実行の登録と `launchd.err.log`）を示して終わる |

- ジョブが動いている最中なら、手動の 1 回実行は新しく起こさない（`kickstart` に `-k` を付けない）。進行 Bot は、実行の終わり（「[状態遷移と判定][design-flow]」の「1 回の実行の流れ」の順 10）にも依頼のファイルを見て、あれば点検だけを行う。人の `check` は、実行中であることを示して待ち、`run.checkWaitMinutes` を過ぎたら、終わった後に `check` を実行し直すか、`/dev-autopilot` で結果を見るように示す
- 進行 Bot がマシンの設定を読めずに止まったときは、記録の置き場が分からず、結果を書けない。人の `check` は、待つ時間を過ぎたら、`launchd.err.log` を見るように示す
- 依頼と結果のファイルは記録の置き場にあり、AI は sandbox の中から書けない
- 人の Claude Code が sandbox を有効にしていると、Bash の道具から記録の置き場（作業フォルダの外）に書けない（FACT-71）。人が呼ぶ Skill のうち、記録の置き場に依頼のファイルを置くもの（`check`、`/dev-autopilot` の手動の 1 回実行、`/verify-issue` などから呼ぶ intake の Skill の下書き）は、依頼のファイル（`check-request.json`・`run-request.json`・`intake-request.json`）を置くコマンドと、示した後に結果のファイル（`check-result-<依頼の ID>.json`・`intake-result-<依頼の ID>.json`）を消すコマンドを、sandbox の外で動かす許可を人に求めてから動かす（[ADR-20261022-004][adr-intake-req] の決定 6）。人の設定が sandbox の外でコマンドを動かすことを許していない（`sandbox.allowUnsandboxedCommands` が `false`）か、人が許可しなければ、依頼を置かず、ジョブも起こさずに、sandbox の中からは記録の置き場に書けないので依頼を出せないことを理由として示して止まる
- sandbox の外で動かすのは、記録の置き場に書くコマンドだけにする。結果を待つことと表示することは、sandbox の中で行う。許可を求めるコマンドは、書く先のパスと書く中身が、人の読むコマンドの文に見える形にする。人が、記録の置き場に書くだけのコマンドかを読んで許せるようにするため（[ADR-20261022-004][adr-intake-req] の決定 6）
- ジョブを起こす `launchctl kickstart`（`-k` なし）と、lock と結果のファイルを読むことを、sandbox の中の Bash の道具から使えるかは、確かめていない（ジョブを起こすほうは要件の UNDECIDED-75。人が試す）。使えないと分かったコマンドにも、同じ扱い（sandbox の外で動かす許可を人に求め、許されなければ理由を示して止まる）を掛ける。導入と点検の実装（段階 3）で、sandbox を有効にした人の Claude Code から試す（[ADR-20261022-004][adr-intake-req] の確認方法）

### モデルを使わない試し

毎回の実行の最初に、進行 Bot が、AI のセッションを起動せずに試す（[SETUP-2][setup] の「進行 Bot は毎回の実行の最初に同じ点検を回し」）。リポの設定の PR で環境変数の名前が変わっても、次の実行で確かめるため。

- 子プロセスの環境変数：AI のセッションを起動するときと同じ処理で環境変数を組み立てる。名前が、決めた名前と `safety.env.names` の足す名前だけであること、`GH_TOKEN` と `PNPM_HOME` が無いこと、値に PAT かアプリの秘密鍵とほかの鍵の中身と、秘密の置き場のパスが無いことを確かめる（[SAFE-1][safety]、[SAFE-10][safety]、[AI のセッション][design-ai]の「環境変数」）
- objects を書き換えた clone からの fetch：一時フォルダで、次の順に試す（[ADR-20261013-001][adr-workspace] の確認方法。ローカルのパスからの fetch でも失敗することは FACT-57、clone に commit を足さないと通ることは FACT-61）。一時フォルダは試しの後に消す
  - 試しのためだけの小さなリポ（commit 1 つ）を作り、`--no-hardlinks` で clone を 2 つ作る（書き換える clone と、対照の clone）。ハードリンクで objects を共有すると、書き換えが小さなリポにも及ぶため（FACT-19）
  - どちらの clone にも、中身の違う commit を 1 つずつ足す。書き換える clone では、足した commit の blob の loose object のファイルを書き換える。clone が小さなリポと同じ commit しか持たないと、fetch は objects を送らずに通るため（FACT-61）
  - 小さなリポに、進行 Bot が取り込み（「[状態遷移と判定][design-flow]」の「push の手順」の順 2）で使うのと同じ形（`git fetch <パス> <ブランチ>`）と、同じ環境変数（`GIT_CONFIG_GLOBAL` などの git の設定）と `core.hooksPath` で、書き換えた clone から fetch する。終了コードが 0 でなく、`FETCH_HEAD` が無いか空であることを確かめる（FACT-61）
  - 続けて、対照の clone から同じ形で fetch し、終了コードが 0 で `FETCH_HEAD` が空でないことを確かめる。対照が通らなければ、パスや設定の誤りで書き換えに依らずに失敗しているので fail にする（失敗したことだけで pass と読まない）
  - 進行 Bot の clone を試しに使わない。進行 Bot の clone から clone すると objects のファイルがハードリンクで共有され、書き換えが進行 Bot の clone に及ぶうえ、試しが通ってしまったときに、書き換えた objects を取り込むため
- sandbox が読み書きを止めることと、道具の一覧は、モデルを使わずには試せないので、試しのセッションで確かめる

### 試しのセッション

- AI のセッションと同じ起動の形（安全設定、環境変数、引数。[AI のセッション][design-ai]）で、短いセッションを 1 つ起こす。名前は `check-<日時>`（[AI のセッション][design-ai]の「セッションの名前」）
- エージェント定義は、作業 AI（`dev-autopilot:change:worker`）を使う。道具がいちばん広い役で、境界を試すため
- モデルは `claude-haiku-5-5`（`haiku` の別名は旧版に解けるので、完全な ID で書く。[IMPL-29][impl]）、ターンの上限は 10、時間の上限は 10 分、費用の上限は `limits.session.costUsd`（試しのスクリプトの実行と 2 つの読み取りは、4 ターン・7 秒で終わった。FACT-49）。モデルとターンと時間の上限は、試しの中身と一緒に Plugin の版で変えるので、設定に持たず、進行 Bot のプログラムに書く
- 作業場所は、ほかの役と同じく、当てる先のリポを origin から clone し、進行 Bot の clone で控えた base ブランチの先端の SHA で worktree を切る（作業 AI と同じ。[AI のセッション][design-ai]の「役とエージェント定義」）。`pnpm install` も当てる先のリポのロックファイルで試す。本物のセッションと同じ条件で確かめるため。clone を作った後に、`find <clone>/.git/objects -type f -links +1` の出力が空であることを確かめ、空でなければ fail にする（AI の作業場所の clone が、objects をハードリンクで共有していないこと。[SETUP-2][setup]、[ADR-20261013-001][adr-workspace] の確認方法）。GitHub の組の、base ブランチの先端に `.claude/settings.local.json` と `CLAUDE.local.md` が無いことの確かめが fail なら、試しのセッションを起こさない（試しの worktree も base ブランチの先端で切るので、local の設定と指示を読むセッションになるため）。sandbox の組は、試しのセッションを起動できないときとして fail にする（上の「組と回数」）
- 進行 Bot は、セッションを起こす前に、ほかのセッションの代わりの試験用のフォルダ 2 つ（`<置き場>/sessions/check-other-<日時>/` と `<sessionTmp.root>/check-other-<日時>/`）と、Claude Code のユーザーごとの一時フォルダの中の試験用のファイル（`/tmp/claude-<uid>/dap-check-probe-<日時>.txt`）を作る。中身は、どれも意味の無い文字列のファイル 1 つ。名前と ID の組を記録に残してから作り、試しの後に消す（残骸と見分けるため）
- 進行 Bot が、試しのスクリプト（Node で動く。`plugin/src/` の中に持つ）をセッションのフォルダに置き、AI に決めたコマンドで 1 回だけ実行させる。スクリプトは、下の表の項目を順に試し、結果を標準出力に書く。結果には、読めたかどうかだけを書き、読めたファイルの中身を書かない。結果をファイルに書かないのは、AI が書けるファイル（セッションの `tmp/` と worktree）を判定の元にしないため
- Read ツールと Write ツールの試しは、AI が道具を呼ばないと行えないので、AI に、決めたパスを Read ツールで 1 つずつ読ませ、決めたパスに Write ツールで書かせる。Read ツールで読ませるのは、進行 Bot がセッションのフォルダの外（`<置き場>/check-probe.txt`。中身は意味の無い文字列）に置いた試験用のファイルと、写しの `src/` の中のファイルと、写しの `skills/` の中のファイル。Write ツールで書かせるのは、写しの `skills/` の中の、まだ無いファイルの名前（[ADR-20261017-004][adr-human] の確認方法）。本物の秘密のファイルは Read ツールで読ませない（拒否されなかったとき、中身が AI と記録に渡るため）
- 確かめる項目と、期待する結果と、判定の元

  | 項目 | 期待する結果 | 判定の元 |
  | --- | --- | --- |
  | ホームのファイル、`<置き場>/controller/`、写しの `src/`、試験用のほかのセッションのフォルダ 2 つ、`/tmp/claude-<uid>/` の試験用のファイル、秘密の置き場、記録の置き場を Bash で読む | すべて読めない | Bash の結果 |
  | macOS の人の一時フォルダとキャッシュのフォルダ（`getconf DARWIN_USER_TEMP_DIR` と `DARWIN_USER_CACHE_DIR`）と `/Users/Shared` に、進行 Bot が置いた試験用のファイル（試しの後に消す）を Bash で読む。`/Volumes` の一覧を出す | すべて読めず、一覧も出せない（FACT-63） | Bash の結果 |
  | 自分の clone と worktree、写しの `skills/` を Bash で読む | 読める | Bash の結果 |
  | 写し、`<置き場>/controller/`、worktree の `gitdir` のファイル、ホーム、試験用のほかのセッションのフォルダ 2 つ、`/tmp/claude-<uid>/` に Bash で書く | すべて書けない（ファイルができない） | Bash の結果と、進行 Bot がディスクで確かめた有無 |
  | `$TMPDIR` | `<短い一時フォルダ>/claude-<uid>` で、書ける | Bash の結果 |
  | `/tmp/claude-<uid>/` の一覧を出す | 出せない | Bash の結果 |
  | worktree で `git commit --allow-empty`、`pnpm install --frozen-lockfile`、`pnpm store path` | commit と install が通り、store のパスがセッションの `tmp/` の下 | Bash の結果 |
  | 安全設定の `env`（`NODE_USE_ENV_PROXY`、`npm config get cache`） | `1` と、`$TMPDIR` の下（FACT-47） | Bash の結果 |
  | 試験用のファイル、写しの `src/` の中のファイル、写しの `skills/` の中のファイルを Read ツールで読む | 前の 2 つは拒否、`skills/` の中のファイルは読める | `stream-json` の Read の結果と `permission_denials` |
  | 写しの `skills/` の中に Write ツールで書く | 拒否（ファイルができない） | `stream-json` の Write の結果と `permission_denials`、進行 Bot がディスクで確かめた有無 |
  | worktree の `.claude/settings.json`・`.claude/skills/` の下・`.mcp.json` に、Write ツールと Bash で書く | すべて書けない（Claude Code の守り。FACT-77、[ADR-20261027-003][adr-claude-dir]） | `stream-json` の Write と Bash の結果と `permission_denials`、進行 Bot がディスクで確かめた有無 |
  | 自動のメモリ。進行 Bot が、試しのセッションの作業フォルダに対応する `~/.claude/projects/<作業フォルダの実体のパスの / と . を - にしたもの>/memory/MEMORY.md` に合言葉を置き（試しの後にファイルだけ消す）、固定の指示文に書いた対照の合言葉と一緒に、文脈にある合言葉をすべて返させる | メモリの合言葉は返らず、対照の合言葉は返る（FACT-62） | `stream-json` の AI の文（`assistant` のメッセージと `result`） |
  | 道具の一覧と MCP のサーバーの一覧 | `mcp__` の道具と `Agent`・`Task`・`Skill` が無く、サーバーが空 | `init` のイベント |

- 進行 Bot は、AI の返事ではなく、次で判定する
  - Bash の結果：`stream-json` の Bash の呼び出し（`tool_use`）が決めたコマンド（試しのスクリプトの実行）であることを確かめ、Bash の呼び出しの結果（`tool_result`。Claude Code がコマンドを実行して返したもの）を読む。あわせて、書けないはずの場所にファイルができていないことを、進行 Bot がディスクで確かめる
  - Read ツール：`stream-json` の Read の呼び出しと結果を照らし、試験用のファイルと写しの `src/` の中のファイルが `result` の `permission_denials` にあること、写しの `skills/` の中のファイルの結果がエラーでないこと（[ADR-20261015-001][adr-read] の確認方法、[SETUP-2][setup]。拒否が `permission_denials` に載ることは FACT-49 で確かめた）
  - Write ツール：写しの `skills/` への Write の呼び出しが `permission_denials` にあり、ファイルができていないこと（[ADR-20261017-004][adr-human] の確認方法）
  - 自動のメモリ：AI の文で判定する例外。メモリの中身が AI の文脈に入ったかは、AI の文でしか見られないので、AI の文にメモリの合言葉が無く、対照の合言葉があることで判定する。対照の合言葉が返らなければ、AI が指示どおりに返したと言えないので fail にする（FACT-62 と同じ確かめ方）
  - 道具の一覧：初期化の記録（`init` のイベント）の一覧に、`mcp__` で始まる道具と `Agent`・`Task`・`Skill` が無く、MCP のサーバーの一覧が空であること（[ADR-20261017-005][adr-mcp] の決定 4。Agent は初期化の記録で `Task` の名前で出るので、両方を探す。[ADR-20261017-002][adr-review] と [ADR-20261021-001][adr-review-skill] の確認方法）。道具の名前は、名前の全体が一致するかで探す。一覧には `TaskStop` という別の道具があり、名前の前の部分で探すと取り違えるため（FACT-49）
- AI が決めたコマンドを実行しなかったか、決めたコマンドと違う Bash を実行したとき、決めたパスを読まなかったか書かなかったとき、ターンか時間か費用の上限に当たったとき、出力が解析できないときは fail にする。確かめられなかったものを pass と読まないため
- 終わったら、「止め方と利用枠の数え方」の手順で残りを確かめ（[AI のセッション][design-ai]）、セッションのフォルダと試験用のフォルダとファイルを消す。費用は実行の記録に残し、1 回の実行と 1 日の累計に数える（[設定][design-settings]の「費用の上限の値」）。結果は、上の「組と回数」の省く条件に使う値と一緒に記録に残す（残す欄は、[状態遷移と判定][design-flow]の「手元の記録」の `sessions.jsonl` が正本）
- 試しのセッションのフォルダが消せずに残ったら（途中で落ちた、残りが止まったと確かめられない）、名前から Issue か PR を引けない残骸として扱い、点検を fail にする。人が中身を確かめて消す（[状態遷移と判定][design-flow]の「作業場所の残骸」）

### PAT の権限

GitHub App に切り替えるまで（導入の間）、進行 Bot は fine-grained PAT で GitHub を読み、点検を回す。PAT のままでは点検が fail になる（上の「組と回数」）ので、作業には入らない。切り替えた後は、下の「GitHub App の権限」による（[ADR-20261020-002][adr-app]、[ADR-20261024-002][adr-no-stages] の決定 2）。

進行 Bot の fine-grained PAT は、当てる先のリポだけに絞り、次の権限だけを付ける（[ADR-20261017-004][adr-human] が基本設計に残した、GitHub の点検に要る読み取りの権限を含む）。権限の名前は [GitHub の公式文書][doc-pat-permissions] による。

| 権限 | 読み取りか書き込みか | 使うところ |
| --- | --- | --- |
| Metadata（リポ） | 読み取り | ruleset を読む（`GET /repos/{owner}/{repo}/rulesets` は Metadata の読み取りで通る） |
| Issues（リポ） | 読み取り | Issue、ラベル、コメント、timeline を読む。書くのはアプリに切り替えた後 |
| Pull requests（リポ） | 読み取り | PR と、PR のコメントを読む |
| Commit statuses（リポ） | 読み取り | CI の結果（コミットの状態）。結合した状態（`/status`）は、状態が 0 件でも `pending` を返すので、CI が緑かの判定には使わず、必須チェックを名前で照らす（FACT-53） |
| Actions（リポ） | 読み取り | ワークフローの実行（人の承認が要るものを見つける。[IMPL-27][impl]） |
| Variables（リポ） | 読み取り | 全体の止める合図を読む。書き込みは付けない（[ADR-20261017-007][adr-stop]） |
| Projects（Organization） | 読み取り | Project の項目と表示の欄を読む（FACT-6）。表示の欄は判定に読まず、点検で欄と選択肢が実在するかを確かめる |

- PAT を使うのは、GitHub App に切り替える前の点検の読み取りだけなので、書き込みの権限を付けない（[ADR-20261020-002][adr-app] の決定 1）。進行 Bot に誤りがあっても、メンテナのアカウントで印を書けない。書き込みを付けるのは、アプリの権限（下の「GitHub App の権限」）だけ
- Administration（ruleset を作る管理の権限）と Contents は付けない。ruleset は人の `setup` が作り（[SETUP-1][setup]）、push は deploy key で行う（[SAFE-15][safety]）
- CI のチェックの実行（check runs）は、fine-grained PAT に Checks の権限が無くても、公開のリポなら読める（FACT-52）。非公開のリポでは読めない（403。FACT-54）。非公開のリポは、GitHub App に切り替えた後に、アプリの Checks の読み取りで読む（[ADR-20261020-002][adr-app]）
- 書き込みの権限の試し：点検用の名前の変数（`DEV_AUTOPILOT_PROBE`）を作る試しと、切り替えの前は、点検用の名前のラベル（`dev-autopilot-probe`）を作る試しを行う。試しの前に、点検用の変数とラベルが無いことを読んで確かめ、あれば fail にする（前の試しで消し損ねた名前があると、書けても「すでにある」で断られ、書けないと読み誤るため）。403 で断られたときだけ pass。作れてしまったら fail にし、作った変数かラベルを消す。ほかの応答（404・409・422・5xx・時間切れなど）は、確かめられなかったとして fail にする（未確認を pass と読まない。合図が `stop` の間の未確認の warn とは別）。全体の止める合図の変数には書き込まない（読んだ後に人が `stop` に変えると、書き戻しで `run` に戻してしまうため）
- 書き込みの権限の試しは、書けないことを確かめる試し。アプリはラベルを作れるので、切り替えた後は変数の試しだけを行う

### GitHub App の権限

進行 Bot が当てる先のリポに初めて書く前に、GitHub App に切り替える（[ADR-20261024-002][adr-no-stages] の決定 2）。印を書く前に替えるので、PAT のアカウントで書いた印は無く、読み続ける手当ては要らない（[ADR-20261020-002][adr-app]、[ADR-20261020-001][adr-bots]）。

| 権限 | 読み取りか書き込みか | 使うところ |
| --- | --- | --- |
| Metadata（リポ） | 読み取り | ruleset などを読む |
| Issues（リポ） | 書き込み | ラベル、印とコメント、🟢 から Issue を作る |
| Pull requests（リポ） | 書き込み | PR を作る、ready にする、PR にコメントする |
| Checks（リポ） | 読み取り | CI のチェックの実行（非公開のリポを含む。[IMPL-27][impl]） |
| Actions（リポ） | 読み取り | ワークフローの実行（[IMPL-27][impl]） |
| Commit statuses（リポ） | 読み取り | コミットの状態（結合した状態は判定に使わない。FACT-53） |
| Variables（リポ） | 読み取り | 全体の止める合図を読む。書き込みは付けない（[ADR-20261017-007][adr-stop]） |
| Projects（組織） | 書き込み | Project の表示の欄 |
| Secrets（リポ） | 読み取り | 当てる先のリポの秘密と、リポに届く組織の秘密の名前を読み、点検で CI の秘密の置き方を確かめる（値は読めない。上の「GitHub の準備」の表の下、[ADR-20261028-002][adr-app-secrets]） |

- Contents の書き込みと Administration は付けない。push は deploy key で行い（[IMPL-15][impl]）、ruleset は人の `setup` が作る（[SETUP-1][setup]）。Webhook は使わない
- 点検は、アプリのインストールの情報（`GET /app/installations/{installation_id}`。秘密鍵で作った JWT で読む）の `permissions` を上の表と比べ、表に無い権限か、表より強い権限（表で読み取りとした権限が書き込み）があれば fail にする。`repository_selection` が `selected` でないか、インストールが読めるリポ（`GET /installation/repositories`。Metadata の読み取りだけを要求し、リポを絞らずに作ったトークンで読む）に当てる先のリポのほかがあれば fail にする。[ADR-20261020-002][adr-app] の決定 2（当てる先のリポだけに入れ、権限は毎回の実行に要るものだけ）を、人の記憶でなく点検で守るため
- 進行 Bot が毎回の実行で使うトークンは、表の権限と当てる先のリポだけを要求して作る（要求の `permissions` と `repositories` で絞る。アプリに無い権限は、要求しても付かない。[GitHub の公式文書][doc-app-token]）。アプリに広い権限が付いていても、毎回の実行のトークンは表より広くならない（重ねる守り）
- アプリの登録、当てる先のリポへの導入、秘密鍵の作成は、人が行う。`setup` は手順を示し、点検が確かめる（[SETUP-1][setup] の「人がやるもの」）
- 秘密鍵のファイル（`github-app-key.pem`）は秘密の置き場に置く（権限 600）。アプリの ID とインストールの ID は、マシンの設定の `githubApp.appId` と `githubApp.installationId`（[設定][design-settings]。全体の止める合図を読む前に要るため）
- 進行 Bot は、実行のたびに秘密鍵で JWT（アプリが自分を証明する、署名付きの短い文字列）を作り、1 時間で切れるインストールのトークンに引き換えて使う。`gh` を呼ぶ前に残りの時間を見て、10 分を切っていれば作り直す（AI のセッションの時間の上限 45 分の間に読む全体の止める合図の読み取りでも切れないように、余裕を持たせる）。10 分は GitHub のトークンの有効期間で決まる値なので、設定に持たず、プログラムに書く。Node の標準の機能（`node:crypto`）で作り、npm の依存を足さない（上の「依存」）
- アプリの投稿者の名前は、REST の `login` では `<アプリの名前>[bot]` になり、GraphQL では形が違うことがある（確かめていない）。取り方と比べ方は、切り替えの前の試し（下の切り替えの順の 2）で決め、決めた形を入れた版の Plugin（順の 3）で、点検と印を読む処理を同じ形にそろえる（[ADR-20261020-002][adr-app] の決定 4 と確認方法）
- 切り替えの順

  | 順 | すること |
  | --- | --- |
  | 1 | 人がアプリを登録して当てる先のリポに導入し、秘密鍵を秘密の置き場の `github-app-key.pem` に置く。試しのためだけのリポ（公開と非公開の 2 つ）にも、同じアプリを一時的に導入する |
  | 2 | 一度だけ、試しのためだけのリポと Project で、登録したアプリを使い、[ADR-20261020-002][adr-app] の確認方法の 5 つと、ruleset を読めるか（要件の UNDECIDED-69）を確かめて、結果を FACT に残す：Issue と PR のコメント・ラベル・Project の表示の欄を書けること、止める合図の変数を読めて作成・更新・削除が断られること、公開と非公開のリポでチェックの実行を読めること、コメントの投稿者の名前の形と進行 Bot が名前を取る方法、インストールのトークンが 1 時間で切れて作り直せること、インストールのトークンで、3 つの ruleset の `bypass_actors` と base ブランチに効く規則（`GET /repos/{owner}/{repo}/rules/branches/{baseBranch}`）を読めること（試しのリポには、前もって上の「ruleset の形と確かめ方」と同じ形の ruleset を 3 つ作っておく）。ruleset の欄を読めなかったときの扱いは、試しの結果を見て ADR で決める。ADR で決めるまでは、点検が読めない欄を fail にするので、順 5 を通せない（上の「ruleset の形と確かめ方」）。試しの後に、試しのリポからアプリを外す（当てる先のリポだけに入れる。[ADR-20261020-002][adr-app] の決定 2） |
  | 3 | 順 2 で決めた投稿者の名前の取り方と比べ方を入れた版の Plugin を入れ、`setup` を実行し直して、写しを名前の取り方を入れた版にする。点検と印を読む処理が、確かめた形で名前を比べるため |
  | 4 | 人が、進行 Bot を動かすマシンの設定に `githubApp.*` を書く。`safety.botAccounts` をアプリの投稿者の名前にする PR を、リポの設定に出してマージする |
  | 5 | 人が GitHub で PAT を削除し、秘密の置き場の `github-pat` を消す。PAT のファイルが残っている間は、点検が fail にする |
  | 6 | 当てる先のリポでは、全体の止める合図を `stop` にしたまま、点検の読み取りと、アプリの権限とリポの範囲の確かめを通す。書き込みの権限の試しは、合図が `stop` の間は未確認の warn になる（上の「組と回数」）。書けることは、合図を `run` にした後の最初の実行の書き込みと読み戻し（「[状態遷移と判定][design-flow]」の「push の手順」の順 8 など）で確かめる |
  | 7 | 人が、点検の fail が無くなってから、全体の止める合図を `run` にする |

- 点検は、アプリに切り替えていない（`githubApp.*` が `null` か、秘密鍵が無いか、PAT のファイルが残っている）とき fail にする。書き始める前に替える決まりを、人の記憶でなく点検で守るため

## 状態の表示

人が呼ぶ `/dev-autopilot` は、状態の表示と、手動の 1 回実行を行う（[STRUCT-3][struct] の構成）。進行 Bot を直接起動せず（手動の 1 回実行は、`run-request.json` を置いてから同じジョブを起こす）、GitHub は進行 Bot が自分の GitHub の認証で読む（[ADR-20261017-004][adr-human] の決定 2）。

- 進行 Bot は、毎回の実行の終わりに、記録の置き場の `status.json` を書き直す。中身は、実行の時刻、全体の止める合図の値と全体の停止の Issue があるか、点検の fail と warn、基線の結果（緑でない必須チェックの名前）、作業の単位ごとの主な流れの状態と重ねる 3 本の軸の値・次の工程・止めた理由（`stop` の印の記号）・外れた着手の条件、見送った手動の依頼（上の「見送った実行と、起動しなかった定期の実行」）。点検だけの実行は、点検の結果だけを書き直す
- 人の番の作業の単位（主な流れの状態が下書き済みかレビュー合格か、止め方・安全・先の依頼のどれかに値がある。[FLOW-39][flow-rules] のビューの例と同じ条件）は、「人の番」として分けて出す
- `/dev-autopilot` は、マシンの設定（[設定][design-settings]の「マシンの設定」）から記録の置き場を見つけ、`status.json` を読んで表示する。GitHub を読まないので、表示は最新の実行の時点のもので、実行の時刻を添える。記録の目印のファイル（`records-created.json`）を作った日が今日なら、「`setup` をした日は AI のセッションを起動しない。最初の実行は次の日から」と出す（[AI のセッション][design-ai]の「起動の前の確かめ」）
- 手動の 1 回実行は、lock を見て実行の最中なら、依頼を置かずに、実行中であることを示して終わる。実行の最中でなければ、`run-request.json` を置いてから（書くコマンドは sandbox の外で動かす。上の「点検だけで動かす仕組み」の表の下）、定期の実行と同じジョブを起こす（上の「定期の実行の登録」の手動の 1 回実行。`-k` を付けない。上の「見送った実行と、起動しなかった定期の実行」）
- `status.json` は記録の置き場にあり、AI は sandbox の中から読めず、書けない

[setup]: ../../requirements/06-setup-and-stages.md
[struct]: ../../requirements/07-structure.md
[impl]: ../../requirements/02-implement-review-fix.md
[safety]: ../../requirements/05-safety.md
[flow-rules]: ../../requirements/01-issue-flow.md
[design-settings]: ./01-settings.md
[adr-human]: ../../adr/adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-plugin]: ../../adr/adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-stop]: ../../adr/adr-20261017-007-stop-switch-in-repo-variable.md
[adr-tmp]: ../../adr/adr-20261017-006-per-session-short-tmpdir.md
[fig-setup-flow]: ./diagrams/04-setup-flow.drawio.svg
[fig-program-copy]: ./diagrams/04-program-copy.drawio.svg
[adr-read]: ../../adr/adr-20261015-001-read-allowlist-and-secret-file.md
[doc-systemd-timer]: https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html
[doc-loginctl]: https://www.freedesktop.org/software/systemd/man/latest/loginctl.html
[fig-schedule]: ./diagrams/04-schedule.drawio.svg
[adr-mcp]: ../../adr/adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[design-flow]: ./02-flow-and-judgment.md
[design-ai]: ./03-ai-sessions.md
[doc-pat-permissions]: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
[fig-check]: ./diagrams/04-check.drawio.svg
[adr-flow]: ../../adr/adr-20261015-006-run-stages-baseline-agent-field-and-local-records.md
[adr-workspace]: ../../adr/adr-20261013-001-fresh-clone-per-ai-session.md
[adr-review]: ../../adr/adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-bots]: ../../adr/adr-20261020-001-bot-accounts-as-list.md
[adr-app]: ../../adr/adr-20261020-002-switch-to-github-app-before-writing.md
[adr-review-skill]: ../../adr/adr-20261021-001-review-skill-text-without-skill-tool.md
[adr-store]: ../../adr/adr-20261015-002-no-shared-dependency-store.md
[adr-marker-keys]: ../../adr/adr-20261021-004-marker-key-list-and-resume-after-invalid-markers.md
[doc-rules]: https://docs.github.com/en/rest/repos/rules
[doc-app-token]: https://docs.github.com/en/rest/apps/apps#create-an-installation-access-token-for-an-app
[adr-intake-req]: ../../adr/adr-20261022-004-intake-skill-via-request-file.md
[adr-no-stages]: ../../adr/adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[adr-status]: ../../adr/adr-20261026-001-use-status-field-for-state.md
[adr-claude-dir]: ../../adr/adr-20261027-003-approved-claude-md-by-ai-and-claude-dir-by-humans.md
[adr-ci-secrets]: ../../adr/adr-20261028-001-ci-secrets-only-in-base-branch-environments.md
[adr-app-secrets]: ../../adr/adr-20261028-002-app-reads-secret-names-for-checks.md
