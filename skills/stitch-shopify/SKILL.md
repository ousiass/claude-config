---
name: stitch-shopify
description: Stitchモックzipを展開しShopify OS 2.0テーマ（Atomic Design snippets構造）に変換する
user-invocable: true
---

# Stitch Shopify

Stitchで生成したモックzipをShopify Online Store 2.0のテーマディレクトリに変換する。
snippets階層をAtomic Design（atoms / molecules / organisms）で構造化し、テキストは`section.settings`、画像は`image_picker`に差し替えたカスタマイズ可能なテーマを生成する。

## 引数

```
/stitch-shopify [zipファイルパス] [出力先ディレクトリ(省略時: zipと同階層の shopify-theme/)]
```

## 前提

- 入力: Stitchの生zip（`stitch/p_XXX/code.html` 形式、`/stitch-deploy` と同じ構造）
- 出力: `shopify theme dev` / `shopify theme push` でそのまま動作するテーマディレクトリ
- CSS: Tailwind CDNを `theme.liquid` に置いて維持する（再コンパイルしない）
- `code.html` のHTML中身は、Liquid化に必要な箇所以外は極力変更しない
- ユーザー確認なしに既存ファイルを上書きしない

## 出力ディレクトリ構造

```
shopify-theme/
├── assets/                     # screen.png 等を配置
├── config/
│   ├── settings_schema.json
│   └── settings_data.json
├── layout/
│   └── theme.liquid            # Tailwind CDN + 共通head
├── sections/
│   ├── header.liquid
│   ├── footer.liquid
│   └── main-*.liquid           # ページごと（main-index, main-product 等）
├── snippets/
│   ├── atoms/*.liquid          # button, heading, input, icon 等
│   ├── molecules/*.liquid      # form-field, product-card, search-bar 等
│   └── organisms/*.liquid      # hero, product-grid, article-list 等
├── templates/
│   ├── index.json
│   ├── product.json
│   ├── collection.json
│   ├── cart.json
│   ├── page.json
│   ├── blog.json
│   ├── article.json
│   ├── 404.json
│   └── search.json
└── locales/
    └── en.default.json
```

snippetsのサブディレクトリは `{% render 'atoms/button' %}` のようにパス付きで参照する（Shopify CLI 3系以降が対応）。

## ワークフロー

### Phase 1: 準備

1. 引数からzipパスを取得。未指定なら作業ディレクトリ内の `stitch*.zip` を探す
2. zipを一時ディレクトリに展開
3. 出力先ディレクトリの候補を決定（引数 > `zipと同階層/shopify-theme/`）

### Phase 2: ページ解析とテンプレートマッピング

各 `p_XXX/code.html` について：

1. ページ名推定（`/stitch-deploy` と同ロジック）：
   `<title>` → `<h1>` → `<h2>` → ディレクトリサフィックスの順
2. Shopify標準テンプレートに自動マッピング（キーワード照合）：
   - ホーム/トップ/Home → `index`
   - 商品一覧/コレクション/Collection → `collection`
   - 商品詳細/Product → `product`
   - カート/Cart → `cart`
   - 404/見つかりません → `404`
   - 検索/Search → `search`
   - ブログ/Blog → `blog`
   - 記事/Article → `article`
   - その他（About、問い合わせ等）→ `page.{slug}` 代替テンプレート

マッピング表をユーザーに提示し承認を得る：

```
番号 | Stitch   | 推定ページ名 | Shopifyテンプレート
-----|----------|-------------|---------------------
1    | p_001    | ホーム       | index
2    | p_002    | 商品一覧     | collection
3    | p_003    | 商品詳細     | product
4    | p_004    | About       | page.about
...
```

ユーザーが修正指示を出したら反映。承認されるまで次に進まない。

#### 欠落テンプレートの確認

Shopify標準9種（`index`, `product`, `collection`, `cart`, `page`, `blog`, `article`, `404`, `search`）のうちモックに無いものを列挙し、**AskUserQuestionで各テンプレートごとに対応を選ばせる**：

- **自動雛形生成**: 最小構成のsection（タイトル＋本文のsettingsのみ）を生成
- **Stitchで再生成**: 処理を一旦中断し、ユーザーがStitchで追加モックを生成後に再実行
- **今回はスキップ**: テンプレートを作らない

「Stitchで再生成」が1つでも選ばれたら、その時点で処理を終え「追加分を含めた新しいzipで再実行してください」と案内する。

### Phase 3: Atomic Design分解

全ページのHTMLを横断的にスキャンし、再利用可能な要素を識別する。
**判定基準とTailwindクラス署名の早見表は `references/atomic-patterns.md` を参照**（分類に迷ったら必ず読む）。

