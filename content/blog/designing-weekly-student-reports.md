---
title: "Redoing the Schema for Weekly Student Reports"
date: 2026-02-02T12:45:00+09:00
draft: false
categories: ["Backend"]
tags: ["database-design", "mysql", "rest-api", "flask", "react", "system-design"]
summary: "Turning one-off student comments into weekly cumulative reports for two programs with different calendars, and the upsert that would've silently doubled every comment."
---

The academy platform I built stored one comment per student per subject, and reports got generated from that. The academy wanted something else: comments that build up week by week, with reports at fixed points in the term. Writing the code was the easy part. The hard part was modelling a calendar that's different per program — and one question I almost didn't ask, about how MySQL even detects a duplicate, found a bug that would never have thrown an error.

This was a refactor of a live system teachers use every week, February 2026. Talked it through with Gemini and had Cursor do most of the editing.

## Where it started

Comments lived in one table:

```text
student_comments
  comment_id       PK
  student_id       FK
  teacher_id       FK
  comment_date     timestamp
  comment_text     text
  subject_name     varchar(50)    <- a string, not a reference
  week_start_date  date
```

Before changing anything I went back over two of my own earlier choices.

Why does `week_start_date` exist? It's a grouping key. `comment_date` is when the row was written, `week_start_date` is which week it belongs to. Without it every query would have to figure out "which week is this timestamp in" on the fly. So keeping it was right, it just needed a clear rule (below).

`subject_name` was a string. I did that to save a join. But a string means "IB Computer Science", "IB CS" and "Computer Sci" all end up existing eventually, grouping breaks, and renaming a subject means rewriting thousands of rows. So it became `subject_id`, FK to `subjects`. Boring but correct.

## Reports aren't "week 6" for everyone

The academy runs a 12-week bootcamp, and we wanted two reports per term. Obvious design is "Week 6 report" and "Week 12 report". Didn't want that because it's wrong for one group:

- Grade 12 (Seven Attack) does 6 weeks of humanities then 6 weeks of sciences. Their "week 12" is really week 6 of sciences. Their reports are "Humanities" and "Sciences".
- Grades 9–11 (IBP) are a different program with no split. Their reports really are "Week 6" and "Week 12".

So a report isn't a week number, it's a named period with a cutoff date, and which periods apply depends on the student's program.

## Dates go in the database

First instinct was env vars, then talked myself out of it. Env vars are deploy-time config. Every term I'd be editing them and redeploying just to change a date that belongs to the school calendar. Stuff that changes on the business's schedule and not the code's belongs in the DB:

```sql
CREATE TABLE report_periods (
  id              INT AUTO_INCREMENT PRIMARY KEY,
  display_name    VARCHAR(100),            -- "Humanities (Weeks 1-6)"
  cutoff_date     DATE,                    -- when this report opens
  target_program  ENUM('IBP', 'SevenAttack')
);
```

Four rows cover the whole school, two per program.

Later I wondered about adding a `grade_id` instead, since "Seven Attack = grade 12" felt like it should be explicit. Didn't. The "Week 6" report applies to grades 9, 10 and 11, so a grade column means three copies of the same row and changing one date means editing three rows. `target_program` is the right level. Grade → program is a separate fact; with two programs it's one small function, and if programs ever multiply it gets its own `program_grades` table.

On the UI side, a report button stays disabled until its cutoff with "Opens on …", so nobody calls asking why it's empty. That's just convenience though — the backend has to refuse early reports too, or anyone can just hit the URL.

## What counts as a week?

Classes run on different days (some Saturday, some Sunday), so "which week is this" needs one rule: `week_start_date` is always the Monday of that week. A Saturday class and a Sunday class in the same week both store the same Monday, so "Week 3 comments" is an equality match, not a date range puzzle.

Teachers never see that date. They pick "Week 3" and the frontend converts it to the Monday, counting from the term start date (from config, not hardcoded). DB stores exact dates, UI talks in weeks like teachers actually think.

A cumulative report is then just a range:

```sql
SELECT ...
FROM student_comments
WHERE student_id = ?
  AND week_start_date >= ?   -- period start
  AND week_start_date <= ?   -- period cutoff
ORDER BY week_start_date;
```

## Separate endpoints or one big one?

The old API had grown some weird names (`/api/student/submit-comment`, `/api/comments?student_id=…`) and some redundancy. I wanted it resource-shaped:

| Purpose | Endpoint | Method |
|---|---|---|
| Term config | `/api/config/term` | GET |
| Student list + badges | `/api/students` | GET |
| A student's comments | `/api/students/:id/comments` | GET |
| A student's marks | `/api/students/:id/marks` | GET |
| Save a comment | `/api/comments` | POST |

Then the question I actually cared about:

> is it right to call multiple endpoints? or should I make a single endpoint … what I heard for dashboard is, you don't want multiple endpoints. Because then some might be there while some other marks are still loading.

