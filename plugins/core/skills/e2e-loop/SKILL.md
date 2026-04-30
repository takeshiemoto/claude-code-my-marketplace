---
name: core:e2e-loop
description: E2Eテストを全パスし最終レビューでSHIP_OK判定が出るまで自動修正ループを回す。再発防止のためガイドラインへの教訓追記まで自動化する。「E2E ループ」「E2E 自動修正」「e2e 直して」等の依頼時、または /core:e2e-loop で呼び出す。
user-invocable: true
allowed-tools: Bash, Read, Edit, Write, Glob, Grep, Task, SlashCommand
---

# e2e-loop

E2E テストを全パスさせ、`codex:review` と `coderabbit` の両レビュアーで SHIP_OK 判定が出るまで自動反復する skill。`ralph-loop` に専用プロンプトを投げてループを回し、終了時に再発防止の教訓をガイドラインへ追記する。

## 引数

```sh
/core:e2e-loop [E2E実行コマンド]
```

引数が空なら `package.json` の `scripts` から自動検出する。検出失敗時は停止してユーザーに尋ねる。

## 前提（プリフライト）

必須プラグイン: `ralph-loop` / `codex` / `coderabbit`。**いずれか欠けたら直ちに停止** してインストール手順 + 認証手順を案内する。

### 存在確認の手段（共通仕様）

存在確認は **副作用ゼロ** で行う。具体的には:

1. システムリマインダ等で利用可能と分かっている skill / slash command の一覧から該当エントリの有無を確認する。
2. 一覧で確定できない場合のみ、`<command> --help` のように **副作用のないフラグ** で試す。
3. **副作用持ちの起点コマンド (`/codex:setup` / `/coderabbit:auth` 等) は存在確認では使わない。** 後段の認証確認でのみ実行する。

この順序を逆にすると、ユーザーの意図しない設定変更や認証フローが起動するリスクがある。

### 確認順序

1. **ralph-loop の存在確認** (`/ralph-loop`)
   - 不在なら停止:

     ```text
     ralph-loop プラグインが見つかりません。以下を実行してください:
       /plugin install ralph-loop@anthropics/claude-plugins-official
     ```

2. **codex の存在確認** (`/codex:review` が一覧に存在するかで判定。ループ本体が `/codex:review --background` を必須で呼ぶため、`/codex:setup` のみ存在で `/codex:review` が不在のケースは「不在」扱い)
   - 不在なら停止:

     ```text
     codex プラグインが見つかりません (または /codex:review が利用不可)。
       /plugin install codex@openai/codex-plugin-cc
       /codex:setup
     ```

   - 存在するなら次の **認証確認** に進む。`/codex:setup` は副作用持ち (認証フローを起動しうる) のため、実行前にユーザーへ「これから `/codex:setup` を実行して認証状態を確認します」と一言告知してから発火する。未認証なら停止して案内。

3. **coderabbit の存在確認** (`/coderabbit:review` が一覧に存在するかで判定)
   - 不在なら停止:

     ```text
     coderabbit プラグインが見つかりません。
       /plugin install coderabbit@coderabbitai/coderabbit-claude-code
     ```

   - 存在するなら次の **認証確認** に進む: CodeRabbit プラグインの推奨手順に従う。未認証なら停止して案内。

「直ちに停止」を優先し、複数プラグインの不在を 1 回でまとめ報告するモードは取らない（逐次停止）。

## 作業ディレクトリ準備

`.loop/` ディレクトリの作成は **E2E 実行コマンドが確定した直後・ループ起動の直前** に行う。プリフライトや PM 判定で停止する可能性がある段階では作らない（停止確定なら永続化ディレクトリの作成は不要）。

- `.loop/` ディレクトリを作成（既存ならそのまま）
- `.gitignore` に `.loop/` が含まれていなければ追加を提案（強制しない）

## E2E 実行コマンドの決定

`${ARGUMENTS}` が空でなければ、その文字列をそのまま `${E2E_COMMAND}` として採用しループ起動へ。

空の場合は以下で自動検出する。**検出失敗時は停止し、ユーザーに正しいコマンドを尋ねる**。

