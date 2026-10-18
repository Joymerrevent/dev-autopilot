# マージ

- MERGE-1 マージの実行は人だけが行う。進行ボットは MERGE-2 の条件でマージの可否を判定し、結果を人に渡すところまでを担う。AI のセッションには `gh pr merge` を許さず、進行ボットも `gh pr merge` を呼ばない
- MERGE-2 進行ボットが「マージしてよい」と判定する PR の条件（すべて満たす）
  - dev-autopilot が作った PR（進行ボットの署名付きの印が PR に付いていることで判定する。作者のアカウントでは判定しない。IMPL-20）で、向き先が `develop`
  - IMPL-27 の終了条件を満たしている（最新のラウンドの印が `verdict=pass` で、印の `head` が PR の head と一致し、CI 緑）
  - IMPL-15 の漏えいの検査を、判定の時点で最終の head の diff 全体に掛け直して通っている（過去の記録に頼らない）
  - 批評 AI が 🔴 を「再現できない」として落としたラウンドが無い（あれば `merge=human`。IMPL-22）
  - 変更したファイルがすべて MERGE-4 の許可一覧にあり、diff の状態がすべて追加（A）か変更（M）で、mode が `100644`（新規でもシンボリックリンクと実行属性付きは `merge=human`）。
    変更ファイルの一覧は API でなく、進行ボットが自分の clone で `git diff --raw <base>...<head>` を取って作る（API のページ送りの漏れを「一覧に無い」と読まないため）
  - 変更したファイルに、方針に係るパス（下の一覧）が 1 つも無い
- MERGE-3 判定の結果は進行ボットが PR のコメントに印として書く（形は IMPL-18 と同じ。`merge=ready` か `merge=human`、満たさない条件、head の SHA）。`ready` なら Status を Ready to Merge のまま（FLOW-23）人に渡す。`human` なら理由を添えて、Ready to Merge のまま人に渡す。
  判定の時点で CI が緑でなければ（IMPL-27）、判定を次の実行に回し、`merge` の印を書かない（`merge=human` にしない。[ADR-20261016-001][adr-pr]）。
  人は印を見てマージし、印と違う判断をしたとき（`ready` をマージしない、`human` をマージする）はその理由を PR にコメントする。この一致・不一致が判定の信頼度の記録になる（FLOW-13、[マージ判定工程（段階 8）][f06] の出口）。
  **一致・不一致の数え方**：`merge=human` の PR が人にマージされたら不一致（進行ボットが次回の実行で検出する）。`merge=ready` の PR は、人が「## 判定と違う判断」の見出しでコメントしたら不一致、マージされたら一致。
  どちらも無く 7 日経った PR には進行ボットが 1 回だけ催促のコメント（1 回かどうかは `note` の印（`kind=remind`）で判定する）を書き、人が動くまでマージ判定工程（段階 8）の 10 件に数えない
- MERGE-4 **進行ボットが「マージしてよい」と判定してよいパスの許可一覧**（拒否一覧ではなく許可一覧。一覧に無いパスを 1 つでも触れば `merge=human`）。設定ファイルに持つ。business-os の初期値（狭く始める）：
  - `docs/usage/**`（利用者向け文書）、`.changeset/*.md`（設定ファイル `config.json` は除く）
  - `plugin/skills/**` と `plugin/scripts/**` は利用者の CC で動く配布物なので初期値に入れない
  - `test/**` と `evals/**` も初期値に入れない。`evals/` には唯一の bash である `evals/scaffold.sh` と `evals/lib/**` の実行されるコードが、`test/**` には hook の fail-closed テストが含まれ、テストの弱体化（消す・skip する・期待値を緩める）を許可一覧では止められず、レビュー頼みになるため
  - `fixtures/**` も初期値から外す（vitest と eval の前提データで、期待値を緩める変更がテストの弱体化に当たり、`test/**` を外した理由がそのまま当てはまる）
  - `fixtures/**` `test/**` `evals/**` `plugin/skills/**` `plugin/scripts/**` は、マージ判定工程（段階 8）で 10 件の一致を確かめた後、別の判断で広げる
  - 判定は diff の状態も見る：削除（D）・改名（R）・種類の変更（T。シンボリックリンク化など）・実行属性の変更は、パスが許可一覧にあっても `merge=human`
  - 許可一覧に **含めない**（常に `merge=human`）の例：`docs/adr/**` `docs/design/**` `ROADMAP.md`、`CLAUDE.md` `.claude/**`、**`dev-autopilot/**`（自己改変）**、`.claude-plugin/**` `.github/**`、
    `plugin/hooks/**` `plugin/agents/**` `plugin/templates/**`（利用者の CC で動く防衛と雛形）、`plugin/scripts/check-repo.ts` など `check:*` の実装、`package.json` `pnpm-lock.yaml` `pnpm-workspace.yaml` `.node-version` `tsconfig.json` `vitest.config.ts` `eslint.config.*` `commitlint.config.*` `.gitleaks.toml` `.gitignore` `.markdownlint*`、`release/*` ブランチと `main` 向けの PR
