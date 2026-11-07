
# Session 6 — Student Lab Guide

## Lab 3 — Secure Container, Liveness/Readiness & Compose (5% of the course)

**Work in your squad. Every member builds and runs the image on their own machine.**

By the end of today your application ships as an image that someone else can run
without asking you anything, and you can prove three things about it: it does not
contain a compiler, it does not run as root, and it tells the truth about its own
health.

**What you hand in (Lab 3, 5% of the course):**

1. `Dockerfile` — multi-stage, non-root, cache-ordered
2. `.dockerignore`
3. `docker-compose.yml` with the API and the database, one command to start
4. `docs/adr-0001-base-image.md` — your base-image decision, with numbers
5. The output of `image-report.sh`, `verify-nonroot.sh` and `readiness-experiment.sh`

**Nothing counts unless you can reproduce it on demand, in front of the
instructor, with the command that produced it.**

There are **nine checkpoints**.

### Map of the files you will touch

All paths are relative to the root of your team repository.


| Step | File                                                                                      | Action                                                                            |
| ------ | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| 0    | `scripts/image-report.sh`, `scripts/verify-nonroot.sh`, `scripts/readiness-experiment.sh` | **Copy in** from the Session 6 course materials                                   |
| 2–3 | `Dockerfile.todo` → `Dockerfile`                                                         | **Rename** with `git mv`, then **replace** the contents                           |
| 4    | `.dockerignore`                                                                           | **Create**                                                                        |
| 4    | `src/DevOpsLab.Domain/Entities/Product.cs`                                                | **Temporary edit** for the cache measurement, then `git restore`                  |
| 4    | `Dockerfile`                                                                              | **Temporary edit** (wrong layer order) for the measurement, then `git restore`    |
| 5    | `Dockerfile.alpine`                                                                       | **Create** — the comparison image, never used by Compose                         |
| 5    | `docs/adr-0001-base-image.md`                                                             | **Create** — you fill it in during Steps 4–8                                    |
| 6    | `Dockerfile`                                                                              | **Temporary edit** (the write-permission trap), then `git restore`                |
| 7    | `docker-compose.yml`                                                                      | **Edit** — add the `api` service, a private network, remove the published `1433` |
| 7    | `docker-compose.todo.yml`                                                                 | **Delete** with `git rm` — its content is now in `docker-compose.yml`            |
| 7    | `.env`                                                                                    | **Edit** — add `API_PORT`. **Never committed**                                   |
| 7    | `.env.example`                                                                            | **Edit** — add `API_PORT=8080` so the next person knows it exists                |
| 8    | `src/DevOpsLab.WebApi/HealthChecks/HealthCheckExtensions.cs`                              | **Temporary edit** — the one-word defect — then `git restore`                   |

**Not committed, ever:** `.env`, the `trap-logs/` folder from Step 6, and any
change to `Product.cs` or `HealthCheckExtensions.cs`. All three temporary edits
are experiments; each one is undone in the step that makes it.

When you finish, the files you changed look like this:

```
.dockerignore                       (new)
.env.example                        (edited)
Dockerfile                          (renamed from Dockerfile.todo, rewritten)
Dockerfile.alpine                   (new)
docker-compose.yml                  (edited)
docker-compose.todo.yml             (deleted)
docs/
└── adr-0001-base-image.md          (new)
scripts/
├── image-report.sh                 (new)
├── readiness-experiment.sh         (new)
└── verify-nonroot.sh               (new)
```

