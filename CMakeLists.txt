cmake_minimum_required(VERSION 3.14)
project(v3d VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Interface library
add_library(v3d INTERFACE)
add_library(v3d::v3d ALIAS v3d)

target_include_directories(v3d INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

# Installation targets
include(GNUInstallDirs)
install(TARGETS v3d EXPORT v3dTargets)
install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(DIRECTORY shaders/ DESTINATION ${CMAKE_INSTALL_DATADIR}/v3d/shaders)
install(EXPORT v3dTargets
    FILE v3dConfig.cmake
    NAMESPACE v3d::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/v3d
)

# Optional components
option(V3D_BUILD_EXAMPLES "Build V3D examples" ON)
option(V3D_BUILD_TESTS "Build V3D unit tests" ON)

if(V3D_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

if(V3D_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
