---
status: accepted
created: 2026-10-13
updated: 2026-10-14
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# AI の作業場所は AI のセッションごとに origin から clone し直し、進行役は AI が触った clone で git を実行しない

要件の UNDECIDED-1〜4（[決めていないこと][undecided]の「AI の作業場所」）を決める。
確かめることの UNDECIDED-24 は、本 ADR の決定で確かめる必要が無くなる。

## 背景と課題（Context and Problem Statement）

要件で決まっていて、本 ADR で選択肢にしない前提：

- 進行役は専用の clone を 2 つ持つ。進行役だけが書き、fetch・merge・push を行う「進行役の clone」と、AI の worktree を切る「AI の作業場所」の clone（SAFE-16、IMPL-1）
- 作業フォルダが worktree のとき、sandbox は親 clone の `.git` への書き込みを既定で許す。AI は親 clone の objects・refs・`gitdir` を書ける（SAFE-11、FACT-9）
- 進行役は worktree の中でも親 clone の中でも git を実行しない。親 clone で行うのは worktree の作成と削除だけ（SAFE-2 の (b)、SAFE-12）。AI の作業場所の refs は基準に使わない（SAFE-13）
- AI の commit は、進行役の clone へ fetch で取り込み、検査してから push する（SAFE-12、IMPL-15）

前提の上に、次の 4 つの穴が残っている。

1. UNDECIDED-1：進行役は AI の作業場所の clone で worktree の作成と削除しか行わないので、clone に無い commit（`develop` の新しい commit、人の push、載せ直しの結果）から worktree を作れない
2. UNDECIDED-2：同じ clone に複数の Issue の worktree があると、ある Issue の AI が別の Issue の `gitdir` や refs を書き換えられる
3. UNDECIDED-3：AI が clone の objects のファイルを書き換えると、同じ clone から作ったレビュー用の worktree に書き換えた中身がエラーなしで出て、`git fetch origin` でも直らない（FACT-18）。
   作業 AI が、レビュー AI と批評者に PR と違う中身を読ませられる
4. UNDECIDED-4：SAFE-12 は「AI は自分の `commondir` を書き換えられる」を前提に書かれているが、FACT-9 では sandbox が `.git/worktrees/<名前>/commondir` を保護していて書けなかった

UNDECIDED-1 を「AI が触った clone で進行役が `git fetch origin` を実行する」形で解くには、AI が触った clone での fetch が安全であることが前提になる（UNDECIDED-24）。
2026-10-13 に、前提を崩す事実を確かめた（補足情報の試行）。

- 普通の clone の `.git/` に `commondir` というファイルを置いて別のフォルダを指させると、git は別のフォルダの `config` を読む。進行役の `git fetch origin` で、別のフォルダの `config` に書いた `core.fsmonitor` のコマンドが実行された
- sandbox が保護すると確かめたのは worktree の `commondir`（`.git/worktrees/<名前>/commondir`）だけで、AI が親 clone の `.git/commondir` を新しく作れるかは確かめていない

課題：AI のセッション（計画役・作業 AI・レビュー AI・批評者）ごとに、作業場所をどう用意し、前のセッションの後に増えた commit をどう渡すか。

## 判断の決め手（Decision Drivers）

- AI が進行役の clone に書けないこと（SAFE-16）
- 前のセッションの AI が細工した `.git` を、次のセッションが読まないこと。とくにレビュー AI と批評者が、PR の本当の中身を読むこと
- 進行役が git を実行する clone を、進行役の clone と、AI がまだ触っていない clone に限ること。AI が触った clone は、FACT-12 で確かめた操作（進行役の clone へ fetch するときの取り出し元）にだけ使うこと
- 守りが sandbox の保護の一覧だけに頼らないこと。保護の一覧は Claude Code の版で変わりうるうえ、確かめていない経路がある
- 手順が少なく、手元の記録に頼らないこと（手元の記録を使うかは UNDECIDED-8 で決める）

## 検討した選択肢（Considered Options）

- 案A：AI のセッションごとに origin から clone し直す
- 案B：Issue ごとの clone を使い続け、進行役が `git fetch origin` で新しい commit を届ける
- 案C：Issue ごとの clone を使い続け、進行役の clone から push で新しい commit を届ける
- 案D：作業 AI は案B の形にし、レビュー AI と批評者だけ案A の形にする

