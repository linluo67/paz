# Project Review Document: Plants vs Zombies Web Game

**Review Date:** March 30, 2026  
**Project Type:** Static Web-based Tower Defense Game  
**Review Methodology:** Static Code Analysis + Limited Functional Verification  

---

## 1. Executive Summary

This project is a browser-based Plants vs Zombies-style tower defense game implemented with vanilla HTML, CSS, JavaScript, and HTML5 Canvas. The game features a 5x9 grid battlefield where players deploy plants to defend against waves of attacking zombies. The implementation demonstrates functional gameplay mechanics including plant placement, zombie spawning, projectile combat, resource management (sun collection), and win/lose conditions.

**Overall Assessment:** The codebase achieves its core gameplay objectives but exhibits significant architectural and maintainability concerns typical of prototype-grade implementations. The tight coupling between modules, extensive use of global state, and lack of testing infrastructure present risks for long-term maintenance and feature extension.

---

## 2. Technical Environment & Execution Model

### 2.1 Technology Stack

| Component | Technology | Version/Notes |
|-----------|------------|---------------|
| Runtime | Web Browser | Modern browsers with ES6+ support |
| Markup | HTML5 | Single entry point: `index.html` |
| Styling | CSS3 | Two stylesheets: `common.css`, `style.css` |
| Logic | Vanilla JavaScript | ES6 classes, no transpilation |
| Rendering | HTML5 Canvas 2D API | 1400x600 canvas element |
| Animation | CSS Animations + Canvas Frame Loop | Hybrid approach |
| Dependencies | jQuery | Listed in package.json but not actively used |

### 2.2 Execution Flow

The application follows a straightforward script-loading pattern:

1. **index.html** loads CSS resources and defines the DOM structure
2. Four JavaScript files are loaded sequentially via `<script>` tags:
   - `common.js` - Image assets and configuration constants
   - `scene.js` - Game entity classes (SunNum, Card, Plant, Zombie, Bullet, Car, Animation)
   - `game.js` - Game engine, state machine, and event handling
   - `main.js` - Global initialization and game startup

3. `window._main = new Main()` creates the global game instance
4. `window._main.start()` initializes the game loop

### 2.3 Build & Test Infrastructure

The `package.json` contains a placeholder test script that merely echoes an error message. There is no actual build pipeline, test framework, or CI configuration. This is a pure static file project that runs directly in a browser without compilation.

---

## 3. Architecture & Module Analysis

### 3.1 Module Responsibilities

#### common.js (179 lines)
**Purpose:** Centralized asset configuration and utility functions

**Key Components:**
- `imageFromPath()` - Factory function for creating Image objects
- `keyframesOptions` - CSS animation configuration object
- `allImg` - Comprehensive image asset registry organized by entity type

**Observations:**
- Clean separation of asset paths from game logic
- Uses wildcard pattern (`*`) for sprite animation sequences
- Hardcoded base path (`./images/`) limits deployment flexibility

#### scene.js (~1000 lines)
**Purpose:** Entity class definitions and combat mechanics

**Class Hierarchy:**
```
Role (base class)
├── Plant (extends Role)
└── Zombie (extends Role)

Independent Classes:
├── SunNum (UI resource display)
├── Shovel / Shovel_move (removal tool)
├── Car (row defense mechanism)
├── Card (plant selection UI)
├── Bullet (projectile)
└── Animation (sprite management)
```

**Critical Implementation Details:**
- **Plant Class:** Implements state machine for different plant behaviors (idle, attack, digest). Contains hardcoded plant-specific logic in `setPlantFn` object.
- **Zombie Class:** Manages complex animation states including split head/body animations for death sequences.
- **Animation Class:** Handles sprite sheet loading and frame indexing with configurable FPS.
- **Collision Detection:** Implemented inline within `canAttack()` methods using coordinate comparison.

#### game.js (630 lines)
**Purpose:** Game engine, rendering loop, and input handling

**Game States:**
```javascript
state_LOADING: 0    // Initial screen
state_START: 1      // Pre-game animation
state_RUNNING: 2    // Active gameplay
state_STOP: 3       // Paused
state_PLANTWON: 4   // Victory
state_ZOMBIEWON: 5  // Defeat
```

**Core Responsibilities:**
- Canvas rendering orchestration via `setTimer()` loop at configured FPS
- Mouse event handling for plant placement and shovel usage
- State-based rendering (different draw calls per game state)
- Win/lose condition detection

#### main.js (232 lines)
**Purpose:** Global configuration and game initialization

**Key Configurations:**
- Initial sun value: 9999 (effectively unlimited for testing)
- Zombie spawn limit: 50
- Spawn intervals: 12 seconds for zombies, 20 seconds for sun
- Plant costs and cooldowns defined in `cards_info`

**Global Timer Management:**
- `sunTimer` - Periodic sun generation
- `zombieTimer` - Staggered zombie activation
- Plant-specific timers (for sunflowers)

### 3.2 Architectural Patterns

**Pattern Used:** Object-Oriented with Global State

