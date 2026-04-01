---
name: env-setup
description: 仕様書に環境構築セクション（CI, lint, format, linterly, hooks）を追加する
user-invocable: true
---

# env-setup

既存の仕様書に対して環境構築ドキュメントを作成する。技術スタックを読み取り、推奨構成を提示しながらヒアリングで確定していく。

## 前提条件

- Claude Code 環境
- spec-gen 等で作成済みの仕様書があること

## 引数

- **パス指定** (例: `/env-setup docs/`): 仕様書ディレクトリを直接指定
- **引数なし**: 自動探索で仕様書を検出

## 生成するドキュメント

`docs/environment/setup.md`（既存仕様書ディレクトリがあればそこに配置）

## フェーズ1: 現状把握

### 1-1: 仕様書の探索と読み込み

引数パスがあればそこを、なければ以下を Glob で探索:
```
**/spec/**/*.md
**/specs/**/*.md
**/docs/**/*.md
**/specifications/**/*.md
**/design/**/*.md
```

**必ず以下を Read で読み込む:**
- アーキテクチャ設計（技術スタック、レイヤー構造）
- コンポーネント設計（依存関係）

### 1-2: プロジェクト既存設定の確認

以下をチェック（存在するもののみ）:
- `package.json` / `go.mod` / `Cargo.toml` 等（言語・依存）
- `.github/workflows/` （既存CI）
- ESLint / Biome / Prettier / `.linterly.yml` 等（既存lint/format設定）
- `lefthook.yml` / `.husky/` （既存hooks）
- `Makefile` / `Taskfile.yml` / `justfile`（既存タスクランナー）

### 1-3: 技術スタック整理

仕様書とプロジェクト設定から技術スタックを整理し、ユーザーに確認:
- 言語 / フレームワーク
- パッケージマネージャ
- 既存ツールの有無

## フェーズ2: ヒアリングと構成決定

各セクションについて推奨構成を提示し、`AskUserQuestion` で確認する。
不明点はまとめて聞く（1回4問以内）。

### 2-1: CI/CD

推奨を提示して確認:
- CI プラットフォーム（GitHub Actions 推奨）
- トリガー（push, PR）
- ジョブ構成（lint → test → build）
- デプロイ設定（必要なら）

### 2-2: Lint

技術スタックに応じた推奨を提示:

| 言語 | 推奨リンター |
|------|------------|
| TypeScript/JavaScript | Biome or ESLint |
| Go | golangci-lint |
| Rust | clippy |
| Python | Ruff |

### 2-3: Format

| 言語 | 推奨フォーマッタ |
|------|--------------|
| TypeScript/JavaScript | Biome or Prettier |
| Go | gofmt（標準） |
| Rust | rustfmt |
| Python | Ruff format |

### 2-4: Linterly

行数管理ツール。詳細は `references/linterly.md` 参照。

**方針:**
- `rules.max_lines_per_file` / `rules.max_lines_per_directory` はデフォルト値（300/2000）を使う。yml にはコメントアウト状態で記載し「特別な理由がない限り変更しない」旨を注記する
- `.linterlyignore` に手書きソースコードの除外パターンは基本追加しない旨をコメントで記載する
- `count_mode: all` は変更しない
- `warning_threshold: 10` を推奨（早めに警告を出す）
- `language: ja` 固定

### 2-5: Git Hooks

推奨を提示:
- ツール（Lefthook 推奨）
- pre-commit: lint + format + linterly
- pre-push: test

### 2-6: ディレクトリ構造

仕様書のアーキテクチャ設計にディレクトリ構造が記載されていれば、そこへのリンクを貼る。
なければ簡易的なディレクトリ構成をヒアリングして記載する。

## フェーズ3: ドキュメント作成

### ドキュメント構成

```markdown
# 環境構築

## 概要
<!-- 技術スタックのサマリー -->

## ディレクトリ構造
<!-- アーキテクチャ設計へのリンク or 簡易記載 -->

## 開発環境セットアップ
<!-- 必要なツール、バージョン、インストール手順 -->

## CI/CD
<!-- パイプライン構成、トリガー、ジョブ -->

## Lint
<!-- ツール、設定ファイル、ルール方針 -->

## Format
<!-- ツール、設定ファイル -->

## Linterly
<!-- 行数制限、設定 -->

## Git Hooks
<!-- ツール、フック構成 -->

## 改訂履歴
```

### 作成フロー

1. ヒアリング内容に基づきドキュメント作成
2. ユーザーにレビュー提示（`AskUserQuestion` で確認、「問題なし」選択肢を含める）
3. フィードバックがあれば修正 → 再確認（なくなるまで繰り返す）
4. Format & Lint（設定があれば実行）
5. CLAUDE.md の規約に従いコミット

## ルール

- 不明点は推測せず必ずユーザーに確認する
- **質問には必ず `AskUserQuestion` を使い選択式で提示する。** テキストだけで質問しない
- 1回あたり4問以内にまとめる
- 各選択肢の `description` に推奨理由・メリデメを記載する
- 既存設定がある場合はそれを尊重し、上書きではなく拡張・統合を優先する
- **既存の仕様書ディレクトリが見つかった場合、デフォルトパスではなく既存パスに配置する**
- TaskCreate/TaskUpdate で進捗を管理する
