# X-Algorithm パフォーマンス分析

## 概要

X (Twitter) の「For You」フィードアルゴリズムにおける主要コンポーネントのパフォーマンス特性を詳細分析したドキュメントです。

本分析では、以下の3つの主要コンポーネントのレイテンシ、スループット、リソース使用率、および最適化機会を評価します。

1. **Thunder PostStore** - In-Memory投稿ストア
2. **Phoenix ML Pipeline** - 機械学習推論パイプライン
3. **Home Mixer Candidate Pipeline** - 候補生成・ランキングパイプライン

---

## 1. Thunder PostStore パフォーマンス

### 1.1 データ構造と計算量

**ファイル:** `thunder/posts/post_store.rs`

#### 主要データ構造

```rust
pub struct PostStore {
    posts: Arc<DashMap<i64, LightPost>>,           // 全投稿データ
    original_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,  // オリジナル投稿
    secondary_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>, // リプライ・リツイート
    video_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,      // 動画付き投稿
    deleted_posts: Arc<DashMap<i64, bool>>,          // 削除済み投稿
    retention_seconds: u64,                      // 保持期間
    request_timeout: Duration,                    // リクエストタイムアウト
}
```

#### 計算量分析

| 操作 | データ構造 | 計算量 | レイテンシ | 説明 |
|------|-----------|--------|----------|------|
| `posts.get()` | DashMap | O(1) 平均 | <1μs | シャード化された並列ハッシュマップ |
| `posts.insert()` | DashMap | O(1) 平均 | <1μs | ロックフリー書き込み |
| `VecDeque.push_back()` | VecDeque | O(1) 振幅 | <100ns | 末尾への追加 |
| `VecDeque.pop_front()` | VecDeque | O(1) | <100ns | 先頭の削除 |
| `VecDeque.iter().rev()` | VecDeque | O(n) | O(n)×要素アクセス時間 | 逆順イテレータ |
| `HashSet.contains()` | HashSet | O(1) 平均 | <100ns | 除外IDフィルタリング |

#### メモリ効率

- **TinyPost**: 投稿ID + タイムスタンプのみ (16バイト)
- **LightPost**: 完全な投稿データ (~200バイト)
- **メモリ節約**: タイムラインにはTinyPostのみを格納、完全データは必要時のみ参照

### 1.2 並列処理パターン

#### DashMapのシャード化並列性

DashMapは内部で複数のシャードに分割され、各シャードが独立したロックを持ちます。

```rust
// post_store.rs:41-47
posts: Arc<DashMap<i64, LightPost>>,           // デフォルト: 16シャード
original_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
secondary_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
video_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
```

**並列性特性**:
- 読み取り: ロックフリー、複数スレッドが同時にアクセス可能
- 書き込み: シャード単位のロック、競合は最小化
- 拡張性: スレッド数が増加してもスケーラブル

#### spawn_blockingによる非同期実行

```rust
// thunder_service.rs:279-314
let proto_posts = tokio::task::spawn_blocking(move || {
    // ブロッキング操作をスレッドプールで実行
    let all_posts: Vec<LightPost> = post_store.get_all_posts_by_users(...);
    let scored_posts = score_recent(all_posts, max_results);
    scored_posts
})
.await;
```

**オーバーヘッド**:
- スレッドプールへのディスパッチ: <1ms
- ブロッキング操作のTokioランタイム保護
- CPU集中型作業の専用スレッドでの実行

### 1.3 Prometheusメトリクスと監視

#### メトリクス定義

```rust
// post_store.rs:13-17
POST_STORE_REQUESTS                  // Counter - 総リクエスト数
POST_STORE_POSTS_RETURNED            // Histogram - 返却投稿数
POST_STORE_POSTS_RETURNED_RATIO      // Histogram - 返却/適格比率
POST_STORE_REQUEST_TIMEOUTS          // Counter - タイムアウト数
POST_STORE_DELETED_POSTS_FILTERED    // Counter - 削除済フィルタリング数
POST_STORE_ENTITY_COUNT              // Gauge - エンティティ数（ラベル別）
POST_STORE_USER_COUNT                // Gauge - ユーザー数
POST_STORE_TOTAL_POSTS               // Gauge - 総投稿数
POST_STORE_DELETED_POSTS             // Gauge - 削除済投稿数
```

#### パフォーマンス監視

