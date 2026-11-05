---
title: "Pen-Testing My Own Upload Endpoint"
date: 2026-07-01T20:01:00+09:00
draft: false
categories: ["Security"]
tags: ["security", "path-traversal", "flask", "docker", "appsec", "gemini"]
summary: "Tried to path-traverse my own homework upload API. It held up, but only because of a timestamp prefix I never meant as a security thing."
---

Tried to break the file upload on my own homework-submission API, and it held. Cool, right? Then I figured out *why* it held, and it was a timestamp prefix I added for a completely different reason. Nothing in my code was actually blocking anything.

The API is part of a class platform I built — students upload files, teachers grade them. Flask on Gunicorn, in Docker. I talked through it with Gemini as I went, which was mostly useful as something to argue with. This is the write half. [The read half](/blog/reading-my-own-etc-passwd/), about the download endpoint, was two days before.

(Hostnames and internal paths below are changed. Posting my live site's exact directory layout in a post about leaking info would be kinda ironic.)

## Reading files, and what Docker was hiding

Path traversal is the classic one. If an endpoint builds a path by gluing user input onto a base directory, something like `../../../../etc/passwd` can climb out of the folder it's supposed to stay in. You don't even need to know how deep you are, extra `../` don't hurt since going above `/` just stays at `/`.

First weird result: `/root/.bashrc` was readable but `/home/ubuntu/.bashrc` wasn't. My first guess (and Gemini's) was permissions — home dirs are often `700` so a low-privilege web user can't get in.

Wasn't that. The app runs in a container, as root *inside* the container, and the image has no `ubuntu` user at all. `/home/ubuntu` only existed because my compose file mounts a host path with that name. Same reason `/etc/nginx`, `/var/log/syslog` etc. were missing — a slim container just doesn't have any of that.

For a bit that felt reassuring. It shouldn't. Docker does stop a traversal from reaching the host because the host's directories aren't mounted in. But inside the container it's arguably easier for an attacker since the layout is so predictable. Source is almost always at `/app`, secrets are often in `/proc/self/environ`. So a read bug couldn't touch the host, but it could walk straight into `/app` and read my source and config. Docker limits the damage, it doesn't fix the code.

My error handling leaked stuff too. A payload that hit a directory came back as:

```json
{
  "error": "[Errno 21] Is a directory: '/home/ubuntu/app/homework/../../../...'"
}
```

That one message gives away the install path, the OS user and the project name. Returning raw exception strings to the client is its own bug. Catch it, log it on the server, send back something generic.

## Writing files

Writing is worse than reading. If you control where an uploaded file lands, you can overwrite `/app/app.py` with your own code and the next request runs it. So I went after the upload handler, which looked roughly like this:

```python
file = request.files.get("file")
if file:
    # reject a few characters
    if re.compile(r'[<>#%]').search(file.filename):
        return {"message": "No special characters!"}, 400
    timestamp = get_current_time().strftime("%Y%m%d%H%M%S")
    filename = f"{timestamp}_{file.filename}"
    file_path = os.path.join(UPLOAD_FOLDER, filename)
    file.save(file_path)
```

Made an upload with a traversal filename aimed at `/app` and sent it with `curl` (a browser strips paths from filenames on its own). Nothing got overwritten:

```json
{ "message": "[Errno 2] No such file or directory:
  '/home/ubuntu/app/homework/19000101213025_//../../../../../../app/tests/__init__'" }
```

Attack failed. But "it didn't work" isn't the same as "it can't work", so I wanted to know why.

My code puts the timestamp in front of the filename with no slash in between. So the payload doesn't start a clean `../` sequence, it becomes part of one segment: `19000101213025_//../../...`. For Linux to treat `..` as "go up", it has to be its own segment between slashes. So the first thing the OS looks for is a directory literally called `19000101213025_` inside my homework folder. That doesn't exist (timestamp is generated fresh, to the second, every request), so resolution stops right there with `No such file or directory` and the traversal never runs.

So yeah, the prefix "works". Only because the OS can't find a folder with a random name though. I didn't block anything, the way the string happened to be built saved me.

## Gemini's bypasses

I kept asking Gemini to find a way around it, and it kept giving confident answers that fell apart when I actually tried them.

First it said `os.path.join` would collapse the path. `os.path.join` really does throw away earlier parts when a later argument is absolute, so supposedly a payload starting with `/` would wipe my homework prefix and land in `/app/..`. But that only happens when the absolute path is a *separate argument*. My slash was inside one string that was already concatenated (`"timestamp_/app/.."`), so `join` never sees a new absolute path. My error messages already showed it wasn't possible. When I said that, it went *"I apologize — I misspoke."*

Next was filename truncation: make the filename longer than the 255-byte limit so the kernel cuts it off and drops the timestamp. Tried it. ext4 on a current kernel doesn't silently truncate, it just returns `[Errno 36] File name too long`.

Then null-byte injection, `filename.py%00/../../`. That's a real technique in Python 2 and some C/PHP stacks, but Python 3 raises `ValueError: embedded null byte`. Gemini did mention it doesn't apply to Python 3... after suggesting it first.

All of these sounded exactly as sure as the explanations that turned out right. Only way to tell was to run each one against the real endpoint and read the error.

It was quite a fun experience trying different methods and seeing what might be a point for exploitation. But I guess these days the modern frameworks/libraries are pretty safe (except for things like CVE-2024-36039 in pymysql which allowed for SQL injection!)

## So is it safe?

Against everything I could come up with, it holds. But it's accidentally safe, which is still a risk, because what's protecting me is a side effect and not something I decided:

- If a later version drops the timestamp or moves it after the filename, the hole comes back and nobody notices.
- If the app ever gets a "create folder" feature, someone could make a folder with the right timestamp name and the whole "that folder can't exist" thing breaks.
- If `UPLOAD_FOLDER` ever becomes a relative path, a stray `os.chdir` in one of my Celery workers changes where everything lands.
- On Windows, or with a cloud storage SDK that auto-creates parent directories, the OS doesn't fail in the convenient way anymore.

None of these are exploitable today, but each is one refactor away.

Fix is to just not let dots and slashes into the path at all, clean the filename before `os.path.join` or the OS ever see it:

```python
from werkzeug.utils import secure_filename

if file:
    if file.filename == "":
        return {"error": "No file name"}, 400
    safe_name = secure_filename(file.filename)   # ../../app.py -> app.py
    timestamp = get_current_time().strftime("%Y%m%d%H%M%S")
    filename = f"{timestamp}_{safe_name}"
    file_path = os.path.join(UPLOAD_FOLDER, filename)
    file.save(file_path)
```

`secure_filename` strips traversal sequences, slashes and risky characters. Now the timestamp isn't the only thing holding it together — both would have to fail for the bug to come back.

## What the endpoint still needs

Found three more things while testing:

- No allowlist for file types. A student can upload `payload.py` or `x.sh`. Right now it just sits in a folder, but if any worker ever reads/imports/runs stuff from there, that's RCE. Should only allow what I actually accept (`.pdf`, `.docx`, …).
- No size limit. `file.save()` writes whatever comes in, so a script uploading huge files in a loop could fill the disk and take the service down. Flask's `MAX_CONTENT_LENGTH` fixes that.
- Errors still go straight to the client. `except Exception as e: return str(e)` is the same path leak from the read side.

Main thing I took from this: when an attack fails, find out why before you decide you're safe. Mine failed because of a coincidence I would've shipped without thinking twice.
