# ==============================================================================
# Threading Dependencies
# ==============================================================================

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

# Promote to CACHE so sibling directory scopes (e.g. lib/assets/) can see it.
# fsc_add_external_project only sets this in *this* directory's scope via
# PARENT_SCOPE, which siblings do not inherit.
set(FSC_TBB_INSTALL_DIR "${FSC_TBB_INSTALL_DIR}" CACHE STRING
    "TBB install prefix" FORCE)

if(NOT FSC_TBB_INJECT_ARGS)
    set(FSC_TBB_INJECT_ARGS
        "-DTBB_ROOT=${FSC_TBB_INSTALL_DIR}"
        "-DCMAKE_PREFIX_PATH=${FSC_TBB_INSTALL_DIR}"
        CACHE STRING "CMAKE_ARGS to inject TBB into sub-builds" FORCE)
endif()

# Always sync to the target that actually exists this configure. When TBB
# is reused from a shared prefix, FSC_TBB_TARGET is "" and this clears the
# stale "_TBB" from a prior configure.
set(FSC_TBB_BUILD_TARGET "${FSC_TBB_TARGET}" CACHE STRING
    "ExternalProject target that must finish before consumers of TBB" FORCE)

message(STATUS "Threading dependencies configured")
