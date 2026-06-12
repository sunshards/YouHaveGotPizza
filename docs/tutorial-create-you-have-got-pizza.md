# Teaching Tutorial: Create, Test, and Publish You Have Got Pizza

This tutorial guides first-year computer science and computer engineering
students through the complete creation of **You Have Got Pizza**, a small PRG32
cartridge game. The exercise is an imitation game: students begin from an
existing working idea, reconstruct it step by step, and then personalize it with
their own design decisions while keeping the technical architecture sound.

The tutorial is organized as one-hour laboratories. Each laboratory has a
practical deliverable, a short conceptual target, and a visible test. The goal
is not only to finish a game, but to learn a professional habit: design a small
change, implement it, deploy it, observe the result, and debug from evidence.

## Learning Outcomes

By the end of the sequence, students should be able to:

- explain the PRG32 cartridge lifecycle: `init`, `update`, and `draw`;
- organize simple game state in C using constants, arrays, and structures;
- read joystick input and convert it into movement on a discrete level map;
- implement collision, scoring, lives, reset, and game-over logic;
- draw a complete 320x200 scene with primitive graphics operations;
- relate C control flow to the equivalent RISC-V assembly implementation;
- build portable cartridges for ESP32-C6 hardware and QEMU;
- package metadata, icon, screenshot, and architecture variants for the
  Cartridge Store;
- publish a cartridge responsibly, including license and asset provenance.

## Course Setting

Recommended preparation:

- one introductory lecture on C variables, functions, arrays, and `struct`;
- one introductory lecture on CPU state, memory, and the fetch-execute cycle;
- a working PRG32 checkout and, when available, one PRG32 ESP32-C6 board per
  small group;
- QEMU available for students who do not have immediate hardware access.

Students may work in pairs. One student acts as the driver, typing and running
the build; the other acts as the observer, reading compiler messages, checking
the expected behavior, and keeping the lab notebook. Swap roles at least once
per laboratory.

## The Code-Deploy-Debug Cycle

Every laboratory uses the same disciplined loop. This is the most important
professional habit in the exercise.

1. **State the expected behavior.**
   Write one sentence in the lab notebook before touching the code. Example:
   "When the player overlaps a dough ingredient on the same row, the score
   increases by 25 and the ingredient changes to collected."

2. **Edit one small part.**
   Change a single function or a small related group of constants. Avoid
   changing movement, rendering, and packaging in the same step.

3. **Build locally.**
   For ESP32-C6:

   ```bash
   export PRG32_REPO=$HOME/src/PRG32
   export PRG32_ARCHITECTURE=esp32c6
   scripts/build.sh
   ```

   For QEMU:

   ```bash
   export PRG32_REPO=$HOME/src/PRG32
   export PRG32_ARCHITECTURE=qemu
   scripts/build.sh
   ```

4. **Deploy to the available target.**
   Upload to hardware:

   ```bash
   python3 "$PRG32_REPO/tools/prg32_game.py" upload \
     dist/you-have-got-pizza-esp32c6.prg32 \
     --url http://192.168.4.1
   ```

   Stage for QEMU:

   ```bash
   python3 "$PRG32_REPO/tools/prg32_game.py" upload-qemu \
     dist/you-have-got-pizza-qemu.prg32 \
     --flash "$PRG32_REPO/build-qemu/flash_image.bin" \
     --partitions "$PRG32_REPO/partitions_prg32.csv"
   ```

5. **Observe and record.**
   Test only the behavior named in step 1. Record pass/fail and one visible
   fact from the screen, serial monitor, or QEMU window.

6. **Debug from the symptom.**
   If the behavior is wrong, classify the fault before editing:

   | Symptom | Likely location |
   |---|---|
   | Build fails | syntax, missing include, wrong symbol name |
   | Cartridge uploads but does not start | entry prefix or metadata mismatch |
   | Player moves incorrectly | input mask, coordinate update, bounds |
   | Object appears in wrong place | row table, draw coordinates, sprite size |
   | Collision feels unfair | hit-box width, row equality, reset timing |
   | Store rejects the bundle | manifest fields, missing architecture files, token |

The loop is deliberately short. In a first-year course, many bugs become
unmanageable only because too many changes are made before the next test.

## Laboratory 1: Understand the Game and the Cartridge Contract

Estimated duration: 1 hour.

### Goal

Students understand what the game is, what makes it original, and how PRG32
calls a cartridge.

