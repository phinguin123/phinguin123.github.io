---
title: "Who Actually Visited the Landing Page?"
date: 2025-11-24T11:43:00+09:00
draft: false
categories: ["DevOps"]
tags: ["nginx", "logs", "awk", "linux", "analytics", "docker"]
summary: "Answering a marketing question with grep and awk on nginx access logs: requests vs. visitors, bots, timezones, and why the log can't tell you what the browser never sent."
---

The academy was running Instagram ads for a winter program landing page, and we wanted to know who was actually visiting and from where. No analytics tool on the page, just nginx access logs. My first count said 2,514 visits from Instagram. By the end it was about 50 ad clicks lol, and almost every step in between was me counting the wrong thing.

Site ran on Docker Compose behind an nginx container. Started with unique IPs:

```bash
docker compose logs nginx --no-log-prefix | grep "/2025/winter/<page>" \
  | awk '{print $1}' | sort | uniq | wc -l
```

Then the actual question:

> But is it also possible to know from where they visited? like from instagram? or from chrome? or from google? or what

Kind of. The nginx combined log format quotes two fields the client sends: Referer (the page the browser was on before) and User-Agent (what the browser says it is). Split each line on `"` instead of spaces and they're `$4` and `$6`:

```bash
... | awk -F'"' '{print $4}' | sort | uniq -c | sort -nr
```

## Counting requests, not page views

First breakdown sorted referers into buckets:

```text
2514 Instagram (Ads & Profile)
1933 Other/Internal: https://<site>/2025/winter/<page>
 580 Direct Traffic (No Referer)
 217 Other/Internal: http://<internal-container-ip>/assets/...webp
 ...
```

2,514 was obviously way too high. The reason is the first `grep` — it matches `/2025/winter/<page>` *anywhere* in the line, including the Referer. When someone opens the page, the browser then requests the JS bundle, CSS and a dozen images, and every one of those carries the page URL as Referer. So one visitor from an ad shows up as like 15 "Instagram visits."

Filtering out asset extensions brought it down to something that looked real:

```text
579 Direct Traffic (No Referer)
 52 Instagram (Ads & Profile)
 11 Google Search
 11 Facebook
```

Closer, but the filter (`grep -vE "assets|css|js|webp|..."`) is still a substring match on the whole line, so a page view whose User-Agent or tracking ID happens to contain `js` or `ico` gets thrown out. The precise version matches the request field itself, `$2` when splitting on quotes:

```bash
awk -F'"' '$2 ~ /^GET \/2025\/winter\/<page>(\?| )/'
```

Then a page view is a page view, and nothing in Referer or User-Agent can add or remove lines.

Bucketing had the same kind of problem. `ref ~ /instagram|ig|fbclid/` treats any referer with the letters `ig` in it as Instagram. Matching the actual signals (`instagram.com`, `utm_source=ig`, `fbclid`) is just as easy and doesn't misfire.

## Link previews and fake phones

Splitting "Direct" by User-Agent cleared up a lot:

```text
91 Direct (Typed URL / QR Code)
66 Instagram (In-App Browser)
55 KakaoTalk (Chat Link)
53 Instagram (Ad/Bio Click)
...
```

KakaoTalk's and Instagram's in-app browsers often don't send a Referer, but they say who they are in the User-Agent (`KAKAOTALK`, `Instagram`). So a lot of "Direct" was really social traffic.

Some lines weren't people at all. When a link gets shared in a chat, the app's server fetches the page to make a preview card — `facebookexternalhit`, `kakaotalk-scrap`. Every share = a "visit" nobody made.

## Whose clock is it?

Next was traffic by hour, which had a huge spike at 05:00. Gemini assumed the logs were UTC and turned it into "2 PM in Korea, your golden time." Checked that instead of building on it:

```text
$ date
Mon Nov 24 11:53:37 KST 2025
```

Server was on KST, so the spike really was 5 AM. Narrowing to that hour on that day and looking at User-Agents explained it:

```text
85  Mozilla/5.0 (Linux; U; Android 2.0; en-us; Droid Build/ESD20) ...
31  Mozilla/5.0 (Linux; Android 10; SM-G981B) ... Chrome/80.0.3987.162 ...
 6  ... KAKAOTALK ...
 5  facebookexternalhit/1.1; kakaotalk-scrap/1.0 ...
```

Android 2.0 is from 2009 and Chrome 80 is from 2020. That's a scanner pretending to be a phone, not 116 parents up at dawn ㅋㅋ

Two things I'd add now. `date` checks the *host's* clock, but the log is written by the container, and an nginx container has its own timezone — usually UTC unless `TZ` is set. The reliable check is the offset in every log line, `[20/Nov/2025:05:12:33 +0000]` or `+0900`. Here the User-Agents settled it anyway, but the rest of the hourly table depends on getting that right.

Also, hourly totals across several days hide single-day events. The 05:00 bucket was mostly one burst on one day. Grouping by date + hour would've shown it as a one-day spike instead of a pattern. And a few dozen visits per hour isn't nearly enough data to decide when to run ads, which is what Gemini went on to recommend.

The "ultimate" filter it wrote at the end dropped Android 2.0 but not the Chrome 80 one it had *just* called a bot. User-Agent blocklists always lag behind. A better test for "human" is behaviour: a real browser that loads the page also fetches the JS bundle and images, and most simple scanners don't. Counting page views whose client also requested the bundle beats any list of bot names.

## Explaining data that doesn't exist

That left 91 "Direct" visits. Asked what Direct actually means and got the usual list: QR codes, links in SMS or email, incognito. None of that matched what I knew:

> I have not qr code. I don't think so many people use incognite, I don't think people use sms that much or emailt o send this link. I don't think they use discord that much. Then can your really analyze what these are

Next theory was people opened the link inside Instagram or KakaoTalk and then hit "Open in Chrome/Safari", which starts a fresh request with no Referer. Plausible, might even be most of them. But it's still a guess and the log can't confirm it, because the info was never sent. No amount of `awk` gets back a field the browser left out.

The fix is on the other end: put the source in the link. The paid Instagram ad already did this — its URL had `utm_source=ig`, and those were the only visits I could attribute for sure. Giving every link we hand out its own tag (`utm_source=kakao` for the KakaoTalk share, another for the blog, one per poster if there's ever a QR code) turns "Direct" from a mystery into a number. Query params survive app switches, Referers don't.

## Smaller things

- Unique IPs aren't unique people. Mobile carriers put tons of phones behind one shared IP, and one person on Wi-Fi then LTE is two IPs. Rough proxy at best.
- `docker compose logs` only goes back to when the container was created. Recreate the nginx container (config change, deploy) and the history's gone. For anything that matters, write the access log to a mounted file or ship it somewhere that outlives the container.
- Some referers made no sense. A few pointed at the internal Docker address of a container, which no outside browser should ever see. Gemini said it was "your server loading its own images", but servers don't send Referer headers, clients do. A leaked internal address usually means a redirect or absolute URL exposed the upstream host somewhere. Didn't trace it at the time, should.

## What the numbers said

After all the filtering the honest summary was small: roughly 120 visits from Instagram (ad clicks + in-app browsing), about 55 from KakaoTalk shares, a handful from Facebook and Google, and about 90 with no source. Good enough for a decision — the ad works, word of mouth on KakaoTalk is real, search brings basically nobody — as long as nobody treats it as more precise than it is.

What I'd tell myself before starting: requests, page views and people are three different numbers, so pick which one you're counting before piping stuff into `awk`. And if you want to know where visitors come from, tag the link when you hand it out.
