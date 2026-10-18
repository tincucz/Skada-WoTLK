# Skada — Tuan's fork

A private fork of **[bkader/Skada-WoTLK](https://github.com/bkader/Skada-WoTLK)** (the Wrath of the Lich King `3.3.5` rewrite, not the original Zarnivoop addon).

| | |
|---|---|
| **Upstream base** | `bkader/Skada-WoTLK` tag **1.8.87** (2024-03-23) |
| **This fork** | **1.8.88** — `## Author: Kader (bkader#5341), edited by Tuan` |
| **Client** | 3.3.5a (`## Interface: 30300`) |
| **Scope** | private use only — raid-segment correctness on Warmane Icecrown |

Everything below is measured against a clean checkout of upstream **1.8.87**, diffed line by line.

---

## Files that differ from upstream

Line counts are **indent-normalised** (leading whitespace stripped), because some of this
fork's edits were written with 4-space indentation while upstream uses tabs.

| File | ours-only | upstream-only | what it is |
|---|---|---|---|
| `Core/Core.lua` | 112 | 26 | the main body of work |
| `Core/Tables.lua` | 38 | 3 | ignore list + groupings + custom phases |
| `Modules/Enemies.lua` | 38 | 6 | frozen-time DPS + custom-unit lifetime |
| `Core/Functions.lua` | 8 | 0 | central frozen-time in `GetActiveTime` |
| `Modules/Nickname.lua` | 3 | 282 | forbidden-word filter **removed** |
| `Modules/Potions.lua` | 36 | 33 | reindent + older revision (see notes) |
| `Modules/Auras.lua` | 5 | 0 | aura stack-count fix |
| `Core/Options.lua` | 2 | 2 | slider max raised |
| `Skada.toc` | 2 | 2 | version / author |

No files added, no files removed — the fork is the same 172 files as upstream.

---

## 1. Segment lifecycle (the reason this fork exists)

Upstream decides a segment is over by asking the boss mod, then waiting for the whole group to
leave combat. On Warmane that leaves pulls merged and fights cut in half. Changes:

- **`FORCE_STOP_BOSSES`** — a list of bosses whose death does not reliably drop the raid out of
  combat. Their `UNIT_DIED` calls `StopSegment("<name> defeated.")`:
  ```
  37955  Blood Queen Lana'thel   Icecrown Citadel
  39863  Halion                  Ruby Sanctum (physical)
  40142  Halion                  Ruby Sanctum (twilight)
  39746  General Zarithrian      Ruby Sanctum
  39747  Saviana Ragefire        Ruby Sanctum
  39751  Baltharus the Warborn   Ruby Sanctum
  39899  Baltharus the Warborn   Ruby Sanctum (clone)
  ```

- **`combat_tick`** — a *stopped* segment is final, so it is now closed after `smartwait`
  (default 3s) **even if leftover adds keep the group in combat**, instead of waiting for
  `PLAYER_REGEN_ENABLED` that may never come.

- **Boss detection no longer depends on the boss mod.** Upstream only self-detects a boss death
  when `not Skada.bossmod`; with DBM installed that branch never ran, and DBM's Ruby Sanctum
  module does not report `EndCombat`, so nothing ever stopped the segment. Now Skada always
  self-detects; `BossDefeated()` still bails on an already-finished set, so a boss mod that
  reports first wins.

- **Boss detection after trash.** `check_boss_fight` gated re-detection behind
  `set.gotboss == nil`, but pulling trash first sets it to `false` — so walking into a boss
  through its trash left the segment named after the trash forever and nothing stopped it.
  Now the gate is `not set.gotboss`.

- **`PLAYER_REGEN_ENABLED`** rewritten: honour a stopped segment, and hold while
  `LKTransitioning()`.

## 2. Filters / ignore lists

- **`Core/Tables.lua` → `ignored_creatures`** is populated in this fork (upstream ships it
  empty): Icecrown Citadel critters (`37914` Black Rat, `37915` Blood Parasite, `36649` Blood
  Spider, `37782/37783/38228/38229/38231/38232/38369/38370` insects) and common critters
  (`1420/3444/2110` Dig Rat, `34632` Ogre Pinata).
- **`IGNORED_NAMES` + `ShouldIgnoreCreature()`** in `Core.lua` — a name-keyed fallback (gated to
  raid/party instances because it is localised), checked on the destination of every combat
  event, and again in `PLAYER_REGEN_DISABLED` so a critter on your cursor cannot open a segment.
- **Nickname forbidden-word filter — removed** (this fork). `Modules/Nickname.lua` shipped a
  275-word blacklist plus a `strfind(strlower(name), word)` loop that rejected any nickname
  containing one of them. Both the table and the loop are gone; the length / character /
  repeated-letter checks remain.
- **Boss-add groupings** — `grouped_units` gained name-keyed fallbacks for the Lady Deathwhisper
  encounter (`Cult Adherent`, `Empowered Adherent`, `Reanimated Adherent`, `Cult Fanatic`,
  `Deformed Fanatic`, `Reanimated Fanatic`, `Darnavan` → `Adds`), because the numeric IDs were
  not matching on this realm.
- Related but **not part of Skada**: the `iTargetingFrames` NPC blacklist carries the same
  encounter filters for nameplates, e.g. `38068` Mana Void (Blood Prince Council), `40683`
  Living Ember, `40681` Living Inferno, plus the ICC trash set.

## 3. Lich King

- **`LKTransitioning()`** — true while the Lich King is still above his final-phase threshold
  (11% HP), i.e. during the brief out-of-combat transitions of that fight. While it is true the
  segment is **not** closed by `combat_tick` or `PLAYER_REGEN_ENABLED`, so his phase changes
  cannot split the encounter. A stopped segment is never held.

## 4. Custom units & frozen DPS

- **`Core/Tables.lua`** — custom unit/phase entries:
  - `[36853]` **Sindragosa** — `{ "%s - Ground Phase (0-52s)", start = 1, stop_time = 52 }`,
    then `{ L["%s - Phase 2"], start = 0.35 }`.
  - `[36980]` **Ice Tomb** — `{ name = "Ice Tomb (< 15%)", start = 0.15 }`.
- **`Modules/Enemies.lua`** — consumes `stop_time`: once a custom unit has been up for
  `unit.stop_time`, the elapsed time is snapshotted onto the actor (`actor.fixed_time`) and
  `unit.done` is set, so its DPS/DTPS **stops fluctuating** instead of drifting as the fight
  continues. Used for Sindragosa's ground phase.
- **`Modules/Enemies.lua`** — custom units now record their own start time (`t.starttime`), and a
  unit that finishes by itself (`unit.curval <= unit.minval`, e.g. an Ice Tomb being destroyed)
  stamps `actor.fixed_time` with its **real elapsed lifetime**. Previously only the `stop_time`
  path froze anything, so self-finishing units kept dividing by the live segment time.
- **`Core/Functions.lua`** — the freeze is applied centrally in `Skada:GetActiveTime`, which every
  DPS/DTPS getter funnels through. Before this, only two `mode:Update` spots in `Enemies.lua`
  honoured `actor.fixed_time`; drill-downs, set summaries and tooltips still divided by the
  growing segment time, so a frozen unit's number kept falling forever after the freeze point.

## 5. Fixes

- **`Modules/Auras.lua`** — aura stack counts are now initialised and incremented (`n`), so
  `FormatNumber` can no longer be handed `nil`.
- **`Core/Options.lua`** — two sliders raised `max = 25` → `max = 50`.
- **`Core/Core.lua`** — `setstokeep`/`setslimit` clamped to 0–50 and
  `maxmeme = min(150, max(30, maxsets + 10))`.
- **`Core/Core.lua`** — tooltip layout reworked: spells/targets first, a spacer line after the
  Targets block, then Activity, then Comparison.

---

## Known divergence (not intentional)

`Modules/Potions.lua` in this fork is **reindented** (tabs → 4 spaces) and appears to be based on
a slightly older revision than upstream 1.8.87 — it is missing a handful of upstream
`potion_ids` entries and the `mode_spell.nototal = true` line. Worth re-syncing from upstream
if potion tracking matters.

## Notes for whoever maintains this

- Two of the edits (`Core/Core.lua`, `Core/Tables.lua`, `Modules/Enemies.lua`) mix 4-space
  indentation into a tab-indented codebase. A `git diff --ignore-all-space` is the sane way to
  read them.
- The `combat_tick` / `PLAYER_REGEN_*` behaviour is deliberately more aggressive than upstream
  about closing segments. If a fight starts getting cut short, `smartwait` (General options,
  under Smart Stop) is the knob.
- Upstream is at 1.8.87 and has since moved on; this fork was cut from that tag, not from `main`.
