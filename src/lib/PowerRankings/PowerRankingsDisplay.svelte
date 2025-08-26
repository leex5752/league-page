<script>
  import { onMount, onDestroy } from "svelte";

  // Props from your page
  export let graphs = [];
  export let curGraph = 0;
  export let leagueTeamManagers;

  // ----- Layout: responsive container measured by ResizeObserver
  let host;           // outer container
  let width = 0;      // measured width
  let height = 360;   // default height; adjust below based on width if you like
  let ro;

  function measure() {
    const w = host?.getBoundingClientRect()?.width ?? 0;
    width = w;
    // Option: keep a stable aspect ratio so the parent can compute height
    // Comment this out if you want a fixed height.
    height = Math.max(320, Math.round(w * 0.6));
  }

  onMount(() => {
    measure();
    ro = new ResizeObserver(measure);
    if (host) ro.observe(host);
  });

  onDestroy(() => ro?.disconnect());

  // ----- Data plumbing
  $: graph = graphs?.[curGraph];
  $: data = graph?.stats ?? [];

  // Accessors
  const label = (d) => d.manager ?? d.rosterID ?? "";
  const value = (d) => Number(d[graph?.field ?? "value"] ?? 0);

  // Simple scales (vanilla)
  $: maxVal = Math.max(0, ...data.map(value));
  $: margin = { top: 24, right: 16, bottom: 56, left: 56 };
  $: innerW = Math.max(0, width - margin.left - margin.right);
  $: innerH = Math.max(0, height - margin.top - margin.bottom);

  $: n = data.length || 1;
  $: band = innerW / n;
  const barPad = 0.15; // 15% padding inside each band
  $: barW = Math.max(1, band * (1 - barPad));

  const xPos = (i) => margin.left + i * band + (band - barW) / 2;
  const yPos = (v) =>
    margin.top + innerH - (maxVal ? (v / maxVal) * innerH : 0);

  // Tooltip (optional)
  let tip = { show: false, x: 0, y: 0, text: "" };
  function showTip(event, d) {
    tip = {
      show: true,
      x: event.offsetX + 12,
      y: event.offsetY + 12,
      text: `${label(d)}: ${value(d)}`
    };
  }
  function hideTip() {
    tip.show = false;
  }
</script>

<!-- Root stays in normal flow; height comes from content, not absolute positioning -->
<div class="chart-root" bind:this={host}>
  <header class="chart-header">
    <h3>{graph?.header ?? "Bar Chart"}</h3>
    {#if graph?.short}<small>{graph.short}</small>{/if}
  </header>

  <!-- SVG is block-level; viewBox makes it responsive -->
  <svg
    class="chart-svg"
    {width}
    {height}
    viewBox={`0 0 ${width} ${height}`}
    preserveAspectRatio="xMidYMid meet"
    on:mouseleave={hideTip}
  >
    <!-- Axes (very simple placeholders) -->
    <g class="y-axis">
      <line x1={margin.left} y1={margin.top} x2={margin.left} y2={margin.top + innerH} />
      <line x1={margin.left} y1={margin.top + innerH} x2={margin.left + innerW} y2={margin.top + innerH} />
    </g>

    <!-- Bars -->
    {#each data as d, i}
      {#key i}
        <g
          class="bar"
          on:mousemove={(e) => showTip(e, d)}
          on:mouseleave={hideTip}
        >
          <rect
            x={xPos(i)}
            y={yPos(value(d))}
            width={barW}
            height={margin.top + innerH - yPos(value(d))}
            rx="4"
            ry="4"
          />
          <!-- X labels -->
          <text
            x={xPos(i) + barW / 2}
            y={margin.top + innerH + 18}
            text-anchor="middle"
            class="x-label"
          >
            {label(d)}
          </text>
        </g>
      {/key}
    {/each}

    <!-- Y title -->
    {#if graph?.y}
      <text
        x="0"
        y="0"
        transform={`translate(16, ${margin.top + innerH / 2}) rotate(-90)`}
        dominant-baseline="middle"
        class="axis-title"
      >
        {graph.y}
      </text>
    {/if}

    <!-- X title -->
    {#if graph?.x}
      <text
        x={margin.left + innerW / 2}
        y={margin.top + innerH + 40}
        text-anchor="middle"
        class="axis-title"
      >
        {graph.x}
      </text>
    {/if}
  </svg>

  <!-- Tooltip lives AFTER the SVG so it's not clipped; still inside flow -->
  {#if tip.show}
    <div
      class="tooltip"
      style={`left:${tip.x}px; top:${tip.y}px;`}
      aria-hidden="true"
    >
      {tip.text}
    </div>
  {/if}
</div>

<style>
  /* Keep everything block-level so it stacks under your text */
  .chart-root {
    display: block;
    position: relative;  /* for the tooltip to position within */
    width: 100%;
    /* If you ever had float elements above, this makes sure we drop below them */
    clear: both;
  }

  .chart-header {
    display: flex;
    align-items: baseline;
    gap: 0.5rem;
    margin-bottom: 0.5rem;
  }

  .chart-svg {
    display: block;      /* critical: avoid inline SVG overlap */
    width: 100%;
    height: auto;        /* scales via viewBox */
    overflow: visible;
  }

  .y-axis line {
    stroke: #ccc;
    stroke-width: 1;
    shape-rendering: crispEdges;
  }

  .bar rect {
    /* No specific color to keep it theme-agnostic; add one if you want */
    fill: currentColor;      /* inherits from text color */
    opacity: 0.9;
  }

  .bar rect:hover {
    opacity: 1;
  }

  .x-label, .axis-title {
    font-size: 12px;
    fill: currentColor;
  }

  /* Tooltip in the same stacking context; no fixed/absolute to the page */
  .tooltip {
    position: absolute;
    pointer-events: none;
    background: rgba(0,0,0,0.75);
    color: #fff;
    padding: 6px 8px;
    border-radius: 6px;
    font-size: 12px;
    white-space: nowrap;
    transform: translate(0, -100%);
    z-index: 1;
  }
</style>
