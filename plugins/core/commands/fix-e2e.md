---
description: E2Eテストを全パスし最終レビューでSHIP_OK判定が出るまで自動修正ループを回す
argument-hint: [E2E実行コマンド。省略時は package.json から自動検出]
allowed-tools: Bash, Read, Edit, Write, Glob, Grep, Task, SlashCommand
---

# fix-e2e

ユーザー指定 E2E 実行コマンド: `${ARGUMENTS}`（空なら自動検出）

これから ralph-loop プラグインに専用プロンプトを投入し、SHIP_OK が出るまで自動反復させます。

## Step 1: プリフライトチェック（依存プラグインの存在確認）

**必須**: ralph-loop / pr-review-toolkit。**任意**: codex / coderabbit（あれば追加レビュアとして使用）。

`${REVIEWERS}` を `[pr-review-toolkit]` で初期化し、以下の順で確認していく。

1. **ralph-loop の存在確認**（`/ralph-loop` が利用可能か試行）
   - 不在なら **直ちに停止**してインストール手順を提示:

     ```text
     ralph-loop プラグインが見つかりません。以下を実行してください:
       /plugin install ralph-loop@anthropics/claude-plugins-official
     ```

2. **pr-review-toolkit の存在確認**（`/pr-review-toolkit:review-pr` が利用可能か試行）
   - 不在なら **直ちに停止**してインストール手順を提示:

     ```text
     pr-review-toolkit プラグインが見つかりません。
     インストールしてから再実行してください。
     ```

3. **codex の存在確認**（`/codex:setup` 相当を試行）
   - 利用可能なら `${REVIEWERS}` に `codex` を追加
   - 不在なら以下を表示して続行（停止しない）:

     ```text
     codex プラグインが見つかりません（任意）。使用したい場合:
       /plugin install codex@openai/codex-plugin-cc
       /codex:setup
     ```

4. **coderabbit の存在確認**
   - 利用可能なら `${REVIEWERS}` に `coderabbit` を追加
   - 不在なら以下を表示して続行（停止しない）:

     ```text
     coderabbit プラグインが見つかりません（任意）。使用したい場合:
       /plugin install coderabbit@coderabbitai/coderabbit-claude-code
     ```

## Step 2: 認証確認

- `${REVIEWERS}` に `codex` が含まれる場合: `/codex:setup` を実行して認証状態を確認
- `${REVIEWERS}` に `coderabbit` が含まれる場合: プラグインの推奨手順で認証状態を確認
- 未認証のものがあれば、ユーザーに認証手順を案内して停止

`pr-review-toolkit` 単独の場合は追加認証不要なので Step 3 へ。

## Step 3: 作業ディレクトリ準備

- `.loop/` ディレクトリを作成（既存ならそのまま）
- `.gitignore` に `.loop/` が含まれていなければ、含めることをユーザーに提案（追加は強制しない）

## Step 4: E2E 実行コマンドの決定

`${ARGUMENTS}` が空でなければ、その文字列をそのまま `${E2E_COMMAND}` として採用し Step 5 へ。

空の場合は以下で自動検出する。**この検出に失敗したら停止し、ユーザーに正しいコマンドを尋ねる**。

### 4.1 パッケージマネージャ判定（リポジトリルートの lockfile）

- `pnpm-lock.yaml` → `pnpm`
- `yarn.lock` → `yarn`
- `bun.lock` または `bun.lockb` → `bun`
- `package-lock.json` → `npm`
- 上記どれも無い → 停止してユーザーに実行方法を尋ねる

複数 lockfile が同居するリポジトリは想定外として停止し、どれを使うか尋ねる。

### 4.2 E2E スクリプトの検出

1. Read ツールで `package.json` を読む（無ければ停止）
2. `scripts` のキーから、キー名に `e2e` を **大文字小文字無視で含むもの** を全て抽出
3. 候補数で分岐:
   - **1 件**: そのキーを採用
   - **複数件**: 各キー名と `scripts[key]` の本体を箇条書きで提示し、「どれを使いますか？ もしくは引数で直接コマンドを指定してください」と尋ねて停止
   - **0 件**: 停止してユーザーに E2E 実行方法を尋ねる

### 4.3 コマンド組立

`<PM> run <script>` 形式で組み立てる（npm/pnpm/yarn/bun いずれもこの形式で動作）。

例:

- pnpm + `test:e2e` → `pnpm run test:e2e`
- npm + `e2e` → `npm run e2e`

組み立てた文字列を `${E2E_COMMAND}` として Step 5 に渡す。

## Step 5: ループ起動

プロンプト本体は `${CLAUDE_PLUGIN_ROOT}/prompts/fix-e2e-loop.md` にあります。

1. Read ツールで `${CLAUDE_PLUGIN_ROOT}/prompts/fix-e2e-loop.md` を読む
2. 内容のプレースホルダを置換:
   - `{{E2E_COMMAND}}` → Step 4 で確定した `${E2E_COMMAND}`
   - `{{REVIEWERS}}` → Step 1 で確定した `${REVIEWERS}` をカンマ区切り文字列にしたもの（例: `pr-review-toolkit,codex,coderabbit` / `pr-review-toolkit,codex` / `pr-review-toolkit`）
3. SlashCommand ツールで以下を実行:

   ```text
   /ralph-loop "<置換後のプロンプト全文>" --completion-promise "SHIP_OK" --max-iterations 20
   ```

## Step 6: 完了処理

ralph-loop が終了したら（SHIP_OK 到達 / max-iterations 到達 / BLOCKED）、
`.loop/report.md` の内容をユーザーに提示し、最終承認を仰いでください。
ここが唯一の人間介入点です。
