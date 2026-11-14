<h1 align="center">Functional Pipelines Style</h1>

<p align="center">
  <img src="alchemist.png" alt="An alchemist at his alembic; surrounded by vignettes of scientific pursuits. Wood engraving by Dalziel, 18--. Source: Wellcome Collection." width="293px" height="400px"/>
  <br/>
  <em>Every function body is a single expression:</em>
  <br/>
  <em>one data-first pipe that composes small steps.</em>
  <br/>
  <br/>
</p>

## Quick example

```ts
export const placePlayerSpawn = (tiles: Tile[][], levelType: LevelType) =>
  pipe(
    { tiles, levelType },
    findPlayerSpawnCandidates,
    sortPlayerSpawnCandidates,
    pickPlayerSpawnCandidate,
    createPlayerSpawnPatches,
    patchPlayerSpawnTiles,
  );
```

## Contents

- [`FUNCTIONAL_PIPELINES_STYLE.md`](FUNCTIONAL_PIPELINES_STYLE.md): the style guide itself, written primarily for AI agents.
- [`refactor/`](refactor/): AI prompts for refactoring an
  existing codebase into this style.
- [`review/`](review/): AI prompts for reviewing an existing codebase against
  this style, without changing a line of it.

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
   the plan and everything that only served the refactor: the scripts in
   `refactor-plan/tools/` (such as the style audit), any refactoring scripts,
   reports or notes left in the codebase, and the references to them. If some
   tasks could not be finished, it keeps the plan for you.

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

## Reviewing a codebase

A review tells you how far a codebase is from the style. It changes no code.

Give [`00_MASTER.md`](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/review/00_MASTER.md)
to your best model and run it in your repository. Use the highest effort
level. It works best in a tool that lets the model start other sub-agents; if
yours cannot, the prompt says how to run the whole review in one session.

The prompt has placeholders, like `{{SCOPE}}` or `{{REPORT_DIR}}`. You can
find them in its _Inputs_ table. If you leave a placeholder empty, the prompt
uses the default value from that table. Three of them are about size:
`{{MAX_FILES_PER_MODULE}}` and `{{MAX_LINES_PER_MODULE}}` set how much code
one agent takes on, and `{{PARALLEL_REVIEWERS}}` how many agents run at once.

### How it works

The review is two nested loops: every module, and for each module every rule.
A **module** is a folder of code small enough for one agent to hold: a folder
in a lib's `src/`, a package's own files, or a part of a folder too big for one
session. A project of thousands of files becomes a few hundred modules, and the
agent reviewing one reads each rule once and applies it to every file in front
of it.

1. **Queue.** The master agent fetches the style guide, turns its contents into
   a checklist of every rule, lists every source file in scope, and cuts the
   repository into modules: it finds the packages, walks down the folders of
   each one and keeps the largest folder that still fits the budget of files
   and lines. Every file lands in exactly one module. The queue, one row per
   module, goes in `style-review/`.
2. **Review.** It starts one
   [reviewer](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/review/01_REVIEWER.md)
   per module, several at a time. The reviewer reads the module's files, then
   walks the checklist from the first rule to the last and decides, for every
   rule and every file, whether the file keeps it, breaks it, or is not the
   kind of file the rule is about. Each violation is written down with the line
   it is on and the rule it breaks. The rules that are about a folder rather
   than a file — the layout, a spec beside every file, what sits in
   `internal/`, the imports that cross a boundary — are answered by the session
   that has the whole folder in front of it. Two wider passes cover the rest:
   one per package, for its layout, its barrel and its project files, and one
   for the repository, for the libs and apps split, the dependencies, the
   aliases and the commands.
3. **Report.** When every module is done, a
   [reporter](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/review/02_REPORTER.md)
   collects the findings into `style-review/REPORT.md` and renders
   `style-review/REPORT.html`, a page built from
   [`REPORT_TEMPLATE.html`](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/review/REPORT_TEMPLATE.html)
   and styled with [simple.css](https://simplecss.org/). It works one module at
   a time, so even a large review fits in one session. The page counts the
   violations by rule, by module and by file, and every violation links to the
   line it is on and to the rule it breaks.

A module that turns out not to fit is never half-reviewed: the reviewer stops
and says so, the master agent splits the module into smaller ones, and the
parts go back in the queue. Nothing outside `style-review/` is ever written to,
and no agent commits or changes a branch. A violation needs a line to quote and
a numbered rule it breaks: where the guide leaves a case open, the review
records an open question instead of guessing. If the run is cut off, start the
master prompt again: it reads its log and carries on where it stopped.

### When it is done

Open `style-review/REPORT.html` in a browser. The summary says which rules are
broken most often and which modules and files are worst. To fix what it found,
give
[`refactor/00_MASTER.md`](https://github.com/ChrisAraneo/functional-pipelines-style/blob/master/refactor/00_MASTER.md)
the module the report puts first.

## License & art attribution

Markdown and HTML files are licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Art is sourced from the [Wellcome Collection](https://wellcomecollection.org/works/q7v7c5yr/images?id=ydxwbs8x) ([Public Domain Mark 1.0](https://creativecommons.org/publicdomain/mark/1.0/)). *"*An alchemist at his alembic; surrounded by vignettes of scientific pursuits*, wood engraving by Dalziel (18--)"*

## Author

Krzysztof Pająk (Chris Araneo) - chris.araneo@gmail.com
