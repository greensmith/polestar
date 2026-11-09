# Polestar Completionist ("Full Zone Sweep") Route & Guide Plan

## 1. What is Polestar Forge (and What "Forge Mode" Means Here)

**Polestar Forge** (`rust/crates/forge*`) is the Rust pipeline in this repository that builds Polestar's leveling guides from *WoW: Forever* game data:
1. **`forge-gamedata` & `forge-data`**: Extracts Forever's quests, NPCs, items, prerequisites, and coordinates into `data/forever/`.
2. **`forge-world`**: Builds the faction world model (`world-alliance.json`), including quest availability rules (`Rules`), prerequisite chains (`prereqs.rs`), mob/object camps (`camps.rs`), subzone clusters (`atlas.rs`), and an in-zone TSP/precedence tour solver (`tour::plan_order`).
3. **`forge-nav` & `forge-walks`**: Computes walking, flight, boat, and tram travel times between every quest point (`walks-alliance.bin`).
4. **`forge-sim`**: Simulates a character playing through quests and searches for an optimal quest order (`Order`), outputting `<Start>.json` (the route) and `<Start>.skips.json` (every quest skipped and why).
5. **`forge-writer` & `forge-guide`**: Turns a route `<Start>.json` into `.guide` text files (adding hub sweeps, `along` steps, tips, and travel transitions) and compiles them into Lua for the in-game addon.

### Why Standard Forge Routes Skip Quests
Currently, `forge-sim` optimizes strictly for **minimum model hours to level 60**:
- Each race does only its own 1–12 starting zone (`dwarf/01-dun-morogh`) and joins a single shared Alliance route at `02-westfall` (`rust/crates/forge-sim/src/shared.rs`).
- Quests that are grey (`passby::gray_level` in `rust/crates/forge-sim/src/search/passby.rs`) or have lower XP/minute than grinding are skipped or dropped (`edits.rs`).
- Other races' starting zones (`Elwynn Forest`, `Teldrassil`) are never entered in a Dwarf's route.

### How a "Completionist Forge Mode" Solves Your Goal
Because Forge already knows **every Alliance quest in WoW: Forever**, its **exact quest level**, **minimum level**, **race/class restrictions**, **prerequisite/breadcrumb links**, and **walking/travel distances**, we don't have to hand-merge `.guide` files line by line. Instead, we can add a **Completionist ("Zone Sweep") planner mode** that outputs a complete `.guide` pack (or local guide tree).

---

## 2. Completionist Progression Rules (Full Zone Sweep)

Based on your requirements (starting with **Alliance — Dwarf**):
1. **Complete all quests in the current zone at or below character level (`quest_level <= char_level`)**:
   - Never skip quests just because they turned green or grey (`ignore gray_level` XP-rate cutoffs).
   - Include relevant Dwarf & class quests (`only Dwarf`, `only <Class>`) as they unlock.
2. **Zone transition trigger (`remaining quests >= char_level + 1`)**:
   - Stay in the current zone as long as there is any available, solo-completable quest with `quest_level <= floor(char_level)` (or linked same-level follow-up in the zone).
   - When the only remaining unfinished quests in the zone are higher level (`quest_level >= floor(char_level) + 1`) or locked behind external prerequisites, move to the next zone that has open quests with `quest_level <= floor(char_level)`.
