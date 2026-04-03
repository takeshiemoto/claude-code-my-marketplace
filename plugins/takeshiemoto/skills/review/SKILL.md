---
name: takeshiemoto:review
description: コード差分をレビュー指針に基づいてレビューする。「レビューして」「review」等の依頼時、または /takeshiemoto:review で呼び出す。
user-invocable: true
allowed-tools: Bash, Agent
---

# review

ベースブランチとの差分を観点別のサブエージェントで並列レビューする。

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

### Step 2: 並列レビュー

以下の3つのエージェントを **単一メッセージで同時に** Agent ツールで起動する。

| name | subagent_type |
| --- | --- |
| `structure-reviewer` | `structure-reviewer` |
| `type-reviewer` | `type-reviewer` |
| `react-reviewer` | `react-reviewer` |

各 Agent 呼び出しの `prompt` には以下をそのまま含める:

```text
以下の差分をレビューしてください。

## 変更ファイル一覧
<Step 1 の --name-only 結果>

## 差分
<Step 1 の git diff 結果>
```

### Step 3: 結果統合

全エージェントの結果を以下のルールで統合して報告する:

- 重複する指摘はマージする
- ファイルパス順に並べ替える
- 全エージェントが指摘ゼロなら「指摘なし」とだけ報告する
