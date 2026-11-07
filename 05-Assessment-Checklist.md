# Session 6 — Assessment Checklist

**Instructor only.** Lab 3 is worth **5% of the course**. Mark during Block G,
live, on the student's machine.

**Principle, unchanged since Session 2:** a configuration that *looks* right is
not evidence. Every row below is marked from a command run in front of you, or it
is not marked.

---

## Part A — Individual (observed during the session)

| Student | Built a working image | Ran the non-root proof | Wrote a prediction in Block F | Reverted the mis-tag | Notes |
|---|---|---|---|---|---|
| | ☐ | ☐ | ☐ | ☐ | |
| | ☐ | ☐ | ☐ | ☐ | |
| | ☐ | ☐ | ☐ | ☐ | |
| | ☐ | ☐ | ☐ | ☐ | |

> The Block F prediction column is not about being right. Record whether the
> student **committed to an answer before measuring**. A student who waits to see
> the result and then says "that is what I thought" has not done the exercise, and
> will do the same thing during the Game Day in Session 14.

---

## Part B — Team: Lab 3 evidence (5%)

Each row: the team runs the command, you read the output.

| # | Evidence | Command the team must run | Weight | Pass |
|---|---|---|---|---|
| B1 | A multi-stage image that runs | `docker build` then `docker run` | 1 | ☐ |
| B2 | Runtime stage contains no SDK | `./scripts/image-report.sh` — size next to the SDK image | 0.5 | ☐ |
| B3 | Build context is small | first line of `docker build` | 0.5 | ☐ |
| B4 | Non-root, proved three ways | `./scripts/verify-nonroot.sh IMG CONTAINER` | 1 | ☐ |
| B5 | `docker compose up -d` from cold, first request succeeds | `docker compose down -v && docker compose up -d --build` | 1 | ☐ |
| B6 | Liveness/readiness behave correctly with the database stopped | `./scripts/readiness-experiment.sh` | 1 | ☐ |
| B7 | ADR records the decision, the numbers and what was given up | read `docs/adr-0001-base-image.md` | 1 | ☐ |

**Total 6 points → scaled to 5%.**

### Marking notes

- **B2** — if the team cannot say roughly how much smaller their runtime image is
  than the SDK image, they have not looked. Half marks.
- **B4** — a team that proves non-root by running `id` has not understood the
  image they chose. Ask what happens when `id` is missing. If the answer is "then
  it is not root", no marks for this row: the absence of a binary proves nothing.
- **B6** — this must be the script, not a screenshot and not a verbal claim. The
  script exits non-zero when the tagging is wrong; that is the whole point.
- **B7** — reject *"slightly harder to debug"*. Require the capability given up and
  its replacement named explicitly. This row is where the engineering judgement is.

---

## Part C — Understanding check (ask two per team, at random)

| # | Question | A good answer contains |
|---|---|---|
| C1 | Why is the restore step above the source copy? | The cache is invalidated downwards; the source changes every commit, restore is slow |
| C2 | Your image has no shell. How do you find out why it crashed at 02:00? | `docker logs`, `docker inspect`, `docker image history`, `docker cp` — and that this was a known cost, recorded in the ADR |
| C3 | `EXPOSE 8080` — what does it do? | Documentation inside the image. `-p` / `ports:` publishes |
| C4 | Why 8080 and not 80? | A non-root process cannot bind a privileged port |
| C5 | You stop the database. Why does `/healthz/live` stay 200? | Liveness asks whether the **process** should be killed; a dependency outage is not a reason to kill it |
| C6 | In the mis-tagged run, why did `/healthz/ready` return **200**? | The only check moved away; a health report with no entries is Healthy. An empty gate is a green gate |
| C7 | What does `condition: service_healthy` change? | Waits for the database to answer a query, not merely for the container to start |
| C8 | Why is there no `healthcheck:` on the API service? | A container healthcheck runs inside the container; this image has no shell or curl. Readiness is watched from outside |
| C9 | Your container starts and logs nothing. No error. What do you check first? | Directory ownership versus the runtime user. The absence of an error is not evidence of success |

> C6 is the discriminator for a Good grade. A student who can explain the 200
> understands why a gate must be verified by making it fail — which is the thing
> this whole course is about.

---

## Part D — Instructor's own review of the submitted image

Run this once per team on the merged branch, outside class.

| # | Check | Result |
|---|---|---|
| D1 | `docker image history --no-trunc` — no secret in any layer | ☐ |
| D2 | `.Config.Env` — no connection string, no password | ☐ |
| D3 | `.Config.User` non-empty and not root | ☐ |
| D4 | `docker compose config` with the password variable unset → refuses to start | ☐ |
| D5 | `docker compose config` — the database port is not published | ☐ |
| D6 | The `ready` endpoint lists at least one check in its response body | ☐ |

> **If D1 or D2 fails, Session 7 starts with a rotation, not a scan.** Tell the
> team the same day. The password in a pushed image is already in every clone;
> rotate first, scrub history second — that order is not negotiable and it is
> examined in Session 7.

---

## Part E — Before Session 7

| # | Follow-up | Done |
|---|---|---|
| E1 | Every team's `docker compose up -d` works from cold on a classroom machine | ☐ |
| E2 | Every team reverted the Block F mis-tag — verify, do not ask | ☐ |
| E3 | Every team has `docs/adr-0001-base-image.md` committed | ☐ |
| E4 | Sabotage cards dealt; peer reviews due before Session 7 | ☐ |
| E5 | Any machine that cannot run Docker is escalated **this week** | ☐ |

> **E5 matters more than it looks.** A machine that cannot run Docker cannot sit
> Periodic Test 1 in Session 8, cannot do the Blue-Green lab in Session 11, and
> cannot run the Game Day in Session 14. Solve it now, not in Session 8.

---

## Carry-forward risks into Session 7

| Risk | Signal today | What it costs later |
|---|---|---|
| A secret is baked into an image | D1/D2 fails | Session 7 opens with an emergency rotation |
| The team never understood the empty-gate result | C6 answered vaguely | Session 7's scanner "runs" and blocks nothing, and they will not notice |
| The base image was chosen by copying another team | ADR has no numbers | Session 7's vulnerability counts will be unexplainable |
| Non-root was faked with `APP_UID` | B4 passed on a screenshot | Container runs as root in Staging from Session 9 onwards |
| Disk full on classroom machines | builds slowing through Block F | Session 7 adds scans and SBOM artifacts and will not fit |

---

## Recording

| Team | B1 | B2 | B3 | B4 | B5 | B6 | B7 | /6 | Lab 3 (5%) | Comment |
|---|---|---|---|---|---|---|---|---|---|---|
| | | | | | | | | | | |
| | | | | | | | | | | |
| | | | | | | | | | | |
