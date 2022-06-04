---
title: Ubuntu 16.04 安装 Gazebo
date: 2021-05-29 21:05:49
categories: ROS
tags:
- ROS
---

Gazebo 是一款 3D 动态模拟器，能够准确有效地模拟复杂室内和室外环境中的机器人群体。虽然类似于游戏引擎，Gazebo 提供了更高保真度的物理模拟、一套传感器以及用户和程序接口。
<!--more-->
Gazebo 的典型用途包括：
 * 测试机器人算法，
 * 设计机器人，
 * 使用真实场景执行回归测试

Gazebo 的一些主要功能包括：
 * 多个物理引擎，
 * 丰富的机器人模型和环境库，
 * 各种各样的传感器，
 * 方便的编程和图形界面

Gazebo 的安装方法如下。

在 Gazebo 的 [下载页面](http://gazebosim.org/download) 下载 Gazebo 的 `.deb` 包。

![Gazebo Download](../images/1315506-427e5a7473ff38b1.png)

实际下载回来的是一个脚本文件。为该文件添加可执行权限：
```
$ chmod a+x gazebo8_install.sh
```

然后执行该脚本文件：
```
$ ./gazebo8_install.sh 
[sudo] hanpfei0306 的密码： 
Downloading keys
OK
Retrieving packages
*** Error in `appstreamcli': double free or corruption (fasttop): 0x0000000001d8fbd0 ***
======= Backtrace: =========
/lib/x86_64-linux-gnu/libc.so.6(+0x777e5)[0x7f671d60f7e5]
/lib/x86_64-linux-gnu/libc.so.6(+0x8037a)[0x7f671d61837a]
/lib/x86_64-linux-gnu/libc.so.6(cfree+0x4c)[0x7f671d61c53c]
/usr/lib/x86_64-linux-gnu/libappstream.so.3(as_component_complete+0x439)[0x7f671d994d19]
/usr/lib/x86_64-linux-gnu/libappstream.so.3(as_data_pool_update+0x44a)[0x7f671d995f0a]
/usr/lib/x86_64-linux-gnu/libappstream.so.3(as_cache_builder_refresh+0x1c2)[0x7f671d98b272]
appstreamcli(ascli_refresh_cache+0x12e)[0x4049de]
appstreamcli(as_client_run+0x6fb)[0x403ceb]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf0)[0x7f671d5b8830]
appstreamcli(_start+0x29)[0x403519]
======= Memory map: ========
00400000-00408000 r-xp 00000000 08:04 2490418                            /usr/bin/appstreamcli
00607000-00608000 r--p 00007000 08:04 2490418                            /usr/bin/appstreamcli
00608000-00609000 rw-p 00008000 08:04 2490418                            /usr/bin/appstreamcli
0107d000-02bd7000 rw-p 00000000 00:00 0                                  [heap]
7f6714000000-7f6714021000 rw-p 00000000 00:00 0 
7f6714021000-7f6718000000 ---p 00000000 00:00 0 
7f6718909000-7f6718d94000 r--p 00000000 08:04 2491866                    /usr/lib/locale/locale-archive
7f6718d94000-7f671a64a000 r-xp 00000000 08:04 2490706                    /usr/lib/x86_64-linux-gnu/libicudata.so.55.1
7f671a64a000-7f671a849000 ---p 018b6000 08:04 2490706                    /usr/lib/x86_64-linux-gnu/libicudata.so.55.1
7f671a849000-7f671a84a000 r--p 018b5000 08:04 2490706                    /usr/lib/x86_64-linux-gnu/libicudata.so.55.1
7f671a84a000-7f671a84b000 rw-p 018b6000 08:04 2490706                    /usr/lib/x86_64-linux-gnu/libicudata.so.55.1
7f671a84b000-7f671a84f000 r-xp 00000000 08:04 2369831                    /lib/x86_64-linux-gnu/libuuid.so.1.3.0
7f671a84f000-7f671aa4e000 ---p 00004000 08:04 2369831                    /lib/x86_64-linux-gnu/libuuid.so.1.3.0
7f671aa4e000-7f671aa4f000 r--p 00003000 08:04 2369831                    /lib/x86_64-linux-gnu/libuuid.so.1.3.0
. . . . . .
7f671e6c0000-7f671e6c1000 r--p 00025000 08:04 2369408                    /lib/x86_64-linux-gnu/ld-2.23.so
7f671e6c1000-7f671e6c2000 rw-p 00026000 08:04 2369408                    /lib/x86_64-linux-gnu/ld-2.23.so
7f671e6c2000-7f671e6c3000 rw-p 00000000 00:00 0 
7ffeaf4a6000-7ffeaf4c7000 rw-p 00000000 00:00 0                          [stack]
7ffeaf4d2000-7ffeaf4d4000 r--p 00000000 00:00 0                          [vvar]
7ffeaf4d4000-7ffeaf4d6000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
Aborted (core dumped)
E: Problem executing scripts APT::Update::Post-Invoke-Success 'if /usr/bin/test -w /var/cache/app-info -a -e /usr/bin/appstreamcli; then appstreamcli refresh > /dev/null; fi'
E: Sub-process returned an error code
OK
Installing Gazebo
正在读取软件包列表... 完成
正在分析软件包的依赖关系树       
正在读取状态信息... 完成       
有一些软件包无法被安装。如果您用的是 unstable 发行版，这也许是
因为系统无法达到您要求的状态造成的。该版本中可能会有一些您需要的软件
包尚未被创建或是它们已被从新到(Incoming)目录移出。
下列信息可能会对解决问题有所帮助：

