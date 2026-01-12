# 智能车仿真 (ROS Noetic版本)

本项目已从ROS Kinetic迁移到ROS Noetic (Ubuntu 20.04)。

## 环境要求
- Ubuntu 20.04 LTS
- ROS Noetic
- Gazebo 11

## 安装依赖

```bash
# 安装ROS Noetic控制器相关包
sudo apt-get install ros-noetic-controller-manager
sudo apt-get install ros-noetic-gazebo-ros-control
sudo apt-get install ros-noetic-effort-controllers
sudo apt-get install ros-noetic-joint-state-controller

# 安装ackermann消息包
sudo apt-get install ros-noetic-ackermann-msgs

# 安装SLAM和导航相关包
sudo apt-get install ros-noetic-gmapping
sudo apt-get install ros-noetic-amcl
sudo apt-get install ros-noetic-map-server
sudo apt-get install ros-noetic-move-base
sudo apt-get install ros-noetic-global-planner
sudo apt-get install ros-noetic-teb-local-planner

# 安装rtabmap (可选，用于SLAM)
sudo apt-get install ros-noetic-rtabmap-ros
```

## 编译步骤

```bash
# 1. 进入工作空间目录
cd ~/Desktop/myracecar/racecar_ws

# 2. 加载ROS环境
source /opt/ros/noetic/setup.bash

# 3. 编译工作空间
catkin_make

# 4. 加载工作空间环境
source devel/setup.bash
```

## 运行仿真

### 方式1：跑道场景
```bash
# 终端1：启动仿真环境
source /opt/ros/noetic/setup.bash
source ~/Desktop/myracecar/racecar_ws/devel/setup.bash
roslaunch racecar_gazebo racecar_runway.launch
```

### 方式2：隧道场景
```bash
roslaunch racecar_gazebo racecar_tunnel.launch
```

### 键盘控制
启动仿真后，可以使用键盘控制小车：
- W: 前进
- S: 后退
- A: 左转
- D: 右转
- Q: 退出

### SLAM建图
```bash
# 终端1：启动仿真
roslaunch racecar_gazebo racecar_runway.launch

# 终端2：启动gmapping
roslaunch racecar_gazebo gmapping.launch
```

### 导航
导航功能需要配合RViz使用，通过设置目标点来实现自主导航。

```bash
# 终端1：启动导航（已包含仿真环境、地图服务器和move_base）
roslaunch racecar_gazebo racecar_runway_navigation.launch

# 终端2：启动RViz可视化
roslaunch racecar_gazebo racecar_rviz.launch
```

**使用步骤：**
1. 启动导航launch文件后，Gazebo仿真环境会自动加载
2. 在另一个终端启动RViz
3. 在RViz中使用"2D Nav Goal"工具（工具栏中的箭头图标）
4. 在地图上点击并拖动设置目标位置和朝向
5. 小车会自动规划路径并导航到目标点

## ROS Kinetic到Noetic的主要变更

1. **Python版本**: 从Python 2升级到Python 3
   - 所有Python脚本shebang改为 `#!/usr/bin/env python3`
   - `Tkinter` 改为 `tkinter`
   - `print` 语句改为 `print()` 函数
   - 整数除法 `/` 改为 `//`

2. **CMakeLists.txt**: 
   - `cmake_minimum_required(VERSION 2.8.3)` 改为 `cmake_minimum_required(VERSION 3.0.2)`

3. **package.xml**: 
   - 使用 format 2 (`<package format="2">`)
   - `run_depend` 改为 `exec_depend`

4. **OpenCV**: 
   - 移除了固定的OpenCV路径设置
   - 使用系统OpenCV (`find_package(OpenCV REQUIRED)`)
   - 头文件从 `<opencv/highgui.h>` 改为 `<opencv2/highgui.hpp>`

5. **不兼容的包**: 
   - `hokuyo_node` 已禁用（依赖已废弃的driver_base包）
   - 仿真中使用Gazebo的激光雷达插件替代

## 故障排查

### 报错找不到Python脚本节点
确保脚本有执行权限：
```bash
chmod +x ~/Desktop/myracecar/racecar_ws/src/racecar_gazebo/scripts/*.py
chmod +x ~/Desktop/myracecar/racecar_ws/src/racecar_control/scripts/*.py
```

### 报错找不到控制器类型
安装effort_controllers：
```bash
sudo apt-get install ros-noetic-effort-controllers
```

## 参考教程
原始代码使用参考教程：[教程地址](https://www.guyuehome.com/6463)

