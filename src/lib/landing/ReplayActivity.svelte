<script>
  // The landing page's agent-activity panel, replaying a scripted demo run on a loop
  // instead of polling a backend. Messages match what the real agents logged.
  // Newest first, in a window of fixed height: each new line slides in at the top, and older
  // ones move down and fade out at the bottom, so the panel never changes size.
  import { onMount } from 'svelte';
  import { fly } from 'svelte/transition';
  import { flip } from 'svelte/animate';
  import { cubicOut } from 'svelte/easing';

  // Each run is a new item and request, so their IDs count up from run to run
  const item = (run) => `itm_${String(142 + run).padStart(4, '0')}`;
  const req = (run) => `req_${String(31 + run).padStart(4, '0')}`;
  const SCRIPT = [
    { agent: 'orchestrator', message: () => 'New item received' },
    { agent: 'parser', message: () => 'Identified black Columbia puffer jacket, size M' },
    { agent: 'pricing', message: () => 'Going rate $35–55 from listings online' },
    { agent: 'orchestrator', message: (run) => `Saved ${item(run)}; searchable now` },
    { agent: 'requester', message: () => 'Inferred snow and sub-freezing winters; needs insulated, water-resistant' },
    { agent: 'orchestrator', message: (run) => `Found 3 matches for ${req(run)}` },
    { agent: 'orchestrator', message: (run) => `Reserved ${item(run)}; donor notified` }
  ];
  const STEP_MS = 2400;
  const REST_MS = 4500; // pause on the finished run before the next one
  const KEEP = 10; // lines kept, more than the window shows, so the bottom stays full

  const AGENT_COLOR = { parser: 'text-[#f6bd66]', pricing: 'text-[#e0805f]', requester: 'text-[#a2c19b]', orchestrator: 'text-night-soft' };

  const clock = (at) =>
    new Date(at).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit', hour12: false });

  // One finished run, newest line first. The prerendered page gets fixed times; the browser
  // swaps in times from just now.
  const finishedRun = (time) =>
    SCRIPT.map((s, i) => ({ agent: s.agent, message: s.message(0), id: `p${i}`, time: time(i) })).reverse();

  let rows = $state(finishedRun((i) => `14:02:${String(10 + i * 2)}`));

  onMount(() => {
    const now = Date.now();
    rows = finishedRun((i) => clock(now - (SCRIPT.length - 1 - i) * STEP_MS));
    if (matchMedia('(prefers-reduced-motion: reduce)').matches) return;
    let run = 1;
    let n = 0;
    let timer;
    const next = () => {
      const s = SCRIPT[n];
      rows = [{ agent: s.agent, message: s.message(run), id: `r${run}-${n}`, time: clock(Date.now()) }, ...rows].slice(0, KEEP);
      n += 1;
      if (n === SCRIPT.length) (n = 0), (run += 1);
      timer = setTimeout(next, n === 0 ? REST_MS : STEP_MS);
    };
    timer = setTimeout(next, REST_MS);
    return () => clearTimeout(timer);
  });
</script>

<div class="rounded-3xl bg-night p-5 text-night-ink sm:p-6">
  <div class="flex items-center gap-2 font-mono text-xs text-night-soft">
    <span class="size-2 animate-pulse rounded-full bg-lamp"></span>
    replay of a demo run
  </div>
  <ol class="mt-4 flex h-56 flex-col gap-3 overflow-hidden font-mono text-[0.8rem] leading-snug [mask-image:linear-gradient(to_bottom,black_65%,transparent)]">
    {#each rows as e (e.id)}
      <li class="grid shrink-0 grid-cols-[4.5rem_7.5rem_1fr] gap-2 max-sm:grid-cols-[7rem_1fr]"
        in:fly={{ y: -10, duration: 700, easing: cubicOut }}
        animate:flip={{ duration: 600, easing: cubicOut }}>
        <span class="text-night-soft/70 tabular-nums max-sm:hidden">{e.time}</span>
        <span class={AGENT_COLOR[e.agent] ?? 'text-night-soft'}>{e.agent}</span>
        <span>{e.message}</span>
      </li>
    {/each}
  </ol>
</div>
