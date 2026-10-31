[English](README.md) | [简体中文](README_zh.md) | [日本語](README_ja.md) | **Português (Brasil)**

# MAD Action

Execute um relatório `.mad` no GitHub Actions com a engine headless do MAD. Um relatório
versionado em um repositório passa a fazer parte do seu pipeline: atualize os dados contra um
banco de dados, renderize o relatório e envie a exportação como um artefato de build.

Este repositório é gerado a partir de `satellites/mad-action/` em
[mthomason/MAD](https://github.com/mthomason/MAD). Abra issues e pull
requests lá, não aqui.

## Início rápido

A fixture incluída `fixtures/London.portable.mad` é um relatório baseado em SQLite que
não precisa de servidor de banco de dados nem de credenciais:

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

A exportação grava `mad-output/report.html`. Os demais comandos imprimem diretamente no log
do job: `inspect`, `schema`, `query` e `refresh`.

## Um banco de dados no workflow

Um arquivo `.mad` armazena sua definição de conexão — driver, host, porta e nome do
banco de dados —, mas nunca um nome de usuário ou senha. Passe as credenciais a partir de secrets e
sobrescreva a conexão armazenada quando o servidor de CI for diferente:

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

`host.docker.internal` é o endereço de loopback do runner visto de dentro do contêiner da
Action; a própria Action adiciona esse mapeamento de host automaticamente. O documento ainda precisa ter
o esquema e a consulta correspondentes ao banco de dados contra o qual será executado — a fixture de
MySQL no repositório do MAD (`CLI/Fixtures/MySQL.employees.mad`) é um exemplo completo e funcional,
e `MADMySQL/TestDatabase/mysql-test-db.sh` popula o banco de exemplo `employees`
consultado por ela.

Um exemplo com PostgreSQL:

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

A fixture de PostgreSQL no repositório do MAD
(`CLI/Fixtures/Postgres.chinook.mad`) é um exemplo funcional completo, e
`MADPostgres/TestDatabase/postgres-test-db.sh` popula o banco de exemplo `chinook` que ela
consulta.

## Credenciais

O formato `.mad` mantém deliberadamente as credenciais do banco de dados de origem fora do
documento: o aplicativo as armazena nas Chaves (Keychain) do iOS/macOS, e o store guarda apenas
a definição da conexão. No CI, o workflow fornece as credenciais a partir de
`${{ secrets.* }}`; o documento permanece limpo e pode ser versionado com segurança.
O [Capítulo 5 da especificação do formato](https://github.com/mthomason/MAD/blob/master/docs/05-companion-files.md)
explica a separação entre arquivos complementares e o Keychain. Nunca coloque uma senha em um
arquivo de workflow ou em um `.mad`; utilize sempre um secret de repositório ou de ambiente.

`host`, `port` e `database` aplicam-se a documentos conectados a servidores de banco de dados. Um
documento baseado em SQLite não possui conexão de servidor e recusa esses parâmetros, portanto deixe-os
sem definição ao usar a fixture incluída.

## Parâmetros de entrada (Inputs)

| Input | Obrigatório | Padrão | Significado |
|---|---|---|---|
| `command` | sim | — | `inspect`, `schema`, `query`, `refresh` ou `export`. |
| `document` | sim | — | Caminho relativo ao workspace para o arquivo `.mad`. |
| `format` | não | `html` | `html`, `html-flow`, `text`, `csv`, `json`, `json-report` ou `pdf`. PDF é recusado pela imagem Linux. |
| `out-dir` | não | `mad-output` | Diretório relativo ao workspace para `export`; o arquivo gerado é `report.<ext>`. |
| `username` | não | — | Usuário do banco de dados de origem. Passe via secret. |
| `password` | não | — | Senha do banco de dados de origem. Passe via secret. |
| `host` | não | host salvo | Sobrescreve o host do banco de dados de origem. |
| `port` | não | porta salva | Sobrescreve a porta do banco de dados de origem. |
| `database` | não | nome salvo | Sobrescreve o nome do banco de dados de origem. |
| `rows` | não | todas | Interrompe a busca após este número de linhas (`refresh`). |
| `image` | não | `ghcr.io/mthomason/mad:latest` | Avançado: fixa um build específico da engine. |

`@v1` é a tag principal móvel da Action e `:latest` é a da engine; por padrão, uma execução
utiliza a engine mais recente. Fixe `image:` em uma tag de SHA curto para obter uma engine estritamente
reproduzível, e fixe a Action em um commit específico caso também precise congelar o wrapper.

## Licença

MIT; consulte [LICENSE](LICENSE). Os nomes MAD e a arte visual dos aplicativos são reservados nos termos do
[NOTICE](NOTICE) e não fazem parte da concessão da licença MIT.
