---
name: se-cal-export
description: Export your last week's Google Calendar activities to the SE team's shared Drive folder for manager review. Run /se-cal-export to export now. Run /se-cal-export setup to also schedule automatic weekly exports every Monday at 9am Singapore time.
---

# SE Calendar Export

Export last week's Google Calendar activities as a CSV to the shared SE team Drive folder for manager review.

**Shared folder:** https://drive.google.com/drive/folders/1-9jFoXHh3Y7vx0LB4z7QVDIZVT4mSBVJ
**Folder ID:** `1-9jFoXHh3Y7vx0LB4z7QVDIZVT4mSBVJ`

---

## Step 1 — Check authentication

Before doing anything, verify both MCPs are connected:
- If `mcp__google_calendar` is not authenticated, call the authenticate tool and complete the OAuth flow.
- If `mcp__google_drive` is not authenticated, call the authenticate tool and complete the OAuth flow.

---

## Step 2 — Calculate last week's date range

Use the current date from the system context. Compute:
- **week_start**: the most recent Monday that is at least 7 days ago (i.e. the Monday of last week)
- **week_end**: the Sunday immediately following week_start

Use this Bash snippet to get the values:

```bash
python3 -c "
from datetime import date, timedelta
today = date.today()
# Days since last Monday (0=Mon ... 6=Sun)
days_since_monday = today.weekday()
# Start of THIS week
this_monday = today - timedelta(days=days_since_monday)
# Last week
last_monday = this_monday - timedelta(days=7)
last_sunday = last_monday + timedelta(days=6)
print(f'WEEK_START={last_monday}')
print(f'WEEK_END={last_sunday}')
"
```

Store `WEEK_START` and `WEEK_END` for use in subsequent steps.

---

## Step 3 — Fetch calendar events

Call `mcp__google_calendar__google_calendar-get_events` with:
- `calendar_id`: `"primary"`
- `time_min`: `"{WEEK_START}T00:00:00+08:00"` (Singapore time UTC+8)
- `time_max`: `"{WEEK_END}T23:59:59+08:00"`
- `max_results`: `100`
- `detailed`: `true`

Note the authenticated user's email from the response (format: `name@domain.com`) — you will use it to name the output file.

---

## Step 4 — Identify the user

Extract the user's email from the calendar response header line:
`Successfully retrieved N events from calendar 'primary' for {email}`

Derive the filename stem:
- Take the local part of the email before `@` (e.g. `marcus.guan` from `marcus.guan@okta.com`)
- Replace `.` with `_` → `marcus_guan`
- Final filename: `{stem}_calendar_{WEEK_START}_to_{WEEK_END}.csv`

---

## Step 5 — Parse events into CSV rows

Process all events with this Python script via Bash. Paste the full raw API result text as the `RAW` variable:

