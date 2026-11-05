---
title: "What the Upstream Image Was Hiding"
date: 2025-12-05T12:00:00+09:00
draft: false
categories: ["Dev"]
tags: ["docker", "nginx", "arm64", "oracle-cloud", "cloudflare", "debugging"]
summary: "Splitting an all-in-one Docker image into separate services broke a bunch of assumptions I didn't know existed. Every \"how did this work before?\" had the same answer."
---

The upstream online judge ships its backend as one image that does everything: Python, nginx, supervisor and a prebuilt frontend zip, all in one container. I wanted hot reload, so I split it into separate frontend and backend services. Then stuff kept breaking, and every time I asked "wait how did this work before?", the answer was sitting right there in the upstream Dockerfile.

[Part 1](/blog/dev-prod-on-two-arm-servers/) is the dev/staging/prod layout this all runs on.

## Before and after

```text
UPSTREAM (one container)                 MINE (split for development)
┌────────── oj-backend ──────────┐       ┌── oj-frontend ──┐   ┌─── oj-backend ───┐
│ supervisor                     │       │ nginx:alpine    │   │ supervisor       │
│  ├─ nginx  → serves dist.zip   │       │ built Vue files │──▶│  └─ gunicorn     │
│  └─ gunicorn @ 127.0.0.1:8000  │       │ :80 / :443      │   │     0.0.0.0:8000 │
│ :8000 published as :80         │       └─────────────────┘   └──────────────────┘
└────────────────────────────────┘        (dev: Vue dev server with hot reload)
```

Left side: nginx talks to gunicorn inside one container so `127.0.0.1` is enough, and the frontend is a zip somebody else built. Right side: separate containers on a Docker network, and I'm building the frontend myself for the first time. Pretty much every problem below comes from that one change.

## Frontend container that keeps exiting with code 0

Prod frontend container restarted forever with exit code 0. Clean exit, over and over.

