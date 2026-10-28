---
status: accepted
created: 2026-10-27
updated: 2026-10-28
author: Claude Code（メンテナの依頼で起票）
decision-makers: メンテナ
---

# 人が認めた CLAUDE.md は作業 AI が変えてよく、.claude と .mcp.json は認めても人が変える

要件の UNDECIDED-84（人が `## 仕組みの変更の承認` で認めたマージまで効かないパスを、作業 AI のセッションの安全設定で書けるようにするか。[決めていないこと][undecided]の「ADR で決めること」の「作業の単位の状態遷移」）を決める。
本 ADR は、[ADR-20261024-004][adr-master]（変えてはいけないパスを効く時期で 2 つの組に分け、人の承認で変えられるようにする）を前提にする。

改める決定と、据え置く決定は次のとおり。

| ADR | 扱い | 改める決定 | 据え置く決定 |
| --- | --- | --- | --- |
| [ADR-20261024-004][adr-master]（マスターの下書き、検疫、変えてはいけないパス） | 一部を改める | 決定 9 の、マージまで効かない組の「認めたパスは、AI が変えて push してよい」を、`CLAUDE.md`・`dev-autopilot/**`・規約の正典に限る。`.claude/**` と `.mcp.json` は、認めても人が変える（本 ADR の決定 1・2） | 決定 9 のほかの項目（2 つの組の分け方、push で効く組の扱い、照らす時、`## 仕組みの変更の承認` の権限、マージの判定をいつも「人が判断する」にすること）。決定 1〜8 |

## 背景と課題（Context and Problem Statement）

- [ADR-20261024-004][adr-master] の決定 9 は、マージまで効かない組（進行 Bot のリポの設定 `.claude/dev-autopilot.json`、ほかの `.claude/**`、`CLAUDE.md`、`.mcp.json`、`dev-autopilot/**`、規約の正典）のパスを、maintain 以上の人が `## 仕組みの変更の承認` で認めれば、AI が変えて push してよいとした
- 作業 AI のセッションの安全設定（基本設計の「[AI のセッション][design-ai]」の「セッションごとの安全設定」）は、承認の有無を問わず、worktree の `.claude/**`・`CLAUDE.md`・`.mcp.json` への Write と Edit の道具を止める（`permissions.deny`）。このままでは、認めても作業 AI が変えられない（2026-10-25 に、基本設計への反映の途中で見つけた）
- 2026-10-27 に、作業 AI のセッションと同じ形（`claude -p --setting-sources local --settings`、sandbox あり）で確かめた（FACT-77）
  - `CLAUDE.md`（根と下のフォルダ）は、Write の道具でも sandbox の中の Bash でも書ける
  - `.claude/` の下（`.claude/dev-autopilot.json` を含む）と `.mcp.json` は、Write の道具では書けない。`permissions.allow` で名指ししても、`--permission-mode acceptEdits` でも断られる
  - `.claude/settings.json`・`.claude/settings.local.json`・`.claude/skills/`・`.claude/agents/`・`.claude/commands/` の下と `.mcp.json` は、sandbox の中の Bash でも書けない。`sandbox.filesystem.allowWrite` で名指ししても止められる
  - sandbox の中の Bash は、`.claude/dev-autopilot.json` には書ける（安全設定の `deny` は Write と Edit の道具にしか効かない）

課題：人が認めたマージまで効かないパスのうち、どれを作業 AI に書かせ、どれを人が変えるか。

## 判断の決め手（Decision Drivers）

- 安全の境界は、要る分だけ緩める。Claude Code 自身の守りを外す起動の形（`--permission-mode bypassPermissions`、sandbox の外のコマンド）は使わない
- AI のセッションが読まない（効かない）ファイルは、認めれば AI に変えさせてよい。AI のセッションは `--setting-sources local` で起こすので、worktree の `CLAUDE.md` とプロジェクト設定を読まない（FACT-11）。人がマージするまで、どこにも効かない
- 道具によって書けたり書けなかったりするパスを、AI に変えさせる前提にしない（片方の道具で書けても、確かめずに通る経路になる）
- 人が変えるパスは、AI のセッションを起こす前に分かり、止まってから人に渡す

