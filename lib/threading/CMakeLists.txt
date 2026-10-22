# ==============================================================================
# Threading Dependencies
# ==============================================================================

find_package(TBB QUIET)   # NOT REQUIRED

if(TBB_FOUND)
    message(STATUS "Using system TBB at ${TBB_DIR}")

    # TBB_DIR points at <prefix>/lib/cmake/TBB or <prefix>/lib64/cmake/TBB.
    get_filename_component(_tbb_prefix "${TBB_DIR}/../../.." ABSOLUTE)

    set(FSC_TBB_INJECT_ARGS
        "-DTBB_DIR=${TBB_DIR}"
        "-DCMAKE_PREFIX_PATH=${_tbb_prefix}"
        CACHE STRING "CMAKE_ARGS to inject TBB into sub-builds" FORCE)
    set(FSC_TBB_BUILD_TARGET "" CACHE STRING
        "ExternalProject target that must finish before consumers of TBB" FORCE)
endif()

fsc_add_external_project(
    NAME               TBB
    GITHUB_REPOSITORY  uxlfoundation/onetbb
    GIT_TAG            v2023.2.0-rc1
    TEST_TARGET        TBB::tbb
    INSTALL_SUBDIR     tbb
    INSTALL_HEADER     include/oneapi/tbb.h
    INTERFACE_LIB_NAME ""
    CMAKE_ARGS
        -DTBB_TEST=OFF
        -DTBB_EXAMPLES=OFF
        -DTBB_STRICT=OFF
        "-DCMAKE_SHARED_LINKER_FLAGS=-Wl,--undefined-version"
        "-DCMAKE_C_SHARED_LINKER_FLAGS=-Wl,--undefined-version"
        "-DCMAKE_CXX_SHARED_LINKER_FLAGS=-Wl,--undefined-version"
)

if(NOT FSC_TBB_INJECT_ARGS AND NOT FSC_TBB_BUILD_TARGET)

    # FSC_TBB_INSTALL_DIR is visible HERE (PARENT_SCOPE from the helper),
    # so we can read it now and cache the result for siblings.
    set(FSC_TBB_INJECT_ARGS
        "-DTBB_ROOT=${FSC_TBB_INSTALL_DIR}"
        "-DCMAKE_PREFIX_PATH=${FSC_TBB_INSTALL_DIR}"
        CACHE STRING "CMAKE_ARGS to inject TBB into sub-builds" FORCE)
    set(FSC_TBB_BUILD_TARGET "_TBB" CACHE STRING
        "ExternalProject target that must finish before consumers of TBB" FORCE)

endif()

# Create an interface library to abstract the actual TBB target name
add_library(tbb INTERFACE)

if(TARGET TBB::tbb)
    target_link_libraries(tbb INTERFACE TBB::tbb)
elseif(TARGET TBB)
    target_link_libraries(tbb INTERFACE TBB)
endif()

message(STATUS "Threading dependencies configured")
