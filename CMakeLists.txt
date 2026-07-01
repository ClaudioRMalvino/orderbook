cmake_minimum_required(VERSION 3.20)
project(orderbook VERSION 0.1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)


add_library(orderbook INTERFACE)
target_include_directories(orderbook INTERFACE include)

option(ORDERBOOK_BUILD_TESTS "Build tests" ON)
option(ORDERBOOK_BUILD_BENCHMARKS "Build benchmarks" OFF)

include(FetchContent)

if(ORDERBOOK_BUILD_TESTS)
  FetchContent_Declare(
    catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG        v3.15.1
  )
  FetchContent_MakeAvailable(catch2)
  enable_testing()
  add_subdirectory(tests)
endif()

if(ORDERBOOK_BUILD_BENCHMARKS)
  set(BENCHMARK_ENABLE_TESTING OFF)
  FetchContent_Declare(
    googlebenchmark
    GIT_REPOSITORY https://github.com/google/benchmark.git
    GIT_TAG        v1.9.5
  )
  FetchContent_MakeAvailable(googlebenchmark)
  add_subdirectory(benchmarks)
endif()
