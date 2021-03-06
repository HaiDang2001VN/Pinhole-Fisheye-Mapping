# This file is part of REMODE - REgularized MOnocular Depth Estimation.
#
# Copyright (C) 2014 Matia Pizzoli <matia dot pizzoli at gmail dot com>
# Robotics and Perception Group, University of Zurich, Switzerland
# http://rpg.ifi.uzh.ch
#
# REMODE is free software: you can redistribute it and/or modify it under the
# terms of the GNU General Public License as published by the Free Software
# Foundation, either version 3 of the License, or any later version.
#
# REMODE is distributed in the hope that it will be useful, but WITHOUT ANY
# WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
# FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.


SET(PROJECT_NAME pinhole_mapping)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8.3)
PROJECT(${PROJECT_NAME})
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/CMakeModules/")
SET(CMAKE_BUILD_TYPE Release) # Release, RelWithDebInfo

find_package(CUDA REQUIRED)
SET(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS};-O3 -use_fast_math)

# Specify compute capability
list(APPEND CUDA_NVCC_FLAGS -arch=sm_50)
list(APPEND CUDA_NVCC_FLAGS -gencode arch=compute_50,code=sm_50)

SET(CUDA_PROPAGATE_HOST_FLAGS OFF)

SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")


option(BUILD_ROS_NODE "Build ROS node" ON)
if(BUILD_ROS_NODE)
  ADD_DEFINITIONS(-DBUILD_ROS_NODE)
endif(BUILD_ROS_NODE)

option(BUILD_TESTS "Build test programs" ON)
if(BUILD_TESTS)
  ADD_DEFINITIONS(-DRMD_BUILD_TESTS)
endif(BUILD_TESTS)

# NCC patch size (must be odd integer, typical values: 3, 5, 7)
ADD_DEFINITIONS(-DRMD_CORR_PATCH_SIDE=5)
# Maximum extent of epipolar line search in pixels
ADD_DEFINITIONS(-DRMD_MAX_EXTENT_EPIPOLAR_SEARCH=100)

find_package(OpenCV REQUIRED)
find_package(Eigen REQUIRED)
find_package(Boost REQUIRED COMPONENTS system filesystem)


if(BUILD_ROS_NODE)
  find_package(catkin REQUIRED COMPONENTS
    roscpp roslib cmake_modules cv_bridge image_transport pcl_ros sensor_msgs geometry_msgs)
  catkin_package(
      DEPENDS OpenCV Eigen Boost
      CATKIN_DEPENDS roscpp roslib image_transport pcl_ros
      INCLUDE_DIRS include
      LIBRARIES pinhole_mapping pinhole_mapping_cuda
  )
else()
  set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
  set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
endif(BUILD_ROS_NODE)

include_directories(
  include
  ${OpenCV_INCLUDE_DIRS}
  ${Eigen_INCLUDE_DIRS}
  ${Boost_INCLUDE_DIRS}
)

# Set link libraries
list(APPEND LINK_LIBS
  ${OpenCV_LIBS}
  ${Boost_LIBRARIES}
)

if(BUILD_ROS_NODE)
  include_directories(${catkin_INCLUDE_DIRS})
  list(APPEND LINK_LIBS ${catkin_LIBRARIES})
endif(BUILD_ROS_NODE)

# Set CUDA sourcefiles
list(APPEND CU_SOURCE_FILES
    src/check_cuda_device.cu
    src/seed_matrix.cu
)

# Set sourcefiles
list(APPEND SOURCE_FILES
    src/depthmap.cpp
)

CUDA_ADD_LIBRARY(
    pinhole_mapping_cuda
    ${CU_SOURCE_FILES}
)

target_link_libraries(
    pinhole_mapping_cuda
    ${LINK_LIBS}
)

ADD_LIBRARY(
    pinhole_mapping
    ${SOURCE_FILES}
)

target_link_libraries(
    pinhole_mapping
    ${LINK_LIBS}
)

# Build the ROS node
add_executable(
  pinhole_mapping_node
  src/depthmap_node.cpp
  src/publisher.cpp
  src/main_ros.cpp
)

target_link_libraries(
  pinhole_mapping_node
  pinhole_mapping
  pinhole_mapping_cuda
  ${OpenCV_LIBS}
)