> **Where to run the commands.** Every command is run from the **root of your
> repository** — the folder that contains `DevOpsLab.sln`. On Windows, run them in
> **Git Bash**, not PowerShell: the three scripts are bash scripts, and the
> `\` line continuations below are bash syntax.

---

## Checkpoint 0 — Before you start (10 min)

Run all five. If any fails, say so now, not in twenty minutes.

```bash
docker version                 # client AND server must answer
docker compose version
docker image ls | grep -E 'dotnet/(sdk|aspnet)|mssql'   # the pre-pulled images
docker system df               # images + volumes; you need several GB free
git status                     # clean tree, on main, up to date
```

The pre-pulled images you should see:

```
mcr.microsoft.com/dotnet/sdk               8.0
mcr.microsoft.com/dotnet/aspnet            8.0-noble-chiseled
mcr.microsoft.com/dotnet/aspnet            8.0-alpine
mcr.microsoft.com/mssql/server             2022-latest
```

If one is missing, pull it **now** — it is faster before thirty people need it:

```bash
docker pull mcr.microsoft.com/dotnet/sdk:8.0
```

Create the branch:

```bash
git switch main && git pull
git fetch upstream
git checkout main
git merge upstream/main
git switch -c feat/containerise-api
```

**Copy in the three scripts.** Your instructor gives you
`image-report.sh`, `verify-nonroot.sh` and `readiness-experiment.sh` from the
Session 6 course materials. Put them in the existing `scripts/` folder (next to
the Session 4 scripts), then:

```bash
chmod +x scripts/image-report.sh scripts/verify-nonroot.sh scripts/readiness-experiment.sh
git add scripts/
git update-index --chmod=+x scripts/image-report.sh scripts/verify-nonroot.sh scripts/readiness-experiment.sh
```

The last line records the executable bit in Git, so it survives a clone on
Windows.

have a `.env` from Session 1:

```bash
ls .env || cp .env.example .env
```

> **If `docker version` prints a client but no server**, Docker Desktop is not
> running, or virtualisation is off in your BIOS. Tell the instructor
> immediately and pair with a teammate. Do not spend the session on it.

> **Checkpoint 0** — all five commands answered, the four images are present,
> the three scripts are in `scripts/` and executable.

---

## Step 1 — Read what you are being asked for (5 min)

Open `Dockerfile.todo`. It is not a hint sheet; it is the acceptance criteria for
Lab 3. Five requirements, and you will be asked about all five:

1. multi-stage build
2. a base image you **chose and justified**
3. a non-root user, **proved**
4. layer order that survives a source-only change
5. no SDK, no tests, no `.git`, no secret in the final image

> **One correction to that file.** It suggests proving non-root with
> `docker run --rm <image> id`. That does not work on the image you are about to
> build, and the reason is in Step 6. Keep reading.

---

## Step 2 — The build stage (10 min)

**File:** `Dockerfile.todo` → **rename** to `Dockerfile`, then **replace**
everything in it with the build stage below.

```bash
git mv Dockerfile.todo Dockerfile
```

Type it — do not paste it. Each line is something you will be asked to explain.

```dockerfile
# syntax=docker/dockerfile:1

# --- Stage 1: build ----------------------------------------------------------
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# 1. Project files first. This layer changes only when a dependency changes.
COPY DevOpsLab.sln ./
COPY src/DevOpsLab.Domain/DevOpsLab.Domain.csproj                 src/DevOpsLab.Domain/
COPY src/DevOpsLab.Application/DevOpsLab.Application.csproj       src/DevOpsLab.Application/
COPY src/DevOpsLab.Infrastructure/DevOpsLab.Infrastructure.csproj src/DevOpsLab.Infrastructure/
COPY src/DevOpsLab.WebApi/DevOpsLab.WebApi.csproj                 src/DevOpsLab.WebApi/

# 2. Restore only the web project's graph. Test projects never enter the image.
RUN dotnet restore src/DevOpsLab.WebApi/DevOpsLab.WebApi.csproj

# 3. Now the source. A one-character change here invalidates from this line down.
COPY src/ src/

# 4. Publish. --no-restore asserts that step 2 really did the work.
RUN dotnet publish src/DevOpsLab.WebApi/DevOpsLab.WebApi.csproj \
        --configuration Release \
        --no-restore \
        --output /app/publish \
        /p:UseAppHost=false
```


| Line                                        | Why it is there                                                                                                                             |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `# syntax=docker/dockerfile:1`              | Uses the current Dockerfile syntax (BuildKit). Must be the **first** line                                                                   |
| `FROM … sdk:8.0 AS build`                  | The SDK has the compiler. `AS build` names the stage so the next stage can copy from it                                                     |
| `COPY *.csproj` before `COPY src/`          | The `.csproj` files change rarely; the source changes on every commit. Slow steps sit **below** rarely-changing lines                       |
| `dotnet restore` on the WebApi project only | Restores exactly what the API needs. The test projects are not copied, so they cannot be restored or shipped                                |
| `--no-restore` on publish                   | Not an optimisation — an **assertion**. If the restore layer did not really work, the build fails here instead of silently restoring again |
| `/p:UseAppHost=false`                       | Do not produce a native launcher; the runtime starts the app with `dotnet DevOpsLab.WebApi.dll`                                             |

> **Do not copy the test projects.** `tests/` is not mentioned anywhere in the
> Dockerfile. CI runs the tests (Session 5); the image only ships the API.

---

## Step 3 — The runtime stage (10 min)

**File:** `Dockerfile` — **append** the runtime stage below the build stage.

```dockerfile
# --- Stage 2: runtime --------------------------------------------------------
# noble-chiseled: Ubuntu 24.04, no shell, no package manager, non-root by default.
FROM mcr.microsoft.com/dotnet/aspnet:8.0-noble-chiseled AS runtime
WORKDIR /app

# 8080, not 80: a non-root process cannot bind a port below 1024.
ENV ASPNETCORE_HTTP_PORTS=8080
EXPOSE 8080

# Only the published output crosses from the build stage.
COPY --from=build /app/publish .

# The chiseled image already defaults to `app` (UID 1654). Say it anyway, so the
# intent survives a change of base image.
USER app

ENTRYPOINT ["dotnet", "DevOpsLab.WebApi.dll"]

# No HEALTHCHECK on purpose: it would run INSIDE the container, and this image
# has no shell and no curl. See docs/adr-0001-base-image.md.
```