3. **Zone Selection Order (Dwarf Example)**:
   - **Phase 1 (Starter Sweep)**:
     - Complete Dun Morogh (Coldridge Valley -> Kharanos -> Gol'Bolar -> east Dun Morogh; all quests ≤ char level + Dwarf/class quests).
     - Take the Deeprun Tram (completing tram quests `6661`, `6662`) to Stormwind / Elwynn Forest -> sweep all Elwynn Forest quests (Northshire -> Goldshire -> Eastvale).
     - Travel to Teldrassil / Darkshore -> sweep all lower-level Teldrassil & early Darkshore quests.
   - **Phase 2 (Secondary Zone Sweep & Returns)**:
     - Return to Loch Modan / Westfall / Darkshore / Redridge / Ashenvale as your character level now matches or exceeds the remaining quests (`+1` quests from earlier visits are now `<= char_level`).
   - **Breadcrumb & Linked Quests**:
     - Pick up breadcrumb quests leading to future zones when visiting hubs, and complete multi-zone linked chains when visiting the target zone at the appropriate level.
4. **Solo / Group Handling**:
   - Keep Polestar's standard rule for elite/dungeon quests: include outdoor elite quests behind `when group <quest>` or `optional` so a solo player never gets blocked.

---

## 3. Architecture & Implementation Steps

### Phase 1: Environment & Data Preparation (On Machine with WoW: Forever Client)
- Build the `forge` binary (`cd rust && cargo build --release -p forge`).
- Run `forge doctor` and `forge build --until walks --starts Dwarf --out forge-out` (or `forge data build`) to generate:
  - `data/forever/` (quest, NPC, object, zone reference data)
  - `forge-out/world/world-alliance.json` (all placeable Alliance quests, prerequisites, and camps)
  - `forge-out/walks/walks-alliance.bin` (point-to-point walking & transit distances)

### Phase 2: Audit Alliance Quest Pool & Skips for Dwarf
- Inspect `forge-out/routes/Dwarf.skips.json` and `world-alliance.json` to catalog all Alliance quests playable by a Dwarf:
  - Note which quests in `human/01-elwynn-forest.guide` and `nightelf/01-teldrassil.guide` are truly race-locked in the game DB (`Rules::new(&world, "Dwarf", ...)`) vs. open to all Alliance races.
  - Identify quests excluded by `insertable.rs` only for `low-value` / `xp-rate` reasons so the completionist planner includes them.

### Phase 3: Implement Completionist Zone-Sweep Planner in `forge-sim`
- Add a completionist planner module (`rust/crates/forge-sim/src/completionist.rs`) and CLI flag (`forge sim route --mode completionist` or `forge sim completionist`):
  - **Candidate Pool**: All `world-alliance.json` quests available to the target race (`Rules` for `Dwarf`), plus class quests.
  - **Zone State Machine**:
    1. In the current zone, collect all open quests (prerequisites satisfied, `min_level <= char_level`, `quest_level <= floor(char_level)`).
    2. Order all open in-zone steps (accepts, objective tasks, turn-ins) using `forge_world::tour::plan_order` and hub-sweep grouping (`hubs.rs`) to minimize walking inside the zone.
    3. Replay steps through `Evaluator` (`eval.rs`) to track live character XP and level (`char_level`), unlocking newly eligible `<= char_level` quests in the same zone dynamically.
    4. When no quest with `quest_level <= floor(char_level)` remains in the current zone, pick the next zone with available `<= floor(char_level)` quests (prioritizing lowest minimum unanswered quest level, then travel distance / starter zone order: Dun Morogh -> Elwynn Forest -> Teldrassil -> Darkshore -> Loch Modan -> Westfall -> Redridge -> Duskwood -> Wetlands -> Ashenvale, etc.).
    5. Fallback: if *all* zones only have quests with `quest_level > floor(char_level)`, relax the threshold to `floor(char_level) + 1` (lowest available quest level first) so the route never deadlocks.
  - Output a standard Forge route JSON (`Dwarf-completionist.json`) compatible with `forge-writer`.

### Phase 4: Generate & Package the Completionist Guides
- Use `forge-writer` (or a completionist guide emitter per zone stay) to write the `.guide` files from the generated route order:
  - Emit one `.guide` file per zone stay (e.g., `01-dun-morogh.guide`, `02-elwynn-forest.guide`, `03-teldrassil.guide`, `04-darkshore.guide`, `05-loch-modan.guide`, `06-westfall.guide`, etc.) chained with `next`.
  - Package as a standalone community guide pack (`polestar new completionist-alliance`) so it loads cleanly alongside Polestar under **Community guides** without breaking git updates to the main Polestar repo.
- Validate with `forge guide lint` and `polestar playtest` across classes to ensure 0 errors and no stuck steps.
