# reply-templates

CodeRabbit reject reply の文面テンプレート。CLAUDE.md 規約に従い「結論ファースト・前置きなし・感謝なし・絵文字なし」。

各テンプレは 1〜2 行。`{{...}}` はプレースホルダ、`decisions.json` の対応フィールドから埋める。

## download_filename_truthy

```text
download.suggestedFilename() の truthy 確認は内容比較ではなく存在確認のため、ガイドラインの「内容依存比較を避ける」方針と矛盾しません。修正不要と判断します。
```

## out_of_scope_pr

```text
本 PR のスコープ ({{pr_scope}}) 外のため、別タスクで扱います。
```

## existing_guard

```text
{{existing_location}} で既に待機しているため、ここでの追加待機は不要です。
```

## intentional_layout_branch

```text
この分岐は固定レイアウト差分の吸収であり、時間競合由来の非決定分岐ではありません。条件分岐 E2E 禁止規定の対象外です。
```

## over_defensive

```text
追加の defensive 実装は循環的複雑度を増やす割に実益が薄いため、現状維持とします。
```

## format_only

```text
動作には影響しないため不要と判断します。
```

## suggested_filename_truthy_alias

`download_filename_truthy` の別名。CodeRabbit が `suggestedFilename()` 表記を使ってくる場合用。同一文面。

```text
download.suggestedFilename() の truthy 確認は内容比較ではなく存在確認のため、ガイドラインの「内容依存比較を避ける」方針と矛盾しません。修正不要と判断します。
```

## existing_addressed

```text
{{commit_or_id}} で既に対応済みです。
```

## fixture_initial_navigation

```text
{{fixture_name}} は未使用変数ではなく、テスト前の初期遷移を発火させる役割があります。削除すると遷移前提が崩れるため現状維持とします。
```

## reply 出力規則

- 末尾に句点を必ずつける。
- 「修正します」と書かない (reject なので)。
- `{{...}}` プレースホルダが残ったまま送信しない。埋めるべき値が無い場合は当該プレースホルダ以下を削る。
- 1 つの reject に複数テンプレが当てはまる場合、最も具体的な 1 つを選ぶ。複数貼らない。
- テンプレに合致しない場合は `reply_template: "custom"` とし、`decisions.json` の `reason` を本文に流用 (1〜2 行に収める)。
