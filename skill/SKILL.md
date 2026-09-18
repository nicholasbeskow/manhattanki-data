---
name: manhattanki-pool
description: Use when someone says they made Anki cards, finished a deck, tagged a lecture, or wants to add to / update / fix the Manhattanki card pool. Writes schedule.json in the manhattanki-data repo, seeding each day's lecture list from morsanki-data so nobody retypes the schedule.
---

# Manhattanki pool

Manhattanki is an Anki add-on showing what the pool has made for each day of
the curriculum. This skill is how anything gets into it.

**The data repo is `manhattanki-data`. The file is `schedule.json`.**

## The one idea

Manhattanki lists the **same lectures on the same days** as Morsanki — same
curriculum. What differs is what the pool made for each one.

So a day is half **seeded** from `morsanki-data` and half **ours**:

| Seeded, copy verbatim | Ours, only from the person |
|---|---|
| `label`, `code`, `instructor`, `optional_unlock`, `slides` | `tags`, `query`, `note`, `links`, `cards` |

Read `reference/schema.md` before your first write. It documents every field.

## Procedure

1. **Pull first.** Clone or `git pull` `manhattanki-data`. Several people share
   this repo; writing from a stale copy is how one person's afternoon gets
   silently reverted.

2. **Read the existing `schedule.json`** before writing anything. Work out the
   target date from what the person said ("today", "Tuesday", an explicit
   date). If it is ambiguous, ask — a deck filed under the wrong day is worse
   than one filed a minute later.

3. **Seed the day's lecture shells from `morsanki-data`, never by hand.** Fetch:

   ```
   https://raw.githubusercontent.com/nicholasbeskow/morsanki-data/main/schedule.json
   ```

   Find the same date. For each lecture, carry over `label`, `code`,
   `instructor`, `optional_unlock`, and `slides` **verbatim**. Drop `tags`,
   `query`, `note`, `links`, and `cards` — those describe the class's decks,
   not the pool's.

   Retyping a lecture title is how the two add-ons quietly stop matching.

4. **Seed the whole day, not just the lectures with pool cards.** The dialog
   answers "what was on today, and what have we made for it". A lecture with no
   pool deck is a useful, honest blank — and it is the row somebody fills in
   next week. Only skip a day entirely if upstream has no lectures for it.

5. **If the date is missing upstream, say so and stop.** Do not invent shells.
   A date Morsanki does not know is either a typo or a day with no class, and
   guessing produces a row nobody can trace back to anything.

6. **Merge, never replace.** If the date is already in `manhattanki-data`,
   update matching lectures in place — matched on `code` when present,
   otherwise on `label` — and preserve every pool field already there that this
   turn is not changing. Someone adding a deck to a day a roommate already
   filled must not wipe their entries.

7. **Keep the order.** `unlocks` sorted by ISO date ascending; each day's
   `lectures` in upstream order, so the dialog reads like the day did.

8. **Never write an `exams` key.** Manhattanki has no countdown. Exam dates
   belong to Morsanki, and a second copy is a second source of truth.

9. **Never invent a tag.** If the person did not give one, omit `tags`
   entirely. A guessed tag produces a Browse button that confidently returns
   zero cards while the row still claims a card count — a wrong answer wearing
   the costume of a right one.

10. **Never copy a tag from `morsanki-data`.** It points at the class's cards.
    Copying it makes Manhattanki's Browse button open Morsanki's deck, which is
    exactly the mixing this add-on exists to prevent. This is the single
    easiest mistake to make while seeding, because the tag is sitting right
    there in the object you are copying. Pool tags are prefixed
    `#Manhattanki::` by convention — if a tag you are about to write starts
    with `#Morsanki::`, you have made this mistake.

11. **Back up before you overwrite.** Copy the current `schedule.json` to
    `backups/schedule-YYYY-MM-DD-HHMM.json` in the same commit. Cheap, and it
    means a bad write is recoverable even by someone who does not want to read
    a git log.

12. **Validate, then commit.** Run both checks below. Write the file with
    2-space indent and a trailing newline so diffs stay readable. Commit with a
    message naming the date and what was added, and push.

13. **Report back**: the date written, how many shells were seeded, how many
    carry pool decks, and the commit URL.

## Validation

Run this before every commit:

```python
import datetime as dt, json, pathlib

data = json.loads(pathlib.Path("schedule.json").read_text())
assert "exams" not in data, "Manhattanki has no countdown"
dates = []
for day in data["unlocks"]:
    d = dt.date.fromisoformat(day["date"])
    dates.append(d)
    for lec in day["lectures"]:
        assert isinstance(lec.get("label"), str) and lec["label"].strip()
        if "tags" in lec:
            assert all(isinstance(t, str) and t.strip() for t in lec["tags"])
        if "cards" in lec:
            assert isinstance(lec["cards"], int)
        for link in lec.get("links", []):
            assert link.get("label") and link.get("url")
assert dates == sorted(dates), "unlocks must be sorted by date"
print(f"ok: {len(dates)} days")
```

And this one after seeding, which catches the mistake step 10 warns about:

```python
import json, pathlib, urllib.request

POOL = json.loads(pathlib.Path("schedule.json").read_text())
with urllib.request.urlopen(
        "https://raw.githubusercontent.com/nicholasbeskow/"
        "morsanki-data/main/schedule.json") as r:
    UPSTREAM = json.load(r)

class_tags = {
    t
    for day in UPSTREAM.get("unlocks", [])
    for lec in day.get("lectures", [])
    for t in lec.get("tags", [])
}
leaked = sorted(
    t
    for day in POOL["unlocks"]
    for lec in day["lectures"]
    for t in lec.get("tags", [])
    if t in class_tags or t.startswith("#Morsanki::")
)
assert not leaked, f"class tags copied into the pool: {leaked}"
print("ok: no class tags leaked into the pool")
```

## What this skill must never do

- **Touch `morsanki-data`.** Read only. The class schedule is not ours to edit.
- **Create, edit, or delete cards** in anyone's Anki collection. This writes one
  JSON file and nothing else.
- **Write a `cards` count it was not given.** The count describes the pool's
  deck, and only the person who made it knows the number.
- **Write an add-on setting.** `sits_below`, `show_watched` and the rest live in
  each person's own Anki config, not in shared data.
