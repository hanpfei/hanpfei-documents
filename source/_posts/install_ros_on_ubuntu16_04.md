---
title: Ubuntu 16.04 安装 ROS
date: 2018-01-29 21:05:49
categories: ROS
tags:
- ROS
---

Robot Operating System (ROS) 是一个得到广泛应用机器人系统的软件框架，它包含了一系列的软件库和工具用于构建机器人应用。从驱动到最先进的算法，以及强大的开发者工具，ROS 包含了开发一个机器人项目所需要的所有东西。且它们都是开源的。
<!--more-->
ROS 虽然名为机器人操作系统，但它与我们一般概念中的操作系统，如 Windows，Linux，iOS 和 Android 这些不同。Windows，Linux，iOS 和 Android 这些操作系统为我们管理计算机的物理硬件资源，如 CPU、内存、磁盘、网络及外设，提供如进程、线程和文件这样的抽象，并提供如读文件、写文件、创建进程、创建线程及启动线程这样的操作。ROS 所工作的层级并没有这么低，它基于一般概念中的操作系统来运行，官方推荐基于 Ubuntu Linux 运行，并在 Ubuntu Linux 操作系统提供的抽象和操作的基础之上，提供了更高层的抽象，如节点、服务、消息、主题等，以及更高层的操作，如主题的发布、主题的订阅、服务的查询与连接等。同时 ROS 还提供开发机器人项目所需的工具和功能库。

ROS 发行版是一个版本标识的 ROS 包集合，这些与 Linux 发行版（如 Ubuntu）类似。ROS 发行版的目的是让开发者可以基于一个相对稳定的代码库来工作，直到他们可以平稳地向前演进。一旦发行版发布，官方就会限制对其的改动，而仅仅提供对于核心包的 bug fixes 和非破坏性的增强。

当前（2018-01-28） ROS 系统已经发布了多个版本。ROS 最新的一些版本如下：

