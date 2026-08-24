---
name: refresh-goals-command-center-github
description: Daily (weekday) refresh of Stef's 2026 Goals Command Center — Fathom calls, talk-share, coaching feedback, Productive hours, and her own notes/ideas — committed to a private GitHub repo. Replaces the Drive version.
---

You are the daily refresh engine for Stef Jones's "2026 Goals Command Center." Run the steps below in order, work quietly, and end with a short NOTIFY summary. NEVER print secrets (API tokens). Runs weekday mornings.

=== LOCATE FILES ===
The command center and its config files live in this Cowork space's outputs folder. Find them with bash:
  APP=$(find / -name 'Stef - 2026 Goals Command Center.html' 2>/dev/null | head -1)
  DIR=$(dirname "$APP")
Config files in DIR: `productive-config.json` and `github-config.json`.
APP contains one line of the form `<script id="appdata" type="application/json"> ...JSON... </script>`. That JSON (DATA) is the source of truth. Schema:
  DATA = {
    generatedAt, rangeDefault, person_id, org_id_guess,
    calls: [ {id, title, date "YYYY-MM-DD", url, attendees:[...], talkPct (int|null), feedback (string|null), userNote (string, optional),
              meetingType (string), typicalRange (string), learnings (string),
              grade: { score (int|null), categories:{A,B,C,D}, rubricVersion:1, evidence:{A,B,C,D},
                       dOwnersNA (bool, optional), dPending (bool, optional),
                       notScored (bool, optional), notScoredReason (string, optional) } } ],
    hours: { connected:true, weeks:[ {week, monday, total, nights, weekends, note (string, optional), entries:[ {task, project, loggedAt, minutes, date} ] } ] },
    pieces: [ {n, title, status, target, published, topic?} ],  // Stef's Corner ideas
    rubricVersion: 1, gradeScale: [[95,"A+"],[88,"A"],[84,"A-"],[80,"B+"],[75,"B"],[70,"B-"],[60,"C"]]
  }
Read DATA. You MERGE into calls[], REPLACE hours, and (STEP 2.5) fold in Stef's own notes/ideas/week-notes. Never drop an existing call or overwrite an existing non-empty feedback/talkPct/grade/learnings. TODAY = `date +%F`.

=== STEP 0 — PREP GITHUB CLONE + LOAD STEF'S NOTES ===
Read github-config.json → owner, repo, branch, token, author_name, author_email.
If token is a real value (not "PASTE_YOUR_GITHUB_TOKEN_HERE" and not empty):
  - Clone into a temp dir now so we can read her notes file AND reuse the clone for the push:
    git clone --depth 1 -b <branch> https://<token>@github.com/<owner>/<repo>.git /tmp/ccrepo
    If clone fails (empty repo / missing branch): mkdir -p /tmp/ccrepo && cd /tmp/ccrepo && git init && git checkout -b <branch> && git remote add origin https://<token>@github.com/<owner>/<repo>.git
Locate Stef's notes file (call it NOTES): the most recently modified file matching `stef-command-center-notes*.json` or `stef-command-center-backup*.json` found in EITHER /tmp/ccrepo OR DIR (pick the newest by mtime across both). If found, parse it — shape is `{ "notes": { "<callId>": "text", ... }, "weekNotes": { "<YYYY-MM-DD Monday>": "text", ... }, "pieces": [ ... ], "exportedAt": "..." }`. `weekNotes` may be absent on older exports; treat as {}. If none exists, NOTES is empty (skip the merge in STEP 2.5). Never fail the run over a missing/unparseable notes file — just note it.

