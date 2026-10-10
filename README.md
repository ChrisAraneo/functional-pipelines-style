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

Give [`00_MASTER.md`](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/refactor/00_MASTER.md)
to your best model and run it in your repository. Use the highest effort
level. The tool you use must let the model start other sub-agents.

The prompt has placeholders, like `{{SCOPE}}` or `{{PLAN_DIR}}`. You can find
them in its _Inputs_ table. If you leave a placeholder empty, the prompt uses
the default value from that table.

### How it works

The master agent runs the whole refactor without you. It starts other agents,
checks their work and decides what runs next:

1. **Plan.** It starts one agent that reads the style guide and your code and
   writes a plan in `refactor-plan/`. The plan is a list of small tasks.
2. **Execute.** It starts one agent for each task. These agents can use a
   lower effort level. Each one does its task and checks that it works.
3. **Detail.** When a task is only outlined, or an agent gets stuck, it starts
   a detail agent that adds the exact steps to the task.
4. **Clean.** When all tasks are done, it starts a clean agent that deletes
   the plan. If some tasks could not be finished, it keeps the plan for you.

When an agent asks a question, the master agent answers it. It always picks
the safe choice: the code must work the same way, and the public API must not
change.

The master agent runs one agent at a time. Before each one, it saves a
snapshot of the uncommitted changes outside the repository, in
`../refactor-scratch/`. It uses the snapshot to check what the agent changed,
and to undo the agent's work if needed. A task that fails too many times is
set aside, and the run goes on without it. If the run cannot go on safely, the
master agent stops and tells you why. If the run is cut off, start the master
prompt again: it reads its log and carries on where it stopped.

No agent ever commits or changes a branch. All changes, including the plan,
stay uncommitted on your current branch. You can start from a working tree
that already has uncommitted changes.

### When it is done

Read the final report first. It lists the decisions the master agent made for
you, the tasks it set aside, and the code it left out. Then check the
uncommitted changes and commit them yourself.

## Author

This experimental style is brought to you by:

Krzysztof Pająk (Chris Araneo) - chris.araneo@gmail.com

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
