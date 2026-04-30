# judge-criteria

CodeRabbit 指摘の採否を判定する基準。`core:arbiter` サブエージェントへ毎回先頭注入する。

## 大原則

1. **盲従しない**。AI レビュアの提案は「論点の発見」までは信頼するが、最終的な採否はコードベースの実状とプロジェクト文脈で決める。
2. **テストを緩める方向の adopt は出さない**。期待値の弱化、リトライ追加で flake を誤魔化す、`expect.toPass` 内に副作用ラップなどは reject。
3. **病的な正確さより単純さ**。KISS / YAGNI / DRY を優先。「将来の拡張に備えて」は基本却下。
4. **PR スコープを越えない**。新規 feature 追加 / 大規模リファクタ / 別領域の i18n 整備などは reject (DEFER)。

## 判定軸

### adopt にする典型

- 動作バグ・null 安全性・リソースリークに直結
- セキュリティ問題 (認証バイパス / SQLi / XSS / 重複副作用)
- 既存ガイドライン (`.claude/rules/` など) と明確に矛盾
- 1〜数行で機械的に直せ、読みやすさが上がる
- DOM 順序依存 (`.first()` / `.nth(0)` / `.last()`) や `expect.toPass` 副作用ラップなど、ガイドラインのアンチパターン集に明示されているもの
- ロケータのスコープ不足 (ページ全体検索) が flake 要因になり得るもの
- アクセシビリティ属性追加で改善するもの (label/role 連携、aria-label)

### reject にする典型

- `download.suggestedFilename()` のような **存在確認のみ** の assertion を「内容比較してない」という理由で削れと言ってくるもの (内容比較禁止であって存在確認は許容)
- placeholder → getByLabel への置換提案で **実装側に label/input 連携が無い** もの (実装変更が必要なら adopt 候補だが、リスクが高ければ慎重に。spec だけ書き換えるとテストが壊れる)
- spec へ防御的な `expect.poll` / `try-catch` を増やす提案
- 型重複や naming など本 PR スコープから外れる trivial 整理
- 既に対応済み箇所への重複指摘 (行ズレ / 別箇所と取り違え)
- `waitForURL` を `expect(page).toHaveURL` へ機械的に置換する提案など、現状で問題が出ていない書き換え
- レイアウト差分吸収のための条件分岐を「条件分岐 E2E は禁止」というガイドライン名で機械的に reject すべきと言ってくるもの (時間競合由来の分岐ではない)
- 「このループで AI レビューの提案を全部適用」しない判断を、ユーザーが過去に明示的に下している場合 (`.coderabbit-loop/<run-id>/rounds/*/decisions.json` の過去ラウンドで同じ趣旨を reject していたら整合させる)

### グレーゾーンの判断

- 採否判断にコード読みが必要なもの:
  1. ファイル全体を `Read` で読み込み
  2. 周辺コミット履歴を `git log -p <file> | head -200` で確認
  3. 修正コストとリスクを天秤
  4. 不確実なら reject (false positive 避けるより false negative 避ける、ただし「テスト緩める」方向の adopt は出さない)
- AI レビュアが提示する diff 例にバグがあるケースが頻発する。提案文ではなく **指摘の論点** だけ拾い、修正方針は自分で組み立て直す。

## 出力ルール

`decisions.json` を以下フォーマットで出力すること。前置き・後書き・コードブロック装飾は不要。

```json
[
  {
    "id": "<コメントID or finding ID>",
    "decision": "adopt|reject",
    "severity": "critical|major|minor|trivial",
    "path": "<相対パス>",
    "line": <number>,
    "reason": "<1行根拠>",
    "fix_hint": "<adoptの修正方針1行 or null>",
    "reply_template": "<rejectの返信テンプレキー or null>"
  }
]
```

### reply_template キー命名

`rules/reply-templates.md` のキーと一致させる。代表的なキー:

- `download_filename_truthy`: suggestedFilename truthy 確認は許容
- `out_of_scope_pr`: 本 PR スコープ外
- `existing_guard`: 既に他箇所で待機している
- `intentional_layout_branch`: 意図的なレイアウト差分吸収
- `over_defensive`: 過剰な defensive 提案
- `format_only`: フォーマット差のみ・現状で動作問題なし
- `custom`: テンプレに無い場合 → reason をそのまま使う

## 過去判定との整合性

ラウンド 2 以降は前ラウンドの `decisions.json` を必ず参照し、**同じ論点に同じ判定** を出す。揺れた場合は以下のいずれかを選ぶ:

- 過去ラウンドが reject かつ理由が依然有効 → 今回も reject
- 状況変化 (実装が変わって適用可能になった等) を `reason` に明記して切り替え
