**English** | [简体中文](README_zh.md)

# MAD Action

Run a `.mad` report in GitHub Actions with the MAD headless engine. A report
checked into a repository becomes part of its pipeline: refresh it against a
database, render it, and upload the export as a build artifact.

This repository is generated from `satellites/mad-action/` in
[mthomason/MAD](https://github.com/mthomason/MAD). Open issues and pull
requests there, not here.

## Quickstart

The checked-in `fixtures/London.portable.mad` is a SQLite-backed report that
needs no database server or credentials:

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

The export writes `mad-output/report.html`. The other verbs print to the job
log: `inspect`, `schema`, `query`, and `refresh`.

## A database in the workflow

A `.mad` stores its connection definition — driver, host, port, and database
name — but never a username or password. Pass credentials from secrets and
override the stored connection where the CI server differs:

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

`host.docker.internal` is the runner's loopback as seen from the action's
container; the action adds that host mapping for you. The document still needs
the schema and query that match the database it runs against — the MySQL
fixture in the MAD repository (`CLI/Fixtures/MySQL.employees.mad`) is a worked
example, and `MADMySQL/TestDatabase/mysql-test-db.sh` seeds the `employees`
sample it queries.

A PostgreSQL example:

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

The PostgreSQL fixture in the MAD repository
(`CLI/Fixtures/Postgres.chinook.mad`) is a worked example, and
`MADPostgres/TestDatabase/postgres-test-db.sh` seeds the `chinook` sample it
queries.

## Credentials

The `.mad` format deliberately keeps the source-database credentials out of
the document: the app stores them in the iOS keychain, and the store holds only
the connection definition. In CI the workflow supplies them from
`${{ secrets.* }}`; the document stays clean and can be committed.
[Chapter 5 of the format](https://github.com/mthomason/MAD/blob/master/docs/05-companion-files.md)
explains the companion-file and keychain split. Never put a password in a
workflow file or a `.mad`; use a repository or environment secret.

`host`, `port`, and `database` apply to a server-backed document. A
SQLite-backed document has no server connection and refuses them, so leave
them unset for the checked-in fixture.

## Inputs

| Input | Required | Default | Meaning |
|---|---|---|---|
| `command` | yes | — | `inspect`, `schema`, `query`, `refresh`, or `export`. |
| `document` | yes | — | Workspace-relative path to the `.mad`. |
| `format` | no | `html` | `html`, `html-flow`, `text`, `csv`, `json`, `json-report`, or `pdf`. PDF is refused by the Linux image. |
| `out-dir` | no | `mad-output` | Workspace-relative directory for `export`; the file is `report.<ext>`. |
| `username` | no | — | Source-database user name. Pass a secret. |
| `password` | no | — | Source-database password. Pass a secret. |
| `host` | no | stored host | Source-database host override. |
| `port` | no | stored port | Source-database port override. |
| `database` | no | stored name | Source-database name override. |
| `rows` | no | all | Stop after this many rows (`refresh`). |
| `image` | no | `ghcr.io/mthomason/mad:latest` | Advanced: pin an engine build. |

`@v1` is the action's moving major tag and `:latest` is the engine's; a run
uses the newest engine. Pin `image:` to a short-sha tag for a reproducible
engine, and pin the action to a commit if you need the wrapper frozen too.

## Licence

MIT; see [LICENSE](LICENSE). The MAD names and app artwork are reserved under
[NOTICE](NOTICE) and are not part of the MIT grant.
