---
title: "Dev and Prod on One Server"
date: 2026-02-05T12:56:00+09:00
draft: false
categories: ["DevOps"]
tags: ["docker", "docker-compose", "linux", "nginx", "vite", "devops"]
summary: "Running dev and prod side by side on one AWS box. Compose project names fixed the collisions, and made it really easy to forget everything Docker doesn't own."
---

On the online judge I had [two identical ARM servers, one per environment](/blog/dev-prod-on-two-arm-servers/). For the academy platform in February 2026 I had one AWS instance. I already developed remotely on it (my laptop couldn't run the stack), and now prod had to live there too:

> I know it is a bad idea but currently I just want to make my dev and prod environment in the same server … So what is it that I just have to be careful of?

Everyone says it's a bad idea, but nobody tells you what exactly to watch out for. Short version of what I found: Compose project names fixed the containers colliding, but dev and prod were still quietly sharing the same database and the same source folder.

## The starting setup

Three Compose files:

- `docker-compose.yml`: the base (backend, frontend, nginx, Redis, Celery worker, Celery beat, Flower).
- An override for dev (Vite dev server, Flask debug, live-mounted code).
- `docker-compose.prod.yml` for prod (Gunicorn, built frontend).

One at a time worked. Both at once broke immediately.

## First collision: container names

The base file had `container_name: flask_backend`, `container_name: vite_frontend` and so on. Docker won't allow two containers with the same name, so whichever started second failed.

Fix is to delete every `container_name` and let Compose generate names from a project name:

```bash
docker compose -p prod -f docker-compose.yml -f docker-compose.prod.yml up -d --build
docker compose -p dev  up -d --build
```

Now the backend is `prod-backend-1` and `dev-backend-1`.

## Why can't two projects use the same container name?

Didn't want to just apply the fix, I wanted to know why it worked. Asked how Linux namespaces work, then about what looked like a contradiction:

> If you are separating the namespace, isn't it okay to have the same container name?

Turns out "namespace" means two different things here:

- Linux namespaces are kernel isolation. A container is a process started with `clone()` and flags like `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWNS`, so it gets its own process tree, network stack and mounts. That's what the process inside sees.
- A container's *name* is an identifier in the Docker daemon, which keeps one flat host-wide table of containers. Two entries can't share a key.

Then I compared it to folders:

> it seems to me like creating a new folder. You can't have files with same names in a single folder but you can have files with same names in different folders. To me it seems like that is what -p is doing, then why isn't it okay to have same container name?

That's actually right, and it's how Compose works. The daemon has no idea what a "project" is. Compose fakes it on the client side — prefixes names (`dev-backend-1`) and labels each container with its project. So the service name `backend` can repeat across projects, like a relative path. `container_name` is like an absolute path: "ignore the project, use exactly this global name", which is exactly what collides.

At one point the answers mixed these up. I got told the same name "would work" in both projects, then that it would crash. Both were true, about different names:

| Name | Example | Scope | Can repeat across projects? |
|---|---|---|---|
| Service name | `backend` | Inside one Compose project | Yes |
| Container name | `dev-backend-1` (or a hardcoded `flask_backend`) | The whole Docker host | No |

Same thing got me later. `docker compose -p dev down dev-celery-worker` failed because Compose commands take service names (`celery-worker`), not container names.

## Second collision: ports, and lists that merge

Names fixed, dev still wouldn't start. Two things going on.

Lists merge across files. I'd put `ports:` in the base file and different ports in the dev override, thinking the override replaces them. It doesn't — Compose merges list entries like ports, so dev tried to bind the base file's port *and* its own. So anything that has to differ between environments (host ports especially) just doesn't go in the base file. Prod gets 80/443, dev gets 8080 and friends.

Then a ghost from the old naming. Even with clean files, dev said 8080 was taken, and `netstat` showed nothing obvious. `docker ps -a` found it: an old `vite_frontend` container from before all this, under the hardcoded name. Belonged to no project, so neither `-p dev` nor `-p prod` could see it.

## Which file is the default

I renamed the dev file to `docker-compose.override.yml`, which Compose loads automatically. So dev is the short command, `docker compose -p dev up`, and prod has to list its files with `-f`, which disables the auto override.

That's the opposite of what I did on the [two-server setup](/blog/dev-prod-on-two-arm-servers/), where the default file was prod so a careless `up` starts the safe stack. Here the careless command is the dangerous one: `docker compose -p prod up` without `-f` loads dev settings into the prod project. So I stopped typing prod commands by hand and made a Makefile:

```makefile
CMD_DEV  = docker compose -p dev
CMD_PROD = docker compose -p prod -f docker-compose.yml -f docker-compose.prod.yml

up:        ; $(CMD_DEV) up -d --build
up-prod:   ; $(CMD_PROD) up -d --build
down:      ; $(CMD_DEV) down
down-prod: ; $(CMD_PROD) down
```

`make down` only touches dev. Stopping prod needs its own explicitly named target.

## Before that: a dev frontend that was secretly prod

Two weeks earlier, in January, the dev stack didn't work at all — `502 Bad Gateway`. Instead of guessing I traced the request one hop at a time:

> when I visit localhost:28136 … [it] redirects me to localhost:80 using [the] tunnel … then it gets redirected to localhost:80 in the nginx container. The nginx container then tries to call the frontend, but frontend has 8080:80 but the nginx config in docker-dev.conf has frontend as frontend:3000?

Yep that was it. Dev nginx expected the Vite dev server on 3000, but the frontend container was running the *production* image — a multi-stage Dockerfile whose last stage is nginx serving built files on 80. In dev the override only added a volume and a port, so the container just ran that last stage. Nobody listening on 3000, hence 502.

Fix is to only build part of the Dockerfile for dev: `target:` tells Compose to stop at a named stage. But the existing first stage also ran `npm run build`, so I asked:

> don't you need to divide the stage into 3 parts? 1 for npm install, 1 for npm build and 1 for copying? because if you target the 1st stage, good that npm install but there is also npm build, then the docker image would be significantly larger for no reason, containing the static files

Yes. Dev needs dependencies but not a prod bundle, so split the stages by purpose:

```dockerfile
FROM node:20-alpine AS deps          # shared: install dependencies once
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM deps AS dev                     # dev target: source is bind-mounted, nothing built
CMD ["npm", "run", "dev", "--", "--host"]

FROM deps AS build                   # prod only: compile the bundle
COPY . .
RUN npm run build

FROM nginx:alpine AS prod            # prod only: serve static files
COPY --from=build /app/dist /usr/share/nginx/html
```

Dev uses `target: dev`, prod builds to the end. The `--host` matters — without it Vite listens on the container's own `127.0.0.1` and nginx in another container gets refused. Same 502 again.

One more thing that looks backwards until you get how mounts work. Dev bind-mounts `./frontend:/app` for live editing, and a bind mount *replaces* what's at that path. My local folder has no `node_modules` (and shouldn't, packages built on another OS can have the wrong native binaries), so the mount hides the modules installed in the image. Standard fix is an anonymous volume that keeps that one directory from the image:

