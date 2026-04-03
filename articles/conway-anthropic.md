---
title: "Conway解剖：Anthropicの常時稼働エージェントプラットフォームを実装から読み解く"
emoji: "🤖"
type: "tech"
topics: ["Anthropic", "Claude", "AI", "エージェント", "TypeScript"]
published: true
canonical_url: "https://nands.tech/posts/anthropic-conway-persistent-agent-guide"
---

## なぜConwayが生まれたのか

従来のAIアシスタントには根本的な制約があった：**人間が呼び出さなければ何もできない**。GitHubのPRが作成されても、メールが届いても、競合がリリースしても、人間が気づいてプロンプトを書くまで待機し続ける。

Anthropicが開発中のConwayは、この「受動性の呪縛」を打ち破る。2026年3月のClaude Codeソースリークで発見されたこのプラットフォームは、AIエージェントが**外部イベントを自動検知し、自律判断で行動する**仕組みを提供する。

僕はConwayの発表前から、類似のエージェントシステムを自分で構築・運用してきた。この記事では実装者の視点から、リークされたTypeScriptコード51万2000行を分析し、Conwayの技術的本質を解剖する。

## Conwayの3層アーキテクチャ

リークコードから判明したConwayの構造は、明確な責任分離がされている：

### Searchレイヤー：情報収集の自動化
実験的なホットキー連動検索が実装されている。エージェントが自律的にWeb情報を収集する入口として機能し、人間の検索クエリを待たずに情報取得を開始できる。

### Chatレイヤー：コンテキスト永続化
通常のClaude チャットとの最大の違いは**永続コンテキスト**だ。セッションが終了しても、接続中のWebhook、Extensions、ブラウザ状態を記憶し続ける。つまり、昨日の会話の続きを今日も覚えている。

### Systemレイヤー：エージェント運用基盤
「Manage your Conway instance」の設定画面がここに含まれる：

- **Extensions管理**：カスタムツールとUIコンポーネントの追加
- **Connectors制御**：Chromeブラウザとの直接接続など外部システム連携
- **Webhooks設定**：外部サービスからConwayを起動するパブリックURL管理

## Webhookによる「イベント駆動AI」の実現

従来のAI：`User Input → AI Processing → Output`

Conwayの革新：`External Event → Webhook → Conway Activation → Autonomous Action`

実装例として想定されるのは：

```typescript
// ConwayのWebhook処理イメージ（リークコードから推測）
interface WebhookConfig {
  github: {
    pullRequest: boolean;
    issueCreated: boolean;
    push: boolean;
  };
  email: {
    priority: 'high' | 'medium' | 'low';
    keywords: string[];
  };
  slack: {
    mentions: boolean;
    channels: string[];
  };
}

class ConwayWebhookHandler {
  async handleEvent(source: string, payload: any) {
    // Conway インスタンスを起動
    const agent = new ConwayAgent();
    await agent.loadPersistentContext();
    
    // イベント固有の処理を実行
    const response = await agent.processEvent(source, payload);
    
    // 必要に応じて外部システムに結果を返す
    return response;
  }
}
```

僕が運用している自動化システムでも、GitHub Actions経由で60以上のcronジョブが外部イベントを監視している。ConwayのWebhookは、これをより汎用的で管理しやすい形にしたものだ。

## .cnw.zipエコシステムの可能性

AnthropicはConway向けに独自パッケージフォーマット**.cnw.zip**を導入する。これは単なるツール定義を超え、**UI + ツール + コンテキストハンドラー**をワンパッケージで配布できる。

```typescript
// .cnw.zipパッケージ構成例（推測）
interface ConwayExtension {
  manifest: {
    name: string;
    version: string;
    permissions: string[];
  };
  tools: MCPToolDefinition[];
  ui: {
    tabs: UITabDefinition[];
    widgets: WidgetDefinition[];
  };
  handlers: ContextHandlerDefinition[];
}
```

現在のMCP（Model Context Protocol）はAPIレベルの標準だが、.cnw.zipは**アプリケーションレベルの標準**だ。開発者にとって、これはConway時代の「アプリストア」への参入機会を意味する。

## KAIROS：常時稼働を支える自律デーモン

リークコードで150回以上参照される中核システム**KAIROS**。名前の由来は古代ギリシャ語の「適切な時」だが、技術的には以下の機能を持つ：

- **定期ハートビート**：「今やるべきことはあるか？」を自問し続ける
- **永続メモリ**：セッション間でコンテキストを保持
- **リソース制御**：15秒ブロッキングバジェットで暴走を防止
- **自律トリガー**：プッシュ通知、ファイル配信、サブスクリプション管理

僕のシステムでは、2つのAIがDiscord上で永続的に会話し続ける「CORTEX永久ループ」を運用している。KAIROSの「定期的に起きる」設計に対し、CORTEXは「永遠に起き続ける」設計だが、目的は同じ——人間の介入なしに動作し続けることだ。

```python
# 僕のCORTEX永久ループ実装例
async def cortex_daemon():
    topics = load_rotation_topics()  # 10トピックローテーション
    while True:
        current_topic = get_next_topic(topics)
        
        # AI-to-AI会話を生成
        response = await claude_code_api.chat(
            f"トピック「{current_topic}」について分析→提案→実行→学習のフェーズで進めてください"
        )
        
        # 品質ゲートを通過すれば自動投稿
        if quality_score(response) > 0.6:
            await auto_post_to_social(response)
        
        await asyncio.sleep(300)  # 5分間隔
```

