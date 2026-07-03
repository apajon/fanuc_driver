# SPDX-FileCopyrightText: 2025, FANUC America Corporation
# SPDX-FileCopyrightText: 2025, FANUC CORPORATION
#
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.22)
project(rmi)

add_library(rmi SHARED src/rmi.cpp src/serialization.cpp src/deserialization.cpp)
target_link_libraries(rmi PRIVATE reflectcpp Sockpp::sockpp)
# Localize symbols pulled in from the statically-linked reflectcpp / yaml-cpp
# archives so they are NOT exported into rmi.so's dynamic symbol table.
# Otherwise this .so exports yaml-cpp 0.9.0 symbols that collide (ODR) with
# the system yaml_cpp_vendor (0.8.x) used by EtherCAT ros2_control plugins
# (ethercat_driver_ros2 / EthercatDriver), causing a SIGSEGV in
# EthercatDriver::setupSlave (YAML::LoadFile) when both .so files are
# co-loaded in the same ros2_control process.
target_link_options(rmi PRIVATE -Wl,--exclude-libs,ALL)
target_include_directories(rmi PUBLIC
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
set_target_properties(rmi PROPERTIES CXX_STANDARD 20)

if (BUILD_TESTING)
  add_subdirectory(test)
endif ()
