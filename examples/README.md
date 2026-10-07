# 疑似リポジトリと CLI 実行例

ここでは、リポジトリ側の公開契約と、管理ツール側の設定を分けて示します。

```text
examples/
├── hello-service/            # 独立リポジトリを模したディレクトリ
│   ├── git-service.yaml      # 公開契約（仕様案）
│   ├── api/openapi.yaml
│   ├── catalog.json
│   ├── compose.yaml
│   ├── public/index.html
│   └── README.md
└── manager/
    └── dev.yaml              # リポジトリ外の設定（仕様案）
```

## git_service CLI の仕様案

**CLI は未実装です。以下のコマンド、オプション、出力は利用体験を示す設計例であり、現時点では実行できません。** 作業ディレクトリは、このプロジェクトのルートを想定しています。

### 公開情報の発見と取得

```bash
# クエリ・入力・操作の一覧と説明を取得する。
git_service inspect --repo ./examples/hello-service

# スキーマをそのまま標準出力に返す。
git_service query get_openapi --repo ./examples/hello-service

# カタログを取得する。
git_service query get_catalog --repo ./examples/hello-service

# 独立した Git リポジトリでは ref を指定できる想定。
# 下記 URL は例示用であり、存在するリポジトリを示していません。
git_service query get_openapi \
  --repo https://git.example.com/team/hello-service.git \
  --ref v1.0.0
```

ローカルの疑似リポジトリは作業ツリーを読む想定です。リモート取得は、指定 ref にある公開契約と成果物を読み、取得元のコミット SHA を追跡する想定です。

### 外部設定を使った起動・状態確認・停止

```bash
# 管理設定から対象リポジトリと HTTP_PORT=18080 を解決して起動する。
git_service up --config ./examples/manager/dev.yaml \
  --environment dev --instance hello-dev

# 想定出力: instance=hello-dev url=http://localhost:18080
curl http://localhost:18080/
# Hello from git_service

git_service status --config ./examples/manager/dev.yaml \
  --environment dev --instance hello-dev

# 同じリポジトリを別インスタンスとして起動する（HTTP_PORT=18081）。
git_service up --config ./examples/manager/dev.yaml \
  --environment dev --instance hello-dev-2

git_service down --config ./examples/manager/dev.yaml \
  --environment dev --instance hello-dev

git_service down --config ./examples/manager/dev.yaml \
  --environment dev --instance hello-dev-2
```

管理ツールは設定値を操作の入力に渡し、宣言された pipeによって Compose の環境変数へ変換します。インスタンスごとの project name も設定で明示します。ポートはこの例では明示的に割り当てています。自動的な衝突検出・割り当ては将来の機能です。

## 入力の受け渡し（仕様案）

```yaml
definitions:
  env_values: &env_values
    pipe:
      - items($inputs)
      - map("{k}={v}")
      - quote()
      - map("-e {v}")
      - concat(" ")

actions:
  run:
    executor: shell
    command:
      - docker compose -f compose.yaml run --detach
      - *env_values
      - web
```

入力定義は省略しています。完全な例は `hello-service/git-service.yaml` を参照してください。`$inputs` はその操作の入力です。管理設定から値を渡し、必須項目を検証します。

`definitions` で pipe を共有します。`command` の各断片をスペースで連結して POSIX sh で実行する仕様案です。入力は `quote()` でクォート・エスケープします。作業ディレクトリはリポジトリルートです。

`APP_MODE=dev` は `-e APP_MODE=dev` に展開され、コンテナへ直接渡されます。`quote()` の後の `map` は、引用済み文字列を `{v}` として受け取り、固定の `-e ` を付けます。

完全な設定例では Compose の補間用に `process_values` も定義し、コマンドの先頭に環境変数代入を置いています。`env` コマンドは使いません。`env_values` は全操作入力を `-e` 引数へ変換します。

```bash
HTTP_PORT=18080 COMPOSE_PROJECT_NAME=hello-dev APP_MODE=dev \
  docker compose -f compose.yaml run --detach \
  -e HTTP_PORT=18080 -e COMPOSE_PROJECT_NAME=hello-dev -e APP_MODE=dev web
```

公開ポートは既定では使いません。一時コンテナは返された ID を使って `docker rm -f <container-id>` で削除します。

```bash
# 未実装 CLI の設計例。
git_service action run --config ./examples/manager/dev.yaml \
  --environment dev --instance hello-dev --input APP_MODE=dev
```

このサンプルの実行先には Docker CLI と Docker 実行環境が必要です。

## 現在実行できる同等の操作

git_service を使わずに、このサンプルのファイル取得と起動を確かめるコマンドです。起動には Docker と Compose plugin が必要です。イメージ未取得の場合はダウンロードされます。

```bash
# クエリの実体は、まず単純なファイル読み取り。
cat examples/hello-service/api/openapi.yaml
cat examples/hello-service/catalog.json

# 外部設定で指定した値を、ここでは手動で環境変数として渡す。
HTTP_PORT=18080 docker compose -p hello-dev \
  -f examples/hello-service/compose.yaml up -d
HTTP_PORT=18081 docker compose -p hello-dev-2 \
  -f examples/hello-service/compose.yaml up -d

curl http://localhost:18080/
curl http://localhost:18081/

HTTP_PORT=18080 docker compose -p hello-dev \
  -f examples/hello-service/compose.yaml ps

HTTP_PORT=18080 docker compose -p hello-dev \
  -f examples/hello-service/compose.yaml down
HTTP_PORT=18081 docker compose -p hello-dev-2 \
  -f examples/hello-service/compose.yaml down
```

この直接実行例は `manager/dev.yaml` を読みません。管理設定の解決・入力検証・操作の振り分けを git_service が担う、という境界を示しています。

## pipe のフィルター

- `items($inputs)`：辞書をキー `k` と値 `v` の組の列にする。
- `map("{k}={v}")`：各要素をテンプレートで文字列にする。
- `quote()`：各文字列をシェル向けにクォート・エスケープする。
- `concat(" ")`：要素をスペースで結合し、最終文字列を返す。
