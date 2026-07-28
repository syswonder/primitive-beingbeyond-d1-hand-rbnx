# primitive-beingbeyond-d1-hand-rbnx

*[中文版](./README_CN.md)*

`robonix.primitive.beingbeyond.d1.hand` — Robonix primitive for the BeingBeyond D1
five-finger dexterous hand (Linker, six axes). Talks to the `beingbeyond_d1_sdk`
DexHand CAN link directly, with **no ROS backend**.

## Capabilities

| Contract | Transport | What it does |
|---|---|---|
| `robonix/primitive/hand/driver` | gRPC | lifecycle (`CMD_INIT` / `ACTIVATE` / `DEACTIVATE` / `SHUTDOWN`) |
| `robonix/primitive/hand/info` | gRPC | each axis's finger and type, so consumers can map fingers |
| `robonix/primitive/hand/move_joint` | gRPC | axis-level position control |
| `robonix/primitive/hand/set_joint_speed_limits` | gRPC | per-axis speed ceilings |
| `robonix/primitive/hand/set_joint_torque_limits` | gRPC | per-axis torque ceilings |
| `robonix/primitive/hand/get_state` | gRPC | read the current axis angles |

The finger-level contracts (`move_finger` / `set_finger_*`) and the `state_joint` /
`state_finger` streams (`topic_out`, which would need a ROS backend this deployment
does not run) are **intentionally not provided**. Consumers read the axis↔finger
mapping from `info` and then drive `move_joint`.

## Configuration

Fields, units and defaults are in `config.spec`. The essentials:

- `hand_type` (`right` | `left`, default `right`) — selects the axis signs / mapping
  inside the DexHand SDK.
- `can_iface` (default `can0`) — the CAN interface the hand is wired to.
- `baudrate` (default `1000000`) — CAN bitrate; must match the hand's firmware.

Every field also falls back to a matching `D1_HAND_*` env var.

## Dependencies and permissions

- Python **3.10** (the wheel is `cp310` + `manylinux_2_17_x86_64`, so nothing else
  will install it).
- `beingbeyond_d1_sdk` (≥ 0.2.0) and `robonix_api`. The SDK wheel ships in the
  `robot-beingbeyond-d1` deployment repo under `tools/func_verify/lib/` — just
  `pip install` that whl.
- CAN permissions: DexHand runs `ip link set …` at init to bring the interface up.
  When not running as root, bring the interface up beforehand or export
  `PREFLIGHT_SUDO_PASS`.
- `scripts/start.sh` defaults to `$HOME/miniconda3/envs/bb_d1_robonix/bin/python3`;
  override with `BLOCK_GRASP_PYTHON`.

## Build and run

```bash
bash scripts/build.sh      # rbnx codegen, generating the gRPC stubs
bash scripts/start.sh      # or let rbnx boot start it from a deployment manifest
rbnx caps -v | grep hand   # check all 6 capabilities registered and the provider is ACTIVE
```

Shutdown is handled by the Driver's `CMD_SHUTDOWN` (`on_shutdown` stops hand output).
There is no cleanup beyond that, so no `scripts/stop.sh` is provided.

## Safety

- Do the read-only checks first (`info` / `get_state`) before commanding `move_joint`.
- For the first open/close, lower the ceiling with `set_joint_torque_limits` to avoid
  pinching a finger or damaging a knuckle.
- Providers stop hardware output on input timeout, process exit, or `CMD_SHUTDOWN`.

## Layout

```
package_manifest.yaml   config.spec   README.md   README_CN.md
scripts/{build.sh, start.sh}
d1_hand/{__init__.py, main.py}
```

## License

MulanPSL-2.0
