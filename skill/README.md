# Using the Manhattanki pool skill in Cowork

`SKILL.md` in this folder is the skill. This file is how you and your roommates
actually use it.

## What it does, in one sentence

You tell it what you made; it writes `schedule.json` in `manhattanki-data`, and
the add-on picks it up.

## Setup, once per person

1. **Get the skill into Cowork.** Copy this folder — `SKILL.md` plus
   `reference/schema.md` — into the place Cowork loads skills from. The
   reference file has to come along: `SKILL.md` tells the skill to read it
   before its first write.
2. **Give Cowork access to `manhattanki-data`.** It needs to clone, commit, and
   push. Everyone in the pool needs write access on the GitHub repo.
3. **Check it can see `morsanki-data` too.** Read-only is enough — that is where
   the lecture list comes from. It is a public repo, so nothing to grant.

## A normal turn

> "I made 40 cards on the brachial plexus today, tag is
> `#Manhattanki::MSK::Nerve_Injuries_Upper_Limb`"

The skill will:

1. pull `manhattanki-data`
2. pull today's lecture list from `morsanki-data` and seed the whole day
3. attach your tag and count to the matching lecture
4. validate, commit, push
5. tell you the date, how many lectures it seeded, how many have pool decks,
   and the commit link

**You never type out the day.** The lecture titles, codes, instructors, and
slide links all come from `morsanki-data` — the same list Morsanki shows you.
You supply only what the pool made: the tag, the count, any note, any links.

That is also why a lecture nobody has made cards for still shows up, blank. It
is the day as it actually was, and it is the row you fill in next.

## Checking the add-on got it

Open Anki, click the yellow mark, and look at the day.

If it has not changed yet, hit **Refresh** in the dialog. Timing:

- the add-on refreshes in the background at launch and after every sync
- the raw GitHub URL is cached about 300 seconds
- the dialog's Refresh button tries the GitHub API first, cached about 60
  seconds

So a change can take a minute to show up. That is caching, not a failure.

## Several people, one repo

Everyone commits to `manhattanki-data` directly. Two rules keep that from
hurting:

- **Pull before you write.** The skill does this, but if you edit the JSON by
  hand, you have to.
- **The skill merges, it does not replace.** Adding your deck to a day your
  roommate already filled updates your lecture and leaves theirs alone. This is
  step 6 in `SKILL.md`, and it is the rule that makes a shared repo survivable.

If two of you write the same day at the same moment, git will raise a conflict
on push. Pull, let the skill re-apply your change, push again.

## Backing out a bad write

`git revert` the commit in `manhattanki-data` and hit Refresh in the dialog.

That is the whole fix. Every write also drops a timestamped copy in
`backups/`, so if you would rather not read a git log, the previous version is
sitting right there as a file. The add-on caches the fetch and nothing else — there is
no local state to clean up and nothing to uninstall.

## What this never touches

- **Your Anki collection.** The skill writes one JSON file. It does not create,
  edit, delete, or suspend a single card.
- **`morsanki-data`.** Read-only. The class schedule is not the pool's to edit.
- **Your Watched ticks.** Those live on your own machine, in the add-on's
  `user_files/watched.json`. Nothing uploads them and nobody else sees them.
- **Morsanki.** Different add-on, different folder, different data. Ticking a
  row in one does not touch the other.
