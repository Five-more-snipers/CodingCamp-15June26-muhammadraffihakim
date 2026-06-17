# Design Document — To-Do Life Dashboard

## Overview

The To-Do Life Dashboard is a single-page, client-side web application built with plain HTML, CSS, and Vanilla JavaScript. There is no build step, no framework, and no network dependency at runtime. All state is held in memory during a session and flushed to `localStorage` on every mutation, so the page can be refreshed at any time without losing data.

The page is divided into four **widgets** rendered in a CSS Grid layout:

| Widget | Purpose |
|---|---|
| **Greeting** | Live clock, date, time-of-day greeting, optional user name |
| **Focus Timer** | Pomodoro countdown with configurable duration |
| **To-Do List** | Task CRUD with inline editing and completion state |
| **Quick Links** | Saved bookmarks that open in a new tab |

A theme toggle (light / dark) sits in the page header and applies to all widgets simultaneously.

---

## Architecture

### Module Organisation

Because the project is a single JavaScript file (`js/app.js`), the code is structured as a collection of **pure functions and small sub-modules** wrapped in an IIFE to avoid polluting the global scope. There are no `import` / `export` statements (no bundler), so logical separation is achieved by naming convention and comment sections.

```
js/app.js
├── CONSTANTS          – localStorage keys, defaults, limits
├── Storage            – thin read/write wrappers around localStorage
├── State              – in-memory application state object
├── Greeting           – clock, date, greeting logic + DOM update
├── Timer              – countdown logic, button state machine
├── TodoList           – task CRUD, validation, render
├── QuickLinks         – link CRUD, validation, render
├── Theme              – toggle, persist, apply
└── Bootstrap          – DOMContentLoaded handler that wires everything up
```

All sub-modules are plain objects (namespaces) whose methods close over `State` and call `Storage` to persist on every change.

### Data Flow

```
User Interaction
      │
      ▼
Event Listener (delegated or direct)
      │
      ▼
Validate Input  ──► Show inline error → stop
      │
      ▼
Mutate State (in-memory)
      │
      ├──► Storage.save(key, value)   ← synchronous, before returning
      │
      └──► render*(…)                 ← re-render only the affected widget
```

Re-renders are **targeted**: each widget has a single `render()` function that rebuilds only its own DOM subtree from the current state slice, avoiding full-page repaints.

### Timer State Machine

```
IDLE ──start──► RUNNING ──stop──► PAUSED ──start──► RUNNING
  ▲                │                  │
  └───reset────────┘                  │
  ▲                                   │
  └────────────reset──────────────────┘
RUNNING ──reaches 00:00──► COMPLETED
COMPLETED ──reset──► IDLE
```

---

## Components and Interfaces

### Greeting Widget

**HTML structure (within `<header>`):**

```html
<header id="greeting-widget">
  <div class="greeting-clock">
    <span id="time-display">00:00:00</span>
  </div>
  <div class="greeting-date">
    <span id="date-display">Monday, 14 July 2025</span>
  </div>
  <div class="greeting-message">
    <span id="greeting-display">Good Morning</span>
  </div>
  <div class="greeting-name-form">
    <input  id="name-input" type="text" maxlength="50"
            placeholder="Enter your name…" autocomplete="off">
    <button id="btn-save-name">Save</button>
    <span   id="name-error" class="error-hint" aria-live="polite"></span>
  </div>
  <button id="btn-theme-toggle" aria-label="Toggle theme">🌙</button>
</header>
```

**JS interface — `Greeting` namespace:**

```
Greeting.init()               – attach clock interval, restore name, render
Greeting.tick()               – compute time/date/greeting, update DOM spans
Greeting.saveName(value)      – validate (<= 50 chars, non-empty), persist, render
Greeting.clearName()          – remove from state + storage, render
Greeting.renderGreeting()     – compose "[Message], [Name]!" string → #greeting-display
```

**Clock interval:** `setInterval(Greeting.tick, 1000)` started in `Greeting.init()`.

---

### Focus Timer Widget

**HTML structure (within `<section id="timer-section">`):**

