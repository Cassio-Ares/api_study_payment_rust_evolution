# Erros

 - erro genérico sem significado 
````
) -> Result<Payment, Box<dyn std::error::Error>>
````

- erro personalizado com significado e HTTP code
````
// ApiError é a função personalizada para tratar erros
) -> Result<Payment, ApiError>
````

## error.rs

- Parte 1:
````
use actix_web::{http::StatusCode, HttpResponse, ResponseError};
use serde::Serialize;
use std::fmt;

// enum
// É como criar um "vocabulário de erros" para sua API — 
// cada situação tem um nome e um significado claro.

// #[derive(Debug)] -> para poder imprimir no log com {:?}
// #[derive(Serialize)] -> posso converter de Rust para JSON
#[derive(Debug, Serialize)]
pub enum ApiError {
  // Cada variante carrega uma String — a mensagem do erro
  NotFound(String),            // Recurso não encontrado 404
  InvalidInput(String),        // Dados inválidos do clinte 400
  InternalServerError(String), // Erro inesperado do servidor 500
  DatabaseError(String),       // erro específico do banco 500
  Unaunthorized(String),       // sem permissão 401
  Conflict(String)             // conflito de dados (ex: duplicado) 409
}
````

- Parte 2:
````
// impl fmt::Display
// Display -> define como o erro aparece como texto
// é o que o to_string() usa
// e o que aparece nos logs do servidor

impl fmt::Display for ApiError {
  // f -> o escritor de text
  // fmt::Result -> Ok se escreveu, Err se falhou
  fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
    match self {
                              // write!(f, ...) -> escreve no f; 
      ApiError::NotFound(msg) => write!(
                    // "{}" é substituido pela mensagem
        f, "Not Found: {}", msg
                     //|-> aparece no log com: "Not found: Payment not found"
      ),

      ApiError::InvalidInput(msg) => write!(f, "Invalid Input: {}", msg),

      ApiError::InternalServerError(msg) =>
                write!(f, "Internal Server Error: {}", msg),

      ApiError::DatabaseError(msg) =>
                write!(f, "Database Error: {}", msg),

      ApiError::Unauthorized(msg) =>
                write!(f, "Unauthorized: {}", msg),

      ApiError::Conflict(msg) =>
                write!(f, "Conflict: {}", msg),
    }
  }
}
````

- Na prática:

````
let err = ApiError::NotFound("Payment not found".to_string());
println!("{}", err); // → "Not Found: Payment not found"
log::error!("{}", err); // → aparece no log do servidor
````

- Parte 3:

````
// ResponseError -> trait do Actix que transforma ApiError em uma resposta HTTP
// É o tradutor de erro do rust

impl ResponseError for ApiError {
  fn error_response(&self) -> HttpResponse {
    match self {
      // HttpResponse: build(StatusCode) -> cria resposta com o status code
      // .json(msg) -> corpo de resposta em json

      ApiError::NotFound(msg) =>          
                HttpResponse::build(StatusCode::NOT_FOUND) // 404
                .json(msg) 

      ApiError::InvalidInput(msg) =>
              HttpResponse::build(StatusCode::BAD_REQUEST) // 400
               .json(msg),
      
      ApiError::InternalServerError(msg) =>
               HttpResponse::build(StatusCode::INTERNAL_SERVER_ERROR) // 500
                  .json(msg),

      ApiError::DatabaseError(msg) =>
               HttpResponse::build(StatusCode::INTERNAL_SERVER_ERROR) // 500
                  .json(msg),
      // DatabaseError e InternalServerError → ambos 500
      // mas separados para o LOG identificar a origem

      ApiError::Unauthorized(msg) =>
               HttpResponse::build(StatusCode::UNAUTHORIZED) // 401
                  .json(msg),

      ApiError::Conflict(msg) =>
              HttpResponse::build(StatusCode::CONFLICT) // 409
                    .json(msg),

    }
  }
}
````

### Na prática — o Actix chama isso automaticamente:

````
pub async fn get_payment_handler(...) -> Result<HttpResponse, ApiError> {
    let payment = service::get_payment(uuid).await?;
    // se retornar Err(ApiError::NotFound("..."))
    // o Actix chama error_response() automaticamente
    // cliente recebe: HTTP 404 { "Not Found: Payment not found" }
    Ok(HttpResponse::Ok().json(payment))
}
````


- Parte 4:
    (impl From<sqlx::Error> for ApiError -> converte erro sqlx em ApiError)


````
// From -> trait de conversão automatica 
// Como transformar um sqlx::Error em ApiError
// É o que permite o `?` funcionar em funções que retornam ApiError

impl From<sqlx::Error> for ApiiError{
  fn from(err: sqlx::Error)-> ApiError{
    // qualquer erro do SQLx vira DatabaseError
    ApiError::DatabaseError(err.to_string())
  }
}
````

- Pq do `FROM`?

Sem From (manual -> .map_err(|e| ApiError::DatabaseError(e.to_string()))?)

````
let payment = sqlx::query_as!(Payment, "SELECT...")
    .fetch_one(&pool)
    .await
    .map_err(|e| ApiError::DatabaseError(e.to_string()))?;
````

- Com From (automático)

````
let payment = sqlx::query_as!(Payment, "SELECT ...")
               .fetch_one(&pool)
               .await?  // <- o ? converte sqlx::Error → ApiError automaticamente!
````

- Parte 5:
    (impl From<validator::ValidationErrors> for ApiError  -> mesma ideia do anterior converte erros do validator em ApiError )

````
impl From<validator::ValidationErrors> for ApiError {
  fn from(err: validator::ValidationErrors) -> ApiError {
    // serde_json::to_string(&err) -> converte o erro para json string
    // .unwrap_or_else(|_| "Validation error".to_string()) -> se a conversão falhar 
    // use a  mensagem genérica "Validation error"

    ApiError::InvalidInput(
            serde_json::to_string(&err)
                .unwrap_or_else(|_| "Validation error".to_string())
        )
  }
}
````

- Na prática:

```
new_payment.validate()?;
```

`validate()` -> retorna Err(ValidationErrors)
 `?` -> converte ValidationErrors, ApiError::InvalidInput automaticamente
  cliente recebe: HTTP 400 com os detalhes dos erros

  Error return:
   Resposta JSON não padronizada
   Hoje retorna só a string da mensagem:
   HttpResponse::build(StatusCode::NOT_FOUND).json(msg)
   
- cliente recebe: "Payment not found"

  OBS: 
  Os passos 4 e 5 são formas de personalizar e automatizar cada vez mais o erro tanto, com impl "alguma propriedade (ex: From<sqlx::Error>, From<validator::ValidationErrors> ) for ApiError usando o valor 
  especifico (ex: ApiError::InvalidInput, ApiError::DatabaseError) com impl assim podemos melhorar e personalizar os erros das regras de negócio.


> new_payment.validate()?; ->  
    cliente recebe: "Payment not found"


## ## Erro com resposta mas completa 

- {
    "error": "NOT_FOUND",
    "code": "resource_not_found",
    "message": "Payment not found"
  }