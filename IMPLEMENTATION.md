## Overview

This document explains, in simple terms, how the slide-to-confirm demo is wired together and what each key file is responsible for.

---

## `src/lib/components/SlideToConfirm.svelte`

**Purpose:** A reusable “slide to confirm” UI component with different difficulty levels.

- **What it shows**
  - A pill-shaped track with a draggable circular handle.
  - A progress fill that moves as you drag.
  - Optional hard-mode “target zone” you must land in.
  - A label that either shows instructions, a countdown timer, or a success message.

- **Props (inputs)**
  - `difficulty`: `"easy" | "medium" | "hard"` – how strict the confirmation is.
    - **easy**: just drag all the way to the right, no time limit.
    - **medium**: drag all the way to the right **before time runs out**.
    - **hard**: drag into a specific yellow “target zone” **before time runs out**.
  - `label`: text shown before you start (e.g. “Slide to confirm”).
  - `successLabel`: text shown after success (e.g. “Confirmed!”).
  - `timeLimit`: how long you have to complete (in ms, default 1000 = 1s).
  - `resetDelay`: how long to show success before it auto-resets.
  - `onconfirm`: callback function run when the user successfully confirms.

- **Internal state**
  - `status`: `"idle" | "dragging" | "success"` – what phase the slider is in.
  - `progress`: number between 0 and 1 that represents how far the handle has moved.
  - `timeRemaining`: countdown in seconds (used for medium/hard).
  - `targetZone`: start/end percentages for the “hard” yellow zone.
  - Various internal values to track pointer drag distance and timing.

- **How interaction works**
  - When you press/drag on the handle:
    - It measures the track width to know how far the handle can move.
    - It updates `progress` based on the pointer’s position, clamped to the track.
    - For medium/hard, it starts a timer and updates `timeRemaining` using `requestAnimationFrame`.
  - **Easy mode:** as soon as `progress >= 1`, it triggers success.
  - **Medium mode:** as soon as `progress >= 1` **and** there is still time left, it triggers success.
  - **Hard mode:** success is only checked when you release; if you release while:
    - there is time left, and
    - `progress` is within the target zone
    - then it calls `onconfirm` and shows success.
  - If you fail (don’t reach the end/zone in time), it resets back to idle.

- **Keyboard support**
  - Space/Enter starts a “keyboard drag” and also finishes it (like pointer down/up).
  - Arrow Right/Left nudge progress forwards/backwards in small steps.
  - The same completion rules (easy/medium/hard) apply to keyboard interaction.

- **Styling**
  - CSS at the bottom handles:
    - Track appearance, focus ring, success colors.
    - Progress fill animation and success state (full green).
    - Hard-mode yellow dashed target zone.
    - Handle visuals, shadows, pointer/keyboard animations.
    - Label and icons (arrow/checkmark).

In short: this file encapsulates all the logic and visuals for a robust, accessible, “slide to confirm” control that can be dropped into any page and wired via the `onconfirm` prop.

---

## `src/routes/demo/+page.svelte`

**Purpose:** A demo page that showcases the `SlideToConfirm` component in different difficulty modes and a “real-world” example.

- **Imports and local state**
  - Imports `SlideToConfirm`.
  - Tracks:
    - `lastConfirmed`: which difficulty was last successfully confirmed.
    - `confirmCount`: how many confirmations have happened in total.
    - `scenarioPhase`: `"idle" | "confirming" | "confirmed"` for the meme-coin example.
  - `handleConfirm(difficulty)`: updates `confirmCount` and `lastConfirmed`, and auto-clears the message after 3 seconds (only if the same difficulty is still the last one).
  - `startScenario()` / `completeScenario()`: move the real-world example through its phases.
  - `difficulties`: an array describing each difficulty (key, title, description, color) used to render the grid.

- **Head metadata**
  - Sets the page `<title>` and `<meta name="description">` for SEO and browser tab text.

- **Layout sections**
  1. **Header**
     - Title and subtitle explaining what the page is about.
  2. **Cards grid**
     - Maps over `difficulties` to render one card per difficulty.
     - Each card shows a badge, a short description, and a `SlideToConfirm` component.
     - The slider passes:
       - `difficulty={diff.key}`
       - `onconfirm={() => handleConfirm(diff.key)}`
     - So each slider updates the shared confirmation status when completed.
  3. **Status bar**
     - Shows:
       - A default hint when nothing has been confirmed yet.
       - Or a green-ish “X confirmed! (N total)” message when something was confirmed recently.
     - Uses `lastConfirmed` and `confirmCount` to know what to show.
  4. **Real-world scenario**
     - Simulates a high-stakes action: “Invest $1 billion dollars into a shitty meme coin”.
     - Three phases:
       - `idle`: shows a red button. Clicking it sets `scenarioPhase` to `"confirming"`.
       - `confirming`: shows a warning text and a **hard** `SlideToConfirm` with a label like “Slide to invest”. On success (`onconfirm`), it calls `completeScenario()` and moves to `"confirmed"`.
       - `confirmed`: shows a success block with a check icon and a snarky success message.

- **Styling**
  - Defines the entire visual layout of the demo page:
    - Overall page width and padding.
    - Card grid (3 columns on desktop, 1 on mobile).
    - Card styling (borders, background, hover shadow).
    - Status bar styling and “active” state.
    - Scenario card, button, and success message styling.

In short: this file is the **showcase UI** for `SlideToConfirm`—it demonstrates how to wire the component up, track confirmations, and integrate it into a realistic flow.

---

## `src/routes/demo/+page.server.js`

**Purpose:** Provide server-side data for the `/demo` page, while reusing any shared layout data.

- **What it does**
  - Exports a `load` function that SvelteKit runs on the server when the `/demo` route is requested.
  - Calls `parent()` to get any data provided by the parent layout’s `load` function (e.g. global navigation, site config).
  - Returns `{ ...parentData }`, effectively passing all parent data straight through to this page.

- **Why it exists**
  - Keeps the `/demo` page compatible with the rest of the app’s data loading.
  - Makes it easy to add extra server-side data later (you can return more fields alongside `parentData`).

In short: this file just forwards layout data to the demo page, so the demo integrates cleanly with the app’s global data flow.