### Activities

1. Play or watch the finished cartridge.
2. Read the first comment block in `c/game.c`.
3. Identify the three exported cartridge functions:
   `you_have_got_pizza_c_init`, `you_have_got_pizza_c_update`, and
   `you_have_got_pizza_c_draw`.
4. Draw a simple loop diagram:
   `init` once, then repeated `update` and `draw`.
5. Discuss why `update` and `draw` are separated.

### Deliverable

A one-page design note containing:

- the player's objective;
- the losing condition;
- the three cartridge functions and their responsibilities;
- one proposed personal variation that remains copyright-clean.

### Checkpoint

The student can answer: "Which function changes the score, and which function
draws it?"

## Laboratory 2: Prepare the Workspace and Build the First Cartridge

Estimated duration: 1 hour.

### Goal

Students build the unmodified cartridge and learn the local directory layout.

### Activities

1. Clone or locate both repositories:

   ```bash
   mkdir -p $HOME/src
   cd $HOME/src
   git clone https://github.com/riscv-prg32/PRG32.git
   git clone https://github.com/riscv-prg32/YouHaveGotPizza.git
   ```

2. Export ESP-IDF as required by PRG32:

   ```bash
   . $HOME/esp-idf/export.sh
   ```

3. Enter the game repository and configure the PRG32 path:

   ```bash
   cd $HOME/src/YouHaveGotPizza
   export PRG32_REPO=$HOME/src/PRG32
   ```

4. Build the portable cartridge:

   ```bash
   export PRG32_ARCHITECTURE=esp32c6
   scripts/build.sh
   ```

5. Repeat for QEMU:

   ```bash
   export PRG32_ARCHITECTURE=qemu
   scripts/build.sh
   ```

### Deliverable

Two generated files:

- `dist/you-have-got-pizza-esp32c6.prg32`;
- `dist/you-have-got-pizza-qemu.prg32`.

### Debug Notes

If `scripts/build.sh` says that portable cartridge builds are unsupported, the
PRG32 checkout is too old for this workflow. Update PRG32 to a branch or
release with portable ABI-table cartridge tooling, or use the legacy format
shown in `docs/build-and-publish.md`.

## Laboratory 3: Model the Game State

Estimated duration: 1 hour.

### Goal

Students recognize the data model behind a visible game.

### Activities

1. Inspect the constants at the top of `c/game.c`: screen size, actor size,
   number of rows, ladders, ingredients, enemies, and lives.
2. Study the `Actor` and `Ingredient` structures.
3. Locate these global variables: `player`, `enemies`, `ingredients`,
   `frame_no`, `score`, `lives`, `fed_students`, `message_timer`, and
   `game_over`.
4. Fill a table in the lab notebook:

   | Variable | Type | Meaning | Changes in |
   |---|---|---|---|
   | `score` | `uint16_t` | player score | collection, pizza completion |
   | `lives` | `uint8_t` | remaining attempts | collision, new game |
   | `frame_no` | `uint32_t` | animation clock | every update |

5. Change one harmless constant, such as `MAX_LIVES`, rebuild, and observe the
   HUD.

### Deliverable

A completed state table and one verified constant change.

### Concept Link

This laboratory connects directly to computer architecture: a game is state in
memory plus instructions that transform it. The screen is a visible projection
of that memory.

## Laboratory 4: Initialize the World

Estimated duration: 1 hour.

### Goal

Students implement reproducible starting conditions.

### Activities

1. Read `setup_ingredients`, `reset_player`, `reset_enemies`, and
   `start_new_game`.
2. Draw the four platform rows and four ladder x-positions on graph paper.
3. Modify one ingredient position in `setup_ingredients`.
4. Rebuild, deploy, and verify that only that ingredient moved.
5. Restore the original position or choose a new position that improves the
   level.

### Deliverable

A level-map sketch showing rows, ladders, player start, enemies, and
ingredients.

### Debug Notes

Coordinate bugs are common. PRG32 uses a top-left origin: increasing `x` moves
right, increasing `y` moves down. The game stores platform heights in `row_y`,
then derives actor `y` positions from the current row.

## Laboratory 5: Read Input and Move the Player

Estimated duration: 1 hour.

### Goal

Students convert joystick bits into controlled movement.

### Activities

