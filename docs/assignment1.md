# Single-Player MUD — Prototype v0.1
## Software Requirements & Implementation Specification

**Status:** Ready for implementation  
**Project type:** Rust command-line application  
**Audience:** Junior software developer  
**Deliverable:** Runnable, tested demonstration  
**Scope:** Three rooms, one player, one movable item

---

## 1. Project Overview

### 1.1 Objective

Develop a minimal, text-based, single-player Multi-User Dungeon (MUD)-style game engine in Rust.

The prototype shall demonstrate:

- A navigable game world.
- A player capable of interacting with that world.
- A consistent system for tracking object locations and containment.
- Separation of static game definitions from mutable game state.
- Loading and saving game state.
- Basic command processing.
- Automated verification of core game rules.

The application is intended to establish the foundations of a larger game engine. However, **only the functionality specified in this document is required**.

### 1.2 Expected User Experience

The user launches the application in a terminal.

The application loads the game world, creates a new game session, displays the starting room, and prompts for commands.

The user can navigate between rooms, inspect their surroundings, pick up an item, drop it, inspect inventory, and save or restore progress.

The game continues until the user enters the quit command or terminates the application.

### 1.3 Success Criteria

The prototype is considered complete when:

1. All required commands operate correctly.
2. The world maintains a consistent state after every action.
3. Static game definitions are loaded from external files.
4. Game progress can be saved and restored.
5. Required automated tests pass.
6. The project builds and runs using standard Cargo commands.
7. Another developer can follow the README to run and test the application.

---

## 2. Required Game World

The prototype shall contain exactly three rooms, one player, and one item.

### 2.1 Rooms

| Room ID | Name | Exits |
|---|---|---|
| `village_square` | Village Square | North → Forest Path, East → Old Shop |
| `forest_path` | Forest Path | South → Village Square |
| `old_shop` | Old Shop | West → Village Square |

Suggested room descriptions:

**Village Square**

> A weathered stone fountain stands in the center of a quiet village square. Narrow streets lead between old buildings, while a wooded path disappears to the north.

**Forest Path**

> Tall trees crowd the narrow forest path. Fallen leaves cover the ground, and the sounds of the village fade behind you.

**Old Shop**

> Dusty shelves line the walls of a small shop. A wooden counter stands beneath a narrow window.

These descriptions must be stored externally, not hardcoded into command-processing functions.

### 2.2 Player

The world shall contain one player character.

Initial state:

| Property | Value |
|---|---|
| ID | `player` |
| Name | Player |
| Initial parent | `village_square` |
| Inventory | Empty |

The player has no combat statistics or character progression in this version.

### 2.3 Item

The world shall contain one movable item.

| Property | Value |
|---|---|
| Instance ID | `rusted_key_001` |
| Definition ID | `rusted_key` |
| Display name | Rusted Key |
| Description | A small iron key, heavily tarnished with age. |
| Initial parent | `forest_path` |

The key has no special function in this prototype. Its purpose is to demonstrate identification, containment, and movement.

---

## 3. User Commands

Commands shall be entered as plain text at a terminal prompt.

Input shall be case-insensitive and ignore leading or trailing whitespace.

### 3.1 Required Commands

| Command | Behavior |
|---|---|
| `look` | Display current room, visible items, and exits |
| `north` | Attempt to move north |
| `south` | Attempt to move south |
| `east` | Attempt to move east |
| `west` | Attempt to move west |
| `take key` | Attempt to pick up the rusted key |
| `drop key` | Attempt to drop the rusted key |
| `inventory` | Display carried items |
| `save` | Save the current game state |
| `load` | Restore the saved game state |
| `help` | Display supported commands |
| `quit` | Terminate the application |

The full name `rusted key` shall also be accepted for `take` and `drop`.

Additional command aliases are optional and must not interfere with required commands.

### 3.2 Command Processing

Command processing shall be divided conceptually into:

1. **Input:** Read text from the user.
2. **Parsing:** Interpret the text as a supported command.
3. **Validation:** Determine whether the requested action is permitted.
4. **Execution:** Apply the relevant state changes.
5. **Response:** Display the result to the user.

The parser must not directly modify world state.

### 3.3 Movement

When the user enters a movement command:

- Determine the player's current room.
- Locate the corresponding exit.
- If the exit exists, move the player to its destination.
- Display the destination room description.
- If the exit does not exist, report that movement is impossible.