```html
<section id="timer-section">
  <h2 class="widget-title">Focus Timer</h2>
  <div id="timer-display" class="timer-display">25:00</div>
  <div id="timer-completion" class="timer-completion hidden"
       aria-live="polite">
    🎉 Session complete! Take a break.
  </div>
  <div class="timer-controls">
    <button id="btn-start">Start</button>
    <button id="btn-stop"  disabled>Stop</button>
    <button id="btn-reset">Reset</button>
  </div>
  <div class="timer-config">
    <label for="duration-input">Duration (min)</label>
    <input  id="duration-input" type="number"
            min="1" max="120" value="25">
    <button id="btn-save-duration">Save</button>
    <span   id="duration-error" class="error-hint" aria-live="polite"></span>
  </div>
</section>
```

**JS interface — `Timer` namespace:**

```
Timer.init()                  – restore duration from storage, render, wire buttons
Timer.start()                 – set state.timer.running=true, create interval
Timer.stop()                  – clear interval, state.timer.running=false
Timer.reset()                 – stop, restore duration, clear completion banner
Timer.tick()                  – decrement secondsLeft; if 0 → complete()
Timer.complete()              – stop, show completion banner, update button states
Timer.saveDuration(value)     – validate 1-120, persist, reset timer to new duration
Timer.render()                – update #timer-display, toggle button disabled states,
                                toggle #timer-completion visibility,
                                toggle #duration-input disabled
Timer.formatTime(seconds)     – returns "MM:SS" string
```

---

### To-Do List Widget

**HTML structure (within `<section id="todo-section">`):**

```html
<section id="todo-section">
  <h2 class="widget-title">To-Do List</h2>
  <div class="todo-input-row">
    <input  id="task-input" type="text" maxlength="200"
            placeholder="Add a new task…" autocomplete="off">
    <button id="btn-add-task">Add</button>
  </div>
  <span id="task-error" class="error-hint" aria-live="polite"></span>
  <ul   id="task-list" aria-label="Task list"></ul>
</section>
```

Each task item rendered by `TodoList.renderItem(task)`:

```html
<li class="task-item" data-id="<task.id>">
  <input type="checkbox" class="task-check"
         aria-label="Mark task complete" [checked]>
  <span  class="task-text [task-done]">[task.text]</span>
  <button class="btn-edit-task"   aria-label="Edit task">✏️</button>
  <button class="btn-delete-task" aria-label="Delete task">🗑️</button>
</li>
```

When editing, `renderItem` swaps the `<span>` for:

```html
<input  type="text" class="task-edit-input"
        value="[task.text]" maxlength="200">
<button class="btn-save-edit"   aria-label="Save edit">✔</button>
<button class="btn-cancel-edit" aria-label="Cancel edit">✖</button>
<span   class="edit-error error-hint" aria-live="polite"></span>
```

**JS interface — `TodoList` namespace:**

```
TodoList.init()               – restore tasks from storage, render
TodoList.addTask(text)        – validate (1-200 chars, non-whitespace), create task
                                object, push to state.tasks, persist, render
TodoList.deleteTask(id)       – filter state.tasks, persist, render
TodoList.toggleTask(id)       – flip task.done, persist, render
TodoList.startEdit(id)        – set task.editing=true, render
TodoList.confirmEdit(id,text) – validate non-empty, update task.text, editing=false,
                                persist, render
TodoList.cancelEdit(id)       – set task.editing=false, render
TodoList.render()             – rebuild #task-list innerHTML from state.tasks
TodoList.renderItem(task)     – returns HTMLElement for one task
```

Events are **delegated** to `#task-list` via `event.target.closest('[data-id]')` to handle dynamically added items without re-binding on every render.

---

### Quick Links Widget

**HTML structure (within `<section id="links-section">`):**

```html
<section id="links-section">
  <h2 class="widget-title">Quick Links</h2>
  <div class="links-input-row">
    <input  id="link-name-input" type="text" maxlength="100"
            placeholder="Link name…" autocomplete="off">
    <span   id="link-name-error" class="error-hint" aria-live="polite"></span>
    <input  id="link-url-input"  type="url"  maxlength="2048"
            placeholder="https://…" autocomplete="off">
    <span   id="link-url-error"  class="error-hint" aria-live="polite"></span>
    <button id="btn-add-link">Add</button>
  </div>
  <div id="links-list" aria-label="Quick links"></div>
</section>
```