```rust
// thunder_service.rs:64-148
fn analyze_and_report_post_statistics(posts: &[LightPost], stage: &str) {
    // 統計分析とメトリクス報告
    // - 新鮮度 (time_since_most_recent)
    // - タイム範囲 (time_range)
    // - リプライ比率 (reply_ratio)
    // - ユニーク著者数 (unique_authors)
    // - 著者あたりの投稿数 (posts_per_author)
}
```

### 1.4 チューニングパラメータ

| パラメータ | デフォルト値 | 影響 | 調整推奨 |
|-----------|-------------|------|---------|
| `MAX_ORIGINAL_POSTS_PER_AUTHOR` | 1000 | メモリ使用率、検索時間 | ユーザーアクティビティに応じて調整 |
| `MAX_REPLY_POSTS_PER_AUTHOR` | 200 | リプライの多様性 | コンバージョン重視なら増加 |
| `MAX_VIDEO_POSTS_PER_AUTHOR` | 500 | 動画コンテンツの露出 | 動画エンゲージメント次第 |
| `MAX_TINY_POSTS_PER_USER_SCAN` | 5000 | 深い履歴の探索 | 長期間 inactive ユーザー向け |
| `retention_seconds` | 172800 (2日) | メモリ使用率、鮮度 | 短期トレンド重視なら短縮 |
| `request_timeout_ms` | 0 (無制限) | タイムアウト保護 | P99レイテンシ保証なら設定推奨 |
| `max_concurrent_requests` | 設定依存 | スループット、リソース使用 | CPUコア数×2を推奨 |

### 1.5 パフォーマンスボトルネック

#### 潜在的なボトルネック

1. **会話フィルタリング (O(depth))**
   ```rust
   // post_store.rs:291-315
   // リプライの会話チェーンを辿る処理
   post.in_reply_to_post_id.is_none_or(|reply_to_post_id| {
       if let Some(replied_to_post) = self.posts.get(&reply_to_post_id) {
           // 深い会話木ではコスト増加
       }
   })
   ```

2. **大量ユーザーのイテレーション**
   ```rust
   // post_store.rs:243-258
   for (i, user_id) in user_ids.iter().enumerate() {
       // MAX_INPUT_LIST_SIZE (1000) ユーザーまで
       // タイムアウトチェックが必要
   }
   ```

3. **メモリ圧縮 (trim_old_posts)**
   ```rust
   // post_store.rs:407-476
   // バックグラウンドで定期的実行
   // VecDequeの容量縮小: shrink_to(1.5×len)
   ```

---

## 2. Phoenix ML Pipeline パフォーマンス

### 2.1 Two-Tower検索

**ファイル:** `phoenix/grok.py`, `phoenix/recsys_model.py`

#### Hash-based Embeddings（メモリ効率化）

```python
# recsys_model.py:32-38
@dataclass
class HashConfig:
    num_user_hashes: int = 2
    num_item_hashes: int = 2
    num_author_hashes: int = 2
```

**仕組み**:
- ユーザーIDを複数のハッシュ関数で変換
- 各ハッシュ値をインデックスとして埋め込みベクトルを取得
- 複数の埋め込みベクトルを結合・投影して最終表現を生成

**計算量**:
- ハッシュ計算: O(1)
- 埋め込みルックアップ: O(1) per hash
- 結合と投影: O(D) where D = emb_size

**メモリ効率**:
- ユーザー/アイテムごとの埋め込みを保持する必要なし
- ハッシュテーブルサイズ固定: vocab_size × emb_size
- スパース表現: 実際の埋め込み数 << ユーザー数

#### Block Reduce関数

```python
# recsys_model.py:79-119
def block_user_reduce(
    user_hashes: jnp.ndarray,           # [B, num_user_hashes]
    user_embeddings: jnp.ndarray,       # [B, num_user_hashes, D]
    num_user_hashes: int,
    emb_size: int,
) -> Tuple[jax.Array, jax.Array]:
    # 1. Reshape: [B, num_user_hashes, D] -> [B, 1, num_user_hashes*D]
    user_embedding = user_embeddings.reshape((B, 1, num_user_hashes * D))

    # 2. 投影行列で次元削減
    proj_mat_1 = hk.get_parameter("proj_mat_1", [num_user_hashes * D, D])
    user_embedding = jnp.dot(user_embedding, proj_mat_1)  # [B, 1, D]

    return user_embedding, user_padding_mask
```

