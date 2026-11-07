# Session 6 — Lesson Plan

**Course** MH24 — Software Application Development (DevOps) · 75 hours · 3 credits
**Session** 6 of 15 · Chapter 4 · **4 hours (240 minutes)**
**Title** Docker Multi-stage, Container Security, Health Checks & Docker Compose
**Hour split** Theory 1h · Practice 3h
**Objectives covered** KT3 · KT5 · KN3 · NLTC2
**Assessment** **Lab 3 — Secure Container, Liveness/Readiness & Compose (5% of the course)**

---

## 1. Learning outcomes

| # | Outcome | Evidence observed in class |
|---|---|---|
| LO1 | Explain image layers and the build cache; predict which layers a change invalidates | Student states, before building, how many layers a one-line C# edit rebuilds |
| LO2 | Write a multi-stage Dockerfile whose runtime stage contains no SDK, no tests, no `.git` | A built image, and `image-report.sh` showing its size next to the SDK image |
| LO3 | State the chiseled/alpine trade-off in terms of *what you lose*, not only what you gain | A written ADR naming the diagnostic capability given up and what replaces it |
| LO4 | Run the container as a non-root user and prove it without `id` | `verify-nonroot.sh` passing, plus `docker top` output |
| LO5 | Explain why liveness and readiness are different questions, and what each one authorises the platform to do | Student answers "what should the platform do when this returns 503?" for each endpoint |
| LO6 | Bring up API + database with one command, with the API waiting for a *usable* database | `docker compose up -d` from a cold machine, first request succeeds |
| LO7 | Predict and then measure what a one-word tagging mistake does to both endpoints | A written prediction, and the `readiness-experiment.sh` output that contradicts it |

## 2. Preparation

**Verify before the session** (Session 5 homework)

- [ ] Every team's `ci.yml` is green on `main` and is a **required** status check
- [ ] `defaults: run: shell: bash` is present in every team's workflow (the Session 5 repair)
- [ ] Each team's suite still passes on a clean clone — today adds a container, not a rewrite
- [ ] `docs/` exists in every repository; today it gains `adr-0001-base-image.md`

**Instructor checklist**

- [ ] **Pull the images onto every classroom machine the day before**: `mcr.microsoft.com/dotnet/sdk:8.0`, `aspnet:8.0-noble-chiseled`, `aspnet:8.0-alpine`, `mssql/server:2022-latest`. Thirty students pulling 1.5 GB at once is how Block C is lost.
- [ ] Docker Desktop actually **running** on every machine, virtualisation enabled in BIOS, administrator rights present. This is on the Session 1 checklist; re-verify today, because today it is the entire lesson.
- [ ] Build both reference images in your demo repository and run `image-report.sh`. **Replace the placeholder numbers in the slides with yours.**
- [ ] Run `readiness-experiment.sh` **both ways** — correct and mis-tagged — and keep both outputs as the offline fallback.
- [ ] Run `docker exec -it <container> sh` against the chiseled image once, and keep the error text.
- [ ] Sabotage cards printed, one per team, cut; keep the key.
- [ ] Disk space checked. Images, layers and a stopped SQL Server volume fill a classroom machine faster than anyone expects.

> **Carry-forward from Session 5.** You closed with: *"CI runs your tests on a
> machine you do not control, and that is the point."* Open with the next
> sentence: **today you build the machine and hand it over.**

## 3. Timeline

| Block | Time | Min | Mode | Content |
|---|---|---|---|---|
| A | 00:00–00:15 | 15 | Plenary | Recap; "it works on my machine" as an engineering defect |
| B | 00:15–00:55 | 40 | **Theory** | Image anatomy: layers, cache, build context, multi-stage |
| — | 00:55–01:05 | 10 | Break | |
| C | 01:05–01:55 | 50 | Hands-on | Write the Dockerfile, first green build, `.dockerignore`, measure |
| D | 01:55–02:20 | 25 | **Demo** | **The image you chose has no shell** — base-image trade-off |
| — | 02:20–02:30 | 10 | Break | |
| E | 02:30–02:55 | 25 | Hands-on | Non-root: prove it, and the write-permission trap |
| F | 02:55–03:40 | 45 | **Pivot** | Compose, and one word that moves an outage |
| G | 03:40–04:00 | 20 | Plenary | Lab 3 evidence, sabotage cards, close |