| Line                                            | Why it is there                                                                                                                                     |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `FROM … aspnet:8.0-noble-chiseled`             | Runtime only — no SDK, no compiler. Chiseled also has no shell and no package manager                                                              |
| `ASPNETCORE_HTTP_PORTS=8080`                    | A non-root process cannot bind port 80. .NET 8 containers listen on 8080 for exactly this reason. Moving back to 80 quietly undoes your `USER` line |
| `EXPOSE 8080`                                   | **Documentation only.** It publishes nothing. `-p 8080:8080` or `ports:` in Compose publishes                                                       |
| `COPY --from=build /app/publish .`              | The **only** thing that crosses from stage 1. The SDK, the source and `obj/` stay behind                                                            |
| `USER app`                                      | Run as the non-root user that the .NET 8 images define                                                                                              |
| `ENTRYPOINT [...]` (exec form, square brackets) | The process is `dotnet` itself, not a shell — there is no shell to start it with                                                                   |

Your complete `Dockerfile` is now the build stage from Step 2 followed by this
runtime stage — two `FROM` lines in total.

---

## Step 4 — `.dockerignore`, build, run, measure (20 min)

### 4.1 Create `.dockerignore`

**File:** `.dockerignore` — **create** it in the repository root, next to the
`Dockerfile`.

`docker build .` first **uploads the whole directory** to the Docker daemon — the
*build context*. Everything in it can end up in the image, including things you
would never push to GitHub. `.dockerignore` is the boundary of what you hand over.

```
# Build output and IDE state
**/bin/
**/obj/
**/out/
**/TestResults/
**/coverage*/
.git/
.github/
.vs/
.vscode/
.idea/
**/*.user
**/*.suo

# Secrets and local configuration
**/.env
**/.env.*
**/appsettings.*.Local.json
**/*.pfx
**/*.p12
**/*.key
**/secrets.json

# Not needed to run the service
docs/
**/*.md
!README.md
docker-compose*.yml
nginx/
scripts/
```

> `.vs/` matters more than it looks: Visual Studio keeps large index databases in
> it, and without this line they are uploaded on every build.

### 4.2 Build

```bash
docker build -t devopslab-api:local .
```

**Watch the context line near the top of the output:**

> `transferring context: 4.21MB` — good.
> `transferring context: 312MB` — your `.dockerignore` is missing, misnamed, or not
> in the same folder as the `Dockerfile`. Fix it before you continue.

Write the context size down — it goes in the ADR.

### 4.3 Run

```bash
docker run --rm -d --name api-test -p 8080:8080 devopslab-api:local
docker logs api-test                      # JSON log lines, no exception
curl -i http://localhost:8080/healthz/live
curl -i http://localhost:8080/healthz/ready
```

> **Checkpoint 1** — `/healthz/live` answers `200`. `/healthz/ready` answers
> `503`, and that is **correct**: inside this container, `localhost:1433` is the
> container itself, and there is no database there. Leave it; Step 7 fixes it and
> Step 8 explains why the distinction matters.

Keep `api-test` running — Steps 5 and 6 use it.


```git add .dockerignore Dockerfile
git add .dockerignore Dockerfile
git commit -m"feat: add Dockerfile for multi-stage build and create .dockerignore"
```

### 4.4 Measure the cache (10 min)

You are going to time a rebuild after a **one-line source change**, with the layer
order right and then wrong. Four builds, two numbers.

> **Do not use `touch`.** Docker decides whether a `COPY` layer can be reused from
> the **content** of the files, not their modification time. `touch` changes
> nothing Docker looks at, so the build is fully cached and the number means
> nothing. Change the content.

**A — correct order.**

```bash
docker build -t devopslab-api:local .                       # warm the cache
echo "// cache test" >> src/DevOpsLab.Domain/Entities/Product.cs
time docker build -t devopslab-api:local .                  # ← number A
git restore src/DevOpsLab.Domain/Entities/Product.cs
```

Look at the build output for number A: the `dotnet restore` step should say
`CACHED`; the `COPY src/ src/` step and the publish step run again.

**B — wrong order.** **Temporarily edit** `Dockerfile`: move the `COPY src/ src/`
line **above** the `RUN dotnet restore` line.

Dockerfile

```Shell
# 3. Now the source. A one-character change here invalidates from this line down.
COPY src/ src/

# 2. Restore only the web project's graph. Test projects never enter the image.
RUN dotnet restore src/DevOpsLab.WebApi/DevOpsLab.WebApi.csproj
```

