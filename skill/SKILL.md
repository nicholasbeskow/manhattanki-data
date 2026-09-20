---
name: manhattanki-pool
description: "Maintain schedule.json in the manhattanki-data repo, the shared card pool the Manhattanki Anki add-on reads, and publish lecture .apkg packages as GitHub Release assets. Use whenever someone says they made cards, finished a deck, tagged a lecture, or wants to add to, update or fix the Manhattanki pool."
---

# Manhattanki pool

Manhattanki is an Anki add-on showing what the pool has made for each day of the
curriculum. This skill is how anything gets into it.

**The data repo is `manhattanki-data`. The file is `schedule.json`.**

---

## Step 0 — Request folder access, every run

**Nick's standing instruction: request these folders at the start of every run.
Do it before anything else and without asking him first.** Every step of this
skill depends on them, and they are usually not connected to the session.

Make **one** `device_request_folder_access` call with all three folders:

```json
{
  "paths": [
    "/Users/beskow/Documents/Manhattan Project",
    "/Users/beskow/Documents/AnkiUnlocks",
    "/Users/Shared/morsanki-push"
  ],
  "reason": "Manhattanki pool: read the lecture .apkg exports, run the seeder and the Anki count helper, and drop the deck + schedule into the push folder."
}
```

| Folder | Why |
|---|---|
| `Documents/Manhattan Project` | `_exports/<CODE>.apkg`, the decks Nick exported |
| `Documents/AnkiUnlocks` | `scripts/seed_manhattanki.py`, `scripts/anki_client.py`, `restore_schedule.py`, `manhattanki_skill/` |
| `/Users/Shared/morsanki-push` | the drop folder and the watcher's `log` |

- Request all three even if one looks connected already. Asking again for a folder
  that is already granted costs nothing.
- Nick expects this request, so it is a normal step, not overreach.
- **If the request is denied or blocked** (for example by auto mode's permission
  check), do not work around it. Stop and tell Nick the three folders, and give him
  two ways to fix it: add them with **"Add folder"** in the desktop app, or turn
  off auto mode for the task so the prompt reaches him. Then wait.
- Once access is granted, `device_bash` mounts each folder by name under
  `$HOME/mnt/` (`$HOME/mnt/Manhattan Project`, `$HOME/mnt/AnkiUnlocks`,
  `$HOME/mnt/morsanki-push`). Use those paths in the commands below wherever this
  file writes `~/Documents/...` or `/Users/Shared/morsanki-push`.

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

**A card can legitimately carry more than one lecture tag.** Tags are many-to-many;
a concept two lectures on the same test both teach belongs to both. So a lecture's
`cards` count is not additive with its neighbours' — each row honestly reports the
total its own tag returns, and the same note can sit behind two rows. Do not "fix"
that, and never ask for a tag to be removed to make the numbers sum.

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

The **only** exception is the placeholder pair for a lecture nobody has built yet,
which carries no `cards` count and a wildcard `query` — see the optional-tag section
below.

---

## The optional tag — how a lecture reaches subscribed roommates

Every lecture has **two** tags with the **same leaf**:

```
ManhattanProject::C1::T3::L10_Cytokines_and_Complement
AnkiHub_Optional::Manhattanki::C1::T3::L10_Cytokines_and_Complement
```

They exist because the pool's cards reach a roommate by two different routes:

| Cards | Route | Tag that matters |
|---|---|---|
| In-house (`Manhattan Project::<CODE>` deck) | the Manhattan Project deck on AnkiHub | `AnkiHub_Subdeck::Manhattan_Project::<CODE>`, plus a note type belonging to that deck |
| AnKing cards tagged in place | the **Manhattanki** optional tag group on the AnKing deck | `AnkiHub_Optional::Manhattanki::…` |

**A `ManhattanProject::` tag on an AnKing note is local and reaches nobody.** Neither
does anything else edited on an AnKing note — `Lecture Notes`, `Extra`, art. Only the
optional tag crosses, and only when it is submitted: Browser → select →
**AnkiHub → Suggest Optional Tags**, every time notes are newly tagged. Nothing about
this is automatic, so a lecture tagged and never suggested is a lecture only Nick has.

### The mirror rule

**Whatever carries one tag carries the other, with the identical leaf.** Tag an AnKing
card into a lecture → add the optional tag in the same pass. Untag one → untag the
other, and suggest that removal on AnkiHub, or the roommates keep a card Nick cut.

Check it before reporting a lecture done — this should return nothing:

```
tag:AnkiHub_Optional::Manhattanki::C1::T3::L##_* -tag:ManhattanProject::C1::T3::L##_*
```

and so should the reverse, restricted to AnKing (`-"deck:Manhattan Project"`).

In-house notes need only the `ManhattanProject::` tag and the subdeck tag; do **not**
give them an optional tag. Optional tag groups belong to the AnKing deck.

### Never rename these tags in the Browser

Find&Replace or a rename in the tag sidebar **takes the parents with it**, so renaming
`AnkiHub_Optional::Manhattanki` collapses the whole family into whatever it is renamed
to and silently merges it with the `ManhattanProject::` tags — 232 notes, in one
keystroke, on 2026-09-18. Retag by adding the tag you want and removing the one you
don't, never by renaming a parent.

### In `schedule.json`

A lecture row lists **both** tags; the add-on ORs them, so Browse works for whoever
gets the cards by either route:

```json
"tags": ["ManhattanProject::C1::T3::L10_Cytokines_and_Complement",
         "AnkiHub_Optional::Manhattanki::C1::T3::L10_Cytokines_and_Complement"]
```

**A not-yet-built lecture may carry a placeholder pair** — the one sanctioned exception
to "never invent a tag" below, and only when the row has no `cards` count to be wrong
about. Derive the leaf from the class label and add a wildcard `query` so the button
still finds the cards when the real leaf turns out different:

```json
"query": "tag:ManhattanProject::C1::T3::L20_* OR tag:AnkiHub_Optional::Manhattanki::C1::T3::L20_*"
```

Replace the placeholder with the real tag when the lecture is built.

---

## Procedure

0. **Request folder access.** See Step 0 above. Every run, first.

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
   `findNotes` — the add-on shows cards. It is simply the total the tag returns,
   AnKing and in-house together, exactly what Browse shows. No splits, no
   suspended breakdown.

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

   **Take the count immediately before the drop.** A card-building session in
   another window changes it mid-turn — the same tag query returned 102 and then
   74 minutes apart during the C1T3L07 run. Re-measure if the number is more than
   a few minutes old.

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
| `drop/manhattanki-deck.apkg` + `drop/manhattanki-deck.json` | a GitHub Release asset on `manhattanki-data` |

Each has its own validator and its own clone, so a drop for one pool can never
reach the other's repo.

Write, then rename, so the watcher can never see a partial file:

```bash
cp /tmp/manhattanki.json /Users/Shared/morsanki-push/drop/.tmp
mv /Users/Shared/morsanki-push/drop/.tmp /Users/Shared/morsanki-push/drop/manhattanki.json
sleep 10 && tail -20 /Users/Shared/morsanki-push/log
```

That folder is requested in Step 0 on every run; if it is somehow still missing,
request it again with `device_request_folder_access` on `/Users/Shared/morsanki-push`.

Always read the log and report what it said. **A file sitting in `failed/` means
the push did not happen** — read the log, fix the cause, drop again. Never edit
the clones in `repo/` or `repo-manhattanki/` directly. The log's timestamps are
Mac-local while the bridge shell reports UTC, so a brand-new entry can look four
hours old; match on the commit hash, not the clock.

**Never run a git command against `repo-manhattanki/` from the bridge.** The
bridge cannot unlink, so an aborted `git fetch`/`reset` leaves `.git/index.lock`
behind and every later watcher run dies with "another git process seems to be
running" — and the bridge cannot delete the lock either.
`ensure_manhattanki_clone` clears a lock older than a minute when no git process
is running, so a stuck drop recovers on the next pass, but the drop that hit the
lock has already been moved to `failed/` and must be dropped again. Reading the
clone's files is fine; only git commands are the hazard.

### Publishing a deck package

This is how a lecture's cards reach the roommates. **Box is retired** — no shared
links, no uploads, and the Box MCP cannot carry a binary anyway. AnkiHub is also
out: Nick is not paying for a subscription.

The export is the one manual step. AnkiConnect is reachable (the Anki MCP server
calls it on every tool call) but the server exposes a fixed action list that
**omits `exportPackage`**, so ask Nick to select the lecture tag in the Browser
and export to `Manhattan Project/_exports/<CODE>.apkg`. Take a real Anki export,
never a `genanki`-generated one: Anki preserves note GUIDs, so importing applies
the lecture tag to the roommates' **existing AnKing cards**. A generated package
carries no GUIDs (`notesInfo` does not return them) and imports as new notes,
losing the entire AnKing half of the lecture.

Then drop the package with a sidecar naming the lecture:

```bash
D=/Users/Shared/morsanki-push/drop
cp "$HOME/mnt/Manhattan Project/_exports/C1T3L10.apkg" "$D/.t.apkg"
printf '%s\n' '{"code":"C1T3L10","label":"Cytokines and Complement"}' > "$D/.t.json"
mv "$D/.t.json" "$D/manhattanki-deck.json"
mv "$D/.t.apkg" "$D/manhattanki-deck.apkg"   # the .apkg LAST -- it is the trigger
sleep 40 && tail -8 /Users/Shared/morsanki-push/log
```

The handler validates the code against `C\d+T\d+L\d+`, confirms the file is
actually a zip, then publishes it as a release asset under the tag `deck-<CODE>`
and writes the URL to `$BASE/last-deck-url`:

```
https://github.com/nicholasbeskow/manhattanki-data/releases/download/deck-C1T3L10/C1T3L10.apkg
```

Re-publishing the same code clobbers the asset, so **the URL is stable** — set it
in `schedule.json` once and later re-exports need no schedule change. Release
assets keep the binaries out of git history; the fallback path commits
`decks/<CODE>.apkg` if `gh` ever breaks, and the log says which path it took. A
20–35 MB package is normal, because the export carries the slide images and the
AnKing media for the tagged cards.

A sidecar of `{"code": "C1T3L10", "retire": true}` with no `.apkg` deletes the
release and its tag.

Then add the link to the lecture in `schedule.json`, label **"Download here
first"** to match the existing rows:

```json
"links": [{"label": "Download here first",
           "url": "https://github.com/nicholasbeskow/manhattanki-data/releases/download/deck-C1T3L10/C1T3L10.apkg"}]
```

The add-on hands links to `aqt.utils.openLink`, so it opens in the browser and
downloads — it does not import. Nothing about the URL needs to be an API path.

An export carries whatever suspension state the cards had at export time. Nick's
call: the roommates sort that out themselves — do not warn about it or hold an
export back over it.

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

For a deck package, also confirm the asset actually serves:

```bash
curl -sIL -o /dev/null -w '%{http_code}\n' \
  https://github.com/nicholasbeskow/manhattanki-data/releases/download/deck-C1T3L10/C1T3L10.apkg
```

Pool members' add-ons hit the raw URL, so a refresh within ~5 minutes of a push
shows the previous data and the next one corrects it. That is a CDN property, not
a bug — do not go looking for one, and do not republish to force it.

---

## Backing out a bad write

Nothing is ever lost: every push is a commit, and the watcher writes a
timestamped copy into `backups/` before each one.

```bash
cd ~/Documents/AnkiUnlocks
python3 scripts/restore_schedule.py --pool manhattanki --list
python3 scripts/restore_schedule.py --pool manhattanki --show a1b2c3d
python3 scripts/restore_schedule.py --pool manhattanki --restore a1b2c3d --reason "..."
```

`--show` says what restoring would remove and bring back before anything moves;
`--restore` drops it and the watcher pushes it as a **new** commit, so the
history it undoes stays readable. `git revert` in the repo does the same thing by
hand.

Either way, hit **Refresh** in the add-on's dialog afterwards. That is the whole
fix — the add-on caches the fetch and nothing else, so there is no local state to
clean up and nothing to uninstall.

---

## What this skill must never do

- **Skip the Step 0 folder request**, or work around a denied one.
- **Touch `morsanki-data`.** Read-only. The class schedule is not the pool's to
  edit.
- **Produce or imply a Discord post.** Those cover Morsanki only.
- **Create, edit, suspend or delete cards** in anyone's collection. This writes
  one JSON file and publishes one release asset.
- **Write a `cards` count it did not measure** against live Anki.
- **Copy a `#Morsanki::` tag, or any upstream `tags`, `query`, `note`, `links` or
  `cards`,** into this file.
- **Write an `exams` key.**
- **Write an add-on setting.** `sits_below`, `show_watched` and the rest live in
  each person's own Anki config, not in shared data.
- **Push from the bridge or the cloud container.** Use the drop folder.
- **Run any git command against the clones from the bridge.** It leaves a lock
  the bridge cannot remove.
- **Upload anything to Box, or suggest it.** Retired.
- **Rename a `ManhattanProject::` or `AnkiHub_Optional::` tag in the Browser.** A
  rename takes the parents with it and merges the two families.
- **Add a `ManhattanProject::` tag to an AnKing note without the matching optional
  tag** (or remove one without the other).
- **Verify a push against the raw URL.** Use the API.
