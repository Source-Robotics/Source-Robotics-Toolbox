# Source Robotics Toolbox

**Python toolbox for robot joints driven by Source Robotics motor drivers.** It turns a motor's encoder ticks into
the joint's position and speed after the gear reduction, and back, and finds the joint's true position at power-up
with sector homing.

[Guides](https://docs.source-robotics.com/Spectral-BLDC-docs/latest/guides/) ·
[PyPI](https://pypi.org/project/SourceRoboticsToolbox/) ·
[Examples](https://github.com/Source-Robotics/Source-Robotics-Toolbox/tree/main/Examples) ·
[Discord](https://discord.com/invite/prjUvjmGpZ)

[![PyPI](https://img.shields.io/pypi/v/SourceRoboticsToolbox?style=flat-square&color=143952&label=pypi)](https://pypi.org/project/SourceRoboticsToolbox/)
[![Python](https://img.shields.io/badge/python-3.10%2B-143952?style=flat-square)](https://pypi.org/project/SourceRoboticsToolbox/)
[![License](https://img.shields.io/badge/license-MIT-143952?style=flat-square)](https://github.com/Source-Robotics/Source-Robotics-Toolbox/blob/main/LICENSE)
[![Discord](https://img.shields.io/discord/1072498136284667955?style=flat-square&logo=discord&logoColor=white&label=discord&color=292B37)](https://discord.com/invite/prjUvjmGpZ)

What it does:

* Sector homing
* Robot joint mastering/calibration
* Converting encoder ticks to joint position and speed
* Converting joint position and speed to encoder ticks
* Encoder ticks to and from radians and degrees, at the joint

## Install

```
pip install SourceRoboticsToolbox
```

The toolbox has no dependencies of its own. Paired with the [Spectral-BLDC](https://pypi.org/project/Spectral-BLDC/)
library, which reads the drivers' encoders over CAN, it lets you write simple yet capable code for robot control.
Spectral-BLDC needs a CAN adapter that speaks slcan, such as the
[CANvas USB-to-CAN adapter](https://source-robotics.com/products/canvas-usb-to-can-adapter):

```
pip install Spectral-BLDC
```

## Quick start

```python
import SourceRoboticsToolbox
import time

# This is represents encoder ticks where your joint should be at 0 rad
# Usually you know that joint position from your robots DH table / kinematic diagram
Master_position = 14000 
Joint_reduction_ratio = 6 # Joint has reduction ratio of 6:1

Joint = SourceRoboticsToolbox.Joint(encoder_resolution = 14,
                                    master_position=Master_position,
                                    gear_ratio = Joint_reduction_ratio,
                                    offset = 0, #Change If your motor is not at 0 rad at master position 
                                    dir = 0) # Change joint direction

current_motor_encoder_position = 10000 #This emulates the data you would get from your motor encoder
# Init flag; used to run .determine_sector once
Intial = 0

while True:

    # This will give you Joint position in radians 
    # NOTE that joint position is value after the gear reduction of your actuator
    position_values =  Joint.get_joint_position(current_motor_encoder_position)
    print(position_values)

    # This will perform sector homing and determine the sector motor is in
    if Intial ==0 :
        Intial = 1
        Joint.determine_sector(current_motor_encoder_position)

    time.sleep(0.5)
```

With real motors, the encoder reading comes from the driver: `two_spectral_micro.py` in
[Examples](https://github.com/Source-Robotics/Source-Robotics-Toolbox/tree/main/Examples) reads two Spectral Micro
drivers over CAN.

## Sector homing

<img src="https://raw.githubusercontent.com/Source-Robotics/Source-Robotics-Toolbox/main/Images/sector_image.png" alt="Six sectors of a 6:1 gear reduction, each one revolution of the motor, from 0 to 16383 encoder ticks" width="480">

The image represents a 6:1 gear reduction with a Spectral micro BLDC driver. Each sector represents one full
rotation of the BLDC motor. For the motor encoder, each sector begins at 0 encoder ticks and ends at 16,383 encoder
ticks. Sector homing allows you to record the initial position of the encoder and map it to a specific value for
your joint. With this setup, if your robot joint boots in the correct sector, it will know its true position.

Note: This is a simplified explanation. The algorithm is a bit more complex and includes some smart features. To
learn more, check the [source code](https://github.com/Source-Robotics/Source-Robotics-Toolbox/blob/main/SourceRoboticsToolbox/SourceRoboticsToolbox.py).

As the gear reduction ratio increases, this method becomes less effective. However, it works well for ratios up to
15:1.

In the code: `master_position` is the encoder reading, within one motor revolution, at which the joint is at its
zero; call `determine_sector()` once with the first encoder reading after power-up, then `get_joint_position()`
with every reading.

## Works with

| Product | Encoder | `encoder_resolution` | Reading the encoder |
|---|---|---|---|
| Spectral Micro BLDC controller | 14-bit, 16384 ticks per revolution | 14 (the default) | [Spectral-BLDC](https://github.com/Source-Robotics/Spectral-BLDC-Python) 1.20 or newer, firmware 107 |
| STEPFOC stepper controller | 14-bit, 16384 ticks per revolution | 14 (the default) | [Spectral-BLDC](https://github.com/Source-Robotics/Spectral-BLDC-Python) 1.20 or newer, firmware 108 |

The toolbox only does the arithmetic, so another encoder works with its own resolution in bits. Older firmware: the
compatibility pages of the
[Spectral Micro](https://docs.source-robotics.com/Spectral-BLDC-docs/latest/appendix/compatibility/) and
[STEPFOC](https://docs.source-robotics.com/STEPFOC-docs/latest/appendix/compatibility/) manuals.

## Examples

In [Examples](https://github.com/Source-Robotics/Source-Robotics-Toolbox/tree/main/Examples):

| Example | What it does |
|---|---|
| `basic.py` | the quick start: one joint with a 6:1 reduction and its master position at 14000 ticks, fed an emulated encoder reading of 10000 ticks; runs sector homing once and prints the joint position in radians every 0.5 s |
| `two_spectral_micro.py` | two Spectral Micro drivers (node IDs 0 and 1) on one CAN bus with an slcan adapter (`COM41`, 1 Mbit/s): asks both for their encoder data, runs sector homing on each joint's first reading and prints the encoder ticks and the joint positions (reductions 3:1 and 0.5:1, the first joint offset by −π/2); needs Spectral-BLDC and numpy |

The toolbox at work in the manuals and the Spectral-BLDC examples:

| Where | What |
|---|---|
| [Bilateral teleoperation guide](https://docs.source-robotics.com/Spectral-BLDC-docs/latest/guides/bilateral-teleoperation/) | two motors coupled so that each follows the other and feels its force; Spectral Micro manual |
| [Quadruped guide](https://docs.source-robotics.com/Spectral-BLDC-docs/latest/guides/quadruped/) | one leg of a quadruped robot, two motors on one CAN bus; Spectral Micro manual |
| [SSG-48 teleoperation](https://docs.source-robotics.com/SSG48-gripper-docs/latest/guides/teleoperation/) and [MSG teleoperation](https://docs.source-robotics.com/MSG-gripper-docs/latest/guides/teleoperation/) | two grippers on one CAN bus, each following the other, with force feedback; the gripper manuals |
| [Spectral-BLDC advanced examples](https://github.com/Source-Robotics/Spectral-BLDC-Python/tree/main/examples/Advanced) | `Bilateral teleop 2DOF` and `SSG48 gripper bilateral teleop` |

## API reference

<details>
<summary>The <code>Joint</code> class: every call, its arguments and what it returns</summary>

### `Joint(encoder_resolution=14, master_position=9569, gear_ratio=1, offset=0, dir=0)`

- `encoder_resolution`: the encoder's resolution in bits; 14 for 16384 ticks per revolution.
- `master_position`: the encoder reading, in ticks within one motor revolution, at which the joint is at its zero.
- `gear_ratio`: the joint's reduction, 6 for 6:1.
- `offset`: the joint's angle at the master position, in radians; change it if your motor is not at 0 rad at the
  master position.
- `dir`: 0, or 1 to reverse the joint's direction.

### `determine_sector(initial)`
Sector homing: finds the sector from the first encoder reading after power-up. Call it once.

### `get_joint_position(motor_position)` → `float`
The joint position in radians, after the gear reduction, from the motor's encoder ticks.

### `get_joint_speed(motor_speed)` → `float`
The joint speed in rad/s, after the gear reduction, from the motor's speed in ticks/s.

### `get_encoder_position(joint_position)` → `int`
The motor's encoder ticks for a joint position in radians.

### `get_encoder_speed(joint_speed)` → `float`
The motor's speed in ticks/s for a joint speed in rad/s (marked "TODO check" in the code).

### `ticks_to_radians(ticks)`, `radians_to_ticks(radians)`, `ticks_to_degrees(ticks)`, `degrees_to_ticks(degrees)`
Conversions at the joint, through the gear ratio; the ticks are the motor's.

### `unwrap_position(position_ticks)` → `int`
The position within one motor revolution, 0 to 16383 for a 14-bit encoder.

</details>

## Community

Ask in [Discussions](https://github.com/orgs/Source-Robotics/discussions), chat on
[Discord](https://discord.com/invite/prjUvjmGpZ), report a bug in
[Issues](https://github.com/Source-Robotics/Source-Robotics-Toolbox/issues). Follow Source Robotics:

[![Discord](https://img.shields.io/discord/1072498136284667955?style=flat-square&logo=discord&logoColor=white&label=discord&color=143952)](https://discord.com/invite/prjUvjmGpZ)
[![Discussions](https://img.shields.io/badge/discussions-ask_a_question-143952?style=flat-square&logo=github&logoColor=white)](https://github.com/orgs/Source-Robotics/discussions)
[![YouTube](https://img.shields.io/badge/youtube-%40source--robotics-143952?style=flat-square&logo=youtube&logoColor=white)](https://www.youtube.com/@source-robotics)
[![Instagram](https://img.shields.io/badge/instagram-%40source__robotics-143952?style=flat-square&logo=instagram&logoColor=white)](https://www.instagram.com/source_robotics/)
[![X](https://img.shields.io/badge/x-%40SourceRobotics-143952?style=flat-square&logo=x&logoColor=white)](https://x.com/SourceRobotics)
[![LinkedIn](https://img.shields.io/badge/linkedin-Source_Robotics-143952?style=flat-square&logo=data%3Aimage%2Fsvg%2Bxml%3Bbase64%2CPHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI%2BPHBhdGggZmlsbD0id2hpdGUiIGQ9Ik0yMC40NDcgMjAuNDUyaC0zLjU1NHYtNS41NjljMC0xLjMyOC0uMDI3LTMuMDM3LTEuODUyLTMuMDM3LTEuODUzIDAtMi4xMzYgMS40NDUtMi4xMzYgMi45Mzl2NS42NjdIOS4zNTFWOWgzLjQxNHYxLjU2MWguMDQ2Yy40NzctLjkgMS42MzctMS44NSAzLjM3LTEuODUgMy42MDEgMCA0LjI2NyAyLjM3IDQuMjY3IDUuNDU1djYuMjg2ek01LjMzNyA3LjQzM2MtMS4xNDQgMC0yLjA2My0uOTI2LTIuMDYzLTIuMDY1IDAtMS4xMzguOTItMi4wNjMgMi4wNjMtMi4wNjMgMS4xNCAwIDIuMDY0LjkyNSAyLjA2NCAyLjA2MyAwIDEuMTM5LS45MjUgMi4wNjUtMi4wNjQgMi4wNjV6bTEuNzgyIDEzLjAxOUgzLjU1NVY5aDMuNTY0djExLjQ1MnpNMjIuMjI1IDBIMS43NzFDLjc5MiAwIDAgLjc3NCAwIDEuNzI5djIwLjU0MkMwIDIzLjIyNy43OTIgMjQgMS43NzEgMjRoMjAuNDUxQzIzLjIgMjQgMjQgMjMuMjI3IDI0IDIyLjI3MVYxLjcyOUMyNCAuNzc0IDIzLjIgMCAyMi4yMjIgMGguMDAzeiIvPjwvc3ZnPg%3D%3D)](https://www.linkedin.com/company/source-robotics/)
[![Blog](https://img.shields.io/badge/blog-source--robotics.com-143952?style=flat-square&logo=rss&logoColor=white)](https://source-robotics.com/blogs/blog)

## Safety, liability and disclaimer

1. The software and hardware are still in development and may contain bugs, errors, or incomplete features.
2. Users are encouraged to use this software and hardware responsibly and at their own risk.

## Licensing

Project is under [MIT Licence](https://github.com/Source-Robotics/Source-Robotics-Toolbox/blob/main/LICENSE)
