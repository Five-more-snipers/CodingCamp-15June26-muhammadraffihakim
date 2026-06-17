# Implementation Plan: To-Do Life Dashboard

## Overview

Implement a single-page vanilla JS dashboard (one HTML file, one CSS file, one JS file) covering a live Greeting widget, Focus Timer with configurable Pomodoro, To-Do List, Quick Links panel, light/dark theme toggle, and full localStorage persistence. A Vitest + fast-check property-based test suite validates all 22 correctness properties from the design.

---

## Tasks

- [x] 1. Scaffold HTML structure and CSS foundation
  - [x] 1.1 Fix `index.html` to match the design's semantic structure
    - Replace duplicate/invalid IDs and placeholder markup
    - Add `<header id="greeting-widget">` with `#time-display`, `#date-display`, `#greeting-display`, `#name-input`, `#btn-save-name`, `#name-error`, and `#btn-theme-toggle`
    - Restructure `<section id="timer-section">` with `#timer-display`, `#timer-completion`, `#btn-start`, `#btn-stop` (disabled), `#btn-reset`, `#duration-input`, `#btn-save-duration`, `#duration-error`
    - Restructure `<section id="todo-section">` with `#task-input`, `#btn-add-task`, `#task-error`, `#task-list`
    - Restructure `<section id="links-section">` with `#link-name-input`, `#link-name-error`, `#link-url-input`, `#link-url-error`, `#btn-add-link`, `#links-list`
    - Wrap all sections in `<main class="dashboard-grid">`
    - Add a `<script>` snippet before `</head>` for FOUC prevention (reads `tld_theme` and sets `data-theme` synchronously)
    - _Requirements: 1.1, 1.2, 3.1, 4.1, 5.1, 7.1, 8.4, 10.4_

  - [x] 1.2 Set up `css/style.css` — CSS reset, custom properties, and layout
    - Add box-sizing reset and remove default margins/padding
    - Define `:root` CSS custom properties for light theme: `--color-bg`, `--color-surface`, `--color-text`, `--color-primary`, `--color-danger`, `--color-border`, `--color-muted`, `--transition-theme: 300ms ease`
    - Define `[data-theme="dark"]` overrides for all custom properties
    - Add universal `transition` rule on `*` for `background-color`, `color`, `border-color` using `var(--transition-theme)`
    - Implement `.dashboard-grid` with `display: grid; grid-template-columns: repeat(auto-fit, minmax(320px, 1fr)); gap: 1.5rem`
    - Style `.widget` cards with `background: var(--color-surface)`, `border`, `border-radius`, `padding`
    - Style header layout, `.error-hint`, `.hidden` utility class, `.timer-display`, `.task-item`, `.link-item`, `.task-done` (strikethrough + opacity), `.timer-completion`
    - _Requirements: 8.1, 8.2, 8.4, 10.1, 10.4_

- [x] 2. Implement CONSTANTS, Storage module, and State object in `js/app.js`
  - [x] 2.1 Write the IIFE skeleton, CONSTANTS block, and Storage namespace
    - Open an IIFE: `(function() { ... })();`
    - Define `const KEYS = { USER_NAME: "tld_userName", TASKS: "tld_tasks", LINKS: "tld_links", POMO_DURATION: "tld_pomoDuration", THEME: "tld_theme" }`
    - Define validation limit constants: `NAME_MAX = 50`, `TASK_MAX = 200`, `LINK_NAME_MAX = 100`, `LINK_URL_MAX = 2048`, `DURATION_MIN = 1`, `DURATION_MAX = 120`
    - Implement `Storage.save(key, value)` — `JSON.stringify` inside `try/catch`, silently fail
    - Implement `Storage.load(key, defaultValue)` — `JSON.parse` inside `try/catch`; on failure delete the key and return `defaultValue`
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5_
  - [x] 2.2 Write property test for Storage round-trip and corrupted-entry fallback (P5, P11, P16, P20, P21, P22)
    - **Property 5: User Name Storage Round-Trip** — `fc.string({minLength:1,maxLength:50})` — Validates: Requirements 2.3, 9.2
    - **Property 11: Pomodoro Duration Storage Round-Trip** — `fc.integer({min:1,max:120})` — Validates: Requirements 4.3, 9.2
    - **Property 22: Corrupted Storage Graceful Fallback** — `fc.string()` (invalid JSON) — Validates: Requirements 9.5
    - _Requirements: 9.1–9.5_

  - [x] 2.3 Define the in-memory State object
    - Declare `const State` with shape: `{ userName: "", tasks: [], links: [], timer: { durationMinutes: 25, secondsLeft: 1500, running: false, status: "IDLE", intervalId: null }, theme: "light" }`
    - _Requirements: 9.1_

