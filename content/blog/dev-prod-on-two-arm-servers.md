---
title: "Dev, Staging, Prod on Two Oracle ARM Boxes"
date: 2025-12-01T11:08:00+09:00
draft: false
categories: ["Dev"]
tags: ["docker", "docker-compose", "arm64", "oracle-cloud", "devops"]
summary: "Setting up a native arm64 dev → staging → prod flow on Oracle's free Ampere A1, with an x86 desktop and two identical servers."
---

My desktop is x86 and both servers are ARM, so anything I build locally is built for the wrong CPU. That one thing ended up shaping the whole setup: I develop *on* the dev server over SSH, and that box runs a hot-reload dev stack next to a staging stack that pulls the exact same images prod does.

[Part 2](/blog/what-the-upstream-image-was-hiding/) is everything that broke along the way.

The app is the online judge from [the ARM build post](/blog/online-judge-on-oracle-arm/): Django backend, Vue frontend, C/Python judge server, Postgres and Redis, all in Docker Compose. Oracle free tier gives 4 OCPU / 24GB of Ampere A1, which I split into two identical 2 OCPU / 12GB boxes — one prod, one dev ([why that split](/blog/before-the-first-container/)).

Most of this came out of about five days of going back and forth with Gemini (3.1 Pro at the time) in December 2025.

## What it ended up looking like

```text
   my desktop (x86)
   VS Code Remote-SSH ── just the screen and keyboard
          │
          ▼
┌─────────── dev box (arm64, 2 OCPU / 12GB) ────────────┐
│                                                        │
│  ~/onlinejudge-dev          ~/onlinejudge-staging      │
│  docker-compose.dev.yml     docker-compose.yml         │
│  build: from source         image: <hub>/...:latest    │
│  ./backend:/app (live)      no source used             │
│  ports 8000 / 8088          ports 80 / 443             │
│  dev_* volumes              prod_* volumes             │
│  oj-dev-network             oj-network                 │
│        │                          ▲                    │
└────────┼──────────────────────────┼────────────────────┘
         │ docker build / push      │ docker compose pull
         ▼                          │
    ┌─────────────┐                 │
    │ Docker Hub  │─────────────────┤
    │ arm64 only  │                 │
    └─────────────┘                 ▼
                       ┌──── prod box (arm64) ────┐
                       │ ~/onlinejudge            │
                       │ docker-compose.yml       │
                       │ ports 80 / 443 only      │
                       └──────────────────────────┘
```

## Developing on the server

Two options: code locally and use the dev server for testing, or just code on the dev server itself.

Desktop is x86, servers are ARM, so that kinda decides it. Anything I build locally — image or compiled binary — is for the wrong CPU, and the way around that is QEMU (already ruled out for the judge, too slow) or cross-compiling.

I'd also done things in the wrong order. Prod came first: I got the judge running on the prod box by building and tweaking directly on the server, and only then needed somewhere to actually develop:

> I already have the production server running … But I deployed first. Now I have to make the development environment.

Developing on my laptop meant cross-arch builds again. So:

> do you think it is a good idea to just use the development server as the actual development environment?

That's what the dev box became. VS Code connects over SSH, and the code, containers and DB all live on a machine with the same OS, arch and size as prod. If it runs on dev, it runs on prod. Only downside is no offline work.

Also simplified builds. The suggested commands built for both `linux/amd64` and `linux/arm64`, and I asked — I'm only ever using A1 servers, why build amd64 at all? No reason. Every machine that'll ever pull these images is ARM, so building for Intel just doubles build time for an image nobody runs.

## Dev and staging on the same box

Prod already worked like this: build images → push to Docker Hub → prod pulls → `docker compose up`. So the dev box runs two different things:

- Dev builds from source and bind-mounts it, so saving a file changes what's running.
- Staging pulls the same Docker Hub images prod will pull. It's the check that what I'm about to ship actually works.

Two stacks on one host can't share ports or data. Staging takes 80/443 like prod, dev uses 8000 for backend and 8088 for frontend. Each has its own named volumes and network.

They're in separate folders too. Compose uses the folder name as the project name, so `onlinejudge-dev` and `onlinejudge-staging` get separate containers, networks and volumes even with identical service names. So the folder split isn't just being tidy, it's what actually keeps them apart.

## Two files, three environments

Asked how many compose files I'd need and got "3 active docker-compose configurations." Next answer said keep two files in git.

> but you said 3 active docker compose now you are saying 2..?

Both were half right:

| Environment | Server | Folder | File |
|---|---|---|---|
| Production | prod box | `~/onlinejudge` | `docker-compose.yml` |
| Staging | dev box | `~/onlinejudge-staging` | `docker-compose.yml` |
| Development | dev box | `~/onlinejudge-dev` | `docker-compose.dev.yml` |

Prod and staging use the same file on purpose. Staging is only worth anything if it's the same as prod.

The default filename goes to prod. Plain `docker compose up` should always start the safe stable stack, so typing it in the wrong folder can't bring up debug mode with live source. Dev you have to ask for:

```bash
docker compose -f docker-compose.dev.yml up -d
```

