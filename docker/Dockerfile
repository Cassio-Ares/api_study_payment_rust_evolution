FROM rust:latest AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bullseye-slim
WORKDIR /app

COPY --from=builder /app/target/release/api_study_payment_rust_evolution .

CMD ["./api_study_payment_rust_evolution"]