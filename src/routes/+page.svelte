<script lang="ts">
import gsap from 'gsap';
import { onMount } from 'svelte';

const WEEKS_PER_YEAR = 52;
const LIFE_YEARS = 80;

const colors = [
    '#0d5c63',
    '#1b6e73',
    '#297f82',
    '#379091',
    '#44a1a0',
    '#51acae',
    '#5eb7bc',
    '#6bc2ca',
    '#78cdd7',
];

const categories = [
    { id: 'sleep', label: 'Sleep', color: colors[3], hoursPerDay: 7 },
    { id: 'work', label: 'Work / School', color: colors[4], hoursPerDay: 8 },
    { id: 'commute', label: 'Commute', color: colors[5], hoursPerDay: 1.5 },
    { id: 'chores', label: 'Chores & Errands', color: colors[6], hoursPerDay: 1.5 },
    { id: 'fitness', label: 'Fitness & Health', color: colors[7], hoursPerDay: 0.5 },
    { id: 'doomscroll', label: 'Doomscrolling', color: '#78c8cf', hoursPerDay: 2.5 },
] as const;

// biome-ignore lint/style/useConst: Svelte 5 $state requires let
let age = $state(25);
// biome-ignore lint/style/useConst: Svelte 5 $state requires let
let isVertical = $state(false);
// biome-ignore lint/style/useConst: Svelte 5 $state requires let
let showScrollIndicator = $state(true);

function weeksToYears(w: number) {
    return (w / WEEKS_PER_YEAR).toFixed(1);
}

function hoursToYears(h: number) {
    return (h / (24 * 365.25)).toFixed(1);
}

function getCatWeeks(cat: (typeof categories)[number]) {
    const remaining = Math.max(0, LIFE_YEARS - age);
    return Math.round((Math.min(cat.hoursPerDay, 24) / 24) * WEEKS_PER_YEAR * remaining);
}

function getFreeWeeks() {
    let used = 0;
    for (const c of categories) used += getCatWeeks(c);
    return Math.max(0, (LIFE_YEARS - age) * WEEKS_PER_YEAR - used);
}

function getPercent(cat: (typeof categories)[number]) {
    const rem = (LIFE_YEARS - age) * WEEKS_PER_YEAR;
    return rem === 0 ? 0 : Math.round((getCatWeeks(cat) / rem) * 100);
}

function getFreePercent() {
    const rem = (LIFE_YEARS - age) * WEEKS_PER_YEAR;
    return rem === 0 ? 0 : Math.round((getFreeWeeks() / rem) * 100);
}

function blockPos(idx: number): number {
    let pos = 0;
    for (let i = 0; i < idx; i++) pos += getPercent(categories[i]);
    return pos;
}

function checkVertical() {
    isVertical = window.innerWidth < 768 || window.innerHeight > window.innerWidth * 1.3;
}

let cur = -1;
let busy = false;
let lastTime = 0;
let accum = 0;
let lastScroll = 0;

function animProps() {
    if (isVertical) {
        return {
            exit: (fwd: boolean) => ({ y: fwd ? '-100%' : '100%' }),
            enter: (fwd: boolean) => ({ y: fwd ? '100%' : '-100%' }),
            textExit: (fwd: boolean) => ({ opacity: 0, y: fwd ? -30 : 30 }),
            textEnter: { opacity: 1, y: 0 },
            fillSize: 'height',
            fillStart: '0%',
            fillAxis: 'y',
        };
    }
    return {
        exit: (fwd: boolean) => ({ x: fwd ? '-100%' : '100%' }),
        enter: (fwd: boolean) => ({ x: fwd ? '100%' : '-100%' }),
        textExit: (fwd: boolean) => ({ opacity: 0, x: fwd ? -40 : 40 }),
        textEnter: { opacity: 1, x: 0 },
        fillSize: 'width',
        fillStart: '0%',
        fillAxis: 'x',
    };
}

