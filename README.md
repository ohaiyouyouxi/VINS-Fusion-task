# VINS-Fusion-task
## 1. 环境配置

本次使用的环境：

| 环境 | 版本 |
| --- | --- |
| 操作系统 | Ubuntu 20.04 |
| ROS | Noetic |
| OpenCV | 4.2 |
| SLAM 框架 | VINS-Fusion |
| 数据集 | EuRoC MAV / KITTI |

VINS-Fusion 是香港科技大学开源的视觉惯性状态估计框架，支持单目 + IMU、双目、双目 + IMU，以及 GPS 融合等功能。

### 1.1 安装依赖

首先需要安装 ROS Noetic，以及 Ceres Solver、Eigen3、OpenCV 等依赖。

```bash
sudo apt update
sudo apt install -y git cmake build-essential \
    libeigen3-dev libceres-dev libopencv-dev
```

这里默认已经完成 ROS Noetic 的安装。

### 1.2 下载 VINS-Fusion

进入 ROS 工作空间：

```bash
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/src

git clone https://github.com/HKUST-Aerial-Robotics/VINS-Fusion.git
```

然后编译：

```bash
cd ~/catkin_ws
catkin_make
source devel/setup.bash
```

正常情况下应该可以编译完成，不过我在第一次编译时遇到了 OpenCV 相关的报错。

## 2. 遇到的问题及解决方法

### 2.1 OpenCV 版本兼容问题

编译到 `camera_models` 模块时出现大量错误，例如：

```text
error: 'CV_GRAY2RGB' was not declared in this scope
error: 'CV_AA' was not declared in this scope
error: 'CV_GRAY2BGR' was not declared in this scope
error: 'CV_BGR2GRAY' was not declared in this scope
error: 'CV_CALIB_CB_FAST_CHECK' was not declared in this scope
```

相关文件主要有：

```text
camera_models/src/calib/CameraCalibration.cc
camera_models/src/chessboard/Chessboard.cc
camera_models/src/intrinsic_calib.cc
```

查了一下发现，这是因为 VINS-Fusion 中使用了旧版本 OpenCV 的一些宏定义，而 Ubuntu 20.04 环境中的 OpenCV 4 已经不再支持这些旧宏。

### 2.2 解决方法

我的解决方法比较直接：**把旧的 OpenCV 宏替换成新的枚举常量。**

例如：

| 原来的写法 | 修改后的写法 |
| --- | --- |
| `CV_GRAY2RGB` | `cv::COLOR_GRAY2RGB` |
| `CV_GRAY2BGR` | `cv::COLOR_GRAY2BGR` |
| `CV_BGR2GRAY` | `cv::COLOR_BGR2GRAY` |
| `CV_AA` | `cv::LINE_AA` |
| `CV_THRESH_BINARY` | `cv::THRESH_BINARY` |
| `CV_THRESH_BINARY_INV` | `cv::THRESH_BINARY_INV` |
| `CV_ADAPTIVE_THRESH_MEAN_C` | `cv::ADAPTIVE_THRESH_MEAN_C` |
| `CV_SHAPE_CROSS` | `cv::MORPH_CROSS` |
| `CV_SHAPE_RECT` | `cv::MORPH_RECT` |
| `CV_TERMCRIT_EPS` | `cv::TermCriteria::EPS` |
| `CV_TERMCRIT_ITER` | `cv::TermCriteria::COUNT` |
| `CV_RETR_CCOMP` | `cv::RETR_CCOMP` |
| `CV_CHAIN_APPROX_SIMPLE` | `cv::CHAIN_APPROX_SIMPLE` |

另外，棋盘格标定相关宏也需要修改：

```cpp
CV_CALIB_CB_ADAPTIVE_THRESH
// 改成
cv::CALIB_CB_ADAPTIVE_THRESH

CV_CALIB_CB_NORMALIZE_IMAGE
// 改成
cv::CALIB_CB_NORMALIZE_IMAGE

CV_CALIB_CB_FILTER_QUADS
// 改成
cv::CALIB_CB_FILTER_QUADS

CV_CALIB_CB_FAST_CHECK
// 改成
cv::CALIB_CB_FAST_CHECK
```

主要修改了 `Chessboard.cc` 和 `CameraCalibration.cc`，同时处理了 `intrinsic_calib.cc` 中的旧宏。