---

## Block A — "It works on my machine" is a defect report (15 min)

**Open with a question, not a definition.**

> Session 5 proved your tests pass on a GitHub runner. So the code is fine. Why
> would it still break in production?

Collect answers on the board. Steer towards the list of things that decide how a
program behaves and that **Git does not version**:

- the runtime version actually installed
- system libraries, and their versions
- environment variables
- the time zone, the locale, the character set
- the user the process runs as, and what that user may write to

Then the framing for the session:

> A container is not a virtual machine and it is not a deployment tool. It is a
> way to put that list under version control.

**Name the sequencing rule out loud**, because students will meet it again in
Chapter 6: the artifact that gets promoted from Staging to Production is the
**container image**, not the compiled binaries. That is why containers are taught
before advanced CI, and not the other way round.

**Hand out** the Student Lab Guide, the Container Reference Card and the Image
Review Checklist. The Assessment Checklist stays with you.

---

## Block B — Image anatomy (40 min, theory)

**B1. Image and container (5 min).** An image is a read-only stack of layers plus
metadata. A container is one running process with a writable layer on top. Two
containers from one image share the layers; they do not share the writable layer.

**B2. Layers and the cache (15 min).** Each instruction produces a layer. Docker
reuses a layer when the instruction **and everything above it** are unchanged.

Put this on the board and ask for predictions before giving the answer:

```
COPY . .                      ← the whole source
RUN dotnet restore            ← 40 seconds
RUN dotnet publish            ← 30 seconds
```

> You change one character in one `.cs` file. How many seconds does the next
> build take?

The answer is 70, every time, forever — because `COPY . .` changed, so everything
below it is invalidated. Then show the fix, and make the students state the rule
themselves:

> **Anything that changes on every commit must sit below anything that is slow.**

**B3. The build context (8 min).** `docker build .` first **uploads the directory**
to the daemon. Everything in it can end up in the image. `.dockerignore` is not
an optimisation; it is the boundary of what you are handing over. `bin/`, `obj/`,
`.git/` and a local `.env` are the usual passengers.

**B4. Multi-stage (12 min).** The build stage needs a compiler. The runtime stage
must not have one. Walk the reference `Dockerfile` top to bottom and ask at each
line: *what is in the final image because of this line?*

Close the block with the sentence the rest of the session tests:

> Everything in the runtime image is attack surface you chose to ship.

---

## Block C — Write the Dockerfile (50 min, hands-on)

Students work from **Steps 1–4** of the Lab Guide. Target: a green build and an
image that runs.

**Circulate with four questions, not with answers:**

1. "What is in your build context?" — the first line of `docker build` prints its
   size. A 300 MB context means `.dockerignore` is missing or wrong.
2. "Which layer does your next commit invalidate?" — if the answer is "all of
   them", the project-file copy is in the wrong place.
3. "Is the SDK in your final image?" — ask them to find out rather than answer.
4. "Where does the connection string come from?" — if the answer is "the
   Dockerfile", stop them. That is sabotage card 8, arriving early and for free.

**Checkpoint before the break.** Everyone must have:

```
docker build -t devopslab-api:local .
docker run --rm -p 8080:8080 devopslab-api:local
curl -i http://localhost:8080/healthz/live
```

A student whose build fails at the break will not catch up. Pair them with a team
that is finished and keep moving.

---

## Block D — The image you chose has no shell (25 min, demo)

**Do not explain first. Let them try it.**

Ask the room to get a shell inside the running container, the way everyone always
does:

```
docker exec -it devopslab-api sh
```

It fails. There is no `sh`. There is no `bash`, no `apt`, no `curl`, no `id`.

> You picked the most secure base image on the list, ten minutes ago, without
> reading what it costs. The bill arrives the first time production misbehaves at
> two in the morning.

**Then the comparison (10 min).** Build the alpine variant, run `image-report.sh`
on both, and put the real numbers on the board. Discuss what each column actually
decides:

- *Size* — the least interesting. It changes a pull time.
- *USER* — decides what an exploit gets.
- *Has a shell* — decides how you debug, and whether a container `HEALTHCHECK`
  can exist at all.