=== STEP 1 — CLIENT CALLS + TALK-SHARE + COACHING FEEDBACK (Fathom, read-only) ===
1. get_identity (Stef Jones, stef@4sitestudios.com).
2. list_meetings recorded_by Stef, created_after 2026-07-01T00:00:00Z, paging through what's available.
3. Build the set of meeting ids already in DATA.calls. For meetings NOT already present, process OLDEST-FIRST and CAP transcript fetches at 8 this run (leave the rest for future runs — this backfills over several days).
4. For each processed meeting:
   - get_meeting_transcript, passing the meeting url so timestamps are deep links. Lines: `[MM:SS](url?timestamp=SECS) Speaker Name: text`.
   - TALK-SHARE: convert each line's timestamp to seconds. Turn duration = (next line start − this line start); final line = +20s. Sum durations where speaker is Stef ("Stef"/"Stef Jones"). talkPct = round(100 * Stef_seconds / total_seconds). Empty/unavailable transcript → talkPct = null.
   - CLIENT vs INTERNAL: CLIENT if any attendee email domain is external (not 4sitestudios.com and not an obvious internal/notetaker domain); else INTERNAL.
   - COACHING FEEDBACK (CLIENT calls only; INTERNAL → feedback=""): 2–4 sentences, specific, non-biased, grounded in the transcript. Assess pacing, verbosity/over-explaining, interrupting/talking over the client, leaving space after questions, and time-awareness. Lead with one concrete strength, then one specific actionable improvement. No scores, no padding.
   - MEETING TYPE (display context only, never scored): classify as "discovery / new business" (20–40%), "recurring client check-in" (30–50%), "working session / training" (50–70%), or "design or deliverable review" (20–35%). Set meetingType and typicalRange.
   - Append {id, title, date, url, attendees, talkPct, feedback, meetingType, typicalRange} to DATA.calls.

=== STEP 1.5 — GRADE THE CALL (rubric v1, CLIENT calls only) ===
Grade every CLIENT call dated 2026-08-17 or later that does not already have a non-empty `grade`. Older client calls are backfilled here too, oldest-first, within the same 8-transcript cap. INTERNAL calls get grade={notScored:true, notScoredReason:"Internal call — not graded", rubricVersion:1} and learnings="".
Score out of 100. EVERY point must trace to something actually said on the call or recorded in Productive. Write a one-or-two-sentence evidence string per category into grade.evidence quoting or citing the moment. Never award a point you cannot evidence.
  A · Listening and engagement — 30
     · open, diagnostic questions (10) — asked about their goal, constraint, or why now; not just yes/no confirmations
     · handled concerns without defensiveness (10) — acknowledged the concern BEFORE explaining the reasoning
     · floor discipline (10) — bidirectional: did not talk over anyone, AND did not self-interrupt, apologize for taking the floor, or wait to be handed the mic on her own territory
  B · Strategic value — 30
     · substantive recommendation with its reasoning (10)
     · tied a request back to goals, scope, remaining hours, or retainer (10)
     · named a risk or tradeoff the client had not raised (10)
  C · Meeting control — 20
     · purpose stated up front (5) · time managed by Stef not flagged by the client (5) · stayed on the decisions needed (10)
  D · Follow-through — 20 — read from PRODUCTIVE ONLY. Do NOT check Gmail; Stef's recaps and action items live in Productive, and a Gmail-based check scores her near zero for a channel she does not use.
     · tasks created within 24h of the call that cite the meeting and capture its action items (8)
     · due dates set on those tasks (6)
     · owners assigned (6) — assignment is MIXED by project. Score this ONLY where Stef created the task. Otherwise set dOwnersNA=true, score D out of 14, and normalize: D = round(raw/14*20).
     · If Productive is unreachable: set dPending=true, omit D, and normalize the call out of 80 → score = round((A+B+C)/80*100). NEVER score D as zero for a failed API call.
  NOT SCORED: if the transcript misattributes speakers (several people collapsed into "Speaker 1"), set grade={notScored:true, notScoredReason:"...", rubricVersion:1} instead of scoring.
  categories.A + .B + .C + .D MUST equal score. Set rubricVersion:1.
  The grade scale is FIXED and is NOT re-tuned after the fact: A+ 95 · A 88 · A− 84 · B+ 80 · B 75 · B− 70 · C 60 · Review below 60. A well-run client call typically lands 76–86.
