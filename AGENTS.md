# AGENTS.md

## Project Overview

DoomVLM is a single Jupyter notebook (`doom_vlm.ipynb`) that lets Vision Language Models play Doom via the ViZDoom engine. The AI sees game screenshots, decides actions through OpenAI-compatible tool calling API (`shoot(column)` / `move(direction)`), and the notebook translates these into game inputs.

**Game modes:** Solo scenarios (11 classic ViZDoom challenges), Deathmatch Benchmark (sequential fair comparison vs bots), Deathmatch Arena (multiprocessing PvP, all agents simultaneously).

**Backend:** Any OpenAI-compatible API with vision + tool calling. Recommended: LM Studio with Qwen 3.5 models.

## Dev Environment Setup

```bash
# Install Python dependencies
pip install -r requirements.txt

# Start LM Studio server (install from https://lmstudio.ai/download)
lms server start

# Download a model
lms get qwen3.5-0.8b

# Launch the notebook
jupyter lab doom_vlm.ipynb
```

System dependencies for recording:
- macOS: `brew install ffmpeg`
- Linux: `apt-get install ffmpeg fonts-dejavu-core libsdl2-2.0-0 zstd`

## Architecture

The entire project lives in a single notebook with 13 cells:

| Cell | Purpose |
|------|---------|
| 0 | Markdown: title and documentation |
| 1 | CSS: hide code inputs from view |
| 2 | Setup: imports, constants, scenario catalog, `AgentConfig` dataclass |
| 3 | Agent configuration UI: per-agent widgets (name, API, model, prompts, parameters) |
| 4 | Game settings UI: game type, DM mode, timing, bots, recording |
| 5 | Image utilities and VLM integration: `call_vlm()`, `parse_vlm_response()`, `build_action()` |
| 6 | Game engine: `EpisodeRecorder`, `setup_dm_host/client()`, `_run_dm_episode()`, `_run_solo_episode()` |
| 7 | Display and runners: `GameDisplay`, `run_benchmark()`, `run_arena()` |
| 8 | Run/Stop buttons and orchestrator: `_run_game_inner()` |
| 9 | Markdown: results section header |
| 10 | Results tables display |
| 11 | Show recordings (GIF/MP4) |
| 12 | ZIP-package results for download |

## Key Functions and Classes

**VLM Integration (cell 5):**
- `call_vlm(messages, tools, api_url, model, ...)` — sends base64 screenshot + prompt to API, returns `(response_dict, latency)`
- `parse_vlm_response(response)` — extracts tool calls, returns `dict(shoot, cell, move, reason)`, fallback to `move="forward"` on failure
- `build_action(parsed, grid_turn_deltas)` — converts parsed response to 7-element ViZDoom action vector
- `make_vlm_tools(grid_cols, shoot_desc, move_desc, ...)` — generates OpenAI tool definitions

**Game Engine (cell 6):**
- `EpisodeRecorder` — records per-tic frames with stat overlays, outputs GIF or MP4
- `setup_dm_host(scenario, map_name, num_players, ...)` — initializes ViZDoom host for multiplayer
- `setup_dm_client(scenario, map_name, agent_name, ...)` — connects client to host
- `_run_dm_episode(game, agent_cfg, display, ...)` — main game loop, calls VLM each step
- `_run_solo_episode(game, agent_cfg, display, ...)` — solo scenario loop

**Display (cell 7):**
- `GameDisplay` — thread-safe live multi-agent display using ipywidgets
- `run_benchmark(agents, scenario, ...)` — sequential benchmark runner
- `run_arena(agents, scenario, ...)` — multiprocessing arena runner

**Data (cell 2):**
- `AgentConfig` — dataclass with all per-agent settings (name, api_url, model, prompts, parameters, tool descriptions)
- `SCENARIOS` — dict of all available scenarios with paths and map names

## Known Issues

1. **ipywidgets + threads:** `widgets.Output()` context manager is broken from background threads (ipywidgets#2358). Use `VBox.children` assignment instead — works via comm protocol from any thread.

2. **macOS multiprocessing:** `multiprocessing.set_start_method("fork")` from Jupyter can deadlock on macOS. Arena mode uses fork with a fallback. Future improvement: migrate to threads or extract game loop to a `.py` module for "spawn" compatibility.

3. **VLM response parsing:** Some models emit `<think>...</think>` tags or special tokens (`<|im_end|>`, `<|eot_id|>`). These are stripped before parsing. Empty responses fallback to `move="forward"`.

## Conventions

- Code comments in English
- Thread-safe widget updates via ipywidgets comm protocol (`.value`, `.children` assignments), never `with Output():`
- `tool_choice: "required"` in API calls to force tool call responses
- Paths use `Path.cwd()` — all relative, no hardcoded user paths
- Agent colors: green (0), red (1), blue (2), yellow (3)
