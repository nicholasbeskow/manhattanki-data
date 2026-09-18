---
name: morsanki-schedule
description: "Maintain schedule.json in the morsanki-data repo, the shared file every classmate's MorsAnki Anki addon reads. Use whenever a week's Anki unlock posts have been built for Discord, or when exam dates need adding or correcting. For the Manhattanki pool — cards Nick or his roommates made, ManhattanProject:: tags, no Discord — use manhattanki-pool instead."
---

# MorsAnki schedule maintenance

`schedule.json` lives in **`nicholasbeskow/morsanki-data`** (branch `main`) and is
fetched by every classmate's MorsAnki addon at startup and after each sync. It
feeds two features: the deck-browser exam countdown and the daily unlocks dialog.

```json
{ "exams": [...], "unlocks": [...] }
```

Both keys must always be present. `unlocks` comes from the week's Discord posts.
`exams` comes from the study tracker. They are maintained independently and must
never clobber each other.

**The audience is the whole class, not Nick.** Most classmates run the stock
AnkiHub tags and never received his re-tags — he is not a deck maintainer and the
current one is inactive. Where a representation choice is open, it is settled by
what works in *their* collection, not his. They also update the addon at their
own pace, so a field a new version understands must degrade gracefully on an old
one.

---

## There are two pools now. Settle which one before writing.

| | Morsanki | Manhattanki |
|---|---|---|
| What it is | the class deck everyone runs | what Nick and his roommates made |
| Repo | `morsanki-data` | `manhattanki-data` |
| Skill | **this one** | `manhattanki-pool` |
| Source of truth | the week's Discord posts | the person who made the deck |
| Tags | `#Morsanki::…` | `ManhattanProject::…` |
| `exams` key | yes | never |
| Discord | posted weekly | never posted |

**Discord posts cover Morsanki only**, and always will. A request about a Discord
post, an unlock week, an exam date, or a Box re-export for the class is a
Morsanki turn and belongs here.

A request about cards *Nick or a roommate made* — "I carded that lecture", "our
deck", "the pool", a `ManhattanProject::` tag, a count he measured himself — is a
Manhattanki turn. **Stop and hand it to `manhattanki-pool`.** Do not write a pool
deck into this file under any circumstance.

If a turn does not name a pool, **ask**. The two files look nearly identical, so
a write into the wrong one validates cleanly, pushes cleanly, and is discovered
only when somebody's Browse button opens the wrong deck. One question costs
seconds; that costs an afternoon.

Never write both repos in one turn without saying so explicitly.

**A pool tag can never appear in this file.** The push watcher now refuses any
drop containing a tag with `ManhattanProject` or `Manhattanki` in it, so the
mistake cannot reach the class's add-on — but the refusal is the backstop, not
the plan.

---

**The published file is the memory.** Nothing carries between chats except what
is in `schedule.json` itself. Always start from the live file rather than from
anything a conversation claims is there.

---

## The three things that fail silently

The addon drops bad data without a word, and GitHub serves stale data without a
word, so none of these announces itself:

- **A link whose `url` does not start with `https://` simply vanishes.** No
  warning, no error — the row just renders without it. `http`, `file:`,
  `javascript:`, and bare strings with no scheme are all dropped.
- **`"optional_unlock": "true"` as a string is truthy in Python**, so it appears
  to work — then breaks the day someone writes `"false"`, which is also truthy.
  Write a real JSON boolean, never a quoted one.
- **The raw URL can be up to five minutes stale, and nothing defeats it.**
  Measured 2026-09-06: `raw.githubusercontent.com` served the old file right
  after a push, with `x-cache: HIT` and `max-age=300`, ignoring both a `?t=`
  cache-buster and a `Cache-Control: no-cache` request header. **Verify every
  push against the GitHub API, never the raw URL** — the raw URL will tell you
  your own push did not land when it did:

  ```bash
  curl -s https://api.github.com/repos/nicholasbeskow/morsanki-data/contents/schedule.json \
    | python3 -c "import json,sys,base64; print(base64.b64decode(json.load(sys.stdin)['content']).decode())"
  ```

  Classmates' addons hit the raw URL, so a sync within ~5 minutes of a push shows
  the previous data and the next sync corrects it. That is a CDN property, not a
  bug in the fetch code — do not go looking for one.

  **This includes Nick's own addon.** When he says a change did not take, or
  sends a screenshot of the old version, check the raw URL's `source-age` header
  before touching anything. Under 300 it is simply cached, and clicking refresh
  does not help — the cache is Fastly's and his addon re-requests the same edge
  copy. Say so and wait it out; do not republish.

  ```bash
  curl -s -D- -o /tmp/raw.json \
    https://raw.githubusercontent.com/nicholasbeskow/morsanki-data/main/schedule.json \
    | grep -i -E "x-cache|source-age"
  ```

  A fourth, smaller one: the **GitHub API itself lags a second or two behind the
  watcher's commit line**. Checking the instant the log prints can return the
  pre-push content and look like the push was dropped. Wait a few seconds and
  re-check before concluding anything.

---

## The lecture schema

