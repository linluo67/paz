# Project Review Document: Plants vs. Zombies JavaScript Implementation

**Document Version:** 1.0  
**Review Date:** 2026-03-30  
**Project Name:** paz (Plants vs. Zombies JavaScript Version)  
**Review Method:** Static Code Analysis + Limited Scope Functional Verification

---

## 1. Project Overview

This project is a browser-based tower defense game inspired by the classic "Plants vs. Zombies" franchise. It is implemented using vanilla HTML, CSS, JavaScript, and the HTML5 Canvas API, without any modern frontend framework or build toolchain. The game allows players to place various plants on a lawn grid to defend against waves of approaching zombies.

**Key Characteristics:**
- Single-page application with no module bundler
- Direct script loading via `<script>` tags
- Sprite-based animation system using Canvas 2D rendering
- Event-driven game loop with fixed frame rate
- No automated testing infrastructure

---

## 2. Runtime and Technical Environment

### 2.1 Technology Stack

| Component | Technology | Notes |
|-----------|------------|-------|
| Markup | HTML5 | Single entry file: `index.html` |
| Styling | CSS3 | Two stylesheets: `common.css`, `style.css` |
| Logic | ES6 JavaScript | Classes, arrow functions, template literals |
| Rendering | HTML5 Canvas | 1400x600 pixel game canvas |
| Animation | Web Animations API | Used for sun collection effects |
| Audio | HTML5 Audio | Background music and sound effects |

### 2.2 Execution Requirements

- **Runtime:** Modern web browser with ES6 support (Chrome, Firefox, Edge, Safari)
- **Server:** Any static file server (required for audio/image loading due to CORS)
- **Dependencies:** jQuery 3.6.0 (listed in package.json but not actively used in core code)

### 2.3 Launch Procedure

1. Clone repository or download source files
2. Serve the project directory via a local HTTP server
3. Open `index.html` in a web browser
4. Click the start button to begin the game

**Note:** The `package.json` contains no meaningful npm scripts. The `test` command merely echoes an error message.

---

## 3. Directory Structure and Module Responsibilities

```
paz/
├── index.html              # Entry point, DOM structure, script loading
├── css/
│   ├── common.css          # CSS reset and utility classes
│   └── style.css           # Game-specific styles
├── js/
│   ├── common.js           # Image path mapping, helper functions
│   ├── scene.js            # Core game objects (classes)
│   ├── game.js             # Game engine, state machine, rendering
│   └── main.js             # Initialization, configuration, timers
├── images/                 # Sprite assets (plants, zombies, cards, etc.)
├── docs/                   # Documentation (newly created)
├── package.json            # Project metadata (minimal)
└── README.md               # Basic usage instructions
```

### 3.1 Module Responsibility Analysis

| Module | Primary Responsibility | Lines of Code (Approx.) |
|--------|----------------------|------------------------|
| `common.js` | Image path configuration, `imageFromPath()` helper | ~180 |
| `scene.js` | Game object classes: `SunNum`, `Card`, `Plant`, `Zombie`, `Bullet`, `Animation`, `Car`, `Shovel` | ~1000 |
| `game.js` | Game engine: state machine, Canvas rendering, input handling, collision detection | ~630 |
| `main.js` | Entry point: `Main` class, initialization, timer management | ~230 |

**Observation:** The module boundaries are reasonably clear, but `scene.js` has grown too large, containing nearly all game entity classes. This creates a "fat model" situation that complicates maintenance.

---

## 4. Architecture and Module Analysis

### 4.1 Overall Architecture Pattern

The project follows a **procedural game loop pattern** with object-oriented entity modeling:

