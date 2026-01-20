# X For You Feed Algorithm - 完全理解ガイド

**読者対象**: エンジニア、データサイエンティスト、MLエンジニア
**前提知識**: Rust/Pythonの基礎、機械学習/推薦システムの基礎

---

## はじめに - このアルゴリズムが解決する問題

X（旧Twitter）の「For You」フィードには、毎分膨大な数の投稿がされています。

**問題の規模**:
- 1秒間に数千件の新規投稿
- 各ユーザーは数千〜数万人をフォロー
- 表示可能な投稿は数千万〜数億件

**解決すべき課題**:
1. **候補取得**: 数億件から数百件をどう選ぶ？
2. **ランキング**: どの投稿を優先するか？
3. **パフォーマンス**: 100〜200msで応答するには？

この文書では、xAIが公開した「For You」アルゴリズムを、**初心者でも完全に理解できるレベル**で解説します。

---

## 第1章: システム全体像 - まずは全体を見る

### 1.1 リクエストからレスポンスまでの流れ

ユーザーがXアプリを開いて「For You」タブをタップしたとき、何が起こるか？

```
┌────────────────────────────────────────────────────────────────────┐
│ 1. ユーザー操作                                                       │
│    - アプリを開く                                                       │
│    - "For You" タップ                                                   │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 2. gRPC リクエスト                                                    │
│    home_mixer_server:GetScoredPosts(user_id, following_ids, ...)   │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 3. Home Mixer (Rust) - 指揮者                                      │
│    ├─ Query Hydration   → ユーザー情報取得                           │
│    ├─ Candidate Sources → 候補投稿収集                               │
│    │   ├─ Thunder        → フォロー中の投稿 (In-Network)         │
│    │   └─ Phoenix        → ML検索した投稿 (Out-of-Network)     │
│    ├─ Hydrators        → 投稿データ補強                               │
│    ├─ Filters          → 不適切投稿除外                               │
│    ├─ Scorers          → MLスコア計算                                 │
│    └─ Selector         → Top-K選択                                   │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ 4. レスポンス                                                         │
│    ranked_posts[] (昇順ソート済み)                                  │
└────────────────────────────────────────────────────────────────────┘
```

### 1.2 タイムラインによる詳細フロー

```
ユーザーリクエスト (t=0ms)
    │
    ├─────────────────────────────────────────────────────────────┤
    │ Phase 1: Query Hydration (並列実行, ~5-10ms)                │
    ├─────────────────────────────────────────────────────────────┤
    │ • UserActionSeqQueryHydrator                                   │
    │   └→  最近のエンゲージメント履歴取得                             │
    │ • UserFeaturesQueryHydrator                                    │
    │   └→ フォローリスト、ミュート済みキーワード等取得                    │
    ├─────────────────────────────────────────────────────────────┤
    │ Phase 2: Candidate Sourcing (並列実行, ~10-30ms)              │
    ├─────────────────────────────────────────────────────────────┤
    │ • ThunderSource (In-Network)                                  │
    │   └→ get_in_network_posts() 呼び出し: ~1ms                     │
    │       ├─ フォロー中ユーザーの最新投稿                                  │
    │       ├─ オリジナル投稿 + リプライ/リポスト                          │
    │       └─ 動画投稿も別途取得                                          │
    │                                                              │
    │ • PhoenixSource (Out-of-Network)                             │
    │   └→ retrieve() 呼び出し: ~10-50ms                               │
    │       ├─ Two-Towerモデルでユーザー埋め込み生成                       │
    │       ├─ ANN検索で全球コーパスから探索                                  │
    │       └─ 関連度の高いTop-N候補を取得                                 │
    ├─────────────────────────────────────────────────────────────┤
    │ Phase 3: Candidate Hydration (並列実行, ~10-20ms)             │
    ├─────────────────────────────────────────────────────────────┤
    │ • CoreDataCandidateHydrator                                    │
    │   └→ 投稿本文、メディアエンティティ等                                │
    │ • GizmoduckHydrator                                             │
    │   └→ 作者情報(ユーザー名、認証ステータス等)                         │
    │ • VideoDurationHydrator                                          │
    │   └→ 動画再生時間                                                   │
    │ • ...                                                           │
    ├─────────────────────────────────────────────────────────────┤
    │ Phase 4: Pre-Scoring Filtering (逐次実行, ~5-10ms)          │
    ├─────────────────────────────────────────────────────────────┤
    │ • DropDuplicatesFilter                                         │
    │   └→ 重複投稿ID除外                                               │
    │ • AgeFilter                                                     │
    │   └→ 古すぎる投稿除外 (Snowflake IDから年齢計算)                    │
    │ • SelfTweetFilter                                               │
    │   └→ 自分の投稿除外                                               │
    │ • MutedKeywordFilter                                            │
    │   └→ ミュート済みキーワード含む投稿除外                           │
    │ • ...                                                           │
    │ ※ 各フィルタは前のフィルタの出力全てに対して適用                  │
    ├─────────────────────────────────────────────────────────────┤
    │ Phase 5: Scoring (逐次実行, ~20-100ms)                          │
    ├─────────────────────────────────────────────────────────────┤
    │ • PhoenixScorer                                                 │
    │   └→ Phoenix gRPC呼び出し                                         │
    │       ├─ Grok Transformerで14アクション確率予測                       │
    │       └─ P(favorite), P(reply), P(retweet), ...                     │
    │                                                              │
    │ • WeightedScorer                                                │
    │   └→ weighted_score = Σ(weight × P(action))                      │
    │       ├─ 正アクションは正の重み                                        │
    │       └─ 負アクション(not_interested, block等)は負の重み              │
    │                                                              │
    │ • AuthorDiversityScorer                                        │
    │   └─ 同一作者の連続出現を抑制                                        │
    │   └─ score × (0.5 × 0.5^position + 0.5)                           │
    │                                                              │
    │ • OONScorer                                                     │
    │   └─ Out-of-Network投稿への調整                                      │
    ├─────────────────────────────────────────────────────────────┤
    │ Phase 6: Selection (~1ms)                                        │
    ├─────────────────────────────────────────────────────────────┤
    │ • TopKScoreSelector                                            │
    │   └→ スコア降順ソートしてTop-K件を選択                           │
    │                                                              │
    │ Phase 7: Post-Selection Filtering (逐次実行, ~5ms)           │
    ├─────────────────────────────────────────────────────────────┤
    │ • VFFilter                                                      │
    │   └→ Visibility Filteringサービスで安全性チェック                    │
    │       └─ 削除、スパム、暴力等の投稿除外                              │
    │ • DedupConversationFilter                                       │
    │   └─ 同一スレッドの重複除外                                         │
    ├─────────────────────────────────────────────────────────────┤
    │ Phase 8: Side Effects (非同期/Fire-and-Forget)                  │
    ├─────────────────────────────────────────────────────────────┤
    │ • CacheRequestInfoSideEffect                                    │
    │   └→ 次回リクエスト用にキャッシュ                                    │
    └─────────────────────────────────────────────────────────────┘
                              ↓
                        レスポンス返却 (Total: 100-200ms)
```

