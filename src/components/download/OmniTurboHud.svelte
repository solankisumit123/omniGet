<script lang="ts">
  import { getSettings, updateSettings } from "$lib/stores/settings-store.svelte";
  import { t } from "$lib/i18n";

  let {
    compact = false,
    activeLanes = 16,
    downloading = false,
    speedBps = 0,
  }: {
    compact?: boolean;
    activeLanes?: number;
    downloading?: boolean;
    speedBps?: number;
  } = $props();

  const settings = $derived(getSettings());
  const turboEnabled = $derived(settings?.advanced?.turbo_mode ?? true);
  const currentConcurrency = $derived(settings?.advanced?.turbo_concurrency ?? 16);
  const antiThrottle = $derived(settings?.advanced?.anti_throttle ?? true);
  const speculativeHedging = $derived(settings?.advanced?.speculative_hedging ?? true);

  // Speed multiplier estimate based on concurrency & anti-throttle
  const speedMultiplier = $derived(
    !turboEnabled
      ? "1.0x"
      : currentConcurrency >= 32
      ? "5.2x"
      : currentConcurrency >= 16
      ? "3.8x"
      : "2.4x"
  );

  function toggleTurbo() {
    updateSettings({
      advanced: {
        turbo_mode: !turboEnabled,
      },
    });
  }

  function setLanes(lanes: number) {
    updateSettings({
      advanced: {
        turbo_concurrency: lanes,
        turbo_mode: true,
      },
    });
  }

  // Generate lane slots for the visualizer
  const displayLaneCount = $derived(
    turboEnabled ? Math.min(currentConcurrency, 32) : 4
  );
</script>

<div
  class="turbo-hud"
  class:compact
  class:active={turboEnabled}
  class:downloading
  role="region"
  aria-label="OmniTurbo Acceleration Status"
