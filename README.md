# cinematic-webgl-skill

A Claude Code skill for building award-style, scroll-driven 3D websites:
**Astro + three.js (WebGPU / TSL, with WebGL2 fallback) + Lenis + anime.js + nanostores**.

It teaches the techniques behind sites that feel "silky" on every scroll: a fixed full-screen canvas,
scroll measured in viewports, double-smoothed camera motion, a stage timeline with handover glitches,
a uniform-driven VHS/CRT post stack, GPGPU particles, depth-relit images, kinetic image cylinders,
Gaussian splats and a designed preloader.

## Live preview

See a site built with this skill: **[haxinja.com](https://haxinja.com)**

## Contents

```
cinematic-webgl-site/
|-- SKILL.md              the playbook: stack, why it feels smooth, build order, design tokens, rules
`-- references/
    `-- recipes.md        code recipes: scroll shell, damped camera, timeline, TSL tape/CRT post,
                          GPGPU particles, depth relighting, kinetic cylinders, splats, preloader
```

## Install

### Auto install (let your coding agent do it)

Paste one of these prompts into your agent.

**Claude Code**

```text
Install the skill from https://github.com/abalesluke/cinematic-webgl-skill as a global skill.
Clone it into ~/.claude/skills/cinematic-webgl-site so SKILL.md sits directly in that folder,
then confirm the skill is available.
```

**Codex**

```text
Install the skill from https://github.com/abalesluke/cinematic-webgl-skill as a global skill.
Clone it into your global skills directory as a folder named cinematic-webgl-site, so SKILL.md
and the references folder sit directly inside it, then confirm the skill is available.
```

**Any other agent that supports skills**

```text
Install https://github.com/abalesluke/cinematic-webgl-skill as a global skill named
cinematic-webgl-site in your skills directory, keeping SKILL.md and references/ together.
```

### Manual install

Clone it straight into your skills folder to use it in every project:

```bash
git clone https://github.com/abalesluke/cinematic-webgl-skill.git ~/.claude/skills/cinematic-webgl-site
```

Or clone it into a single project's `.claude/skills/cinematic-webgl-site` folder.

To update later, run `git pull` inside that folder.

### Use it

Ask for an immersive, scroll-animated or "Awwwards-style" site, or run `/cinematic-webgl-site` in Claude Code.

## Contributing

Issues and pull requests are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Contributors

[![contributors](https://contrib.rocks/image?repo=abalesluke/cinematic-webgl-skill)](https://github.com/abalesluke/cinematic-webgl-skill/graphs/contributors)
