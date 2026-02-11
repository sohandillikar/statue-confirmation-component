<script>
  import SlideToConfirm from "$lib/components/SlideToConfirm.svelte";

  let lastConfirmed = $state("");
  let confirmCount = $state(0);

  /** @param {string} difficulty */
  function handleConfirm(difficulty) {
    confirmCount++;
    lastConfirmed = difficulty;
    // Clear after a few seconds
    setTimeout(() => {
      if (lastConfirmed === difficulty) {
        lastConfirmed = "";
      }
    }, 3000);
  }

  // ── Real-world scenario state ─────────────────────────────────
  /** @type {'idle' | 'confirming' | 'confirmed'} */
  let scenarioPhase = $state("idle");

  function startScenario() {
    scenarioPhase = "confirming";
  }

  function completeScenario() {
    scenarioPhase = "confirmed";
  }

  /** @type {Array<{key: 'easy' | 'medium' | 'hard', title: string, description: string, color: string}>} */
  const difficulties = [
    {
      key: "easy",
      title: "Easy",
      description:
        "Slide the handle all the way to the right. No time pressure.",
      color: "#22c55e",
    },
    {
      key: "medium",
      title: "Medium",
      description: "Slide all the way to the right within 1 second.",
      color: "#f59e0b",
    },
    {
      key: "hard",
      title: "Hard",
      description: "Slide into the highlighted target zone within 1 second.",
      color: "#ef4444",
    },
  ];
</script>

<svelte:head>
  <title>Slide to Confirm Demo</title>
  <meta
    name="description"
    content="Interactive slide-to-confirm component demo with multiple difficulty levels."
  />
</svelte:head>