1. Read `you_have_got_pizza_c_update`.
2. Locate `prg32_input_read` and `BTN_MOVE_MASK`.
3. Study `move_player`.
4. Identify these movement rules:
   - left and right change `player.x`;
   - bounds keep the actor inside the visible screen;
   - up and down work only when the player is aligned with a ladder;
   - row changes update `player.y`.
5. Experiment with the horizontal speed by changing `3` to `2` or `4`.

### Deliverable

A short test report:

- expected speed change;
- observed speed change;
- whether ladder alignment still feels fair.

### Debug Notes

If the player jumps rows without pressing up or down, inspect the branch that
checks `PRG32_BTN_UP` and `PRG32_BTN_DOWN`. If the player cannot climb, inspect
the distance threshold in `nearest_ladder_index`.

## Laboratory 6: Draw the Piazza and the Actors

Estimated duration: 1 hour.

### Goal

Students learn that rendering is a sequence of deliberate drawing operations.

### Activities

1. Read `draw_piazza`, `draw_ingredient`, `draw_player`, `draw_enemy`,
   `draw_hud`, and `draw_recipe_hint`.
2. Identify the draw order. Background comes first; player and enemies come
   after platforms and ingredients; HUD comes near the end.
3. Change one color constant and rebuild.
4. Add one small rectangle to the Piazza background.
5. Verify that the new drawing operation does not hide the player, enemies, or
   ingredients.

### Deliverable

A screenshot or photo of the modified scene, plus the exact function where the
change was made.

### Concept Link

Draw order is the simplest form of layering. Later graphics systems provide
sprites, tiles, depth buffers, and compositors, but the basic question is the
same: what is drawn first, and what is drawn over it?

## Laboratory 7: Collect Ingredients and Score Points

Estimated duration: 1 hour.

### Goal

Students implement game rules based on position and state.

### Activities

1. Read `collect_ingredients`.
2. Explain why an ingredient must be:
   - not already collected;
   - on the same row as the player;
   - close enough in `x`.
3. Change the points for collecting an ingredient from `25` to another value.
4. Rebuild, deploy, collect one ingredient, and verify the score.
5. Trace the all-done check that completes a pizza and resets ingredients.

### Deliverable

A rule trace for one collection event:

```text
player row = ...
ingredient row = ...
horizontal distance = ...
collected before = ...
score before = ...
score after = ...
```

### Debug Notes

If the same ingredient scores repeatedly, check whether `collected` is set. If
the pizza never completes, check the loop that searches for any ingredient still
uncollected.

## Laboratory 8: Add Enemies, Collisions, Lives, and Game Over

Estimated duration: 1 hour.

### Goal

Students connect autonomous motion, collision detection, and state reset.

### Activities

1. Read `move_enemies`.
2. Identify how direction changes at the left and right edges.
3. Study the row-changing behavior that happens every few seconds near a
   ladder.
4. Read `check_collisions`.
5. Modify the collision threshold by a small amount and test whether the game
   feels easier or harder.
6. Verify game over and restart with `START`.

### Deliverable

A short balancing note recommending one collision threshold and explaining why.

### Concept Link

Collision detection here is intentionally simple: same row plus horizontal
distance. This prepares students for later courses where collision becomes
geometry, bounding boxes, spatial partitioning, or physics.

## Laboratory 9: Personalize Original Assets

Estimated duration: 1 hour.

### Goal

Students personalize the game without violating copyright or breaking the
runtime.

### Activities

1. Read `assets/original_art.md`.
2. Inspect `assets/generate_assets.py`.
3. Modify one generated visual element, such as the splash screen, background,
   or sprite colors.
4. Regenerate assets:

   ```bash
   cd assets
   python3 generate_assets.py
   cd ..
   ```

5. If the runtime should match the new asset style, update the rectangle drawing
   functions in `c/game.c`.
6. Update the asset provenance note if needed.

### Deliverable

One personalized original asset and a provenance sentence explaining who made
it and how.

### Publication Rule

Do not copy sprites, logos, music, names, level art, or sound effects from
commercial games. A classroom imitation game imitates a design problem, not
protected assets.

## Laboratory 10: Compare C and RISC-V Assembly

Estimated duration: 1 hour.

### Goal

Students see that the same game responsibilities can be implemented at two
levels of abstraction.

### Activities

1. Open `assembly/game.S`.
2. Find the assembly entry points:
   `you_have_got_pizza_init`, `you_have_got_pizza_update`, and
   `you_have_got_pizza_draw`.
