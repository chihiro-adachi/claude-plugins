---
name: codex-plan-review
description: This skill should be used when the user asks to "review plan with codex", "planをレビュー", "計画をレビュー", "Codexにレビューさせて", "codexでレビュー", or when a plan has been completed and the user asks to "review it" or "レビューして". Also triggers after plan mode completion when user mentions reviewing.
---

# Codex Plan Review

Claude Code の plan モードで生成された実装計画を、Codex MCP サーバー経由でレビューするスキル。コードベースも参照し、計画の妥当性を検証する。指摘の評価・修正・再レビューのループを自動で行う。

## フロー

### ステップ 1: plan ファイルの特定

- ユーザーが特定ファイルを指定していればそれを使用する
- 会話内で直前に作成・参照された plan ファイル名があればそれを優先する
- 上記いずれでも特定できなければ `ls -t ~/.claude/plans/*.md` で候補を取得する
  - 候補が1件 → そのファイルを使用
  - 候補が複数 → ファイル名を一覧表示し、ユーザーに選択を求める

### ステップ 2: Codex MCP でレビュー実行

以下のプロンプトテンプレートから prompt 文字列を構築し、`mcp__codex__codex({ prompt: <構築した文字列> })` を呼び出す。

**注意事項:**
- `{plan_path}` は実際の絶対パスに置換する。内容のインライン展開は行わない
- `approval-policy` は指定しない（Codex がファイルを読み取れるようにするため）

#### プロンプトテンプレート

```
以下のファイルと、計画内で言及されているファイルやモジュールを読み込み、
実装計画をコードベースと照らし合わせて厳密にレビューしてください。

- 実装計画: {plan_path}

目的は、実装前に計画の品質を上げ、手戻り・抜け漏れ・設計事故を減らすことです。

レビュー観点:
1. 実装順序と依存関係に無理がないか
2. 技術的な問題点、抜け漏れ、曖昧さがないか
3. 既存機能への回帰リスクが考慮されているか
4. テスト計画が十分か
5. データ移行、ロールバック、運用監視の考慮が必要なのに漏れていないか
6. セキュリティ、性能、権限、障害時対応の観点で不足がないか
7. スコープが過大ではないか。より安全な分割や段階導入の余地がないか

レビュー方針:
- 抽象論ではなく、計画のどの部分をどう直すべきか具体的に指摘してください
- 指摘は「実装時に事故や手戻りにつながるもの」を優先してください
- 軽微な好みよりも、正しさ・安全性・保守性・実行可能性を優先してください

verdict 判定基準:
- LGTM: medium 以上の指摘がない（low のみ、または指摘なし）
- NEEDS_FIX: medium 以上の指摘がある
- BLOCKED: critical の指摘がある（設計の根本的な見直しや前提の確認が必要）

回答は日本語で、必ず以下の JSON 形式のみで返してください（それ以外のテキストは含めないでください）:
{
  "overview": {
    "verdict": "LGTM" または "NEEDS_FIX" または "BLOCKED",
    "summary": "2〜4文の要約"
  },
  "findings": [
    {
      "id": "F1, F2, ... の連番",
      "title": "短い見出し",
      "severity": "critical/high/medium/low",
      "category": "sequencing/technical/regression/testing/migration/security/performance/operations/scope",
      "problem": "何が問題か",
      "impact": "そのまま進めると何が起こるか",
      "suggested_change": "計画にどう追記・修正すべきか",
      "affected_step": "該当する計画のステップ名や箇所"
    }
  ],
  "requirement_questions": ["計画の前提が曖昧で実装前に確認すべき事項"],
  "suggested_diff": "元の計画をどう直すべきか、追記案の形で簡潔に"
}

findings が無い場合は空配列、requirement_questions が無い場合も空配列にしてください。
```

#### パースと threadId の記録

1. `mcp__codex__codex` の応答から JSON 部分を抽出してパースする
   - 応答全体が JSON でない場合は、`{` で始まる行から `}` で終わる行までを抽出する
   - パース失敗の場合: 1回だけ再試行する（同じプロンプトで MCP ツールを再呼び出し）
   - 再試行でもパース失敗の場合: Codex の応答をそのままユーザーに表示してフローを中断する
2. Codex がエラーを返した場合は、エラー内容をユーザーに報告してフローを中断する
3. **応答に含まれる `threadId` を記録しておく。** ステップ 4 の再レビューで同一セッションを継続するために必要

### ステップ 3: 指摘の評価と修正

1. `overview.verdict` が `"LGTM"` → ステップ 5 へ
2. `overview.verdict` が `"BLOCKED"` → ブロッカーをユーザーに報告し、ユーザーの判断を仰ぐ
3. `requirement_questions` がある場合 → ユーザーに確認事項として提示する
4. `findings` 配列の各指摘を評価する:
   - 技術的に妥当か
   - plan のコンテキストに合っているか
   - severity と impact の妥当性
5. 妥当と判断した指摘のみ、Edit ツールで plan ファイルを修正する。`suggested_diff` も参考にする
6. 却下した指摘は `id` と却下理由を記録する

### ステップ 4: 再レビューループ

ループ回数をカウントする（最大5回）。ステップ 2 で記録した `threadId` を使い、`mcp__codex__codex-reply` で同一セッションを継続する。

#### 再レビュープロンプトテンプレート

```
plan ファイルを修正しました。

{修正内容の説明: どの指摘を採用し、どう修正したか}

{却下した指摘がある場合のみ以下を追加:
以下の指摘は検討済みです。これらについては再度指摘しないでください：
- F1: <却下理由>
- F3: <却下理由>
}

修正済みの plan ファイルを再度読み込み、同じ観点・同じ出力形式でレビューしてください。

- 実装計画: {plan_path}
```

上記テンプレートから prompt を構築し、`mcp__codex__codex-reply({ threadId: <記録した threadId>, prompt: <構築した文字列> })` を呼び出す。

#### ループ制御

- パースはステップ 2 と同様
- `overview.verdict` が `"LGTM"` → ステップ 5 へ
- `"NEEDS_FIX"` → ステップ 3 に戻る
- `"BLOCKED"` → ステップ 3 に戻る（ブロッカー対応）
- 5回到達で `"LGTM"` にならなければ → ステップ 5 へ
- **重要:** 全指摘を採用した場合でも、`overview.verdict` が `"LGTM"` になるまでループを継続すること。独自判断で打ち切らないこと

### ステップ 5: 完了報告

最終結果をユーザーに報告する:
- 総評（verdict と summary）
- 採用した指摘と修正内容
- 却下した指摘と理由
- 確認事項（あれば）
- ループ回数
- 最終ステータス（LGTM / 5回到達で終了）
