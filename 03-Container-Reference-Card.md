# Container & Health Check Reference Card

Session 6 · MH24 — Software Application Development (DevOps)
Keep this next to the keyboard. Everything here is used again in Sessions 7, 9, 11 and 13.

---

## 1. The five nouns

| Noun | What it is | What it is not |
|---|---|---|
| **Image** | A read-only stack of layers plus metadata | Not a virtual machine, not a running thing |
| **Layer** | The filesystem change produced by one instruction | Not a file, not a version |
| **Container** | One process from an image, with a writable layer on top | Not a small VM — there is no kernel, no init, no boot |
| **Build context** | The directory uploaded to the daemon before the build | Not "the files the Dockerfile mentions" |
| **Registry** | Where images are stored and pulled from | Not a backup (Session 8) |

---

## 2. The cache rule

Docker reuses a layer when the instruction **and every layer above it** are
unchanged.

> **Anything that changes on every commit must sit below anything that is slow.**

```dockerfile
COPY *.csproj ./          # changes rarely
RUN dotnet restore        # slow
COPY src/ src/            # changes constantly
RUN dotnet publish        # slow, but unavoidable
```

Check your own order with one question: *after a one-character change in a `.cs`
file, which instruction is the first one to re-run?* If the answer is `restore`,
the order is wrong.

---

## 3. Multi-stage, annotated

```dockerfile
# syntax=docker/dockerfile:1

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build      # has a compiler
WORKDIR /src
COPY *.sln ./                                       # 1. dependency manifests
COPY src/*/*.csproj src/                            #    (one COPY per project)
RUN dotnet restore <web project>                    # 2. the slow, cacheable step
COPY src/ src/                                      # 3. the volatile step
RUN dotnet publish <web project> --no-restore \     # 4. --no-restore asserts step 2 worked
        -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0-noble-chiseled AS runtime   # no compiler
WORKDIR /app
ENV ASPNETCORE_HTTP_PORTS=8080                      # 8080 so a non-root user can bind it
EXPOSE 8080                                         # documentation only
COPY --from=build /app/publish .                    # only the published output crosses
USER app                                            # non-root
ENTRYPOINT ["dotnet", "DevOpsLab.WebApi.dll"]
```

**The rule the whole file serves:** everything in the runtime image is attack
surface you chose to ship.

---

## 4. Choosing a base image

| Variant | Shell | Package manager | Globalization | Notes |
|---|---|---|---|---|
| `8.0` (full runtime) | yes | yes | yes | Largest surface. Runs as root unless you add `USER app` |
| `8.0-alpine` | yes | `apk` | needs `icu-libs` | musl libc. A shell means `docker exec` and `HEALTHCHECK` work |
| `8.0-noble-chiseled` | **no** | **no** | **no** | Smallest surface, non-root by default. No `exec`, no `curl`, no `HEALTHCHECK` |
| `8.0-noble-chiseled-extra` | no | no | **yes** | Chiseled plus ICU and tzdata |

**Size is the least interesting column.** The two that decide your night shift are
*runs as root* and *has a shell*.

Whatever you choose, write it in `docs/adr-0001-base-image.md` with the numbers
from `image-report.sh` and a Consequences section that names what you gave up.

---

## 5. Non-root: claim, and proof

```dockerfile
USER app        # the .NET 8 images define the user `app`, UID 1654
```

| Proof | Command | Trust it? |
|---|---|---|
| What the image declares | `docker image inspect --format '{{.Config.User}}' IMG` | Yes — empty means **root** |
| The published non-root UID | `docker image inspect --format '{{json .Config.Env}}' IMG` | Informational. Read the value; never assume it |
| What the kernel sees | `docker top CONTAINER` | **Yes** — the host process table cannot be faked by the image |
| `docker run --rm IMG id` | — | Only on images that have `id`. Chiseled does not; its absence proves nothing |

`./scripts/verify-nonroot.sh IMAGE [CONTAINER]` runs the first three and exits
non-zero on failure.

**The trap:** every path the application writes to must be owned by the runtime
user. A directory created by `RUN mkdir` belongs to root. The container will start
anyway, serve traffic, write nothing, and report no error.

