# Repository

Neste projeto criamos em repository

client_repository.rs -> O contrato
- O que repository deve ter e fazer mas não sabe como fazer

ex:
````
async fn create(&self, new_client:CreateClientDTO) -> Result<Client, ApiError>
````

sqlx_client_repository.rs -> A implementação dos contratos

- Como fazer usando SQLx + PostgreSQL 

ex: 
````
pub async create(pool: $self, new_client: CreateClientDto) -> Result<Client, ApiError>{
  let client = query_as::<Postgres, Client>(
    "INSERT INTO ..."
  )

  Ok(client)
}
````

- Porque da separação se o codigo fica mais verboso? 

O contrato existe pra quando o projeto cresce. É uma decisão de arquitetura, não de funcionamento. O código sem o contrato roda igual — ele só fica mais difícil de testar, de trocar, e de manter com o tempo.
É o tipo de coisa que parece burocracia no início e você agradece 6 meses depois.

Então a resposta honesta é:
Se você nunca vai testar, nunca vai trocar de banco, e trabalha sozinho, pode usar direto e não perde quase nada.

- Porque do Repository se posso fazer tudo no Service ou tudo no Handler?

ex:
````
pub struct ClientService {
    pool: PgPool, // ← o service CONHECE o banco
}

impl ClientService {
    pub async fn create_client(&self, dto: CreateClientDto) -> Result<Client, ApiError> {
       // validações
       // regras de negocio
       
        sqlx::query_as("INSERT INTO clients...") // ← o service FALA com o banco 
            .fetch_one(&self.pool) diretamente
            .await
    }
}
````

O service virou duas coisas ao mesmo tempo: lógica de negócio + acesso a dados. Isso viola o Princípio da Responsabilidade Única.

Com separação - desacoplado:
O service não sabe qual banco usa ele só conhece o contrato

ex:
````
pub struct ClientService{
  repository: Arc<dyn ClientRepository> // contrato
}
// Para trocar de banco → cria novo arquivo de implementação
// o service, handler, routes → não mudam nada

impl ClientService {
    pub fn new(repository: Arc<dyn ClientRepository>) -> Self {
        ClientService { repository }
    }

    pub async fn create_client(&self, new_client: CreateClientDto) -> Result<Client, ApiError> {
        // A validação de entrada já ocorre no DTO (CreateClientDto) via #[validate]
        // e no Newtype Pattern. Se chegou aqui, os dados básicos são válidos.
        self.repository.create(new_client).await
    }
  }
````

### Contrato

obs: 

#[async_trait] 
// O Rust tem uma limitação:

// traits NÃO suportam async fn nativamente

Você escreve:

````
#[async_trait]
pub trait ClientRepository {
    async fn create(&self, ...) -> Result<Client, ApiError>;
}

// A macro transforma em:
pub trait ClientRepository {
    fn create(&self, ...) -> Pin<Box<dyn Future<Output = Result<Client, ApiError>> + Send>>;
    // Future = o tipo que representa uma operação assíncrona
    // Pin<Box<...>> = necessário para o Rust gerenciar o Future na memória
}
// Você não precisa escrever isso — a macro faz por você
````

````
use async_trait::async_trait;
use uuid::Uuid;

use crate::model::client_model::{
    Client, CreateClientDto, UpdateClientDto
}

use crate::errors::ApiErros;

// #[async_trait] -> macro que permite async fn neste trait
// pub trait -> contrato público que qualquer struct pode implementar
// Send + Sync -> garante que pode ser usado entre threads com segurança
// Send  = pode ser ENVIADO para outra thread
// Sync  = pode ser ACESSADO por múltiplas threads ao mesmo tempo
// necessário porque o Actix usa múltiplas threads

#[async_trait]
pub trait ClientRepository: 
````

