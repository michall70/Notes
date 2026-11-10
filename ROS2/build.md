```bash
colcon build --packages-select my_first_pkg --symlink-install

soruce /home/michall/AAAProjects/go2_ws/ros2_ws/install/setup.bash

ros2 launch my_first_pkg kitti_pub_sub.launch.py 
```