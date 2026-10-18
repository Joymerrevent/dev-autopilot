# ADR 一覧

起票した ADR を、番号の順に並べる。起票の手順と status の運用は [README][readme] にある。
2026-10-08 に起票した 9 本は、設計が日ごとに変わる段階では要件メモとの二重保守になるため 2026-10-10 にいったん削除した（[要件メモ][memo]の冒頭の決定）。
段階 4（棚卸しと受け入れ。「[段階][stages]」）の後に、[要件][req]の決定を元に書き直す。
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
| ADR-20261018-003 | [作業 AI が最初の実装に `sonnet` を使ってよいかは、人が手直しせずにマージできたかで決め、修正で `opus` に切り替えた回数は、AI が修正を行う段階から数える][adr-20261018-003-count-model-raise-at-stage-7] | proposed | 2026-10-18 |

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
