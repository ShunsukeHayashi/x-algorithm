# X-Algorithm コード追跡分析詳細

## 概要

X (Twitter) の「For You」フィードアルゴリズムにおける主要コンポーネントの実行フローを詳細に分析したドキュメントです。

## 分析対象コンポーネント

### 1. Thunder - In-Memory Post Store

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

#### insert_posts() 実行フロー

```
ユーザー入力: Vec<LightPost>
       ↓
1. フィルタリング
   • 保持期間チェック: current_time - created_at <= retention_seconds
   • 未来の投稿を除外: created_at < current_time
       ↓
2. ソ�り替え: created_at 昇順
       ↓
3. insert_posts_internal() 内部処理
       ↓
各投稿ごと:
  ├─ deleted_posts に含まれる？
  │   └─ Yes → スキップ
  │   └─ No  → 続行
  ├─ posts へ挿入 (既存の場合はスキップ)
  ├─ TinyPost 作成
  ├─ タイムライン追加
  │   ├─ is_original? → original_posts_by_user
  │   └─ else → secondary_posts_by_user
  └─ has_video? → video_posts_by_user
```

#### get_all_posts_by_users() 実行フロー

```
入力: user_ids, exclude_tweet_ids, start_time, request_user_id
       ↓
1. following_users_set 作成 (user_ids から)
       ↓
2. get_posts_from_map() - original posts
   ├─ 最大 MAX_ORIGINAL_POSTS_PER_AUTHOR まで取得
   ├─ exclude_tweet_ids で除外
   ├─ deleted_posts で除外
   ├─ 自分自身のリツイート除外
   └─ following_users_set による会話フィルタリング
       ↓
3. get_posts_from_map() - secondary posts
   ├─ 最大 MAX_REPLY_POSTS_PER_AUTHOR まで取得
   ├─ 追信元が following_users_set に含まれるかチェック
   └─ 会話チェーンをたどる
       ↓
4. 結合して返却
```

#### タイムアウト処理

```rust
// trim_old_posts() - 背景タスクで定期実行
while let Some(oldest_post) = user_posts.front() {
    if current_time - oldest_post.created_at > retention_seconds {
        user_posts.pop_front();  // 最古い投稿を削除
        posts_map.remove(&post_id); // 全投稿マップから削除
        trimmed += 1;
    } else {
        break;  // 有効期限内の投稿のみ残す
    }
}
```

---

### 2. Phoenix - ML パイプライン

**ファイル:** `phoenix/grok.py`

#### make_recsys_attn_mask() - 重要な実装

この関数は、候補候補間の独立スコアリングを実現するためのアテンションマスクを作成します。

```
シーケンス構造:
[ユーザー][履歴][履歴][候補][候補][候補]
  0      1      2      3      4      5      6

アテンションマスク:
  1. 全体は因果関係 (causal mask)
  2. 候補間同士 (positions 4-6) はお互いに attend できない (0)
  3. 候補は自分自身 (対角成分) は attend 可能 (1)

結果: 候補A はユーザー+履歴+自分自身のみを見て、候補BやCを見ない
```

**コード:**
```python
# 1. 全体の因果関係マスク
causal_mask = jnp.tril(jnp.ones((seq_len, seq_len)))

# 2. 候補間同士のアテンションをゼロに
attn_mask[:, :, candidate_start_offset:, candidate_start_offset:] = 0

# 3. 候補の自己アテンションを1に復元
candidate_indices = jnp.arange(candidate_start_offset, seq_len)
attn_mask[:, :, candidate_indices, candidate_indices] = 1
```

---

### 3. Home Mixer - Candidate Pipeline

**ファイル:** `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`

#### パイプライン構成

```rust
pub struct PhoenixCandidatePipeline {
    query_hydrators: Vec<Box<dyn QueryHydrator>>,
    sources: Vec<Box<dyn Source>>,
    hydrators: Vec<Box<dyn Hydrator>>,
    filters: Vec<Box<dyn Filter>>,
    scorers: Vec<Box<dyn Scorer>>,
    selector: TopKScoreSelector,
    post_selection_hydrators: Vec<Box<dyn Hydrator>>,
    post_selection_filters: Vec<Box<dyn Filter>>,
    side_effects: Arc<Vec<Box<dyn SideEffect>>>,
}
```

#### 9ステップ実行フロー

