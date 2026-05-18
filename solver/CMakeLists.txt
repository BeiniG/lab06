cmake_minimum_required(VERSION 3.10)
project(solver)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_subdirectory(solver_lib)

add_executable(solver equation.cpp)
target_link_libraries(solver PRIVATE solver_lib formatter formatter_ex)