`label` is required. `code`, `instructor`, `cards`, `note`, `optional_unlock`,
`links`, and `query` are optional. `tags` is required and may be empty.

```json
{
  "date": "2026-09-09",
  "lectures": [
    {
      "label": "Skeletal Muscle",
      "code": "C1T2L25",
      "instructor": "Scallan",
      "cards": 111,
      "note": "Redownload — I fixed 12 cloze cards",
      "links": [
        {"label": "Updated deck (Box)", "url": "https://usf.box.com/s/abc123"}
      ],
      "tags": ["#Morsanki::Year_1::C1_Canc_Bio_MSK::MSK::Test_3::lectures_co28::L03_Skeletal_Muscle_Physio"]
    },
    {
      "label": "Anatomy Review Session",
      "code": "C1T2L26",
      "instructor": "Bharadwaj",
      "cards": 57,
      "optional_unlock": true,
      "tags": ["#Morsanki::Year_1::C1_Canc_Bio_MSK::MSK::Anatomy::Review"]
    }
  ]
}
```

Emit keys in that order — `label`, `code`, `instructor`, `cards`,
`optional_unlock`, `note`, `links`, `query`, `tags` — so a rebuild diffs only
where the data changed instead of burying a few real edits under a hundred lines
of reordering.

**How a row renders**, which is what decides where text belongs: `label` is the
heading; `code`, `instructor`, "Optional unlock" and the card count share the
grey line beneath it, in that order; `note` sits under that as its own line;
`links` render last as clickable text. So an optional row with a count reads
`C1T2L23 · Noujaim · Optional unlock · 36 cards`, and a row missing `code` and
`instructor` simply starts further along — which is why the (UPDATED) rows look
different from their neighbours.

**The order of that line cannot be changed from the data side** — it is renderer
code in `~/morsanki-addon`, which this skill does not touch. If Nick asks for a
different arrangement, say so plainly and hand him a one-paragraph spec for
Claude Code rather than inventing a data-side hack. (The optional/cards order
above is itself the result of one such request, shipped in addon v1.3.3.)

**`note`** is one short plain-text sentence. It renders as plain text under the
lecture: not clickable, not searched, so it must never contain a tag or a URL.
Omit the key entirely when there is nothing to say — never `"note": ""`. Two
kinds exist, handled differently:

- **Generated**, for a lecture already unlocked earlier — see "Repeated
  lectures". `extract_unlocks.py` writes these from the post label; never
  hand-write one.
- **Maintainer-written**, in Nick's own words ("Redownload — I fixed 12 cloze
  cards", "Deck made by Anish"). **Never invent one.** If a note seems useful,
  ask him for the wording.

**`links`** is a list of downloads for a lecture — an updated deck on Box, a
shared PDF. `url` must start with `https://` (see above). `label` is optional and
falls back to the hostname, so a bare URL still renders; it is also the natural
place for the instruction, e.g. `"Download deck here first"`, which usually reads
better than a separate note saying the same thing. Any number is fine, each on
its own line. Omit the key when there are none — never `"links": []`.

### Box links arrive in a fixed shape

This is the default to expect whenever Nick sends a Box link:

```
Hyperlink Text: Download here first
Link: [https://usf.box.com/s/aaa…](https://usf.box.com/s/bbb…)
```

The first line is `links[].label`, used verbatim. The second is `links[].url`.
Box writes markdown whose *visible text* is one share URL and whose *target* is
another — **the target, the parenthesized half, wins**; confirmed with Nick
2026-09-06. Do not stop to ask which of the two is real. If he sends a bare URL
with no hyperlink text, default the label to `Download here first`.

A Box link almost always means a re-exported deck, which is the (UPDATED) row
below — a new row on the same day, not an edit to the existing lecture.

**`optional_unlock`** marks an unlock as optional; its cards are excluded from
the day's total, so the total reflects what is actually expected rather than
everything available. Omit the key when false. Because the exclusion is
automatic, an optional row **can** carry a real `cards` count without inflating
the day — and since addon v1.3.3 both render together.

### `query`: a raw Anki search for the button

Added in addon v1.3.3, for the cases the tag list cannot express.

> `query` is an optional Anki search used verbatim by Open in Browse, overriding
> `tags`. Use it only when the tag list cannot express what you need — negation,
> a `deck:` term, a hand-tuned filter. Write a search you have actually run in
> the Browser; it is passed through unmodified, so a mistake shows as an invalid
> search rather than being corrected. Omit the key entirely when the tags
> suffice, which is almost always. `tags` stays required either way.

Four operational consequences of that contract:

- **`tags` stays populated, and that is the rollout story, not boilerplate.** A
  classmate on an older addon ignores `query` and falls back to the tag — a
  broader button, never a broken one. That is what makes it safe to start using
  `query` immediately instead of waiting for everyone to update. Never ship a
  `query` row with empty `tags`.
- **Verbatim has a JSON edge.** A query containing double quotes — a phrase
  search, `deck:"MCOM::AnKing Step Deck"` — needs them escaped as `\"` in the
  file. Round-trip the string through `json.loads(json.dumps(q))` and confirm it
  is unchanged before publishing.
