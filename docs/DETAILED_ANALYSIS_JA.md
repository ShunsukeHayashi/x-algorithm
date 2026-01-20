# X For You Feed Algorithm - 詳細分析レポート

**作成日**: 2026-01-20
**バージョン**: xai-org/x-algorithm (Apache License 2.0)

---

## 1. システム概要

X (旧Twitter) の「For You」フィードを動かすレコメンデーションアルゴリズム。xAI がオープンソースとして公開。

### 1.1 コア原則

1. **手作り特徴量なし**: GrokベースのTransformerが全ての処理を担当
2. **候補者アイソレーション**: 各投稿のスコアは他候補に依存しない
3. **ハッシュベース埋め込み**: 複数のハッシュ関数で埋め込み検索
4. **マルチアクション予測**: 14種類のアクション確率を同時予測

### 1.2 アーキテクチャ概要

```
User Request
    ↓
Home Mixer (Rust) ← オーケストレーション
    ↓
    ├→ Thunder (Rust)     ← In-Network: フォロー中の投稿
    │   └── In-Memory Store
    │
    └→ Phoenix (Python/JAX)  ← Out-of-Network: ML検索+ランキング
        ├── Two-Tower Retrieval
        └── Grok Transformer Ranking
    ↓
Scoring Pipeline
    ↓
Ranked Feed Response
```

---

## 2. Thunder (Rust) - In-Network 投稿ストア

### 2.1 責務

- Kafkaからの投稿イベントをリアルタイム取り込み
- ユーザー別のインメモリ投稿ストア維持
- サブミリ秒級のIn-Network投稿検索

### 2.2 データ構造

```rust
pub struct PostStore {
    // 投稿ID → 完全投稿データ
    posts: Arc<DashMap<i64, LightPost>>,

    // ユーザーID → タイムライン (最小参照のみ)
    original_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,      // オリジナル投稿
    secondary_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,    // リプライ/リポスト
    video_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>,         // 動画投稿

    deleted_posts: Arc<DashMap<i64, bool>>,
    retention_seconds: u64,
    request_timeout: Duration,
}
```

**TinyPost**: 投稿ID + 作成日時のみの軽量構造体
**LightPost**: 完全投稿データ (proto定義)

### 2.3 Kafka インジェスト

**スレッド構造**:
- `kafka_num_threads` 並列スレッドでパーティションを分割
- 各スレッドが `batch_size` ずつメッセージを処理
- `Semaphore(3)` で同時処理数を制限

**イベント処理**:
1. `TweetCreateEvent`: 投稿作成 → `insert_posts()`
2. `TweetDeleteEvent`: 投稿削除 → `mark_as_deleted()`

**video判定**:
```rust
fn is_eligible_video(tweet: &Tweet) -> bool {
    tweet.media[0].video_info.duration_millis >= MIN_VIDEO_DURATION_MS
}
```

### 2.4 In-Network検索フロー

```rust
get_all_posts_by_users(user_ids, exclude_ids) {
    // 1. Original posts取得
    original = get_posts_from_map(
        original_posts_by_user,
        max_per_user=MAX_ORIGINAL_POSTS_PER_AUTHOR
    )

    // 2. Secondary posts取得 (リプライ/リポスト)
    secondary = get_posts_from_map(
        secondary_posts_by_user,
        max_per_user=MAX_REPLY_POSTS_PER_AUTHOR
    )

    // 3. フィルタリング
    // - deleted_posts除外
    // - 自己リツイート除外
    // - exclude_tweet_ids除外
    // - リプライチェーン特殊処理

    return original + secondary
}
```

**最適化**:
- `spawn_blocking` で計算集約処理を実行 (Tokioランタイム保護)
- `request_timeout` で最大処理時間制限
- 最新投稿から逆順に `MAX_TINY_POSTS_PER_USER_SCAN` 件のみ走査

### 2.5 自動保守タスク

1. **Stats Logger** (5分間隔):
   - ユーザー数、総投稿数、削除投稿数を記録
   - Prometheus メトリクス更新

2. **Auto-Trim** (2分間隔):
   - `retention_seconds` 経過の投稿を自動削除
   - `VecDeque` の容量最適化 (shrink_to_fit)