3. Compare one C function with its assembly counterpart, preferably player
   movement or drawing.
4. Identify three common low-level operations:
   - load or store game state;
   - branch on a condition;
   - call a PRG32 runtime function.
5. Discuss why the C version is easier to modify first, and why the assembly
   version is valuable for understanding the machine.

### Deliverable

A two-column comparison of one C fragment and the corresponding assembly
fragment.

### Instructor Note

Do not require first-year students to rewrite the whole game in assembly.
Instead, ask them to explain a short fragment precisely. Correct explanation is
more valuable than blind transcription.

## Laboratory 11: Metadata, Versioning, and Store Readiness

Estimated duration: 1 hour.

### Goal

Students prepare the cartridge as a publishable software artifact.

### Activities

1. Inspect `metadata/metadata.json`, `metadata/manifest.json`, and
   `metadata/colophon.json`.
2. Identify the title, version, authors, license, tags, architecture list,
   icon, screenshot, and colophon.
3. Confirm that `assets/icon.png` and `assets/screenshot.png` represent the
   current game.
4. Update the version only when the cartridge behavior or publishable content
   changes.
5. Build both architecture variants:

   ```bash
   export PRG32_REPO=$HOME/src/PRG32

   export PRG32_ARCHITECTURE=esp32c6
   scripts/build.sh

   export PRG32_ARCHITECTURE=qemu
   scripts/build.sh
   ```

6. Pack the Store bundle:

   ```bash
   scripts/pack-store-bundle.sh
   ```

### Deliverable

`dist/you-have-got-pizza-store-bundle.zip`.

### Debug Notes

If packing fails, check that both architecture-specific cartridges exist in
`dist`. The bundle script copies:

- `dist/you-have-got-pizza-esp32c6.prg32`;
- `dist/you-have-got-pizza-qemu.prg32`;
- metadata, icon, and screenshot files.

## Laboratory 12: Publish on the Cartridge Store

Estimated duration: 1 hour.

### Goal

Students publish, verify, and document the final cartridge.

### Activities

1. Confirm that the Store URL and token are available. The token may be provided
   through the environment:

   ```bash
   export PRG32_STORE_TOKEN=replace-with-the-class-token
   ```

2. Publish the bundle:

   ```bash
   python3 "$PRG32_REPO/tools/prg32_game.py" publish-bundle \
     dist/you-have-got-pizza-store-bundle.zip \
     --store-url http://192.168.1.42:5080 \
     --token "$PRG32_STORE_TOKEN"
   ```

3. Open the Cartridge Store page.
4. Verify that the listing shows:
   - title;
   - icon;
   - screenshot;
   - summary;
   - version;
   - supported architectures.
5. Install or download the cartridge from the Store if the classroom setup
   supports that workflow.

### Deliverable

A publication report containing:

- Store URL;
- cartridge title and version;
- publication time;
- one screenshot of the Store listing;
- one sentence describing what the student changed from the base game.

### Debug Notes

If publication fails with an authentication error, check the token. If the Store
accepts the upload but the listing looks wrong, inspect the metadata and asset
files, rebuild both variants, repack the bundle, and publish again.

## Final Assessment Rubric

| Criterion | Excellent | Satisfactory | Needs work |
|---|---|---|---|
| Design clarity | Objective, rules, and feedback are clear | Main objective works | Player cannot infer rules |
| Code structure | Changes are localized and readable | Code works with minor roughness | Changes break existing structure |
| Debug discipline | Uses short code-deploy-debug cycles with notes | Tests after major steps | Makes many changes without evidence |
| Technical correctness | Build, deploy, collision, scoring, and reset all work | Core game works with small issues | Game does not run reliably |
| Originality and ethics | Assets and theme are original and documented | Mostly original with minor omissions | Uses unclear or copied assets |
| Publication quality | Store bundle is complete and verified | Bundle publishes with minor metadata issues | Bundle cannot be published |

## Suggested Extensions

After the required laboratories, stronger groups may attempt one extension:

- add a second level layout after a fixed number of pizzas;
- introduce a bonus ingredient with a timer;
- convert one rectangle sprite to a bitmap-backed sprite;
- add a difficulty setting through metadata or compile-time constants;
- port one small C function into assembly and test for identical behavior.

Extensions should follow the same rule as the core tutorial: one expected
behavior, one small edit, one build, one deployment, one observation.