- **Run the stored string, not the one you meant.** Read the query back out of
  the merged file, run it through `findCards`, and require the result to equal
  the row's `cards` value. That single check catches escaping damage, a stale
  edit, and a wrong count in one go.
- **Whitespace-only counts as absent** and falls back to `tags`. Do not rely on
  that as a way to disable a query — delete the key instead, per the sticky-field
  rule below.

**`query` is maintainer-authored.** It appears nowhere in the Discord posts, so
the merge must carry it forward exactly like `note` and `links`, or the next
rebuild silently reverts the button to the plain tag. It belongs in the carry
list; see step 3.

### Do not record attendance

There is no `mandatory` field, and one must never be added. The Discord source
often marks lectures "(MANDATORY — Professor)"; that information stays in the
Discord post and does not go into `schedule.json`. The addon must not be treated
as the source of truth for what attendance is required. Drop the marker when
writing the JSON — keep the instructor name, discard the MANDATORY word.

---

## The one rule everything else serves

**Tags are never authored here. They are extracted from the Discord posts.**

`build_week.py` in `~/Documents/AnkiUnlocks` is the only thing that touches the
live Anki collection and resolves each tag by lookup against the real deck. The
post it writes is correct by construction. If you re-derive, retype, correct, or
"clean up" a tag on the way into JSON, you have created a second source that can
disagree with the first — and the failure is invisible: the addon renders a
normal-looking **Open in Browse** button that finds zero cards.

So: run the extractor, never transcribe. Do not read tags off the screen and type
them into a JSON blob, not even one, not even a short one.

**Changing which tags a lecture carries is a change to the Discord post, not to
the JSON.** Edit the fenced query block in that day's `.md`, then rebuild, and it
flows through the extractor and survives every future rebuild. A tag written
straight into the JSON is the one thing preservation does *not* cover, so it
would silently vanish next time. Tell Nick afterward that the posted Discord
message now differs from his local file — and, when the change narrows an unlock,
say how many cards it drops (check with `findNotes`, e.g. `A OR B` minus `C`).

**Never construct an Anki path of any kind — tag or deck.** The `AddedByNick`
deck tree moved from `MCOM::MCOM_In_House::…` to `MCOM_In_House::…` *between two
calls in one session*, and a hardcoded prefix returned zero notes rather than an
error, which reads exactly like "Anki is down". Match on a suffix against
`deckNames`. Related: **`getTags` times out** — ~46,000 tags exceeds the
`device_bash` limit — so read tags from `notesInfo` on a note set and counts from
`findCards`.

---

## Updating unlocks

### 1. Find the week

The posts live in `~/Documents/AnkiUnlocks/weeks/<YYYY-MM-DD>/`, one folder per
week named for its Monday, containing `1_Monday_*.md` … `5_Friday_*.md` and a
`0_Overview_*.md`. Use the folder for the week being posted. If it does not
exist, the posts have not been built yet — stop and say so; do not build them
from the tracker yourself.

An edit to an *already posted* week is legitimate (a mid-week correction, a
republished deck) — just say afterward that the live Discord message now differs
from the local file and needs repasting. Keep a running list across a session;
by the end there are usually several.

### 2. Extract

```bash
cd ~/Documents/AnkiUnlocks
python3 scripts/extract_unlocks.py weeks/<YYYY-MM-DD> > /tmp/week.json
```

The year comes from the folder name, not from the post headers (the headers
carry no year, and a wrong one displaces every unlock by 365 days while looking
entirely plausible). `tag_map.xlsx` is found automatically beside the week
folders. The script reads `.xlsx` with the standard library — **do not
reintroduce an openpyxl dependency**; it is not installed for the `python3` that
runs these scripts on the Mac, and that only surfaces at run time.

If `scripts/extract_unlocks.py` is missing, **stop and tell Nick**. Do not fall
back to reading the markdown by hand, and do not use
`morsanki-addon/tools/convert_unlocks.py` — that parses a different, older
markdown shape and silently returns nothing on real posts.

The extractor raises rather than guesses. Four errors matter:

- **"tags present in the post but not captured by any lecture"** — the post
  format changed. Fix the extractor's `LEC` heading regex; do not work around it
  by hand. This guard already caught one real drift (the Aug 24 posts put the
  instructor outside the bold, later weeks put it inside), and it is why a format
  change costs a fixed regex instead of a silently half-empty week.
- **"deck-filtered query in an unexpected shape"** — see the co28/co30 section.
- **"tag_map tag is not published anywhere in this week's posts"** — see the
  repeated-lectures section.
- **"note points at … which is not in this week"** / **"none of its tags appear
  that day"** — a generated date failed its cross-check; see below.

Days with no lectures (holidays, Test days, White Coat) produce no entry at all.
That is correct: the addon shows "No unlocks scheduled" for a date it has no
entry for.

**The backtick label can carry both a count and the optional marker** —
`optional redownload · 157 cards`. `CARDS` is an unanchored search, so it picks
the number out of any label; `optional` anywhere in the same label sets the flag.
This was an anchored `^(\d+) cards?$` until 2026-09-06, which is why the early
(UPDATED) rows shipped without counts — a limitation of the parser that got
described to Nick as if it were a property of the addon. It was not.

