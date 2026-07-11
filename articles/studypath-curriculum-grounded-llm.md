---
title: "AIに勝手な参考書を勧めさせない — カリキュラム準拠でLLM出力を縛った受験学習アプリの設計"
emoji: "📚"
type: "tech"
topics: ["Nextjs", "OpenAI", "個人開発", "TypeScript", "AI"]
published: true
---

## はじめに / 何を作ったか

「志望校と今の状況を入れると、本番までの長期プランと"今日やること"まで出してくれる」受験戦略AI **StudyPath AI** を個人開発しています。

- 技術スタック: Next.js 15 / React 19 / Firebase / OpenAI SDK / Capacitor（iOS・Android配布）
- 想定読者: 個人開発でLLMプロダクトを作る人 / EdTechに関心がある人
- デモ/URL: （公開URLは追記予定）

この記事は「作った自慢」ではなく、**LLMの出力を"信頼できる形"に縛るための設計判断**を主役にします。生成AIプロダクトで誰もがぶつかる「AIが適当なことを言う」問題への、具体的な対処です。

## 課題と背景

家庭教師をしていて、「今日何をやればいいか」で手が止まる"受け身"の生徒を多く見てきました。自走できる子向けのサービスは多いのに、その層を真正面から狙うものが少ない。そこを埋めたくて作りました。

ただ、受験の学習プランをLLMに任せると致命的な問題が出ます。**AIが実在しない参考書や、塾の方針と噛み合わない本を平気で勧める**のです。受験は「何を使うか」の信頼性が命なので、ここを解けないとプロダクトになりません。

設計の主眼は次の3つに絞りました。

1. LLM出力をカリキュラム（定番参考書リスト）に準拠させる
2. 出力構造を JSON Schema で契約し、UIが壊れないようにする
3. 完了履歴を渡して「同じ課題を再生成しない」を守らせる

以下、それぞれの実装を最小コードで示します。

## 設計判断1: LLM出力をカリキュラムに準拠させる

`curriculum`（塾の参考書DB準拠）と `ai_auto`（AIおまかせ）の2モードを持ち、`profile.planMode` で切り替えます。準拠モードでは、`curriculum.json` 由来の教材だけをプロンプトに入れ、「この教材・ステップのみ使え」と縛ります。

```ts
// api/generate/route.ts
const planMode: 'ai_auto' | 'curriculum' =
  profile?.planMode === 'ai_auto' ? 'ai_auto' : 'curriculum';

let curriculumBlock = '';
if (planMode === 'curriculum') {
  const curriculumContext = buildCurriculumPrompt(subjects);
  const exclusiveGroups = getExclusiveGroups(subjects);
  curriculumBlock = `## カリキュラムデータベース (この教材・ステップのみ使用してください)
${curriculumContext}

## 排他関係 (どちらか1冊のみ採用すること - 重要!)
${exclusiveGroups.length > 0 ? exclusiveGroups.join('\n') : '(なし)'}

⚠️ 全てのTodoには materialId/stepIndex を必ず指定してください。`;
}
```

ポイントは、プロンプトを組み立てる側で**「採用しない教材はそもそもプロンプトに入れない」**こと。同じ用途で複数の定番書（排他グループ）がある場合、AIに選ばせず、採用1冊だけを渡して他は除外します。「両方使ってしまう事故」をプロンプト設計で防ぐわけです。

```ts
// data/curriculum.ts
export function buildCurriculumPrompt(subjects: string[]): string {
  const lines: string[] = [];
  const exclusiveGroups = computeExclusiveGroups(subjects);
  const droppedIds = new Set<string>();
  for (const g of exclusiveGroups) {
    for (const m of g.materials) {
      if (m.id !== g.selected.id) droppedIds.add(m.id); // 採用されなかった側
    }
  }
  for (const sub of subjects) {
    for (const mat of getMaterialsForSubject(sub)) {
      if (droppedIds.has(mat.id)) continue; // ← プロンプトから除外
      lines.push(`- **[${mat.id}] ${mat.name}** (全${mat.totalSteps}ステップ)`);
    }
  }
  return lines.join('\n');
}
```

「AIに賢く選ばせる」より「AIに渡す選択肢を先に絞る」ほうが、信頼性は圧倒的に上がりました。

## 設計判断2: JSON Schema で出力を契約する

