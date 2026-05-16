# Quadcopter Velocity-Tracking Gymnasium Environment

This document describes `QuadcopterVelocityEnv`, a Gymnasium-compatible environment built on top of the dynamics from [`bobzwik/Quadcopter_SimCon`](https://github.com/bobzwik/Quadcopter_SimCon).

The environment defines a velocity-following reinforcement learning task. At every step, the agent sends normalized motor commands, and the quadcopter should track a target planar velocity:

```python
target_velocity = [vx_target, vy_target, vyaw_target]
```

where:

- `vx_target` is the desired x-axis velocity
- `vy_target` is the desired y-axis velocity
- `vyaw_target` is the desired yaw rate

The environment supports two observation modes:

1. `full`: the policy observes the full simulator state
2. `realistic`: the policy does **not** observe ground-truth `x`, `y`; it only observes orientation, altitude-related quantities, body rates, motors, and the target command

Observation noise and random wind can be enabled and configured.

---

## Installation

Install the dependencies used by the original simulator, then install Gymnasium:

```bash
pip install gymnasium numpy scipy matplotlib
```

Then run your scripts from the `Simulation/` directory, or make sure `Simulation/` is on your Python path.

---

## Basic Usage

```python
from quad_velocity_env import QuadcopterVelocityEnv

env = QuadcopterVelocityEnv(obs_mode="full")

obs, info = env.reset(
    seed=0,
    options={
        "target_velocity": [1.0, 0.0, 0.3],
    },
)

for _ in range(1000):
    action = env.action_space.sample()

    obs, reward, terminated, truncated, info = env.step(action)

    if terminated or truncated:
        obs, info = env.reset()
```

---

## Environment Goal

The goal is to follow a commanded velocity:

```python
[vx_target, vy_target, vyaw_target]
```

The agent is rewarded for:

- matching the target horizontal velocity `(vx, vy)`
- matching the target yaw rate `vyaw`
- staying close to the initial altitude
- avoiding excessive roll and pitch
- avoiding excessive angular rates
- avoiding unnecessarily large or rapidly changing motor commands

The default reward is intended as a reasonable starting point. You can replace it with your own reward function.

---

## Action Space

The action is a 4-dimensional normalized motor command:

```python
action_space = Box(low=-1.0, high=1.0, shape=(4,), dtype=np.float32)
```

Each action element corresponds to one motor:

```python
action = [m1, m2, m3, m4]
```

The environment maps this normalized action into the simulator’s physical motor-speed command range:

```python
motor_cmd = min_motor_speed + 0.5 * (action + 1.0) * (max_motor_speed - min_motor_speed)
```

So:

- `-1.0` means minimum motor speed
- `0.0` means the middle of the allowed motor-speed range
- `1.0` means maximum motor speed

The environment uses the original simulator’s low-level dynamics update:

```python
quad.update(t, dt, motor_cmd, wind)
```

---

## Observation Modes

The environment supports two observation modes.

---

## 1. Full-State Observation Mode

Use:

```python
env = QuadcopterVelocityEnv(obs_mode="full")
```

The observation contains:

```text
target velocity:
  vx_target, vy_target, vyaw_target

position:
  x, y, z

orientation:
  q0, q1, q2, q3

linear velocity:
  vx, vy, vz

body angular velocity:
  p, q, r

motor speeds:
  wM1, wM2, wM3, wM4
```

The full observation has dimension:

```python
obs.shape == (20,)
```

Layout:

```text
index   value
0       vx_target
1       vy_target
2       vyaw_target
3       x
4       y
5       z
6       q0
7       q1
8       q2
9       q3
10      vx
11      vy
12      vz
13      p
14      q
15      r
16      wM1
17      wM2
18      wM3
19      wM4
```

This mode is useful for debugging, algorithm development, and comparing against fully observed baselines.

---

## 2. Realistic Observation Mode

Use:

```python
env = QuadcopterVelocityEnv(obs_mode="realistic")
```

The realistic observation does **not** expose ground-truth horizontal position.

It contains:

```text
target velocity:
  vx_target, vy_target, vyaw_target

altitude-related state:
  z

orientation:
  q0, q1, q2, q3

body angular velocity:
  p, q, r

linear velocity:
  vx, vy, vz

motor speeds:
  wM1, wM2, wM3, wM4
```

The realistic observation has dimension:

```python
obs.shape == (16,)
```

Layout:

```text
index   value
0       vx_target
1       vy_target
2       vyaw_target
3       z
4       q0
5       q1
6       q2
7       q3
8       p
9       q
10      r
11      vx
12      vy
13      vz
14      wM1
15      wM2
16      wM3
17      wM4
```

This mode is closer to a realistic onboard sensing setup, where the agent does not get perfect global `x`, `y` information.

---

## Observation Noise

Observation noise is enabled through the `observation_noise_std` argument.

Example:

```python
env = QuadcopterVelocityEnv(
    obs_mode="full",
    observation_noise_std={
        "pos": 0.005,
        "quat": 0.001,
        "vel": 0.03,
        "omega": 0.01,
        "motor": 2.0,
    },
)
```

Supported noise groups:

```python
{
    "target": 0.0,
    "pos": 0.002,
    "z": 0.002,
    "quat": 0.001,
    "vel": 0.02,
    "omega": 0.005,
    "motor": 1.0,
}
```

Noise is Gaussian:

```python
noisy_value = true_value + Normal(0, std)
```

After quaternion noise is applied, the observed quaternion is normalized again.

To disable noise for a specific group, set its standard deviation to `0.0`.

---

## Random Wind

Random wind can be enabled at construction time:

```python
env = QuadcopterVelocityEnv(
    random_wind=True,
    wind_magnitude=3.0,
    wind_heading_deg=90.0,
    wind_elevation_deg=25.0,
)
```

You can also enable or disable wind later:

```python
env.enable_random_wind(True, magnitude=2.0, heading_deg=180.0, elevation_deg=25.0)
```

Or configure it during reset:

```python
obs, info = env.reset(
    options={
        "random_wind": True,
        "wind_magnitude": 4.0,
        "wind_heading_deg": 120.0,
        "wind_elevation_deg": 25.0,
    }
)
```

The default random-wind type is:

```python
wind_type="RANDOMSINE"
```

You can also use:

```python
wind_type="FIXED"
wind_type="SINE"
```

If `random_wind=False` or `wind_magnitude <= 0.0`, the environment uses no wind.

---

## Setting the Target Velocity

You can set a fixed target velocity at reset:

```python
obs, info = env.reset(
    options={
        "target_velocity": [1.0, -0.5, 0.2],
    }
)
```

You can also set it manually during an episode:

```python
env.set_target_velocity(1.0, 0.0, 0.3)
```

The target is clipped using:

```python
max_target_xy_speed
max_target_yaw_rate
```

Default limits:

```python
max_target_xy_speed = 5.0
max_target_yaw_rate = np.deg2rad(150.0)
```

To sample a random target velocity at reset:

```python
obs, info = env.reset(
    options={
        "sample_target_velocity": True,
    }
)
```

---

## Reward Function

The default reward has the form:

```python
reward = alive_bonus - cost
```

The cost includes:

```text
horizontal velocity tracking error
+ yaw-rate tracking error
+ altitude drift penalty
+ roll/pitch penalty
+ angular-rate penalty
+ action magnitude penalty
+ action-rate penalty
```

The default weights are:

```python
RewardWeights(
    vel_xy=1.0,
    yaw_rate=0.35,
    z=0.10,
    roll_pitch=0.05,
    ang_rate=0.01,
    action=0.002,
    action_rate=0.002,
    alive=0.02,
)
```

You can change them:

```python
from quad_velocity_env import QuadcopterVelocityEnv, RewardWeights

env = QuadcopterVelocityEnv(
    reward_weights=RewardWeights(
        vel_xy=2.0,
        yaw_rate=0.5,
        z=0.05,
        roll_pitch=0.02,
        ang_rate=0.005,
        action=0.001,
        action_rate=0.001,
        alive=0.01,
    )
)
```

---

## Custom Reward Function

You can pass your own reward function.

The custom function receives:

```python
reward_fn(target_velocity, full_state, action, env)
```

where:

- `target_velocity` is `[vx_target, vy_target, vyaw_target]`
- `full_state` is the simulator state
- `action` is the normalized action in `[-1, 1]^4`
- `env` is the environment instance

Example:

```python
import numpy as np
from quad_velocity_env import QuadcopterVelocityEnv


def my_reward(target_velocity, state, action, env):
    vx_t, vy_t, vyaw_t = target_velocity

    z = state[2]
    vx = state[7]
    vy = state[8]
    r = state[12]

    vel_error = np.array([vx - vx_t, vy - vy_t])
    yaw_error = r - vyaw_t

    return -(
        1.0 * np.dot(vel_error, vel_error)
        + 0.3 * yaw_error ** 2
        + 0.05 * z ** 2
        + 0.001 * np.dot(action, action)
    )


env = QuadcopterVelocityEnv(
    obs_mode="full",
    reward_fn=my_reward,
)
```

The custom reward has access to the full simulator state even when the policy uses `obs_mode="realistic"`. This is useful because rewards are often computed using quantities that are unavailable to the policy.

---

## Episode Termination

The environment terminates early if `terminate_on_unstable=True` and one of the following happens:

- the simulator state becomes non-finite
- the quadcopter moves too far vertically
- roll or pitch exceeds the maximum allowed tilt

The relevant parameters are:

```python
terminate_on_unstable=True
max_abs_z=5.0
max_tilt_rad=np.deg2rad(80.0)
```

The environment is truncated when the episode reaches:

```python
episode_seconds
```

Default:

```python
episode_seconds = 10.0
```

---

## Frame Convention

The original simulator uses a global orientation convention from `config.orient`.

The environment constructor exposes this as:

```python
orient="NED"
```

or:

```python
orient="ENU"
```

Default:

```python
orient="NED"
```

In `NED`, positive `z` points downward. The default reward uses `z ** 2`, so the sign convention does not matter for the altitude-drift penalty. However, if you implement your own altitude target, make sure you account for the selected frame convention.

---

## Example: Full-State Training Environment

```python
from quad_velocity_env import QuadcopterVelocityEnv

env = QuadcopterVelocityEnv(
    obs_mode="full",
    dt=0.005,
    episode_seconds=10.0,
    random_wind=True,
    wind_magnitude=2.0,
    observation_noise_std={
        "pos": 0.005,
        "quat": 0.001,
        "vel": 0.02,
        "omega": 0.005,
        "motor": 1.0,
    },
)

obs, info = env.reset(
    seed=0,
    options={
        "sample_target_velocity": True,
    },
)
```

---

## Example: Realistic-State Training Environment

```python
from quad_velocity_env import QuadcopterVelocityEnv

env = QuadcopterVelocityEnv(
    obs_mode="realistic",
    dt=0.005,
    episode_seconds=10.0,
    random_wind=True,
    wind_magnitude=3.0,
)

obs, info = env.reset(
    seed=0,
    options={
        "target_velocity": [0.5, 0.0, 0.2],
    },
)
```

---

## Gymnasium API

The environment follows the Gymnasium API:

```python
obs, info = env.reset(seed=0, options={...})
obs, reward, terminated, truncated, info = env.step(action)
```

This means it can be used with Gymnasium-compatible RL libraries.

For example, with Stable-Baselines3 versions that support Gymnasium environments:

```python
from stable_baselines3 import PPO
from quad_velocity_env import QuadcopterVelocityEnv

env = QuadcopterVelocityEnv(obs_mode="realistic")

model = PPO("MlpPolicy", env, verbose=1)
model.learn(total_timesteps=100_000)
```

---

## Notes

- The policy action is a low-level motor-speed command.
- The target velocity is part of the observation in both modes.
- Observation noise is applied after the true observation is assembled.
- The realistic observation mode hides ground-truth horizontal position.
- The reward can still use the full simulator state, even when the policy cannot observe it.
- Random wind can be enabled globally, during reset, or through `enable_random_wind()`.
- The environment is designed as a starting point for velocity tracking and sim-to-real style experiments.
