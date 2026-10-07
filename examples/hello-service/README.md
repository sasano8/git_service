# hello-service

独立した Git リポジトリを模したディレクトリです。入れ子の `.git` は含みません。

- `git-service.yaml`: 公開するクエリ、入力、操作の仕様案。
- `api/openapi.yaml`: `get_openapi` が加工せずに返すスキーマ。
- `catalog.json`: `get_catalog` が返すカタログ。
- `compose.yaml` と `public/`: 起動操作で使う小さな HTTP サービス。

設定値は `../manager/dev.yaml` に置き、対象リポジトリに `.env` を作る手順を省く想定です。実行例は [examples README](../README.md) を参照してください。