修改时需要确认相关文件包含：

```cpp
#include <opencv2/imgproc.hpp>
#include <opencv2/calib3d.hpp>
```

修改完成后重新编译：

```bash
cd ~/catkin_ws
catkin_make
source devel/setup.bash
```

解决了这些兼容问题后，VINS-Fusion 就可以正常编译和运行了。

## 3. 运行测试

主要参考了 VINS-Fusion 官方 README 和网上的一些入门教程。

### 3.1 单目 + IMU

打开不同终端，分别运行：

```bash
roscore
```

启动 RViz：

```bash
roslaunch vins vins_rviz.launch
```

启动 VINS：

```bash
rosrun vins vins_node \
  ~/catkin_ws/src/VINS-Fusion/config/euroc/euroc_mono_imu_config.yaml
```

如果需要闭环检测，再启动：

```bash
rosrun loop_fusion loop_fusion_node \
  ~/catkin_ws/src/VINS-Fusion/config/euroc/euroc_mono_imu_config.yaml
```

最后播放数据集：

```bash
rosbag play ~/vicon_room1/V1_01_easy/V1_01_easy.bag
```

不启动 `loop_fusion_node` 就是开环测试，启动后可以观察闭环优化的效果。

### 3.2 双目 + IMU

与单目基本一致，只需要切换配置文件：

```bash
rosrun vins vins_node \
  ~/catkin_ws/src/VINS-Fusion/config/euroc/euroc_stereo_imu_config.yaml
```

需要闭环时：

```bash
rosrun loop_fusion loop_fusion_node \
  ~/catkin_ws/src/VINS-Fusion/config/euroc/euroc_stereo_imu_config.yaml
```

### 3.3 纯双目

```bash
rosrun vins vins_node \
  ~/catkin_ws/src/VINS-Fusion/config/euroc/euroc_stereo_config.yaml
```

### 3.4 双目 + GPS

这部分使用 KITTI Raw 数据集。

启动 RViz：

```bash
roslaunch vins vins_rviz.launch
```

启动 GPS 全局融合节点：

```bash
rosrun global_fusion global_fusion_node
```

运行 KITTI 测试程序：

```bash
rosrun vins kitti_gps_test \
  ~/catkin_ws/src/VINS-Fusion/config/kitti_raw/kitti_10_03_config.yaml \
  ~/2011_09_26/2011_09_26_drive_0005_sync/
```

## 4. 运行效果展示

这次主要测试了以下五种情况。

### 4.1 单目 + IMU（开环）

不使用闭环检测，仅依靠视觉和 IMU 信息估计运动轨迹。

🎬 [查看运行视频：单目 + IMU + 开环](videos/单目+IMU+开环.mp4)

### 4.2 单目 + IMU（闭环）

加入闭环检测后，可以利用识别到的历史位置对轨迹进行全局优化，减小累计漂移。

🎬 [查看运行视频：单目 + IMU + 闭环](videos/单目+IMU+闭环.mp4)

### 4.3 纯双目

仅使用双目相机进行定位，不依赖 IMU。通过双目之间的视差恢复深度信息。

🎬 [查看运行视频：双目](videos/双目.mp4)

### 4.4 双目 + GPS

使用 KITTI 数据集测试双目视觉里程计与 GPS 信息的全局融合。

🎬 [查看运行视频：双目 + GPS](videos/双目+GPS.mp4)

### 4.5 双目 + IMU（开环与闭环）

同时使用双目相机和 IMU，并观察加入闭环优化前后的轨迹表现。

🎬 [查看运行视频：双目 + IMU + 开环 + 闭环](videos/双目+IMU+开环+闭环.mp4)

## 5. 总结

这次主要是学习 VINS-Fusion 的安装和基本使用，跑通了不同传感器组合下的定位流程。

过程中花费时间最多的反而是环境配置，尤其是 OpenCV 4 与旧代码之间的兼容问题。不过通过替换旧宏，最终解决了编译错误。

通过这次实验，对视觉里程计、IMU 融合以及闭环优化有了更直观的认识。

目前主要完成了数据集回放和 RViz 可视化，后续希望进一步了解 VINS-Fusion 的算法实现，并尝试在 Gazebo / PX4 仿真环境中使用。
