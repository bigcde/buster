# Beat Busters — Prototype Build Spec (for Codex)

## Your task

Build a playable **browser prototype** of Beat Busters: a 1v1 real-time strategy game where the player drops units **on the beat** of a song to destroy the enemy's tower. Opponent is a simple AI. No online play, no accounts, no monetization.

The prototype must answer one question: **Is dropping cards on the beat fun, or does it distract from strategy?** Keep everything else minimal. Prefer simple, readable code over clever code.

Work through the milestones in order. After each milestone, the game must build and run with `npm run dev`, and `npm test` must pass.

---

## 1. Tech stack (fixed)

- **Vite + TypeScript (strict)**
- **Phaser 3** for rendering, input, scenes
- **Web Audio API** (via a small custom module) for music and the beat clock. Do **not** use Phaser's audio timing for judging.
- **Vitest** for unit tests
- No backend. No external art or audio files: all graphics are simple shapes, all music is synthesized in code.

Target: mobile browser (Chrome Android, Safari iOS) and desktop. **Landscape**, 16:9, scaled with `Phaser.Scale.FIT`. Touch and mouse both work.

---

## 2. Project structure

```
src/
  main.ts                 // Phaser game config
  config/
    constants.ts          // all tunable numbers live here
    units.ts              // unit definitions (data only)
    song.ts               // song / beatmap definition
    towerLayouts.ts       // block grid layouts
  audio/
    BeatClock.ts          // AudioContext-based clock: current beat, time to nearest beat
    Synth.ts              // procedural drum + bass + chord loop
  core/                   // pure logic, NO Phaser imports (unit-testable)
    judge.ts              // Perfect / Good / Miss from timing offset
    groove.ts             // energy system
    phases.ts             // song section -> match phase + modifiers
    tower.ts              // block grid, damage, support/collapse check
    combat.ts             // targeting, damage resolution
    ai.ts                 // opponent decision logic
  scenes/
    BootScene.ts
    CalibrationScene.ts   // tap test for audio latency
    MatchScene.ts         // the game
    ResultScene.ts
  ui/
    BeatRing.ts           // shrinking ring that shows the beat
    CardHand.ts           // 3 unit cards + Groove bar
    JudgePopup.ts         // "PERFECT" / "GOOD" / "MISS" text
tests/
  judge.test.ts
  groove.test.ts
  tower.test.ts
  phases.test.ts
```

Rule: everything in `core/` is pure TypeScript with no Phaser or DOM imports, so it can be tested and later reused for online multiplayer.

---

## 3. Beat clock and music

### BeatClock
- Built on one shared `AudioContext`. Song start time = `ctx.currentTime` when playback starts (+ small scheduling lead).
- `songTime()` = `ctx.currentTime - startTime - latencyOffset` (latency from calibration, stored in `localStorage`).
- `beatIndexAt(t)`, `nearestBeatOffsetMs(t)` → signed ms to the nearest beat (negative = early).
- `secondsPerBeat = 60 / bpm`.

### Synth (procedural song, no audio files)
- BPM **110**, 4/4, length **2:30** (≈ 275 beats).
- Schedule notes ahead using the standard Web Audio lookahead scheduler pattern (e.g. 100 ms lookahead, 25 ms timer).
- Instruments from oscillators + noise: kick (beats 1 and 3), snare (2 and 4), hi-hat (8ths), bass line, simple chord pad.
- Song structure must audibly change per section:
  - **Intro**: kick + hat only
  - **Verse**: full groove
  - **Build-up**: snare rolls getting denser, rising filter on the pad
  - **Drop**: everything loud, bass doubled
  - **Outro**: full groove, fades in the last 8 beats
- Audio must start only after a user tap (mobile autoplay rule). Show a "Tap to start" screen.

### song.ts
```ts
export const SONG = {
  bpm: 110,
  durationSec: 150,
  sections: [
    { name: 'intro',   start: 0,  end: 20 },
    { name: 'verse',   start: 20, end: 70 },
    { name: 'buildup', start: 70, end: 90 },
    { name: 'drop',    start: 90, end: 110 },
    { name: 'outro',   start: 110, end: 150 },
  ],
};
```

