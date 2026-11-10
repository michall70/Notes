> 只有raw话题会有数据，其他话题不发
> 官方也不提供支持

/utlidar/client_cmd
/utlidar/cloud
/utlidar/cloud_base
/utlidar/cloud_deskewed
/utlidar/grid_map
/utlidar/height_map
/utlidar/height_map_array
/utlidar/imu
/utlidar/lidar_state
/utlidar/mapping_cmd
/utlidar/range_info
/utlidar/range_map
/utlidar/robot_odom
/utlidar/robot_pose
/utlidar/server_log
/utlidar/switch
/utlidar/voxel_map
/utlidar/voxel_map_compressed


ros2 bag record /utlidar/cloud /utlidar/imu -o ~/utlidar_bags/utlidar_bags_$(date +%Y%m%d_%H%M%S)