# Model e Validação 

# model 
Primeiro vamos compreender algumas lógicas que são importantes para construirmos os models:

- struct -> molde de dados

````
pub struct User {
  pub uuid: Uuid,
  pub name: String,
  pub emai: String, 
}
````
OU 
````
pub struct ClientName(pub String); // tuple struct com 1 campo
````

- impl -> adiciona comportamentos a este model

1- impl Nome_Da_Struct (ex: ClientName)
        └── "Estou definindo métodos que só existem em ClienteName
              Você mesmo está inventando esses métodos.

```
// COMPORTAMENTO: define o que a struct SABE FAZER
impl ClientName {
    pub fn new(value: String) -> Self {
        ClientName(value)
    }
    
    pub fn len(&self) -> usize {
        self.0.len() // self.0 = primeiro campo da tuple struct
    }
}
```

2- impl AlgumaTrait for NomeDaStruct (ex: impl From<String> for ClientName)
            └── "Estou cumprindo um CONTRATO que já existe no Rust/biblioteca"
                 O From, Display, etc. já existem você só esta dizendo 
                 como sua struct se comporta dentro deste contrato

````
impl From<String> for ClientName {
     └── TRAIT             └──  TIPO que recebe a trait
     fn from(value: String) {
        ClientName(value)
     }
}
````


Quando você usa a struct (ex: ClientName), 
você tem acesso aos campos (dados) + todos os métodos definidos nos blocos "impl".

- for -> significa implemente um trait (contrato, caracteristica) para um tipo 

````
impl [O QUE VOCÊ QUER ENSINAR] for [QUEM VAI APRENDER]

impl fmt::Display for PlanType
//   ^^^^^^^^^^^^      ^^^^^^^^
//   trait Display      quem implementa

````

## Leganda do Model usado no projeto

````
#[derive(
    Debug,          // permite imprimir com {:?} no log/terminal
                    //   sem Debug: println!("{:?}", client) → ERRO
                    //   com Debug: println!("{:?}", client) → Client { id: ..., name: ... } 
    
    Clone,          // permite copiar o valor com .clone()
                    // sem clone: let b = a → move (a não existe mais)
                    // com clone: let b = a.clone() a e b existem e tem o valor

    PartialEq,      // permite comparar com == e !=
                    // sem PartialEq: if client_a == client_b → ERRO
                    // com PartialEq: if client_a == client_b → funciona  

    Eq,             // versão completa do PartialEq
                    //   PartialEq: "pode ser igual" (floats por ex não são sempre iguais)
                    //   Eq: "sempre tem resposta definida para igualdade"
                    //   use Eq quando todos os campos também são Eq

    Serialize,      // Rust -> JSON (para Enviar dados)
                    // Client { name: "João" } vira -> { "name": "João" }


    Deserialize,    // JSON -> Rust (para receber dados)
                    // { "name": "João" } vira -> Client { name: "João" }

            
    Validate,       // habilita .validate() na struct
                    // permite usar #[validate(...)] nos campos
                    // sem Validate: .validate() da erro de compilação

    sqlx::Type      // ensina o SQLx como ler/escrever este tipo no banco
                    // necessário para enums que existem no PostgreSQL
                    // ex: PlanType -> "mensal", "anual" no banco


    sqlx::FromRow,  // converte uma linha do banco → struct Rust
                    // sem: extração manual campo por campo
                    // com: SQLx converte automaticamente        
    
)]
````

````
use serde ::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{NaiveDateTime, Utc};
use validator::Validate;
use std::fmt;

// Encapsula a validação dentro do Tipo
// ClientName garante que nome sempre seja valido

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, Validate)]
pub struct ClientName(
  //validate no campo da tuple struct
  #[validate(length(
    min = 3,
    max = 100,
    message = "Client name must be between 3 and 100 characters"
  ))]
  pub String,  // o único campo — o nome em si
)

// From<String> -> permite converter String - ClientName automaticamente
// "João".to_string().into() -> ClientName("João")

impl From<String> for ClientName{
  fn from(value: String) -> Self{
    ClientName(value)
  }
}

// Display define como aparece quando impresso com {}

impl fmt::Display for ClientName {
  fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
    write!(f, "{}", self.0) // self.0 acessa o String interno
  }
}
````