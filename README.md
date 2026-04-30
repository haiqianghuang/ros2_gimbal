# control_gimbal_node

`control_gimbal_node` 是一个 ROS 2 云台控制节点，基于仓库内的底层 `components/control/gimbal` C 驱动封装而成。

当前节点默认对接 `drv_udp_TZ0xxx` UDP 驱动，并在包内直接提供统一的 ROS 2 消息与服务接口。

## 功能概览

- 发布云台状态话题 `/gimbal/state`
- 提供云台模式切换服务 `/gimbal/set_mode`
- 提供目标角度/角速度设置服务 `/gimbal/set_target`
- 提供软限位设置服务 `/gimbal/set_limits`
- 提供变倍控制服务 `/gimbal/set_zoom`
- 支持直接链接已安装的 `libgimbal.so`
- 若系统中未预装 `gimbal` 库，可回退为使用仓库内 `components/control/gimbal` 源码直接构建

## 目录结构

```text
middleware/ros2/control/gimbal/
├── CMakeLists.txt
├── package.xml
├── README.md
├── msg/
│   ├── GimbalEuler.msg
│   └── GimbalState.msg
├── srv/
│   ├── SetGimbalLimits.srv
│   ├── SetGimbalMode.srv
│   ├── SetGimbalTarget.srv
│   └── SetGimbalZoom.srv
└── src/
    └── gimbal_server_node.cpp
```

相关依赖：

- `middleware/ros2/control/gimbal/msg`、`middleware/ros2/control/gimbal/srv`: ROS 2 消息与服务定义
- `components/control/gimbal`: 底层云台驱动实现

## 依赖

### ROS 2 依赖

- `rclcpp`
- `rosidl_default_generators`
- `std_msgs`
- `ament_cmake`

### 系统依赖

- `Threads`
- `libm`
- ROS 2 Humble 或兼容版本

### 底层驱动依赖

默认驱动名：`drv_udp_TZ0xxx`

如果运行时报错：

```text
[GIMBAL] Driver not found: drv_udp_TZ0xxx
```

通常说明可执行文件没有正确链接驱动实现，或者底层 `gimbal` 组件没有按预期安装/编译。

## 构建方式

推荐在仓库根目录构建，不要在当前包目录里直接裸跑 `colcon build`。

### 方式一：使用仓库统一构建脚本

在仓库根目录：

```bash
source build/envsetup.sh
export ROS_DISTRO=humble
export ROS_SETUP=/opt/ros/humble/setup.bash

./build/build.sh ros2
```

如果只编译当前相关包：

```bash
source build/envsetup.sh
export ROS_DISTRO=humble
export ROS_SETUP=/opt/ros/humble/setup.bash

./build/build.sh package middleware/ros2/control/gimbal
```

### 方式二：直接用 colcon 构建相关包

```bash
cd /path/to/robotic_sdk
source /opt/ros/humble/setup.bash

colcon build \
  --merge-install \
  --install-base /path/to/robotic_sdk/output/staging \
  --build-base /path/to/robotic_sdk/output/build/ros2/middleware \
  --base-paths middleware/ros2/control/gimbal \
  --packages-select control_gimbal_node
```

### 构建成功后的检查项

```bash
ls output/staging/share/control_gimbal_node/package.sh
ls output/staging/share/control_gimbal_node/cmake/control_gimbal_nodeConfig.cmake
ls output/staging/lib/control_gimbal_node/gimbal_server_node
```

如果上述文件存在，说明接口定义和节点已经正确安装到 `output/staging`。

## 运行方式

```bash
cd /path/to/robotic_sdk
source /opt/ros/humble/setup.bash
source output/staging/setup.bash

ros2 run control_gimbal_node gimbal_server_node \
  --ros-args \
  -p bind_ip:=0.0.0.0 \
  -p bind_port:=4900 \
  -p device_ip:=192.168.44.160 \
  -p device_port:=4900
```

正常启动后，终端会输出节点 ready 日志，类似：

```text
gimbal_server_node ready: driver=drv_udp_TZ0xxx local=0.0.0.0:4900 remote=192.168.44.160:4900
```

## 参数说明

节点在启动时支持以下 ROS 参数：

| 参数名 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `driver_name` | `string` | `drv_udp_TZ0xxx` | 底层驱动名称 |
| `bind_ip` | `string` | `0.0.0.0` | 本地绑定 IP |
| `bind_port` | `int` | `4900` | 本地绑定端口 |
| `device_ip` | `string` | `192.168.44.160` | 云台设备 IP |
| `device_port` | `int` | `4900` | 云台设备端口 |
| `resend_period_s` | `double` | `0.02` | 速度控制命令重发周期 |
| `tick_hz` | `double` | `50.0` | 底层驱动 tick 频率 |
| `state_publish_hz` | `double` | `20.0` | 状态发布频率 |
| `stable_threshold_deg` | `double` | `1.5` | 判定稳定的角度阈值 |
| `frame_id` | `string` | `base_link` | 状态消息坐标系 |
| `use_limits` | `bool` | `false` | 启动时是否启用限位 |
| `min_angle_deg` | `double[]` | `[-120,-179,-180]` | 最小角度限位 |
| `max_angle_deg` | `double[]` | `[90,179,180]` | 最大角度限位 |
| `max_speed_deg_s` | `double[]` | `[50,50,50]` | 最大速度限位 |

说明：

- `min_angle_deg`、`max_angle_deg`、`max_speed_deg_s` 支持长度为 3 的数组。
- 若只传 1 个值，节点会自动扩展到 3 个轴。

## ROS 接口

### 话题

