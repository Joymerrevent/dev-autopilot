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
