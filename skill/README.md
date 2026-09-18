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
2. **Make sure the pushing works.** Cowork has no GitHub credentials, so it
   cannot push. On Nick's Mac a launchd agent does it, watching
   `/Users/Shared/morsanki-push/drop/`; `AnkiUnlocks/scripts/install_push_watcher.command`
   installs it. If you are running this on your own machine instead, give Cowork
   somewhere it can push from and tell the skill so — it will not guess.
3. **Nothing to grant for `morsanki-data`.** It is public, and the seeder only
   reads it.

## A normal turn

> "I made cards on cytokines and complement today, tag is
> `ManhattanProject::C1::T3::L10_Cytokines_and_Complement`"

The skill will:

1. check which pool you mean, if it is not obvious
2. run `seed_manhattanki.py`, which pulls the day's lecture list from
   `morsanki-data` and seeds the whole day
3. attach your tag to the matching lecture and count the cards in live Anki
4. validate, drop, push
5. tell you the date, how many lectures it seeded, how many have pool decks, how
   many of your cards are suspended, and the commit

**You never type out the day.** The lecture titles, codes, instructors and slide
links all come from `morsanki-data` — the same list Morsanki shows you. You
supply only what the pool made: the tag, any note, any links.

That is also why a lecture nobody has made cards for still shows up, blank. It is
the day as it actually was, and it is the row you fill in next.

## Two pools, and why the skill asks

Morsanki is the class deck: Discord posts, `#Morsanki::` tags, exam countdown,
`morsanki-data`. Manhattanki is what we made: `ManhattanProject::` tags, no
countdown, no Discord, `manhattanki-data`.

The two files look nearly identical, so a write into the wrong one validates,
pushes, and is only discovered when a Browse button opens the wrong deck. If the
skill cannot tell which pool you mean, it asks. Answer it rather than working
around it.

## Checking the add-on got it

Open Anki, click the yellow mark, and look at the day.

If it has not changed yet, hit **Refresh** in the dialog. Timing:

- the add-on refreshes in the background at launch and after every sync
- the raw GitHub URL is cached about 300 seconds
- the dialog's Refresh button tries the GitHub API first, cached about 60 seconds

So a change can take a minute to show up. That is caching, not a failure, and
nothing you click makes it faster.

## Several people, one repo

Everyone commits to `manhattanki-data`. Two rules keep that from hurting:

- **Pull before you write.** The seeder reads the live file every run, so the
  skill does this for you; if you edit the JSON by hand, you have to.
- **The skill merges, it does not replace.** Adding your deck to a day your
  roommate already filled updates your lecture and leaves theirs alone.

If two of you write the same day at the same moment, git raises a conflict on
push and the drop lands in `failed/`. Re-run the skill; the reseed picks up
whatever landed first.

## Backing out a bad write

`git revert` the commit in `manhattanki-data` and hit Refresh in the dialog.

That is the whole fix. Every write also drops a timestamped copy in `backups/`,
so if you would rather not read a git log, the previous version is sitting right
there as a file. The add-on caches the fetch and nothing else — there is no local
state to clean up and nothing to uninstall.

## What this never touches

- **Your Anki collection.** The skill writes one JSON file. It does not create,
  edit, delete, or suspend a single card. It only *reads* card counts.
- **`morsanki-data`.** Read-only. The class schedule is not the pool's to edit.
- **Discord.** Unlock posts cover Morsanki only.
- **Your Watched ticks.** Those live on your own machine, in the add-on's
  `user_files/watched.json`. Nothing uploads them and nobody else sees them.
- **Morsanki.** Different add-on, different folder, different data. Ticking a row
  in one does not touch the other.