---

## 第2章: Thunder - フォロー中投稿を1ミリ秒で取得

### 2.1 Thunderの役割と重要性

**Thunder**は「In-Network（ネットワーク内）」投稿の高速検索エンジンです。

**なぜ高速なのか？**
- 全ての投稿を**メモリ**に保持
- データベースへの問い合わせなし
- ユーザー別にインデックス済み

**イメージ図**:
```
Kafka Topic: tweet_events
    ↓
    ┌─────────────────────────────────────────────────────────────┐
    │ Thunder Kafka Consumers (複数スレッドで並列処理)              │
    │                                                              │
    │  Thread 1: Partitions 0-99                                     │
    │  Thread 2: Partitions 100-199                                   │
    │  Thread 3: Partitions 200-299                                   │
    │  ...                                                          │
    │                                                              │
    │  処理内容:                                                    │
    │  1. Kafkaからポーリング                                          │
     2. Protobufデシリアライズ                                          │
    │  3. PostStore.insert_posts()                                   │
    │  4. Offsetコミット                                               │
    └─────────────────────────────────────────────────────────────┘
                              ↓
    ┌─────────────────────────────────────────────────────────────┐
    │              PostStore (In-Memory Data Store)                │
    │  ┌──────────────────────────────────────────────────────┐   │
    │  │ DashMap<i64, LightPost> posts                              │   │
    │  │  投稿ID → 完全投稿データ                                      │   │
    │  └──────────────────────────────────────────────────────┘   │
    │  ┌──────────────────────────────────────────────────────┐   │
    │  │ DashMap<i64, VecDeque<TinyPost>>                        │   │
    │  │                                                            │   │
    │  │ original_posts_by_user                                    │   │
    │  │   [user_id] → [post_id, created_at, ...] (newest first) │   │
    │  │                                                            │   │
    │  │ secondary_posts_by_user                                   │   │
    │  │   [user_id] → [post_id, created_at, ...] (newest first) │   │
    │  │                                                            │   │
    │  │ video_posts_by_user                                       │   │
    │  │   [user_id] → [post_id, created_at, ...] (newest first) │   │
    │  └──────────────────────────────────────────────────────┘   │
    └─────────────────────────────────────────────────────────────┘
```

### 2.2 PostStoreデータ構造の詳細解説

#### LightPost (完全投稿データ)

```rust
pub struct LightPost {
    pub post_id: i64,              // 投稿ID
    pub author_id: i64,            // 作者ID
    pub created_at: i64,           // 作成日時 (Unix秒)
    pub in_reply_to_post_id: Option<i64>,   // リプライ元投稿ID
    pub in_reply_to_user_id: Option<i64>,   // リプライ元ユーザーID
    pub is_retweet: bool,          // リツイートか？
    pub is_reply: bool,            // リプライか？
    pub source_post_id: Option<i64>,     // リツイート元投稿ID
    pub source_user_id: Option<i64>,     // リツイート元ユーザーID
    pub has_video: bool,           // 動画付きか？
    pub conversation_id: Option<i64>,     // スレッドID
}
```

**設計のポイント**:
- `Option<T>` で値がない場合を表現
- リツイートチェーンをたどれる (`source_post_id`, `source_user_id`)

#### TinyPost (最小参照データ)

```rust
pub struct TinyPost {
    pub post_id: i64,
    pub created_at: i64,
}
```

**なぜ2種類のデータ構造なのか？**

| 構造 | 用途 | サイズ |
|------|------|------|
| `LightPost` | 返却値として完全投稿データを渡す | 大 |
| `TinyPost` | タイムラインインデックスとして保持 | 小 |

**メモリ効率の工夫**:
- 全投稿を `LightPost` で1箇所に保持
- ユーザー別タイムラインは `TinyPost` の `VecDeque` で保持
- 投稿検索時は `TinyPost` → `LightPost` の2段階ルックアップ

### 2.3 Kafka インジェストの仕組み

#### tweet_events_listener.rs (v2)

```rust
async fn process_tweet_events_v2(
    consumer: Arc<RwLock<KafkaConsumer>>,
    post_store: Arc<PostStore>,
    batch_size: usize,
    semaphore: Arc<Semaphore>,
) -> Result<()> {
    let mut message_buffer = Vec::new();

    loop {
        // 1. Kafkaからポーリング
        let messages = consumer.poll(batch_size).await?;
        message_buffer.extend(messages);

        // 2. バッチサイズに達したら処理
        if message_buffer.len() >= batch_size {
            // セ�aphore取得 (CPU過使用防止)
            let permit = semaphore.acquire_owned().await?;

            // ブロッキングタスクで処理 (Tokioランタイム保護)
            tokio::task::spawn_blocking(move || {
                let _permit = permit; // drop時に解放

                // デシリアライズ + 投稿挿入
                let (light_posts, delete_tweets) = deserialize_batch(messages)?;
                post_store.insert_posts(light_posts);
                post_store.mark_as_deleted(delete_tweets);
            });
        }
    }
}
```

**設計の工夫**:

1. **Semaphore制限** (`Arc<Semaphore>`):
   - Kafkaキャッチアップ中はCPUを解放
   - リクエスト処理の応答性を確保

2. **spawn_blocking**:
   - Rustのランタイム (Tokio) はシングルスレッドでイベントループ
   - 重い計算はブロッキングプールへオフロード
   - メインループをブロッキングしない

3. **バッファリング**:
   - `batch_size` 分だけ溜まってから処理
   - デシリアライズオーバーヘッドを削減

#### デシリアライズ処理

