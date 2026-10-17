# 調査とレビューの記録（要件ではない）

## 同じ仕組みの有無と、業界の実践（2026-10-08 に調べた範囲）

「タスクが起票され、AI が自律的に判断して作業し、PR にして完了させる」仕組みは、2025〜2026 年に製品として複数ある。
dev-autopilot と同じ形（別の AI がレビューし、収束したらマージの可否を判定して人に渡す）は、公開されている範囲では見つからなかった。近いものを並べる。

### 比較

| 仕組み | タスクの受け取り | 隔離と権限 | レビューと往復 | 人の関所 | 上限 |
|---|---|---|---|---|---|
| GitHub Copilot coding agent | Issue を Copilot に割り当てる。書き方の指針：問題・受け入れ条件・触るファイルのヒント。向く仕事は不具合修正・UI の微修正・テスト・文書・技術的負債。人に残す仕事はセキュリティ・認証・本番障害・曖昧な要件 | Actions 上で動く。firewall 既定で有効。触れるのは自分のリポだけ。push は `copilot/*` だけ。Actions は人が「承認して実行」を押すまで走らない | PR は常に draft。レビューコメントを `@copilot` で渡す。GitHub は「Start a review でまとめて渡す」を勧める（コメントごとに 1 セッション起こさない） | draft PR・Actions の承認・マージはすべて人 | 1 タスク 59 分（延長不可）。1 セッション 1 premium request |
| OpenAI Codex（クラウド） | タスクを文で渡す。`AGENTS.md` が案内・テスト手順・慣習を持つ | タスクごとに独立した sandbox。並列 | 成果は PR。`@codex review` で PR をレビューさせられる | PR の確認とマージは人 | 未確認 |
| Google Jules | タスクを文で渡す。`AGENTS.md` を読む | タスクごとに独立した VM | **計画を先に出し、人が承認してからコードを書く**（離れると時間で自動承認） | 計画の承認・PR のマージ | 1 日 15 / 100 / 300 件、同時 3 / 15 / 60 件（プラン別） |
| Cognition Devin | Playbook（手順と成功条件）と Knowledge（組織の文脈・規約） | VM ごと。並列の Devin を 1 体がまとめる | 人のレビューが前提。「テストのロジックとコードの質は人が見る」 | マージは人 | セッションごとの ACU 上限 |
| OpenHands resolver | Issue に `fix-me` ラベル | GitHub Action | 成功なら draft PR、失敗ならブランチだけ push して Issue にコメント | draft PR の確認 | 未確認 |
| Claude Code Routine | 時刻・API・GitHub のイベント（PR と Release だけ） | Anthropic のクラウド。利用者本人として動く（commit・PR は本人名義）。push は `claude/*` ブランチ。接続した MCP は確認なしで書ける | 無し（1 回の実行） | ブランチ保護だけ | 最短 1 時間。時刻起動 100 回/時、API 30 回/時/Routine。**「実行が緑でも作業が成功したとは限らない」** |
| GitHub Agentic Workflows（gh-aw） | Markdown の workflow を Actions に変換 | **エージェントは読むだけ。書き込みは safe outputs（構造化した要求を別の権限付きジョブが検証して実行）** | 無し | safe outputs の上限 | 既定 20 分・500 ターン |
| Ralph Wiggum ループ（Huntley。公式 Plugin `ralph-wiggum` あり） | `PROMPT.md` と `@fix_plan.md`（優先順の TODO） | 同じプロンプトをシェルの while で回す。**毎回新しい文脈**。状態はファイルと git | テストを「背圧」として即座に回す | 回す側が作る | `--max-iterations` を必ず付ける（既定は無制限） |
| Anthropic「長時間動くエージェントの harness」 | 機能一覧の JSON（`passes` だけを書き換えてよい） | 進捗ファイルと git が記憶 | 新しい機能の前に「まず動くこと」を端から端まで確かめる | 無し | 1 セッション 1 機能 |
| Kiro（AWS） | `requirements.md`（EARS 形式の受け入れ条件）→ `design.md` → `tasks.md` | 開発環境 | 無し | 各段階を人が承認 | 無し |
| Linear Agents | Issue をエージェントに委任。**人が Assignee のまま、エージェントは contributor** | 製品ごと | 製品ごと | 人が責任を持つ | 無し |

