cmake_minimum_required(VERSION 3.10) #指定最小版本
project(distance_calculator)         #设置项目名称

# 指定类型，常见有Debug Release等
if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "Setting build type to 'Release' as none was specified.")
    set(CMAKE_BUILD_TYPE "Release" CACHE
        STRING "Choose the type of build." FORCE)
endif()
set(CMAKE_CXX_FLAGS "-Wall -Wextra")             # 编译标志 -Wall 开启所有警告  -Wextra 启用一些未由-Wall启用的额外警告标志
set(CMAKE_CXX_FLAGS_DEBUG "-g")                  # 包含标准测试包
set(CMAKE_CXX_FLAGS_RELEASE "-O2 -march=native") # 02优化

# 找到Boost包
find_package(Boost
    REQUIRED
    COMPONENTS program_options graph regex
    )

set(CMAKE_CXX_STANDARD 14)

add_executable(distance_calculator main.cpp) #生成可执行包  具体源码分析看mian.cpp
target_link_libraries(distance_calculator ${Boost_LIBRARIES}) # 设置需要链接的库
