---
name: manhattanki-pool
description: "Maintain schedule.json in the manhattanki-data repo, the shared card pool the Manhattanki Anki add-on reads. Use whenever someone says they made cards, finished a deck, tagged a lecture, or wants to add to, update or fix the Manhattanki pool."
---

# Manhattanki pool

Manhattanki is an Anki add-on showing what the pool has made for each day of the
curriculum. This skill is how anything gets into it.

**The data repo is `manhattanki-data`. The file is `schedule.json`.**

---

## Which pool? Settle this before writing anything

There are **two pools, two repos, two add-ons**, and they share a curriculum:

| | Morsanki | Manhattanki |
|---|---|---|
| What it is | the class deck everyone runs | what Nick and his roommates made |
| Repo | `morsanki-data` | `manhattanki-data` |
| Skill | `morsanki-schedule` | this one |
| Source of truth | the week's Discord posts | the person who made the deck |
| Tags | `#Morsanki::…` | `ManhattanProject::…` |
| `exams` key | yes | **never** |
| Discord | posted weekly | **never posted** |

**Discord posts cover Morsanki only.** Nothing in this skill produces, edits, or
implies a Discord post. If a turn seems to want one, it is a Morsanki turn.

If the request does not name a pool, **ask which one** before writing. The two
files look nearly identical, so a write into the wrong repo validates cleanly,
pushes cleanly, and is only discovered when somebody's Browse button opens the
wrong deck. One question costs a few seconds; that costs an afternoon.

Signals, when the person is not there to ask: a card count they measured, a
lecture *they* carded, "our deck", "the pool", a `ManhattanProject::` tag → this
skill. A week's unlock posts, a tag list from Discord, an exam date, a Box
re-export for the class → `morsanki-schedule`.

**Never write both repos in one turn without saying so explicitly**, and never
carry a value from one file into the other except through the seeder below.

---

## The one idea

Manhattanki lists the **same lectures on the same days** as Morsanki — same
curriculum. What differs is what the pool made for each one.

So a day is half **seeded** from `morsanki-data` and half **ours**:

| Seeded, copied verbatim | Ours, only from the person |
|---|---|
| `label`, `code`, `instructor`, `slides` | `tags`, `query`, `note`, `links`, `cards`, `optional_unlock` |

Read the field reference before your first write. It documents every field, and
it is the half of this skill that is not in this file:

- installed beside this one as `reference/schema.md`, or
- `curl -s https://raw.githubusercontent.com/nicholasbeskow/manhattanki-data/main/skill/reference/schema.md`
  when the Cowork copy is a lone `SKILL.md` (a proposed skill carries one file).

**`optional_unlock` is a pool field, not a seeded one.** Upstream it marks a
class-deck artifact — a Box re-export, somebody's alternate deck, the mock
practical. Those are Morsanki's, they have no pool equivalent, and seeding them
leaves a row that stays blank forever while describing the other pool's decks.
The seeder skips them. Manhattanki's optional rows are for the **pool's** extras.

---

## Tags are `ManhattanProject::`, hierarchical

The live convention, written by the `manhattan-project-cards` skill and already
on the cards:

```
ManhattanProject::C1::T3::L10_Cytokines_and_Complement
```

Course, test, lecture — split into three levels, never the flat `C1T3L10` form,
so a whole course or a single test selects at once
(`tag:ManhattanProject::C1::T3::*`). One tag covers a lecture: it carries both
the pool's new cards and the AnKing cards tagged in place.

**Never copy a tag from `morsanki-data`.** It points at the class's cards, and
copying it makes Manhattanki's Browse button open Morsanki's deck — exactly the
mixing this add-on exists to prevent. This is the easiest mistake to make while
seeding, because the tag is sitting right there in the object being copied. The
seeder never copies one and the leak check below catches it if anything else
does.

**Never invent a tag.** If the person did not give one, omit `tags` entirely. A
guessed tag produces a Browse button that confidently returns zero cards while
the row still claims a card count — a wrong answer wearing the costume of a
right one.

---

## Procedure

1. **Confirm the pool.** See above.