---

## 6. Diagnosing a container with no shell

| You need | Command |
|---|---|
| What it said | `docker logs --tail 100 -f CONTAINER` |
| How it was configured | `docker inspect CONTAINER` |
| How the image was built | `docker image history --no-trunc IMAGE` |
| A file from inside | `docker cp CONTAINER:/app/appsettings.json -` |
| Which user it really runs as | `docker top CONTAINER` |
| Whether a secret got baked in | `docker image inspect --format '{{json .Config.Env}}' IMAGE` |

---

## 7. Liveness and readiness

| | `/healthz/live` | `/healthz/ready` |
|---|---|---|
| The question | Should this **process be killed**? | Should **traffic be sent** here? |
| Checks | Only the process itself | Every dependency needed to serve a real request |
| Must it touch the database? | **Never** | **Yes** |
| On failure, the platform should | restart the container | stop sending traffic, do not restart |

```csharp
services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { LiveTag })
    .AddCheck<DatabaseHealthCheck>("database", tags: new[] { ReadyTag });   // ← ready, not live

endpoints.MapHealthChecks("/healthz/live",  new HealthCheckOptions {
    Predicate = r => r.Tags.Contains(LiveTag) });
endpoints.MapHealthChecks("/healthz/ready", new HealthCheckOptions {
    Predicate = r => r.Tags.Contains(ReadyTag) });
```

**Stop only the database and you should see:**

| Tagging | `/healthz/live` | `/healthz/ready` |
|---|---|---|
| Correct | 200 | 503 |
| Database tagged `live` | **503** | **200** |

The second row is the defect. The 503 is the obvious half; the **200** is the
dangerous one — `ready` has no checks left, and a health report with no entries is
Healthy. **An empty gate is a green gate.**

What it costs depends on the platform, so be precise:

- **Docker / Compose** — marks the container unhealthy. Does **not** restart it.
- **Kubernetes** — restarts the pod on liveness failure. A restart cannot fix a
  database outage, so it restarts again.
- **Session 11** — the Blue-Green switch waits on readiness. Wrong endpoint,
  wrong decision, automatic rollback of a healthy release.

> A dependency outage is never a reason to kill your own process.

---

## 8. Compose, the four lines that matter

```yaml
services:
  api:
    depends_on:
      sqlserver:
        condition: service_healthy        # "started" is not "usable"
    environment:
      ConnectionStrings__Default: "...${MSSQL_SA_PASSWORD:?set it in .env}..."
    ports:
      - "8080:8080"                       # the database port is NOT published
    networks: [backend]                   # service name = hostname
```

| Line | Why |
|---|---|
| `condition: service_healthy` | Plain `depends_on` waits for the container to start, not for SQL Server to answer a query |
| `${VAR:?message}` | Compose refuses to start when the variable is missing, instead of inventing a default password |
| no published `1433` | The API reaches the database by service name on a private network |
| no `healthcheck:` on the API | A container healthcheck runs *inside* the container; a chiseled image has nothing to run it with. Watch readiness from outside |

---

## 9. Before you approve a pull request that adds a Dockerfile

- [ ] Runtime stage is **not** an SDK image
- [ ] `USER` is present and names a user the base image defines
- [ ] Port is 8080 or another unprivileged port
- [ ] Project files are copied and restored **above** the source copy
- [ ] `.dockerignore` exists and excludes `bin/`, `obj/`, `.git/`, `.env`, keys
- [ ] No connection string, password or token in `ENV` or `ARG`
- [ ] Every path the app writes to is owned by the runtime user
- [ ] The base-image choice is recorded in an ADR, with numbers

---

## 10. Commands worth memorising

```bash
docker build -t name:tag .                   # watch line 1: context size
docker image ls                              # size column
docker image history --no-trunc IMAGE        # what each layer added
docker image inspect --format '{{.Config.User}}' IMAGE
docker top CONTAINER
docker logs --tail 100 -f CONTAINER
docker compose up -d --build
docker compose ps                            # STATUS shows healthy / unhealthy
docker compose logs -f api
docker compose down -v                       # -v also removes the volume: a true cold start
docker system df                             # where the disk went
```
