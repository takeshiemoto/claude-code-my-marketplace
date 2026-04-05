---
name: takeshiemoto:weekly-digest
description: 直近1週間のテック動向を7エージェントで並列調査し日本語で要約する。「今週のニュース」「テックダイジェスト」「weekly-digest」等の依頼時、または /takeshiemoto:weekly-digest で呼び出す。
user-invocable: true
allowed-tools: Agent
---

# weekly-digest

直近1週間のテック動向を7つの専門エージェントで並列に Web 検索し、日本語で要約するスキル。

## 使用方法

```sh
/takeshiemoto:weekly-digest
```

引数不要。実行時点から直近1週間の情報を収集する。

## 実行フロー

### Step 1: 並列調査

以下の7つのエージェントを **単一メッセージで同時に** Agent ツールで起動する。全エージェント `model: "opus"` を指定する。

| name | subagent_type | model |
| --- | --- | --- |
| `digest-react` | `takeshiemoto:digest-react` | `opus` |
| `digest-claude` | `takeshiemoto:digest-claude` | `opus` |
| `digest-codex` | `takeshiemoto:digest-codex` | `opus` |
| `digest-typescript` | `takeshiemoto:digest-typescript` | `opus` |
| `digest-frontend` | `takeshiemoto:digest-frontend` | `opus` |
| `digest-go-rust` | `takeshiemoto:digest-go-rust` | `opus` |
| `digest-web-buzz` | `takeshiemoto:digest-web-buzz` | `opus` |

各エージェントへの prompt:

```text
今日は <今日の日付> です。直近1週間（<1週間前の日付> 〜 <今日の日付>）の動向を調査してください。
```

### Step 2: 統合・要約

全エージェントの結果を統合し、以下のフォーマットで日本語の要約を出力する。

```text
# 週間テックダイジェスト（<1週間前の日付> 〜 <今日の日付>）

## 今週のハイライト
<全トピックを横断して、最も重要な3〜5件をピックアップ>

---

## React
<digest-react の結果を要約>

## Claude / Anthropic
<digest-claude の結果を要約>

## Codex / AI コーディングツール
<digest-codex の結果を要約>

## TypeScript
<digest-typescript の結果を要約>

## フロントエンド
<digest-frontend の結果を要約>

## Go & Rust
<digest-go-rust の結果を要約>

## バズ記事・話題
<digest-web-buzz の結果を要約>
```

### 統合ルール

- 各エージェントの報告をそのまま貼り付けない。要点を抽出して再構成する
- 重複する情報は1箇所にまとめる
- 情報がなかったトピックは「特筆すべき動きなし」と記載する
- 出典 URL は残す
- 「今週のハイライト」は、インパクト・影響範囲の大きさで選定する
