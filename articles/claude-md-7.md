---
title: "CLAUDE.md の書き方とプロジェクト規模別の設計パターン7選"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "ai"]
published: true
---

# CLAUDE.md の書き方とプロジェクト規模別の設計パターン7選

**この記事の要点**
- CLAUDE.md はClaude Codeが自動的に読み込むコンテキストファイルで、プロジェクト固有のルールをAIに継続的に伝える仕組みです
- 個人スクリプトからエンタープライズまで、規模によって最適な構成は大きく異なります
- グローバル（`~/.claude/CLAUDE.md`）とローカル（プロジェクトルートの`CLAUDE.md`）を組み合わせることで重複なく管理できます

---

CLAUDE.md の書き方を検索しているエンジニアの多くが直面するのは「どこまで書けばいいのか」という問題です。チームに「とりあえずコーディング規約を書いておいて」と言われてファイルを作ったものの、数百行に膨れ上がって誰も読まなくなった──そういう経験をした方は少なくないはずです。

この記事では、個人ツールからエンタープライズまで、プロジェクト規模に応じた7つの設計パターンを具体的なテンプレートとともに紹介します。

---

## CLAUDE.md とは何か

CLAUDE.md は [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) が自動的にコンテキストとして読み込むMarkdownファイルです。プロジェクトルートまたは `.claude/` ディレクトリに置くことで、AIセッションを開始するたびに内容が適用されます。

配置場所の優先順位は次のとおりです。

| パス | スコープ |
|---|---|
| `~/.claude/CLAUDE.md` | 全プロジェクト共通（グローバル） |
| `{project}/.claude/CLAUDE.md` | プロジェクト全体 |
| `{project}/src/CLAUDE.md` | `src/` 以下のサブディレクトリ |

