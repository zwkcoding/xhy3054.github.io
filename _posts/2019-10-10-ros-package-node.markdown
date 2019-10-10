---
layout: post
title: ros中的包与节点与roslaunch的使用
date: 2019-10-10 10:07:24.000000000 +09:00
img:  one-piece/one-piece17.jpg # Add image post (optional)
tag: [tools]
---

之前写过一篇[关于ros入门的博客]()，但是这段时间我再看时发现当时比较关注全面性，所以在里面可以看到ros里几乎所有的基本知识。但是通篇看下来每一个讲的都没有很仔细，都是概念性的讲解。使得看完后，大家似乎对于ros的实现机制与功能模块有了一定的理解，但是如何使用ros呢？如何开发ros呢？好像还是有一些不是很懂。所以这一篇博客就会从ros使用的案例来说明，如何使用一个最小ros的功能包来跑起来节点。

# 1. 功能包
> **功能包（package）**是构成ROS的基本单元。ROS应用程序是以功能包为单位开发的。功能包包括至少一个以上的节点或拥有用于运行其他功能包的节点的配置文件。它还包含功能包所需的所有文件，如用于运行各种进程的ROS依赖库、数据集和配置文件等。

- 在工作空间中创建包`new_package`，依赖于包`depend_p1`与`depend_p2`
```bash
cd ~/catkin_ws/src
catkin_create_pkg new_package depend_p1 depend_p2 
```
---

- 查看一个包的直接依赖`rospack depends1 new_packsge`

- 查看一个包的所有依赖（包括直接与间接）`rospack depends new_package`

- 编译包，直接编译整个工作空间就可以了
```
cd ~/catkin_ws
catkin_make
```
---

> **元功能包（metapackage）**是一个具有共同目的的功能包的集合。例如，导航元功能包包含AMCL、DWA、EKF和map_server等10余个功能包。

# 2. 节点
> **ros的节点**说到底就是一个操作系统上运行起来的程序。而这个可执行文件是依赖于某一个功能包的。

如同前一节，假设我们创建了一个功能包`ros_tutorials_topic`，这个功能包主要的作用就是向初学者演示两个节点如何通过话题（topic）来通信。

- 首先，这个功能包中会有两个节点（也就是会生成两个可执行文件），一个是话题发布节点，一个是话题订阅节点。

- 生成这两个节点的方法比较简单，就是在功能包文件夹下新建`src`文件夹，在里面编写两个节点的源代码，然后编译生成可执行文件即可。在CMakeLists中如下生成的两个可执行文件就是该功能包的两个节点。（添加依赖、链接什么的省略了）

```
add_executable(topic_publisher src/topic_publisher.cpp)
add_executable(topic_subscriber src/topic_subscriber.cpp)
```

- 在使用`catkin_make`编译之后可执行文件（节点就生成了），下面就可以运行这两个节点了。

## 2.1 使用`rosrun`运行节点

- 下面三条命令分别新开一个终端执行
```bash
$ roscore
$ rosrun ros_tutorials_topic topic_publisher
$ rosrun ros_tutorials_topic topic_subscriber
```

> 使用`rosrun`来启动节点，只需要知道节点所属的功能包的名字，然后知道节点对应的可执行文件的名称，即可找到该节点对应的可执行文件并启动节点了。（如果是python脚本的节点，可执行文件的名称就是python脚本的名字去除后缀名）

## 2.2 使用`roslaunch`运行节点

- 使用`rosrun`来运行节点实在太麻烦，需要开太多的终端。此时我们可以使用`roslaunch`来同时启动多个节点。同时，它还有很多灵活的地方，比如修改参数与节点的名称，设置节点的命名空间等。

- 首先需要创建一个`×.launch`文件，它基于`XML`，提供了标签选项。

