---
title: "Claude Sonnet 5.5 への移行手順：Sonnet 5 との違い・影響する変更 5 つ・effort の設定"
emoji: "🔁"
type: "tech"
topics: ["claude", "claudecode", "anthropic", "llm", "api"]
published: true
---

Claude Sonnet 5.5 が出て、Sonnet 5 はレガシーのモデルになりました。単価は変わりません。ただし、Sonnet 5 で動いているコードに影響する変更が 5 つあります。effort の各レベルも調整し直されています。この記事では、Sonnet 5 から Sonnet 5.5 に移るときに Claude Code と API で書き換える点を、順に説明します。

作業は「変更に当たるコードを直す → モデル ID を替える → effort を測り直す」の順に進めます。

## Sonnet 5 から Sonnet 5.5 に替えると料金と仕様は何が変わる？

料金は変わらず、モデル ID と知識の区切りが変わります。API の料金は、どちらも入力 $2・出力 $10 / MTok です。知識の区切り (Reliable knowledge cutoff) は、Sonnet 5.5 が Jun 2026、Sonnet 5 が Jan 2026 です。

| 項目 | Sonnet 5 | Sonnet 5.5 |
| --- | --- | --- |
| Claude API のモデル ID | `claude-sonnet-5` | `claude-sonnet-5-5` |
| 入力 / 出力 (1 MTok あたり) | $2 / $10 | $2 / $10 |
| キャッシュの読み込み (1 MTok あたり) | $0.20 | $0.20 |
| 知識の区切り | Jan 2026 | Jun 2026 |
| Claude API での既定の effort | high | high |
| 位置づけ | レガシー (Sonnet 5.5 への移行の検討を案内) | 移行先 |

Sonnet 5 の ID は、Sonnet 5 のページに書かれているものです。

比べる相手として、Opus 5.5 の料金も挙げておきます。入力 $4・出力 $20・キャッシュの読み込み $0.20 / MTok です。Claude API の ID は、Opus 5.5 が `claude-opus-5-5`、Fable 5.1 が `claude-fable-5-1` です (Models overview)。

## Sonnet 5 で動いているコードに影響する変更は？

公式の Sonnet 5.5 のページと Sonnet 5 のページによれば、Sonnet 5 で動いているコードに影響する変更が 5 つあります。

1. **前もって考える処理を切るには `between_tools` を使う**: 思考を切る書き方が変わります。書き換え方は後の節で示します。
2. **ツールの強制指定がエラーになる**: ツールを強制指定しているリクエストを探します。
3. **思考のブロックがモデルと会話に結び付く**: 影響する箇所と直し方は、下に挙げる Sonnet 5.5 のページで確認してください。
4. **以前のコンピューター操作のツールを受け付けない**: Claude API と Google Cloud では、`computer_20251124` を受け付けません。このツールを指定している箇所を探します。
5. **アドバイザーツールが Sonnet 5 などをアドバイザーとして受け付けない**: アドバイザーツールで Sonnet 5 などをアドバイザーに指定している設定を探します。

2・3・5 の具体的な直し方は、次の公式ページで確認してください。

- Sonnet 5.5 のページ: https://platform.claude.com/docs/en/models/sonnet-5-5/overview
- Sonnet 5 のページ: https://platform.claude.com/docs/en/models/sonnet-5/overview

この 5 つとは別に、表示に影響する変更もあります。ツール呼び出しの間の文章を利用者に流しているアプリケーションは、後の API の節で対応を確認してください。

:::message alert
モデル ID だけを替えて切り替えると、上の 5 つに当たるコードは影響を受けます。たとえば、ツールの強制指定はエラーになります。ID を替える前に、該当する箇所を探してください。
:::

## Claude Code で Sonnet 5.5 に切り替えるには？

