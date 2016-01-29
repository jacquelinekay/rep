REP: #
Title: Proposal for Dynamically Reloading the Robot Description
Authors: Jackie Kay and Ian McMahon
Status: Draft
Type: Standards
Content-type: text/x-rst
Created: 29-Jan-2016
Post-History

Abstract
========


Specification
=============

Motivation
==========

Since the beginnings of ROS, the `robot_description` parameter has been
immutable: it is set on the parameter server on robot bringup, loaded
into memory on initialization of other nodes, and never checked again.
However, in reality, the physical model of the robot is mutable.
Consider a robot calibration process that writes a new URDF after the robot
takes data about its own kinematics.
Or, consider a robot with hotswappable end effectors (able to be changed at runtime).
In the current paradigm, if the `robot_description` parameter changes,
then all the nodes that load the parameter must be reset for the change to be propagated throughout the system.
It is highly inconvenient to restart nodes during runtime, since it incurs a
large time cost (waiting for the nodes to shut down and then reconnect to the
network) and code complexity cost (due to the orchestration required to restart
nodes and restore their previous state).
Therefore the community should consider a standard and convenient way to
propagate changes to `robot_description` to `robot_state_publisher` and other
nodes that take `robot_description` as an input.


Rationale
=========

Backwards Compatibility
=======================

Reference Implementation
========================


disorganized parts of the discussion so far:
==============================================
It has been suggested that `robot_description` should have been constructed as
a latched topic in the first place.
However, changing `robot_description` from a parameter to a topic would require
a massive amount of downstream refactoring.
This approach may be considered for ROS 2, while a more backwards compatible
solution will be proposed in this REP.

`dynamic_reconfigure` offers a way to reconfigure a node's parameters at runtime.
However, the `dynamic_reconfigure` API is Python-only, which is a limitation
for downstream nodes (`robot_state_publisher`, for example, is implemented in C++).
Besides, because the `dynamic_reconfigure` API is separate from the `rosparam`
API, this approach has the same disadvantage due to refactoring as changing
`robot_description` to a latched topic.
This is another limitation addressed in ROS 2, since parameters will be dynamic by default.

Comparing current reference implementations/workarounds:

`mutable_robot_state_publisher <https://github.com/RethinkRobotics/mutable_robot_state_publisher>`
is a fork of robot_state_publisher developed for a robot with interchangeable grippers, the Baxter Research Robot.

From Ian McMahon:
"mutable_robot_state_publisher is capable of dynamically accepting joint and
link fragments from nodes on the network, incorporating them into the URDF,
and rebuild any relevant kinematic chains (KDL chains for Baxter's arms).
We introduce a new parameter called robot_base_description which contains the
core robot URDF and is treated as modifiable by the robot_state_publisher.
Also, rather than a Service or Action interface, we use standard Topics with
a URDFConfiguration.msg to add or remove fragments.
Those new fragments are continually published to the topic with unchanging
timestamps. Its assumed that when an timestamp is updated, the
robot_description parameter needs to be rebuilt.
Any new node can come up to speed (and stay updated) by listening to the same
/robot/urdf topic, then building onto the URDF located in robot_base_description parameter."

There is currently a pull request in `robot_state_publisher <https://github.com/ros/robot_state_publisher/pull/31>`
which adds a `std_srvs/Trigger` service to the `robot_state_publisher` node, `reload_robot_model`.

From Nikolas Engelhard:
"Node now advertises a reload_robot_model (std_srvs/Trigger) service that allows
another node to trigger a reload of the robot model (e.g. after an calibration
that changed the urdf-model).

So far, the robot_state_publisher had to be restarted, which is not suitable
for an automated process. With the new adaption, the server can be restarted
by calling the service:
::

    c = rospy.ServiceProxy("/robot_state_publisher/reload_robot_model", Trigger)
    result = c()
    print result.success, result.message

If the reload failed (e.g. due to a wrong urdf-model (rosparam set
/robot_description "InvalidFooBar"), the publisher stops and no tf-frames are
published anymore so that no one is working with the deprecated model. If the
/robot_description is corrected and the service called again, the publisher
continues to publish frames.

Publication of frames is stopped before loading the model, so that reading and
writing does not happen at the same time."