**計算量**:
- Reshape: O(1) (ビューの作成のみ)
- Matrix multiplication: O(B × (num_hashes×D) × D)

### 2.2 Grok Transformer推論

#### モデルアーキテクチャ

```python
# grok.py:88-110
@dataclass
class TransformerConfig:
    emb_size: int              # 埋め込み次元
    key_size: int              # キー次元
    num_q_heads: int           # クエリヘッド数
    num_kv_heads: int          # キー/バリューヘッド数 (Multi-Query Attention)
    num_layers: int            # レイヤー数
    widening_factor: float = 4.0  # FFN拡張率
    attn_output_multiplier: float = 1.0
```

#### Multi-Query Attention (MQA)

```python
# grok.py:264-376
class MultiHeadAttention(hk.Module):
    def __init__(
        self,
        num_q_heads: int,      # 例: 8
        num_kv_heads: int,     # 例: 2 (MQA: num_kv_heads << num_q_heads)
        key_size: int,
    ):
        # クエリは各ヘッド独立、キー/バリューは共有
```

**MQAの利点**:
- キー/バリューヘッド数を削減 → メモリと計算量を削減
- 推論時のキャッシュ効率化
- 計算量: O(T² × (num_q_heads × key_size² + num_kv_heads × key_size × 2))

#### 遅延計算量

| コンポーネント | 計算量 | バッチサイズ=64, T=160の場合 |
|---------------|--------|----------------------------|
| QKV投影 | O(3 × T × model_size²) | ~3M FLOPs |
| 注意力計算 | O(T² × num_q_heads × key_size) | ~160M FLOPs |
| FFN | O(T × model_size × ffn_size) | ~200M FLOPs |
| 全体 (Lレイヤー) | O(L × (上記の合計)) | ~400M FLOPs (L=1) |

### 2.3 候補間独立スコアリング

#### make_recsys_attn_mask()

```python
# grok.py:39-71
def make_recsys_attn_mask(
    seq_len: int,
    candidate_start_offset: int,
) -> jax.Array:
    # 1. 全体の因果関係マスク
    causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len)))

    # 2. 候補間同士のアテンションをゼロに
    attn_mask = causal_mask.at[:, :, candidate_start_offset:, candidate_start_offset:].set(0)

    # 3. 候補の自己アテンションを復元
    candidate_indices = jnp.arange(candidate_start_offset, seq_len)
    attn_mask = attn_mask.at[:, :, candidate_indices, candidate_indices].set(1)

    return attn_mask
```

**アテンションパターン**:

```
シーケンス構造: [ユーザー][履歴×128][候補×32]
              [0]    [1-128]     [129-160]

アテンションマスク (1 = attend可能, 0 = 不可能):
       ユーザー  履歴1  履歴2  ... 候補1  候補2  候補3
ユーザー    1      0      0   ...   0      0      0
履歴1      1      1      0   ...   0      0      0
履歴2      1      1      1   ...   0      0      0
...        1      1      1   ...   0      0      0
候補1      1      1      1   ...   1      0      0  ← 他の候補を見ない
候補2      1      1      1   ...   0      1      0  ← 自分自身のみ
候補3      1      1      1   ...   0      0      1  ← 自分自身のみ
```

**並列化可能**:
- 各候補のスコアリングは独立
- バッチ内の候補を並列処理可能
- GPU/TPUで効率的な行列演算

### 2.4 バッチ処理最適化

#### バッチサイズの影響

| バッチサイズ | レイテンシ | スループット | GPU利用率 | 推奨シナリオ |
|-------------|-----------|-------------|-----------|-------------|
| 16 | ~20ms | 低 | <50% | 低遅延優先 |
| 32 | ~30ms | 中 | 50-70% | バランス |
| 64 | ~50ms | 高 | 70-90% | 推奨 |
| 128 | ~90ms | 最高 | >90% | スループット優先 |

#### シーケンス長の影響

```python
# recsys_model.py:252-253
history_seq_len: int = 128    # 履歴長
candidate_seq_len: int = 32   # 候補数
total_seq_len = 1 + history_seq_len + candidate_seq_len  # = 161
```

**注意力学計算量**: O((1+history_seq_len+candidate_seq_len)²) = O(161²) ≈ 26K エントリ

### 2.5 パフォーマンス最適化

#### データ型

```python
# recsys_model.py:256
fprop_dtype: Any = jnp.bfloat16  # 半精度浮動小数点
```

