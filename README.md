# LIO-SAM-DetailedNote
LIO-SAM源码详细注释，3D SLAM融合激光、IMU、GPS，因子图优化。
LIO-SAM code explained, 3D slam with lidar, imu, gps and factor graph optimization.

LIO-SAM的代码十分轻量，只有四个cpp文件，很值得读一读呢。code is very light, only have 4 cpp files, worth a read.

关于LIO-SAM的论文解读，网上已经有很多文章啦，同系列的LOAM、A-LOAM、LEGO-LOAM等，在网上都可以找到相关的解读文章。所以本文旨在对源代码进行阅读学习，积累一些工程上的经验。这里记录下来，希望可以帮到有需要的同学，如有错误的地方，请您批评指正。

:) 如果对您有帮助，帮我点个star呦~

## 目录（知乎）
- [LIO-SAM源码解析：准备篇](https://zhuanlan.zhihu.com/p/352039509)
- [LIO-SAM源码解析(一)：ImageProjection](https://zhuanlan.zhihu.com/p/352120054)
- [LIO-SAM源码解析(二)：FeatureExtraction](https://zhuanlan.zhihu.com/p/352144126)
- [LIO-SAM源码解析(三)：IMUPreintegration](https://zhuanlan.zhihu.com/p/352146800)
- [LIO-SAM源码解析(四)：MapOptimization](https://zhuanlan.zhihu.com/p/352148894)

## 整体流程

代码结构图
![Image](https://github.com/smilefacehh/LIO-SAM-DetailedNote/blob/main/system.png)

因子图
![Image](https://github.com/smilefacehh/LIO-SAM-DetailedNote/blob/main/factor.png)

1、激光运动畸变校正。利用当前帧起止时刻之间的IMU数据、IMU里程计数据计算预积分，得到每一时刻的激光点位姿，从而变换到初始时刻激光点坐标系下，实现校正。

2、提取特征。对经过运动畸变校正之后的当前帧激光点云，计算每个点的曲率，进而提取角点、平面点特征。

3、scan-to-map匹配。提取布局关键帧map的特征点，与当前帧特征点执行scan-to-map匹配，更新当前帧的位姿。

4、因子图优化。添加激光里程计因子、GPS因子、闭环因子，执行因子图优化，更新所有关键帧位姿。

5、闭环检测。在历史关键帧中找候选闭环匹配帧，执行scan-to-map匹配，得到位姿变换，构建闭环因子，加入到因子图中一并优化。

eng:

1. Lidar correction. Using imu odometer data between each time stamp to pre calculate points. Then, using lidar pointclouds and compare it with the pre calculated points from imu and do correction. 
2. Collect the corrected points and  
## 一、激光运动畸变校正（ImageProjection）

**功能简介**

1.利用当前激光帧起止时刻间的imu数据计算旋转增量，IMU里程计数据（来自ImuPreintegration）计算平移增量，进而对该帧激光每一时刻的激光点进行运动畸变校正（利用相对于激光帧起始时刻的位姿增量，变换当前激光点到起始时刻激光点的坐标系下，实现校正）；

2.同时用IMU数据的姿态角（RPY，roll、pitch、yaw）、IMU里程计数据的的位姿，对当前帧激光位姿进行粗略初始化。

**订阅**

1.订阅原始IMU数据；

2.订阅IMU里程计数据，来自ImuPreintegration，表示每一时刻对应的位姿；

3.订阅原始激光点云数据。

**发布**

1.发布当前帧激光运动畸变校正之后的有效点云，用于rviz展示；

2.发布当前帧激光运动畸变校正之后的点云信息，包括点云数据、初始位姿、姿态角、有效点云数据等，发布给FeatureExtraction进行特征提取。


## 二、点云特征提取（FeatureExtraction）

**功能简介**

对经过运动畸变校正之后的当前帧激光点云，计算每个点的曲率，进而提取角点、平面点（用曲率的大小进行判定）。

**订阅**

订阅当前激光帧运动畸变校正后的点云信息，来自ImageProjection。

**发布**

1.发布当前激光帧提取特征之后的点云信息，包括的历史数据有：运动畸变校正，点云数据，初始位姿，姿态角，有效点云数据，角点点云，平面点点云等，发布给MapOptimization；

2.发布当前激光帧提取的角点点云，用于rviz展示；

3.发布当前激光帧提取的平面点点云，用于rviz展示。


## 三、IMU预积分（ImuPreintegration）

### 1.TransformFusion类

**功能简介**

主要功能是订阅激光里程计（来自MapOptimization）和IMU里程计，根据前一时刻激光里程计，和该时刻到当前时刻的IMU里程计变换增量，计算当前时刻IMU里程计；rviz展示IMU里程计轨迹（局部）。

**订阅**

1.订阅激光里程计，来自MapOptimization；

2.订阅imu里程计，来自ImuPreintegration。

**发布**

1.发布IMU里程计，用于rviz展示；

2.发布IMU里程计轨迹，仅展示最近一帧激光里程计时刻到当前时刻之间的轨迹。


### 2. ImuPreintegration类

**功能简介**

1.用激光里程计，两帧激光里程计之间的IMU预计分量构建因子图，优化当前帧的状态（包括位姿、速度、偏置）;

2.以优化后的状态为基础，施加IMU预计分量，得到每一时刻的IMU里程计。

**订阅**

1.订阅IMU原始数据，以因子图优化后的激光里程计为基础，施加两帧之间的IMU预计分量，预测每一时刻（IMU频率）的IMU里程计；

2.订阅激光里程计（来自MapOptimization），用两帧之间的IMU预计分量构建因子图，优化当前帧位姿（这个位姿仅用于更新每时刻的IMU里程计，以及下一次因子图优化）。

**发布**

1.发布imu里程计；


## 四、因子图优化（MapOptimization）

**功能简介**

1.scan-to-map匹配：提取当前激光帧特征点（角点、平面点），局部关键帧map的特征点，执行scan-to-map迭代优化，更新当前帧位姿；

2.关键帧因子图优化：关键帧加入因子图，添加激光里程计因子、GPS因子、闭环因子，执行因子图优化，更新所有关键帧位姿；

3.闭环检测：在历史关键帧中找距离相近，时间相隔较远的帧设为匹配帧，匹配帧周围提取局部关键帧map，同样执行scan-to-map匹配，得到位姿变换，构建闭环因子数据，加入因子图优化。

**订阅**

1.订阅当前激光帧点云信息，来自FeatureExtraction；

2.订阅GPS里程计；

3.订阅来自外部闭环检测程序提供的闭环数据，本程序没有提供，这里实际没用上。

**发布**

1.发布历史关键帧里程计；

2.发布局部关键帧map的特征点云；

3.发布激光里程计，rviz中表现为坐标轴；

4.发布激光里程计；

5.发布激光里程计路径，rviz中表现为载体的运行轨迹；

6.发布地图保存服务；

7.发布闭环匹配局部关键帧map；

8.发布当前关键帧经过闭环优化后的位姿变换之后的特征点云；

9.发布闭环边，rviz中表现为闭环帧之间的连线；

10.发布局部map的降采样平面点集合；

11.发布历史帧（累加的）的角点、平面点降采样集合；

12.发布当前帧原始点云配准之后的点云；


如有错误请您批评指正，希望内容对您有帮助，更多细节可以查看代码注释~