2. **Read the live file through the GitHub API, never the raw URL.**
   `raw.githubusercontent.com` is a Fastly edge cache serving up to five minutes
   stale, and neither a `?t=` buster nor `Cache-Control: no-cache` defeats it.
   The seeder already reads both repos through the API.

3. **Seed with the script, never by hand.**

   ```bash
   cd ~/Documents/AnkiUnlocks
   python3 scripts/seed_manhattanki.py > /tmp/manhattanki.json
   ```

   It fetches both repos, copies each lecture's `label`, `code`, `instructor` and
   `slides` verbatim, skips upstream optional rows, merges pool fields already
   published, and sorts by date. Retyping a lecture title is how the two add-ons
   quietly stop matching — so do not retype one.

   `--from YYYY-MM-DD` limits reseeding to days on or after a date; earlier days
   are passed through untouched.

4. **Seed the whole day, not just the lectures with pool cards.** The dialog
   answers "what was on today, and what have we made for it". A lecture with no
   pool deck is a useful, honest blank — and it is the row somebody fills in next
   week.

5. **If a date is missing upstream, say so and stop.** Do not invent shells. A
   date Morsanki does not know is either a typo or a day with no class.

6. **Attach the pool fields the person gave you**, on the matching lecture.
   Match on `code` when it has one, otherwise on `label`.

7. **Measure `cards` against live Anki; never estimate it.** Use `findCards`, not
   `findNotes` — the add-on shows cards. Report how many are suspended, because
   an AnKing-heavy lecture tag is mostly suspended until somebody unsuspends it.

   ```bash
   cd ~/Documents/AnkiUnlocks && python3 -c "
   import sys; sys.path.insert(0,'scripts')
   from anki_client import request
   T='ManhattanProject::C1::T3::L10_Cytokines_and_Complement'
   print(len(request('findCards', query='tag:'+T)),
         len(request('findCards', query='tag:'+T+' is:suspended')))"
   ```

   If Anki does not answer, **stop** and say so. Never fall back to a cached or
   estimated count.

8. **Merge, never replace.** Several people share this repo. A date already
   present keeps every pool field this turn is not changing; adding a deck to a
   day a roommate already filled must not wipe their entries. The seeder does
   this — do not hand-assemble a day around it.

9. **Never write an `exams` key.** Manhattanki has no countdown. Exam dates
   belong to Morsanki, and a second copy is a second source of truth.

10. **Never invent a `note`.** A note is the maintainer's own words. If one
    seems useful, ask for the wording.

11. **Validate, publish, verify** — the three sections below, in that order.

12. **Report back**: the date(s) written, how many shells were seeded, how many
    carry pool decks, what was measured against live Anki, and the commit.

---

## Validation

Run both before every publish.

```python
import datetime as dt, json, pathlib

d = json.loads(pathlib.Path("/tmp/manhattanki.json").read_text())
assert "exams" not in d, "Manhattanki has no countdown"
dates = []
for day in d["unlocks"]:
    dates.append(dt.date.fromisoformat(day["date"]))
    for lec in day["lectures"]:
        assert isinstance(lec.get("label"), str) and lec["label"].strip()
        for t in lec.get("tags", []):
            assert isinstance(t, str) and t.strip()
        if "cards" in lec:
            assert isinstance(lec["cards"], int) and not isinstance(lec["cards"], bool)
        if "optional_unlock" in lec:
            assert isinstance(lec["optional_unlock"], bool)   # never the string "true"
        for link in lec.get("links", []):
            assert link.get("label") and link["url"].startswith("https://")
assert dates == sorted(dates), "unlocks must be sorted by date"
print(f"ok: {len(dates)} days")
```

And the leak check, which catches the one mistake seeding invites:

```python
import json, pathlib

d = json.loads(pathlib.Path("/tmp/manhattanki.json").read_text())
leaked = sorted(t for day in d["unlocks"] for lec in day["lectures"]
                for t in lec.get("tags", [])
                if "Morsanki" in t or t.startswith("AnkiHub_Subdeck::"))
assert not leaked, f"class tags copied into the pool: {leaked}"
print("ok: no class tags leaked into the pool")
```

The push watcher runs the same two checks again on the Mac and refuses the drop
if either fails, so a leak cannot reach the repo even if this step is skipped.

---

## Publishing