### パッケージマネージャ判定（リポジトリルートの lockfile）

- `pnpm-lock.yaml` → `pnpm`
- `yarn.lock` → `yarn`
- `bun.lock` または `bun.lockb` → `bun`
- `package-lock.json` → `npm`
- どれも無い、または複数同居 → 停止してユーザーに尋ねる

### E2E スクリプトの検出

1. Read で `package.json` を読む（無ければ停止）
2. `scripts` のキーから、キー名に `e2e` を **大文字小文字無視で含むもの** を全て抽出
3. 候補数で分岐:
   - **1 件**: そのキーを採用
   - **複数件**: 各キー名と本体を箇条書きで提示し、「どれを使いますか？ もしくは引数で直接コマンドを指定してください」と尋ねて停止
   - **0 件**: 停止してユーザーに尋ねる

### コマンド組立

`<PM> run <script>` 形式で組み立てる。例: pnpm + `test:e2e` → `pnpm run test:e2e`

## ループ起動

ralph-loop は受け取ったプロンプト文字列を `$ARGUMENTS` 経由で bash に渡すため、本文中のバッククオートや `$(...)` が **command substitution として展開されてしまう**。本 skill のループ本体プロンプト（`loop-prompt.md`）は backtick を多数含むため、**直接渡しは厳禁**。代わりに以下のブリッジで吸収する。

1. Read ツールで `${CLAUDE_PLUGIN_ROOT}/skills/e2e-loop/loop-prompt.md` を読む
2. プレースホルダを置換:
   - `{{E2E_COMMAND}}` → 確定した `${E2E_COMMAND}`
3. **置換後の本文を Write ツールで `.loop/prompt.md` に書き出す**（既存なら上書き）
4. SlashCommand ツールで以下を実行（プロンプト文字列に backtick / `$` / `$(` を含めないこと）:

   ```text
   /ralph-loop ".loop/prompt.md の指示を読み実行せよ。各イテレーションは .loop/prompt.md と .loop/ 配下の永続化ファイルを再読込してから始めること。" --completion-promise "SHIP_OK" --max-iterations 20
   ```

ループ本体の指示は `.loop/prompt.md` にあり、ralph-loop が再投入するブートストラップ文は短文かつ shell-safe な ASCII/かな漢字のみで構成する。

## 完了処理

ralph-loop が終了したら（SHIP_OK 到達 / max-iterations 到達 / BLOCKED）:

1. `.loop/report.md` の内容をユーザーに提示する。
2. `.loop/lessons.md` を読み、**採用候補が 1 件以上ある場合のみ** `## ガイドライン追記候補` として原則一覧を提示し、ユーザーに `[全採用 / 個別選択 / 全却下]` を尋ねる。以下は提示スキップ:
   - ファイルが存在しない
   - ファイルは存在するが本文が「候補なし。」のみ、または `## 新規` 節が空
   - skill 側の固有名フィルタ（注意節参照）を通した結果 0 件になる
   - その他、採用候補が 0 件と読める
3. 採用分はガイドラインファイルの末尾に skill が直接追記する。**書き込みは skill 本体が行い、ループ本体には書かせない** (ralph-loop が暴走しても外部ファイルへの汚染を防ぐ)。
   - 追記先決定順:
     1. リポジトリ root に `AGENTS.md` があれば `AGENTS.md`
     2. なければ `CLAUDE.md`
     3. どちらも無ければユーザーに `[AGENTS.md を新規作成 / 別パスを指定 / 追記スキップ]` を尋ねる
   - 追記後、`git diff` を表示して結果を見せる（コミットはしない）
4. ralph-loop の最終結論（SHIP_OK / BLOCKED / max 到達）と教訓追記の実施状況を 1 セットで報告して終了。

## 注意

- ガイドライン追記の最終承認は **必ず人間** が行う。skill が独断で `AGENTS.md` を書き換えることは禁止。
- 教訓は思想レベルの抽象原則のみ。固有名（関数名・ファイル名・テスト名）が含まれていたら採用前に skill が弾く。
- `.loop/` 配下の永続化ファイルはループ間で持ち越される。中断・再開時は前回の続きから再開する。