```Then:

```bash
docker build -t devopslab-api:wrong-order .                 # warm the cache
echo "// cache test" >> src/DevOpsLab.Domain/Entities/Product.cs
time docker build -t devopslab-api:wrong-order .            # ← number B
git restore src/DevOpsLab.Domain/Entities/Product.cs Dockerfile
docker image rm devopslab-api:wrong-order
```
This time `dotnet restore` runs again. Write both numbers (the `real` line) in the
ADR. If the difference on your machine is small, say so — a number you measured
beats a claim you copied.

> **Checkpoint 2** — two build times recorded, `git status` shows no change to
> `Product.cs`, and the `Dockerfile` is back in the right order.

---

## Step 5 — The image you chose (Block D — follow along, 25 min)

### 5.1 Try to get a shell

Try the thing every developer tries first:

```bash
docker exec -it api-test sh
```
It fails — something like `exec: "sh": executable file not found in $PATH`. The
chiseled image has no shell, no package manager, no `curl`, no `id`.

**This is the trade-off you accepted in Step 3 without reading it.**

### 5.2 Diagnose without a shell

Run each of these against `api-test` and note what each one tells you:

```bash
docker logs --tail 50 api-test                               # what did it say?
docker inspect api-test --format '{{json .Config.Env}}'      # how was it configured?
docker image history --no-trunc devopslab-api:local          # how was the image built?
docker cp api-test:/app/appsettings.json -                   # a file from inside
docker top api-test                                          # who really runs it?
```
These five are how you diagnose a container with no shell. You will name them in
the ADR.

> Look carefully at the output of `docker cp … appsettings.json`. What is in the
> connection string? Note it in the ADR under *Consequences* — Session 7's
> scanner will ask you about it.

### 5.3 Build the comparison image

**File:** `Dockerfile.alpine` — **create** it in the repository root. It has the
same build stage; only the runtime stage differs.

```dockerfile
# syntax=docker/dockerfile:1
# Alternative runtime: Alpine. Built only to compare; Compose never uses it.

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY DevOpsLab.sln ./
COPY src/DevOpsLab.Domain/DevOpsLab.Domain.csproj                 src/DevOpsLab.Domain/
COPY src/DevOpsLab.Application/DevOpsLab.Application.csproj       src/DevOpsLab.Application/
COPY src/DevOpsLab.Infrastructure/DevOpsLab.Infrastructure.csproj src/DevOpsLab.Infrastructure/
COPY src/DevOpsLab.WebApi/DevOpsLab.WebApi.csproj                 src/DevOpsLab.WebApi/
RUN dotnet restore src/DevOpsLab.WebApi/DevOpsLab.WebApi.csproj
COPY src/ src/
RUN dotnet publish src/DevOpsLab.WebApi/DevOpsLab.WebApi.csproj \
        --configuration Release --no-restore --output /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
WORKDIR /app

# Alpine ships without full ICU. Install it, or run invariant globalization —
# decide on purpose, not from a production bug report.
RUN apk add --no-cache icu-libs curl
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false

ENV ASPNETCORE_HTTP_PORTS=8080
EXPOSE 8080
COPY --from=build /app/publish .

# The plain (non-chiseled) .NET images DEFINE the user `app` but do not SELECT it.
USER app

# Possible only because this image has a shell and curl.
HEALTHCHECK --interval=10s --timeout=3s --start-period=30s --retries=3 \
    CMD curl -fsS http://localhost:8080/healthz/ready || exit 1

ENTRYPOINT ["dotnet", "DevOpsLab.WebApi.dll"]
```
Build it and compare:

```bash
docker build -f Dockerfile.alpine -t devopslab-api:alpine .
./scripts/image-report.sh devopslab-api:local devopslab-api:alpine mcr.microsoft.com/dotnet/sdk:8.0
```
The SDK image is in the list on purpose: it shows what multi-stage saved you.

### 5.4 Write the ADR

**File:** `docs/adr-0001-base-image.md` — **create** it now and fill it in as you
go. It is finished in Step 8.

```markdown
# ADR-0001 — Runtime base image

## Status
Accepted — Session 6

## Context
- image-report.sh output (paste the table: chiseled, alpine, SDK)
- Build context size (Step 4.2): ___ MB
- Rebuild after a one-line source change (Step 4.4):
  correct layer order ___ s · wrong layer order ___ s

## Decision
<which runtime image, and why — one paragraph>

