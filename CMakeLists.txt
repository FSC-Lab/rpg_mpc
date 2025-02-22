# rpg_quadrotor_mpc A model predictive control implementation for quadrotors.
# Copyright (C) 2017-2018 Philipp Foehn, Robotics and Perception Group,
# University of Zurich
#
# Intended to be used with rpg_quadrotor_control and rpg_quadrotor_common.
# https://github.com/uzh-rpg/rpg_quadrotor_control
#
# This program is free software: you can redistribute it and/or modify it under
# the terms of the GNU General Public License as published by the Free Software
# Foundation, either version 3 of the License, or (at your option) any later
# version.
#
# This program is distributed in the hope that it will be useful, but WITHOUT
# ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
# FOR A PARTICULAR PURPOSE.  See the GNU General Public License for more
# details.
#
# You should have received a copy of the GNU General Public License along with
# this program.  If not, see <http://www.gnu.org/licenses/>.

cmake_minimum_required(VERSION 3.8.3)
project(rpg_mpc)

find_package(
  catkin REQUIRED COMPONENTS autopilot geometry_msgs quadrotor_common roscpp
                             std_msgs trajectory_msgs)

find_package(Eigen3 REQUIRED)
# activate c++ 11
if(CMAKE_COMPILER_IS_GNUCC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif()

# ARM NEON flags
if("${CMAKE_HOST_SYSTEM_PROCESSOR}" STREQUAL "armv7l")
  set(CMAKE_C_FLAGS
      "${CMAKE_C_FLAGS} -march=armv7-a -mfpu=neon -mfloat-abi=hard -funsafe-math-optimizations"
  )
  set(CMAKE_CXX_FLAGS
      "${CMAKE_CXX_FLAGS} -march=armv7-a -mfpu=neon -mfloat-abi=hard -funsafe-math-optimizations"
  )
  message("enabling ARM neon optimizations")
endif()

# flags for speed (should already be enabled by default)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fopenmp -O3")

catkin_package(
  INCLUDE_DIRS
  LIBRARIES
  CATKIN_DEPENDS
  autopilot
  geometry_msgs
  quadrotor_common
  roscpp
  std_msgs
  trajectory_msgs
  DEPENDS)

add_library(
  mpc_solver
  externals/qpoases/SRC/Bounds.cpp
  externals/qpoases/SRC/Constraints.cpp
  externals/qpoases/SRC/CyclingManager.cpp
  externals/qpoases/SRC/Indexlist.cpp
  externals/qpoases/SRC/MessageHandling.cpp
  externals/qpoases/SRC/QProblem.cpp
  externals/qpoases/SRC/QProblemB.cpp
  externals/qpoases/SRC/SubjectTo.cpp
  externals/qpoases/SRC/Utils.cpp
  externals/qpoases/SRC/EXTRAS/SolutionAnalysis.cpp
  model/quadrotor_mpc_codegen/acado_qpoases_interface.cpp
  model/quadrotor_mpc_codegen/acado_integrator.c
  model/quadrotor_mpc_codegen/acado_solver.c
  model/quadrotor_mpc_codegen/acado_auxiliary_functions.c)

target_include_directories(
  mpc_solver PUBLIC model/quadrotor_mpc_codegen/ externals/qpoases
                    externals/qpoases/INCLUDE externals/qpoases/SRC)

add_library(mpc_wrapper src/mpc_wrapper.cpp)
target_include_directories(
  mpc_wrapper PUBLIC include model/quadrotor_mpc_codegen ${catkin_INCLUDE_DIRS})
target_link_libraries(mpc_wrapper PRIVATE mpc_solver ${catkin_LIBRARIES}
                                          Eigen3::Eigen)

add_library(mpc_controller src/mpc_controller.cpp)
target_include_directories(mpc_controller PUBLIC include ${catkin_INCLUDE_DIRS})
target_link_libraries(mpc_controller PUBLIC mpc_wrapper ${catkin_LIBRARIES}
                                            Eigen3::Eigen)

# make an executable
add_executable(mpc_controller_node test/mpc_node.cpp)
target_include_directories(mpc_controller
                           PRIVATE include ${catkin_INCLUDE_DIRS} Eigen3::Eigen)
target_link_libraries(mpc_controller_node mpc_controller mpc_wrapper mpc_solver
                      ${catkin_LIBRARIES})

if(catkin_LIBRARIES MATCHES "autopilot")

  message("Building with MPC Autopilot")
  add_executable(autopilot_mpc_instance src/autopilot_mpc_instance.cpp)
  target_link_libraries(autopilot_mpc_instance mpc_controller mpc_wrapper
                        mpc_solver)

endif(catkin_LIBRARIES MATCHES "autopilot")
