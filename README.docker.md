# Running taskapp with Docker

This project already had a working `docker-compose.yaml` and Dockerfiles for
both services. Most of the wiring was correct — the audit below found a few
real issues worth fixing rather than a from-scratch setup.

## 1. Run it

```bash
cp .env.example .env
nano .env   # set MYSQL_ROOT_PASSWORD and JWT_SECRET
docker compose up -d --build
```

Then open `http://localhost` — nginx serves the built React app and proxies
`/api/*` to the Spring Boot backend.

## 2. What was checked and found correct

Worth noting, since it's easy to assume a project needs more fixing than it
does: the frontend↔backend↔nginx routing was already wired correctly end to
end — `frontend/.env.production` sets `REACT_APP_API_URL=/api` (a relative
path), `nginx.conf` proxies `/api` to `http://backend:3030/api`, and the
Spring controllers are mapped at `/api/auth` and `/api/tasks` — all
consistent. CORS, the JWT bearer-token flow, and the MySQL datasource env
vars (`SPRING_DATASOURCE_URL`/`_USERNAME`/`_PASSWORD`) were also already
correctly wired through docker-compose. Dependencies (Spring Boot 3.4.5,
Java 21, React 19, react-router 7) are all current — nothing needed
upgrading here.

## 3. What was found and fixed

### The JWT_SECRET environment variable was silently ignored

`docker-compose.yaml` passes a `JWT_SECRET` environment variable to the
backend, but `JwtUtils.java` reads a property called `secreteJwtString` —
and Spring's environment-variable-to-property matching can't bridge those
two names (they don't share the same word content, so no amount of
relaxed-binding makes `JWT_SECRET` resolve to `secreteJwtString`). The
practical effect: **no matter what you set `JWT_SECRET` to, the app always
signed tokens with the literal hardcoded string in
`application.properties`.** Fixed by changing that property to
`secreteJwtString=${JWT_SECRET:dev-only-insecure-default...}`, so it now
actually reads the env var, with a fallback for running the jar directly
outside Docker.

### Three endpoints didn't check task ownership (IDOR)

`getTaskById`, `updateTask`, and `deleteTask` in `TasksServiceImpl` looked
up a task by ID and returned/modified/deleted it — without ever checking
that the task belonged to the logged-in user. Any authenticated user could
view, edit, or delete **any other user's task** just by knowing or
guessing its numeric ID. The other three task methods
(`getAllMyTasks`, `getMyTasksByCompletionStatus`, `getMyTasksByPriority`)
already correctly scope by the current user, so this was clearly meant to
be enforced everywhere and just wasn't. Fixed by checking
`task.getUser().getId().equals(currentUser.getId())` in all three,
throwing the same `NotFoundException` used elsewhere (returning 404 rather
than 403 — standard practice, so an attacker probing IDs can't tell the
difference between "doesn't exist" and "not yours").

### Real secrets were committed to the repo

- `application.properties` had a real-looking personal MySQL password
  (`anambra247`) and, in a commented-out AWS RDS connection block, a
  second real-looking password. Neither is actually used when running via
  Docker (the compose file's `SPRING_DATASOURCE_*` env vars already
  correctly override them), but they were still sitting in the file.
  Replaced with `${SPRING_DATASOURCE_*:...}` placeholders and a generic
  `changeme` default.
- **`backend/target/` — a compiled Maven build directory — was checked
  into the zip**, and it carried its own stale copy of
  `application.properties` with all of the above secrets baked in
  (`.gitignore` listed `*.class` and `*.jar` but not `target/` itself, so
  the non-class files inside it, like the properties copy, weren't
  excluded). Deleted the directory and added `target/` to `.gitignore`.

If any of these credentials were ever real and in use elsewhere, rotate
them — they were exposed in this repo.

## 4. Orchestration improvement

`backend` had no `HEALTHCHECK`, so `frontend`'s `depends_on: backend` only
waited for the container to *start*, not for Spring Boot to actually finish
booting — meaning the very first requests through nginx after a fresh
`docker compose up` could hit a backend that wasn't listening yet. Added a
TCP-level healthcheck (this app doesn't have `spring-boot-starter-actuator`,
so there's no `/actuator/health` to hit) and changed `frontend`'s
dependency to `condition: service_healthy`.

## 5. Known limitations

- No TLS/reverse proxy beyond the frontend's own nginx — fine for local
  use, not for a public deployment as-is.
- `mysql`'s port (3306) and the backend's port (3030) are both published
  to the host for convenience/debugging; for a genuinely public deployment
  you'd likely keep those internal-only.
- This backend has no `spring-boot-starter-actuator`, so the healthcheck
  above is a blunt TCP check rather than an actual readiness probe (e.g. DB
  connectivity). Adding actuator would be a reasonable follow-up if you
  want a real `/actuator/health` endpoint.
