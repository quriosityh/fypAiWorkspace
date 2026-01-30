```
apps/api/src/
  main/               # app bootstrap + server lifecycle
    app.ts
    server.ts
  config/             # env, cors, rate limit, helmet, logging config
  common/             # shared utilities (errors, logger, http responses)
  infra/
    db/
      client.ts       # drizzle client + connection
      schema.ts       # tables
      migrations/
    http/
      middleware/
        auth.ts
        logger.ts
        errors.ts
      validators/     # shared zod helpers
  modules/
    health/
      routes.ts
      controller.ts
    auth/             # Clerk integration helpers
      service.ts
      routes.ts (if needed)
    users/
      routes.ts
      controller.ts
      service.ts
      repository.ts
      dto.ts
      validations.ts
    listings/
      routes.ts
      controller.ts
      service.ts
      repository.ts
      dto.ts
      validations.ts
    categories/
      routes.ts
      controller.ts
      service.ts
      repository.ts
    uploads/
      routes.ts
      controller.ts
      service.ts
    bookings/
      routes.ts
      controller.ts
      service.ts
      repository.ts
    messages/
      routes.ts
      controller.ts
      service.ts
      socket.ts       # socket.io handlers (later)
    reviews/
      routes.ts
      controller.ts
      service.ts
      repository.ts
  tests/
    integration/
    unit/
    fixtures/
```

- Routes: wiring only; no logic.
- Controllers: translate HTTP ↔ service; no business rules.
- Services: business rules, validation, orchestration; no HTTP specifics.
- Repositories: data persistence; no business rules, no HTTP.
- Validations/DTOs: schemas/types close to the feature.
- Common: reusable, transport-agnostic types/helpers.
- Infra/http: transport-specific middleware (Express).
- Infra/db: DB client + health.