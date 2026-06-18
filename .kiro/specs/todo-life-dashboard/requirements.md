# Requirements Document

## Introduction

The **To-Do Life Dashboard** is a client-side web application that helps users organize their day in a single, clean interface. It is built with HTML, CSS, and Vanilla JavaScript only — no frameworks, no backend. All data persists via the browser's Local Storage API. The dashboard brings together four core widgets: a live Greeting section, a Focus Timer (Pomodoro), a To-Do List, and a Quick Links panel. Three optional challenges are also included: a custom user name in the greeting, a light/dark theme toggle, and a configurable Pomodoro duration.

---

## Glossary

- **Dashboard**: The single-page web application described by these requirements.
- **Widget**: A self-contained UI section within the Dashboard (Greeting, Timer, To-Do, Quick Links).
- **Timer**: The Focus Timer widget that counts down from a configurable duration.
- **Task**: A single to-do item entered by the user.
- **Task_List**: The collection of all Tasks managed by the To-Do widget.
- **Link**: A Quick Link entry consisting of a display name and a URL.
- **Link_Collection**: The set of all Links saved by the user.
- **Local_Storage**: The browser's `localStorage` API used for client-side data persistence.
- **Theme**: The visual color scheme of the Dashboard, either "light" or "dark".
- **User_Name**: An optional display name entered by the user for the greeting.
- **Pomodoro_Duration**: The configurable countdown length (in minutes) for the Timer, defaulting to 25 minutes.
- **Greeting_Message**: The time-of-day salutation shown in the header (e.g., "Good Morning").

---

## Requirements

---

### Requirement 1: Live Greeting Header

**User Story:** As a user, I want to see the current time, date, and a time-appropriate greeting when I open the Dashboard, so that I am immediately oriented to the current moment.

#### Acceptance Criteria

1. THE Dashboard SHALL display the current time in HH:MM:SS format, updated every second.
2. THE Dashboard SHALL display the current date in "Day, DD Month YYYY" format (e.g., "Monday, 14 July 2025").
3. IF the local time is at or after 05:00 and before 12:00, THEN THE Dashboard SHALL set the Greeting_Message to "Good Morning".
4. IF the local time is at or after 12:00 and before 18:00, THEN THE Dashboard SHALL set the Greeting_Message to "Good Afternoon".
5. IF the local time is at or after 18:00 and before 21:00, THEN THE Dashboard SHALL set the Greeting_Message to "Good Evening".
6. IF the local time is at or after 21:00 or before 05:00, THEN THE Dashboard SHALL set the Greeting_Message to "Good Night".
7. THE Dashboard SHALL render the active Greeting_Message visibly in the header area of the page.

---

### Requirement 2: Custom Name in Greeting (Challenge)

**User Story:** As a user, I want to enter my name so that the greeting is personalized to me (e.g., "Good Morning, Alex!").

#### Acceptance Criteria

1. THE Dashboard SHALL provide a text input field (maximum 50 characters) and a Save button where the user can enter a User_Name.
2. WHEN the user submits a non-empty, non-whitespace-only User_Name via the Save button or pressing Enter, THE Dashboard SHALL display the greeting in the format "[Greeting_Message], [User_Name]!" (e.g., "Good Morning, Alex!").
3. THE Dashboard SHALL persist the User_Name in Local_Storage so that it is restored on page reload.
4. WHEN Local_Storage contains a saved User_Name, THE Dashboard SHALL display the personalized greeting immediately on page load without requiring the user to re-enter their name.
5. WHEN the user clears the User_Name field to empty or whitespace-only and saves, THE Dashboard SHALL revert to the non-personalized Greeting_Message and remove the stored User_Name from Local_Storage.
6. IF the user enters more than 50 characters in the User_Name field, THEN THE Dashboard SHALL reject the submission and display an inline validation message indicating the 50-character limit.

---

### Requirement 3: Focus Timer

**User Story:** As a user, I want a countdown timer I can start, stop, and reset, so that I can use the Pomodoro technique to manage my focus sessions.

#### Acceptance Criteria

