# University of Oslo, INF3480/4380 ROS Lecture

## Intro
In this tutorial we work with a bit older version - ROS Indigo and Ubuntu Trusty 14.04. We will be installing MoveIt! And ROS Industrial frameworks and writing a simple script to move the simulated robot. The script will run in our created ROS package and can be easily adaptable to be used with real robot.

If you don't have Ubuntu and ROS on your computer, you can use this virtual machine: http://nootrix.com/software/ros-indigo-virtual-machine/

Having done this tutorial, you will easily complete an optional ROS assignment where you get to execute your written trajectories on the real UR5 robot.

##Install ROS

http://wiki.ros.org/indigo/Installation/Ubuntu

## Install MoveIt! and ROS Industrial

### Install MoveIt!

`sudo apt-get install ros-indigo-moveit-full`

### Install ROS Industrial

`sudo apt-get install ros-indigo-industrial-core`

### Create catkin workspace
Let's create a catkin workspace:
```
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/src
catkin_init_workspace
```
Even though the workspace is empty (there are no packages in the 'src' folder, just a single CMakeLists.txt link) you can still "build" the workspace:
```
cd ~/catkin_ws/
catkin_make
```

Source the catkin_ws so it is found by terminal
```
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
```
To make sure your workspace is properly overlayed by the setup script, make sure ROS_PACKAGE_PATH environment variable includes the directory you're in

`echo $ROS_PACKAGE_PATH`

## Install UR5 Packages
We want to install from source, so we can modify source files easily to our needs:
```
cd ~/catkin_ws/src
git clone https://github.com/ros-industrial/universal_robot.git
cd ..
catkin_make
```

##Create inf3480 catkin package
http://wiki.ros.org/ROS/Tutorials/CreatingPackage

Create a package for INF3480 with required dependencies
```
cd ~/catkin_ws/src
catkin_create_pkg ur5_inf3480 moveit_ros_move_group moveit_planners_ompl moveit_ros_visualization joint_state_publisher robot_state_publisher xacro ur_description moveit_msgs moveit_core moveit_ros_planning_interface
```
Create the launch directory
```
cd ~/catkin_ws/src/ur5_inf3480
mkdir launch
cd launch
```
**Create the launch file for running the UR5 simulation named “ur5_launch_inf3480.launch”**
```
<!-- ROS Demo INF3480 -->
<launch>
    <!-- If sim=false, then robot_ip is required -->
    <arg name="sim" default="true" />
    <arg name="robot_ip" unless="$(arg sim)" />
    <!-- By default, we are not in debug mode -->
    <arg name="debug" default="false" />

    <!-- Limited joint angles are used. Prevents complex robot configurations and makes the planner more efficient -->
    <arg name="limited" default="true" />

    <!-- Load UR5 URDF file - robot description file -->
    <include file="$(find ur5_moveit_config)/launch/planning_context.launch">
      <arg name="load_robot_description" value="true" />
      <arg name="limited" value="$(arg limited)" />
    </include>

    <!-- If sim mode, run the simulator -->
    <group if="$(arg sim)">
      <node name="joint_state_publisher" pkg="joint_state_publisher" type="joint_state_publisher">
        <param name="/use_gui" value="true"/>
        <rosparam param="/source_list">[/move_group/fake_controller_joint_states]</rosparam>
      </node>
    </group>

    <!-- If using real robot, initialise connection to the real robot -->
    <group unless="$(arg sim)">
      <include file="$(find ur5_moveit_config)/launch/ur5_bringup.launch">
        <arg name="robot_ip" value="$(arg robot_ip)" />
      </include>
    </group>

    <!-- Given the published joint states, publish tf for the robot links -->
    <node name="robot_state_publisher" pkg="robot_state_publisher" type="robot_state_publisher" respawn="true" output="screen" />

    <!-- Launch the move group for motion planning -->
    <group if="$(arg sim)">
        <include file="$(find ur5_moveit_config)/launch/move_group.launch">
            <arg name="limited" value="$(arg limited)" />
            <arg name="allow_trajectory_execution" value="true"/>  
            <arg name="fake_execution" value="true"/>
            <arg name="info" value="true"/>
            <arg name="debug" value="$(arg debug)"/>
        </include>
    </group>

    <group unless="$(arg sim)">
        <include file="$(find ur5_moveit_config)/launch/move_group.launch">
            <arg name="limited" default="true" />
            <arg name="publish_monitored_planning_scene" value="true" />
        </include>
    </group>

    <!-- Launch the RViz visualizer -->
    <include file="$(find ur5_moveit_config)/launch/moveit_rviz.launch">
        <arg name="config" value="true" />
    </include>

    <!-- Optionally, you can launch a database to record all the activities -->
    <!-- <include file="$(find ur5_moveit_config)/launch/default_warehouse_db.launch" /> -->

    <!-- Launch our own script -->
    <!-- <node name="inf3480_move_robot" pkg="ur5_inf3480" type="inf3480_move_robot" respawn="false" output="screen"></node> -->
</launch>
```
Let’s try to build the workspace and see if it works!
```
cd ~/catkin_ws
catkin_make
```
Run the launch file

