[English](README.md) | [简体中文](README_zh.md) | [日本語](README_ja.md) | [Português (Brasil)](README_pt-BR.md) | [Deutsch](README_de.md) | **Español**

# MAD Action

Ejecute un informe `.mad` en GitHub Actions con el motor sin interfaz gráfica de MAD. Un informe
incluido en un repositorio pasa a formar parte de su canalización: actualícelo contra una
base de datos, renderícelo y cargue la exportación como artefacto de compilación.

Este repositorio se genera desde `satellites/mad-action/` en
[mthomason/MAD](https://github.com/mthomason/MAD). Abra issues y pull
requests allí, no aquí.

## Inicio rápido

La fixture `fixtures/London.portable.mad` incluida en el repositorio es un informe respaldado por SQLite que
no necesita servidor de base de datos ni credenciales:

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

La exportación escribe `mad-output/report.html`. Los demás verbos imprimen en el registro del
trabajo: `inspect`, `schema`, `query` y `refresh`.

## Una base de datos en el flujo de trabajo

Un `.mad` almacena su definición de conexión —driver, host, puerto y nombre de la base de
datos—, pero nunca un usuario ni una contraseña. Pase las credenciales desde secretos y
sobrescriba la conexión almacenada donde el servidor de CI difiera:

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

`host.docker.internal` es el bucle local del runner tal como lo ve el contenedor de la
acción; la acción añade esa asignación de host por usted. El documento sigue necesitando
el esquema y la consulta que coincidan con la base de datos contra la que se ejecuta: la fixture de MySQL
del repositorio MAD (`CLI/Fixtures/MySQL.employees.mad`) es un ejemplo resuelto, y
`MADMySQL/TestDatabase/mysql-test-db.sh` siembra la muestra `employees`
que consulta.

Un ejemplo de PostgreSQL:

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

La fixture de PostgreSQL del repositorio MAD
(`CLI/Fixtures/Postgres.chinook.mad`) es un ejemplo resuelto, y
`MADPostgres/TestDatabase/postgres-test-db.sh` siembra la muestra `chinook` que
consulta.

## Credenciales

El formato `.mad` mantiene deliberadamente las credenciales de la base de datos de origen fuera
del documento: la aplicación las guarda en el llavero de iOS, y el almacén solo contiene
la definición de la conexión. En CI, el flujo de trabajo las proporciona desde
`${{ secrets.* }}`; el documento permanece limpio y se puede incluir en el repositorio.
[El capítulo 5 del formato](https://github.com/mthomason/MAD/blob/master/docs/05-companion-files.md)
explica la separación entre archivos complementarios y llavero. Nunca ponga una contraseña en un
archivo de flujo de trabajo ni en un `.mad`; use un secreto de repositorio o de entorno.

`host`, `port` y `database` se aplican a un documento respaldado por servidor. Un
documento respaldado por SQLite no tiene conexión de servidor y los rechaza, así que déjelos
sin definir para la fixture incluida en el repositorio.

## Entradas

| Entrada | Obligatoria | Valor predeterminado | Significado |
|---|---|---|---|
| `command` | sí | — | `inspect`, `schema`, `query`, `refresh` o `export`. |
| `document` | sí | — | Ruta relativa al espacio de trabajo del `.mad`. |
| `format` | no | `html` | `html`, `html-flow`, `text`, `csv`, `json`, `json-report` o `pdf`. La imagen de Linux rechaza PDF. |
| `out-dir` | no | `mad-output` | Directorio relativo al espacio de trabajo para `export`; el archivo es `report.<ext>`. |
| `username` | no | — | Nombre de usuario de la base de datos de origen. Pase un secreto. |
| `password` | no | — | Contraseña de la base de datos de origen. Pase un secreto. |
| `host` | no | host almacenado | Sobrescritura del host de la base de datos de origen. |
| `port` | no | puerto almacenado | Sobrescritura del puerto de la base de datos de origen. |
| `database` | no | nombre almacenado | Sobrescritura del nombre de la base de datos de origen. |
| `rows` | no | todas | Detenerse después de este número de filas (`refresh`). |
| `image` | no | `ghcr.io/mthomason/mad:latest` | Avanzado: fija una compilación del motor. |

`@v1` es la etiqueta mayor de la acción, que se va moviendo, y `:latest` la del motor; una ejecución
usa el motor más reciente. Fije `image:` a una etiqueta de SHA corto para un motor
reproducible, y fije la acción a un commit si necesita que el envoltorio también quede congelado.

## Licencia

MIT; consulte [LICENSE](LICENSE). Los nombres de MAD y el arte de las aplicaciones están reservados en
[NOTICE](NOTICE) y no forman parte de la concesión MIT.
