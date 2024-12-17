cmake_minimum_required(VERSION 3.15)
project(MyProject LANGUAGES CXX)

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_COMPILER "clang++")

set(CMAKE_PREFIX_PATH "D:/tools/CPP/vcpkg/installed/x64-windows")
# 设置构建目录
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# 平台和编译器配置
include(cmake/compiler_setup.cmake)

# 源代码
file(GLOB_RECURSE SOURCE_FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")
add_executable(scram_ext ${SOURCE_FILES})
# 链接 libxml2
find_package(LibXml2 REQUIRED)
include_directories(scram_ext ${LIBXML2_INCLUDE_DIR})
target_link_libraries(scram_ext ${LIBXML2_LIBRARIES})
message(STATUS "LIBXML2_INCLUDE_DIR: ${LIBXML2_INCLUDE_DIR}")
message(STATUS "LIBXML2_LIBRARIES: ${LIBXML2_LIBRARIES}")

