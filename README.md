# Functional Pipelines Style

***Every function body is a single expression that composes small steps***.

Quick example:

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
- [`refactor/`](refactor/): AI prompts for refactoring an
  existing codebase into this style.

## Refactoring a codebase

The refactor has four steps:

1. **Plan**: decide what to change.
2. **Execute**: make the changes.
3. **Detail**: add the missing details to the plan.
4. **Clean**: delete the plan when all tasks are done.

You can let one AI agent do all four steps for you. You can do each step
yourself.

The master, plan, execute and clean prompts have placeholders, like `{{SCOPE}}` or
`{{PLAN_DIR}}`. You can find them in the _Inputs_ table of each prompt. If you leave a placeholder
empty, the prompt uses the default value from that table. The detail prompt
has no placeholders. It gets everything it needs from the plan.

No agent ever commits or changes a branch. All changes, including the plan,
stay uncommitted on your current branch. Agents can start from a working tree
that already has uncommitted changes. When the refactor is done, you check
the changes and commit them yourself.

### Auto

Give [`00_MASTER.md`](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/refactor/00_MASTER.md) to your best model. Use
the highest effort level. The tool you use must let the model start other
agents (sub-agents).

The master agent does the whole refactor for you:

- It starts one agent to write the plan.
- It starts one agent for each task. These agents can use a lower effort
  level.
- It starts a detail agent when a task needs more details, or when an agent
  gets stuck.
- When all tasks are done, it starts a clean agent that deletes the plan.
  If some tasks could not be finished, it keeps the plan for you.

When an agent asks a question, the master agent answers it. It always picks
the safe choice: the code must work the same way, and the public API must not
change.

The master agent runs one agent at a time. Before each one, it saves a
snapshot of the uncommitted changes outside the repository. It uses the
snapshot to check what the agent changed, and to undo the agent's work if
needed. When it is done, check the uncommitted changes and commit them. Also
read the final report. It lists the decisions the agent made.

### Manual

1. **Plan.** Give
   [`01_PLAN_ARCHITECT.md`](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/refactor/01_PLAN_ARCHITECT.md) to a strong
   model. Use a high effort level. The agent reads the style guide and your
   code. Then it writes a plan in `refactor-plan/`. It does not change your
   code.
2. **Execute.** Give [`02_EXECUTOR.md`](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/refactor/02_EXECUTOR.md) to
   one agent at a time. Each agent does tasks from the plan and checks that
   they work. It writes its progress in `PROGRESS.md`. Do not run two agents
   at the same time: they share one working tree, and nothing is committed
   between them.
3. **Detail.** In a big codebase, the plan does not fully describe later
   tasks. These tasks are marked `needs-detailing`. First, finish the tasks
   that come before them. Then give
   [`03_DETAILING_ARCHITECT.md`](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/refactor/03_DETAILING_ARCHITECT.md)
   to a strong model. Use a high effort level. The agent adds exact steps to
   these tasks, tests them with a dry run, and marks them as ready. Then go
   back to step 2. Repeat until no tasks need details.
4. **Clean.** When all tasks are done, give
   [`04_CLEANER.md`](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/refactor/04_CLEANER.md) to a model. A medium
   effort level is enough. The agent deletes the plan files in
   `refactor-plan/` and other notes made only for the refactor. It keeps all
   code, tests and config, and every document your codebase needs. Its report
   keeps the useful parts of the plan, like the list of excluded files.

## Author

This experimental code style is brought to you by:

Krzysztof Pająk (Chris Araneo) - chris.araneo@gmail.com

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