Each link rendered by `QuickLinks.renderItem(link)`:

```html
<div class="link-item" data-id="<link.id>">
  <a href="[link.url]" target="_blank" rel="noopener noreferrer"
     class="link-btn">[link.name]</a>
  <button class="btn-delete-link" aria-label="Delete link">🗑️</button>
</div>
```

**JS interface — `QuickLinks` namespace:**

```
QuickLinks.init()             – restore links from storage, render
QuickLinks.addLink(name, url) – validate name (1-100), URL (http/https scheme,
                                non-empty host, <=2048), create link object,
                                push to state.links, persist, render
QuickLinks.deleteLink(id)     – filter state.links, persist, render
QuickLinks.render()           – rebuild #links-list innerHTML from state.links
QuickLinks.renderItem(link)   – returns HTMLElement for one link
QuickLinks.validateURL(url)   – returns {ok, error}; uses new URL() constructor
```

---

### Theme Module

**JS interface — `Theme` namespace:**

```
Theme.init()                  – read from storage → fall back to prefers-color-scheme
                                → apply before first paint
Theme.apply(theme)            – document.documentElement.dataset.theme = theme;
                                update toggle button icon/label
Theme.toggle()                – flip between "light" and "dark", persist, apply
Theme.persist(theme)          – Storage.save(KEYS.THEME, theme)
```

Applying a theme writes `data-theme="light"` or `data-theme="dark"` to `<html>`. All colour values are CSS custom properties scoped under `[data-theme="light"]` / `[data-theme="dark"]`, so a single attribute swap re-themes the entire page instantly (no class toggling on individual elements).

`Theme.init()` is called synchronously from a `<script>` tag placed immediately after `<html>` (or via `DOMContentLoaded` with the default applied via a CSS `:root` rule) to prevent flash of wrong theme.

---

## Data Models

### In-Memory State

```js
const State = {
  userName: "",          // string, "" means no name set
  tasks: [],             // Task[]
  links: [],             // Link[]
  timer: {
    durationMinutes: 25, // integer 1-120
    secondsLeft: 1500,   // integer, derived from durationMinutes on reset/load
    running: false,      // boolean
    status: "IDLE",      // "IDLE" | "RUNNING" | "PAUSED" | "COMPLETED"
    intervalId: null,    // setInterval handle (not persisted)
  },
  theme: "light",        // "light" | "dark"
};
```

### Task Object

```js
{
  id:      string,   // crypto.randomUUID() or Date.now().toString()
  text:    string,   // 1-200 chars
  done:    boolean,
  editing: boolean,  // runtime-only; never persisted
  createdAt: number  // Date.now() timestamp
}
```

### Link Object

```js
{
  id:   string,  // crypto.randomUUID() or Date.now().toString()
  name: string,  // 1-100 chars
  url:  string   // valid http/https URL, <=2048 chars
}
```

### localStorage Schema

| Key | Type | Description |
|---|---|---|
| `tld_userName` | `string` (JSON) | Saved user name or absent |
| `tld_tasks` | `Task[]` (JSON) | Array of task objects (without `editing` field) |
| `tld_links` | `Link[]` (JSON) | Array of link objects |
| `tld_pomoDuration` | `number` (JSON) | Integer minutes 1-120 |
| `tld_theme` | `string` (JSON) | `"light"` or `"dark"` |

All values are stored with `JSON.stringify` and retrieved with `JSON.parse` wrapped in a `try/catch`. On parse failure the corrupted key is deleted and the in-memory default is used.

```js
const KEYS = {
  USER_NAME:     "tld_userName",
  TASKS:         "tld_tasks",
  LINKS:         "tld_links",
  POMO_DURATION: "tld_pomoDuration",
  THEME:         "tld_theme",
};
```

### CSS Custom Properties (Theme)