```
Step 1: Query Hydration (Parallel)
  ├─ UserActionSeqQueryHydrator - ユーザーア作用シーケンス取得
  └─ UserFeaturesQueryHydrator - ユーザーフィーチャー取得

Step 2: Fetch Candidates (Parallel)
  ├─ ThunderSource - In-Network 投稿取得
  └─ PhoenixSource - Out-of-Network 投稿取得

Step 3: Hydration (Parallel)
  ├─ InNetworkCandidateHydrator
  ├─ CoreDataCandidateHydrator (TES Client)
  ├─ VideoDurationCandidateHydrator
  ├─ SubscriptionHydrator
  └─ GizmoduckCandidateHydrator

Step 4: Filter (Sequential) - 10個のフィルタを順次適用
  ├─ DropDuplicatesFilter
  ├─ CoreDataHydrationFilter
  ├─ AgeFilter
  ├─ SelfTweetFilter
  ├─ RetweetDeduplicationFilter
  ├─ IneligibleSubscriptionFilter
  ├─ PreviouslySeenPostsFilter
  ├─ PreviouslyServedPostsFilter
  ├─ MutedKeywordFilter
  └─ AuthorSocialgraphFilter

Step 5: Score (Sequential)
  ├─ PhoenixScorer - ML予測 (14アクション)
  ├─ WeightedScorer - 重み付け結合
  ├─ AuthorDiversityScorer - 多様性スコア
  └─ OONScorer - OONスコア

Step 6: Selection
  └─ TopKScoreSelector - 上位K件を選択

Step 7: Post-Selection Hydration (Parallel)
  └─ VFCandidateHydrator - 可視性フィルタリング

Step 8: Post-Selection Filter (Sequential)
  ├─ VFFilter
  └─ DedupConversationFilter

Step 9: Side Effects (Async - Fire & Forget)
  └─ CacheRequestInfoSideEffect - 非同期で実行
```

---

### 4. データフロー詳細分析

#### Hash-based Embeddings (Phoenix Retrieval)

**目的:** メモリ効率のため、埋め込みベクトルをハッシュ値で間接参照

```
構造:
UserTower → Hash Functions → User Embedding (Cached)

例:
  num_user_hashes = 2
  user_id → hash1, hash2 → embedding1, embedding2

ハッシュ関数の実装 (Python/JAX):
  - ユーザーIDを複数のハッシュ関数で変換
  - 各ハッシュ値をインデックスとして埋め込みベクトルを取得
  - 複数の埋め込みベクトルを平均して最終的なユーザー埋め込みを生成
```

---

## パフォーマンス特性

### Thunder レイテンシ

| オペレーション | レイテンシ | 説明 |
|------------|----------|------|
| DashMap get | <1μs | メモリ内検索 |
| VecDeque pop_front | O(1) | 先頭要素削除 |
| 並列処理 | - | Rayonで spawn_blocking |

### Phoenix ML 推論

| ステージ | レイテンシ | 説明 |
|--------|----------|------|
| Two-Tower検索 | ~10-50ms | ANNインデックス検索 |
| Transformer scoring | ~20-100ms | Grok推論 (バッチ50-100) |
| 全体 | ~100-200ms | Home Mixer全体 |

---

## トレードオフ

1. **Candidate Isolation**: 候補同士が干渉しない独立スコアリング
2. **Progressive Disclosure**: MCPツールの段階的公開
3. **Parallel + Sequential**: パイプラインのハイブリッド実行
4. **Hash-based Caching**: 埋め込みのハッシュベース参照でメモリ効率化

---

## 読査した主要ファイル

| コンポーネント | 主要ファイル | 行数概算 |
|------------|------------|----------|
| Thunder | `thunder/posts/post_store.rs` | ~530行 |
| Thunder | `thunder/kafka/tweet_events_listener_v2.rs` | ~200行 |
| Phoenix | `phoenix/grok.py` | ~600行 |
| Home Mixer | `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` | ~200行 |
| Home Mixer | `home-mixer/scorers/phoenix_scorer.rs` | ~180行 |
| Home Mixer | `home-mixer/scorers/weighted_scorer.rs` | ~100行 |
| Home Mixer | `home-mixer/filters/*` | 10+ ファイル |

---

## 可視化ファイル

作成したPlantUML図:

1. **code_execution_flow.puml** - 全体の実行フロー図
2. **function_activity_diagram.puml** - 関数レベルのアクティビティ図

---

## 分析の次のステップ

1. **パフォーマンス最適化** - ボトルネック特定と改善
2. **A/Bテスト分析** - 異義的なアルゴリズム変更の影響
3. **エッジケース分析** - 例外処理とエラーリカバリー
4. **スケーリング分析** - 大規模トラフィック時の挙動