## 決定（Decision Outcome）

採用した選択肢:「案A：AI のセッションごとに origin から clone し直す」。
理由: 進行役が git を実行する clone を、進行役の clone と AI がまだ触っていない clone に限れる唯一の案だから。
前のセッションが細工した objects を次のセッションが読まず、UNDECIDED-24 の確認も要らなくなる。費用は 1 セッションあたり数秒で済む（補足情報の計測）。

中身：

- **置き場**：AI の作業場所は、設定の置き場（STRUCT-2 の「worktree の置き場」。以下 `<置き場>`）の下に、AI のセッションごとに 1 つのフォルダとして置く。
  人の clone の中にも、人の clone の worktree としても作らない（IMPL-1）。フォルダの名前は略さず、中身が分かる名前にする

  ```text
  <置き場>/
    controller/                  進行役の clone
    sessions/
      <セッションの名前>/         1 回のセッションの作業場所（セッションごとに作り直す）
        clone/                   AI の作業場所の clone（git のデータだけ。ファイルは取り出さない）
        worktree/                clone/ から作る worktree（AI が作業するフォルダ）
  ```

  セッションの名前は、役と対象を略さずに書く（`plan-issue-<番号>`、`work-issue-<番号>`、`review-pr-<PR 番号>-round-<ラウンド>`、`critic-pr-<PR 番号>-round-<ラウンド>`）。
  手元の Mac で動かす間は、`<置き場>` を人の clone と同じ階層のフォルダにしてよい（例：人の clone が `<リポ名>.worktrees/<リポ名>/` なら `<リポ名>.worktrees/autopilot/`）。
  人の作業場所と分けたくなったら `<リポ名>.autopilot/` に移す。移すときは実行を止め、進行役の clone を移して設定を書き換える。AI の作業場所は次のセッションで作り直される

- **clone の作り方**：origin の URL（GitHub）から作る。AI に渡すのは、GitHub にあるもの（進行役が検査して push したもの、人が push したもの）だけにするため。
  進行役の clone には、検査の前の AI の commit と、検査で止めた AI の commit（漏えいの検査に引っかかったもの、進行役・次のセッション・CI が実行するパスに触れたもの。IMPL-15）が入る。
  進行役の clone から作ると、refs の持ち方しだいで、検査を通っていない commit が別の Issue のセッションに届きうる。届かないようにするには clone する refs を絞る手順が要り、守りが手順の正しさに頼る。
  さらに、手元の clone のパスから作ると、git が objects のファイルをハードリンクで共有し、AI の書き換え方によっては元の clone の中身まで変わる（FACT-19）。
  `--no-hardlinks` を付けるか `file://` の URL を使えば共有しないが、検査を通っていない commit が届きうる点は変わらないので、進行役の clone からは作らない。
  進行役の clone から作れば通信と認証が要らず速いはずだが（計測していない）、origin からでも約 3 秒で済む。clone の認証は進行役の鍵で行い、AI のセッションには渡さない（SAFE-1）
- **作る手順**：AI のセッションを起動する前に、進行役が次の 2 つを行う
  1. `git clone --no-checkout --no-tags <origin の URL> <置き場>/sessions/<名前>/clone`
  2. `git -C <置き場>/sessions/<名前>/clone worktree add <置き場>/sessions/<名前>/worktree <SHA>`。作業 AI は `-b <ブランチ>` を付け、ほかの役は `--detach` を付ける。
     `<SHA>` は、計画と最初の実装では進行役の clone で fetch した `origin/develop`、修正とレビューと批評では GitHub の API が返す PR の head（SAFE-13）
- **セッションの間で引き継ぐもの**：進行役が push した commit と、進行役が渡す文（計画、レビューの指摘）だけ。
  前のセッションの作業フォルダの中身（commit していないファイル、ビルドの結果、`node_modules`）は引き継がない。修正の段も新しい作業場所で始める（IMPL-23 の「同じ worktree で」を改める）。依存の導入は各セッションの中で行う（IMPL-1 のまま）
