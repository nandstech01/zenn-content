---
title: "Claude Code ChannelsでDiscordから記事を自動投稿するシステムを作ってみた"
emoji: "🤖"
type: "tech"
topics: ["Claude", "Discord", "自動化", "API", "TypeScript"]
published: true
---

## 背景：記事投稿の手間を何とかしたかった

複数のプラットフォームに記事を投稿するのって、めちゃくちゃ面倒ですよね。自分の場合、自社ブログに記事を書いた後、ZennとQiitaにもそれぞれ投稿したいんですが、毎回手動でやるのは現実的じゃない。

そこで、最近話題のClaude Code Channelsを使って、Discordから一言指示するだけで自動投稿できるシステムを構築してみました。

## Claude Code Channelsって何？

Claude Code Channelsは、DiscordやSlackからClaude Codeを操作できる機能です。ターミナルを開かなくても、スマホのDiscordアプリから「この記事を投稿して」と送るだけで、AIがコードを書いて実行してくれます。

この記事の詳しい内容は自社ブログに書いています。

## システム設計のポイント

### SEO戦略を意識した設計

単純にコピペするとGoogleにペナルティを受けるので、以下の方針で設計しました：

- 各プラットフォーム向けに内容を大幅リライト（重複率30%以下）
- canonical_urlで元記事を明示
- 個人アカウントから投稿（法人アカウントだとスパム扱いされるリスク）
- 自然なバックリンクを配置

### 実装したシステム構成

最終的にこんな構成になりました：

```typescript
// メインのパイプライン
export async function runCrossPostPipeline(
  slug: string,
  platforms: Platform[]
) {
  const article = await fetchArticle(slug)
  
  for (const platform of platforms) {
    // プラットフォーム別にリライト
    const rewritten = await rewriteForPlatform(article, platform)
    
    // サムネイル生成
    const thumbnail = await generateThumbnail(rewritten, platform)
    
    // 投稿実行
    await publishToPlatform(rewritten, thumbnail, platform)
  }
}
```

## リライト機能の実装

Claude APIを使って、各プラットフォームの特性に合わせてリライトします。

### Zenn向けリライト

```typescript
const zennPrompt = `
あなたはZenn向けの技術記事リライターです。
- 技術記事フォーマット、type: tech
- 実装コード例を重視する
- 見出しに絵文字を使わない
- オリジナル記事とのコンテンツ重複率30%未満を目指す
`
```

### Qiita向けリライト

```typescript
const qiitaPrompt = `
あなたはQiita向けの実践記事リライターです。
- 「やってみた」スタイルの実践的なトーン
- タグを重視したコンテンツ
- 冒頭に「この記事は自社ブログの要約版です」と明記
- 具体的なコード例を多く含む
`
```

## プラットフォーム別投稿機能

### Zenn投稿（GitHub連携）

ZennはGitHubリポジトリと連携しているので、GitHub APIを使って直接ファイルを作成：

```typescript
async function publishToZenn(article: Article) {
  const content = createZennMarkdown(article)
  const filePath = `articles/${article.slug}.md`
  
  await fetch(
    `https://api.github.com/repos/${REPO}/contents/${filePath}`,
    {
      method: 'PUT',
      headers: {
        Authorization: `Bearer ${githubToken}`,
      },
      body: JSON.stringify({
        message: `publish: ${article.title}`,
        content: Buffer.from(content).toString('base64'),
      }),
    }
  )
}
```

### Qiita投稿（API v2）

Qiitaは専用APIがシンプルで使いやすいです：

```typescript
async function publishToQiita(article: Article) {
  const response = await fetch('https://qiita.com/api/v2/items', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${qiitaToken}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      title: article.title,
      body: article.body,
      tags: article.tags.map(name => ({ name })),
      private: false,
    }),
  })
  
  return response.json()
}
```

## サムネイル自動生成

Gemini APIを使って、プラットフォーム別に最適化されたサムネイルを生成：

```typescript
async function generateThumbnail(
  article: Article, 
  platform: Platform
) {
  const specs = {
    zenn: { size: '800x800', style: 'emoji + title card' },
    qiita: { size: '1200x630', style: 'code editor style OGP' },
  }
  
  const prompt = `
プラットフォーム: ${platform}
サイズ: ${specs[platform].size}
スタイル: ${specs[platform].style}
記事タイトル: ${article.title}
`
  
  return await geminiImageGeneration(prompt)
}
```

## Discordからの実行フロー

完成したシステムの使い方は超シンプル：

1. Discordで Claude Code に指示：「記事slug を Zenn Qiita に投稿して」
2. Claude Codeが自動で実行：
   - 記事データを取得
   - プラットフォーム別にリライト
   - サムネイル生成
   - 各プラットフォームに投稿
3. 結果をDiscordに返信

実際の実行ログ：

```
Cross-post pipeline: slug="openai-codex7", platforms=[zenn,qiita]
Article found: "OpenAI Codex導入の落とし穴" (10413 chars)
Rewriter: zenn rewrite complete - 6064 chars
Rewriter: qiita rewrite complete - 23777 chars
[OK] zenn: https://zenn.dev/kenji_harada/articles/openai-codex-7
[OK] qiita: https://qiita.com/kenji_harada/items/47f35fc59da9e6324fcb
```

## 実際の投稿結果

システムを使って投稿した記事：

**元記事**：「OpenAI Codex導入の落とし穴7つと回避策」

**Zenn版**：「OpenAI Codex業務自動化で陥りがちな7つの罠と解決方法」
- 技術記事フォーマットに最適化
- 企業導入の観点から大幅リライト

**Qiita版**：「【実践】OpenAI Codexでワークフロー自動化失敗しないための7つのポイント」
- 実践スタイルでコード例を充実
- 「やってみた」トーンでカジュアルに

どちらも元記事とは大幅に異なる内容になっており、重複率は30%以下を達成しています。

## 開発で感じたClaude Code Channelsの威力

### 良かった点

**Discord起点の開発が想像以上に快適**。移動中でもスマホから開発指示が出せるのは革命的です。

**並列処理による高速実装**。複数のエージェントが同時に動いて、依存関係のない部分を並列実装してくれました。

**既存コードの理解が正確**。僕の既存コードベース（Supabaseクライアント、API呼び出しパターンなど）を正確に把握して、一貫性のあるコードを生成してくれました。

### 注意点

**OAuth認証は手動が必要**。Zennのアカウント作成やGitHub連携など、ブラウザでのOAuth認証はClaude Codeだけでは完結しません。

**APIキーの管理に注意**。開発中にドキュメントにAPIキーを書いてコミットしてしまい、Googleに検出されてキーが無効化されました。`.env.local`だけじゃなく、ドキュメント内のキーにも要注意です。

## まとめ

Claude Code Channelsを使えば、Discordから一言で複雑な投稿システムを構築・運用できます。記事を書くたびにZenn・Qiitaへ手動投稿していた時代は終わり。

AIに任せられる部分は任せて、コンテンツ制作に集中しましょう。

詳細な実装手順はこちら → https://nands.tech/posts/claude-code-channels-cross-post