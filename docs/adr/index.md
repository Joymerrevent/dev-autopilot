# ADR 一覧

起票した ADR を、番号の順に並べる。起票の手順と status の運用は [README][readme] にある。
2026-10-08 に起票した 9 本は、設計が日ごとに変わる段階では要件メモとの二重保守になるため 2026-10-10 にいったん削除した（[要件メモ][memo]の冒頭の決定）。
受け入れ工程（段階 4。棚卸しと受け入れ。「[段階][stages]」）の後に、[要件][req]の決定を元に書き直す。
書き直すときは固定したい判断に絞る（候補：AI は GitHub に書かない、マージの実行は人、外の文は AI に渡さない）。

| 番号 | タイトル | status | 作成日 |
| --- | --- | --- | --- |
| ADR-20261013-001 | [AI の作業場所は AI のセッションごとに origin から clone し直し、進行役は AI が触った clone で git を実行しない][adr-20261013-001-fresh-clone-per-ai-session] | accepted | 2026-10-13 |
| ADR-20261015-001 | [AI の読み取りは自分のセッションのフォルダに絞り、秘密は進行役だけが読むファイルに置き、手動実行は launchd のジョブを起こす][adr-20261015-001-read-allowlist-and-secret-file] | accepted | 2026-10-15 |
| ADR-20261015-002 | [AI のセッションは依存の store を共有せず、セッションの一時フォルダの store で依存を導入する][adr-20261015-002-no-shared-dependency-store] | accepted | 2026-10-15 |
| ADR-20261015-003 | [受け入れはマイルストーンを待たずに走らせ、検疫で見つかったら作業指示を書かずに検疫の結果と印だけを投稿する][adr-20261015-003-intake-scope-and-quarantine-report] | accepted | 2026-10-15 |
| ADR-20261015-004 | [進行役の投稿にはすべて署名付きの印を付け、人の操作は操作した人の権限と時刻を確かめてから従う][adr-20261015-004-signed-markers-and-actor-checks] | accepted | 2026-10-15 |
| ADR-20261015-005 | [要件が決めていない設定の値は構造仕様に初期値として書いて設定ファイルで変え、マシンごとの値はリポの外の設定に置く][adr-20261015-005-settings-values-and-machine-settings] | accepted | 2026-10-15 |
| ADR-20261015-006 | [1 回の実行では重い段を 1 つだけ進め、基線が緑でなければ新しい着手だけ止め、進行役は Agent 欄に人の値を書かず、手元の記録は上限の数え上げにだけ使う][adr-20261015-006-run-stages-baseline-agent-field-and-local-records] | accepted | 2026-10-15 |
| ADR-20261016-001 | [CI が赤なら修正に回し、結果が出ていなければ待ち、merge=ready は 1 回の実行で 1 件にして残りは次の実行に回し、PR と Issue はブランチ名と署名付きの印で結びつける][adr-20261016-001-ci-states-merge-ready-limit-and-pr-issue-link] | accepted | 2026-10-16 |
| ADR-20261016-002 | [受付 AI はすべての Issue で作業指示の下書きを書き、AI に渡す文は人が承認した下書きか作業指示コメントだけにする][adr-20261016-002-intake-ai-drafts-every-issue] | accepted | 2026-10-16 |
| ADR-20261016-003 | [dev-autopilot を基本設計の前に別のリポジトリへ移し、ほかのリポへの適用と配布は段階 8 の後に決める][adr-20261016-003-move-to-own-repository-before-basic-design] | accepted | 2026-10-16 |
| ADR-20261017-001 | [dev-autopilot の履歴を新しいリポへ引き継ぎ、PR の番号を business-os の参照に書き換えて、メンテナの鍵で署名を付け直す][adr-20261017-001-carry-history-into-own-repository] | accepted | 2026-10-17 |
| ADR-20261017-002 | [レビューの 3 つの軸は進行ボットが軸ごとに別のセッションで動かしてどの役にも Agent ツールを与えず、PR のコメントは AI に渡さず、規約の正典に触れる diff は push しない][adr-20261017-002-review-axis-sessions-pr-comments-and-canon] | accepted | 2026-10-17 |
| ADR-20261017-003 | [進行ボットを Plugin に同封し、Plugin を入れたプロジェクトで setup が利用者のマシンに版を写して定期の実行を登録する][adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot] | accepted | 2026-10-17 |
| ADR-20261017-004 | [AI のセッションには写しの skills/ だけを読ませ、setup の GitHub の準備は人の gh で行い、check と /dev-autopilot は進行ボットのジョブを起こし、マシンの設定は OS ごとの決まった場所に当てる先のリポごとに置く][adr-20261017-004-plugin-skills-read-and-human-skills-auth] | accepted | 2026-10-17 |
| ADR-20261017-005 | [AI のセッションは MCP のサーバーを読み込まず、claude.ai のコネクタを設定・環境変数・起動の引数と道具の deny で重ねて止める][adr-20261017-005-ai-sessions-load-no-mcp-servers] | accepted | 2026-10-17 |
| ADR-20261017-006 | [AI のセッションの一時フォルダは、セッションごとの短いパスを CLAUDE_CODE_TMPDIR で渡し、共有の一時フォルダとほかのセッションの一時フォルダを読めなくする][adr-20261017-006-per-session-short-tmpdir] | accepted | 2026-10-17 |
| ADR-20261017-007 | [全体を止める合図はリポの変数に置いて実行の途中でも読み、書かずに試す合図はリポの設定に残し、リポの設定は base ブランチの先端から読む][adr-20261017-007-stop-switch-in-repo-variable] | accepted | 2026-10-17 |
| ADR-20261018-001 | [AI のセッションは SIGTERM で止めて残ったコマンドも止め、途中で止めた回の費用は 1 セッションの費用の上限とみなし、利用枠は費用で数える][adr-20261018-001-stop-session-and-count-usage] | accepted | 2026-10-18 |
| ADR-20261018-002 | [計画 AI の自己申告で計画し直すときは、次回の実行に回さずに続けて行い、やり直した計画だけを投稿する][adr-20261018-002-plan-redo-in-same-run] | accepted | 2026-10-18 |
| ADR-20261018-003 | [作業 AI が最初の実装に `sonnet` を使ってよいかは、まず人が手直ししたかで仮に決め、AI のレビューと AI の修正の結果で見直す][adr-20261018-003-count-model-raise-at-stage-7] | superseded by ADR-20261024-002 | 2026-10-18 |
| ADR-20261018-004 | [計画は AI がレビューする。人が計画を読む間は結果を記録だけに残して人の判断と比べ、計画を止めるのに使うかは、その結果を見て決める][adr-20261018-004-ai-reviews-plans-in-shadow-first] | accepted | 2026-10-18 |
| ADR-20261018-005 | [設定の表には、要件が決めた初期値も値で書いて要件へリンクし、要件の値を変えるときは同じ PR で表も直す][adr-20261018-005-settings-table-shows-required-values] | accepted | 2026-10-18 |
| ADR-20261019-001 | [直したはずの指摘がレビューで再び出たら、間が空いていても人に渡す。同じ指摘かは、指摘ごとの ID を review の印に書いて見分ける][adr-20261019-001-repeated-findings-by-ids] | accepted | 2026-10-19 |
| ADR-20261019-002 | [PR のレビューに、3 つの軸が挙げなかった問題を探す「見落とし」の軸を足す。最初は結果を記録だけに残し、人のレビューと比べてから使うかを決める][adr-20261019-002-review-gaps-axis-in-shadow-first] | accepted | 2026-10-19 |
| ADR-20261019-003 | [計画のレビューで直すべき点が見つかったら、同じ回の実行で計画 AI に計画し直させる。計画・実装工程（段階 5）から始める][adr-20261019-003-replan-on-plan-review-from-stage-5] | accepted | 2026-10-19 |
| ADR-20261019-004 | [方針に係る要望は、受付 AI が見分けて下書きに書き添え、計画が push しないパスに触れるなら実装の前に人に渡す][adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths] | accepted | 2026-10-19 |
| ADR-20261020-001 | [自動化用の GitHub アカウントはいまは分けず、印を書いたと認めるアカウントを設定の一覧に持つ][adr-20261020-001-bot-accounts-as-list] | accepted | 2026-10-20 |
| ADR-20261020-002 | [進行 Bot は、GitHub に書き始める前に GitHub App に切り替え、自動化用のアカウントを分ける][adr-20261020-002-switch-to-github-app-before-writing] | accepted | 2026-10-20 |
| ADR-20261021-001 | [レビュー AI に Skill の道具を持たせず、進行 Bot が review Skill の本文に引数を入れて渡す][adr-20261021-001-review-skill-text-without-skill-tool] | accepted | 2026-10-21 |
| ADR-20261021-002 | [書かずに試すのは受け入れ工程（段階 4）だけにし、試した下書きは同じ文の間は作り直さず、試した回の費用を Issue の累計に数えない][adr-20261021-002-dry-run-only-at-intake-stage] | superseded by ADR-20261024-002 | 2026-10-21 |
| ADR-20261021-003 | [承認は agent-ready の付け直しでやり直し、push 未満の人の作業指示は読まず、作業指示が 2 つ以上なら止める][adr-20261021-003-reapprove-by-relabel-and-ignore-outside-instructions] | accepted | 2026-10-21 |
| ADR-20261021-004 | [印の署名を確かめる鍵を一覧にしていつでも替えられるようにし、無効な印で止めた Issue は、人が needs-human を外せば知らせた印を数えずに続ける][adr-20261021-004-marker-key-list-and-resume-after-invalid-markers] | accepted | 2026-10-21 |
| ADR-20261021-005 | [push しないパスのうち要件が名指しするものは進行 Bot のプログラムに固定し、リポの設定には足す分だけを持つ][adr-20261021-005-required-push-blocked-paths-in-program] | accepted | 2026-10-21 |
| ADR-20261021-006 | [当てる先のリポの owner と名前は、リポの設定を読む前に要る項目として、base ブランチの名前と GitHub App の 2 つの ID と一緒にマシンの設定に置く][adr-20261021-006-repo-owner-and-name-in-machine-settings] | accepted | 2026-10-21 |
| ADR-20261021-007 | [レビュー工程（段階 6）の比べるためのレビューは、本番のラウンドの後の実行に別の重い工程として起こし、済んだかを手元の記録で見分け、失敗を実行の失敗に、費用を Issue の累計に数えない][adr-20261021-007-compare-review-done-from-local-records] | superseded by ADR-20261024-002 | 2026-10-21 |
| ADR-20261022-001 | [検疫は、当てる先のリポの GitHub の URL のほかの URL を探して止め、AI には GitHub に投稿する文にほかの URL を書かせない][adr-20261022-001-quarantine-urls-outside-target-repo] | accepted | 2026-10-22 |
| ADR-20261022-002 | [3 つの軸の 🔴 🟡 が 0 件のラウンドは批評 AI のセッションを起こさず、IMPL-18 の 5 つの条件を 3 つの軸のセッションにだけ掛ける][adr-20261022-002-skip-critic-when-no-high-or-medium-findings] | accepted | 2026-10-22 |
| ADR-20261022-003 | [往復の回数は、作業 AI が直すか衝突を解いて push した回数（fix の印の数）で数え、同じ指摘の再発は、AI の変更の後のラウンドでだけ判定する][adr-20261022-003-count-round-trips-by-ai-pushes] | accepted | 2026-10-22 |
| ADR-20261022-004 | [人が呼ぶ intake の Skill は、check と同じく依頼のファイルで進行 Bot のジョブに受付 AI を起こさせ、人が呼ぶ Skill は依頼のファイルを sandbox の外で書く許可を人に求める][adr-20261022-004-intake-skill-via-request-file] | accepted | 2026-10-22 |
| ADR-20261022-005 | [人が戻した時刻より前に閉じた PR は止めずに結びつきから外し、止まった状態から戻す手順はどれも needs-human を付けたまま行って最後に外す][adr-20261022-005-unlink-pr-closed-before-return-and-remove-needs-human-last] | superseded by ADR-20261024-001 | 2026-10-22 |
| ADR-20261024-001 | [作業の単位の状態を主な流れと重ねる 3 本の軸で表し、人の操作はラベルと見出しのコメントで受け、Project の欄は表示だけにする][adr-20261024-001-work-unit-main-flow-and-overlay-axes] | accepted | 2026-10-24 |
| ADR-20261024-002 | [段階 4〜8 と試し（書かずに試す、記録だけの判定、比べるためのレビュー）を持たず、どの作業も最初から決まった担い手が行う][adr-20261024-002-no-stages-4-to-8-and-no-trials] | accepted | 2026-10-24 |
| ADR-20261024-003 | [AI のレビューは必須チェックがすべて通ってから行い、必須チェックが落ちた変更はレビューせずに AI が先に直す][adr-20261024-003-review-after-required-checks-pass] | accepted | 2026-10-24 |
| ADR-20261024-004 | [依頼の今の文をマスターの下書き 1 つにまとめ、検疫は AI に渡す前と GitHub に書く前に毎回行い、変えてはいけないパスは効く時期で分けて人の承認で変えられるようにする][adr-20261024-004-master-draft-quarantine-and-protected-paths] | accepted | 2026-10-24 |
| ADR-20261024-005 | [検疫で見つかったら止め方でなく安全の軸に値を入れて maintain 以上の人の安全の確認だけで外し、AI が書いた文で見つかったら全体の停止の Issue で全体を止める][adr-20261024-005-security-axis-and-halt-issue] | accepted | 2026-10-24 |
| ADR-20261024-006 | [マージと中止の後は片付けの前の状態を挟んで進行 Bot が片付け、先の依頼は先の作業がマージ済みで続け閉鎖済みなら AI が計画を見直し、載せ直しはレビュー合格の PR だけにする][adr-20261024-006-cleanup-states-and-dependencies] | accepted | 2026-10-24 |