**Prefer a post shape that needs no extractor change.** Before reaching for a
regex edit, check whether the row can be written so the existing parser already
handles it — the (UPDATED) row was built twice, and the version needing no code
change is the one that shipped. Any extractor edit has to clear check 17 below,
and the file is not under version control, so copy it aside before patching and
diff against that copy afterward to prove a revert actually reverted.

**Anchor a post insertion on the full lecture heading plus its fence**, and
assert the anchor matches exactly once. Anchoring on a tag string alone silently
matches the whole-day paste block at the top of the file, which drops the new row
at the top of the day instead of under its lecture. That happened, and was caught
only by reading the extract output row by row afterward.

Match the file's existing tag style: the Aug 24 week writes bare `tag:…`, later
weeks write `"tag:…"`. Mixing them in one file weakens the format-drift guard,
which stops checking bare tags once it finds a quoted one.

### 3. Merge — and preserve what the posts cannot know

Read the current `schedule.json` (via the API, per above), merge in memory, write
the whole file back. **Never write a fresh file** — that would wipe every existing
day and the entire `exams` array. A date present in the new extract replaces that
date wholesale; dates not in the extract are left exactly as they are; `exams` is
carried through untouched. Sort `unlocks` by date on the way out.

**`note`, `links` and `query` are maintainer-authored and appear nowhere in the
Discord posts.** A rebuild regenerates a day from the posts alone, so a naive
wholesale replace silently erases every one of them — no error, the rows just
quietly lose their "Redownload — I fixed 12 cloze cards", their Box link, and
their hand-tuned button.

So when replacing a date, carry **`note`, `links`, `optional_unlock` and
`query`** forward from the existing entry for the same lecture — matched on
`code` when it has one, otherwise on `label` — unless the new extract supplies
them itself. Report how many were carried, so it is visible rather than assumed.
If a lecture that had any of them disappears from the rebuilt day, **stop and ask
Nick** before publishing; that content is about to be lost.

One exception: when you are deliberately reshaping a row's own `label`/`code` in
the same run, its old shape shows up in that "about to be lost" report, because
the matcher no longer recognises it as the same row. That is expected — set the
carried fields on the replacement row explicitly and say so, rather than treating
it as a stop.

#### Carried fields are sticky in both directions

The carry rule is "if the rebuild says nothing about this field, keep what is
there". Silence therefore means *keep*, which makes silence useless for removal:

- **Adding** a note, link or query — write it once; every later rebuild keeps it.
- **Changing** one — write the new value once; it overwrites.
- **Removing** one — **delete the key explicitly**. Simply leaving it out of the
  file you build does not remove it; the merge sees nothing new and puts the old
  value straight back.

Deletion only has to happen once. After the published file no longer has the
field, there is nothing left to carry, so it stays gone across sessions with no
memory of any conversation required. This came up for real on 2026-09-06: asked
to drop the mock practical's note in favour of link text, omitting it would have
restored it.

---

## Repeated lectures: shared tags and generated notes

When a lecture's cards were already unlocked earlier, `build_week.py` omits its
tag block and labels it `use Monday's unlock`, `same tag as above`, `included
above`, or `unlocked Mon Aug 31 — already open`. Left alone, the addon renders
that as **"Tag missing"** — wrong, and indistinguishable from a lecture that
genuinely has no tag.

`extract_unlocks.py` handles both halves:

- **The tag** comes from `tag_map.xlsx`, which `resolve_tags.py` builds by lookup
  against the live deck, taking only `status == ok` rows. Because that is a
  second source of tags, it is fenced: **each carried tag must appear verbatim
  somewhere in that same week's posts**, or the extractor stops. That keeps it a
  carry-forward of an already-published tag rather than a fresh derivation. No
  `cards` is set — a repeat adds nothing to its day's total.
- **The note** is generated from the post label and **always names the real
  date**, because "Already unlocked Monday" is ambiguous to anyone reading a week
  later. Format: `Already unlocked Mon, Aug 31`.

| Post label | Note |
|---|---|
| `use Monday's unlock` | Already unlocked Mon, Aug 31 |
| `unlocked Mon Aug 31 — already open` | Already unlocked Mon, Aug 31 |
| `same tag as above` | Same cards as the lecture above |
| `included above` | Included in an earlier unlock this week |
| `use an earlier unlock` | Already unlocked earlier this week |

A weekday name carries no year or week, so the extractor resolves it against the
Monday of the week the lecture falls in, then **checks its own arithmetic**: the
resolved date must be strictly earlier than the lecture, must exist in that
week's posts, and at least one of the lecture's tags must actually appear on that
day. Any failure stops the run rather than publishing a confidently wrong date. A
label that states the date outright is trusted as written and skips those checks,
since it may legitimately point at an earlier week's folder.

That dated form is also the right way to **fill in a `tag TBD` lecture whose
cards opened earlier**. Give the post the real tags in a fence and the label
`unlocked Wed Aug 26 — already open`; the extractor writes the dated note and
deliberately withholds a count, so the day's total does not absorb cards the
class already has. Done for C1T2L09 on 2026-09-06.

