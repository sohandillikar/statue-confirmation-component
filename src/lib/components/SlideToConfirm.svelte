<script>
  /**
   * SlideToConfirm — interactive slide-to-confirm component with 3 difficulty levels.
   *
   * Easy:   Slide handle to 100 %. No time limit.
   * Medium: Slide handle to 100 % within a time limit (default 3 s).
   * Hard:   Slide handle into a highlighted precision target zone within a time limit (default 3 s).
   */

  /** @type {{ difficulty?: 'easy' | 'medium' | 'hard', label?: string, successLabel?: string, timeLimit?: number, resetDelay?: number, onconfirm?: () => void }} */
  let {
    difficulty = "easy",
    label = "Slide to confirm",
    successLabel = "Confirmed!",
    timeLimit = 1000,
    resetDelay = 1500,
    onconfirm = () => {},
  } = $props();

  // ── Internal state ────────────────────────────────────────────────
  /** @type {'idle' | 'dragging' | 'success'} */
  let status = $state("idle");
  let progress = $state(0); // 0 → 1
  let timeRemaining = $state(0); // seconds (displayed)

  // Hard-mode target zone (fraction of track, e.g. { start: 0.5, end: 0.7 })
  let targetZone = $state({ start: 0, end: 0 });

  // DOM refs
  /** @type {HTMLDivElement | null} */
  let trackEl = $state(null);

  // Pointer tracking
  let handleOffset = 0; // offset within handle at grab time
  let maxTravel = 0; // track width minus handle width
  let timerStart = 0; // timestamp when timer began
  let rafId = 0;
  /** @type {ReturnType<typeof setTimeout> | undefined} */
  let resetTimerId;

  const HANDLE_SIZE = 48; // px – keep in sync with CSS
  const TARGET_ZONE_WIDTH = 0.2; // 20 % of track

  // ── Derived ───────────────────────────────────────────────────────
  let hasTimer = $derived(difficulty === "medium" || difficulty === "hard");
  let timeLimitSec = $derived(timeLimit / 1000);

  // ── Helpers ───────────────────────────────────────────────────────

  function generateTargetZone() {
    // Random start between 0.30 and 0.70 so the zone always fits on-screen
    const start = 0.3 + Math.random() * 0.4;
    const end = Math.min(start + TARGET_ZONE_WIDTH, 1);
    targetZone = { start, end };
  }

  function resetComponent() {
    status = "idle";
    progress = 0;
    timeRemaining = 0;
    cancelAnimationFrame(rafId);
    if (resetTimerId !== undefined) clearTimeout(resetTimerId);
    if (difficulty === "hard") {
      generateTargetZone();
    }
  }

  function triggerSuccess() {
    status = "success";
    cancelAnimationFrame(rafId);
    onconfirm();
    resetTimerId = setTimeout(() => {
      resetComponent();
    }, resetDelay);
  }

  function checkEasyCompletion() {
    if (progress >= 1) {
      triggerSuccess();
    }
  }

  function checkMediumCompletion() {
    if (progress >= 1 && timeRemaining > 0) {
      triggerSuccess();
    }
  }

  // Hard-mode: success is checked on pointer-up (release inside zone)

  // ── Timer loop (rAF) ──────────────────────────────────────────────

  function timerLoop() {
    if (status !== "dragging") return;

    const elapsed = performance.now() - timerStart;
    const remaining = Math.max(0, timeLimit - elapsed);
    timeRemaining = remaining / 1000;

    if (remaining <= 0) {
      // Timeout — reset
      timeRemaining = 0;
      resetComponent();
      return;
    }

    rafId = requestAnimationFrame(timerLoop);
  }

  // ── Pointer handlers ──────────────────────────────────────────────

  /** @param {PointerEvent} e */
  function handlePointerDown(e) {
    if (status === "success") return;
    e.preventDefault();

    if (!trackEl) return;
    const trackRect = trackEl.getBoundingClientRect();
    maxTravel = trackRect.width - HANDLE_SIZE;

    // Where inside the handle did the user grab?
    handleOffset = e.clientX - trackRect.left - progress * maxTravel;

    status = "dragging";

    if (hasTimer) {
      timerStart = performance.now();
      timeRemaining = timeLimitSec;
      rafId = requestAnimationFrame(timerLoop);
    }
  }

  /** @param {PointerEvent} e */
  function handlePointerMove(e) {
    if (status !== "dragging") return;
    if (!trackEl) return;

    const trackRect = trackEl.getBoundingClientRect();
    const rawX = e.clientX - trackRect.left - handleOffset;
    const clamped = Math.max(0, Math.min(rawX, maxTravel));
    progress = clamped / maxTravel;

    // Continuous completion checks for easy/medium
    if (difficulty === "easy") checkEasyCompletion();
    if (difficulty === "medium") checkMediumCompletion();
  }

  function handlePointerUp() {
    if (status !== "dragging") return;

    if (difficulty === "hard") {
      // Check if handle is inside target zone
      if (
        timeRemaining > 0 &&
        progress >= targetZone.start &&
        progress <= targetZone.end
      ) {
        triggerSuccess();
        return;
      }
    }

    // Not confirmed — reset
    resetComponent();
  }

  // ── Keyboard support ──────────────────────────────────────────────

  let kbDragging = false;

  /** @param {KeyboardEvent} e */
  function handleKeyDown(e) {
    if (status === "success") return;

    if (e.key === " " || e.key === "Enter") {
      e.preventDefault();
      if (!kbDragging) {
        // Start keyboard drag
        kbDragging = true;
        status = "dragging";
        if (trackEl) {
          maxTravel = trackEl.getBoundingClientRect().width - HANDLE_SIZE;
        }
        if (hasTimer) {
          timerStart = performance.now();
          timeRemaining = timeLimitSec;
          rafId = requestAnimationFrame(timerLoop);
        }
      } else {
        // End keyboard drag (like pointer-up)
        kbDragging = false;
        handlePointerUp();
      }
      return;
    }

    if (!kbDragging && status !== "dragging") return;

    const step = 0.05;
    if (e.key === "ArrowRight") {
      e.preventDefault();
      progress = Math.min(progress + step, 1);
      if (difficulty === "easy") checkEasyCompletion();
      if (difficulty === "medium") checkMediumCompletion();
    }
    if (e.key === "ArrowLeft") {
      e.preventDefault();
      progress = Math.max(progress - step, 0);
    }
  }

  // ── Lifecycle ─────────────────────────────────────────────────────

  $effect(() => {
    if (difficulty === "hard") {
      generateTargetZone();
    }
    // Reset when difficulty prop changes
    return () => {
      cancelAnimationFrame(rafId);
      if (resetTimerId !== undefined) clearTimeout(resetTimerId);
    };
  });

  // ── Computed styles ───────────────────────────────────────────────

  let handleTranslate = $derived(
    `translateX(${progress * (maxTravel || 0)}px)`,
  );
  let fillWidth = $derived(`${progress * 100}%`);

  let targetZoneLeft = $derived(`${targetZone.start * 100}%`);
  let targetZoneWidth = $derived(
    `${(targetZone.end - targetZone.start) * 100}%`,
  );