```
┌─────────────────────────────────────────────────────────────┐
│                        index.html                            │
│  (DOM Structure + Script Loading Order)                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      main.js (Main)                          │
│  - Global state container (window._main)                     │
│  - Entity creation and configuration                         │
│  - Timer management (sun/zombie spawning)                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      game.js (Game)                          │
│  - Game state machine (LOADING/START/RUNNING/STOP/WON)       │
│  - Canvas rendering pipeline                                 │
│  - Input event handling                                      │
│  - Frame-based update loop (setInterval)                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     scene.js (Entities)                      │
│  - Role (base class)                                         │
│  - Plant (extends Role): sunflower, peashooter, etc.         │
│  - Zombie (extends Role): movement, attack, death states     │
│  - Bullet, Card, SunNum, Car, Shovel, Animation              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     common.js (Config)                       │
│  - allImg: sprite path mappings                              │
│  - imageFromPath(): image loader utility                     │
│  - keyframesOptions: animation configuration                 │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Class Hierarchy

```
Role (base)
├── Plant
│   ├── Sunflower (generates sun)
│   ├── Peashooter (shoots bullets)
│   ├── Repeater (shoots two bullets)
│   ├── GatlingPea (rapid fire)
│   ├── WallNut (high HP, damage states)
│   ├── CherryBomb (area explosion)
│   └── Chomper (one-hit kill, digest animation)
└── Zombie
    └── Standard Zombie (walk, attack, die animations)
```

### 4.3 Game State Machine

The `Game` class implements a finite state machine with six states:

| State | Value | Description |
|-------|-------|-------------|
| `state_LOADING` | 0 | Initial loading screen |
| `state_START` | 1 | Start animation playing |
| `state_RUNNING` | 2 | Active gameplay |
| `state_STOP` | 3 | Paused |
| `state_PLANTWON` | 4 | All zombies eliminated |
| `state_ZOMBIEWON` | 5 | Zombie reached the house |

### 4.4 Data Flow

1. **Initialization:** `Main` constructor sets default values → `start()` creates entities → `Game.new()` starts game loop
2. **Game Loop:** `setInterval` at 60 FPS → `setTimer()` dispatches by state → render/update entities
3. **User Input:** DOM event listeners → modify `Game` state flags → next frame reflects changes
4. **Entity Updates:** Each frame, `update()` methods recalculate positions, animations, and collisions

---

## 5. Key Functional Implementation Analysis

### 5.1 Animation System

The `Animation` class implements a **frame-counter based sprite animation** system:

```javascript
// Animation speed is controlled by fps parameter
self[stateName].imgIdx = Math.floor(self[stateName].count / self[stateName].fps)
```

**Strengths:**
- Simple and effective for sprite-based games
- Supports multiple animation states per entity
- Handles composite animations (head/body for dying zombies)

**Weaknesses:**
- Frame timing depends on main loop FPS, not actual elapsed time
- No animation event callbacks (completion detection is manual)
- Animation state transitions are scattered across entity classes

### 5.2 Collision Detection

Collision detection is implemented within `Plant.canAttack()` and `Zombie.canAttack()`:

```javascript
// Bullet-zombie collision (simplified bounding box)
if (Math.abs(zombie.x + bullet.w - bullet.x) < 10 && zombie.life > 0) {
  // Handle hit
}

