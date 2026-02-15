# API設計書：AI英語学習チャットアプリ

## 1. API概要

本システムは、Next.jsのAPI Routesを利用したRESTful APIを提供します。
AIチャット機能、ブックマーク管理機能、音声合成機能を実装します。

### 1.1 ベースURL

- **開発環境**: `http://localhost:3000/api`
- **本番環境**: `https://your-domain.vercel.app/api`

### 1.2 認証方式

Supabase認証を使用し、JWTトークンによる認証を実装します。

- **認証ヘッダー**: `Authorization: Bearer <access_token>`
- トークンはSupabaseクライアントから取得した`access_token`を使用
- 各APIリクエストで認証トークンを検証し、`user_id`を取得

### 1.3 レスポンス形式

#### 成功レスポンス

```json
{
  "success": true,
  "data": { ... }
}
```

#### エラーレスポンス

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ"
  }
}
```

### 1.4 HTTPステータスコード

| コード | 説明 |
|--------|------|
| 200 | 成功 |
| 201 | 作成成功 |
| 400 | リクエストエラー |
| 401 | 認証エラー |
| 403 | 権限エラー |
| 404 | リソースが見つからない |
| 500 | サーバーエラー |

---

## 2. エンドポイント一覧

### 2.1 チャットセッション関連

| メソッド | エンドポイント | 説明 |
|---------|---------------|------|
| POST | `/api/sessions` | 新しいチャットセッションを作成 |
| GET | `/api/sessions` | チャットセッション一覧を取得 |
| GET | `/api/sessions/[sessionId]` | 特定のセッション情報を取得 |
| DELETE | `/api/sessions/[sessionId]` | チャットセッションを削除 |

### 2.2 メッセージ関連

| メソッド | エンドポイント | 説明 |
|---------|---------------|------|
| POST | `/api/sessions/[sessionId]/messages` | ユーザーメッセージを送信し、AIレスポンスを取得（ストリーミング） |
| GET | `/api/sessions/[sessionId]/messages` | セッション内のメッセージ履歴を取得 |

### 2.3 ブックマーク関連

| メソッド | エンドポイント | 説明 |
|---------|---------------|------|
| POST | `/api/bookmarks` | ブックマークを保存 |
| GET | `/api/bookmarks` | ブックマーク一覧を取得 |
| DELETE | `/api/bookmarks/[bookmarkId]` | ブックマークを削除 |

### 2.4 音声関連

| メソッド | エンドポイント | 説明 |
|---------|---------------|------|
| POST | `/api/tts` | テキストを音声に変換（TTS） |

---

## 3. エンドポイント詳細

### 3.1 チャットセッション作成

**エンドポイント**: `POST /api/sessions`

**説明**: 新しいチャットセッションを作成し、AIの初期メッセージを自動送信します。

**認証**: 必須

**リクエストボディ**:

```json
{
  "title": "セッションタイトル（任意）"
}
```

**レスポンス** (201 Created):

```json
{
  "success": true,
  "data": {
    "session": {
      "id": "uuid",
      "user_id": "uuid",
      "title": "セッションタイトル",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    },
    "initial_message": {
      "id": "uuid",
      "role": "assistant",
      "content": "今日はどんな内容を学びたいですか？",
      "sequence": 1,
      "created_at": "2024-01-01T00:00:00Z",
      "bubbles": [
        {
          "id": "uuid",
          "bubble_type": "greeting",
          "content": "今日はどんな内容を学びたいですか？",
          "display_order": 1
        }
      ]
    }
  }
}
```

**エラーレスポンス**:

- `401 Unauthorized`: 認証トークンが無効
- `500 Internal Server Error`: サーバーエラー

---

### 3.2 チャットセッション一覧取得

**エンドポイント**: `GET /api/sessions`

**説明**: 認証済みユーザーのチャットセッション一覧を取得します。

**認証**: 必須

**クエリパラメータ**:

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| limit | number | 任意 | 取得件数（デフォルト: 20） |
| offset | number | 任意 | オフセット（デフォルト: 0） |

**レスポンス** (200 OK):

```json
{
  "success": true,
  "data": {
    "sessions": [
      {
        "id": "uuid",
        "title": "セッションタイトル",
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-01-01T00:00:00Z",
        "message_count": 10
      }
    ],
    "total": 50,
    "limit": 20,
    "offset": 0
  }
}
```

**エラーレスポンス**:

- `401 Unauthorized`: 認証トークンが無効

---

### 3.3 チャットセッション詳細取得

**エンドポイント**: `GET /api/sessions/[sessionId]`

**説明**: 特定のチャットセッションの情報を取得します。

**認証**: 必須

**パスパラメータ**:

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| sessionId | UUID | セッションID |

**レスポンス** (200 OK):

```json
{
  "success": true,
  "data": {
    "session": {
      "id": "uuid",
      "user_id": "uuid",
      "title": "セッションタイトル",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  }
}
```

**エラーレスポンス**:

- `401 Unauthorized`: 認証トークンが無効
- `403 Forbidden`: 他のユーザーのセッションにアクセスしようとした
- `404 Not Found`: セッションが見つからない

---

### 3.4 チャットセッション削除

**エンドポイント**: `DELETE /api/sessions/[sessionId]`

**説明**: チャットセッションを論理削除します。

**認証**: 必須

**パスパラメータ**:

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| sessionId | UUID | セッションID |

**レスポンス** (200 OK):

```json
{
  "success": true,
  "data": {
    "message": "セッションが削除されました"
  }
}
```

**エラーレスポンス**:

- `401 Unauthorized`: 認証トークンが無効
- `403 Forbidden`: 他のユーザーのセッションを削除しようとした
- `404 Not Found`: セッションが見つからない

---

### 3.5 メッセージ送信（AIレスポンス取得）

**エンドポイント**: `POST /api/sessions/[sessionId]/messages`

**説明**: ユーザーメッセージを送信し、AIのレスポンスをストリーミングで取得します。
AIレスポンスは3つの独立したバブル（吹き出し）に分割されて返されます。

**認証**: 必須

**パスパラメータ**:

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| sessionId | UUID | セッションID |

**リクエストボディ**:

```json
{
  "content": "ユーザーのメッセージ内容"
}
```

**レスポンス** (200 OK):

**ストリーミング形式** (Server-Sent Events / SSE):

```
data: {"type":"user_message","data":{"id":"uuid","role":"user","content":"ユーザーのメッセージ","sequence":2,"created_at":"2024-01-01T00:00:00Z"}}

data: {"type":"assistant_message_start","data":{"id":"uuid","role":"assistant","sequence":3}}

data: {"type":"bubble","data":{"id":"uuid","bubble_type":"explanation","content":"解説内容...","display_order":1}}

data: {"type":"bubble","data":{"id":"uuid","bubble_type":"example1","content":"例文1...","display_order":2}}

data: {"type":"bubble","data":{"id":"uuid","bubble_type":"example2","content":"例文2...","display_order":3}}

data: {"type":"assistant_message_complete","data":{"id":"uuid","role":"assistant","content":"完全なメッセージ内容","sequence":3,"created_at":"2024-01-01T00:00:00Z"}}
```

**非ストリーミング形式** (オプション):

```json
{
  "success": true,
  "data": {
    "user_message": {
      "id": "uuid",
      "role": "user",
      "content": "ユーザーのメッセージ",
      "sequence": 2,
      "created_at": "2024-01-01T00:00:00Z"
    },
    "assistant_message": {
      "id": "uuid",
      "role": "assistant",
      "content": "完全なメッセージ内容",
      "sequence": 3,
      "created_at": "2024-01-01T00:00:00Z",
      "bubbles": [
        {
          "id": "uuid",
          "bubble_type": "explanation",
          "content": "解説内容...",
          "display_order": 1
        },
        {
          "id": "uuid",
          "bubble_type": "example1",
          "content": "例文1...",
          "display_order": 2
        },
        {
          "id": "uuid",
          "bubble_type": "example2",
          "content": "例文2...",
          "display_order": 3
        }
      ]
    }
  }
}
```

**エラーレスポンス**:

- `400 Bad Request`: リクエストボディが不正
- `401 Unauthorized`: 認証トークンが無効
- `403 Forbidden`: 他のユーザーのセッションにアクセスしようとした
- `404 Not Found`: セッションが見つからない
- `500 Internal Server Error`: AI APIエラー、サーバーエラー

**実装メモ**:
- Vercel AI SDKを使用してストリーミングレスポンスを実装
- OpenAI APIを呼び出し、レスポンスを3つのバブルに分割
- 各バブルは順次ストリーミングで送信

---

### 3.6 メッセージ履歴取得

**エンドポイント**: `GET /api/sessions/[sessionId]/messages`

**説明**: セッション内のメッセージ履歴を取得します。

**認証**: 必須

**パスパラメータ**:

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| sessionId | UUID | セッションID |

**クエリパラメータ**:

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| limit | number | 任意 | 取得件数（デフォルト: 50） |
| offset | number | 任意 | オフセット（デフォルト: 0） |

**レスポンス** (200 OK):

```json
{
  "success": true,
  "data": {
    "messages": [
      {
        "id": "uuid",
        "role": "user",
        "content": "ユーザーメッセージ",
        "sequence": 1,
        "created_at": "2024-01-01T00:00:00Z",
        "bubbles": null
      },
      {
        "id": "uuid",
        "role": "assistant",
        "content": "AIメッセージ",
        "sequence": 2,
        "created_at": "2024-01-01T00:00:00Z",
        "bubbles": [
          {
            "id": "uuid",
            "bubble_type": "explanation",
            "content": "解説内容...",
            "display_order": 1
          },
          {
            "id": "uuid",
            "bubble_type": "example1",
            "content": "例文1...",
            "display_order": 2
          },
          {
            "id": "uuid",
            "bubble_type": "example2",
            "content": "例文2...",
            "display_order": 3
          }
        ]
      }
    ],
    "total": 20,
    "limit": 50,
    "offset": 0
  }
}
```

**エラーレスポンス**:

- `401 Unauthorized`: 認証トークンが無効
- `403 Forbidden`: 他のユーザーのセッションにアクセスしようとした
- `404 Not Found`: セッションが見つからない

---

### 3.7 ブックマーク保存

**エンドポイント**: `POST /api/bookmarks`

**説明**: メッセージバブルをブックマークに保存します。

**認証**: 必須

**リクエストボディ**:

```json
{
  "bubble_id": "uuid",
  "note": "ユーザーメモ（任意）"
}
```

**レスポンス** (201 Created):

```json
{
  "success": true,
  "data": {
    "bookmark": {
      "id": "uuid",
      "user_id": "uuid",
      "bubble_id": "uuid",
      "note": "ユーザーメモ",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z",
      "bubble": {
        "id": "uuid",
        "bubble_type": "explanation",
        "content": "ブックマークした内容...",
        "display_order": 1,
        "message": {
          "id": "uuid",
          "content": "元のメッセージ内容",
          "session": {
            "id": "uuid",
            "title": "セッションタイトル"
          }
        }
      }
    }
  }
}
```

**エラーレスポンス**:

- `400 Bad Request`: リクエストボディが不正、または既にブックマーク済み
- `401 Unauthorized`: 認証トークンが無効
- `404 Not Found`: バブルが見つからない
- `409 Conflict`: 既にブックマーク済み（UNIQUE制約違反）

---

### 3.8 ブックマーク一覧取得

**エンドポイント**: `GET /api/bookmarks`

**説明**: 認証済みユーザーのブックマーク一覧を取得します。

**認証**: 必須

**クエリパラメータ**:

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| limit | number | 任意 | 取得件数（デフォルト: 20） |
| offset | number | 任意 | オフセット（デフォルト: 0） |
| order_by | string | 任意 | ソート順（`created_at_desc` / `created_at_asc`、デフォルト: `created_at_desc`） |

**レスポンス** (200 OK):

```json
{
  "success": true,
  "data": {
    "bookmarks": [
      {
        "id": "uuid",
        "bubble_id": "uuid",
        "note": "ユーザーメモ",
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-01-01T00:00:00Z",
        "bubble": {
          "id": "uuid",
          "bubble_type": "explanation",
          "content": "ブックマークした内容...",
          "display_order": 1
        },
        "message": {
          "id": "uuid",
          "content": "元のメッセージ内容",
          "created_at": "2024-01-01T00:00:00Z"
        },
        "session": {
          "id": "uuid",
          "title": "セッションタイトル",
          "created_at": "2024-01-01T00:00:00Z"
        }
      }
    ],
    "total": 100,
    "limit": 20,
    "offset": 0
  }
}
```

**エラーレスポンス**:

- `401 Unauthorized`: 認証トークンが無効

---

### 3.9 ブックマーク削除

**エンドポイント**: `DELETE /api/bookmarks/[bookmarkId]`

**説明**: ブックマークを論理削除します。
**注意**: フロントエンド側で削除確認モーダルを表示してからこのAPIを呼び出すこと。

**認証**: 必須

**パスパラメータ**:

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| bookmarkId | UUID | ブックマークID |

**レスポンス** (200 OK):

```json
{
  "success": true,
  "data": {
    "message": "ブックマークが削除されました"
  }
}
```

**エラーレスポンス**:

- `401 Unauthorized`: 認証トークンが無効
- `403 Forbidden`: 他のユーザーのブックマークを削除しようとした
- `404 Not Found`: ブックマークが見つからない

---

### 3.10 音声合成（TTS）

**エンドポイント**: `POST /api/tts`

**説明**: テキストを音声に変換します。Web Speech APIを使用する場合はフロントエンド側で実装し、このAPIは不要です。
OpenAI TTS APIを使用する場合は、このエンドポイントで音声データを生成します。

**認証**: 必須（オプション - 使用量制限のため推奨）

**リクエストボディ**:

```json
{
  "text": "読み上げるテキスト",
  "voice": "alloy" // オプション: alloy, echo, fable, onyx, nova, shimmer
}
```

**レスポンス** (200 OK):

**Content-Type**: `audio/mpeg` または `audio/ogg`

音声データをバイナリ形式で返します。

**エラーレスポンス**:

- `400 Bad Request`: リクエストボディが不正
- `401 Unauthorized`: 認証トークンが無効
- `500 Internal Server Error`: TTS APIエラー

**実装メモ**:
- OpenAI TTS APIを使用する場合のみ実装
- Web Speech APIを使用する場合は、フロントエンド側で実装し、このAPIは不要

---

## 4. エラーハンドリング

### 4.1 エラーコード一覧

| コード | HTTPステータス | 説明 |
|--------|---------------|------|
| `AUTH_REQUIRED` | 401 | 認証が必要 |
| `AUTH_INVALID` | 401 | 認証トークンが無効 |
| `FORBIDDEN` | 403 | アクセス権限がない |
| `NOT_FOUND` | 404 | リソースが見つからない |
| `VALIDATION_ERROR` | 400 | バリデーションエラー |
| `DUPLICATE_BOOKMARK` | 409 | 既にブックマーク済み |
| `AI_API_ERROR` | 500 | AI API呼び出しエラー |
| `TTS_API_ERROR` | 500 | TTS API呼び出しエラー |
| `DATABASE_ERROR` | 500 | データベースエラー |
| `SUPABASE_ERROR` | 500 | Supabase API呼び出しエラー |
| `RLS_VIOLATION` | 403 | RLSポリシー違反（通常はAPI RoutesでService Role Keyを使用するため発生しない） |
| `INTERNAL_ERROR` | 500 | 内部サーバーエラー |

### 4.2 エラーレスポンス例

**バリデーションエラー**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "リクエストボディが不正です",
    "details": {
      "field": "content",
      "reason": "必須フィールドが不足しています"
    }
  }
}
```

**Supabaseエラー**

Supabaseからエラーが返された場合、エラーメッセージを適切に処理します。

```json
{
  "success": false,
  "error": {
    "code": "DATABASE_ERROR",
    "message": "データベースエラーが発生しました",
    "details": {
      "supabase_code": "23505",
      "supabase_message": "duplicate key value violates unique constraint"
    }
  }
}
```

**実装時の注意**: Supabaseのエラーコード（例: `23505`はUNIQUE制約違反）を適切に処理し、ユーザーフレンドリーなエラーメッセージに変換してください。

---

## 5. 認証実装詳細

### 5.1 認証フロー

1. フロントエンドでSupabaseクライアントを使用してユーザー認証
2. 認証成功後、`access_token`を取得
3. 各APIリクエストの`Authorization`ヘッダーに`Bearer <access_token>`を設定
4. API側でSupabaseクライアントを使用してトークンを検証
5. 検証成功後、`user_id`を取得してリクエストを処理

### 5.2 認証ミドルウェア例（Next.js）

**方法1: Service Role Keyを使用（推奨）**

Service Role Keyを使用すると、RLS（Row Level Security）をバイパスできます。
API Routesでは、認証済みユーザーのみがアクセスできるため、この方法が推奨されます。

```typescript
// lib/auth.ts
import { createClient } from '@supabase/supabase-js'

export async function verifyAuth(request: Request): Promise<string | null> {
  const authHeader = request.headers.get('Authorization')
  if (!authHeader?.startsWith('Bearer ')) {
    return null
  }
  
  const token = authHeader.substring(7)
  
  // Service Role Keyを使用してトークンを検証
  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )
  
  const { data: { user }, error } = await supabase.auth.getUser(token)
  if (error || !user) {
    return null
  }
  
  return user.id
}
```

**方法2: Anon Key + User Tokenを使用**

Anon Keyとユーザートークンを使用する場合、RLSポリシーが適用されます。
この方法では、ユーザーは自分のデータのみにアクセスできます。

```typescript
// lib/auth.ts
import { createClient } from '@supabase/supabase-js'

export async function createAuthenticatedClient(request: Request) {
  const authHeader = request.headers.get('Authorization')
  if (!authHeader?.startsWith('Bearer ')) {
    return null
  }
  
  const token = authHeader.substring(7)
  
  // Anon Keyとユーザートークンを使用
  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      global: {
        headers: {
          Authorization: `Bearer ${token}`
        }
      }
    }
  )
  
  const { data: { user }, error } = await supabase.auth.getUser()
  if (error || !user) {
    return null
  }
  
  return { supabase, userId: user.id }
}
```

**推奨**: 方法1（Service Role Key）を使用し、API Routes内で明示的に`user_id`をチェックしてアクセス制御を実装します。

---

## 6. ストリーミング実装詳細

### 6.1 Server-Sent Events (SSE) 形式

Vercel AI SDKを使用してストリーミングレスポンスを実装します。

```typescript
// app/api/sessions/[sessionId]/messages/route.ts
import { streamText } from 'ai'
import { openai } from '@ai-sdk/openai'

export async function POST(request: Request) {
  // 認証処理...
  
  const { content } = await request.json()
  
  const result = streamText({
    model: openai('gpt-4o'),
    prompt: content,
    // 3つのバブルに分割するためのプロンプト設定
  })
  
  return result.toDataStreamResponse()
}
```

### 6.2 ストリーミングイベント形式

- `user_message`: ユーザーメッセージが保存された
- `assistant_message_start`: AIメッセージの生成が開始された
- `bubble`: 各バブルが生成された（複数回送信）
- `assistant_message_complete`: AIメッセージの生成が完了した

---

## 7. データベース連携

### 7.1 Supabaseクライアント設定

**サーバーサイド用クライアント（API Routes）**

API Routesでは、Service Role Keyを使用してSupabaseクライアントを作成します。
このクライアントはRLSをバイパスするため、API Routes内で明示的に`user_id`をチェックする必要があります。

```typescript
// lib/supabase-server.ts
import { createClient } from '@supabase/supabase-js'

export function createServerClient() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    {
      auth: {
        autoRefreshToken: false,
        persistSession: false
      }
    }
  )
}
```

**クライアントサイド用クライアント（フロントエンド）**

フロントエンドでは、Anon Keyを使用してSupabaseクライアントを作成します。
このクライアントはRLSポリシーに従い、ユーザーは自分のデータのみにアクセスできます。

```typescript
// lib/supabase-client.ts
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)
```

### 7.2 RLS（Row Level Security）について

DB設計書に記載されているRLSポリシーにより、データベースレベルでアクセス制御が行われます。

**重要**: API RoutesでService Role Keyを使用する場合、RLSはバイパスされます。
そのため、API Routes内で以下のように明示的に`user_id`をチェックする必要があります：

```typescript
// 例: セッション取得時のアクセス制御
const { data: session, error } = await supabase
  .from('chat_sessions')
  .select('*')
  .eq('id', sessionId)
  .eq('user_id', userId) // 明示的にuser_idをチェック
  .single()

if (!session || error) {
  return new Response(
    JSON.stringify({ success: false, error: { code: 'NOT_FOUND', message: 'セッションが見つかりません' } }),
    { status: 404 }
  )
}
```

### 7.3 主要なクエリ例

**セッション作成**

```typescript
const { data: session, error } = await supabase
  .from('chat_sessions')
  .insert({
    user_id: userId,
    title: title || null
  })
  .select()
  .single()
```

**メッセージ保存**

```typescript
const { data: message, error } = await supabase
  .from('messages')
  .insert({
    session_id: sessionId,
    role: 'user',
    content: content,
    sequence: nextSequence
  })
  .select()
  .single()
```

**バブル保存（複数）**

```typescript
const { data: bubbles, error } = await supabase
  .from('message_bubbles')
  .insert(
    bubbles.map((bubble, index) => ({
      message_id: messageId,
      bubble_type: bubble.type,
      content: bubble.content,
      display_order: index + 1
    }))
  )
  .select()
```

**ブックマーク保存**

```typescript
const { data: bookmark, error } = await supabase
  .from('bookmarks')
  .insert({
    user_id: userId,
    bubble_id: bubbleId,
    note: note || null
  })
  .select()
  .single()
```

**ブックマーク削除（論理削除）**

```typescript
const { error } = await supabase
  .from('bookmarks')
  .update({ deleted_at: new Date().toISOString() })
  .eq('id', bookmarkId)
  .eq('user_id', userId) // アクセス制御
```

詳細なテーブル定義とRLSポリシーはDB設計書を参照してください。

---

## 8. 非機能要件への対応

### 8.1 レスポンス速度

- ストリーミングレスポンスにより、AIの応答を逐次表示
- データベースクエリにインデックスを適切に設定
- Vercelのエッジネットワークを活用

### 8.2 セキュリティ

- Supabase認証による安全なユーザー管理
- RLS（Row Level Security）によるデータアクセス制御
- APIキー等の機密情報は環境変数で管理
- 入力値のバリデーション実装

### 8.3 可用性

- Vercelの自動スケーリング機能を活用
- Supabaseのリアルタイム同期機能を活用（必要に応じて）
- エラーハンドリングとリトライロジックの実装

### 8.4 Supabaseリアルタイム機能

ブックマーク一覧などのリアルタイム更新が必要な場合は、Supabaseのリアルタイム機能を使用できます。

**フロントエンドでの実装例**:

```typescript
// ブックマークのリアルタイム購読
const channel = supabase
  .channel('bookmarks')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'bookmarks',
      filter: `user_id=eq.${userId}`
    },
    (payload) => {
      // ブックマークの変更を処理
      console.log('Bookmark changed:', payload)
    }
  )
  .subscribe()
```

**注意**: API Routesではリアルタイム機能は使用しません。
リアルタイム機能はフロントエンド側で実装してください。

---

## 9. 実装時の注意事項

### 9.1 環境変数

以下の環境変数を設定してください：

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# OpenAI
OPENAI_API_KEY=your_openai_api_key

# Vercel
VERCEL_URL=your_vercel_url
```

**重要**: 
- `NEXT_PUBLIC_SUPABASE_URL`と`NEXT_PUBLIC_SUPABASE_ANON_KEY`はフロントエンドで使用されるため、`NEXT_PUBLIC_`プレフィックスが必要です。
- `SUPABASE_SERVICE_ROLE_KEY`はサーバーサイドでのみ使用し、**絶対にフロントエンドに公開しないでください**。
- Vercelの環境変数設定で、本番環境と開発環境で適切に設定してください。

### 9.2 CORS設定

Next.jsのAPI Routesは、同じオリジンからのリクエストのみを受け付けるため、通常はCORS設定は不要です。
外部からのアクセスが必要な場合は、適切なCORS設定を実装してください。

### 9.3 レート制限

必要に応じてレート制限を実装してください（例: Vercel Edge Config、Upstash等を使用）。

### 9.4 Supabase使用時の注意事項

**RLS（Row Level Security）**

- DB設計書に記載されているRLSポリシーが有効になっていることを確認してください。
- API RoutesでService Role Keyを使用する場合、RLSはバイパスされます。
- そのため、API Routes内で明示的に`user_id`をチェックしてアクセス制御を実装してください。

**トランザクション処理**

- Supabaseでは、複数のINSERT/UPDATE操作をトランザクションで実行する場合は、PostgreSQLのトランザクション機能を使用します。
- 例: メッセージとバブルの保存を同時に行う場合

```typescript
// トランザクション例（SupabaseではRPC関数を使用）
const { data, error } = await supabase.rpc('create_message_with_bubbles', {
  p_session_id: sessionId,
  p_role: 'assistant',
  p_content: content,
  p_sequence: sequence,
  p_bubbles: bubbles
})
```

**接続プール**

- Supabaseは自動的に接続プールを管理します。
- 大量のリクエストを処理する場合は、Supabaseの接続制限に注意してください。

**エラーハンドリング**

- Supabaseのエラーは`error`オブジェクトに含まれます。
- エラーコード（例: `23505`はUNIQUE制約違反）を適切に処理してください。

---

## 10. 将来の拡張性

以下の機能追加を想定した設計：

- **セッション共有機能**: セッションを公開する機能
- **学習進捗管理**: 学習履歴を記録する機能
- **音声データ保存**: TTSで生成した音声を保存する機能
- **バッチ処理**: 大量データの一括処理機能