### dev-autopilot がすでに同じ形になっている点

- 作業ごとに新しい文脈（Ralph、Anthropic の harness、Codex、Jules）
- 書き込みは判断する層ではなく実行する層が持つ（gh-aw の safe outputs。dev-autopilot では進行役）
- 人が Assignee のまま、AI は欄で示す（Linear）
- 人が付けるラベルで着手を決める（OpenHands の `fix-me`、Linear の委任）
- 受け入れ条件を 1 つずつ検証できる形で書く（Copilot の指針、Kiro の EARS、Devin の成功条件）
- 反復とセッションの上限（Ralph の `--max-iterations`、gh-aw の `max-turns`、Copilot の 59 分）

### 取り入れた実践（この節を受けて要件に足したもの）

- 着手前に基線の検査を回し、赤なら積まない（IMPL-9。Anthropic の harness）
- 実装の前に計画をコメントし、段階 5〜7 では人が止められる（IMPL-10。Jules）
- PR は draft で作り、収束したら ready にする（IMPL-11。Copilot、OpenHands）
- テストの削除・skip・期待値の緩和・lint の抑止を禁じ、レビューで 🔴 として探す（IMPL-12。Anthropic の harness）
- 1 セッションの上限を 45 分・200 ターンから始める（IMPL-13。Copilot、gh-aw）
- 指標にマージ後の revert 率を足す（FLOW-13。arXiv 2609.17598）
- 「実行が緑でも作業が成功したとは限らない」を進行役の記録に反映する：実行の成否と作業の成否を別の欄にする（Routine の注意書き）

### dev-autopilot が業界より先に出ている点（注意が要る）

- **開発の PR を AI の判定でマージする**のは、調べた製品のどれもやっていない。Copilot・Codex・Jules・Devin・OpenHands はすべて「PR を人が確認してマージ」で止まる。
  自動 approve まで行くのは CodeRabbit（条件つき）と Copilot（glob と設定で有効化）だけで、マージ自体は人の操作が残る。
  dev-autopilot も同じ位置に置く（段階 8 は、進行役がマージの可否を判定し、人が実行する。実行を進行役に移すのは将来の選択肢。 MERGE-9）。
  実行を進行役に移すなら、向き先が `develop`（`main` ではない）であること、方針に係るパスの除外、revert 率の監視、人のマージと判定が一致することの確認、の 4 つを揃える（MERGE-9）
- **AI 同士のレビューの往復を収束まで自動で回す**のも製品には無い（Copilot は人のレビューコメントに応じるだけ）。公開スキル 2 つ（agent-code-review-loop、copilot-pr-autopilot）が近い。
  収束の規則は「[レビューの Skill][f08]」の方針（🟢 は直さない、解決済みは再提起しない、往復 3 回、head の SHA）で守る

### 失敗の教訓（出どころつき）

- 後のセッションが「もう終わった」と早合点する → 機能一覧と「合格にする前にテスト」で防ぐ（Anthropic）。dev-autopilot では受け入れ条件のテストと IMPL-27 の終了条件
- 一度に全部作ろうとして文脈を使い切る → 1 セッション 1 機能（Anthropic）、文脈が 6〜7 割を超えると質が落ちる（Ralph）。dev-autopilot では 1 Issue 1 セッション、ラウンドごとに新しいセッション
- 途中で止まった実装を次のセッションが引き継げない → 進捗ファイルと git（Anthropic）。dev-autopilot では PR のコメント（計画・対応・レビュー）と commit が記録
- 「未実装」と誤って結論する、置き場だけのプレースホルダ実装を書く（Ralph） → レビューの動作の軸で TODO・スタブ・常に真を返す実装を探す
- 要件を途中で変えると人の新人より適応が悪い（Devin） → 作業指示のハッシュが変わったら止める（TEXT-4）
- 受け入れ率はリポでの実績とともに上がる（31〜37% → 70% という調査） → 段階を踏み、最初は 1 日 1 件から（FLOW-3、「[段階][f06]」）

## `change-review` スキルの評価と、レビューの業界の実践

