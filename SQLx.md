# SQLx 

SQLx é uma bilioteca que faz su Rust conversar com banco de dados 
- ele envia SQL e recebe os resultados já convertidos para suas structs.

## postgres::PgPoolOptions

- postgres      -> submódulo do SQLx específico para PostgresSQL
- PgPoolOptions -> configuração do pool de conexões
- Pool          -> conjunto de conexões abertas e prontas para uso

É como um estacionamento de conexões:

 ┌─────────────────────────────┐
 │     Pool (5 conexões)       │
 │  [conn1] [conn2] [conn3]    │
 │  [conn4] [conn5]            │
 └─────────────────────────────┘

Quando o handler precisa do banco ele:
passo 1- pega uma conexão 
passo 2 - usa a conexão
passo 3- devolve para pool


## PgPoolOptions

````
PgPoolOptions::new()
  .max_connections(5)     // máximo 5 conexões simultâneas
  .connect(&database_url) // conecta ao banco
  .await                  // aguarda a conexão
````

## Pool

Pool<DB> -> o pool em si depois de configurado 

Pool<Postgres> -> o pool conectado ao PostgreSQL

````
let pool:Pool<Postgres> = PgPoolOptions::new()
     .connect(&url)
     .await?
````

PgPoolOptions = o projeto da garagem

Pool = a garagem construída e funcionando

### OBS:

Postgres -> é o tipo que 