---

## 4. Rhythm judging (core/judge.ts)

When the player drops a card, judge the drop time against the nearest beat:

| Result  | Window (abs offset) | Unit power | Spawn |
|---------|---------------------|------------|-------|
| PERFECT | ≤ 80 ms             | 150%       | on the next beat |
| GOOD    | ≤ 140 ms            | 100%       | on the next beat |
| MISS    | > 140 ms            | 70%        | one beat later than normal |

Windows are deliberately wide for the first test; put them in `constants.ts`.

**Combo:** 3 PERFECTs in a row → "ON FIRE": the next card costs 1 Groove less (minimum 1). A GOOD or MISS resets the combo.

"Unit power" multiplies HP and damage of that unit instance.

---

## 5. Groove (energy) — core/groove.ts

- Max **10**. Start at **5**.
- Gain **1 Groove every 4 beats**. In `buildup` and `outro` phases: **1 every 2 beats**.
- Both sides use the same rule (the AI too).
- Card cost is paid at drop time. If not enough Groove, the card cannot be dragged.

---

## 6. Match phases — core/phases.ts

`phaseAt(songTime)` returns the current section and modifiers (`grooveBeatsPerPoint`, `dropAbilityAvailable`).

**Drop ability (MVP: one fixed ability, "Bass Drop"):** During the `drop` section, a big button appears. Each side can use it once. Effect: deals 40 damage to every block in the bottom row of the enemy tower. The AI uses it on a random beat inside the drop section.

**End of match:**
1. A side's **Core** block is destroyed → the other side wins immediately.
2. Song ends with both cores alive → the side that destroyed **more enemy blocks** wins. Tie → draw.

---

## 7. Towers — core/tower.ts

- Each tower is a grid **5 columns × 6 rows** of cells. Player tower on the left edge, enemy tower mirrored on the right edge, both standing on the ground line.
- Block types:

| Type     | HP  | Notes |
|----------|-----|-------|
| wood     | 60  | weak |
| truss    | 140 | strong; takes 2× damage from Bassist |
| speaker  | 80  | when destroyed, explodes: 30 damage to the 4 neighbours |
| core     | 300 | bottom row, centre column. Destroyed = lose |

- One layout for MVP in `towerLayouts.ts` (strings are easy to edit):
```
row 5 (top): . W W W .
row 4:       W S W S W
row 3:       T T T T T
row 2:       W S W S W
row 1:       T W W W T
row 0 (bot): T W C W T
```
`W` wood, `T` truss, `S` speaker, `C` core, `.` empty.

- **Collapse rule:** after any block is destroyed, run a support check: a block is supported if it is on row 0, or the block directly below it exists and is supported. Unsupported blocks fall: animate them dropping, then destroy them (they count as destroyed blocks for scoring). Keep this grid-based and deterministic. No physics engine.

---

## 8. Units — config/units.ts

MVP has **3 units**. Each side has all three cards in hand at all times (no deck draw yet).

| Unit      | Cost | HP  | Damage / hit | Hit every | Range (px) | Speed (px/s) | Targets | Notes |
|-----------|------|-----|--------------|-----------|------------|--------------|---------|-------|
| Drummer   | 3    | 220 | 25           | 1 beat    | 40 (melee) | 60           | units first, then blocks | tank; blocks enemy units from passing |
| Guitarist | 3    | 90  | 20           | 1 beat    | 260        | 50           | units first, then blocks | ranged; projectile is a sound-wave arc |
| Bassist   | 5    | 150 | 35           | 2 beats   | 40 (melee) | 40           | blocks only, ignores units | 2× vs truss; hit also deals 10 to blocks directly above |