An invalid movement attempt must not change the player's location.

### 3.4 Item Interaction

`take key` shall succeed only when the key is an immediate child of the player's current room.

On success, the key's parent changes to the player.

`drop key` shall succeed only when the key is currently carried by the player.

On success, the key's parent changes to the player's current room.

An unsuccessful action must not alter the world.

### 3.5 Inventory

Inventory shall display the items directly contained by the player.

For this prototype, recursive inventory traversal is not required because nested containers are out of scope.

---

## 4. World Data Model

### 4.1 Authoritative World Registry

The application shall maintain a central registry of world entities, keyed by unique identifiers.

The registry shall contain the three rooms, the player, and the key.

Each entity shall have:

- A unique, stable entity ID.
- An entity type.
- A parent reference, or no parent where appropriate.

Rooms are root entities with no parent.

The player and the key have exactly one parent.

Entity IDs must remain stable across save/load operations.

The internal Rust representation of these IDs is left to the developer.

### 4.2 Containment Model

Containment shall be expressed through a single authoritative `ParentId` relationship.

Examples:

**Initial state**

```text
Village Square
└── Player

Forest Path
└── Rusted Key

Old Shop
```

**After entering the forest and taking the key**

```text
Village Square

Forest Path
└── Player
    └── Rusted Key

Old Shop
```

Taking the key changes only its authoritative parent relationship, from `forest_path` to `player`.

### 4.3 Containment Invariants

The following rules shall always hold:

1. Every entity has a unique ID.
2. Every non-root entity has exactly one valid parent.
3. Every parent reference resolves to an existing entity.
4. Containment relationships never form cycles.
5. Rooms cannot be moved or contained by other entities in this prototype.
6. The player may be contained only by a room.
7. The key may be contained by a room or the player.
8. Failed operations must leave world state unchanged.

These rules are mandatory.

### 4.4 Child Index

The application shall provide a way to enumerate the immediate children of an entity.

A `ChildrenByParent` index is the preferred implementation.

If such an index is maintained, it shall be treated as **derived data**, not as a second independent source of truth.

Any successful structural movement must keep this index consistent with authoritative parent relationships.

### 4.5 Structural Movement

All changes to containment shall pass through a centralized world operation, conceptually:

```text
MoveEntity(entityId, destinationParentId)
```

The operation must verify that:

- The entity exists.
- The destination exists.
- The destination is a permitted container type.
- The movement does not create a containment cycle.
- The requested change can complete without corrupting the world.

The operation shall either complete successfully or preserve the previous state.

Game-command restrictions, such as whether the player can physically reach an item, are separate from the structural containment rules.

Direct, uncontrolled mutation of parent relationships elsewhere in the application is prohibited.

---

## 5. Static Game Data

### 5.1 General Requirement

Game descriptions and initial conditions shall be stored in external files.

The exact parsing library is left to the developer.

TOML is the preferred format for this prototype.

### 5.2 Suggested Files

```text
data/
├── rooms.toml
├── items.toml
└── initial_world.toml
```

**rooms.toml**

Contains room identifiers, display names, descriptions, and exit definitions.

**items.toml**

Contains item definitions, including display names and descriptions.

**initial_world.toml**

Contains the entities needed for a new game, including the initial positions of the player and key.

### 5.3 Data Loading

On startup, the application shall:

1. Read the static definition files.
2. Deserialize their contents into appropriate Rust data structures.
3. Validate identifiers and references.
4. Construct the initial world registry.
5. Build any derived indexes.
6. Verify world invariants.
7. Begin accepting commands.

Invalid or missing required data must produce a useful error rather than an unexplained panic.

### 5.4 Runtime Behavior

Definitions shall be loaded once at startup and retained in memory.

On-demand file loading, cache eviction, hot reloading, and dynamic content updates are not required.

Room descriptions and item definitions remain unchanged during a game session.

---

## 6. Mutable World State and Persistence

### 6.1 State Requirements

The application must track the current authoritative state of the world independently of static definitions.

For v0.1, the required mutable state consists of the parent relationships of the player and key.

Room connections, room descriptions, and item descriptions are static and shall not be duplicated in save files.

### 6.2 Save Format

The application shall use JSON for saved game state.

An illustrative save file:

```json
{
  "save_version": 1,
  "entities": {
    "player": {
      "parent_id": "forest_path"
    },
    "rusted_key_001": {
      "parent_id": "player"
    }
  }
}
```

The developer may choose a different internal serialization structure, provided that:

- It represents the required authoritative state.
- Entity identities remain stable.
- It can be loaded back into the application.
- The format includes a version number.
- It does not persist redundant derived indexes.

### 6.3 Save Location

For development, use:

```text
saves/slot1.json
```

The application shall create the save directory when needed.

Save files shall be excluded from Git version control.

A production-quality user data directory is not required for this prototype.

### 6.4 Saving

When the user enters `save`:

1. Validate the current world.
2. Serialize its authoritative mutable state.
3. Write the data to disk.
4. Report success or failure.

An existing valid save should not be left corrupted or partially overwritten if writing fails. A temporary-file-and-rename approach is acceptable.

### 6.5 Loading

When the user enters `load`:

1. Read the save file.
2. Validate the save format and version.
3. Reconstruct candidate world state using the static definitions.
4. Rebuild derived indexes.
5. Validate all world invariants.
6. Replace the current world only if validation succeeds.
7. Report the result.

A missing, malformed, or invalid save must not destroy the current running game state.

### 6.6 Save Compatibility

Only save version 1 must be supported.

Migration between future save formats is out of scope.

---

## 7. Software Architecture

### 7.1 Required Separation of Responsibilities

The application shall separate the following concerns:

| Component | Responsibility |
|---|---|
| Terminal interface | Input and output |
| Command parser | Convert text into commands |
| Game controller | Apply gameplay rules |
| World state | Own and maintain entity relationships |
| Definitions | Provide static game content |
| Persistence | Save and restore authoritative state |

The specific modules, files, and type names are implementation decisions.

### 7.2 Suggested Rust Structure

```text
mud/
├── Cargo.toml
├── Cargo.lock
├── README.md
├── .gitignore
├── data/
│   ├── rooms.toml
│   ├── items.toml
│   └── initial_world.toml
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── command.rs
│   ├── game.rs
│   ├── entity.rs
│   ├── world.rs
│   └── persistence.rs
└── tests/
    └── game_integration.rs
```

This is a suggested structure, not a mandatory file-by-file architecture.

Responsibilities should remain clear regardless of how the developer divides the code.

### 7.3 Rust Requirements

- Use stable Rust and Cargo.
- Target Rust edition 2024.
- Use idiomatic structs and enums for data modeling.
- Use `Option` and `Result` where appropriate.
- Avoid unnecessary `unsafe`.
- Avoid shared mutable global state.
- Keep gameplay rules independently testable from terminal input.
- Use established serialization libraries where appropriate.

The project should not require asynchronous execution, multithreading, a database, a web server, or an external game framework.

---

## 8. Expected Gameplay Transcript

The following illustrates the expected functional behavior. Wording may differ as long as the same information is communicated.

```text
Single-Player MUD v0.1

Village Square
A weathered stone fountain stands in the center
of a quiet village square.

Exits: north, east

> north

Forest Path
Tall trees crowd the narrow forest path.

You see:
- Rusted Key

Exits: south

> take key

You pick up the Rusted Key.

> inventory

You are carrying:
- Rusted Key

> south

Village Square
A weathered stone fountain stands in the center
of a quiet village square.

Exits: north, east

> drop key

You drop the Rusted Key.

> look

Village Square
A weathered stone fountain stands in the center
of a quiet village square.

You see:
- Rusted Key

Exits: north, east

> save

Game saved successfully.

> take key

You pick up the Rusted Key.

> load

Game loaded successfully.

> inventory

You are carrying nothing.

> look

Village Square

You see:
- Rusted Key

Exits: north, east

> quit

Goodbye.
```

---

## 9. Error Handling

The application must handle predictable errors without crashing.

| Situation | Expected behavior |
|---|---|
| Unknown command | Display an informative error |
| Empty input | Ignore and prompt again |
| Invalid exit | Explain that movement is unavailable |
| Take an absent item | Report that the item isn't present |
| Take an already carried item | Reject without changing state |
| Drop an uncarried item | Reject without changing state |
| Missing save file | Report that no save is available |
| Corrupted save file | Reject and preserve current game |
| Invalid entity reference | Reject the operation |
| Containment cycle | Reject the operation |
| Missing or malformed static data | Fail startup with a descriptive error |

