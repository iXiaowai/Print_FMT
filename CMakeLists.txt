cmake_minimum_required(VERSION 3.15)

project(Print_FMT LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_library(fmt STATIC
    src/format.cc
    src/os.cc
    src/fmt-c.cc
)

target_include_directories(fmt
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
)

option(FMT_BUILD_TEST "Build test cases" OFF)

if(FMT_BUILD_TEST)
    enable_testing()
    add_subdirectory(cases)
endif()
