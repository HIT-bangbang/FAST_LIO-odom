# 使用MID-360基于Fast-lio进行定位和输出里程计

# 一、编译FAST-LIO

按照Fastlio官方的方法编译`FAST-LIO2`即可，使用最新版本的faslio

编译Fastlio之前中，需要首先编译 livox 的驱动和ros驱动包，FAST-LIO2目前还是使用的`livox_ros_driver`，但是现在Livox建议使用最新版的驱动`livox_ros_driver2`

所以如果安装的是 `livox_ros_driver2` 就需要稍微修改一下`FAST-LIO2`的源码。具体的修改如下：

* 源码中，所有的`livox_ros_driver` 全部改为 `livox_ros_driver2`
* `CMakeLists.txt`中所有的`livox_ros_driver`改为`livox_ros_driver2`
* `package.xml` 中的`livox_ros_driver`改为`livox_ros_driver2`

**总之就一句话，把功能包里面所有的`livox_ros_driver` 全部改为 `livox_ros_driver2`**


# 二、 重定位并输出里程计

重定位使用的包是 https://github.com/davidakhihiero/FAST_LIO_LOCALIZATION-ROS-NOETIC

这个仓库是基于 `FAST-LIO` 做的重定位。其实它和 `FAST-LIO` 都没啥紧密的关系，也没有修改`FAST-LIO`的源码。它里面真正有用的其实就那三个python脚本，做了重定位，融合里程计和发布`/map`到`/camera_init`的变换关系。而且这个仓库中的`FAST-LIO`版本很老了，也没有mid360的支持，建议依照本文重新建立。

重定位并输出里程计的原理和流程是这样的：

* 首先用Fastlio建好地图，保存为PCD。
* 然后，再重新把Fastlio开起来，这时候Fastlio会输出点云`/cloud_registered`、里程计话题`/Odometry` 以及 `/camera_init`到`/body`的坐标变换，但是只不过是不知道`/camera_init`和地图原点的关系。
* 现在，把FAST_LIO_LOCALIZATION 节点（那里面的三个python脚本）launch起来，这个节点就会加载PCD文件，然后订阅`FAST-LIO`发布的`/cloud_registered` 去做ICP重定位，同时会订阅`/Odometry` 与重定位的结果融合，发布`/map`到`/camera_init`的TF。

所以，其实我们只需要这个仓库里面的那三个pyhon文件就可以了。

### 步骤

1. 构建本仓库中的`re_localization`这个功能包，其实就是新建一个功能包，然后只把`https://github.com/davidakhihiero/FAST_LIO_LOCALIZATION-ROS-NOETIC` 中的`scripts`文件夹中的脚本拷来，其他的都不要。

2. 在 Fastlio 功能包的launch文件中增加重定位使用的launch文件，建本工程的`localization_mid360.launch`。我在这里也复制了一份参数文件并做了些修改`FAST_LIO/config/mid360_localization.yaml`（比如保存PCD，还有些可视化啥的就不需要了）

使用时：

3. 先用启动`FAST_LIO/launch/mapping_mid360.launch`建好地图，保存为PCD。

效果如下：

![图片alt](/img/PCD点云.png)

1. 然后启动`localization_mid360.launch`，以及mid360的驱动包`roslaunch livox_ros_driver2 msg_MID360.launch`。
rviz显示如下：这时事实点云和地图还没有对齐，因为现在还没进行重定位。

![图片alt](/img/重定位之前.png)

2. 此时会要求输入初始位姿，可以直接在RVIZ中点绿色箭头发布初始位姿，也可以用该功能包里提供的脚本：

    rosrun fast_lio_localization publish_initial_pose.py 14.5 -7.5 0 -0.25 0 0

3. 如果初始位姿没为题，重定位一般不会出问题。效果如下：

![图片alt](/img/重定位之后.png)


**到这一步就已经完成定位了，对于3D导航的情况已经直接可以用了**

# 三、 生成并加载 2D 地图，并且从 3D 激光生成 2D scan

**为了适应2D情况下的导航以及 movebase，需要生成并加载 2D 地图，并且从3D激光生成2D scan**

使用离线的方式生成 2D 地图，这里使用的包是 `https://github.com/Hinson-A/pcd2pgm_package`

从 3D 激光生成 2D scan，这里使用的包是 `pointcloud_to_laserscan` `https://github.com/ros-perception/pointcloud_to_laserscan` 这个包可以直接二进制安装

注意：由于我们的激光雷达高出地面，所以如果使用 `/map` 系作为2d地图的坐标系的话，会出现2d地图飘在半空的情况，看上去不太好看。
所以可以创建一个新的/2D_map坐标系在/map系的正下方地面上，作为2d地图话题的坐标系。在这个仓库代码里面没做这个事情

启动流程见 `FAST_LIO/launch/localization_map2d_mid360.launch`

效果如下：

![图片alt](/img/带有2d地图的定位.png)

# 四、一些可能出现的问题

1、激光雷达不是水平放置的情况，生成2D地图可能会出现问题
2、使用 octomap 在线生成二维地图
3、movebase速度滤波器

以上问题可以参考：https://github.com/66Lau/NEXTE_Sentry_Nav/tree/main


# TODO

`FAST_LIO_LOCALIZATION-ROS-NOETIC` 这里面用python写的重定位，运行很吃CPU资源，在香橙派上每一次运行重定位都会把功耗拉高很多
后续使用C++重写ICP重定位，效率会高很多很多，或者直接在FASTLIO上面改。

可以参考这个仓库，这位同志写的非常详细，本项目也是跟着他的思路来做的  https://github.com/66Lau/NEXTE_Sentry_Nav/tree/main