- **批評者**：レビュー AI と同じ PR の head で、別に作った作業場所で起動する（「登場するもの」の「レビュー AI と同じ worktree で」を改める）。レビュー AI も作業フォルダと `.git` を書けるので、レビュー AI の作業場所を批評者に読ませない
- **取り込み**：作業 AI の commit は、進行役の clone で `git fetch <置き場>/sessions/<名前>/worktree <ブランチ>` を実行して取り込む（FACT-12 で、`commondir` を差し替えても発火しないと確かめた操作）。
  AI が書き換えた objects は、受け取り側の検査で失敗し、進行役の clone に入らない（FACT-18）
- **AI のセッションを起動した後の clone**：進行役は、取り込みの fetch の取り出し元にする以外に git を実行しない。`git fetch origin`・`worktree remove`・`branch -D` も行わない
- **消す**：同じ名前で作り直す直前と、片付け（FLOW-27、MERGE-7）のときに、`sessions/<名前>/` をフォルダごと消す。git は使わず、シンボリックリンクの先をたどらない。
  消す前に `sessions/<名前>/clone/.git/worktrees/worktree/commondir` と `gitdir` をファイルとして読み（git は worktree の管理フォルダを、worktree のフォルダの名前 `worktree` で作る）、作ったときの中身と違えば、消さずに作業場所の Issue を `needs-human` にし、`check` で warn にする（SAFE-4）
- **並列（UNDECIDED-2）**：clone はセッションごとに別なので、Issue の間でも、作業とレビューと批評の間でも `.git` を共有しない。WIP の上限を上げても（段階 9）同じ形のまま動く。
  `allowWrite` の列挙はせず、sandbox の既定に任せる（SAFE-11）
- **`commondir` の前提（UNDECIDED-4）**：SAFE-12 の前提を次の文に改める。
  「sandbox は worktree の `commondir` を保護する（FACT-9）。ただし保護の一覧は Claude Code の版で変わりうるうえ、親 clone の `.git/commondir` の作成のように確かめていない経路があり、git は `commondir` を見つけると指す先の `config` を読む」。
  進行役が AI の触った clone で git を実行しない決まりは、sandbox の保護に重ねる守りとして残す。`gitdir` の `denyWrite`（SAFE-11）も残す
- **人の見方**
  - AI のブランチは、進行役が push した後に、人の clone で fetch した `origin/<型>/issue-<番号>` として見る（SourceTree などの git の画面の `origin/` の下）。差分は PR の画面で読む。
    人が引き継ぐ（Agent 欄を `human` にする）ときは、`origin/<型>/issue-<番号>` から手元のブランチを作る
  - `<置き場>` の下の clone を、人の clone の remote に登録しない。AI が書いた refs と objects を人の clone へ直接入れないため
  - `<置き場>` の下のフォルダを、人のエディタのワークスペースに入れない。エディタと拡張機能（git の表示、lint の設定の読み込みなど）が、AI の書いた設定やコードを人の権限で sandbox の外で実行するおそれがあるため（試していない）

### 影響（Consequences）

- 良い点: 進行役が git を実行するのは、進行役の clone と、AI がまだ触っていない clone だけになる。AI が触った clone は、FACT-12 で確かめた取り込みの fetch の取り出し元にするだけ
- 良い点: レビュー AI と批評者は、origin から取ってきたままの PR の中身を読む。作業 AI やレビュー AI が objects を細工しても、次のセッションに届かない（UNDECIDED-3）
- 良い点: Issue の間でもセッションの間でも `.git` を共有しないので、並列にするときに形を切り替えない（UNDECIDED-2）
- 良い点: 作り直すかどうかの判定が無く、手元の記録を使わない。UNDECIDED-24 の確認が要らなくなる
- 悪い点: セッションごとに clone と依存の導入をやり直す。sandbox の外の計測で、clone 約 3 秒、worktree の作成 0.04 秒、`pnpm install` 1〜3 秒。sandbox の中（通信は許可した先だけ）では計測していない
- 悪い点: 作業 AI は、前のセッションの作業フォルダに残したもの（commit していない変更、ビルドの結果）を修正の段で使えない。
  上限（IMPL-13）に当たって途中で止まったセッションの作業（直したファイル、AI が自分で積んだ commit）も、次のセッションには渡らない。
  1 セッションに収まらない大きさの作業は、3 回とも止まって `needs-human` になる（IMPL-14）。引き継ぐかは補足情報の「範囲外」に書いた
- 悪い点: 1 ラウンドあたりの clone が、批評者の分だけ 1 つ増える

### 確認方法（Confirmation）

