---
name: impl-wt-max
description: git worktree で隔離した環境で実装サイクルを回し、PR を作成する。
user-invocable: true
---

# impl-wt-max

`impl-max` の worktree 隔離版。メインの作業ツリーを汚さずに実装を行う。

## 前提条件

- Claude Code 環境
- `git`, `gh` CLI

## 引数

- **Issue 番号** (例: `/impl-wt-max #123`): GitHub Issue から要件を取得
- **Issue URL** (例: `/impl-wt-max https://github.com/owner/repo/issues/123`): 同上
- **テキスト** (例: `/impl-wt-max ユーザー認証機能を追加`): テキストを要件として扱う
- **引数なし**: ユーザーに要件をヒアリング

## フェーズ1: 要件分析とスコープ分割

1. 引数から要件を取得する
   - Issue: `gh issue view` で本文・コメントを読み取る
   - テキスト: そのまま要件として扱う
   - 引数なし: ユーザーにヒアリング
2. 現在のブランチをベースブランチとして記録する（PR のマージ先）
   - `git branch --show-current` を実行し、結果をユーザーに「ベースブランチ: <ブランチ名>」と明示的に表示する
   - このブランチ名をフェーズ3のPR作成時まで保持する
3. 作業ブランチ名を決定する（命名規則は `references/branch-naming.md` を参照）
4. **git worktree を作成する**（手順は `references/worktree-setup.md` を参照）
5. 要件を「独立して実装・テストできる単位」に分割
6. 依存関係を整理し実装順を決定
7. TaskCreate でタスクを作成

- コードベースの探索・理解を行い要件を正確に把握する
- 不確定な仕様はユーザーに確認する
- 破壊的変更がある場合は下位互換性についてユーザーに確認する

## フェーズ2: 実装サイクル（各スコープで繰り返し）

**重要: フェーズ2のすべてのサブエージェント呼び出しで worktree パスを作業ディレクトリとして指定すること。**

#### 2-1: Plan
- `Plan` エージェントで実装計画を立てる
- 変更箇所、影響範囲、テスト要件を明確にする
- プロンプトに worktree パスを含める

#### 2-2: Develop
- `develop` エージェントで実装（テスト含む）
- 最小限の変更で要件を満たす
- プロンプトに worktree パスを含める

#### 2-3: Review
- `review` エージェントでコードレビュー
- 要件適合、コード品質、テスト十分性を評価
- プロンプトに worktree パスを含める

#### 2-4: 改善サイクル
- レビュー指摘あり → `develop` で修正 → `review` で再レビュー → 指摘なしまで繰り返す

#### 2-5: Format & Lint
- プロジェクト設定に従い変更ファイルに format/lint を実行
- **worktree ディレクトリ内で** format/lint コマンドを実行する
- 設定が見つからない場合はスキップ

#### 2-6: Commit（必須）
- **各スコープ完了時に必ずコミットする。スキップ不可。**
- **worktree ディレクトリ内で** `git add` / `git commit` を実行する
- コミットメッセージは CLAUDE.md の規約に従う

## フェーズ3: 完了確認とPR作成

1. 全スコープの実装完了を確認
2. **worktree ディレクトリ内で** 全体テストを実行
3. **worktree ディレクトリ内で** `git push -u origin <作業ブランチ>` を実行
4. `gh pr create --base <ベースブランチ>` でPRを作成
   - **ベースブランチはフェーズ1で記録した開始時のブランチを指定する。`main` や `master` にフォールバックしないこと。**
   - 不明な場合は `git log --oneline --graph HEAD...main` 等で分岐元を確認する
   - Issue 指定時: タイトルに Issue 番号を含め、本文先頭に `Closes #<番号>` を記載
   - PR本文: 変更サマリー + 手動チェックリスト（`templates/pr-checklist.md` を参照）
5. 実装サマリーと **worktree パス** をユーザーに報告する

報告例:
```
## 完了
- PR: <URL>
- Worktree: <パス>（確認後 `git worktree remove <パス>` で削除可能）
```

## ルール

- 各スコープは独立して実装・テスト可能な単位にする
- **各スコープ完了時に必ずコミットする。** コミットせずに次へ進まない
- レビュー指摘はすべての重大度（🔴🟠🟡🟢）で修正する
- TaskCreate/TaskUpdate で進捗を管理する
- **すべての git / ファイル操作は worktree ディレクトリ内で行う。メインの作業ツリーを変更しない。**
