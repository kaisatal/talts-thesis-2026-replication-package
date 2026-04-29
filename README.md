# talts-thesis-2026-replication-package

## Overview

This repository contains all the practical work within the Bachelor's thesis "Self-driving test platform with a LiDAR-based navigation system" (2026).

The specific ROS2 package developed in this work can be found at https://github.com/kaisatal/jeep_driver.
The /software folder contains the whole ROS2 workspace with all the external packages that were used.

## Running the software

### Required hardware:
- The modified Jeep Raptor electric car
- Jetson Nano
- Velodyne VLP-16

### Step 1: Acquiring the map

The position of the car on the ground should be the same for all 3 steps.

Creating a ros2 bag (on Jetson Nano):
```bash
# Terminal 1
docker build -t jeep_ros2 . # Needed once to create jeep_container

sudo docker run -it --rm \
  --name jeep_container \
  --network host \
  --ipc host \
  --gpus all \
  --runtime nvidia \
  --privileged \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e ROS_DOMAIN_ID=0 \
  -e ROS_LOCALHOST_ONLY=0 \
  -e CYCLONEDDS_URI=file:///cyclonedds.xml \
  -v /home/nano/ros_ws/cyclonedds.xml:/cyclonedds.xml \
  -v /home/nano/ros_ws:/ros_ws \
  -v /home/nano/bags:/bags \
  -v /sys:/sys \
  jeep_ros2

ros2 run velodyne_driver velodyne_driver_node \
--ros-args \
-p device_ip:=192.168.2.12 \
-p port:=2369 \
-p model:=VLP16 \
-p interface:=eth1

# Terminal 2
docker exec -it jeep_container bash

ros2 run velodyne_pointcloud velodyne_transform_node \
--ros-args \
-p model:=VLP16 \
-p calibration:=/opt/ros/humble/share/velodyne_pointcloud/params/VLP16db.yaml

# Terminal 3:
docker exec -it jeep_container bash

ros2 bag record -o /bags/bag_for_map_creation /velodyne_packets /velodyne_points
# Save the bag by closing this process (Ctrl+C)
```

Creating the map.pcd from the ros2 bag (on external laptop or Jetson Nano):
```bash
ros2 launch lidarslam lidarslam.launch.py
```

### Step 2: Drawing out the desired path

Reminder: the position of the car on the ground should be the same for all 3 steps.

On Jetson Nano:
```bash
# Terminal 1 (lidar)
sudo docker run -it --rm \
  --name jeep_container \
  --network host \
  --ipc host \
  --gpus all \
  --runtime nvidia \
  --privileged \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e ROS_DOMAIN_ID=0 \
  -e ROS_LOCALHOST_ONLY=0 \
  -e CYCLONEDDS_URI=file:///cyclonedds.xml \
  -v /home/nano/ros_ws/cyclonedds.xml:/cyclonedds.xml \
  -v /home/nano/ros_ws:/ros_ws \
  -v /home/nano/bags:/bags \
  -v /sys:/sys \
  jeep_ros2

ros2 run velodyne_driver velodyne_driver_node \
--ros-args \
-p device_ip:=192.168.2.12 \
-p port:=2369 \
-p model:=VLP16 \
-p interface:=eth1

# Terminal 2 (lidar)
docker exec -it jeep_container bash

ros2 run velodyne_pointcloud velodyne_transform_node \
--ros-args \
-p model:=VLP16 \
-p calibration:=/opt/ros/humble/share/velodyne_pointcloud/params/VLP16db.yaml

# Terminal 3 (keyboard)
docker exec -it jeep_container bash

ros2 run jeep_driver keyboard_node # Manual driving

# Terminal 4 (motors)
docker exec -it jeep_container bash

ros2 launch jeep_driver jeep_driver.launch.py # Motor driver
```

On the external laptop:
```bash
# Terminal 1
sudo docker build -t localization_ros2 . # Needed once to create localization_container

sudo docker run -it --rm \
  --name localization_container \
  --network host \
  --ipc=host \
  --pid=host \
  --privileged \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e ROS_DOMAIN_ID=0 \
  -e ROS_LOCALHOST_ONLY=0 \
  -e CYCLONEDDS_URI=file:///cyclonedds.xml \
  -v /home/ubuntu/ros2_ws/cyclonedds.xml:/cyclonedds.xml \
  -v /home/ubuntu/map:/map \
  -v /home/ubuntu/last_path_bag:/last_path_bag \
  localization_ros2

# Running the following process is the start of the desired path
ros2 launch lidar_localization_ros2 lidar_localization.launch.py

# Terminal 2
sudo docker exec -it localization_container bash

ros2 run jeep_driver last_path_recorder_node
# Closing this process is the end of the desired path
```

### Step 3: Following the desired path

Reminder: the position of the car on the ground should be the same for all 3 steps.

On Jetson Nano:
```bash
# Terminal 1 (lidar)
sudo docker run -it --rm \
  --name jeep_container \
  --network host \
  --ipc host \
  --gpus all \
  --runtime nvidia \
  --privileged \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e ROS_DOMAIN_ID=0 \
  -e ROS_LOCALHOST_ONLY=0 \
  -e CYCLONEDDS_URI=file:///cyclonedds.xml \
  -v /home/nano/ros_ws/cyclonedds.xml:/cyclonedds.xml \
  -v /home/nano/ros_ws:/ros_ws \
  -v /home/nano/bags:/bags \
  -v /sys:/sys \
  jeep_ros2

ros2 run velodyne_driver velodyne_driver_node \
--ros-args \
-p device_ip:=192.168.2.12 \
-p port:=2369 \
-p model:=VLP16 \
-p interface:=eth1

# Terminal 2 (lidar)
docker exec -it jeep_container bash

ros2 run velodyne_pointcloud velodyne_transform_node \
--ros-args \
-p model:=VLP16 \
-p calibration:=/opt/ros/humble/share/velodyne_pointcloud/params/VLP16db.yaml

# Terminal 3 (keyboard)
docker exec -it jeep_container bash

ros2 run jeep_driver keyboard_node # Manual driving

# Terminal 4 (motors)
docker exec -it jeep_container bash

ros2 launch jeep_driver jeep_driver.launch.py # Motor driver
```

On the external laptop:
```bash
# Terminal 1
sudo docker build -t localization_ros2 . # Needed once to create localization_container

sudo docker run -it --rm \
  --name localization_container \
  --network host \
  --ipc=host \
  --pid=host \
  --privileged \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e ROS_DOMAIN_ID=0 \
  -e ROS_LOCALHOST_ONLY=0 \
  -e CYCLONEDDS_URI=file:///cyclonedds.xml \
  -v /home/ubuntu/ros2_ws/cyclonedds.xml:/cyclonedds.xml \
  -v /home/ubuntu/map:/map \
  -v /home/ubuntu/last_path_bag:/last_path_bag \
  localization_ros2

# Running the following process is the start of the desired path
ros2 launch lidar_localization_ros2 lidar_localization.launch.py

# Terminal 2
sudo docker exec -it localization_container bash

ros2 run jeep_driver path_follower_node # Autonomous driving
# To now use the autonomous driving commands, go to the Jetson Nano terminal 3 (keyboard) and press SPACEBAR
```