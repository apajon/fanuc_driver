# SPDX-FileCopyrightText: 2025, FANUC America Corporation
# SPDX-FileCopyrightText: 2025, FANUC CORPORATION
#
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.22)
project(gpio_config)

add_library(gpio_config SHARED src/gpio_config.cpp )
target_link_libraries(gpio_config PRIVATE reflectcpp)
# Localize symbols pulled in from the statically-linked reflectcpp / yaml-cpp
# archives so they are NOT exported into libgpio_config.so's dynamic symbol table.
# Otherwise this .so exports yaml-cpp 0.9.0 symbols that collide (ODR) with
# the system yaml_cpp_vendor (0.8.x) used by EtherCAT ros2_control plugins
# (ethercat_driver_ros2 / EthercatDriver), causing a SIGSEGV in
# EthercatDriver::setupSlave (YAML::LoadFile) when both .so files are
# co-loaded in the same ros2_control process.
target_link_options(gpio_config PRIVATE -Wl,--exclude-libs,ALL)
# --exclude-libs only hides ARCHIVE symbols; gpio_config.cpp also emits
# header-only yaml-cpp template instantiations (weak YAML:: symbols) into its
# own object file. A version script localizes those too, so libgpio_config.so
# stops exporting any yaml-cpp 0.9.0 symbol that could interpose the system
# yaml_cpp_vendor (0.8.x) used by EtherCAT ros2_control plugins at runtime.
target_link_options(gpio_config PRIVATE
    "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/yaml_localize.map")
target_include_directories(gpio_config PUBLIC
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
set_target_properties(gpio_config PROPERTIES CXX_STANDARD 20)

if (BUILD_TESTING)
  add_subdirectory(test)
endif ()