2026-10-07 に `~/.claude/skills/change-review/`（SKILL.md 146 行、参照 5 ファイル、bash 1 本）を読んだ。

### 使ってよい理由

- 読む前に「実行して確かめる」を払う手順で、business-os の PR [#68][bos-pr-68] [#71][bos-pr-71] [#77][bos-pr-77] では実際に 1 回目で重い指摘が出て、2 回目で収束している
- 壊れ方の表（breakage-matrix）を機械的に当てるので、レビュー AI の気分で指摘の網が変わりにくい。fail-open を最重視する点はこのリポの設計哲学と同じ
- 指摘ごとに重要度（🔴 🟡 🟢）・失敗シナリオ・証拠レベルを必ず書かせる。重要度の定義（🔴 通してはいけない、🟡 通す前に直したい、🟢 あとでよい）が印（IMPL-18）の判定にそのまま使える
- 止めどきの規則（🟢 だけになったら直さずに記録して終わる）と「見送った 🟢 は Issue に起票する」が、すでにスキルに書いてある。IMPL-26 はスキルの規則をそのまま機械にしたもの
- 「収束していないなら未収束と書く」を求めるので、時間切れでも「通してよい」を偽らない（安全側）
- レビュー AI は診断だけで修正しない（「頼まれていない修正をしない」）。作業 AI と役割が混ざらない

### 比べたもの（2026-10-08）

| | いまの `change-review` | Claude Code 組み込みの `/code-review`（公式文書） | mattpocock/skills の `code-review`（679K installs、MIT） |
|---|---|---|---|
| 探すもの | 動作の不具合、とくに fail-open（静かに通る） | 論理の不具合・セキュリティ・端の条件・回帰・CLAUDE.md 違反。スタイルは見ない | 規約（repo の文書 + Fowler の smell 12 種）と仕様（Issue との一致・scope creep）。不具合は探さない（組み込みに任せる） |
| 確かめ方 | 実行を読む前に払う。壊れ方の表を機械的に当てる | 観点ごとの並列サブエージェント → 指摘ごとに敵対的な検証（実際のコードで再現）→ 重複除去と重要度づけ | 2 軸を並列サブエージェントで。再検証はしない（「指摘は仮説」と明記） |
| 指摘の形 | 重要度 🔴🟡🟢・失敗シナリオ・証拠レベル・根拠・推奨 | file:line・要約・重要度・分類・「なぜ指摘したか」の証拠。構造化（ReportFindings） | 軸ごとに引用つき（規約の行 / 仕様の行）。再順位づけしない |
| 収束 | 止めどきの規則あり（🟢 だけなら止めて Issue 化） | 再レビューは手動か「push ごと」。収束の規則は無い | 「収束の保証は無い。きれいになるまで回すな」と明記 |
| 無人 | 2 か所で人に聞く。予算なし | `claude -p` と Actions で動く。1 指摘 1 インラインコメント | サブエージェントが再帰して 50 体になる既知の不具合あり |

### 業界の実践（2026-10-08 に調べた範囲。製品の文書・論文・公開スキル）

| 出どころ | 取り入れること |
|---|---|
| GitHub Copilot code review（公式文書） | 既定では approve しない「Comment のみ」のレビュー。低い確信の指摘を抑える。再レビューは「push ごと」を設定で選ぶ。リポの指示ファイル（`.github/copilot-instructions.md`）で規約を渡す |
| CodeRabbit（設定の文書） | レビューの「量」を profile（quiet / chill / assertive）で選び、quiet は重要な指摘だけ行に付けて残りは要約にまとめる。再レビューは差分だけ（incremental）。レビュー済みコミットが 5 件たまると自動で止まる（暴走の歯止め）。コメントが全部解決・最新コミットがレビュー済み・検査が緑なら自動で approve する流れがある |
| Cursor Bugbot（公式文書） | 不具合だけを報告し、スタイルは見ない。重要度 high / medium / low と「blocking か non-blocking か」を分ける。再レビューは前回以降の差分だけ。リポの `.cursor/BUGBOT.md` で文脈を渡す。マージを止めるのは明示的に有効にしたときだけ |
| Graphite（ガイド） | 業界の誤検知率は 5〜15%。精度を優先し、網羅を少し捨てる。開発者の反応を学習に戻す |
| 論文「Adversarial Review」（arXiv 2608.18167） | 作業者・レビュアー・批評者の 3 体で、批評者がレビューを「構造化された反対」で監査する。合意を急ぐと証拠の無い合意（false consensus）になる。体を増やすより、反対を構造化し、証拠を必須にする方が効く |
| 論文「LLM は仕様への適合を見られるか」（arXiv 2603.00539） | LLM は正しいコードを「仕様に合わない」と誤判定しやすく、説明や修正案を求めるほど誤判定が増える。対策は「修正案を実行可能な反証として扱い、元と修正後の両方をテストで検証する」こと |
| 論文「文脈のバイアス」（arXiv 2603.18740） | PR の説明文を細工して「問題なし」と思わせる攻撃が、Claude Code と CodeRabbit の両方で 33 件中 32 件成功。作者の主張をレビュアーに読ませると検出率が下がる |
| CSA の研究ノート（AI エージェントと GitHub Actions） | 「Comment and Control」：Issue や PR のコメント 1 つで鍵をログに流出させる攻撃が実証済み。対策は、判断する層と実行する層を分け、エージェントに書き込みトークンや API キーを持たせない、ツールを最小にする、作者で入力を選別する |
| GitHub Copilot code review（2026-05 / 2026-09 の changelog） | 指摘ごとに重要度 High / Medium / Low を表示。approve は既定で無効で、有効にしても「変更ファイルがすべて指定の glob に当たるとき」だけマージ要件に数える。新しいコミットが push されると approve は取り消される（人のレビューと同じ）。低確信の抑制は文書に無い（未確認） |
| Cursor Bugbot（公式ブログ「Building a better Bugbot」） | 差分の順番を変えた 8 回の並列パス → 似た不具合を束ねる → 1 回しか出なかった不具合を多数決で落とす → 不要な分類（コンパイラ警告・文書の誤り）を除く → 検証用のモデルで誤検知を落とす → 前回の投稿と重複除去。解決率は 52% → 70% 超 |
| CodeRabbit（設定リファレンス・changelog） | 自動 approve の条件：コメントがすべて解決、最新コミットがレビュー済み、失敗した検査が無い。まとめた低優先度の指摘は approve を止めない。重要度の尺度は文書に無い（未確認） |
| Anthropic 公式 Plugin `code-review`（anthropics/claude-code の plugins/） | 並列 4 体（CLAUDE.md 準拠 ×2、差分の明白な不具合、git blame の履歴）→ 指摘ごとに確信度 0〜100 を付け、**80 未満を捨てる** → 1 本のレビューとして投稿。除外：既存の問題、不具合に見えて違うもの、nit、linter が拾うもの |
| 中立の数値（CR-Bench ICLR 2026、2026-01 の脆弱性検出の調査） | 「すべての隠れた問題を探す設計は S/N 比が下がる」。実 OSS で人が 385 件を検分した調査では、最良のツールでも誤発見率 85%（二次情報、未確認）。ベンダー公表の精度（65〜96%）は中立でない |
| GitHub API（公式文書） | Issue 依存の REST（`GET/POST/DELETE /repos/{owner}/{repo}/issues/{n}/dependencies/blocked_by`、検索の `is:blocked` `blocked-by:`）と、Projects v2 の `updateProjectV2ItemFieldValue`（Status は `singleSelectOptionId`）がある。SETUP-5・SETUP-7 は API で実装できる |
| 公開スキル「agent-code-review-loop」 | 作者と Reviewer を別セッションに分け、各ラウンドは「前回の解決ログ」だけを読む。指摘は file:line・重要度・失敗シナリオ・修正案を必須にし、発火条件を書けない指摘は捨てる。最大 3 ラウンド。解決済みの指摘は新しい証拠が無ければ再提起しない。4 段階（blocker / major / minor / nit）で nit は絶対にマージを止めない |
| 公開スキル「copilot-pr-autopilot」 | 収束の判定を決定的なスクリプトで出す（レビューが最新コミットに対して済み、新しいコメントが無く、開いているスレッドに全部返信済み）。10 ラウンドごとに「元の範囲から外れていないか」を振り返る回路遮断器。同じ指摘が再提起される振動は早めに切る。人へ渡した（hand-off）スレッドは失敗ではなく明示の引き渡し |

## 2026-10-11 のレビューで決めたことと、採らなかった案

新しい文脈のレビュアー 3 体（安全側、実現性と運用、整合性。モデルは `opus`）に[要件メモ][memo]を読ませ、58 件の指摘を受けた。重複を除いて 8 つの判断と約 40 件の整理に分け、判断は同日に次のとおり決めた。整理は各節に書き込んだ

1. **AI が書いた worktree の中身を、進行役・次のセッション・CI が信頼して実行しない**（SAFE-2 の経路。[要件メモ][memo]の 12 節の判断 1 の訂正）。採らなかった案：gitdir の書き込みを絞るだけで進行役が worktree の中で git を使い続ける（列挙漏れがそのまま穴になる）、Issue ごとの clone に戻す
2. **上限に当たったときの扱い**：時間もターンも「実行の失敗」で持ち越し、3 回で `needs-human`（IMPL-14 の表）。採らなかった案：ただちに `needs-human`
3. **1 回の実行で進める範囲**：Issue ごとに 1 段だけ。CI は次回に読む。WIP の上限 1。実行時間の上限（FLOW-4）。採らなかった案：待ち点まで進める
4. **信頼の判定**：投稿者でなく署名付きの印と、渡す直前の権限確認で決める。進行役が投稿した文は AI の出力扱い。計画コメントに印と検疫（TEXT-1、TEXT-10、TEXT-12、IMPL-10、IMPL-20）。採らなかった案：自動化用の GitHub アカウントを先に分ける（SETUP-13 の判断は段階 6 のまま）
5. **レビューの pass の条件**：5 条件をすべて満たすときだけ。批評者が落とした 🔴 は `merge=human`（IMPL-18、IMPL-22、MERGE-2）
6. **レビュー AI と作業 AI のモデル**：「モデルも別」を必須から外し、設定で分けられるに弱めた（IMPL-19）
7. **批評者の実体**：進行役が別に起こす独立セッション（IMPL-22。推奨はサブエージェントだったが、隔離を優先した）
8. **worktree の残骸**：warn にしてその Issue だけ `needs-human`。Issue に紐づかない残骸だけ fail（SETUP-2）

レビューが挙げた「見たが無い」観点：patch-id と merge の前提、必須チェック名と ci.yml の一致、Organization 所有の Project と PAT の組み合わせ

## 隔離とモデルの役割分担で採らなかった案

[要件メモ][memo]の 12 節の判断 1（AI のセッションの隔離と鍵の分け方）と判断 6（モデルの役割分担）で、採らなかった案。採った形は「[隔離と鍵の分け方][f05]」と「[モデルの流れ][f02]」にある。

- 隔離の案 B：AI のセッションを別の OS ユーザーで動かす。隔離は最も強いが、専用マシンの構築手順が増え、手元の Mac では運用しにくい
- 隔離の案 C：同じ OS ユーザーのまま、鍵と git 設定の共有を受け入れ、進行役のマージの段階（今の段階 8）を見送る
- モデルの役割分担で取り込まなかったこと：「コントロール役を `opus` にする」（進行役は AI にしない。「[登場するもの][f00]」）、「詰まったら計画役に自動で再計画させる」（3 回で `needs-human` のまま。IMPL-25）、Bedrock の地域の項（利用枠で動かす）

[f00]: ./00-overview.md
[f02]: ./02-implement-review-fix.md
[f05]: ./05-safety.md
[f06]: ./06-setup-and-stages.md
[f08]: ./08-review-skill.md
[memo]: https://github.com/Joymerrevent/business-os/blob/b8bc13eef06d430e24483168cfa0ff8edc1987e0/dev-autopilot/docs/dev-autopilot-requirements.md
[bos-pr-68]: https://github.com/Joymerrevent/business-os/pull/68
[bos-pr-71]: https://github.com/Joymerrevent/business-os/pull/71
[bos-pr-77]: https://github.com/Joymerrevent/business-os/pull/77
