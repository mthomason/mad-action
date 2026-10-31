[English](README.md) | [简体中文](README_zh.md) | **日本語**

# MAD Action

MADヘッドレスエンジンを使用して、GitHub Actions内で`.mad`レポートを実行します。リポジトリにコミットされたレポートをそのままパイプラインの一部として組み込み、データベースに対する更新（リフレッシュ）、レンダリング、およびビルドアーティファクトとしてのエクスポート結果のアップロードを行えます。

このリポジトリは[mthomason/MAD](https://github.com/mthomason/MAD)内の`satellites/mad-action/`から自動生成されています。Issueやプルリクエストは本リポジトリではなく、メインリポジトリ側で作成してください。

## クイックスタート

同梱されている`fixtures/London.portable.mad`はSQLiteベースのレポートであり、データベースサーバーや認証情報は不要です：

```yaml
name: Report

on:
  push:

jobs:
  london:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: mthomason/mad-action@v1
        with:
          command: export
          document: fixtures/London.portable.mad
          format: html
      - uses: actions/upload-artifact@v4
        with:
          name: london-report
          path: mad-output/
```

エクスポートにより`mad-output/report.html`が出力されます。その他のサブコマンド（`inspect`、`schema`、`query`、`refresh`）はジョブログに結果を出力します。

## ワークフロー内でのデータベース接続

`.mad`ファイルには接続定義（ドライバ、ホスト、ポート、およびデータベース名）が保存されますが、ユーザー名やパスワードは決して保存されません。認証情報はGitHub Secretsから渡し、CI上のサーバー接続先が保存値と異なる場合はオプションで上書きします：

```yaml
jobs:
  employees:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.4
        env:
          MYSQL_ROOT_PASSWORD: ${{ secrets.DB_PASSWORD }}
          MYSQL_DATABASE: employees
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping -h 127.0.0.1 -uroot -p$MYSQL_ROOT_PASSWORD"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=10
    steps:
      - uses: actions/checkout@v4
      - uses: mthomason/mad-action@v1
        with:
          command: refresh
          document: reports/employees.mad
          host: host.docker.internal
          port: '3306'
          database: employees
          username: root
          password: ${{ secrets.DB_PASSWORD }}
          rows: '50'
```

`host.docker.internal`はActionのコンテナ内から見たランナーのループバックアドレスであり、Actionが自動的にこのホストマッピングを追加します。なお、ドキュメント側には接続先データベースと一致するスキーマとクエリが必要です。MADリポジトリ内のMySQLフィクスチャ（`CLI/Fixtures/MySQL.employees.mad`）が実動サンプルとなっており、`MADMySQL/TestDatabase/mysql-test-db.sh`がそのクエリ対象である`employees`サンプルデータをシードします。

PostgreSQLの例：

```yaml
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_PASSWORD: ${{ secrets.DB_PASSWORD }}
          POSTGRES_DB: chinook
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U postgres"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=10
    steps:
      - uses: actions/checkout@v4
      - uses: mthomason/mad-action@v1
        with:
          command: refresh
          document: reports/chinook-pg.mad
          host: host.docker.internal
          port: '5432'
          database: chinook
          username: postgres
          password: ${{ secrets.DB_PASSWORD }}
```

MADリポジトリ内のPostgreSQLフィクスチャ（`CLI/Fixtures/Postgres.chinook.mad`）が実動サンプルであり、`MADPostgres/TestDatabase/postgres-test-db.sh`がそのクエリ対象である`chinook`サンプルデータをシードします。

## 認証情報の扱い

`.mad`フォーマットは意図的にソースデータベースの認証情報をドキュメント外に分離しています。アプリは認証情報をキーチェーンに保存し、ドキュメントのストアには接続定義のみを保持します。CI環境ではワークフローが`${{ secrets.* }}`から認証情報を供給するため、ドキュメント自体はクリーンなまま安全にリポジトリへコミットできます。コンパニオンファイルとキーチェーンの分離については[フォーマット仕様書 第5章](https://github.com/mthomason/MAD/blob/master/docs/05-companion-files.md)を参照してください。ワークフローファイルや`.mad`内にパスワードを直接書き込まず、必ずリポジトリまたは環境のSecretを使用してください。

`host`、`port`、および`database`はサーバー型データベースのドキュメントに適用されます。SQLiteベースのドキュメントはサーバー接続を持たずこれらの指定を拒否するため、同梱フィクスチャでは未設定のままにしてください。

## 入力パラメータ（Inputs）

| パラメータ | 必須 | デフォルト | 説明 |
|---|---|---|---|
| `command` | はい | — | `inspect`、`schema`、`query`、`refresh`、または`export`。 |
| `document` | はい | — | ワークスペースからの`.mad`への相対パス。 |
| `format` | いいえ | `html` | `html`、`html-flow`、`text`、`csv`、`json`、`json-report`、または`pdf`。PDFはLinuxイメージでは拒否されます。 |
| `out-dir` | いいえ | `mad-output` | `export`の出力先となるワークスペース相対ディレクトリ。ファイル名は`report.<ext>`になります。 |
| `username` | いいえ | — | ソースデータベースのユーザー名。Secretを渡してください。 |
| `password` | いいえ | — | ソースデータベースのパスワード。Secretを渡してください。 |
| `host` | いいえ | 保存されたホスト | ソースデータベースのホスト名の上書き指定。 |
| `port` | いいえ | 保存されたポート | ソースデータベースのポート番号の上書き指定。 |
| `database` | いいえ | 保存された名前 | ソースデータベース名の上書き指定。 |
| `rows` | いいえ | 全行 | 指定した行数で取得を停止します（`refresh`）。 |
| `image` | いいえ | `ghcr.io/mthomason/mad:latest` | 高度な設定：エンジンビルドのタグを固定します。 |

`@v1`はActionのメジャーバージョンタグであり、`:latest`はエンジンの最新タグです。通常の実行では最新のエンジンが使用されます。エンジンの再現性を固定したい場合は`image:`に短縮SHAタグを指定し、ラッパー自体も固定したい場合はActionをコミットハッシュでピン留めしてください。

## ライセンス

MIT（[LICENSE](LICENSE)を参照）。MADの名称およびアプリアートワークは[NOTICE](NOTICE)の下で留保されており、MITライセンス供与の対象外です。
