cmake_minimum_required(VERSION 3.14)
project(livox_ros_driver2)

#---------------------------------------------------------------------------------------
# 编译标准与基础配置
#---------------------------------------------------------------------------------------
set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 添加宏定义，代码中可能使用了 #ifdef BUILDING_ROS2
add_definitions(-DBUILDING_ROS2)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic -Wno-unused-parameter)
endif()

#---------------------------------------------------------------------------------------
# 查找依赖 (Humble 标准)
#---------------------------------------------------------------------------------------
find_package(ament_cmake_auto REQUIRED)
find_package(PCL REQUIRED)
find_package(std_msgs REQUIRED)
find_package(builtin_interfaces REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(rclcpp_components REQUIRED)

# 自动查找 package.xml 中列出的依赖
ament_auto_find_build_dependencies()

# 查找 APR
find_package(PkgConfig)
pkg_check_modules(APR apr-1)

# 查找 Livox SDK (确保已安装在 /usr/local/lib)
find_library(LIVOX_LIDAR_SDK_LIBRARY liblivox_lidar_sdk_shared.so /usr/local/lib REQUIRED)
find_path(LIVOX_LIDAR_SDK_INCLUDE_DIR
  NAMES "livox_lidar_api.h" "livox_lidar_def.h"
  REQUIRED
)

#---------------------------------------------------------------------------------------
# 生成自定义消息 (livox_interfaces2)
#---------------------------------------------------------------------------------------
set(LIVOX_INTERFACES livox_interfaces2)
rosidl_generate_interfaces(${LIVOX_INTERFACES}
  "msg/CustomPoint.msg"
  "msg/CustomMsg.msg"
  DEPENDENCIES builtin_interfaces std_msgs
  LIBRARY_NAME ${PROJECT_NAME}
)

#---------------------------------------------------------------------------------------
# 构建库文件 (Shared Library)
#---------------------------------------------------------------------------------------
ament_auto_add_library(${PROJECT_NAME} SHARED
  src/livox_ros_driver2.cpp
  src/lddc.cpp
  src/driver_node.cpp
  src/lds.cpp
  src/lds_lidar.cpp
  src/comm/comm.cpp
  src/comm/ldq.cpp
  src/comm/semaphore.cpp
  src/comm/lidar_imu_data_queue.cpp
  src/comm/cache_index.cpp
  src/comm/pub_handler.cpp
  src/parse_cfg_file/parse_cfg_file.cpp
  src/parse_cfg_file/parse_livox_lidar_cfg.cpp
  src/call_back/lidar_common_callback.cpp
  src/call_back/livox_lidar_callback.cpp
)

# 为 Humble 链接消息类型的支持
rosidl_get_typesupport_target(cpp_typesupport_target
  ${LIVOX_INTERFACES} "rosidl_typesupport_cpp")
target_link_libraries(${PROJECT_NAME} "${cpp_typesupport_target}")

# 包含路径
target_include_directories(${PROJECT_NAME} PUBLIC
  ${PCL_INCLUDE_DIRS}
  ${APR_INCLUDE_DIRS}
  ${LIVOX_LIDAR_SDK_INCLUDE_DIR}
  3rdparty
  src
)

# 链接依赖库
target_link_libraries(${PROJECT_NAME}
  ${LIVOX_LIDAR_SDK_LIBRARY}
  ${PCL_LIBRARIES}
  ${APR_LIBRARIES}
)

#---------------------------------------------------------------------------------------
# 注册 Component 和生成执行文件
#---------------------------------------------------------------------------------------
rclcpp_components_register_node(${PROJECT_NAME}
  PLUGIN "livox_ros::DriverNode"
  EXECUTABLE ${PROJECT_NAME}_node
)

#---------------------------------------------------------------------------------------
# 安装与打包
#---------------------------------------------------------------------------------------
ament_auto_package(INSTALL_TO_SHARE
  config
  launch_ROS2
)
