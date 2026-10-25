# マージ

- MERGE-1 マージの実行は人だけが行う。進行 Bot は MERGE-2 の条件でマージの可否を判定し、結果を人に渡すところまでを担う。AI のセッションには `gh pr merge` を許さず、進行 Bot も `gh pr merge` を呼ばない
- MERGE-2 進行 Bot が「マージしてよい」と判定する PR の条件（すべて満たす）
  - dev-autopilot が作った PR（進行 Bot の署名付きの印が PR に付いていることで判定する。作者のアカウントでは判定しない。IMPL-20）で、向き先が `develop`
  - IMPL-27 のレビュー合格の条件を満たしている（最新のラウンドの `review` の印の `result` が `pass` で、印の `head` が PR の head と一致し、CI 緑）
  - IMPL-15 の漏えいの検査を、判定の時点で最終の head の diff 全体に掛け直して通っている（過去の記録に頼らない）
  - 批評 AI が 🔴 を「再現できない」として落としたラウンドが無い（あれば `human` の判定。IMPL-22）
  - 変更したファイルがすべて MERGE-4 の許可一覧にあり、diff の状態がすべて追加（A）か変更（M）で、mode が `100644`（新規でもシンボリックリンクと実行属性付きは `human` の判定）。
    変更ファイルの一覧は API でなく、進行 Bot が自分の clone で `git diff --raw <base>...<head>` を取って作る（API のページ送りの漏れを「一覧に無い」と読まないため）
  - 変更したファイルに、方針に係るパス（下の一覧）が 1 つも無い
- MERGE-3 判定の結果は進行 Bot が PR のコメントに印として書く（`merge` の印。`result` が `ready`（マージしてよい）か `human`（人が判断する）、`reasons` に満たさない条件、`head` に head の SHA。形は「[印と固定の見出しの一覧][f04]」）。判定はレビュー合格の間に行い、どちらでもレビュー合格のまま人に渡す（`human` なら理由を添える。FLOW-37）。
  判定の時点で CI が緑でなければ（IMPL-27）、判定を次の実行に回し、`merge` の印を書かない（`human` の判定にしない。[ADR-20261016-001][adr-pr]）。
  人は印を見てマージし、印と違う判断をしたとき（`ready` をマージしない、`human` をマージする）はその理由を PR にコメントする。この一致・不一致が判定の信頼度の記録になる（FLOW-13。マージの実行を進行 Bot に移すか（MERGE-9）と、ほかのリポに広げる時期（UNDECIDED-82）の判断に使う）。
  **一致・不一致の数え方**：`human` の判定の PR が人にマージされたら不一致（進行 Bot が次回の実行で検出する）。`ready` の判定の PR は、人が「## 判定と違う判断」の見出しでコメントしたら不一致、マージされたら一致。
  どちらも無く 7 日経った PR には進行 Bot が 1 回だけ催促のコメント（1 回かどうかは `remind` の印で判定する）を書き、人が動くまで一致の記録の件数に数えない
- MERGE-4 **進行 Bot が「マージしてよい」と判定してよいパスの許可一覧**（拒否一覧ではなく許可一覧。一覧に無いパスを 1 つでも触れば `human` の判定）。設定ファイルに持つ。business-os の初期値（狭く始める）：
  - `docs/usage/**`（利用者向け文書）、`.changeset/*.md`（設定ファイル `config.json` は除く）
  - `plugin/skills/**` と `plugin/scripts/**` は利用者の CC で動く配布物なので初期値に入れない
  - `test/**` と `evals/**` も初期値に入れない。`evals/` には唯一の bash である `evals/scaffold.sh` と `evals/lib/**` の実行されるコードが、`test/**` には hook の fail-closed テストが含まれ、テストの弱体化（消す・skip する・期待値を緩める）を許可一覧では止められず、レビュー頼みになるため
  - `fixtures/**` も初期値から外す（vitest と eval の前提データで、期待値を緩める変更がテストの弱体化に当たり、`test/**` を外した理由がそのまま当てはまる）
  - `fixtures/**` `test/**` `evals/**` `plugin/skills/**` `plugin/scripts/**` は、一致の記録で 10 件の一致を確かめた後、別の判断で広げる
  - 判定は diff の状態も見る：削除（D）・改名（R）・種類の変更（T。シンボリックリンク化など）・実行属性の変更は、パスが許可一覧にあっても `human` の判定
  - 許可一覧に **含めない**（常に `human` の判定）の例：`docs/adr/**` `docs/design/**` `ROADMAP.md`、`CLAUDE.md` `.claude/**`、**`dev-autopilot/**`（自己改変）**、`.claude-plugin/**` `.github/**`、
    `plugin/hooks/**` `plugin/agents/**` `plugin/templates/**`（利用者の CC で動く防衛と雛形）、`plugin/scripts/check-repo.ts` など `check:*` の実装、`package.json` `pnpm-lock.yaml` `pnpm-workspace.yaml` `.node-version` `tsconfig.json` `vitest.config.ts` `eslint.config.*` `commitlint.config.*` `.gitleaks.toml` `.gitignore` `.markdownlint*`、`release/*` ブランチと `main` 向けの PR
