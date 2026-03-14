---
name: claude-changelog
description: This skill should be used when the user asks to "check Claude Code changelog", "what's new in Claude Code", "Claude Code version changes", "Claude Code release notes", "Claude Code update history", or mentions a specific version like "Claude Code 2.1.17 changes".
---

# Claude Code Changelog Skill

Claude Codeの変更履歴を調べるためのスキル。特定バージョンの変更点や最新のリリースノートを確認できる。

## 使用方法

### 特定バージョンの変更点を確認

ユーザーが特定のバージョン（例: 2.1.17）について質問した場合:

1. WebFetchツールでGitHubのCHANGELOGを取得:
   ```
   URL: https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
   プロンプト: Find and extract the changelog entry for version X.Y.Z
   ```

2. 結果を日本語で簡潔にまとめて報告

### 最新の変更点を確認

ユーザーが「最新の変更」や「最近のアップデート」を知りたい場合:

1. WebFetchツールでCHANGELOGを取得:
   ```
   URL: https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
   プロンプト: Extract the most recent 3 changelog entries with their version numbers and changes
   ```

2. 最新3バージョンの変更点をまとめて報告

### 特定機能の変更履歴を検索

ユーザーが特定機能（例: hooks, MCP, agents）の変更履歴を知りたい場合:

1. WebFetchツールでCHANGELOGを取得:
   ```
   URL: https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
   プロンプト: Find all changelog entries mentioning [機能名]
   ```

2. 関連する変更をバージョン順にまとめて報告

## 情報ソース

- **公式CHANGELOG**: https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
- **リリースページ**: https://github.com/anthropics/claude-code/releases
- **npmパッケージ**: https://www.npmjs.com/package/@anthropic-ai/claude-code

## 出力フォーマット

変更点を報告する際は以下の形式を使用:

```
## Claude Code X.Y.Z の変更点

- **変更カテゴリ**: 変更内容の説明

Sources:
- [CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
```

## 注意事項

- 常にソースURLを含める
- 日本語で回答する
- 技術用語はそのまま使用可
