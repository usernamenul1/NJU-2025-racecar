# 智能车仿真实验报告

## 第1章 方案概述

本方案旨在基于ROS Noetic（Robot Operating System）和Gazebo仿真环境，构建一套智能赛车（Racecar）的自动驾驶仿真系统。该系统针对阿克曼转向（Ackermann Steering）底盘模型进行开发，实现了环境感知、SLAM建图（Simultaneous Localization and Mapping）以及自主导航功能。

本项目的主要技术路线采用经典的ROS导航栈（Navigation Stack），并针对类车机器人（Car-like Robot）的运动学约束进行了针对性配置（如使用TEB局部规划器）。创新点在于实现了从旧版ROS Kinetic到Ubuntu 20.04/ROS Noetic环境的完整迁移与适配，并提供了跑道（Runway）与隧道（Tunnel）多种仿真场景，验证了算法在封闭道路环境下的有效性。

## 第2章 问题描述

在智能车自动驾驶的研究中，主要面临以下关键性问题：

1.  **非完整约束控制**：与差速驱动机器人不同，Racecar模型具有阿克曼转向几何特性，无法原地旋转，存在最小转弯半径限制。
2.  **未知环境建图**：需要在没有先验地图的情况下，利用机载传感器（仿真激光雷达）构建高精度的环境栅格地图。
3.  **动态路径规划**：在规划路径时，不仅要避开静态障碍物，还必须保证生成的路径符合车辆的运动学约束（即曲率连续且不超过最大转向角）。

针对上述问题，本方案采用了基于滤波的SLAM算法解决建图问题，并引入时间弹性带（Timed Elastic Band, TEB）算法优化局部路径规划，以适配阿克曼底盘的运动特性。

## 第3章 技术方案

本系统采用分层架构设计，主要包含驱动层（仿真）、感知层、决策规划层和控制层。

### 3.1 车辆运动学模型
采用阿克曼转向几何模型。车辆后轮驱动，前轮转向。其运动学方程描述了车辆中心位置 $(x, y)$、航向角 $\theta$ 与速度 $v$、转向角 $\delta$ 之间的关系。该模型作为底层控制（Controller）和上层规划（Planner）的基础约束。

### 3.2 SLAM建图算法
采用 **Gmapping** 算法。这是一种基于Rao-Blackwellized粒子滤波器（RBPF）的2D激光SLAM算法。
-   **原理**：将定位问题和建图问题分解，先通过里程计运动模型预测粒子分布，再利用激光雷达观测数据更新粒子权重，最后进行重采样。
-   **优势**：在计算量和建图精度之间取得了较好的平衡，适合构建中小范围的室内或赛道地图。

### 3.3 导航与规划框架
基于 `move_base` 框架实现自主导航：
1.  **全局规划（Global Planner）**：使用全局路径规划器（如Global Planner/Dijkstra）基于静态地图计算从起点到终点的最优路径。
2.  **定位（Localization）**：采用 **AMCL**（自适应蒙特卡洛定位）算法，我们在 `racecar_ws` 中通过安装 `ros-noetic-amcl` 支持该功能。
3.  **局部规划（Local Planner）**：采用 **TEB Local Planner** (`ros-noetic-teb-local-planner`)。
    -   **算法思路**：TEB算法将路径规划问题转化为多目标优化问题。它在优化过程中显式考虑了车辆的运动学约束（如最大转向角、最大速度、加速度），能够生成适合阿克曼小车执行的平滑轨迹，避免了传统DWA算法在窄道或急弯处容易卡死的问题。

## 第4章 方案实现

### 4.1 环境搭建与依赖安装
由于系统环境为Ubuntu 20.04，首先完成了ROS Noetic的部署。工程实现中通过 `apt-get` 安装了关键依赖库：
-   控制器：`ros-noetic-ros-control`, `ros-noetic-tem-local-planner`
-   SLAM与导航：`ros-noetic-gmapping`, `ros-noetic-move-base`

