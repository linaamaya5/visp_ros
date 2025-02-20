cmake_minimum_required(VERSION 3.5)
project(visp_ros)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra)
endif()

find_package(ament_cmake_auto REQUIRED)
ament_auto_find_build_dependencies()

find_package(rclcpp REQUIRED)
find_package(cv_bridge REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(image_geometry REQUIRED)
find_package(image_transport REQUIRED)
find_package(kdl_parser REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(std_msgs REQUIRED)
find_package(std_srvs REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(visp_bridge REQUIRED)
find_package(VISP REQUIRED COMPONENTS visp_core visp_robot)

# Specific packages for franka simulator
find_package(orocos_kdl QUIET)
find_package(iir QUIET)

# messages and services interface generation
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/BlobTracker.msg"
  "msg/ImagePoint.msg"
  "msg/PoseStampedStatus.msg"
  "msg/ProjectedPoint.msg"
  DEPENDENCIES sensor_msgs geometry_msgs
)

set(orocos_kdl_VERSION_MINIMUM "1.3.2")

if(orocos_kdl_FOUND AND(${orocos_kdl_VERSION} VERSION_GREATER ${orocos_kdl_VERSION_MINIMUM}))
  add_definitions("-DVISP_HAVE_OROCOS_KDL")
  message("-- orocos kdl ${orocos_kdl_VERSION} found: We can build franka simulator")

  if(iir_FOUND)
    message("-- iir found")
    add_definitions("-DVISP_HAVE_IIR")
  endif()
elseif(orocos_kdl_FOUND AND((${orocos_kdl_VERSION} VERSION_EQUAL ${orocos_kdl_VERSION_MINIMUM})
  OR(${orocos_kdl_VERSION} VERSION_LESS ${orocos_kdl_VERSION_MINIMUM})))
  message("-- orocos kdl ${orocos_kdl_VERSION} found: but version to old to build franka simulator")
else()
  message("-- orocos kdl not found: Unable to build franka simulator")
endif()

set(srcs
  src/device/framegrabber/vpROSGrabber.cpp
  # src/robot/real-robot/pioneer/vpROSRobotPioneer.cpp
  src/robot/vpROSRobot.cpp
)

if(orocos_kdl_FOUND AND(${orocos_kdl_VERSION} VERSION_GREATER ${orocos_kdl_VERSION_MINIMUM}))
  # Add Franka simulator
  list(APPEND srcs
    src/robot/sim-robot/franka/vpRobotFrankaSim.cpp
    src/robot/sim-robot/franka/vpROSRobotFrankaCoppeliasim.cpp
    src/robot/sim-robot/franka/model/CoriolisMatrix.cpp
    src/robot/sim-robot/franka/model/franka_model.h
    src/robot/sim-robot/franka/model/FrictionTorque.cpp
    src/robot/sim-robot/franka/model/GravityVector.cpp
    src/robot/sim-robot/franka/model/MassMatrix.cpp
  )
endif()

# Declare library
add_library(${PROJECT_NAME}_common ${srcs})

ament_export_dependencies(VISP)
ament_target_dependencies(${PROJECT_NAME}_common
  cv_bridge
  geometry_msgs
  image_geometry
  image_transport
  nav_msgs
  rclcpp
  std_srvs
  visp_bridge
  VISP
  orocos_kdl
)

rosidl_get_typesupport_target(cpp_typesupport_target "${PROJECT_NAME}" "rosidl_typesupport_cpp")

target_include_directories(${PROJECT_NAME}_common PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
  "$<INSTALL_INTERFACE:include/${PROJECT_NAME}>"
  ${VISP_INCLUDE_DIRS}
)

# Node to control Afma6 robot
add_executable(${PROJECT_NAME}_afma6_node nodes/afma6.cpp)
target_link_libraries(${PROJECT_NAME}_afma6_node
  "${cpp_typesupport_target}"
)
ament_target_dependencies(${PROJECT_NAME}_afma6_node
  geometry_msgs
  rclcpp
  visp_bridge
  VISP
)

# Node to track a square made of 4 blobs
add_executable(${PROJECT_NAME}_blob_tracker_node nodes/blob_tracker.cpp)
target_link_libraries(${PROJECT_NAME}_blob_tracker_node

  # ${PROJECT_NAME}_common
  "${cpp_typesupport_target}"

  # ${VISP_LIBRARIES}
)
ament_target_dependencies(${PROJECT_NAME}_blob_tracker_node
  cv_bridge
  rclcpp
  std_msgs
  visp_bridge
  VISP
)

# Node to control Viper650 robot
add_executable(${PROJECT_NAME}_viper650_node nodes/viper650.cpp)
target_link_libraries(${PROJECT_NAME}_viper650_node
  "${cpp_typesupport_target}"
)
ament_target_dependencies(${PROJECT_NAME}_viper650_node
  geometry_msgs
  rclcpp
  sensor_msgs
  visp_bridge
  VISP
)

# Node to control Viper850 robot
add_executable(${PROJECT_NAME}_viper850_node nodes/viper850.cpp)
target_link_libraries(${PROJECT_NAME}_viper850_node
  "${cpp_typesupport_target}"
)
ament_target_dependencies(${PROJECT_NAME}_viper850_node
  geometry_msgs
  rclcpp
  sensor_msgs
  visp_bridge
  VISP
)

# Tutorial for ROS Grabber
add_executable(tutorial-ros-grabber tutorial/grabber/ros/tutorial-ros-grabber.cpp)
target_link_libraries(tutorial-ros-grabber
  ${PROJECT_NAME}_common
  "${cpp_typesupport_target}"
)
ament_target_dependencies(tutorial-ros-grabber
  rclcpp
  sensor_msgs
  visp_bridge
  VISP
)

# ###########################################################
# Build tutorial to simulate franka robot with coppeliasim
# ###########################################################
if(orocos_kdl_FOUND AND(${orocos_kdl_VERSION} VERSION_GREATER ${orocos_kdl_VERSION_MINIMUM}))
  set(tutorial_franka
    tutorial/franka/coppeliasim/test-franka-coppeliasim-controller.cpp
    tutorial/franka/coppeliasim/tutorial-franka-coppeliasim-ibvs-apriltag.cpp
    tutorial/franka/coppeliasim/tutorial-franka-coppeliasim-pbvs-apriltag.cpp
    tutorial/franka/coppeliasim/tutorial-franka-coppeliasim-cartesian-impedance-control.cpp
    tutorial/franka/coppeliasim/tutorial-franka-coppeliasim-joint-impedance-control.cpp
    tutorial/franka/coppeliasim/tutorial-franka-coppeliasim-dual-arm.cpp
  )
  foreach(cpp ${tutorial_franka})
    get_filename_component(bin_name ${cpp} NAME_WE)
    add_executable(${bin_name} ${cpp})
    target_link_libraries(${bin_name}
      ${PROJECT_NAME}_common
      "${cpp_typesupport_target}"
    )
    ament_target_dependencies(${bin_name}
      geometry_msgs
      rclcpp
      sensor_msgs
      visp_bridge
      VISP
    )
    # Installation
    install(
      TARGETS ${bin_name}
      DESTINATION lib/${PROJECT_NAME}
    )
  endforeach()
endif()

ament_export_targets(export_${PROJECT_NAME} HAS_LIBRARY_TARGET)
ament_export_dependencies(
    cv_bridge
    geometry_msgs
    image_geometry
    image_transport
    kdl_parser
    nav_msgs
    sensor_msgs
    std_msgs
    std_srvs
    tf2_ros
    visp_bridge
)

install(
  DIRECTORY include/
  DESTINATION include
)

install(
  TARGETS ${PROJECT_NAME}_common
  EXPORT export_${PROJECT_NAME}
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  RUNTIME DESTINATION bin
  INCLUDES DESTINATION include
)

# Installation
install(
  TARGETS
  ${PROJECT_NAME}_afma6_node
  ${PROJECT_NAME}_blob_tracker_node
  ${PROJECT_NAME}_viper650_node
  ${PROJECT_NAME}_viper850_node

  tutorial-ros-grabber

  DESTINATION lib/${PROJECT_NAME}

  # ARCHIVE DESTINATION lib
  # LIBRARY DESTINATION lib
  # RUNTIME DESTINATION bin
)

install(DIRECTORY
  launch
  config
  DESTINATION share/${PROJECT_NAME}
)
ament_export_dependencies(rosidl_default_runtime)

ament_package()
