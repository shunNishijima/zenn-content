# zenn-content

Zenn 連携用リポジトリ。`articles/*.md` を push すると Zenn が自動で反映する
（`published: true` の記事だけが公開される。`false` はドラフト扱い）。

- 記事の生成・配置は `~/sunLife/sunlife/tools/tech-article-publisher/` の CLI が行う
  （`publish --to zenn`）。
- ローカルプレビュー: `npm i` 後 `npm run preview`。
- 公開は `published: true` にして push（＝人間の手動操作）。

連携設定: Zenn の GitHubデプロイ設定でこのリポジトリを接続する。
