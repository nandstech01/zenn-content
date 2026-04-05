---
title: "GSC×GA4を完全自動化してSEOのボトルネックを発見するAIパイプライン構築記"
emoji: "🤖"
type: "tech"
topics: ["SEO", "API", "AI", "Google Analytics", "Google Search Console"]
published: true
canonical_url: "https://nands.tech/posts/claude-code-gsc-ga4-ai-seo-autonomous-analysis-406853"
---

自分が運営するブログで衝撃的なデータを発見しました。

ある記事が**月間4,000回以上Google検索に表示されているのに、CTRが0.02%**——つまり、ほぼ誰もクリックしていない状況です。

しかも、この致命的なボトルネックを見つけたのは僕ではありません。**AIが自動で検出してSlackに通知**してきたのです。

この記事では、Google Search ConsoleとGoogle Analytics 4のデータを自動取得し、AIが記事を評価・改善提案まで行う完全自動SEO分析システムの構築プロセスを共有します。

## なぜSEO分析を自動化したのか

100記事を超えるブログを運営していると、手動でのSEO分析には限界があります。

従来の作業フロー：
- GSCで検索パフォーマンスを確認
- GA4でユーザー行動を分析
- Excelに転記して比較検討
- 改善対象を手動で選定

この作業を毎週やるとなると、111記事では物理的に不可能です。

そこで構築したのが、**人間が何もしなくても勝手にSEOの課題を見つけてくれる**システムです。

## システムアーキテクチャ

パイプラインは4つのコンポーネントから構成されています：

```typescript
// 1. GSCデータ取得
const fetchGSCData = async () => {
  const auth = new google.auth.OAuth2();
  // OAuth2認証でSearch Console API呼び出し
  const searchconsole = google.searchconsole({ version: 'v1', auth });
  
  const response = await searchconsole.searchanalytics.query({
    siteUrl: SITE_URL,
    requestBody: {
      startDate: '2024-01-01',
      endDate: '2024-01-31',
      dimensions: ['page', 'query'],
      aggregationType: 'auto'
    }
  });
  
  return response.data.rows;
};

// 2. GA4データ取得
const fetchGA4Data = async () => {
  const analyticsDataClient = new BetaAnalyticsDataClient();
  
  const [response] = await analyticsDataClient.runReport({
    property: `properties/${GA4_PROPERTY_ID}`,
    dateRanges: [{ startDate: '30daysAgo', endDate: 'today' }],
    dimensions: [{ name: 'pagePath' }],
    metrics: [
      { name: 'sessions' },
      { name: 'bounceRate' },
      { name: 'averageSessionDuration' }
    ]
  });
  
  return response.rows;
};
```

毎朝6時にGitHub Actionsで自動実行され、全データをSupabaseに蓄積します。

### データベース設計

核となるテーブル構造：

```sql
-- GSC検索パフォーマンス
CREATE TABLE gsc_page_analytics (
  id SERIAL PRIMARY KEY,
  page_path TEXT NOT NULL,
  impressions INTEGER,
  clicks INTEGER,
  ctr DECIMAL,
  position DECIMAL,
  date_collected DATE,
  UNIQUE(page_path, date_collected)
);

-- GA4行動データ
CREATE TABLE ga4_page_analytics (
  id SERIAL PRIMARY KEY,
  page_path TEXT NOT NULL,
  sessions INTEGER,
  bounce_rate DECIMAL,
  avg_session_duration INTEGER,
  date_collected DATE,
  UNIQUE(page_path, date_collected)
);

-- 記事品質スコア
CREATE TABLE article_quality_scores (
  id SERIAL PRIMARY KEY,
  page_path TEXT UNIQUE NOT NULL,
  seo_score DECIMAL,
  engagement_score DECIMAL,
  overall_score DECIMAL,
  grade CHAR(1),
  calculated_at TIMESTAMP DEFAULT NOW()
);
```

## 3軸スコアリングアルゴリズム

記事の品質を多角的に評価するため、3つの軸でスコア算出します。

### SEO軸（40%の重み）

