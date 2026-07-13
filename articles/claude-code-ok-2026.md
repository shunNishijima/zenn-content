---
title: "Claude Code 使い方まとめ【2026年版】設定から実践Tips まで"
emoji: "🤖"
type: "tech"
topics: ["ai", "claude"]
published: true
---

# Claude Code 使い方まとめ【2026年版】設定から実践Tips まで

**この記事の要点**
- Claude CodeはAnthropic製のCLIベースAIコーディングエージェントで、ファイル操作・シェル実行・Web検索をモデルが自律的に完結させる
- CLAUDE.mdによる指示の永続化、Hookシステムによる自動化、MCPサーバー連携が実務での主な差別化ポイント
- 料金はMaxサブスクリプションかAPI従量課金で使用可能。コスト上限の事前設定が必須

## Claude Codeとは何か

Claude Codeは、Anthropicが開発するAIコーディングエージェントです。ターミナル上で自然言語の指示を出すと、モデルが自律的にファイルを読み書きし、Bashコマンドを実行してタスクを完結させます。2025年2月のベータ公開を経て同年5月に正式リリース（GA）され、2026年現在はClaude Sonnet 4.6およびOpus 4.8を利用できます。

VS Code・JetBrainsのIDE拡張としても使えますが、本来の強みはCLIからの自律的なタスク実行にあります。「このバグを直して」と指示すると、関連ファイルの調査・修正・テスト実行まで一気に進める動作は、補完型ツールとは一線を画します。

### 従来のAIコーディングツールとの違い

よく比較されるCursorやGitHub Copilotとの差を表にまとめます。

| ツール | 主な形態 | ファイル操作 | シェル実行 | バックグラウンド実行 |
|---|---|---|---|---|
| GitHub Copilot | IDE補完 / Chat | 限定的 | 不可 | 不可 |
| Cursor | IDEエディタ | あり | 限定的 | 不可 |
| Claude Code | CLI / IDE拡張 | あり | あり | あり |

最大の差は「バックグラウンド実行」です。Claude Codeはサブエージェントを別プロセスで動かしながら、手元では別のタスクを進めるといった並列作業が可能です。

## インストールと初期設定

### npmでインストール

Node.js 18以上が前提です。

```bash
npm install -g @anthropic-ai/claude-code
```

インストール後は `claude` コマンドで起動します。

```bash
claude
```

### 認証方法は3種類

1. **Anthropicアカウントでログイン**（ProまたはMaxプラン）: 起動後にブラウザ認証が走る
2. **APIキーを環境変数にセット**: `export ANTHROPIC_API_KEY="sk-ant-..."` を `.zshrc` などに追記
3. **Claude for Work（Enterprise）**: 組織のIT管理者がAWSやGCP経由で設定

私自身はAPIキー方式で使っています。複数プロジェクトをまたいで使いまわせるので管理が楽です。ただし共有マシンにキーをそのまま書くのは危険なので、`direnv` などで環境ごとに切り替えることをおすすめします。

## 主要なスラッシュコマンド一覧

対話モード中に `/` で始まるコマンドでClaude Code自体を制御できます。

| コマンド | 説明 |
|---|---|
| `/help` | コマンド一覧を表示 |
| `/clear` | 会話履歴をリセット |
| `/compact` | 長い会話を要約してコンテキストを節約 |
| `/model` | 使用モデルを切り替え |
| `/cost` | セッション中のトークン消費量を確認 |
| `/init` | プロジェクトにCLAUDE.mdを自動生成 |
| `/review` | ブランチの差分をコードレビューさせる |

長時間の作業セッションでは `/compact` を定期的に実行する習慣をつけると、コンテキストウィンドウの圧迫を防げます。

## CLAUDE.mdでプロジェクトの動作を制御する

Claude Codeを実務で使う上で最も重要な機能が **CLAUDE.md** です。プロジェクトルートに置くMarkdownファイルで、Claude Codeが起動するたびに読み込む「指示書」として機能します。

```markdown
# CLAUDE.md 例

## プロジェクト概要
Next.js 15 + TypeScriptのECサイト。

## 開発ルール
- コメントは日本語
- テストはvitest。`npm run test` で実行
- `npm run build` が通らない変更はコミットしない

## 禁止事項
- APIキーやシークレットをコードに書かない
- `console.log` を本番コードに残さない
```

