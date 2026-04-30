# Inicio do projeto

- crio o Dockerfile

````
FROM rust:latest AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bullseye-slim
WORKDIR /app

COPY --from=builder /app/target/release/api_study_payment_rust_evolution .

CMD ["./api_study_payment_rust_evolution"]
````

## Rodar o cargo init se o rust na maquina 

- docker run --rm -v $(pwd):/app -w /app rust:latest cargo init


### Cargo.toml inicial
````
[package]
name = "api_study_payment_rust_evolution"
version = "0.1.0"
edition = "2024"

[dependencies]
actix-web = "4
````