Both files are in git. The dev file is just ports, build contexts and mounts, passwords as `${VARIABLES}`. Only `.env` stays out. I also keep both files in both folders — deleting the "unused" one makes every `git pull` complain about local changes, and a compose file does nothing until you run it anyway.

## The network is the wall, not the password

The setup guide said use different passwords for staging and prod. Gemini's reason was "fat-finger protection": you go to wipe staging, you're actually on prod, and the different password saves you.

> but then I don't really relate to the fat finger protection. Because it is a random 32 long text, you never type it, you just use your env variable to enter into the db.

Next it tried a GUI client like DBeaver pointed at the wrong host. I don't use one, I use `docker exec … psql`. Then config drift, like copying prod's `REDIS_URL` into the dev `.env`. That one got me thinking about what can actually reach what:

> actually.. even if you use the same ip and same password in the development, you will never be able to access the database in the production server because I only exposed port 80 and 443.

That's the real protection. Postgres and Redis have no `ports:` in the prod file, so the host isn't even listening on 5432 or 6379, and dev and prod are on separate Docker networks. There's no route to the prod DB to type a password into. Different secrets are still worth it — if staging's `.env` leaks it shouldn't unlock prod — but that's about limiting damage, not typos.

The secrets are Base62 (`a-zA-Z0-9`), not Base64, because the Redis password goes straight into a URL:

```text
REDIS_URL=redis://:${REDIS_PASSWORD}@oj-redis:6379
```

A `/` or `+` from Base64 breaks URL parsing or needs encoding. Base62 never does.

```bash
python3 -c "import secrets,string; print(''.join(secrets.choice(string.ascii_letters+string.digits) for _ in range(32)))"
```

## The dev loop

What changed decides what you run:

| What changed | Command | Time |
|---|---|---|
| Code (`.py`, `.vue`) | nothing (hot reload) | instant |
| Config (`docker-compose*.yml`, `.env`) | `up -d` | seconds |
| Dependencies (`Dockerfile`, `package.json`, `requirements.txt`) | `up -d --build` | a minute or more on 2 OCPU |

Gemini's habit was `--build` every time "to be safe." On a 2-core ARM box that turns every config tweak into a minute+ wait, so I stopped doing that.

The one that cost me time: `docker compose restart` does NOT re-read the YAML. It just reboots the existing container with the old config. Only `up -d` (or `up -d --force-recreate <service>`) applies edits.

Two smaller things:

- `./backend:/app` is the code, `./data/backend:/data` is uploads and config. Backend crashed looking for `/app/data/config/secret.key` until I set `DATA_DIR=/data` so it knew where the second mount was.
- Only 80/443 are open on Oracle and I didn't want to open more for a dev server. So VS Code's Ports tab tunnels 8088 to my own `localhost` over the SSH connection. No firewall changes.

## Shipping: two pipes

The question that finally made it click:

> everytime I change the docker-compose file in the dev folder, don't I have to push and pull and restart again?

No. Code and config travel separately:

| What | Goes through | On the server |
|---|---|---|
| Code (Python, Vue) | `docker build` → `docker push` | `docker compose pull` |
| Config (compose files) | `git commit` → `git push` | `git pull` |
| Secrets (`.env`, TLS keys) | neither | stay on the server |

A deploy is now just:

```bash
git pull && docker compose pull && docker compose up -d
```

Getting there meant re-cloning staging, which started out as a hand-made `mkdir` folder with copy-pasted files and wasn't even a git repo. I also pushed on the obvious problem:

> wait I dont' get the process. Because if you git pull or do git clone, you are cloning all the source code and stuff. You dont' want that.

Right, staging runs from images so the source on its disk never gets used. The alternative is fetching only the compose file. For now I'm fine with the unused source for one-command config updates, but it's on the list below.

## What's still wrong

Works, and way better than where it started. Still not what a real production pipeline looks like:

- Images get built by hand on the dev box, so releases depend on whatever's on that disk (that's exactly how I once shipped an image without its migration files). Should be built in CI on every push to `main` — GitHub Actions has arm64 runners so it can stay native. ([Planning that](/blog/designing-ci-cd-for-two-arm-servers/) came next.)
- Everything is `:latest`, so no way to roll back to "yesterday's image." Tagging with the commit SHA fixes that.
- Staging shares the dev box. A heavy dev build fights staging for the same 2 cores, so staging isn't really a clean copy of prod.
- No healthchecks. `depends_on` only waits for the Postgres container to *start*, not to accept connections. A healthcheck + `condition: service_healthy` would fix it.
- No error tracking. Disabled the old Sentry client to get the backend booting on Python 3.12 ([part 2](/blog/what-the-upstream-image-was-hiding/)) and never replaced it.
- Django 3.2 is past EOL. Upstream pins it, and upgrading is its own project.
- Staging carries source it never uses.

If I rebuilt this tomorrow, CI-built SHA-tagged images would come first. Most of the debugging in part 2 came from not knowing exactly what was inside the image I was running.

Gemini was useful for thinking out loud, less for details ("3" then "2", typo scenarios that didn't match how I actually work). Most of the good decisions came from asking why one more time and checking against my own setup. If you're doing your first dev/prod split, I'd say make staging pull the exact same image as prod, and make the lazy default command the safe one.