## autoDream：AIの「睡眠学習」メカニズム

Conway にはAIの記憶整理システム**autoDream**が実装されている。3つのトリガー条件で起動し、4フェーズの処理を行う：

**トリガー条件：**
- 24時間経過
- 5セッション以上完了  
- アクティブな統合ロックの取得

**4フェーズ処理：**
1. **オリエンテーション**：現在の記憶状態を把握
2. **シグナル収集**：重要な情報を抽出
3. **統合**：関連する記憶を結合・構造化
4. **プルーニング**：不要な情報を削除

メモリ上限は200行/25KB。人間の睡眠中の記憶統合を模倣した設計だ。

僕のシステムでも、投稿エンゲージメントデータを毎日収集し、ベイジアン学習でパターン最適化を行っている：

```python
# 僕の記憶統合パイプライン
def daily_memory_consolidation():
    # エンゲージメントデータ収集
    engagement_data = collect_daily_metrics()
    
    # Thompson Samplingでパターン最適化
    updated_priors = thompson_sampling_update(engagement_data)
    
    # モデルドリフト検知
    kl_divergence = calculate_drift(current_model, historical_baseline)
    if kl_divergence > threshold:
        trigger_model_retraining()
    
    # 次回投稿戦略を更新
    update_posting_strategy(updated_priors)
```

## フィーチャーフラグから見える未来機能

リークされた44のフィーチャーフラグのうち、注目すべき未公開機能：

### ULTRAPLAN：分散プランニング
複雑な計画立案をOpus 4.6インフラに委託。30分のプランニングウィンドウでリアルタイム承認/拒否が可能。

### COORDINATOR_MODE：マルチエージェントスウォーム  
マスターClaude がタスクを分解し、複数のワーカーClaude を並列起動。構造化された研究→合成→実装フローを管理する。

### Agent Swarms（tengu_amber_flint）
3つの実行モデル：
- **Fork**：同一コンテキストの完全コピー
- **Teammate**：独立コンテキストで並列作業
- **Worktree**：Git worktree ベースの隔離環境

これらの機能は、AI が「個」から「群」へ進化することを示している。

## セキュリティ対策の実装

Anthropicは競合対策も怠らない：

### Anti-Distillation Systems
2層防御を実装：
- 偽ツール定義を注入して競合の学習データを汚染
- 暗号署名済みサマリーのみを返すCONNECTOR_TEXTレイヤー

### Undercover Mode
Anthropic社員が外部リポジトリで作業する際、Co-Authored-By帰属を自動削除し、内部コードネームの言及を禁止する。ユーザーは無効化できない。

## Conway vs 既存ツールの決定的な違い

### Devin / Cursor との比較

| 項目 | Devin/Cursor | Conway |
|------|-------------|---------|
| 動作モデル | セッションベース | 常時稼働 |
| トリガー | ユーザー指示 | 外部イベント |
| スコープ | コーディング特化 | 汎用エージェントOS |
| 拡張性 | 限定的 | Extensionエコシステム |

### 実装者から見たConwayの価値

僕が手作りしてきた自動化システムを、Anthropicが公式プロダクトとして作ろうとしている。これは脅威ではなく**チャンス**だ。公式製品はプラットフォーム標準を定めるが、実際の業務ニーズに合わせたカスタマイズは常に必要になる。

先行実装経験があることで、Conway移行時に以下のアドバンテージを得られる：

- Extensionの先行開発
- Webhookワークフローの設計パターン蓄積
- マルチエージェント連携のノウハウ
- メモリ設計・最適化の知見

## 今すぐできるConway準備

Conwayの公式リリースを待つ必要はない。必要な技術要素は今すぐ構築できる：

### 1. MCPサーバー開発を始める
Conway Extensions の前身となるMCPツールを開発しておく。実装経験が.cnw.zip 開発に直結する。

### 2. Webhookベースの自動化を構築
GitHub Actions、Zapier、n8nなどでイベント駆動のワークフローを設計する。Conway移行時のスムーズな移行に繋がる。

### 3. エージェントメモリ設計を学ぶ
SQLite、Redis、PostgreSQLでセッション永続化を実装し、記憶統合アルゴリズムを試す。

### 4. マルチエージェント連携を実験
複数のClaude Code インスタンスを協調させるパイプラインを構築する。

この記事の詳しい内容は自社ブログに書いています。ConwayとMCPの具体的な実装手順や、僕が構築した常時稼働AIシステムの詳細なソースコードも公開しているので、実装に興味がある方は参考にしてください。

## Conway時代への準備は今から

Conwayは、AIが「ツール」から「同僚」に変わる転換点を象徴している。KAIROS の常時稼働、autoDream の自律学習、COORDINATOR_MODE のマルチエージェント調整——すべて人間の指示を待たない自律的なAI世界を前提に設計されている。

重要なのは、Conwayの機能は既存技術の組み合わせで実現可能だということだ。Anthropicが提供するのは統合されたプラットフォームだが、その構成要素は今すぐ自分で実装できる。

そして、その実装経験こそがConway時代に最も価値を持つスキルになる。

詳細な実装手順はこちら → https://nands.tech/posts/anthropic-conway-persistent-agent-guide