# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Table-side companion web page that extends the board game **Chronicles of Avel** (Rebel Studio,
designer Przemek Wojtkowiak) plus its **Adventurer's Toolkit** expansion. Built for playing with
a child. Read this whole file before changing anything: most of the design decisions here are
load-bearing and several were arrived at by correcting an earlier wrong version.

## Commands

```sh
# Run: the page fetches avel-cards.json, so it must be served over http
python3 -m http.server 8000     # then open http://localhost:8000/avel-nights.html

# Validate after any change
node --check <extracted script>                          # syntax-check the JS inside avel-nights.html
python3 -c "import json; json.load(open('avel-cards.json'))"   # card file is valid JSON
```

There is no build step, no lint config, no test framework — testing is jsdom-driven, see
"How to test" below.

## Why it exists

Games finish with turns to spare: the team beats the moon track with slack left over. The page
fills that slack with content that needs no new printed components, since new tiles cannot be
printed. Three additions:

1. A **random event** read aloud at the start of every round, weighted by difficulty.
2. A **tavern** that hands out quests. The tavern is the **castle tile** (there is a tavern in the
   castle courtyard), so no extra tile is needed.
3. A separate **endgame deck** for after the Black Moon rises, and a **map generator**.

## Files

```
avel-nights.html    the whole app: markup, CSS, ~190 lines of vanilla JS, inline SVG art
avel-cards.json     every event, quest, power, probability and map default
```

No build step, no dependencies, no framework, no persistence. The page fetches `avel-cards.json`
from its own directory, so it **must be served over http** — opening it as `file://` fails with a
message telling the user to run `python3 -m http.server 8000`. This is deliberate; do not add an
embedded copy of the content as a fallback, that reintroduces two sources of truth.

## avel-cards.json contract

All text fields are bilingual objects `{"en": "...", "sk": "..."}`. A plain string is still legal
and serves both languages (the page resolves `sk → en → raw string`), so untranslated entries
degrade to English instead of breaking. Structural fields (`tier`, `toolkit`, `team`, `invasion`)
appear once per card so the two languages cannot drift.

```jsonc
{
  "sky":     [{ "tier": "boon|neutral|mild|harsh", "name": {en,sk}, "text": {en,sk}, "toolkit": true? }],
  "march":   [ same shape ],
  "tavern":  [{ "name": {en,sk}, "task": {en,sk}, "reward": {en,sk},
                "team": true?, "invasion": true?, "toolkit": true? }],
  "powers":  [{ "name": {en,sk}, "text": {en,sk} }],
  "gifts":   [{en,sk}, ...],
  "wonders": [{en,sk}, ...],
  "odds":    { "gift": 22, "power": 11, "wonder": 2 },   // remainder is a plain quest
  "weights": { "sky":   { "calm": {"boon":3,"neutral":2,"mild":1,"harsh":0}, "moderate": {...}, "dark": {...} },
               "march": { ... } },                        // copies of each card per tier in the shuffled deck
  "map":     { "tiles": {"1":11,"2":12,"3":13,"4":15},
               "toolkitExtraTiles": 3,
               "craterDistance": {"calm":4,"moderate":3,"dark":2} }
}
```

Current counts: 57 sky, 42 tavern, 34 march, 15 powers, 8 gifts, 4 wonders, of which 20 cards are
`toolkit: true`.

Missing keys degrade instead of crashing: no `weights` means all tiers equally likely, no `odds`
falls back to 22/11/2, empty `gifts` makes gift rolls come out plain.

## Mechanics as implemented

- **Difficulty** (Calm / Restless / Cursed) is a weighting, not a separate deck. Decks are built by
  repeating each card `weight[tier]` times and shuffling, so no immediate repeats.
- **The march has its own weight table.** Calm during the last march is not calm: measured shares
  are 57% harsh on Calm, 74% Restless, 87% Cursed. This was a deliberate correction; do not
  re-unify the two tables.
- **Quest extras** are rolled per offer: 65% plain, 22% gift, 11% power, 2% wonder. Verified at
  65.2 / 21.8 / 11.0 / 2.0 over 200k draws. Powers are not attached to specific quests.
- **Heroes** are a roster of up to 4, each with a name, a portrait (`face`, index into the `FACES`
  array of 8 hand-drawn SVG animal faces), a coat of arms (`arms`, index into `ARMS`, 8 shields),
  a dungeon flag, their own quests (cap 2) and their own powers (one use each, `Cast and spend`
  removes it). Adding a hero goes through a face picker; tapping a portrait or shield on the hero
  card cycles to the next one. Team quests are filtered out when the roster has one hero.
- **Roster persistence.** Hero names, faces and arms are saved to localStorage
  (key `avel-nights-roster`) on every add, remove, rename and cycle, and restored on load. Quests,
  powers, jail state and the round are deliberately session-only. All storage access is wrapped in
  try/catch and degrades silently; corrupt or missing data falls back to the two default heroes.
