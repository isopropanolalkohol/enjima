cmake_minimum_required(VERSION 3.20)
project(enjima LANGUAGES CXX VERSION 0.1.0)


set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
option(BUILD_SHARED_LIBS "Build shared library" OFF)
option(CP_ENABLE_ASAN "Enable Address/UB Sanitizers" ON)


add_library(enjima
        src/metrics/entropy.cpp
        src/metrics/chi2.cpp
        src/metrics/ioc.cpp
        src/metrics/histogram.cpp
        src/detect/probe.cpp
)

target_include_directories(enjima PUBLIC ${PROJECT_SOURCE_DIR}/include)
target_compile_features(enjima PUBLIC cxx_std_20)

# Warnings & Sanitizers
if(MSVC)
    target_compile_options(enjima PRIVATE /W4 /permissive-)
else()
    target_compile_options(enjima PRIVATE -Wall -Wextra -Wpedantic -fvisibility=hidden)
    if(CP_ENABLE_ASAN)
        target_compile_options(enjima PRIVATE -fsanitize=address,undefined)
        target_link_options(enjima PRIVATE -fsanitize=address,undefined)
    endif()
endif()

# Public headers kennen (für IDEs / install)
set_target_properties(enjima PROPERTIES
        PUBLIC_HEADER
        "include/enjima/enjima.hpp;include/enjima/config.hpp;include/enjima/version.hpp"
)

# Tests (Catch2 via FetchContent)
include(FetchContent)
FetchContent_Declare(Catch2
        GIT_REPOSITORY https://github.com/catchorg/Catch2.git
        GIT_TAG v3.6.0)
FetchContent_MakeAvailable(Catch2)

add_executable(cpk_tests tests/basic.cpp)
target_link_libraries(cpk_tests PRIVATE enjima Catch2::Catch2WithMain)
enable_testing()
add_test(NAME enjima_tests COMMAND cpk_tests)

# Install (Headers + Lib)
include(GNUInstallDirs)
install(TARGETS enjima
        EXPORT enjimaTargets
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/enjima
)
install(DIRECTORY include/enjima/metrics include/enjima/detect include/enjima/report
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/enjima)

install(EXPORT enjimaTargets
        FILE enjimaTargets.cmake
        NAMESPACE enjima::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/enjima)

include(CMakePackageConfigHelpers)
configure_package_config_file(
        "${CMAKE_CURRENT_LIST_DIR}/cmake/Config.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/enjimaConfig.cmake"
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/enjima
)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/enjimaConfig.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/enjima)