- [ ] 3. Implement the Greeting widget
  - [x] 3.1 Write pure helper functions `formatTime(date)`, `formatDate(date)`, `getGreeting(hour)`, `composeGreeting(msg, name)`
    - `formatTime` — returns `"HH:MM:SS"` zero-padded
    - `formatDate` — returns `"Weekday, DD Month YYYY"` using day/month name arrays
    - `getGreeting(hour)` — returns one of the four greeting strings based on hour ranges from the requirements
    - `composeGreeting(msg, name)` — returns `"${msg}, ${name}!"` when name is non-empty, else `msg`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 2.2_
  - [x] 3.2 Write property tests for time/date formatting and greeting logic (P1, P2, P3, P4)
    - **Property 1: Time Formatting Correctness** — `fc.date()` — Validates: Requirements 1.1
    - **Property 2: Date Formatting Correctness** — `fc.date()` — Validates: Requirements 1.2
    - **Property 3: Greeting Message by Hour** — `fc.integer({min:0,max:23})` — Validates: Requirements 1.3–1.6
    - **Property 4: Personalized Greeting Composition** — `fc.string()` × `fc.string()` — Validates: Requirements 2.2
    - _Requirements: 1.1–1.7, 2.2_

  - [x] 3.3 Implement the `Greeting` namespace
    - `Greeting.init()` — reads `KEYS.USER_NAME` from Storage, sets `State.userName`, starts `setInterval(Greeting.tick, 1000)`, calls `Greeting.renderGreeting()`
    - `Greeting.tick()` — builds a `new Date()`, updates `#time-display`, `#date-display`, `#greeting-display` via `formatTime`, `formatDate`, `getGreeting`, and `composeGreeting`
    - `Greeting.saveName(value)` — trims input; if empty/whitespace calls `Greeting.clearName()`; if > 50 chars shows `#name-error` and returns; otherwise sets `State.userName`, calls `Storage.save`, clears `#name-error`, calls `Greeting.renderGreeting()`
    - `Greeting.clearName()` — sets `State.userName = ""`, calls `Storage.save(KEYS.USER_NAME, null)`, calls `Greeting.renderGreeting()`
    - `Greeting.renderGreeting()` — composes greeting string and writes to `#greeting-display`
    - Bind `#btn-save-name` click and `#name-input` Enter keydown in `Bootstrap.init()` (wired in task 8)
    - _Requirements: 1.1–1.7, 2.1–2.6_
  - [-] 3.4 Write unit tests for Greeting name validation edge cases
    - Test: name > 50 chars → rejected, `#name-error` shown
    - Test: whitespace-only name → clears stored name, non-personalized greeting shown
    - Test: valid name saved → greeting reads `"[Greeting], [Name]!"`
    - _Requirements: 2.1, 2.5, 2.6_

- [ ] 4. Implement the Theme module
  - [x] 4.1 Implement the `Theme` namespace and FOUC-prevention script
    - `Theme.apply(theme)` — sets `document.documentElement.dataset.theme = theme`, updates `#btn-theme-toggle` icon (🌙 for light → dark, ☀️ for dark → light) and `aria-label`
    - `Theme.persist(theme)` — calls `Storage.save(KEYS.THEME, theme)`
    - `Theme.toggle()` — flips `State.theme` between `"light"` and `"dark"`, calls `Theme.persist` then `Theme.apply`
    - `Theme.init()` — reads `Storage.load(KEYS.THEME)`; falls back to `window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light"`; sets `State.theme`; calls `Theme.apply`
    - Add inline `<script>` block immediately after `<html>` in `index.html` that reads `tld_theme` from `localStorage` and sets `data-theme` before any CSS renders (FOUC guard)
    - _Requirements: 8.1–8.6_
  - [x] 4.2 Write property test for Theme storage round-trip (P21)
    - **Property 21: Theme Storage Round-Trip** — `fc.constantFrom("light","dark")` — Validates: Requirements 8.3
    - _Requirements: 8.3_
  - [-] 4.3 Write unit tests for Theme fallback behaviour
    - Test: no saved theme + `prefers-color-scheme: dark` → applies dark
    - Test: saved theme `"dark"` on load → `document.documentElement.dataset.theme === "dark"`
    - _Requirements: 8.4, 8.5, 8.6_