- 段階 3 の `check`（SETUP-2）で、次を実際に試す
  - objects を書き換えた clone から進行役の clone へ fetch すると失敗し、進行役の clone に書き換えた中身が入らないこと
  - 進行役の clone と AI の作業場所の clone に、リンク数が 2 以上の objects のファイルが無いこと（`find <clone>/.git/objects -type f -links +1` の出力が空）
  - `.git/worktrees/<名前>/gitdir` が sandbox の中から書けないこと
- 結合テストで、偽の git を使って次を検査する
  - 進行役が、AI のセッションを起動した後の clone に対して、取り込みの fetch 以外の git を実行しないこと
  - 消す前に読んだ `commondir` か `gitdir` が作ったときと違うと、消さずに `needs-human` になること
  - 消すときに、作業場所の中のシンボリックリンクの先を消さないこと

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案A：AI のセッションごとに origin から clone し直す

- 良い点: 進行役は、AI が触った clone で git を実行しない（取り込みの fetch の取り出し元にすることを除く）
- 良い点: 前のセッションが細工した objects を、次のセッションが読まない
- 良い点: 片付けがフォルダの削除だけで済む
- 悪い点: セッションごとに clone と依存の導入をやり直す（数秒）

### 案B：Issue ごとの clone を使い続け、進行役が `git fetch origin` で新しい commit を届ける

- 良い点: clone は Issue ごとに 1 回で済み、作業フォルダの中身を修正の段に引き継げる
- 悪い点: AI が書き換えた objects が clone に残る。`git fetch origin` は手元にある objects を取り直さないので直らず、同じ clone から作ったレビュー用の worktree に書き換えた中身が出る（FACT-18）
- 悪い点: 進行役が AI の触った clone で `git fetch origin` を実行する。AI が親 clone の `.git/commondir` を作れると、fetch が AI の用意した設定を読み、コマンドを sandbox の外で実行する（2026-10-13 の試行）。
  sandbox が `.git/commondir` の作成を止めるかは確かめていない（UNDECIDED-24）

### 案C：Issue ごとの clone を使い続け、進行役の clone から push で新しい commit を届ける

- 悪い点: AI が書き換えた objects が clone に残る点は案B と同じ。push も、受け取り側がすでに持っている objects は送らない
- 悪い点: push は AI が触った clone の中で receive-pack を動かす。receive-pack は受け取り側の設定と hook を使うので、案B の `commondir` の経路に加えて hook の経路も開く（試していない）

### 案D：作業 AI は案B の形にし、レビュー AI と批評者だけ案A の形にする

- 良い点: レビュー AI と批評者は、origin から取ってきたままの PR の中身を読む（UNDECIDED-3 は解ける）
- 良い点: 作業 AI は作業フォルダの中身を修正の段に引き継げる
- 悪い点: 作業 AI の clone で `git fetch origin` を実行する点は案B と同じ（UNDECIDED-24）
- 悪い点: 役によって作業場所の作り方が 2 通りになる

## 補足情報（More Information）

- 関連する要件：IMPL-1、IMPL-13、IMPL-16、IMPL-23（[02-implement-review-fix.md][impl]）、SAFE-2 の (b)、SAFE-11、SAFE-12、SAFE-13、SAFE-16（[05-safety.md][safety]）、
  FLOW-27（[01-issue-flow.md][flow]）、MERGE-7（[03-merge.md][merge]）、STRUCT-2（[07-structure.md][struct]）、SETUP-2（[06-setup-and-stages.md][setup]）、
  FACT-9、FACT-12、FACT-18、FACT-19（[10-verified-facts.md][facts]）、「登場するもの」の批評者の行（[00-overview.md][overview]）
- 2026-10-13 の試行（macOS、git 2.50.1、pnpm 10.29.3。Claude Code と sandbox は使わず、AI による書き換えは人の手で再現した）
  - `commondir`：使い捨てのリポジトリで、普通の clone の `.git/commondir` に別のフォルダのパスを書き、別のフォルダの `config` に `core.fsmonitor` のコマンドを書いた。
    clone で `git fetch origin` を実行すると（終了コード 0）、コマンドが実行された。`git rev-parse --git-common-dir` は別のフォルダを返した
  - 費用：business-os を GitHub の URL から `git clone --no-checkout --no-tags` で約 3.0 秒、`git worktree add --detach` で 0.04 秒。
    `pnpm install --frozen-lockfile --ignore-scripts` は、手元の共有の store で 1.1 秒、空の store で 3.0 秒（`node_modules` は 137 MB）。clone の objects にリンク数 2 以上のファイルは無かった
