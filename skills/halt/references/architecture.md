# HALT アーキテクチャ リファレンス

HTMX + Atomic Design + Lit + Templ によるサーバー内蔵型フロントエンドアーキテクチャ。

## 設計思想

- フロントエンドはAPIサーバーの中で動く。SPA は作らない
- サーバーがUI制御を握り、HTMX でハイパーメディア駆動のインタラクションを実現する
- リッチなインタラクションが必要な箇所だけ Lit Web Components で拡張する
- テンプレートは Atomic Design で構造化する

## 技術スタック

| レイヤー | 技術 | 役割 |
|---------|------|------|
| サーバーフレームワーク | Go + Gin | ルーティング + SSR |
| API フレームワーク | Huma | OpenAPI 3.1 自動生成、入出力バリデーション |
| テンプレートエンジン | Templ | 型安全な Go HTML テンプレート |
| インタラクション | HTMX | サーバー駆動の DOM 更新 |
| リッチUI | Lit (Web Components) | エディタ等の高度なインタラクション |
| スタイリング | Tailwind CSS | ユーティリティファースト CSS |
| ビルド | esbuild | TypeScript バンドル |

## ディレクトリ構成

### APIサーバー（レイヤードアーキテクチャ）

```
backend/
├── cmd/                    # エントリーポイント
├── internal/
│   ├── domain/             # ドメインモデル・ビジネスロジック
│   ├── repository/         # データアクセス層
│   ├── service/            # アプリケーションサービス層
│   ├── router/             # ルーティング定義（SSR + API）
│   └── web/                # Web層（以下詳細）
│       ├── handler/        # HTTPハンドラ（機能ドメイン別）
│       │   ├── auth.go
│       │   ├── workspace.go
│       │   ├── project.go
│       │   ├── task.go
│       │   ├── document.go
│       │   └── ...
│       ├── atoms/          # 基本要素（button, input, badge, avatar 等）
│       ├── molecules/      # 複合要素（card, modal, tag_select 等）
│       ├── organisms/      # 機能単位（header, sidebar, command_palette 等）
│       ├── pages/          # ページテンプレート
│       └── layout/         # レイアウトラッパー
└── static/
    └── src/
        ├── components/     # Lit Web Components（後述）
        │   └── lib/        # 共通ユーティリティ
        ├── css/            # Tailwind CSS エントリーポイント
        └── dist/           # ビルド成果物
```

### Lit Web Components

エディタ等、リッチなインタラクションが必要な箇所のみ Web Components として実装する。

```
src/components/
├── {feature-name}/         # 機能単位のコンポーネント
│   └── {feature-name}.ts   # Lit カスタム要素
└── lib/
    ├── api.ts              # HTTP クライアント（CSRF トークン自動付与）
    ├── ws.ts               # WebSocket ラッパー（自動再接続）
    └── logger.ts           # ロギングユーティリティ
```

**命名規則**: `<app-prefix>-{feature-name>` （例: `<my-doc-editor>`, `<my-kanban-board>`）

### ビルド成果物

```
dist/
├── js/
│   ├── {component-name}.js  # 各 Web Component のバンドル
│   └── htmx.min.js          # HTMX ライブラリ
└── css/
    └── app.css               # Tailwind CSS ビルド済み
```

## Templ テンプレート（Atomic Design）

### atoms（原子）

最小のUI要素。単一の責務を持つ。

- button, input, textarea, select
- avatar, badge, spinner
- toast（通知トースト）

### molecules（分子）

atoms を組み合わせた複合要素。

- card, modal
- tag_select, task_row
- doc_item

### organisms（有機体）

molecules と atoms を組み合わせた機能単位。

- header, sidebar
- notification_list
- document_tree_node
- command_palette

### pages（ページ）

organisms を組み合わせたページ全体のテンプレート。

- login, register
- workspace 一覧・詳細
- project 一覧・詳細
- document エディタ、canvas エディタ
- settings

## HTMX パターン

### 基本方針

- サーバーが HTML フラグメントを返し、HTMX が DOM を差し替える
- JSON API は外部クライアント向け。HTMX は HTML エンドポイントを使う
- フォーム送信、リスト更新、モーダル管理は HTMX で処理する

### ルーティング

