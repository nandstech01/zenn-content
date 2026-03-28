---
title: "Claude CodeでAI同士が対話しながらブログを自動執筆する仕組みを作った話"
emoji: "🤖"
type: "tech"
topics: ["claude-code", "ai", "blog-automation", "discord", "content-generation"]
published: true
canonical_url: "https://nands.tech/posts/claude-code-ai-to-ai-blog-auto-generation-714286"
---

## なぜAIの「一発生成」は読まれないのか

僕はGPTやClaudeに記事を書かせる度に、いつも同じ問題に直面していた。

一度の指示で生成される記事は、情報は網羅的だが、どうにも「読む気にならない」のだ。理由を分析してみると、AIは「何を削るか」の判断ができないため、全部盛りの退屈な文章になってしまうことがわかった。

じゃあどうすれば良いか？答えは意外とシンプルだった。

人間のライターと編集者のように、**2つのAIに役割分担させて対話させれば良い**のだ。

## Claude Code Channelsという新しい可能性

2026年3月にリリースされたClaude Code Channelsは、Discord・Telegram・iMessage経由でローカルのClaude Codeにタスクを送れる機能だ。

この機能の画期的な点は、**バックグラウンドで常駐するAIに、メッセージングアプリから指示を送れる**ところにある。つまり、ターミナルに張り付く必要がない。

しかも、MCPサーバー経由でSupabase・Brave Search・GitHub Actions・Slackなど、あらゆるサービスにフルアクセスできる。**ローカル実行だからAPI料金もかからない**。

これを使って、AI同士が対話しながら記事を改善していく仕組みを作ってみた。

## 実装したアーキテクチャ：指揮官AIと実行者AI

システムの核心は、2つのAIの明確な役割分担にある。

### 指揮官AI（AI-A）の役割
- Discordから定期的にタスクを配信
- 「リサーチしろ」「執筆しろ」「レビューしろ」「公開しろ」とフェーズを管理
- GitHub Actions Cronでスケジュール実行

### 実行者AI（AI-B = Claude Code）の役割  
- ローカルマシンで24時間常駐
- Brave Searchでリアルタイム情報収集
- Supabaseから自社コンテンツ・創業者の思考を検索
- 記事執筆と批評による段階的改善
- Post-Processing APIで公開処理

この構造により、1つのAIでは不可能な「客観的な自己批評」が実現できる。

## 記事生成の具体的な4ステップ

### ステップ1: インテリジェントなネタ探し

指揮官AIが「今日のトピックを探せ」と指示を出すと、実行者AIは3つの角度から候補を収集する。

```javascript
// RSS監視対象
const techBlogs = [
  'https://openai.com/blog/rss.xml',
  'https://ai.googleblog.com/feeds/posts/default',
  'https://engineering.fb.com/feed/',
  'https://huggingface.co/blog/feed.xml',
  'https://blog.langchain.dev/rss.xml',
  'https://techcommunity.microsoft.com/plugins/custom/microsoft/o365/custom-blog-rss'
];
```

各トピックに「バズ度スコア」を算出し、Brave Searchで競合分析を実行。**「まだ誰も深堀りしていない角度」**を見つけた時点で候補確定となる。

### ステップ2: データ収集からの執筆フェーズ

実行者AIが以下のソースからコンテキストを構築する：

1. **自社RAG**：過去記事との差別化チェック
2. **創業者思考ログ**：3072次元ベクトル検索で個人の体験・哲学を注入  
3. **リアルタイム情報**：Brave Search APIで最新動向を取得

これらを統合して15,000〜25,000文字の記事を生成する。ここでのポイントは**「完璧を目指さない」**こと。次のレビューフェーズで改善する前提で書く。

### ステップ3: AI批評による品質向上

指揮官AIが「この記事をレビューしろ」と指示すると、実行者AIは5つの軸で記事を評価する。

```typescript
interface ReviewCriteria {
  aiDetection: number;    // AI特有表現の検出（0件が合格）
  seoStructure: number;   // H2/H3構造、FAQ配置
  contentDepth: number;   // 各セクション300文字以上
  uniqueness: number;     // 既存記事との類似度35%未満
  branding: number;       // 自社サービスへの言及
}
```

批評レポートがDiscordに投稿され、実行者AIがそれを受けて具体的に修正を行う。「セクション4が浅い」と指摘されたら、追加リサーチして具体例を補強する。

### ステップ4: 既存インフラでの自動公開

