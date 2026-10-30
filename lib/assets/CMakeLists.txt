# ==============================================================================
# Assets Dependencies
# ==============================================================================

fsc_add_external_project(
    NAME Zstd
    GITHUB_REPOSITORY facebook/zstd
    GIT_TAG            v1.5.7
    INSTALL_SUBDIR     zstd
    INSTALL_HEADER     include/zstd.h
    CMAKE_ARGS
        -DZSTD_BUILD_PROGRAMS=OFF
        -DZSTD_BUILD_STATIC=ON
        -DZSTD_BUILD_SHARED=ON
        "-DCMAKE_C_FLAGS=$ENV{CFLAGS}"
    EXTERNAL_PROJECT_ARGS
        SOURCE_SUBDIR build/cmake
)

# ------------------------------------------------------------------------------
# OpenEXR — must be built explicitly. OIIO's nested build (via
# OpenImageIO_BUILD_MISSING_DEPS) does not propagate CMAKE_C_FLAGS /
# CMAKE_C_STANDARD, so OpenEXRCore gets compiled without C11 atomics and
# fails on `atomic_uintptr_t` with Clang 22.
# ------------------------------------------------------------------------------
fsc_add_external_project(
    NAME OpenEXR
    GITHUB_REPOSITORY AcademySoftwareFoundation/openexr
    GIT_TAG            v3.5.2
    TEST_TARGET        OpenEXR::OpenEXR
    INSTALL_SUBDIR     openexr
    INSTALL_HEADER     include/OpenEXR/ImfHeader.h
    DEPENDS            Zstd
    CMAKE_ARGS
        -DCMAKE_C_STANDARD=11
        "-DCMAKE_C_FLAGS=-std=gnu11 -include stdint.h -include stdatomic.h"
        -DBUILD_TESTING=OFF
        -DOPENEXR_BUILD_TOOLS=OFF
        -DOPENEXR_BUILD_EXAMPLES=OFF
        -DOPENEXR_BUILD_PYTHON=OFF
        -DOPENEXR_INSTALL_PKG_CONFIG=OFF
)

# ------------------------------------------------------------------------------
# libdeflate — OpenImageIO (via its locally-built TIFF) requires it, and
# libdeflate 1.23 uses Clang target attributes that reference the `evex512`
# feature. Clang 22 removed that feature; the attribute is ignored wholesale
# and `_mm512_*` intrinsics then fail with "requires target feature 'avx512f'".
# Strip the two tokens so the remaining AVX-512 features survive.
# ------------------------------------------------------------------------------
fsc_add_external_project(
    NAME libdeflate
    GITHUB_REPOSITORY ebiggers/libdeflate
    GIT_TAG            v1.26
    INSTALL_SUBDIR     libdeflate
    INSTALL_HEADER     include/libdeflate.h
    CMAKE_ARGS
        -DLIBDEFLATE_BUILD_SHARED_LIB=ON
        -DLIBDEFLATE_BUILD_STATIC_LIB=OFF
        -DLIBDEFLATE_BUILD_GZIP=OFF
    EXTERNAL_PROJECT_ARGS
        PATCH_COMMAND
            bash -c "find <SOURCE_DIR>/lib -type f \\( -name '*.h' -o -name '*.c' \\) -exec sed -i -e 's/,no-evex512//g' -e 's/,evex512//g' {} +"
)

# ------------------------------------------------------------------------------
# yaml-cpp — OpenImageIO's nested OpenColorIO build wants yaml-cpp 0.8.0.
# yaml-cpp 0.8.0's emitterutils.cpp uses uint16_t/uint32_t without including
# <cstdint>; Clang 22 no longer picks it up transitively. -include cstdint
# forces the header before every TU.
# ------------------------------------------------------------------------------
fsc_add_external_project(
    NAME yaml-cpp
    GITHUB_REPOSITORY jbeder/yaml-cpp
    GIT_TAG            yaml-cpp-0.9.0
    INSTALL_SUBDIR     yaml-cpp
    INSTALL_HEADER     include/yaml-cpp/yaml.h
    CMAKE_ARGS
        -DYAML_CPP_BUILD_TESTS=OFF
        -DYAML_CPP_BUILD_TOOLS=OFF
        -DYAML_CPP_BUILD_CONTRIB=OFF
        -DYAML_CPP_INSTALL=ON
        "-DCMAKE_CXX_FLAGS=$ENV{CXXFLAGS} -include cstdint"
)

# ------------------------------------------------------------------------------
# Imath — OpenImageIO's nested OpenColorIO build clones and rebuilds Imath
# when its sub-build's find_package(Imath) fails. The sub-build only sees
# the parent's *direct*-DEPENDS paths, not transitive ones (Imath arrives
# via OpenEXR), so OCIO never sees the Imath bundled inside _deps/openexr.
# Prebuilding Imath as its own prefix makes it a direct dep so OIO
# propagates its path to OCIO.
#
# Same $ENV{CXXFLAGS} splice as yaml-cpp: without it, OCIO's sub-build
# falls back to clang's default headers (gcc-15.3.0, no glibc-dev) and
# dies on `math.h`/`stdlib.h` not found.
# ------------------------------------------------------------------------------
fsc_add_external_project(
    NAME Imath
    GITHUB_REPOSITORY AcademySoftwareFoundation/Imath
    GIT_TAG            v3.2.3
    INSTALL_SUBDIR     imath
    INSTALL_HEADER     include/Imath/ImathVec.h
    CMAKE_ARGS
        -DBUILD_TESTING=OFF
        -DIMATH_BUILD_TESTS=OFF
        -DIMATH_INSTALL_PKG_CONFIG=OFF
        "-DCMAKE_CXX_FLAGS=$ENV{CXXFLAGS}"
)

