cmake_minimum_required(VERSION 3.13)

project(example_library C)

# =========================================================================
# Configuration
# =========================================================================

message(STATUS "Configuring external library source: ${PROJECT_NAME}")

# The actual target name must match env_config.json libraries[].name.
# SDK root CMake passes LIB_NAME before add_subdirectory so the target can be
# auto-registered into the final firmware link.
if(NOT DEFINED LIB_NAME)
    set(LIB_NAME ${PROJECT_NAME})
endif()

message(STATUS "  External library target name: ${LIB_NAME}")

# Output directory is passed via CMAKE_CURRENT_BINARY_DIR by parent CMakeLists.txt
# But we can make it more explicit for debugging
if(CMAKE_CURRENT_BINARY_DIR MATCHES "libs/.*")
    message(STATUS "  Library output directory: ${CMAKE_CURRENT_BINARY_DIR}")
endif()

# =========================================================================
# Source Files
# =========================================================================

# 1) Always collect from src/**
# 2) Exclude test/example/benchmark sources from firmware build

set(LIB_SOURCE_GLOBS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c
)

file(GLOB_RECURSE LIB_SOURCES CONFIGURE_DEPENDS ${LIB_SOURCE_GLOBS})

# Exclude non-production source trees by path pattern.
list(FILTER LIB_SOURCES EXCLUDE REGEX "/(test|tests|example|examples|benchmark|benchmarks)/")
list(REMOVE_DUPLICATES LIB_SOURCES)

if(NOT LIB_SOURCES)
    message(FATAL_ERROR "No valid source files found for ${LIB_NAME}. Expected at least one file under src/ or port/**/src/")
endif()

message(STATUS "  Collected ${LIB_NAME} sources: ${LIB_SOURCES}")

# =========================================================================
# Compiler Settings (inherit from SDK)
# =========================================================================

# Use SDK's compiler flags and settings
# Most settings come from toolchain.cmake (via parent CMakeLists.txt)

# You can add library-specific compiler flags here if needed:
# target_compile_options(...) will be set below

# =========================================================================
# Build Library as Part of SDK
# =========================================================================

# Create static library. The parent SDK CMake will register this archive into
# firmware_link_libraries so it participates in the final firmware link.
add_library(${LIB_NAME} STATIC ${LIB_SOURCES})

# Expose public include directories
# BUILD_INTERFACE: When building this library
# INSTALL_INTERFACE: When other targets link to this library
target_include_directories(${LIB_NAME}
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/inc
)

# Add internal include directories (not exposed)
target_include_directories(${LIB_NAME}
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
        ${CMAKE_CURRENT_SOURCE_DIR}
)

# =========================================================================
# Library Output Location
# =========================================================================

# When compiled via SDK's add_subdirectory, output goes to:
# CMAKE_CURRENT_BINARY_DIR = app_dir/qos_build/build/libs/<lib_name>/<lib_version>

# Set where the library .a file will be placed
set(LIBRARY_OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/libs")
file(MAKE_DIRECTORY "${LIBRARY_OUTPUT_DIR}")

set_target_properties(${LIB_NAME} PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY "${LIBRARY_OUTPUT_DIR}"
    LIBRARY_OUTPUT_DIRECTORY "${LIBRARY_OUTPUT_DIR}"
)

message(STATUS "  Library output: ${LIBRARY_OUTPUT_DIR}/lib${LIB_NAME}.a")

message(STATUS "${LIB_NAME} library configured successfully")