---

## 3. Phoenix (Python/JAX) - ML コンポーネント

### 3.1 Two-Tower 検索モデル

**目的**: 数百万投稿 → 数百候補に絞り込み

#### User Tower

```
User Features + Engagement History
        ↓
    [User Hashes × 2]
        ↓
  User Embedding Table
        ↓
   User Embedding [B, 1, D]
```

**構成要素**:
- `num_user_hashes = 2`
- 投稿ハッシュ `num_item_hashes = 2`
- 作者ハッシュ `num_author_hashes = 2`
- Product Surface Embedding (vocab_size=16)

#### Candidate Tower

```
Post Content
        ↓
   [Post Hash × 2]
        ↓
Post Embedding Table
        ↓
   Candidate Embedding [N, D]
```

#### ANN 検索

```python
# Cosine Similarity via Dot Product (both normalized)
similarity = dot_product(user_embedding, candidate_embeddings)
top_k_indices = argsort(similarity)[-K:]
```

### 3.2 Grok Transformer ランキングモデル

#### 3.2.1 候補者アイソレーション (重要設計)

**目的**: 各候補のスコアを他候補から独立にすること

**Attention Mask Pattern**:
``   Keys (Attend TO)
   │ User │ History │ Candidates │
   ────────┼─────────┼────────────┤
Q U │ ✓   │ ✓✓✓    │ ✗✗✗       │
u e H │ ✓   │ ✓✓✓    │ ✗✗✗       │
e r i │ ✓   │ ✓✓✓    │ ✗✗✗       │
r s │ ✓   │ ✓✓✓    │ ✗✗✗       │
i t   │ ✓   │ ✓✓✓    │ ✗✗✗       │
e     ├───────┼─────────┼────────────┤
s  C  │ ✓   │ ✓✓✓    │ diag only   │
a a   │ ✓   │ ✓✓✓    │  ↓        │
n n   │ ✓   │ ✓✓✓    │  ✓        │
d d   │ ✓   │ ✓✓✓    │            │
s i   │ ✓   │ ✓✓✓    │            │
s d   │ ✓   │ ✓✓✓    │            │
s     └───────┴─────────┴────────────┘

✓ = Can attend
✗ = Cannot attend
diag = Self-attention only
```

**実装** (`grok.py:39-71`):
```python
def make_recsys_attn_mask(seq_len, candidate_start_offset):
    # 1. 全体でcausal mask
    causal_mask = jnp.tril(jnp.ones((seq_len, seq_len)))

    # 2. 候補者間のattentionをゼロに
    attn_mask[:, :, candidate_start_offset:, candidate_start_offset:] = 0

    # 3. 候補者のself-attention (対角成分) を復活
    attn_mask[:, :, candidate_indices, candidate_indices] = 1

    return attn_mask
```

#### 3.2.2 ハッシュベース埋め込み

**Block User Reduce**:
```python
# Input: [B, num_user_hashes, D]
# Reshape: [B, 1, num_user_hashes × D]
# Project: [B, 1, D] via learned projection matrix
```

**Block History Reduce**:
```python
# Concatenate:
# - Post embeddings [B, S, num_item_hashes×D]
# - Author embeddings [B, S, num_author_hashes×D]
# - Action embeddings [B, S, D]
# - Product surface embeddings [B, S, D]
# Project: [B, S, D]
```

**Block Candidate Reduce**:
```python
# Concatenate:
# - Post embeddings [B, C, num_item_hashes×D]
# - Author embeddings [B, C, num_author_hashes×D]
# - Product surface embeddings [B, C, D]
# Project: [B, C, D]
```

#### 3.2.3 マルチアクション予測

**14種類のアクション**:

| アクション | 重み符号 | 説明 |
|-----------|----------|------|
| favorite | 正 | いいね |
| reply | 正 | リプライ |
| retweet | 正 | リポスト |
| quote | 正 | 引用リツイート |
| click | 正 | クリック |
| profile_click | 正 | プロフィールクリック |
| video_view | 正 | 動画再生 (VQV条件付き) |
| photo_expand | 正 | 画像展開 |
| share | 正 | シェア |
| follow_author | 正 | 作者フォロー |
| **not_interested** | **負** | 興味なし |
| **block_author** | **負** | 作者ブロック |
| **mute_author** | **負** | 作者ミュート |
| **report** | **負** | 通報 |

**最終スコア計算** (`weighted_scorer.rs:44-69`):
```rust
weighted_score =
    favorite_score × FAVORITE_WEIGHT +
    reply_score × REPLY_WEIGHT +
    retweet_score × RETWEET_WEIGHT +
    ...
    not_interested_score × NOT_INTERESTED_WEIGHT +  // 負
    block_author_score × BLOCK_AUTHOR_WEIGHT +        // 負
    ...