// Zombie-plant collision (range-based)
if (self.x - plant.x < -20 && self.x - plant.x > -60) {
  // Start attacking
}
```

**Issues Identified:**
- Collision logic is tightly coupled with attack logic
- No separation between collision detection and collision response
- Hard-coded magic numbers (-20, -60, 10) reduce maintainability
- No spatial partitioning for performance optimization

### 5.3 Sun Collection System

Suns are created as DOM elements with Web Animations API:

```javascript
let img = document.createElement('img')
img.className += 'sun-img plantSun' + id
container.appendChild(img)
sun.animate(keyframes1, keyframesOptions)
```

**Concerns:**
- Mixes DOM manipulation with Canvas rendering
- Sun elements are not managed by the game state
- Potential memory leaks if elements are not properly cleaned up
- The `sunxiaoshi` timeout reference is attached to `this` but may not be cleared in all cases

### 5.4 Timer Management

Multiple timers are used throughout:

| Timer | Location | Purpose |
|-------|----------|---------|
| `Game.timer` | `game.js` | Main game loop (setInterval) |
| `Main.sunTimer` | `main.js` | Global sun spawning |
| `Main.zombieTimer` | `main.js` | Zombie wave activation |
| `Plant.sunTimer` | `scene.js` | Sunflower sun generation |
| `Card.timer` | `scene.js` | Card cooldown countdown |

**Critical Issue:** Timer cleanup is inconsistent. The `clearTiemr()` method (note: typo in name) clears some timers but not all. Card timers use both `setInterval` and `setTimeout` without proper tracking.

---

## 6. Code Quality Assessment

### 6.1 Positive Aspects

1. **ES6 Class Syntax:** Modern JavaScript patterns improve readability
2. **Static Factory Methods:** `Class.new()` pattern provides clean instantiation
3. **State Machine Pattern:** Clear game state transitions
4. **Sprite Organization:** Well-structured image path configuration
5. **Responsive Controls:** Mouse tracking and click handling work as expected

### 6.2 Code Smells and Anti-Patterns

#### 6.2.1 Global Variable Proliferation

The project relies heavily on `window._main` as a global singleton:

```javascript
window._main = new Main()
window._main.start()
```

This is accessed from multiple files:
- `game.js`: `window._main.plants`, `window._main.zombies`, `window._main.sunnum`
- `scene.js`: `window._main.allSunVal`, `window._main.zombies_info`

**Impact:** Creates implicit dependencies, makes testing difficult, violates encapsulation.

#### 6.2.2 Magic Numbers

Hard-coded values appear throughout:

```javascript
// Position calculations
x: _main.plants_info.x + 80 * (col - 1)
y: _main.plants_info.y + 100 * (row - 1)

// Collision ranges
if (self.x - plant.x < -20 && self.x - plant.x > -60)

// Canvas dimensions
width=1400 height=600
```

**Recommendation:** Extract to named constants in a configuration module.

#### 6.2.3 Inconsistent Naming Conventions

- `clearTiemr()` - typo (should be `clearTimer`)
- `sunxiaoshi` - Chinese Pinyin mixed with English
- `state_NORMALE` in `Car` class (should be `state_NORMAL`)
- Mixed camelCase and snake_case for timer variables

#### 6.2.4 Deeply Nested Conditionals

The `canAttack()` methods contain complex nested logic:

```javascript
for (let zombie of window._main.zombies) {
  if (self.section === 'cherrybomb') {
    if (Math.abs(self.row - zombie.row) <= 1 && ...) {
      // ...
    }
  } else if (self.section === 'chomper' && self.state === self.state_IDLE) {
    if (self.row === zombie.row && ...) {
      // ...
    }
  } else if (self.canShoot && self.row === zombie.row) {
    // ...
  }
}
```

**Recommendation:** Extract into separate strategy methods per plant type.

#### 6.2.5 DOM Manipulation from Entity Classes

The `Plant.setSunTimer()` method directly manipulates the DOM:

```javascript
let img = document.createElement('img')
container.appendChild(img)
```

This violates separation of concerns and makes the code harder to test.

---

## 7. Sampling Test Scope and Methodology

### 7.1 Test Approach

Given the absence of automated tests, the following verification methods were applied:

1. **Static Code Analysis:** Manual review of all JavaScript files
2. **Code Path Tracing:** Following execution flow from entry point
3. **Interface Inspection:** Checking for undefined references and type mismatches
4. **Resource Verification:** Confirming image paths match actual file structure

### 7.2 Findings from Static Analysis

| Check | Result | Notes |
|-------|--------|-------|
| Script loading order | PASS | Correct dependency order in index.html |
| Image path validity | PARTIAL | Most paths correct; some may be case-sensitive on Linux |
| Undefined references | FAIL | `this.sunxiaoshi` may be undefined in some contexts |
| Memory leak potential | WARNING | Timers and DOM elements may not be cleaned up |
| Type safety | N/A | No TypeScript or runtime type checking |

### 7.3 Functional Verification Limitations

Without executing the game in a browser, the following cannot be verified:
- Actual gameplay balance
- Animation smoothness
- Audio playback
- Edge case handling (e.g., rapid clicking, zero sun scenario)

---

## 8. Major Issues Discovered

### 8.1 Critical Issues (Severity: High)

#### Issue 1: Timer Memory Leaks

**Location:** Multiple files  
**Description:** Timers are created but not consistently tracked or cleared. When the game is paused, restarted, or ended, some timers may continue running.

**Evidence:**
```javascript
// Card.drawCountDown() creates interval but doesn't store reference properly
self.timer = setInterval(function () { ... }, 1000)