**分類基準**：

- **atoms** — 単一要素、最小単位（button, heading, input, textarea, select, icon, badge, label, image）
- **molecules** — atomsを2〜3個束ねた機能単位（form-field, search-bar, product-card, nav-item, breadcrumb）
- **organisms** — moleculesとatomsを組み合わせた独立セクション（site-header, site-footer, hero, product-grid, article-list, cta-section）

**抽出手順**：

1. 全ページを走査し、Tailwindクラスの類似度で同種要素をグループ化
2. 2回以上出現するパターンを候補に挙げる（1回きりはsection内にインライン）
3. 候補リストをユーザーに提示：

```
■ atoms 候補
- button-primary   (出現: 23回) - bg-blue-600 text-white rounded
- button-secondary (出現: 8回)  - border text-blue-600
- heading-section  (出現: 12回) - text-2xl font-bold
- input-text       (出現: 15回) - border rounded px-3 py-2
...

■ molecules 候補
- product-card  (出現: 15回) - 画像+商品名+価格+ボタン
- form-field    (出現: 9回)  - label+input+error
...

■ organisms 候補
- site-header  (全ページ)
- site-footer  (全ページ)
- hero         (index, collection)
- product-grid (index, collection)
...
```

ユーザーが粒度を調整（分ける/まとめる/命名変更）。承認されるまで次に進まない。

### Phase 4: Shopifyテーマ生成

出力先ディレクトリが存在する場合は上書き確認を取ってから実行。

#### 4-1. layout/theme.liquid

- Stitchの共通`<head>`（font, meta等）を移植
- Tailwind CDN `<script src="https://cdn.tailwindcss.com"></script>` を追加
- `{{ content_for_header }}`, `{% section 'header' %}`, `{{ content_for_layout }}`, `{% section 'footer' %}` を配置

#### 4-2. snippets/ の生成

承認済みのatoms/molecules/organismsを個別Liquidファイルに：

- **atoms**: propsのみから描画、ロジック無し
  ```liquid
  {% comment %} atoms/button.liquid
    params: label (string), variant (primary|secondary), href (string, optional)
  {% endcomment %}
  {% assign variant = variant | default: 'primary' %}
  {% if variant == 'primary' %}
    {% assign classes = 'bg-blue-600 text-white rounded px-4 py-2' %}
  {% else %}
    {% assign classes = 'border text-blue-600 rounded px-4 py-2' %}
  {% endif %}
  {% if href %}
    <a href="{{ href }}" class="{{ classes }}">{{ label }}</a>
  {% else %}
    <button class="{{ classes }}">{{ label }}</button>
  {% endif %}
  ```

- **molecules**: atomsを`{% render %}`で組み立てる
- **organisms**: 通常sectionから呼ばれる想定。sectionのsettings/blocksをプロパティで受け取る

#### 4-3. sections/ の生成

- `header.liquid`, `footer.liquid`: organisms/site-header, site-footer を呼ぶ薄いラッパー。`{% schema %}` 付き
- ページごとに `main-{template}.liquid`（main-index.liquid, main-product.liquid 等）

**setting type / block / presets の構文は `references/shopify-settings.md` を参照**（schema生成時は必ず確認）。

各sectionに`{% schema %}`を付与し、以下を変換：

| 元の要素 | 変換後 | setting type |
|---------|--------|-------------|
| プレーンテキスト（見出し、段落） | `{{ section.settings.xxx }}` | `text` / `textarea` |
| リッチテキスト（リンク含む段落） | `{{ section.settings.xxx }}` | `richtext` |
| 画像 | `{{ section.settings.image \| image_url: width: 800 \| image_tag }}` | `image_picker` |
| CTAボタン（テキスト＋URL） | setting 2つ（`cta_label`, `cta_url`） | `text`, `url` |
| リスト状の繰り返し（特徴、実績、FAQ等） | `{% for block in section.blocks %}` | blocks配列で定義 |

schemaのpresetsも1つ入れ、テーマエディタからsection追加できるようにする。

#### 4-4. 商品・コレクションsectionのLiquid化

`main-product.liquid` / `main-collection.liquid` のダミーデータ部分は以下に置換：

- 商品リスト → `{% for product in collection.products %}`
- 商品タイトル → `{{ product.title }}`
- 商品価格 → `{{ product.price | money }}`
- 商品画像 → `{{ product.featured_image | image_url: width: 600 | image_tag }}`
- 商品リンク → `{{ product.url }}`
- 商品説明 → `{{ product.description }}`
- 商品フォーム → `{% form 'product', product %}...{% endform %}`