Claude Code では、起動時の `--model` かセッション中の `/model` でモデルを選びます。公式には、どちらも `<alias|name>` の形でエイリアスかモデルの名前を受け付けるとされています。Model configuration のページにある書式の例 (公式) は、エイリアスで指定する次の形です。

```text
claude --model opus
/model sonnet
```

`sonnet` などのエイリアスがどのモデルを指すかは、Model configuration のページで確認してください。`--model` と `/model` には、エイリアスのほかにモデルの名前も渡せます。当社が実測で確かめたのも、`claude -p` の `--model` にモデルの ID `claude-sonnet-5-5` を渡して切り替える形です。

```text
claude --model claude-sonnet-5-5
/model claude-sonnet-5-5
```

1 行目は起動するとき、2 行目はセッションの中で打ちます。効く範囲ごとの操作は次のとおりです。

| 効く範囲 | 操作 |
| --- | --- |
| そのセッションだけ | 引数なしの `/model` で選択画面を開き、Sonnet 5.5 を選んで `s` を押す |
| 新しいセッションの既定にする | 引数なしの `/model` で選択画面を開き、Sonnet 5.5 を選ぶ |
| 設定ファイルに残す | 設定ファイルに `model` キーで ID を書く (例は下のとおり) |

設定ファイルに書く場合は、`model` に ID を書きます。設定ファイルに書くと、以後も続けて効きます。

```json
{"model": "claude-sonnet-5-5"}
```

## Sonnet 5.5 の effort はどのレベルから始めればいい？

Sonnet 5.5 のエージェントのコーディングでは、仕様のはっきりした課題は `medium` から始め、難しい課題や長い課題で `high` に上げるよう公式に勧められています。

Sonnet 5.5 の effort について、押さえておく点は次のとおりです。

- **選べるレベル**: `low`・`medium`・`high`・`xhigh`・`max`
- **既定のレベル**: Claude Code では `medium`、Claude API では `high`
- **Sonnet 5 の設定は持ち越さない**: 公式の Effort のページは、Sonnet 5.5 のレベルは調整し直されていると説明しています。そのため、Sonnet 5 の設定を持ち越さず、自分の評価でレベルを測り直すよう勧めています。

### effort を変える方法

目的ごとの操作は次のとおりです。

| やりたいこと | 操作 |
| --- | --- |
| 今のセッションだけレベルを試す | 引数なしの `/effort` でスライダーを開き、レベルを選んで `s` を押す (`s` は Claude Code v2.1.257 以降) |
| 選んだレベルを既定として保存する | 引数なしの `/effort` でスライダーを開き、レベルを選んで Enter を押す |
| 起動するときに明示的に指定する | `--effort` にレベルを渡す (例: `claude --effort medium`) |
| Sonnet 5.5 のレベルを設定に残す | 設定の `modelSettings` にモデルごとのレベルを書く (下の注意を参照) |

`/effort` にレベル名を続けると、すぐにそのレベルになります。`max` は `modelSettings` にも `effortLevel` にも書けません。

たとえば、仕様のはっきりした作業を `medium` で始め、難しい課題で `high` に上げる流れは次のようになります。

```text
/effort medium
/effort high
```