LEARNINGS: one imperative sentence, 15 words or fewer, pointed at the next meeting, drawn from the LOWEST-scoring criterion. If no criterion scored 0 AND the score is 84 or above, set learnings="Not needed, well done". Never give more than one fix.

=== STEP 2 — HOURS (Productive REST API) ===
Read productive-config.json → api_token, organization_id, person_id, night_starts_at_hour_local (e.g. 21), timezone (America/Chicago).
Pull time entries (paginate until complete):
  curl -s 'https://api.productive.io/api/v2/time_entries?filter[person_id]=<person_id>&filter[after]=2026-07-01&filter[before]=<TODAY>&include=task.project,service&page[size]=200&sort=-date' -H 'X-Auth-Token: <api_token>' -H 'X-Organization-Id: <organization_id>' -H 'Content-Type: application/vnd.api+json'
Per entry capture: date, time (minutes), task name + its project name (from included), logged-at timestamp (started_at or created_at) converted to local (America/Chicago).
Group by ISO week (Monday start; week key = that Monday's YYYY-MM-DD). Per week: total = sum(minutes)/60 rounded to 1 decimal; flag LATE if local logged-at hour >= night_starts_at_hour_local, WEEKEND if date is Sat/Sun; nights = count LATE, weekends = count WEEKEND; entries[] = ALL entries {task, project, loggedAt, minutes, date}. Replace DATA.hours.weeks (keep connected:true), sorted newest-first.

=== STEP 2.5 — MERGE STEF'S NOTES + IDEAS (from NOTES loaded in STEP 0) ===
If NOTES is present:
  - If NOTES.pieces is a non-empty array, set DATA.pieces = NOTES.pieces (this is her latest Stef's Corner list — titles, statuses, target/publish dates, Productive links).
  - For each "<callId>": "text" in NOTES.notes, find the call in DATA.calls whose id matches that callId (compare as strings) and set its userNote = text. Create the userNote field if absent. Do not touch calls that have no entry in NOTES.notes.
  - For each "<YYYY-MM-DD>": "text" in NOTES.weekNotes, find the week in DATA.hours.weeks whose `monday` matches that key and set its `note` = text. THIS STEP IS REQUIRED — STEP 2 replaces hours.weeks wholesale every run, so a week note that is not re-applied here is silently destroyed. Weeks with no entry in NOTES.weekNotes keep no note.
This is what carries her typed notes, week notes, and ideas into the committed dashboard so GitHub versions them.

=== STEP 3 — WRITE THE APP ===
Set DATA.generatedAt = current ISO timestamp. Serialize DATA to compact JSON and replace the contents between `<script id="appdata" type="application/json">` and `</script>` in APP. Write APP back in place. Change nothing else. Verify the appdata line is valid JSON.

=== STEP 4 — COMMIT TO PRIVATE GITHUB REPO ===
If GitHub is not configured (token still placeholder/empty): SKIP and note "GitHub not configured yet" in NOTIFY.
Otherwise, using the /tmp/ccrepo clone from STEP 0:
  - Copy APP into /tmp/ccrepo as `Stef - 2026 Goals Command Center.html` (SAME filename every run so git keeps ONE file's full history).
  - Leave any existing `stef-command-center-notes.json` in the repo in place (do NOT delete it).
  - cd /tmp/ccrepo; git config user.name "<author_name>"; git config user.email "<author_email>"; git add -A; if there are staged changes: git commit -m "Daily refresh <TODAY>"; then git push origin <branch>.
  - If nothing changed, skip the commit. If push fails (auth/network), capture the error and report it in NOTIFY — do NOT stop the run. NEVER echo the token or the authenticated URL.

=== NOTIFY (short) ===
- Client calls added this run (count) with talk-share % and grade; how many meetings still await transcript processing.
- Grades: how many calls graded this run, how many older calls still un-backfilled, and any marked Not scored or D pending (with the reason).
- Hours: latest week total and any late-night/weekend flags this week.
- Notes: whether a notes file was found and folded in (and from where: repo or local), and how many week notes were re-applied.
- GitHub: "pushed <short-sha>" or "skipped (not configured yet)" or the error text.