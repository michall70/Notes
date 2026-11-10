```bash
ros2 bag record ... -o ~/utlidar_bags_$(date +%Y%m%d_%H%M%S)
```

```bash
ros2 bag record /camera/color/image_raw /camera/depth/image_rect_raw /camera/imu /camera/extrinsics/depth_to_color -o ~/rosbags/D435i_bags/D435i_bags_$(date +%Y%m%d_%H%M%S)
```