```css
:root {
  /* Light theme defaults */
  --color-bg:        #f5f5f5;
  --color-surface:   #ffffff;
  --color-text:      #1a1a1a;
  --color-primary:   #4f46e5;
  --color-danger:    #dc2626;
  --color-border:    #d1d5db;
  --color-muted:     #6b7280;
  --transition-theme: 300ms ease;
}

[data-theme="dark"] {
  --color-bg:       #121212;
  --color-surface:  #1e1e1e;
  --color-text:     #e5e5e5;
  --color-primary:  #818cf8;
  --color-danger:   #f87171;
  --color-border:   #374151;
  --color-muted:    #9ca3af;
}
```

All widget colours reference only these custom properties so that swapping `data-theme` on `<html>` instantly re-themes everything.

---

## CSS Layout

```css
/* Dashboard grid: 2×2 on wide viewports, single column on mobile */
.dashboard-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
  gap: 1.5rem;
  padding: 1.5rem;
}
```

Each widget `<section>` is a card:

```css
.widget {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: 0.75rem;
  padding: 1.25rem;
}
```

Transitions on background/color ensure smooth theme switching within 300 ms:

```css
*, *::before, *::after {
  transition: background-color var(--transition-theme),
              color var(--transition-theme),
              border-color var(--transition-theme);
}
```

---

## Event Handling Strategy

### Principle: Minimal Direct Bindings + Event Delegation

- **Direct bindings** are used only for static, always-present controls: `#btn-start`, `#btn-stop`, `#btn-reset`, `#btn-save-name`, `#btn-save-duration`, `#btn-add-task`, `#btn-add-link`, `#btn-theme-toggle`.
- **Delegated listeners** on `#task-list` and `#links-list` handle clicks/changes on dynamically generated items (edit, delete, toggle). The handler reads `event.target.closest('[data-id]')` to find the item and the button class to determine the action.
- **Keyboard events** (`keydown`) are bound to the static input fields for Enter/Escape shortcuts.

### Binding Table

| Element | Event | Handler |
|---|---|---|
| `#btn-save-name` | click | `Greeting.saveName` |
| `#name-input` | keydown (Enter) | `Greeting.saveName` |
| `#btn-theme-toggle` | click | `Theme.toggle` |
| `#btn-start` | click | `Timer.start` |
| `#btn-stop` | click | `Timer.stop` |
| `#btn-reset` | click | `Timer.reset` |
| `#btn-save-duration` | click | `Timer.saveDuration` |
| `#btn-add-task` | click | `TodoList.addTask` |
| `#task-input` | keydown (Enter) | `TodoList.addTask` |
| `#task-list` | click (delegated) | `TodoList.handleClick` |
| `#task-list` | keydown (delegated) | `TodoList.handleKeydown` |
| `#btn-add-link` | click | `QuickLinks.addLink` |
| `#links-list` | click (delegated) | `QuickLinks.handleClick` |

All listeners are attached once in `Bootstrap.init()` which is called inside the `DOMContentLoaded` handler.

---


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

---

### Property 1: Time Formatting Correctness

*For any* `Date` object, `formatTime(date)` SHALL return a string matching the pattern `HH:MM:SS` where HH ∈ [00, 23], MM ∈ [00, 59], and SS ∈ [00, 59].

**Validates: Requirements 1.1**

---

### Property 2: Date Formatting Correctness

*For any* `Date` object, `formatDate(date)` SHALL return a string in the format `"Weekday, DD Month YYYY"` where Weekday is one of the seven English day names, Month is one of the twelve English month names, DD is a zero-padded day number, and YYYY is a four-digit year.

**Validates: Requirements 1.2**

---

### Property 3: Greeting Message by Hour

*For any* integer hour `h` in [0, 23]:
- `getGreeting(h)` returns `"Good Night"` when `h` ∈ [21, 23] ∪ [0, 4]
- `getGreeting(h)` returns `"Good Morning"` when `h` ∈ [5, 11]
- `getGreeting(h)` returns `"Good Afternoon"` when `h` ∈ [12, 17]
- `getGreeting(h)` returns `"Good Evening"` when `h` ∈ [18, 20]

Every input maps to exactly one greeting with no overlaps or gaps.