:::message alert
ユーザー設定の最上位に書いた `effortLevel` は、Opus 5.5 とそれより後に出たモデル (Sonnet 5.5 を含む) には効きません。Sonnet 5 のころに最上位の `effortLevel` でレベルを決めていた場合、Sonnet 5.5 にはその値が使われません。Sonnet 5.5 のレベルを設定に残すときは、`modelSettings` の書式を Model configuration のページ (https://code.claude.com/docs/en/model-config) で確かめ、Sonnet 5.5 のレベルを書きます。
:::

## API で思考を切っていた場合はどう書き換える？

Sonnet 5.5 では、`"disabled"` の代わりに `thinking: {"type": "between_tools"}` を送ります。モデル ID の変更とあわせると、リクエストで書き換える部分は次のようになります。

```json
{"model": "claude-sonnet-5-5", "thinking": {"type": "between_tools"}}
```

この設定は `low`・`medium`・`high` で使えます。Claude API での Sonnet 5.5 の既定の effort は `high` です。`xhigh`・`max` は、使える範囲として挙げられていません。

ツール呼び出しの間に文章を利用者へ表示している場合は、次のどちらかを済ませてください。Sonnet 5.5 ではその文章が思考のブロックで返るため、何もしないとツール呼び出しの間は何も表示されなくなります。

- 表示の値を設定する
- `between_tools` で前もって考える処理を切る

## Sonnet 5.5 と Opus 5.5 はどの場面で使い分ける？

公式の案内では、範囲のはっきりした日常のコーディングは Sonnet 5.5、複雑で長いエージェントのコーディングは Opus 5.5 に向くとされています。

場面ごとに整理すると次のようになります。

| 場面 | 向くとされるモデル | 案内の出どころ |
| --- | --- | --- |
| 範囲のはっきりした日常の作業・不具合の修正 | Sonnet 5.5 | Sonnet 5.5 の発表 |
| 数時間の自律的なコーディングエージェント・大規模なリファクタリング | Opus 5.5 | Choosing the right model |
| どれか迷ったとき | 多くの作業は Opus 5.5 から始める | Models overview |

Opus 5.5 に替える前に、まず effort の調整を検討してください。Choosing the right model のページは、モデルを替えるよりその方がよい手段であることが多いとしています。

当社では、小さな Node.js の課題 6 つを各 3 回、effort をすべて high にして解かせました。3 つのモデルの結果は次のとおりです (時間と費用は 1 回あたりの中央値)。

| モデル | 成功 | 時間 | 費用 (API の定価で換算) |
| --- | --- | --- | --- |
| Sonnet 5.5 | 18/18 | 21 秒 | $0.07 |
| Sonnet 5 | 18/18 | 34 秒 | $0.10 |
| Opus 5.5 | 18/18 | 31 秒 | $0.15 |

- **実行**: Claude Code 2.1.285・2.1.283 の非対話モード (`claude -p`)。2026年9月28日・2026年9月30日 (日本時間)。
- **費用**: API の定価で換算した値で、請求額ではありません。
- **成功数**: 6 課題は、どのモデルも全問成功する易しさでした。成功数については、Sonnet 5.5 に替えても小さな課題で落ちなかったことだけが言えます。能力の差は分かりません。
- **effort**: Sonnet 5.5 は Claude Code の既定 (`medium`) より、Opus 5.5 は Claude API・Claude Code の既定 (`medium`) より 1 つ上の high で測りました。`medium` での結果は分かりません。
- **確かめていない範囲**: 長時間のエージェント作業や大きなリポジトリ。

## まとめ

切り替えるときは、次の順で確かめてください。

1. 5 つの変更に当たる箇所を探して直す (「Sonnet 5 で動いているコードに影響する変更は？」)。
2. モデル ID を `claude-sonnet-5-5` に替える (「Claude Code で Sonnet 5.5 に切り替えるには？」)。
3. effort を測り直す (「Sonnet 5.5 の effort はどのレベルから始めればいい？」)。
4. 最上位の `effortLevel` を使っていたら、`modelSettings` の書式を Model configuration のページで確かめ、Sonnet 5.5 のレベルを書く。
5. 思考を切る書き方と、ツール呼び出しの間の表示を直す (「API で思考を切っていた場合はどう書き換える？」)。

6 つの課題の中身、課題ごとの結果、Fable 5.1 を含む 5 つのモデルの比較表、再現に使ったコマンドは、[Claude Sonnet 5.5 の性能と料金を比較｜Sonnet 5・Opus 5.5・Fable 5.1 と実測](https://nands.tech/posts/claude-sonnet-5-5-vs-sonnet-5-opus-5-5-fable-5-1-iztvw2) にまとめています。