複数ファイルが存在する場合はすべてマージされて読み込まれます。また `@path/to/file` 記法で別ファイルを参照インポートできるため、大規模プロジェクトでは分割管理が有効です（[Claude Code ドキュメント](https://docs.anthropic.com/en/docs/claude-code/memory)参照）。

---

## 規模別パターンの全体像

まず選択の目安を整理します。

| パターン | 想定規模 | CLAUDE.md の行数目安 |
|---|---|---|
| 1. ミニマル | 個人スクリプト | 〜30行 |
| 2. 個人OSS | ソロ開発プロダクト | 50〜80行 |
| 3. 小チーム | 3〜8人 | 80〜150行 |
| 4. 機能チーム | 10〜30人・モノリス | 150〜250行 |
| 5. モノレポ | 複数パッケージ | 親30行＋子50行×N |
| 6. マイクロサービス | 複数リポジトリ | 共通テンプレ＋差分30行 |
| 7. エンタープライズ | コンプライアンス必須 | 200〜400行 |

---

## パターン1: ミニマル（個人スクリプト・CLIツール）

検証用スクリプトや小さなCLIツールでは、CLAUDE.md が重いと逆効果です。必要最小限に絞ります。

```markdown
# Project

個人用データ変換CLIツール。

## Stack
- Python 3.12 / Typer / Rich

## Rules
- 型アノテーション必須
- テストは pytest。モックは最小限にする
- コメントは書かない（自明なコードを書く）
```

**ポイント:** スタックと「やってはいけないこと」だけ書く。コーディング規約はグローバルCLAUDE.mdに委ねる。

---

## パターン2: 個人OSS（ドキュメント重視）

公開リポジトリでは、AIにコントリビューションガイドやREADMEの文体を理解させることが重要です。

```markdown
# my-oss-lib

## Overview
TypeScript製の軽量バリデーションライブラリ。

## Development
- `pnpm test` でテスト実行
- `pnpm build` でdistを生成

## Conventions
- Public APIには JSDoc を必ず書く
- Breaking changeは CHANGELOG.md のUnreleased欄に追記する
- コミットはConventional Commits形式

## Tone (ドキュメント)
- READMEと JSDoc は英語
- issueテンプレへの返信は日本語OK
```

**ポイント:** ドキュメントの言語ルールをここに書いておくと、AIが誤った言語でドキュメントを生成するミスを防げます。

---

## パターン3: 小チーム（3〜8人）

チームが増えると「暗黙知」がAIに伝わらなくなります。決定した設計方針をここに書き留める運用が効きます。

```markdown
# backend-api

## Architecture
- Clean Architecture。usecase/domain/infra の3層
- infra 層から domain 層への依存を禁止する
- DB は PostgreSQL 16。ORMは使わず sqlc を使う

## Testing
- ユニットテストはdomain層のみ対象
- 統合テストは実DBを使う（モック禁止）
  - 理由: モックとの乖離でリリース後にバグが出た経験があるため

## PR Rules
- レビュワーは最低1名
- セキュリティ影響があるPRはチームリードをassign

## Do NOT
- ログに個人情報を出力しない
- エラーハンドリングでpanicを使わない（Go）
```

実際にこのパターンで運用してみて気づいたのですが、「なぜそのルールがあるか」を一言添えると、AIが例外ケースを判断する精度が上がります。単なるルール列挙より、背景を書くほうが効果的です。

---

## パターン4: 機能チーム（10〜30人・モノリス）

複数のチームがひとつのリポジトリを触る場合は、セクションを明確に区切り、自分のチームに関係するものだけ読めばいい構成にします。

```markdown
# monolith-app

## Teams
- Platform: infra/, db/
- Commerce: src/commerce/
- User: src/user/

## Global Rules
@.claude/rules/global.md

## Commerce Team
@.claude/rules/commerce.md

## User Team
@.claude/rules/user.md
```

`.claude/rules/` ディレクトリ以下に分割して `@` インポートで組み立てるのがこの規模のベストプラクティスです。ファイルを分けておくと、チームごとに独立してルールを更新できます。

---

## パターン5: モノレポ（複数パッケージ）

モノレポでは親CLAUDE.mdはグルーコード的な役割にとどめ、実質的なルールはパッケージ側のCLAUDE.mdに委ねます。

```
/CLAUDE.md          ← リポジトリ全体の方針（30行程度）
/apps/web/CLAUDE.md ← Next.js固有のルール
/apps/api/CLAUDE.md ← Hono/Cloudflare Workers固有のルール
/packages/ui/CLAUDE.md ← UIコンポーネントライブラリのルール
```

親の`CLAUDE.md`に書く内容:
```markdown
# monorepo

## Structure
- pnpm workspaces
- Turborepo でビルドオーケストレーション

## Shared Rules
- コミットはConventional Commits
- PRタイトルにスコープを含める（例: `feat(web): ...`）

## Per-package rules
各パッケージの CLAUDE.md を参照すること。
```

---

## パターン6: マイクロサービス（複数リポジトリ）

リポジトリが分かれている場合、共通ルールのコピーが増殖しがちです。これを防ぐには「社内テンプレートリポジトリ」にマスターCLAUDE.mdを置き、各サービスのセットアップスクリプトで配布する方法が有効です。

```bash
# 各リポジトリの初期化スクリプトで実行
curl -s https://raw.githubusercontent.com/your-org/templates/main/CLAUDE.md \
  > .claude/base.md
```

各サービスの`CLAUDE.md`:
```markdown
# payment-service

@.claude/base.md

## Service-specific
- PCI DSS スコープ内。カード番号をログに出力しない
- 外部API呼び出しには必ずタイムアウトを設定する（30秒）
```

**ポイント:** 共通ルールへの差分だけ書く。ベースファイルの内容を各リポジトリに重複させない。

---

## パターン7: エンタープライズ（コンプライアンス・セキュリティ重視）

金融・医療・官公庁向けシステムでは、AIが絶対に踏み越えてはいけない禁止事項を冒頭に集約する構成が安全です。

```markdown
# enterprise-system

## !! BLOCKING REQUIREMENTS !!
以下は絶対に実行しない:
- 本番DBへの直接接続・操作
- シークレット・APIキーをコードに記述
- 外部ネットワークへのデータ送信
- 監査ログの削除・改ざん

## Compliance
- 個人情報はマスキングしてからログに出力する
- 脆弱性対応はCVSS 7.0以上を優先（社内セキュリティポリシー準拠）

## Approval Required
以下の操作はチームリードの承認を得てから実行:
- DBマイグレーション
- 認証・認可ロジックの変更
- 外部連携APIの追加

@.claude/rules/architecture.md
@.claude/rules/testing.md
@.claude/rules/security.md
```

冒頭の `!! BLOCKING REQUIREMENTS !!` セクションは、AIがどのタスク指示を受けても最初にコンテキストとして読むため、禁止事項を確実に認識させる効果があります。

---

## グローバル CLAUDE.md との役割分担

`~/.claude/CLAUDE.md`（グローバル）とプロジェクトの CLAUDE.md の使い分けは、次の基準が実用的です。

| グローバルに書く | プロジェクトに書く |
|---|---|
| コーディングスタイルの個人的好み | プロジェクト固有のアーキテクチャ |
| 絵文字・コメントの書き方方針 | スタック・バージョン情報 |
| 言語設定（日本語で回答する等） | テストの方針 |
| 破壊的操作前の確認ルール | デプロイ・リリース手順 |

---

## よくある質問

**CLAUDE.md が長くなりすぎて管理できない。どうすればいいですか？**

`@` インポート構文でファイルを分割するのが有効です。トピックごとに `.claude/rules/testing.md`、`.claude/rules/security.md` のように分けてメインの CLAUDE.md からインポートすると、変更が局所化されてレビューしやすくなります。また行数よりも「AIが迷ったときに参照すべき情報」に絞って書くことが重要で、コードから読み取れる情報（ファイル構成、依存関係）は書かないほうが記述量を抑えられます。

**チームメンバーが CLAUDE.md を更新しない。どう運用すればいいですか？**

「決定を下したらすぐに書く」習慣をチームに根づかせるのが本質的な解決策ですが、それが難しければ PR テンプレートに「CLAUDE.md の更新が必要か？」のチェックボックスを追加する方法が現実的です。アーキテクチャ上の意思決定（ADR）を CLAUDE.md に書く運用にすると、ドキュメントとAIコンテキストを一元管理できます。

**CLAUDE.md に書いた内容は Claude Code 以外のAIツールでも使えますか？**

現状では CLAUDE.md は Claude Code 固有の仕組みです。ただし Markdown 形式で書かれているため、GitHub Copilot の `.github/copilot-instructions.md` や Cursor の `.cursorrules` に内容を流用することは可能です。複数ツールを使うチームは、マスターとなる `docs/ai-context.md` を用意して各ツール向けファイルにコピーする運用も検討できます（[GitHub Copilot カスタムインストラクション](https://docs.github.com/en/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)参照）。