```bash
python3 << 'PYEOF'
import re, csv, io
from datetime import datetime

RAW = """PASTE_RAW_API_RESULT_HERE"""

# Extract result text from JSON wrapper if present
import json
try:
    data = json.loads(RAW)
    text = data.get('result', RAW)
except:
    text = RAW

# Split into event blocks
blocks = re.split(r'\n- "', text)
blocks = blocks[1:]  # skip header line

OKTA_DOMAINS = {'okta.com', 'resource.calendar.google.com'}

def parse_dt(s):
    s = s.strip()
    if not s or s == 'None' or not re.search(r'T', s):
        return None
    s = re.sub(r'([+-]\d{2}:\d{2})$', '', s)
    try:
        return datetime.fromisoformat(s)
    except:
        return None

def duration_hours(start_str, end_str):
    start = parse_dt(start_str)
    end = parse_dt(end_str)
    if not start or not end:
        return None
    return round((end - start).total_seconds() / 3600, 2)

def classify(attendees_str):
    if not attendees_str or attendees_str == 'None':
        return 'Internal'
    emails = re.findall(r'[\w.+-]+@([\w.-]+)', attendees_str)
    for d in emails:
        if d not in OKTA_DOMAINS:
            return 'External'
    return 'Internal'

def trim_attendees(att, max_n=5):
    if not att or att == 'None':
        return ''
    items = [x.strip() for x in att.split(',')]
    if len(items) <= max_n:
        return ', '.join(items)
    return ', '.join(items[:max_n]) + f' (+{len(items)-max_n} more)'

SKIP_TITLES = {'office', 'no title', ''}

output = io.StringIO()
writer = csv.DictWriter(output, fieldnames=[
    'Title','Start','End','Duration_Hours','Meeting_Type',
    'Location','Meeting_Link','Attendees','Description'
])
writer.writeheader()

for block in blocks:
    lines = block.split('\n')
    header_match = re.match(r'^(.+?)" \(Starts: (.+?), Ends: (.+?)\)$', lines[0])
    if not header_match:
        continue
    title, starts, ends = header_match.group(1), header_match.group(2), header_match.group(3)

    if title.lower() in SKIP_TITLES:
        continue

    dur = duration_hours(starts, ends)
    if dur is None:
        continue

    desc = location = meeting_link = attendees = ''
    for line in lines[1:]:
        if line.startswith('  Description: '):
            desc = line.replace('  Description: ', '').strip()
        elif line.startswith('  Location: '):
            location = line.replace('  Location: ', '').strip()
        elif line.startswith('  Meeting Link: '):
            meeting_link = line.replace('  Meeting Link: ', '').strip()
        elif line.startswith('  Attendees: '):
            attendees = line.replace('  Attendees: ', '').strip()

    if location == 'No Location': location = ''
    if desc == 'No Description': desc = ''
    desc_clean = re.sub(r'<[^>]+>', '', desc)[:200]

    writer.writerow({
        'Title': title,
        'Start': starts,
        'End': ends,
        'Duration_Hours': dur,
        'Meeting_Type': classify(attendees),
        'Location': location,
        'Meeting_Link': meeting_link,
        'Attendees': trim_attendees(attendees),
        'Description': desc_clean
    })

print(output.getvalue())
PYEOF
```

Capture the CSV output as a string variable `CSV_CONTENT`.

---

## Step 6 — Upload to the shared Drive folder

Call `mcp__google_drive__google_drive-create_drive_file` with:
- `file_name`: `"{stem}_calendar_{WEEK_START}_to_{WEEK_END}.csv"` (e.g. `marcus_guan_calendar_2026-08-11_to_2026-08-17.csv`)
- `folder_id`: `"1-9jFoXHh3Y7vx0LB4z7QVDIZVT4mSBVJ"`
- `mime_type`: `"text/csv"`
- `content`: the full `CSV_CONTENT` string from Step 5

If the upload fails due to a duplicate filename, append the current date (`_uploaded_{YYYY-MM-DD}`) before `.csv` and retry once.

---

## Step 7 — Summarise results

After a successful upload, print a short summary:

```
Calendar export complete for {email}
Week: {WEEK_START} to {WEEK_END}

  Total meetings : N events
  Total hours    : X.Xh
  Internal       : Ni events  Xi.Xh  (X%)
  External       : Ne events  Xe.Xh  (X%)

File uploaded: {file_name}
Drive folder : https://drive.google.com/drive/folders/1-9jFoXHh3Y7vx0LB4z7QVDIZVT4mSBVJ
```

---

## Step 8 — Schedule weekly automation (only when args contain "setup")

If the user invoked `/se-cal-export setup`, after the export succeeds, call `CronCreate` to schedule automatic weekly exports:

```
cron     : "0 9 * * 1"         # Monday 9:00am in local timezone (set your system to Asia/Singapore / UTC+8)
prompt   : "/se-cal-export"
recurring: true
durable  : true
```

Then confirm:
```
Weekly schedule set. This skill will run automatically every Monday at 9am (local time).
Note: ensure your system timezone is Asia/Singapore (UTC+8) for the correct fire time.
The schedule persists across Claude Code restarts.
To cancel it later, run: /cron-list and then delete the job ID.
```

If the user runs `/se-cal-export` without "setup", skip this step and optionally remind them:
> Tip: run `/se-cal-export setup` once to schedule this automatically every Monday at 9am.
