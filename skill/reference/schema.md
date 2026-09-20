# `schedule.json` — the field reference

This is the file Manhattanki fetches from `manhattanki-data`. Everything the
add-on displays comes from here.

```json
{
  "unlocks": [
    {"date": "2026-09-17", "lectures": [ /* ... */ ]}
  ]
}
```

`unlocks` is sorted by ISO date, ascending. There is **no `exams` key** —
Manhattanki has no countdown, and exam dates belong to Morsanki.

## The two kinds of field

Manhattanki lists the **same lectures on the same days** as Morsanki. It is the
same curriculum. What differs is what the pool made for each one.

So every lecture object is half seeded and half ours:

| Field | Origin | Rule |
|---|---|---|
| `label` | Seeded from `morsanki-data` | Copy verbatim |
| `code` | Seeded | Copy verbatim |
| `instructor` | Seeded | Copy verbatim |
| `slides` | Seeded | Copy verbatim |
| `optional_unlock` | **Pool** | Marks a *pool* extra. Upstream's optional rows are not seeded. |
| `tags` | **Pool** | Points at the pool's cards. Never copy Morsanki's. |
| `query` | **Pool** | Same. |
| `note` | **Pool** | A note about the pool's deck, not the class's |
| `links` | **Pool** | Whatever the pool attached |
| `cards` | **Pool** | Counts the cards behind the pool's tag — a different set |

The seeded fields are **copied, never retyped**. They are facts about the
curriculum that cannot differ between the two add-ons. Typing them a second time
creates a second source of truth, and a second source of truth can only ever
drift — a lecture renamed upstream quietly becomes two differently-named rows in
two add-ons showing the same day.

`scripts/seed_manhattanki.py` in `~/Documents/AnkiUnlocks` does the copying.

Emit keys in this order, so a reseed diffs only where the data changed:
`label`, `code`, `instructor`, `slides`, `optional_unlock`, `cards`, `note`,
`links`, `query`, `tags`.

## Field by field

### `date` (day object, required)

ISO `YYYY-MM-DD`. Anything else fails to parse and the whole day is dropped.

### `label` (required)

The lecture title, as Morsanki has it. This is the row's headline.

### `code`

The curriculum code, e.g. `C1T2L21`. Shown in the muted line under the title,
and searchable. Omit if the upstream lecture has none.

### `instructor`

Surname, as upstream has it. Shown in the muted line, and searchable.

### `slides`

URL to the lecture slides. Makes the title clickable.

### `optional_unlock` — POOL

`true` marks a row as an extra the **pool** made rather than a scheduled
lecture — a reworked deck, a standalone review deck. Optional rows get a small
two-figure glyph instead of a Watched checkbox.

A real JSON boolean, never the string `"true"`: a quoted `"false"` is truthy in
Python and would mark the row optional while reading as the opposite. Omit the
key when false.

**Upstream optional rows are not seeded.** In `morsanki-data` this flag marks a
class-deck artifact — a Box re-export, somebody's alternate deck, the mock
practical — which has no pool equivalent and would sit here as a permanently
blank row describing the other pool's decks.

An optional row whose `label` names a lecture's code (`C1T2L18 (UPDATED)`)
follows that lecture: ticking the lecture dims its replacement too.

### `tags` — POOL

A list of Anki tags pointing at **the pool's cards**. OR'd together into one
search behind the row's "Open in Browse" button.

The convention, already live on the cards and written by the
`manhattan-project-cards` skill, is hierarchical:

```
ManhattanProject::<COURSE>::<TEST>::<LECTURE>
ManhattanProject::C1::T3::L10_Cytokines_and_Complement
```

Course, test and lecture are separate levels — never the flat `C1T3L10` form —
so a whole course or a single test can be selected at once
(`tag:ManhattanProject::C1::T3::*`). One lecture tag carries both the pool's new
cards and the AnKing cards tagged in place, so one tag covers the lecture.

