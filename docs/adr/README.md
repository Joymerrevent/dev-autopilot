# ADR（設計判断の記録）

設計判断を ADR（Architecture Decision Record）として残すフォルダ。
ADR の書式は [MADR][madr] 4.0.0 フル版を日本語にした [adr-template.md][template] を使う。
起票済みの ADR は [ADR 一覧][index] にまとめる。

## ファイル名は起票日・連番・英語の題でつくる

ファイル名を名前順に並べると、ADR が起票順に並ぶ。

```text
adr-<yyyymmdd>-<nnn>-<title>.md
```

| 部分 | 決まり | 例 |
| --- | --- | --- |
| `adr-` | 固定の接頭辞 | `adr-` |
| `<yyyymmdd>` | 起票日（frontmatter の `created` と同じ日付） | `20261006` |
| `<nnn>` | 起票日ごとの 3 桁の連番。同じ日に複数起票したときに 002, 003 と増やす | `001` |
| `<title>` | 決定内容を表す英語の短い題。小文字の kebab-case（単語をハイフンで区切る） | `use-madr-template` |

区切りはすべてハイフン（`-`）にする。

例: `adr-20261006-001-use-madr-template.md`

ADR の番号の決まりを、起票時に確かめる順に並べる。

- 起票前に、同じ日付の ADR がほかのブランチや open な PR にないかを確かめる。
- ADR の番号は、ファイル名の起票日と連番に `ADR-` を付けたものとする。本文やほかの ADR からは、ADR「MADR テンプレートを採用する」（ADR-20261006-001）のように、題に番号を添えて参照する。
- 一度付けた番号は変えない。欠番を詰めない・再利用しない。

## 起票の PR には ADR 本文と一覧の 1 行だけを入れる

決定を反映する実装は、ADR の承認後に別の PR で行う。起票は次の順に進める。

1. 同じ論点の ADR がすでにないかを、[ADR 一覧][index]・ADR 本文の検索（ADR フォルダで `grep -rli '<キーワード>' .`）・open な PR で探す。
   - 同じ論点の ADR が `proposed` なら、新しく起票せずにその ADR へ追記する。
   - 同じ論点の ADR が `accepted` なら、新しい ADR で置き換える（下記「承認済みの ADR は書き換えず、新しい ADR で置き換える」）。
   - 関連する ADR があれば、新しい ADR の「補足情報」にリンクを書く。
2. [adr-template.md][template] をコピーし、上記の規則でファイル名を付ける。
3. frontmatter の `status` を `proposed`、`created` と `updated` を起票日、`author` を起票者にする。AI が起票したときは AI の名前と、起票を依頼した人を書く（例: `Claude Code（メンテナの依頼で起票）`）。承認・却下は `decision-makers` が行う。
4. 本文を書く。任意の項目（テンプレート内に「任意の項目」とコメントがあるもの）は、不要なら削除してよい。
5. [ADR 一覧][index] に 1 行足す。
6. ADR 本文と一覧の 1 行だけを含む PR を作る。

## ADR と一覧の書き方を揃える

人やスキルによって書き方が揺れやすい部分の決まりを、ADR 本文・一覧の順に並べる。

- frontmatter の `status` は引用符を付けずに書く（例: `status: proposed`、`status: superseded by ADR-<新しい番号>`）。
- Markdown のリンクは参照スタイルで書く。本文には `[表示テキスト][ラベル]` と書き、`[ラベル]: URL` の定義はファイル末尾にまとめる。`[表示テキスト](URL)` の書き方は使わない。
- ADR 一覧の 1 行は、番号・タイトル・status・作成日の順に書き、リンクはタイトルに付ける。リンクのラベルにはファイル名から `.md` を除いたものを使い、定義は一覧のファイル末尾に置く。

ADR 一覧の 1 行の例:

```markdown
| ADR-20261006-001 | [MADR テンプレートを採用する][adr-20261006-001-use-madr-template] | proposed | 2026-10-06 |

[adr-20261006-001-use-madr-template]: ./adr-20261006-001-use-madr-template.md
```

## 承認済みの ADR は、決定を変えるなら新しい ADR、記述の誤りなら訂正の注記で直す