function goTo(index: number) {
    const hero = document.getElementById('hero');
    const secs = document.querySelectorAll<HTMLElement>('.txt-layer');
    const fills = document.querySelectorAll<HTMLElement>('.fill-bar .fill');
    const freeFill = document.querySelector<HTMLElement>('.fill-bar .free-fill');
    const freeIdx = secs.length - 2;

    if (index < -1 || index >= secs.length || busy) return;
    showScrollIndicator = false;
    const now = Date.now();
    if (now - lastTime < 1400) return;
    busy = true;
    lastTime = now;

    const fwd = index > cur;
    const a = animProps();
    const exitPos = a.exit(fwd);
    const enterPos = a.enter(fwd);

    // Outgoing
    if (cur === -1 && hero) {
        gsap.to(hero, { ...exitPos, opacity: 0, duration: 0.8, ease: 'power3.inOut' });
    }
    if (cur >= 0 && cur < secs.length) {
        const s = secs[cur];
        gsap.to(s.querySelectorAll('.txt > *'), {
            ...a.textExit(fwd),
            stagger: 0.03,
            duration: 0.3,
            ease: 'power2.in',
        });
        gsap.to(s, { ...exitPos, duration: 0.7, ease: 'power3.inOut', delay: 0.15 });
    }

    cur = index;

    // Reset fills after current position
    for (let i = index + 1; i < fills.length; i++) {
        gsap.to(fills[i], { [a.fillSize]: '0%', duration: 0.3 });
    }
    if (index <= freeIdx && freeFill) {
        gsap.to(freeFill, { [a.fillSize]: '0%', duration: 0.3 });
    }

    // Incoming
    if (index === -1 && hero) {
        gsap.set(hero, { ...enterPos, opacity: 0 });
        gsap.to(hero, {
            [a.fillAxis]: '0%',
            opacity: 1,
            duration: 0.8,
            ease: 'power3.inOut',
            delay: 0.3,
        });
    }
    if (index >= 0 && index < secs.length) {
        const s = secs[index];
        gsap.set(s, enterPos);
        const tl = gsap.timeline({ delay: 0.3 });
        tl.to(s, { [a.fillAxis]: '0%', duration: 0.9, ease: 'power3.inOut' });
        tl.to(
            s.querySelectorAll('.txt > *'),
            { ...a.textEnter, stagger: 0.08, duration: 0.6, ease: 'power2.out' },
            '-=0.5'
        );

        const fill = fills[index];
        if (fill) {
            if (fwd) {
                gsap.set(fill, { [a.fillSize]: '0%' });
                tl.to(
                    fill,
                    { [a.fillSize]: fill.dataset.pct || '0%', duration: 1.0, ease: 'power2.out' },
                    '-=0.7'
                );
            } else {
                gsap.set(fill, { [a.fillSize]: fill.dataset.pct || '0%' });
            }
        } else if (index === freeIdx && freeFill) {
            if (fwd) {
                gsap.set(freeFill, { [a.fillSize]: '0%' });
                tl.to(
                    freeFill,
                    {
                        [a.fillSize]: freeFill.dataset.pct || '0%',
                        duration: 1.0,
                        ease: 'power2.out',
                    },
                    '-=0.7'
                );
            } else {
                gsap.set(freeFill, { [a.fillSize]: freeFill.dataset.pct || '0%' });
            }
        }
    }

    setTimeout(() => {
        busy = false;
    }, 1400);
}

function onWheel(e: WheelEvent) {
    e.preventDefault();
    const now = Date.now();
    if (now - lastScroll > 400) accum = 0;
    accum += Math.abs(e.deltaY);
    lastScroll = now;
    if (accum >= 2) {
        accum = 0;
        goTo(cur + (e.deltaY > 0 ? 1 : -1));
    }
}

function onKey(e: KeyboardEvent) {
    if (['ArrowDown', ' ', 'PageDown'].includes(e.key)) {
        e.preventDefault();
        goTo(cur + 1);
    }
    if (['ArrowUp', 'PageUp'].includes(e.key)) {
        e.preventDefault();
        goTo(cur - 1);
    }
}

let ty0 = 0;
function onTouchStart(e: TouchEvent) {
    e.preventDefault();
    ty0 = e.touches[0].clientY;
}
function onTouchEnd(e: TouchEvent) {
    e.preventDefault();
    const d = ty0 - e.changedTouches[0].clientY;
    if (Math.abs(d) > 40) goTo(cur + (d > 0 ? 1 : -1));
}

