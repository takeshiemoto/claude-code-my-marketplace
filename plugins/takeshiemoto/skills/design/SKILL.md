---
name: takeshiemoto:design
description: タスクの設計をルール精読・静的解析理解・アーキテクチャ設計・設計評価・コミット計画のフローで行うハーネス。「設計して」「design」等の依頼時、または /takeshiemoto:design で呼び出す。
user-invocable: true
allowed-tools: Bash, Agent
---

# design

タスク要件を受け取り、ルール精読→静的解析理解→設計→設計評価→コミット計画→最終設計の一連のフローで設計を行うハーネス。

## 使用方法

```sh
/takeshiemoto:design <タスク説明>
```

タスク説明は必須。指定がなければ使い方を案内して即時終了する。

## 基本姿勢

- シニアエンジニアとして振る舞う
- 誤記・曖昧さ・不整合に対して厳しく指摘する
- 動くコードではなく、正しいコードを設計する
- YAGNI, KISS, DRY を徹底する
- 推測で補完しない

## 実行フロー

### Step 1: コンテキスト収集（並列）

以下の2つのエージェントを **単一メッセージで同時に** Agent ツールで起動する。

| name | subagent_type | model |
| --- | --- | --- |
| `rule-reader` | `takeshiemoto:rule-reader` | `opus` |
| `script-reader` | `takeshiemoto:script-reader` | `opus` |

rule-reader への prompt:

```text
このリポジトリのルール・規約を精読し、設計時に順守すべき制約を報告してください。
```

script-reader への prompt:

```text
このリポジトリの npm script と静的解析ツールを精読し、実装完了後に実行すべきコマンドを報告してください。
```

### Step 2: 初期設計

design-architect エージェントを起動する。

| name | subagent_type | model |
| --- | --- | --- |
| `design-architect` | `takeshiemoto:design-architect` | `opus` |

prompt:

```text
以下のタスクの設計を行ってください。

## タスク
<ユーザーのタスク説明>

## リポジトリのルール・規約
<Step 1 rule-reader の結果>

## 静的解析・検証コマンド
<Step 1 script-reader の結果>
```

### Step 3: 設計評価（並列）

review スキルで使用している7つのレビューエージェントを **単一メッセージで同時に** Agent ツールで起動する。全エージェント `model: "opus"` を指定する。

| name | subagent_type | model |
| --- | --- | --- |
| `purity-reviewer` | `takeshiemoto:purity-reviewer` | `opus` |
| `state-modeling-reviewer` | `takeshiemoto:state-modeling-reviewer` | `opus` |
| `effect-boundary-reviewer` | `takeshiemoto:effect-boundary-reviewer` | `opus` |
| `type-constraint-reviewer` | `takeshiemoto:type-constraint-reviewer` | `opus` |
| `ai-readability-reviewer` | `takeshiemoto:ai-readability-reviewer` | `opus` |
| `structure-reviewer` | `takeshiemoto:structure-reviewer` | `opus` |
| `hooks-design-reviewer` | `takeshiemoto:hooks-design-reviewer` | `opus` |

各エージェントへの prompt:

```text
以下の設計案を、あなたの専門観点から評価してください。
設計段階のため差分ではなく設計ドキュメントです。
提案されているコード構造・型設計・アプローチに対して、問題点や改善すべき点を指摘してください。

## タスク
<ユーザーのタスク説明>

## 設計案
<Step 2 design-architect の結果>
```

### Step 4: コミット計画

commit-planner エージェントを起動する。

| name | subagent_type | model |
| --- | --- | --- |
| `commit-planner` | `takeshiemoto:commit-planner` | `opus` |

prompt:

```text
以下の設計案と評価結果を踏まえ、コミット計画を作成してください。

## タスク
<ユーザーのタスク説明>

## 設計案
<Step 2 design-architect の結果>

## 設計評価
<Step 3 の全エージェントの結果をエージェント名付きで列挙>

## 静的解析・検証コマンド
<Step 1 script-reader の結果>
```

### Step 5: 最終設計

Step 1〜4 の全結果を統合し、以下のフォーマットで最終設計を出力する。

Step 3 の評価で must レベルの指摘がある場合、その指摘を設計に反映した上で最終設計に含める。反映できない指摘がある場合はその理由を明記する。

```text
## 最終設計

### タスク概要
<タスクの要約>

### 順守すべきルール
<Step 1 rule-reader から抽出した、この設計に関係する制約>

### アーキテクチャ
<Step 2 の設計 + Step 3 の評価を反映した最終的なアーキテクチャ>

### 主要な型・インターフェース設計
<最終的な型定義・インターフェース>

### ファイル変更計画
| ファイル | 操作 | 責務 | 概要 |
|---|---|---|---|
| <パス> | 新規/修正/削除 | <責務> | <変更内容> |

### コミット計画
<Step 4 の結果>

### 設計評価の反映
| 評価エージェント | 指摘 | 深刻度 | 対応 |
|---|---|---|---|
| <エージェント名> | <指摘内容> | must/should/nit | <どう反映したか、または未反映の理由> |

### 実装後の検証コマンド
<Step 1 script-reader から抽出したコマンドを実行順に列挙>
```
