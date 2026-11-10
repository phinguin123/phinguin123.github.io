---
title: "Planning CI/CD for Two ARM Servers"
date: 2026-01-12T16:03:00+09:00
draft: false
categories: ["DevOps"]
tags: ["ci-cd", "github-actions", "docker", "arm64", "oracle-cloud", "cloudflare"]
summary: "Planning the pipeline to replace building images by hand: where builds run, what staging is even for, which registry, and what I'd push back on now."
---

My online judge runs on two Oracle ARM servers, one for dev + staging and one for prod. Until January 2026 I built every image by hand on the dev box. This is me planning an actual pipeline, and what I found wrong with that plan when I came back to it later.

Picks up from [the two-server post](/blog/dev-prod-on-two-arm-servers/), whose "what's still wrong" list started with "images are built by hand, should be built in CI." So in January I sat down to plan it. Started off honest:

> I want to really become a senior full stack developer. Do version control, do CI/CD pipeline (already done a little bit) but mostly just used AI so I am not really sure of the concept. If I were to just do it on my own without AI, I don't think I will be able to do it successfully.

So the goal wasn't a YAML file, it was understanding each piece enough to actually defend it.

## Where do ARM images get built?

Servers are ARM, default GitHub Actions runners are x86. I'd already tried the obvious route:

> I had a freaking hard time with docker buildx. Because of cross compiling had to do a lot of things. Also the GitHub Actions server is limited, so if the memory is too much then it doesn't work.

That's the QEMU penalty. `buildx` on an x86 runner builds ARM images by emulating an ARM CPU in software — works, but slow and memory hungry, and my images were big. Judge image alone was 3.2 GB, backend 1.7 GB. The hosted runner kept running out of memory.

Plan we landed on was a self-hosted runner on the dev box. Already ARM, plenty of RAM, builds natively, so all the `setup-qemu` / `setup-buildx` stuff just goes away. Dev box ends up wearing three hats:

| | What runs there | Reached via |
|---|---|---|
| Dev box | Dev environment (hot reload), staging stack, CI runner | SSH tunnel / staging hostname |
| Prod box | Production stack only | Public domain |

Native builds were the right instinct. But re-reading this, there was an option the plan never even considered — GitHub's own hosted ARM runners. More on that below.

## What's staging for if it's just me?

The plan gave staging its own public hostname. Didn't see the point:

> staging is supposed to be for only specific people. But I have no friends to share it with. Then why can't I just use localhost with https? I can just use ssh tunneling, isn't it?

What made it click: staging tests the *infrastructure*, not the code. Dev runs `npm run dev` and a debug server. Prod runs a built image behind nginx with TLS. Bugs live in that gap — a minified build that acts differently, a proxy header, a cookie that only breaks over HTTPS, a webhook from some third party that can't reach `localhost`. Staging is there to run the prod build and config one step before prod.

But I was half right. What makes something staging is parity (same image, same compose file, same proxy), not being public. A staging stack behind an SSH tunnel is still staging. If it does need to be public (webhooks, testing on a phone over real internet), it should be behind an access gate. The plan used Cloudflare Access, an email-code login in front of the hostname, so bots can't hit a half-finished build.

That also settled the compose layout. Staging doesn't get its own file, it uses the same prod override and the differences live in env vars (domain, DB, image tag). A `docker-compose.staging.yml` would just be one more way for staging and prod to drift apart.

## "The big projects do this." Do they?

To justify the base + override compose pattern, I got told big open-source projects like Airbyte and Sentry use it. So I went and looked:

> how do you know that? … because I went to the GitHub, getsentry/sentry, and I can't find any Dockerfiles

The real answer was actually more useful than the claim. Airbyte moved off Docker Compose entirely to a Kubernetes-based installer. Sentry keeps deployment in a separate repo (`getsentry/self-hosted`) from the app code. App repo = "make the product work", infra repo = "make it run on a server." I don't need two repos as a solo dev, but it's a good split to keep in my head: `src/` is the app, compose files are infra.

Since then whenever someone says "this is the industry standard", I go look at a real repo first.

## Which registry, public or private?

Few days later I looked at where images should live. I'd been pushing to Docker Hub, and GHCR sits right next to the code. The cost answer kept flip-flopping — first "you'll have to pay", then once I said my images are public, "it's free". Then the question that actually mattered:

> I plan on making my website with subscription. But if I just expose it to public then anyone can just copy my repo and just use the exact same website … So does that mean I have to put them in private?

Yes. A public image *is* the product — anyone can `docker pull` it and run my site, including whatever I built on top of the open-source base. And anything baked into an image is public too: config files, startup scripts, a default password in an entrypoint. So the real decision isn't Docker Hub vs GitHub, it's "is this supposed to be public?" For something I want to charge for, no. Which makes image size a cost problem since private storage is metered, and 3.2 GB images are the first thing to shrink with a multi-stage build.

## The plan

```text
push to develop --> self-hosted runner (dev box, ARM)
                      |- docker build (native)
                      |- push image :staging
                      `- restart staging stack on the dev box

merge to main ----> same runner
                      |- build + push image :production
                      `- SSH into prod box -> docker compose pull && up -d
```

## What I'd change now

Came back to this later and found five problems.

### GitHub has hosted ARM runners

"GitHub runners are x86, so either emulate or self-host" was just out of date. GitHub has had hosted arm64 Linux runners (`ubuntu-24.04-arm`) free for public repos since early 2025. For a public repo that's native ARM builds with no QEMU and no server of mine to babysit. Self-hosting mostly makes sense for private repos, where hosted ARM costs money, or when builds need more than a hosted runner has.

### The runner puts prod's key on the weakest box

In this plan the runner lives on the dev box — which is where I experiment — and it holds the SSH key that deploys prod. So the least trusted machine holds the keys to the most trusted one. And it gets worse:

- The runner user goes in the `docker` group, and Docker socket access is basically root on that host.
- GitHub's own docs say only use self-hosted runners with private repos, because a PR from a fork of a public repo can run code on your runner.

If I self-host, I'd want a narrow deploy key on prod (only allowed to pull and restart), a private repo or strict approval for fork PRs, and ideally a runner that isn't my dev environment.

### The TLS advice was a downgrade

The plan suggested Cloudflare SSL mode "Full", called "Full (Strict)" an enterprise extra, and said Cloudflare would hit the origin on port 80. That doesn't even make sense together — talking to the origin over plain HTTP is what Flexible does, and Full uses HTTPS but doesn't validate the origin cert. I'd already set up Full (Strict) with a Cloudflare origin cert on this domain a month before ([here](/blog/what-the-upstream-image-was-hiding/)). Following this would've made a working setup worse.

### Careful with unattended cleanup

To stop the dev box's disk filling up with build layers, the plan added a weekly cron running `docker system prune -af --volumes`. On a box that also has dev and staging data, that's sketchy. `-a` removes every image not used by a running container, so the next build re-downloads everything, and `--volumes` removes volumes not attached to running containers. Depending on Docker version that can include a named DB volume for a stack that just happened to be stopped that night. I'd prune images on a schedule and leave volumes to a manual decision.

### Tags should be immutable

The plan tagged images `:staging` and `:production`. Those are moving labels — they say where an image is deployed, not what it is. Also tagging every build with its commit SHA, and deploying by SHA, makes "what's on prod right now?" actually answerable and rollback a one-liner.

I started this wanting to understand CI/CD enough to do it without AI. Asking why at every step got me most of the way. The plan's worst mistake, the TLS downgrade, only showed up when I checked it against what I'd already built.