- **Quests expire** two rounds after being taken (`until = round + 2`), no penalty.
- **Round counter** clears the current event and re-evaluates expiry.
- **Toolkit toggle** filters `toolkit: true` cards out of all three decks and adds 3 to the default
  map size.
- **Language toggle** (EN / SK, top bar) switches card text and all UI chrome; the choice persists
  in localStorage (`avel-nights-lang`). Cards carry both languages, so decks are language-neutral
  and a switch just re-renders — the drawn card stays and changes language. UI strings live in the
  `T` dictionary in the page; the long help section exists as two `<details>` blocks
  (`help-en`/`help-sk`) toggled by `hidden`. The Slovak card text was machine-written and awaits a
  native read-through.

## Design invariants — do not break these

These exist because the player is a child and because a random draw must never hand out an
unplayable position.

1. No event or omen ever **stuns** a hero. Losing all coins and an item to a card is not fun.
2. No event ever moves the **astrolabe**. That is a ~10% swing in total game length per trigger.
3. No card ever puts a monster **on the castle tile**, and no extra-movement omen may move a monster
   onto it. A monster in the castle is an immediate loss. This is why the "The Tavern Is Taken"
   invasion was deleted when the tavern moved to the castle.
4. Every card must be executable with **components already in the box**: coins, toughness and damage
   markers, walls, seals, traps, monster stacks, the equipment bag, the Beast dial, ballistae.
5. **Invasions** place a monster on a workshop tile and block that tile's action until it is cleared.
   Only "Wall Crawler" has a failure penalty, on purpose.
6. Rewards are claimed only when the hero is **back on the castle tile**. This is the counterweight
   to the tavern being central; without it quest acquisition is nearly free and the surplus-turn
   soak disappears.

## Verified game facts (checked against the rulebooks, not from memory)

Sources, re-fetch these rather than guessing:
- Base rulebook: `https://cdn.1j1ju.com/medias/e6/70/40-chronicles-of-avel-rulebook.pdf`
- Adventurer's Toolkit: `https://files.rebel.pl/files/instrukcje/Rules_Avel-Toolkit.pdf`
- Adventure maps sheet: `https://files.rebel.pl/files/instrukcje/Rules_Avel_Adventure-maps.pdf`

**The eight named tiles in the base game.** Nothing else has a name; the remaining tiles are terrain
carrying a lair symbol, a shortcut symbol, or nothing. Only action tiles get the green ribbon.

| Tile | Action |
|---|---|
| Marketplace | 3 coins to draw an item; sells items back at 2 coins; up to 2 transactions per action |
| Quarry | 3 coins for a wall fragment. Master Hruginir's dwarven masons. Only 3 walls exist |
| Stone circles | 3 coins to seal an empty lair. Theodore the Druid. Only 3 seals exist |
| Elven camp | 3/5/7 coins for a trap. One of each type, one per tile. Traps only hit the Beast |
| Alchemist workshop | 3 coins to upgrade one item. Master Vial |
| Wishing lake | Pay 1 coin, roll 2 green dice for 4 coins / an item / an upgrade. One full reroll |
| Wilderness and abandoned village | Take 2 coins, only if no monsters on the tile |
| Castle | No tile action. Healing Jewel, walls built here, stunned heroes return here. **Our tavern** |

**Adventurer's Toolkit adds three tiles:** Dwarven workshop (3/4/5 coins for a ballista, placed on
any chosen tile, one of each type, one per tile), Flea market (on reveal put 3 random items from the
bag face up on it; the action swaps one of your items for one of them), Fields and forests (take 3
coins if no monsters). Also shoes, three elixirs (teleport / upgrade / extra action), and the Three
Daughters — Envy, Greed and Wrath — who deal 1 unblockable damage before the battle and reward an
animal companion.

**Ballistae** fire at the start of every round before any hero acts: choose a monster or the Beast on
the ballista's tile or adjacent, roll one die of the ballista's colour, an attack symbol deals 1
damage. A kill by ballista forfeits the reward. This start-of-round timing is why several sky cards
hook into them.

**Other confirmed details used by the cards:** 2 actions per turn, 4 in solo; base 2 green dice per
battle; up to 3 clashes; monster dice are 2 black + 3 purple, only 5 in total, hence cards say "if
one is free"; heroic dice are 2 each of green, blue, orange, yellow; rest restores up to 2 toughness,
max 5; 27 coins in the supply; 3 walls, 3 seals, 3 traps; 25 equipment tokens in a bag, drawn by
touch in 5 seconds; weapon/shield/helmet have an upgraded side, elixirs do not; monsters are 24 small
(8 per colour) and 6 big (2 per colour); the moon track's late spaces retire seals, then traps, then
walls the round after the Beast appears; monsters only start moving after the Beast appears and always
take the shortest path to the castle; a monster hitting the walls destroys 1 marker, the Beast
destroys all of them; Beast toughness is 14 at 1–2 players, up to 20; difficulty is officially tuned
by changing Beast toughness; the crater token is public information you may look at any time.