</script>

<svelte:window
  onpointermove={handlePointerMove}
  onpointerup={handlePointerUp}
/>

<div
  class="stc-track"
  class:stc-success={status === "success"}
  bind:this={trackEl}
  role="slider"
  aria-valuemin={0}
  aria-valuemax={100}
  aria-valuenow={Math.round(progress * 100)}
  aria-label="{label} — {difficulty} difficulty"
  tabindex="0"
  onkeydown={handleKeyDown}
>
  <!-- Blue / green fill -->
  <div
    class="stc-fill"
    class:stc-fill-success={status === "success"}
    style="width: {fillWidth}"
  ></div>

  <!-- Hard-mode target zone -->
  {#if difficulty === "hard" && status !== "success"}
    <div
      class="stc-target-zone"
      style="left: {targetZoneLeft}; width: {targetZoneWidth}"
    ></div>
  {/if}

  <!-- Label text -->
  <span class="stc-label">
    {#if status === "success"}
      {successLabel}
    {:else if status === "dragging" && hasTimer}
      {timeRemaining.toFixed(1)}s
    {:else}
      {label}
      <svg
        class="stc-arrow"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <line x1="5" y1="12" x2="19" y2="12" />
        <polyline points="12 5 19 12 12 19" />
      </svg>
    {/if}
  </span>

  <!-- Draggable handle -->
  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <div
    class="stc-handle"
    class:stc-handle-dragging={status === "dragging"}
    class:stc-handle-success={status === "success"}
    style="transform: {handleTranslate}"
    onpointerdown={handlePointerDown}
  >
    {#if status === "success"}
      <!-- Checkmark icon -->
      <svg
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="3"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <polyline points="20 6 9 17 4 12" />
      </svg>
    {:else}
      <!-- Arrow icon -->
      <svg
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2.5"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <polyline points="9 18 15 12 9 6" />
      </svg>
    {/if}
  </div>
</div>

<style>
  /* ── Track (pill container) ─────────────────────────────────────── */
  .stc-track {
    position: relative;
    width: 100%;
    height: 56px;
    border-radius: 999px;
    background: #e5e7eb;
    overflow: hidden;
    cursor: pointer;
    user-select: none;
    touch-action: none;
    outline: none;
  }

  .stc-track:focus-visible {
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.5);
  }

  .stc-track.stc-success {
    cursor: default;
  }

  /* ── Fill (blue → green on success) ────────────────────────────── */
  .stc-fill {
    position: absolute;
    top: 0;
    left: 0;
    height: 100%;
    background: #3b82f6;
    border-radius: 999px 0 0 999px;
    transition: background-color 0.3s ease;
    pointer-events: none;
  }

  .stc-fill-success {
    background: #22c55e !important;
    width: 100% !important;
    border-radius: 999px;
  }

  /* ── Hard-mode target zone ─────────────────────────────────────── */
  .stc-target-zone {
    position: absolute;
    top: 4px;
    bottom: 4px;
    border-radius: 12px;
    background: rgba(251, 191, 36, 0.35);
    border: 2px dashed #f59e0b;
    pointer-events: none;
    z-index: 1;
  }

  /* ── Label text ────────────────────────────────────────────────── */
  .stc-label {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    padding-left: 56px; /* 48px handle + 4px offset + 4px gap - Used to be 56px */
    padding-right: 56px;
    font-size: 0.9rem;
    font-weight: 600;
    color: #6b7280;
    letter-spacing: 0.025em;
    pointer-events: none;
    z-index: 2;
    transition: color 0.3s ease;
  }

  .stc-success .stc-label {
    color: #fff;
  }

  .stc-arrow {
    width: 18px;
    height: 18px;
    opacity: 0.5;
    animation: stc-nudge 1.5s ease-in-out infinite;
  }

  @keyframes stc-nudge {
    0%,
    100% {
      transform: translateX(0);
    }
    50% {
      transform: translateX(4px);
    }
  }

  /* ── Handle (draggable thumb) ──────────────────────────────────── */
  .stc-handle {
    position: absolute;
    top: 4px;
    left: 4px;
    width: 48px;
    height: 48px;
    border-radius: 50%;
    background: #fff;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow:
      0 2px 6px rgba(0, 0, 0, 0.15),
      0 1px 2px rgba(0, 0, 0, 0.1);
    z-index: 3;
    transition:
      transform 0.35s cubic-bezier(0.34, 1.56, 0.64, 1),
      background-color 0.3s ease,
      box-shadow 0.2s ease;
    cursor: grab;
  }

  .stc-handle:active,
  .stc-handle-dragging {
    cursor: grabbing;
    transition:
      background-color 0.3s ease,
      box-shadow 0.2s ease;
    box-shadow:
      0 4px 12px rgba(0, 0, 0, 0.2),
      0 2px 4px rgba(0, 0, 0, 0.1);
  }

  .stc-handle-success {
    background: #22c55e;
    cursor: default;
  }

  .stc-handle svg {
    width: 22px;
    height: 22px;
    color: #3b82f6;
    transition: color 0.3s ease;
  }

  .stc-handle-success svg {
    color: #fff;
  }
</style>
