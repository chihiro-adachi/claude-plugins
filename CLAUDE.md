# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

Claude Code 用プラグインのマーケットプレイスリポジトリ。マーケットプレイス名は `chihiro-adachi-claude-plugins`。

## プラグイン追加手順

1. `plugins/<plugin-name>/` を作成
2. `plugins/<plugin-name>/.claude-plugin/plugin.json` を作成（name, version, description）
3. `.claude-plugin/marketplace.json` の `plugins` 配列にエントリを追加
4. `/reload-plugins` で動作確認

## コーディング規約

- ドキュメントとユーザー向けテキストは日本語で記述する