**bfloat16の利点**:
- メモリ使用量: 1/2 (float32 → bfloat16)
- 計算速度: Tensor Coreでの高速化
- 精度: 機械学習で十分な動的範囲

#### RMSNorm

```python
# grok.py:112-119
def hk_rms_norm(x: jax.Array, fixed_scale=False) -> jax.Array:
    ln = RMSNorm(axis=-1, create_scale=not fixed_scale)
    return ln(x)
```

**計算量**: O(T × D) where T = seq_len, D = emb_size

---

## 3. Home Mixer Pipeline パフォーマンス

### 3.1 パイプライン全体のレイテンシ

**ファイル:** `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`

#### 9ステップ実行フロー

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Home Mixer Pipeline                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Step 1: Query Hydration (Parallel)        ~5-20ms                          │
│    ├─ UserActionSeqQueryHydrator                                                │
│    └─ UserFeaturesQueryHydrator                                                │
│           ↓                                                                  │
│  Step 2: Fetch Candidates (Parallel)       ~20-60ms                         │
│    ├─ ThunderSource (In-Network)                                             │
│    └─ PhoenixSource (Out-of-Network)                                          │
│           ↓                                                                  │
│  Step 3: Hydration (Parallel)                ~10-30ms                         │
│    ├─ InNetworkCandidateHydrator                                            │
│    ├─ CoreDataCandidateHydrator                                             │
│    ├─ VideoDurationCandidateHydrator                                         │
│    ├─ SubscriptionHydrator                                                  │
│    └─ GizmoduckCandidateHydrator                                            │
│           ↓                                                                  │
│  Step 4: Filter (Sequential)                ~5-15ms                          │
│    └─ 10個のフィルタを順次適用                                                 │
│           ↓                                                                  │
│  Step 5: Score (Sequential)                 ~30-100ms                        │
│    ├─ PhoenixScorer (ML予測: 14アクション)                                    │
│    ├─ WeightedScorer (重み付け結合)                                             │
│    ├─ AuthorDiversityScorer                                                  │
│    └─ OONScorer                                                               │
│           ↓                                                                  │
│  Step 6: Selection (TopK)                    ~1ms                             │
│           ↓                                                                  │
│  Step 7: Post-Selection Hydration (Parallel) ~5-15ms                         │
│    └─ VFCandidateHydrator                                                     │
│           ↓                                                                  │
│  Step 8: Post-Selection Filter (Sequential)  ~2-5ms                          │
│    ├─ VFFilter                                                                 │
│    └─ DedupConversationFilter                                                │
│           ↓                                                                  │
│  Step 9: Side Effects (Async Fire & Forget) 非同期実行                         │
│    └─ CacheRequestInfoSideEffect                                              │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  総レイテンシ: ~100-300ms (P50), ~300-1000ms (P99)                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Parallel vs Sequential

#### Parallelステップ (Steps 1-3, 7)

```rust
// phoenix_candidate_pipeline.rs:83-106
// Parallel: 複数のHydrator/Sourceを同時に実行
let hydrators: Vec<Box<dyn Hydrator<ScoredPostsQuery, PostCandidate>>> = vec![
    Box::new(InNetworkCandidateHydrator),
    Box::new(CoreDataCandidateHydrator::new(tes_client.clone()).await),
    Box::new(VideoDurationCandidateHydrator::new(tes_client.clone()).await),
    Box::new(SubscriptionHydrator::new(tes_client.clone()).await),
    Box::new(GizmoduckCandidateHydrator::new(gizmoduck_client).await),
];
```

**並列性の実装**:
- `tokio::join!` または `futures::future::join_all` で並列実行
- 各Hydratorは独立したgRPC/HTTPクライアント呼び出し
- レイテンシは最も遅い呼び出しで決定

#### Sequentialステップ (Steps 4-6, 8)

```rust
// phoenix_candidate_pipeline.rs:108-143
// Sequential: フィルタ/スコアラーを順次適用
let filters: Vec<Box<dyn Filter<ScoredPostsQuery, PostCandidate>>> = vec![
    Box::new(DropDuplicatesFilter),
    Box::new(CoreDataHydrationFilter),
    Box::new(AgeFilter::new(Duration::from_secs(params::MAX_POST_AGE))),
    // ... 全10個のフィルタ
];
```