**Validates: Requirements 1.3, 1.4, 1.5, 1.6**

---

### Property 4: Personalized Greeting Composition

*For any* non-empty, non-whitespace-only `name` string and any valid `greetingMessage` string, `composeGreeting(greetingMessage, name)` SHALL return exactly `"${greetingMessage}, ${name}!"`.

**Validates: Requirements 2.2**

---

### Property 5: User Name Storage Round-Trip

*For any* valid user name string (1–50 non-whitespace characters), saving it via `Storage.save(KEYS.USER_NAME, name)` and then loading it via `Storage.load(KEYS.USER_NAME)` SHALL return a value deeply equal to the original name.

**Validates: Requirements 2.3, 9.2**

---

### Property 6: Clearing Name Removes Personalization

*For any* previously-set non-empty user name, calling `Greeting.saveName("")` or `Greeting.saveName(s)` where `s` is whitespace-only SHALL result in `State.userName === ""` and `Storage.load(KEYS.USER_NAME)` returning `null` or the default empty value.

**Validates: Requirements 2.5**

---

### Property 7: Timer Countdown Decrement Invariant

*For any* valid `durationMinutes` in [1, 120] and *for any* integer `k` where `0 < k < durationMinutes * 60`, after starting the timer at full duration and calling `Timer.tick()` exactly `k` times, `State.timer.secondsLeft` SHALL equal `durationMinutes * 60 − k`.

**Validates: Requirements 3.2**

---

### Property 8: Timer Reset Restores Duration

*For any* valid `durationMinutes` in [1, 120] and after any number of `Timer.tick()` calls (including zero), calling `Timer.reset()` SHALL set `State.timer.secondsLeft` to `durationMinutes * 60` and `State.timer.status` to `"IDLE"`.

**Validates: Requirements 3.4**

---

### Property 9: Timer Button-State Invariant

*For any* timer state:
- WHEN `State.timer.status === "RUNNING"`: the Start button SHALL be disabled, the Stop button SHALL be enabled, and the duration input SHALL be disabled.
- WHEN `State.timer.status` is `"IDLE"`, `"PAUSED"`, or `"COMPLETED"`: the Stop button SHALL be disabled and the Start button SHALL be enabled (except COMPLETED where Start may also be disabled by design).

**Validates: Requirements 3.6, 3.7, 3.8, 4.5**

---

### Property 10: Pomodoro Duration Update

*For any* integer `n` in [1, 120], calling `Timer.saveDuration(n)` SHALL set `State.timer.durationMinutes` to `n` and `State.timer.secondsLeft` to `n * 60`.

**Validates: Requirements 4.2**

---

### Property 11: Pomodoro Duration Storage Round-Trip

*For any* integer `n` in [1, 120], `Storage.save(KEYS.POMO_DURATION, n)` followed by `Storage.load(KEYS.POMO_DURATION)` SHALL return `n`.

**Validates: Requirements 4.3, 9.2**

---

### Property 12: Task Addition Grows Task List

*For any* non-empty, non-whitespace-only task text `t` with length ≤ 200 and *for any* current `State.tasks` of length `n`, calling `TodoList.addTask(t)` SHALL result in `State.tasks` having length `n + 1` and the last task having `text === t` and `done === false`.

**Validates: Requirements 5.2**

---

### Property 13: Whitespace Task Rejection

*For any* string `s` that is empty or composed entirely of whitespace characters, calling `TodoList.addTask(s)` SHALL NOT modify `State.tasks` (length and contents are unchanged).

**Validates: Requirements 5.11**

---

### Property 14: Task Done Toggle Round-Trip

*For any* task in `State.tasks`, calling `TodoList.toggleTask(id)` twice in succession SHALL leave `task.done` equal to its original value (idempotence of double-toggle).

**Validates: Requirements 5.3**

---

### Property 15: Task Deletion Removes Exactly One Task

*For any* non-empty `State.tasks` of length `n` and *for any* valid task `id` in the list, calling `TodoList.deleteTask(id)` SHALL result in `State.tasks` having length `n − 1` with no task whose `id` equals the deleted id.

**Validates: Requirements 5.4**