```
# SSR ルート — Gin 直接（HTML を返す）
GET  /workspaces                → ワークスペース一覧ページ
GET  /projects/:id/tasks        → タスクボード
GET  /projects/:id/docs         → ドキュメント一覧
GET  /documents/:id/edit        → エディタ（Lit Component をレンダリング）

# API ルート — Huma 経由（JSON + OpenAPI 自動生成）
GET  /api/v1/workspaces         → JSON レスポンス
POST /api/v1/tasks              → JSON レスポンス
GET  /api/v1/openapi.json       → OpenAPI 3.1 スペック（Huma 自動生成）

# HTMX 部分更新 — Gin 直接（HTML フラグメントを返す）
PUT  /notifications/read-all    → 通知リスト部分 HTML
POST /tasks/:id/status          → タスク行 HTML
```

**使い分け**: SSR ルートと HTMX 部分更新は Gin ハンドラで直接処理。JSON API は Huma でラップし、OpenAPI スペック自動生成・入出力バリデーションの恩恵を受ける。

## Lit Web Components パターン

### 使用基準

以下に該当する場合のみ Web Component を作成する：

- リアルタイム編集（ドキュメントエディタ、キャンバス）
- 複雑なドラッグ＆ドロップ（カンバン、並び替え）
- Canvas / WebGL 描画
- WebSocket によるリアルタイム同期

HTMX で十分なインタラクション（フォーム、リスト更新、モーダル等）には Web Component を使わない。

### コンポーネント設計

```typescript
@customElement('my-feature-name')
export class MyFeatureName extends LitElement {
  // HTML属性からデータを受け取る
  @property({ type: String }) entityId = '';
  @property({ type: Boolean }) readonly = false;

  // コンポーネント内部状態
  @state() private data: SomeType | null = null;

  // Shadow DOM スタイル
  static styles = css`...`;

  // ライフサイクル
  connectedCallback() { super.connectedCallback(); this.load(); }
  disconnectedCallback() { super.disconnectedCallback(); this.cleanup(); }
}
```

### 特徴的パターン

- **自動保存**: 編集後一定時間（2-3秒）の無操作で自動保存
- **読み取り専用モード**: 全エディタが `readonly` プロパティで切替可能
- **Shadow DOM**: スタイルのカプセル化。外部 CSS の影響を受けない
- **イベント通信**: コンポーネント間は CustomEvent で疎結合に通信

## API クライアント（lib/api.ts）

```typescript
// CSRF トークンを meta タグから自動取得
// credentials: 'same-origin' で Cookie 送信
// レスポンスは JSON 自動パース

get<T>(path: string): Promise<T>
post<T>(path: string, body?): Promise<T>
put<T>(path: string, body?): Promise<T>
patch<T>(path: string, body?): Promise<T>
del<T>(path: string): Promise<T>
uploadFile<T>(path: string, file: File): Promise<T>
```

## WebSocket（lib/ws.ts）

- 自動再接続（指数バックオフ、最大10回リトライ）
- JSON メッセージのパース
- `onOpen`, `onMessage`, `onClose` コールバック

## セキュリティ

- CSRF トークン: meta タグから取得し全リクエストに付与
- 認証: Cookie ベースのセッション（SSR）+ Bearer トークン / API キー（API）
- 読み取り専用モード: 権限に応じてエディタを readonly で描画

## スタイリング方針

| 対象 | 手法 |
|------|------|
| Templ テンプレート | Tailwind CSS ユーティリティクラス |
| Lit Web Components | Shadow DOM 内の css テンプレートリテラル |
| ダークモード | CSS カスタム変数 + Tailwind のダークモード |

## ビルドスクリプト

```json
{
  "build": "npm run build:css && npm run build:js && npm run copy:htmx",
  "build:js": "node esbuild.config.mjs",
  "build:css": "npx @tailwindcss/cli -i src/css/app.css -o dist/css/app.css --minify",
  "copy:htmx": "cp node_modules/htmx.org/dist/htmx.min.js dist/js/",
  "dev": "concurrently \"npm run dev:css\" \"npm run dev:js\""
}
```

**esbuild 設定**:
- エントリーポイント: 各 Web Component ごと
- 出力: ES モジュール形式
- 本番: minify 有効、開発: sourcemap 有効