**Three ways to diagnose a container with no shell (8 min)** — demonstrate each:

| Need | Command |
|---|---|
| What did it say? | `docker logs --tail 100 -f <container>` |
| How was it configured? | `docker inspect <container>` · `docker image history --no-trunc <image>` |
| What is on its filesystem? | `docker cp <container>:/app/appsettings.json -` |

**The consequence nobody expects (7 min).** A container `HEALTHCHECK` runs *inside*
the container. No shell and no `curl` means there is nothing to run. Put the
three options on the board, state that **the course takes the third**, and require
the decision in writing in `docs/adr-0001-base-image.md`:

| Option | What it costs |
|---|---|
| Non-chiseled image in dev, chiseled in production | Two different images — breaks Build Once, Promote Everywhere (Chapter 6) |
| Ship a small health-probe executable in the image | Extra build output and size, for one HTTP GET |
| **No container healthcheck; watch from outside** | `depends_on: service_healthy` cannot be used for the API |

> A watcher that lives inside the thing it watches dies with it. Hold that
> sentence; Session 13 is built on it.

---

## Block E — Non-root, and the trap that follows (25 min, hands-on)

**E1. Prove it (10 min).** The usual proof does not work here:

```
docker run --rm devopslab-api:local id     # there is no id binary
```

The absence of `id` is **not** evidence of anything. Run `verify-nonroot.sh`
instead, and read what it actually asks:

- `docker image inspect --format '{{.Config.User}}'` — what the image declares
- the published non-root UID, if the image publishes one — **read it, do not assume the name**
- `docker top <container>` — what the host kernel says the process owner is

**E2. The trap (15 min).** Have every student add this to their runtime stage,
rebuild, and run:

```dockerfile
RUN mkdir -p /var/log/devopslab
```

The directory is created by root at build time. The non-root process cannot write
to it. The container starts. It serves traffic. It logs nothing — and **nothing
reports an error**.

> A crash tells you where to look. A silent permission failure does not, and your
> dashboard stays green while the evidence you need in Session 14 is not being
> written anywhere.

The fix is `chown` at build time, or writing only to paths the runtime user owns.
Make them state the rule:

> **Every path the application writes to must be owned by the user it runs as.**

---

## Block F — One word moves an outage (45 min) — **the pivot**

**F1. Compose (15 min).** Students merge the `api` service into
`docker-compose.yml` following Step 6 of the Lab Guide, and bring the whole system
up from cold:

```
docker compose up -d --build
```

Two things to draw out while they work:

- `depends_on: condition: service_healthy`, not plain `depends_on`. "Started" is
  not "usable" — the same distinction as liveness versus readiness, one level up.
- The database port is **no longer published**. The API reaches SQL Server by
  service name on a private network. Ask what else used to reach it on 1433.

**F2. The prediction (5 min).** Everyone writes two answers on paper, before
touching anything:

> With the system up and healthy, you stop **only** the database container.
> What does `/healthz/live` return? What does `/healthz/ready` return?

Collect a show of hands for each combination. Do not comment on it.

**F3. The measurement (10 min).** Run the experiment:

```
./scripts/readiness-experiment.sh
```

Correct tagging gives **live 200, ready 503**, and readiness comes back on its own
when the database returns — no restart, no cold start, no lost connections.

**F4. The defect (15 min).** Now have them make the one-word change in
`HealthCheckExtensions.cs` — the database check moves from the `ready` tag to the
`live` tag — rebuild, and predict again before re-running the experiment.

| | `/healthz/live` | `/healthz/ready` |
|---|---|---|
| Correct, database stopped | 200 | 503 |
| Mis-tagged, database stopped | **503** | **200** |

Most of the room predicts the 503. Almost nobody predicts the **200**, and that is
the half that matters: with the database check moved away, `/healthz/ready` has no
checks left, and a health report with no entries is Healthy.

> **An empty gate is a green gate.** You saw this in Session 5 with a pipeline
> that was green with a failing test. It is the same defect wearing different
> clothes, and it will appear again in Session 7 with a vulnerability scan that
> runs and never fails.

**Say what it costs, precisely.** Do not overstate it:

- **Docker Compose** marks the container unhealthy and does **not** restart it.
- **Kubernetes** kills and restarts the pod when liveness fails. A restart cannot
  bring a database back, so it restarts again — throwing away warm connections and
  adding a cold start to an already degraded system. (Kubernetes is outside this
  course. The behaviour is stated so the rule is understood, not merely obeyed.)
- **Session 11** gates the Blue-Green traffic switch on readiness. Point it at the
  wrong endpoint and a healthy release rolls itself back.

Leave this on the board for the rest of the course:

> Liveness answers *should this process be killed?*
> Readiness answers *should traffic be sent here?*
> A dependency outage is never a reason to kill your own process.

**Everyone reverts the defect before Block G.** Verify it; a team that carries this
into Session 11 will lose an afternoon to it.

---

## Block G — Evidence, cards, close (20 min)

**Lab 3 evidence (12 min).** Each team demonstrates, live, on its own machine:

1. `image-report.sh` output for their image — size, layers, USER
2. `verify-nonroot.sh` passing, with the running container named
3. `docker compose up -d` from cold, and a successful first request
4. `readiness-experiment.sh` passing, with the recovery time read aloud
5. `docs/adr-0001-base-image.md` — the decision, the numbers, and what was given up

Mark against the Assessment Checklist as they present. Do not accept a claim
without the command that produced it.

**Sabotage cards (5 min).** Deal one card per team, face down. Homework: apply the
card to your own repository, push, and review the next team's image with the Image
Review Checklist before Session 7. Keep the key.

**Close (3 min).** Set up Session 7 with the thing they just built:

> Your image works, it is small, and it does not run as root. You still have no
> idea what is inside it. Next session we scan it — and we find out whether the
> scanner actually stops anything.

---

## 4. Instructor notes

**The block that overruns is C.** It is the first build on a cold machine. If the
images were not pre-pulled, it will not fit in 50 minutes. If you are behind after
C, cut Block E to the `verify-nonroot.sh` run plus the write-permission trap, and
**protect Block F**. The pivot is the session; everything else is scaffolding.

**Do not let Block D become a security lecture.** The point is not that chiseled
is better. The point is that a security decision has an operational bill, that the
bill is paid by whoever is on call, and that it must be written down before it is
paid.

**On the ADR.** Reject any ADR whose "trade-off" section says *"slightly harder to
debug"*. Require a sentence naming the capability given up and the replacement:
*"no `docker exec`; diagnosis is by `docker logs`, `docker inspect` and
`docker cp`, and readiness is watched from outside the container."*

**On the write-permission trap.** Some students will not believe the container is
really failing, because nothing says so. Make them look for the log file and find
it absent. The lesson is not about permissions; it is that **the absence of an
error is not evidence of success** — which is also why Session 5 measured exit
codes and why Session 7 will plant a CVE on purpose.

**If Docker is unusable on a machine**, that student pairs. They do not spend the
session fixing their laptop. Record it: a machine that cannot run Docker cannot
sit Periodic Test 1 in Session 8, and that is a problem to solve this week, not in
Session 8.

**Common misconceptions, ready to correct:**

| Said in class | Correction |
|---|---|
| "The container is a small VM" | One process, shared kernel. No kernel, no init, no boot |
| "`EXPOSE` publishes the port" | `EXPOSE` is documentation. `-p` or `ports:` publishes |
| "`latest` is a version" | It is a mutable label. Chapter 6 promotes by digest for exactly this reason |
| "Alpine is smaller so it is more secure" | Different libc, no ICU by default, and a shell. Smaller is not the same as safer |
| "The healthcheck restarts my container" | Docker marks it unhealthy. Kubernetes restarts. Know which platform you are on |

## 5. Materials

| Item | Where |
|---|---|
| Student Lab Guide | `02-Student-Lab-Guide.md` — hand out at the start of Block A |
| Container & Health Check Reference Card | `03-Container-Reference-Card.md` — hand out with the Lab Guide |
| Image Review Checklist | `04-Image-Review-Checklist.md` — used in Block G and for the homework review |
| Assessment Checklist | `05-Assessment-Checklist.md` — instructor only |
| Reference Dockerfiles, Compose files, scripts, sabotage cards | `instructor-solution/` — instructor only |
| Starter Project | `Dockerfile.todo`, `docker-compose.todo.yml` are the students' starting points |