```xml
<launch>
    <node pkg="ros_tutorials_topic" type="topic_publisher" name="topic_publisher1"/>
    <node pkg="ros_tutorials_topic" type="topic_subscriber" name="topic_subscriber1"/>
    <node pkg="ros_tutorials_topic" type="topic_publisher" name="topic_publisher2"/>
    <node pkg="ros_tutorials_topic" type="topic_subscriber" name="topic_subscriber2"/>
</launch>
```

- 在上述的标签中，分别代表如下含义
    - $node$ 表示运行的节点
    - $pkg$ 表示该节点所属的功能包名称
    - $type$ 表示实际运行的节点的名称（也就是该节点对应的可执行文件名称）
    - $name$ 节点启动后重新起的名字，如果只有一个同类型的节点，一般取的name与type相同

- 运行该`union.launch`文件

```bash
$ roslaunch ros_tutorials_topic union.launch --screen
```
---
> 此处需要说明的是使用`roslaunch`启动多个节点时，默认这些节点的输出（info、error）等不会打印在终端。所以添加一个`--screen`，则所有节点的输出会打印到终端上，方便调试。

- 查看运行结果

```bash
$ rosnode list
/rosout
/topic_publisher1
/topic_publisher2
/topic_subscriber1
/topic_subscriber2
```
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/ros/launch1.PNG"/>
</div>

> 我们会发现此时两个订阅者都订阅了两个发布者发布的话题，这是因为两个发布者发布的话题名称相同，两个订阅者订阅的话题名称也一样。

- 为了使得`subscriber1`只订阅`publisher1`发布的话题，`subscriber2`只订阅`publisher2`发布的话题。我们可以将launch文件进行如下改写

```xml
<launch>
    <group ns="ns1">
        <node pkg="ros_tutorials_topic" type="topic_publisher" name="topic_publisher"/>
        <node pkg="ros_tutorials_topic" type="topic_subscriber" name="topic_subscriber"/>
    </group>
    <group ns="ns2">
        <node pkg="ros_tutorials_topic" type="topic_publisher" name="topic_publisher"/>
        <node pkg="ros_tutorials_topic" type="topic_subscriber" name="topic_subscriber"/>
    </group>
</launch>
```
---

- 上面新的标签如下：
    - $group$ 是对指定节点进行分组的标签
    - $ns$ 是group的选项，代表命名空间，是给这个组起了一个名字，使其与其他组区分开来

- 新的launch文件运行后，使用`rqt_graph`来查看各个节点之间的关联，可以看到我们实现了目的。

<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/ros/launch2.PNG"/>
</div>

## 2.3 launch的标签
在launch文件中根据XML的编写方式可以灵活的实现ros节点的启动。launch中会使用到的标签如下：

- `launch` 指roslaunch语句的开始与结束

- `node` 这是节点运行的标签，可以在标签里修改功能包、节点名称、执行名称等

- `machine` 可以设置运行该节点的PC名称、address、ros-root还有ros-package-path等

- `include` 可以加载属于同一个功能包或不同功能包的另一个launch文件，并运行

- `remap` 可以更改节点名称、话题名称等等，在节点中用到的ROS变量的名称

- `env` 设置环境变量，如路径和IP（较少使用）

- `param` 设置参数名称、类型、值等

- `rosparam` 可以像rosparam命令一样，查看修改load、dump和delete等参数信息。

- `group` 用于分组正在运行的节点

- `test` 用于测试节点。类似于node，但是可以有用于测试的选型

- `arg` 可以在launch文件中定义一个变量，以便在像下面这样运行时更改参数

此处举一个例子，大致演示一下如何使用`arg`与`param`

```xml
<launch>
    <arg name="update_period" default="10"/>
    <param name="timing" value="$(arg update_period)"/>
</launch>
```
---
在运行上述launch文件时，可以指定上述参数的值
```bash
$ roslaunch my_package my_package.launch update_period:=30
```
---

# 参考文献
- [1]. ROS Robot Programming. YoonSeok Pyo, HanCheol Cho, RyuWoon Jung, TaeHoon Lim