Rules:
- Units **attack on beats** (the "Hit every" column), so the whole battlefield pulses with the music. This is important for the feel.
- Spawn at the owner's tower base, walk toward the enemy tower along the ground.
- A unit stops walking when a valid target is in range.
- Melee units attack the lowest existing block in the nearest enemy column.
- Ranged units can hit any block in the nearest column.
- Unit power from the judge result (150/100/70%) multiplies HP and damage.
- Drummer counters Guitarist, Guitarist counters Bassist, Bassist breaks towers. Keep numbers in `constants.ts`/`units.ts` so they can be tuned.

---

## 9. AI opponent — core/ai.ts

Simple, readable, beatable:
- Checks once per beat.
- If Groove ≥ 7, or the player has 2+ units near the AI tower → play a card.
- Card choice: if player has a Guitarist on field → Drummer. If player has Drummers on field → Guitarist. Otherwise random weighted (Bassist 40%, others 30%).
- Timing quality is random: 50% PERFECT, 35% GOOD, 15% MISS (constants, for a difficulty setting later).

---

## 10. Screen and UI (MatchScene)

Landscape layout:
- Left: player tower. Right: enemy tower. Ground line across the bottom third.
- **Bottom bar:** 3 cards (name, cost, colour) + Groove bar with 10 segments.
- **Bottom centre:** the **Beat Ring**: a circle that shrinks into a fixed target ring on every beat. This must work with sound off, so the player can always see the beat.
- **Vibration:** call `navigator.vibrate(15)` on each beat if supported (toggle in a small settings icon; default ON).
- **Top:** song progress bar with the 5 sections marked, current phase name, blocks destroyed for each side.
- **Judge popup:** PERFECT / GOOD / MISS text over the spawn point, PERFECT with a flash.
- **Drop:** drag a card onto the battlefield (anywhere on your half) and release. The judge uses the **release time**. A tap on a card + tap on field also works.
- **Beat pulse:** on each beat, scale towers and units to 1.04 and back (~80 ms). Cheap and important.
- Colours: player units blue outline, enemy units red outline. Shapes only (rectangles/circles + text labels).

---

## 11. Calibration scene

- Before the first match (and from settings): play a click track at 100 BPM for 8 beats, ask the player to tap on each click.
- Latency offset = median of (tap time − nearest beat time). Store in `localStorage`.
- "Skip" button uses offset 0.

---

## 12. Telemetry for the playtest (local only)

At match end, store in `localStorage` and show on the Result screen:
- counts of PERFECT / GOOD / MISS, longest combo
- cards played per type
- match result and duration
- whether the player pressed "Play again" (store on press)

Add a "Copy results" button that copies the JSON to clipboard, so testers can send it.

---

## 13. Milestones

1. **Beat clock + synth + Beat Ring.** Song plays with clear sections; ring is in sync; tap anywhere shows PERFECT/GOOD/MISS. Calibration scene works.
2. **Towers.** Both towers render from the layout; clicking a block damages it (debug); collapse rule works; speaker explosions work.
3. **Units + Groove + cards.** Player can drop all 3 units, they walk, fight, damage blocks on beats. Judge result changes unit power.
4. **AI + phases + win conditions + Bass Drop.** Full match from start to result screen.
5. **Polish for playtest.** Beat pulse, judge popups, vibration, telemetry, Result screen, "Play again".

## 14. Tests (Vitest) — required

- `judge`: boundaries at 0, ±80, ±81, ±140, ±141 ms; combo counting and reset; ON FIRE discount never below 1.
- `groove`: gain rate per phase, cap at 10, cost payment.
- `tower`: layout parsing; damage; collapse of an unsupported column; speaker explosion chain; core destruction flag.
- `phases`: correct section for boundary times (0, 19.99, 20, 150).

## 15. Out of scope (do not build)

Online multiplayer, accounts, shop, monetization, more units, deck building, real music files, 3D art, native app packaging.

## 16. Definition of done

- `npm run dev` → playable full match vs AI on a phone browser in landscape.
- `npm test` passes.
- All tunable numbers are in `config/`.
- A short `README.md`: how to run, how to change BPM/windows/unit stats, how to read telemetry.
