<div align="center">

# shell-geinin

Claude Code と Codex CLI のためのクロスプラットフォーム対応 Agent Skill

[![ライセンス: MIT](https://img.shields.io/github/license/redpeacock78/shell-geinin?style=flat-square)](LICENSE)
[![Skills CLIでインストール](https://img.shields.io/badge/install-npx%20skills-111827?style=flat-square)](https://github.com/vercel-labs/skills)

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

マニフェストを変更した場合は、次のコマンドでも確認できます。

```sh
claude plugin validate . --strict
jq empty .claude-plugin/plugin.json .codex-plugin/plugin.json
```

## 📚 参考資料

- [Vercel Labs Skills CLI](https://github.com/vercel-labs/skills)
- [Comamocaのリポジトリ作成と公開の手順](https://scrapbox.io/comamoca/%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%AE%E4%BD%9C%E6%88%90%E3%83%BB%E5%85%AC%E9%96%8B%E6%96%B9%E6%B3%95)
- [Comamoca/baserepo](https://github.com/Comamoca/baserepo)

## 📜 ライセンス

[MIT](LICENSE)