```rust
// Kafkaメッセージ → Protobuf → LightPost
fn deserialize_batch(messages: Vec<KafkaMessage>)
    -> Result<(Vec<LightPost>, Vec<TweetDeleteEvent>)>
{
    let mut create_tweets = Vec::new();
    let mut delete_tweets = Vec::new();

    for tweet_event in deserialize_kafka_messages(messages, deserialize_tweet_event)? {
        match tweet_event.event_variant {
            in_network_event::EventVariant::TweetCreateEvent(create) => {
                create_tweets.push(LightPost {
                    post_id: create.post_id,
                    author_id: create.author_id,
                    created_at: create.created_at,
                    in_reply_to_post_id: create.in_reply_to_post_id,
                    // ... その他フィールド
                });
            }
            in_network_event::EventVariant::TweetDeleteEvent(delete) => {
                delete_tweets.push(delete);
            }
        }
    }

    Ok((create_tweets, delete_tweets))
}
```

### 2.4 投稿挿入の詳細

```rust
pub fn insert_posts(&self, mut posts: Vec<LightPost>) {
    // 1. 保持期間外の投稿をフィルタ
    let current_time = now();
    posts.retain(|p| {
        p.created_at < current_time
            && current_time - p.created_at <= self.retention_seconds
    });

    // 2. 投稿作成時でソート (新しい順)
    posts.sort_unstable_by_key(|p| p.created_at);

    // 3. 挿入処理
    for post in posts {
        // 削除済みチェック
        if self.deleted_posts.contains_key(&post.post_id) {
            continue;
        }

        // すでに存在するならスキップ (重複挿入防止)
        if self.posts.insert(post.post_id, post).is_some() {
            continue;
        }

        let tiny_post = TinyPost::new(post.post_id, post.created_at);

        // オリジナル投稿 or リプライ/リポストに分類
        if !post.is_reply && !post.is_retweet {
            self.original_posts_by_user
                .entry(post.author_id)
                .or_default()
                .push_back(tiny_post);
        } else {
            self.secondary_posts_by_user
                .entry(post.author_id)
                .or_default()
                .push_back(tiny_post);
        }

        // 動画投稿にも登録
        if video_eligible(post) {
            self.video_posts_by_user
                .entry(post.author_id)
                .or_default()
                .push_back(tiny_post);
        }
    }
}
```

**重要なロジック**:

1. **重複チェック**: `insert().is_some()` で既存チェック
2. **削除優先**: `deleted_posts` にあればスキップ
3. **VecDeque使用**: `push_back()` で常に新しい順

### 2.5 In-Network検索の実装

```rust
pub fn get_all_posts_by_users(
    &self,
    user_ids: &[i64],
    exclude_tweet_ids: &HashSet<i64>,
    start_time: Instant,
    request_user_id: i64,
) -> Vec<LightPost> {
    let following_users_set: HashSet<i64> = user_ids.iter().copied().collect();

    // 1. Original posts取得
    let mut all_posts = self.get_posts_from_map(
        &self.original_posts_by_user,
        user_ids,
        MAX_ORIGINAL_POSTS_PER_AUTHOR,  // 例: 100件/ユーザー
        exclude_tweet_ids,
        &HashSet::new(),
        start_time,
        request_user_id,
    );

    // 2. Secondary posts取得
    let secondary_posts = self.get_posts_from_map(
        &self.secondary_posts_by_user,
        user_ids,
        MAX_REPLY_POSTS_PER_AUTHOR,  // 例: 50件/ユーザー
        exclude_tweet_ids,
        &following_users_set,  // リプライチェーン特殊処理用
        start_time,
        request_user_id,
    );

    all_posts.extend(secondary_posts);
    all_posts
}
```

#### get_posts_from_map の詳細

```rust
pub fn get_posts_from_map(
    &self,
    posts_map: &Arc<DashMap<i64, VecDeque<TinyPost>>>,
    user_ids: &[i64],
    max_per_user: usize,
    exclude_tweet_ids: &HashSet<i64>,
    following_users: &HashSet<i64>,
    start_time: Instant,
    request_user_id: i64,
) -> Vec<LightPost> {
    let mut light_posts = Vec::new();

    for (i, user_id) in user_ids.iter().enumerate() {
        // タイムアウトチェック
        if !self.request_timeout.is_zero() && start_time.elapsed() >= self.request_timeout {
            log::error!("Timed out fetching posts for user={}; Processed: {}/{}",
                request_user_id, i, user_ids.len());
            break;
        }

        // ユーザーのタイムライン取得
        if let Some(user_posts_ref) = posts_map.get(user_id) {
            let user_posts = user_posts_ref.value();

            // 最新順に走査 (rev() + iter())
            let tiny_posts_iter = user_posts
                .iter()
                .rev()  // 新しい順
                .filter(|post| !exclude_tweet_ids.contains(&post.post_id))
                .take(MAX_TINY_POSTS_PER_USER_SCAN); // 最大走査数制限

            // TinyPost → LightPost に変換
            let light_post_iter = tiny_posts_iter
                .filter_map(|tiny_post| self.posts.get(&tiny_post.post_id)
                    .map(|r| *r.value())
                );

            // 追加フィルタリング
            let filtered_post_iter = light_post_iter.filter(|post| {
                // 削除済み除外
                if self.deleted_posts.get(&post.post_id).is_some() {
                    return false;
                }

                // 自己リツイート除外
                if post.is_retweet && post.source_user_id == Some(request_user_id) {
                    return false;
                }

                // リプライチェーン特殊処理
                if !following_users.is_empty() {
                    if let Some(reply_to_id) = post.in_reply_to_post_id {
                        if let Some(replied_to_post) = self.posts.get(&reply_to_id) {
                            // 返信先がフォロー済みユーザーのオリジナル投稿ならOK
                            if !replied_to_post.is_retweet
                                && !replied_to_post.is_reply
                                && post.in_reply_to_user_id
                                    .map(|uid| following_users.contains(&uid))
                                    .unwrap_or(false)
                            {
                                return true;
                            }
                            return false;
                        }
                    }
                }

                true
            });

            // 最大件数制限
            light_posts.extend(filtered_post_iter.take(max_per_user));
        }
    }

    light_posts
}
```

**リプライチェーン特殊処理の意図**:

```
ユーザーA が ユーザーB をフォローしている場合:

OKパターン:
  Bが投稿 → Aがそれにリプライ  ← 表示OK
    ↓
  CがBの投稿にリプライ         ← 表示OK (間接的な関係)

NGパターン:
  Bが投稿 → Xがそれにリプライ  ← 表示NG (Xはフォローしてない)
```

### 2.6 自動保守メカニズム

#### 統計ロガー (5分間隔)

