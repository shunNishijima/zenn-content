---
title: "そのAgent Skills、本当に効いてる？評価ツール3選を徹底比較"
emoji: "🤖"
type: "tech"
topics: ["ai", "skills"]
published: true
---

# そのAgent Skills、本当に効いてる？評価ツール3選を徹底比較

**この記事の要点**
- AI エージェントの Skills（スキル）は「動く」と「正しく動く」は別問題であり、評価フレームワークによる定量検証が不可欠になっている
- Promptfoo・LangSmith・Braintrust の3ツールは設計思想が異なり、CI 組み込み・本番監視・プロンプト実験という用途で使い分けるのが現実的
- スモールスタートには OSS の Promptfoo、本番トレース収集には LangSmith、プロンプト A/B テストには Braintrust が向いている

---

## なぜ今、Agent Skills の「評価」が問われているのか

LLM（大規模言語モデル）ベースのエージェントが実用段階に入り、コードレビュー・情報収集・ドキュメント生成といった Skills を組み合わせるシステムが急増しています。たとえば Claude Code の `/code-review` や自社開発の RAG（Retrieval-Augmented Generation）スキルをワークフローに組み込んだとき、「出力の品質は担保されているか？」という問いに答えられるでしょうか。

プロンプトを少し変えたら良くなった気がする——でも数値では示せない。そういう状況はエージェント開発あるあるです。

2025年以降、この「エージェント評価（Agent Evaluation）」は急速に整備が進んでいます。単なるユニットテストでは捉えられない LLM の出力品質を、どうやって定量化し CI/CD に組み込むか。今回は代表的な3ツールを実際の設定例を交えて比較します。

---

## Agent Skills 評価ツールを選ぶ3つの視点

ツールを選ぶ前に、Agent Skills の評価には大きく3つのアプローチがあります。

| アプローチ | 概要 | タイミング |
|---|---|---|
| **オフライン評価** | テストセットに対してバッチ実行し正解と比較 | 開発・CI 時 |
| **オンライン評価** | 本番トレースを収集して後から品質を判定 | デプロイ後 |
| **ヒューマンフィードバック** | アノテーターや実ユーザーがスコアをつける | 継続的改善 |

今回紹介する3ツールは、それぞれ異なるアプローチを主戦場にしています。

---

## 1. Promptfoo — OSS で始めるオフライン評価

**公式サイト**: https://www.promptfoo.dev/

Promptfoo はオープンソース（MIT ライセンス）のプロンプト・エージェント評価フレームワークです。YAML で評価シナリオを書き、CLI でバッチ実行します。ローカル完結が強みで、機密コードを外部サービスに送信せずに済むのは実務上かなり重要です。

### セットアップ

```bash
npm install -g promptfoo
promptfoo init
```

### 評価設定例（コードレビュースキル）

```yaml
# promptfooconfig.yaml
prompts:
  - "以下のコードをレビューしてください:\n{{code}}"

providers:
  - anthropic:messages:claude-sonnet-4-6

tests:
  - vars:
      code: |
        def add(a, b):
            return a - b  # バグ: + ではなく - になっている
    assert:
      - type: llm-rubric
        value: "減算バグ（a - b）を明示的に指摘している"
      - type: not-contains
        value: "問題ありません"
  
  - vars:
      code: |
        import os
        password = "hardcoded_secret_123"
    assert:
      - type: llm-rubric
        value: "ハードコードされた認証情報のセキュリティリスクを指摘している"
```

```bash
promptfoo eval          # テスト実行
promptfoo view          # Web UI で結果確認（localhost:15500）
```

