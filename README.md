# Lego_LOAM_20.04

해당 깃허브에 올라온 패키지는 아래 사항이 적용된 버전임.

1. gtsam 4.1.1 설치
https://github.com/borglab/gtsam 에서 4.1.1 release 다운 받고 다운 받은 root폴더에서
```
mkdir build
cd build
cmake ..
make check (optional, runs unit tests)
sudo make install
```

2. opencv 4.4.0 설치 (생략 가능)  <br/>https://velog.io/@minukiki/Ubuntu-20.04%EC%97%90-OpenCV-4.4.0-%EC%84%A4%EC%B9%98


2. LeGO-LOAM WS에 다운 
<br/>https://github.com/RobustFieldAutonomyLab/LeGO-LOAM


3. CmakeList.txt 교체 (https://github.com/RobustFieldAutonomyLab/LeGO-LOAM/issues/247)


4. LeGO-LOAM의 include 폴더의 utility.h 파일에서 다음과 같이 변경 https://leedaehan.tistory.com/3
<br/>Before 
<br/>#include <opencv/cv.h> <br/>   
    After 
<br/>#include <opencv2/imgproc.hpp>


5. RVIZ에서 "error Failed to transform from frame [/camera] to frame [map]"라는 메세지와 함께 맵과 포인트클라우드가 출력이 안됨
    /camera to camera and /camera_init to camera_init 로 /제거 하면 됨


6. ros launch param 수정
```
<param name="pointCloudTopic_param" type="string" value="/spot1/velodyne_points"/> 
<param name="imuTopic_param" type="string" value="/spot1/vn100/imu"/>
```

7. 소스코드에서 ros param 수정
    utility.h 에서 다음과 같이 주석처리 및 const 없애기
```
// extern const string pointCloudTopic = "/velodyne_points";
extern string pointCloudTopic = "/velodyne_points";

// extern const string imuTopic = "/imu/data";
extern string imuTopic = "/imu/data";>
```
imageProjection.cpp main에 클래스 인스턴스화 전에 다음 추가

ros::param::get("pointCloudTopic_param", pointCloudTopic);

featureAssociation.cpp main에 클래스 인스턴스화 전에 다음 추가

ros::param::get("imuTopic_param", imuTopic);

mapOptmization.cpp main에 클래스 인스턴스화 전에 다음 추가

ros::param::get("imuTopic_param", imuTopic);

8. roslaunch lego_loam run.launch 실행시 [mapOptmization-7] process has died 에러가 뜨면
```
sudo apt-get install libparmetis-dev
```
# LeGO-LOAM

This repository contains code for a lightweight and ground optimized lidar odometry and mapping (LeGO-LOAM) system for ROS compatible UGVs. The system takes in point cloud  from a Velodyne VLP-16 Lidar (palced horizontally) and optional IMU data as inputs. It outputs 6D pose estimation in real-time. A demonstration of the system can be found here -> https://www.youtube.com/watch?v=O3tz_ftHV48
<!--
[![Watch the video](/LeGO-LOAM/launch/demo.gif)](https://www.youtube.com/watch?v=O3tz_ftHV48)
-->
<p align='center'>
    <img src="/LeGO-LOAM/launch/demo.gif" alt="drawing" width="800"/>
</p>

## Lidar-inertial Odometry

An updated lidar-initial odometry package, [LIO-SAM](https://github.com/TixiaoShan/LIO-SAM), has been open-sourced and available for testing.

## Dependency

- [ROS](http://wiki.ros.org/ROS/Installation) (tested with indigo, kinetic, and melodic)
- [gtsam](https://github.com/borglab/gtsam/releases) (Georgia Tech Smoothing and Mapping library, 4.0.0-alpha2)
  ```
  wget -O ~/Downloads/gtsam.zip https://github.com/borglab/gtsam/archive/4.0.0-alpha2.zip
  cd ~/Downloads/ && unzip gtsam.zip -d ~/Downloads/
  cd ~/Downloads/gtsam-4.0.0-alpha2/
  mkdir build && cd build
  cmake ..
  sudo make install
  ```

## Compile

You can use the following commands to download and compile the package.

```
cd ~/catkin_ws/src
git clone https://github.com/RobustFieldAutonomyLab/LeGO-LOAM.git
cd ..
catkin_make -j1
```
When you compile the code for the first time, you need to add "-j1" behind "catkin_make" for generating some message types. "-j1" is not needed for future compiling.

## The system

LeGO-LOAM is speficifally optimized for a horizontally placed VLP-16 on a ground vehicle. It assumes there is always a ground plane in the scan. The UGV we are using is Clearpath Jackal. It has a built-in IMU. 

<p align='center'>
    <img src="/LeGO-LOAM/launch/jackal-label.jpg" alt="drawing" width="400"/>
</p>

The package performs segmentation before feature extraction.

<p align='center'>
    <img src="/LeGO-LOAM/launch/seg-total.jpg" alt="drawing" width="400"/>
</p>

Lidar odometry performs two-step Levenberg Marquardt optimization to get 6D transformation.

<p align='center'>
    <img src="/LeGO-LOAM/launch/odometry.jpg" alt="drawing" width="400"/>
</p>

## New Lidar

The key thing to adapt the code to a new sensor is making sure the point cloud can be properly projected to an range image and ground can be correctly detected. For example, VLP-16 has a angular resolution of 0.2&deg; and 2&deg; along two directions. It has 16 beams. The angle of the bottom beam is -15&deg;. Thus, the parameters in "utility.h" are listed as below. When you implement new sensor, make sure that the ground_cloud has enough points for matching. Before you post any issues, please read this.

```
extern const int N_SCAN = 16;
extern const int Horizon_SCAN = 1800;
extern const float ang_res_x = 0.2;
extern const float ang_res_y = 2.0;
extern const float ang_bottom = 15.0;
extern const int groundScanInd = 7;
```

Another example for Velodyne HDL-32e range image projection:

```
extern const int N_SCAN = 32;
extern const int Horizon_SCAN = 1800;
extern const float ang_res_x = 360.0/Horizon_SCAN;
extern const float ang_res_y = 41.333/float(N_Scan-1);
extern const float ang_bottom = 30.666666;
extern const int groundScanInd = 20;
```

**New**: a new **useCloudRing** flag has been added to help with point cloud projection (i.e., VLP-32C, VLS-128). Velodyne point cloud has "ring" channel that directly gives the point row id in a range image. Other lidars may have a same type of channel, i.e., "r" in Ouster. If you are using a non-Velodyne lidar but it has a similar "ring" channel, you can change the PointXYZIR definition in utility.h and the corresponding code in imageProjection.cpp.

For **KITTI** users, if you want to use our algorithm with  **HDL-64e**, you need to write your own implementation for such projection. If the point cloud is not projected properly, you will lose many points and performance.

If you are using your lidar with an IMU, make sure your IMU is aligned properly with the lidar. The algorithm uses IMU data to correct the point cloud distortion that is cause by sensor motion. If the IMU is not aligned properly, the usage of IMU data will deteriorate the result. Ouster lidar IMU is not supported in the package as LeGO-LOAM needs a 9-DOF IMU.

## Run the package

1. Run the launch file:
```
roslaunch lego_loam run.launch
```
Notes: The parameter "/use_sim_time" is set to "true" for simulation, "false" to real robot usage.

2. Play existing bag files:
```
rosbag play *.bag --clock --topic /velodyne_points /imu/data
```
Notes: Though /imu/data is optinal, it can improve estimation accuracy greatly if provided. Some sample bags can be downloaded from [here](https://github.com/RobustFieldAutonomyLab/jackal_dataset_20170608). 

## New data-set

This dataset, [Stevens data-set](https://github.com/TixiaoShan/Stevens-VLP16-Dataset), is captured using a Velodyne VLP-16, which is mounted on an UGV - Clearpath Jackal, on Stevens Institute of Technology campus. The VLP-16 rotation rate is set to 10Hz. This data-set features over 20K scans and many loop-closures. 

<p align='center'>
    <img src="/LeGO-LOAM/launch/dataset-demo.gif" alt="drawing" width="600"/>
</p>
<p align='center'>
    <img src="/LeGO-LOAM/launch/google-earth.png" alt="drawing" width="600"/>  
</p>

## Cite *LeGO-LOAM*

Thank you for citing [our *LeGO-LOAM* paper](./Shan_Englot_IROS_2018_Preprint.pdf) if you use any of this code: 
```
@inproceedings{legoloam2018,
  title={LeGO-LOAM: Lightweight and Ground-Optimized Lidar Odometry and Mapping on Variable Terrain},
  author={Shan, Tixiao and Englot, Brendan},
  booktitle={IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS)},
  pages={4758-4765},
  year={2018},
  organization={IEEE}
}
```

## Loop Closure

The loop-closure method implemented in this package is a naive ICP-based method. It often fails when the odometry drift is too large. For more advanced loop-closure methods, there is a package called [SC-LeGO-LOAM](https://github.com/irapkaist/SC-LeGO-LOAM), which features utilizing point cloud descriptor.

## Speed Optimization

An optimized version of LeGO-LOAM can be found [here](https://github.com/facontidavide/LeGO-LOAM/tree/speed_optimization). All credits go to @facontidavide. Improvements in this directory include but not limited to:

    + To improve the quality of the code, making it more readable, consistent and easier to understand and modify.
    + To remove hard-coded values and use proper configuration files to describe the hardware.
    + To improve performance, in terms of amount of CPU used to calculate the same result.
    + To convert a multi-process application into a single-process / multi-threading one; this makes the algorithm more deterministic and slightly faster.
    + To make it easier and faster to work with rosbags: processing a rosbag should be done at maximum speed allowed by the CPU and in a deterministic way.
    + As a consequence of the previous point, creating unit and regression tests will be easier.




