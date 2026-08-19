<div align="center">

# shell-geinin

Claude Code と Codex CLI のためのクロスプラットフォーム対応 Agent Skill

[![ライセンス: MIT](https://img.shields.io/github/license/redpeacock78/shell-geinin?style=flat-square)](LICENSE)
[![CI](https://github.com/redpeacock78/shell-geinin/actions/workflows/ci.yml/badge.svg)](https://github.com/redpeacock78/shell-geinin/actions/workflows/ci.yml)
[![Skills CLIでインストール](https://img.shields.io/badge/install-npx%20skills-111827?style=flat-square)](https://github.com/vercel-labs/skills)
[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/redpeacock78/shell-geinin)

</div>

[🍔 English](README.md) | 🍡 日本語

> 実行中のシェルとデータモデルに合う、明確で安全なコマンドを選ぶためのAgent Skillです。

## 🚀 まず使う

```sh
npx skills add redpeacock78/shell-geinin
```

インストール後は、リポジトリ調査、ファイル検索、Gitの確認、構造化データの整理、コマンド設計に利用できます。

ホストエージェントから次の名前で呼び出せます。

- Claude Code：`/shell-geinin`
- Codex CLI：`$shell-geinin`

## ⬇️ インストール

Claude Code と Codex CLI の両方に `shell-geinin` をインストールする場合は、次のコマンドを使います。

```sh
npx skills add redpeacock78/shell-geinin \
  --skill shell-geinin \
  --agent claude-code codex \
  --global \
  --yes
```

プロジェクト単位で使う場合は `--global` を省略します。

シンボリックリンクではなく独立したコピーを配置する場合は `--copy` を追加します。

> [!TIP]
> インストール前に `npx skills add redpeacock78/shell-geinin --list` を実行すると、このリポジトリが公開しているSkillを確認できます。

## 🧠 できること

- OS名だけで判断せず、実行中のシェルを調べてからコマンドを選ぶ
- POSIX系シェルではテキストストリームを、PowerShellではオブジェクトを保ったまま処理する
- データの絞り込み、項目選択、グループ化、上限設定を生成元に近い場所で行う
- ファイル名、クォート、順序、空白、信頼できない入力を正しさの問題として扱う
- シェルパイプラインが適さない場合は、`awk`、`sed`、`jq`、`git`、`gh` などの保守されたツールへ切り替える

## 🌐 対応する実行環境

| 実行環境 | 方針 |
| --- | --- |
| POSIX系シェル、macOS、BSD、Linux、BusyBox | 移植性を保ったテキストストリーム処理 |
| PowerShell | オブジェクトとプロパティを保ったパイプライン処理 |
| Windows `cmd.exe` | クォートを明示した単純なテキスト処理 |
| WSL、MSYS2、Cygwin、Git Bash | Unix系パイプラインとパス境界の両立 |

## 📊 効果の測定

この測定では、エージェントがコンテキストへ持ち込むコマンド文字列と標準出力の量を調べました。

Claude CodeやCodex CLIの内部生成トークン、システムプロンプト、ツール連携のトークン、標準エラー出力、課金トークンは測定対象に含めていません。

意図は、短い構文による削減と、生成元での上限設定や項目射影による削減を分けて確認することです。

### 測定方法

- 主なトークナイザには `o200k_base` を使い、`cl100k_base` でも傾向を確認しました。
- 合計トークン数は、コマンド文字列のトークン数と標準出力のトークン数の合計です。
- 入力には、このリポジトリのファイル一覧、5,000行のログ、120件のコミット履歴、1,000件のJSONレコードを使いました。
- 追加測定では、条件を固定した20万行のログ、10万件のNDJSON、1,000件のコミット履歴を使いました。
- ベースラインは広い範囲を取得し、shell-geinin型は生成元で出力を制限するか、必要な項目だけを射影しました。
- ファイル一覧と検索結果の25件取得は、出力がバイト単位で一致することを確認しました。

代表的な比較は次のとおりです。

```sh
grep -n 'shell' events.log
rg -n -m 25 'shell' events.log

git log --all --stat --oneline
git log -20 --oneline --no-decorate

jq -c '.[]' items.json
jq -c '.[] | {id,status}' items.json
```

| パターン | コマンドトークン | 出力トークン | 合計トークン | 削減率 |
| --- | ---: | ---: | ---: | ---: |
| ファイル一覧、同じ出力 | 26 → 14 | 87 → 87 | 113 → 101 | 10.6% |
| 検索結果の先頭25件、同じ出力 | 12 → 12 | 517 → 517 | 529 → 529 | 0.0% |
| 検索結果の全件取得 → 先頭25件 | 8 → 12 | 107,335 → 517 | 107,343 → 529 | 99.5% |
| Git履歴の全件stat → 直近20件の要約 | 9 → 12 | 1,122 → 184 | 1,131 → 196 | 82.7% |
| JSONの全項目 → `id,status` | 8 → 12 | 40,923 → 9,000 | 40,931 → 9,012 | 78.0% |

大きな削減を生むのは、コマンド構文の短縮よりも、出力の上限設定と項目の射影です。

同じ出力を返す検索結果の比較では、どちらも25行を返すため、トークン削減率は0.0%でした。
生成元で制限する形式は処理量やサブプロセス数を減らす可能性がありますが、その効果はこの表のトークン数に含めていません。

### 大規模・複雑なケース

ここでは、広い範囲を取得する要求と、作業目的に合わせて絞った要求を意図的に比較しています。
出力はバイト単位では一致しないため、同じ出力が必要な場合は上の比較表を基準にしてください。

入力は、20万行・27.1 MBのログ、10万件・16.9 MBのNDJSON、1,000件のコミット履歴です。
主な `o200k_base` の測定結果は次のとおりです。

```sh
rg -n 'status=(ERROR|WARN) service=(api|worker)' events-200k.log
rg -n -m 50 'status=(ERROR|WARN) service=(api|worker)' events-200k.log

jq -c '.' records-100k.ndjson
jq -c -n 'limit(50; inputs | select(.status == "error" and .service == "api") | {id,status,service})' records-100k.ndjson

git -C history-repo log --format=fuller --stat
git -C history-repo log -50 --format='%h %s' --no-decorate
```

| パターン | コマンドトークン | 出力トークン | 合計トークン | 削減率 |
| --- | ---: | ---: | ---: | ---: |
| 20万行ログ、条件一致の全行 → 先頭50行 | 21 → 25 | 1,846,421 → 2,257 | 1,846,442 → 2,282 | 99.9% |
| 10万件NDJSON、全レコード → `error/api` の必要項目・先頭50件 | 10 → 40 | 4,599,001 → 689 | 4,599,011 → 729 | 99.98% |
| 1,000件のコミット、詳細全件 → 直近50件の1行要約 | 14 → 21 | 93,562 → 667 | 93,576 → 688 | 99.3% |

`cl100k_base` でのクロスチェックは、削減率を小数点以下1桁に丸めると99.9%、100.0%、99.3%でした。
したがって、主な要因はトークナイザの癖ではなく、出力件数と1件あたりの項目幅です。
これらは条件を固定した測定値であり、すべてのリポジトリや検索で同じ削減率になることを保証するものではありません。

### インラインPythonとshell-geininの比較

大きな構造化データを扱うとき、エージェントは使い捨ての `python3 -c` を書きがちです。
ここでは処理内容と標準出力を揃え、コマンド文字列と共通の出力だけを測定しました。
3組とも標準出力がバイト単位で一致しています。

`o200k_base` の値は「コマンドトークン + 出力トークン = 合計トークン」で示します。

| パターン | インラインPython | shell-geinin型 | 合計削減率 |
| --- | ---: | ---: | ---: |
| 20万行ログ、複数条件検索と先頭50件 | 73 + 2,257 = 2,330 | 25 + 2,257 = 2,282 | 2.1% |
| 10万件NDJSON、条件選択、項目射影、先頭50件 | 89 + 689 = 778 | 40 + 689 = 729 | 6.3% |
| 20万行ログ、サービス別集計 | 89 + 10 = 99 | 62 + 10 = 72 | 27.3% |

代表的なコマンドの組み合わせは次のとおりです。

```sh
python3 -c 'import re,itertools; p=re.compile(r"status=(ERROR|WARN) service=(api|worker)"); [print(f"{n}:{line}",end="") for n,line in itertools.islice(((n,line) for n,line in enumerate(open("events-200k.log"),1) if p.search(line)),50)]'
rg -n -m 50 'status=(ERROR|WARN) service=(api|worker)' events-200k.log

python3 -c 'import json,itertools; records=(json.loads(line) for line in open("records-100k.ndjson")); selected=(r for r in records if r["status"]=="error" and r["service"]=="api"); [print(json.dumps({key:r[key] for key in ("id","status","service")},separators=(",",":"))) for r in itertools.islice(selected,50)]'
jq -c -n 'limit(50; inputs | select(.status == "error" and .service == "api") | {id,status,service})' records-100k.ndjson

python3 -c 'from collections import Counter; counts=Counter(line.split()[2].split("=")[1] for line in open("events-200k.log") if line.split()[1].split("=")[1] in ("ERROR","WARN") and line.split()[2].split("=")[1] in ("api","worker")); print("\n".join(f"{key} {counts[key]}" for key in sorted(counts)))'
awk '$2 ~ /^status=(ERROR|WARN)$/ && $3 ~ /^service=(api|worker)$/ { sub(/^service=/, "", $3); count[$3]++ } END { for (service in count) print service, count[service] }' events-200k.log | sort
```

この比較は、Pythonが常に不適切だと示すものではありません。
両方が数千行を返す場合は共通の出力が大半を占めるため、構文の差による削減は小さくなります。
結果が短い集計ではコマンド文字列の割合が増えるため、shell-geinin型の削減率が大きくなります。
シェルの組み合わせが読みにくく保守しにくくなる場合は、テスト可能なスクリプトへ昇格する方針を取ります。

### Codex CLIでの制御A/B測定

上の測定は、シェルのコマンド文字列と標準出力を使った代理指標です。
Codexでは `codex exec --json` から改行区切りのJSONイベントを取得でき、`codex exec resume` で同じセッションを継続できます。
仕様は[OpenAI公式のCodex CLIコマンドリファレンス](https://learn.chatgpt.com/docs/developer-commands?surface=cli)を参照してください。

以前の1セッション測定は、以下の結果で置き換えました。
以前のbaselineにはターンごとのSkill使用禁止を入れていなかったため、Codex自身がインストール済みSkillを読み込める状態でした。
今回のbaselineでは、コマンドイベントに `SKILL.md` 参照がないことを確認しています。

#### 測定設計

- 各条件を独立した `M=5` セッションで実行し、各セッションで同じ `N=10` タスクを実行
- baselineは暖気を置かず、1ターン目から本番タスクを実行
- shell-geinin型はSkillだけを読む暖気を1ターン実行し、その後に同じ10タスクを実行
- 環境はmacOS/zsh、Codex CLI `0.148.0`、読み取り専用Sandbox、推論強度 `low`、同じローカル設定モデル
- 入力データは固定生成した12万行、11.4 MBのログと、5万件、5.5 MBのNDJSON
- 各タスクは読み取り専用で、同じ順序で実行し、短い集計値または選択レコードだけを返す
- `M` は独立セッション間の揺らぎを測り、`N` はSkill暖気の償却を測る

baselineには、Agent Skillや `SKILL.md` を読まないこと、shell-geininを使わないこと、インラインPythonは許可することを毎ターン指示しました。
shell-geinin型では、暖気ターンでSkill全体を読み、そのセッションを継続しました。
この指示は測定ハーネスの制御であり、通常のタスク入力を推奨するものではありません。

**定常比較**ではshell-geinin型の暖気を除外します。
**償却後比較**ではshell-geinin型だけ `(暖気 + Nタスク) / N` を計算し、baselineには架空の暖気を加えません。

- `input` と `cached input` は各 `turn.completed.usage` の値
- `uncached input` は `input - cached input`
- `command chars` と `stdout bytes` は完了したコマンドイベントから集計
- `shell I/O proxy` は `command chars + stdout bytes` で、トークナイザーの値でも課金トークンでもない
- `output` と `reasoning` は副次指標として記録し、主要な代理指標の比較には加えない

#### 定常状態の結果

10タスクを実行した50ターンの平均です。
括弧内は5セッションのセッション平均の中央値です。

| タスクあたりの指標 | baseline | shell-geinin型 | 平均差 |
| --- | ---: | ---: | ---: |
| Input tokens | 121,986 (119,854) | 129,494 (125,492) | +6.2% |
| Uncached input tokens | 10,493 (10,695) | 6,977 (5,121) | -33.5% |
| Command chars | 549 (544) | 517 (476) | -5.9% |
| 標準出力bytes | 21,300 (343) | 261 (273) | -98.8% |
| Shell I/O proxy | 21,849 (930) | 778 (733) | -96.4% |
| Output tokens、副次指標 | 535 (541) | 615 (544) | +15.0% |
| Reasoning tokens、副次指標 | 97 (95) | 213 (169) | +121.0% |

baselineの1タスクでは、中間コマンドが1,049,466 bytesの標準出力を返しました。
最終的な集計値は正しかったものの、出力上限を設けていない処理でした。
平均値にはこの外れ値を含め、セッション中央値も併記しています。

回答内容を確認したところ、baselineは50タスク中49タスク、shell-geinin型も50タスク中49タスクで期待した集計値を返しました。
失敗したのはbaselineのセッション3のタスク1（4つのカウントをすべて0と回答）と、shell-geinin型のセッション4のタスク1（fixtureが非標準形式だと誤認してカウントを返さなかった）です。
この2ターンもトークン集計には含めているため、表の数値は試行した作業量であり、正答率を補正した削減量ではありません。

暖気自体の平均は、Input tokensが119,673、Uncached input tokensが22,137、Shell I/O proxyが20,073 bytesでした。
暖気を含めて `N=10` で割ったセッション全体の平均は次のとおりです。

| セッション償却後のタスクあたり指標 | baseline `N/N` | shell-geinin型 `(暖気 + N) / N` | 平均差 |
| --- | ---: | ---: | ---: |
| Input tokens | 121,986 | 141,461 | +16.0% |
| Uncached input tokens | 10,493 | 9,191 | -12.4% |
| Shell I/O proxy | 21,849 | 2,785 | -87.3% |

baselineの外れ値が償却後Shell I/Oの平均を大きく押し上げています。
セッション中央値ではbaselineが930、shell-geinin型が2,741であり、10タスクを実行しても暖気が常に回収されるわけではありません。

`N` を変えたときの推移は次のとおりです。

| N | 定常Uncached input、baseline | 定常Uncached input、shell-geinin型 | 償却後Shell I/O、baseline | 償却後Shell I/O、shell-geinin型 |
| ---: | ---: | ---: | ---: | ---: |
| 1 | 23,497 | 8,914 | 2,114 | 21,856 |
| 2 | 16,406 | 6,543 | 1,375 | 11,076 |
| 5 | 11,966 | 8,017 | 43,397 | 5,247 |
| 10 | 10,493 | 6,977 | 21,849 | 2,785 |

この結果は、今回のfixture、CLIバージョン、モデル設定、タスク列に対する制御A/Bです。
定常状態ではshell-geinin型が未キャッシュ入力とコマンド出力を抑えましたが、総Input tokens、モデルのOutput、Reasoning、暖気の回収期間はタスクによって変わります。
したがって、一般的な削減率としては扱えません。

1セッションずつ再現する場合は、次の形にします。

```sh
# baseline：1ターン目から本番タスク。暖気なし。
codex exec --json --sandbox read-only \
  'CONTROL ARM. Agent SkillやSKILL.mdを読まず、通常のコマンドで測定タスクを実行する。' \
  </dev/null \
  > baseline-task01.jsonl

# shell-geinin型：Skillだけを暖気し、同じタスクでresumeする。
codex exec --json --sandbox read-only \
  'shell-geininのSKILL.md全体を暖気として読む。fixtureはまだ調べない。' \
  </dev/null \
  > treatment-warmup.jsonl
THREAD_ID=$(jq -r 'select(.type == "thread.started") | .thread_id' treatment-warmup.jsonl | head -1)
codex exec resume --json "$THREAD_ID" \
  'baselineと同じ測定タスクを実行する。' </dev/null > treatment-task01.jsonl

jq -c 'select(.type == "turn.completed") | .usage' treatment-task01.jsonl
jq -c 'select(.type == "item.completed" and .item.type == "command_execution") | .item' treatment-task01.jsonl
```

これは記事のローカル使用量ダッシュボードに対するCodexネイティブな測定源です。
JSONLを集計し、SQLite、FastAPI、OTLPは必要になった場合だけ可視化層として追加します。

## 📁 リポジトリ構成

```text
skills/shell-geinin/SKILL.md       # Agent Skillsのエントリポイント
skills/shell-geinin/references/    # 詳細なガイダンス
.claude-plugin/plugin.json         # Claude Code用メタデータ
.codex-plugin/plugin.json          # Codex CLI用メタデータ
README.ja.md                       # 日本語ドキュメント
```

`skills/<name>/SKILL.md` という標準配置によりSkills CLIから発見できます。

プラグイン用のメタデータも同梱しているため、Claude CodeとCodex CLIのネイティブなワークフローにも対応します。

## 🛠️ 開発

```sh
git clone https://github.com/redpeacock78/shell-geinin.git
cd shell-geinin
npx skills add . --list
git diff --check
```

pushとプルリクエストでは、`.github/workflows/ci.yml` の軽量なパッケージ検証が実行されます。
JSONマニフェスト、Skillの配置とfrontmatter、文書で参照するファイル、空白エラーを確認します。

マニフェストを変更した場合は、次のコマンドでも確認できます。

```sh
claude plugin validate . --strict
jq empty .claude-plugin/plugin.json .codex-plugin/plugin.json
```

## 📚 参考資料

- [Vercel Labs Skills CLI](https://github.com/vercel-labs/skills)
- [OpenAI Codex CLI developer commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli)
- [Comamocaのリポジトリ作成と公開の手順](https://scrapbox.io/comamoca/%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%AE%E4%BD%9C%E6%88%90%E3%83%BB%E5%85%AC%E9%96%8B%E6%96%B9%E6%B3%95)
- [Comamoca/baserepo](https://github.com/Comamoca/baserepo)

## 📜 ライセンス

[MIT](LICENSE)
