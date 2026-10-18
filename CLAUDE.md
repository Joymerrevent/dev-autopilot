# dev-autopilot — 開発ガイド

dev-autopilot は、人が処理対象にした Issue を AI が実装し、別の AI がレビューし、指摘が無くなるまで直し、マージしてよいかを判定して人に渡す自律開発ループ（Claude Code の Plugin。開発中）。
目的と前提は [00-overview.md][overview] にある。今は要件・ADR・図と検査の設定だけで、コードはまだ無い。
当てる先は、マージ判定工程（段階 8）の後に決めるまで business-os（Joymerrevent/business-os）だけ（[07-structure.md][structure] の「分離の時期の目安」）。

**業務の固有名詞・人名・秘密を書かない。** 固有名詞の混入を探す検査はまだ無い（UNDECIDED-46）ので、書く前に確かめる。

## 何がどこにあるか

- `docs/requirements/`  要件。読み方と番号の索引は [README][req-readme]、決めていないことは [09-undecided.md][undecided]
- `docs/adr/`           設計判断。**ここが正典**。規則は [README][adr-readme]、一覧は [index.md][adr-index]
- `docs/diagrams/`      図（要件の写し）。直し方は [README][diagrams]
- `docs/design/`        設計。基本設計（構造仕様）は `basic/`、詳細設計は `detailed/`（まだ無い）。書いている途中。読み方と文書の一覧は [README][design]
- `plugin/` `test/` `fixtures/`  予定の構成（[07-structure.md][structure]）。まだ無い。進行 Bot は `plugin/src/` に置いて Plugin に同封し、単体テストは同封しないように `test/unit/` に置く
- `.github/`            CI（`ci`・`commitlint`）と Dependabot
- `.claude/`            Claude Code の禁止の規則（`settings.json`）

## 守ること

1. ADR の判断に反する変更をしない。変えたいなら ADR を `proposed` で起票する（`documentation-and-adrs` スキルと [ADR の規則][adr-readme]）。
   **CC が単独で `accepted` にしない。** メンテナが決める
2. 要件は [README][req-readme] の「要件を変えるとき」に従って直す。選ぶ判断は整理の中で決めず、[09-undecided.md][undecided] に足して ADR で決める。
   図が写している要件を変えたら、同じ PR で図も直す
3. bash のスクリプトを足さない。進行 Bot とスクリプトは TypeScript で書き、Node 24 が直接実行する（[07-structure.md][structure]）
4. Markdown のリンクは参照スタイル（`[表示][ラベル]` と、ファイル末尾の `[ラベル]: パス`）。markdownlint が inline のリンクを止める。Markdown は Prettier で整形しない
5. 検査は `pnpm lint` と `pnpm format:check`。成否は終了コードで見る。commit のときに lint-staged・commitlint・gitleaks がフックで走る
6. ブランチは `main` と `develop`。PR は `develop` 向けにする。`main` と `develop` は ruleset で PR 必須、必須チェックは `ci` と `commitlint`。
   公開済みの履歴を書き換えない。Conventional Commits
7. business-os の中身を import しない（STRUCT-1）。business-os の PR と Issue の番号は、「business-os の PR #77」のように business-os のものと分かる書き方にし、リンクを付ける
8. dev-autopilot を、dev-autopilot 自身のリポに当てない（作業 AI が進行 Bot のコードを書き換える経路になる。当てるなら別の ADR で決める）

[overview]: docs/requirements/00-overview.md
[structure]: docs/requirements/07-structure.md
[req-readme]: docs/requirements/README.md
[undecided]: docs/requirements/09-undecided.md
[adr-readme]: docs/adr/README.md
[adr-index]: docs/adr/index.md
[diagrams]: docs/diagrams/README.md
[design]: docs/design/README.md