```rust
pub fn start_stats_logger(self: Arc<Self>) {
    tokio::spawn(async move {
        let mut interval = tokio::time::interval(Duration::from_secs(5));

        loop {
            interval.tick().await;

            // Prometheusメトリクス更新
            POST_STORE_USER_COUNT.set(self.original_posts_by_user.len() as f64);
            POST_STORE_TOTAL_POSTS.set(self.posts.len() as f64);
            POST_STORE_DELETED_POSTS.set(self.deleted_posts.len() as f64);

            // ログ出力
            info!("PostStore Stats: {} users, {} total posts, {} deleted posts",
                user_count, total_posts, deleted_posts
            );
        }
    });
}
```

#### 自動トリム (2分間隔)

```rust
pub async fn trim_old_posts(&self) -> usize {
    let current_time = now();
    let mut total_trimmed = 0;

    // 各ユーザーのタイムラインから古い投稿を削除
    for mut entry in posts_by_user.iter_mut() {
        let user_posts = entry.value_mut();

        // 最古の投稿から保持期間をチェック
        while let Some(oldest) = user_posts.front() {
            if current_time - oldest.created_at > retention_seconds {
                let trimmed = user_posts.pop_front().unwrap();
                posts_map.remove(&trimmed.post_id);  // 全データからも削除
                trimmed += 1;
            } else {
                break;
            }
        }

        // 空になったユーザーエントリを削除
        if user_posts.is_empty() {
            posts_by_user.remove_if(&user_id, |_, posts| posts.is_empty());
        }
    }

    total_trimmed
}
```

**キャパシティ最適化**:
```rust
// VecDequeの容量が不必要に大きい場合に縮小
if user_posts.capacity() > user_posts.len() * 2 {
    user_posts.shrink_to_fit();
}
```

---

## 第3章: Phoenix - ML検索とランキング

### 3.1 Phoenixの2段階アーキテクチャ

```
┌──────────────────────────────────────────────────────────────────┐
│                          グローバルコーパス                          │
│                     (数百万〜数億投稿)                                │
└──────────────────────────────────────────────────────────────────┘
                              ↓
                    ┌─────────────────┐
                    │  Retrieval Stage  │
                    │   (Two-Tower)    │
                    └─────────────────┘
                              ↓
        ┌─────────────────────────────────────┐
        │  ~1,000 candidates (百万→千)          │
        └─────────────────────────────────────┘
                              ↓
                    ┌─────────────────┐
                    │   Ranking Stage   │
                    │ (Grok Transformer)│
                    └─────────────────┘
                              ↓
        ┌─────────────────────────────────────┐
        │  ~50-100 candidates (千→50)          │
        │  (各投稿に14アクション確率)           │
        └─────────────────────────────────────┘
```

### 3.2 Retrieval: Two-Tower モデル

#### モ式図

```
┌─────────────────────┐                  ┌──────────────────────┐
│      User Tower      │                  │    Candidate Tower   │
│  (ユーザー埋め込み)    │                  │   (投稿埋め込み)     │
└─────────────────────┘                  └──────────────────────┘
            │                                      │
            │                                      │
     User Features +                          Post Content +
    Engagement History                        Metadata
            │                                      │
            ▼                                      ▼
    ┌───────────────┐                      ┌──────────────┐
    │ User Hash × 2 │                      │Post Hash × 2│
    └───────────────┘                      └──────────────┘
            │                                      │
            ▼                                      ▼
    User Embedding Table                    Candidate Embedding Table
    [num_users × D]                         [num_posts × D]
            │                                      │
            ▼                                      ▼
    ┌──────────────────────────────────────────────┐
    │           L2 Normalize                          │
    │    user_embedding / ||user_embedding||           │
    └──────────────────────────────────────────────┘
            │                                      │
        [B, 1, D]                           [N, D]
            │                                      │
            ▼                                      ▼
    ┌──────────────────────────────────────────────┐
    │         Dot Product Similarity                │
    │     user · candidate^T                       │
    └──────────────────────────────────────────────�
            │
            ▼
    ┌──────────────────────────────────────────────┐
    │              Top-K Selection                  │
    │     argmax(similarity) = top_k_indices        │
    └──────────────────────────────────────────────┘
```

#### JAXでの実装

```python
def build_inputs(batch: RecsysBatch, embeddings: RecsysEmbeddings):
    # User埋め込み構築
    user_embeddings, user_padding_mask = block_user_reduce(
        batch.user_hashes,           # [B, num_user_hashes]
        embeddings.user_embeddings,  # [B, num_user_hashes, D]
        hash_config.num_user_hashes,
        config.emb_size,
    )
    # Output: [B, 1, D]

    # History埋め込み構築
    history_embeddings, history_padding_mask = block_history_reduce(
        batch.history_post_hashes,           # [B, S, num_item_hashes]
        embeddings.history_post_embeddings,  # [B, S, num_item_hashes, D]
        embeddings.history_author_embeddings, # [B, S, num_author_hashes, D]
        embeddings.history_actions_embeddings, # [B, S, D]
        batch.history_product_surface,        # [B, S]
        hash_config.num_item_hashes,
        hash_config.num_author_hashes,
        config.emb_size,
    )
    # Output: [B, S, D]

    # 候補者埋め込み構築
    candidate_embeddings, candidate_padding_mask = block_candidate_reduce(
        batch.candidate_post_hashes,        # [B, C, num_item_hashes]
        embeddings.candidate_post_embeddings,  # [B, C, num_item_hashes, D]
        embeddings.candidate_author_embeddings, # [B, C, num_author_hashes, D]
        batch.candidate_product_surface,     # [B, C]
        hash_config.num_item_hashes,
        hash_config.num_author_hashes,
        config.emb_size,
    )
    # Output: [B, C, D]

    # 結合
    embeddings = jnp.concatenate([
        user_embeddings,        # [B, 1, D]
        history_embeddings,      # [B, S, D]
        candidate_embeddings,    # [B, C, D]
    ], axis=1)
    # Output: [B, 1+S+C, D]
```

### 3.3 Ranking: Grok Transformer

#### 候補者アイソレーションの詳細

**問題**: 通常のTransformerでは、全トークンが全トークンにattendできる

**解決**: 候補者同士のattendを禁止した特別なAttention Maskを使用

#### Attention Maskの生成

