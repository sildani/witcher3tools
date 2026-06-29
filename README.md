# Witcher 3 Next-Gen Character Builder

## 1. Summary

The Witcher 3 Next-Gen Character Builder is a client-side, single-file web application built using HTML5, CSS3, and modern Vanilla JavaScript. It simulates the character progression systems from *The Witcher 3: Wild Hunt (Next-Gen Update)*, including Skill Trees, Mutagen Equip Slots, the Blood and Wine Mutations Flow System, and the dynamic Character Loadout Panel.

This document details the architectural pattern, data structures, state serialization mechanics, and the specific design choices behind the state import/export engine.

---

## 2. Architectural Overview

The application adheres to an **unidirectional data flow pattern with declarative UI synchronization** (similar in spirit to a lightweight React component, but engineered entirely in optimized raw JavaScript).

```
[ User Interaction ] ───> [ State Mutator / Events ]
                                    │
                                    ▼
                          [ Centralized State Object ]
                                    │
                                    ▼
                         [ updateTotals() Coordinator ]
                          /          │          \
                         ▼           ▼           ▼
             [ Render Grid ] [ Sync Loadout ] [ Recalc AP ]

```

Instead of modifying DOM nodes direct-inline across disparate click events, all actions alter a globally scoped application state object. An event coordinator (`updateTotals()`) then parses this state to completely redraw dynamic UI regions and adjust visual rules sequentially. This avoids race conditions and ensures that layout logic remains perfectly aligned with user data.

---

## 3. Data Structures & State Model

The builder models complex, multi-layered data arrays efficiently to support high-speed point aggregation and serialization.

### 3.1 Static Configuration Schemes

* `skillTrees`: An object mapping tree keys (`Combat`, `Signs`, `Alchemy`, `General`) to sub-arrays containing individual skill nodes, their maximum point thresholds (`max`), and minimum required branch investments (`req`).
* `mutagensList`: A reference index detailing the colors and names of all craftable mutagens.
* `mutationsData`: A directed-graph structure modeling the Blood and Wine mutation network. Each mutation node contains a `parents` array parameter used to calculate whether dependencies are resolved.

### 3.2 Dynamic Runtime State

The running instance tracks state using five reactive global collections:

```javascript
let allocatedPoints = { Combat: [...], Signs: [...], Alchemy: [...], General: [...] };
let characterSlots = { 1: null, 2: null, ... 12: null };
let centralMutationSlots = { 1: null, 2: null, 3: null, 4: null };
let mutagenSlots = { 1: null, 2: null, 3: null, 4: null };
let slottedMutation = null;

```

---

## 4. Feature Implementation & System Interconnects

### 4.1 Dependency Validation Loop

When a skill point is incremented or decremented via standard mouse click interactions, the application evaluates the tree's running total configuration.

* **Skill Trees:** Decrementing a skill triggers a speculative evaluation loop. If dropping a point reduces the total branch investment below the `req` threshold of any *other* invested skill in that tree, the application rolls back the mutation and blocks execution.
* **Mutations Tree:** Removing a researched mutation checks all structural child nodes down the lineage. If a child depends exclusively on the selected mutation for its unlock criteria, removal is safely aborted.

### 4.2 Drag-and-Drop Subsystem

The interaction layer bridges the character inventory and the equipped layout pane by binding data transfers natively via standard browser interfaces. Drag handles serializes payload identifiers into a standard JSON string representation:

```javascript
e.dataTransfer.setData('text/plain', JSON.stringify({ type: 'skill', tree: currentTree, index: index, name: skill.name }));

```

When dropping elements onto targets like the regular slots, mutagen rings, or the mutation core, target event listeners parse incoming keys, perform validation checks, strip existing attachments from other nodes to maintain unique constraints, and run synchronization hooks.

---

## 5. URL Import/Export Engine

The application includes a compact import/export codec to share builds instantly without database backing.

### 5.1 State Compression Matrix

Because raw JSON strings exceed character boundaries safe for URL query strings, data undergoes positional array compression prior to alphanumeric conversion.

Instead of writing verbose properties, fields are converted into minified variables inside a compact export layout object:

| Key | Original Data Property | Compressed Packing Strategy |
| --- | --- | --- |
| **`a`** | `allocatedPoints` | Maps into an array of simple combined numeric strings (e.g., `["33233333330000000000", "000...", ...]`). |
| **`b`** | `characterSlots` & `mutagenSlots` | Contains a continuous array mapping nested pairs. Index 0–11 stores `[treeIndex, skillIndex]` tuples; index 12–15 stores flat mutagen array offsets. |
| **`c`** | `slottedMutation` | Flat string literal storing the active central mutation identifier key. |
| **`d`** | `mutationsData` | Arrays of keys mapping only the researched mutations to omit default values. |
| **`e`** | `centralMutationSlots` | Nested tuples mapping active central ability skill pairings `[treeIndex, skillIndex]`. |

### 5.2 Base64 Obfuscation Codec

Following matrix minification, the unified object passes into standard browser stringifiers:

1. **Serialization:** The state payload object converts to a JSON text string.
2. **Binary Conversion:** The text string translates into an ASCII string using standard Base64 standard algorithms (`btoa`).
3. **URL Cleanup:** Trailing structural characters (such as padding tokens like `=`) are safely stripped via regex routines to ensure reliable text handling across modern message clients and web headers.

### 5.3 Auto-Hydration Phase

On client instantiation, a initialization script uses `URLSearchParams` to extract parameters matching the identifier `?build=`.

* If found, the application runs the token through decoding routines (`atob`), initializes variable properties, and passes elements back into global memory maps.
* The system cross-references properties during this step; if an invalid allocation or unmet skill lock condition is evaluated, it resets gracefully to avoid code breaking.

---

## 6. Design and User Experience (UX) Decisions

### 6.1 Unified Single-File Deliverable

* **Decision:** Compile all application tiers (Structure, Presentation, Logic) inside a standalone text wrapper.
* **Reasoning:** Eliminates asset loading failures and security CORS errors associated with fetching relative asset files locally. Users can run the planner without configuration dependencies directly from any folder, disk image, or server endpoint.

### 6.2 Diamond & Grid Responsive Flex Frameworks

* **Decision:** Use structural layout styles like grid positioning matrices alongside variable `transform: rotate(45deg)` shifts for mutagen boxes.
* **Reasoning:** Mirroring *The Witcher 3*'s layout exactly requires precise alignment. Applying individual container transformations preserves accurate click targeting, while flexible grid layouts ensure the interface collapses cleanly onto single columns when viewed on mobile screens.

### 6.3 State Overlay Modal & Copy Mechanism

* **Decision:** Displays a tailored dialog modal containing an unmodifiable text input box combined with an active "Copy Link" response button instead of relying solely on standard browser dialog prompts.
* **Reasoning:** Standard prompt windows are intrusive, block screen rendering, and degrade application presentation. A custom overlay allows the page to change button text dynamically to provide visual confirmation (e.g., changing from "Copy Link" to "Copied!") while keeping the user fully immersed in the interface.
