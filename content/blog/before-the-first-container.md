---
title: "Before the First Container"
date: 2025-11-24T16:29:00+09:00
draft: false
categories: ["Dev"]
tags: ["open-source", "licensing", "oracle-cloud", "arm64", "ssh", "security"]
summary: "Day zero of the online judge: can I even charge for it, how to split two free ARM boxes, not getting a surprise bill, and locking down SSH."
---

Before writing any code for the online judge I had four questions. Am I even allowed to charge money for something built on an open-source project? How do I split Oracle's free ARM capacity? How do I not get a surprise bill once I'm on a paid account? And who can actually reach the server?

This is kind of the prequel to [building the judge for ARM](/blog/online-judge-on-oracle-arm/) and [the two-server dev/prod setup](/blog/dev-prod-on-two-arm-servers/). In November 2025 I'd picked [QingdaoU OnlineJudge](https://github.com/QingdaoU/OnlineJudge) as the base. I wanted to customise it and maybe charge for it, so first question was the license:

> if I were to use this for getting money like 1 dollar per month then do I have to checkt eh license? but I customized it. then it is no longer same as the open source then isn't it okay?

## Is it mine if I changed it?

My first thought was that if I customise it enough it becomes a new project. Nope. Spent quite a while on this one:

> what makes a brick and what makes a blueprint? if I steal 10 bricks is that okay? if I steal 10000 bricks is that okay? or is 10000 bricks equivalent to 1 blueprint?

> What if the online judge qingdao repo was used for training chatgpt? then later I asked chatgpt to create me a repo for online judge, and it happened to give me the architecture that is really identical to qingdao.

> then what if I just give the entire repo to gemini and tell him to make me a OJ repo so that I don't infringe the license. Is that okay?

What I got out of it:

- Changing code doesn't reset who owns it. A modified version is a derivative work. Renaming variables or having an AI restructure everything ("code laundering") is still deriving from the original, because copyright covers the expression of the code, not just the exact text.
- Small snippets and common patterns are fine. Ten lines off a Q&A site or a standard REST layout isn't copying a project. Taking the whole structure and logic is.
- I also asked how the author would ever find out. Which is kinda the wrong question, because for this project doing it properly costs almost nothing anyway.

So the real question wasn't "how much do I have to change", it was "what does the license actually ask for".

### What Gemini said vs. what the repos say

Gemini said the whole thing is MIT and gave me a `LICENSE` file to paste, with a copyright line for Qingdao University and one for me. When I went and checked the actual repos:

| Repo            | Role                             | License |
| --------------- | -------------------------------- | ------- |
| `OnlineJudge`   | Django backend                   | MIT     |
| `OnlineJudgeFE` | Vue frontend                     | MIT     |
| `JudgeServer`   | Judge service                    | SATA    |
| `Judger`        | Sandbox that runs submitted code | SATA    |

So three things were off:

1. It's two licenses, not one. SATA ("Star And Thank Author") is MIT plus a request to star the project and thank the authors. It also says forks can add their name to the copyright and project-URL sections but "shall not delete or modify anything else in these two sections." And the judge — the part I rebuilt the most for ARM — is SATA.
2. The copyright line was just made up. The MIT repos actually say `Copyright (c) 2017-present OnineJudge` (typo is theirs). You don't paraphrase an attribution notice, you keep it as is.
3. "Own your project" is a stretch. My copyright line covers my changes. Upstream code is still theirs.

Both are permissive though, so commercial use is fine and $1/month is fine. The main thing is the copyright notice has to stay in "all copies or substantial portions" — and a Docker image is a copy. So the license files need to actually be inside the images I push, not just in the git repo, which is super easy to lose in a multi-stage build that only copies build output. (Not a lawyer obviously, I'd check again before actually charging.)

## Splitting the free tier

The obvious free option, `VM.Standard.E2.1.Micro` (1/8 OCPU, 1 GB), can't run Postgres + Redis + Django + a compiler-heavy judge at the same time. The actual good free thing is Ampere A1: 4 OCPU and 24 GB total, split however you want.

> but do I use 4 ocpu and 24 gb ram or 4 ocpu and 12gb ram or 2 ocpu and 12 gb ram

> but if I am going to use ampere and use 4 ocpu and 24 ram, that means by second which is the dev server would be using the micro instance. 1/8th ocpu and 1 ram. Is that okay?

Went with two identical 2 OCPU / 12 GB boxes, one prod and one dev. Looking back there's an even better reason than I had at the time: Micro is AMD x86 and A1 is ARM. A Micro dev server would have the same architecture mismatch as my desktop (I'd also asked "can't I just use my home desktop computer as dev server?"). Two identical A1 boxes means if it runs on dev, it runs on prod. That became the [first decision in the two-server setup](/blog/dev-prod-on-two-arm-servers/).