#### `/gimbal/state`

消息类型：`control_gimbal_node/msg/GimbalState`

字段说明：

- `mode`: 当前云台模式
- `has_feedback`: 是否收到有效反馈
- `has_target`: 是否已有目标值
- `stable`: 是否已稳定
- `status_code`: 底层状态码
- `angle`: 当前角度
- `speed`: 当前角速度
- `target`: 最近一次目标值

查看话题：

```bash
ros2 topic echo /gimbal/state
```

### 服务

#### `/gimbal/set_mode`

服务类型：`control_gimbal_node/srv/SetGimbalMode`

模式枚举：

- `0`: `MODE_OFF`
- `1`: `MODE_ANGLE_ABS`
- `2`: `MODE_ANGLE_REL`
- `3`: `MODE_SPEED`
- `4`: `MODE_FOLLOW`
- `5`: `MODE_FPV`
- `6`: `MODE_LOCK`
- `7`: `MODE_CALIBRATE`

示例：设置绝对角模式

```bash
ros2 service call /gimbal/set_mode control_gimbal_node/srv/SetGimbalMode "{mode: 1}"
```

#### `/gimbal/set_target`

服务类型：`control_gimbal_node/srv/SetGimbalTarget`

示例：设置目标角度

```bash
ros2 service call /gimbal/set_target control_gimbal_node/srv/SetGimbalTarget \
  "{target: {pitch: 10.0, yaw: 0.0, roll: 0.0}}"
```

说明：

- 在 `MODE_ANGLE_ABS` 下，目标值表示绝对角度。
- 在 `MODE_ANGLE_REL` 下，目标值表示相对当前姿态的增量。
- 在 `MODE_SPEED` 下，目标值表示角速度指令。

#### `/gimbal/set_limits`

服务类型：`control_gimbal_node/srv/SetGimbalLimits`

示例：设置角度和速度限位

```bash
ros2 service call /gimbal/set_limits control_gimbal_node/srv/SetGimbalLimits "{
  min_angle: {pitch: -30.0, yaw: -90.0, roll: -10.0},
  max_angle: {pitch: 30.0, yaw: 90.0, roll: 10.0},
  max_speed: {pitch: 20.0, yaw: 20.0, roll: 20.0}
}"
```

#### `/gimbal/set_zoom`

服务类型：`control_gimbal_node/srv/SetGimbalZoom`

缩放枚举：

- `0`: `ZOOM_STOP`
- `1`: `ZOOM_IN`
- `2`: `ZOOM_OUT`

示例：放大

```bash
ros2 service call /gimbal/set_zoom control_gimbal_node/srv/SetGimbalZoom \
  "{direction: 1, speed_level: 2}"
```

## 运行验证

当前包没有定义 `ament_add_gtest()`、`add_test()` 或 `launch_testing`，因此没有现成的自动化测试用例。

建议采用以下最小验证流程：

1. 确认节点启动成功。
2. 确认 `/gimbal/state` 持续发布。
3. 调用 `/gimbal/set_mode` 并观察返回值是否为 `success: true`。
4. 调用 `/gimbal/set_target`，再观察 `/gimbal/state` 中的 `has_target`、`target`、`status_code` 是否变化。
5. 如设备在线，确认云台实体动作与命令一致。

验证命令：

```bash
ros2 node list
ros2 topic list | grep gimbal
ros2 service list | grep gimbal
ros2 topic echo /gimbal/state
```

## 常见问题

### 1. 找不到 `control_gimbal_node`

报错示例：

```text
Could not find a package configuration file provided by "control_gimbal_node"
```

处理方法：

- 确保已构建 `middleware/ros2/control/gimbal`
- 确保构建安装前缀统一为 `output/staging`
- 运行前执行：

```bash
source /opt/ros/humble/setup.bash
source output/staging/setup.bash
```

### 2. 缺少 `package.sh`

报错示例：

```text
Failed to find the following files:
.../share/control_gimbal_node/package.sh
```

处理方法：

- 不要只在 `middleware/ros2/control/gimbal` 目录里单独裸跑 `colcon build`
- 构建时把 `middleware/ros2/control/gimbal` 纳入 `--base-paths`
- 或先单独编译 `control_gimbal_node`

### 3. 驱动找不到

报错示例：

```text
[GIMBAL] Driver not found: drv_udp_TZ0xxx
[GIMBAL] No driver found: drv_udp_TZ0xxx
```

处理方法：

- 确保当前版本已经包含驱动源码链接修复
- 重新编译 `control_gimbal_node`
- 若系统已安装 `libgimbal.so`，检查其是否确实包含 `drv_udp_TZ0xxx`

### 4. 节点启动了，但没有反馈

可能原因：

- `device_ip` 或 `device_port` 配置错误
- 本机绑定 IP/端口不对
- 云台设备未上电或网络不通
- 底层协议不匹配

建议检查：

```bash
ping 192.168.44.160
```

并确认设备网络拓扑、端口、防火墙设置正确。

## 开发说明

当前节点实现位于：

- `src/gimbal_server_node.cpp`

核心行为：

- 周期调用 `gimbal_tick()` 驱动底层状态机
- 周期调用 `gimbal_get_state()` 发布状态
- 服务回调内调用底层 `gimbal_set_mode()`、`gimbal_set_target()`、`gimbal_set_limits()`、`gimbal_set_zoom()`

## 后续可补充项

当前 README 已覆盖基础使用。若后续继续完善，建议增加：

- launch 文件
- 参数 YAML 示例
- 真机联调示意图
- 自动化测试或 launch_testing 用例
- 不同云台型号的驱动适配说明