- MERGE-5 人がマージするときは `gh pr merge --squash --match-head-commit <印の head SHA>` を使い、印の SHA と実際の head の一致をサーバ側でも強制する（人の手順として利用の文書に書く。将来、実行を進行 Bot に移すときも同じコマンド）。1 回の実行で `ready` の判定にするのは 1 件まで（人が 1 件マージするごとに、残りは載せ直しと CI を経て判定し直す）。判定する順は PR の番号の小さい順。1 件目の `ready` の判定の PR がマージされない間は、1 件目の印を変えず、2 件目以降を判定せずに待つ。2 件目以降の、`ready` の判定になるはずの PR は、判定を次の実行に回して `merge` の印を書かず（`human` の判定にしない）、レビュー合格のままにする。`human` の判定は 1 件の上限に数えない（[ADR-20261016-001][adr-pr]）。
  ruleset（SETUP-1 が作る）に「base と最新であること」（必須チェックの strict）を入れ、古い base で pass した PR はマージされない。載せ直しは進行 Bot が自分の clone で `git merge origin/develop` を行い、結果の SHA を push する（force push を使わない。worktree の中では git を実行しない。「[安全側に倒す設計][f05]」）
- MERGE-6 **載せ直しで head が変わったときの扱い**。載せ直しは進行 Bot が行い、載せ直すのはレビュー合格の PR だけ（FLOW-44、[ADR-20261024-006][adr-cleanup]）。載せ直しの前後で `develop` に対する差分の patch-id（`git patch-id --stable --verbatim`。行番号を無視し、周辺の行と空白は含む。`--verbatim` を付けるのは、既定では空白の違いを無視し、インデントだけの変更を同一と読むため）が同一なら、
  進行 Bot が同じ内容（`round`・件数・`result`・`findings`）の印を新しい head に付け直す。再レビューはせず、往復（IMPL-24）にも数えない。CI は新しい head で改めて緑を要求する。
  patch-id が変わった（周辺が変わった、衝突した）ときだけ再レビューを行う。往復（IMPL-24）に数えるのは、衝突を作業 AI が解いたときだけで、進行 Bot だけの載せ直しで差分が変わった再レビューは往復に数えない。
  衝突の解消は作業 AI の修正セッション（IMPL-23 と同じ形）で行い、修正と同じく対応コメントと `fix` の印を書いて、その結果に再レビューを掛ける。作業 AI を起こす前に往復の回数を確かめ、すでに上限に達していれば、衝突を解かずに停止中（`round-limit`）にする（[ADR-20261022-003][adr-rounds]）
- MERGE-7 マージ後（進行 Bot は次回の実行でマージされた PR を検出する。マージ済み）：進行 Bot が片付ける。Issue を閉じ（`develop` 向けのため自動では閉じない）、作業場所（clone と worktree）をフォルダごと消し（ブランチはリポ設定 `delete_branch_on_merge` が消す）、`cleanup` の印を書いて完了にする（FLOW-42）。片付けは軸の値を問わず行う。依存していた作業の単位は、先の作業がマージ済みになった時点で続け（FLOW-43）、PR はレビュー合格になってから載せ直す（FLOW-44）
- MERGE-8 マージの実行を人に残すので、利用者の全体ルール「PR のマージは Claude が実行しない」の例外は要らない。進行 Bot は `gh pr merge` を呼ばない。判定（MERGE-2）の実装は vitest で検査する
- MERGE-9 マージの実行を進行 Bot に移すか（将来の選択肢）。一致の記録で、10 件で人の判断と進行 Bot の判定が全件一致し、revert が 0 件（組み直す前のマージ判定工程（段階 8）の出口と同じ数。[ADR-20261024-002][adr-no-stages] の決定 6）を満たしても自動では移さない。移すなら ADR で決め、そのときに揃える条件は「[同じ仕組みの有無と、業界の実践][f11]」の 4 つ（向き先が `develop`、方針に係るパスの除外、revert 率の監視、人の判断と判定の一致）と、利用者の全体ルールの例外の扱い

[f05]: ./05-safety.md
[f11]: ./11-records.md
[adr-pr]: ../adr/adr-20261016-001-ci-states-merge-ready-limit-and-pr-issue-link.md
[adr-rounds]: ../adr/adr-20261022-003-count-round-trips-by-ai-pushes.md
[adr-cleanup]: ../adr/adr-20261024-006-cleanup-states-and-dependencies.md
[adr-no-stages]: ../adr/adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[f04]: ./04-external-text-and-markers.md
