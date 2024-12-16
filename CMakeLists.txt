cmake_minimum_required(VERSION 3.15)
project(MyProject LANGUAGES CXX)

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if (NOT DEFINED CMAKE_TOOLCHAIN_FILE)
    set(CMAKE_TOOLCHAIN_FILE "D:/tools/CPP/vcpkg/scripts/buildsystems/vcpkg.cmake")
endif()
# 设置构建目录
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

set(LIBXML2_INCLUDE_DIR "D:/tools/CPP/vcpkg/installed/x64-windows/include/libxml2/libxml/")
set(LIBXML2_LIBRARY "D:/tools/CPP/vcpkg/installed/x64-windows/lib/libxml2.lib")
# 平台和编译器配置
include(cmake/compiler_setup.cmake)

# 源代码
add_executable(my_program src/main.cpp)
# 链接 libxml2
find_package(LibXml2 REQUIRED)
target_include_directories(my_program PRIVATE ${LIBXML2_INCLUDE_DIR})
target_link_libraries(my_program PRIVATE ${LIBXML2_LIBRARY})

