# ==============================================================================
# FrozenStarCrystal-graphics - Root CMakeLists.txt
# ==============================================================================
cmake_minimum_required(VERSION 4.0)

# ==============================================================================
# Bootstrap shared FSC tooling (FSCProject.cmake + friends)
# ==============================================================================
# The tooling lives in core's repo. We look for a sibling checkout first
# (fast, allows local edits), then fall back to downloading core's tarball
# into build/_fsc-tooling/.
#
# Override the search with:
#   -DFSC_CORE_DIR=/path/to/FrozenStarCrystal-core
# or the environment variable FSC_CORE_DIR.

set(FETCHCONTENT_QUIET FALSE CACHE BOOL
    "Show FetchContent/CPM download progress" FORCE)

# ==============================================================================
# Bootstrap shared FSC tooling
# ==============================================================================
# fsc_find_tooling() resolves the cmake/ directory containing FSCProject.cmake.
# It checks a sibling checkout first (fast, local edits), then falls back to
# downloading core's master tarball into build/_fsc-tooling/.
#
# Override the search with:
#   -DFSC_CORE_DIR=/path/to/FrozenStarCrystal-core
# or the environment variable FSC_CORE_DIR.

include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/FSCBootstrap.cmake")

fsc_find_tooling(_fsc_tooling_dir)
list(APPEND CMAKE_MODULE_PATH "${_fsc_tooling_dir}")
include(FSCProject)

project(FrozenStarCrystal-assets
    VERSION 0.1.0
    LANGUAGES CXX C
)

fsc_initialize_project()

# Load developer overrides for local FSC package paths (optional file).
fsc_load_local_paths()

# ------------------------------------------------------------------------------
# Dependencies (lib/ still does FetchContent for vk, vma, slang, core, ...)
# ------------------------------------------------------------------------------
add_subdirectory(lib)

# ------------------------------------------------------------------------------
# Build the module via the canonical entry point
# ------------------------------------------------------------------------------
add_module(
    PUBLIC_DEPS
        FrozenStarCrystal-core
        GTMesh
        oiio
        FFmpeg
)

# ------------------------------------------------------------------------------
# Assets
# ------------------------------------------------------------------------------
if(NOT EXISTS "${CMAKE_SOURCE_DIR}/assets")
    file(MAKE_DIRECTORY "${CMAKE_SOURCE_DIR}/assets")
endif()

if(PROJECT_IS_TOP_LEVEL AND EXISTS "${CMAKE_SOURCE_DIR}/assets")
    add_custom_target(copy_assets ALL
        COMMAND ${CMAKE_COMMAND} -E copy_directory
            "${CMAKE_SOURCE_DIR}/assets"
            "${OUTPUT_DIR}/assets"
        COMMENT "Copying assets to build directory")
    add_dependencies(${PROJECT_NAME} copy_assets)
endif()

# ------------------------------------------------------------------------------
# Tests
# ------------------------------------------------------------------------------
option(ENABLE_TESTS "Build tests" OFF)
option(FSC_TESTS    "Force enable tests" OFF)
if((ENABLE_TESTS AND PROJECT_IS_TOP_LEVEL) OR FSC_TESTS)
    add_subdirectory(tests)
endif()

fsc_finalize_top_level()
