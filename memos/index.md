# リモートアクションの作り方

## Composite Action 編

### メタデータファイル

アクションの内容を記載したファイルを `メタデータファイル` という。

メタデータファイルは `action.yaml` という名称で、リポジトリのルートディレクトリに配置する必要がある。

メタデータファイルの中身は以下のような形式 (ローカルアクションとほぼ同じ)。

```yaml
name: Action Name
description: Description of Action
inputs:
  var1:
    required: true
    description: hogehoge
  var2:
    required: false
    default: fugafuga

outputs:
  var1: ...

runs:
  using: composite
  steps:
    - shell: bash
      run: echo "OK"
```

### 自動テスト

GitHub Actions の自動テストでは以下の 4 フェーズを意識すると良い。

1. Setup: 自動テストに必要なものを事前に生成するフェーズ
1. Exercise: アクションを実行するフェーズ
1. Verify: アクションの結果が正しいことを確かめるフェーズ
1. Teardown: 自動テストで作成されたアーティファクトを削除するフェーズ

本リポジトリのアクションに当てはめると以下のような感じ。

#### 1. Setup

PR を作るための適当な差分を用意する。

```yaml
- id: setup
  env:
    ARTIFACT_FILE: "dummy.txt"
  run: echo "this is a test artifact" >> "${ARTIFACT_FILE}"
```

#### 2. Exercise

アクションを実行する。

```yaml
- id: exercise
  uses: ./action.yaml
  with:
    message: ${MESSAGE}
```

#### 3. Verify

アクション結果が期待通りであることを確かめる。

```yaml
- id: verify
  run: |
    set -x
    test "$(gh pr view "${{ steps.exercise.outputs.branch }}" --json title -q .title)" = "${MESSAGE}"
```

今回であれば gh コマンドに `--json` を指定することで JSON 形式で結果を取得し (これだけだと `{"title": "..."}` みたいな形式) 、さらに `-q .title` も付けることで title の値のみを取り出している。

#### 4. Teardown

テスト用に作成されたものを削除して元の状態に戻すフェーズ。
今回の例では以下の 2 つを削除する必要がある。

- 作成された PR
- 上記 PR を作成するために使用したブランチ

そのため以下のようなステップを定義して上記 2 つを削除している。

※ ダミーファイルは GHA を実行している runner ホスト内の中間生成物なので気にしなくて良い。

```yaml
- id: teardown
  if: always()
  run: gh pr close "${{ github.head_ref }}" -d || true
```

ポイントは以下の 2 点。

- テスト結果が期待通りにならなかった場合には直前の Verify フェーズが異常終了になる
  そのような場合でも Teardown が確実に実行されるように、`if: always()` を指定している
- (今回は気にしなくて良いが) Teardonw で実行するコマンドが複数ある場合には、途中で失敗しても最後まで実行されるようにする必要がある
  そのため、`|| true` とパイプでつなぎ、コマンドが異常終了しても処理が継続されるようにしている