## Consequences
- What we gain:
- What we give up (name the capability, not "slightly harder to debug"):
- How we diagnose a running container without a shell:
- How we check the container's health, given that a HEALTHCHECK needs a shell:
- Readiness recovery time after a database restart (Step 8b): ___ s
```
> An ADR whose Consequences section says *"slightly harder to debug"* will be sent
> back. Name the capability you gave up — *"no `docker exec`"* — and name what
> replaces it — *"`docker logs`, `docker inspect`, `docker cp`; readiness is watched
> from outside the container"*.

> **Checkpoint 3** — `docker exec` failed on chiseled; you ran the five diagnostic
> commands; `image-report.sh` printed three rows; the ADR has its Context section
> filled in.

---

## Step 6 — Prove you are not root (20 min)

### 6.1 The proof everybody reaches for — and why it fails

```bash
docker run --rm devopslab-api:local id
```
This does **not** run `id`. The image has an `ENTRYPOINT`, so anything after the
image name is passed **as an argument to it**: Docker runs
`dotnet DevOpsLab.WebApi.dll id`, and the API starts. Press `Ctrl+C`.

To really ask for `id`, replace the entrypoint:

```bash
docker run --rm --entrypoint id devopslab-api:local     # fails: no id binary
docker run --rm --entrypoint id devopslab-api:alpine    # uid=1654(app) gid=1654(app)
```
On chiseled it fails because there is no `id`. **The absence of `id` proves
nothing** — not that you are root, not that you are not.

### 6.2 Ask the image and the kernel

```bash
./scripts/verify-nonroot.sh devopslab-api:local api-test
```
It must end with `PASS`. If it does not, read which check failed:


| Check         | What it reads                            | What a failure means                            |
| --------------- | ------------------------------------------ | ------------------------------------------------- |
| declared user | `docker image inspect` → `.Config.User` | no `USER` line, or `USER root`                  |
| published UID | `APP_UID` in the image environment       | informational only — never trust it on its own |
| actual owner  | `docker top` — the host process table   | the image lied, or the run overrode the user    |

Paste the output into your pull request later (Step 9).

> **Checkpoint 4** — `verify-nonroot.sh` prints `PASS` for `devopslab-api:local`
> with `api-test` running.

### 6.3 The trap — a write that is silently denied

This experiment is **temporary**. You will undo it at the end of the step.

The chiseled image has **no shell**, so `RUN mkdir` cannot run in the runtime
stage — `RUN` needs `/bin/sh`. Create the directory in the build stage and copy it
across, which is what a real Dockerfile would do.

**Temporarily edit** `Dockerfile`:

① At the **end of the build stage** (after the publish), add:

```dockerfile
RUN mkdir -p /out/log/devopslab
```
② In the **runtime stage**, just above `USER app`, add:

```dockerfile
COPY --from=build /out/log/devopslab /var/log/devopslab
```
`COPY` creates files and folders owned by **root** unless you say otherwise.

Rebuild, and run a container that tells Serilog to also write a log file there.
No code change is needed — Serilog reads extra sinks from configuration, and
environment variables are configuration:

```bash
docker build -t devopslab-api:trap .
docker run --rm -d --name api-trap -p 8081:8080 \
  -e Serilog__WriteTo__0__Name=File \
  -e Serilog__WriteTo__0__Args__path=/var/log/devopslab/app.log \
  -e Serilog__WriteTo__0__Args__rollingInterval=Day \
  devopslab-api:trap

curl -s http://localhost:8081/healthz/live
curl -s http://localhost:8081/healthz/live
docker logs --tail 5 api-trap              # console logging works, no error
docker cp api-trap:/var/log/devopslab ./trap-logs
ls -la ./trap-logs                         # empty — the log file was never written
```
The container started. It served requests. Its console log looks normal. The log
file does not exist, and **nothing reported an error**.

**Fix it.** Change line ② to give the folder to the runtime user:

```dockerfile
COPY --from=build --chown=app:app /out/log/devopslab /var/log/devopslab
```
(You cannot use `RUN chown` here for the same reason you could not use
`RUN mkdir`: no shell. If `--chown=app:app` is rejected, use the numeric form
`--chown=1654:1654`.)

```bash
docker stop api-trap
rm -rf ./trap-logs
docker build -t devopslab-api:trap .
# run the same `docker run … devopslab-api:trap` command again, curl it twice, then:
docker cp api-trap:/var/log/devopslab ./trap-logs
ls -la ./trap-logs                         # now a file such as app20261105.log
```
**Undo the experiment:**

```bash
docker stop api-trap
rm -rf ./trap-logs
docker image rm devopslab-api:trap
git restore Dockerfile
git status                                 # Dockerfile must not appear as modified
```
> **Every path the application writes to must be owned by the user it runs as.**
> And: **the absence of an error is not evidence of success.** It is the same
> lesson as Session 5's exit codes, and Session 7 will charge you for it again.

> **Checkpoint 5** — you saw the empty folder, then the log file after the fix,
> and `git status` shows the `Dockerfile` unchanged.

---

## Step 7 — Compose: the whole system, one command (20 min)

### 7.1 Clear the way

Two things are still holding resources that Compose is about to need:

```bash
docker stop api-test                        # it holds port 8080
docker compose down -v                      # the Session 1 database, with the OLD file
```
Run `down -v` **before** you edit the file. The old file and the new one use
different project names; if you edit first, Compose no longer recognises the old
`devopslab-sqlserver` container and refuses to create a new one with the same
name.

### 7.2 Rewrite `docker-compose.yml`

**File:** `docker-compose.yml` — **edit**. Replace the whole file with the version
below. It keeps the Session 1 database, adds the `api` service from
`docker-compose.todo.yml`, puts both on a private network and stops publishing
`1433`.

```yaml
# The API and its database, brought up by one command:
#   docker compose up -d --build
name: devopslab