# ------------------------------------------------------------------------------
# expat — required by OpenColorIO when OCIO_INSTALL_EXT_PACKAGES=NONE.
# ------------------------------------------------------------------------------
fsc_add_external_project(
    NAME expat
    GITHUB_REPOSITORY libexpat/libexpat
    GIT_TAG            R_2_9_0
    INSTALL_SUBDIR     expat
    INSTALL_HEADER     include/expat.h
    CMAKE_ARGS
        -DEXPAT_BUILD_TESTS=OFF
        -DEXPAT_BUILD_TOOLS=OFF
        -DEXPAT_BUILD_EXAMPLES=OFF
        -DEXPAT_BUILD_DOCS=OFF
        -DEXPAT_SHARED_LIBS=ON
        "-DCMAKE_C_FLAGS=$ENV{CFLAGS}"
        "-DCMAKE_CXX_FLAGS=$ENV{CXXFLAGS}"
    EXTERNAL_PROJECT_ARGS
        SOURCE_SUBDIR expat
)

# ------------------------------------------------------------------------------
# pystring — required by OpenColorIO when OCIO_INSTALL_EXT_PACKAGES=NONE.
# ------------------------------------------------------------------------------
fsc_add_external_project(
    NAME pystring
    GITHUB_REPOSITORY imageworks/pystring
    GIT_TAG            v1.2.0
    INSTALL_SUBDIR     pystring
    INSTALL_HEADER     include/pystring.h
    CMAKE_ARGS
        -DCMAKE_POLICY_VERSION_MINIMUM=3.5
        "-DCMAKE_CXX_FLAGS=$ENV{CXXFLAGS}"
)

# ------------------------------------------------------------------------------
# minizip-ng — required by OpenColorIO 2.5.1 when ext packages are disabled.
# ------------------------------------------------------------------------------
fsc_add_external_project(
    NAME minizip-ng
    GITHUB_REPOSITORY zlib-ng/minizip-ng
    GIT_TAG            4.2.2
    INSTALL_SUBDIR     minizip-ng
    INSTALL_HEADER     include/minizip-ng/mz.h
    CMAKE_ARGS
        -DMZ_BUILD_TESTS=OFF
        -DMZ_BUILD_UNIT_TESTS=OFF
        -DMZ_COMPAT=OFF
        -DMZ_ZLIB=ON
        "-DCMAKE_C_FLAGS=$ENV{CFLAGS}"
)

# ------------------------------------------------------------------------------
# OpenColorIO — prebuilt so OIIO's find_package(OpenColorIO) succeeds and
# it never enters build_OpenColorIO.cmake (which re-clones Imath with
# no CXXFLAGS and dies on clang's default toolchain paths).
# ------------------------------------------------------------------------------
fsc_add_external_project(
    NAME OpenColorIO
    GITHUB_REPOSITORY AcademySoftwareFoundation/OpenColorIO
    GIT_TAG            v2.6.0
    INSTALL_SUBDIR     opencolorio
    INSTALL_HEADER     include/OpenColorIO/OpenColorIO.h
    DEPENDS            Imath yaml-cpp expat pystring minizip-ng
    CMAKE_ARGS
        -DOCIO_BUILD_APPS=OFF
        -DOCIO_BUILD_TESTS=OFF
        -DOCIO_BUILD_GPU_TESTS=OFF
        -DOCIO_BUILD_PYTHON=OFF
        -DOCIO_BUILD_DOCS=OFF
        -DOCIO_INSTALL_EXT_PACKAGES=NONE
        -Dminizip-ng_ROOT=${CMAKE_BINARY_DIR}/_deps/minizip-ng
        -DEXPAT_ROOT=${CMAKE_BINARY_DIR}/_deps/expat
        -Dpystring_ROOT=${CMAKE_BINARY_DIR}/_deps/pystring
        "-DCMAKE_CXX_FLAGS=$ENV{CXXFLAGS}"
)

fsc_add_make_project(
    NAME               FFmpeg
    GITHUB_REPOSITORY  FFmpeg/FFmpeg
    GIT_TAG            n9.0.2
    INSTALL_SUBDIR     ffmpeg
    INSTALL_HEADER     include/libavcodec/avcodec.h
    INTERFACE_LIB_NAME libavcodec.so
    CONFIGURE_ARGS
        --disable-programs
        --disable-doc
        --disable-debug
        --disable-x86asm
        --enable-pthreads
    CFLAGS
        "-pthread"
    LDFLAGS
        "-pthread -latomic"
)

