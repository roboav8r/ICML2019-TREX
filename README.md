# ICML2019-TREX (openai_ros)

This is a ROS implementation of the TREX code described in the master branch. While the master branch is designed to improve performance of RL agents in the Atari and Mujoco environments, this branch extends TREX functionality to the domain of ROS robots. It does this by using the openai_ros package, originally developed by The Construct and modified for the University of Texas' robots.

There are a few changes from the original code, notably:
- RL and preference model training & evaluation are performed sequentially by different scripts
- The use of ROS parameters instead of arguments
- The use of stable_baselines instead of openai-baselines
- Creation of custom ROS environments for Gym, which are loaded into the Gym environment register at runtime
- General updates to use more recent versions of Tensorflow, Gym, and other supporting software

# Prerequisites
This assumes you have the following software installed:
- Ubuntu 20.04
- ROS Noetic (required since TREX uses Python 3 and Noetic is the only Python 3 version of ROS 1)
- Conda (optional, but recommended to ensure Python 3.6 version compatibility and package installation)

This software was run using NVidia driver 440.100, CUDA 10.1, cuDNN 7.6.5 on a GeForce RTX 2060 Super 8GB.

# Installation

NOTE: The UTBox folders and Github repos may require special access.

If you can't get access to UTBox links or UT Nuclear Robotics Github, please contact Dr. Pryor. For access to this (or any other Github repo), please contact me.

## Copy Gazebo models into your model folder
First, copy over the gazebo model/mesh files from the cloud drive here: https://utexas.app.box.com/folder/121632471058

Copy the /Stairs and /ut_mesh folders into your ~/.gazebo/models folder.

For example, the completed file path should be ~/.gazebo/models/ut_mesh

## Create & activate Conda environment (Optional but recommended)
In a terminal:
```
conda create --name TREX python=3.6 setuptools=45 numpy=1.16
conda activate TREX
export PYTHONPATH=~/anaconda3/envs/TREX/lib/python3.6/site-packages:$PYTHONPATH # Necessary to use Conda with ROS

```
## Install apt packages
In a terminal:
```
sudo apt-get update && sudo apt-get install cmake libopenmpi-dev python3-dev zlib1g-dev git
```
## Create a catkin workspace, clone ROS packages, and build catkin workspace
```
cd ~
mkdir -p trex_ros_ws/src
cd ~/trex_ros_ws/src
git clone https://github.com/roboav8r/openai_ros.git
git clone -b openai --single-branch https://github.com/UTNuclearRobotics/walrus_description
git clone -b openai --single-branch https://github.com/UTNuclearRobotics/walrus_gazebo
git clone -b openai_ros --single-branch https://github.com/roboav8r/ICML2019-TREX.git
chmod +x ICML2019-TREX/scripts/* # Ensure all TREX scripts are executable
cd ~/trex_ros_ws
pip install -r src/ICML2019-TREX/requirements.txt # Install remaining Python packages
catkin build
```

## Clone Turtlebot3 model and environment
If you desire to use the Turtlebot3 Burger by ROBOTIS for simulation, clone and build the following packages in your catkin workspace.
```
cd ~/trex_ros_ws/src
git clone https://github.com/AdrianAbeyta/turtlebot3.git
cd turtlebot3
git clone https://github.com/AdrianAbeyta/turtlebot3_simulations.git
cd ~/trex_ros_ws
catkin build
```

# Usage
TREX (openai_ros) uses ROS to initiate all trainings via a launch file. Each launch file contains the scripts and configuration paramaters necessary for operation. Configurations for modifying variables can be can be found in the config folder along with a description of each variable. 

## Preparing the workspace
Before each session, make sure that the Conda environment is active and appended to the PYTHONPATH variable (if applicable), and that the workspace is sourced:
```
conda activate TREX # If applicable
export PYTHONPATH=~/anaconda3/envs/TREX/lib/python3.6/site-packages:$PYTHONPATH # Necessary ONLY IF using Conda with ROS
source ~/trex_ros_ws/devel/setup.bash
```
## Killing Gazbeo
Sometimes the Gazebo application can remain open in the background after training and cause some issues layer on when trying to retrain.  Set the following alias in your .bashrc to kill all gzservers and gzclients, in the case that Gazebo does not shut down after training.
```
alias killgazebo="killall -9 gazebo & killall -9 gzserver  & killall -9 gzclient"
```

## Modify config files as needed
The provided /config/\*yaml files were developed on my personal computer; you will need to change the trex_ros_ws parameter.

For example, if you wish to train the walrus_balance task, change ros_ws_abspath from "/home/jd/trex_ros_ws" to "/home/[your user name]/trex_ros_ws"

## Generate checkpointed RL models
Note: By default, the example displayed will demonstrate learned obstical avoidance of the Turtlebot3 Burger by ROBOTIS. For demonstration of obstical avoidance via the Walrus platform, . 

- In order to train the TREX preference model, you first must develop checkpointed RL models for use as demonstrations. Stable-baselines Proximal Policy Optimization (PP02) was used in combination with MLP in order to generate the checkpoints. In the terminal of your choice, use the roslaunch command to begin training:

```
$ roslaunch trex_openai_ros gen_checkpoints_ppo2.launch # By default, uses the Turtlebot3 with turtlebot_world environment
```
or
```
$ roslaunch trex_openai_ros gen_checkpoints_ppo2.launch robot:='walrus' task:='walrus_balance' # To generate checkpointed balancing Walrus agents
```
or
```
$ roslaunch trex_openai_ros gen_checkpoints_ppo2.launch robot:='walrus' task:='walrus_stairs' # To generate checkpointed stairclimbing Walrus agents
```

Other available walrus tasks include walrus_nav (a 2d navigation task).

Gazebo should appear and a visual representation of training will begin. The checkpoints will be saved in the designated checkpoint_dir set in the appropriate config/\*yaml file.

## Train the TREX preference model
- Once checkpoints are generated, a neural network will be created from these checkpoints to approximate a learned reward function. In the terminal of your choice, use the roslaunch command to begin training:

```
  $ roslaunch trex_openai_ros gen_pref_model.launch robot:='walrus' task:='walrus_balance'
```
The checkpoints will be loaded based on the designated save_path set in the launch file. The learned reward function will be saved in the designated pref_model_dir set in the config file. 


# Known Issues / Potential Improvements / Future Work

- Currently, the preference model only accepts GTraj trajectory types. This could be expanded to The other trajectory types in mujoco/preference_learning.py 
- Occasional simulation robot reset issues during training.
- Set up the git repo to add other repositories as submodules, so that a git clone --recursive of this repo is the only clone command needed.
- Alternate meshes of the UT campus can be found in https://utexas.app.box.com/folder/121689530768. These meshes feature fewer polygons for quicker loading, and have color as well. There is also a Pandas datafile which contains a lookup table to determine the terrain type for a given (x,y) position on the mesh. Ideally, you could get your current odometry (i.e. x,y pose) in gazebo, query the Pandas database with your (x,y), and determine the terrain classification. We wanted to implement this function but ran out of time in Summer 2020. The enclosed .boxnote contains more information.
- Change the ros_ws_abspath to a relative path, or one that uses ~/ so that the user doesn't need to change it manually
- Retrain tensorflow TREX learned reward with PPO2 and do quatitative results. 