<!--
行の例（表の最後に足す）:
| ADR-20261006-001 | [MADR テンプレートを採用する][adr-20261006-001-use-madr-template] | proposed | 2026-10-06 |

参照定義の例（ファイル末尾に足す）:
[adr-20261006-001-use-madr-template]: ./adr-20261006-001-use-madr-template.md
-->

[readme]: ./README.md
[memo]: https://github.com/Joymerrevent/business-os/blob/b8bc13eef06d430e24483168cfa0ff8edc1987e0/dev-autopilot/docs/dev-autopilot-requirements.md
[stages]: ../requirements/06-setup-and-stages.md
[req]: ../requirements/README.md
[adr-20261013-001-fresh-clone-per-ai-session]: ./adr-20261013-001-fresh-clone-per-ai-session.md
[adr-20261015-001-read-allowlist-and-secret-file]: ./adr-20261015-001-read-allowlist-and-secret-file.md
[adr-20261015-002-no-shared-dependency-store]: ./adr-20261015-002-no-shared-dependency-store.md
[adr-20261015-003-intake-scope-and-quarantine-report]: ./adr-20261015-003-intake-scope-and-quarantine-report.md
[adr-20261015-004-signed-markers-and-actor-checks]: ./adr-20261015-004-signed-markers-and-actor-checks.md
[adr-20261015-005-settings-values-and-machine-settings]: ./adr-20261015-005-settings-values-and-machine-settings.md
[adr-20261015-006-run-stages-baseline-agent-field-and-local-records]: ./adr-20261015-006-run-stages-baseline-agent-field-and-local-records.md
[adr-20261016-001-ci-states-merge-ready-limit-and-pr-issue-link]: ./adr-20261016-001-ci-states-merge-ready-limit-and-pr-issue-link.md
[adr-20261016-002-intake-ai-drafts-every-issue]: ./adr-20261016-002-intake-ai-drafts-every-issue.md
[adr-20261016-003-move-to-own-repository-before-basic-design]: ./adr-20261016-003-move-to-own-repository-before-basic-design.md
[adr-20261017-001-carry-history-into-own-repository]: ./adr-20261017-001-carry-history-into-own-repository.md
[adr-20261017-002-review-axis-sessions-pr-comments-and-canon]: ./adr-20261017-002-review-axis-sessions-pr-comments-and-canon.md
[adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot]: ./adr-20261017-003-bundle-bot-in-plugin-and-run-from-snapshot.md
[adr-20261017-004-plugin-skills-read-and-human-skills-auth]: ./adr-20261017-004-plugin-skills-read-and-human-skills-auth.md
[adr-20261017-005-ai-sessions-load-no-mcp-servers]: ./adr-20261017-005-ai-sessions-load-no-mcp-servers.md
[adr-20261017-006-per-session-short-tmpdir]: ./adr-20261017-006-per-session-short-tmpdir.md
[adr-20261017-007-stop-switch-in-repo-variable]: ./adr-20261017-007-stop-switch-in-repo-variable.md
[adr-20261018-001-stop-session-and-count-usage]: ./adr-20261018-001-stop-session-and-count-usage.md
[adr-20261018-002-plan-redo-in-same-run]: ./adr-20261018-002-plan-redo-in-same-run.md
[adr-20261018-003-count-model-raise-at-stage-7]: ./adr-20261018-003-count-model-raise-at-stage-7.md
[adr-20261018-004-ai-reviews-plans-in-shadow-first]: ./adr-20261018-004-ai-reviews-plans-in-shadow-first.md
[adr-20261018-005-settings-table-shows-required-values]: ./adr-20261018-005-settings-table-shows-required-values.md
[adr-20261019-001-repeated-findings-by-ids]: ./adr-20261019-001-repeated-findings-by-ids.md
[adr-20261019-002-review-gaps-axis-in-shadow-first]: ./adr-20261019-002-review-gaps-axis-in-shadow-first.md
[adr-20261019-003-replan-on-plan-review-from-stage-5]: ./adr-20261019-003-replan-on-plan-review-from-stage-5.md
[adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths]: ./adr-20261019-004-flag-policy-requests-and-stop-plans-on-blocked-paths.md
[adr-20261020-001-bot-accounts-as-list]: ./adr-20261020-001-bot-accounts-as-list.md
[adr-20261020-002-switch-to-github-app-before-writing]: ./adr-20261020-002-switch-to-github-app-before-writing.md
[adr-20261021-001-review-skill-text-without-skill-tool]: ./adr-20261021-001-review-skill-text-without-skill-tool.md
[adr-20261021-002-dry-run-only-at-intake-stage]: ./adr-20261021-002-dry-run-only-at-intake-stage.md
[adr-20261021-003-reapprove-by-relabel-and-ignore-outside-instructions]: ./adr-20261021-003-reapprove-by-relabel-and-ignore-outside-instructions.md
[adr-20261021-004-marker-key-list-and-resume-after-invalid-markers]: ./adr-20261021-004-marker-key-list-and-resume-after-invalid-markers.md
[adr-20261021-005-required-push-blocked-paths-in-program]: ./adr-20261021-005-required-push-blocked-paths-in-program.md
[adr-20261021-006-repo-owner-and-name-in-machine-settings]: ./adr-20261021-006-repo-owner-and-name-in-machine-settings.md
[adr-20261021-007-compare-review-done-from-local-records]: ./adr-20261021-007-compare-review-done-from-local-records.md
[adr-20261022-001-quarantine-urls-outside-target-repo]: ./adr-20261022-001-quarantine-urls-outside-target-repo.md
[adr-20261022-002-skip-critic-when-no-high-or-medium-findings]: ./adr-20261022-002-skip-critic-when-no-high-or-medium-findings.md
[adr-20261022-003-count-round-trips-by-ai-pushes]: ./adr-20261022-003-count-round-trips-by-ai-pushes.md
[adr-20261022-004-intake-skill-via-request-file]: ./adr-20261022-004-intake-skill-via-request-file.md
[adr-20261022-005-unlink-pr-closed-before-return-and-remove-needs-human-last]: ./adr-20261022-005-unlink-pr-closed-before-return-and-remove-needs-human-last.md
[adr-20261024-001-work-unit-main-flow-and-overlay-axes]: ./adr-20261024-001-work-unit-main-flow-and-overlay-axes.md
[adr-20261024-002-no-stages-4-to-8-and-no-trials]: ./adr-20261024-002-no-stages-4-to-8-and-no-trials.md
[adr-20261024-003-review-after-required-checks-pass]: ./adr-20261024-003-review-after-required-checks-pass.md
[adr-20261024-004-master-draft-quarantine-and-protected-paths]: ./adr-20261024-004-master-draft-quarantine-and-protected-paths.md
[adr-20261024-005-security-axis-and-halt-issue]: ./adr-20261024-005-security-axis-and-halt-issue.md
[adr-20261024-006-cleanup-states-and-dependencies]: ./adr-20261024-006-cleanup-states-and-dependencies.md