The codebase employs ES6 classes but relies heavily on global state through `window._main`. This creates implicit dependencies between modules that are not visible in function signatures.

**State Management Approach:**
- Game state is distributed across multiple objects
- `window._main` serves as a global registry for all game entities
- Individual classes access global state directly (e.g., `window._main.zombies`)

---

## 4. Code Quality Assessment

### 4.1 Strengths

1. **Clear Entity Modeling:** The class hierarchy (Role → Plant/Zombie) provides a reasonable foundation for entity behavior.

2. **Animation System:** The `Animation` class abstracts sprite sequence management effectively, supporting different frame rates per animation.

3. **State Machine Implementation:** Both Plant and Zombie classes use explicit state constants and switch-statement-based behavior selection.

4. **Functional Separation:** The four-file organization (config, entities, engine, initialization) follows logical boundaries.

5. **Visual Feedback:** The game provides clear visual states for selection, cooldowns, and damage through transparency and animation changes.

### 4.2 Critical Issues (By Severity)

#### HIGH SEVERITY

**1. Global State Pollution & Hidden Dependencies**

The codebase extensively uses `window._main` as a global namespace:

```javascript
// From scene.js - direct global access
for (let zombie of window._main.zombies) { ... }

// From game.js - accessing global configuration
_main.plants_info.x + 80 * (col - 1)
```

**Impact:** This makes unit testing impossible, prevents module reuse, and creates invisible coupling that complicates refactoring.

**2. Memory Leak Risks from Timer Management**

Multiple timer types are created without consistent cleanup:
- Card cooldown timers (`setTimeout`)
- Sun generation timers (`setInterval`)
- Animation state transitions (`setTimeout`)
- Plant sun generation (`setInterval`)

The `clearTiemr()` method (note the typo) attempts cleanup but may miss edge cases:

```javascript
// In main.js - typo in method name
clearTiemr () { ... }

// In game.js - called during state transitions
_main.clearTiemr()
```

**Impact:** Extended gameplay sessions may accumulate orphaned timers, degrading performance.

**3. Array Modification During Iteration**

Several locations modify arrays while iterating:

```javascript
// From scene.js - modifying bullets while iterating
self.bullets.forEach(function (bullet, j) {
    if (/* collision */) {
        self.bullets.splice(j, 1)  // Modifies array during iteration
    }
})
```

**Impact:** This can cause elements to be skipped or incorrect indices after removal.

**4. Hardcoded Magic Numbers**

Coordinate calculations rely on unexplained constants:

```javascript
// From game.js
row = Math.floor((y - 75) / 100) + 1
col = Math.floor((x - 175) / 80) + 1
```

The values 75, 100, 175, 80 represent grid offsets and cell dimensions but are scattered throughout the code.

**Impact:** Layout changes require finding and updating multiple locations; high risk of inconsistency.

#### MEDIUM SEVERITY

**5. Inconsistent State Management**

The game state machine is distributed:
- `game.state` tracks high-level game states
- Individual entities track their own state (Plant.state, Zombie.state)
- Animation states are managed separately

There's no single source of truth for "what is happening in the game."

**6. Tight Coupling of Concerns**

The `Plant` class contains:
- Rendering logic (`draw()`)
- Animation management (`update()`)
- Combat logic (`canAttack()`)
- Game rule enforcement (`shoot()`)
- DOM manipulation (`setSunTimer()` creates DOM elements)

This violates the Single Responsibility Principle.

**7. Error-Prone Coordinate Calculations**

Grid position calculations appear in multiple places with slight variations:
- `game.js`: `Math.floor((y - 75) / 100) + 1`
- `main.js`: `y: 92 + 100 * (row - 1)`

These calculations must remain synchronized but are not centralized.

**8. Zombie Spawning Logic Flaw**

The zombie spawn mechanism creates all zombies upfront and then activates them:

```javascript
// From main.js
setZombiesInfo () {
    for (let i = 0; i < iMax; i++) {
        self.zombies_info.position.push({...})
    }
}
```

All 50 zombies are created at game start (in `setRoles()`), then activated sequentially. This wastes memory and initialization time.

#### LOW SEVERITY

**9. Typo in Public Method**

`clearTiemr()` should be `clearTimer()` - this typo appears in both definition and call sites.

**10. Unused Dependencies**

jQuery is listed in package.json but the codebase uses vanilla JavaScript DOM methods.

**11. Inconsistent Code Style**

Mixed indentation (2-space and 4-space), inconsistent semicolon usage, and varying comment density.

---

## 5. Testing & Verification Approach

### 5.1 Testing Limitations

Due to the project's static nature and lack of test infrastructure, the following approach was taken:

1. **Static Analysis:** Code review focusing on:
   - Variable scope and lifecycle
   - Timer creation/cleanup patterns
   - Array mutation patterns
   - State transition logic

2. **Dependency Analysis:** Tracing module interactions through `window._main`

3. **Logic Verification:** Reviewing game rule implementations:
   - Collision detection algorithms
   - State machine transitions
   - Win/lose condition checks

