cmake_minimum_required(VERSION 3.15)
project(VehPhys VERSION 0.1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Allow custom find modules in cmake/
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Library definition
add_library(vehphysics SHARED)

# Folder settings
target_include_directories(vehphysics PUBLIC 
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

# Automatically collect source and header files from the project structure
file(GLOB_RECURSE VEH_PHYSICS_PUBLIC_HEADERS CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/include/*.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/*.hpp"
)

file(GLOB_RECURSE VEH_PHYSICS_PRIVATE_HEADERS CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.hpp"
)

file(GLOB_RECURSE VEH_PHYSICS_SOURCES CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cc"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cxx"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp"
)

# Keep configuration valid even before the first .cpp file exists.
if(NOT VEH_PHYSICS_SOURCES)
    set(VEH_PHYSICS_SOURCES "${CMAKE_CURRENT_BINARY_DIR}/vehphysics_dummy.cpp")
    file(WRITE "${VEH_PHYSICS_SOURCES}" "void vehphysics_dummy_symbol() {}\n")
endif()

target_sources(vehphysics PRIVATE
    ${VEH_PHYSICS_SOURCES}
    ${VEH_PHYSICS_PRIVATE_HEADERS}
    ${VEH_PHYSICS_PUBLIC_HEADERS}
)

# Optional subprojects
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/CMakeLists.txt")
    add_subdirectory(examples)
endif()

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/tests/CMakeLists.txt")
    enable_testing()
    add_subdirectory(tests)
endif()