# ============== 基础设置 ==============
cmake_minimum_required(VERSION 2.8.3)  # CMake 最低版本要求
project(fast_lio)  # 项目名称（ROS 包名），对应 package.xml 中的 <name>


SET(CMAKE_BUILD_TYPE "Debug")  # 编译模式：Debug（保留调试信息）或 Release（优化性能）

# C++14 标准
ADD_COMPILE_OPTIONS(-std=c++14 )
ADD_COMPILE_OPTIONS(-std=c++14 )
set( CMAKE_CXX_FLAGS "-std=c++14 -O3" )  # -O3 表示最高优化级别

# 定义宏 ROOT_DIR，值为当前源码目录路径
# 代码中可用 ROOT_DIR 获取项目根目录（用于日志、PCD输出路径）
add_definitions(-DROOT_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/\")

# 启用异常处理
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fexceptions" )
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
# -pthread：启用多线程支持
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14 -pthread -std=c++0x -std=c++14 -fexceptions")

# 打印 CPU 架构信息
message("Current CPU archtecture: ${CMAKE_SYSTEM_PROCESSOR}")
# 仅对 x86/AMD64 架构启用多核处理
if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)" )
  include(ProcessorCount)
  ProcessorCount(N)  # 获取 CPU 核心数
  message("Processer number:  ${N}")
  
  # 根据核心数设置并行线程数
  if(N GREATER 4)
    add_definitions(-DMP_EN)         # 启用多核
    add_definitions(-DMP_PROC_NUM=3) # 使用3个核心
    message("core for MP: 3")
  elseif(N GREATER 3)
    add_definitions(-DMP_EN)
    add_definitions(-DMP_PROC_NUM=2)
    message("core for MP: 2")
  else()
    add_definitions(-DMP_PROC_NUM=1) # 单核
  endif()
else()
  add_definitions(-DMP_PROC_NUM=1)   # ARM 等架构默认单核
endif()

# 查找并启用 OpenMP（并行计算库）
find_package(OpenMP QUIET)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}   ${OpenMP_C_FLAGS}")

# Python 支持
find_package(PythonLibs REQUIRED)
find_path(MATPLOTLIB_CPP_INCLUDE_DIRS "matplotlibcpp.h")

# ROS 依赖包
find_package(catkin REQUIRED COMPONENTS # 查找 catkin（ROS1 的编译系统）和所需的 ROS 包
  geometry_msgs      # 几何消息：Point, Pose, Quaternion, Vector3 等
  nav_msgs           # 导航消息：Odometry, Path 等
  sensor_msgs        # 传感器消息：PointCloud2, Imu 等
  roscpp             # ROS C++ 客户端库
  rospy              # ROS Python 客户端库（此项目未实际使用）
  std_msgs           # 基础消息：Header, String, Int32 等
  pcl_ros            # PCL 与 ROS 的桥接
  tf                 # 坐标变换库
  livox_ros_driver   # Livox 雷达驱动（提供 CustomMsg 消息定义）
  message_generation # 消息生成工具（用于编译自定义消息）
  eigen_conversions  # Eigen 与 ROS 消息的转换
)

# 第三方库依赖
find_package(Eigen3 REQUIRED)  # Eigen3：线性代数库（矩阵运算、四元数等）
find_package(PCL 1.8 REQUIRED)  # PCL：点云处理库（1.8以上版本）

message(Eigen: ${EIGEN3_INCLUDE_DIR})

# 头文件搜索路径
include_directories(
  ${catkin_INCLUDE_DIRS}   # ROS 包的头文件（sensor_msgs, nav_msgs 等）
  ${EIGEN3_INCLUDE_DIR}    # Eigen 头文件
  ${PCL_INCLUDE_DIRS}      # PCL 头文件
  ${PYTHON_INCLUDE_DIRS}   # Python 头文件（用于绘图）
  include                  # 本项目的 include 目录（ikd-Tree, IKFoM 等）
)

# 自定义消息
add_message_files(   # 声明要编译的自定义消息文件
  FILES
  Pose6D.msg  # 在 msg/ 目录下，定义了 IMU 预积分状态
)

# 生成消息代码（C++ 头文件和 Python 模块）
# DEPENDENCIES：如果 Pose6D.msg 用到了其他包的消息类型，在这里声明
generate_messages(
  DEPENDENCIES
  geometry_msgs  # 当前 Pose6D.msg 用的都是基础类型，这里是预留
)

# catkin 包声明：声明本包的依赖关系，供其他包使用时参考
catkin_package(
  # 运行时依赖的 ROS 包
  CATKIN_DEPENDS geometry_msgs nav_msgs roscpp rospy std_msgs message_runtime
  # 运行时依赖的系统库
  DEPENDS EIGEN3 PCL
  # 对外暴露的头文件目录（此处为空）
  INCLUDE_DIRS
)

# 编译目标
add_executable(fastlio_mapping    # 创建可执行文件
  src/laserMapping.cpp            # 主程序
  include/ikd-Tree/ikd_Tree.cpp   # ikd-tree 实现
  src/preprocess.cpp              # 点云预处理
)

target_link_libraries(fastlio_mapping   # 链接库
  ${catkin_LIBRARIES}   # ROS 库（roscpp, tf, pcl_ros 等）
  ${PCL_LIBRARIES}      # PCL 库
  ${PYTHON_LIBRARIES}   # Python 库（用于 matplotlib）
)

# Python 头文件路径（用于 matplotlibcpp）
target_include_directories(fastlio_mapping PRIVATE ${PYTHON_INCLUDE_DIRS})