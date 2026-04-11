---
name: takeshiemoto:backlog
description: 現在の作業コンテキスト(タスク/目的/現状/次)を claude-code-histories に時系列MDで吐き出す。「backlog」等の依頼時、または /takeshiemoto:backlog で呼び出す。
user-invocable: true
allowed-tools: Bash, Write
---

# backlog

現在の会話から作業コンテキストを抽出し、時系列で並ぶMDファイルとして外部ディレクトリに書き出す。

## 使用方法

```sh
/takeshiemoto:backlog
```

引数不要。実行時点のコンテキストを自動抽出する。

## 出力先

`/Users/takeshiemoto/ghq/github.com/takeshiemoto/claude-code-histories/`

## ファイル名規則

`YYYYMMDD-HHMMSS-<slug>.md`

- タイムスタンプは `date +%Y%m%d-%H%M%S` (ローカルTZ)
- `<slug>` は作業の本質を表す英語 kebab-case (3〜5語)
- 例: `20260411-104523-marketplace-backlog-skill.md`

## MD テンプレート

```markdown
# <タスク1行>

## 目的
<何を解決しようとしているか>

## 現状
<いまどこまで進んだか>

## 次
<次のアクション>
```

記述ルール:

- frontmatter なし。日時はファイル名、タイトルは h1 で十分
- 各セクションは1〜3行。余計な装飾・前置きを書かない
- 推測で補わない。会話から読み取れる事実のみ
- 未定の項目は `(未定)` と明示する

## 実行フロー

### Step 1: コンテキスト抽出

現在の会話から以下を抽出する。

- タスク(1行サマリー)
- 目的(なぜこの作業をしているか)
- 現状(今どこまで進んだか)
- 次のアクション(未定なら `(未定)`)

### Step 2: slug 生成

タスクの本質を英語 kebab-case で 3〜5 語に要約する。

### Step 3: タイムスタンプ取得

```sh
date +%Y%m%d-%H%M%S
```

### Step 4: ファイル書き出し

- 出力ディレクトリが無ければ `mkdir -p` で作成する
- `Write` ツールで `<タイムスタンプ>-<slug>.md` を書き出す

### Step 5: 報告

書き出したファイルの絶対パスを1行で報告する。