A generated note **must never overwrite one Nick wrote**. The merge compares: if
the published note is not one of the generated forms, it wins and the conflict is
reported rather than silently resolved. A generated note regenerating to the same
text is not a conflict.

If a tag fails the verbatim check, **stop and ask Nick**. Do not carry it anyway.

A carried tag also legitimately fails a *single-day* grep — it lives in the
Monday post, not the day that reuses it. Search the whole week folder before
calling it a miss.

A lecture whose post says `tag missing` or `tag TBD` genuinely has none, and
`tag_map` shows it as `missing`. Write `"tags": []` and leave it alone.

---

## The co28/co30 pairs

Three lectures were re-tagged under `lectures_co30` because the inherited co28
tags did not match the class of 2030 schedule: **Concepts of Cell Signaling**,
**Shoulder**, and **Radiology Upper Limb** (`CO30_PAIRS` in
`AnkiUnlocks/scripts/config.py`). For these, `build_week.py` emits a *query*, not
a tag:

```
("tag:…co28::L06_Concepts_of_Cell_Signaling" deck:"MCOM::AnKing Step Deck" OR "tag:…co30::L27_…")
```

**The addon keeps the co28 tag alone.** The co30 half is dropped, and so is the
deck filter — `tags` is a plain list the addon ORs, and it cannot express
`deck:`. The reason is the audience: the co30 tags exist in Nick's collection,
not in most classmates'. A co30 tag renders a button that finds nothing for
nearly everyone; the co28 tag exists in every collection.

The cost, stated plainly: in Nick's own collection that button opens the whole
co28 tag, including the in-house cards his re-tags were meant to replace. The
Discord post stays the precise unlock; the addon button is the approximate one.
That trade is deliberate.

`extract_unlocks.py` applies this mechanically — never hand-edit a pair. It also
**omits `cards`** for a paired lecture: the post's count measures the
deck-filtered query while the stored tag opens a different set, and a different
one again in each classmate's collection. No number beats a wrong number.

If the extractor raises "deck-filtered query in an unexpected shape", the query
is not the two-tag form it knows. Stop and ask Nick; do not pick a tag yourself.

**`query` could now close this gap** — the exact deck-filtered search is
expressible verbatim, which would make the button and the post identical for the
first time. It is not applied, because it changes three published lectures and
the existing trade was a deliberate decision. Offer it; do not do it unasked.

---

## Republished decks: the (UPDATED) row

When Nick re-tags a lecture and exports it to Box for the class to import, the
update is a **second row on the same day**, never an edit to the original
lecture. The original stays exactly as posted, so classmates who already unlocked
it see the update as an addition.

```json
{
  "label": "C1T2L19 (UPDATED)",
  "code": "",
  "instructor": "",
  "cards": 157,
  "optional_unlock": true,
  "links": [{"label": "Download here first", "url": "https://usf.box.com/s/…"}],
  "tags": ["#Morsanki::…::Test_2::lectures_co30::L19_MSK_Embryology"]
}
```

Each choice is load-bearing:

- **The title goes in `label`, not `code`.** `label` is the row heading and
  `code` the grey line beneath it, so `"code": "C1T2L13 (UPDATED)"` buries the
  title under the subtext. That was the first publish and it had to be redone —
  check what renders, not just what validates.
- **`code` and `instructor` are empty**, which keeps the post heading clear of
  the `CODE` regex and means **no extractor change is needed**. The cost is
  cosmetic and Nick has seen it: the grey line starts at "Optional unlock"
  instead of a code, so these rows look different from their neighbours.
- **`optional_unlock: true`**, so a re-download does not inflate the day's total
  with cards the original row already counted.
- **`cards` is the count of what the button opens** — safe on an optional row,
  and since v1.3.3 it renders after the optional marker. Omit it only when the
  button and the post disagree about what they open.
- **Casing is `(UPDATED)`**, even when he types `(Updated)`.
- **Leave the day's header total alone** — a re-download adds no new material.

The post block, which produces all of the above with no code change:

```
**:white_large_square: C1T2L19 (UPDATED)** · `optional redownload · 157 cards`
:inbox_tray: Download here first (157 cards): https://usf.box.com/s/…
```
tag:#Morsanki::…::lectures_co30::L19_MSK_Embryology
```
```

`note` and `links` are attached at merge time; the download line is plain text
the extractor ignores.

**Which tag the button carries depends on what the export contains**, and it is
the opposite of the CO30_PAIRS rule above:

- **The whole tag exported** — Nick's own cards *plus* the AnKing cards a retag
  script carried onto the co30 tag → **the co30 tag alone**. After the import a
  classmate's co30 tag finds all of them, so one tag is exact. Pairs fall back to
  co28 only because they have no export at all.
- **His personal cards only** → co30 finds just those, and the still-relevant
  AnKing cards need the co28 tag ORed alongside — at the cost of also opening the
  in-house co28 cards the update replaces.

Ask which the export is; do not infer it. Flag the import cost once: an `.apkg`
carrying AnKing notes matches them by GUID, so Anki updates those notes' fields
in place and overwrites any edits a classmate made. Scheduling survives.