**Cowork's folder bridge has no GitHub credentials** — `gh` and the git identity
live on the Mac, not in the bridge VM. A clone works (the repo is public); a push
has nothing to authenticate with. The cloud container has no GitHub access at
all. Never try to push from either, and never ask for a token.

A launchd agent watches `/Users/Shared/morsanki-push/drop/` and pushes whatever
lands there using Nick's own credentials. It handles both pools from one folder:

| Drop file | Lands in |
|---|---|
| `drop/schedule.json` | `morsanki-data/schedule.json` |
| `drop/manhattanki.json` | `manhattanki-data/schedule.json` |
| `drop/manhattanki-skill.json` | `manhattanki-data/skill/*.md` |

Each has its own validator and its own clone, so a drop for one pool can never
reach the other's repo.

Write, then rename, so the watcher can never see a partial file:

```bash
cp /tmp/manhattanki.json /Users/Shared/morsanki-push/drop/.tmp
mv /Users/Shared/morsanki-push/drop/.tmp /Users/Shared/morsanki-push/drop/manhattanki.json
sleep 10 && tail -20 /Users/Shared/morsanki-push/log
```

That folder is often **not** one of the session's connected folders. Request it
with `device_request_folder_access` on `/Users/Shared/morsanki-push`.

Always read the log and report what it said. **A file sitting in `failed/` means
the push did not happen** — read the log, fix the cause, drop again. Never edit
the clones in `repo/` or `repo-manhattanki/` directly. The log's timestamps are
Mac-local while the bridge shell reports UTC, so a brand-new entry can look four
hours old; match on the commit hash, not the clock.

### Publishing an edit to this skill

The repo is the **single source of truth** for this skill — `skill/SKILL.md`,
`skill/README.md`, `skill/reference/schema.md`. A Cowork-installed copy is a
copy; when you change one, change the repo too, in the same session, or the two
quietly stop matching.

```bash
python3 - <<'PY' > /tmp/bundle.json
import json, pathlib
root = pathlib.Path.home() / "Documents/AnkiUnlocks/manhattanki_skill"
files = {str(p.relative_to(root)): p.read_text()
         for p in root.rglob("*.md")}
json.dump({"files": files}, open("/dev/stdout", "w"), indent=2)
PY
cp /tmp/bundle.json /Users/Shared/morsanki-push/drop/.tmp
mv /Users/Shared/morsanki-push/drop/.tmp /Users/Shared/morsanki-push/drop/manhattanki-skill.json
```

The watcher refuses any path that is not `skill/**.md` or `README.md`.

---

## Verify, afterwards

Through the **GitHub API**, never the raw URL:

```bash
curl -s https://api.github.com/repos/nicholasbeskow/manhattanki-data/contents/schedule.json \
  | python3 -c "import json,sys,base64; print(base64.b64decode(json.load(sys.stdin)['content']).decode())"
```

The API itself lags a second or two behind the watcher's commit line, so checking
the instant the log prints can return the pre-push content and look like a
dropped push. Wait a few seconds and re-check before concluding anything.

Pool members' add-ons hit the raw URL, so a refresh within ~5 minutes of a push
shows the previous data and the next one corrects it. That is a CDN property, not
a bug — do not go looking for one, and do not republish to force it.

---

## Backing out a bad write

`git revert` the commit in `manhattanki-data` and hit **Refresh** in the add-on's
dialog. That is the whole fix — the add-on caches the fetch and nothing else.
Every write also drops a timestamped copy in `backups/`.

---

## What this skill must never do

- **Touch `morsanki-data`.** Read-only. The class schedule is not the pool's to
  edit.
- **Produce or imply a Discord post.** Those cover Morsanki only.
- **Create, edit, suspend or delete cards** in anyone's collection. This writes
  one JSON file and nothing else.
- **Write a `cards` count it did not measure** against live Anki.
- **Copy a `#Morsanki::` tag, or any upstream `tags`, `query`, `note`, `links` or
  `cards`,** into this file.
- **Write an `exams` key.**
- **Write an add-on setting.** `sits_below`, `show_watched` and the rest live in
  each person's own Anki config, not in shared data.
- **Push from the bridge or the cloud container.** Use the drop folder.
- **Verify a push against the raw URL.** Use the API.
