---
name: parameter-interactions
description: Config parameters that interact and must be tuned together — avoid independent changes
metadata:
  type: project
---

## kick_range + chase_threshold + near_ball_range
These three create the chase→adjust→kick band. Must be tuned together:
- `kick_range` (0.7) = inner boundary — robot enters kick state
- `chase_threshold` (1.2) = outer boundary — robot stops chasing and enters adjust
- Band = 1.2 - 0.7 = 0.5m for adjust
- `near_ball_range` (1.2) = where speed tapering starts — should match chase_threshold

Changing any one without the others can recreate kick/adjust oscillation (P1 bug).

## near_ball_speed_limit + near_ball_range + Chase.dist
- `near_ball_speed_limit` (0.45 m/s) must be low enough to not overshoot `kick_range` (0.7m)
- `Chase.dist` (0.1m) is the final stop distance
- If `near_ball_speed_limit` is raised, also raise `Chase.safe_dist` to maintain braking margin

## kick_theta_range + auto_visual_kick_enable_angle
Both gate kick decisions on angular alignment. If `kick_theta_range` is tightened, verify it doesn't conflict with the RL kick angle window. Striker should reach angle < `kick_theta_range` (0.15 rad) AND the RL kick should only fire if within `auto_visual_kick_enable_angle` (0.5 rad). The RL window is wider by design.

## min_marker_count + max_residual
`min_marker_count` (4) controls how often localization is attempted; `max_residual` (0.25) controls acceptance quality. Lowering `min_marker_count` alone without tightening `max_residual` accepts more noisy poses. These were changed together in P6.

## ball_control_cost_threshold + Cost EMA alpha
`ball_control_cost_threshold` (3.5) determines when role switches fire. If EMA alpha is changed (currently 0.2 = slow filter), the effective threshold band changes because smoothed cost converges slowly. A faster EMA (e.g. 0.4) + same threshold means more frequent oscillation — lower threshold too if EMA is sped up.
