cmake_minimum_required(VERSION 4.0)
project(asio_reproduction)

set(CMAKE_CXX_STANDARD 23)

find_package(Boost)
add_compile_options(-fsanitize=thread)
add_link_options(-fsanitize=thread)
add_executable(asio_reproduction main.cpp)
target_link_libraries(asio_reproduction Boost::headers)