onMount(() => {
    checkVertical();
    window.addEventListener('resize', checkVertical);

    const secs = document.querySelectorAll<HTMLElement>('.txt-layer');
    const axis = isVertical ? 'y' : 'x';
    for (const s of secs) {
        gsap.set(s, { [axis]: '100%' });
        gsap.set(s.querySelectorAll('.txt > *'), { opacity: 0, [axis]: 30 });
    }

    const fills = document.querySelectorAll<HTMLElement>('.fill-bar .fill');
    const fillProp = isVertical ? 'height' : 'width';
    for (const f of fills) gsap.set(f, { [fillProp]: '0%' });
    const freeFill = document.querySelector<HTMLElement>('.fill-bar .free-fill');
    if (freeFill) gsap.set(freeFill, { [fillProp]: '0%' });

    const free = document.getElementById('free');
    if (free) gsap.set(free.querySelectorAll('.txt > *'), { opacity: 0, [axis]: 30 });
    const outro = document.getElementById('outro');
    if (outro) gsap.set(outro.querySelectorAll('.txt > *'), { opacity: 0, [axis]: 30 });

    gsap.from('.hero-title', { y: 60, opacity: 0, duration: 1.2, ease: 'power3.out', delay: 0.2 });
    gsap.from('.hero-sub', { y: 40, opacity: 0, duration: 1, ease: 'power3.out', delay: 0.5 });
    gsap.from('.age-picker', { y: 30, opacity: 0, duration: 0.8, ease: 'power3.out', delay: 0.8 });

    window.addEventListener('wheel', onWheel, { passive: false });
    window.addEventListener('keydown', onKey);
    window.addEventListener('touchstart', onTouchStart, { passive: false });
    window.addEventListener('touchend', onTouchEnd, { passive: false });

    return () => {
        window.removeEventListener('wheel', onWheel);
        window.removeEventListener('keydown', onKey);
        window.removeEventListener('touchstart', onTouchStart);
        window.removeEventListener('touchend', onTouchEnd);
        window.removeEventListener('resize', checkVertical);
    };
});
</script>

<svelte:head>
    <title>Where Does Your Life Go?</title>
</svelte:head>