本文を書き直してよいのは `proposed` の間だけで、`proposed` 以外の status を付けられるのは決定者だけである。

| status | 意味 | 付ける人 |
| --- | --- | --- |
| `proposed` | 提案中。議論の結果で本文を書き直してよい | 起票者 |
| `accepted` | 承認済み | 決定者（frontmatter の `decision-makers`） |
| `rejected` | 却下 | 決定者 |
| `deprecated` | 非推奨。決定の前提がなくなった | 決定者 |
| `superseded by ADR-<番号>` | 新しい ADR で置き換えた | 決定者 |

`accepted` の後の直し方は、決定を変えるか、記述の誤りを直すかで分かれる。
どちらか迷ったら「直した後の注記を読んで、決定が変わったと思う人がいるか」で決める。いるなら決定の変更として新しい ADR を起票し、いないなら訂正の注記で直す。

| 直したいこと | 直し方 | 古い ADR の status |
| --- | --- | --- |
| 決定をすべて置き換える | 新しい ADR を起票し、承認後に古い ADR へ「置き換えの追記」を足す | `superseded by ADR-<新しい番号>` に変える |
| 決定の一部だけを改める | 新しい ADR を起票し、承認後に古い ADR へ「改めた旨の注記」を足す | `accepted` のまま |
| 事実の誤り・古くなった記述を直す（決定は変わらない） | 古い ADR を直接直し、「訂正の注記」を足す | `accepted` のまま |
| 決定に関わる新しい事実を書き足す（決定は変わらない） | 古い ADR に「追記」を足す | `accepted` のまま |
| 本文に書いたファイルのパスが移った | 本文は決定したときのまま残し、末尾の「パスの注記」に今の場所を書く | `accepted` のまま |

### 決定を変えるときは、古い ADR に注記を足して本文を残す

- 新しい ADR の冒頭には、どの ADR のどの決定を置き換える（または改める）か、どの決定を据え置くかを書き、古い ADR へリンクする。
- すべて置き換えたときは、古い ADR の「決定（Decision Outcome）」節の先頭に追記を足す。古い本文は当時の決定の記録として残す。

  ```markdown
  > **［2026-10-06 追記・supersede］** レポートの保存先の決定は [ADR-<新しい番号>][new-adr] で置き換えた。レポートに Allure を使うこと自体は変わらない。以下の記述は当時の決定の記録として残す。
  ```

- 一部だけ改めたときは、古い ADR のタイトルのすぐ下に注記を足す。古い ADR の status は `accepted` のまま変えない。

  ```markdown
  > **Amended by [ADR-<新しい番号>][new-adr]（2026-10-06）**: 実行環境の決定のうち、CI での実行方法を改めた。ローカルでの実行方法は据え置く。
  ```

### 決定を変えない直しは、黙って直さずに注記を残す

- 事実の誤りや古くなった記述は、誤った記述に取り消し線（`~~…~~`）を引き、そのすぐ後ろに「訂正（日付）」で始まる注記を続ける。
  注記には、何が違ったか・今はどうか・決定そのものは変わらないことの 3 つを書く。読んだ人が、決定が変わったのかを判断できるようにするため。

  ```markdown
  ~~Node 18 以上で動かす。~~
  **訂正（2026-10-06）**: Node 18 は EOL のため、最低バージョンは Node 22 になっている。既定の HTTP クライアントを fetch にするという決定は変わらない。
  ```

- 誤りが ADR 全体にかかるときは、タイトルのすぐ下に引用ブロックで訂正の注記を置く。
- 決定は変えずに新しい事実を書き足すときは、関係する箇所のすぐ後ろに追記を足す。

  ```markdown
  > **［2026-10-06 追記］** 表に無かったエラーコード 113 も、同じ扱いで別のエラーとして返す。
  ```

- 本文に書いたファイルのパスは、決定したときのまま残す。あとでファイルを移したり名前を変えたりしたら、ADR の末尾に「パスの注記」節を作り（あれば使い）、今の場所を 1 行足す。
  リンクの参照定義（ファイル末尾の `[ラベル]: パス`）は、移したときに今のパスへ直す。

  ```markdown
  ## パスの注記

  - `src/client.ts` → `src/porters-client.ts`（2026-10-06・ファイル名の規則の変更による）
  ```

