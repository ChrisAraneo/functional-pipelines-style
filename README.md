# Functional Pipelines Style

***Every function body is a single expression that composes small steps***.

Quick example to understand the main idea of this style:

```ts
export const placePlayerSpawn = (tiles: Tile[][], levelType: LevelType) =>
  flow(
    findPlayerSpawnCandidates,
    sortPlayerSpawnCandidates,
    pickPlayerSpawnCandidate,
    createPlayerSpawnPatches,
    patchPlayerSpawnTiles,
  )({ tiles, levelType });
```

## What's in this repository?

- [`FUNCTIONAL_PIPELINES_STYLE.md`](FUNCTIONAL_PIPELINES_STYLE.md): the style guide itself, written primarily for AI agents.
- [`refactor/prompts/`](refactor/prompts/): AI prompts for refactoring an
  existing codebase into this style.

## Refactoring a codebase

The refactor runs in two stages. Each prompt has placeholders (`{{SCOPE}}`,
`{{PLAN_DIR}}`, …) listed in its _Inputs_ table; any you leave blank fall back
to the defaults given there.

1. **Plan.** Give
   [`01_ARCHITECT.md`](refactor/prompts/01_ARCHITECT.md) to a capable model
   running at high effort. It studies the guide and your codebase and writes a
   detailed work plan to `refactor-plan/`. It does not change any code.
2. **Execute.** Give [`02_EXECUTOR.md`](refactor/prompts/02_EXECUTOR.md) to one
   or more worker agents. Each one carries out tasks from the plan, verifies
   them and records its progress in `PROGRESS.md`. Several agents can work in
   parallel by taking different lanes.

## Author

This experimental code style is brought to you by Krzysztof Pająk (Chris Araneo) - chris.araneo@gmail.com

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
