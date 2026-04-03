---
name: takeshiemoto:review
description: コード差分をレビュー指針に基づいてレビューする。「レビューして」「review」等の依頼時、または /takeshiemoto:review で呼び出す。
user-invocable: true
allowed-tools: Bash, Agent
---

# review

ベースブランチとの差分を観点別のサブエージェントで並列レビューし、相互レビューで議論した上で、オーケストレーターが集約・判定して最終報告する。

## 使用方法

```sh
/takeshiemoto:review --base <ベースブランチ>
```

`--base` は必須。指定がなければ使い方を案内して即時終了する。

## 実行フロー

### Step 1: 差分収集

以下を実行し、結果を保持する。

```bash
git diff <base>...HEAD --name-only
git diff <base>...HEAD
```

直近コミットのみとの比較は禁止。必ず `<base>...HEAD` を使う。

### Step 2: 並列レビュー（一次レビュー）

以下の7つのエージェントを **単一メッセージで同時に** Agent ツールで起動する。全エージェント `model: "opus"` を指定する。

| name | subagent_type | model | color |
| --- | --- | --- | --- |
| `structure-reviewer` | `takeshiemoto:structure-reviewer` | `opus` | `#E74C3C` |
| `type-reviewer` | `takeshiemoto:type-reviewer` | `opus` | `#3498DB` |
| `react-component-reviewer` | `takeshiemoto:react-component-reviewer` | `opus` | `#2ECC71` |
| `react-effect-reviewer` | `takeshiemoto:react-effect-reviewer` | `opus` | `#F39C12` |
| `react-hooks-reviewer` | `takeshiemoto:react-hooks-reviewer` | `opus` | `#9B59B6` |
| `react-async-reviewer` | `takeshiemoto:react-async-reviewer` | `opus` | `#1ABC9C` |
| `readability-reviewer` | `takeshiemoto:readability-reviewer` | `opus` | `#E67E22` |

各 Agent 呼び出しの `prompt` には以下をそのまま含める:

```text
以下の差分をレビューしてください。

## 変更ファイル一覧
<Step 1 の --name-only 結果>

## 差分
<Step 1 の git diff 結果>
```

### Step 3: 相互レビュー（議論）

Step 2 で指摘が出たエージェントが2つ以上ある場合にのみ実行する。全エージェントが指摘ゼロ、または指摘があるエージェントが1つだけの場合は Step 4 に進む。

#### 3-1. 指摘の編纂

Step 2 の全エージェントの結果を、エージェント名付きで1つのドキュメントにまとめる。

```text
## structure-reviewer の指摘
<結果>

## type-reviewer の指摘
<結果>

## react-component-reviewer の指摘
<結果>

## react-effect-reviewer の指摘
<結果>

## react-hooks-reviewer の指摘
<結果>

## react-async-reviewer の指摘
<結果>

## readability-reviewer の指摘
<結果>
```

#### 3-2. 相互レビューの実行

Step 2 で指摘を出した全エージェントを **単一メッセージで同時に** 再度起動する。指摘ゼロだったエージェントは起動しない。`model: "opus"` を指定する。

各 Agent 呼び出しの `prompt` には以下を含める:

```text
あなたは先ほど一次レビューを行いました。以下は全レビュアーの指摘一覧です。
あなたの専門観点から、他のレビュアーの指摘に対して意見を述べてください。

## あなたの役割
<このエージェントの name>

## 全レビュアーの指摘一覧
<3-1 で編纂したドキュメント>

## 差分（参照用）
<Step 1 の git diff 結果>

## 回答ルール
- 他のレビュアーの指摘に対して、あなたの専門観点から賛成・反対・補足を述べる
- 反対する場合は、あなたの観点でなぜ問題があるか具体的な根拠を示す
- 他の指摘を受けて、自分の一次レビューの指摘を撤回・修正する場合はその旨を明示する
- 議論の余地がなければ「追加意見なし」とだけ報告する
```

### Step 4: 集約と妥当性判定

Step 2 の一次レビュー結果と Step 3 の相互レビュー結果（実施した場合）を踏まえ、以下の判定を行う。

#### 4-1. 議論の反映

相互レビューで出た意見を反映する:

- 他のレビュアーから反対意見が出て、反対の根拠がより強い場合はその指摘を除外する
- レビュアー自身が撤回した指摘は除外する
- 補足意見で強化された指摘はその根拠を統合する

#### 4-2. 重複排除

複数のエージェントが同一箇所に対して同種の指摘をしている場合、最も具体的な指摘を1つ残し、他は除外する。

#### 4-3. 過剰指摘のフィルタリング

以下に該当する指摘は除外する:

- 差分に含まれないコードへの指摘（既存コードの改善提案は対象外）
- 好みレベルの指摘（動作・保守性に実質的な影響がないもの）
- 根拠がレビュー観点の条項に紐づいていない指摘

#### 4-4. 深刻度の付与

残った指摘に深刻度を付与する:

- **must**: 修正しなければマージすべきでない（型安全性の欠損、責務の重大な混在、禁止パターンへの抵触）
- **should**: 修正を強く推奨する（設計改善、可読性の大幅な向上）
- **nit**: 軽微な改善提案

### Step 5: 最終報告

集約結果をファイルパス順に以下のフォーマットで報告する。

```text
## レビュー結果

### <ファイルパス>

**[must]** L<行番号>: <指摘内容>
観点: <どのレビュー観点の条項に基づくか>
<相互レビューで議論があった場合はその結論を付記>
<修正案があればコードブロックで提示>

**[should]** L<行番号>: <指摘内容>
観点: <どのレビュー観点の条項に基づくか>
<修正案があればコードブロックで提示>

**[nit]** L<行番号>: <指摘内容>
```

全エージェントが指摘ゼロ、または集約後に有効な指摘が残らなかった場合は「指摘なし」とだけ報告する。