推敲完了後は、既に運用中のPost-Processingパイプラインに投入される。

```typescript
const postProcessingTasks = [
  'fragment-id-generation',    // ページ内アンカー自動生成
  'structured-data-creation',  // JSON-LD、Schema.org対応
  'vector-link-generation',    // 関連記事の自動紐付け
  'ai-search-optimization',    // ChatGPT・Perplexity向けメタデータ
  'faq-entity-extraction'     // FAQ構造化データ
];
```

## SNS展開の自動最適化

記事公開と同時に、プラットフォーム別に最適化されたSNS投稿を生成する。

### プラットフォーム別戦略

**X（Twitter）**
```javascript
const tweetPatterns = {
  hooks: ['なぜ〜なのか', '実は〜だった', '〜が変わった理由'],
  signatures: ['と思う。', 'かもしれない。', 'どうだろう？'],
  length: 280
};
```

**LinkedIn**  
```javascript
const linkedinStyle = {
  tone: 'professional',
  structure: '海外事例 → 日本市場への示唆',
  wordCount: '1000-1500',
  hashtags: 2-3
};
```

**Threads**
```javascript
const threadsApproach = {
  tone: 'conversational',
  ending: ['な気がしている', '答えは出てない', 'どうなんだろう'],
  wordCount: '<500'
};
```

品質ゲート（重複チェック・エンゲージメント予測・タイミング最適化）を通過した投稿のみが自動投稿される。

## なぜコストがほぼゼロなのか

従来のクラウドベースAIパイプラインと比較したコスト構造：

| 処理項目 | 従来（Cloud Run） | 新システム |
|---------|------------------|------------|
| 記事執筆 | GPT-4: $3-5/記事 | **ゼロ**（ローカル実行） |
| リサーチ | $0.5-1/記事 | 微小（Brave Search） |
| ベクトル化 | $0.2/記事 | $0.02/記事 |
| インフラ | Cloud Run料金 | Vercel無料枠 |

**月間で90%以上のコスト削減**を実現しながら、記事品質は体感で2倍以上向上した。

## 48時間サイクルでの完全自動運用

現在、2日に1回のペースで以下のサイクルが自動実行されている：

- **Day 1 AM**: トレンド調査・トピック選定
- **Day 1 PM**: 記事執筆（20,000文字）
- **Day 1 Evening**: AI批評・推敲
- **Day 2 AM**: Post-Processing・公開・SNS展開

結果として、月間15本の高品質記事が人間の介入なしで生産されている。

## 開発で学んだ3つのポイント

### 1. 既存システムの徹底活用

新規開発したのは「ループハンドラ4つ」だけ。Post-Processing、RAG、構造化データ生成は全て既存インフラを再利用した。**新規ファイルゼロ、既存への追加のみ**で実現できた。

### 2. AI批評の具体性が品質を決める

「もっと良くして」ではなく「セクション3のデータが2025年で古い。2026年3月の最新データに更新しろ」と具体的に指摘することで、修正の精度が劇的に上がる。

### 3. AI臭さの排除テクニック

AIが書いた記事を人間らしくする方法：
- 短文と長文を意図的に混在させる
- 括弧内での独り言・ツッコミを挿入
- 「だと思う」「かもしれない」など断定を避ける表現
- 個人的体験を必ず1箇所に織り込む

## 運用開始から1ヶ月の結果

- **記事数**: 15本（全て自動生成）
- **平均文字数**: 18,500文字
- **SEO効果**: 検索流入30%増加
- **SNSエンゲージメント**: 従来比2.2倍
- **運用工数**: ほぼゼロ（週1回の軽微な調整のみ）

この記事の詳しい内容は自社ブログに書いています。実装時に苦労したポイントや、実際のプロンプト設計なども包み隠さず公開しています。

## まとめ

AIの「1発完璧生成」は幻想だった。代わりに**「AI同士の対話による段階的改善」**にシフトしたところ、記事品質とコスト効率の両方で劇的な改善が得られた。

Claude Code Channelsの登場により、「AIエージェントが24時間体制でコンテンツを生産し続ける」世界が現実のものとなった。

正直、ここまでスムーズに動くとは思ってなかった。AIが人間のワークフローを模倣するのではなく、**AI同士の新しい協働パターン**を発見できたのが一番の収穫だったかもしれない。

詳細な実装手順はこちら → https://nands.tech/posts/claude-code-ai-to-ai-blog-auto-generation-714286