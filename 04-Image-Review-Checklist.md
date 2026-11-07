# Session 6 — Image Review Checklist

Use this to review **another team's** image, in the pull request, before Session 7.
One sabotage card has been applied to it. You are not told which.

A finding is only a finding when you can name **the command you ran** and **what it
printed**. "Looks fine" is not a review.

---

## How to use it

1. Clone the branch. Build it yourself — do not trust a screenshot.
2. Work through Parts A to E in order. Each part has a command.
3. Write one review comment per finding, in this shape:

   > **Part C, non-root.** `docker image inspect --format '{{.Config.User}}'`
   > prints an empty string, so the container runs as root. `APP_UID=1654` is set
   > in the environment, which looks like a fix but changes nothing — only
   > `Config.User` is enforced. Add `USER app` to the runtime stage.

4. If you find nothing, say what you ran and that it passed. An empty review is
   not a pass; it is a review that did not happen.

---

## Part A — What is in the image?

| # | Check | Command | Pass |
|---|---|---|---|
| A1 | The runtime stage is not an SDK image | read the second `FROM` | ☐ |
| A2 | Image size is in the same order of magnitude as the reference | `./scripts/image-report.sh IMG` | ☐ |
| A3 | No test project, no `.git`, no `docs/` inside | `docker image history --no-trunc IMG` | ☐ |
| A4 | No secret in the image environment | `docker image inspect --format '{{json .Config.Env}}' IMG` | ☐ |
| A5 | No secret in any layer command | `docker image history --no-trunc IMG \| grep -i -E 'password\|secret\|token\|key'` | ☐ |

> A4 and A5 are different checks. A password can be in `ENV` (A4) or in a `RUN`
> line that no longer exists in the final filesystem but is still in the history
> (A5). Both reach anyone who can pull the image.

---

## Part B — What is in the build context?

| # | Check | Command | Pass |
|---|---|---|---|
| B1 | `.dockerignore` exists | `ls -la .dockerignore` | ☐ |
| B2 | It excludes `bin/`, `obj/`, `.git/` | read it | ☐ |
| B3 | It excludes `.env`, `*.key`, `*.pfx`, `secrets.json` | read it | ☐ |
| B4 | Context size is small | first line of `docker build` | ☐ |

> A context above ~50 MB for this project means B1–B3 are not doing their job.
> Report the number you saw.

---

## Part C — Who does it run as?

| # | Check | Command | Pass |
|---|---|---|---|
| C1 | The image declares a user | `docker image inspect --format '{{.Config.User}}' IMG` — empty means **root** | ☐ |
| C2 | That user exists in the base image | start the container; a missing user fails at start-up | ☐ |
| C3 | The running process is not root on the host | `docker top CONTAINER` | ☐ |
| C4 | The port is unprivileged | `ASPNETCORE_HTTP_PORTS` is 8080, not 80 | ☐ |
| C5 | Every written path is owned by the runtime user | find the log/temp paths, then check the container writes them | ☐ |

> C5 is the one teams miss, because it fails **silently**. The container starts,
> serves traffic, and writes nothing. Do not look for an error; look for the file
> that should exist and does not.
>
> Short cut for C1–C3: `./scripts/verify-nonroot.sh IMG CONTAINER`.

---

## Part D — Does the cache work?

| # | Check | Command | Pass |
|---|---|---|---|
| D1 | Project files are copied and restored above the source copy | read the build stage | ☐ |
| D2 | A one-line source change does not re-run `restore` | `touch` a `.cs` file, rebuild, watch which step re-runs | ☐ |
| D3 | The author recorded a before/after build time | read the pull request body | ☐ |

---

## Part E — Does it tell the truth about its health?

| # | Check | Command | Pass |
|---|---|---|---|
| E1 | `/healthz/live` does not touch the database | read `AddApplicationHealthChecks` | ☐ |
| E2 | `/healthz/ready` does touch the database | same place | ☐ |
| E3 | With the database stopped: live 200, ready 503 | `./scripts/readiness-experiment.sh` | ☐ |
| E4 | `/healthz/ready` has at least one check behind it | `curl -s .../healthz/ready` and read the `checks` array | ☐ |
| E5 | Readiness recovers without a restart | the recovery line of the same script | ☐ |

> E4 looks redundant next to E3 and is not. A readiness endpoint whose checks all
> moved elsewhere answers **200 with an empty list**. It is green because it is
> empty. Read the body, not the status code.

---

## Part F — Compose

| # | Check | Command | Pass |
|---|---|---|---|
| F1 | `depends_on` uses `condition: service_healthy` | read the file | ☐ |
| F2 | The database port is not published to the host | `docker compose config` | ☐ |
| F3 | No password literal anywhere in the compose file | read it; `${VAR:?...}` is what you want to see | ☐ |
| F4 | Compose refuses to start with the variable unset | `env -u MSSQL_SA_PASSWORD docker compose config` | ☐ |
| F5 | `docker compose up -d` works from a cold machine | `docker compose down -v` first, then up | ☐ |

---

## Result

**Defect found:** ☐ yes  ☐ no

**Which sabotage card do you think it was, and what gave it away?**

```
Card number: ____    Evidence (the command and its output):


```

**What did the author do well?** Name one thing specifically. A review that only
lists faults teaches the author to hide things from the next one.

```


```

**Reviewer:** ______________________  **Team reviewed:** ______________________
