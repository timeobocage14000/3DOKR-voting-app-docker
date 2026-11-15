# Voting app with Docker

A voting app (Cats vs Dogs) running in containers with Docker Compose.
Each module has its own Dockerfile and everything starts with a single command.

GitHub repository: https://github.com/timeobocage14000/3DOKR-voting-app-docker

## Architecture

- `vote`: web page to vote (Python / Flask), port 8080
- `result`: web page showing the results (Node.js), port 8081
- `worker`: takes the votes from Redis and writes them to PostgreSQL (.NET)
- `redis`: queue for the votes
- `db`: PostgreSQL database

There are three networks:

- `frontend`: `vote` and `result` (the only published ports)
- `redis_net` (internal): `vote`, `redis`, `worker`
- `db_net` (internal): `worker`, `db`, `result`

This way `vote` cannot reach the database, and `result` cannot reach Redis.

## Requirements

Docker with Docker Compose v2.

## Running the project

Create the database password (the file is not versioned):

```bash
mkdir -p secrets
openssl rand -base64 24 | tr -d '\n' > secrets/db_password.txt
chmod 644 secrets/db_password.txt
```

Then start everything:

```bash
docker compose up -d --build --wait
```

- Vote: http://localhost:8080
- Results: http://localhost:8081

To stop: `docker compose down` (the votes are kept) or `docker compose down -v` (also deletes the votes).

Note: PostgreSQL only reads the password the first time the volume is created. If `secrets/db_password.txt` is changed afterwards, run `docker compose down -v` before starting again.

## Technical choices

- Official images with the full version tag and the Alpine variant (lighter)
- Multi-stage builds for `vote`, `result` and `worker`: dependencies or compilation are handled in the first stage
- Non-root user in each image with a numeric UID (vote 10001, result 1000, worker 1654), `USER` placed before the `COPY` instructions, which use `--chown`
- One `.dockerignore` per module
- `CMD` and `ENTRYPOINT` in exec form
- One healthcheck per service, and `depends_on` with `service_healthy`
- Named volumes `db_data` and `redis_data`
- The database password is a Docker secret (`POSTGRES_PASSWORD_FILE`), it is not in the code
- `restart: unless-stopped` and CPU and memory limits for every service

## Versions

The original versions are no longer maintained, so I changed them (checked on endoflife.date on 11/11/2026):

| Component | Before | Now | Why |
|-----------|--------|-----|-----|
| Python | 3.11 | 3.14.10 | supported until 2030 |
| Node.js | 18 | 26.13.0 | LTS version |
| .NET | 7 | 10.0 | LTS version |
| PostgreSQL | latest | 18.7 | pinned version |
| Redis | latest | 8.10.3 | pinned version |

The NuGet packages and the Python dependencies are also up to date and pinned.
To update: change the tag in the Dockerfile or in `compose.yaml`, then run `docker compose build --pull` and `docker compose up -d --wait`.

## Code changes

- `vote/app.py`: the Redis address comes from `REDIS_HOST` and debug mode is disabled
- `result/server.js`: the PostgreSQL connection comes from environment variables, the password is read from a file
- `result/package-lock.json`: dependencies updated with `npm update` (security fixes)
- `worker/Program.cs`: same for PostgreSQL and Redis, and GSS encryption disabled (the library is not in Alpine)
- `worker/Worker.csproj`: `net7.0` becomes `net10.0`, packages updated
- `run/*.bash`: removed, replaced by `docker compose`

## Tests done

- the 5 containers are `healthy` after `docker compose up -d --build --wait`
- a vote reaches PostgreSQL and the results page responds
- the votes are still there after `docker compose down` then `up`
- from `vote` the `db` host cannot be reached, from `result` the `redis` host cannot be reached