```python
def make_recsys_attn_mask(seq_len: int, candidate_start_offset: int):
    # 1. Causal mask (下三角行列)
    causal_mask = jnp.tril(jnp.ones((seq_len, seq_len)))
    # [[1, 0, 0, 0],
    #  [1, 1, 0, 0],
    #  [1, 1, 1, 0],
    #  [1, 1, 1, 1]]

    # 2. 候補者間のattendantをゼロに (右下ブロック)
    attn_mask = causal_mask.at[:, :, candidate_start_offset:, candidate_start_offset:].set(0)
    # [[1, 0, 0, 0],
    #  [1, 1, 0, 0],
    #  [1, 1, 1, 0],    ← ここをゼロに
    #  [1, 1, 1, 1]]

    # 3. 対角成分 (自己attenttion) を復活
    candidate_indices = jnp.arange(candidate_start_offset, seq_len)
    attn_mask = attn_mask.at[:, :, candidate_indices, candidate_indices].set(1)
    # [[1, 0, 0, 0],
    #  [1, 1, 0, 0],
    #  [1, 1, 1, 1],    ← ここを1に
    #  [1, 1, 1, 1]]    ← ここも1

    return attn_mask  # [1, 1, seq_len, seq_len]
```

**視覚的な理解**:

```
ユーザーと履歴は完全に双方向、
候補者はユーザー/履歴のみ参照可

       │  User  Hist  Hist  Hist  Cand1 Cand2 Cand3
       │
   ────┼──────┼─────┼─────┼─────┼─────┼─────┐
U │ ✓ │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
s ────┼──────┼─────┼─────┼─────┼─────┼─────┤
e │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
r │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
   │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
H │   │ ✓    │ ✓   │ ✓   │  ✗   │  �   │  ✗
i │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
s │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
t │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
o │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
r │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
y │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
   │   │ ✓    │ ✓   │ ✓   │  ✗   │  ✗   │  ✗
C ────┼──────┼─────┼─────┼─────┼─────┼─────┤
a │   │ ✓    │ ✓   │ ✓   │  ✓   │  ✗   │  ✗
n │   │ ✓    │ ✓   │ ✓   │  ✓   │  ✗   │  ✗
d │   │ ✓    │ ✓   │ ✓   │  ✓   │  ✗   │  ✗
i │   │ ✓    │ ✓   │ ✓   │  ✓   │  ✗   │  ✗
d │   │ ✓    │ ✓   │ ✓   │  ✓   │  ✗   │  ✗
e │   │ ✓    │ ✓   │ ✓   │  ✓   │  ✗   │  ✗
s │   │ ✓    │ ✓   │ ✓   │  ✓   │  ✗   │  ✗
   └───┴──────┴─────┴─────┴─────┴─────┴─────┘
        │
        ▼
    ✓ = attend可能
    ✗ = attend禁止
```

**なぜこの設計なのか？**

| デザイン | 理由 |
|--------|------|
| 候補者はユーザー/履歴のみattend | ユーザーの興味に基づいてスコアリング |
| 候補者同士はattend禁止 | 他候補の影響を受けない |
| スコアの一貫性 | バッチサイズ変更でもスコア不変 |

#### Transformer Forward Pass

```python
def __call__(
    self,
    embeddings: jax.Array,  # [B, 1+S+C, D]
    padding_mask: jax.Array,  # [B, 1+S+C]
    candidate_start_offset: int,  # 1+S
) -> TransformerOutput:
    # Attention Mask生成
    attn_mask = make_recsys_attn_mask(
        seq_len,
        candidate_start_offset
    )

    # Transformer Layers
    for i in range(num_layers):
        h = DecoderLayer(i)(
            h,
            attn_mask,
            padding_mask
        )
        h = h.embeddings

    # 出力から候補者部分を抽出
    candidate_embeddings = h[:, candidate_start_offset:, :]

    # Layer Normalization
    candidate_embeddings = layer_norm(candidate_embeddings)

    # Unembedding (logits)
    logits = jnp.dot(candidate_embeddings, unembed_matrix)
    # logits: [B, C, num_actions]

    return TransformerOutput(embeddings=h)
```

### 3.4 14アクション予測

#### アクション一覧と重み

```rust
// weighted_scorer.rs

fn compute_weighted_score(candidate: &PostCandidate) -> f64 {
    let combined_score =
        // 正の重み (エンゲージメント促進)
        favorite_score × FAVORITE_WEIGHT +
        reply_score × REPLY_WEIGHT +
        retweet_score × RETWEET_WEIGHT +
        quote_score × QUOTE_WEIGHT +
        click_score × CLICK_WEIGHT +
        profile_click_score × PROFILE_CLICK_WEIGHT +
        video_view_score × VIDEO_VIEW_WEIGHT +
        photo_expand_score × PHOTO_EXPAND_WEIGHT +
        share_score × SHARE_WEIGHT +
        follow_author_score × FOLLOW_AUTHOR_WEIGHT +
        dwell_score × DWELL_WEIGHT +

        // 負の重み (エンゲージメント抑制)
        not_interested_score × NOT_INTERESTED_WEIGHT +
        block_author_score × BLOCK_AUTHOR_WEIGHT +
        mute_author_score × MUTE_AUTHOR_WEIGHT +
        report_score × REPORT_WEIGHT;
}
```

**VQV (Video Quality View) の特殊扱い**:

```rust
fn vqv_weight_eligibility(candidate: &PostCandidate) -> f64 {
    if candidate.has_video
        && candidate.video_duration_ms > Some(MIN_VIDEO_DURATION_MS)
    {
        VQV_WEIGHT  // 動画あり & 閾基準時間超過
    } else {
        0.0
    }
}
```

---

## 第4章: Home Mixer - パイプラインオーケストレーション

### 4.1 CandidatePipeline トレイトの実装

#### Source トレイト

```rust
#[async_trait]
trait Source<Q, C>: Send + Sync
where
    Q: HasRequestId + Clone + Send + Sync + 'static,
    C: Clone + Send + Sync + 'static,
{
    // 実行するかどうか
    fn enable(&self, query: &Q) -> bool;

    // 候補を取得
    fn get_candidates(&self, query: &Q) -> Future<Vec<C>>;
}
```

**実装例: ThunderSource**