### 5.2 Identified Logic Concerns

**Collision Detection:**
The bullet-zombie collision uses a simple distance check:

```javascript
if (Math.abs(zombie.x + bullet.w - bullet.x) < 10 && zombie.life > 0)
```

This checks horizontal overlap only and may produce false positives/negatives depending on sprite alignment.

**Win Condition:**
The plant victory condition triggers when `zombies.length === 0` after array splice. This could trigger prematurely if zombies are still being spawned.

**Zombie Movement:**
The Car class has contradictory movement logic:

```javascript
if(game.state === game.state_RUNNING && this.x >= 800){
    this.x -= 1
} else if (game.state === game.state_RUNNING && this.x <= 150){
    this.x += 1
}
```

This creates oscillation when x is between 150 and 800, which appears unintentional.

---

## 6. Risk Assessment

| Risk | Likelihood | Impact | Mitigation Priority |
|------|------------|--------|---------------------|
| Memory leaks from uncleared timers | High | Medium | High |
| State inconsistency bugs | Medium | High | High |
| Array index errors during combat | Medium | Medium | Medium |
| Difficulty extending with new plants/zombies | High | Medium | Medium |
| Performance degradation on long sessions | Medium | Medium | Medium |
| Canvas coordinate misalignment | Low | Low | Low |

---

## 7. Recommendations

### 7.1 Immediate Actions (High Priority)

1. **Centralize State Management**
   - Create a dedicated `GameState` class that owns all entity collections
   - Pass state references through constructors instead of global access
   - Implement observer pattern for state change notifications

2. **Fix Timer Lifecycle**
   - Audit all `setTimeout`/`setInterval` calls
   - Implement a `TimerManager` that tracks active timers
   - Ensure cleanup on game state changes and entity destruction

3. **Extract Configuration**
   - Move all magic numbers to a `config.js` file:
     - Grid dimensions (cell size, offsets)
     - Timing constants (cooldowns, spawn intervals)
     - Game balance values (health, damage, costs)

4. **Fix Array Mutation Patterns**
   - Use `filter()` to create new arrays instead of splicing during iteration
   - Or iterate backwards when removing elements

### 7.2 Short-Term Improvements (Medium Priority)

5. **Implement Module Pattern**
   - Wrap each file in IIFE or use ES6 modules
   - Explicitly export public APIs
   - Remove dependency on `window._main`

6. **Separate Concerns**
   - Extract rendering logic into dedicated `Renderer` classes
   - Create `Physics` module for collision detection
   - Move DOM manipulation out of entity classes

7. **Add Input Validation**
   - Validate grid coordinates before array access
   - Check for null/undefined before dereferencing

8. **Implement Proper Zombie Spawning**
   - Create zombies on-demand instead of upfront
   - Use object pooling to reuse zombie instances

### 7.3 Long-Term Enhancements (Lower Priority)

9. **Testing Infrastructure**
   - Add Jest or Vitest for unit testing
   - Create test utilities for canvas mocking
   - Write tests for collision detection and state transitions

10. **Build Pipeline**
    - Add bundler (Vite, Webpack, or Rollup)
    - Implement minification for production
    - Add linting (ESLint) and formatting (Prettier)

11. **Type Safety**
    - Migrate to TypeScript for compile-time type checking
    - Define interfaces for entity configurations

12. **Performance Optimization**
    - Implement spatial hashing for collision detection
    - Use `requestAnimationFrame` instead of `setInterval` for game loop
    - Add canvas dirty-rectangle rendering

---

## 8. Conclusion

This Plants vs Zombies implementation successfully demonstrates core gameplay mechanics and provides a functional browser-based gaming experience. The animation system and state machine implementations show solid understanding of game development fundamentals.

However, the codebase exhibits architectural patterns that limit maintainability and extensibility. The heavy reliance on global state, tight coupling between modules, and inconsistent resource management create technical debt that will compound with each new feature.

**Recommendation:** The project is suitable for educational purposes and personal use in its current state. For production use or collaborative development, a refactoring pass addressing the high-priority issues (state management, timer lifecycle, configuration centralization) should be completed before adding new features.

The existing code structure provides a solid foundation for these improvements - the class hierarchy is reasonable, and the separation into four modules provides clear refactoring boundaries.

---

## Appendix: File Inventory

| File | Lines | Purpose |
|------|-------|---------|
| index.html | 88 | Entry point, DOM structure |
| css/common.css | 50 | Reset and utility styles |
| css/style.css | 195 | Game-specific styles |
| js/common.js | 179 | Asset configuration |
| js/scene.js | ~1000 | Entity classes |
| js/game.js | 630 | Game engine |
| js/main.js | 232 | Initialization |
| package.json | 14 | Dependency metadata |

**Total JavaScript:** ~2000 lines  
**Total Project:** ~2100 lines + image assets

---

*Document generated through static code analysis. Functional verification was limited to logic tracing; actual runtime behavior may vary.*
