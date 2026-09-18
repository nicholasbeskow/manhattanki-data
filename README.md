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
          "label": "Nerve Injuries of Upper Limb",
          "code": "C1T2L21",
          "instructor": "Kothari",
          "tags": ["#Manhattanki::MSK::Nerve_Injuries_Upper_Limb"],
          "cards": 48
        }
      ]
    }
  ]
}
```

Full field reference: `cowork/reference/schema.md` in the add-on repo.

## The one rule

Every lecture object is half **seeded** and half **ours**:

| Seeded from `morsanki-data`, copy verbatim | Ours, only from the person who made the deck |
|---|---|
| `label`, `code`, `instructor`, `optional_unlock`, `slides` | `tags`, `query`, `note`, `links`, `cards` |

Manhattanki lists the same lectures on the same days as Morsanki — same
curriculum. Retyping a lecture title is how the two quietly stop matching, so
the shells are copied, never typed.

**Never copy a `#Morsanki::` tag into this file.** It points at the class's
cards, and copying it makes Manhattanki's "Open in Browse" open Morsanki's
deck — exactly the mixing this add-on exists to prevent. Pool tags are prefixed
`#Manhattanki::`.

## Other rules

- `date` is always ISO `YYYY-MM-DD`; `unlocks` is sorted by it, ascending.
- **No `exams` key.** Manhattanki has no countdown — exam dates live in
  `morsanki-data` and one copy is enough.
- Seed the *whole* day, not just the lectures with pool cards. A blank row is
  honest, and it is the row someone fills in next.
- Omit `tags` entirely rather than guessing one. A guessed tag gives a Browse
  button that returns zero cards while the row still claims a count.
- 2-space indent, trailing newline, so diffs stay readable.

## Backups

Every write drops a timestamped copy in `backups/`. To back out a bad write,
`git revert` the commit and hit **Refresh** in the add-on's dialog — the add-on
caches the fetch and nothing else, so that is the whole fix.

## Who can write

Everyone in the pool. Pull before you write; the skill merges rather than
replaces, so adding to a day a roommate already filled leaves their entries
alone.