```rust
#[async_trait]
impl Source<ScoredPostsQuery, PostCandidate> for ThunderSource {
    fn enable(&self, query: &ScoredPostsQuery) -> bool {
        true  // 常に有効
    }

    async fn get_candidates(&self, query: &ScoredPostsQuery) -> Future<Vec<PostCandidate>> {
        // gRPC呼び出し
        let response = thunder_client.get_in_network_posts(
            query.user_id as i64,
            query.following_user_ids.to_vec(),
            query.exclude_tweet_ids.to_vec(),
        ).await?;

        // Proto → PostCandidate 変換
        response.posts.into_iter().map(|p| PostCandidate {
            tweet_id: p.tweet_id,
            author_id: p.author_id,
            // ...
        }).collect()
    }
}
}
```

#### Filter トレイト

```rust
#[async_trait]
trait Filter<Q, C>: Send + Sync {
    fn enable(&self, query: &Q) -> bool;

    async fn filter(
        &self,
        query: &Q,
        candidates: Vec<C>
    ) -> Future<FilterResult>;
}

struct FilterResult<C> {
    pub kept: Vec<C>,      // 残す候補
    pub removed: Vec<C>,   // 除外した候補
}
```

**実装例: AgeFilter**

```rust
struct AgeFilter {
    max_age: Duration,
}

impl Filter<ScoredPostsQuery, PostCandidate> for AgeFilter {
    fn enable(&self, _query: &ScoredPostsQuery) -> bool {
        true
    }

    async fn filter(
        &self,
        _query: &ScoredPostsQuery,
        candidates: Vec<PostCandidate>,
    ) -> Future<FilterResult<PostCandidate>> {
        let (kept, removed): (Vec<_>, Vec<_>) = candidates
            .into_iter()
            .partition(|c| self.is_within_age(c.tweet_id));

        Ok(FilterResult { kept, removed })
    }
}
```

**partition()の使い方**:
```rust
let (kept, removed): (Vec<_>, Vec<_>) = candidates
    .into_iter()
    .partition(|c| condition(c));

// kept: 条件を満たす要素
// removed: 条件を満たさない要素
```

### 4.2 パイプライン実行エンジン

```rust
pub async fn execute(&self, query: Q) -> PipelineResult<Q, C> {
    // Phase 1: Query Hydration (並列)
    let hydrated_query = self.hydrate_query(query).await;
    // ↑ join_allで全QueryHydratorを並列実行

    // Phase 2: Candidate Sourcing (並列)
    let candidates = self.fetch_candidates(&hydrated_query).await;
    // ↑ join_allで全Sourceを並列実行

    // Phase 3: Hydration (並列)
    let hydrated_candidates = self.hydrate(&hydrated_query, candidates).await;
    // ↑ join_allで全Hydratorを並列実行

    // Phase 4: Filter (逐次)
    let (kept_candidates, filtered_candidates) = self
        .filter(&hydrated_query, hydrated_candidates.clone()).await;
    // ↑ エラー時はバックアップして元の候補を保持

    // Phase 5: Score (逐次)
    let scored_candidates = self.score(&hydrated_query, kept_candidates).await;

    // Phase 6: Selection
    let selected_candidates = self.select(&hydrated_query, scored_candidates);

    // Phase 7: Post-Selection Hydration (並列)
    let post_selection_hydrated = self
        .hydrate_post_selection(&hydrated_query, selected_candidates)
        .await;

    // Phase 8: Post-Selection Filter (逐次)
    let (final_candidates, post_filtered) = self
        .filter_post_selection(&hydrated_query, post_selection_hydrated)
        .await;
    filtered_candidates.extend(post_filtered);

    // Phase 9: Side Effects (非同期)
    self.run_side_effects(Arc::new(SideEffectInput {
        query: Arc::new(hydrated_query),
        selected_candidates: final_candidates.clone(),
    }));
    // ↑ spawnして即座に返却 (完了待ちしない)

    PipelineResult {
        retrieved_candidates: hydrated_candidates,
        filtered_candidates,
        selected_candidates: final_candidates,
        query: Arc::new(hydrated_query),
    }
}
```

**並列/逐次の使い分け**:

| フェーズ | 実行方法 | 理由 |
|--------|----------|------|
| Query Hydration | 並列 | 依存関係なし |
| Candidate Sources | 並列 | 依存関係なし |
| Hydration | 並列 | 候補間で依存なし |
| Filters | 逐次 | 順次適用が重要 |
| Scorers | 逐次 | 前のスコアを利用 |
| Side Effects | 並列(非同期) | 結果に影響なし |

---

## 第5章: スコアリングの詳細

### 5.1 PhoenixScorer - ML予測の取得

```rust
pub struct PhoenixScorer {
    phoenix_client: Arc<dyn PhoenixPredictionClient + Send + Sync>,
}

async fn score(&self, query: &ScoredPostsQuery, candidates: &[PostCandidate])
    -> Result<Vec<PostCandidate>, String>
{
    // 1. 予測リクエスト作成
    let tweet_infos: Vec<TweetInfo> = candidates.iter().map(|c| {
        TweetInfo {
            tweet_id: c.tweet_id,
            author_id: c.author_id,
            // ...
        }
    }).collect();

    // 2. Phoenix gRPC呼び出し
    let response = self.phoenix_client.predict(
        query.user_id as u64,
        query.user_action_sequence.clone(),
        tweet_infos,
    ).await?;

    // 3. 予測結果のパース
    let mut scored = Vec::new();
    for candidate in candidates {
        let lookup_id = candidate.retweeted_tweet_id
            .unwrap_or(candidate.tweet_id);

        let phoenix_scores = predictions_map
            .get(&lookup_id)
            .map(|preds| extract_phoenix_scores(preds))
            .unwrap_or_default();

        scored.push(PostCandidate {
            phoenix_scores,
            prediction_request_id: Some(request_id),
            last_scored_at_ms: now_millis(),
            ..candidate.clone()
        });
    }

    Ok(scored)
}
```

**PhoenixScoresの構造**:

```rust
pub struct PhoenixScores {
    // 離散アクション (14種類)
    pub favorite_score: Option<f64>,
    pub reply_score: Option<f64>,
    pub retweet_score: Option<f64>,
    pub quote_score: Option<f64>,
    pub click_score: Option<f64>,
    pub profile_click_score: Option<f64>,
    pub video_view_score: Option<f64>,
    pub photo_expand_score: Option<f64>,
    pub share_score: Option<f64>,
    pub follow_author_score: Option<f64>,
    pub not_interested_score: Option<f64>,
    pub block_author_score: Option<f64>,
    pub mute_author_score: Option<f64>,
    pub report_score: Option<f64>,

    // 連続アクション
    pub dwell_time: Option<f64>,  // 滞在時間
}
```

### 5.2 WeightedScorer - 重み付き結合

