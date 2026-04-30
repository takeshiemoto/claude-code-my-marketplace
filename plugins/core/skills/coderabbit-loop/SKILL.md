---
name: core:coderabbit-loop
description: PRの CodeRabbit レビュー指摘を judge → patch → reply → re-review のループで自立的に消化する。「coderabbit ループ」「自動レビュー対応」「fb ループ回して」等の依頼時、または `/core:coderabbit-loop <PR#>` で呼び出す。
user-invocable: true
allowed-tools: Bash, Read, Edit, Write, Grep, Glob, Agent, SlashCommand, ToolSearch
---

# coderabbit-loop

PR に対する CodeRabbit AI のレビュー指摘を、収束条件を満たすまで反復的に処理する skill。各ラウンドで以下を回す:

1. **COLLECT** — CodeRabbit 指摘を取得して構造化
2. **JUDGE** — `core:arbiter` サブエージェントに採否を判定させる
3. **PATCH** — adopt 分を修正し、`type-check` と `lint` を pass させてコミット
4. **REPLY** — reject 分に PR 上で端的に返信
5. **RE-REVIEW** — `/coderabbit:review` を再実行して残課題を確認
6. 収束判定 → 続行 or 終了

main の context を肥大化させないため、判定は必ず Agent ツール経由で `core:arbiter` に委譲する。スキル本体は dispatcher として状態管理と修正適用に集中する。

## 引数

```sh
/core:coderabbit-loop <PR#> [--base <branch>] [--max-rounds <N>] [--dry-run]
```

- `<PR#>` 必須。GitHub PR 番号。
- `--base` 既定値はそのブランチの merge base。`/coderabbit:review --base <branch>` に渡す。
- `--max-rounds` 既定 3。これを超えたら強制終了。
- `--dry-run` 採否判定までは行うが、修正・reply・commit は行わない。

引数が不足していればユーザーに尋ねて停止。GitHub URL（`https://github.com/.../pull/N`）を貼られたら N を抽出する。

## 前提

- CWD が PR の対象 git リポジトリで、ブランチが PR の head と一致していること。`gh pr view <PR#> --json headRefName` と `git rev-parse --abbrev-ref HEAD` を比較し、不一致なら停止。
- `gh` 認証済み (`gh auth status`)。
- `coderabbit` CLI 認証済み (`coderabbit auth status`)。未認証なら `coderabbit auth login` を案内して停止。
- `codex` CLI が `codex --version` で応答する。`codex:rescue` skill が利用可能ならそちら優先、403/認証エラーで失敗したら `codex exec --skip-git-repo-check --sandbox workspace-write` に自動フォールバック。
- `core:arbiter` サブエージェントが存在 (`Agent` ツールから呼べる)。

## 状態管理

`.coderabbit-loop/<run-id>/` 配下に状態を永続化する（`<run-id>` は `<PR#>-<unix-epoch>`）。`.gitignore` に `.coderabbit-loop/` が無ければユーザーに追加を提案。

```text
.coderabbit-loop/<run-id>/
  meta.json                 # PR#, base, max-rounds, started-at
  rounds/
    01/
      raw.json              # CodeRabbit 生データ
      decisions.json        # arbiter の採否判定
      patches.json          # 適用したコミット (id, files, decision-ids)
      replies.json          # 送信した reply (decision-id, comment-id, reply-id)
      summary.md            # ラウンドサマリ
    02/
      ...
  final-report.md           # 収束時の最終サマリ
```

## フロー詳細

### Phase: COLLECT

ラウンド 1 は GitHub PR 上の既存レビューコメント、ラウンド 2 以降はローカル `coderabbit review` の結果が対象。

**ラウンド 1**:

```bash
gh api repos/{owner}/{repo}/pulls/<PR#>/comments --paginate \
  | jq '[.[] | select(.user.login == "coderabbitai[bot]") | {id, path, line, body}]' \
  > rounds/01/raw.json
```

**ラウンド 2 以降**:

```bash
coderabbit review --agent -t all --base <base> 2>&1 \
  | jq -s '[.[] | select(.type == "finding")]' \
  > rounds/NN/raw.json
```

`raw.json` が空なら **このラウンドで収束** とみなして Phase 6 へ。

### Phase: JUDGE

採否判定を `Agent` で `core:arbiter` に委譲する。**メイン側で raw.json の中身を読まない** (context保護)。

Agent prompt に渡すもの:

- 入力: `rounds/NN/raw.json` のフルパス
- 判定基準: `${CLAUDE_PLUGIN_ROOT}/skills/coderabbit-loop/rules/judge-criteria.md` のフルパス
- 出力先: `rounds/NN/decisions.json`
- 文脈: PR# / branch / base / round / 過去の `decisions.json` (ラウンド 2 以降は前ラウンドの判例参照)

arbiter には以下フォーマットで `decisions.json` を書き出させる:

```json
[
  {
    "id": "3164713954",
    "decision": "adopt|reject",
    "severity": "critical|major|minor|trivial",
    "path": "src/...",
    "line": 56,
    "reason": "1行理由",
    "fix_hint": "adopt時のみ。修正方針1行",
    "reply_template": "reject時のみ。reply-templates.md のキー名"
  }
]
```

