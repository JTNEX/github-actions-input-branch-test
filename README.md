# GitHub Actions Input & Branch Test Lab

`workflow_dispatch` の input、`workflow_call` の input、Main ダミー + PR 本番パターンを検証するためのモックリポジトリです。

## 結論（Main ダミーに必須なもの）

| 項目 | Main に必要？ | 説明 |
|---|---|---|
| **ファイルパス**（`.github/workflows/xxx.yml`） | **必須** | 同じパスが default ブランチに無いと PR / 手動実行の対象にならない |
| **試したい `on:` トリガー** | **必須** | PR 検証なら `pull_request`、手動実行なら `workflow_dispatch` を main に書いておく |
| 最低限の `jobs`（1 step でよい） | **必須** | 空ファイルは YAML として無効 |
| `name:`（表示名） | 不要 | PR / feature 側で自由に変えてよい |
| `inputs:` | **不要** | feature だけに足しても、ブランチ選択で UI に出る（Test 05 で確認） |
| jobs / steps の中身 | 不要 | `echo dummy` で十分。PR では **PR ブランチ側** が使われる |

### Main ダミーの最小例

```yaml
# .github/workflows/ci.yml  ← このパスが main にあることが唯一の硬い条件
name: CI   # 何でもよい

on:
  pull_request:       # PR で試すなら
  workflow_dispatch:  # 手動実行も試すなら

jobs:
  placeholder:
    runs-on: ubuntu-latest
    steps:
      - run: echo "dummy"
```

### 実行時に何が使われるか

| 操作 | フォーム（inputs） | 実行される YAML |
|---|---|---|
| PR を出す | — | **PR ブランチ** |
| Run workflow + ブランチ選択 | **選んだブランチ** | **選んだブランチ** |

**以前の誤り:** 「UI の inputs は main 固定」→ **誤り**。`Use workflow from` でブランチを変えると、inputs / default もそのブランチの YAML に追従する（Test 05・手動確認済み）。

---

## リポジトリ構成

| ワークフロー | 目的 |
|---|---|
| `dispatch-all-input-types.yml` | string / choice / boolean / environment の各型を1回で確認 |
| `dispatch-required-optional.yml` | required と default 付き optional の差分 |
| `dispatch-branch-context.yml` | 手動実行時に選んだブランチの ref / sha / ファイル内容 |
| `reusable-callee.yml` | `workflow_call` 側（input 定義） |
| `reusable-caller.yml` | `workflow_call` 側（input 受け渡し） |
| `pr-test-dummy.yml` | Main ダミー（PR 検証用のプレースホルダ） |
| `test-extra-input.yml` | feature だけの input が UI に出るか（**出る**と確認済み） |

## ログの読み方（grep 用）

| 検索キーワード | 意味 |
|---|---|
| `SUMMARY \| TEST=` | どのテストか・どのブランチか |
| `VERDICT \|` | 判定結果 |
| `[INPUT]` | 実際に渡された input 値 |
| `[BRANCH]` | ref_name / sha など |
| `[FILE]` | チェックアウトされたブランチの YAML 内容 |

### よく使う判定

```
VERDICT | workflow_version=FEATURE_ENHANCED   ← PR で PR 側が使われた
VERDICT | workflow_version=MAIN_DUMMY         ← main のダミーが使われた
VERDICT | feature_only_input_received=YES     ← feature 専用 input が渡った
```

## 検証チェックリスト

### A. workflow_dispatch の input（Test 01 / 02）

Actions → 対象ワークフロー → Run workflow → ブランチを選んで実行。

| 確認項目 | 期待結果 |
|---|---|
| required 未入力 | UI で実行不可 |
| optional 未入力 | 選んだブランチの `default:` or 空 |
| ブランチを変える | **フォームの項目・default も変わる**（Test 02 / 05） |

### B. ブランチ切り替え（Test 03）

`main` と `feature/dispatch-defaults-b` で Run workflow し、`[BRANCH] ref_name` と `[FILE]` を比較。

### C. workflow_call（Test 04b → 04a）

caller の `[INPUT sent]` と callee の `[INPUT from caller]` が一致するか。

### D. Main ダミー + PR（PR Test）

[PR #1](https://github.com/JTNEX/github-actions-input-branch-test/pull/1): ログで `VERDICT | workflow_version=FEATURE_ENHANCED`。

### E. feature だけの input（Test 05）— 確認済み

対象: **Test 05** / ブランチ `feature/extra-input-only`

| ブランチ | UI のフォーム |
|---|---|
| `main` | `common_input` のみ |
| `feature/extra-input-only` | `common_input` + **`feature_only_input`**（default も feature 側） |

```bash
gh workflow run "Test 05 - Extra Input (main baseline)" \
  --repo JTNEX/github-actions-input-branch-test \
  --ref feature/extra-input-only \
  -f common_input=cli-common \
  -f feature_only_input=cli-feature-only
```

## ブランチ一覧

| ブランチ | 内容 |
|---|---|
| `main` | 全ワークフローのベース版 |
| `feature/enhanced-pr-test` | `pr-test-dummy.yml` 拡張版（PR #1） |
| `feature/dispatch-defaults-b` | Test 02 の default を `BRANCH_B_DEFAULT` に変更 |
| `feature/extra-input-only` | Test 05 に `feature_only_input` 追加（PR #2） |
