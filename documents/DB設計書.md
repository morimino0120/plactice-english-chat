# データベース設計書：AI英語学習チャットアプリ

## 1. 設計概要

本システムは、ユーザーとAIの対話を管理し、ブックマーク機能を提供するためのデータベース設計です。
リレーショナルデータベース（PostgreSQL、MySQL等）を想定していますが、FirebaseやSupabaseでも同様の構造で実装可能です。

---

## 2. ER図（概念）

```
[users] 1 ────< [chat_sessions] 1 ────< [messages]
                                              │
                                              │ 1
                                              │
                                              └───< [message_bubbles] 1 ────< [bookmarks]
```

---

## 3. テーブル定義

### 3.1 users（ユーザー）

ユーザー情報を管理するテーブル。

| カラム名 | データ型 | 制約 | 説明 |
|---------|---------|------|------|
| id | UUID | PRIMARY KEY | ユーザーID（一意） |
| email | VARCHAR(255) | UNIQUE, NOT NULL | メールアドレス |
| name | VARCHAR(100) | | ユーザー名 |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 作成日時 |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 更新日時 |

**インデックス:**
- `idx_users_email` ON `users` (`email`)

**備考:**
- 認証機能を使用しない場合は、このテーブルは省略可能
- Firebase Authenticationを使用する場合、`id`はFirebase UIDと連携

#### Supabase使用時の推奨設計

Supabaseを使用する場合、以下の2つのアプローチがあります：

**アプローチ1: `auth.users`と統合（推奨）**

Supabaseは自動的に`auth.users`テーブルを作成します。アプリケーション固有のユーザー情報は`public.profiles`テーブルで管理することを推奨します。

```sql
-- public.profilesテーブル（Supabase推奨パターン）
CREATE TABLE public.profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    name VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- RLS（Row Level Security）ポリシー
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile"
    ON public.profiles FOR SELECT
    USING (auth.uid() = id);

CREATE POLICY "Users can update own profile"
    ON public.profiles FOR UPDATE
    USING (auth.uid() = id);
```

この場合、他のテーブルの`user_id`は`auth.users.id`を参照します。

**アプローチ2: 独自のusersテーブルを使用**

認証をSupabase以外で管理する場合や、`auth.users`と分離したい場合は、現在の設計のまま使用可能です。この場合、`id`は`UUID`型で`DEFAULT gen_random_uuid()`を設定します。