```yaml
volumes:
  - ./frontend:/app
  - /app/node_modules
```

Catch: that volume only gets filled from the image when it's first created. Add a package, rebuild, and the container can still see the old modules until the volume is recreated (`docker compose up -V` renews anonymous volumes).

## Where both bugs started: July 2025

Both problems were already there six months before, when I was first learning Docker. In July 2025 I had Gemini explain multi-stage builds and write my first frontend Dockerfile. Re-reading that conversation, I asked the right question and accepted the wrong answer:

> but the dockerfile is for development. Do I need to maintain dockerfile for deveopment and production separately?

Answer was no, one multi-stage file serves both because the builder stage "acts as your development environment". Only true if something tells Compose to stop at that stage... and nothing did. Without `target:`, a build always runs to the last stage, which was nginx. So the dev container ran prod. That's the 502 above, just waiting.

The compose file I pasted in that same conversation already had `container_name: flask_backend` in it too. The exact name that collided in February.

Two more things from that conversation:

- The example Dockerfile that started it all didn't even work. It was a Python multi-stage example: `pip install -r requirements.txt` in a builder stage, then `COPY --from=builder /app /app` into a fresh image. But pip installs into the interpreter's `site-packages`, not `/app`. All that stage copied was `requirements.txt`, so the final image had none of the dependencies, while the explanation said it copied "your installed Python packages." Should've run the example before learning from it. (Multi-stage does pay off for Python when the builder needs compilers the runtime doesn't, and you copy a virtualenv or built wheels across explicitly.)
- `node:lts-alpine` keeps moving. I asked "should I use the exact version? or lts or what? what is the standard" and went with `lts`. But `lts` silently jumps to the next major when Node's LTS line changes, so the same Dockerfile can build on Node 20 one month and 22 the next. Pinning the major the project actually uses (`node:22-alpine`, matching `@types/node` in my `package.json`) gets patch updates without surprise major upgrades.

