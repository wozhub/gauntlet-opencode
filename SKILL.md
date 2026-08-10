---
name: gauntlet-loop
description: >-
  Aim-prompt skill for opencode. Fill THING / REFERENCE / STACK into Matt
  Shumer's three-paragraph build-against-a-reference prompt. Default: return
  the filled prompt (compose only). With `--run`: execute via real opencode
  primitives — task() fan-out, oracle critic, visual-qa + playwright for
  actual screenshot diff, background_output for iteration. Human is the
  brake; /cost is the ceiling. Per-project skill; never installs globally.
  Triggers: /gauntlet-loop, gauntlet loop, aim prompt, build against
  reference, build like <game>, ship at AAA quality, Shumer prompt.
---

# Gauntlet Loop (opencode)

Matt Shumer's aim prompt, wired to real opencode tools. Port of
[duolahypercho/gauntlet-loop](https://github.com/duolahypercho/gauntlet-loop).

## Default: compose only

Return the filled three paragraphs in a fenced `text` block and **stop**.
Do not execute. Do not spawn subagents. The user runs it if they want.

This is the default because opencode's implementation gate (`AGENTS.md`)
requires an explicit run verb per turn. `--run` is that verb here.

## On `--run`: execute against real primitives

Only when the user's current message contains `--run` (or "run it",
"execute", "go"). Never invented, never carried over from a previous turn.

### Cycle (one round per user "continue")

1. **Fill nouns.** Infer THING / REFERENCE / LOOK / TIER / AREA_1 / AREA_2 /
   CHECK / STACK from args + cwd. One question max if THING is missing.
2. **Build slice.** If no playable slice exists yet, delegate one:
   ```
   task(
     category="visual-engineering",
     load_skills=["frontend"],
     prompt="Build minimum playable [THING] in [STACK]. Real render loop,
             input, one enemy/obstacle, one win/lose signal. No stubs.
             Return the entrypoint path and run command.")
   ```
3. **Fan out workstreams** — 2 to 4 parallel tasks, one per area
   (AREA_1, AREA_2, plus juice/audio if warranted). All background:
   ```
   task(category="visual-engineering", load_skills=["frontend"],
        run_in_background=true, prompt="Push [AREA] toward [REFERENCE] level.
        Constraint: game stays playable, no capture farms, no headless CPU
        pegging. Deliver a diff, not a rewrite.")
   ```
4. **Wait.** End response. Collect on `<system-reminder>` via
   `background_output(task_id="bg_...")`. Do not poll.
 5. **Screenshot the game — TWO frames, always.** Use `skill("playwright")`
    for web stacks (THREE.js, Phaser, PixiJS, Babylon). For native (Godot,
    Unity): `interactive_bash` a real window and a one-shot screenshot tool.
    Two light in-game frames — **not** a capture harness:
    - `./.gauntlet/frame-<cycle>.png` — default/wide framing. Reviews
      composition, HUD, palette, whole-scene read.
    - `./.gauntlet/frame-<cycle>-close.png` — zoomed in on the densest
      cluster of the thing being polished. Reviews detail.

    **Both are mandatory.** A single wide frame cannot distinguish "not
    built" from "built and invisible" — verified in the guerra-chica field
    trial, where cycles 4, 5 and 6 each shipped unit and building detail
    that was sub-pixel in the only frame the critic ever saw, so the critic
    correctly kept re-requesting work that already existed. Three cycles.
    If the app has no zoom, add the smallest possible debug hook
    (`window.__<app>.camera.setZoom/setTarget`) — that is a product
    feature, not a harness.
6. **Visual gap report — parent model reads the frame directly.**
   **DO NOT** delegate to `task(subagent_type="multimodal-looker")` — that
   subagent is routed to Haiku 4.5 in the current opencode config and
   hallucinates ~25% of findings on visual detail (verified in the
   guerra-chica cycle-1 field trial: 5 of 19 gaps were fabricated —
   claimed missing features that clearly shipped). Instead:
    1. `read` **both** `./.gauntlet/frame-<cycle>.png` and
       `./.gauntlet/frame-<cycle>-close.png` — attaches the wide and the
       detail frame to your (parent) context as images.
    2. `read` 3–5 stills from `./reference/*.{png,jpg}` — attaches the
       target aesthetic images to your context.
    3. Do the compare **yourself**. Output a plain-text gap list, one
       line per gap in format `AREA: what is wrong`, 8–20 gaps, no
       praise, no softening. `AREA` ∈ {palette, material, silhouette,
       faction-color, terrain, camera, hud, shadow, scale, background,
       other}.
    4. **Report two numbers, not impressions:** px per grid cell and px
       per subject silhouette, in the wide frame, against the same two
       numbers measured off the reference. These make a framing
       regression self-evident on cycle 1 instead of cycle 7.

   **Requires the parent to be image-capable.** If it isn't, this step
   cannot run correctly — stop the cycle and tell the human. Falling
   back to multimodal-looker is worse than stopping, because a
   confidently-wrong gap list poisons step 7 (Oracle plans fixes for
   phantom problems).
 7. **Critic reasoning.** `task(subagent_type="oracle", ...)` — takes the
    parent's textual gap list **plus** the game's source paths, returns a
    prioritised, minimal-change fix plan. Oracle blocks the final answer —
    wait for it.

    **Framing gate — two rules, cheap, and they save whole cycles:**
    - No detail item may be proposed for a feature that is not measurably
      visible in the close capture.
    - No detail item may be proposed at all while a framing or scale item
      is still open. Fix what the frame shows before adding to it.