<div class="demo-page">
  <!-- Header -->
  <header class="demo-header">
    <h1>Slide to Confirm</h1>
    <p class="demo-subtitle">
      An interactive confirmation component that requires deliberate user
      action. Try each difficulty level below.
    </p>
  </header>

  <!-- Cards grid -->
  <div class="demo-grid">
    {#each difficulties as diff (diff.key)}
      <div class="demo-card">
        <div class="card-header">
          <span class="difficulty-badge" style="background: {diff.color}">
            {diff.title}
          </span>
          {#if diff.key === "medium" || diff.key === "hard"}
            <span class="timer-badge">1s</span>
          {/if}
        </div>

        <p class="card-description">{diff.description}</p>

        <div class="slider-area">
          <SlideToConfirm
            difficulty={diff.key}
            onconfirm={() => handleConfirm(diff.key)}
          />
        </div>
      </div>
    {/each}
  </div>

  <!-- Status bar -->
  <div class="status-bar" class:status-active={lastConfirmed !== ""}>
    {#if lastConfirmed}
      <svg
        class="status-icon"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2.5"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <polyline points="20 6 9 17 4 12" />
      </svg>
      <span>
        <strong
          >{lastConfirmed.charAt(0).toUpperCase() +
            lastConfirmed.slice(1)}</strong
        >
        confirmed!
        <span class="confirm-count">({confirmCount} total)</span>
      </span>
    {:else}
      <span class="status-hint"
        >Slide a handle to see the confirmation in action</span
      >
    {/if}
  </div>

  <!-- Real-world scenario -->
  <section class="scenario-section">
    <h2 class="scenario-heading">Real-World Example</h2>
    <p class="scenario-subtitle">
      See the confirmation component used as a safeguard for a high-stakes
      action.
    </p>

    <div class="scenario-card">
      {#if scenarioPhase === "idle"}
        <button class="scenario-btn" onclick={startScenario}>
          Invest $1 billion dollars into a shitty meme coin
        </button>
      {:else if scenarioPhase === "confirming"}
        <p class="scenario-prompt">Are you sure? Slide to confirm.</p>
        <SlideToConfirm
          difficulty="hard"
          label="Slide to invest"
          successLabel="Invested!"
          timeLimit={1000}
          onconfirm={completeScenario}
        />
      {:else}
        <div class="scenario-success">
          <svg
            class="scenario-success-icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2.5"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <polyline points="20 6 9 17 4 12" />
          </svg>
          <p class="scenario-success-text">
            You've successfully invested $1 billion dollars into a shitty meme
            coin, you moron.
          </p>
        </div>
      {/if}
    </div>
  </section>
</div>

<style>
  .demo-page {
    max-width: 960px;
    margin: 0 auto;
    padding: 3rem 1.5rem 4rem;
  }

  /* ── Header ────────────────────────────────────────────────────── */
  .demo-header {
    text-align: center;
    margin-bottom: 2.5rem;
  }

  .demo-header h1 {
    font-size: 2rem;
    font-weight: 700;
    color: #f9fafb;
    margin: 0 0 0.5rem;
  }

  .demo-subtitle {
    font-size: 1.05rem;
    color: #9ca3af;
    max-width: 520px;
    margin: 0 auto;
    line-height: 1.6;
  }

  /* ── Grid ──────────────────────────────────────────────────────── */
  .demo-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 1.5rem;
    margin-bottom: 2rem;
  }

  @media (max-width: 768px) {
    .demo-grid {
      grid-template-columns: 1fr;
    }
  }

  /* ── Card ──────────────────────────────────────────────────────── */
  .demo-card {
    background: #fff;
    border: 1px solid #e5e7eb;
    border-radius: 16px;
    padding: 1.5rem;
    display: flex;
    flex-direction: column;
    gap: 1rem;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.06);
    transition: box-shadow 0.2s ease;
  }

  .demo-card:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  }

  .card-header {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .difficulty-badge {
    display: inline-block;
    padding: 0.25rem 0.75rem;
    border-radius: 999px;
    color: #fff;
    font-size: 0.8rem;
    font-weight: 600;
    letter-spacing: 0.03em;
  }

  .timer-badge {
    display: inline-flex;
    align-items: center;
    padding: 0.2rem 0.5rem;
    border-radius: 6px;
    background: #f3f4f6;
    color: #6b7280;
    font-size: 0.75rem;
    font-weight: 500;
    font-variant-numeric: tabular-nums;
  }

  .card-description {
    font-size: 0.9rem;
    color: #6b7280;
    line-height: 1.5;
    margin: 0;
    flex: 1;
  }

  .slider-area {
    margin-top: auto;
  }

  /* ── Status bar ────────────────────────────────────────────────── */
  .status-bar {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
    padding: 1rem 1.5rem;
    border-radius: 12px;
    background: #f9fafb;
    border: 1px solid #e5e7eb;
    font-size: 0.95rem;
    color: #6b7280;
    transition: all 0.3s ease;
  }

  .status-active {
    background: #f0fdf4;
    border-color: #bbf7d0;
    color: #166534;
  }

  .status-icon {
    width: 20px;
    height: 20px;
    color: #22c55e;
    flex-shrink: 0;
  }

  .status-hint {
    font-style: italic;
    opacity: 0.7;
  }

  .confirm-count {
    opacity: 0.6;
    font-size: 0.85rem;
  }

  /* ── Real-world scenario ───────────────────────────────────────── */
  .scenario-section {
    margin-top: 3rem;
    text-align: center;
  }

  .scenario-heading {
    font-size: 1.5rem;
    font-weight: 700;
    color: #f9fafb;
    margin: 0 0 0.4rem;
  }

  .scenario-subtitle {
    font-size: 0.95rem;
    color: #9ca3af;
    margin: 0 0 1.5rem;
  }

  .scenario-card {
    background: #fff;
    border: 1px solid #e5e7eb;
    border-radius: 16px;
    padding: 2rem;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.06);
    max-width: 520px;
    margin: 0 auto;
  }

  .scenario-btn {
    display: inline-block;
    padding: 0.85rem 1.75rem;
    background: #dc2626;
    color: #fff;
    font-size: 1rem;
    font-weight: 600;
    border: none;
    border-radius: 10px;
    cursor: pointer;
    transition:
      background 0.2s ease,
      transform 0.1s ease;
  }

  .scenario-btn:hover {
    background: #b91c1c;
  }

  .scenario-btn:active {
    transform: scale(0.98);
  }

  .scenario-prompt {
    font-size: 0.95rem;
    font-weight: 600;
    color: #374151;
    margin: 0 0 1rem;
  }

  .scenario-success {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.75rem;
    padding: 1rem;
    background: #f0fdf4;
    border: 1px solid #bbf7d0;
    border-radius: 12px;
  }

  .scenario-success-icon {
    width: 32px;
    height: 32px;
    color: #22c55e;
  }

  .scenario-success-text {
    font-size: 1rem;
    font-weight: 600;
    color: #166534;
    margin: 0;
    line-height: 1.5;
  }
</style>
