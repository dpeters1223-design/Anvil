# VR build loop on Big Iron: spec for the implementing session

Audience: the Claude Code session setting up Big Iron. If something in this spec turns out to be wrong on the real machine, do what works and say so in your report.

## Goal

Build Anvil: a standalone, reusable, unattended build loop for any Unity VR project (games, physics lessons, anything else), with Claude as the controller. It lives in its own git repo, `anvil`, separate from every project it builds. The loop must keep working while David's laptop is off.

## Hard constraints of this machine

- CPU-only inference. Always pass `num_gpu: 0` to Ollama.
- 6 cores, 32 GB RAM. Run one local model at a time. Never run a model while Unity is compiling or testing.
- Workers are `qwen2.5-coder:7b` and `deepseek-coder-v2:16b`. They are weak on Unity 6 and Meta XR APIs. Give them only small, test-defined tasks.
- Unity 6000.3.28f1, Personal license, expires around Dec 11 2026, renewed by hand.
- This is Ethan's server. No sudo from the loop, nothing outside the project and engine folders, no new always-on services without David asking Ethan.

## Layout

Separate the engine from the projects.

```
~/anvil/                    engine, its own git repo
  orchestrator.py           the loop
  gate/unity_gate.sh        compile + tests
  gate/blender_gate.sh      headless Blender checks (phase 3)
  controller/instructions.md
  templates/                BRIEF.md, CLAUDE.md, architecture.md, queue.json
~/<project>/                one git repo per VR project
  BRIEF.md                  what this experience is, art direction, perf budget
  CLAUDE.md                 standing Quest rules
  docs/architecture.md      source of truth
  tasks/queue.json          task queue
  tasks/runs/               one report per loop run
  QUESTIONS.md              things only David can decide
  STOP                      if this file exists, the loop exits cleanly
```

`~/seedfall` is read-only for this work. You may read it for ideas and copy patterns that work (queue format, run reports, "two independent implementers judged by tests"), but never modify it, commit to it, or touch its git history. Seedfall can become an Anvil project later, when David says so.

## Who does what

| Role | Does | Never does |
|---|---|---|
| Orchestrator (Python) | Picks tasks, calls models, applies files, runs gates, commits, enforces limits | Makes design decisions |
| Claude controller | Splits the brief into tasks, writes the tests first, writes hard code itself (XR wiring, shaders, editor scripts, Blender scripts), reviews, decides merge or redo | Runs unbounded; it gets one bounded job per call |
| Local workers | Implement one small pure-C# class or method so that existing tests pass | Touch scenes, prefabs, project settings, or the shell |
| Gates | Compile, run EditMode tests, later render screenshots | Get skipped |

Workers have no tools. The orchestrator sends a prompt to Ollama's HTTP API, gets file contents back, and writes them into a git worktree on a task branch.

## Task rules

Each task in the queue has: id, type (`logic`, `scene`, `asset`, `review`), owner (`worker` or `claude`), the files it may touch, the test that proves it, status, attempt count.

- `logic` tasks go to workers. Claude writes the NUnit EditMode test first and commits it. Both workers attempt the task one after the other. The first attempt that passes the gate goes to Claude for review.
- `scene` and `asset` tasks go to Claude. Scenes are built by C# editor scripts run with `-executeMethod`, not by hand and not through Unity MCP (there is no editor window here).
- After 3 failed worker attempts, the task is reassigned to Claude.
- After 2 failed Claude attempts, the task is marked blocked, a note goes in `QUESTIONS.md`, and the loop moves on.

## Unity gate

```
~/Unity/Hub/Editor/6000.3.28f1/Editor/Unity \
  -batchmode -nographics -projectPath <worktree> \
  -runTests -testPlatform EditMode \
  -testResults <run>/results.xml -logFile <run>/unity.log
```

- Pass means exit code 0 and no failed tests in `results.xml`.
- Feed back only the compiler errors and failed test names, trimmed, never the whole log.
- If the log shows a license error, stop the loop, write `QUESTIONS.md`, and do not count it as a task failure.
- Check that Android Build Support is installed for this editor. Quest APK builds need it. Report if it is missing; do not install it without David.

## Claude controller calls

- Auth: `CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token` (David runs this on his laptop). Keep it in a file with mode 600 owned by `dpeters`. `ANTHROPIC_API_KEY` must be unset.
- Call headless with `claude -p`, working directory set to the task worktree. Check `claude --help` for the current flags for JSON output, turn limits and tool permissions, and restrict tools to that worktree.
- Write `controller/instructions.md` for this headless format. Seedfall's version, written for pasting into ChatGPT by hand, can be a starting point.
- Budget: a cap on Claude calls per hour and per day, set in a config file. On a usage-limit or auth error, sleep and retry later. Never spin.

## Safety rails

- All work happens on task branches in worktrees. Only the orchestrator merges, and only after gate pass plus Claude approval.
- Push to GitHub after each merge so David can follow along from anywhere.
- Never force-push, never rewrite history, never delete branches that hold unmerged work.
- Run under `tmux` first. Move to a systemd user service only after the test passes and Ethan is fine with it.

## Monitoring (required by Ethan, who owns the server)

Ethan must be able to see what Anvil is doing at any time from his Mac, without needing David.

- The orchestrator rewrites `~/anvil/status/STATUS.md` every loop: running or stopped, current project and task, which model is active, last gate result, tasks done today, Claude calls used today, last error.
- It appends one line per event to `~/anvil/status/anvil.log`.
- Install a command `anvil-status` that prints STATUS.md plus current CPU and RAM use, and `anvil-status -f` that follows the log live.
- Ethan's user must be able to read `~/anvil/status/` and run `anvil-status` without sudo. Use group read permissions on that one folder only. The token file stays mode 600.
- Include `anvil-stop`, which creates the STOP file so the loop exits cleanly after the current step. Ethan may run it.
- No web dashboard or open port in Phase 1. If one is added later, bind it to the Tailscale interface only, read-only, and only after Ethan agrees.

## Test plan

Phase 1, the proof (do this first, nothing else):
1. `claude -p "say hello"` works on Big Iron with the token.
2. The Unity gate runs on an empty Unity 6 project and reports pass.
3. One `logic` task runs end to end with no human input: Claude writes a test for a `GravityToggle` class (on/off state, returns the gravity vector), both workers attempt it, the gate judges, Claude reviews, the orchestrator merges and pushes.
4. Record wall-clock time per step in the run report.

Phase 2: one `scene` task. Claude writes an editor script that builds a scene with a floor, a light and a sphere using `GravityToggle`. Gate: it compiles and an EditMode test finds the objects in the scene.

Phase 3: Blender headless (`blender -b -P script.py`), GLB export into `Assets/`, low-sample Cycles CPU renders that Claude reviews as images. Then Unity screenshots, which need a display and a GPU: test whether the iGPU under Xvfb can render before relying on it.

Phase 4: the Quest skills (Meta agentic-tools, Unity's official plugin) added to `.claude/skills`, and an APK build.

Report back after Phase 1 with: what was borrowed from Seedfall, timings, and anything in this spec that turned out to be wrong.