8. **Report + brake.** One paragraph: what shipped, top gaps, top fixes,
   current `/cost`. Ask the user to say "continue" or stop. **Do not
   auto-loop.** This is the opencode-specific brake — the original said
   "never ask continue"; here we must, because opencode's gate is per-turn.

### The prompt template (Shumer, unchanged skeleton)

```text
I want you to build [THING] at the level of [REFERENCE]. It should be
utterly perfect, [LOOK], with every single thing done at [TIER] quality,
from [AREA_1] to [AREA_2] to anything you could think of.

Fan out sub-agents and have sub-agents tackle each one individually so
that the [THING] is utterly perfect. Have a separate sub-agent check it
[CHECK] to ensure it is [TIER]. That separate sub-agent should be a
really harsh critic, and if it isn't [TIER], it should keep going.

Don't stop until each sub-agent is utterly wowed with the quality when
compared with [REFERENCE]. It should literally compare them side by
side blind and say which one looks better. Do this in [STACK]. Fan out
sub-agents.
```

### Noun defaults (games)

| Slot | Default |
|---|---|
| `THING` | a game (FPS, roguelike, arena survivor, …) |
| `REFERENCE` | a real shipped game (Call of Duty, Hades, Brotato, …) |
| `LOOK` | `visually beautiful` |
| `TIER` | `AAA` |
| `CHECK` | `visually` |
| `STACK` | from args → project → THREE.js / Phaser / PixiJS by genre |
| `AREA_1` / `AREA_2` | genre workstreams (textures/physics, combat feel/lighting, sprite readability/juice) |

If the model can obviously beat `REFERENCE` on day one, pick a harder one.

## Rhetoric → opencode primitive

| Original phrase | Real tool in opencode |
|---|---|
| "fan out sub-agents" | `task(..., run_in_background=true)` × 2–4 |
| "parent sees the frame" | parent model `read`s **both** frames (wide + close) + `./reference/*.{png,jpg}` — attaches images to context. **Do NOT** use `multimodal-looker` (downgraded to Haiku 4.5, hallucinates on visual detail) |
| "separate harsh critic" | `task(subagent_type="oracle", ...)` — reasons about fixes from the gap report |
| "blind compare side by side" | `skill("playwright")` wide + close screenshots, read by the parent against `./reference/*.{png,jpg}` |
| "/loop until utterly perfect" | user says "continue"; `background_output()` collects |
| "ultracode" | (dead token — dropped) |
| "you are the brake" | `/cost` + explicit user "continue" per cycle |

## Do not invent

- Helper scripts, capture harnesses, critic contracts, scoreboards
- `GAUNTLET_STATE.md` / round ledgers / architecture contracts as the job
- Stop rules ("N flat rounds", "good enough", "ready for review")
- Softening the critic or lowering the reference
- Auto-continuing without the user — opencode's gate forbids it
- Capture farms / headless engine loops that peg CPU for screenshots
- Endless image-gen or Blender-only rounds that never land in the build
- Turning this into a marketing-site skill by default — games first

## Do not do (opencode-specific)

- Do not install this skill globally. It lives per-project only.
- Do not fire the critic and the workstreams in the same wave — one wave
  builds, next wave critiques. Opencode's one-exploration-wave rule.
- Do not skip the cost line. `/cost` is the brake.
- Do not treat Oracle's return as advisory — its return blocks final answer
  per your `AGENTS.md`.

## Compose only (explicit)

If the user says "compose only" / "just the prompt" / "don't run": return
the filled three paragraphs in a fenced `text` block and stop.

## Credit

- Prompt: [Matt Shumer](https://github.com/mshumer/Claude-of-Duty) — MIT
- Skill packaging: [duolahypercho/gauntlet-loop](https://github.com/duolahypercho/gauntlet-loop) — MIT
- This opencode port: MIT, upstream copyrights preserved in `NOTICE`

Fills: [examples.md](examples.md).