下列软件包有未满足的依赖关系：
 gazebo8 : 依赖: libgazebo8 (= 8.2.0-1~xenial) 但是它将不会被安装
           依赖: libsdformat5 但是它将不会被安装
           推荐: gazebo8-plugin-base
 libgazebo8-dev : 依赖: libsdformat5-dev 但是它将不会被安装
                  依赖: libgazebo8 (= 8.2.0-1~xenial) 但是它将不会被安装
                  依赖: gazebo8-plugin-base (= 8.2.0-1~xenial)
E: 无法修正错误，因为您要求某些软件包保持现状，就是它们破坏了软件包间的依赖关系。
Complete.
Type gazebo to start the
```

按照提示输入密码。脚本会自动执行 `apt-get` 安装所有的依赖。但在通过 `apt-get` 安装依赖时，报出了如上面所示的错误，比较关键的是如下这些错误信息：
```
*** Error in `appstreamcli': double free or corruption (fasttop): 0x0000000001d8fbd0 ***
```

即检测到二进制可执行文件 `appstreamcli` 在执行时发生了 double free 问题。这个问题在 [Ask Ubuntu](https://askubuntu.com/questions/943463/library-corruption-error-during-apt-get-update) 中已经有人遇到过，并已经有了解法了：
```
sudo apt-get purge libappstream3
```

修复上面 `apt-get` 的问题之后，再次执行 Gazebo 的安装脚本 `gazebo8_install.sh`：
```
$ ./gazebo8_install.sh 
Downloading keys
OK
Retrieving packages
OK
Installing Gazebo
正在读取软件包列表... 完成
正在分析软件包的依赖关系树       
正在读取状态信息... 完成       
下列软件包是自动安装的并且现在不需要了：
  fltk1.3-doc fluid fonts-lato gazebo7-common gir1.2-gtk-2.0 gnome-software-common libapr1-dev libaprutil1-dev libassimp-dev libassimp3v5 libatk1.0-dev
  libbz2-dev libcairo-script-interpreter2 libcairo2-dev libcollada-dom2.4-dp-dev libcollada-dom2.4-dp0 libconsole-bridge-dev libconsole-bridge0.2v5
  libeigen3-dev libflann-dev libflann1.8 libfltk-cairo1.3 libfltk-forms1.3 libfltk-gl1.3 libfltk-images1.3 libfltk1.3 libfltk1.3-dev libfontconfig1-dev
  libfreetype6-dev libgdk-pixbuf2.0-dev libgl2ps-dev libgl2ps0 libgtk2.0-dev libgtkspell3-3-0 libharfbuzz-dev libharfbuzz-gobject0 libhdf5-mpi-dev
  libhdf5-openmpi-10 libhdf5-openmpi-dev libinput-dev libjbig-dev libjs-jquery-ui libjs-sphinxdoc libjs-underscore libjsoncpp-dev liblog4cxx-dev
  liblog4cxx10-dev liblog4cxx10v5 liblz4-dev liblzma-dev libnetcdf-c++4 libnetcdf-cxx-legacy-dev libogg-dev libopenni-dev libopenni-sensor-pointclouds0
  libopenni0 libpango1.0-dev libpcl-apps1.7 libpcl-common1.7 libpcl-dev libpcl-features1.7 libpcl-filters1.7 libpcl-io1.7 libpcl-kdtree1.7 libpcl-keypoints1.7
  libpcl-octree1.7 libpcl-outofcore1.7 libpcl-people1.7 libpcl-recognition1.7 libpcl-registration1.7 libpcl-sample-consensus1.7 libpcl-search1.7
  libpcl-segmentation1.7 libpcl-surface1.7 libpcl-tracking1.7 libpcl-visualization1.7 libpcl1.7 libpoco-dev libpococrypto9v5 libpocodata9v5 libpocofoundation9v5
  libpocomysql9v5 libpoconet9v5 libpoconetssl9v5 libpocoodbc9v5 libpocosqlite9v5 libpocoutil9v5 libpocoxml9v5 libpocozip9v5 libproj-dev libpyside-py3-2.0
  libpyside2-dev libpyside2.0 libqgsttools-p1 libqhull-dev libqhull7 libqt5clucene5 libqt5designercomponents5 libqt5help5 libqt5multimediaquick-p5
  libqt5multimediawidgets5 libqt5quickparticles5 libqt5scripttools5 libqt5svg5-dev libqt5webkit5-dev libqt5x11extras5-dev libqt5xmlpatterns5
  libqt5xmlpatterns5-dev libqt5xmlpatterns5-private-dev libqtwebkit-dev libruby2.3 libsctp-dev libsctp1 libshiboken-py3-2.0 libshiboken2-dev libshiboken2.0
  libtheora-dev libtiff5-dev libtiffxx5 libtinyxml2-dev liburdfdom-dev liburdfdom-headers-dev liburdfdom-model-state0.4 liburdfdom-model0.4 liburdfdom-sensor0.4
  liburdfdom-tools liburdfdom-world0.4 libvtk-java libvtk5.10 libvtk6-dev libvtk6-java libvtk6-qt-dev libvtk6.2 libvtk6.2-qt libwxbase3.0-0v5 libwxgtk3.0-0v5
  libxcb-shm0-dev libxcomposite-dev libxft-dev libyaml-cpp-dev libyaml-cpp0.5v5 linux-headers-4.4.0-21 linux-headers-4.4.0-21-generic
  linux-image-4.4.0-21-generic linux-image-extra-4.4.0-21-generic openni-utils pyqt5-dev python-attr python-autobahn python-concurrent.futures python-cycler
  python-empy python-glade2 python-gobject-2 python-gtk2 python-imaging python-lz4 python-matplotlib python-matplotlib-data python-mpi4py python-msgpack
  python-netifaces python-nose python-numpy python-opengl python-pam python-pyasn1-modules python-pydot python-pyparsing python-pyqt5 python-pyqt5.qtopengl
  python-pyqt5.qtsvg python-pyqt5.qtwebkit python-pyside2 python-pyside2.qtconcurrent python-pyside2.qtcore python-pyside2.qtgui python-pyside2.qthelp
  python-pyside2.qtnetwork python-pyside2.qtprintsupport python-pyside2.qtqml python-pyside2.qtquick python-pyside2.qtquickwidgets python-pyside2.qtscript
  python-pyside2.qtsql python-pyside2.qtsvg python-pyside2.qttest python-pyside2.qtuitools python-pyside2.qtwebkit python-pyside2.qtwebkitwidgets
  python-pyside2.qtwidgets python-pyside2.qtx11extras python-pyside2.qtxml python-rosdep python-serial python-service-identity python-sip python-sip-dev
  python-snappy python-trollius python-twisted python-twisted-bin python-twisted-core python-txaio python-tz python-vtk6 python-wxgtk3.0 python-wxtools
  python-wxversion qtbase5-private-dev qtdeclarative5-dev qtdeclarative5-private-dev qtmultimedia5-dev qtscript5-dev qtscript5-private-dev qttools5-dev
  qttools5-dev-tools qttools5-private-dev rake ros-kinetic-actionlib ros-kinetic-actionlib-msgs ros-kinetic-actionlib-tutorials ros-kinetic-angles
  ros-kinetic-bond ros-kinetic-bond-core ros-kinetic-bondcpp ros-kinetic-bondpy ros-kinetic-camera-calibration ros-kinetic-camera-calibration-parsers
  ros-kinetic-camera-info-manager ros-kinetic-catkin ros-kinetic-class-loader ros-kinetic-cmake-modules ros-kinetic-collada-parser ros-kinetic-collada-urdf
  ros-kinetic-common-msgs ros-kinetic-common-tutorials ros-kinetic-compressed-depth-image-transport ros-kinetic-compressed-image-transport
  ros-kinetic-control-msgs ros-kinetic-cpp-common ros-kinetic-cv-bridge ros-kinetic-depth-image-proc ros-kinetic-desktop ros-kinetic-diagnostic-aggregator
  ros-kinetic-diagnostic-analysis ros-kinetic-diagnostic-common-diagnostics ros-kinetic-diagnostic-msgs ros-kinetic-diagnostic-updater ros-kinetic-diagnostics
  ros-kinetic-dynamic-reconfigure ros-kinetic-eigen-conversions ros-kinetic-eigen-stl-containers ros-kinetic-executive-smach ros-kinetic-filters
  ros-kinetic-gazebo-msgs ros-kinetic-gencpp ros-kinetic-geneus ros-kinetic-genlisp ros-kinetic-genmsg ros-kinetic-gennodejs ros-kinetic-genpy
  ros-kinetic-geometric-shapes ros-kinetic-geometry ros-kinetic-geometry-msgs ros-kinetic-geometry-tutorials ros-kinetic-gl-dependency ros-kinetic-image-common
  ros-kinetic-image-geometry ros-kinetic-image-pipeline ros-kinetic-image-proc ros-kinetic-image-publisher ros-kinetic-image-rotate ros-kinetic-image-transport
  ros-kinetic-image-transport-plugins ros-kinetic-image-view ros-kinetic-interactive-marker-tutorials ros-kinetic-interactive-markers
  ros-kinetic-joint-state-publisher ros-kinetic-kdl-conversions ros-kinetic-kdl-parser ros-kinetic-laser-assembler ros-kinetic-laser-filters
  ros-kinetic-laser-geometry ros-kinetic-laser-pipeline ros-kinetic-librviz-tutorial ros-kinetic-map-msgs ros-kinetic-media-export ros-kinetic-message-filters
  ros-kinetic-message-generation ros-kinetic-message-runtime ros-kinetic-mk ros-kinetic-nav-msgs ros-kinetic-nodelet ros-kinetic-nodelet-core
  ros-kinetic-nodelet-topic-tools ros-kinetic-nodelet-tutorial-math ros-kinetic-octomap ros-kinetic-opencv3 ros-kinetic-orocos-kdl ros-kinetic-pcl-conversions
  ros-kinetic-pcl-msgs ros-kinetic-pcl-ros ros-kinetic-perception ros-kinetic-perception-pcl ros-kinetic-pluginlib ros-kinetic-pluginlib-tutorials
  ros-kinetic-polled-camera ros-kinetic-python-orocos-kdl ros-kinetic-python-qt-binding ros-kinetic-qt-dotgraph ros-kinetic-qt-gui ros-kinetic-qt-gui-cpp
  ros-kinetic-qt-gui-py-common ros-kinetic-qwt-dependency ros-kinetic-random-numbers ros-kinetic-resource-retriever ros-kinetic-robot ros-kinetic-robot-model
  ros-kinetic-robot-state-publisher ros-kinetic-ros ros-kinetic-ros-base ros-kinetic-ros-comm ros-kinetic-ros-core ros-kinetic-ros-tutorials ros-kinetic-rosbag
  ros-kinetic-rosbag-migration-rule ros-kinetic-rosbag-storage ros-kinetic-rosbash ros-kinetic-rosboost-cfg ros-kinetic-rosbuild ros-kinetic-rosclean
  ros-kinetic-rosconsole ros-kinetic-rosconsole-bridge ros-kinetic-roscpp ros-kinetic-roscpp-core ros-kinetic-roscpp-serialization ros-kinetic-roscpp-traits
  ros-kinetic-roscpp-tutorials ros-kinetic-roscreate ros-kinetic-rosgraph ros-kinetic-rosgraph-msgs ros-kinetic-roslang ros-kinetic-roslaunch ros-kinetic-roslib
  ros-kinetic-roslint ros-kinetic-roslisp ros-kinetic-roslz4 ros-kinetic-rosmake ros-kinetic-rosmaster ros-kinetic-rosmsg ros-kinetic-rosnode ros-kinetic-rosout
  ros-kinetic-rospack ros-kinetic-rosparam ros-kinetic-rospy ros-kinetic-rospy-tutorials ros-kinetic-rosservice ros-kinetic-rostest ros-kinetic-rostime
  ros-kinetic-rostopic ros-kinetic-rosunit ros-kinetic-roswtf ros-kinetic-rqt-action ros-kinetic-rqt-bag ros-kinetic-rqt-bag-plugins
  ros-kinetic-rqt-common-plugins ros-kinetic-rqt-console ros-kinetic-rqt-dep ros-kinetic-rqt-graph ros-kinetic-rqt-gui ros-kinetic-rqt-gui-cpp
  ros-kinetic-rqt-gui-py ros-kinetic-rqt-image-view ros-kinetic-rqt-launch ros-kinetic-rqt-logger-level ros-kinetic-rqt-moveit ros-kinetic-rqt-msg
  ros-kinetic-rqt-nav-view ros-kinetic-rqt-plot ros-kinetic-rqt-pose-view ros-kinetic-rqt-publisher ros-kinetic-rqt-py-common ros-kinetic-rqt-py-console
  ros-kinetic-rqt-reconfigure ros-kinetic-rqt-robot-dashboard ros-kinetic-rqt-robot-monitor ros-kinetic-rqt-robot-plugins ros-kinetic-rqt-robot-steering
  ros-kinetic-rqt-runtime-monitor ros-kinetic-rqt-rviz ros-kinetic-rqt-service-caller ros-kinetic-rqt-shell ros-kinetic-rqt-srv ros-kinetic-rqt-tf-tree
  ros-kinetic-rqt-top ros-kinetic-rqt-topic ros-kinetic-rqt-web ros-kinetic-rviz ros-kinetic-rviz-plugin-tutorials ros-kinetic-rviz-python-tutorial
  ros-kinetic-self-test ros-kinetic-sensor-msgs ros-kinetic-shape-msgs ros-kinetic-smach ros-kinetic-smach-msgs ros-kinetic-smach-ros ros-kinetic-smclib
  ros-kinetic-stage ros-kinetic-stage-ros ros-kinetic-std-msgs ros-kinetic-std-srvs ros-kinetic-stereo-image-proc ros-kinetic-stereo-msgs ros-kinetic-tf
  ros-kinetic-tf-conversions ros-kinetic-tf2 ros-kinetic-tf2-eigen ros-kinetic-tf2-geometry-msgs ros-kinetic-tf2-kdl ros-kinetic-tf2-msgs ros-kinetic-tf2-py
  ros-kinetic-tf2-ros ros-kinetic-theora-image-transport ros-kinetic-topic-tools ros-kinetic-trajectory-msgs ros-kinetic-turtle-actionlib ros-kinetic-turtle-tf
  ros-kinetic-turtle-tf2 ros-kinetic-turtlesim ros-kinetic-urdf ros-kinetic-urdf-parser-plugin ros-kinetic-urdf-tutorial ros-kinetic-vision-opencv
  ros-kinetic-visualization-marker-tutorials ros-kinetic-visualization-msgs ros-kinetic-visualization-tutorials ros-kinetic-viz ros-kinetic-webkit-dependency
  ros-kinetic-xacro ros-kinetic-xmlrpcpp ruby ruby-did-you-mean ruby-minitest ruby-net-telnet ruby-power-assert ruby-test-unit ruby2.3 rubygems-integration sbcl
  shiboken2 sip-dev tcl-dev tcl-vtk6 tcl8.6-dev tk-dev tk8.6-dev ttf-liberation vtk6 x11proto-composite-dev
使用'sudo apt autoremove'来卸载它(它们)。
将会同时安装下列软件：
  gazebo7-common gazebo8-common gazebo8-plugin-base libavdevice-dev libavdevice-ffmpeg56 libavfilter-dev libavresample-dev libbullet-dev libbulletsoftbody2.83.6
  libgazebo8 libignition-cmake-dev libignition-math3 libignition-math3-dev libignition-math4 libignition-math4-dev libignition-msgs libignition-msgs-dev
  libignition-msgs0 libignition-msgs0-dev libignition-transport3 libignition-transport3-dev libpostproc-dev libqwt-headers libqwt-qt5-6 libqwt-qt5-dev
  libsdformat5 libsdformat5-dev libsdl1.2debian libzmq3-dev sdformat-sdf
建议安装：
  gazebo8-doc
下列软件包将被【卸载】：
  gazebo7 gazebo7-plugin-base libgazebo7 libgazebo7-dev libignition-math2 libignition-math2-dev libsdformat4 libsdformat4-dev ros-kinetic-desktop-full
  ros-kinetic-gazebo-dev ros-kinetic-gazebo-plugins ros-kinetic-gazebo-ros ros-kinetic-gazebo-ros-pkgs ros-kinetic-simulators
下列【新】软件包将被安装：
  gazebo8 gazebo8-common gazebo8-plugin-base libavdevice-dev libavdevice-ffmpeg56 libavfilter-dev libavresample-dev libbullet-dev libbulletsoftbody2.83.6
  libgazebo8 libgazebo8-dev libignition-cmake-dev libignition-math3 libignition-math3-dev libignition-math4 libignition-math4-dev libignition-msgs
  libignition-msgs-dev libignition-msgs0 libignition-msgs0-dev libignition-transport3 libignition-transport3-dev libpostproc-dev libqwt-headers libqwt-qt5-6
  libqwt-qt5-dev libsdformat5 libsdformat5-dev libsdl1.2debian libzmq3-dev
下列软件包将被升级：
  gazebo7-common sdformat-sdf
升级了 2 个软件包，新安装了 30 个软件包，要卸载 14 个软件包，有 368 个软件包未被升级。
需要下载 97.7 MB 的归档。
解压缩后将会空出 339 MB 的空间。
您希望继续执行吗？ [Y/n] Y
 . . . . . .
Complete.
Type gazebo to start the simulator.
```

一大堆依赖会被安装。安装完成后，即可通过 `gazebo` 命令来启动 Gazebo 了。Gazebo 启动之后界面如下：

![Gazebo Main GUI](../images/1315506-ffdffcf63f81489e.png)

Gazebo 是开源的，可以通过如下的方式来下载源码：
```
hg clone https://bitbucket.org/osrf/gazebo
```

即官方是通过 **Mercurial** 这一版本工具来管理项目源码的。不过我们可以将 **Mercurial** 工程转换为 Git 工程。方法如下：
```
~/gazebo8$ hg clone https://bitbucket.org/osrf/gazebo
destination directory: gazebo
applying clone bundle from https://api.media.atlassian.com/file/764236f6-f6d2-46d8-b836-c95f6603d1a0/binary?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiIxZWZmNzMzZC05MmQwLTQ5Y2EtOGQxNS0wZmUyNDJjN2M0OWYiLCJhY2Nlc3MiOnsidXJuOmZpbGVzdG9yZTpmaWxlOjc2NDIzNmY2LWY2ZDItNDZkOC1iODM2LWM5NWY2NjAzZDFhMCI6WyJyZWFkIl19LCJuYmYiOjE1MTY4Nzg2NDksImV4cCI6MTUxNjg3OTA2OX0.ZKT8yBn3k9pPNgVC7pK6Wpe4JFNcZE-IYJdl1gNNxKs&client=1eff733d-92d0-49ca-8d15-0fe242c7c49f
adding changesets
adding manifests                                                                                                                                                  
adding file changes                                                                                                                                               
added 35951 changesets with 147449 changes to 11883 files (+581 heads)                                                                                            
finished applying clone bundle
searching for changes
adding changesets                                                                                                                                                 
adding manifests
adding file changes
added 167 changesets with 290 changes to 143 files (+3 heads)
updating to branch default
3423 files updated, 0 files merged, 0 files removed, 0 files unresolved
~/gazebo8$ mkdir gazebo_repo
~/gazebo8$ cd gazebo_repo/
~/gazebo8/gazebo_repo$ git clone git://repo.or.cz/fast-export.git .
~/gazebo8/gazebo_repo$ rm -rf .git .gitignore
~/gazebo8/gazebo_repo$ git init
~/gazebo8/gazebo_repo$ ./hg-fast-export.sh -r ../gazebo
```

在执行上面的 `hg-fast-export.sh` 命令时，可能报出如下的错误：
```
~/gazebo8/gazebo_repo$ ./hg-fast-export.sh -r ../gazebo
Warning: sanitized branch [test time stepping] to [test_time_stepping]
Warning: sanitized branch [gazebo7__default] to [gazebo7_default]
Warning: sanitized branch [Fix cmake] to [Fix_cmake]
Error: repository has at least one unnamed head: hg r6420
git-fast-import statistics:
---------------------------------------------------------------------
Alloc'd objects:       5000
Total objects:            0 (         0 duplicates                  )
      blobs  :            0 (         0 duplicates          0 deltas of          0 attempts)
      trees  :            0 (         0 duplicates          0 deltas of          0 attempts)
      commits:            0 (         0 duplicates          0 deltas of          0 attempts)
      tags   :            0 (         0 duplicates          0 deltas of          0 attempts)
Total branches:           0 (         0 loads     )
      marks:           1024 (         0 unique    )
      atoms:              0
Memory total:          2282 KiB
       pools:          2048 KiB
     objects:           234 KiB
---------------------------------------------------------------------
pack_report: getpagesize()            =       4096
pack_report: core.packedGitWindowSize = 1073741824
pack_report: core.packedGitLimit      = 8589934592
pack_report: pack_used_ctr            =          0
pack_report: pack_mmap_calls          =          0
pack_report: pack_open_windows        =          0 /          0
pack_report: pack_mapped              =          0 /          0
---------------------------------------------------------------------
```

即：
```
Error: repository has at least one unnamed head: hg r6420
```

这个错误可通过为 `hg-fast-export.sh` 命令添加 `--force` 选项来解决（[StackOverflow](https://stackoverflow.com/questions/1389307/convert-a-mercurial-repository-to-git) 有对这个问题的讨论），像下面这样：
```
~/gazebo8/gazebo_repo$ ./hg-fast-export.sh -r ../gazebo --force
```

转换完成后，执行如下命令来清理导入工具：
```
~/gazebo8/gazebo_repo$ git clean -f # remove fast-export files
```

[Gazebo 官网地址](http://gazebosim.org/tutorials?cat=guided_b&tut=guided_b1)

Done.