---

### Property 16: Task List Storage Round-Trip

*For any* sequence of `addTask` / `deleteTask` / `toggleTask` operations, the value obtained by `Storage.load(KEYS.TASKS)` SHALL be deeply equal to `State.tasks` serialized without the runtime-only `editing` field.

**Validates: Requirements 5.9, 9.2**

---

### Property 17: Link Validation Correctness

*For any* string `url`:
- If `url` begins with `http://` or `https://`, has a non-empty host, and has length ≤ 2048, `QuickLinks.validateURL(url)` SHALL return `{ ok: true }`.
- Otherwise (non-http/https scheme, empty host, or length > 2048), it SHALL return `{ ok: false }`.

**Validates: Requirements 7.2, 7.7**

---

### Property 18: Link Addition Grows Link Collection

*For any* valid `name` (1–100 chars) and valid `url`, calling `QuickLinks.addLink(name, url)` SHALL result in `State.links.length` increasing by exactly 1 and the new link having `link.name === name` and `link.url === url`.

**Validates: Requirements 7.2**

---

### Property 19: Link Deletion Removes Exactly One Link

*For any* non-empty `State.links` of length `n` and *for any* valid link `id`, calling `QuickLinks.deleteLink(id)` SHALL result in `State.links` having length `n − 1` with no link whose `id` equals the deleted id.

**Validates: Requirements 7.4**

---

### Property 20: Link Collection Storage Round-Trip

*For any* `State.links`, `Storage.load(KEYS.LINKS)` after any mutation SHALL return a value deeply equal to `State.links`.

**Validates: Requirements 7.5, 9.2**

---

### Property 21: Theme Storage Round-Trip

*For any* `theme` ∈ `{ "light", "dark" }`, calling `Theme.apply(theme)` (which persists) followed by `Storage.load(KEYS.THEME)` SHALL return `theme`.

**Validates: Requirements 8.3**

---

### Property 22: Corrupted Storage Graceful Fallback

*For any* string `s` that is not valid JSON, calling `Storage.load(key, defaultValue)` after writing raw string `s` to `localStorage[key]` SHALL return `defaultValue` without throwing an exception.

**Validates: Requirements 9.5**

---

## Error Handling

### Input Validation

All validation is synchronous and happens before any state mutation:

- **User name**: `validateName(value)` — trims whitespace, checks length ≤ 50. Returns `{ ok, error }`.
- **Task text**: `validateTaskText(value)` — trims, checks length 1–200. Returns `{ ok, error }`.
- **Pomodoro duration**: `validateDuration(value)` — parses as integer, checks 1 ≤ n ≤ 120. Returns `{ ok, error }`.
- **Link name**: `validateLinkName(value)` — trims, checks length 1–100. Returns `{ ok, error }`.
- **Link URL**: `validateURL(value)` — uses `new URL(value)` in a try/catch, checks scheme is `http:` or `https:` and hostname is non-empty, checks length ≤ 2048. Returns `{ ok, error }`.

Inline error messages are written to the corresponding `<span class="error-hint">` element and cleared on the next successful submission.

### localStorage Errors

All `localStorage` reads and writes are wrapped:

```js
const Storage = {
  save(key, value) {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch (e) {
      // Silently fail; in-memory state is still authoritative
    }
  },
  load(key, defaultValue = null) {
    try {
      const raw = localStorage.getItem(key);
      if (raw === null) return defaultValue;
      return JSON.parse(raw);
    } catch (e) {
      // Corrupted entry — delete it to prevent repeated failures
      try { localStorage.removeItem(key); } catch (_) {}
      return defaultValue;
    }
  }
};
```

When `localStorage` is unavailable (e.g., private browsing with storage blocked) the `save` calls silently no-op and the app continues functioning with in-memory state for the session.

### Timer Edge Cases

- Calling `Timer.start()` while already `RUNNING` is a no-op (guarded by button disabled state + status check).
- `Timer.tick()` checks `secondsLeft <= 0` before decrementing to prevent going negative.
- `clearInterval` is always called before setting a new interval to prevent duplicate intervals.

---

## Testing Strategy