- 誤字・脱字は、注記を付けずに直してよい。
- どの直し方でも、直した日に frontmatter の `updated` をその日の日付にする。

### AI は status を変えず、決定の反映は承認の後に行う

status を変えるのは決定者だけで、ほかの文書や実装への反映は `accepted` になるまで待つ。

- `status` を変えたときも、`updated` をその日の日付にする。
- Claude Code などの AI は、`proposed` 以外の status を付けない。AI が起票した ADR も、承認・却下は決定者が行う。
- ADR の決定内容を、ほかの文書（`CLAUDE.md`・設計書など）や実装へ反映するのは、`accepted` になった後に行う。

## 棚卸しでは ADR の決定と設定ファイルの実物を突き合わせる

ADR の決定は、設定ファイルやコードに前提として焼き付く。ADR 本文と ADR 同士のリンクを読むだけでは、設定ファイルの実物とのずれに気づけない。
`accepted` の ADR を見直すときは、決定が焼き付いた先を 1 本ずつ開いて、いまの実物と突き合わせる。

- 突き合わせる先は、次のようなファイルである。
  - CI のワークフローとインフラの定義
  - `package.json` の `engines` と依存、依存更新ボットの設定
  - lint・型検査の設定と、テスト・カバレッジの対象・除外の設定
- ADR が「除外する」「〜のみ」「暫定」「将来」と書いた対象は、いま実物がどうなっているかを必ず確かめる。除外したはずのディレクトリに、後から実際の処理が入っていることがある。
- ずれを見つけても、`accepted` の ADR を直接直さない。決定を変えるなら新しい ADR で置き換え、設定を直すなら実装の PR を別に作る。

## Claude Code のスキルを使って ADR を書く

ADR の起票を Claude Code に任せる場合は、[documentation-and-adrs][addy-skill] スキルを入れる。
documentation-and-adrs は [skills.sh][skills-sh] で公開されているスキルで、配布元は `addyosmani/agent-skills`（MIT ライセンス）である。

### 推奨スキルは既存の規則に合わせて起票する

documentation-and-adrs の SKILL.md には、起票の前に既存の ADR とプロジェクトの規則を読むと書かれている。
ファイル名・番号・見出しも、既存の規則があればスキルの既定より優先すると明記されている。
ADR 用の主なスキル 3 つを、この README の規則に合わせやすい順に並べる。

| スキル | 既存の規則への合わせ方 | この README の規則との相性 |
| --- | --- | --- |
| [addyosmani/agent-skills@documentation-and-adrs][addy-skill] | 既存の規則があれば既定より優先すると明記している | 合う |
| [wshobson/agents@architecture-decision-records][wsh-skill] | 既存の規則に合わせる指示がない。`ADR-0001` 形式の例と `docs/adr/` の構成を示す | 合わせるには毎回指示が必要 |
| [existential-birds/beagle@adr-writing][beagle-skill] | MADR を使うが、保存先 `docs/adrs/NNNN-…md` と status `draft` を手順に固定している | 合わない |

documentation-and-adrs は ADR 専用ではなく、コードコメント・API ドキュメント・README・CHANGELOG の書き方も含む。

### 推奨スキルを入れる

```bash
npx skills add addyosmani/agent-skills@documentation-and-adrs -g -y
```

`-g` はユーザー単位（全プロジェクト共通）で入れる指定。外すと、コマンドを実行したプロジェクトだけに入る。

ほかの ADR 用スキルを探す場合は、[find-skills][find-skills] スキルに「ADR を書くスキルを探して」と頼むか、`npx skills find adr` を実行する。find-skills は次のコマンドで入る。

```bash
npx skills add vercel-labs/skills@find-skills -g -y
```

[madr]: https://adr.github.io/madr/
[template]: ./adr-template.md
[index]: ./index.md
[skills-sh]: https://skills.sh/
[addy-skill]: https://skills.sh/addyosmani/agent-skills/documentation-and-adrs
[wsh-skill]: https://skills.sh/wshobson/agents/architecture-decision-records
[beagle-skill]: https://skills.sh/existential-birds/beagle/adr-writing
[find-skills]: https://skills.sh/vercel-labs/skills/find-skills
