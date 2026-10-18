# Skada-WoTLK-3.3.5a — Tuan's fork

A private-server fork of **[bkader/Skada-WoTLK](https://github.com/bkader/Skada-WoTLK)**, the
Wrath of the Lich King `3.3.5` rewrite of Skada.

![GitHub last commit](https://img.shields.io/github/last-commit/tincucz/Skada-WoTLK)
![Version](https://img.shields.io/badge/version-1.8.88-blue)
![Interface](https://img.shields.io/badge/interface-30300-lightgrey)

> **Base:** upstream tag `1.8.87` (2024-03-23) · **This fork:** `1.8.88`
> **Full breakdown:** **[`Skada/FORK-TUAN.md`](Skada/FORK-TUAN.md)** — every changed file, diffed
> line by line against upstream.

This is upstream Skada with a focused set of **raid-segment fixes** aimed at Warmane
(Icecrown). Nothing else was touched: no files added, none removed, same 172 files as upstream.

---

## What's different from upstream

### Segment lifecycle
Upstream ends a segment by asking the boss mod and then waiting for the *whole group* to leave
combat. On this realm that merges pulls and cuts fights in half. This fork:

- **Force-stops segments on boss death** for bosses whose death does not reliably drop the raid
  out of combat — Blood Queen Lana'thel, both Halion forms, and the Ruby Sanctum mini-bosses
  (Zarithrian, Saviana, Baltharus + clone).
- **Closes a stopped segment** after `smartwait` (3s) even if leftover adds keep the group in
  combat, instead of waiting for a `PLAYER_REGEN_ENABLED` that may never come.
- **Self-detects boss deaths even with DBM loaded.** Upstream only did this when no boss mod was
  present; DBM's Ruby Sanctum module never reports `EndCombat`, so nothing ever stopped those
  segments.
- **Detects a boss after its trash.** Walking into a boss through trash used to leave the segment
  named after the trash forever, and nothing stopped it.

### Filters
- `ignored_creatures` is populated (upstream ships it empty): Icecrown Citadel critters and
  common critters (Dig Rats, Ogre Pinata).
- Name-keyed fallback (`ShouldIgnoreCreature`) so a stray cleave kill on a critter cannot open or
  split a segment.
- **The nickname forbidden-word filter was removed.** The 275-word blacklist and its check are
  gone; the length / character / repeated-letter checks remain.
- `grouped_units` gained name-keyed fallbacks for the Lady Deathwhisper adds.

### Lich King
- Segments are held open during the Lich King's brief out-of-combat transitions (while he is
  above his final-phase threshold), so his phase changes cannot split the encounter.

### Custom units & frozen DPS
- Custom phase entries for **Sindragosa** (ground phase with a `stop_time`) and her **Ice Tomb**.
- Once a custom unit's `stop_time` elapses, elapsed time is snapshotted onto the actor so its
  DPS/DTPS **stops fluctuating** instead of drifting for the rest of the fight.

### Fixes
- Aura stack counts are initialised, so `FormatNumber` can't be handed `nil`.
- Two options sliders raised from 25 to 50; segment counts clamped to a sane range.
- Tooltip layout reworked (spells/targets → activity → comparison).

> **Known divergence:** `Modules/Potions.lua` is reindented (tabs → 4 spaces) and trails upstream
> 1.8.87 by a few entries. It is not an intentional change — see the note in `FORK-TUAN.md`.

---

## Install

1. If you used Skada before, delete its files from `Interface\AddOns` **and** all `Skada.lua` /
   `Skada.lua.bak` from your `WTF` folder.
2. Drop the `Skada` folder from this repo into `Interface\AddOns`.
3. Optional: `SkadaImprovement` and `SkadaStorage` are included here too.

## Credits & license

- Original addon: **Zarnivoop**
- WotLK rewrite (`1.8.x`): **bkader** ([Skada-WoTLK](https://github.com/bkader/Skada-WoTLK))
- This fork: **Tuan** — raid-segment fixes for a private `3.3.5a` realm

Licensed under **MIT/X** (see [`LICENSE.md`](LICENSE.md)). Upstream copyright and attribution are
retained in full; this fork claims nothing over the original work.