### Dual Testing Approach

The codebase uses both **unit/example tests** and **property-based tests** (PBT) as complementary layers:

- **Unit / example tests** cover specific scenarios, DOM state after interactions, and edge cases.
- **Property-based tests** verify universal correctness properties across a wide input space (minimum 100 iterations each).

### Property-Based Testing Library

Use **[fast-check](https://github.com/dubzzz/fast-check)** for JavaScript property-based testing. Tests are written as standalone `*.test.js` files (or a single `app.test.js`) and run with a test runner like **Vitest** (zero-config, ESM-compatible) or **Jest**.

Because the application is a single JS file, pure logic functions (`formatTime`, `formatDate`, `getGreeting`, `composeGreeting`, `validateURL`, `validateName`, `validateTaskText`, `validateDuration`, `Storage`, `TodoList`, `Timer`, `QuickLinks`) must be **exported** from `app.js` (or extracted to a testable module) for the test suite to import them.

### Property Test Configuration

Each property test runs a **minimum of 100 iterations**. Each test is tagged with a comment referencing the design property:

```js
// Feature: todo-life-dashboard, Property 3: Greeting Message by Hour
it("getGreeting returns correct message for any hour 0-23", () => {
  fc.assert(
    fc.property(fc.integer({ min: 0, max: 23 }), (hour) => {
      const result = getGreeting(hour);
      if (hour >= 5  && hour < 12) expect(result).toBe("Good Morning");
      else if (hour >= 12 && hour < 18) expect(result).toBe("Good Afternoon");
      else if (hour >= 18 && hour < 21) expect(result).toBe("Good Evening");
      else                              expect(result).toBe("Good Night");
    }),
    { numRuns: 100 }
  );
});
```

### Test Coverage by Category

| Property | Type | fast-check Arbitrary |
|---|---|---|
| P1 Time Formatting | PBT | `fc.date()` |
| P2 Date Formatting | PBT | `fc.date()` |
| P3 Greeting by Hour | PBT | `fc.integer({min:0, max:23})` |
| P4 Greeting Composition | PBT | `fc.string()` × `fc.string()` |
| P5 Name Storage RT | PBT | `fc.string({minLength:1,maxLength:50})` |
| P6 Name Clear | PBT | `fc.string()` (whitespace) |
| P7 Timer Decrement | PBT | `fc.integer({min:1,max:120})` × `fc.nat()` |
| P8 Timer Reset | PBT | `fc.integer({min:1,max:120})` × `fc.nat()` |
| P9 Button State | PBT | Timer status enum |
| P10 Duration Update | PBT | `fc.integer({min:1,max:120})` |
| P11 Duration Storage RT | PBT | `fc.integer({min:1,max:120})` |
| P12 Task Addition | PBT | `fc.string({minLength:1,maxLength:200})` |
| P13 Whitespace Task | PBT | `fc.stringOf(fc.constantFrom(' ','\t','\n'))` |
| P14 Toggle Round-Trip | PBT | Task object arbitrary |
| P15 Task Delete | PBT | Task list arbitrary |
| P16 Task Storage RT | PBT | Sequence of operations |
| P17 URL Validation | PBT | Valid + invalid URL arbitraries |
| P18 Link Addition | PBT | `fc.webUrl()` × `fc.string()` |
| P19 Link Delete | PBT | Link list arbitrary |
| P20 Link Storage RT | PBT | Link list arbitrary |
| P21 Theme Storage RT | PBT | `fc.constantFrom("light","dark")` |
| P22 Corrupted Storage | PBT | `fc.string()` (invalid JSON) |

### Unit / Example Tests

- Timer start → stop → secondsLeft unchanged
- Timer reaches 00:00 → completion banner visible, status = COMPLETED
- Name input > 50 chars → rejected (edge case)
- Duration < 1 or > 120 → rejected (edge case)
- Edit task: confirm with empty text → rejected
- Edit task: cancel → text unchanged
- Link with non-http URL → rejected
- On load with saved theme = "dark" → `document.documentElement.dataset.theme === "dark"`
- On load with corrupted task JSON → tasks = [], no JS error thrown