ROS 系统版本 | 时间发布 | 支持时间 |
-------|----------|----------|
[ROS Lunar Loggerhead](http://wiki.ros.org/lunar/Installation) | May 23rd, 2017 | May, 2019 |
[ROS Kinetic Kame](http://wiki.ros.org/kinetic/Installation) | May 23rd, 2016 | LTS，April, 2021 (Xenial EOL) |
[ROS Jade Turtle](http://wiki.ros.org/jade/Installation) | May 23rd, 2015 | 	May, 2017 |
[ROS Indigo Igloo](http://wiki.ros.org/indigo/Installation) | July 22nd, 2014 | LTS，April, 2019(Trusty EOL) |
[ROS Hydro Medusa](http://wiki.ros.org/hydro/Installation) |September 4th, 2013 | May, 2015 |

ROS 基本上保持每年一个新版本，每两年一个长期发行版的发布节奏。关于 ROS 版本发布的更多内容，如更多的发行版的介绍，发布的计划等，可以参考 ROS 官方站点的 [Distributions](http://wiki.ros.org/Distributions) 主页。

目前官方推荐使用最近的一个长期支持版本，即 [ROS Kinetic Kame](http://wiki.ros.org/kinetic/Installation)，求新的同时兼顾稳定性无疑应该采用这一版本，如果想要尝试最新的功能特性则可以使用最新的发行版 [ROS Lunar Loggerhead](http://wiki.ros.org/lunar/Installation)。

ROS 的安装步骤如下。

## 配置 Ubuntu 仓库

配置 Ubuntu 仓库，以允许 "restricted"，"universe" 和 "multiverse"。打开 新立得包管理器，如下图：

![新立得包管理器](https://www.wolfcstech.com/images/1315506-0fb6e90c3631d23c.png)

选择 设置 -> 软件库(R)，弹出如下对话框：

![Ubuntu 仓库](https://www.wolfcstech.com/images/1315506-82755b8ea03fe580.png)

打开 "Ubuntu 软件" Tab 页，勾选 "restricted"，"universe" 和 "multiverse" 等选项，如上图所示。通常情况下，这些选项都是默认选中的，因此这一步一般不会遇到什么问题。

## 设置 sources.list

为 Ubuntu 的包管理器增加源，设置计算机接受来自于 packages.ros.org 的软件。
```
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
```

这一步会根据 Ubuntu Linux 发行版本的不同，添加不同的源。Ubuntu 的版本通过 `lsb_release -sc` 获得。

一旦添加了正确的软件库，操作系统就知道去哪里下载程序，并根据命令自动安装软件。

## 设置密钥

这一步是为了确认源代码是正确的，并且没有人在未经所有者授权的情况下，修改任何程序代码。通常情况下，当添加完软件库时，已经添加了软件库的密钥，并将其添加到操作系统的可信任列表中。

设置密钥的命令如下：
```
sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net:80 --recv-key 421C365BD9FF1F717815A3895523BAEEB01FA116
```

如果在连接密钥服务器时遇到了问题，可以尝试在上面的命令中用 `hkp://pgp.mit.edu:80` 或 `hkp://keyserver.ubuntu.com:80` 来替换。

## 安装

首先，需要确保包管理器的索引已经更新至最新：
```
sudo apt-get update
```

ROS 中有非常多不同的库和工具。官方提供了四种默认的配置来安装 ROS。也可以独立地安装 ROS 包。

**桌面完整安装（推荐采用）**：ROS，[rqt](http://wiki.ros.org/rqt), [rviz](http://wiki.ros.org/rviz)，机器人通用库，2D/3D 仿真器，导航及 2D/3D 感知

```
$ sudo apt-get install ros-kinetic-desktop-full
正在读取软件包列表... 完成
正在分析软件包的依赖关系树       
正在读取状态信息... 完成       
. . . . . .
将会同时安装下列软件：
  binfmt-support blt cpp-5 docutils-common docutils-doc fltk1.3-doc fluid fonts-lyx freeglut3 freeglut3-dev g++-5
  g++-5-multilib gazebo7 gazebo7-common gazebo7-plugin-base gcc-5 gcc-5-base gcc-5-base:i386 gcc-5-multilib
  gfortran gfortran-5 hddtemp hdf5-helpers lib32asan2 lib32atomic1 lib32cilkrts5 lib32gcc-5-dev lib32gomp1
  lib32itm1 lib32mpx0 lib32quadmath0 lib32stdc++-5-dev lib32stdc++6 lib32ubsan0 libaec-dev libaec0 libapr1-dev
  libaprutil1-dev libarmadillo6 libarpack2 libasan2 libassimp-dev libassimp3v5 libatomic1 libavcodec-dev
  libavformat-dev libavutil-dev libblas-dev libboost-all-dev libboost-atomic-dev libboost-atomic1.58-dev
  libboost-atomic1.58.0 libboost-chrono-dev libboost-chrono1.58-dev libboost-chrono1.58.0 libboost-context-dev
  libboost-context1.58-dev libboost-context1.58.0 libboost-coroutine-dev libboost-coroutine1.58-dev
  libboost-coroutine1.58.0 libboost-date-time-dev libboost-date-time1.58-dev libboost-dev libboost-exception-dev
  libboost-exception1.58-dev libboost-filesystem-dev libboost-filesystem1.58-dev libboost-graph-dev
  libboost-graph-parallel-dev libboost-graph-parallel1.58-dev libboost-graph-parallel1.58.0
  libboost-graph1.58-dev libboost-graph1.58.0 libboost-iostreams-dev libboost-iostreams1.58-dev
  libboost-locale-dev libboost-locale1.58-dev libboost-locale1.58.0 libboost-log-dev libboost-log1.58-dev
  libboost-log1.58.0 libboost-math-dev libboost-math1.58-dev libboost-math1.58.0 libboost-mpi-dev
  libboost-mpi-python-dev libboost-mpi-python1.58-dev libboost-mpi-python1.58.0 libboost-mpi1.58-dev
  libboost-mpi1.58.0 libboost-program-options-dev libboost-program-options1.58-dev libboost-python-dev
  libboost-python1.58-dev libboost-python1.58.0 libboost-random-dev libboost-random1.58-dev libboost-random1.58.0
  libboost-regex-dev libboost-regex1.58-dev libboost-regex1.58.0 libboost-serialization-dev
  libboost-serialization1.58-dev libboost-serialization1.58.0 libboost-signals-dev libboost-signals1.58-dev
  libboost-signals1.58.0 libboost-system-dev libboost-system1.58-dev libboost-test-dev libboost-test1.58-dev
  libboost-test1.58.0 libboost-thread-dev libboost-thread1.58-dev libboost-timer-dev libboost-timer1.58-dev
  libboost-timer1.58.0 libboost-tools-dev libboost-wave-dev libboost-wave1.58-dev libboost-wave1.58.0
  libboost1.58-dev libboost1.58-tools-dev libbulletcollision2.83.6 libbulletdynamics2.83.6 libcc1-0 libcilkrts5
  libcollada-dom2.4-dp-dev libcollada-dom2.4-dp0 libconsole-bridge-dev libconsole-bridge0.2v5
  libcurl4-openssl-dev libdap-dev libdap17v5 libdapclient6v5 libdapserver7v5 libeigen3-dev libepsilon1
  libflann-dev libflann1.8 libfltk-cairo1.3 libfltk-forms1.3 libfltk-gl1.3 libfltk-images1.3 libfltk1.3
  libfltk1.3-dev libfreeimage-dev libfreeimage3 libfreexl1 libgazebo7 libgazebo7-dev libgcc-5-dev libgdal-dev
  libgdal1i libgeos-3.5.0 libgeos-c1v5 libgeos-dev libgfortran-5-dev libgfortran3 libgif-dev libgl2ps-dev
  libgl2ps0 libglade2-0 libgomp1 libgtest-dev libgts-0.7-5 libgts-bin libgts-dev libhdf4-0-alt libhdf4-alt-dev
  libhdf5-10 libhdf5-cpp-11 libhdf5-dev libhdf5-mpi-dev libhdf5-openmpi-10 libhdf5-openmpi-dev libhwloc-dev
  libhwloc-plugins libhwloc5 libibverbs-dev libibverbs1 libignition-math2 libignition-math2-dev libinput-bin
  libinput-dev libinput10 libitm1 libjasper-dev libjbig-dev libjs-jquery-ui libjsoncpp-dev libjxr0 libkmlbase1
  libkmldom1 libkmlengine1 liblapack-dev libldap2-dev liblinearmath2.83.6 liblog4cxx-dev liblog4cxx10-dev
  liblog4cxx10v5 liblsan0 liblz4-dev liblzma-dev libminizip1 libmpx0 libmysqlclient-dev libmysqlclient20
  libnetcdf-c++4 libnetcdf-cxx-legacy-dev libnetcdf-dev libnetcdf11 libnuma-dev libodbc1 libogdi3.2 libogg-dev
  libogre-1.9-dev libogre-1.9.0v5 libopenjp2-7 libopenmpi-dev libopenmpi1.10 libopenni-dev
  libopenni-sensor-pointclouds0 libopenni0 libpcl-apps1.7 libpcl-common1.7 libpcl-dev libpcl-features1.7
  libpcl-filters1.7 libpcl-io1.7 libpcl-kdtree1.7 libpcl-keypoints1.7 libpcl-octree1.7 libpcl-outofcore1.7
  libpcl-people1.7 libpcl-recognition1.7 libpcl-registration1.7 libpcl-sample-consensus1.7 libpcl-search1.7
  libpcl-segmentation1.7 libpcl-surface1.7 libpcl-tracking1.7 libpcl-visualization1.7 libpcl1.7 libpoco-dev
  libpococrypto9v5 libpocodata9v5 libpocofoundation9v5 libpocomysql9v5 libpoconet9v5 libpoconetssl9v5
  libpocoodbc9v5 libpocosqlite9v5 libpocoutil9v5 libpocoxml9v5 libpocozip9v5 libpq-dev libproj-dev libproj9
  libprotoc-dev libprotoc9v5 libpyside-py3-2.0 libpyside2-dev libpyside2.0 libqhull-dev libqhull7 libqt4-dev
  libqt4-dev-bin libqt4-help libqt4-opengl-dev libqt4-scripttools libqt4-test libqt5clucene5 libqt5concurrent5
  libqt5core5a libqt5dbus5 libqt5designer5 libqt5designercomponents5 libqt5gui5 libqt5help5
  libqt5multimediaquick-p5 libqt5network5 libqt5opengl5 libqt5opengl5-dev libqt5printsupport5
  libqt5quickparticles5 libqt5scripttools5 libqt5sql5 libqt5svg5-dev libqt5test5 libqt5webkit5-dev libqt5widgets5
  libqt5x11extras5-dev libqt5xml5 libqt5xmlpatterns5 libqt5xmlpatterns5-dev libqt5xmlpatterns5-private-dev
  libqtwebkit-dev libquadmath0 libsdformat4 libsdformat4-dev libshiboken-py3-2.0 libshiboken2-dev libshiboken2.0
  libsimbody-dev libsimbody3.5v5 libspatialite-dev libspatialite7 libspnav0 libstdc++-5-dev libstdc++6
  libstdc++6:i386 libsuperlu4 libswresample-dev libswscale-dev libsz2 libtar-dev libtar0 libtbb-dev libtheora-dev
  libtiff5-dev libtiffxx5 libtinyxml-dev libtinyxml2-2v5 libtinyxml2-dev libtsan0 libubsan0 liburdfdom-dev
  liburdfdom-headers-dev liburdfdom-model-state0.4 liburdfdom-model0.4 liburdfdom-sensor0.4 liburdfdom-tools
  liburdfdom-world0.4 liburiparser1 libusb-1.0-0-dev libusb-1.0-doc libuuid1 libvtk-java libvtk5.10 libvtk6-dev
  libvtk6-java libvtk6-qt-dev libvtk6.2 libvtk6.2-qt libwacom-bin libwacom-common libwacom2 libwebp-dev
  libwebpdemux1 libx32asan2 libx32atomic1 libx32cilkrts5 libx32gcc-5-dev libx32gomp1 libx32itm1 libx32quadmath0
  libx32stdc++-5-dev libx32stdc++6 libx32ubsan0 libxerces-c-dev libxerces-c3.1 libxmu-dev libxmu-headers
  libyaml-cpp-dev libzzip-0-13 mpi-default-bin mpi-default-dev ocl-icd-libopencl1 odbcinst odbcinst1debian2
  openmpi-bin openmpi-common openni-utils proj-bin proj-data pyqt5-dev python-attr python-autobahn
  python-catkin-pkg python-catkin-pkg-modules python-chardet python-concurrent.futures python-cycler
  python-defusedxml python-docutils python-ecdsa python-empy python-glade2 python-gobject-2 python-gtk2
  python-imaging python-lz4 python-matplotlib python-matplotlib-data python-mpi4py python-msgpack
  python-netifaces python-nose python-opengl python-pam python-paramiko python-pil python-pyasn1-modules
  python-pydot python-pygments python-pyqt5 python-pyqt5.qtopengl python-pyqt5.qtsvg python-pyqt5.qtwebkit
  python-pyside2 python-pyside2.qtconcurrent python-pyside2.qtcore python-pyside2.qtgui python-pyside2.qthelp
  python-pyside2.qtnetwork python-pyside2.qtprintsupport python-pyside2.qtqml python-pyside2.qtquick
  python-pyside2.qtquickwidgets python-pyside2.qtscript python-pyside2.qtsql python-pyside2.qtsvg
  python-pyside2.qttest python-pyside2.qtuitools python-pyside2.qtwebkit python-pyside2.qtwebkitwidgets
  python-pyside2.qtwidgets python-pyside2.qtx11extras python-pyside2.qtxml python-roman python-rosdep
  python-rosdistro python-rosdistro-modules python-rospkg python-rospkg-modules python-serial
  python-service-identity python-sip python-sip-dev python-snappy python-tk python-trollius python-twisted
  python-twisted-bin python-twisted-core python-txaio python-vtk6 python-wxgtk3.0 python-wxtools python-wxversion
  python-zope.interface qt4-linguist-tools qt4-qmake qt5-qmake qtbase5-dev qtbase5-dev-tools qtbase5-private-dev
  qtdeclarative5-dev qtdeclarative5-private-dev qtmultimedia5-dev qtscript5-dev qtscript5-private-dev
  qttools5-dev qttools5-dev-tools qttools5-private-dev ros-kinetic-actionlib ros-kinetic-actionlib-msgs
  ros-kinetic-actionlib-tutorials ros-kinetic-angles ros-kinetic-bond ros-kinetic-bond-core ros-kinetic-bondcpp
  ros-kinetic-bondpy ros-kinetic-camera-calibration ros-kinetic-camera-calibration-parsers
  ros-kinetic-camera-info-manager ros-kinetic-catkin ros-kinetic-class-loader ros-kinetic-cmake-modules
  ros-kinetic-collada-parser ros-kinetic-collada-urdf ros-kinetic-common-msgs ros-kinetic-common-tutorials
  ros-kinetic-compressed-depth-image-transport ros-kinetic-compressed-image-transport ros-kinetic-control-msgs
  ros-kinetic-cpp-common ros-kinetic-cv-bridge ros-kinetic-depth-image-proc ros-kinetic-desktop
  ros-kinetic-diagnostic-aggregator ros-kinetic-diagnostic-analysis ros-kinetic-diagnostic-common-diagnostics
  ros-kinetic-diagnostic-msgs ros-kinetic-diagnostic-updater ros-kinetic-diagnostics
  ros-kinetic-dynamic-reconfigure ros-kinetic-eigen-conversions ros-kinetic-eigen-stl-containers
  ros-kinetic-executive-smach ros-kinetic-filters ros-kinetic-gazebo-dev ros-kinetic-gazebo-msgs
  ros-kinetic-gazebo-plugins ros-kinetic-gazebo-ros ros-kinetic-gazebo-ros-pkgs ros-kinetic-gencpp
  ros-kinetic-geneus ros-kinetic-genlisp ros-kinetic-genmsg ros-kinetic-gennodejs ros-kinetic-genpy
  ros-kinetic-geometric-shapes ros-kinetic-geometry ros-kinetic-geometry-msgs ros-kinetic-geometry-tutorials
  ros-kinetic-gl-dependency ros-kinetic-image-common ros-kinetic-image-geometry ros-kinetic-image-pipeline
  ros-kinetic-image-proc ros-kinetic-image-publisher ros-kinetic-image-rotate ros-kinetic-image-transport
  ros-kinetic-image-transport-plugins ros-kinetic-image-view ros-kinetic-interactive-marker-tutorials
  ros-kinetic-interactive-markers ros-kinetic-joint-state-publisher ros-kinetic-kdl-conversions
  ros-kinetic-kdl-parser ros-kinetic-laser-assembler ros-kinetic-laser-filters ros-kinetic-laser-geometry
  ros-kinetic-laser-pipeline ros-kinetic-librviz-tutorial ros-kinetic-map-msgs ros-kinetic-media-export
  ros-kinetic-message-filters ros-kinetic-message-generation ros-kinetic-message-runtime ros-kinetic-mk
  ros-kinetic-nav-msgs ros-kinetic-nodelet ros-kinetic-nodelet-core ros-kinetic-nodelet-topic-tools
  ros-kinetic-nodelet-tutorial-math ros-kinetic-octomap ros-kinetic-opencv3 ros-kinetic-orocos-kdl
  ros-kinetic-pcl-conversions ros-kinetic-pcl-msgs ros-kinetic-pcl-ros ros-kinetic-perception
  ros-kinetic-perception-pcl ros-kinetic-pluginlib ros-kinetic-pluginlib-tutorials ros-kinetic-polled-camera
  ros-kinetic-python-orocos-kdl ros-kinetic-python-qt-binding ros-kinetic-qt-dotgraph ros-kinetic-qt-gui
  ros-kinetic-qt-gui-cpp ros-kinetic-qt-gui-py-common ros-kinetic-qwt-dependency ros-kinetic-random-numbers
  ros-kinetic-resource-retriever ros-kinetic-robot ros-kinetic-robot-model ros-kinetic-robot-state-publisher
  ros-kinetic-ros ros-kinetic-ros-base ros-kinetic-ros-comm ros-kinetic-ros-core ros-kinetic-ros-tutorials
  ros-kinetic-rosbag ros-kinetic-rosbag-migration-rule ros-kinetic-rosbag-storage ros-kinetic-rosbash
  ros-kinetic-rosboost-cfg ros-kinetic-rosbuild ros-kinetic-rosclean ros-kinetic-rosconsole
  ros-kinetic-rosconsole-bridge ros-kinetic-roscpp ros-kinetic-roscpp-core ros-kinetic-roscpp-serialization
  ros-kinetic-roscpp-traits ros-kinetic-roscpp-tutorials ros-kinetic-roscreate ros-kinetic-rosgraph
  ros-kinetic-rosgraph-msgs ros-kinetic-roslang ros-kinetic-roslaunch ros-kinetic-roslib ros-kinetic-roslint
  ros-kinetic-roslisp ros-kinetic-roslz4 ros-kinetic-rosmake ros-kinetic-rosmaster ros-kinetic-rosmsg
  ros-kinetic-rosnode ros-kinetic-rosout ros-kinetic-rospack ros-kinetic-rosparam ros-kinetic-rospy
  ros-kinetic-rospy-tutorials ros-kinetic-rosservice ros-kinetic-rostest ros-kinetic-rostime ros-kinetic-rostopic
  ros-kinetic-rosunit ros-kinetic-roswtf ros-kinetic-rqt-action ros-kinetic-rqt-bag ros-kinetic-rqt-bag-plugins
  ros-kinetic-rqt-common-plugins ros-kinetic-rqt-console ros-kinetic-rqt-dep ros-kinetic-rqt-graph
  ros-kinetic-rqt-gui ros-kinetic-rqt-gui-cpp ros-kinetic-rqt-gui-py ros-kinetic-rqt-image-view
  ros-kinetic-rqt-launch ros-kinetic-rqt-logger-level ros-kinetic-rqt-moveit ros-kinetic-rqt-msg
  ros-kinetic-rqt-nav-view ros-kinetic-rqt-plot ros-kinetic-rqt-pose-view ros-kinetic-rqt-publisher
  ros-kinetic-rqt-py-common ros-kinetic-rqt-py-console ros-kinetic-rqt-reconfigure
  ros-kinetic-rqt-robot-dashboard ros-kinetic-rqt-robot-monitor ros-kinetic-rqt-robot-plugins
  ros-kinetic-rqt-robot-steering ros-kinetic-rqt-runtime-monitor ros-kinetic-rqt-rviz
  ros-kinetic-rqt-service-caller ros-kinetic-rqt-shell ros-kinetic-rqt-srv ros-kinetic-rqt-tf-tree
  ros-kinetic-rqt-top ros-kinetic-rqt-topic ros-kinetic-rqt-web ros-kinetic-rviz
  ros-kinetic-rviz-plugin-tutorials ros-kinetic-rviz-python-tutorial ros-kinetic-self-test
  ros-kinetic-sensor-msgs ros-kinetic-shape-msgs ros-kinetic-simulators ros-kinetic-smach ros-kinetic-smach-msgs
  ros-kinetic-smach-ros ros-kinetic-smclib ros-kinetic-stage ros-kinetic-stage-ros ros-kinetic-std-msgs
  ros-kinetic-std-srvs ros-kinetic-stereo-image-proc ros-kinetic-stereo-msgs ros-kinetic-tf
  ros-kinetic-tf-conversions ros-kinetic-tf2 ros-kinetic-tf2-eigen ros-kinetic-tf2-geometry-msgs
  ros-kinetic-tf2-kdl ros-kinetic-tf2-msgs ros-kinetic-tf2-py ros-kinetic-tf2-ros
  ros-kinetic-theora-image-transport ros-kinetic-topic-tools ros-kinetic-trajectory-msgs
  ros-kinetic-turtle-actionlib ros-kinetic-turtle-tf ros-kinetic-turtle-tf2 ros-kinetic-turtlesim
  ros-kinetic-urdf ros-kinetic-urdf-parser-plugin ros-kinetic-urdf-tutorial ros-kinetic-vision-opencv
  ros-kinetic-visualization-marker-tutorials ros-kinetic-visualization-msgs ros-kinetic-visualization-tutorials
  ros-kinetic-viz ros-kinetic-webkit-dependency ros-kinetic-xacro ros-kinetic-xmlrpcpp sbcl sdformat-sdf
  shiboken2 sip-dev tango-icon-theme tcl-dev tcl-vtk6 tcl8.6-dev tk-dev tk8.6-blt2.5 tk8.6-dev ttf-bitstream-vera
  ttf-liberation unixodbc unixodbc-dev uuid-dev vtk6
建议安装：
  blt-demo gcc-5-locales gcc-5-doc libstdc++6-5-dbg lib32stdc++6-5-dbg libx32stdc++6-5-dbg gazebo7-doc
  libgomp1-dbg libitm1-dbg libatomic1-dbg libasan2-dbg liblsan0-dbg libtsan0-dbg libubsan0-dbg libcilkrts5-dbg
  libmpx0-dbg libquadmath0-dbg gfortran-multilib gfortran-doc gfortran-5-multilib gfortran-5-doc libgfortran3-dbg
  ksensors liblapack-doc-man liblapack-doc libboost-doc libboost1.58-doc gccxml libmpfrc++-dev libntl-dev doxygen
  default-jdk fop libbullet2-dev libbullet2 libcurl4-doc libcurl3-dbg libidn11-dev librtmp-dev libeigen3-doc
  libmrpt-dev libgdal-doc libgts-doc libhdf4-doc hdf4-tools libnetcdf4 libhdf5-doc libhwloc-contrib-plugins
  libjs-jquery-ui-docs liblog4cxx-doc liblzma-doc netcdf-bin netcdf-doc libmyodbc odbc-postgresql tdsodbc
  unixodbc-bin ogdi-bin ogre-1.9-doc libogre-1.9.0v5-dbg opennmpi-doc openni-doc libpcl-doc libpoco-doc
  libpococrypto9v5-dbg libpocodata9v5-dbg libpocofoundation9v5-dbg libpocomysql9v5-dbg libpoconet9v5-dbg
  libpoconetssl9v5-dbg libpocoodbc9v5-dbg libpocosqlite9v5-dbg libpocoutil9v5-dbg libpocoxml9v5-dbg
  libpocozip9v5-dbg postgresql-doc-9.5 firebird-dev libsqlite0-dev qt4-dev-tools qt4-doc libqt5libqgtk2
  qt5-image-formats-plugins spacenavd libstdc++-5-doc tbb-examples libtbb-doc libtinyxml-doc java-virtual-machine
  libvtk5-dev vtk-doc vtk-examples vtk6-doc vtk6-examples libxerces-c-doc opencl-icd openmpi-checkpoint
  texlive-latex-recommended texlive-latex-base texlive-lang-french fonts-linuxlibertine | ttf-linux-libertine
  python-gtk2-doc python-gobject-2-dbg dvipng inkscape ipython python-cairocffi python-configobj
  python-excelerator python-gobject python-matplotlib-doc python-qt4 python-scipy python-tornado python-traits
  texlive-extra-utils texlive-latex-extra ttf-staypuft python-coverage python-nose-doc libgle3 python-pam-dbg
  python-pil-doc python-pil-dbg python-pyqt5-dbg python-sip-doc tix python-tk-dbg python-twisted-bin-dbg
  python-qt3 python-txaio-doc mayavi2 sbcl-doc sbcl-source slime gnome-icon-theme kdelibs-data tcl-doc tcl8.6-doc
  tk-doc tk8.6-doc
下列软件包将被【卸载】：
  libcurl4-gnutls-dev
下列【新】软件包将被安装：
  binfmt-support blt docutils-common docutils-doc fltk1.3-doc fluid fonts-lyx freeglut3 freeglut3-dev gazebo7
  gazebo7-common gazebo7-plugin-base gfortran gfortran-5 hddtemp hdf5-helpers libaec-dev libaec0 libapr1-dev
  libaprutil1-dev libarmadillo6 libarpack2 libassimp-dev libassimp3v5 libavcodec-dev libavformat-dev
  libavutil-dev libblas-dev libboost-all-dev libboost-atomic-dev libboost-atomic1.58-dev libboost-atomic1.58.0
  libboost-chrono-dev libboost-chrono1.58-dev libboost-chrono1.58.0 libboost-context-dev libboost-context1.58-dev
  libboost-context1.58.0 libboost-coroutine-dev libboost-coroutine1.58-dev libboost-coroutine1.58.0
  libboost-date-time-dev libboost-date-time1.58-dev libboost-dev libboost-exception-dev
  libboost-exception1.58-dev libboost-filesystem-dev libboost-filesystem1.58-dev libboost-graph-dev
  libboost-graph-parallel-dev libboost-graph-parallel1.58-dev libboost-graph-parallel1.58.0
  libboost-graph1.58-dev libboost-graph1.58.0 libboost-iostreams-dev libboost-iostreams1.58-dev
  libboost-locale-dev libboost-locale1.58-dev libboost-locale1.58.0 libboost-log-dev libboost-log1.58-dev
  libboost-log1.58.0 libboost-math-dev libboost-math1.58-dev libboost-math1.58.0 libboost-mpi-dev
  libboost-mpi-python-dev libboost-mpi-python1.58-dev libboost-mpi-python1.58.0 libboost-mpi1.58-dev
  libboost-mpi1.58.0 libboost-program-options-dev libboost-program-options1.58-dev libboost-python-dev
  libboost-python1.58-dev libboost-python1.58.0 libboost-random-dev libboost-random1.58-dev libboost-random1.58.0
  libboost-regex-dev libboost-regex1.58-dev libboost-regex1.58.0 libboost-serialization-dev
  libboost-serialization1.58-dev libboost-serialization1.58.0 libboost-signals-dev libboost-signals1.58-dev
  libboost-signals1.58.0 libboost-system-dev libboost-system1.58-dev libboost-test-dev libboost-test1.58-dev
  libboost-test1.58.0 libboost-thread-dev libboost-thread1.58-dev libboost-timer-dev libboost-timer1.58-dev
  libboost-timer1.58.0 libboost-tools-dev libboost-wave-dev libboost-wave1.58-dev libboost-wave1.58.0
  libboost1.58-dev libboost1.58-tools-dev libbulletcollision2.83.6 libbulletdynamics2.83.6
  libcollada-dom2.4-dp-dev libcollada-dom2.4-dp0 libconsole-bridge-dev libconsole-bridge0.2v5
  libcurl4-openssl-dev libdap-dev libdap17v5 libdapclient6v5 libdapserver7v5 libeigen3-dev libepsilon1
  libflann-dev libflann1.8 libfltk-cairo1.3 libfltk-forms1.3 libfltk-gl1.3 libfltk-images1.3 libfltk1.3
  libfltk1.3-dev libfreeimage-dev libfreeimage3 libfreexl1 libgazebo7 libgazebo7-dev libgdal-dev libgdal1i
  libgeos-3.5.0 libgeos-c1v5 libgeos-dev libgfortran-5-dev libgif-dev libgl2ps-dev libgl2ps0 libglade2-0
  libgtest-dev libgts-0.7-5 libgts-bin libgts-dev libhdf4-0-alt libhdf4-alt-dev libhdf5-10 libhdf5-cpp-11
  libhdf5-dev libhdf5-mpi-dev libhdf5-openmpi-10 libhdf5-openmpi-dev libhwloc-dev libhwloc-plugins libhwloc5
  libibverbs-dev libibverbs1 libignition-math2 libignition-math2-dev libinput-bin libinput-dev libjasper-dev
  libjbig-dev libjs-jquery-ui libjsoncpp-dev libjxr0 libkmlbase1 libkmldom1 libkmlengine1 liblapack-dev
  libldap2-dev liblinearmath2.83.6 liblog4cxx-dev liblog4cxx10-dev liblog4cxx10v5 liblz4-dev liblzma-dev
  libminizip1 libmysqlclient-dev libmysqlclient20 libnetcdf-c++4 libnetcdf-cxx-legacy-dev libnetcdf-dev
  libnetcdf11 libnuma-dev libodbc1 libogdi3.2 libogg-dev libogre-1.9-dev libogre-1.9.0v5 libopenjp2-7
  libopenmpi-dev libopenmpi1.10 libopenni-dev libopenni-sensor-pointclouds0 libopenni0 libpcl-apps1.7
  libpcl-common1.7 libpcl-dev libpcl-features1.7 libpcl-filters1.7 libpcl-io1.7 libpcl-kdtree1.7
  libpcl-keypoints1.7 libpcl-octree1.7 libpcl-outofcore1.7 libpcl-people1.7 libpcl-recognition1.7
  libpcl-registration1.7 libpcl-sample-consensus1.7 libpcl-search1.7 libpcl-segmentation1.7 libpcl-surface1.7
  libpcl-tracking1.7 libpcl-visualization1.7 libpcl1.7 libpoco-dev libpococrypto9v5 libpocodata9v5
  libpocofoundation9v5 libpocomysql9v5 libpoconet9v5 libpoconetssl9v5 libpocoodbc9v5 libpocosqlite9v5
  libpocoutil9v5 libpocoxml9v5 libpocozip9v5 libpq-dev libproj-dev libproj9 libprotoc-dev libprotoc9v5
  libpyside-py3-2.0 libpyside2-dev libpyside2.0 libqhull-dev libqhull7 libqt4-dev libqt4-dev-bin libqt4-help
  libqt4-opengl-dev libqt4-scripttools libqt4-test libqt5clucene5 libqt5concurrent5 libqt5designer5
  libqt5designercomponents5 libqt5help5 libqt5multimediaquick-p5 libqt5opengl5-dev libqt5quickparticles5
  libqt5scripttools5 libqt5svg5-dev libqt5webkit5-dev libqt5x11extras5-dev libqt5xmlpatterns5
  libqt5xmlpatterns5-dev libqt5xmlpatterns5-private-dev libqtwebkit-dev libsdformat4 libsdformat4-dev
  libshiboken-py3-2.0 libshiboken2-dev libshiboken2.0 libsimbody-dev libsimbody3.5v5 libspatialite-dev
  libspatialite7 libspnav0 libsuperlu4 libswresample-dev libswscale-dev libsz2 libtar-dev libtar0 libtbb-dev
  libtheora-dev libtiff5-dev libtiffxx5 libtinyxml-dev libtinyxml2-2v5 libtinyxml2-dev liburdfdom-dev
  liburdfdom-headers-dev liburdfdom-model-state0.4 liburdfdom-model0.4 liburdfdom-sensor0.4 liburdfdom-tools
  liburdfdom-world0.4 liburiparser1 libusb-1.0-0-dev libusb-1.0-doc libvtk-java libvtk5.10 libvtk6-dev
  libvtk6-java libvtk6-qt-dev libvtk6.2 libvtk6.2-qt libwebp-dev libwebpdemux1 libxerces-c-dev libxerces-c3.1
  libxmu-dev libxmu-headers libyaml-cpp-dev libzzip-0-13 mpi-default-bin mpi-default-dev ocl-icd-libopencl1
  odbcinst odbcinst1debian2 openmpi-bin openmpi-common openni-utils proj-bin proj-data pyqt5-dev python-attr
  python-autobahn python-catkin-pkg python-catkin-pkg-modules python-chardet python-concurrent.futures
  python-cycler python-defusedxml python-docutils python-ecdsa python-empy python-glade2 python-gobject-2
  python-gtk2 python-imaging python-lz4 python-matplotlib python-matplotlib-data python-mpi4py python-msgpack
  python-netifaces python-nose python-opengl python-pam python-paramiko python-pil python-pyasn1-modules
  python-pydot python-pygments python-pyqt5 python-pyqt5.qtopengl python-pyqt5.qtsvg python-pyqt5.qtwebkit
  python-pyside2 python-pyside2.qtconcurrent python-pyside2.qtcore python-pyside2.qtgui python-pyside2.qthelp
  python-pyside2.qtnetwork python-pyside2.qtprintsupport python-pyside2.qtqml python-pyside2.qtquick
  python-pyside2.qtquickwidgets python-pyside2.qtscript python-pyside2.qtsql python-pyside2.qtsvg
  python-pyside2.qttest python-pyside2.qtuitools python-pyside2.qtwebkit python-pyside2.qtwebkitwidgets
  python-pyside2.qtwidgets python-pyside2.qtx11extras python-pyside2.qtxml python-roman python-rosdep
  python-rosdistro python-rosdistro-modules python-rospkg python-rospkg-modules python-serial
  python-service-identity python-sip python-sip-dev python-snappy python-tk python-trollius python-twisted
  python-twisted-bin python-twisted-core python-txaio python-vtk6 python-wxgtk3.0 python-wxtools python-wxversion
  python-zope.interface qt4-linguist-tools qt4-qmake qt5-qmake qtbase5-dev qtbase5-dev-tools qtbase5-private-dev
  qtdeclarative5-dev qtdeclarative5-private-dev qtmultimedia5-dev qtscript5-dev qtscript5-private-dev
  qttools5-dev qttools5-dev-tools qttools5-private-dev ros-kinetic-actionlib ros-kinetic-actionlib-msgs
  ros-kinetic-actionlib-tutorials ros-kinetic-angles ros-kinetic-bond ros-kinetic-bond-core ros-kinetic-bondcpp
  ros-kinetic-bondpy ros-kinetic-camera-calibration ros-kinetic-camera-calibration-parsers
  ros-kinetic-camera-info-manager ros-kinetic-catkin ros-kinetic-class-loader ros-kinetic-cmake-modules
  ros-kinetic-collada-parser ros-kinetic-collada-urdf ros-kinetic-common-msgs ros-kinetic-common-tutorials
  ros-kinetic-compressed-depth-image-transport ros-kinetic-compressed-image-transport ros-kinetic-control-msgs
  ros-kinetic-cpp-common ros-kinetic-cv-bridge ros-kinetic-depth-image-proc ros-kinetic-desktop
  ros-kinetic-desktop-full ros-kinetic-diagnostic-aggregator ros-kinetic-diagnostic-analysis
  ros-kinetic-diagnostic-common-diagnostics ros-kinetic-diagnostic-msgs ros-kinetic-diagnostic-updater
  ros-kinetic-diagnostics ros-kinetic-dynamic-reconfigure ros-kinetic-eigen-conversions
  ros-kinetic-eigen-stl-containers ros-kinetic-executive-smach ros-kinetic-filters ros-kinetic-gazebo-dev
  ros-kinetic-gazebo-msgs ros-kinetic-gazebo-plugins ros-kinetic-gazebo-ros ros-kinetic-gazebo-ros-pkgs
  ros-kinetic-gencpp ros-kinetic-geneus ros-kinetic-genlisp ros-kinetic-genmsg ros-kinetic-gennodejs
  ros-kinetic-genpy ros-kinetic-geometric-shapes ros-kinetic-geometry ros-kinetic-geometry-msgs
  ros-kinetic-geometry-tutorials ros-kinetic-gl-dependency ros-kinetic-image-common ros-kinetic-image-geometry
  ros-kinetic-image-pipeline ros-kinetic-image-proc ros-kinetic-image-publisher ros-kinetic-image-rotate
  ros-kinetic-image-transport ros-kinetic-image-transport-plugins ros-kinetic-image-view
  ros-kinetic-interactive-marker-tutorials ros-kinetic-interactive-markers ros-kinetic-joint-state-publisher
  ros-kinetic-kdl-conversions ros-kinetic-kdl-parser ros-kinetic-laser-assembler ros-kinetic-laser-filters
  ros-kinetic-laser-geometry ros-kinetic-laser-pipeline ros-kinetic-librviz-tutorial ros-kinetic-map-msgs
  ros-kinetic-media-export ros-kinetic-message-filters ros-kinetic-message-generation ros-kinetic-message-runtime
  ros-kinetic-mk ros-kinetic-nav-msgs ros-kinetic-nodelet ros-kinetic-nodelet-core
  ros-kinetic-nodelet-topic-tools ros-kinetic-nodelet-tutorial-math ros-kinetic-octomap ros-kinetic-opencv3
  ros-kinetic-orocos-kdl ros-kinetic-pcl-conversions ros-kinetic-pcl-msgs ros-kinetic-pcl-ros
  ros-kinetic-perception ros-kinetic-perception-pcl ros-kinetic-pluginlib ros-kinetic-pluginlib-tutorials
  ros-kinetic-polled-camera ros-kinetic-python-orocos-kdl ros-kinetic-python-qt-binding ros-kinetic-qt-dotgraph
  ros-kinetic-qt-gui ros-kinetic-qt-gui-cpp ros-kinetic-qt-gui-py-common ros-kinetic-qwt-dependency
  ros-kinetic-random-numbers ros-kinetic-resource-retriever ros-kinetic-robot ros-kinetic-robot-model
  ros-kinetic-robot-state-publisher ros-kinetic-ros ros-kinetic-ros-base ros-kinetic-ros-comm
  ros-kinetic-ros-core ros-kinetic-ros-tutorials ros-kinetic-rosbag ros-kinetic-rosbag-migration-rule
  ros-kinetic-rosbag-storage ros-kinetic-rosbash ros-kinetic-rosboost-cfg ros-kinetic-rosbuild
  ros-kinetic-rosclean ros-kinetic-rosconsole ros-kinetic-rosconsole-bridge ros-kinetic-roscpp
  ros-kinetic-roscpp-core ros-kinetic-roscpp-serialization ros-kinetic-roscpp-traits ros-kinetic-roscpp-tutorials
  ros-kinetic-roscreate ros-kinetic-rosgraph ros-kinetic-rosgraph-msgs ros-kinetic-roslang ros-kinetic-roslaunch
  ros-kinetic-roslib ros-kinetic-roslint ros-kinetic-roslisp ros-kinetic-roslz4 ros-kinetic-rosmake
  ros-kinetic-rosmaster ros-kinetic-rosmsg ros-kinetic-rosnode ros-kinetic-rosout ros-kinetic-rospack
  ros-kinetic-rosparam ros-kinetic-rospy ros-kinetic-rospy-tutorials ros-kinetic-rosservice ros-kinetic-rostest
  ros-kinetic-rostime ros-kinetic-rostopic ros-kinetic-rosunit ros-kinetic-roswtf ros-kinetic-rqt-action
  ros-kinetic-rqt-bag ros-kinetic-rqt-bag-plugins ros-kinetic-rqt-common-plugins ros-kinetic-rqt-console
  ros-kinetic-rqt-dep ros-kinetic-rqt-graph ros-kinetic-rqt-gui ros-kinetic-rqt-gui-cpp ros-kinetic-rqt-gui-py
  ros-kinetic-rqt-image-view ros-kinetic-rqt-launch ros-kinetic-rqt-logger-level ros-kinetic-rqt-moveit
  ros-kinetic-rqt-msg ros-kinetic-rqt-nav-view ros-kinetic-rqt-plot ros-kinetic-rqt-pose-view
  ros-kinetic-rqt-publisher ros-kinetic-rqt-py-common ros-kinetic-rqt-py-console ros-kinetic-rqt-reconfigure
  ros-kinetic-rqt-robot-dashboard ros-kinetic-rqt-robot-monitor ros-kinetic-rqt-robot-plugins
  ros-kinetic-rqt-robot-steering ros-kinetic-rqt-runtime-monitor ros-kinetic-rqt-rviz
  ros-kinetic-rqt-service-caller ros-kinetic-rqt-shell ros-kinetic-rqt-srv ros-kinetic-rqt-tf-tree
  ros-kinetic-rqt-top ros-kinetic-rqt-topic ros-kinetic-rqt-web ros-kinetic-rviz
  ros-kinetic-rviz-plugin-tutorials ros-kinetic-rviz-python-tutorial ros-kinetic-self-test
  ros-kinetic-sensor-msgs ros-kinetic-shape-msgs ros-kinetic-simulators ros-kinetic-smach ros-kinetic-smach-msgs
  ros-kinetic-smach-ros ros-kinetic-smclib ros-kinetic-stage ros-kinetic-stage-ros ros-kinetic-std-msgs
  ros-kinetic-std-srvs ros-kinetic-stereo-image-proc ros-kinetic-stereo-msgs ros-kinetic-tf
  ros-kinetic-tf-conversions ros-kinetic-tf2 ros-kinetic-tf2-eigen ros-kinetic-tf2-geometry-msgs
  ros-kinetic-tf2-kdl ros-kinetic-tf2-msgs ros-kinetic-tf2-py ros-kinetic-tf2-ros
  ros-kinetic-theora-image-transport ros-kinetic-topic-tools ros-kinetic-trajectory-msgs
  ros-kinetic-turtle-actionlib ros-kinetic-turtle-tf ros-kinetic-turtle-tf2 ros-kinetic-turtlesim
  ros-kinetic-urdf ros-kinetic-urdf-parser-plugin ros-kinetic-urdf-tutorial ros-kinetic-vision-opencv
  ros-kinetic-visualization-marker-tutorials ros-kinetic-visualization-msgs ros-kinetic-visualization-tutorials
  ros-kinetic-viz ros-kinetic-webkit-dependency ros-kinetic-xacro ros-kinetic-xmlrpcpp sbcl sdformat-sdf
  shiboken2 sip-dev tango-icon-theme tcl-dev tcl-vtk6 tcl8.6-dev tk-dev tk8.6-blt2.5 tk8.6-dev ttf-bitstream-vera
  ttf-liberation unixodbc unixodbc-dev uuid-dev vtk6
下列软件包将被升级：
  cpp-5 g++-5 g++-5-multilib gcc-5 gcc-5-base gcc-5-base:i386 gcc-5-multilib lib32asan2 lib32atomic1
  lib32cilkrts5 lib32gcc-5-dev lib32gomp1 lib32itm1 lib32mpx0 lib32quadmath0 lib32stdc++-5-dev lib32stdc++6
  lib32ubsan0 libasan2 libatomic1 libcc1-0 libcilkrts5 libgcc-5-dev libgfortran3 libgomp1 libinput10 libitm1
  liblsan0 libmpx0 libqt5core5a libqt5dbus5 libqt5gui5 libqt5network5 libqt5opengl5 libqt5printsupport5
  libqt5sql5 libqt5test5 libqt5widgets5 libqt5xml5 libquadmath0 libstdc++-5-dev libstdc++6 libstdc++6:i386
  libtsan0 libubsan0 libuuid1 libwacom-bin libwacom-common libwacom2 libx32asan2 libx32atomic1 libx32cilkrts5
  libx32gcc-5-dev libx32gomp1 libx32itm1 libx32quadmath0 libx32stdc++-5-dev libx32stdc++6 libx32ubsan0
升级了 59 个软件包，新安装了 653 个软件包，要卸载 1 个软件包，有 394 个软件包未被升级。
需要下载 445 MB 的归档。
解压缩后会消耗 2,139 MB 的额外空间。
您希望继续执行吗？ [Y/n] Y
```

这种方式的安装，所需要安装的包非常多。

**桌面安装**：ROS，[rqt](http://wiki.ros.org/rqt), [rviz](http://wiki.ros.org/rviz)，机器人通用库
```
sudo apt-get install ros-kinetic-desktop
```

**ROS-Base (Bare Bones)**：ROS 包，构建和通信库。没有 GUI 工具。
```
sudo apt-get install ros-kinetic-ros-base
```

**独立的包安装**：可以安装一个特定的 ROS 包（用实际的包名来替换下面的命令中的 "PACKAGE"）。
```
sudo apt-get install ros-kinetic-PACKAGE
```

如：
```
sudo apt-get install ros-kinetic-slam-gmapping
```

要找到可用的包，可以使用：
```
apt-cache search ros-kinetic
```

通常在做开发时，采用 **桌面完整安装** 比较方便一点，可以一股脑将所有有可能用到的软件包都安装进来。对于实际的机器，则通常采用 **ROS-Base (Bare Bones)** + **独立的包** 的方式进行安装。

## 初始化 rosdep
在使用 ROS 之前，需要先初始化 `rosdep`。`rosdep` 使得你可以为你想要编译的源码，以及需要运行的 ROS 核心组件，简单地安装系统依赖。
```
sudo rosdep init
rosdep update
```

## 环境设置

如果在每次一个新的终端启动时，ROS 环境变量都能自动地添加进你的 bash 会话是非常方便，这可以通过如下命令来实现：
```
echo "source /opt/ros/kinetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

如果安装了多个 ROS 发行版，则 `~/.bashrc ` 必须只 `source` 当前正在使用的那一版的 `setup.bash`。

如果你只想要修改当前 shell 的环境，则输入如下的命令来替换上面的命令：
```
source /opt/ros/kinetic/setup.bash
```

## 构建包所需的依赖

到这一步，应该已经安装好了运行核心 ROS 包的所有东西。要创建和管理你自己的 ROS workspace，还有单独发布的许多的工具。比如，[rosinstall](http://wiki.ros.org/rosinstall) 是一个常用的命令行工具，使你可以通过一个命令为 ROS 包简单地下载许多源码树。

要安装这个工具及其它的依赖以构建 ROS 包，则运行：
```
sudo apt-get install python-rosinstall python-rosinstall-generator python-wstool build-essential
```

完成完整的 ROS 安装之后，可以对安装做一个简单的测试。可以通过 `roscore` 和 `turtlesim` 来做测试。

`roscore` 测试如下：
```
$ roscore
... logging to /home/hanpfei0306/.ros/log/3e61b674-03cf-11e8-ac54-9cb70ddc3658/roslaunch-hanpfei0306-31481.log
Checking log directory for disk usage. This may take awhile.
Press Ctrl-C to interrupt
Done checking log file disk usage. Usage is <1GB.

started roslaunch server http://hanpfei0306:44979/
ros_comm version 1.12.12


SUMMARY
========

PARAMETERS
 * /rosdistro: kinetic
 * /rosversion: 1.12.12

NODES

auto-starting new master
process[master]: started with pid [31495]
ROS_MASTER_URI=http://hanpfei0306:11311/

setting /run_id to 3e61b674-03cf-11e8-ac54-9cb70ddc3658
process[rosout-1]: started with pid [31508]
started core service [/rosout]
```

`turtlesim` 测试如下：
![Turtlesim](https://www.wolfcstech.com/images/1315506-000c62e7a4440c3f.png)

注意 `turtlesim` 的运行依赖于 `roscore` 的运行，因此在测试 `turtlesim` 需要同时运行 `roscore`。

参考文档
[ROS Distributions](http://wiki.ros.org/Distributions)
[Ubuntu install of ROS Kinetic](http://wiki.ros.org/kinetic/Installation/Ubuntu)

Done.

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done。
