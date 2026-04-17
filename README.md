# Lark Base — Landing Page Concept

A single-file scroll-driven landing page exploring premium section-coverage transitions. Built as a concept to probe how far you can push "feel" on the classic pinned-section-slide-up pattern.

Live: open `index.html` in any modern browser. No build step.

## What it is

Four full-viewport sections stacked in z-order inside a fixed stage. As you scroll, each successive section slides up to cover the previous one. The one being covered recedes visually (scales down, blurs, loses opacity, rounds its corners) so it reads as a card being pushed back into a stack rather than a flat layer swap.

Sections:

| # | Theme | Content |
|---|-------|---------|
| 1 | Dark  | Hero — "AI Work OS for Every Business", pill input, 3 feature cards, logo row |
| 2 | Light | Products — sidebar tabs (App Modes / Database / Views / Workflows / Dashboards / Permissions) + dashboard preview + 3 feature callouts |
| 3 | Dark  | Use cases — 2×2 grid: Sales & CRM (pipeline), PM (gantt), OKRs (progress bars), HR (candidate table) |
| 4 | Light | CTA — primary/secondary buttons + 10K teams / 100M records / 50 countries / 99.9% uptime stats |

## Design intent

The pinned-section-slide-up pattern is common (Apple, Linear, Stripe all use it) but looks flat most of the time. The goal here was to stack enough **depth + physics + choreography** signals on top of the basic `translateY` that transitions feel _heavy_ — like real material being pushed around, not sprites sliding on a plane.

**Three depth cues on every transition**

- **Scale + opacity + blur** on the receding layer — parallax recession
- **Growing top-edge shadow** on the incoming layer — implies a light direction and casts onto the still-visible portion of the receding layer
- **Corner-radius morph** (`0 → 20px` outgoing, `28 → 0` incoming) — reads as cards, not planes

**Two physics cues on the page as a whole**

- **Lenis smooth scroll** — turns the wheel/trackpad's discrete ticks into continuous motion with inertia. Changes how the _whole_ page feels, not just the transitions
- **GSAP `scrub: 1`** — animations lag scroll position by up to 1s and ease toward the target, so the transition rubber-bands instead of snapping. This is where the "weight" comes from

**One choreography cue inside each arriving section**

- Heading first → supporting copy → card grids. `y: 24 → 0` + `opacity: 0 → 1`, `power3.out`, stagger 0.04s. Starts at progress 0.15 of the transition timeline so content is mostly settled by the time the slide itself lands — avoids the "section arrives empty" feel

## Animation breakdown

Per transition (3 total: s1→s2, s2→s3, s3→s4), a single GSAP timeline scrubbed to a 1-viewport scroll range:

```
timeline position  →  0.0 ─────────────────────────── 1.0
                          ├─── outgoing recede ────────┤
                          ├─── incoming slide ─────────┤
                              ├── content stagger ──┤
                              0.15              ~0.65
```

**Outgoing** (`power1.inOut`):

| Property | From | To |
|---|---|---|
| `scale` | `1` | `0.93` |
| `opacity` | `1` | `0.55` |
| `filter` | `blur(0)` | `blur(4px)` |
| `border-radius` | `0` | `20px` |

**Incoming** (`power2.out`):

| Property | From | To |
|---|---|---|
| `translateY` | `100%` | `0` |
| `border-top-*-radius` | `28px` | `0` |
| `box-shadow` | `none` | `0 -28px 90px rgba(0,0,0,0.55)` |

The shadow casts _upward_ (negative y) so it falls on the still-visible top portion of the receding layer as the incoming section rises. It's most visible mid-transition; when the incoming section fully covers, the shadow clips off-screen (intentional).

**Incoming content** (`power3.out`, stagger 0.04s, duration 0.32s): `y: 24 → 0`, `opacity: 0 → 1`.

## Nav theme

Nav color flips at progress > 0.5 via a CSS class toggle — it runs on its own 0.55s `ease` transition rather than being scrubbed. Intentional: when you scroll fast, the nav settles to the destination cleanly instead of chopping with your scroll input. Scrubbing it alongside the slide would require `@property` declarations for animatable color custom properties; the class-swap path is simpler and reads fine.

## Tech stack

- **GSAP 3.12.5** + **ScrollTrigger** — timeline orchestration, scrub, per-section triggers
- **Lenis 1.1.18** — smooth inertial scroll
- No framework. No build step. Single HTML, CDN scripts, one `<script>` block at the bottom.

Lenis + GSAP share the same RAF ticker so they stay in lockstep:

```js
lenis.on('scroll', ScrollTrigger.update);
gsap.ticker.add((t) => lenis.raf(t * 1000));
gsap.ticker.lagSmoothing(0);
```

Total document height is 400vh — 3 transitions × 1vh of scroll each, plus 1vh of final rest. The visible viewport is always a 100vh fixed stage; sections are absolutely positioned inside it.

## Performance notes

- Each section has `will-change: transform, filter, opacity, border-radius` + `backface-visibility: hidden` to promote to its own compositor layer
- `transform: translate3d(...)` used instead of 2D for GPU layer promotion
- `filter: blur(4px)` on full-viewport layers is the most expensive piece. Fine on modern desktop (holds 60fps on M-series); will likely drop frames on older mobile. If that matters, dropping blur to 2px or removing it entirely reclaims most of the cost without losing much feel
- No scroll-linked layout reads in the hot path — Lenis handles input, GSAP handles the animation, neither reads layout per-frame

## Known trade-offs

- **Viewport-locked layout** — each section is exactly 100vh. On tall desktops (>900px) the CTA section reads sparse; on short viewports (<720px) content risks crowding. There's a narrow band where neither feels perfect
- **Scale gap on receding sections** — `scale: 0.93` leaves a ~3.5vh frame around each receding layer against body `#000`. Dark-on-black is invisible; light-on-black reads as a thin dark border (kept because it looks intentional, like a raised card). Tighten to `0.96` or flip to `transform-origin: top center` if you want clean edges
- **Nav theme lag** — can be up to 0.5s behind the transition during fast scrolls. Rare and subtle, but noticeable if you're looking
- **No `prefers-reduced-motion`** handling yet — should flip to a crossfade or instant swap when set

## Things left to explore

- **Content-aware parallax** inside each section (heading drifts slower than body, body slower than background, etc.)
- **Snap points** between sections — natural for a 4-section deck but hurts scrubbing; needs thought
- **Mobile polish** — Lenis handles touch but the blur + fixed-stage combo can chug on older phones; worth gating or degrading
- **Reduced-motion fallback** — see above

## File layout

```
.
└── index.html   # single-file concept — styles, markup, and script all inline
```

That's the whole thing.
