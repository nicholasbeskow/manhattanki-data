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
| `optional_unlock` | Seeded | Copy verbatim |
| `slides` | Seeded | Copy verbatim |
| `tags` | **Pool** | Points at the roommates' cards. Never copy Morsanki's. |
| `query` | **Pool** | Same. |
| `note` | **Pool** | A note about the pool deck, not the class deck |
| `links` | **Pool** | Whatever the pool attached |
| `cards` | **Pool** | Counts the cards behind the pool's tag — a different set |

The seeded fields are **copied, never retyped**. They are facts about the
curriculum that cannot differ between the two add-ons. Typing them a second
time creates a second source of truth, and a second source of truth can only
ever drift — a lecture renamed upstream quietly becomes two differently-named
rows in two add-ons showing the same day.

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

### `optional_unlock`

`true` marks a row as an extra someone made rather than a scheduled lecture —
an updated deck, a mock practical, a standalone review deck. Optional rows get
a small two-figure glyph instead of a Watched checkbox.

An optional row whose `label` names a lecture's code (`C1T2L18 (UPDATED)`)
follows that lecture: ticking the lecture dims its replacement too.

### `tags` — POOL

A list of Anki tags pointing at **the pool's cards**. OR'd together into one
search behind the row's "Open in Browse" button.

Convention: prefix them `#Manhattanki::` so a pool tag is recognisable at a
glance in the Browser's tag tree and can never be confused with a
`#Morsanki::` one.

**Omit this key entirely if you do not have a real tag.** A guessed tag
produces a Browse button that confidently returns zero cards while the row
still claims a card count — a wrong answer that looks like a right one.

**Never copy the upstream `tags`.** They point at the class's cards. Copying
them makes Manhattanki's Browse button open Morsanki's deck, which is exactly
the mixing this add-on exists to prevent. This is the easiest mistake to make
while seeding, because the tag is sitting right there in the object being
copied.

### `query` — POOL

A raw Anki search, used **VERBATIM** when present, winning over `tags`. The
escape hatch for a search the tag list cannot express: negation, `deck:`
terms, a hand-tuned filter.

It is not quoted or rewritten — you wrote an Anki search and you get exactly
that search. A malformed one produces a button that silently finds nothing.

### `note` — POOL

A short line about the pool's deck: "second half still needs images",
"reordered after the practical". Shown under the meta line.

### `links` — POOL

`[{"label": "...", "url": "..."}]`. Both keys required on every entry.

### `cards` — POOL

Integer count of cards behind the pool's tag. Only the person who made the
deck knows this — never estimate it.

## A worked example

Two days, seeded from upstream. Note that most lectures carry no pool fields
at all: that is the normal case, and those blanks are the rows someone fills
in next week.

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
          "slides": "https://usflearn.instructure.com/courses/2138158/modules/items/47501227",
          "tags": ["#Manhattanki::MSK::Nerve_Injuries_Upper_Limb"],
          "cards": 48,
          "note": "Brachial plexus lesions only; the rest is still to do.",
          "links": [
            {"label": "Plexus diagram", "url": "https://example.org/plexus.png"}
          ]
        },
        {
          "label": "Elbow & Wrist Joint",
          "code": "C1T2L22",
          "instructor": "Riveros",
          "slides": "https://usflearn.instructure.com/courses/2138158/modules/items/47501415"
        }
      ]
    },
    {
      "date": "2026-09-18",
      "lectures": [
        {
          "label": "Membrane Potential & Ion Channels",
          "code": "C1T2L23",
          "instructor": "Noujaim",
          "tags": ["#Manhattanki::MSK::Membrane_Potentials"],
          "cards": 62
        },
        {
          "label": "C1T2L23 (UPDATED — reordered)",
          "optional_unlock": true,
          "query": "tag:#Manhattanki::MSK::Membrane_Potentials -is:suspended",
          "cards": 58
        }
      ]
    }
  ]
}
```