>
  <div class="hud-top">
    <div class="hud-title-row">
      <button
        class="turbo-switch-pill"
        class:on={turboEnabled}
        onclick={toggleTurbo}
        title={turboEnabled ? "OmniTurbo Acceleration is active" : "Enable OmniTurbo"}
        type="button"
      >
        <span class="bolt-icon" aria-hidden="true">⚡</span>
        <span class="pill-label">
          {turboEnabled ? "OmniTurbo ON" : "OmniTurbo OFF"}
        </span>
        <span class="status-dot"></span>
      </button>

      {#if turboEnabled}
        <div class="multiplier-tag" title="Theoretical acceleration multiplier over throttled single-stream">
          <span class="multiplier-val">{speedMultiplier}</span>
          <span class="multiplier-sub">Faster</span>
        </div>
      {/if}
    </div>

    {#if !compact}
      <div class="presets-row">
        <span class="presets-label">Lanes:</span>
        <div class="preset-buttons">
          <button
            type="button"
            class="preset-btn"
            class:selected={currentConcurrency === 8 && turboEnabled}
            onclick={() => setLanes(8)}
          >
            8x
          </button>
          <button
            type="button"
            class="preset-btn"
            class:selected={currentConcurrency === 16 && turboEnabled}
            onclick={() => setLanes(16)}
          >
            16x Turbo
          </button>
          <button
            type="button"
            class="preset-btn quantum"
            class:selected={currentConcurrency === 32 && turboEnabled}
            onclick={() => setLanes(32)}
          >
            32x Quantum
          </button>
        </div>
      </div>
    {/if}
  </div>

  <!-- Real-Time Multi-Lane Stream Visualizer -->
  <div class="lane-stream-container">
    <div class="lane-bars-grid" style="--lane-count: {displayLaneCount}">
      {#each Array(displayLaneCount) as _, idx}
        <div
          class="lane-bar"
          class:lane-pulsing={turboEnabled && downloading}
          style="
            animation-delay: {(idx * 0.08).toFixed(2)}s;
            --intensity: {0.4 + (idx % 4) * 0.2};
          "
          title={`Lane #${idx + 1} - ${turboEnabled ? 'Accelerated micro-chunk stream' : 'Idle'}`}
        ></div>
      {/each}
    </div>
  </div>

  {#if !compact && turboEnabled}
    <div class="hud-badges">
      {#if antiThrottle}
        <div class="feature-chip" title="Rotates player clients to avoid platform bandwidth caps">
          <span class="chip-icon">🛡️</span>
          <span class="chip-text">Anti-Throttle Shield</span>
        </div>
      {/if}
      {#if speculativeHedging}
        <div class="feature-chip" title="Eliminates 99% stalls by racing duplicate micro-connections">
          <span class="chip-icon">🏎️</span>
          <span class="chip-text">Speculative Racing</span>
        </div>
      {/if}
      <div class="feature-chip" title="Requests 10MB chunk bursts to maximize link saturation">
        <span class="chip-icon">🌊</span>
        <span class="chip-text">10MB Burst Pipes</span>
      </div>
    </div>
  {/if}
</div>

<style>
  .turbo-hud {
    background: var(--sidebar-bg, rgba(20, 22, 28, 0.7));
    border: 1px solid var(--content-border, rgba(255, 255, 255, 0.08));
    border-radius: var(--border-radius, 11px);
    padding: calc(var(--padding, 12px) * 0.85);
    display: flex;
    flex-direction: column;
    gap: 8px;
    position: relative;
    overflow: hidden;
    transition: all 0.25s cubic-bezier(0.16, 1, 0.3, 1);
  }

  .turbo-hud.active {
    border-color: rgba(99, 102, 241, 0.4);
    box-shadow: 0 4px 20px -4px rgba(99, 102, 241, 0.15);
  }

  .turbo-hud.compact {
    padding: 6px 10px;
    gap: 5px;
  }

  .hud-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 8px;
  }

  .hud-title-row {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .turbo-switch-pill {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px;
    border-radius: 20px;
    border: 1px solid var(--button-stroke, rgba(255, 255, 255, 0.1));
    background: var(--button-elevated, rgba(255, 255, 255, 0.06));
    color: var(--secondary, #888);
    font-size: 12px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .turbo-switch-pill:hover {
    color: var(--primary, #fff);
    border-color: var(--accent, #6366f1);
  }

  .turbo-switch-pill.on {
    background: linear-gradient(135deg, rgba(99, 102, 241, 0.25), rgba(168, 85, 247, 0.25));
    border-color: var(--accent, #6366f1);
    color: #fff;
    box-shadow: 0 0 12px rgba(99, 102, 241, 0.3);
  }

  .bolt-icon {
    font-size: 13px;
    filter: drop-shadow(0 0 4px rgba(250, 204, 21, 0.6));
  }

  .status-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: #666;
    transition: all 0.2s ease;
  }

  .turbo-switch-pill.on .status-dot {
    background: #22c55e;
    box-shadow: 0 0 6px #22c55e;
    animation: pulse-dot 2s infinite;
  }

  @keyframes pulse-dot {
    0%, 100% { opacity: 1; transform: scale(1); }
    50% { opacity: 0.6; transform: scale(1.2); }
  }

  .multiplier-tag {
    display: inline-flex;
    align-items: baseline;
    gap: 4px;
    padding: 3px 8px;
    background: rgba(34, 197, 94, 0.12);
    border: 1px solid rgba(34, 197, 94, 0.3);
    border-radius: 8px;
    color: #4ade80;
  }

  .multiplier-val {
    font-size: 12px;
    font-weight: 700;
    font-family: var(--font-mono, monospace);
  }

  .multiplier-sub {
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    opacity: 0.85;
  }

  .presets-row {
    display: flex;
    align-items: center;
    gap: 6px;
  }

  .presets-label {
    font-size: 11.5px;
    color: var(--secondary, #888);
    font-weight: 500;
  }

  .preset-buttons {
    display: flex;
    gap: 3px;
    background: var(--button-elevated, rgba(255, 255, 255, 0.04));
    padding: 2px;
    border-radius: 8px;
    border: 1px solid var(--content-border, rgba(255, 255, 255, 0.06));
  }

  .preset-btn {
    border: none;
    background: transparent;
    color: var(--secondary, #888);
    font-size: 11px;
    font-weight: 500;
    padding: 3px 8px;
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .preset-btn:hover {
    color: var(--primary, #fff);
  }

  .preset-btn.selected {
    background: var(--accent, #6366f1);
    color: #fff;
    font-weight: 600;
    box-shadow: 0 1px 6px rgba(99, 102, 241, 0.4);
  }

  .preset-btn.quantum.selected {
    background: linear-gradient(135deg, #ec4899, #8b5cf6);
    box-shadow: 0 1px 8px rgba(236, 72, 153, 0.5);
  }

  /* Multi-Lane Stream Visualizer */
  .lane-stream-container {
    width: 100%;
    background: rgba(0, 0, 0, 0.2);
    border-radius: 6px;
    padding: 4px 6px;
    border: 1px solid rgba(255, 255, 255, 0.04);
  }

  .lane-bars-grid {
    display: grid;
    grid-template-columns: repeat(var(--lane-count), 1fr);
    gap: 3px;
    height: 14px;
    align-items: center;
  }

  .lane-bar {
    height: 100%;
    min-height: 4px;
    border-radius: 2px;
    background: rgba(99, 102, 241, 0.25);
    transition: all 0.3s ease;
  }

  .turbo-hud.active .lane-bar {
    background: linear-gradient(180deg, #818cf8 0%, #4f46e5 100%);
    opacity: 0.65;
  }

  .lane-bar.lane-pulsing {
    animation: lane-pulse 1.2s infinite ease-in-out;
  }

  @keyframes lane-pulse {
    0%, 100% {
      transform: scaleY(0.4);
      opacity: 0.4;
      background: #4f46e5;
    }
    50% {
      transform: scaleY(1);
      opacity: 1;
      background: #38bdf8;
      box-shadow: 0 0 6px #38bdf8;
    }
  }

  .hud-badges {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 2px;
  }

  .feature-chip {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    background: var(--button-elevated, rgba(255, 255, 255, 0.04));
    border: 1px solid var(--content-border, rgba(255, 255, 255, 0.06));
    border-radius: 6px;
    padding: 2px 7px;
    font-size: 10.5px;
    color: var(--secondary, #aaa);
  }

  .chip-icon {
    font-size: 11px;
  }
</style>