`codex:rescue` 経由でも同じ JSON を出させる（rescue skill 側で `codex exec` まで回せる）。403 エラーが出たらメイン側で `codex exec --skip-git-repo-check --sandbox workspace-write -C <repo> "<prompt>"` に切り替えて同等プロンプトを投げ、stdout 末尾の JSON を `decisions.json` に保存。

### Phase: PATCH

`decisions.json` を読み込み、`decision == "adopt"` のものを **テーマ単位でグルーピング** して順次修正する。

グルーピング規則 (シンプルに):

1. `path` が同一 → 同コミット候補
2. `fix_hint` が同種パターン (例: `toPass` 除去 / `.first()` 除去 / placeholder→getByLabel) → 同コミット候補
3. グループ内の id 数が 10 件を超えたら 2 つに分割

各グループにつき:

1. `Edit` / `Write` で修正適用
2. `npm run type-check` (or 該当言語の型チェック) を実行。失敗したらそのグループの修正を `git restore --staged --worktree` で巻き戻し、`patches.json` に `failed: true` で記録、次グループへ
3. `npm run lint` (or `pnpm lint` 等プロジェクト準拠) を実行。失敗したら同様に巻き戻す
4. `git add` 対象ファイルを stage、`git commit` する。コミットメッセージは:

```text
fix(<scope>): <グループ要約>

CodeRabbit 指摘 (<id1>, <id2>, ...) 対応。

<fix_hint を箇条書き>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

`<scope>` は `path` の最上位 feature 名 (例: `e2e`)。コミット作成後 `patches.json` に追記。

`--dry-run` 時はこの Phase を skip し、`patches.json` に `dry_run: true` だけ記録。

### Phase: REPLY

`decision == "reject"` のものに対し PR 上で reply を送信する。

```bash
gh api repos/{owner}/{repo}/pulls/<PR#>/comments/<comment-id>/replies \
  -f body="<reply 本文>"
```

reply 本文は `rules/reply-templates.md` の `reply_template` キーから取得する。テンプレが無い場合は arbiter が出した `reason` をそのまま使う。短く端的に、感謝・前置き・絵文字なし (CLAUDE.md 規約)。

送信成功したら `replies.json` に `{decision_id, comment_id, reply_id, body}` を追記。

`--dry-run` 時は送信せず、本文だけ `replies.json` に `pending: true` で残す。

### Phase: RE-REVIEW

ラウンドの締めくくりとして再レビューを起動する。

```bash
coderabbit review --agent -t all --base <base>
```

ただし **ラウンド 1 が GitHub PR 上のレビュー対象だった場合**、最初の RE-REVIEW は coderabbit CLI で同じ branch に対して走らせる。これによってラウンド 2 以降は均質な入力に統一される。

### Phase: SUMMARY & 収束判定

`rounds/NN/summary.md` に以下を書く:

```markdown
# Round NN summary

- collected: <件数>
- adopt: <件数>
- reject: <件数>
- adopt-applied: <PATCHで成功した件数>
- adopt-failed: <PATCHで失敗した件数>
- replies-sent: <件数>
- commits: <SHA list>
```

**収束条件 (どれか1つ満たせば終了)**:

- `re_review.findings.length == 0`
- `decisions.adopt.length == 0`
- `round >= max-rounds`
- `re_review` 結果に `severity in ["critical", "major"]` が 0 件、かつ `adopt-applied == adopt`

満たさなければ次ラウンドへ。

## 終了処理

最終ラウンド完了後 `final-report.md` を生成し、ユーザーに提示する:

```markdown
# CodeRabbit feedback loop summary

- PR: #<PR#>
- ベース: <base>
- ラウンド数: <N>

## ラウンド別

| Round | collected | adopt | reject | applied | failed | replies |
| ----- | --------- | ----- | ------ | ------- | ------ | ------- |
| ...   |           |       |        |         |        |         |

## 追加コミット

- <SHA> <メッセージ1行>
- ...

## 残課題

- <re_review に残った findings の要約。なければ「無し」>

## 次アクション提案

- push: `git push`
- PR 更新確認: `gh pr view <PR#> --web`
```

`git push` は **絶対に skill 内で実行しない**。最終報告で提示するのみ。push 可否はユーザー判断。

## 失敗時の挙動

- `gh` / `coderabbit` / `codex` のいずれかが認証切れ: 即停止し、認証手順を案内
- `decisions.json` の生成に 3 度失敗 (codex / codex exec フォールバックでも) : 該当ラウンドを `BLOCKED` にして停止、`final-report.md` に状態を記録
- `type-check` / `lint` が **修正前から失敗している**: skill が壊したものではないので停止し、ユーザーに修正を依頼
- 同一ファイルを 2 連続ラウンドで触り、かつ adopt 件数が減らない: 「無限ループ寸前」と判断して停止

## 注意

- skill の各 Phase 出力 (raw.json / decisions.json / patches.json) はメイン context に流し込まない。必ずファイル経由で受け渡しする。Agent への指示も「ファイルパスを渡す」スタイルを徹底する。
- arbiter の判定を **盲従しない**。`type-check` / `lint` が落ちる修正は apply せず巻き戻す。
- ガイドラインの参照ファイル (`.claude/rules/` 等) があれば、JUDGE Phase の Agent prompt で fix-target ファイルと一緒に渡す。
- reply 文は CLAUDE.md 規約に従い「結論ファースト・前置きなし・感謝なし」。長くても 2 行。
