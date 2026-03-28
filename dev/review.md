# Code Review Report: Assessment Tool

**Date:** March 28, 2026
**Type:** Single-file HTML/JS/CSS Web Application

## Executive Summary

The Assessment Tool is a feature-rich, client-side application with complex functionalities, including state management, drag-and-drop interactions, CSV parsing, and AI integration. While the functionality is impressive for a single file, the codebase suffers from architectural bottlenecks, critical security vulnerabilities (XSS), and performance issues related to DOM manipulation.

Below is a detailed breakdown of the identified issues categorized by severity.

---

## Critical Issues (High Priority)

### 1. Cross-Site Scripting (XSS) Vulnerabilities

**Issue:** The application heavily relies on `innerHTML` for rendering the UI, directly interpolating user input and CSV data without proper sanitization.

**Example:** In `renderStudents()`, student names from the CSV are injected directly: `<span class="flex-1 text-sm text-slate-700">${s.name}</span>`. If a CSV contains a malicious payload (e.g., `<img src=x onerror=alert(1)>` in the Name column), it will be executed in the browser.

**Recommendation:** Use `textContent` when creating DOM elements dynamically, or pass all dynamic strings through the existing (but currently underutilized) `escapeHtml` function before adding them to `innerHTML`.

### 2. State Management & QuotaExceededError Risks

**Issue:** The app stores everything in `localStorage`, which typically has a strict 5MB limit.

- While the base64 PDF strings are kept in memory during chat (which is good), the state object (configs, student data, scores, detailed feedback notes) grows infinitely.
- When `QuotaExceededError` occurs, `saveState()` fails silently (it catches the error and alerts the user, but the in-memory state and the UI proceed as if saved). If the user refreshes, data is lost.

**Recommendation:** Implement an IndexedDB solution for larger storage capacity, or aggressively warn users when approaching storage limits (e.g., checking `JSON.stringify(state).length`).

### 3. DOM Thrashing & Re-rendering Performance

**Issue:** Functions like `renderAssessment()` and `renderDashboard()` rebuild the entire DOM of their respective views upon minor interactions.

**Example:** Clicking a single score button (`setScore(...)`) calls `renderAssessment()`. This destroys and recreates the entire page's HTML, including textareas. While `oninput` events on textareas avoid this, a user clicking a score while another team member is typing (if syncing was real-time) or quickly interacting will experience input loss or focus loss.

**Recommendation:** Adopt a more granular DOM update strategy (updating only the modified elements' classes) or migrate to a reactive framework like React, Vue, or Svelte.

---

## Functional & Logic Bugs (Medium Priority)

### 1. Fragile Date Parsing Logic

**Issue:** The `parseDatetimeString(str)` function uses a hardcoded Dutch regex for months: `(jan|feb|mrt|apr|mei|jun|jul|aug|sep|okt|nov|dec)`.

If a user imports a CSV with English dates, standard ISO dates (`YYYY-MM-DD`), or numerical dates (`01-05-2025`), the function fails to extract the date properly.

**Recommendation:** Broaden the date parsing logic. Use standard JS `Date` parsing capabilities or accept standard `DD-MM-YYYY` formats alongside the text-based regex.

### 2. Custom CSV Parser Limitations

**Issue:** The `parseCSV(text)` function is custom-built and may fail on edge cases standard to CSV files (e.g., handling escaped quotes `""` properly within quoted fields, or unexpected line breaks).

**Recommendation:** For robust CSV handling, integrate a lightweight, battle-tested library like PapaParse, or enforce strict formatting warnings in the UI.

### 3. AI JSON Extraction Brittleness

**Issue:** The AI parsing logic relies on a regex `` /```json\s*([\s\S]*?)```/ `` or `` /```\s*([\s\S]*?)```/ ``. If the LLM returns the JSON object purely without markdown codeblocks (which happens occasionally), the fallback `[null, text]` assumes the entire text is JSON. If there is conversational text before the JSON, `JSON.parse` will crash.

**Recommendation:** Improve the extraction regex to look for the first `{` and last `}` if the markdown block is missing: `text.substring(text.indexOf('{'), text.lastIndexOf('}') + 1)`.

### 4. Inline Event Handlers & Variable Scope

**Issue:** The HTML strings make heavy use of inline handlers: `onclick="deleteStudent(${s.id}, '${s.name.replace(/'/g, "\\'")}')"`.

This forces variables into the global scope (polluting `window`) and makes the code difficult to debug. Furthermore, it violates strict Content Security Policies (CSP) which forbid `unsafe-inline` execution.

**Recommendation:** Use event delegation. Attach a single event listener to the parent container and determine the action based on `event.target.closest('[data-action]')`.

---

## Architecture & Code Quality (Low/Maintenance Priority)

### 1. Tailwind CSS via CDN

**Issue:** The application uses `<script src="https://cdn.tailwindcss.com?plugins=forms,container-queries"></script>`. The Tailwind documentation explicitly states that the CDN script should not be used in production as it compiles CSS on the fly in the browser, causing performance overhead and FOUC (Flash of Unstyled Content).

**Recommendation:** Compile the Tailwind CSS using the CLI locally and serve a static `.css` file.

### 2. Global Variable Pollution

**Issue:** Variables like `state`, `currentView`, `currentTeamGroup`, `chatMessages`, and `dragData` are declared globally. This risks unintended side effects, especially as the app scales.

**Recommendation:** Encapsulate state and logic within a main `App` class or an IIFE (Immediately Invoked Function Expression) to protect the global namespace.

### 3. Redundant / Dead Code

**Issue:** The view routing system maps `settings` to `console` (`if (view === 'settings') { view = 'console'; ... }`). However, there is a dedicated `<section id="view-settings">` and a `renderSettings()` function that is mostly overlapping with `renderConsoleAiModel()`.

**Recommendation:** Clean up dead routing and consolidate redundant view logic to make the codebase leaner.

### 4. Accessibility (a11y)

**Issue:** Several interactive elements (like the drag-and-drop calendar slots or tag chips) lack proper ARIA attributes and keyboard support. For example, a user cannot easily drop a team into a slot using only the keyboard.

**Recommendation:** Ensure interactive non-button elements have `tabindex="0"`, appropriate `role` attributes, and `onkeydown` listeners mapped to space/enter keys.