- [ ] 5. Implement the Focus Timer widget
  - [x] 5.1 Write `Timer.formatTime(seconds)` and the Timer state helpers
    - `Timer.formatTime(seconds)` — returns `"MM:SS"` zero-padded string
    - Write `Timer.render()` — updates `#timer-display` text, toggles `disabled` on `#btn-start`, `#btn-stop`, `#duration-input`, toggles `.hidden` on `#timer-completion`; implements the full button-state machine per requirements 3.6–3.8 and 4.5
    - _Requirements: 3.1, 3.6, 3.7, 3.8, 4.5_
  - [x] 5.2 Implement `Timer.start()`, `Timer.stop()`, `Timer.tick()`, `Timer.complete()`, `Timer.reset()`
    - `Timer.start()` — guard: no-op if `State.timer.status === "RUNNING"`; clears any existing interval; sets `running=true`, `status="RUNNING"`; creates new `setInterval(Timer.tick, 1000)`; calls `Timer.render()`
    - `Timer.tick()` — if `secondsLeft <= 0` calls `Timer.complete()` and returns; decrements `secondsLeft`; calls `Timer.render()`
    - `Timer.stop()` — `clearInterval`, sets `running=false`, `status="PAUSED"`; calls `Timer.render()`
    - `Timer.reset()` — `clearInterval`, restores `secondsLeft = durationMinutes * 60`, sets `status="IDLE"`, `running=false`; clears completion banner; calls `Timer.render()`
    - `Timer.complete()` — `clearInterval`, sets `status="COMPLETED"`, `running=false`; calls `Timer.render()`
    - _Requirements: 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8_
  - [-] 5.3 Write property tests for Timer countdown invariants (P7, P8, P9)
    - **Property 7: Timer Countdown Decrement Invariant** — `fc.integer({min:1,max:120})` × `fc.nat()` — Validates: Requirements 3.2
    - **Property 8: Timer Reset Restores Duration** — `fc.integer({min:1,max:120})` × `fc.nat()` — Validates: Requirements 3.4
    - **Property 9: Timer Button-State Invariant** — Timer status enum — Validates: Requirements 3.6, 3.7, 3.8, 4.5
    - _Requirements: 3.2, 3.4, 3.6, 3.7, 3.8, 4.5_

  - [~] 5.4 Write unit tests for Timer edge cases
    - Test: start → stop → `secondsLeft` unchanged
    - Test: tick to 00:00 → `status === "COMPLETED"`, completion banner visible
    - Test: calling `start()` while already RUNNING is a no-op (no duplicate intervals)
    - _Requirements: 3.2, 3.5, 3.6_

  - [-] 5.5 Implement `Timer.saveDuration(value)` and `Timer.init()`
    - `validateDuration(value)` — parses as integer, checks `1 ≤ n ≤ 120`; returns `{ ok, error }`
    - `Timer.saveDuration(value)` — validates; on failure writes to `#duration-error` and returns; on success sets `State.timer.durationMinutes = n`, persists via `Storage.save(KEYS.POMO_DURATION, n)`, clears error, calls `Timer.reset()`
    - `Timer.init()` — loads `KEYS.POMO_DURATION` from Storage (default 25), sets `State.timer.durationMinutes` and `secondsLeft`, calls `Timer.render()`
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_
  - [~] 5.6 Write property tests for Pomodoro duration update (P10, P11)
    - **Property 10: Pomodoro Duration Update** — `fc.integer({min:1,max:120})` — Validates: Requirements 4.2
    - **Property 11: Pomodoro Duration Storage Round-Trip** — `fc.integer({min:1,max:120})` — Validates: Requirements 4.3, 9.2
    - _Requirements: 4.2, 4.3_