Normal gameplay errors should be returned as expected results, not treated as programming exceptions.

---

## 10. Automated Test Requirements

The project shall include automated tests runnable through:

```bash
cargo test
```

### 10.1 Core Acceptance Tests

| ID | Test | Required result |
|---|---|---|
| T01 | Initialize game | Three rooms, player, and key exist |
| T02 | Inspect starting room | Correct description and exits |
| T03 | Move north | Player enters Forest Path |
| T04 | Move through invalid exit | Player location unchanged |
| T05 | Take key | Key becomes child of player |
| T06 | Inspect inventory | Key appears exactly once |
| T07 | Inspect room after pickup | Key no longer appears in room |
| T08 | Drop key | Key becomes child of current room |
| T09 | Attempt invalid pickup | State remains unchanged |
| T10 | Save and restore | Original saved relationships restored |
| T11 | Load malformed save | Active game state preserved |
| T12 | Load invalid parent reference | Save rejected |
| T13 | Validate containment | No missing parents or cycles |
| T14 | Verify child index | Index matches authoritative parent IDs |
| T15 | Parse commands | Expected command variants recognized |
| T16 | Unknown command | Rejected without modifying state |

### 10.2 Testing Principles

Tests should interact with the world and game logic directly where possible.

A test should not need to launch an interactive terminal session to verify room movement or item containment.

Tests must verify **state changes**, not merely the presence of expected output text.

For example, a successful pickup test should establish that:

- The key's parent is now the player.
- The key is no longer listed among the room's children.
- The key appears exactly once in the player's inventory.
- The world remains valid.

---

## 11. Required Deliverables

The completed assignment shall include:

1. A working Rust application.
2. The external game-definition files.
3. Working save and load functionality.
4. Automated unit and integration tests.
5. A README explaining build, execution, and testing.
6. A brief description of the chosen architecture and significant implementation decisions.
7. A Git repository containing the complete source, configuration, and test files.

The README shall include the commands:

```bash
cargo build
cargo run
cargo test
cargo fmt --check
cargo clippy -- -D warnings
```

All must succeed on the developer's supported environment, with formatting and lint issues resolved.

Build output, temporary files, and local save files must not be committed.

---

## 12. Explicitly Out of Scope

Do not implement the following in v0.1:

- Combat.
- Character statistics.
- Equipment slots or modifiers.
- NPCs or autonomous actors.
- Game ticks, timers, or event queues.
- Container items or nested inventories.
- Procedural world generation.
- Graphics or a full-screen TUI.
- Multiplayer or networking.
- Asynchronous execution.
- Database persistence.
- Quest or dialogue systems.
- Scripting engines.
- Plugin systems.
- General-purpose ECS frameworks.
- Save migration between game versions.

These are potential future enhancements and must not be introduced as prerequisites for the requested demonstration.

---

## 13. Implementation Discretion

The developer is expected to make reasonable technical decisions without requiring detailed instructions for every function.

Examples of decisions left to the developer:

- How to represent entity identifiers in Rust.
- Whether to use a single entity enum or separate typed structures.
- How to represent and validate command arguments.
- Whether to maintain child indexes incrementally or reconstruct them after changes.
- Which established serialization crates to use.
- How to organize modules and tests.
- How to represent recoverable application errors.

Decisions must preserve the requirements and invariants specified above.

Unnecessary abstractions or infrastructure should be avoided.

### Architectural priorities

In order:

1. Correctness.
2. Consistency of authoritative state.
3. Readability and maintainability.
4. Testability.
5. Extensibility.
6. Performance.

The prototype is too small to justify speculative optimization.

---

## 14. Definition of Done

The assignment is complete when a reviewer can:

- Clone the repository.
- Build and run the program.
- Move through all three rooms.
- Pick up and drop the rusted key.
- Observe correct containment behavior.
- Save progress, change the world, and reload the earlier state.
- Verify that invalid commands and failed operations preserve world consistency.
- Execute the automated tests successfully.
- Read the source and identify the boundaries between definitions, authoritative state, mechanics, persistence, and terminal I/O.

**Final acceptance condition:** The developer shall demonstrate that every meaningful gameplay state change occurs through controlled world operations, without creating conflicting copies of authoritative state.

The intended outcome is a small, correct, comprehensible foundation for future game development—not a feature-rich game.

---

*End of Prototype v0.1 Specification*