```sql
CREATE TABLE public.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

**推奨: アプローチ1（`auth.users`と統合）**
- Supabaseの認証機能を最大限活用できる
- セキュリティが強化される（RLSが自動的に適用される）
- メンテナンスが容易

---

### 3.2 chat_sessions（チャットセッション）

チャットセッション（会話の単位）を管理するテーブル。

| カラム名 | データ型 | 制約 | 説明 |
|---------|---------|------|------|
| id | UUID | PRIMARY KEY | セッションID（一意） |
| user_id | UUID | FOREIGN KEY, NOT NULL | ユーザーID（users.idまたはauth.users.id参照） |
| title | VARCHAR(255) | | セッションタイトル（最初のメッセージから自動生成等） |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 作成日時 |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 更新日時 |
| deleted_at | TIMESTAMP | | 削除日時（論理削除用） |

**外部キー:**
- 通常の場合: `fk_chat_sessions_user_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`) ON DELETE CASCADE
- Supabase使用時: `fk_chat_sessions_user_id` FOREIGN KEY (`user_id`) REFERENCES `auth.users` (`id`) ON DELETE CASCADE

**インデックス:**
- `idx_chat_sessions_user_id` ON `chat_sessions` (`user_id`)
- `idx_chat_sessions_created_at` ON `chat_sessions` (`created_at`)

**備考:**
- セッションごとに会話履歴を管理
- 論理削除に対応（deleted_atを使用）
- Supabase使用時は`user_id`は`auth.users.id`を参照

---

### 3.3 messages（メッセージ）

ユーザーとAIのメッセージを管理するテーブル。

| カラム名 | データ型 | 制約 | 説明 |
|---------|---------|------|------|
| id | UUID | PRIMARY KEY | メッセージID（一意） |
| session_id | UUID | FOREIGN KEY, NOT NULL | セッションID（chat_sessions.id参照） |
| role | VARCHAR(20) | NOT NULL | メッセージの役割（'user' / 'assistant'） |
| content | TEXT | NOT NULL | メッセージ内容 |
| sequence | INTEGER | NOT NULL | セッション内での順序（1から開始） |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 作成日時 |

**外部キー:**
- `fk_messages_session_id` FOREIGN KEY (`session_id`) REFERENCES `chat_sessions` (`id`) ON DELETE CASCADE

**インデックス:**
- `idx_messages_session_id` ON `messages` (`session_id`)
- `idx_messages_session_sequence` ON `messages` (`session_id`, `sequence`)

**備考:**
- `role`は'user'（ユーザーメッセージ）または'assistant'（AIメッセージ）を格納
- `sequence`はセッション内でのメッセージの順序を管理
- AIのメッセージは、後述の`message_bubbles`テーブルで3つの吹き出しに分割される

---

### 3.4 message_bubbles（メッセージバブル）

AIメッセージを3つの独立した吹き出し（バブル）に分割して管理するテーブル。

| カラム名 | データ型 | 制約 | 説明 |
|---------|---------|------|------|
| id | UUID | PRIMARY KEY | バブルID（一意） |
| message_id | UUID | FOREIGN KEY, NOT NULL | メッセージID（messages.id参照） |
| bubble_type | VARCHAR(50) | | バブルの種類（'explanation' / 'example1' / 'example2' 等） |
| content | TEXT | NOT NULL | バブルの内容（テキスト） |
| display_order | INTEGER | NOT NULL | 表示順序（1, 2, 3） |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 作成日時 |

**外部キー:**
- `fk_message_bubbles_message_id` FOREIGN KEY (`message_id`) REFERENCES `messages` (`id`) ON DELETE CASCADE

**インデックス:**
- `idx_message_bubbles_message_id` ON `message_bubbles` (`message_id`)
- `idx_message_bubbles_display_order` ON `message_bubbles` (`message_id`, `display_order`)

**備考:**
- AIの1つのメッセージ（`messages`テーブル）に対して、最大3つのバブルが作成される
- `bubble_type`は任意（解説、例文1、例文2等を識別するため）
- `display_order`で表示順序を管理（1, 2, 3）
- 各バブルは個別にブックマーク可能

---

### 3.5 bookmarks（ブックマーク）

ユーザーが保存したメッセージバブルを管理するテーブル。

| カラム名 | データ型 | 制約 | 説明 |
|---------|---------|------|------|
| id | UUID | PRIMARY KEY | ブックマークID（一意） |
| user_id | UUID | FOREIGN KEY, NOT NULL | ユーザーID（users.idまたはauth.users.id参照） |
| bubble_id | UUID | FOREIGN KEY, NOT NULL | バブルID（message_bubbles.id参照） |
| note | TEXT | | ユーザーが追加したメモ（任意） |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 作成日時 |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 更新日時 |
| deleted_at | TIMESTAMP | | 削除日時（論理削除用） |

**外部キー:**
- 通常の場合: `fk_bookmarks_user_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`) ON DELETE CASCADE
- Supabase使用時: `fk_bookmarks_user_id` FOREIGN KEY (`user_id`) REFERENCES `auth.users` (`id`) ON DELETE CASCADE
- `fk_bookmarks_bubble_id` FOREIGN KEY (`bubble_id`) REFERENCES `message_bubbles` (`id`) ON DELETE CASCADE

**インデックス:**
- `idx_bookmarks_user_id` ON `bookmarks` (`user_id`)
- `idx_bookmarks_created_at` ON `bookmarks` (`user_id`, `created_at` DESC)
- `UNIQUE idx_bookmarks_user_bubble` ON `bookmarks` (`user_id`, `bubble_id`)

**備考:**
- 1つのバブルに対して、1ユーザーは1回のみブックマーク可能（UNIQUE制約）
- 論理削除に対応（deleted_atを使用）
- `note`フィールドでユーザーがメモを追加可能（将来的な拡張）
- Supabase使用時は`user_id`は`auth.users.id`を参照

---

## 4. データフロー

### 4.1 チャットセッション開始時

1. `chat_sessions`テーブルに新しいセッションを作成
2. `messages`テーブルにAIの初期メッセージ（「今日はどんな内容を学びたいですか？」）を保存
   - `role` = 'assistant'
   - `sequence` = 1
3. 初期メッセージが3つのバブルに分割される場合は、`message_bubbles`テーブルに3レコード作成

### 4.2 ユーザーメッセージ送信時

1. `messages`テーブルにユーザーメッセージを保存
   - `role` = 'user'
   - `sequence` = 前回のメッセージの`sequence` + 1

### 4.3 AIレスポンス生成時

1. `messages`テーブルにAIメッセージを保存
   - `role` = 'assistant'
   - `sequence` = 前回のメッセージの`sequence` + 1
2. AIレスポンスを3つのバブルに分割
3. `message_bubbles`テーブルに3レコード作成
   - `display_order` = 1, 2, 3

### 4.4 ブックマーク保存時

1. `bookmarks`テーブルに新しいレコードを作成
   - `user_id` = 現在のユーザーID
   - `bubble_id` = ブックマーク対象のバブルID

### 4.5 ブックマーク削除時

1. `bookmarks`テーブルの該当レコードの`deleted_at`を更新（論理削除）

---

## 5. 主要なクエリ例

### 5.1 セッション一覧取得

```sql
SELECT 
    cs.id,
    cs.title,
    cs.created_at,
    COUNT(m.id) AS message_count