**Every lecture row lists two tags with the same leaf** — the pool's own
`ManhattanProject::C1::T3::L##_Name` and the AnkiHub optional tag
`AnkiHub_Optional::Manhattanki::C1::T3::L##_Name`. They are OR'd, so the button works
both for whoever has the cards locally and for a roommate whose AnKing cards carry only
the optional tag from the Manhattanki tag group. The SKILL's optional-tag section has
the mirror rule.

A lecture nobody has built yet may carry a placeholder pair plus a wildcard `query`
(`tag:ManhattanProject::C1::T3::L20_* OR tag:AnkiHub_Optional::Manhattanki::C1::T3::L20_*`)
and no `cards` count. Replace the placeholder with the real tag once the lecture is built.

**Omit this key entirely if you do not have a real tag.** A guessed tag produces
a Browse button that confidently returns zero cards while the row still claims a
card count — a wrong answer that looks like a right one.

**Never copy the upstream `tags`.** They point at the class's cards. Copying them
makes Manhattanki's Browse button open Morsanki's deck, which is exactly the
mixing this add-on exists to prevent. This is the easiest mistake to make while
seeding, because the tag is sitting right there in the object being copied. Both
the skill and the push watcher refuse any tag containing `Morsanki` or starting
with `AnkiHub_Subdeck::`.

### `query` — POOL

A raw Anki search, used **VERBATIM** when present, winning over `tags`. The
escape hatch for a search the tag list cannot express: negation, `deck:` terms, a
hand-tuned filter.

It is not quoted or rewritten — you wrote an Anki search and you get exactly that
search. A malformed one produces a button that silently finds nothing, so run it
in the Browser first. Keep `tags` populated alongside it: a pool member on an
older add-on ignores `query` and falls back to the tag, which is a broader
button, never a broken one.

A query containing double quotes needs them escaped as `\"` in the file. Whitespace-only counts as absent.

### `note` — POOL

A short line about the pool's deck: "second half still needs images", "reordered
after the practical". Shown under the meta line. Plain text — never a tag, never
a URL. Omit the key entirely when there is nothing to say; never `"note": ""`.

The maintainer's own words. Never invent one.

### `links` — POOL

`[{"label": "...", "url": "..."}]`. Both keys required on every entry, and `url`
must start with `https://` — a link with any other scheme is dropped by the
add-on silently, with the row rendering as though it never had one.

### `cards` — POOL

Integer count of cards behind the pool's tag, measured against live Anki with
`findCards` — cards, not notes, because the add-on shows cards. Never estimate
it, and never carry one over from the class file: the class's count measures a
different set.

Worth reporting alongside it how many are suspended. A lecture tag that swept in
AnKing matches is mostly suspended until somebody unsuspends it, and the count
alone does not show that.

## A worked example

Two days, seeded from upstream. Note that most lectures carry no pool fields at
all: that is the normal case, and those blanks are the rows someone fills in
next week.

```json
{
  "unlocks": [
    {
      "date": "2026-09-17",
      "lectures": [
        {
          "label": "MHC & Antigen Presentation",
          "code": "C1T3L09",
          "instructor": "Pross",
          "slides": "https://usflearn.instructure.com/courses/2138158/modules/items/47501227"
        },
        {
          "label": "Cytokines & Complement",
          "code": "C1T3L10",
          "instructor": "Pross",
          "cards": 96,
          "note": "49 of these are AnKing matches, still suspended.",
          "tags": ["ManhattanProject::C1::T3::L10_Cytokines_and_Complement"]
        }
      ]
    },
    {
      "date": "2026-09-18",
      "lectures": [
        {
          "label": "Introduction to Virology",
          "code": "C1T3L11",
          "instructor": "Pross"
        },
        {
          "label": "C1T3L10 (reordered)",
          "optional_unlock": true,
          "cards": 47,
          "query": "tag:ManhattanProject::C1::T3::L10_Cytokines_and_Complement -is:suspended",
          "tags": ["ManhattanProject::C1::T3::L10_Cytokines_and_Complement"]
        }
      ]
    }
  ]
}
```
