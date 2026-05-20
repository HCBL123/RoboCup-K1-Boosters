# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RoboCup K1 Football Demo — a ROS2-based autonomous soccer strategy system running on the Booster K1 humanoid robot. The robot plays in RoboCup Humanoid League matches as either a striker or goalkeeper, using vision, localization, behavior trees, and multi-robot communication.

### Competition Format

- **We control 1 robot only.** Teammate is from a different team with a different codebase.
- **Mixed-team format**: our K1 robot + 1 robot from another team cooperate against the opposing 2-robot team.
- **Both robots connect to a shared GameController computer.** Match state (INITIAL/READY/SET/PLAY/END, freekicks) comes from that machine via UDP. `game_control_ip` must be set correctly on match day.
- **No reliable inter-robot communication with teammate.** `brain_communication.h` UDP messages will not be understood by the other team's robot. Coordinate via field positioning only — treat teammate as semi-autonomous agent with unpredictable behavior.
- **Both roles (striker + goalkeeper) must be match-ready.** Role is decided at match time based on what the teammate plays. Cannot know in advance — both subtrees must be fully tuned and functional.
- **Role conflict risk**: if teammate also plays striker, goal is undefended. Mitigate by pre-agreeing on roles with teammate team before the match, or by implementing a positioning-based heuristic to decide role dynamically.

---

## Build & Run Commands

All commands must be run from the repo root (`K1_Football_Demo/`). The workspace uses `colcon` (ROS2 build tool).

```bash
# Full build (all packages)
./scripts/build.sh

# Build only the brain package (faster iteration cycle)
./scripts/build_brain.sh

# Build with debug symbols
./scripts/build_debug.sh

# Start all nodes (vision + brain + game_controller) in background
./scripts/start.sh

# Start only the brain node (when vision is already running)
./scripts/start_brain.sh

# Stop all running nodes
./scripts/stop.sh

# Launch with launch-time overrides (tree, role, pos, sim, disable_log, disable_com)
ros2 launch brain launch.py tree:=game.xml role:=striker pos:=right disable_log:=true
```

After building, `source install/setup.bash` before running any `ros2` commands directly.

---

## Code Architecture

### Core Object Model

`Brain` (`src/brain/include/brain.h`) is the central ROS2 node. It owns five major sub-objects:

| Object | File | Purpose |
|---|---|---|
| `BrainConfig` | `include/brain_config.h` | Read-only config loaded from YAML at startup |
| `BrainData` | `include/brain_data.h` | All runtime mutable state (ball, pose, teammates, etc.) |
| `BrainTree` | `include/brain_tree.h` | BehaviorTree.cpp execution engine + all BT node classes |
| `RobotClient` | `include/robot_client.h` | Hardware commands to Booster SDK (velocity, kick, stand-up) |
| `Locator` | `include/locator.h` | Particle filter self-localization using field markers |
| `BrainCommunication` | `include/brain_communication.h` | UDP multi-robot and GameController communication |
| `BrainLog` | `include/brain_log.h` | Rerun visualization/logging |

The `Brain::tick()` is called at ~30 Hz by a ROS2 timer. Each tick: updates memory/predictions → runs `tree->tick()` → handles special states.

### Coordinate System

- **Field frame**: center of field is origin; +X points toward opponent's goal (forward); +Y points left; theta is counterclockwise positive.
- **Robot frame**: robot center is origin; +X forward, +Y left.
- `brain->data->robotPoseToField` — robot's pose in field frame (updated from odometry + localization).
- Key transforms: `brain->data->robot2field()` / `brain->data->field2robot()`.

### Behavior Trees

XML files live in `src/brain/behavior_trees/`. The active tree is specified at launch (default: `game.xml`).

- `game.xml` — full match tree; obeys GameController states (INITIAL → READY → SET → PLAY → END) and handles freekick sub-states.
- `demo.xml`, `chase.xml` — simplified trees for demos.
- `subtrees/subtree_striker_play.xml` — striker logic: find ball → chase → adjust → kick/RLVisionKick.
- `subtrees/subtree_goal_keeper_play.xml` — keeper logic: guard goal, chase ball, block.
- `subtrees/subtree_striker_freekick.xml` / `subtree_goal_keeper_freekick.xml` — free-kick positioning.

All BT node classes are declared in `brain_tree.h` and registered in `BrainTree::init()` (`src/brain/src/brain_tree.cpp`). To add a new BT node: declare its class in `brain_tree.h`, implement `tick()` in `brain_tree.cpp`, register it with `REGISTER_BUILDER(ClassName)` in `BrainTree::init()`.

### Strategy Decision Flow (Striker)

1. `StrikerDecide` evaluates `ball_range`, `ball_confidence`, teammate cost rankings → outputs `decision` string.
2. Decision values: `"find"` → scan for ball; `"chase"` → run toward ball; `"adjust"` → fine-position behind ball facing goal; `"auto_visual_kick"` → RL+vision kick; `"kick"` → standard kick action; `"assist"` → support position.
3. `CalcKickDir` computes optimal kick direction to goal before the decide step.