// Plant.setSunTimer() creates timeout attached to 'this' context
this.sunxiaoshi = setTimeout(() => { ... }, 4000)
```

**Impact:** Memory leaks, unexpected behavior after game state changes.

**Recommendation:** Create a centralized timer manager that tracks all active timers and provides cleanup methods.

#### Issue 2: Inconsistent State Management

**Location:** `game.js`, `scene.js`  
**Description:** Game state is distributed across multiple objects (`Game.state`, entity states, UI flags) without a single source of truth.

**Evidence:**
```javascript
// Game checks its own state
if (game.state === game.state_RUNNING) { ... }

// But entities also have internal states
self.state = self.state_ATTACK

// And UI has separate flags
g.canDrawMousePlant = true
```

**Impact:** State synchronization bugs, difficult to debug state-related issues.

**Recommendation:** Implement a centralized state store (even a simple one) to manage all game state.

### 8.2 Moderate Issues (Severity: Medium)

#### Issue 3: Tight Coupling Between Modules

**Location:** Cross-file references  
**Description:** Modules directly access each other's internals through `window._main`.

**Evidence:**
```javascript
// scene.js accessing main.js state
window._main.allSunVal

// scene.js accessing game.js entities
window._main.zombies

// game.js accessing main.js configuration
_main.plants_info.x
```

**Impact:** Cannot test modules in isolation, changes propagate unexpectedly.

**Recommendation:** Use dependency injection or a service locator pattern.

#### Issue 4: Error Handling Absence

**Location:** All JavaScript files  
**Description:** No try-catch blocks, no error boundaries, no validation of external resources.

**Evidence:**
```javascript
// Image loading without error handling
let img = new Image()
img.src = './images/' + src
return img