### 4.2 工作空间构建
项目代码组织在 `racecar_ws` 工作空间下：
-   `src/racecar_description`：定义URDF模型文件，描述车辆物理属性。
-   `src/racecar_gazebo`：包含Gazebo世界文件（`.world`）及Launch启动脚本。
-   `src/system`：包含底盘通信与传感器驱动逻辑。

编译流程如下：
```bash
cd ~/Desktop/myracecar/racecar_ws
source /opt/ros/noetic/setup.bash
catkin_make
source devel/setup.bash
```

### 4.3 功能模块实现
1.  **仿真启动**：通过 `roslaunch racecar_gazebo racecar_runway.launch` 加载Gazebo物理引擎、车辆模型及控制器。
2.  **建图实现**：利用 `gmapping.launch` 启动SLAM节点，订阅 `/scan` 激光雷达数据和 `/tf` 坐标变换，输出占用栅格地图。
3.  **自主导航**：配置 `move_base` 参数，加载 `teb_local_planner_params.yaml`，启动 `racecar_runway_navigation.launch` 实现从地图加载、定位到路径规划的全流程。

## 第5章 测试分析

### 5.1 测试环境
-   **操作系统**：Ubuntu 20.04 LTS
-   **软件版本**：ROS Noetic, Gazebo 11, Rviz
-   **硬件平台**：PC仿真

### 5.2 基础控制测试
启动仿真后，通过键盘控制节点发送 `cmd_vel` 指令。
-   **操作**：按下W/S/A/D键。
-   **结果**：观察到Gazebo中车辆能够响应加速、减速及转向指令，车轮转动方向与阿克曼几何一致，里程计数据更新正常。

### 5.3 建图效果测试
在跑道场景下启动Gmapping，控制小车低速行驶一圈。
-   **过程**：在Rviz中实时观察地图构建过程。
-   **结果**：生成了清晰的赛道边缘栅格地图，闭环处无明显重影，证明里程计与激光雷达外参标定准确，SLAM算法参数（粒子数、更新阈值）设置合理。

### 5.4 导航避障测试
启动 `racecar_runway_navigation.launch`。
-   **输入**：在Rviz中使用 "2D Nav Goal" 指定赛道终点。
-   **过程**：
    1.  全局路径（绿线）迅速生成。
    2.  局部路径（红线）动态调整，且始终保持平滑弧线，符合车辆转向限制。
    3.  小车沿路径行驶，自动修正偏差。
-   **结论**：车辆成功到达目标点，且在弯道处能够自动减速并调整转向角，TEB规划器有效发挥了作用。

## 第6章 作品总结

本项目成功在ROS Noetic环境下构建了一套功能完善的智能车仿真系统。
1.  **创新与工作量**：完成了跨ROS版本的代码迁移与依赖修复，整合了阿克曼运动学模型与现代导航算法栈。
2.  **技术路线**：选择TEB规划器是解决类车机器人导航的关键决策，确保证了仿真与实际物理特性的一致性。
3.  **效果**：通过Gazebo与Rviz的联合调试，验证了建图的完整性和导航的鲁棒性。

**展望**：未来可将此仿真算法一直部署到搭载Jetson Nano或树莓派的实体Racecar硬件上，并引入摄像头视觉信息，通过融合视觉与激光雷达数据进一步提升复杂环境下的感知能力。

## 参考文献

1.  Quigley, M., et al. "ROS: an open-source Robot Operating System." ICRA workshop on open source software. Vol. 3. No. 3.2. 2009.
2.  Grisetti, Giorgio, Cyrill Stachniss, and Wolfram Burgard. "Improved techniques for grid mapping with rao-blackwellized particle filters." IEEE transactions on Robotics 23.1 (2007): 34.
3.  Rösmann, Christoph, et al. "Kinodynamic trajectory optimization and control for car-like robots." 2012 IEEE/RSJ International Conference on Intelligent Robots and Systems. IEEE, 2012.
4.  Fox, Dieter. "KLD-sampling: Adaptive particle filters." Advances in neural information processing systems. 2002.
