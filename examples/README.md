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

管理ツールは設定値を環境変数として Compose に渡し、インスタンスごとに Compose project name を分ける想定です。ポートはこの例では明示的に割り当てています。自動的な衝突検出・割り当ては将来の機能です。

## 入力の受け渡し（採用した仕様案）

入力は操作ごとの `inputs` で定義します。入力を自動的に環境変数や引数に変換しません。

- `process_env` は実行先の Compose プロセスにだけ環境変数を渡します。ホスト全体の環境設定は変更しません。
- `input` はその操作の入力を参照します。
- `instance_input` は起動時に保存したインスタンス入力を参照します。`status`・`down` でポートの再入力は不要です。
- `arguments` の `expand` は、指定した入力を宣言順に引数配列へ展開します。

```yaml
definitions:
  env_flags: &env_flags ["-e", "{name}={value}"]

actions:
  run:
    arguments:
      - run
      - expand:
          inputs: [APP_MODE]
          format: *env_flags
      - web
  debug:
    arguments:
      - run
      - expand:
          inputs: [APP_MODE, LOG_LEVEL]
          format: *env_flags
      - web
```

`definitions` は共通定義用として許可し、実行対象にしない領域です。YAML のアンカーとエイリアスで `format` だけを共有し、`inputs` は操作ごとに指定します。アンカーは参照より前に定義します。上の抜粋では入力定義を省略していますが、各操作の `inputs` に型と必須条件を定義する必要があります。`debug` は再利用の説明例であり、疑似リポジトリの実際の公開操作には含めていません。

`APP_MODE=dev` なら `['run', '--detach', '-e', 'APP_MODE=dev', 'web']` に展開します。値は一つの引数として保持し、シェル文字列として解釈しません。必須入力が欠けた場合は実行前にエラーにする想定です。整数は十進文字列として渡します。

```bash
# 未実装 CLI の設計例。hello-dev の保存済み設定を利用する。
git_service action run --config ./examples/manager/dev.yaml \
  --environment dev --instance hello-dev --input APP_MODE=dev

# 同等の直接実行。返されたコンテナ ID は後で削除する。
HTTP_PORT=18080 docker compose -p hello-dev \
  -f examples/hello-service/compose.yaml run --detach -e APP_MODE=dev web
# docker rm -f <返されたコンテナID>
```

`-e` は Compose の `run` が解釈し、コンテナ内に `APP_MODE` を渡します。`up` にはこの `-e` を使いません。`HTTP_PORT` は `process_env` から Compose のポート補間に使われ、コンテナ内へは渡されません。`run` は既定でサービスの公開ポートを使わないため、この例では HTTP の外部公開を行いません。

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