// DOM queries without null checks
document.getElementsByClassName('plantSun' + id)[0].onclick = function() { ... }
```

**Impact:** Silent failures, hard-to-debug runtime errors.

**Recommendation:** Add error handling for resource loading and DOM operations.

#### Issue 5: Hard-coded Configuration

**Location:** Throughout codebase  
**Description:** Game parameters are embedded directly in code rather than in configuration.

**Examples:**
- Initial sun value: 9999
- Zombie HP: 10
- Sun generation interval: 15 seconds
- Canvas dimensions: 1400x600

**Impact:** Difficult to balance game, requires code changes for tuning.

**Recommendation:** Extract to a `config.js` module.

### 8.3 Minor Issues (Severity: Low)

#### Issue 6: Typographical Errors

- `clearTiemr()` should be `clearTimer()`
- `state_NORMALE` should be `state_NORMAL`
- Comment inconsistency: "阳关元素" should be "阳光元素"

#### Issue 7: Unused Dependencies

**Location:** `package.json`  
**Description:** jQuery 3.6.0 is listed as a dependency but not used in the codebase.

#### Issue 8: Inconsistent Code Style

- Some functions use `function` keyword, others use arrow functions
- Indentation is inconsistent in places
- Some files have trailing whitespace

---

## 9. Strengths and Commendable Practices

### 9.1 Architectural Strengths

1. **Clear Entry Point:** `index.html` provides an unambiguous starting point
2. **Class-Based Design:** ES6 classes with inheritance (`Plant extends Role`, `Zombie extends Role`)
3. **Factory Pattern:** Static `new()` methods provide consistent instantiation
4. **State Machine:** Game states are explicitly defined and managed

### 9.2 Implementation Strengths

1. **Sprite Animation System:** Flexible enough to handle multiple animation states
2. **Entity Component Design:** Each entity manages its own state and rendering
3. **Responsive UI:** Card tooltips, hover states, and visual feedback work correctly
4. **Asset Organization:** Clear directory structure for images by category

### 9.3 Documentation Strengths

1. **README:** Basic usage instructions provided
2. **Inline Comments:** Chinese comments explain key logic sections
3. **Preview Images:** Screenshots help users understand the game

---

## 10. Risk Assessment

| Risk Category | Likelihood | Impact | Risk Level |
|---------------|------------|--------|------------|
| Memory leaks from uncleared timers | High | High | **Critical** |
| State synchronization bugs | Medium | High | **High** |
| Browser compatibility issues | Low | Medium | **Low** |
| Performance degradation with many entities | Medium | Medium | **Medium** |
| Resource loading failures | Medium | High | **High** |
| Code maintainability decline | High | Medium | **High** |

---

## 11. Improvement Recommendations

### 11.1 Short-Term (Immediate Actions)

1. **Fix Timer Management**
   - Create a `TimerManager` class to track all active timers
   - Implement `pauseAll()` and `resumeAll()` methods
   - Ensure all timers are cleared on game end

2. **Add Error Handling**
   - Wrap image loading in error handlers
   - Add null checks for DOM queries
   - Log errors to console for debugging

3. **Extract Configuration**
   - Create `config.js` with all game parameters
   - Make initial sun, zombie HP, timers configurable

4. **Fix Typographical Errors**
   - Correct method names and variable names
   - Update comments for accuracy

### 11.2 Medium-Term (Refactoring)

1. **Module Separation**
   - Split `scene.js` into individual files per class
   - Consider using ES6 modules with `import/export`

2. **State Management Refactoring**
   - Create a central `GameState` object
   - Use observer pattern for state changes
   - Remove direct `window._main` access

3. **Collision System Improvement**
   - Separate collision detection from response
   - Use a spatial hash for performance
   - Define collision layers/groups

4. **Add Basic Testing**
   - Set up Jest or Mocha
   - Write unit tests for core classes
   - Add integration tests for game flow

### 11.3 Long-Term (Architectural)

1. **Build Toolchain**
   - Add Vite or Webpack for bundling
   - Enable code splitting and minification
   - Add source maps for debugging

2. **TypeScript Migration**
   - Add type definitions
   - Enable strict mode
   - Improve IDE support and catch errors at compile time

3. **Game Engine Abstraction**
   - Consider using a lightweight engine (Phaser, PixiJS)
   - Or create a minimal custom engine layer

---

## 12. Final Review Conclusion

### 12.1 Overall Assessment

This project demonstrates a **functional but architecturally fragile** implementation of a Plants vs. Zombies-style game. The core gameplay mechanics are implemented correctly, and the sprite-based animation system is effective for this type of game. However, the codebase exhibits several patterns that will impede long-term maintenance and scalability.

### 12.2 Key Findings Summary

| Aspect | Rating | Justification |
|--------|--------|---------------|
| Functionality | Good | Core game loop works, all plant types functional |
| Code Organization | Fair | Clear module boundaries, but `scene.js` is too large |
| Code Quality | Fair | Modern syntax, but global variables and magic numbers |
| Maintainability | Poor | Tight coupling, no tests, scattered state |
| Performance | Unknown | Cannot verify without runtime testing |
| Documentation | Fair | Basic README, inline comments in Chinese |

### 12.3 Recommendation

**Conditional Approval for Educational/Demo Purposes**

The project is suitable as a demonstration of Canvas-based game development techniques and can serve as a learning resource. However, it is **not recommended for production deployment** without significant refactoring, particularly:

1. Timer management overhaul
2. State management centralization
3. Addition of automated tests
4. Error handling implementation

### 12.4 Next Steps

1. Address critical issues (timer leaks, state synchronization)
2. Set up a basic test framework
3. Refactor for better modularity
4. Consider TypeScript migration for larger scale development

---

**Document Prepared By:** Code Review Analysis  
**Review Methodology:** Static Code Analysis + Architectural Assessment  
**Confidence Level:** High (based on comprehensive code review)