services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: devopslab-sqlserver
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: "${MSSQL_SA_PASSWORD:?set MSSQL_SA_PASSWORD in .env}"
      MSSQL_PID: "Developer"
    # No `ports:` any more. The API reaches SQL Server by service name.
    networks:
      - backend
    volumes:
      - mssql-data:/var/opt/mssql
    healthcheck:
      # Healthy only once SQL Server answers a query: "running" is not "usable".
      test:
        [
          "CMD-SHELL",
          "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P \"$$MSSQL_SA_PASSWORD\" -C -Q 'SELECT 1' || exit 1"
        ]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s

  api:
    build:
      context: .
      dockerfile: Dockerfile
    image: devopslab-api:local
    container_name: devopslab-api
    depends_on:
      sqlserver:
        condition: service_healthy
    environment:
      ASPNETCORE_ENVIRONMENT: "Development"
      ASPNETCORE_HTTP_PORTS: "8080"
      # Configuration comes from the environment, never from the image.
      ConnectionStrings__Default: >-
        Server=sqlserver,1433;Database=DevOpsLab;User Id=sa;
        Password=${MSSQL_SA_PASSWORD:?set MSSQL_SA_PASSWORD in .env};
        TrustServerCertificate=True;Encrypt=True
    ports:
      - "${API_PORT:-8080}:8080"
    networks:
      - backend
    restart: unless-stopped
    # NO healthcheck block, on purpose: it would run inside the container, and the
    # chiseled image has no shell and no curl. Readiness is watched from outside.

networks:
  backend:
    driver: bridge

volumes:
  mssql-data:
```
Four decisions in that file, and you will be asked about each:


| Decision                                               | Why                                                                                                                                                     |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `condition: service_healthy`, not plain `depends_on`   | Plain `depends_on` waits for the container to **start**. SQL Server needs another 20 seconds or so before it answers a query. "Started" is not "usable" |
| `${MSSQL_SA_PASSWORD:?...}` — the `:?` form           | Compose **refuses to start** when the variable is missing, instead of inventing a default password that then runs in production for two years           |
| No `ports:` on `sqlserver`, both services on `backend` | The API finds the database by its **service name** (`Server=sqlserver,1433`) on a private network. Nothing outside needs the port                       |
| No `healthcheck:` on `api`                             | Deliberate, not forgotten. Record in the ADR how you check readiness instead                                                                            |

> **Consequence of removing `1433`.** If you used to run the API with `dotnet run`
> against this database, that no longer works: `localhost:1433` is closed. Run the
> API in Compose instead. Your integration tests are not affected — they start
> their own SQL Server with Testcontainers (Session 4).

### 7.3 Delete the TODO file

**File:** `docker-compose.todo.yml` — **delete** it. Its content now lives in
`docker-compose.yml`.

```bash
git rm docker-compose.todo.yml
```
### 7.4 Environment files

**File:** `.env` — **edit** (it is in `.gitignore` and `.dockerignore`; it is never
committed). Make sure it contains at least:

```
MSSQL_SA_PASSWORD=Your_strong_Passw0rd
API_PORT=8080
```
`MSSQL_PORT` and `ConnectionStrings__Default` from Session 1 can stay; Compose no
longer uses them.

**File:** `.env.example` — **edit**. Add the new variable, so the next person to
clone the repository knows it exists:

```
API_PORT=8080
```
### 7.5 Prove the password guard works

```bash
mv .env .env.bak
docker compose config > /dev/null           # must FAIL: "required variable MSSQL_SA_PASSWORD is missing a value"
mv .env.bak .env
docker compose config > /dev/null && echo "config OK"
```
> `env -u MSSQL_SA_PASSWORD docker compose config` is **not** a valid test while
> `.env` exists: Compose reads the value from `.env` anyway.

### 7.6 Bring it up from cold

```bash
docker compose up -d --build
docker compose ps                           # sqlserver: healthy · api: Up
curl -s http://localhost:8080/healthz/ready
```
Read the body, not just the status code. It must list a check named `database`:

```json
{
  "status": "Healthy",
  "checks": [ { "name": "database", "status": "Healthy", ... } ]
}
```
> **Checkpoint 6** — from `docker compose down -v` to a `200` on `/healthz/ready`
> with one command, and the body lists the `database` check.

---

## Step 8 — One word (Block F — do not skip, do not read ahead)

### 8a. Predict

Write both answers on paper **before running anything**:

> The system is up and healthy. You stop **only** the database container.
>
> `/healthz/live` returns ______
> `/healthz/ready` returns ______

### 8b. Measure

```bash
./scripts/readiness-experiment.sh
```
It stops the database, reads both endpoints, starts the database again and
measures how long readiness takes to come back. It must end with `PASS`. Write the
recovery time in the ADR.

> **Checkpoint 7** — `readiness-experiment.sh` prints `PASS`; your paper
> prediction is next to it.

### 8c. Break it on purpose

**Temporarily edit** `src/DevOpsLab.WebApi/HealthChecks/HealthCheckExtensions.cs`.
On the `DatabaseHealthCheck` line, change `ReadyTag` to `LiveTag`. One word:

```csharp
.AddCheck<DatabaseHealthCheck>("database", failureStatus: HealthStatus.Unhealthy, tags: new[] { LiveTag });
```
Rebuild only the API:

```bash
docker compose up -d --build api
curl -s http://localhost:8080/healthz/ready       # read the "checks" array
```
**Predict again**, on the same sheet of paper — what will the two endpoints return
with the database stopped *now*? Then:

```bash
./scripts/readiness-experiment.sh
```
It fails. Read both `FAIL` messages before you read on.

### 8d. What it means


|                              | `/healthz/live` | `/healthz/ready` |
| ------------------------------ | ----------------- | ------------------ |
| Correct, database stopped    | 200             | 503              |
| Mis-tagged, database stopped | **503**         | **200**          |

The 503 on liveness is the half most people predict. The **200 on readiness** is
the half that matters: you moved the only check away from `ready`, and a health
report with no entries is Healthy. You saw it already — the `checks` array was
empty.

> **An empty gate is a green gate.**

And the cost, which depends on who runs your container:

- **Docker Compose** marks it unhealthy at most. It does not restart it.
- **Kubernetes** kills and restarts the pod when liveness fails. Restarting does
  not bring a database back, so it restarts again — each time throwing away warm
  connections and adding a cold start to a system that is already struggling.
- **In Session 11** your Blue-Green deployment waits on readiness before switching
  traffic. Gate it on the wrong endpoint and a perfectly good release rolls itself
  back.

> Liveness answers *should this process be killed?*
> Readiness answers *should traffic be sent here?*
> A dependency outage is never a reason to kill your own process.

### 8e. Revert — now

```bash
git restore src/DevOpsLab.WebApi/HealthChecks/HealthCheckExtensions.cs
docker compose up -d --build api
./scripts/readiness-experiment.sh                  # must print PASS again
git status                                         # HealthCheckExtensions.cs must not appear
```
A team that carries this into Session 11 will lose an afternoon to it.

> **Checkpoint 8** — the defect is reverted, the experiment passes again, and the
> ADR has the recovery time.

---

## Step 9 — Hand in (10 min)

Check what you are about to commit:

```bash
git status
```
It must show exactly these changes — no more:

```
.dockerignore                                   (new)
.env.example                                    (modified)
Dockerfile                                      (renamed from Dockerfile.todo, modified)
Dockerfile.alpine                               (new)
docker-compose.todo.yml                         (deleted)
docker-compose.yml                              (modified)
docs/adr-0001-base-image.md                     (new)
scripts/image-report.sh                         (new)
scripts/readiness-experiment.sh                 (new)
scripts/verify-nonroot.sh                       (new)
```
If `.env`, `trap-logs/`, `Product.cs` or `HealthCheckExtensions.cs` appear, an
experiment was not undone — go back to the step that made it.

Commit, push, and open a pull request. The body must contain, in this order:

1. The `image-report.sh` table for the images you built
2. `verify-nonroot.sh` output, with the container name
3. The build time before and after fixing the layer order
4. `readiness-experiment.sh` output, passing, with the recovery time
5. A link to `docs/adr-0001-base-image.md`

CI must be green — `build-and-test` has been a required status check since
Session 5, and that has not changed. A teammate reviews and approves the latest
push before you merge.

> **Checkpoint 9** — the pull request is open with all five items, and CI is green.

### Sabotage card (homework, dealt in Block G)

Your team receives one card face down. Before Session 7:

1. Apply the card to **your own** repository on a branch `sabotage/s06-card`, push,
   and open a pull request. Do not say which card it is.
2. Review the next team's sabotage pull request with the
   `S06-Image-Review-Checklist`. Build their branch yourself — do not trust a
   screenshot.
3. Close your sabotage PR without merging once it has been reviewed, and delete
   the branch.

---

## Troubleshooting


| Symptom                                                    | Cause                                                              | Fix                                                                                                                   |
| ------------------------------------------------------------ | -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `docker version` shows no Server                           | Docker Desktop not running, or virtualisation off                  | Start Docker Desktop; tell the instructor; pair with a teammate                                                       |
| `transferring context` is hundreds of MB                   | `.dockerignore` missing, misnamed or in the wrong folder           | It must be called exactly `.dockerignore`, next to the `Dockerfile`                                                   |
| `COPY failed: … DevOpsLab.WebApi.csproj: not found`       | Wrong path or wrong **case**                                       | Paths are case-sensitive inside the build. Match the folder names exactly                                             |
| `error NU1301: Unable to load the service index`           | No network during restore                                          | Check the proxy/Wi-Fi; retry. Pre-pulled images do not include NuGet packages                                         |
| Publish fails with `--no-restore` complaining about assets | The restore layer did not cover a project                          | A `.csproj` copy line is missing above the restore                                                                    |
| `exec: "sh": executable file not found`                    | Chiseled image has no shell                                        | Expected. Use `docker logs`, `inspect`, `cp`, `top`                                                                   |
| `docker run IMAGE id` starts the API instead               | `id` was passed as an argument to the `ENTRYPOINT`                 | `docker run --rm --entrypoint id IMAGE`                                                                               |
| `/bin/sh: not found` during `docker build`                 | A `RUN` line in the **chiseled** runtime stage                     | Move the work to the build stage and `COPY` the result                                                                |
| `unable to find user app`                                  | `USER` names a user the base image does not define                 | `USER app` on .NET 8 images; check spelling                                                                           |
| Container exits at once, permission error on bind          | Port moved to 80                                                   | `ASPNETCORE_HTTP_PORTS=8080`                                                                                          |
| `port is already allocated` on 8080                        | `api-test` (Step 4) is still running                               | `docker stop api-test`                                                                                                |
| `container name "/devopslab-sqlserver" is already in use`  | Old Session 1 container                                            | `docker rm -f devopslab-sqlserver`, then `docker compose up -d`                                                       |
| `service "api" refers to undefined network backend`        | Top-level `networks:` block missing                                | Add `networks: backend: driver: bridge` at the end of the file                                                        |
| `required variable MSSQL_SA_PASSWORD is missing a value`   | No `.env`, or the variable is empty                                | Create `.env` (Step 7.4). This error is the guard working                                                             |
| API container restarts in a loop                           | It started before the database was usable                          | `condition: service_healthy` on `depends_on`                                                                          |
| `/healthz/ready` returns 503 with Compose up               | Database still starting, or wrong connection string                | `docker compose ps` — wait for `healthy`; `docker compose logs api`                                                  |
| `readiness-experiment.sh` shows `000` for ready            | The readiness check took longer than the script's 5-second timeout | Add `Connect Timeout=3;` to the connection string in `docker-compose.yml`; tell the instructor the value you measured |
| `/usr/bin/env: 'bash\r': No such file or directory`        | The script was saved with Windows line endings                     | `sed -i 's/\r$//' scripts/*.sh`                                                                                       |
| `Permission denied` running `./scripts/...`                | Executable bit missing                                             | `chmod +x scripts/*.sh` and `git update-index --chmod=+x` (Checkpoint 0)                                              |
| Disk full, builds slow down                                | Old images, build cache, volumes                                   | `docker system df`, then `docker image prune` and `docker builder prune`                                              |

---

## Before Session 7

- [ ]  Pull request merged through a green `build-and-test`
- [ ]  `docker compose up -d` works from cold on a classroom machine, not only yours
- [ ]  The Block F mis-tag is reverted — `curl …/healthz/ready` lists the `database` check
- [ ]  Sabotage card applied, and the next team's PR reviewed with the Image Review Checklist
- [ ]  Backlog items 14, 15 and 16 moved to Done — 14 with the image report, 15 with
  the `verify-nonroot.sh` output and the ADR, 16 with the readiness-experiment
  output. (Item 15's acceptance text says `docker run --rm <image> id`; Step 6.1
  explains why `verify-nonroot.sh` is the evidence instead.)

**What you take into Session 7.** You have an image. It is small, it does not run
as root, and it answers honestly about its own health. You still have **no idea
what is inside it**: which libraries, which versions, which known vulnerabilities.
Next session you scan it — and then you find out whether the scanner actually
stops anything, or whether it is one more green tick with nothing behind it.