- [~] 6. Checkpoint — Ensure Greeting, Theme, and Timer tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Implement the To-Do List widget
  - [x] 7.1 Write To-Do validation helper and task object factory
    - `validateTaskText(value)` — trims, checks length 1–200; returns `{ ok, error }`
    - `createTask(text)` — returns `{ id: crypto.randomUUID(), text: text.trim(), done: false, editing: false, createdAt: Date.now() }`
    - _Requirements: 5.2, 5.11_
  - [ ] 7.2 Write property tests for task validation and addition (P12, P13)
    - **Property 12: Task Addition Grows Task List** — `fc.string({minLength:1,maxLength:200})` filtered to non-whitespace — Validates: Requirements 5.2
    - **Property 13: Whitespace Task Rejection** — `fc.stringOf(fc.constantFrom(' ','\t','\n'))` — Validates: Requirements 5.11
    - _Requirements: 5.2, 5.11_

  - [~] 7.3 Implement `TodoList.addTask`, `TodoList.deleteTask`, `TodoList.toggleTask`, `TodoList.render`, `TodoList.renderItem`
    - `TodoList.addTask(text)` — validates; on failure shows `#task-error`, returns; on success creates task, pushes to `State.tasks`, persists (strip `editing` field), clears error, calls `TodoList.render()`
    - `TodoList.deleteTask(id)` — filters `State.tasks`, persists, renders
    - `TodoList.toggleTask(id)` — flips `task.done`, persists, renders
    - `TodoList.renderItem(task)` — returns `<li>` element with checkbox, text span (`.task-done` class when `done`), edit and delete buttons; when `task.editing === true` renders inline edit `<input>`, save/cancel buttons, and error hint
    - `TodoList.render()` — rebuilds `#task-list` innerHTML from `State.tasks` via `renderItem`
    - _Requirements: 5.2, 5.3, 5.4, 5.9, 5.10, 5.11_
  - [~] 7.4 Write property tests for task toggle and deletion (P14, P15, P16)
    - **Property 14: Task Done Toggle Round-Trip** — task object arbitrary — Validates: Requirements 5.3
    - **Property 15: Task Deletion Removes Exactly One Task** — task list arbitrary — Validates: Requirements 5.4
    - **Property 16: Task List Storage Round-Trip** — sequence of operations — Validates: Requirements 5.9, 9.2
    - _Requirements: 5.3, 5.4, 5.9_

  - [~] 7.5 Implement `TodoList.startEdit`, `TodoList.confirmEdit`, `TodoList.cancelEdit`, and `TodoList.init`
    - `TodoList.startEdit(id)` — sets `task.editing = true`, calls `TodoList.render()`
    - `TodoList.confirmEdit(id, text)` — validates non-empty; on failure writes `.edit-error` hint on the inline input and returns; on success sets `task.text`, `task.editing = false`, persists, renders
    - `TodoList.cancelEdit(id)` — sets `task.editing = false`, renders (original text preserved)
    - `TodoList.init()` — loads tasks from `Storage.load(KEYS.TASKS, [])`, ensures each task has `editing: false`, sets `State.tasks`, calls `TodoList.render()`
    - _Requirements: 5.5, 5.6, 5.7, 5.8, 5.10_
  - [~] 7.6 Implement `TodoList.handleClick` and `TodoList.handleKeydown` for event delegation
    - `TodoList.handleClick(e)` — `e.target.closest('[data-id]')` to get `id`; dispatch to `toggleTask`, `deleteTask`, `startEdit`, `confirmEdit`, `cancelEdit` based on target class
    - `TodoList.handleKeydown(e)` — Enter on `.task-edit-input` → `confirmEdit`; Escape → `cancelEdit`
    - _Requirements: 5.3, 5.4, 5.5, 5.6, 5.8_
  - [~] 7.7 Write unit tests for To-Do edit flow edge cases
    - Test: confirm edit with empty text → rejected, error hint shown
    - Test: cancel edit → task text unchanged, display mode restored
    - Test: on load with corrupted task JSON → `State.tasks === []`, no JS error
    - _Requirements: 5.7, 5.8, 9.5_