```typescript
const calculateSEOScore = (gscData) => {
  // CTR相対評価：順位別期待CTRに対する実績比較
  const expectedCTR = getExpectedCTRByPosition(gscData.position);
  const ctrRatio = (gscData.ctr / expectedCTR) * 100;
  
  // 順位評価：上位ほど高スコア
  const positionScore = Math.max(0, 100 - (gscData.position - 1) * 5);
  
  // 表示回数：対数スケールで評価
  const impressionScore = Math.min(100, Math.log10(gscData.impressions) * 25);
  
  return (ctrRatio * 0.4) + (positionScore * 0.4) + (impressionScore * 0.2);
};
```

期待CTRカーブは検索順位から算出。1位なら約30%、5位なら約5%のCTRが期待値となり、実際のCTRとの比較で相対評価します。

### エンゲージメント軸（40%の重み）

```typescript
const calculateEngagementScore = (ga4Data) => {
  // 滞在時間評価：180秒以上で満点
  const durationScore = Math.min(100, (ga4Data.avgSessionDuration / 180) * 100);
  
  // 直帰率評価：30%未満で満点
  const bounceScore = Math.max(0, 100 - (ga4Data.bounceRate * 100 / 30) * 100);
  
  // セッション数：対数スケール
  const sessionScore = Math.min(100, Math.log10(ga4Data.sessions) * 30);
  
  return (durationScore * 0.4) + (bounceScore * 0.4) + (sessionScore * 0.2);
};
```

### コンバージョン軸（20%の重み）

```typescript
const calculateConversionScore = (ga4Data) => {
  // CV率とイベント発生率で評価
  const conversionRate = ga4Data.conversions / ga4Data.sessions;
  const eventDensity = ga4Data.events / ga4Data.sessions;
  
  const cvScore = Math.min(100, conversionRate * 1000);
  const eventScore = Math.min(100, eventDensity * 20);
  
  return (cvScore * 0.5) + (eventScore * 0.5);
};
```

### 最終スコアとグレード判定

```typescript
const calculateOverallScore = (seoScore, engagementScore, conversionScore) => {
  return (seoScore * 0.4) + (engagementScore * 0.4) + (conversionScore * 0.2);
};

const assignGrade = (score) => {
  if (score >= 80) return 'S';
  if (score >= 65) return 'A';
  if (score >= 50) return 'B';
  if (score >= 30) return 'C';
  return 'D';
};
```

## 改善機会の自動検出

スコアリングと並行して、具体的な改善アクションを自動特定します。

### Quick Wins検出

```typescript
const detectQuickWins = (articles) => {
  return articles.filter(article => 
    article.position >= 4 && 
    article.position <= 15 && 
    article.impressions >= 100
  ).map(article => ({
    ...article,
    opportunity_type: 'quick_win',
    priority: article.position <= 7 ? 'high' : 'medium',
    potential_impact: estimateClickIncrease(article)
  }));
};
```

検索順位4-15位で表示回数100回以上の記事を「あと少しでTOP3入りできる記事」として検出。これらはタイトル改善だけでクリック数が2-3倍になる可能性があります。

### 低CTR記事検出

```typescript
const detectLowCTR = (articles) => {
  return articles.filter(article => {
    const expectedCTR = getExpectedCTRByPosition(article.position);
    return article.ctr < (expectedCTR * 0.3); // 期待CTRの30%未満
  }).map(article => ({
    ...article,
    opportunity_type: 'low_ctr',
    severity: article.impressions > 1000 ? 'critical' : 'medium'
  }));
};
```

実際の検出結果で最も衝撃的だったのがこちら：

```
CRITICAL: 「AIエージェントとは」記事
→ 月間表示回数: 4,241回
→ 実際のCTR: 0.02%
→ 期待CTR: 1.5%（10.8位相当）
→ 機会損失: 月間約60クリック
```

手動チェックでは完全に見落としていた重大な機会損失です。

## Slack通知の自動化

検出結果は毎朝Slackに自動通知されます：

```typescript
const sendSlackReport = async (opportunities) => {
  const quickWins = opportunities.filter(o => o.opportunity_type === 'quick_win');
  const lowCTR = opportunities.filter(o => o.opportunity_type === 'low_ctr');
  
  const message = {
    channel: '#seo-alerts',
    text: '🔍 Daily SEO Report',
    blocks: [
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*Quick Wins検出: ${quickWins.length}件*\n${formatQuickWins(quickWins)}`
        }
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*低CTR記事: ${lowCTR.length}件*\n${formatLowCTR(lowCTR)}`
        }
      }
    ]
  };
  
  await slack.chat.postMessage(message);
};
```

