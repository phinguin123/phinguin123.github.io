---
title: "Building an Online Judge on Oracle ARM"
date: 2025-11-27T10:00:00+09:00
draft: false
categories: ["Dev"]
tags: ["docker", "arm64", "oracle-cloud", "online-judge"]
summary: "Building QingdaoU/OnlineJudge natively for arm64 on Oracle's free tier, and where Gemini got it wrong."
---

Side project — an online judge for IB CS practice problems. Server is Oracle's free tier Ampere A1 (4 cores, 24GB RAM, free, 갓라클!!). Problem is basically everything QingdaoU/OnlineJudge ships is built for x86. ARM doesn't run that.

One way is to use QEMU but obviously since it is an emulator it is too heavy. A judge that compiles and runs student code under a time limit doesn't have room to also be emulating a different CPU architecture while it does it, so skipped that. Meant building the whole stack — Django backend, Vue frontend, the C/Python judge server — natively for arm64 instead of just pulling images.

Did most of this with Gemini. But I guess because it was only 3.1 Pro at that time (which kinda sucks a lot..), it was quite misleading many of the times... So here is my experience with it building a cross platform service.

## Problem 1: Frontend has no Dockerfile

`OnlineJudgeFE` just doesn't have one. Gemini's first answer was that the prebuilt frontend image is x86-specific so I'd need my own for ARM. Didn't really buy that — it's compiled Vue output, static JS/CSS, that's not an architecture thing the way a compiled binary is. Whatever the actual reason the repo ships without one, I built it myself anyway since I wanted to control the build instead of depending on a downloaded clone. `node:16` stage, `npm run build:dll` then `npm run build`, copied the output into the backend image later.

```dockerfile
# frontend build stage, bookworm instead of buster (more on that below)
FROM node:16-bookworm AS frontend-builder
WORKDIR /frontend
COPY frontend/package.json frontend/yarn.lock ./
RUN npm install
COPY frontend/ .
ENV NODE_ENV=production
RUN npm run build:dll
RUN npm run build
```

```dockerfile
# then in the actual backend stage
COPY --from=frontend-builder /frontend/dist /app/dist/
```

## Problem 2: Judge server was missing half of what it needs

First draft of the judge-server Dockerfile just installed Python and gcc. I went and grabbed the actual upstream Dockerfile off GitHub, pasted it in, told Gemini to recheck itself against the real one. Turns out it missed a lot — needs a compiler for every language submissions can be in (ended up with `openjdk-17-jdk`, `golang-go`, `gcc`, `g++`, `nodejs`), plus three specific Linux users the C sandbox expects by exact UID — `compiler` (901), `code` (902), `spj` (903) — and the container has to run `privileged: true` or `libseccomp` just doesn't work (which is very important as a security researcher). None of that was in the first version. If I hadn't checked, every submission would've failed with permission errors and zero obvious reason why. This I guess is the reason one can't just vibe code everything on their own.

```dockerfile
RUN apt-get update && apt-get install -y \
    python3-minimal python3-venv libpython3-stdlib libpython3-dev \
    golang-go \
    openjdk-17-jdk \
    gcc g++ \
    nodejs npm \
    strace \
    libseccomp-dev \
    curl gnupg \
    && rm -rf /var/lib/apt/lists/*

# for C sandbox
RUN useradd -u 901 -r -s /sbin/nologin -M compiler && \
    useradd -u 902 -r -s /sbin/nologin -M code && \
    useradd -u 903 -r -s /sbin/nologin -M -G code spj
```

```yaml
# docker-compose.yml
services:
  oj-judge-server:
    build:
      context: ./judge-server
    privileged: true # libseccomp needs this, especially on ARM
    environment:
      SERVICE_URL: http://oj-judge-server:8080
      BACKEND_URL: http://oj-backend:8000/api/judge_server_heartbeat/
      TOKEN: "CHANGE_THIS_SECRET"
      cpu_core: 4
```

## Problem 3: requirements.txt that doesn't exist, twice

Build failed looking for `requirements.txt` in the judge-server context. Gemini's answer: your clone is probably broken, re-clone it. I'd already looked — there's no `requirements.txt` in that repo and dependencies are hardcoded into the install step instead. Prompted again and gave me a "fix" that still had `COPY requirements.txt .` in it somewhere, failed the exact same way again.

```dockerfile
# this "fix" still failed, same error, different line number
COPY requirements.txt .
RUN python3 -m venv .venv && \
    .venv/bin/pip install --no-cache-dir -r requirements.txt && \
    .venv/bin/pip install /app/dist/*.whl
```

Only the second time did it actually drop the file reference and just hardcode the real list — `flask`, `gunicorn`, `idna`, `psutil`, `requests`. Which, to be fair, is exactly what the official Dockerfile installs. Just took insisting twice before it stopped assuming I'd messed up the clone.

```dockerfile
# what actually worked, no requirements.txt involved
RUN python3 -m venv .venv && \
    .venv/bin/pip install --no-cache-dir flask gunicorn idna psutil requests && \
    .venv/bin/pip install /app/dist/*.whl
```

## Found the setup.py thing before it did

Building the Python bindings for the C sandbox used to be `python3 setup.py bdist_wheel`. Failed — no setup.py. Before asking anything I just looked at what was actually in that folder: `_judger`, `.gitignore`, `pyproject.toml`, `setup.cfg`. No setup.py. Figured the project moved to `pyproject.toml` packaging and said so. Gemini confirmed and gave me the current command for it:

```bash
# old, no longer works once a project drops setup.py for pyproject.toml
python3 setup.py bdist_wheel

# current
pip install build
python3 -m build
```

Reads `pyproject.toml`, spits out the same `.whl` the old command used to.

## Buster's dead so don't use

Separate from all of the above — backend base image was `python:3.9-slim-buster`, `apt-get update` started 404ing on every mirror. Buster got moved to Debian's EOL archive. Nothing to catch here, just an old base image.

```text
E: The repository 'http://deb.debian.org/debian buster Release' does not have a Release file.
E: The repository 'http://deb.debian.org/debian-security buster/updates Release' does not have a Release file.
```

```dockerfile
# before
FROM python:3.9-slim-buster

# after
FROM python:3.9-slim-bookworm
```

Swapped to `python:3.9-slim-bookworm` and it was fine.

## Moved the build context, broke every COPY

Needed the backend Dockerfile to grab the frontend's build output, so I changed the Compose build context from `backend/` to the repo root. Broke literally every `COPY` in the backend Dockerfile since they were all written assuming they ran from inside `backend/`. Fix was just simple — `COPY requirements.txt .` became `COPY backend/requirements.txt .`, same for the rest of the file.

```yaml
# docker-compose.yml
services:
  oj-backend:
    build:
      context: . # was ./backend, now the repo root
      dockerfile: backend/Dockerfile
```

```dockerfile
# before (Dockerfile assumed build context was backend/)
COPY requirements.txt .
COPY deploy/requirements.txt /app/deploy/

# after (build context is now the repo root, every path needs the prefix)
COPY backend/requirements.txt .
COPY backend/deploy/requirements.txt /app/deploy/
```

After all that it finally built and ran natively on ARM. Then I needed an actual dev environment and staging next to prod, which broke a whole bunch of other things — that's [the next post](/blog/dev-prod-on-two-arm-servers/).
