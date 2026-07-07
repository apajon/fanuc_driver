# SPDX-FileCopyrightText: 2025, FANUC America Corporation
# SPDX-FileCopyrightText: 2025, FANUC CORPORATION
#
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.22)
project(stream_motion)

option(BUILD_EXAMPLES "Build examples" OFF)
option(FANUC_SM_TIMING_DEBUG "Enable Stream Motion timing CSV instrumentation" OFF)

add_library(stream_motion SHARED src/stream.cpp src/byte_ops.cpp)
set_target_properties(stream_motion PROPERTIES CXX_STANDARD 17)
target_link_libraries(stream_motion PRIVATE Sockpp::sockpp)
if (FANUC_SM_TIMING_DEBUG)
  # PUBLIC so consumers that construct StreamMotionConnection (e.g. fanuc_client) share
  # the same class layout; otherwise the guarded member buffer would cause an ODR mismatch.
  target_compile_definitions(stream_motion PUBLIC FANUC_SM_TIMING_DEBUG)
endif ()
target_include_directories(stream_motion PUBLIC
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

if (BUILD_TESTING)
  add_subdirectory(test)
endif ()
