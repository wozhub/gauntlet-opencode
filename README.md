# gauntlet-opencode

Opencode port of [duolahypercho/gauntlet-loop](https://github.com/duolahypercho/gauntlet-loop),
which packaged Matt Shumer's aim prompt from
[mshumer/Claude-of-Duty](https://github.com/mshumer/Claude-of-Duty).

Fill three paragraphs — THING, REFERENCE, STACK — and either return the
prompt (compose-only, default) or execute it against **real** opencode
primitives: `task()` fan-out, `oracle` critic, `visual-qa` + `playwright`
for actual screenshot diff, `background_output` for iteration.

**Not globally installed.** Drop into a project when you want it there.

## Install (per-project)

```bash
# clone into the project's opencode skills dir
git clone https://github.com/wozhub/gauntlet-opencode \
    .opencode/skills/gauntlet-loop

# or as a submodule if you want pinned updates
git submodule add https://github.com/wozhub/gauntlet-opencode \
    .opencode/skills/gauntlet-loop
```

Restart opencode inside that project. Invoke `/gauntlet-loop`.

## Use

```text
/gauntlet-loop a Brotato-like arena survivor in Phaser
/gauntlet-loop a THREE.js FPS against COD MW
/gauntlet-loop --run a Hades-like roguelike in PixiJS
```

- **default:** returns the filled three-paragraph prompt, stops.
- **`--run`:** executes via opencode primitives (see `SKILL.md`).

## What this is *not*

- Not global — never installs to `~/.config/opencode/`.
- Not a harness — no state files, no scoreboards, no round ledgers.
- Not a stop-condition machine — the human is the brake, `/cost` is the ceiling.

## Credit

- Prompt: Matt Shumer — [Claude-of-Duty](https://github.com/mshumer/Claude-of-Duty)
- Skill packaging: [duolahypercho/gauntlet-loop](https://github.com/duolahypercho/gauntlet-loop)
- Opencode port: this repo

## License

MIT. See [LICENSE](LICENSE); upstream copyrights preserved in [NOTICE](NOTICE).