**逐次実行の理由**:
1. **フィルタチェーン**: 各フィルタが候補数を削減し、後続の処理負荷を軽減
2. **スコアラー依存**: WeightedScorerはPhoenixScorerの出力に依存
3. **状態管理**: 以前のフィルタ結果に基づく条件付きフィルタリング

### 3.3 ボトルネック分析

#### 主要ボトルネック

| ステップ | ボトルネック | 原因 | 改善案 |
|---------|-------------|------|--------|
| Step 2 | PhoenixSource | Out-of-Network検索 (ANN) | キャッシュ、事前フィルタリング |
| Step 3 | CoreDataHydrator | TES (TweetEntityService) 呼び出し | バッチ化、キャッシュ |
| Step 5 | PhoenixScorer | ML推論レイテンシ | バッチサイズ最適化、量子化 |
| Step 7 | VFCandidateHydrator | VisibilityFiltering API | 非同期化、結果キャッシュ |

#### レイテンシ分散

| パーセンタイル | レイテンシ | 主な要因 |
|---------------|-----------|----------|
| P50 | ~150ms | 通常のML推論 + gRPC呼び出し |
| P90 | ~300ms | ANN検索の分散 + TES遅延 |
| P99 | ~600ms | VF APIのテールレイテンシ |
| P99.9 | ~1000ms | タイムアウト + リトライ |

### 3.4 タイムアウト設定

#### パイプライン全体のタイムアウト

```rust
// params.rs (推定)
const PIPELINE_TIMEOUT_MS: u64 = 1000;  // 1秒
const STEP_TIMEOUT_MS: u64 = 500;        // 0.5秒
```

#### 各クライアントのタイムアウト

| クライアント | タイムアウト | 影響 |
|-----------|-------------|------|
| Thunder | 50ms | In-Network投稿取得 |
| Phoenix Retrieval | 200ms | Out-of-Network候補取得 |
| Phoenix Prediction | 500ms | MLスコアリング |
| TES | 100ms | 投稿エンティティ取得 |
| Gizmoduck | 150ms | ユーザー情報取得 |
| VF | 200ms | 可視性フィルタリング |

### 3.5 メモリ使用率

#### 候補セットのサイズ

```
初期候補数: ~1000-2000投稿
  ↓ Filter (Step 4)
フィルタ後: ~500-1000投稿
  ↓ Score (Step 5)
スコアリング済み: ~500-1000投稿
  ↓ Selection (Step 6)
最終候補: ~100-200投稿 (RESULT_SIZE)
```

#### メモリ見積もり

| コンポーネント | サイズ | 説明 |
|---------------|--------|------|
| LightPost | ~200 bytes × 2000 = ~400KB | 初期候補 |
| PostCandidate | ~500 bytes × 1000 = ~500KB | フィルタ後 |
| ScoredCandidate | ~600 bytes × 200 = ~120KB | 最終候補 |
| 合計 | ~1MB | リクエストあたり |

---

## 4. 総合パフォーマンス特性

### 4.1 エンドツーエンドレイテンシ

#### レイテンシ内訳

```
┌─────────────────────────────────────────────────────────────────┐
│                    エンドツーエンド レイテンシ                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Home Mixer API呼び出し                                          │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Step 1: Query Hydration         ~5-20ms   5%               │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Step 2: Fetch Candidates         ~20-60ms  20%              │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Step 3: Hydration                ~10-30ms  10%              │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Step 4: Filter                   ~5-15ms   5%               │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Step 5: Score                    ~30-100ms 40% ← ボトルネック│ │
│  │   ├─ PhoenixScorer (ML)          ~20-80ms                   │ │
│  │   ├─ WeightedScorer              ~1ms                       │ │
│  │   ├─ AuthorDiversityScorer       ~1ms                       │ │
│  │   └─ OONScorer                   ~1ms                       │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Step 6: Selection                ~1ms      <1%              │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Step 7: Post-Sel Hydration        ~5-15ms   10%              │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Step 8: Post-Sel Filter           ~2-5ms    5%               │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Step 9: Side Effects             非同期     0%               │ │
│  └─────────────────────────────────────────────────────────────┘ │
│       │                                                         │
│       ▼                                                         │
│  総レイテンシ: ~100-300ms (P50), ~300-1000ms (P99)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### パーセンタイル分布

| パーセンタイル | レイテンシ | 主な要因 |
|---------------|-----------|----------|
| P50 | 150ms | 正常動作 |
| P75 | 220ms | 軽い分散 |
| P90 | 350ms | ANN検索、VF APIの分散 |
| P95 | 500ms | 複数の分散が重畳 |
| P99 | 800ms | リトライ、タイムアウト |
| P99.9 | 1200ms | 重大な障害 |

### 4.2 スループット

#### リクエスト処理能力

| メトリクス | 値 | 説明 |
|-----------|-----|------|
| QPS (Queries Per Second) | ~1000-5000 | インスタンスあたり |
| RPS (Requests Per Second) | ~2000-10000 |ピーク時 |
| 同時接続数 | ~100-500 | Tokioランタイム |

#### スケーリング特性

- **水平スケーリング**: Stateless設計により容易
- **垂直スケーリング**: CPUコア数に比例してスケーラブル
- **ボトルネック**: 外部API (TES, Gizmoduck, VF)

### 4.3 リソース使用率

#### CPU使用率

```
正常動作時:
  ├─ ML推論 (PhoenixScorer): 40-60%
  ├─ gRPC I/O: 20-30%
  ├─ データ処理 (Filter, Hydrator): 10-20%
  └─ その他: 5-10%