## 検討した選択肢（Considered Options）

- 案 A：認めた `CLAUDE.md` だけ、作業 AI のセッションで Write と Edit の禁止を外す。`.claude/**` と `.mcp.json` は、認めても人が変える
- 案 B：安全設定は緩めず、`CLAUDE.md` も `.claude/**`・`.mcp.json` も、認めても人が変える
- 案 C：認めたパスがあるときだけ、作業 AI のセッションを Claude Code の守りを外す形（`bypassPermissions`、sandbox の外のコマンドを許す）で起こし、`.claude/**` と `.mcp.json` も AI に変えさせる

## 決定（Decision Outcome）

採用した選択肢:「案 A」。

理由: `CLAUDE.md` は、Claude Code の守りの中でも AI が書け、AI のセッションは読まないので、認めれば AI に変えさせても人がマージするまで効かない。`.claude/**` と `.mcp.json` は、Claude Code 自身が AI に書かせず、書かせるには守り全体を外す必要があるので、認めても人が変える。安全の境界を、Claude Code の守りの中で要る分だけ緩める。

### 決定 1：認めた CLAUDE.md は、作業 AI に書かせる

- `## 仕組みの変更の承認` に `CLAUDE.md` があるときだけ、作業 AI のセッション（変更の作成と直し）の安全設定から、そのパスの Write と Edit の禁止を外す。認めていない `CLAUDE.md` は、今のまま止める
- 承認の読み方（maintain 以上の人、パスの大小を区別した照らし方）と、push の前に変更が触れたパスを照らすことは、今のまま（[ADR-20261024-004][adr-master] の決定 9）

### 決定 2：.claude と .mcp.json は、認めても人が変える

- `.claude/**`（push で効く組の `.claude/settings.local.json` のほか、進行 Bot のリポの設定 `.claude/dev-autopilot.json` を含む）と `.mcp.json` は、`## 仕組みの変更の承認` で認めても、作業 AI に変えさせない。安全設定の禁止も外さない
- 計画が変える見込みのパス（`plan.paths`）に、認めた `.claude/**` か `.mcp.json` が入っていれば、作業 AI を起こす前に、計画済みのまま停止中にする。止めた理由のコメントに、人が変えるパスを書く。人は、作業ブランチにそのパスの変更を push してから、`## 再開 続きから` を書く。止めた理由の記号と、人が push した後の続け方（作業 AI が残りの変更を作る形）は、基本設計で決める（push で効く組の `manual-push` の流れにそろえる）
- 作業 AI の差分に `.claude/**` か `.mcp.json` が入っていれば、認めていても、push せずに停止中（`protected-path`）にする。sandbox の中の Bash は `.claude/dev-autopilot.json` に書けるので、AI が書いた変更を通さない

### 影響（Consequences）

- 良い点: 認めた `CLAUDE.md` の変更を、作業 AI が作れる。人が変える手間が減る
- 良い点: Claude Code の守り（Write の道具の守りと sandbox）を外さずに済む
- 良い点: `.claude/**` と `.mcp.json` は、AI のセッションを起こす前に人に渡すので、AI が書けずに失敗し続けることが無い
- 悪い点: `.claude/**`（Skill、エージェント定義、設定）と `.mcp.json` の変更は、認めても人が作る。人の手間が残る
- 悪い点: 安全設定の組み立てに、承認に応じて禁止を外す分かれ道が増える
- 中立: Claude Code の版が変わると、守りの範囲が変わりうる。点検の試しのセッションで、FACT-77 の振る舞いを確かめ直す（基本設計の「[導入と点検][design-setup]」の「試しのセッション」）

### 確認方法（Confirmation）

- 単体テストで、安全設定の組み立てが、認めた `CLAUDE.md` のパスの Write と Edit の禁止だけを外し、認めていない `CLAUDE.md` と `.claude/**`・`.mcp.json` の禁止を残すことを確かめる
- 結合テストで、認めた `CLAUDE.md` を作業 AI が変えると push まで進み、マージの判定が「人が判断する」になることを確かめる
- 結合テストで、計画が認めた `.claude/skills/…` を変える見込みのとき、作業 AI を起こさずに停止中になることと、作業 AI の差分に `.claude/dev-autopilot.json` が入ると、認めていても push せずに `protected-path` で止まることを確かめる
- 点検の試しのセッションで、Write の道具と sandbox の中の Bash が `.claude/**` と `.mcp.json` に書けないことを確かめる

