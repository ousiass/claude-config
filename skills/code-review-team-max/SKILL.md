---
name: code-review-team-max
description: Agent Teamsで専門レビュアーが並列でコードレビューを行う。
user-invocable: true
---

# code-review-team-max

専門分野ごとのレビュアーが並列でコードレビューを行い、多角的なレビューで網羅性と品質を向上させる。

> **注意**: Agent Teams を使用するため `code-review-max` より多くのトークンを消費する。小規模プロジェクトには `code-review-max` を推奨。

## 前提条件

- Claude Code 環境
- `gh` CLI（GitHub Issue 出力時）

## 引数

`code-review-max` と同じ:
- 引数なし: プロジェクト全体をチェック
- パス指定: 特定のディレクトリやファイルに絞ってチェック

## フェーズ1: プロジェクト構造の把握

`code-review-max` のフェーズ1と同じ手順でプロジェクト構造を把握し、チェック観点を特定する。

## フェーズ2: チーム編成と並列レビュー

1. エージェントチームを作成
2. レビュアーをスポーン:

#### 必須レビュアー
- **architecture-reviewer** × 1: ファイル行数、構成、重複、依存関係、複雑度、デッドコード、マジックナンバー、テスタビリティ
- **quality-reviewer** × 1: Makefile、Lint/Format、Git Hooks、環境変数、エラー処理、ログ、テスト、CI/CD、.gitignore

#### 条件付きレビュアー
- **frontend-reviewer**（FE あり）: `code-review-max` の `references/check-frontend.md` 全項目
- **backend-reviewer**（BE あり）: `code-review-max` の `references/check-backend.md` 全項目
- **security-reviewer**（大規模 PJ）: セキュリティ脆弱性、認証・認可、入力検証、OWASP Top 10
- **performance-reviewer**（大規模 PJ）: N+1 クエリ、メモリリーク、パフォーマンスボトルネック

3. 各プロンプトに含める: チェック項目詳細、重大度基準、レポート形式
4. 共有タスクリストにレビュアーごとのタスクを作成

## フェーズ3: レポート統合

1. 全レビュアーの結果を収集
2. 重複する指摘を統合
3. `code-review-max` のレポート形式（`templates/report.md`）で統合レポートを生成
4. `AskUserQuestion` で出力先確認: GitHub Issue / ローカル MD
5. 要約をユーザーに報告

## フェーズ4: チーム解散

全チームメイトにシャットダウン要求 → チームリソースをクリーンアップ

## チームメイトの役割

| 役割 | 人数 | 条件 | 責務 |
|------|------|------|------|
| architecture-reviewer | 1 | 必須 | アーキテクチャ、構造、依存関係 |
| quality-reviewer | 1 | 必須 | テスト、CI/CD、lint/format、エラー処理 |
| frontend-reviewer | 0〜1 | FE あり | フロントエンド固有 |
| backend-reviewer | 0〜1 | BE あり | バックエンド固有 |
| security-reviewer | 0〜1 | 大規模 PJ | セキュリティ脆弱性 |
| performance-reviewer | 0〜1 | 大規模 PJ | パフォーマンス問題 |

## ルール

- レビュアーは読み取り専用。コード変更不可
- 各レビュアーは担当ドメインのみレビュー
- 指摘にはファイルパスと行番号を含める
- リードは結果を統合し重複を除去する
- 重大度基準は `code-review-max` と同一
- チームメイトがエラーで停止した場合、リードが再スポーンまたは対処する
- 共有タスクリストで進捗を管理する
