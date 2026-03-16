# codex-plan-review

Claude Code の実装計画を Codex MCP サーバー経由でレビューするプラグイン。

## 前提条件

- [Codex CLI](https://github.com/openai/codex) がインストール済みであること（MCP サーバーとして利用）

## できること

- plan ファイル（`~/.claude/plans/`）を Codex MCP サーバーでレビュー
- コードベースを参照し、計画が既存コードと整合しているか検証
- PBI（要件ファイル）がある場合は要件充足もチェック
- 指摘の妥当性を Claude Code が評価し、妥当なもののみ反映
- 最大5回の修正 → 再レビューループ

## 使い方

`/codex-plan-review` を実行、または自然言語で依頼する。

```
/codex-plan-review
```

```
planをCodexにレビューさせて
```

```
計画をレビューして
```