My questions back then held up better than the answers. I asked if two stages' storage would just add up to one big image ("won't it be the same as single container with single image?"), and what the "unoptimized" version of the dependency-caching trick was. Those made layers and the build cache click. But I didn't know enough yet to check the answers, and two of them became the bugs I spent January and February fixing.

## Reaching the dev site

Two networking things while wiring dev.

Host ports vs container ports. First I pointed dev nginx at `frontend:3001` and `backend:5001`, the ports I published to the host. Wrong — nginx is inside the Docker network and talks to containers directly on their internal ports (`3000`, `5000`). Published ports are for stuff outside Docker, like me.

Vite hot reload behind a proxy. The browser kept failing to reach `localhost:3000`. Vite tells the browser where to open its HMR WebSocket, and it was advertising its internal port. Either set `server.hmr.clientPort` to the port the browser actually uses, or what I suggested: forward that port straight over SSH and skip nginx for HMR:

```bash
ssh -L 8080:localhost:8080 -L 3001:localhost:3001 ubuntu@<server>
```

That way dev ports never need opening in the firewall, they only exist through the tunnel.

## What `-p` isolates, and what it doesn't

After all that everything ran side by side, and it'd be easy to call it isolated and stop. Re-reading my config for this post though, this is the part that worries me most.

A Compose project scopes what Docker owns. Does nothing for what lives outside Docker:

| Resource | Separated by `-p`? | In my setup |
|---|---|---|
| Container names | Yes (prefix) | Separate, once `container_name` was gone |
| Networks | Yes (`dev_default` vs `prod_default`) | Separate |
| Named volumes | Yes (prefixed) | Separate, e.g. each project's Redis data |
| Host ports | No, host-global | Kept apart by hand |
| Bind-mounted folders | No | Both projects mounted `./backend:/app` from the same checkout |
| The database | No | Both overrides loaded the same `env.local`, the same RDS database |
| Outside side effects | No | Dev's Celery worker and beat ran against that same database |
| CPU / RAM | No | Shared box |

Two rows are the real risk:

- Same database. Dev code ran against prod data. A migration, a test script, or a buggy feature in dev would change real students' data. Fix is a separate dev DB (or at least a separate schema) and an `env.dev` that *can't* point at prod.
- Same source folder. Both projects ran from one directory and mounted the same `./backend`. Editing a file "in dev" edited the code prod would load on its next restart. Separate checkouts (like `~/app-prod` deployed from a git tag) make "what's in prod" a deliberate thing, not whatever I last saved.

And scheduled jobs. Dev ran its own Celery beat. Separate Redis but shared DB means each environment's scheduler runs the same periodic tasks against the same data, twice. Anything a task does outside its container — writing rows, sending messages — happens once per environment.

So that's what "dev and prod on one server is a bad idea" actually means. Not containers colliding, `-p` fixes that. It's everything they share without anyone deciding they should.

## If I did it again

Separate the data before the containers: dev DB and its own env file first. Name collisions fail loudly, a shared DB fails silently. Prod runs from its own checkout at a known commit, and dev's Celery beat stays off whenever it shares anything with prod.

Also be more careful with cleanup commands. To clear that ghost container I was told to run `docker rm -f $(docker ps -a -q)`, which removes every container on the box, prod included. `docker rm -f vite_frontend` was all it needed. On a shared server, narrowest command that does the job.

And keep checking answers against actual output. One answer said `docker compose down` doesn't take service names. My own error — "services [dev-celery-worker] not running in project dev" — showed it treated the argument as a service name. Real problem was I'd passed a container name.
