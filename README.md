# R2 ROS2 Navigation Workspace

这是 R2 机器人导航工作区，基于 Ubuntu 22.04 + ROS 2 Humble。工程整合了 Odin1 定位驱动、OctoMap 三维地图与路径规划、底盘串口速度控制，以及面向 Web/Action 的导航启动入口。

## 目录结构

```text
r2_nav_ws2/
├── chassis_maps/                  # 本机地图数据
├── src/
│   ├── odin_ros_driver/           # Odin1 ROS2 驱动
│   ├── jie_3d_nav/
│   │   ├── jie_map_msgs/          # 地图保存/加载等接口
│   │   ├── jie_octomap/           # OctoMap 导入、管理、Web/GUI 工具
│   │   └── octo_planner/          # 三维路径规划与路径跟踪控制
│   ├── r2_nav_interfaces/         # R2 导航 Action 接口
│   ├── r2_nav_action_server/      # R2 NavigateToPose Action 服务端
│   ├── r2_nav_bringup/            # 整机启动文件和导航配置
│   └── serial_communication_pkg/  # /cmd_vel 到串口协议转换
├── build/                         # colcon 编译产物，已忽略
├── install/                       # colcon 安装产物，已忽略
└── log/                           # colcon 日志，已忽略
```

## 环境要求

- Ubuntu 22.04
- ROS 2 Humble
- colcon
- OpenCV、PCL、Eigen、OctoMap、Open3D
- rosbridge_server、rviz2
- Python 依赖：PyYAML、PyQt5、VTK、NumPy、Pillow、pyserial

安装三维导航相关依赖：

```bash
cd /home/xie/techx_R2_algorithm/r2_nav_ws2/src/jie_3d_nav
bash install_deps_humble.sh
```

## 编译

从工作区根目录编译全部包：

```bash
cd /home/xie/techx_R2_algorithm/r2_nav_ws2
source /opt/ros/humble/setup.bash
colcon build
source install/setup.bash
```

只编译核心导航相关包：

```bash
colcon build --packages-select \
  odin_ros_driver \
  jie_map_msgs \
  jie_octomap \
  octo_planner \
  r2_nav_interfaces \
  r2_nav_action_server \
  r2_nav_bringup \
  serial_communication_pkg
```

如果只想重编 Odin 驱动：

```bash
colcon build --packages-select odin_ros_driver \
  --cmake-args -DBUILD_SYSTEM=ROS2 -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

如果只是想隐藏 CMake 的开发者 warning：

```bash
colcon build --cmake-args -Wno-dev
```

## 常用启动

启动 R2 + Odin1 + 三维地图 + Web 导航整套流程：

```bash
cd /home/xie/techx_R2_algorithm/r2_nav_ws2
source install/setup.bash
ros2 launch r2_nav_bringup r2_odin_web_nav.launch.py
```

常用参数：

```bash
ros2 launch r2_nav_bringup r2_odin_web_nav.launch.py \
  launch_rviz:=true \
  launch_web:=true \
  launch_rosbridge:=true \
  launch_serial:=true \
  web_http_port:=8080
```

只启动 Web 手动控制 + 串口控制：

```bash
ros2 launch r2_nav_bringup r2_web_serial_control.launch.py
```

只启动串口速度转换节点：

```bash
ros2 launch serial_communication_pkg cmd_vel_to_serial.launch.py
```

Odin1 驱动单独启动：

```bash
ros2 launch odin_ros_driver odin1_ros2.launch.py
```

## 关键配置

整机导航配置：

```text
src/r2_nav_bringup/config/r2_nav_params.yaml
```

这里主要配置：

- Odin1 重定位 `.bin` 地图路径
- Web/RViz 可视化 `.pcd` 地图路径
- OctoMap 地图包目录
- `odin1_base_link -> chassis_base_link` 静态 TF
- 规划器参数
- 路径跟踪控制参数
- Action 服务参数

串口配置：

```text
src/serial_communication_pkg/config/serial_params.yaml
```

默认串口为：

```yaml
serial_port: "/dev/ttyUSB0"
baud_rate: 115200
```

如果串口设备名变化，先查看：

```bash
ls /dev/ttyUSB*
```

然后修改 `serial_params.yaml`。

## Web 与 Action

默认 Web 端口：

```text
http://localhost:8080
```

ROSBridge 默认 websocket：

```text
ws://localhost:9090
```

R2 导航 Action 名称：

```text
/r2_navigate_to_pose
```

Action 接口定义位于：

```text
src/r2_nav_interfaces/action/NavigateToPose.action
```

## 地图与 Odin1 注意事项

启动 `r2_odin_web_nav.launch.py` 前，请确认 `r2_nav_params.yaml` 中这些路径在当前电脑上真实存在：

```yaml
maps:
  relocalization_bin_file: "..."
  relocalization_pcd_file: "..."
  map_package_dir: "..."
```

Odin USB 权限建议配置 udev 规则，避免每次使用 sudo。参考：

```text
src/odin_ros_driver/README.md
```

如果 Odin 驱动运行时报 USB 权限或设备忙，先检查：

```bash
lsusb
ps aux | grep host_sdk_sample
```

## Git 日常工作流

查看当前状态：

```bash
git status
```

提交修改：

```bash
git add .
git commit -m "说明本次修改"
git push
```

拉取远程更新：

```bash
git pull --no-rebase
```

查看提交历史：

```bash
git log --oneline --graph --decorate -n 20
```

撤销未提交的单个文件修改：

```bash
git restore path/to/file
```

回滚已经提交并推送过的提交：

```bash
git revert <commit-id>
git push
```

## 常见现象

编译时看到这类 warning 通常可以忽略：

```text
Policy CMP0148 is not set
The FindPythonInterp and FindPythonLibs modules are removed
```

这是 ROS Humble 的接口生成脚本在新 CMake 下触发的开发者警告，不代表编译失败。

判断是否编译成功，看最后是否有：

```text
Summary: N packages finished
```

Odin 驱动编译时如果看到 OpenCV 版本冲突 warning，通常不影响编译；如果图像相关节点运行时崩溃，再优先排查 OpenCV 环境。

## 清理与重编

普通重编：

```bash
colcon build
```

清 CMake 缓存重编：

```bash
colcon build --cmake-clean-cache
```

彻底清理编译产物：

```bash
rm -rf build install log
colcon build
```