Verify every count against live Anki before writing anything. On C1T2L13 the
union of Nick's stated query turned out to equal the co30 tag alone (36 cards),
because the retag script had already carried the 17 AnKing cards onto it; the
co28 half of his query was redundant, and knowing that changed the design.

### Two variants that have come up

**Someone else's deck, no co30 tag.** C1T2L08 is Anish's; his 77 notes already
carry the class's co28 `PreLab4&5…` tag, so no new tag is needed — after import,
a classmate's existing button finds them. The row still gets its own entry, a
`note` crediting the author in Nick's words, and the same co28 tag. Look the
author up rather than assuming: the five stray cards on that tag turned out to be
**Michael Martin's**, not Anish's.

**The tag opens more than the download.** Where extra cards have accreted onto a
shared tag, the Discord paste excludes them and `query` makes the button match.
Prefer `-nid:` over phrase matching — shorter, and immune to someone editing the
card text. Get the ids from `findNotes`, and say plainly that if those ids differ
in a classmate's collection the exclusion simply does nothing, which is no worse
than the tag alone. The permanent fix is removing the tag upstream in AnkiHub,
which fixes every collection at once; suggest it, and never `removeTags` on
shared AnkiHub notes locally — the next sync overwrites it.

---

## Exams

Two fields, both required. Nothing else — unknown keys are silently ignored.

```json
{ "date": "2026-09-14", "name": "Test 2" }
```

### The name is rendered inside a sentence

The addon prints **`<N> days until <name>`**, so the name must read naturally in
that slot. This is the part most likely to be got wrong:

| Name | Renders as | |
|---|---|---|
| `Test 2` | "8 days until Test 2" | works |
| `Anatomy Practical` | "22 days until Anatomy Practical" | works |
| `Exam: MSK Block 1` | "13 days until Exam: MSK Block 1" | awkward |
| `TEST 2!!!` | "8 days until TEST 2!!!" | no |
| `Test 2 is on Monday` | "8 days until Test 2 is on Monday" | no |

Use a bare noun phrase, as students say it out loud. No leading label, no
punctuation flourishes, no trailing clause, no date inside the name.

Behaviour that is already handled, so do not design around it: exams are sorted
by date on load (write them in any order), anything dated before today is hidden
automatically (a same-day exam shows "0 days until…"), and `&` and `<` in a name
are escaped safely.

### Where the dates come from

**The study tracker is the source, not Canvas.** Canvas has every real exam as an
assignment with `due_at: null` — Test 1, Test 2, Test 3, Anatomy Practical, MOCK
Practical, and the NBME final all have no due date. The only dated Canvas
assignments are prelab/postlab quizzes, which are not exams and would clutter the
countdown. Sourcing exams from Canvas assignment due dates yields the wrong list.

Read the dates from `data/C1_Study_Tracker.xlsx` (read-only, always): the
`Lectures` sheet carries them as rows whose lecture cell is the exam name
(`TEST 1!!!`, `ANATOMY PRACTICAL`, `NBME FINAL EXAM!!!!!!!!!`).

**Cross-check against Canvas modules**, which do carry exam dates as weekday
`SubHeader` items — `mcp__remote-devices__canvas__list_modules` with
`course_id=2138158`, `search_term="Week N"`, `include_items=true`. Pass
`include_items` only alongside a specific `search_term`; Canvas applies the
search within the fetched page when items are included, so a broad query returns
nothing and returns it silently. Canvas is a second opinion, not the source — if
the two disagree, surface it rather than picking one.

Course 1, verified against both sources on 2026-09-06: Test 1 `2026-08-24`, Mock
Anatomy Practical `2026-09-09`, Test 2 `2026-09-14`, Anatomy Practical
`2026-09-28`, Test 3 `2026-09-29`, NBME Final Exam `2026-10-01`.

---

## Publishing

Cowork's folder bridge has **no GitHub credentials** — `gh` and the git identity
live on the Mac, not in the bridge VM. A clone works (the repo is public); a push
has nothing to authenticate with. Never try to push from the bridge, and never
ask Nick for a token.

The cloud container has no GitHub access at all — even an API read returns 403 —
so run every `curl` against the repo through `device_bash` on the Mac.

**The normal path is the drop folder.** A launchd agent watches
`/Users/Shared/morsanki-push/drop/` and pushes whatever lands there using Nick's
own credentials. It serves both pools from that one folder, by filename:

| Drop file | Lands in | Written by |
|---|---|---|
| `schedule.json` | `morsanki-data/schedule.json` | this skill |
| `manhattanki.json` | `manhattanki-data/schedule.json` | `manhattanki-pool` |
| `manhattanki-skill.json` | `manhattanki-data/skill/*.md` | `manhattanki-pool` |

Separate clones (`repo/`, `repo-manhattanki/`) and separate validators, so a drop
for one pool cannot reach the other's repo. **This skill writes
`schedule.json` and nothing else.**

Write the **complete merged file** — both keys, every existing day — never a
fragment:

```bash
# write, then rename, so the watcher can never see a partial file
cp new.json /Users/Shared/morsanki-push/drop/.tmp
mv /Users/Shared/morsanki-push/drop/.tmp /Users/Shared/morsanki-push/drop/schedule.json
sleep 8 && tail -20 /Users/Shared/morsanki-push/log
```

That folder is often **not one of the session's connected folders** — typically
only `AnkiUnlocks` and `ankihost` are. Request it with
`device_request_folder_access` on `/Users/Shared/morsanki-push`; it is granted
immediately and mounts at `$HOME/mnt/morsanki-push`. Suggest Nick add it
permanently so the prompt is not needed next time.

The log's timestamps are Mac-local while the bridge shell reports UTC, so a
brand-new entry can look four hours old. Match on the commit hash, not the clock,
and check that the drop folder is now empty and `failed/` still is.

Always read the log afterward and report what it said, then confirm the result
through the **GitHub API**, not the raw URL. The agent fetches fresh, refuses a
file where exams shrank or unlock days disappeared, backs up before writing, and
moves a rejected drop into `failed/` rather than retrying forever. **A file
sitting in `failed/` means the push did not happen** — read the log, fix the
cause, drop again. Never edit the clone in `repo/` directly.

### The watcher refuses a drop that quietly destroys something

Validation used to check only that a drop was well formed. It now also checks it
against the live file, because the dangerous drop is the well-formed one — a
rebuild that lost half a day, a merge that dropped the maintainer-written fields,
the other pool's file in the wrong lane. None of these announce themselves; the
addon simply renders a thinner day.

The drop is refused, and moved to `failed/`, if any of these is true:

- a lecture present in a published day is **missing** from the new one (matched
  on `code`, else `label`)
- a lecture that had tags now has **none** — replacing a tag with a different one
  is fine and always has been; emptying it is what gets caught
- a `note`, `links` or `query` that was published has been **dropped**
- an existing exam's date or name has **changed**
- any tag contains `ManhattanProject` or `Manhattanki`
- `exams` shrank, or a whole day disappeared (the two original checks)

The refusal names every offending row, so the log says what happened rather than
that something did.

**A removal you actually intend is one extra step**, not a fight with the guard.
Write the reason into `drop/schedule.override`, then drop:

```bash
echo "dropping the mock practical note, per Nick 2026-09-17" \
  > /Users/Shared/morsanki-push/drop/schedule.override
cp new.json /Users/Shared/morsanki-push/drop/.tmp
mv /Users/Shared/morsanki-push/drop/.tmp /Users/Shared/morsanki-push/drop/schedule.json
```

The override is one shot — consumed by the run it authorises — and the reason
lands in the log beside the regressions it allowed. Deliberate removals are still
real (see the sticky-field rule), so this exists to make them *visible*, not to
discourage them. Never write an override to make a refusal go away without
reading what it caught.

### Rolling back

Nothing is ever lost: every push is a commit, and the watcher writes a
timestamped copy into the repo's `backups/` before each one. To go back:

```bash
cd ~/Documents/AnkiUnlocks
python3 scripts/restore_schedule.py --list
python3 scripts/restore_schedule.py --show a1b2c3d
python3 scripts/restore_schedule.py --restore a1b2c3d --reason "bad merge"
```

`--show` says what restoring would remove and bring back before anything moves.
`--restore` fetches that version through the API, writes the override for it, and
drops it; the rollback lands as a **new commit**, so the history it undoes stays
readable. It takes `--pool manhattanki` for the other repo.

Classmates' addons then pick the rollback up on their next fetch — within about
five minutes, per the raw-URL cache above. There is nothing for anyone to
uninstall or clear.

It lives in `/Users/Shared` for the reason documented in `AnkiUnlocks/README.md`:
macOS TCC gives launchd agents no consent for `~/Documents`, so an agent pointed
at anything under there dies with `Operation not permitted` before it can even
read its own script.

**Fallbacks**, if the drop folder is not connected or the agent is not installed:
`scripts/push_unlocks.command` and `scripts/push_exams.command`, which Nick runs
by double-clicking, and `scripts/install_push_watcher.command` to set the agent
up. When writing a new `.command`, `chmod 755` it **after** any later `sed -i` —
an in-place edit drops the exec bit, and macOS then refuses the file with a
permissions error that looks like something much worse.

`~/Documents/AnkiUnlocks` is **not** a git repo — local files only, no remote. The
scripts and week folders there have no history and no backup beyond the disk, so
copy any script aside before patching it.

**`device_bash` cannot delete.** `rm` fails with "Operation not permitted". Move
scratch files into a `_to_delete/` subfolder under the same connected folder and
tell Nick they are there, rather than requesting a delete-permission prompt for
something you created yourself.

---

## Before you publish

The addon fails *silently by design* — a malformed entry is dropped with no error
shown to anyone — so nothing here announces itself. Check all of it:

1. The file parses as valid JSON.
2. Both `exams` and `unlocks` exist and are arrays.
3. Every `date` is ISO `YYYY-MM-DD` and a real date.
4. Every exam has only `date` and `name`, and the name reads correctly in
   "N days until —".