- [ ] 8. Implement the Quick Links widget
  - [x] 8.1 Write `QuickLinks.validateURL`, link name validator, and link object factory
    - `validateLinkName(value)` — trims, checks 1–100 chars; returns `{ ok, error }`
    - `QuickLinks.validateURL(url)` — tries `new URL(url)` in try/catch; checks `protocol` is `http:` or `https:`, `hostname` non-empty, `url.length ≤ 2048`; returns `{ ok, error }`
    - `createLink(name, url)` — returns `{ id: crypto.randomUUID(), name: name.trim(), url }`
    - _Requirements: 7.1, 7.2, 7.7_
  - [~] 8.2 Write property tests for URL validation and link CRUD (P17, P18, P19, P20)
    - **Property 17: Link Validation Correctness** — valid + invalid URL arbitraries — Validates: Requirements 7.2, 7.7
    - **Property 18: Link Addition Grows Link Collection** — `fc.webUrl()` × `fc.string()` — Validates: Requirements 7.2
    - **Property 19: Link Deletion Removes Exactly One Link** — link list arbitrary — Validates: Requirements 7.4
    - **Property 20: Link Collection Storage Round-Trip** — link list arbitrary — Validates: Requirements 7.5, 9.2
    - _Requirements: 7.2, 7.4, 7.5, 7.7_

  - [~] 8.3 Implement `QuickLinks.addLink`, `QuickLinks.deleteLink`, `QuickLinks.render`, `QuickLinks.renderItem`, and `QuickLinks.init`
    - `QuickLinks.addLink(name, url)` — validates name and URL separately; on failure writes to `#link-name-error` or `#link-url-error` respectively and returns; on success creates link, pushes to `State.links`, persists, clears errors, renders
    - `QuickLinks.deleteLink(id)` — filters `State.links`, persists, renders
    - `QuickLinks.renderItem(link)` — returns a `.link-item` div with `<a href target="_blank" rel="noopener noreferrer">` and a delete button
    - `QuickLinks.render()` — rebuilds `#links-list` innerHTML; renders empty state message if no links
    - `QuickLinks.init()` — loads from `Storage.load(KEYS.LINKS, [])`, sets `State.links`, renders
    - `QuickLinks.handleClick(e)` — delegates delete to `deleteLink` by `data-id`
    - _Requirements: 7.1–7.7_
  - [~] 8.4 Write unit tests for Quick Links edge cases
    - Test: non-http URL → rejected, `#link-url-error` shown
    - Test: name > 100 chars → rejected
    - Test: link button click → opens `target="_blank"` (check `rel="noopener noreferrer"`)
    - _Requirements: 7.2, 7.3, 7.7_

- [~] 9. Checkpoint — Ensure To-Do and Quick Links tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Bootstrap: wire everything together in `DOMContentLoaded`
  - [~] 10.1 Implement the `Bootstrap.init` function and `DOMContentLoaded` handler
    - Create `Bootstrap.init()` that calls in order: `Theme.init()`, `Greeting.init()`, `Timer.init()`, `TodoList.init()`, `QuickLinks.init()`
    - Attach all direct event bindings (see design Binding Table):
      - `#btn-save-name` click + `#name-input` Enter → `Greeting.saveName`
      - `#btn-theme-toggle` click → `Theme.toggle`
      - `#btn-start` click → `Timer.start`
      - `#btn-stop` click → `Timer.stop`
      - `#btn-reset` click → `Timer.reset`
      - `#btn-save-duration` click → `Timer.saveDuration` (reads `#duration-input` value)
      - `#btn-add-task` click + `#task-input` Enter → `TodoList.addTask` (reads `#task-input` value)
      - `#task-list` click → `TodoList.handleClick` (delegated)
      - `#task-list` keydown → `TodoList.handleKeydown` (delegated)
      - `#btn-add-link` click → `QuickLinks.addLink` (reads both input values)
      - `#links-list` click → `QuickLinks.handleClick` (delegated)
    - Register `document.addEventListener("DOMContentLoaded", Bootstrap.init)`
    - _Requirements: 1.1, 3.1, 5.1, 7.1, 8.1, 10.1, 10.3_
  - [~] 10.2 Write property tests for Greeting name storage round-trip (P5, P6)
    - **Property 5: User Name Storage Round-Trip** — `fc.string({minLength:1,maxLength:50})` — Validates: Requirements 2.3, 9.2
    - **Property 6: Clearing Name Removes Personalization** — `fc.string()` (whitespace) — Validates: Requirements 2.5
    - _Requirements: 2.3, 2.5_