Looked inside (with Cursor's help): image was `node:16`, CMD was just `node`, no nginx anywhere. Node starts with no script, has nothing to do, exits successfully. Docker restarts it. Repeat.

At some point a "fix" had been added to the compose file:

```yaml
command: >
  sh -c "
  apt-get update -qq &&
  apt-get install -y -qq nginx &&
  useradd -r -s /bin/false nginx || true &&
  sed -i 's|proxy_pass http://oj-backend:8080;|proxy_pass http://oj-backend:8000;|g' /app/deploy/nginx.conf &&
  nginx -c /app/deploy/nginx.conf -g 'daemon off;'
  "
```

That installs nginx from the internet *every time the container starts*, then patches the backend port with `sed`. It worked, but the site literally couldn't boot without a working package mirror.

Real fix is a multi-stage build: build with Node, serve with nginx, throw Node away.

```dockerfile
FROM node:16-bookworm AS builder
WORKDIR /app
COPY package.json yarn.lock ./
RUN npm install
COPY . .
RUN npm run build:dll && npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY deploy/nginx.conf /etc/nginx/nginx.conf
```

Port goes in `nginx.conf` itself (`proxy_pass http://oj-backend:8000;`). Also had to remove the config's own `daemon off;`, because `nginx:alpine` already runs it in the foreground and nginx refuses a duplicate directive.

## There was never a frontend container

So why did the original never need a frontend service at all? Upstream backend Dockerfile:

```dockerfile
FROM alpine:3.19 AS downloader
RUN wget https://github.com/QingdaoU/OnlineJudgeFE/releases/download/oj_2.7.5/dist.zip && unzip dist.zip
...
RUN apk add ... supervisor openssl nginx ...
COPY --from=downloader /app/dist/ /app/dist/
```

The backend image downloads a prebuilt frontend, installs nginx inside the backend, and runs nginx + gunicorn together under supervisor. One container, one port, no CORS questions. For distribution that makes sense, anyone can run the whole site with one image.

For development it's painful though, you can't touch the Vue code without rebuilding and restarting Python too. Splitting was right for me. I just didn't realise splitting meant taking over every job the monolith was quietly doing.

## `npm install` had never actually run

Dev frontend died during install:

```text
npm ERR! command git --no-replace-objects ls-remote ssh://git@github.com/itargaryen/simple-hotkeys.git
npm ERR! fatal: Could not read from remote repository.
```

A package install that needs GitHub over SSH?? At this point I honestly didn't understand what npm and yarn were doing, so I stopped and figured that out first before trying more fixes.

`simple-hotkeys` isn't in `package.json`. It's a dependency of the editor package `tar-simditor`, declared as `github:itargaryen/simple-hotkeys`, so it comes from a git repo, not the npm registry.

Gemini's fixes kept assuming files that didn't exist:

- "Edit `package-lock.json`." There is no `package-lock.json`, the project only has `yarn.lock`.
- "Change the `ssh://` line in `yarn.lock`." There isn't one. `yarn.lock` resolves it over HTTPS from `codeload.github.com`. It's npm, ignoring `yarn.lock`, that goes over SSH.

And it "worked before" because nobody had ever run `npm install` on my server. The upstream image downloaded `dist.zip`, which someone else built on a machine that had GitHub SSH keys. I was doing the real build for the first time and hitting the error the zip had been hiding.

Fix: it's a yarn project, so use yarn, and force GitHub SSH URLs onto HTTPS.

```yaml
command: >
  sh -c "git config --global url.'https://github.com/'.insteadOf git@github.com: &&
  git config --global url.'https://github.com/'.insteadOf ssh://git@github.com/ &&
  yarn install &&
  npm run build:dll &&
  npm run dev"
```

Two smaller dev-only ones right after:

- `EACCES` on a build file. An earlier container running as root had written into my mounted folder. The suggested fix was `chmod -R 777`, and when I asked if that's dangerous it got taken back. Right fix is `sudo chown -R $USER:$USER ./frontend`.
- `Exited with code 3` from `opn`. Vue dev config had `autoOpenBrowser: true`, a vue-cli default that makes sense on a laptop. On a headless server there's no browser to open so the dev server just crashed. Set it to `false`.

## Two things that assumed a git checkout

Two failures from the first prod deploy, a week before, are the same kind of thing: upstream code assuming it's being built from its own git repo.

First was a build that called git:

```text
Error: Command failed: git rev-parse HEAD
fatal: not a git repository (or any of the parent directories): .git
    at Object.<anonymous> (/frontend/config/dev.env.js:2:39)
```

The webpack config stamps a version string at build time by shelling out to `git rev-parse HEAD`. Always works on the maintainers' machines. Inside a Docker build it doesn't, because `.git` is excluded by `.dockerignore` (as it should be). The first suggested fix guessed what my config "probably looks like" and guessed wrong. When I showed the real file, turns out the prod config imports the dev one, so the git call in `dev.env.js` runs on every build. Fix is to compute the hash outside and pass it in:

```dockerfile
ARG GIT_COMMIT_HASH=00000
ENV GIT_COMMIT_HASH=$GIT_COMMIT_HASH
```

```js
// dev.env.js: use what the build was given; never shell out to git
let commit = process.env.GIT_COMMIT_HASH || '00000'
```

```bash
docker build --build-arg GIT_COMMIT_HASH=$(git rev-parse --short HEAD) ./frontend
```

Second was three repos inside one. When I first put the project in git, `git add .` said:

```text
warning: adding embedded git repository: backend
warning: adding embedded git repository: frontend
warning: adding embedded git repository: judge-server
```

I'd cloned the three upstream projects into one folder and each still had its own `.git`. Git doesn't track a repo's files from inside another repo, it just records a pointer, so anyone cloning mine (including the prod server) would get three empty folders. Since I was changing all three a lot anyway, I just removed the inner `.git` folders and made it one repo. Submodules are for when you want to keep following upstream unchanged, which wasn't me.

## The "cleaner" Dockerfile

Backend dev container failed with `supervisord: not found` and `chown: unknown user/group server:spj`.

The backend Dockerfile had been rewritten to be "cleaner", and the cleanup dropped packages from the original `apk add` line — `supervisor`, `shadow` (for `useradd`/`groupadd`), `nginx`. The entrypoint still called all of them.

Same thing as the judge server in [the ARM build post](/blog/online-judge-on-oracle-arm/). The upstream Dockerfile looks messy because it's actually doing stuff. Diff against it before "cleaning" it.

## 502 Bad Gateway

Staging returned nginx's `502 Bad Gateway` and the nginx logs said nothing useful. So I tested each layer from inside the frontend container:

```bash
docker compose exec oj-frontend ping -c 2 oj-backend
# ok: DNS and the Docker network are fine

docker compose exec oj-frontend sh -c "apk add --no-cache curl && curl -v http://oj-backend:8000/"
# connect to 172.20.0.5 port 8000 failed: Connection refused
```

Host reachable, port refused, so the backend wasn't listening on the interface nginx was calling. In the monolith nginx and gunicorn shared a container, so binding gunicorn to `127.0.0.1` was fine. Across containers it has to be `0.0.0.0`.

The source already said `--bind 0.0.0.0:8000`. The image on Docker Hub didn't — the build had reused a cached layer with the old config. `docker build --no-cache`, push, pull, connected.

Checking what was actually in the image got me too. `docker run --rm <image> cat /app/deploy/supervisord.conf` didn't print the file, because the image has an `ENTRYPOINT`, so `cat …` got handed to the entrypoint script, which tried to boot the whole backend with no DB configured. To look inside an image you override the entrypoint:

```bash
docker run --rm --entrypoint cat <image> /app/deploy/supervisord.conf
```

### Python 3.12 removed something the old code needed

Those accidental boot logs did show a real problem: `AttributeError: module 'ssl' has no attribute 'wrap_socket'`, from `raven`, the super old deprecated Sentry client. Image is `python:3.12-alpine`, and 3.12 removed `ssl.wrap_socket`. I disabled the raven app and the Sentry log handler to get it booting, so no error tracking for now.

Two more, both from not knowing exactly what's inside the image:

- Missing migrations on staging. The migration files were on the dev disk, but the image was built before they got generated. An image is a snapshot of the folder at build time.
- Admin panel with no CSS. `collectstatic` needed a `STATIC_ROOT`, pointed it at `DATA_DIR/public` which nginx already served.

ARM side had nothing to fix. Building on arm64 for arm64 doesn't need `buildx`, and Docker Hub's "only the available single-platform image was pushed" just means it's arm64-only. Which is correct, everything that pulls it is arm64.

## HTTPS behind Cloudflare

Domain is on Cloudflare. First option I got offered was Flexible mode: HTTPS from browser to Cloudflare, plain HTTP from Cloudflare to my server, no server config.

> but why not use full for encryption mode and not flexible

Flexible means login passwords cross the internet unencrypted between Cloudflare and Oracle, and with Django forcing HTTPS it's also a classic redirect loop. Went with Full (Strict) and a Cloudflare Origin Certificate — free, long-lived, only trusted by Cloudflare. Came as `.pem`, nginx reads that fine.

Certs aren't baked into the image. The image has the nginx config, and the keys get mounted read-only at runtime:

```yaml
oj-frontend:
  image: <hub>/onlinejudge-frontend:latest
  ports: ["80:80", "443:443"]
  volumes:
    - ./ssl:/etc/nginx/ssl:ro
```

Then Cloudflare said 523 Origin is Unreachable. Oracle has two firewalls: the Security List in the console and iptables on the Ubuntu image, which drops new connections by default. I'd only opened the first one.

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save
```

Last bug was a leftover. Nginx couldn't load the cert, and while checking the mount I read through the staging compose file:

> you dont' need this ??

It still had the runtime `apt-get install nginx` block from the very first "fix". The frontend image was `nginx:alpine` by now, which doesn't even have `apt-get`. Deleted the whole `command:` block and the image started nginx the way it's supposed to.

## Looking back

Almost every bug here was a job the all-in-one image had been doing quietly — serving the frontend, keeping nginx and gunicorn on the same localhost, shipping a frontend someone else built, installing what the entrypoint needs. When I split it I took over all of those without realising. Next time I'd read the upstream Dockerfile and entrypoint line by line *before* changing the architecture, not after each thing breaks.

Honestly the annoying part wasn't any one bug, it was that they all had the same answer. "How did this work before?" → it's in the upstream Dockerfile. Every time.

Testing one layer at a time helped a lot too. Ping, then curl, then config, and a vague 502 turned into a one-line fix.