### Configuration System

- Primary config: `src/brain/config/config.yaml`
- Local override (not committed, robot-specific): `src/brain/config/config_local.yaml` — values here override `config.yaml`
- All tunable parameters (thresholds, speeds, strategy flags) are in `config.yaml`. Key sections: `strategy`, `obstacle_avoidance`, `locator`, `robot`, `ball_predictor`.
- `BrainConfig` stores all static params; `Brain::loadConfig()` populates it from ROS2 parameters (which come from YAML).

### Key Parameters to Know

| Parameter | Location | Effect |
|---|---|---|
| `strategy.ball_confidence_threshold` | config.yaml | Min confidence to trust ball detection |
| `strategy.ball_memory_timeout` | config.yaml | Seconds before ball memory expires |
| `strategy.near_ball_speed_limit` | config.yaml | Speed cap when robot is close to ball (avoids overshooting) |
| `strategy.enable_auto_visual_kick` | config.yaml | Enable RL+vision kick mode |
| `strategy.kick_range` / `kick_theta_range` | config.yaml | Distance and angle tolerance for kick decision |
| `robot.odom_factor` | config.yaml | Odometry scale correction |
| `locator.min_marker_count` / `max_residual` | config.yaml | Particle filter quality gates |

---

## Working with This Codebase

### Before Suggesting Any Change

1. **Real-match impact first**: Reason about what physically happens on the field if this change is active. A wrong threshold could make the robot spin in place, walk into the goal, or kick at the wrong time.
2. **Failure modes**: Identify edge cases — ball not visible, localization drift, robot fallen, multiple opponents, kickoff restrictions.
3. **Reevaluate before implementing**: Review logic → simulate mentally what could go wrong in a real match → fix → re-check. Repeat until the change is safe.

### When Implementing a Change

Always preserve the original code by commenting it out and annotating what changed:

```cpp
// This is the old code
// double threshold = 1.0;

// This is what I changed — reason: tighter threshold avoids premature kick when ball is still moving
double threshold = 0.6;
```

### Where to Make Common Changes

| Change type | File(s) |
|---|---|
| Tune speed/distance thresholds | `src/brain/config/config.yaml` |
| Adjust behavior sequence / game logic | `src/brain/behavior_trees/*.xml` |
| BT node parameters (in XML) | The relevant `<NodeName param=.../>` in the XML |
| BT node logic | `src/brain/src/brain_tree.cpp` + `include/brain_tree.h` |
| Robot motion commands | `src/brain/src/robot_client.cpp` |
| Ball/robot state processing | `src/brain/src/brain.cpp` (`detectionsCallback`, `updateMemory`, etc.) |
| Localization | `src/brain/src/locator.cpp` |
| Multi-robot cooperation | `src/brain/src/brain_communication.cpp` |

### Roles and Deployment

- `config.yaml`: set `game.player_role` to `striker` or `goal_keeper`, `game.player_id` (1–5), `game.team_id` (must match GameController).
- **Role is decided at match time** based on what the teammate team plays. Both striker and keeper subtrees must be fully ready before any match.
- Set `game.treat_person_as_robot: false` in production — only `true` for debugging.
- `game_control_ip` must match the GameController machine's IP on the match network. Verify this before every match.
- **Do not rely on `brain_communication.h` to coordinate with teammate** — they run different code. Positioning heuristics only.
- FastDDS profile: `configs/fastdds.xml` (UDP-only profile is loaded at startup by `start.sh`).

### Logging

Enable Rerun log by setting `rerunLog.enable_tcp: true` in config and pointing `server_ip` at a machine running Rerun Viewer. File logging uses `rerunLog.enable_file: true` with `log_dir` path. Disable at launch with `disable_log:=true`.

---

## Specialized Agents

Four project agents live in `.claude/agents/`. Use them via the `Agent` tool with `subagent_type` matching the agent name.

| Agent | Trigger |
|---|---|
| `soccer-behavior-simulator` | Mentally simulate robot behavior in 2v2 match scenarios without running code — validate BT changes, config tuning, new subtrees, or any logic before deploy |
| `strategy-improvement-advisor` | Analyze current strategy for improvements — use after match logs, observed suboptimal behavior, or pre-tournament tuning; produces simulation-ready proposals |
| `k1-simulation-integrator` | Receive simulation engineer reports (observed failures, edge cases, behavioral insights) and safely integrate fixes into codebase following project coding standards |
| `robocup-stability-verifier` | Cross-validate any significant code change (BT logic, config, motion, localization) between coding and simulation teams before deploying to physical robot |

### Recommended Workflow

```
Observe issue / idea
       ↓
strategy-improvement-advisor  →  produces improvement proposal
       ↓
soccer-behavior-simulator     →  mentally validates proposal across match scenarios
       ↓
k1-simulation-integrator      →  integrates sim engineer feedback into codebase
       ↓
robocup-stability-verifier    →  final gate before deploying to physical robot
```