メイン商品/コレクション領域以外（お客様の声、特徴紹介等）はsettings/blocks化。

#### 4-5. templates/

Shopify OS 2.0 JSON形式。該当する `main-{template}` section を参照：

```json
{
  "sections": {
    "main": { "type": "main-index", "settings": {} }
  },
  "order": ["main"]
}
```

Phase 2で決めた代替テンプレート（例: `page.about.json`）も同様に生成。

#### 4-6. config/, locales/, assets/

- `config/settings_schema.json`: `theme_info` のみ最小構成
- `config/settings_data.json`: `{"current": {}}`
- `locales/en.default.json`: `{}`（空。将来の翻訳用）
- `assets/`: 各 `screen.png` を `assets/mock-{slug}.png` で配置（参考素材として。theme自体では使わない）

### Phase 5: 自己チェック

以下の整合性チェックを行い結果を報告：

- 全sectionに `{% schema %}` があるか
- `templates/*.json` の `type` が実在するsectionを指しているか
- `{% render 'xxx/yyy' %}` の参照先snippetが全て存在するか
- snippets間の引数名が渡し側と受け側で一致しているか
- `assets/`, `config/`, `layout/`, `sections/`, `snippets/`, `templates/`, `locales/` が全て存在するか

異常があれば修正してから次フェーズへ。

### Phase 6: Git commit & push（自動）

出力ディレクトリを含むgitリポジトリを検出して自動でcommit/pushする。

1. **リポジトリ判定**: 出力先から `git rev-parse --show-toplevel` を実行
   - **gitリポジトリ内の場合**: そのリポジトリで commit/push
   - **gitリポジトリ外の場合**: AskUserQuestionで以下を選ばせる
     - `git init` して初期コミットを作る（remoteは後で設定）
     - gitステップをスキップ

2. **ステージング**: 出力先ディレクトリのみを対象にする
   ```
   git add <出力先ディレクトリ>
   ```
   他のワーキングツリーの変更は巻き込まない。

3. **コミットメッセージ**: `CLAUDE.md` の規約に従い `<type>: <説明>` 形式
   - 初回生成: `feat: Stitchモックから生成したShopifyテーマを追加`
   - 既存上書き: `update: Stitchモックから生成したShopifyテーマを更新`
   - type は英語、説明は日本語
   - `--no-verify` 禁止。pre-commit/pre-pushが失敗したら原因を調査・修正してから再実行

4. **プッシュ前の確認**: 以下を1画面で提示してユーザーに最終確認：
   ```
   ブランチ: <current-branch>
   リモート: <remote-url>
   コミット: <commit-message>
   変更ファイル数: <N>
   ```
   承認後に `git push` を実行。リモートが未設定なら push はスキップし案内のみ。

5. **現在ブランチがmain/masterの場合は警告**: 新規ブランチ作成を推奨し、AskUserQuestionで
   - 新規ブランチ `stitch-shopify/<日付>` を切って commit/push
   - そのままmain/masterに commit/push
   を選ばせる。

### Phase 7: 完了案内

1. 出力ディレクトリの `tree` を表示
2. Phase 5 の自己チェック結果と Phase 6 の git 操作結果をサマリ表示
3. 次アクションを案内：
   ```
   cd <出力先>
   shopify theme check     # 静的検査
   shopify theme dev       # ローカルプレビュー
   shopify theme push -u   # 未公開テーマとしてアップロード
   ```

## ルール

- ユーザー確認を取る箇所（必須）：
  1. ページ→テンプレートのマッピング表
  2. 欠落テンプレートの対応（AskUserQuestion）
  3. Atomic Design分解の粒度
  4. 出力先ディレクトリが既存の場合の上書き
  5. git push 直前の最終確認（ブランチ/リモート/コミット）
  6. main/masterブランチへの直pushを避けるかの確認
- テキストは必ずsettings化する（ハードコードしない）
- 画像は必ず`image_picker`化する（URLハードコードしない）
- 独自CSSファイルは作らない。Tailwind CDNで表現する
- atoms同士は相互参照しない（molecules→atoms、organisms→molecules/atomsの一方向のみ）
- 一時展開ディレクトリは処理完了後に削除する
- Stitchで再生成の選択肢が出たら処理を中断し、追加zipで再実行させる
- git操作では `--no-verify` を使わない。pre-commit/pre-pushフックは必ず通す
- `git add` は出力先ディレクトリに限定し、他の変更を巻き込まない
- リモート未設定なら push はスキップ（エラー停止しない）
