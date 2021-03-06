---
title: "An example to use message_filters in ROS" 
layout: post
comments: true
date: 2018-07-15 20:00
image: /assets/images/red_light.jpg
headerImage: false
tag:
- ros kinetic
- message_filters
- sync ROS messages
category: blog
author: aravind krishnan
description: Using message_filters in ROS to sync two nodes.
# jemoji: '<img class="emoji" title=":ramen:" alt=":ramen:" src="https://assets.github.com/images/icons/emoji/unicode/1f35c.png" height="20" width="20" align="absmiddle">'
---

As quoted in ROS Wiki "__message_filters__: _a set of message filters which take in messages and may output those messages at a later time_, _based on the conditions that filter needs met_." [ROS Wiki message_filters](http://wiki.ros.org/message_filters)    

I assume that you are a little familiar with ROS - Robot Operating System and you know how to create a ROS package, build it and run the nodes. 

## System details are:  
* ubuntu 16.04
* ROS Kinetic
* code in C++

This blog will bluntly explain a simple example to use message_filters in ROS, specifically the __Policy-Based Synchronizer__ which syncs (approximate time) two nodes based on the __Approximate Time Policy__. The test workspace has a package named `learn_msg_filter`  with three cpp files.  
* firstNode.cpp - publisher
* secondNode.cpp - publisher
* combinedNode.cpp - message_filter subscriber  

For those who are here to understand directly from the code, visit my [git repo](https://github.com/aravindk2604/test_ws)  

The intention here, is to experiment by creating custom messages in ROS and try to use `std_msgs/Header` to work with `message_filters` to sync two ROS nodes that run at different frequencies (5Hz and 20Hz).  

The steps to follow are:  

* Create a ROS workspace, a package (eg. *learn_msg_filter*)
* Create a custom msg, for instance _NewString.msg_ 
* Write two separate nodes that publish data
* Write one node that subscribes to the two publishers, using message_filters 
  

## Create a custom msg

After you create a ROS workspace and a package, you start by creating your own msg. If you are not aware of how to create a custom message in ROS then you can learn from here - [How to create msg in ROS](http://wiki.ros.org/ROS/Tutorials/CreatingMsgAndSrv).  

The contents of *NewString.msg* are:

```
std_msgs/Header header
string st
```

> The `std_msgs/Header` is the datatype that consists of the timestamp and quite important for *message_filters*.  
  
The contents of `std_msgs/Header` are:

```
uint32 seq
time stamp
string frame_id
```

The *message_filters* check the **stamp** timestamp variable that consists of seconds and nanoseconds. If this value is equal or agreeably equal (according to approximate time policy) then the callback mechanism is executed and it implies that the nodes in consideration are in sync. You will understand better when I explain the code.  

Assuming you created your own message, the *catkin_make* command would have created a *.h* file inside the path *~/your_workspace/devel/include/NewString.h*. This is the file that you should inlude in the nodes that you create. 
  
## Write two publisher nodes

* The first node publishes a string *"Hello "* at 5Hz and the second node publishes a string *"world!"* at 20Hz. This string is part of the *NewString.msg* that I created. 

The *header* part of this custom msg has three parts as mentioned above. 

* The *uint32 seq* is auto generated by ROS and it is a continuous number.
* The *stamp* was assigned the value using **ros::Time::now()** which fills in the *seconds* and the *nanoseconds*.
* The *frame_id* was arbitrarily named **/myworld** for the first node and as **/robot** for the second node.

**message_filters** sees its usage greatly while dealing with the image and point cloud data and the official ROS Wiki describes an example similarly. Thus the *frame_id* in the *header* datatype might get names like */world_frame* , */robot_frame*, */camera_frame* and so on.

## First node
```
#include "ros/ros.h"
#include <sstream>
#include <bits/stdc++.h>
#include "learn_msg_filter/NewString.h"

int main(int argc, char** argv) {

    ros::init(argc, argv, "firstNode");
    ros::NodeHandle nh;
    ROS_INFO_STREAM("First node started.");
    ros::Publisher pub = nh.advertise<learn_msg_filter::NewString>("chatter", 5);
    ros::Rate loop_rate(5);

    while(ros::ok()) {
        learn_msg_filter::NewString msg;
        
        msg.header.stamp = ros::Time::now();
        msg.header.frame_id = "/myworld";

        std::stringstream ss;
        ss << "hello ";
        msg.st = ss.str();
        
        pub.publish(msg);

        ros::spinOnce();
        loop_rate.sleep();
    }

    return 0;    
}
```

>The important thing to note is that I have included the header file of the custom message **learn_msg_filter/NewString.h** so that I can use it to declare a custom variable **learn_msg_filter::NewString msg**.


## Second node
```
#include "ros/ros.h"
#include <bits/stdc++.h>
#include <sstream>
#include "learn_msg_filter/NewString.h"

int main(int argc, char** argv) {

    ros::init(argc, argv, "secondNode");
    ros::NodeHandle nh;
    ROS_INFO_STREAM("Second Node started");

    ros::Publisher pub = nh.advertise<learn_msg_filter::NewString>("anotherChatter", 5);
    ros::Rate loop_rate(20);

    while(ros::ok()) {
        learn_msg_filter::NewString msg;

        msg.header.stamp = ros::Time::now();
        msg.header.frame_id = "/robot";

        std::stringstream ss;
        ss << "world!";
        msg.st = ss.str();
        pub.publish(msg);

        ros::spinOnce();
        loop_rate.sleep();
    }
    return 0;
}
```


## Combined node
```
#include "ros/ros.h"
#include <sstream>
#include <bits/stdc++.h>
#include <message_filters/subscriber.h>
#include <message_filters/synchronizer.h>
#include <message_filters/sync_policies/approximate_time.h>
#include "learn_msg_filter/NewString.h"
#include <std_msgs/String.h>

using namespace message_filters;

void callback(const learn_msg_filter::NewString::ConstPtr& f1, 
              const learn_msg_filter::NewString::ConstPtr& s1) {

    std_msgs::String out_String;
    out_String.data = f1->st + s1->st;
    ROS_INFO_STREAM(out_String);
}

int main(int argc, char** argv) {

    ros::init(argc, argv, "combinedNode");
    ros::NodeHandle nh;
    
    message_filters::Subscriber<learn_msg_filter::NewString> f_sub(nh, "chatter", 1);
    message_filters::Subscriber<learn_msg_filter::NewString> s_sub(nh, "anotherChatter", 1);

    typedef sync_policies::ApproximateTime<learn_msg_filter::NewString, learn_msg_filter::NewString> MySyncPolicy;

    Synchronizer<MySyncPolicy> sync(MySyncPolicy(10), f_sub, s_sub);
    sync.registerCallback(boost::bind(&callback, _1, _2));
    ROS_INFO_STREAM("checking ...");
    
    ros::spin();
    return 0;
}
```

Please note that the *Subscriber* here is a method of **message_filters** and takes in the custom message *learn_msg_filter::NewString*. This code subscribes to the two publishers and *ApproximateTime Policy* is used as a synchronizer to sync them. *ExactTime Policy* doesn't work here because the timestamps of the nodes that are to be synced must be the same. Since we deliberately publish the data at different rates - 5Hz and 20Hz, *ApproximateTime Policy* was used as a demonstration.  

  
The **callback** function simply concatenates the string data from the two subscribed nodes and is output as a confirmation that the first two nodes are synced.  

## Run the Code

### Terminal 1
```
roscore
```


### Terminal 2
```
git clone git@github.com:aravindk2604/test_ws.git
cd ~/test_ws/
catkin_make
source devel/setup.bash
roslaunch learn_msg_filter combined.launch
```

Output

```
SUMMARY

PARAMETERS
 * /rosdistro: kinetic
 * /rosversion: 1.12.13

NODES
  /
    firstNode (learn_msg_filter/firstNode)
    secondNode (learn_msg_filter/secondNode)

ROS_MASTER_URI=http://192.xxx.x.xxx:11311

process[firstNode-1]: started with pid [7957]
process[secondNode-2]: started with pid [7958]

```


### Terminal 3
```
cd ~/test_ws/
source devel/aetup.bash
rosrun learn_msg_filter combinedNode
```

Output
```
[ INFO] [1531778649.755397007]: checking ...
[ INFO] [1531778650.233821710]: data: hello world!

[ INFO] [1531778650.243030068]: data: hello world!

[ INFO] [1531778650.443058176]: data: hello world!

[ INFO] [1531778650.643062540]: data: hello world!

[ INFO] [1531778650.842686741]: data: hello world!

[ INFO] [1531778651.043033154]: data: hello world!

[ INFO] [1531778651.243016201]: data: hello world!

[ INFO] [1531778651.443006341]: data: hello world!

```


Here are the output from the two publisher nodes:
### First node output
```
header: 
  seq: 1800
  stamp: 
    secs: 1531779001
    nsecs: 433235260
  frame_id: "/myworld"
st: "hello "
---
header: 
  seq: 1801
  stamp: 
    secs: 1531779001
    nsecs: 633267250
  frame_id: "/myworld"
st: "hello "
---
header: 
  seq: 1802
  stamp: 
    secs: 1531779001
    nsecs: 833271380
  frame_id: "/myworld"
st: "hello "
---
header: 
  seq: 1803
  stamp: 
    secs: 1531779002
    nsecs:  33272517
  frame_id: "/myworld"
st: "hello "
```

### Second node output
```
header: 
  seq: 9341
  stamp: 
    secs: 1531779108
    nsecs: 492513964
  frame_id: "/robot"
st: "world!"
---
header: 
  seq: 9342
  stamp: 
    secs: 1531779108
    nsecs: 542521933
  frame_id: "/robot"
st: "world!"
---
header: 
  seq: 9343
  stamp: 
    secs: 1531779108
    nsecs: 592509847
  frame_id: "/robot"
st: "world!"
---
header: 
  seq: 9344
  stamp: 
    secs: 1531779108
    nsecs: 642521959
  frame_id: "/robot"
st: "world!"
---
header: 
  seq: 9345
  stamp: 
    secs: 1531779108
    nsecs: 692493104
  frame_id: "/robot"
st: "world!"
---
header: 
  seq: 9346
  stamp: 
    secs: 1531779108
    nsecs: 742521642
  frame_id: "/robot"
st: "world!"
---
header: 
  seq: 9347
  stamp: 
    secs: 1531779108
    nsecs: 792529680
  frame_id: "/robot"
st: "world!"
---
header: 
  seq: 9348
  stamp: 
    secs: 1531779108
    nsecs: 842525970
  frame_id: "/robot"
st: "world!"
```

> The output from the nodes were recorded at different time and thus the timestamps will vary greatly. Ideally, logging the output from these two nodes and then fetching the data from that log file would yield possible sync in timestamps between the two nodes.
> 
> Also note the difference in output rate is clearly higher with the second node because it runs at 20Hz, whereas the first node runs at 5Hz.


Suggestions to improve this example are most welcome. This was an attempt for me to understand and use message_filters for a datatype different from the commonly used *Image* and *CameraInfo* topics under *sensor_msgs* as given in the official ROS Wiki page.