# manhattanki-data

The card pool consumed by the **Manhattanki** Anki add-on. Written by the
`manhattanki-pool` Cowork skill; read by every pool member's Anki at startup and
after each collection sync.

Sibling to [`morsanki-data`](https://github.com/nicholasbeskow/morsanki-data),
which holds the class schedule. This one holds what *we* made.

## schedule.json

```json
{
  "unlocks": [
    {
      "date": "2026-09-17",
      "lectures": [
        {
          "label": "Cytokines & Complement",
          "code": "C1T3L10",
          "instructor": "Pross",
          "cards": 96,
          "tags": ["ManhattanProject::C1::T3::L10_Cytokines_and_Complement"]
        }
      ]
    }
  ]
}
```

Full field reference: `skill/reference/schema.md`.

## The one rule

Every lecture object is half **seeded** and half **ours**:

| Seeded from `morsanki-data`, copy verbatim | Ours, only from the person who made the deck |
|---|---|
| `label`, `code`, `instructor`, `slides` | `tags`, `query`, `note`, `links`, `cards`, `optional_unlock` |

Manhattanki lists the same lectures on the same days as Morsanki — same
curriculum. Retyping a lecture title is how the two quietly stop matching, so the
shells are copied by `AnkiUnlocks/scripts/seed_manhattanki.py`, never typed.

**Never copy a `#Morsanki::` tag into this file.** It points at the class's
cards, and copying it makes Manhattanki's "Open in Browse" open Morsanki's
deck — exactly the mixing this add-on exists to prevent. Pool tags are
`ManhattanProject::<COURSE>::<TEST>::<LECTURE>`.

## Other rules

- `date` is always ISO `YYYY-MM-DD`; `unlocks` is sorted by it, ascending.
- **No `exams` key.** Manhattanki has no countdown — exam dates live in
  `morsanki-data` and one copy is enough.
- Seed the *whole* day, not just the lectures with pool cards. A blank row is
  honest, and it is the row someone fills in next.
- Upstream's `optional_unlock` rows are **not** seeded: each one is a class-deck
  artifact with no pool equivalent. Here the flag marks a pool extra.
- Omit `tags` entirely rather than guessing one. A guessed tag gives a Browse
  button that returns zero cards while the row still claims a count.
- `cards` counts cards, measured in live Anki with `findCards`. Never estimated.
- 2-space indent, trailing newline, so diffs stay readable.

## How a write reaches here

Cowork has no GitHub credentials. A launchd agent on Nick's Mac watches
`/Users/Shared/morsanki-push/drop/` and pushes what lands there:

| Drop file | Lands in |
|---|---|
| `schedule.json` | `morsanki-data/schedule.json` |
| `manhattanki.json` | `manhattanki-data/schedule.json` |
| `manhattanki-skill.json` | `manhattanki-data/skill/*.md` |

Separate clones and separate validators, so a drop for one pool cannot reach the
other's repo.

## Backups

Every write drops a timestamped copy in `backups/`. To back out a bad write,
`git revert` the commit and hit **Refresh** in the add-on's dialog — the add-on
caches the fetch and nothing else, so that is the whole fix.

## Who can write

Everyone in the pool. The skill reads the live file before every write and merges
rather than replaces, so adding to a day a roommate already filled leaves their
entries alone.
