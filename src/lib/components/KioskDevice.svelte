<script>
  // A drawing of the kiosk around a live demo of its screen (/kiosk in an iframe): two LEDs that
  // run the real kiosk's patterns, and the big button. Press it, or click the screen; hold to sell.
  //
  // Until someone tries it, a strip of tape on the bezel says "press me!" and the button pings.
  import { onMount } from 'svelte';
  import { fade } from 'svelte/transition';
  import { createLeds } from '$lib/kiosk/leds.js';

  let figure;
  let frame;
  let pressed = $state(false);
  let d5 = $state(0);
  let d16 = $state(0);
  let shown = $state(false); // scrolled into view: the tape drops on and the ping starts
  let used = $state(false); // pressed, or the screen clicked: the ping stops

  const send = (msg) => frame?.contentWindow?.postMessage({ rehomeKiosk: true, ...msg }, location.origin);

  function press(e) {
    e?.preventDefault();
    if (pressed) return;
    pressed = used = true;
    send({ type: 'button', pressed: true });
  }

  function release() {
    if (!pressed) return;
    pressed = false;
    send({ type: 'button', pressed: false });
  }

  function onkeydown(e) {
    if ((e.key === ' ' || e.key === 'Enter') && !e.repeat) press(e);
  }

  function onkeyup(e) {
    if (e.key === ' ' || e.key === 'Enter') release();
  }

  onMount(() => {
    const leds = createLeds(([a, b]) => ((d5 = a), (d16 = b)));
    const onMessage = (e) => {
      if (e.origin !== location.origin || e.source !== frame?.contentWindow || !e.data?.rehomeKiosk) return;
      if (e.data.type === 'leds') leds.set(e.data.pattern, e.data.then);
      if (e.data.type === 'leds' && e.data.pattern === 'tick') used = true; // a countdown: the screen was clicked
    };
    addEventListener('message', onMessage);
    // Pause the video and its ASCII rendering while the kiosk is off screen
    const seen = new IntersectionObserver(([entry]) => send({ type: 'visible', visible: entry.isIntersecting }));
    seen.observe(frame);
    const reveal = new IntersectionObserver(([entry]) => entry.isIntersecting && ((shown = true), reveal.disconnect()), { threshold: 0.6 });
    reveal.observe(figure);
    return () => {
      leds.destroy();
      seen.disconnect();
      reveal.disconnect();
      removeEventListener('message', onMessage);
    };
  });

  // An LED: dark plastic when off, its color and a soft glow as it brightens
  const led = (level, rgb) =>
    `background: color-mix(in oklab, rgb(${rgb}) ${Math.round(level * 100)}%, #3a2f40); ` +
    `box-shadow: 0 0 ${(level * 18).toFixed(1)}px ${(level * 3).toFixed(1)}px rgb(${rgb} / ${(level * 0.7).toFixed(2)})`;
</script>

<figure bind:this={figure} class="mx-auto w-68 max-w-full"><!-- a set width, so what sits on the bezel can't resize it -->
  <div class="rounded-[2.2rem] bg-night p-3 pb-0 shadow-[0_40px_60px_-35px_rgb(29_21_34/0.75)] ring-1 ring-black/20">
    <div class="relative aspect-[480/800] overflow-hidden rounded-[1.5rem] bg-[#141017] ring-1 ring-white/5">
      <iframe bind:this={frame} src="/kiosk" title="ReHome kiosk demo" loading="lazy"
        class="absolute inset-0 size-full border-0"></iframe>
    </div>
    <div class="flex items-center gap-1 px-4 py-5">
      <div class="mr-auto flex items-center gap-3" aria-hidden="true">
        <span class="size-3 rounded-full" style={led(d5, '92 214 120')}></span><!-- D5 was a green LED -->
        <span class="size-3 rounded-full" style={led(d16, '224 92 72')}></span>
      </div>
      <!-- A hand-written label on masking tape, the way you'd mark up a demo at an expo table -->
      <span class="drop-shadow-[0_2px_1.5px_rgb(0_0_0/0.4)] transition delay-300 duration-500 ease-[cubic-bezier(0.34,1.56,0.64,1)] motion-reduce:transition-none
        {shown ? '-rotate-5 opacity-100' : '-translate-y-3 -rotate-14 scale-125 opacity-0'}" aria-hidden="true">
        <span class="tape block px-2 py-1 font-display text-xs font-semibold whitespace-nowrap text-night">press me!</span>
      </span>
      <svg viewBox="0 0 22 12" class="w-5 shrink-0 overflow-visible text-lamp" fill="none" aria-hidden="true">
        <path d="M1 8 C 6 3, 13 2, 20 6 M17.7 2.1 L20 6 L15.5 6" pathLength="1" stroke="currentColor" stroke-width="1.8"
          stroke-linecap="round" stroke-linejoin="round" stroke-dasharray="1"
          class="transition-[stroke-dashoffset] delay-[900ms] duration-700 ease-out motion-reduce:transition-none"
          style:stroke-dashoffset={shown ? 0 : 1} />
      </svg>
      <button type="button" aria-label="The kiosk's button: press it, or hold it to sell instead"
        class="group relative grid size-16 cursor-pointer touch-none place-items-center rounded-full bg-[#2a2130] ring-1 ring-white/10 select-none"
        onpointerdown={press} onpointerup={release} onpointerleave={release} onpointercancel={release}
        {onkeydown} {onkeyup} oncontextmenu={(e) => e.preventDefault()}>
        {#if shown && !used}
          <span class="ping pointer-events-none absolute inset-2.5 rounded-full ring-2 ring-lamp motion-reduce:hidden" out:fade={{ duration: 300 }}></span>
        {/if}
        <span class="size-11 rounded-full bg-lamp shadow-[inset_0_-0.3rem_0_rgb(0_0_0/0.25)] transition-[transform,translate,filter,box-shadow] duration-150 group-hover:-translate-y-px group-hover:brightness-110 group-hover:shadow-[inset_0_-0.3rem_0_rgb(0_0_0/0.25),0_0_22px_4px_rgb(242_169_59/0.5)]"
          style:transform={pressed ? 'translateY(2px) scale(0.94)' : 'none'}></span>
      </button>
    </div>
  </div>
  <figcaption class="mt-4 text-center font-mono text-xs text-ink-soft">press the button · hold it to sell</figcaption>
</figure>

<style>
  /* Masking tape: warm paper with torn, zigzag ends */
  .tape {
    background: linear-gradient(rgb(255 255 255 / 0.14), transparent 40%, rgb(0 0 0 / 0.05)), #eadbb9;
    clip-path: polygon(4% 0, 96% 0, 100% 18%, 96.5% 36%, 100% 54%, 96% 72%, 99.5% 88%, 96% 100%, 4% 100%, 0.5% 84%, 4% 66%, 0 48%, 3.5% 30%, 0 14%);
  }

  /* A slow ring out from the button, like a ripple */
  .ping {
    animation: ping 2.6s cubic-bezier(0.2, 0.6, 0.35, 1) infinite;
  }
  @keyframes ping {
    0% { transform: scale(1); opacity: 0.7; }
    75%, 100% { transform: scale(1.75); opacity: 0; }
  }
</style>