UIは `analysis / phases / todos / encouragement` が必ず揃っている前提で描画します。1つでも欠けると壊れるので、OpenAI の `response_format: { type: 'json_schema', strict: true }` にスキーマを渡し、**必須フィールドをスキーマレベルで保証**します。

```ts
// lib/constants.ts
export const PLAN_JSON_SCHEMA = {
  type: 'object' as const,
  properties: {
    analysis: { type: 'string' as const, description: '現状分析と志望校とのギャップ (3-5文)' },
    phases: { /* name, startDate, endDate, ... を required */ },
    todos: {
      type: 'array' as const,
      items: {
        type: 'object' as const,
        properties: {
          material:   { type: 'string' as const, description: 'カリキュラムの教材名' },
          materialId: { type: 'string' as const, description: '教材ID (例: 英語_01)' },
          stepIndex:  { type: 'number' as const, description: 'ステップ番号 (0始まり)' },
          // ... subject, task, estimatedMinutes, dueDate, phase, order
        },
        required: ['subject', 'task', 'material', 'materialId', 'stepIndex',
                   'estimatedMinutes', 'dueDate', 'phase', 'order'],
        additionalProperties: false,
      },
    },
    encouragement: { type: 'string' as const, description: '穏やかで前向きな励まし (2-3文)' },
  },
  required: ['analysis', 'phases', 'todos', 'encouragement', 'nextExamTarget', 'finalTarget'],
  additionalProperties: false,
};
```

`additionalProperties: false` と `required` を全階層で効かせると、パースの try/catch とバリデーションコードがごっそり消えます。「AIの出力を信じてUIを書く」のではなく「契約を満たさない出力はそもそも返らせない」に倒すのが安定への近道でした。

## 設計判断3: 完了履歴を渡して"同じ課題を再生成しない"

学習は継続なので、毎回まっさらなプランを出されると困ります。**直近200件の完了タスクを渡し、「続きから」生成させます**。まずクライアントで履歴を組み立て:

```tsx
// page.tsx
const completedHistory = todos
  .filter(t => t.status === 'done')
  .slice(-200) // 直近200件 (プロンプト肥大化を抑制)
  .map(t => ({
    subject: t.subject, material: t.material, materialId: t.materialId,
    unit: t.unit, stepIndex: t.stepIndex, task: t.task, completedAt: t.completedAt,
  }));
```

サーバー側で教材ごとに畳んで、**絶対遵守ルールとしてプロンプトに展開**します。

```ts
// api/generate/route.ts
return `## 🔴 既に完了済みのタスク・教材 (新プランに含めない、続きから生成すること)
${lines.join('\n')}

### 完了履歴の扱い (絶対遵守)
1. 同じ materialId + stepIndex の組合せは新プランに含めないこと
2. 完了済の教材は「続きの stepIndex」から再開する
3. ある教材を最後まで完了していたら、応用編 or 別教材の次フェーズへ進める`;
```

設計判断1で `materialId` / `stepIndex` を必須化しておいたおかげで、この「重複排除ルール」がキー比較で機械的に効きます。3つの判断が噛み合う形です。

## 結果 / 学び

- 本番デプロイ済み。利用状況の数値は集計でき次第このセクションに追記します（現時点では実数のみ載せる方針のため未記載）。
- 一番の学び: **LLMの信頼性は「賢いプロンプト」より「渡す選択肢と出力契約を先に絞る設計」で作られる**。カリキュラム準拠・JSON Schema・完了履歴の3つは、いずれも「AIの自由度を意図的に下げる」方向の設計でした。
- もう一つ: 当初入れていたRPG風のゲーミフィケーション（XP・ストリーク演出）は、受け身層にはむしろプレッシャーになると判断して撤去し、"穏やかな継続装置"へ振り直しました。ターゲットによって「続く仕掛け」の形は変わります。

## まとめ

- **競合空白**（受け身層）を狙い、**カリキュラム準拠**でLLM出力の信頼性を担保し、**JSON Schema + 完了履歴**で壊れず継続する形にした
- 生成AIプロダクトの肝は「AIを賢くする」より「AIに渡す入力と、返させる出力の形を設計で縛る」こと

StudyPath AI: （公開URLは追記予定）

<!--
◀ 公開前に要確認（推定で埋めない）:
  1. 冒頭と末尾の「公開URL」→ 本番ドメインに差し替え
  2. 「結果」セクションの数値 → 実数があれば追記、無ければこのまま
-->
