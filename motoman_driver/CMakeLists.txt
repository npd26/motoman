cmake_minimum_required(VERSION 2.8.3)
project(motoman_driver)

# Load catkin and all dependencies required for this package
find_package(
  catkin
  REQUIRED COMPONENTS
    actionlib
    actionlib_msgs
    control_msgs
    industrial_msgs
    industrial_robot_client
    industrial_utils
    motoman_msgs
    roscpp
    sensor_msgs
    simple_message
    std_msgs
    std_srvs
    trajectory_msgs
    urdf
)

find_package(Boost REQUIRED COMPONENTS system thread)

#######################################
## Adding directories and definitions #
#######################################
include_directories(
  include
  ${Boost_INCLUDE_DIRS}
  ${catkin_INCLUDE_DIRS})

add_definitions(-DLINUXSOCKETS=1)  #use linux sockets for communication
add_definitions(-DROS=1)

catkin_package(
  CATKIN_DEPENDS
    actionlib
    actionlib_msgs
    control_msgs
    industrial_msgs
    industrial_robot_client
    industrial_utils
    motoman_msgs
    roscpp
    sensor_msgs
    simple_message
    std_msgs
    std_srvs
    trajectory_msgs
    urdf
  INCLUDE_DIRS
    include
)


set(MSG_SRC_FILES
  src/simple_message/messages/motoman_read_single_io_message.cpp
  src/simple_message/messages/motoman_read_single_io_reply_message.cpp
  src/simple_message/messages/motoman_write_single_io_message.cpp
  src/simple_message/messages/motoman_write_single_io_reply_message.cpp
  src/simple_message/motoman_motion_ctrl.cpp
  src/simple_message/motoman_motion_ctrl_message.cpp
  src/simple_message/motoman_motion_reply.cpp
  src/simple_message/motoman_motion_reply_message.cpp
  src/simple_message/motoman_read_single_io.cpp
  src/simple_message/motoman_read_single_io_reply.cpp
  src/simple_message/motoman_write_single_io.cpp
  src/simple_message/motoman_write_single_io_reply.cpp
)

set(CLIENT_SRC_FILES
  src/industrial_robot_client/joint_feedback_ex_relay_handler.cpp
  src/industrial_robot_client/joint_feedback_relay_handler.cpp
  src/industrial_robot_client/joint_relay_handler.cpp
  src/industrial_robot_client/joint_trajectory_interface.cpp
  src/industrial_robot_client/joint_trajectory_streamer.cpp
  src/industrial_robot_client/motoman_utils.cpp
  src/industrial_robot_client/robot_state_interface.cpp
  src/simple_message/joint_feedback_ex.cpp
  src/simple_message/joint_traj_pt_full_ex.cpp
  src/simple_message/messages/joint_feedback_ex_message.cpp
  src/simple_message/messages/joint_traj_pt_full_ex_message.cpp
)


###########
## Build ##
###########

#-------------------------------------------------------------
# dx100 uses same byte-ordering as most i386-based PCs

# Simple message library
add_library(motoman_simple_message ${MSG_SRC_FILES})
target_link_libraries(motoman_simple_message
  simple_message)

# Industrial client library
add_library(motoman_industrial_robot_client ${CLIENT_SRC_FILES})
target_link_libraries(motoman_industrial_robot_client
  industrial_robot_client
  industrial_utils)
add_dependencies(motoman_industrial_robot_client ${catkin_EXPORTED_TARGETS})

# Robot state node
add_executable(motoman_robot_state
  src/robot_state_node.cpp)
target_link_libraries(motoman_robot_state
  motoman_simple_message
  motoman_industrial_robot_client
  ${catkin_LIBRARIES})
set_target_properties(motoman_robot_state
  PROPERTIES OUTPUT_NAME robot_state
  PREFIX "")

# Motion streaming node
add_executable(motoman_motion_streaming_interface
  src/joint_streaming_node.cpp
  src/joint_trajectory_streamer.cpp
  src/motion_ctrl.cpp)
target_link_libraries(motoman_motion_streaming_interface
  motoman_simple_message
  motoman_industrial_robot_client
  ${catkin_LIBRARIES})
set_target_properties(motoman_motion_streaming_interface
  PROPERTIES OUTPUT_NAME motion_streaming_interface
  PREFIX "")


#----------------------------------------------------------------
# FS100 uses opposite byte-ordering from most i386-based PCs
# Simple message library
add_library(motoman_simple_message_bswap ${MSG_SRC_FILES})
set_target_properties(motoman_simple_message_bswap PROPERTIES COMPILE_DEFINITIONS "BYTE_SWAPPING")
target_link_libraries(motoman_simple_message_bswap
  simple_message_bswap)

# Industrial client library
add_library(motoman_industrial_robot_client_bswap ${CLIENT_SRC_FILES})
set_target_properties(motoman_industrial_robot_client_bswap PROPERTIES COMPILE_DEFINITIONS "BYTE_SWAPPING")
target_link_libraries(motoman_industrial_robot_client_bswap
  industrial_robot_client_bswap
  industrial_utils)
add_dependencies(motoman_industrial_robot_client_bswap ${catkin_EXPORTED_TARGETS})

# Robot state node
add_executable(motoman_robot_state_bswap
  src/robot_state_node.cpp)
target_link_libraries(motoman_robot_state_bswap
  motoman_simple_message_bswap
  motoman_industrial_robot_client_bswap
  ${catkin_LIBRARIES})
set_target_properties(motoman_robot_state_bswap
  PROPERTIES OUTPUT_NAME robot_state_bswap
  PREFIX "")

# Motion streaming node
add_executable(motoman_motion_streaming_interface_bswap
  src/joint_streaming_node.cpp
  src/joint_trajectory_streamer.cpp
  src/motion_ctrl.cpp)
target_link_libraries(motoman_motion_streaming_interface_bswap
  motoman_simple_message_bswap
  motoman_industrial_robot_client_bswap
  ${catkin_LIBRARIES})
set_target_properties(motoman_motion_streaming_interface_bswap
  PROPERTIES OUTPUT_NAME motion_streaming_interface_bswap
  PREFIX "")

add_executable(${PROJECT_NAME}_joint_trajectory_action
  src/industrial_robot_client/joint_trajectory_action.cpp
  src/joint_trajectory_action_node.cpp)
target_link_libraries(${PROJECT_NAME}_joint_trajectory_action
  industrial_robot_client
  motoman_industrial_robot_client
  ${catkin_LIBRARIES})


#############
## Install ##
#############

# binaries
install(TARGETS
  ${PROJECT_NAME}_joint_trajectory_action
  motoman_motion_streaming_interface
  motoman_motion_streaming_interface_bswap
  motoman_robot_state
  motoman_robot_state_bswap

  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

# libraries
install(TARGETS
  motoman_industrial_robot_client
  motoman_industrial_robot_client_bswap
  motoman_simple_message
  motoman_simple_message_bswap

  DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
)

# headers
install(DIRECTORY include/${PROJECT_NAME}/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
)

# other files
install(DIRECTORY
    Inform
    launch
    MotoPlus
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION})

install(PROGRAMS src/move_to_joint.py
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION})


#############
## Testing ##
#############

if(CATKIN_ENABLE_TESTING)
  find_package(roslint REQUIRED)
  roslint_cpp()
endif()
