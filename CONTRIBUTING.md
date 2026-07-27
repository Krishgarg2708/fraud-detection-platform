# Contributing

This is primarily a personal portfolio project, but issues and PRs are welcome.

## Local development

Each service under `services/` is independently runnable:

```bash
# Java services (auth, transaction, rules-engine, risk-scoring, alert, notification)
cd services/<service-name>
mvn spring-boot:run

# Python ML service
cd services/ml-inference-service
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Or bring up everything together:

```bash
docker compose up --build
```

## Running tests

```bash
# any Java service
cd services/<service-name> && mvn test

# ML service
cd services/ml-inference-service && pytest tests/ -v
```

## Code style

- Java: standard Spring Boot conventions, Lombok for boilerplate, constructor injection via `@RequiredArgsConstructor`.
- Python: type hints throughout, Pydantic for all request/response schemas.
- Keep services independently deployable — no shared mutable state between services outside of Kafka topics and their own database.

## Commit messages

Conventional, imperative mood: `feat: add impossible travel rule`, `fix: correct refresh token rotation`, `docs: update roadmap`.