人間がやることは朝Slackを開いて「この記事のタイトルを修正するか」と判断するだけです。

## 実際の成果

111記事をスコアリングした結果：

| グレード | 記事数 | 割合 |
|---------|--------|------|
| S | 0 | 0% |
| A | 1 | 0.9% |
| B | 10 | 9.0% |
| C | 28 | 25.2% |
| D | 72 | 64.9% |

**Sランク記事がゼロ**という衝撃的な結果。つまり全記事に改善余地があります。

検出された改善機会：
- Quick Wins: 18件
- 低CTR記事: 3件  
- 高直帰率記事: 12件
- 下降トレンド記事: 8件

特に「月間4,000表示でCTR 0.02%」の記事発見は、タイトル1行の修正で月間数十クリックを獲得できる即効性の高い改善機会でした。

## Claude Codeでの実装体験

このシステムの核となるスコアリングロジックは、Claude Codeで実装しました。

従来のコーディング環境と比較した Claude Codeの利点：
- **API連携が簡単**：googleapis、@google-analytics/dataの使い方を質問しながら実装
- **複雑な計算ロジックの可読性**：期待CTRカーブの算出など、数式をコードで表現する際のサポート
- **エラーハンドリングの提案**：API制限やネットワークエラーに対する適切な例外処理

特に印象的だったのは、「記事の品質を数値化したい」という曖昧な要求から、3軸スコアリングのアルゴリズム設計まで対話的に進められたことです。

## システムの拡張性

現在のパイプラインは単体で動作しますが、さらなる自動化の可能性があります：

### 記事改善の自動実行
```typescript
// タイトル改善提案の自動生成
const generateTitleSuggestions = async (article) => {
  const prompt = `
    以下の記事データから、CTRを向上させるタイトル改善案を3つ提案してください：
    - 現在のタイトル: ${article.title}
    - 検索順位: ${article.position}
    - 現在のCTR: ${article.ctr}%
    - 主要検索クエリ: ${article.top_queries.join(', ')}
  `;
  
  return await callClaudeAPI(prompt);
};
```

### コンテンツ戦略への反映
```typescript
// 高パフォーマンス記事の特徴分析
const analyzeTopPerformers = (articles) => {
  const topArticles = articles.filter(a => a.grade === 'A' || a.grade === 'B');
  
  return {
    commonKeywords: extractCommonKeywords(topArticles),
    averageWordCount: calculateAverageWordCount(topArticles),
    successPatterns: identifyContentPatterns(topArticles)
  };
};
```

## 導入時の注意点

### API制限の管理
```typescript
// レート制限対応の実装例
const withRetry = async (apiCall, maxRetries = 3) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await apiCall();
    } catch (error) {
      if (error.status === 429) {
        await sleep(2 ** i * 1000); // Exponential backoff
        continue;
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
};
```

### データの信頼性確保
GSCデータは2-3日の遅延があるため、確定データのみを使用する設計が重要です：

```typescript
const getReliableDataRange = () => {
  const today = new Date();
  const endDate = new Date(today.setDate(today.getDate() - 3));
  const startDate = new Date(today.setDate(today.getDate() - 33));
  
  return { startDate, endDate };
};
```

## まとめ

手動でのSEO分析から完全自動化への移行で得られた最大の価値は、**見落としの撲滅**でした。

人間の目では「4,000表示のCTR 0.02%」という異常値を毎週発見するのは困難です。しかしシステムなら確実に検出し、改善提案まで自動実行します。

この記事の詳しい内容は自社ブログに書いています。特にAPI認証の詳細手順や、実際のコード全量、テーブル設計の考え方などをより詳しく解説しています。

現在のSEO分析が手動中心の方は、まず「Quick Wins検出」から自動化を始めることをお勧めします。検索順位4-10位の記事を特定するだけでも、大幅な工数削減と改善機会の発見につながります。

詳細な実装手順はこちら → https://nands.tech/posts/claude-code-gsc-ga4-ai-seo-autonomous-analysis-406853