- [ ] 11. Set up Vitest + fast-check test infrastructure
  - [x] 11.1 Initialise `package.json` and install Vitest + fast-check
    - Run `npm init -y` if `package.json` does not exist
    - Install dev dependencies: `vitest@^2.0.0`, `fast-check@^3.0.0`, `@vitest/coverage-v8@^2.0.0` (exact minor versions pinned)
    - Add scripts to `package.json`: `"test": "vitest --run"`, `"test:watch": "vitest"`, `"test:coverage": "vitest --run --coverage"`
    - Create `vitest.config.js` with `environment: "jsdom"` so DOM APIs are available
    - _Requirements: 10.1_
  - [~] 11.2 Export pure functions from `js/app.js` for testing and create `js/app.test.js`
    - Add a guarded export block at the bottom of the IIFE: `if (typeof module !== "undefined") { module.exports = { formatTime, formatDate, getGreeting, composeGreeting, validateTaskText, validateDuration, validateLinkName, Storage, /* etc. */ }; }`
    - Alternatively refactor to use `globalThis.__tld_exports` for test access under jsdom
    - Create `js/app.test.js` with the full suite of 22 property-based tests plus the named unit tests from the design's "Unit / Example Tests" section
    - Annotate every test with a comment: `// Property N: <title>` matching design document numbering
    - Verify all 22 properties and unit tests are represented
    - _Requirements: 10.1_
  - [~] 11.3 Write remaining property tests not yet covered in earlier tasks (P5, P6 if not done in 10.2)
    - Confirm properties P1–P22 are all represented in `js/app.test.js`
    - _Requirements: 9.1–9.5_

- [~] 12. Final checkpoint — full test suite green
  - Ensure all tests pass, ask the user if questions arise.


---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP; the `*` tasks are never auto-implemented by a coding agent.
- Each task references specific requirements from `requirements.md` for full traceability.
- Checkpoints (tasks 6, 9, 12) are gates — do not proceed to the next group until all tests in the previous group pass.
- Property-based tests require pure, exported functions; the export guard at the bottom of the IIFE (task 11.2) is the mechanism that makes app.js testable without a bundler.
- The FOUC-prevention `<script>` block (task 1.1 / 4.1) must run synchronously before any CSS is parsed, so it is placed immediately after `<html>` or at the very top of `<head>` — not deferred.
- All `localStorage` access goes through `Storage.save` / `Storage.load` — never directly — to ensure graceful degradation (Requirement 9.4).
- Timer `clearInterval` is always called before `setInterval` to prevent duplicate intervals regardless of the state machine path.

---

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2"] },
    { "id": 1, "tasks": ["2.1", "11.1"] },
    { "id": 2, "tasks": ["2.2", "2.3"] },
    { "id": 3, "tasks": ["3.1", "4.1", "5.1"] },
    { "id": 4, "tasks": ["3.2", "3.3", "4.2", "5.2", "7.1", "8.1"] },
    { "id": 5, "tasks": ["3.4", "4.3", "5.3", "5.5", "7.2", "8.2", "7.3"] },
    { "id": 6, "tasks": ["5.4", "5.6", "7.4", "7.5", "8.3"] },
    { "id": 7, "tasks": ["7.6", "7.7", "8.4"] },
    { "id": 8, "tasks": ["10.1"] },
    { "id": 9, "tasks": ["10.2", "11.2"] },
    { "id": 10, "tasks": ["11.3"] }
  ]
}
```