5. Every lecture has a non-empty `label` and a `tags` array (possibly empty).
6. `tags` is an array of strings — never a bare string, never null.
7. `cards`, when present, is a plain integer — never `true`/`false`.
8. No lecture has a `mandatory` field.
9. Every `links[].url` starts with `https://`.
10. `note`, if present, is a non-empty string containing no URL and no tag.
11. `optional_unlock`, if present, is a real boolean — not the string "true".
12. No `note`, `links` or `query` that existed before the rebuild has gone
    missing after it — and anything Nick asked to have *removed* is actually
    absent, not carried back by the merge.
13. No maintainer-written note has been replaced by a generated one.
14. Every dated "Already unlocked" note names a day that really did unlock those
    cards.
15. Day count only grew, unless a deletion was explicitly requested. Diff the set
    of dates before and after and say which dates changed.
16. Every tag in the new days appears verbatim in the source post. Diff them
    mechanically — `grep` the post for each tag string — do not eyeball it.
17. Re-running the extractor over the weeks already live still reproduces them
    exactly. This is the regression test, and it has caught real bugs. Write its
    output into a **freshly created** directory: a leftover file at the same
    `/tmp` path from an earlier session can be owned by another user, the
    redirect then fails, and the comparison reports a difference that is really a
    permissions error — so the check never ran (2026-09-06). Redirect stderr the
    same way in the baseline and the comparison run, or the extractor's progress
    lines land in one file and not the other and every week reads as changed.
    Expect `note`/`links`/`query` to differ: the raw extract never contains them.
18. Every `query`, read back **out of the merged file** and run through
    `findCards`, returns exactly its row's `cards` value — and that row still has
    a non-empty `tags` fallback.
19. A dated backup exists — the watcher writes one, a `.command` commits one.
    If the watcher refused the drop, read what it named before doing anything
    else; an override is a decision, not a retry.
20. The published result was confirmed through the API, not the raw URL.
21. Report the change to Nick as lecture-level differences, not a line count —
    and produce it by diffing the merged file against the live one, so it says
    what actually changed rather than what was intended.

A lecture with **no tag is normal** — roughly a quarter have none. Write
`"tags": []`. Never invent a tag to fill the gap and never drop the lecture: an
untagged lecture renders correctly with a "Tag missing" marker, while a wrong tag
renders as a working button that finds nothing.

---

## Known, and not worth rediscovering

- **The addon sums card counts; the posts union them.** `count_week()`
  deduplicates cards shared between two tags, the addon adds the `cards` values.
  Sep 2 computes to 471 in the addon against 450 in the post header. Pre-existing
  and not fixable from the data side — the addon has counts, not card ids.
- **A published count can go stale.** The Axilla union was posted as 223 and is
  now 228; `PreLab4&5…` was posted as 37 and now opens 250, because other
  people's cards landed on the tag. `schedule.json` stores tags, not numbers, so
  the row keeps working — but say so when you notice, and do not silently
  "correct" a posted number without asking.

---

## Do not

- Do not edit any file in `~/morsanki-addon`. That is the addon, it is under
  active development in Claude Code, and this skill is the data side only. That
  includes `tools/convert_unlocks.py` and the row renderer — if a layout change
  needs the renderer, write Nick a short spec and stop.
- Do not add a `mandatory` field, ever. See "Do not record attendance".
- Do not write a card total for a day. The addon computes it from the `cards`
  values of the non-optional lectures, so a hand-written total could drift out of
  sync with the real cards and would be ignored anyway.
- Do not invent a `note`. Generated repeat notes come from the extractor;
  everything else is Nick's wording, so ask for it.
- Do not add or change a tag directly in the JSON. It belongs in the Discord
  post, or the next rebuild drops it.
- Do not write a `query` you have not run, and do not publish one without its
  `tags` fallback.
- Do not verify a push against the raw URL. Use the API.
- Do not write a `ManhattanProject::` or `Manhattanki` tag into this file, ever.
  That is the other pool; the watcher refuses the drop.
- Do not write to `manhattanki-data`, or drop `manhattanki.json` /
  `manhattanki-skill.json`. That is `manhattanki-pool`'s repo and its lane.
- Do not write to `data/C1_Study_Tracker.xlsx`. Read-only, always.
- Do not create, rename, or restructure files in `morsanki-data` other than
  adding backups under `backups/`.
- Do not reformat or reorder entries you were not asked to change.
- Do not add fields beyond `label`, `code`, `instructor`, `slides`, `cards`,
  `note`, `optional_unlock`, `links`, `query`, `tags` (unlocks) and `date`,
  `name` (exams). Unknown keys are ignored by the addon, so they are silently
  useless. (`slides` — the Canvas module URL that makes a title clickable — was
  shipped after the first version of this list and spent a while missing from
  it. It is a real field, it is on most lectures, and `manhattanki-pool` seeds
  from it.)
- Do not use computer use for any of this. It was tried and it was bad: Terminal
  can only be granted click-only, and Finder's background clicks land on the
  filename label rather than selecting the file, so nothing opens.
- Do not guess a tag, a date, or a card count. Every number in this file has a
  source; if the source is unreachable, stop and say so.