**Real names available for flavour:** Dergar (famous wizard), Mirko (his student), Gileada/Gilead
(knight), Kurodar (god of the Black Moon), Theodore, Master Vial, Hruginir, Pancifer Darbulous the
Third. Do not invent new ones.

## What is invented and not from the game

The tavern, the innkeeper, the castle dungeon, quests, powers, gifts, wonders, invasions and the
event decks. All of it. That is fine, it is a house extension, but never present invented terms as
official rules, and never add a named location that is not in the table above.

**How a fabrication got in once:** an earlier version rendered hex tile art with tiles called
"Healing Grove" and "Crystal Mine". Neither exists in Avel. They came from an AI-generated tile image
supplied as a *style* reference and were absorbed as *content*. The art was also labelling the stone
circles as "Standing Stones". If given generated art as reference, take the style and check every
name against the rulebook.

## Map generator

Produces the shape of the land, never which tile is which, so face-down discovery survives.

- Hex grid, axial coordinates, castle at origin, pointy-top rendering.
- The castle always has exactly 3 neighbours, which with the castle is the box's 4 starting tiles,
  all marked FACE UP.
- Crater distance comes from the difficulty: 4 tiles out on Calm, 3 on Restless, 2 on Cursed. The
  adventure maps sheet says beginners leave 3 tiles between crater and castle, closer is harder.
- Algorithm: lay a random road from the castle out to the crater distance **first**, force the castle
  to 3 neighbours, then grow the remaining cells with a compact or sprawling bias that never touches
  the castle again. The road-first step matters: a naive compact blob never reaches distance 4 at 12
  tiles, and the first version failed 100% of the time on Calm.
- Validated over 900 generated maps, 150 per difficulty-and-shape pair: zero failures, crater at
  exactly the intended distance, castle always 3 neighbours, always fully connected.

## How to test

The container has no browser. Serve the folder and drive it with jsdom, which needs a `fetch`
polyfill via `beforeParse` because the installed jsdom has none:

```js
import { JSDOM } from "jsdom";
const dom = await JSDOM.fromURL("http://localhost:8200/avel-nights.html", {
  runScripts: "dangerously", resources: "usable", pretendToBeVisual: true,
  beforeParse(win){ win.fetch = async u => {
    const r = await fetch(new URL(u, "http://localhost:8200/").href);
    const t = await r.text();
    return { ok: r.ok, status: r.status, json: async () => JSON.parse(t) };
  };}
});
```

Worth re-running after any change: cards load; a sky card and a march omen draw; an offer that grants
a power lands on the chosen hero only; completing it moves the power to that hero; casting spends it;
advancing the round clears the event; switching difficulty rebuilds the deck; toolkit cards appear
only with the toolkit on; map geometry checks above. Also `node --check` the extracted script and
`json.load` the card file.

## Open items and known weaknesses

- **Partial persistence.** The roster (names, portraits, coats of arms) survives a reload via
  localStorage; the round number, quests and powers still do not. If that ever gets extended, it
  must keep degrading silently when storage is unavailable.
- **Slovak translation is in but unreviewed.** Every entry in `avel-cards.json` carries an `sk`
  field written by Claude, informal register, aimed at read-aloud-to-a-child. A native-speaker pass
  over the card file is the remaining work; terminology to check first: zásoba (supply),
  odolnosť (toughness), stret (clash), políčko (tile), počítadlo Beštie (Beast dial).
- **Map tile counts are derived, not counted.** 11/12/13/15 by player count came from reading the
  setup diagram captions, not from counting the box. There is a +/- control as a hedge. Confirm
  against the physical box and fix the defaults.
- **The coin economy is the tightest constraint.** 27 coins total, and the dungeon costs 2 coins to
  escape. Press Gang plus Tax Day back to back on Cursed can leave nobody able to pay; the house rule
  is that a prisoner may spend a whole turn doing nothing to walk out.
- **Frostbind and Mason's Word are the strongest powers** because during the last march the only
  currencies are rounds and walls. If banked in pairs they soften the endgame. Possible fixes: cap
  held powers at 2, or move them to invasion rewards only.
- **Storm Finger** lets a hero snipe across the board, bypassing travel and risk. Fine as a one-shot;
  restrict to adjacent tiles if it becomes the default answer to every monster.

## Style constraints for the output itself

Card text is read aloud to a child: short, concrete, no jargon, resolves in under 20 seconds, no
bookkeeping beyond a token. Palette is forest and bark (`--ink #141A12`, `--card #232D1E`,
`--edge #3F4E34`, parchment text, gold rewards, moss quests, brick red for darkness, ember for
trouble, teal for powers) — no violet anywhere, that was explicitly rejected. Art is hand-written
inline SVG, offline, no external fonts or images.