1. THE Timer SHALL initialize with the Pomodoro_Duration (defaulting to 25:00) on page load.
2. WHEN the user activates the Start button, THE Timer SHALL begin counting down one second at a time.
3. WHEN the Timer is counting down and the user activates the Stop button, THE Timer SHALL pause at the current remaining time.
4. WHEN the user activates the Reset button, THE Timer SHALL stop counting, restore the display to the Pomodoro_Duration, and re-enable the Start button.
5. WHEN the Timer reaches 00:00, THE Timer SHALL stop automatically and display a visible on-page completion message (e.g., a banner or highlighted text) notifying the user that the session is complete.
6. WHILE the Timer is counting down, THE Dashboard SHALL disable the Start button to prevent duplicate intervals.
7. WHILE the Timer is counting down, THE Dashboard SHALL enable the Stop button.
8. WHEN the Timer has completed or been reset and is not counting down, THE Dashboard SHALL disable the Stop button.

---

### Requirement 4: Configurable Pomodoro Duration (Challenge)

**User Story:** As a user, I want to set a custom focus duration, so that I can adapt the timer to sessions shorter or longer than 25 minutes.

#### Acceptance Criteria

1. THE Dashboard SHALL provide a numeric input field (defaulting to 25) that allows the user to set the Pomodoro_Duration in whole minutes.
2. WHEN the user sets a Pomodoro_Duration and clicks the Save button, THE Timer SHALL update its display to reflect the new duration.
3. THE Dashboard SHALL persist the Pomodoro_Duration in Local_Storage so that the custom value is restored on page reload.
4. IF the user enters a value less than 1 or greater than 120, THEN THE Dashboard SHALL reject the input and display an inline validation message indicating the valid range (1–120 minutes) without updating the Pomodoro_Duration.
5. WHILE the Timer is counting down, THE Dashboard SHALL disable the duration input field to prevent mid-session changes.

---

### Requirement 5: To-Do List

**User Story:** As a user, I want to add, complete, and delete tasks, so that I can track what needs to be done today.

#### Acceptance Criteria

1. THE Dashboard SHALL provide a text input and an Add button for creating new Tasks.
2. WHEN the user submits non-empty task text between 1 and 200 characters, THE Dashboard SHALL append the new Task to the Task_List and render it in the UI.
3. WHEN the user marks a Task as done via its checkbox, THE Dashboard SHALL apply a strikethrough style and reduce the opacity of that Task's text to indicate completion.
4. WHEN the user activates the Delete control on a Task, THE Dashboard SHALL remove that Task from the Task_List and from the UI.
5. WHEN the user activates the Edit control on a Task, THE Dashboard SHALL replace the Task's display text with an editable inline input field pre-filled with the current task text.
6. WHEN the user confirms an edit (via Enter key or a Save button) with non-empty text, THE Dashboard SHALL update the Task's text in the Task_List and revert to display mode.
7. IF the user attempts to confirm an edit with empty or whitespace-only text, THEN THE Dashboard SHALL not save the change and SHALL display a validation hint on the input.
8. WHEN the user cancels an edit (via Escape key), THE Dashboard SHALL discard the change and revert the Task to display mode with the original text.
9. THE Dashboard SHALL persist the Task_List in Local_Storage so that Tasks survive page reload.
10. WHEN the page loads, THE Dashboard SHALL restore all previously saved Tasks from Local_Storage and render them in the Task_List.
11. IF the user submits empty or whitespace-only task text, THEN THE Dashboard SHALL not add the Task and SHALL display a validation hint on the text input until non-empty text is entered.

---

### Requirement 6: Prevent Duplicate Tasks (Challenge — Not Selected)

**User Story:** As a user, I want the system to prevent me from adding duplicate tasks, so that my task list stays clean and uncluttered.

#### Acceptance Criteria

1. This challenge was not selected and is excluded from the MVP scope.

---

### Requirement 7: Quick Links

**User Story:** As a user, I want to add buttons that open my favorite websites, so that I can navigate to frequently visited pages in one click.

#### Acceptance Criteria

