# codex-plan-check

Claude Code の実装計画を Codex MCP サーバー経由でレビューするプラグイン。

## 前提条件

- [Codex CLI](https://github.com/openai/codex) がインストール済みであること（MCP サーバーとして使用）

## できること

- plan ファイル（`~/.claude/plans/`）を Codex でレビュー
- PBI（要件ファイル）がある場合は要件充足もチェック
- レビュー結果を severity 順に整理して報告

## 使い方

`/codex-plan-check` を実行、または自然言語で依頼する。

```
/codex-plan-check
```

```
planをCodexにチェックさせて
```

```
計画をチェックして
```

## codex-plan-review との違い

| | codex-plan-check | codex-plan-review |
|---|---|---|
| レビュー実行 | 1回のみ | 最大5回ループ |
| plan 自動修正 | なし | 妥当な指摘を自動反映 |
| 用途 | 指摘の確認・参考 | 指摘の自動反映 |
