# szu-xxlab-lidar_cam_calib_pipeline
This project is a sensor calibration framework designed specifically for Livox MID360 LiDAR, FLIR cameras, and Hikvision industrial cameras. The framework integrates multiple advanced calibration methods and provides a complete multi-sensor system calibration solution.

# Calibration Process Documentation

## Reference Projects

This project references the following projects:

- Camera ROS Driver  
  https://github.com/ShuyangUni/hdr_bracketing_cam_ctrl

- Kalibr Calibration Toolbox  
  https://github.com/ethz-asl/kalibr?tab=readme-ov-file

- Livox ROS Driver 2  
  https://github.com/Livox-SDK/livox_ros_driver2

- Livox SDK2  
  https://github.com/Livox-SDK/Livox-SDK2

- Visual-LiDAR Calibration Tool1  
  https://github.com/hku-mars/livox_camera_calib

- Visual-LiDAR Calibration Tool2 (koide3)  
  https://github.com/koide3/direct_visual_lidar_calibration

## Software Requirements

- Ubuntu 20.04
- [ROS Noetic Ninjemys](https://wiki.ros.org/noetic/Installation/Ubuntu)
- OpenCV (installed with ROS)
- Eigen (installed with ROS)
- [Spinnaker 2.7.0.128](https://www.flir.com/products/spinnaker-sdk/)
- [YAML-CPP](https://github.com/jbeder/yaml-cpp)
- Livox SDK2 https://github.com/Livox-SDK/Livox-SDK2
- **Ceres Solver**  
  Follow [Ceres Installation Guide](http://ceres-solver.org/installation.html).
- **PCL (Point Cloud Library)**  
  Follow [PCL Installation Guide](http://www.pointclouds.org/downloads/linux.html). (Our code is tested with PCL 1.7)

## Calibration Steps

### 1. Create Your Own Workspace and Get Source Code

Since this repository contains only documentation and does not include the source code of third-party projects, you need to create your own workspace and download all required packages:

```bash
mkdir -p calib_ws/src
cd calib_ws/src

# Clone all required repositories
git clone https://github.com/ShuyangUni/hdr_bracketing_cam_ctrl
git clone https://github.com/ethz-asl/kalibr
git clone https://github.com/Livox-SDK/livox_ros_driver2
git clone https://github.com/Livox-SDK/Livox-SDK2
git clone https://github.com/hku-mars/livox_camera_calib
git clone https://github.com/koide3/direct_visual_lidar_calibration

# Return to workspace root
cd ..
```

Before compiling, locate the `hdr_bracketing_cam_ctrl/src/camera/camera_auto.cc` file and comment out lines 49 (EnableAutoExposure...) and 51 (EnableAutoGain...) to disable auto exposure and auto gain.

```bash
catkin_make
```

### 2. Node Startup Instructions

Refer to the FLIR camera ROS driver link to properly configure and start the FLIR camera.

#### Start FLIR Camera Node

```bash
roslaunch hdr_attr_ctrl test_camera_auto.launch
```

#### Start MID360 Node

```bash
roslaunch livox_ros_driver2 rviz_MID360.launch
```

After starting these nodes, you can use rosbag to record the corresponding topic information as needed:

- Image topic: `/bfly/cam0/image_raw`
- IMU topic: `/livox/imu`
- LiDAR topic: `/livox/lidar`

```bash
rosbag record /bfly/cam0/image_raw /livox/imu /livox/lidar
```

### 3. Using Kalibr (Camera Intrinsics and Camera-IMU Extrinsics Calibration)

Build the Kalibr Docker environment:

```bash
cd calib_ws/src/kalibr
docker build -t kalibr -f Dockerfile_ros1_20_04 . # It is recommended to run Kalibr using Docker. Remember to select the appropriate Dockerfile according to your Ubuntu version
```

Mount the data folder to the container's /data path:

```bash
FOLDER=/path/to/your/data/on/host
xhost +local:root
docker run -it -e "DISPLAY" -e "QT_X11_NO_MITSHM=1" \
    -v "/tmp/.X11-unix:/tmp/.X11-unix:rw" \
    -v "$FOLDER:/data" kalibr
```

Load the ROS environment variables, place your data files in the /data directory, and run Kalibr to perform camera intrinsic calibration:

```bash
source devel/setup.bash
rosrun kalibr kalibr_calibrate_cameras \
    --bag /data/cam_april.bag --target /data/april_6x6.yaml \
    --models pinhole-radtan pinhole-radtan \
    --topics /cam0/image_raw /cam1/image_raw
```

Select the appropriate camera projection model and provide corresponding files according to the script:

For camera intrinsic calibration, you need to provide:
- bag: Bag file containing recorded camera image topics using rosbag
- target: YAML file describing the calibration board

For IMU-camera extrinsic calibration, you need to provide:
- bag: Bag file containing recorded camera image topics (note: multiple topics should be recorded simultaneously)
- target: YAML file describing the calibration board and another YAML file describing camera intrinsics

### 4. IMU Error Bias Calibration

Refer to allan_variance_ros for error bias calibration. Due to real-time interference from the LiDAR, we recommend skipping this calibration step and directly using the following parameters to construct an imu.yaml file for IMU-camera extrinsic calibration.

```yaml
# Accelerometer parameters
accelerometer_noise_density: 1.0e-01
accelerometer_random_walk: 1.0e-02

# Gyroscope parameters
gyroscope_noise_density: 1.0e-02 
gyroscope_random_walk: 1.0e-03

rostopic: '/livox/imu' 
update_rate: 200 
```

### 5. LiDAR-Camera Extrinsic Calibration

This project uses two methods for LiDAR-camera extrinsic calibration:

#### Using livox_camera_calib Method

You need to place the LiDAR stationary in a structured scene, then use rqt_image_view (ensure roscore has been started first) to capture an image. After the LiDAR starts for 10-15 seconds, use rosbag to record 40-50 seconds of LiDAR data.

Rename the recorded bag file to lidar_data.bag and place it in the path specified by the bag_to_pcd.launch parameters, or you can modify the path parameters.

Convert the bag file to pcd files:

```bash
roslaunch livox_camera_calib bag_to_pcd.launch
```

Place the pcd files and image files in the /livox_camera_calib/calib_dataset/single_scene_calibration folder using the correct naming convention (e.g., 0.pcd, 0.png). livox_camera_calib also provides a multi-scene fusion calibration method. For detailed steps, please refer to: https://github.com/hku-mars/livox_camera_calib

Then run directly:

```bash
roslaunch livox_camera_calib calib.launch
```

#### Using camera_lidar_calib-koide3 Method

We also used camera_lidar_calib-koide3 for LiDAR-camera calibration. You can refer to the project link for more comprehensive tutorials: https://github.com/koide3/direct_visual_lidar_calibration

##### Pull Docker Image

```bash
docker pull koide3/direct_visual_lidar_calibration:noetic  # Note to match the ROS version
```

##### Run Preprocessing

```bash
export bag_path=/home/zhh/cam_lidar
export preprocessed_path=/home/zhh/cam_lidar

docker run   -it   --rm   --net host   -e DISPLAY=$DISPLAY   -v $HOME/.Xauthority:/root/.Xauthority   -v $bag_path:/tmp/input_bags   -v $preprocessed_path:/tmp/preprocessed   koide3/direct_visual_lidar_calibration:noetic rosrun direct_visual_lidar_calibration preprocess -av   --camera_model plumb_bob   --camera_intrinsic 598.5157734878896,598.9676211435436,503.59316072891943,382.99488802097426   --camera_distortion_coeffs -0.0948369813355068,0.08122161857152066,-0.0002740847842610672,0.0002300092734366941,0.00000000   /tmp/input_bags /tmp/preprocessed
data_path: /tmp/input_bags
dst_path : /tmp/preprocessed
```

##### Manual Initial Guess

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

##### Perform Calibration

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

##### View Calibration Results

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

- [Camera ROS Driver](https://github.com/ShuyangUni/hdr_bracketing_cam_ctrl)
- [Kalibr Calibration Toolbox](https://github.com/ethz-asl/kalibr?tab=readme-ov-file)
- [Livox ROS Driver 2](https://github.com/Livox-SDK/livox_ros_driver2)
- [Livox SDK2](https://github.com/Livox-SDK/Livox-SDK2)
- [Livox Camera Calibration Tool](https://github.com/hku-mars/livox_camera_calib)
- [Direct Visual-LiDAR Calibration Tool (koide3)](https://github.com/koide3/direct_visual_lidar_calibration)

We express our sincere gratitude to the developers of these projects!
