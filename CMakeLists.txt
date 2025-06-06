cmake_minimum_required(VERSION 4.0)
project(ESST LANGUAGES C CXX ASM HIP)
set(CMAKE_C_COMPILER hipcc)
set(CMAKE_CXX_COMPILER hipcc)
option(BUILD_OLD_VERSION "Build the OLD version of the stresser (uses mainOld.cpp)" OFF)
option(BUILD_LIBRARY "Build as static library instead of executable" OFF)

list(APPEND CMAKE_PREFIX_PATH "/opt/rocm")  # Or your ROCm install path

# Find ROCm and HIP
find_package(HIP REQUIRED)
find_package(ROCmCMakeBuildTools REQUIRED)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_HIP_STANDARD 20)
set(CMAKE_HIP_STANDARD_REQUIRED ON)

# Optimization flags (Fixed CMAKE_FIP_FLAGS typo)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -march=native -mtune=native -Wall -pthread -Wextra -std=c++20")
set(CMAKE_HIP_FLAGS "${CMAKE_HIP_FLAGS} -O3 -march=native -mtune=native -Wall -pthread -Wextra -std=c++20 -use_fast_math")

# Enable NASM and HIP
enable_language(ASM_NASM)
enable_language(HIP)

include_directories(include)
include_directories(rocm/*.h)

# Source files
file(GLOB MAIN_HEADER_FILE "include/core.hpp")
file(GLOB ASM_SRC "asm/*.asm")
file(GLOB HIP_SRC "rocm/*.hip.cpp")  # Your HIP source files
file(GLOB HIP_HEADERS "rocm/*.h")

if(BUILD_LIBRARY)
    # Build static library
    set(LIBRARY_NAME "esst")
    add_library(${LIBRARY_NAME} STATIC
            ${ASM_SRC}
            ${MAIN_HEADER_FILE}
            ${HIP_SRC}
            ${HIP_HEADERS}
    )

    set_property(SOURCE ${ASM_SRC} PROPERTY LANGUAGE ASM_NASM)
    set_property(SOURCE ${HIP_SRC} PROPERTY LANGUAGE HIP)

    target_include_directories(${LIBRARY_NAME} PRIVATE rocm/ include/)
    target_include_directories(${LIBRARY_NAME} PUBLIC include/)  # Public for library users

    target_link_libraries(${LIBRARY_NAME} PUBLIC hip::host)
    target_link_libraries(${LIBRARY_NAME} PUBLIC hip::device)

    # Set output name to ensure .a extension
    set_target_properties(${LIBRARY_NAME} PROPERTIES OUTPUT_NAME "esst")

else()
    # Build executable
    if(BUILD_OLD_VERSION)
        set(EXECUTABLE_NAME "esstOld")
        set(MAIN_SOURCE_FILE "src/mainOld.cpp")
    else()
        set(EXECUTABLE_NAME "esst")
        set(MAIN_SOURCE_FILE "src/main.cpp")
    endif()

    # Add executable - INCLUDE HIP_SRC HERE
    add_executable(${EXECUTABLE_NAME}
            ${MAIN_SOURCE_FILE}
            ${ASM_SRC}
            ${HIP_SRC}  # This was missing!
    )

    set_property(SOURCE ${ASM_SRC} PROPERTY LANGUAGE ASM_NASM)

    # Set HIP files to use HIP language
    set_property(SOURCE ${HIP_SRC} PROPERTY LANGUAGE HIP)

    target_include_directories(${EXECUTABLE_NAME} PRIVATE rocm/ include/)

    target_link_libraries(${EXECUTABLE_NAME} PUBLIC hip::host)
    target_link_libraries(${EXECUTABLE_NAME} PUBLIC hip::device)
endif()