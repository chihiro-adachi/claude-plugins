---
name: codex-plan-review
description: This skill should be used when the user asks to "review plan with codex", "planをレビュー", "計画をレビュー", "Codexにレビューさせて", "codexでレビュー", or when a plan has been completed and the user asks to "review it" or "レビューして". Also triggers after plan mode completion when user mentions reviewing.
---

# Codex Plan Review

Claude Code の plan モードで生成された実装計画を、OpenAI Codex CLI でレビューするスキル。

## フロー

### ステップ 0: 事前チェック

1. Bash ツールで `which codex` を実行し、Codex CLI がインストールされているか確認する
   - 見つからない場合: 「Codex CLI がインストールされていません。`npm install -g @openai/codex` でインストールしてください。」と報告して終了
2. Bash ツールで `ls ~/.claude/plans/*.md 2>/dev/null` を実行し、plan ファイルの存在を確認する
   - ファイルが0件の場合: 「~/.claude/plans/ に plan ファイルが見つかりません。」と報告して終了

### ステップ 1: plan ファイルの特定

- ユーザーが特定ファイルを指定していればそれを使用する
- 会話内で直前に作成・参照された plan ファイル名があればそれを優先する
- 上記いずれでも特定できなければ `ls -t ~/.claude/plans/*.md` で候補を取得する
  - 候補が1件 → そのファイルを使用
  - 候補が複数 → ファイル名を一覧表示し、ユーザーに選択を求める

### ステップ 2: PBI（要件）の取得

以下の優先順位で PBI を取得する:

1. 会話内で参照された要件ファイルやプロンプト（ユーザーが plan 作成時に指定した PBI）があればそれを使用する
2. 上記がなければ、Read ツールで plan ファイルを読み込み、以下のパターンで PBI への参照を探す:
   - ヘッダー部の `**Spec:**` や `**PBI:**` に続く `@docs/...` や `docs/...` 形式のパス参照
   - 見出しに requirements / spec / PBI / 要件 / 仕様 を含むセクション内の `docs/` 配下の Markdown ファイルパス
   - 抽出対象は `docs/` 配下の Markdown ファイル（`.md`）のみ。`src/` 等のコードファイルは無視する
   - 該当するファイルがあれば Read ツールで読み込み、PBI として使用する
3. いずれでも取得できなければ PBI なしで続行する（技術レビューのみ）

### ステップ 3: Codex CLI でレビュー実行

1. Bash ツールで `mktemp` を実行し、JSON Schema 用のテンポラリファイルパスを取得する
2. Bash ツールの heredoc で JSON Schema をそのファイルに書き出す（Write ツールは未読ファイルへの書き込みを拒否するため使用不可）:

```bash
cat > <スキーマファイル> << 'EOF'
<以下の JSON Schema の内容>
EOF
```

   以下の JSON Schema を書き出す:

```json
{
  "type": "object",
  "required": ["status", "findings"],
  "additionalProperties": false,
  "properties": {
    "status": {
      "type": "string",
      "enum": ["LGTM", "NEEDS_WORK"]
    },
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "severity", "rationale", "suggested_change"],
        "additionalProperties": false,
        "properties": {
          "id": { "type": "string" },
          "severity": { "type": "string", "enum": ["high", "medium", "low"] },
          "rationale": { "type": "string" },
          "suggested_change": { "type": "string" }
        }
      }
    }
  }
}
```

3. Bash ツールで `mktemp` を実行し、プロンプト用のテンポラリファイルパスを取得する
4. Bash ツールの heredoc でプロンプトをそのファイルに書き出す（Write ツールは未読ファイルへの書き込みを拒否するため使用不可）:

```bash
cat > <プロンプトファイル> << 'EOF'
<以下のプロンプトの内容>
EOF
```

   以下のプロンプトを書き出す:

**PBI ありの場合:**
```
以下の実装計画を、要件定義と照らし合わせてレビューしてください。

評価観点：
1. 要件・完了条件を満たした計画になっているか
2. 技術的な問題点・抜け漏れ
3. 改善点

## 要件定義
<PBI の内容をここに展開>

## 実装計画
<plan の内容をここに展開>
```

**PBI なしの場合:**
```
以下の実装計画をレビューしてください。

評価観点：
1. 技術的な問題点・抜け漏れ
2. 改善点

## 実装計画
<plan の内容をここに展開>
```

5. Bash ツールで以下を実行する（`timeout: 300000`（5分）を指定すること。Codex CLI のレビューは120秒を超えることがあり、デフォルトタイムアウトではバックグラウンドに移行してしまう）:
```bash
cat <プロンプトファイル> | codex exec --output-schema <スキーマファイル> -
```

6. 出力を JSON としてパースする
   - **注意:** Codex CLI の出力にはヘッダ（バージョン情報等）とプロンプトエコーが含まれる。出力の末尾にある JSON オブジェクト（`{` で始まる行以降）を抽出してパースすること
   - パース失敗の場合: 1回だけ再試行する（同じコマンドを再実行）
   - 再試行でもパース失敗の場合: テンポラリファイルを削除（ステップ 6 のクリーンアップ）してから、出力をそのままユーザーに表示してフローを中断する

7. Codex CLI がエラー（非ゼロ終了コード）を返した場合は、テンポラリファイルを削除（ステップ 6 のクリーンアップ）してから、エラー内容をユーザーに報告してフローを中断する

### ステップ 4: 指摘の評価と修正

1. JSON の `status` が `"LGTM"` であれば → ステップ 6 へ
2. `findings` 配列の各指摘を評価する:
   - 技術的に妥当か
   - plan のコンテキストに合っているか
   - PBI の要件に照らして正当か（PBI がある場合）
3. 妥当と判断した指摘のみ、Edit ツールで plan ファイルを修正する
4. 却下した指摘は `id` と却下理由を記録する

### ステップ 5: 再レビューループ

- ループ回数をカウントする（最大5回）
- **プロンプトファイルを新しい内容で上書きする**。以下を含める:
  1. 元のレビュー指示（PBI あり/なしに応じたプロンプト）
  2. 却下した指摘の除外指示（却下した指摘がない場合は、このセクション自体を省略する）。形式:
  ```
  以下の指摘は検討済みです。これらについては再度指摘しないでください：
  - F1: <却下理由>
  - F3: <却下理由>
  ```
  3. 修正済みの plan の最新内容（Read ツールで再読み込み）
  4. PBI の内容（ある場合）
- 更新したプロンプトファイルで codex exec を再実行する（ステップ 3 の手順 5〜7 と同様）
- JSON の `status` が `"LGTM"` であれば → ステップ 6 へ
- `"NEEDS_WORK"` であれば → ステップ 4 に戻る
- 5回到達で `"LGTM"` にならなければ → ステップ 6 へ
- **重要:** 全指摘を採用した場合でも、`status` が `"LGTM"` になるまでループを継続すること。独自判断で打ち切らないこと

### ステップ 6: クリーンアップと完了報告

1. Bash ツールでテンポラリファイル（schema ファイル、プロンプトファイル）を削除する:
```bash
rm -f <スキーマファイル> <プロンプトファイル>
```

2. 最終結果をユーザーに報告する:
   - 採用した指摘と修正内容
   - 却下した指摘と理由
   - ループ回数
   - 最終ステータス（LGTM / 5回到達で終了）

## 注意事項

- 文書レビューとして割り切る: Codex には plan と PBI のみ渡す。cwd 内の関連コードは渡さない
- 日本語で報告する
- 技術用語はそのまま使用可
