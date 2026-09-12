# taskapp

A task manager: Spring Boot (Java 21) + MySQL backend, React frontend, JWT auth.

## Run it

```bash
cp .env.example .env
# set MYSQL_ROOT_PASSWORD and JWT_SECRET in .env
docker compose up -d --build
```

Then open `http://localhost`.

See **[README.docker.md](./README.docker.md)** for what was audited/fixed
to get this running cleanly as a single compose file.