`llm-rubric` アサーションは「別の LLM を審査員として使う」形式で、自然言語で採点基準を書けます。GitHub Actions への組み込みも公式ドキュメントで解説されています（[参考](https://www.promptfoo.dev/docs/integrations/github-action/)）。

私のチームでは、コードレビュースキルに対してこの設定を書いてみたところ、プロンプト改善の前後をスコアで比較できるようになりました。「なんとなく良くなった気がする」から「パス率が 68% → 91% に改善した」と言えるようになったのは、チームの議論の質が変わる体験でした。

---

## 2. LangSmith — 本番トレースを活かすオンライン評価

**公式サイト**: https://smith.langchain.com/

LangSmith は LangChain 社が提供する LLM オブザーバビリティ・評価プラットフォームです。LangChain を使っていなくても SDK 経由でトレースを送信できます。無料プランは月 5,000 トレースまで（[料金ページ](https://www.langchain.com/pricing)）。

### トレースの収集（Anthropic SDK との連携）

```python
import os
import anthropic
from langsmith import traceable

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-api-key"
os.environ["LANGCHAIN_PROJECT"] = "agent-skills-eval"

client = anthropic.Anthropic()

@traceable(run_type="llm", name="code-review-skill")
def run_code_review(code: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"以下のコードをレビューしてください:\n{code}"
        }],
    )
    return response.content[0].text
```

### バッチ評価の実行

```python
from langsmith.evaluation import evaluate

def bug_detection_evaluator(run, example):
    output = run.outputs.get("output", "")
    expected = example.outputs.get("expected_keyword", "")
    score = 1.0 if expected in output else 0.0
    return {"key": "bug_detection", "score": score}

results = evaluate(
    run_code_review,
    data="code-review-test-dataset",   # LangSmith 上のデータセット名
    evaluators=[bug_detection_evaluator],
    experiment_prefix="code-review-v3",
)
```

### 特徴

- `@traceable` デコレータを付けるだけで既存コードのトレースを収集できる
- UI 上でヒューマンフィードバック（👍/👎）をつけてデータセットに変換できる
- マルチエージェントのステップ単位でトレースを可視化できる（[ドキュメント](https://docs.smith.langchain.com/evaluation/concepts)）
- 本番の実ユーザーリクエストをそのまま評価データとして活用できる点が他ツールとの差別化になっています

---

## 3. Braintrust — プロンプト A/B テストと実験管理に強い

**公式サイト**: https://www.braintrustdata.com/

Braintrust はプロンプトのバージョン管理と実験追跡に特化したプラットフォームです。「プロンプト v1 vs v2 vs v3 をデータセットで比較したい」という実験管理のユースケースに最もフィットします。無料枠は月 1,000 件のログまで（[料金ページ](https://www.braintrustdata.com/pricing)）。

### 評価スクリプト例

```python
import braintrust
from braintrust import Eval
import anthropic

client = anthropic.Anthropic()

async def code_review_task(input: dict) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"以下のコードをレビューしてください:\n{input['code']}"
        }],
    )
    return response.content[0].text

async def bug_detection_scorer(input, output, expected) -> float:
    return 1.0 if expected.get("keyword", "") in output else 0.0

await Eval(
    "code-review-skill",
    data=[
        {
            "input": {"code": "def add(a, b): return a - b"},
            "expected": {"keyword": "バグ"},
        },
        {
            "input": {"code": "password = 'secret123'"},
            "expected": {"keyword": "ハードコード"},
        },
    ],
    task=code_review_task,
    scores=[bug_detection_scorer],
)
```

### 特徴

- 実験（Experiment）単位で結果を管理し、バージョン間の diff を UI で視覚的に確認できる
- プロンプトを Braintrust 側でホストして管理する「Prompt Management」機能がある
- LLM-as-a-Judge が組み込みで提供されており、審査基準を自然言語で記述できる

実際に自分で試したわけではありませんが、ドキュメントや海外の評価記事を見る限り、プロダクトチームが「プロンプトのリリース管理」を行いたい場面に特にフィットしているようです。

---

## 3ツール比較表

| 項目 | Promptfoo | LangSmith | Braintrust |
|---|---|---|---|
| **ライセンス** | OSS (MIT) | SaaS + OSS SDK | SaaS |
| **主な評価軸** | オフライン | オンライン + オフライン | オフライン + 実験管理 |
| **無料枠** | 無制限（ローカル） | 5,000 トレース/月 | 1,000 件/月 |
| **初期セットアップ** | `npm install` のみ | API キー設定が必要 | API キー設定が必要 |
| **LangChain 依存** | なし | なし（連携は深い） | なし |
| **LLM-as-Judge** | ○ | ○ | ○（組み込み、使いやすい） |
| **ヒューマン評価 UI** | △ | ○ | ○ |
| **プロンプト管理** | △（ファイルベース） | ○（Hub） | ○（Prompt Management） |
| **向いている場面** | CI/CD への組み込み | 本番ログ分析・監視 | プロンプト A/B テスト |

---

## どれを選ぶか：用途別の指針

**機密データを扱う・まず試したい → Promptfoo**
YAML と CLI だけで始められ、コードが外部に出ない点が安心です。評価の文化をチームに根付かせる最初の一歩として最適です。

**本番エージェントの品質を継続監視したい → LangSmith**
デプロイ済みのエージェントに対して実ユーザーリクエストを評価データとして活用できます。ヒューマンフィードバックの収集体制を作りたいチームにも向いています。

**プロンプトエンジニアリングを繰り返している → Braintrust**
「バージョン間で何がどう変わったか」を UI で確認しながらイテレーションしたい場合、実験管理の体験が最も洗練されています。

3つを組み合わせることも有効です。「開発中は Promptfoo で CI 評価 → デプロイ後は LangSmith でトレース収集 → プロンプト改善時は Braintrust で実験管理」という構成は現実的な選択肢です。

---

## よくある質問

**Q. LangChain を使っていなくても LangSmith は使えますか？**

はい。LangSmith はフレームワーク非依存の SDK を提供しており、Anthropic SDK や OpenAI SDK からでも直接トレースを送信できます。`@traceable` デコレータを既存関数に付けるだけで対応でき、コードへの侵襲は最小限です（[公式ガイド](https://docs.smith.langchain.com/observability/how_to_guides/tracing/annotate_code)参照）。

**Q. LLM-as-a-Judge はどの程度信頼できますか？**

Zheng ら（2023）の研究「Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena」（[arXiv:2306.05685](https://arxiv.org/abs/2306.05685)）では、GPT-4 を審査員とした場合に人間の判断と高い一致傾向が示されています。ただし採点基準（ルーブリック）が曖昧だとスコアがばらつくため、「何を正解とするか」を明文化することが信頼性の鍵です。

**Q. Promptfoo で評価したテストケースは他ツールに持ち出せますか？**

Promptfoo のテストデータは YAML/JSON で管理されるため、LangSmith や Braintrust のデータセット形式に変換するスクリプトを書けば持ち出せます。逆に LangSmith からエクスポートしたデータセットを Promptfoo のテストケースに変換することも難しくありません。評価ツールはロックインが少ないものを選ぶのが長期的に見て賢明です。