1. THE Dashboard SHALL provide a name input field (maximum 100 characters), a URL input field (maximum 2048 characters), and an Add button for creating new Links.
2. WHEN the user submits a valid Link (non-empty name and a URL with an `http://` or `https://` scheme and a non-empty host), THE Dashboard SHALL add the Link to the Link_Collection and render a clickable button for it.
3. WHEN the user activates a Link button, THE Dashboard SHALL open the associated URL in a new browser tab.
4. WHEN the user activates the Delete control on a Link, THE Dashboard SHALL remove that Link from the Link_Collection and remove its button from the UI.
5. THE Dashboard SHALL persist the Link_Collection in Local_Storage so that Links survive page reload.
6. WHEN the page loads, THE Dashboard SHALL restore all previously saved Links from Local_Storage and render them as clickable buttons, or render an empty panel if Local_Storage is unavailable or unreadable.
7. IF the user submits a Link with an empty name, a name exceeding 100 characters, an empty URL, a URL exceeding 2048 characters, or a URL without an `http://` or `https://` scheme and non-empty host, THEN THE Dashboard SHALL not add the Link, SHALL display a validation hint adjacent to the offending field, and SHALL clear the hint on the next successful submission.

---

### Requirement 8: Light / Dark Mode (Challenge)

**User Story:** As a user, I want to switch between a light and a dark theme, so that I can use the Dashboard comfortably in different lighting conditions.

#### Acceptance Criteria

1. THE Dashboard SHALL provide a toggle control (e.g., a button or switch) that switches the active Theme between "light" and "dark".
2. WHEN the user activates the theme toggle, THE Dashboard SHALL apply the selected Theme to all Dashboard UI components within 300 milliseconds without a page reload.
3. THE Dashboard SHALL persist the active Theme in Local_Storage so that the user's preference is restored on page reload; if Local_Storage is unavailable, the selected Theme SHALL remain active for the current session only.
4. WHEN the page loads, THE Dashboard SHALL read the saved Theme from Local_Storage and apply it before rendering content, preventing a flash of the wrong theme.
5. WHERE no Theme preference has been saved in Local_Storage, THE Dashboard SHALL check the system `prefers-color-scheme` media feature; if it is "dark" the Dashboard SHALL apply the "dark" Theme, otherwise it SHALL default to the "light" Theme.
6. IF Local_Storage is unavailable or a Theme read fails on load, THEN THE Dashboard SHALL apply the system `prefers-color-scheme`-derived default without displaying an error to the user.

---

### Requirement 9: Data Persistence and Storage

**User Story:** As a user, I want all my settings and data to be automatically saved, so that I do not lose anything when I close or refresh the browser.

#### Acceptance Criteria

1. THE Dashboard SHALL use Local_Storage as the sole persistence mechanism for all user data.
2. WHEN any user data changes (Tasks, Links, User_Name, Pomodoro_Duration, Theme), THE Dashboard SHALL write the updated data to Local_Storage synchronously before returning from the event handler.
3. THE Dashboard SHALL store each data category under a distinct Local_Storage key to avoid key collisions.
4. IF Local_Storage is unavailable or a read/write operation fails, THEN THE Dashboard SHALL continue to function in-session with data maintained in memory, and SHALL NOT display a JavaScript error to the user.
5. WHEN the page loads and Local_Storage contains data that cannot be parsed (e.g., corrupted JSON), THE Dashboard SHALL discard the corrupted entry, initialize that data category to its default empty state, and continue loading normally without displaying a JavaScript error to the user.

---

### Requirement 10: Performance and Compatibility

**User Story:** As a user, I want the Dashboard to load fast and respond instantly to my interactions, so that it does not slow me down.

#### Acceptance Criteria

1. THE Dashboard SHALL load all content using only HTML, CSS, and Vanilla JavaScript with no external frameworks or libraries, and SHALL be fully interactive within 3 seconds on a standard broadband connection (≥10 Mbps).
2. THE Dashboard SHALL meet all acceptance criteria in the latest major stable release of Chrome, Firefox, Edge, and Safari available at the time of testing.
3. WHEN the user interacts with any control (add, delete, toggle, start timer), THE Dashboard SHALL update the UI within 100ms of the interaction.
4. THE Dashboard SHALL consist of exactly one HTML file, one CSS file inside `css/`, and one JavaScript file inside `js/`.