`/init` を実行すると既存のコードベースを解析してCLAUDE.mdの初稿を自動生成してくれます。ゼロから書くより格段に速いので、新しいプロジェクトで使い始める際は必ず試す価値があります。

グローバル設定は `~/.claude/CLAUDE.md` に書きます。「絵文字は使わないで」「応答は日本語で」といった個人の好みはここに一度書けば全プロジェクトに適用されます。

## Hookシステムで繰り返し作業を自動化する

Claudeがツールを呼ぶ前後に任意のシェルコマンドを差し込める **Hookシステム** が `.claude/settings.json` から設定できます。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint -- --fix"
          }
        ]
      }
    ]
  }
}
```

上記の設定では、Claude Codeがファイルを編集するたびにlintが自動で走ります。「書き換えたら必ずフォーマットしてほしい」という要求を都度指示しなくて済むようになります。

対応しているイベントは `PreToolUse`（ツール実行前）・`PostToolUse`（ツール実行後）・`Stop`（Claude停止時）などです。

## MCPサーバー連携でできることを拡張する

MCP（Model Context Protocol）はClaudeが外部ツールと通信するためのプロトコルです。MCPサーバーを追加すると、標準機能にない操作をClaude Codeに任せられます。

設定例（`.claude/settings.json`）:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

代表的なMCPサーバーの例:

- **GitHub MCP**: Issue作成・PRレビュー・ブランチ操作
- **PostgreSQL MCP**: SQLクエリの直接実行
- **Playwright MCP**: ブラウザ操作の自動化
- **Slack MCP**: チャンネルへの通知送信

MCPのエコシステムは2026年現在も拡大が続いており、主要SaaSの多くにMCPサーバーが存在します。「Claude Codeを自社の社内ツールと連携させたい」という用途にも使えます。

## コスト管理の考え方

### 料金体系

Claude Codeは以下のいずれかで使用できます。正確な最新料金は [Anthropic公式の料金ページ](https://www.anthropic.com/pricing) を確認してください。

- **Claude Maxプラン**: サブスクリプション型。月単位の利用上限が設定されているため費用が予測しやすい
- **APIキー経由の従量課金**: トークン数に応じた課金。使用量が多いとMaxプランより高くなるケースがある

### コスト上限は必ず設定する

APIキーで使う場合、Anthropicダッシュボードから月間の上限額（spending limit）を設定できます。ループ処理が意図せず走り続けるなどのミスで費用が膨らむリスクがあるため、**最初にかならず上限を設定することを強くおすすめします**。

セッション中は `/cost` で消費量を随時確認できます。

## よくあるつまずきポイント

### 許可ダイアログが頻繁に出る

デフォルトでは操作ごとに確認が求められます。信頼できる操作は `.claude/settings.json` の `permissions.allow` でホワイトリスト化できます。

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git status)",
      "Bash(git diff*)"
    ]
  }
}
```

### CLAUDE.mdが読み込まれていない気がする

サブディレクトリ内で `claude` を起動すると、親ディレクトリのCLAUDE.mdが読み込まれないことがあります。プロジェクトルートで起動するか、`--cwd` オプションでルートを明示しましょう。

```bash
claude --cwd /path/to/project
```

### コンテキストが長くなって応答が遅くなる

長時間セッションでは `/compact` でコンテキストを圧縮するか、タスク切り替え時に `/clear` でリセットします。新しいタスクに移るたびにリセットする方が、モデルの精度も安定しやすい傾向があります。

## よくある質問

**Q. Claude CodeとCursorはどちらを使えばよいですか？**

A. 用途によって異なります。CLIから自律的にタスクを完結させたい、シェル操作やCI/CDを組み合わせたいならClaude Codeが向いています。エディタ内での補完やUI上での差分確認を重視するならCursorが扱いやすいです。実際には両方を使い分けているエンジニアが多く、排他的に選ぶ必要はありません。

**Q. Claude Codeが生成したコードの著作権はどうなりますか？**

A. Anthropicの[利用規約](https://www.anthropic.com/legal/aup)では、ユーザーが生成したアウトプットの権利を制限しない旨が明記されています。ただし法的な判断は状況によって異なるため、企業での本格利用時は法務部門への確認を推奨します。

**Q. CLAUDE.mdはチームで共有できますか？**

A. `.claude/settings.json` とプロジェクトの `CLAUDE.md` はGitリポジトリに含めてチーム全員で共有できます。APIキーは個人ごとに発行するのが基本です。組織での管理にはClaude for Work（Enterprise）の利用を検討してください。
