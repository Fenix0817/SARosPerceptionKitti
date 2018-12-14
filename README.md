SARosPerceptionKitti
=================

ROS package for the Perception (Sensor Processing, Detection, Tracking and Evaluation) of the KITTI Vision Benchmark 

<p align="center">
  <img src="./videos/semantic.gif">
</p>

<p align="center">
  <img src="./videos/rviz.gif">
</p>

### Setup

1) [Install ROS Kinetic on Ubuntu 16.04](http://wiki.ros.org/kinetic/Installation/Ubuntu)
2) [Setup ROS Workspace](http://wiki.ros.org/catkin/Tutorials/create_a_workspace):  
```
mkdir -p ~/catkin_ws/src  
cd ~/catkin_ws/src  
git pull https://github.com/appinho/SARosPerceptionKitti.git  
cd ..  
catkin_make  
source devel/setup.bash  
```

3) [Install Kitti2Bag](https://github.com/tomas789/kitti2bag)

```
pip install kitti2bag
```

### Dataset aquisition

1) Download the `synced+rectified data` and its `calibration` files from a single scenario (e.g. `0060` from the demo above) of the [KITTI Raw Dataset](http://www.cvlibs.net/datasets/kitti/raw_data.php):  

```
wget http://kitti.is.tue.mpg.de/kitti/raw_data/2011_09_26_drive_0002/2011_09_26_drive_0060_sync.zip
wget http://kitti.is.tue.mpg.de/kitti/raw_data/2011_09_26_calib.zip

```

2) Unzip the files and merge them into one ROSbag file:  

```
unzip 2011_09_26_drive_0060_sync.zip
unzip 2011_09_26_calib.zip
kitti2bag -t 2011_09_26 -r 0060 raw_synced . 
```

3) Synchronize the ROSbag file's sensor data:  

* The script matches the timestamps of the Velodyne point cloud data with the Camara data to perform Sensor Fusion in a synchronized way within the ROS framework 

```
cd ~/catkin_ws/src/SARosPerceptionKitti/pre_processing/
python sync_rosbag.py kitti_2011_09_26_drive_0060_synced.bag
```

4) Generate preprocessed semantic segmentated images:  

* The Camera data is preprocessed within a Deep Neural Network to create semantic segmentated images. With this step a "real-time" performance on any device (CPU usage) can be guaranteed

```
mkdir ~/kitti_data/0060/segmented_semantic_images/
cd ~/kitti_data/0060/segmented_semantic_images/
```

* For scenario `0060` you can [download my results](https://drive.google.com/file/d/1ihGnk5x9OlzF4X-YJXFsKB8rYSLyo0YF/view?usp=sharing) and store them within the directory

* For any other scenario follow this steps: Well pre-trained network with an IOU of 73% can be found here: [Finetuned Google's DeepLab on KITTI Dataset](https://github.com/hiwad-aziz/kitti_deeplab)

4) Final folder structure  

```
    ~                                        # Home directory
    ├── catkin_ws                            # Catkin workspace
    │   ├── src                              # Clone repo in here
    │       └── SARosPerceptionKitti         # Repo
    ├── kitti_data                           # Data folder
    │   ├── 0001                             # Scenario 0001
    │   ├── ...                              # Any other scenario
    │   ├── 0060                             # Demo scenario 0060
    │   │   ├── segmented_semantic_images    # Folder for semantic images (Copy download in here)
    │   │   │   ├── 0000000000.png           # Semantic image from first time frame
    │   │   │   ├── 0000000001.png           # Semantic image from second time frame
    │   │   │   └── ...  
    │   │   └── synchronized_data.bag        # Synchronized ROSbag file
    │   ├── ...
```

* IMPORTANT: Go into `sensor_processing/src/sensor_processing_lib/sensor_fusion.cpp` under method `processImage()` to hardcode your home directory!

```
    // HARDCODE HOME DIRECTORY HERE
    path_name << "/YOUR_HOME_DIRECTORY/kitti_data/"
```
5) Run a ROS Package:  

* Launch one of the following ROS nodes together with the scenario identifier and wait until RViz is fully loaded:  

```
roslaunch sensor_processing sensor.launch scenario:=0060
roslaunch detection detection.launch scenario:=0060
roslaunch tracking tracking.launch scenario:=0060
roslaunch evaluation evaluation.launch scenario:=0060
```

* Play back the synchronized ROSbag file (here at 25% speed):  

```
cd ~/kitti_data/0060/
rosbag play -r 0.25 synchronized_data.bag
```

### Troubleshooting

* SEMANTIC IMAGES WARNING: Go to sensor.cpp line 543 in sensor_processing_lib and hardcode your personal home directory! ([see full discussion here](https://github.com/appinho/SARosPerceptionKitti/issues/10))
* Make sure to close RVIz and restart the ROS launch command if you want to execute the scenario again. Otherwise it seems like the data isn't moving anymore ([see here](https://github.com/appinho/SARosPerceptionKitti/issues/7))
* Make sure the scenario is encoded as 4 digit number, like above `0060`
* Make sure the images are encoded as 10 digit numbers starting from `0000000000.png`
* Make sure the resulting semantic segmentated images have the color encoding of the [Cityscape Dataset](https://www.cityscapes-dataset.com/examples/)

### Discussion

Evaluation results for 7 Scenarios `0011,0013,0014,0018,0056,0059,0060`

| Class        |  MOTP   |  MODP   |
| ------------ |:-------:|:-------:|
| Car          | 0.715273| 0.785403|
| Pedestrian   | 0.581809| 0.988038|

### Areas for Improvements

* Improving the Object Detection so that the object's shape, especially for cars, is incorporated and that false classification within the semantic segmentation can be tolerated
* Applying the VoxelNet

### Contact

Send me an email simonappel62@gmail.com if you have any questions, wishes or ideas to find an even better solution! Happy to collaborate :)

<!--
## Evaluation for 7 Scenarios 0011,0013,0014,0018,0056,0059,0060

| Class        | MOTA    | MOTP    |  MOTAL  |    MODA |    MODP |
| ------------ |:-------:|:-------:|:-------:|:-------:|:-------:|
| CAR          | 0.250970| 0.715273| 0.274552| 0.274903| 0.785403|
| PEDESTRIAN   |-0.015038| 0.581809|-0.015038|-0.015038| 0.988038|


[157, 154, 280, 306, 378, 1283, 17]
[64, 10, 10, 72, 11, 196, 0]
[39, 75, 120, 39, 33, 569, 0]

[8, 0, 1, 0, 4, 18, 18]
[3, 0, 2, 0, 0, 52, 0]
[172, 0, 63, 0, 25, 177, 46]

## Pipeline

### 1a) Sensor Fusion: Velodyne Point Cloud Processing

* [Ground extraction & Free space estimation](http://wiki.ros.org/but_velodyne_proc)

### 1b) Sensor Fusion: Raw Image Processing

* [Semantic segmentation](https://github.com/martinkersner/train-DeepLab)

### 1c) Sensor Fusion: Mapping Point Cloud and Image

### 2 Detection: DBSCAN Clustering

### 3 Tracking: UKF Tracker

-->
