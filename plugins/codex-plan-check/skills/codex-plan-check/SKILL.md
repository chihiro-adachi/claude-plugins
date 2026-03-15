---
name: codex-plan-check
description: This skill should be used when the user asks to "check plan with codex", "planをチェック", "計画をチェック", "Codexにチェックさせて", "codexでチェック", "planを単発レビュー", or wants a one-shot Codex review of a plan without iterative re-review loops.
---

# Codex Plan Check

Claude Code の plan モードで生成された実装計画を、Codex MCP サーバー経由でレビューするスキル。

## フロー

### ステップ 1: plan ファイルの特定

- ユーザーが特定ファイルを指定していればそれを使用する
- 会話内で直前に作成・参照された plan ファイル名があればそれを優先する

### ステップ 2: PBI（要件）の取得

1. 会話内で参照された要件ファイルやプロンプト（ユーザーが plan 作成時に指定した PBI）があればそれを使用する
2. 取得できなければ PBI なしで続行する（技術レビューのみ）

### ステップ 3: plan ファイルと PBI の読み込み

Read ツールで plan ファイルの内容を読み込む。PBI がある場合は PBI ファイルも読み込む。

### ステップ 4: Codex MCP でレビュー実行

`mcp__codex__codex` ツールを呼び出してレビューを実行する。

**PBI ありの場合:**

```
mcp__codex__codex({
  prompt: "以下の実装計画を、要件定義と照らし合わせてレビューしてください。\n\n評価観点：\n1. 要件・完了条件を満たした計画になっているか\n2. 技術的な問題点・抜け漏れ\n3. 改善点\n\n回答は必ず以下の JSON 形式のみで返してください（それ以外のテキストは含めないでください）:\n{\"status\": \"LGTM\" または \"NEEDS_WORK\", \"findings\": [{\"id\": \"文字列\", \"severity\": \"high/medium/low\", \"rationale\": \"指摘理由\", \"suggested_change\": \"改善案\"}]}\n\nfindings が無い場合は空配列にしてください。\n\n## 要件定義\n\n<PBI の内容をここに展開>\n\n## 実装計画\n\n<plan の内容をここに展開>",
  approval-policy: "never"
})
```

**PBI なしの場合:**

```
mcp__codex__codex({
  prompt: "以下の実装計画をレビューしてください。\n\n評価観点：\n1. 技術的な問題点・抜け漏れ\n2. 改善点\n\n回答は必ず以下の JSON 形式のみで返してください（それ以外のテキストは含めないでください）:\n{\"status\": \"LGTM\" または \"NEEDS_WORK\", \"findings\": [{\"id\": \"文字列\", \"severity\": \"high/medium/low\", \"rationale\": \"指摘理由\", \"suggested_change\": \"改善案\"}]}\n\nfindings が無い場合は空配列にしてください。\n\n## 実装計画\n\n<plan の内容をここに展開>",
  approval-policy: "never"
})
```

**注意事項:**
- plan と PBI の内容はファイルパスではなく、ステップ 3 で読み込んだテキスト本文をプロンプトにインライン展開すること
- `approval-policy` は `"never"` を指定する（レビューのみでファイル操作は不要なため）

### ステップ 5: レビュー結果のパースと報告

1. `mcp__codex__codex` の応答から JSON 部分を抽出してパースする
   - 応答全体が JSON でない場合は、`{` で始まる行から `}` で終わる行までを抽出する
   - パース失敗の場合: Codex の応答をそのままユーザーに表示してフローを中断する

2. レビュー結果をユーザーに報告する:
   - ステータス（LGTM / NEEDS_WORK）を表示する
   - `findings` がある場合、各指摘を severity 順（high → medium → low）に整理して表示する:
     - 指摘ID
     - 重要度（severity）
     - 指摘理由（rationale）
     - 改善案（suggested_change）