Disk has the same trap btw. Free tier is 200 GB of block storage *total*, shared across all boot volumes. Give the first instance 200 GB and the second one gets nothing.

## Out of capacity, and a bill with no ceiling

Creating the instance failed:

```text
Out of capacity for shape VM.Standard.A1.Flex in availability domain AD-1.
```

Seoul only has one availability domain so "try another AD" doesn't help. Home region can't be changed, and Always Free only applies in the home region, so going to Tokyo would mean paying full price.

Fix was upgrading to Pay As You Go. Always Free stuff stays free, capacity is way easier to get, and Oracle stops reclaiming idle instances. Catch is there's no guardrail anymore:

> okay so I did the pay as you go method. But now how do I make sure I don't suddenlyt pay like 1000 dollars

On a free account, making a fifth OCPU is just an error. On PAYG it starts billing. So the first thing I did was a $1 budget with an alert at 1%, so any real spend sends me an email.

Thing is, a budget alert is an alarm, not a cap. It emails you and stops nothing. The real protection is knowing the free limits (4 OCPU, 24 GB, 200 GB storage, home region only) and checking before creating anything.

## One SSH key or two?

> then for the dev server and prod server, instead of having 2 different keys to manage, I just have one?

Answer was one key, and for my situation that's right. Both private keys would sit in the same `~/.ssh` on my laptop, so if someone can steal one they can steal both. Second key = more hassle, zero extra protection. Named it `oracle-oj-identity.key` instead of something vague like `main.key`.

The way I think about it now: one key per *holder*, not per server. While I'm the only holder, one key. Seven weeks later [the CI/CD plan](/blog/designing-ci-cd-for-two-arm-servers/) put a prod deploy key on the dev box's runner, which is a different holder, so that one needs its own narrow key that I can revoke without touching mine.

Small debugging thing: first Remote-SSH connection failed with `no such identity: /home/<user>/.ssh/...` then `Permission denied (publickey)`. The editor was running on Windows and my SSH config had a Linux-style `IdentityFile` path that didn't exist on the machine actually running `ssh`. Key was fine, path pointed nowhere.

## Who can reach the server

Oracle's default security list allowed SSH from `0.0.0.0/0`:

> okay so how do I make sure only specific IP can access it. So that only my public ip can access it or something

Changed the source to my own IP with `/32`. Then:

> what about other rules? should I just disable them all/

Good thing I asked first, because they weren't all the same kind of rule:

| Rule                              | What it is                                                                       | Delete?                                                                   |
| --------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| TCP 22 from `0.0.0.0/0`           | SSH from anywhere                                                                | Restrict to my IP                                                         |
| ICMP type 3, code 4 from anywhere | "Fragmentation needed": how path MTU discovery works                             | No. Without it, large packets can silently disappear and connections hang |
| ICMP type 3 from the VCN          | Unreachable messages inside the private network                                  | Keep                                                                      |
| UDP 34197 from anywhere           | Left over from an earlier project on this account (it's Factorio's default port) | Yes                                                                       |

"Disable them all" would've killed the one rule that looks useless but quietly keeps TCP working. And the Factorio one is from an old project lol, should've cleaned that up back then.

Two things about `/32`:

- Home IPs change. When my ISP gives me a new one I'm locked out, and the way back in is the Oracle console (edit the rule or use the serial console). Better to know that before it happens.
- Oracle's Ubuntu images only allow key auth anyway, so bots weren't going to guess a password. Restricting the source still cuts the log noise and hides `sshd` from the whole internet, but it's reducing exposure, not fixing a weak login.

Also the security list is only one of Oracle's *two* firewalls. The instance has its own iptables rules too, which got me later when opening 80 and 443 ([details](/blog/what-the-upstream-image-was-hiding/)).

## Two questions that shaped the rest

First was installing Docker. Gemini said `curl -fsSL https://get.docker.com | sh`. I asked why not the official apt repo method from Docker's docs, and it agreed — the script is for quick setup, apt repo is the one meant for servers. Piping a random remote script into a root shell is not really something you want on prod.

Second was native ARM vs emulation, and this one basically decided the whole project:

> what do you think of writing the docker image in arm and not using emulator like qemu? because... it adds overhead? how much overhead do you think it adds? 10%?

It's not 10%, and overhead isn't even the main problem. An online judge's verdicts are timing measurements. A correct solution that runs slower because the CPU is emulated can come back as Time Limit Exceeded, so the judge becomes unfair, not just slow. On top of that the sandbox uses seccomp, and seccomp filters are tied to an architecture's syscall numbers — under user-mode emulation the kernel sees QEMU's arm64 calls, not the x86 ones the filter was written for. So everything got built natively for arm64. That's [the first post](/blog/online-judge-on-oracle-arm/).

Was kind of hyped trying to make and monetize the platform. Making the problems and the test cases was fun and kind of nostalgic, brought back the memories from when I was in college.