```

---

## 4. Home Mixer (Rust) - オーケストレーション層

### 4.1 Candidate Pipeline フレームワーク

**トレイト定義**:
```rust
trait Source<Q, C> {
    fn enable(&self, query: &Q) -> bool;
    fn get_candidates(&self, query: &Q) -> Future<Vec<C>>;
}

trait Hydrator<Q, C> {
    fn enable(&self, query: &Q) -> bool;
    fn hydrate(&self, query: &Q, candidates: &[C]) -> Future<Vec<Hydrated>>;
}

trait Filter<Q, C> {
    fn enable(&self, query: &Q) -> bool;
    fn filter(&self, query: &Q, candidates: Vec<C>) -> Future<FilterResult>;
}

trait Scorer<Q, C> {
    fn enable(&self, query: &Q) -> bool;
    fn score(&self, query: &Q, candidates: &[C]) -> Future<Vec<Score>>;
}

trait Selector<Q, C> {
    fn enable(&self, query: &Q) -> bool;
    fn select(&self, query: &Q, candidates: Vec<C>) -> Vec<C>;
}
```

### 4.2 パイプライン実行順序

```
1. Query Hydration (Parallel)
   ├─ UserActionSeqQueryHydrator (Engagement history)
   └─ UserFeaturesQueryHydrator (Following list)

2. Candidate Sourcing (Parallel)
   ├─ ThunderSource (In-Network posts)
   └─ PhoenixSource (Out-of-Network posts)

3. Candidate Hydration (Parallel)
   ├─ CoreDataCandidateHydrator (Post text, media)
   ├─ GizmoduckHydrator (Author info)
   ├─ VideoDurationHydrator
   ├─ SubscriptionHydrator
   └─ VFCandidateHydrator

4. Pre-Scoring Filtering (Sequential)
   ├─ DropDuplicatesFilter
   ├─ CoreDataHydrationFilter
   ├─ AgeFilter (max_age cutoff)
   ├─ SelfTweetFilter
   ├─ RetweetDeduplicationFilter
   ├─ IneligibleSubscriptionFilter
   ├─ PreviouslySeenPostsFilter
   ├─ PreviouslyServedPostsFilter
   ├─ MutedKeywordFilter
   └─ AuthorSocialgraphFilter

5. Scoring (Sequential)
   ├─ PhoenixScorer (ML predictions)
   ├─ WeightedScorer (Combine actions)
   ├─ AuthorDiversityScorer (Attenuate repeats)
   └─ OONScorer (OON adjustment)

6. Selection
   └─ TopKScoreSelector (Sort & truncate)

7. Post-Selection Hydration (Parallel)

8. Post-Selection Filtering (Sequential)
   ├─ VFFilter (Safety check)
   └─ DedupConversationFilter

9. Side Effects (Async/Fire-and-forget)
   └─ CacheRequestInfoSideEffect