fsc_add_external_project(
    NAME OpenVDB
    GITHUB_REPOSITORY AcademySoftwareFoundation/openvdb
    GIT_TAG            v14.0.0
    INSTALL_SUBDIR     openvdb
    INSTALL_HEADER     include/openvdb/openvdb.h
    DEPENDS            Zstd
    CMAKE_ARGS
        -DOPENVDB_BUILD_BINARIES=OFF
        -DOPENVDB_BUILD_PYTHON_MODULE=OFF
        -DOPENVDB_BUILD_UNITTESTS=OFF
        -DUSE_NANOVDB=OFF
        "-DTBB_ROOT=${FSC_TBB_INSTALL_DIR}"
        "-DCMAKE_PREFIX_PATH=${FSC_TBB_INSTALL_DIR}"
        "-DCMAKE_CXX_FLAGS=$ENV{CXXFLAGS}"
)

# ==============================================================================
# OpenImageIO
# ==============================================================================
# Prefer the system/Nix-provided build. That gets us a fully-built OIIO plus
# all its transitive deps (imath, openexr, tiff, png, …) without ever hitting
# OpenImageIO_BUILD_MISSING_DEPS, and therefore without triggering the nested
# OpenEXR build that fails on Clang 22 with `atomic_uintptr_t`.
# TODO improve dependencies

fsc_add_external_project(
    NAME OpenImageIO
    GITHUB_REPOSITORY AcademySoftwareFoundation/OpenImageIO
    GIT_TAG            v3.2.2.0
    TEST_TARGET        OpenImageIO::OpenImageIO
    INSTALL_SUBDIR     openimageio
    INSTALL_HEADER     include/OpenImageIO/imageio.h
    DEPENDS            OpenEXR libdeflate Imath OpenColorIO FFmpeg OpenVDB Zstd
    EXTERNAL_PROJECT_DEPENDS ${FSC_TBB_BUILD_TARGET}
    CMAKE_ARGS
        -DOpenImageIO_BUILD_MISSING_DEPS=all
        -DUSE_OpenColorIO=ON
        -DUSE_QT=OFF
        -DUSE_PYTHON=OFF
        -DUSE_PTEX=OFF
        -DUSE_LIBHEIF=OFF
        -DUSE_FFMPEG=ON
        -DUSE_OPENCV=OFF
        -DUSE_DCMTK=OFF
        -DUSE_NUKE=OFF
        -DUSE_OPENVDB=ON
        -DOIIO_BUILD_TOOLS=OFF
        -DOIIO_BUILD_TESTS=OFF
        "-DOpenVDB_ROOT=${FSC_OpenVDB_INSTALL_DIR}"
        ${FSC_TBB_INJECT_ARGS}
        "-DZstd_ROOT=${CMAKE_BINARY_DIR}/_deps/zstd"
        "-Dzstd_ROOT=${CMAKE_BINARY_DIR}/_deps/zstd"
        "-DCMAKE_CXX_FLAGS=$ENV{CXXFLAGS}"
)

add_library(oiio INTERFACE)

if(TARGET OpenImageIO::OpenImageIO)
    target_link_libraries(oiio INTERFACE OpenImageIO::OpenImageIO)
elseif(TARGET OpenImageIO)

    # Provide -L<openimageio>/lib64 so -lOpenImageIO_Util (from PUBLIC_DEPS)
    # resolves. fsc_add_external_project only resolves the primary
    # libOpenImageIO.so; OpenImageIO_Util is a second library with no
    # interface entry, so the linker needs its directory on the search path.
    if(TARGET OpenImageIO AND FSC_OpenImageIO_INSTALL_DIR)
        set(_oiio_libdir "")
        foreach(_d lib lib64 "lib/${CMAKE_LIBRARY_ARCHITECTURE}")
            if(EXISTS "${FSC_OpenImageIO_INSTALL_DIR}/${_d}")
                set(_oiio_libdir "${FSC_OpenImageIO_INSTALL_DIR}/${_d}")
                break()
            endif()
        endforeach()
        if(_oiio_libdir)
            target_link_directories(OpenImageIO INTERFACE "${_oiio_libdir}")
            message(STATUS "OpenImageIO libdir: ${_oiio_libdir}")
        else()
            message(WARNING "OpenImageIO libdir not found under ${FSC_OpenImageIO_INSTALL_DIR}")
        endif()
        #target_link_libraries(OpenImageIO INTERFACE OpenImageIO_Util)
    endif()

    target_link_libraries(oiio INTERFACE OpenImageIO)
else()
    message(FATAL_ERROR
        "OpenImageIO not available: neither OpenImageIO::OpenImageIO nor "
        "OpenImageIO target exists")
endif()

fsc_add_library(
    NAME GTMesh
    GITHUB_REPOSITORY Tomas-Jakubec/GTMesh
    GIT_TAG          master
    CPM_ARGS         DOWNLOAD_ONLY YES
    POST_ADD_COMMANDS [[
        add_library(GTMesh INTERFACE)
        target_include_directories(GTMesh INTERFACE "${GTMesh_SOURCE_DIR}/src")
    ]]
)