- 前の版：2026-10-11〜12 に、同じ論点を ADR-20261011-001 として起票した（[閉じた PR #95 の版][prev]。要件メモの番号で書かれている）。
  決定（案A）は同じ。本 ADR は今の要件の番号で書き直し、批評者を別の作業場所にすること、UNDECIDED-4 と UNDECIDED-24 の扱い、2026-10-13 の試行を足した
- 範囲外：依存の store（pnpm の store）をセッションの間で共有するか。前のセッションが細工したものを次のセッションが読む経路として、objects と同じ種類の論点になる。
  手元の Mac（APFS）では、store と `node_modules` のファイルは共有していなかった（リンク数 1）。専用マシンのファイルシステムでは確かめていない。要件に反映するときに、[決めていないこと][undecided]に足す
- 範囲外：上限（IMPL-13）に当たって止まったセッションの作業を、次のセッションに引き継ぐか。本 ADR では引き継がず、次のセッションは GitHub にある状態（push 済みの commit と計画）から始める。
  今の要件も「続きから再開する」とは書いておらず、IMPL-13 は途中の状態を commit しないと決めている。ただし Issue ごとの worktree を使い続ける今の要件の形では、やり直しの作業 AI が前のセッションの残したファイルと commit を目にしていた。
  引き継ぐ形の案として、止まったセッションでも AI が commit した分だけを進行役の clone へ取り込み、漏えいの検査と実行するパスの検査（IMPL-15）を通して作業ブランチへ push し、次のセッションをその commit から始める形がある。
  IMPL-13 の「途中の状態を commit しない」を変え、未完成の作業を GitHub に push することになるので、別に決める。段階 5 で上限に当たった回数を記録してから決める。要件に反映するときに、[決めていないこと][undecided]に足す
- 要件への反映（承認の後に、別の PR で行う）
  - IMPL-1：AI の作業場所の clone は、AI のセッションごとに origin の URL から作り直すことを書き足す
  - IMPL-16：「AI の作業場所の clone から PR の head で切る専用の worktree」を、「レビューのたびに origin の URL から clone し、PR の head で切る worktree」に改める
  - IMPL-23：「作業 AI を同じ worktree で起動し」を「作業 AI を新しい作業場所で起動し」に改める
  - IMPL-13：途中で止まったセッションの作業場所は、作り直すか片付けるまで残すことを書き足す
  - 「登場するもの」の批評者の行：「レビュー AI と同じ worktree で」を「レビュー AI と同じ PR の head で、別に作った作業場所で」に改める
  - SAFE-2 の (b) と SAFE-12：親 clone で行う操作を「AI のセッションを起動する前の clone と worktree の作成」と「フォルダごとの削除」に改め、`commondir` の前提を本 ADR の決定の文に直す
  - SAFE-16：AI の作業場所の clone はセッションごとに origin の URL から作り、手元の clone のパスからは作らないことを書き足す
  - FLOW-27、MERGE-7：「worktree を消す」を「作業場所（clone と worktree）をフォルダごと消す」に改める
  - 10-verified-facts.md：2026-10-13 の 2 つの試行を FACT-20・FACT-21 として足す
  - 09-undecided.md：UNDECIDED-1〜4 と UNDECIDED-24 を欠番にする（本 ADR で決めた）。依存の store の論点と、上限で止まった作業を引き継ぐかの論点を足す

[undecided]: ../requirements/09-undecided.md
[impl]: ../requirements/02-implement-review-fix.md
[safety]: ../requirements/05-safety.md
[flow]: ../requirements/01-issue-flow.md
[merge]: ../requirements/03-merge.md
[struct]: ../requirements/07-structure.md
[setup]: ../requirements/06-setup-and-stages.md
[facts]: ../requirements/10-verified-facts.md
[overview]: ../requirements/00-overview.md
[prev]: https://github.com/Joymerrevent/business-os/blob/e1153bab318044dfafbf4232be6bc1a32668d4d7/dev-autopilot/docs/adr/adr-20261011-001-ai-worktree-clone-and-fetch-only.md