```rust
fn compute_weighted_score(candidate: &PostCandidate) -> f64 {
    let s = &candidate.phoenix_scores;

    // 全アクションの重み付き合計
    let combined =
        apply(s.favorite_score, FAVORITE_WEIGHT) +
        apply(s.reply_score, REPLY_WEIGHT) +
        apply(s.retweet_score, RETWEET_WEIGHT) +
        apply(s.quote_score, QUOTE_WEIGHT) +
        apply(s.click_score, CLICK_WEIGHT) +
        apply(s.profile_click_score, PROFILE_CLICK_WEIGHT) +
        apply(s.video_view_score, vqv_weight) +  // 条件付き
        apply(s.share_score, SHARE_WEIGHT) +
        apply(s.dwell_score, DWELL_WEIGHT) +
        apply(s.follow_author_score, FOLLOW_AUTHOR_WEIGHT) +
        apply(s.not_interested_score, NOT_INTERESTED_WEIGHT) +  // 負
        apply(s.block_author_score, BLOCK_AUTHOR_WEIGHT) +      // 負
        apply(s.mute_author_score, MUTE_AUTHOR_WEIGHT) +        // 負
        apply(s.report_score, REPORT_WEIGHT);                    // 負

    offset_score(combined_score)
}

fn apply(score: Option<f64>, weight: f64) -> f64 {
    score.unwrap_or(0.0) * weight
}
```

**offset_score() の役割**:

```rust
fn offset_score(combined: f64) -> f64 {
    if combined_score < 0.0 {
        // 負のスコアを正規化してシフト
        // 例: combined=-100, WEIGHTS_SUM=10, NEGATIVE_SCORES_OFFSET=50
        // → (-100 + 50) / 10 = -5
        (combined + NEGATIVE_WEIGHTS_SUM) / WEIGHTS_SUM * NEGATIVE_SCORES_OFFSET
    } else {
        // 正のスコアは単純にシフト
        combined + NEGATIVE_SCORES_OFFSET
    }
}
```

**イメージ**:
```
負のスコアの分布:
  -100  -50    0    50    100
    │     │     │     │     │
    ▼     ▼     ▼     ▼     ▼
  ──────┴─────┴─────┴─────┴─────→
    正規化           シフト
```

### 5.3 AuthorDiversityScorer - 作者多様性確保

```rust
fn multiplier(&self, position: usize) -> f64 {
    // position: そのユーザーの今回の表示回数目
    // FLOOR: 下限値 (例: 0.5)
    // DECAY_FACTOR: 減衰率 (例: 0.5)
    (1.0 - FLOOR) × DECAY_FACTOR^position + FLOOR
}
```

**具体例** (`FLOOR=0.5, DECAY=0.5`):

```
position | multiplier | スコアへの影響
---------|-----------|---------------
0        | 1.00      | 100% (維持)
1        | 0.75      | 75%  (25%減)
2        | 0.625     | 62.5% (37.5%減)
3        | 0.5625    | 56.25% (43.75%減)
4        | 0.53125   | 53.125% (46.875%減)

例:
投稿A (元スコア100) × 1.0 = 100
投稿B (元スコア100) × 0.75 = 75  ← 同作者2回目
投稿C (元スコア100) × 0.625 = 62.5 ← 同作者3回目
```

**処理フロー**:

```rust
// 1. 元スコアで降順ソート
let mut ordered: Vec<(usize, &PostCandidate)> = candidates
    .iter()
    .enumerate()
    .collect();
ordered.sort_by(|(_, a), (_, b)| {
    b.weighted_score.partial_cmp(&a.weighted_score).unwrap()
});

// 2. 作者ごとの出現回数をカウントしながらスコア調整
for (original_idx, candidate) in ordered {
    let entry = author_counts.entry(candidate.author_id).or_insert(0);
    let position = *entry;  // このユーザーで何回目か
    *entry += 1;

    let multiplier = self.multiplier(position);
    let adjusted_score = candidate.weighted_score * multiplier;

    scored[original_idx] = PostCandidate {
        score: Some(adjusted_score),
        ..
    };
}
```

---

## 第6章: 実践的な理解 - 具体例で追う

### 6.1 完全なリクエストフロー

#### ステップ1: ユーザーリクエスト

```protobuf
message ScoredPostsRequest {
    int64 user_id = 1;
    repeated int64 following_user_ids = [123, 456, 789];
    repeated int64 exclude_tweet_ids = [111, 222];
    bool is_video_request = false;
    int64 max_results = 50;
    bool debug = false;
}
```

#### ステップ2: Query Hydration

```rust
// UserActionSeqQueryHydrator
user_action_sequence: UserActionSequence {
    user_id: 1,
    actions: [
        Action { tweet_id: 100, action_type: "favorite" },
        Action { tweet_id: 101, action_type: "reply" },
        Action { tweet_id: 102, action_type: "retweet" },
        // ... 最近のエンゲージメント履歴
    ],
}

// UserFeaturesQueryHydrator
user_features: UserFeatures {
    following_user_ids: [123, 456, 789],
    muted_keywords: ["spam", "abuse"],  // ミュート済みキーワード
    blocked_user_ids: [999],           // ブロック済みユーザー
    // ...
}
```

#### ステップ3: Candidate Sourcing

```rust
// ThunderSource (In-Network)
let in_network_posts = thunder.get_in_network_posts(
    user_id: 1,
    following_user_ids: [123, 456, 789],
    exclude_tweet_ids: [111, 222],
    max_results: 200,
).await?;
// 例: 150件取得 (0.5ms)

// PhoenixSource (Out-of-Network)
let oon_posts = phoenix.retrieve(
    user_action_sequence,
    top_k: 200,
).await?;
// 例: 180件取得 (30ms)
```

#### ステップ4: Hydration

```rust
// CoreDataCandidateHydrator
for post in combined_posts {
    post.tweet_text = fetch_tweet_text(post.tweet_id).await?;
    post.media_entities = fetch_media(post.tweet_id).await?;
    // ...
}

// GizmoduckHydrator
for post in combined_posts {
    post.author_username = fetch_username(post.author_id).await?;
    post.author_verified = fetch_verification(post.author_id).await?;
    // ...
}
```

#### ステップ5: Pre-Scoring Filtering