FROM chat_sessions cs
LEFT JOIN messages m ON cs.id = m.session_id
WHERE cs.user_id = ? AND cs.deleted_at IS NULL
GROUP BY cs.id, cs.title, cs.created_at
ORDER BY cs.created_at DESC;
```

### 5.2 チャット履歴取得（メッセージとバブルを含む）

```sql
SELECT 
    m.id AS message_id,
    m.role,
    m.content AS message_content,
    m.sequence,
    m.created_at AS message_created_at,
    mb.id AS bubble_id,
    mb.bubble_type,
    mb.content AS bubble_content,
    mb.display_order
FROM messages m
LEFT JOIN message_bubbles mb ON m.id = mb.message_id
WHERE m.session_id = ?
ORDER BY m.sequence ASC, mb.display_order ASC;
```

### 5.3 ブックマーク一覧取得

```sql
SELECT 
    b.id AS bookmark_id,
    b.created_at AS bookmarked_at,
    mb.content AS bubble_content,
    mb.bubble_type,
    m.content AS original_message,
    cs.title AS session_title
FROM bookmarks b
INNER JOIN message_bubbles mb ON b.bubble_id = mb.id
INNER JOIN messages m ON mb.message_id = m.id
INNER JOIN chat_sessions cs ON m.session_id = cs.id
WHERE b.user_id = ? AND b.deleted_at IS NULL
ORDER BY b.created_at DESC;
```

---

## 6. 非機能要件への対応

### 6.1 レスポンス速度

- インデックスを適切に設定し、クエリパフォーマンスを最適化
- `message_bubbles`テーブルは`message_id`と`display_order`でインデックス化

### 6.2 スケーラビリティ

- 必要に応じてパーティショニングを検討（例：`messages`テーブルを`created_at`でパーティション分割）
- アーカイブ機能を実装する場合は、古いデータを別テーブルに移動

---

## 7. 実装時の注意事項

### 7.1 Firebase / Supabase使用時

- **Firebase Firestore**: コレクション構造として上記テーブル構造を参考に設計
- **Supabase (PostgreSQL)**: 上記テーブル定義をそのまま使用可能
  - **重要**: Supabase使用時は`auth.users`テーブルが自動的に作成されるため、`public.profiles`テーブルでアプリケーション固有のユーザー情報を管理することを推奨
  - 他のテーブルの`user_id`は`auth.users.id`を参照する
  - RLS（Row Level Security）を有効化し、適切なポリシーを設定する
- リアルタイム更新が必要な場合は、Supabaseのリアルタイム機能やFirebaseのリスナーを活用

### 7.2 データ整合性

- 外部キー制約により、親レコード削除時に子レコードも自動削除（CASCADE）
- ブックマークの重複を防ぐため、`bookmarks`テーブルにUNIQUE制約を設定

### 7.3 セキュリティ

- ユーザー認証を実装し、`user_id`によるアクセス制御を実施
- RLS（Row Level Security）をSupabaseで使用する場合は、各テーブルにポリシーを設定

### 7.4 Supabase使用時の実装例

以下は、Supabaseでテーブルを作成する際のSQL例です：

```sql
-- 1. profilesテーブル（auth.usersと連携）
CREATE TABLE public.profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    name VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- RLSポリシー
CREATE POLICY "Users can view own profile"
    ON public.profiles FOR SELECT
    USING (auth.uid() = id);

