# 标定流程文档

## 参考项目

本项目参考了以下几个项目：

- FLIR相机的ROS驱动  
  https://github.com/ShuyangUni/hdr_bracketing_cam_ctrl

- Kalibr标定工具  
  https://github.com/ethz-asl/kalibr?tab=readme-ov-file

- Livox ROS驱动2  
  https://github.com/Livox-SDK/livox_ros_driver2

- Livox SDK2  
  https://github.com/Livox-SDK/Livox-SDK2

- 相机激光雷达外参标定工具1  
  https://github.com/hku-mars/livox_camera_calib

- 相机激光雷达外参标定工具2（koide3）  
  https://github.com/koide3/direct_visual_lidar_calibration

## 软件环境要求

- Ubuntu 20.04
- [ROS Noetic Ninjemys](https://wiki.ros.org/noetic/Installation/Ubuntu)
- OpenCV（随ROS安装）
- Eigen（随ROS安装）
- [Spinnaker 2.7.0.128](https://www.flir.com/products/spinnaker-sdk/)
- [YAML-CPP](https://github.com/jbeder/yaml-cpp)
- Livox SDK2 https://github.com/Livox-SDK/Livox-SDK2
- **Ceres优化库**  
  参照[Ceres安装指南](http://ceres-solver.org/installation.html)进行安装。
- **PCL点云库**  
  参照[PCL安装指南](http://www.pointclouds.org/downloads/linux.html)进行安装。（代码在PCL1.7上测试通过）

## 标定步骤

### 1. 创建自己的工作空间并获取源码

由于本仓库仅包含文档，不包含第三方项目的源码，您需要创建自己的工作空间并下载所有必需的包：

```bash
mkdir -p calib_ws/src
cd calib_ws/src

# 克隆所有必需的仓库
git clone https://github.com/ShuyangUni/hdr_bracketing_cam_ctrl
git clone https://github.com/ethz-asl/kalibr
git clone https://github.com/Livox-SDK/livox_ros_driver2
git clone https://github.com/Livox-SDK/Livox-SDK2
git clone https://github.com/hku-mars/livox_camera_calib
git clone https://github.com/koide3/direct_visual_lidar_calibration

# 返回工作空间根目录
cd ..
```

如果你的相机是Flir系列的，在编译前，需要找到`hdr_bracketing_cam_ctrl/src/camera/camera_auto.cc`文件，将第49行（EnableAutoExposure相关）和第51行（EnableAutoGain相关）注释掉，以关闭自动曝光与自动增益功能。

```bash
catkin_make
```

### 2. 节点启动说明

参考FLIR相机的ROS驱动链接，正确配置和启动FLIR相机。

#### 启动FLIR相机节点

```bash
roslaunch hdr_attr_ctrl test_camera_auto.launch
```

#### 启动MID360节点

```bash
roslaunch livox_ros_driver2 rviz_MID360.launch
```

启动这些节点后，可以根据需求使用rosbag录制相应的话题信息：

- 图像话题：`/bfly/cam0/image_raw`
- IMU话题：`/livox/imu`
- 激光雷达话题：`/livox/lidar`

```bash
rosbag record /bfly/cam0/image_raw /livox/imu /livox/lidar
```

### 3. 使用Kalibr进行标定（相机内参标定与相机-IMU外参标定）

编译Kalibr的Docker环境：

```bash
cd calib_ws/src/kalibr
docker build -t kalibr -f Dockerfile_ros1_20_04 . # 推荐使用Docker运行Kalibr标定工具，请根据Ubuntu版本选择合适的Dockerfile
```

将数据文件夹挂载到容器的/data路径中：

```bash
FOLDER=/path/to/your/data/on/host
xhost +local:root
docker run -it -e "DISPLAY" -e "QT_X11_NO_MITSHM=1" \
    -v "/tmp/.X11-unix:/tmp/.X11-unix:rw" \
    -v "$FOLDER:/data" kalibr
```

加载ROS环境变量，然后将数据文件放入/data目录中，运行Kalibr进行相机内参标定：

```bash
source devel/setup.bash
rosrun kalibr kalibr_calibrate_cameras \
    --bag /data/cam_april.bag --target /data/april_6x6.yaml \
    --models pinhole-radtan pinhole-radtan \
    --topics /cam0/image_raw /cam1/image_raw
```

选择正确的相机成像模型，根据脚本提供对应的文件：

在进行相机内参标定时，需要提供以下文件：
- bag：使用rosbag录制的相机图像话题的bag文件
- target：标定板的yaml文件

在进行IMU和相机外参标定时，需要提供以下文件：
- bag：使用rosbag录制的相机图像话题的bag文件（注意：需同时录制多个话题）
- target：标定板的yaml文件以及相机内参描述的yaml文件

### 4. IMU误差偏移标定

参考allan_variance_ros进行误差偏移标定。由于激光雷达存在实时干扰，我们建议您跳过此标定步骤，直接使用以下参数构建imu.yaml文件来进行IMU与相机的外参标定。

```yaml
# 加速度计参数
accelerometer_noise_density: 1.0e-01
accelerometer_random_walk: 1.0e-02

# 陀螺仪参数
gyroscope_noise_density: 1.0e-02 
gyroscope_random_walk: 1.0e-03

rostopic: '/livox/imu' 
update_rate: 200 
```

### 5. 激光雷达-相机外参标定

本项目使用两种方法进行激光雷达-相机外参标定：

#### 使用livox_camera_calib方法

您需要将激光雷达静止放置在一个具有丰富结构特征的场景中，然后使用rqt_image_view（请确保已先启动roscore）命令获取一张图像。在雷达启动10-15秒后，使用rosbag录制40-50秒的激光雷达数据。

将录制的数据包重命名为lidar_data.bag，并存放在bag_to_pcd.launch文件中对应路径参数指定的路径下，您也可以修改该路径参数。

将bag文件转换为pcd文件：

```bash
roslaunch livox_camera_calib bag_to_pcd.launch
```

将pcd文件和图像文件按照正确的命名方式（如0.pcd，0.png）放到/livox_camera_calib/calib_dataset/single_scene_calibration文件夹中。livox_camera_calib还提供了多场景融合的标定方法，详细步骤请参考：https://github.com/hku-mars/livox_camera_calib

然后直接运行：

```bash
roslaunch livox_camera_calib calib.launch
```

#### 使用camera_lidar_calib-koide3方法

我们还使用camera_lidar_calib-koide3进行了激光雷达-相机标定，您可以参考该项目链接获取更详细的教程：https://github.com/koide3/direct_visual_lidar_calibration

##### 拉取Docker镜像

```bash
docker pull koide3/direct_visual_lidar_calibration:noetic  # 注意匹配ROS版本
```

##### 运行预处理

```bash
export bag_path=/home/zhh/cam_lidar
export preprocessed_path=/home/zhh/cam_lidar

docker run   -it   --rm   --net host   -e DISPLAY=$DISPLAY   -v $HOME/.Xauthority:/root/.Xauthority   -v $bag_path:/tmp/input_bags   -v $preprocessed_path:/tmp/preprocessed   koide3/direct_visual_lidar_calibration:noetic rosrun direct_visual_lidar_calibration preprocess -av   --camera_model plumb_bob   --camera_intrinsic 598.5157734878896,598.9676211435436,503.59316072891943,382.99488802097426   --camera_distortion_coeffs -0.0948369813355068,0.08122161857152066,-0.0002740847842610672,0.0002300092734366941,0.00000000   /tmp/input_bags /tmp/preprocessed
data_path: /tmp/input_bags
dst_path : /tmp/preprocessed
```

##### 手动初始化猜测值

```bash
docker run \
  --rm \
  --net host \
  --gpus all \
  -e DISPLAY=$DISPLAY \
  -v $HOME/.Xauthority:/root/.Xauthority \
  -v $preprocessed_path:/tmp/preprocessed \
  koide3/direct_visual_lidar_calibration:noetic \
  rosrun direct_visual_lidar_calibration initial_guess_manual /tmp/preprocessed
```

##### 执行标定

```bash
docker run \
  --rm \
  --net host \
  --gpus all \
  -e DISPLAY=$DISPLAY \
  -v $HOME/.Xauthority:/root/.Xauthority \
  -v $preprocessed_path:/tmp/preprocessed \
  koide3/direct_visual_lidar_calibration:noetic \
  rosrun direct_visual_lidar_calibration calibrate /tmp/preprocessed
```

##### 查看标定结果

```bash
docker run \
  --rm \
  --net host \
  --gpus all \
  -e DISPLAY=$DISPLAY \
  -v $HOME/.Xauthority:/root/.Xauthority \
  -v $preprocessed_path:/tmp/preprocessed \
  koide3/direct_visual_lidar_calibration:noetic \
  rosrun direct_visual_lidar_calibration viewer /tmp/preprocessed
```


本项目参考并整合了以下开源项目：

- [FLIR相机的ROS驱动](https://github.com/ShuyangUni/hdr_bracketing_cam_ctrl)
- [Kalibr标定工具](https://github.com/ethz-asl/kalibr?tab=readme-ov-file)
- [Livox ROS驱动2](https://github.com/Livox-SDK/livox_ros_driver2)
- [Livox SDK2](https://github.com/Livox-SDK/Livox-SDK2)
- [Livox相机标定工具](https://github.com/hku-mars/livox_camera_calib)
- [相机激光雷达直接标定工具（koide3）](https://github.com/koide3/direct_visual_lidar_calibration)

我们对这些项目的开发者表示诚挚的感谢！