```

### 4.3 gRPC サービス

```protobuf
service ScoredPostsService {
    rpc GetScoredPosts(ScoredPostsRequest) returns (ScoredPostsResponse);
}
```

**エンドポイント**: `localhost:14534`

---

## 5. スコアリングパイプライン詳細

### 5.1 PhoenixScorer

**役割**: Grok TransformerからML予測を取得

```rust
async fn score(query, candidates) {
    // 1. Phoenix gRPC呼び出し
    let response = phoenix_client.predict(
        user_id,
        user_action_sequence,
        tweet_infos
    ).await;

    // 2. 予測マップ構築 [tweet_id -> ActionPredictions]
    let predictions_map = build_predictions_map(&response);

    // 3. 各候補にスコア割り当て
    candidates.iter().map(|c| {
        let lookup_id = c.retweeted_tweet_id.or(c.tweet_id);
        let scores = predictions_map.get(&lookup_id);
        PhoenixScores { ... }
    });
}
```

**ActionPredictions**:
- `action_probs`: 離散アクション (14種)
- `continuous_values`: 連続値 (DwellTime)

### 5.2 WeightedScorer

**役割**: アクション重み付き結合

```rust
fn compute_weighted_score(candidate: &PostCandidate) -> f64 {
    // VQV重みの条件適用
    let vqv_weight = if has_video && duration > MIN_VIDEO_DURATION_MS {
        VQV_WEIGHT
    } else {
        0.0
    };

    // 全アクション重み付き合計
    combined_score =
        favorite_score × FAVORITE_WEIGHT +
        reply_score × REPLY_WEIGHT +
        vqv_score × vqv_weight +  // 条件付き
        ...
        not_interested_score × NOT_INTERESTED_WEIGHT +  // 負
        block_author_score × BLOCK_AUTHOR_WEIGHT;

    // スコアオフセット適用
    offset_score(combined_score)
}
```

**オフセット処理**:
```rust
fn offset_score(combined_score: f64) -> f64 {
    if combined_score < 0.0 {
        // 負のスコアを正規化してシフト
        (combined_score + NEGATIVE_WEIGHTS_SUM) / WEIGHTS_SUM
            * NEGATIVE_SCORES_OFFSET
    } else {
        // 正のスコアはオフセット追加のみ
        combined_score + NEGATIVE_SCORES_OFFSET
    }
}
```

### 5.3 AuthorDiversityScorer

**役割**: 同一作者の連続出現を抑制

```rust
fn multiplier(position: usize) -> f64 {
    (1.0 - FLOOR) × DECAY_FACTOR^position + FLOOR
}
```

**例**: `FLOOR=0.5, DECAY=0.5`
```
position 0: multiplier = 1.0   (100%)
position 1: multiplier = 0.75  (75%)
position 2: multiplier = 0.625 (62.5%)
position 3: multiplier = 0.562 (56.2%)
```

**処理フロー**:
1. `weighted_score` で降順ソート
2. 各候補の作者の出現回数をカウント
3. カウント位置に応じてスコアに乗数を適用

---

## 6. フィルタリング戦略詳細

### 6.1 AgeFilter

```rust
fn is_within_age(tweet_id: i64) -> bool {
    duration_since_creation_opt(tweet_id)  // Snowflake IDから計算
        .map(|age| age <= max_age)
        .unwrap_or(false)
}
```

**Snowflake ID**: 作成日時がエンコードされているため、IDから年齢を計算可能

### 6.2 MutedKeywordFilter

**処理フロー**:
1. ユーザーのミュートキーワードをトークナイズ
2. 各候補のツイート本文をトークナイズ
3. `MatchTweetGroup` でパターンマッチング
4. マッチした候補を除外

**使用ライブラリ**: `xai_post_text`

### 6.3 VFFilter (Visibility Filter)

```rust
fn should_drop(reason: &Option<FilteredReason>) -> bool {
    match reason {
        Some(FilteredReason::SafetyResult(safety)) => {
            matches!(safety.action, Action::Drop(_))
        }
        Some(_) => true,  // その他の理由は全て除外
        None => false,     // 理由なし = 表示可能
    }
}
```

**除外理由**:
- SafetyResult: Drop
- 削除済み
- スパム
- 暴力・ゴア等
- 規約違反

---

## 7. パフォーマンス特性

### 7.1 レイテンシ

| コンポーネント | 目標レイテンシ |
|---------------|--------------|
| Thunder In-Network検索 | < 1ms |
| Phoenix Retrieval | 10-50ms |
| Phoenix Ranking | 20-100ms |
| Home Mixer全体 | 100-200ms |

### 7.2 スループット

| ステージ | スループット |
|--------|----------|
| Thunder | Max 1ms request timeout |
| Kafka | 100ms poll timeout |
| Phoenix | Batch scoring |

### 7.3 キャパシティ設計

| リソース | 設定 |
|---------|------|
| Thunder `max_concurrent_requests` | Semaphoreで制限 |
| Kafka `batch_size` | 調整可能 |
| Post Store | In-memory DashMap |

---

## 8. 設計上の意思決定

### 8.1 手作り特徴量の排除

**Before**: エンジニアが特徴量を設計 (例: 投稿長、時間帯、媒体種)

**After**: Grok Transformerに全てを学習させさせる

**利点**:
- データパイプラインの簡素化
- 新しいパターンの自動検出
- エンジニアリング工数の削減

### 8.2 候補者アイソレーション

**問題**: バッチ内の他候補に依存するとスコアが不安定

**解決**: Attention Maskで候補間のattentionを禁止

**利点**:
- スコアのキャッシュ可能
- バッチサイズ変更でスコア不変
- 理論的な一貫性

### 8.3 ハッシュベース埋め込み

**動機**: 埋め込みテーブルのサイズ削減

**仕組み**:
- 複数ハッシュ関数でユーザー/投稿/作者をハッシュ化
- ハッシュ値をインデックスとして埋め込み検索
- 複数の埋め込みを結合して投影

**利点**:
- メモリ効率
- 衝突耐性 (複数ハッシュで緩和)

---

## 9. デプロイメント構成

### 9.1 Home Mixer Cluster

- **言語**: Rust
- **プロトコル**: gRPC
- **ポート**: 14534
- **スケーリング**: Horizontal Pod Autoscaler

### 9.2 Thunder Cluster

- **言語**: Rust
- **ストレージ**: In-memory (per-user shards)
- **入力**: Kafka (tweet_events topic)
- **保持期間**: `retention_seconds` (デフォルト2日)

### 9.3 Phoenix Cluster

- **言語**: Python/JAX
- **Retrieval**: Two-Tower + ANN index
- **Ranking**: Grok Transformer on TPU/GPU
- **スケーリング**: Model runner per device

---

## 10. 主要設定パラメータ

### 10.1 Thunder

| パラメータ | デフォルト | 説明 |
|-----------|----------|------|
| `post_retention_seconds` | 172800 (2日) | 投稿保持期間 |
| `request_timeout_ms` | 0 | リクエストタイムアウト (0=無制限) |
| `max_concurrent_requests` | - | Semaphore最大値 |
| `kafka_batch_size` | - | Kafka処理バッチサイズ |
| `MAX_ORIGINAL_POSTS_PER_AUTHOR` | - | 作者別最大投稿数 |
| `MAX_REPLY_POSTS_PER_AUTHOR` | - | 作者別最大リプライ数 |

### 10.2 Phoenix

| パラメータ | サンプル値 | 説明 |
|-----------|----------|------|
| `emb_size` | 128 | 埋め込み次元 |
| `history_seq_len` | 128 | 履歴最大長 |
| `candidate_seq_len` | 32 | 候補者最大数 |
| `num_layers` | 2 | Transformer層数 |
| `num_q_heads` | 2 | クエリーヘッド数 |
| `widening_factor` | 2 | FFN拡大率 |
| `attn_output_multiplier` | 0.125 | Attention出力スケール |

### 10.3 ハッシュ設定

```rust
HashConfig {
    num_user_hashes: 2,
    num_item_hashes: 2,
    num_author_hashes: 2,
}
```

---

## 11. まとめ

X For You Feed アルゴリズムは、以下の特徴を持つ大規模レコメンデーションシステム:

1. **Rust (Thunder/Home Mixer)** で高性能オーケストレーション
2. **Python/JAX (Phoenix)** で柔軟なML推論
3. **Grok Transformer** の活用で手作り特徴量を排除
4. **候補者アイソレーション** でスコアの一貫性を保証
5. **ハッシュベース埋め込み** でメモリ効率を最適化

この設計により、数百万投稿から秒級でパーソナライズされたフィードを生成可能。

---

**参考リポジトリ**: https://github.com/xai-org/x-algorithm