```
入力: 330件 (150 from Thunder + 180 from Phoenix)

AgeFilter:
  - 保持期間外の古い投稿を除外
  - 例: 20件除外
  → 310件

SelfTweetFilter:
  - 自分の投稿を除外
  - 例: 5件除外
  → 305件

PreviouslySeenPostsFilter:
  - 既に見た投稿を除外
  - Redisに記録済み
  - 例: 35件除外
  → 270件

MutedKeywordFilter:
  - ミュート済みキーワード含む投稿を除外
  - 例: 10件除外
  → 260件

AuthorSocialgraphFilter:
  - ブロック/ミュート済みユーザーの投稿を除外
  - 例: 5件除外
  → 255件

最終候補数: 255件
```

#### ステップ6: Scoring

```rust
// PhoenixScorer
// ML予測リクエスト
phoenix_response = phoenix.predict(
    user_id=1,
    user_action_sequence=seq,
    candidates=tweet_infos,
).await?;

// 例: 候補者Aの予測
Candidate A:
  favorite_score: 0.75     → 75%の確率でいいね
  reply_score: 0.30        → 30%の確率でリプライ
  retweet_score: 0.15      → 15%の確率でリポスト
  block_author_score: 0.05  → 5%の確率でブロック

// WeightedScorer
// 重み付け結合
weighted_score =
    0.75 × 1.0 +     // favorite
    0.30 × 2.0 +     // reply (重み高め)
    0.15 × 1.5 +     // retweet
    0.05 × -10.0      // block (負の重み)
= 1.35

// AuthorDiversityScorer
// 同一作者の3回目の投稿なら
multiplier = 0.625
adjusted_score = 1.35 × 0.625 = 0.84
```

#### ステップ7: Selection

```rust
// 降順ソート
let mut ranked: Vec<_> = candidates.sort_by(|a, b| {
    b.score.partial_cmp(&a.score).unwrap()
});

// Top-K選択
let final_posts = ranked.into_iter().take(50).collect();
```

---

## 第7章: デプロイメントと運用

### 7.1 本番環境構成

```
┌─────────────────────────────────────────────────────────────────┐
│                         Cloud Load Balancer                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   Home Mixer Pods (5-10 replicas)           │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  │
│  │ Pod 1   │  │ Pod 2   │  │ Pod 3   │  │ Pod 4   │  │
│  │ :14534 │  │ :14534 │  │ :14534 │  │ :14534 │  │
│  └────────┘  └────────┘  └────────┘  └────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
        ┌───────────────────────────────────────┐
        │              Thunder Cluster            │
        │  ┌────────┐  ┌────────┐  ┌────────┐   │
        │  │ Shard1 │  │ Shard2 │  │ Shard3 │   │
        │  │ :8000  │  │ :8001  │  │ :8002  │   │
        │  └────────┘  └────────┘  └────────┘   │
        └───────────────────────────────────────┘
                              ↓
        ┌───────────────────────────────────────┐
        │           Phoenix Cluster              │
        │  ┌────────┐  ┌────────┐  ┌────────┐   │
        │  │Retriever│  │Retriever│  │ Ranker  │   │
        │  │ :8080  │  │ :8081  │  │ :8082  │   │
        │  └────────┘  └────────┘  └────────┘   │
        └───────────────────────────────────────┘
                              ↓
        ┌───────────────────────────────────────┐
        │          Data Layer                   │
        │  ┌────────┐  ┌────────┐  ┌────────┐ │
        │  │ Kafka  │  │Social  │  │Content │ │
        │  │        │  │Graph DB│  │ Store  │ │
        │  └────────┘  └────────┘  └────────┘ │
        │  ┌─────────────────────────────────┐ │
        │  │    Redis (Cache)              │ │
        │  └─────────────────────────────────┘ │
        └───────────────────────────────────────┘
```

### 7.2 スケーリング戦略

#### 水平スケーリング

| コンポーネント | スケール方法 | トリガー |
|---------------|-----------|--------|
| Home Mixer | HPA | CPU使用率 |
| Thunder | Shard数固定 | デプロイ時決定 |
| Phoenix | デプロイ済み固定 | -

#### 垂直スケーリング

| コンポーネント | 方法 | 時間 |
|---------------|------|------|
| Phoenix Retriever | ANNインデックス再構築 | 毎日 |
| Phoenix Ranker | モデル更新 | 毎週 |

---

## 第8章: まとめと学び

### 8.1 このシステムの最大の特徴

**1. 手作り特徴量の排除**
- エンジニアが特徴量を設計する必要がない
- Grok Transformerが全てを学習
- シンプルなデータパイプライン

**2. 候補者アイソレーション**
- 各投稿のスコアが独立
- キャッシュ可能で一貫性

**3. パフォーマンスと精度の両立**
- Thunder: In-Networkは1ms以下
- Phoenix: Out-of-Networkも数十ms
- 全体で100-200ms

### 8.2 エンジニアリングの洞察

**1. Rust + Pythonのハイブリッド**
- 高速なオーケストレーション (Rust)
- 柔軟なML推論 (Python/JAX)

**2. In-Memoryの重要性**
- Thunderは全投稿をメモリ保持
- サブミリ秒応答を可能に

**3. Async/Awaitの活用**
- Kafka処理は`spawn_blocking`でオフロード
- 副些段階で並列化を実現

### 8.3 改善の余地

**潜在的な改善点**:
1. Phoenix Rankerのバッチサイズ動的調整
2. 候補者アイソレーション度合いの検証
3. 新しいアクションタイプの追加容易さ

---

## 付録: 用語集

| 用語 | 説明 |
|------|------|
| In-Network | フォロー中ユーザーの投稿 |
| Out-of-Network | フォローしていないユーザーの投稿 |
| Candidate | 候補投稿のこと |
| Hydration | データ補完（付加情報の取得） |
| Candidate Isolation | 候補者が互いに参照しない設計 |
| Hash-based Embedding | ハッシュ関数で埋め込み検索 |
| Two-Tower Model | ユーザー/候補者で別々の埋め込みモデル |
| ANN (Approximate Nearest Neighbor) | 近似検索の高速化アルゴリズム |
| DashMap | Rustの並行HashMap実装 |
| VecDeque | Rustの両端キュー（先頭/末尾への追加が高速）|

---

**参考文献**:
- xai-org/x-algorithm GitHub: https://github.com/xai-org/x-algorithm
- Grok-1 Open Source: https://github.com/xai-org/grok-1
- JAX Documentation: https://jax.readthedocs.io/

---

**変更履歴**:
- 2026-01-20: 初版
- 作成者: Claude (Anthropic)
- ライセンス: Apache License 2.0