`roslaunch ur5_inf3480 ur5_launch_inf3480.launch`

Try playing around with motion planners by manually defining the position of the robot

##Add our own node
Create the src directory
```
cd ~/catkin_ws/src/ur5_inf3480
mkdir src
cd src
```
##Download the script into the src folder

`wget https://raw.githubusercontent.com/jmiseikis/ur5_inf3480/master/src/inf3480_move_robot.cpp`

Modify the CMakeLists.txt to compile the new node. The following lines should be included or uncommented
```
include_directories(
  ${catkin_INCLUDE_DIRS}
)
```
`add_executable(inf3480_move_robot src/inf3480_move_robot.cpp)`
```
target_link_libraries(inf3480_move_robot
  ${catkin_LIBRARIES}
)
```
In the launch file, uncomment the last line:

`<node name="inf3480_move_robot" pkg="ur5_inf3480" type="inf3480_move_robot" respawn="false" output="screen"></node>`

Now, let’s take a detailed look into the source code and how to plan robot movements!

Full catkin package can be found:
https://github.com/jmiseikis/ur5_inf3480

You can check out the source code:
git clone https://github.com/jmiseikis/ur5_inf3480.git

If you make any modifications and updates, please feel free to fork the repository and send pull request! Especially as suggestions for additions for next year students, it’s highly appreciated!

## (Optional Extra) Leap Motion

If you have Leap Motion sensor, we can integrate Leap Motion controller into the script.

You will need to download and install Leap Motion SDK. Please follow the instructions for installation and test if it's functioning correctly https://developer.leapmotion.com/v2

And install the leap motion ROS package

`sudo apt-get install ros-indigo-leap-motion`

And follow instructions to setup the rest of the enviroment http://wiki.ros.org/leap_motion

Please note that sometimes you need to restart leapd daemon service, because WebSockets for transmitting data get 'stuck'

`sudo service leapd restart`

To test the Leap Motion with ROS, you can run the following scripts. Always have the Leap Motion controller connected via USB.

In one terminal window start ROS Core

`roscore`

While in another terminal window run the Leap Motion sender script

`rosrun leap_motion sender.py`

Now you should see the live text stream with data coming from Leap Motion sensor.

In our script, Leap Motion controls the location of the obstacle. To run it, uncomment the whole section between the following two lines to control the obstacle position in RViz.

Uncomment all the lines from 

`// ############ LEAP MOTION ############`

To

`// ############ END ############`

While leaving all the pre-programmed robot move scripts commented out, like this:
```
// Pre-programmed robot move
//moveRobotToHome();
//moveRobotToHomeWithFloor();
//moveRobotCartesianPath();
```
But it can be easily integrate to guide the end effector position of the robot. Can you make it happen?

# What if I'm using ROS Jade or other ROS version?

There were few issues with ROS Jade not having debian packages (cannot use apt-get to install them) for MoveIt!

In this case, you have to compile MoveIt! from source by following this guide, use ROS Jade Steps 2&3: http://moveit.ros.org/install/ 

# How to run Ubuntu on Mac

I have received many questions during the lecture about how to dual boot macs to run Ubuntu. So here are some links to the instructions for you to follow.

## Create bootable Ubuntu USB

Check the second answer (with 6 votes) how to do it using terminal http://askubuntu.com/questions/28495/how-do-i-get-my-mac-to-boot-from-an-ubuntu-usb-key

## How to install Ubuntu alongside OS X

Follow these instructions to install rEFInd boot loader and Ubuntu. **BUT BE AWARE** I suggest to use the "Something else" option in the Ubuntu installation step instead of "Install Ubuntu alongside OS X". I had that screwing up my OS X installation, **so use at your own risk!** I definitely recommend to have a back up before trying!

http://www.howtogeek.com/187410/how-to-install-and-dual-boot-linux-on-a-mac/

For "Something else" you can follow part of this guide (steps from "Resize Partitions" to "Installer") http://www.makeuseof.com/tag/install-linux-macbook-pro/

Also, sometimes wireless drivers do not work for Macs, so you need to install two debian packages (simply double-click them in Ubuntu) to make WiFi card work. They are located as a part of this repository: https://github.com/jmiseikis/ur5_inf3480/tree/master/UbuntuWiFiFix

# Contact

For any other issues or if you want to use this package, please contact me. You can find my contact details on UiO profile:
http://www.mn.uio.no/ifi/english/people/aca/justinm/