CREATE POLICY "Users can update own profile"
    ON public.profiles FOR UPDATE
    USING (auth.uid() = id);

-- 2. chat_sessionsテーブル
CREATE TABLE public.chat_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    title VARCHAR(255),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP
);

ALTER TABLE public.chat_sessions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own sessions"
    ON public.chat_sessions FOR SELECT
    USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own sessions"
    ON public.chat_sessions FOR INSERT
    WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own sessions"
    ON public.chat_sessions FOR UPDATE
    USING (auth.uid() = user_id);

-- 3. messagesテーブル
CREATE TABLE public.messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    sequence INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

ALTER TABLE public.messages ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view messages in own sessions"
    ON public.messages FOR SELECT
    USING (
        EXISTS (
            SELECT 1 FROM public.chat_sessions cs
            WHERE cs.id = messages.session_id
            AND cs.user_id = auth.uid()
        )
    );

-- 4. message_bubblesテーブル
CREATE TABLE public.message_bubbles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id UUID NOT NULL REFERENCES public.messages(id) ON DELETE CASCADE,
    bubble_type VARCHAR(50),
    content TEXT NOT NULL,
    display_order INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

ALTER TABLE public.message_bubbles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view bubbles in own sessions"
    ON public.message_bubbles FOR SELECT
    USING (
        EXISTS (
            SELECT 1 FROM public.messages m
            INNER JOIN public.chat_sessions cs ON m.session_id = cs.id
            WHERE m.id = message_bubbles.message_id
            AND cs.user_id = auth.uid()
        )
    );

-- 5. bookmarksテーブル
CREATE TABLE public.bookmarks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    bubble_id UUID NOT NULL REFERENCES public.message_bubbles(id) ON DELETE CASCADE,
    note TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP,
    UNIQUE(user_id, bubble_id)
);

ALTER TABLE public.bookmarks ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can manage own bookmarks"
    ON public.bookmarks FOR ALL
    USING (auth.uid() = user_id);

-- インデックス作成
CREATE INDEX idx_chat_sessions_user_id ON public.chat_sessions(user_id);
CREATE INDEX idx_chat_sessions_created_at ON public.chat_sessions(created_at);
CREATE INDEX idx_messages_session_id ON public.messages(session_id);
CREATE INDEX idx_messages_session_sequence ON public.messages(session_id, sequence);
CREATE INDEX idx_message_bubbles_message_id ON public.message_bubbles(message_id);
CREATE INDEX idx_message_bubbles_display_order ON public.message_bubbles(message_id, display_order);
CREATE INDEX idx_bookmarks_user_id ON public.bookmarks(user_id);
CREATE INDEX idx_bookmarks_created_at ON public.bookmarks(user_id, created_at DESC);
```

---

## 8. 将来の拡張性

以下の機能追加を想定した設計：

- **ユーザーメモ機能**: `bookmarks.note`フィールドを活用
- **セッション共有機能**: `chat_sessions`テーブルに`is_public`フラグを追加可能
- **学習進捗管理**: 新しいテーブル（`learning_progress`等）を追加可能
- **音声データ保存**: `message_bubbles`テーブルに`audio_url`カラムを追加可能