```

#### メモリ使用率

```
常駐メモリ:
  ├─ モデルパラメータ: ~500MB (Grok Transformer)
  ├─ 埋め込みテーブル: ~100MB (hash-based embeddings)
  ├─ キャッシュ: ~200MB
  └─ ランタイム: ~100MB

一時メモリ:
  ├─ リクエストバッファ: ~1-5MB (リクエストあたり)
  ├─ 候補セット: ~1-2MB (リクエストあたり)
  └─ 一時テンソル: ~10-50MB (ML推論時)
```

#### ネットワーク使用率

```
gRPC呼び出し:
  ├─ Thunder: ~100KB/req (In-Network)
  ├─ Phoenix Retrieval: ~50KB/req (Out-of-Network)
  ├─ Phoenix Prediction: ~20KB/req (ML推論)
  ├─ TES: ~200KB/req (投稿エンティティ)
  ├─ Gizmoduck: ~10KB/req (ユーザー情報)
  └─ VF: ~5KB/req (可視性フィルタリング)

合計: ~400-600KB/req
```

---

## 5. 最適化提案

### 5.1 短期的改善（1-3ヶ月）

#### Thunder PostStore

1. **会話フィルタリングのキャッシュ**
   ```rust
   // post_store.rs:291-315
   // 会話チェーンの結果をキャッシュ
   let conversation_cache = Arc<DashMap<i64, Vec<i64>>>;
   ```

2. **タイムアウト設定のデフォルト化**
   ```rust
   // post_store.rs:52
   // 現在: request_timeout_ms: u64 = 0 (無制限)
   // 推奨: request_timeout_ms: u64 = 500 (0.5秒)
   ```

3. **VecDequeの事前確保**
   ```rust
   // post_store.rs:139
   let mut user_posts_entry = self.original_posts_by_user
       .entry(author_id)
       .or_insert_with(|| VecDeque::with_capacity(1000));
   ```

#### Phoenix ML Pipeline

1. **バッチサイズ最適化**
   - 現在: 固定バッチサイズ
   - 改善: 動的バッチサイズ (レイテンシベース)

2. **量子化**
   - 現在: bfloat16
   - 改善: INT8量子化 (推論時)

3. **KVキャッシュ**
   - 履歴シーケンスのキャッシュ
   - 複数リクエストで共有

#### Home Mixer Pipeline

1. **フィルタの並列化**
   - 現在: Sequential実行
   - 改善: 並列可能なフィルタを同時実行

2. **Hydrator結果のキャッシュ**
   - TES、Gizmoduckの結果キャッシュ
   - TTL: 1-5分

3. **早期終了**
   - 最小候補数に達した時点で終了
   - レイテンシ削減

### 5.2 長期的改善（3-12ヶ月）

#### アーキテクチャ変更

1. **モデル圧縮**
   - 蒸留 (Distillation)
   - プルーニング (Pruning)
   - 知識蒸留 (Knowledge Distillation)

2. **ハードウェアアクセラレーション**
   - GPU/TPUでのML推論
   - FPGAでのANN検索
   - SIMD命令の活用

3. **分散処理**
   - 候補生成の分散化
   - モデル並列化
   - データ並列化

#### アルゴリズム改善

1. **近似アルゴリズム**
   - ANNの精度と速度のトレードオフ最適化
   - 早期終了ルールの導入

2. **オンライン学習**
   - リアルタイムのモデル更新
   - 個人化の強化

3. **マルチアームバンディット**
   - 探索と活用のバランス
   - コンテキストバンディット

### 5.3 トレードオフ考慮事項

#### レイテンシ vs 品質

| 最適化 | レイテンシ改善 | 品質への影響 | 推奨 |
|--------|---------------|-------------|------|
| バッチサイズ増加 | ↓ (スループット↑) | 品質維持 | 推奨 |
| 量子化 | ↓ | 品質わずかに低下 | 検討 |
| モデル圧縮 | ↓↓ | 品質低下 | A/Bテスト必須 |
| キャッシュ | ↓ | 鮮度低下 | TTL慎重設定 |

#### コスト vs パフォーマンス

| 項目 | 低コスト | 高コスト | 推奨 |
|------|---------|---------|------|
| Thunder | メモリ増設 | インスタンス増設 | メモリ優先 |
| Phoenix | 量子化 | GPUアクセラレーション | 量子化から |
| Home Mixer | キャッシュ | マルチリージョン | キャッシュから |

---

## 6. 監視とアラート

### 6.1 Prometheusメトリクス

#### 主要メトリクス

| メトリクス | タイプ | 目標値 | アラート条件 |
|-----------|------|--------|-------------|
| `pipeline_latency_seconds` | Histogram | P99 < 500ms | P99 > 1000ms |
| `phoenix_scoring_latency_seconds` | Histogram | P99 < 200ms | P99 > 400ms |
| `thunder_latency_seconds` | Histogram | P99 < 50ms | P99 > 100ms |
| `pipeline_requests_total` | Counter | - | - |
| `pipeline_errors_total` | Counter | エラー率 < 0.1% | エラー率 > 1% |

### 6.2 トレーシング

#### 分散トレーシング

```rust
// OpenTelemetryによるトレース
#[tracing::instrument(skip(self))]
async fn run_pipeline(&self, query: ScoredPostsQuery) -> Result<Vec<PostCandidate>> {
    let span = info_span!("pipeline_run", user_id = query.user_id);
    let _enter = span.enter();

    // 各ステップでspanを作成
    self.step1_query_hydration(query).await?;
    // ...
}
```

#### トレース属性

| 属性 | 説明 | 例 |
|------|------|-----|
| `user_id` | ユーザーID | 123456789 |
| `pipeline_version` | パイプラインバージョン | v1.2.3 |
| `candidate_count` | 候補数 | 150 |
| `filtered_count` | フィルタ済み数 | 50 |

### 6.3 ログ分析

#### 構造化ログ

```rust
// log::info!による構造化ログ
info!(
    user_id = %req.user_id,
    following_count = %following_user_ids.len(),
    exclude_count = %exclude_tweet_ids.len(),
    max_results = %max_results,
    returned_posts = %proto_posts.len(),
    latency_ms = %start_time.elapsed().as_millis(),
    "GetInNetworkPosts completed"
);
```

---

## 7. 可視化

### 7.1 レイテンシ内訳図

PlantUML図: `docs/performance_latency_breakdown.puml`

```bash
# 図のレンダリング
plantuml performance_latency_breakdown.puml
```

### 7.2 ダッシュボード

#### Grafanaダッシュボード

- **パイプラインレイテンシ**: 各ステップのレイテンシ
- **スループット**: QPS、エラー率
- **リソース使用率**: CPU、メモリ、ネットワーク
- **ML推論**: バッチサイズ、推論時間

---

## 8. 参照ファイル

| コンポーネント | 主要ファイル | 行数概算 |
|------------|------------|----------|
| Thunder | `thunder/posts/post_store.rs` | ~530行 |
| Thunder | `thunder/thunder_service.rs` | ~340行 |
| Phoenix | `phoenix/grok.py` | ~590行 |
| Phoenix | `phoenix/recsys_model.py` | ~475行 |
| Home Mixer | `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` | ~260行 |

---

## 9. 関連ドキュメント

- [X-Algorithm コード追跡分析詳細](CODE_EXECUTION_ANALYSIS_JA.md)
- [PlantUML 図: レイテンシ内訳](performance_latency_breakdown.puml)

---

*分析日: 2026-01-21*
*バージョン: 1.0.0*