## 選択肢ごとの長所と短所（Pros and Cons of the Options）

### 案 A：認めた CLAUDE.md だけ AI に書かせ、.claude と .mcp.json は人が変える

- 良い点: Claude Code の守りの中で、要る分だけ緩める
- 良い点: 決定 9 の「認めれば AI が変えてよい」を、書けるパスでは保てる
- 悪い点: `.claude/**` と `.mcp.json` は人が変える

### 案 B：安全設定は緩めず、すべて人が変える

- 良い点: 安全設定の分かれ道が増えない
- 悪い点: AI が書けて、AI のセッションが読まない `CLAUDE.md` まで、人が変える
- 悪い点: 決定 9 の「認めれば AI が変えてよい」が、マージまで効かない組のうち `dev-autopilot/**` と規約の正典にしか残らない

### 案 C：認めたときだけ Claude Code の守りを外して起こす

- 良い点: `.claude/**` と `.mcp.json` も AI に変えさせられる
- 悪い点: 守りを外すと、認めたパスのほかにも、Claude Code が守っているすべてのファイルと、sandbox の外のコマンドが AI に開く。認めたパスだけに絞る手段が無い
- 悪い点: 承認 1 つで、安全の境界が大きく変わる

## 補足情報（More Information）

- 2026-10-25 に、基本設計への反映の途中で、決定 9 と安全設定の食い違いを見つけ、UNDECIDED-84 として足した。2026-10-27 に前提を確かめ（FACT-77。#163）、候補を書ける `CLAUDE.md` と書けない `.claude/**`・`.mcp.json` に分けた
- 本 ADR で決めないこと
  - 認めた `.claude/**` か `.mcp.json` を計画が変える見込みのときの、止めた理由の記号と、人が push した後の続け方（基本設計）
  - AI の作業ブランチの CI を秘密を持たない形で動かすか（[ADR-20261024-004][adr-master] の決定 9 の最後の箇条のまま、別の ADR で決める）
- 関連する要件：IMPL-8、IMPL-15（[02-implement-review-fix.md][impl]）、SAFE-2（[05-safety.md][safety]）、UNDECIDED-84（[09-undecided.md][undecided]）、FACT-11、FACT-70、FACT-77（[10-verified-facts.md][facts]）
- 関連する ADR：[ADR-20261024-004][adr-master]
- 要件と設計への反映（承認の後に、別の PR で 1 本にまとめて行う）
  - 02-implement-review-fix.md：IMPL-15 のマージまで効かない組の扱い
  - 09-undecided.md：UNDECIDED-84 を欠番にする（元の文を取り消し線で残す）
  - 基本設計の「[設定][design-settings]」：`.claude/**` の扱いの箇条
  - 基本設計の「[状態遷移と判定][design-flow]」：計画の工程の中の順番（照らすパス）、push の手順（変えてはいけないパス）、止めた理由の記号
  - 基本設計の「[AI のセッション][design-ai]」：セッションごとの安全設定の `permissions.deny`
  - 基本設計の「[導入と点検][design-setup]」：試しのセッションで確かめること
  - 基本設計の「[テスト][design-tests]」：確認方法のテスト
  - [ADR-20261024-004][adr-master]：タイトルの下に、本 ADR で決定 9 の一部を改めた旨の注記を足す

[adr-master]: ./adr-20261024-004-master-draft-quarantine-and-protected-paths.md
[impl]: ../requirements/02-implement-review-fix.md
[safety]: ../requirements/05-safety.md
[undecided]: ../requirements/09-undecided.md
[facts]: ../requirements/10-verified-facts.md
[design-settings]: ../design/basic/01-settings.md
[design-flow]: ../design/basic/02-flow-and-judgment.md
[design-ai]: ../design/basic/03-ai-sessions.md
[design-setup]: ../design/basic/04-setup-and-check.md
[design-tests]: ../design/basic/05-tests.md
