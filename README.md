# GitHub Actions Input & Branch Test Lab

`workflow_dispatch` の input、`workflow_call` の input、Main ダミー + PR 本番パターンを検証するためのモックリポジトリです。

## リポジトリ構成

| ワークフロー | 目的 |
|---|---|
| `dispatch-all-input-types.yml` | string / choice / boolean / environment の各型を1回で確認 |
| `dispatch-required-optional.yml` | required と default 付き optional の差分 |
| `dispatch-branch-context.yml` | 手動実行時に選んだブランチの ref / sha / ファイル内容 |
| `reusable-callee.yml` | `workflow_call` 側（input 定義） |
| `reusable-caller.yml` | `workflow_call` 側（input 受け渡し） |
| `pr-test-dummy.yml` | Main ダミー（PR 検証用のプレースホルダ） |

## ログの読み方（grep 用）

各ワークフローは **固定フォーマット** で出力します。Actions のログ検索（Ctrl+F）で次を探すと一目でわかります。

| 検索キーワード | 意味 |
|---|---|
| `SUMMARY \| TEST=` | どのテストか・どのブランチか |
| `VERDICT \|` | 判定結果（YES/NO や workflow_version） |
| `[INPUT]` | 実際に渡された input 値 |
| `[BRANCH]` | ref_name / sha など |
| `[FILE]` | チェックアウトされたブランチの YAML 内容 |
| `HINT \|` | 次に何を比較すればよいか |

### よく使う判定

```
VERDICT | workflow_version=FEATURE_ENHANCED   ← PR で PR 側が使われた
VERDICT | workflow_version=MAIN_DUMMY         ← main のダミーが使われた
VERDICT | optional_with_default_matches_yaml_default=YES  ← default が YAML 通り
```

## 検証チェックリスト

### A. workflow_dispatch の input

1. **Actions → 対象ワークフロー → Run workflow** を開く
2. ブランチを `main` に固定して実行
3. 各 input を変えて再実行し、ログの `INPUT REPORT` と一致するか確認

| 確認項目 | 期待結果 |
|---|---|
| required 未入力 | UI で実行ボタンが無効、または実行前にエラー |
| optional 未入力 | `inputs.xxx` が空 or default 値 |
| boolean 未チェック | `false` として渡る |
| choice | 定義した choice のみ選択可能 |
| environment | 選択した Environment の protection ルールが適用される |

### B. ブランチ切り替え

1. `dispatch-branch-context.yml` を **Run workflow** で実行
2. ブランチを `main` → `feature/enhanced-pr-test` に切り替えて再実行
3. ログの `BRANCH CONTEXT` を比較

| 変数 | ブランチ切替で変わる？ |
|---|---|
| `github.ref_name` | はい |
| `github.sha` | はい（ブランチ先頭のコミット） |
| `github.workflow_ref` | はい（ブランチ名部分） |
| チェックアウトした workflow ファイル内容 | はい（そのブランチの版が使われる） |

### C. workflow_call の input

1. `reusable-caller.yml` を手動実行
2. `callee_message` / `callee_flag` を変えてログを確認
3. callee 側で `inputs.message` / `inputs.flag` が期待通りか確認

### D. Main ダミー + PR パターン

1. `feature/enhanced-pr-test` から `main` へ PR を作成
2. PR の Checks タブで `PR Test (Dummy on Main)` が走るか確認
3. Main のダミーは `echo placeholder` のみ、PR ブランチ版は `echo ENHANCED` と表示される

**ポイント:** 一致が必要なのは **ファイルパス**（`.github/workflows/pr-test-dummy.yml`）。`name:` や中身は PR 側で自由に変更可能。

## ブランチ一覧

| ブランチ | 内容 |
|---|---|
| `main` | 全ワークフローのベース版（PR ダミー含む） |
| `feature/enhanced-pr-test` | `pr-test-dummy.yml` を拡張版に差し替え（PR 検証用） |
| `feature/dispatch-defaults-b` | `dispatch-required-optional.yml` の default 値を変更（ブランチ差分検証用） |
