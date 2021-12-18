---
title: "Inteceptor Robot: Final report"
date: 2021-12-17T20:36:16-08:00
draft: false
author: ["Robert Peltekov", "Saahil Parikh", "Ryan Adolf"]
---

{{< youtube a_u6m24022I >}}

# Introduction

Interceptor is a robot that can stop objects from rolling off a table. It detects a moving tennis ball, predicts where the ball will roll off the table, then moves its gripper to cut off the ball’s path. Our application shows how robots can quickly predict outcomes of physical situations and respond to them. Potential applications may be ensuring safety in environments where robots work (e.g. if a part goes flying, can the robot stop it from hitting a human?) or working with humans (e.g. performing daycare for a baby that’s throwing its toys around).

# Design Criteria

The robot must be able to:
1. Use AR tags to figure out how the camera, robot, and table are positioned relative to each other.
2. Accurately detect the position of the tennis ball.
3. Accurately determine the trajectory of the tennis ball, with minimal noise.
4. Move its gripper to intercept the tennis ball, but avoid hitting the table.
5. Perform steps 2-3 fast enough to intercept the ball before it rolls off.

# Trade-offs
- Vision system: We had to choose between making our position prediction fast versus making it accurate. We chose accuracy because our design criteria because any noise in the position data will make velocity data even noisier. For this reason we chose to use a Kinect with point cloud data and process images at medium resolution, compared to faster solution like using a non-depth camera or sampling at lower resolution.
- Kalman filter: There’s an inherent tradeoff between exactly following input data and relying on the predicted state. We tuned the filter to closely fit the input data with a small amount of estimation since the ball movements can be erratic.
- Flexibility of motion planning: We chose to use the ROS MoveIt package to do motion planning for our arm. The package is flexible enough that it allows us to intercept the ball no matter what the height of the table is and also avoid hitting the table. However, MoveIt is slow and we considered replacing it with a custom solution like other teams have done. We evaluated using a hashmap to find joint positions, but this approach does not simultaneously meet our flexibility and obstacle avoidance requirements.

# Implementation

## Hardware
- Sawyer: Multi jointed arm robot
- Intel Realsense: Time of Flight Depth Camera
- Tennis Ball: Test object

## Software

{{< ourcode >}}
Our code: <a href="https://github.com/SaahilParikh/Interceptor">https://github.com/SaahilParikh/Interceptor</a>
{{< /ourcode >}}

### AR Alvar node
We use two ar_track_alvar nodes for detecting AR tags on the table in the viewport of the depth camera and the robot:
For the camera, we run a standard ar_track_alvar node. This node runs for the whole duration of the project, and the camera is always in view of the ar tag.
For the robot, we modified a copy of the ar_track_alvar source to publish transforms from the Sawyer arm camera with different names. This node only runs for the initialization phase of operation where the arm is in position to see the non moving ar tag. After Initialization is over and the static transform of the ar tag to the robot is published, this node shuts down.
### Tf Broadcaster node
This node publishes a static transform from the ar tag to the base of the robot
### Ball segment node
Using OpenCV, the Ball Segment node publishes the current position of the ball relative to the constructed TF reference tree. The segmentation of the images happens in the following manner:
1. Transform color RGB image from HSV (hue, saturation, value)
2. Threshold filter for the tennis ball neon green yellow hue
3. Use CV Erosion to smooth image and remove noise
4. Use CV Contours to trace contours of isolations in image
5. Publish an image with pixels filled only in the area of the largest contour
6. Mask the depth points with the segmented ball image
7. Calculate the center point from the resulting masked set of depth points
8. Publish the center point in the space of the camera frame
### Motion predict node
Using the ball coordinates transformed relative to the reference frame AR tag on the table. The node uses a Kalman filter to smooth position data and predict the velocity. With the predicted velocity, the node determines where the ball will intersect the edge of the table.
### IK node
Using the goal position from the motion predict node, we run ik using tracik to move sawyers gripper to the goal position. ik is implemented using move it which also allows us to implement obstacle avoidance to not hit the table or the side walls. The ik node waits for the goal position to update ( using std and stats to define update ) and then quickly moved sawyers multijointed arm. the ik solver uses distance to assist in the rtt search over the space.
### Realsense node
We use Intel’s ROS package for the Realsense camera to publish point clouds and images from the depth camera.
Additionally the realsense node takes care of the following functions: Unwarping the camera image using the camera matrix; Computing the color image projection onto the depth cloud and publishing the combined point cloud.
### MoveIt node
The MoveIt node performs inverse kinematics and path planning, creating a series of steps of joint angles for the Sawyer to move to so that it reaches a desired end effector position without hitting the table.
### Joint trajectory action server node
Not a node that we made. this node is to allow sawyer to know it’s joints?

# Results

{{< youtube BneLKw1n3bc >}}

The robot is able to successfully predict where the ball will fall from the table and move its gripper to this spot; our only major issue is the lag in the system, which causes the robot to move a few seconds after the ball has started rolling. If we were to continue this project, we would implement a different inverse kinematics solver utilizing less joints of the sawyer arm to more efficiently compute the IK problem and more easily follow a path within the constrained set of movements we need.

{{< slides >}}

# Team

{{< profile name="Robert Peltekov" image="/robert.jpg" >}}
A junior double majoring in EECS and Business at UC Berkeley. Love everything engineering-related from robotics, hardware design to space! In freetime I love to skate both in parks and down hills!
{{< /profile >}}

{{< profile name="Saahil Parikh" image="/saahil.jpg" >}}
A third year eecs student.
{{< /profile >}}

{{< profile name="Ryan Adolf" image="/ryan.jpg" >}}
It’s my third year at Berkeley majoring in EECS. I love engineering, design, art, and just making stuff in general!
{{< /profile >}}

----

# Demo Day Presentation

<div style="position: relative; padding-bottom: 61.25%; height: 0; overflow: hidden;">
  <iframe src="https://docs.google.com/presentation/d/e/2PACX-1vQupm6jq58CvvSuSTmhdv3kLBg41_WNj4BERRBdKcsQbPSMDoHZ8UJmelcilAyYsgLP-digruSQ76fV/embed" style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border:0;" allowfullscreen title="Google Slides Presentation"></iframe>
</div>
