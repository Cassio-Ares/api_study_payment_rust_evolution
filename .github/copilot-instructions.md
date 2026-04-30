# Project Guidelines

## Code Style

Use Rust 2021 edition. Avoid experimental or invalid editions like 2024.

## Architecture

Study project for payment API using Actix-web framework and SQLX for database interactions.

## Build and Test

- Build: `cargo build --release`
- Test: `cargo test`
- Docker: `docker build . -t api-payment`

## Conventions

- Documentation in Portuguese
- See [1-START_DOCKER.md](1-START_DOCKER.md) for Docker setup
- Fix Dockerfile issues: use "release" instead of "realease", proper JSON syntax for CMD
