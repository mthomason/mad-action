[English](README.md) | **简体中文**

# MAD Action

使用 MAD 无头引擎在 GitHub Actions 中运行 `.mad` 报表。检入仓库的报表会成为其流水线的一部分：针对数据库刷新它、渲染它，并将导出结果作为构建产物上传。

本仓库由 [mthomason/MAD](https://github.com/mthomason/MAD) 中的 `satellites/mad-action/` 生成。请在那里而不是此处提交 issue 和拉取请求。

## 快速开始

已检入的 `fixtures/London.portable.mad` 是一个由 SQLite 支持的报表，不需要数据库服务器或凭据：

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

导出会写入 `mad-output/report.html`。其他动词输出到作业日志：`inspect`、`schema`、`query` 和 `refresh`。

## 工作流中的数据库

`.mad` 存储其连接定义——驱动、主机、端口和数据库名——但从不存储用户名或密码。从 secrets 传递凭据，并在 CI 服务器不同的情况下覆盖存储的连接：

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

`host.docker.internal` 是从 action 的容器中看到的 runner 回环地址；action 会为你添加该主机映射。文档仍然需要与其所针对运行的数据库相匹配的架构和查询——MAD 仓库中的 MySQL 测试夹具（`CLI/Fixtures/MySQL.employees.mad`）是一个完整示例，`MADMySQL/TestDatabase/mysql-test-db.sh` 会初始化它所查询的 `employees` 示例。

一个 PostgreSQL 示例：

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

MAD 仓库中的 PostgreSQL 测试夹具（`CLI/Fixtures/Postgres.chinook.mad`）是一个完整示例，`MADPostgres/TestDatabase/postgres-test-db.sh` 会初始化它所查询的 `chinook` 示例。

## 凭据

`.mad` 格式有意不将源数据库凭据放入文档：应用将其存储在 iOS 钥匙串中，存储只保留连接定义。在 CI 中，工作流从 `${{ secrets.* }}` 提供它们；文档保持干净，可以提交。[格式第 5 章](https://github.com/mthomason/MAD/blob/master/docs/zh/05-companion-files.md)解释了伴随文件与钥匙串的分工。切勿将密码放入工作流文件或 `.mad`；请使用仓库或环境 secret。

`host`、`port` 和 `database` 适用于由服务器支持的文档。由 SQLite 支持的文档没有服务器连接，会拒绝这些参数，因此对于已检入的测试夹具请将它们留空。

## 输入

| 输入 | 必填 | 默认值 | 含义 |
|---|---|---|---|
| `command` | 是 | — | `inspect`、`schema`、`query`、`refresh` 或 `export`。 |
| `document` | 是 | — | `.mad` 的工作区相对路径。 |
| `format` | 否 | `html` | `html`、`html-flow`、`text`、`csv`、`json`、`json-report` 或 `pdf`。Linux 镜像拒绝 PDF。 |
| `out-dir` | 否 | `mad-output` | `export` 的工作区相对目录；文件名为 `report.<ext>`。 |
| `username` | 否 | — | 源数据库用户名。请传入 secret。 |
| `password` | 否 | — | 源数据库密码。请传入 secret。 |
| `host` | 否 | 存储的主机 | 源数据库主机覆盖值。 |
| `port` | 否 | 存储的端口 | 源数据库端口覆盖值。 |
| `database` | 否 | 存储的名称 | 源数据库名称覆盖值。 |
| `rows` | 否 | 全部 | 达到此行数后停止（`refresh`）。 |
| `image` | 否 | `ghcr.io/mthomason/mad:latest` | 高级：固定引擎构建版本。 |

`@v1` 是该 action 的移动主版本标签，`:latest` 是引擎的标签；每次运行都使用最新的引擎。将 `image:` 固定到 short-sha 标签可获得可复现的引擎；如果需要连封装层也一并冻结，则将 action 固定到某个提交。

## 许可证

MIT；参见 [LICENSE](LICENSE)。MAD 名称和应用美术资源依据 [NOTICE](NOTICE) 保留，不属于 MIT 授权的一部分。
