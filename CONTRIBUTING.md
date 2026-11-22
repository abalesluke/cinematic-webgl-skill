# Contributing

Thanks for helping improve the cinematic-webgl-site skill. Bug reports, new recipes, fixes for
API changes and better explanations are all welcome.

## Ways to contribute

- **Report a problem:** open an issue if a recipe no longer works with a newer three.js, Lenis or
  Astro release, or if the skill steers an agent in the wrong direction. Include the library
  versions, what you asked the agent for, and what happened.
- **Suggest a technique:** open an issue describing the effect, where it is used, and a link to a
  public tutorial or demo that explains it.
- **Send a pull request:** fixes, new recipes and clearer wording.

## Repository layout

```
SKILL.md              the playbook the agent reads first
references/
  recipes.md          code recipes the playbook points to
README.md             overview, install and attribution
```

Keep `SKILL.md` and `references/` at the repository root so the repo can be cloned directly into a
skills folder.

## Guidelines

### Keep the skill focused
- `SKILL.md` is loaded into the agent's context, so keep it short: principles, stack, build order
  and rules. Put long code in `references/recipes.md`.
- Add a recipe only when it solves a real need for scroll-driven WebGL sites.
- Update the frontmatter `description` in `SKILL.md` if you add a capability the agent should
  trigger on.

### Make recipes correct
- Test every snippet against the library version you name. Mention the version when an API is new
  or has changed (for example `RenderPipeline` vs `PostProcessing` in three.js).
- Animate effects through uniforms, never through shader recompiles, and keep the damping and
  reduced-motion rules from `SKILL.md`.
- Prefer small, readable snippets over full apps.

### Respect other people's work
- Credit the source of every technique you adapt: author, article title, and a link to the post
  or repository. Add it inline in the recipe.
- Do not copy code, shaders, 3D scans, models, images or fonts from other sites, minified bundles
  or paid assets. Describe the technique and write your own implementation.
- Only include snippets from open-source projects when their license allows it, and keep their
  license notice.

### Writing style
- Plain, direct language. Short sentences.
- Use hyphens, commas or colons instead of em dashes.
- Use code formatting for file names, functions and values.

## Pull request checklist

- [ ] Recipes were tested against the versions they mention.
- [ ] New techniques are credited in the recipe.
- [ ] No third-party code or assets were copied without permission.
- [ ] `SKILL.md` stays concise, and long code lives in `references/`.
- [ ] The skill still installs by cloning the repo into a skills folder.

## Testing your change

1. Clone your branch into your skills folder, for example
   `git clone -b your-branch <your-fork-url> ~/.claude/skills/cinematic-webgl-site`.
2. Start a new agent session and ask it to build or extend a scroll-driven WebGL site.
3. Check that the agent picks up the skill and that the result runs without console errors.

## Commit messages

Use a short imperative summary, for example `Add depth-of-field recipe` or
`Fix bloom import for three r188`.
