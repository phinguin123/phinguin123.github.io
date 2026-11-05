---
title: "I Pulled /etc/passwd Off My Own Server"
date: 2026-06-29T20:48:00+09:00
draft: false
categories: ["Security"]
tags: ["security", "path-traversal", "nginx", "flask", "appsec", "gemini"]
summary: "An nginx 400 felt like a fix. send_from_directory felt like a fix. Then a different endpoint handed me /etc/passwd, and the real question was how bad that actually is."
---

Spent an evening trying to break the file download side of my own academy platform. It's a homework site with file upload/download — Flask behind nginx, in Docker, on an AWS EC2 box. Students upload actual files to this every week, so I wanted to know if it was exploitable before someone else found out for me.

Two things I thought were protecting me weren't, and then a different endpoint just handed me `/etc/passwd`. The more interesting part was after that though, figuring out how bad it actually was.

This is the read half. [The write half](/blog/accidentally-safe-isnt-safe/), about the upload endpoint, was two days later.

## "Is an nginx filter enough?"

That's what I started with, and no. I'd half assumed blocking `../` at the reverse proxy would be enough. Two problems.

First is encoding. A filter looking for the literal `../` doesn't see `%2e%2e%2f`, and the backend decodes it back to `../` after the filter already let it through. Make the filter decode first and double encoding (`%252e…`) gets through the same gap, since the proxy decodes once and the app decodes again. Blocking character sequences is basically a game you keep losing.

Second, nginx only normalizes the *path*. When I put an encoded traversal in the URL path, nginx returned `400 Bad Request` — normalized the URI, saw it go above root, refused. Felt like a win. But nginx doesn't touch the query string, and my file endpoint takes the filename as `?file=…`. That goes straight to Python untouched, same as anything in a POST body. So the 400 only covered one of three ways in, and it just happened to be the one I tested first.

## The fix that actually works

The real check has to be in the app, where the string turns into a file path. My download endpoint used Flask's `send_from_directory`:

```python
return send_from_directory(upload_folder, filename)
```

That one is safe. It resolves the absolute path of `upload_folder + filename` and refuses anything that ends up outside `upload_folder`. `send_file()` and a plain `open(path).read()` don't do that, they open whatever you give them. So it's not "Flask is safe/unsafe", it's which function you call.

I'd also written my own guard above it:

```python
if ".." in filename or filename.startswith("/"):
    abort(400, "Invalid filename")
```

Turns out this is redundant and also kinda wrong. Redundant because `send_from_directory` already does the real check. Wrong because it only catches what I thought of — it checks for a leading `/`, but a Windows path like `C:\…` or a backslash gets right past it. The framework's check enforces the thing I actually want ("stay inside this folder"), so that's the one to trust.

## Then it returned /etc/passwd

After convincing myself the download path was fine, I sent a plain `?file=../../../../../etc/passwd` to the API... and got `/etc/passwd` back.

Not the endpoint I'd been reading. A different code path was serving files with a raw reader instead of `send_from_directory`. I checked one endpoint and just assumed the rest of the app was the same. The safe function sitting in the file I had open said nothing about the route that was actually live on that URL. Should've tested what's deployed, not what's on my screen.

## How bad is it?

First reaction was kind of a shrug. It's a stock Ubuntu box and `/etc/passwd` doesn't even have password hashes anymore, those are in `/etc/shadow` which a low-privilege app user can't read.

Gemini went the complete opposite way. According to it, reading `/etc/passwd` was the start of stealing AWS credentials from the EC2 metadata service, reverse engineering `.pyc` files, reading root cron jobs and the kernel version to find a local privilege escalation exploit, and taking over the whole machine.

Didn't buy it:

> but what can you do with path traversal only..? you can't just do LPE cuz you can't even access it

This bug lets you *read* files. Not write, not execute. There's nothing to run a privesc exploit with. The AWS metadata thing is a different kind of bug entirely (SSRF, making the server send a request for you), and a file read doesn't give you that either. Once I said it plainly, Gemini agreed path traversal alone is "a passive window, not an active execution tool."

"Critical" and "nothing" are both wrong here. The better question is: with only file read, as this app's user, inside this container, what's actually worth reading?

## What a read bug can actually get you

So I went and checked these on my own box:

- Secrets in env or on disk: `/proc/self/environ`, a `.env`, config files with DB strings, API keys, cloud creds. On mine these were basically empty, which is the main reason this stayed low impact.
- Source code: `app.py`, `config.py`, `.pyc` caches. Not a shell, but a map of hidden routes, auth logic and other bugs to chain.
- SSH keys: `/home/<user>/.ssh/id_rsa`, if the box has a key for GitHub or other servers. That gets you onto other machines without running anything on this one.
- Session/DB files: file-based sessions, a SQLite file, a Redis dump. Read those and you can hijack a logged-in session, admin included, no exploit needed.
- Chaining: read a config, get a secret admin token, log in through the normal admin page.

Where I landed: on a box with nothing valuable to read, it's low. But "nothing valuable" is about today's deployment, not the code. Apps pile up keys and sessions over time, so it gets fixed either way.

## What I changed, and what I'd still check

- Every route that touches a file goes through `send_from_directory` (or the same kind of canonical-path check), including the old one I forgot about. Quick way to find them is grep for `send_file(` and `open(` near user input.
- Even better, don't let users name files at all. Store a UUID and look up the real path on the server. Kills traversal and the next problem too.
- Authorize every request. My endpoint served any file in the folder to anyone who knew the name, so user B could read user A's upload just by guessing (IDOR). Check the caller owns the file first, and make sure old endpoints do the same check.
- Never let the proxy execute anything from the upload folder. My nginx passes file requests to Flask instead of serving them itself, so an uploaded script just comes back as bytes. A proxy that runs scripts out of the upload folder is how a file bug turns into RCE.
- Run as non-root. The app runs as a normal user in a container, so even a working read is limited to what that user can see. Big part of why this stayed small.

The download endpoint turned out to be the boring part honestly. If RCE is hiding in a Flask app it's more likely in `pickle.loads` on untrusted data, `subprocess(..., shell=True)` with user input, or `render_template_string` built from an f-string (SSTI). Those are next on the list.

*Together with [the write-side post](/blog/accidentally-safe-isnt-safe/), this was one pass over the same app: can someone read files they shouldn't, and can they write files where they shouldn't? Answers were "yes, through one forgotten endpoint" and "not quite, but for the wrong reason."*