<main class:vertical={isVertical}>
    <!-- Persistent fill bar -->
    <div class="fill-bar" class:vertical={isVertical}>
        {#each categories as cat, i}
            <div
                class="fill"
                style="background:{cat.color}; {isVertical ? 'bottom' : 'left'}:{blockPos(i)}%; {isVertical ? 'height' : 'width'}:0%"
                data-pct="{getPercent(cat)}%"
            ></div>
        {/each}
        <div
            class="fill free-fill"
            style="background:#ffffff; {isVertical ? 'bottom' : 'left'}:{blockPos(categories.length)}%; {isVertical ? 'height' : 'width'}:0%"
            data-pct="{100 - blockPos(categories.length)}%"
        ></div>
    </div>

    <!-- Hero -->
    <section class="hero" id="hero">
        <div class="hero-content">
            <p class="hero-kicker">A visualization of time</p>
            <h1 class="hero-title">
                Where Does Your
                <br>
                Life Go?
            </h1>
            <p class="hero-sub">
                You have roughly 4,000 weeks to live.
                <br>
                Here's where they actually go.
            </p>
            <div class="age-picker">
                <label for="age-input">I am</label>
                <input id="age-input" type="number" bind:value={age} min="0" max="80">
                <span>years old</span>
            </div>
        </div>
        {#if showScrollIndicator}
            <div
                class="scroll-indicator"
                role="button"
                tabindex="0"
                onclick={() => goTo(cur + 1)}
                onkeydown={(e) => { if (e.key === 'Enter' || e.key === ' ') goTo(cur + 1); }}
            >
                {#if isVertical}
                    <svg
                        width="28"
                        height="28"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="#eee"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <line x1="12" y1="5" x2="12" y2="19"></line>
                        <polyline points="19 12 12 19 5 12"></polyline>
                    </svg>
                {:else}
                    <svg
                        width="28"
                        height="28"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="#eee"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <line x1="5" y1="12" x2="19" y2="12"></line>
                        <polyline points="12 5 19 12 12 19"></polyline>
                    </svg>
                {/if}
            </div>
        {/if}
    </section>

    <!-- Category sections -->
    {#each categories as cat, i}
        <section class="sec txt-layer" id={cat.id}>
            <div class="txt">
                <span class="num">0{i + 1}</span>
                <h2 class="label">{cat.label}</h2>
                <div class="big">{getCatWeeks(cat).toLocaleString()}</div>
                <div class="unit">weeks</div>
                <div class="sub">
                    {hoursToYears(getCatWeeks(cat) * 24 * 7)}
                    years &middot; {getPercent(cat)}% of remaining life
                </div>
                <p class="desc">
                    {#if cat.id === 'sleep'}
                        About {cat.hoursPerDay} hours a night. A third of your life, spent
                        unconscious.
                    {:else if cat.id === 'work'}
                        {cat.hoursPerDay}
                        hours a day, five days a week. The bulk of your waking life, spent making
                        someone else money.
                    {:else if cat.id === 'commute'}
                        {cat.hoursPerDay}
                        hours a day sitting in traffic or on a train. That's
                        {hoursToYears(getCatWeeks(cat) * 24 * 7)}
                        years of your life, spent going nowhere.
                    {:else if cat.id === 'chores'}
                        Cooking, cleaning, laundry, groceries, dishes. The invisible tax of being
                        alive.
                    {:else if cat.id === 'fitness'}
                        If you're lucky. Most people get less.
                    {:else}
                        Scrolling. Tapping. Refreshing. The hours vanish and you can't remember a
                        single thing you saw.
                    {/if}
                </p>
            </div>
        </section>
    {/each}

    <!-- Free Time -->
    <section class="sec txt-layer" id="free">
        <div class="txt free-txt">
            <h2 class="label">Actual Free Time</h2>
            <div class="big free-num">{getFreeWeeks().toLocaleString()}</div>
            <div class="unit">weeks</div>
            <div class="sub">
                {weeksToYears(getFreeWeeks())}
                years &middot; {getFreePercent()}% of remaining life
            </div>
            <p class="desc free-desc">
                {#if getFreeWeeks() > 0}
                    For everything. Hobbies, friends, travel, love, rest, building things, thinking,
                    staring at the ceiling.
                {:else}
                    The math doesn't look good.
                {/if}
            </p>
        </div>
    </section>

    <!-- Outro -->
    <section class="sec txt-layer" id="outro">
        <div class="txt outro-txt">
            <p class="outro-text">
                This isn't meant to be depressing. It's a reminder.
                <br>
                The time is going to pass anyway.
            </p>
        </div>
    </section>
</main>

<style>
main {
    background: #07070a;
    color: #eee;
    font-family: "Inter", sans-serif;
    font-weight: 300;
    line-height: 1.6;
    overflow: hidden;
    height: 100vh;
    height: 100dvh;
}

/* ── Fill bar ── */
.fill-bar {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 100vh;
    height: 100dvh;
    z-index: 0;
}

.fill-bar .fill {
    position: absolute;
    top: 0;
    bottom: 0;
    will-change: width;
}

/* Vertical fill bar */
.fill-bar.vertical {
    left: 0;
    right: auto;
    width: 100vw;
    height: 100vh;
    height: 100dvh;
}

.fill-bar.vertical .fill {
    top: auto;
    bottom: 0;
    left: 0;
    right: 0;
    will-change: height;
}

/* ── Hero ── */
.hero {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 2;
}

.hero-content {
    position: relative;
    z-index: 1;
    text-align: center;
    padding: 2rem;
}

.hero-kicker {
    font-family: "JetBrains Mono", monospace;
    font-size: 0.75rem;
    text-transform: uppercase;
    letter-spacing: 0.25em;
    color: #5eb7bc;
    margin-bottom: 2rem;
}

.hero-title {
    font-family: "Newsreader", serif;
    font-weight: 300;
    font-size: clamp(3rem, 10vw, 8rem);
    line-height: 0.95;
    letter-spacing: -0.03em;
    margin: 0 0 2.5rem;
}

.hero-sub {
    font-size: clamp(1rem, 2vw, 1.3rem);
    color: #777;
    max-width: 460px;
    margin: 0 auto 4rem;
}

.age-picker {
    display: inline-flex;
    align-items: baseline;
    gap: 0.6rem;
    font-size: 1.15rem;
    color: #555;
}

.age-picker input {
    width: 52px;
    background: transparent;
    border: none;
    border-bottom: 1px solid #444;
    color: #eee;
    font-family: "Newsreader", serif;
    font-size: 1.6rem;
    text-align: center;
    outline: none;
    padding: 0.2rem;
    transition: border-color 0.3s;
    -moz-appearance: textfield;
}

.age-picker input::-webkit-outer-spin-button,
.age-picker input::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
}

.age-picker input:hover {
    border-color: #666;
}

.age-picker input:focus {
    border-color: #5eb7bc;
    background: rgba(255, 255, 255, 0.06);
}

/* ── Scroll indicator ── */
.scroll-indicator {
    position: absolute;
    bottom: 2rem;
    left: 50%;
    transform: translateX(-50%);
    z-index: 3;
    animation: bounce 2s infinite;
    opacity: 0.5;
    cursor: pointer;
    transition: opacity 0.2s;
}

.scroll-indicator:hover {
    opacity: 1;
}

@keyframes bounce {
    0%,
    20%,
    50%,
    80%,
    100% {
        transform: translateX(-50%) translateY(0);
    }
    40% {
        transform: translateX(-50%) translateY(-8px);
    }
    60% {
        transform: translateX(-50%) translateY(-4px);
    }
}

/* ── Sections ── */
.sec {
    position: absolute;
    inset: 0;
    overflow: hidden;
    z-index: 1;
    background: transparent;
}

.txt {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    justify-content: center;
    padding: 4rem;
    z-index: 1;
}

.num {
    font-family: "Newsreader", serif;
    font-size: 0.9rem;
    color: #eee;
    opacity: 0.5;
}

.label {
    font-family: "Fraunces", serif;
    font-weight: 300;
    font-size: clamp(2.5rem, 6vw, 5rem);
    line-height: 1.05;
    margin: 0.4rem 0 2.5rem;
    color: #eee;
}

.big {
    font-family: "Newsreader", serif;
    font-size: clamp(3.5rem, 10vw, 7rem);
    font-weight: 300;
    line-height: 1;
    color: #eee;
}

.unit {
    font-family: "JetBrains Mono", monospace;
    font-size: 0.75rem;
    text-transform: uppercase;
    letter-spacing: 0.15em;
    color: #eee;
    opacity: 0.5;
    margin-top: 0.25rem;
}

.sub {
    font-size: 0.95rem;
    color: #eee;
    opacity: 0.6;
    margin-top: 0.75rem;
}

.desc {
    font-size: 1.05rem;
    color: #eee;
    opacity: 0.7;
    line-height: 1.7;
    max-width: 480px;
}

/* ── Free section ── */
.free-txt {
    text-align: center;
    align-items: center;
}

.free-txt .num,
.free-txt .label,
.free-txt .big,
.free-txt .unit,
.free-txt .sub,
.free-num,
.free-desc,
.free-desc :global(strong) {
    color: #eee;
}

.free-desc {
    opacity: 0.8;
    max-width: 500px;
}

.free-desc :global(strong) {
    font-weight: 400;
}

/* ── Outro ── */
.outro-txt {
    text-align: center;
    align-items: center;
}

.outro-text {
    font-family: "Newsreader", serif;
    font-style: italic;
    font-size: 1.3rem;
    color: #ccc;
    max-width: 480px;
    margin: 0 auto 4rem;
    line-height: 1.8;
}

/* ── Mobile / vertical adjustments ── */
@media (max-width: 768px) {
    .txt {
        padding: 2rem 1.5rem;
    }
    .hero-title {
        font-size: clamp(2.5rem, 12vw, 5rem);
    }
    .label {
        font-size: clamp(2rem, 8vw, 3.5rem);
    }
    .big {
        font-size: clamp(2.5rem, 12vw, 5rem);
    }
}
</style>
