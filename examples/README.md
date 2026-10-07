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

管理ツールは設定値を操作の入力に渡し、宣言された引数展開によって Compose の環境変数へ変換します。インスタンスごとの project name も設定で明示します。ポートはこの例では明示的に割り当てています。自動的な衝突検出・割り当ては将来の機能です。

## 入力の受け渡し（仕様案）

`executor: shell` は `command` と `arguments` を実行します。Docker Compose 専用の executor は設けません。作業ディレクトリは対象リポジトリのルートです。引数は配列として保持し、シェル文字列として評価しません。

```yaml
definitions:
  env_values: &env_values ["{name}={value}"]
  env_flags: &env_flags ["-e", "{name}={value}"]

actions:
  run:
    executor: shell
    command: env
    arguments:
      - expand:
          inputs: [HTTP_PORT, COMPOSE_PROJECT_NAME]
          format: *env_values
      - docker
      - compose
      - -f
      - compose.yaml
      - run
      - --detach
      - expand:
          inputs: [APP_MODE]
          format: *env_flags
      - web
```

入力定義はこの抜粋では省略しています。完全な例は `hello-service/git-service.yaml` にあります。操作ごとに `inputs` を定義し、管理設定の値を操作入力として渡します。`status`・`down` も同じ管理設定から入力を受け取り、暗黙の保存済み入力参照は使いません。

`definitions` は実行対象にしない共通定義領域です。`format` は YAML アンカーで共有し、展開対象の `inputs` は各操作で選びます。必須入力の欠落は実行前にエラーにし、整数は十進文字列に変換する想定です。各 format 要素は一つの引数になり、入力値中の空白などで分割しません。

上の例は、リポジトリルートで次を実行することに相当します。

```bash
env HTTP_PORT=18080 COMPOSE_PROJECT_NAME=hello-dev \
  docker compose -f compose.yaml run --detach -e APP_MODE=dev web
```

`env` が Compose プロセスへ環境変数を渡し、Compose の `run -e` がコンテナへ環境変数を渡します。環境変数の渡し先は、引数の位置とコマンドが決めます。`up` では `-e` を使わず、`env` で渡された `HTTP_PORT` をポート補間に使います。

```bash
# 未実装 CLI の設計例。
git_service action run --config ./examples/manager/dev.yaml \
  --environment dev --instance hello-dev --input APP_MODE=dev
```

`run` はサービスの公開ポートを既定では使いません。一時コンテナの削除は、返された ID を使って `docker rm -f <container-id>` で行います。このサンプルの実行先には Docker CLI と Docker 実行環境が必要です。既定候補の Alpine だけで Docker が利用できるという意味ではありません。

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