Gemini recommended one aggregated endpoint (a "backend for frontend"). Seemed wasteful:

> If I make a single endpoint then it should just only be used for that specific page and nothing else. Then it is useless.

Then it gave me a "rule": 1–3 endpoints separate, 6+ aggregate. Why would the number matter though?

> so you said 4 endpoints. What if there are 5 endpoints? what about 6?

And that's where it fell apart, because there's no magic number. The "rule of 6" is the HTTP/1.1 per-host connection limit, which HTTP/2 gets rid of, and prod nginx serves HTTP/2. What actually matters:

- What loads at the same time? Page load: config + student list. Opening a student: comments + marks. Saving: one POST. Never more than two requests at once, both parallel with `Promise.all`.
- Any waterfalls? Request B needing A's result is what makes many calls slow, not the count. None here.
- Do the pieces change at different rates? Term config changes once a term, comments every minute. Separate endpoints can be cached separately.
- Will anything else reuse them? A report page needs marks without comments. A page-shaped endpoint ties the API to one layout.

So separate resource endpoints, fetched in parallel. If one view ever needs a bunch of things at once over a slow network, an `?include=` on the resource is the usual middle ground, not a one-off endpoint.

## Editing a comment, and why PUT exists

Then I noticed a gap:

> what if you change the comment...? there is nothing for that..? don't you have to put?

Answer was an upsert — one save that inserts the first time and updates after. Works because a comment has a natural key: exactly one comment per student, per subject, per week. The frontend doesn't need to track a `comment_id` or know if one exists. Save works like saving a doc, not submitting a form.

So then why does PUT exist? Because most data has no natural key. Two identical comments on a social post are two different things, so editing one has to name it (`PUT /comments/500`). PUT is also idempotent, same request twice = same result. Here the natural key gives the same guarantee.

## The upsert couldn't actually work

The upsert got recommended twice, as this:

```sql
INSERT INTO student_comments (student_id, subject_id, week_start_date, comment_text, teacher_id)
VALUES (?, ?, ?, ?, ?)
ON DUPLICATE KEY UPDATE comment_text = VALUES(comment_text), teacher_id = VALUES(teacher_id);
```

Looked done. But I didn't get how MySQL decides what a "duplicate" is:

> how tf does duplicate key update even work? like how does it check the duplication. isn't it if I make a coupling for those specific ids?

And that was the bug. `ON DUPLICATE KEY UPDATE` only fires when an insert hits a PRIMARY or UNIQUE key. My table's only unique key was `comment_id`, which auto-increments, so every insert is "new". The upsert would never update anything. It would just quietly insert a second, third, fourth row for the same student-subject-week, and the report would show all of them. No error. Ever.

The "coupling" I was describing is a composite unique key, which the whole design depended on:

```sql
ALTER TABLE student_comments
  ADD UNIQUE KEY unique_weekly_comment (student_id, subject_id, week_start_date);
```

Two related things:

- `subject_id` was nullable. In MySQL `NULL` never equals `NULL`, so rows with a null subject slip past the unique key. Every comment has a subject anyway, so `NOT NULL`.
- Backend is raw `pymysql`, no ORM, so safety is on me: parameterized `%s` placeholders, explicit commit/rollback, closing the connection in `finally`. (Gemini first wrote it in SQLAlchemy, which I don't even use, and one of the frontend rewrites dropped the subject filter teachers rely on. Caught both in review.)

## What I'd change now

Design held up, but re-reading it there are four things I'd do differently.

Check for existing duplicates before adding the unique key. The table already had thousands of rows. If any student-subject-week already showed up twice, the `ALTER TABLE` fails — or worse, someone "fixes" it by deleting rows without looking. Run `GROUP BY student_id, subject_id, week_start_date HAVING COUNT(*) > 1` first, decide what to keep, then add the key.

Give each period a start date too, not just a cutoff. This one comes from my own Seven Attack requirement lol. "Everything up to the cutoff" is right for IBP's cumulative reports. But the Sciences report covers weeks 7–12, and "everything before the cutoff" would pull humanities weeks into it. `report_periods` should have `start_date` and `cutoff_date`.

Name the upsert for what it is. A save keyed by a natural key is idempotent, which is literally PUT's contract. `PUT /api/students/:id/comments/:subjectId/:week` describes it better than `POST /api/comments`.

Last write wins. If two teachers edit the same student-subject-week at the same time, the later save silently overwrites the first. Gemini called this "graceful", it's really just silent. Fine for this app since one subject teacher owns each comment, but if that changes, the fix is optimistic concurrency — send the `updated_at` you loaded and reject the save if it changed.

Pretty much every good decision here came from not taking the first answer: the env var, the magic number of endpoints, the missing PUT, and mostly "how does it actually detect a duplicate?" Really glad I asked that one, because the upsert looked finished and would've doubled every comment.
