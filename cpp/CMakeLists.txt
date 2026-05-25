# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

cmake_minimum_required(VERSION 3.16)
project(mosaic-cpp-test CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Arrow REQUIRED)

set(MOSAIC_FFI_LIB "" CACHE FILEPATH "Path to libpaimon_mosaic_ffi shared library")
if(NOT MOSAIC_FFI_LIB)
    if(APPLE)
        set(MOSAIC_FFI_LIB "${CMAKE_SOURCE_DIR}/../target/release/libpaimon_mosaic_ffi.dylib")
    elseif(WIN32)
        set(MOSAIC_FFI_LIB "${CMAKE_SOURCE_DIR}/../target/release/paimon_mosaic_ffi.dll")
    else()
        set(MOSAIC_FFI_LIB "${CMAKE_SOURCE_DIR}/../target/release/libpaimon_mosaic_ffi.so")
    endif()
endif()

include(FetchContent)
FetchContent_Declare(
    googletest
    URL https://github.com/google/googletest/archive/refs/tags/v1.15.2.tar.gz
    URL_HASH SHA256=7b42b4d6ed48810c5362c265a17faebe90dc2373c885e5216439d37927f02926
)
set(BUILD_GMOCK OFF CACHE BOOL "" FORCE)
set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

add_executable(test_mosaic test_mosaic.cpp)
target_include_directories(test_mosaic PRIVATE ${CMAKE_SOURCE_DIR}/../include)
target_link_libraries(test_mosaic Arrow::arrow_shared ${MOSAIC_FFI_LIB} GTest::gtest_main)

option(MOSAIC_ENABLE_SANITIZERS "Build with AddressSanitizer + UndefinedBehaviorSanitizer" OFF)
if(MOSAIC_ENABLE_SANITIZERS)
    target_compile_options(test_mosaic PRIVATE
        -fsanitize=address,undefined -fno-omit-frame-pointer -O1 -g)
    target_link_options(test_mosaic PRIVATE
        -fsanitize=address,undefined)
endif()

enable_testing()
include(GoogleTest)
get_filename_component(MOSAIC_FFI_DIR "${MOSAIC_FFI_LIB}" DIRECTORY)
if(APPLE)
    set(MOSAIC_TEST_LIB_ENV "DYLD_FALLBACK_LIBRARY_PATH=${MOSAIC_FFI_DIR}")
else()
    set(MOSAIC_TEST_LIB_ENV "LD_LIBRARY_PATH=${MOSAIC_FFI_DIR}")
endif()
gtest_discover_tests(test_mosaic
    DISCOVERY_MODE PRE_TEST
    PROPERTIES ENVIRONMENT "${MOSAIC_TEST_LIB_ENV}")