- MERGE-5 人がマージするときは `gh pr merge --squash --match-head-commit <印の head SHA>` を使い、印の SHA と実際の head の一致をサーバ側でも強制する（人の手順として利用の文書に書く。将来、実行を進行ボットに移すときも同じコマンド）。1 回の実行で `merge=ready` にするのは 1 件まで（人が 1 件マージするごとに、残りは載せ直しと CI を経て判定し直す）。判定する順は PR の番号の小さい順。2 件目以降の、`merge=ready` になるはずの PR は、判定を次の実行に回して `merge` の印を書かず（`merge=human` にしない）、Status は `Ready to Merge` のままにする。`merge=human` の判定は 1 件の上限に数えない（[ADR-20261016-001][adr-pr]）。
  ruleset（SETUP-1 が作る）に「base と最新であること」（必須チェックの strict）を入れ、古い base で pass した PR はマージされない。載せ直しは進行ボットが自分の clone で `git merge origin/develop` を行い、結果の SHA を push する（force push を使わない。worktree の中では git を実行しない。「[安全側に倒す設計][f05]」）
- MERGE-6 **載せ直しで head が変わったときの扱い**。載せ直しは進行ボットが行う。載せ直しの前後で `develop` に対する差分の patch-id（`git patch-id --stable --verbatim`。行番号を無視し、周辺の行と空白は含む。`--verbatim` を付けるのは、既定では空白の違いを無視し、インデントだけの変更を同一と読むため）が同一なら、
  進行ボットが同じ内容（round・件数・verdict）の印を新しい head に付け直す。再レビューはせず、往復（IMPL-24）にも数えない。CI は新しい head で改めて緑を要求する。
  patch-id が変わった（周辺が変わった、衝突した）ときだけ再レビューを行い、往復に数える。衝突の解消は作業 AI の修正セッション（IMPL-23 と同じ形）で行い、その結果に再レビューを掛ける
- MERGE-7 マージ後（進行ボットは次回の実行でマージされた PR を検出する）：Issue を閉じる（`develop` 向けのため自動では閉じない）、作業場所（clone と worktree）をフォルダごと消す（ブランチはリポ設定 `delete_branch_on_merge` が消す）、依存していた PR を MERGE-6 の手順で載せ直す
- MERGE-8 マージの実行を人に残すので、利用者の全体ルール「PR のマージは Claude が実行しない」の例外は要らない。進行ボットは `gh pr merge` を呼ばない。判定（MERGE-2）の実装は vitest で検査する
- MERGE-9 マージの実行を進行ボットに移すか（将来の選択肢）。マージ判定工程（段階 8）の出口（10 件で人の判断と進行ボットの判定が全件一致、revert 0 件）を満たしても自動では移さない。移すなら ADR で決め、そのときに揃える条件は「[同じ仕組みの有無と、業界の実践][f11]」の 4 つ（向き先が `develop`、方針に係るパスの除外、revert 率の監視、人の判断と判定の一致）と、利用者の全体ルールの例外の扱い

[f05]: ./05-safety.md
[f06]: ./06-setup-and-stages.md
[f11]: ./11-records.md
[adr-pr]: ../adr/adr-20261016-001-ci-states-merge-ready-limit-and-pr-issue-link.md
