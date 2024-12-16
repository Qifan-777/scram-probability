cmake_minimum_required(VERSION 3.15)
project(MyProject LANGUAGES CXX)

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 设置构建目录
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# 平台和编译器配置
include(cmake/compiler_setup.cmake)

# 源代码
add_executable(my_program src/main.cpp)