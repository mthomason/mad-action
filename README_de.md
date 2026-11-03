[English](README.md) | [简体中文](README_zh.md) | [日本語](README_ja.md) | [Português (Brasil)](README_pt-BR.md) | **Deutsch** | [Español](README_es.md)

# MAD Action

Führen Sie einen `.mad`-Bericht mit der MAD-headless-Engine in GitHub Actions
aus. Ein Bericht, der in ein Repository eingecheckt ist, wird Teil seiner
Pipeline: gegen eine Datenbank aktualisieren, rendern und den Export als
Build-Artefakt hochladen.

Dieses Repository wird aus `satellites/mad-action/` in
[mthomason/MAD](https://github.com/mthomason/MAD) generiert. Stellen Sie Issues
und Pull Requests dort, nicht hier.

## Schnellstart

Die eingecheckte `fixtures/London.portable.mad` ist ein SQLite-gestützter
Bericht, der keinen Datenbankserver und keine Zugangsdaten benötigt:

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

Der Export schreibt `mad-output/report.html`. Die anderen Verben geben ins
Job-Log aus: `inspect`, `schema`, `query` und `refresh`.

## Eine Datenbank im Workflow

Eine `.mad`-Datei speichert ihre Verbindungsdefinition — Treiber, Host, Port und
Datenbankname —, aber niemals einen Benutzernamen oder ein Kennwort. Übergeben
Sie Zugangsdaten aus Secrets und überschreiben Sie die gespeicherte Verbindung
dort, wo der CI-Server abweicht:

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

`host.docker.internal` ist der Loopback des Runners aus Sicht des Containers der
Action; die Action fügt diese Host-Zuordnung für Sie hinzu. Das Dokument braucht
trotzdem das Schema und die Abfrage, die zur Datenbank passen, gegen die es
läuft — das MySQL-Fixture im MAD-Repository (`CLI/Fixtures/MySQL.employees.mad`)
ist ein ausgearbeitetes Beispiel, und `MADMySQL/TestDatabase/mysql-test-db.sh`
legt das `employees`-Beispiel an, das es abfragt.

Ein PostgreSQL-Beispiel:

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

Das PostgreSQL-Fixture im MAD-Repository (`CLI/Fixtures/Postgres.chinook.mad`) ist
ein ausgearbeitetes Beispiel, und `MADPostgres/TestDatabase/postgres-test-db.sh`
legt das `chinook`-Beispiel an, das es abfragt.

## Zugangsdaten

Das `.mad`-Format hält die Zugangsdaten der Quelldatenbank bewusst aus dem
Dokument heraus: Die App speichert sie im iOS-Schlüsselbund, und der Speicher
enthält nur die Verbindungsdefinition. In CI liefert der Workflow sie aus
`${{ secrets.* }}`; das Dokument bleibt sauber und kann eingecheckt werden.
[Kapitel 5 des Formats](https://github.com/mthomason/MAD/blob/master/docs/05-companion-files.md)
erklärt die Aufteilung zwischen Begleitdateien und Schlüsselbund. Legen Sie
niemals ein Kennwort in eine Workflow-Datei oder eine `.mad`-Datei; verwenden Sie
ein Repository- oder Umgebungs-Secret.

`host`, `port` und `database` gelten für ein servergestütztes Dokument. Ein
SQLite-gestütztes Dokument hat keine Serververbindung und verweigert sie, lassen
Sie sie also für das eingecheckte Fixture ungesetzt.

## Eingaben

| Eingabe | Erforderlich | Standard | Bedeutung |
|---|---|---|---|
| `command` | ja | — | `inspect`, `schema`, `query`, `refresh` oder `export`. |
| `document` | ja | — | Workspace-relativer Pfad zur `.mad`-Datei. |
| `format` | nein | `html` | `html`, `html-flow`, `text`, `csv`, `json`, `json-report` oder `pdf`. PDF wird vom Linux-Image verweigert. |
| `out-dir` | nein | `mad-output` | Workspace-relatives Verzeichnis für `export`; die Datei heißt `report.<ext>`. |
| `username` | nein | — | Benutzername der Quelldatenbank. Übergeben Sie ein Secret. |
| `password` | nein | — | Kennwort der Quelldatenbank. Übergeben Sie ein Secret. |
| `host` | nein | gespeicherter Host | Host-Überschreibung der Quelldatenbank. |
| `port` | nein | gespeicherter Port | Port-Überschreibung der Quelldatenbank. |
| `database` | nein | gespeicherter Name | Namensüberschreibung der Quelldatenbank. |
| `rows` | nein | alle | Nach dieser Anzahl Zeilen aufhören (`refresh`). |
| `image` | nein | `ghcr.io/mthomason/mad:latest` | Fortgeschritten: einen Engine-Build fixieren. |

`@v1` ist der bewegliche Major-Tag der Action und `:latest` der der Engine; ein
Lauf verwendet die neueste Engine. Fixieren Sie `image:` auf einen Short-SHA-Tag
für eine reproduzierbare Engine und die Action auf einen Commit, wenn auch der
Wrapper eingefroren sein soll.

## Lizenz

MIT; siehe [LICENSE](LICENSE). Die MAD-Namen und das App-Artwork sind gemäß
[NOTICE](NOTICE) vorbehalten und nicht Teil der MIT-Gewährung.
