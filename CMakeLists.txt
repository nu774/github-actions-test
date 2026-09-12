cmake_minimum_required(VERSION 3.10)

# fdkaac_version is a plain C string literal in version.h ("X.Y.Z");
# reuse it as the CMake project version too.
file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/version.h" FDKAAC_VERSION_LINE
     REGEX "fdkaac_version[ \t]*=")
string(REGEX REPLACE ".*\"([0-9]+\\.[0-9]+\\.[0-9]+)\".*" "\\1"
       FDKAAC_VERSION "${FDKAAC_VERSION_LINE}")

project(fdkaac VERSION ${FDKAAC_VERSION} LANGUAGES C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED OFF)

include(CheckIncludeFile)
include(CheckSymbolExists)
include(CheckFunctionExists)
include(CheckTypeSize)
include(TestBigEndian)

# ---------------------------------------------------------------------------
# libfdk-aac: either the system package (default, via pkg-config) or built
# from source as a static library.
# ---------------------------------------------------------------------------
option(FDKAAC_BUILD_FDK_AAC
    "Fetch and build libfdk-aac from source as a static library instead of using the system package found via pkg-config"
    OFF)

if(FDKAAC_BUILD_FDK_AAC)
    # fdk-aac's own sources are C++; a C-only executable statically linking
    # it still needs the C++ runtime pulled in at final-link time, so we
    # need a C++ compiler available and must link fdkaac as C++.
    enable_language(CXX)
    include(ExternalProject)

    set(FDKAAC_FDK_AAC_REPOSITORY "https://github.com/mstorsjo/fdk-aac"
        CACHE STRING "Git repository libfdk-aac is fetched from when FDKAAC_BUILD_FDK_AAC is ON")
    set(FDKAAC_FDK_AAC_VERSION "v2.0.3"
        CACHE STRING "Version of libfdk-aac to build when FDKAAC_BUILD_FDK_AAC is ON")

    set(FDK_AAC_PREFIX "${CMAKE_BINARY_DIR}/fdk-aac-install")
    set(FDK_AAC_INCLUDE_DIR "${FDK_AAC_PREFIX}/include")
    set(FDK_AAC_LIBRARY
        "${FDK_AAC_PREFIX}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}fdk-aac${CMAKE_STATIC_LIBRARY_SUFFIX}")

    if(NOT CMAKE_BUILD_TYPE)
        set(FDK_AAC_BUILD_TYPE "Release")
    else()
        set(FDK_AAC_BUILD_TYPE "${CMAKE_BUILD_TYPE}")
    endif()

    set(FDKAAC_FDK_AAC_CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${FDK_AAC_PREFIX}
        -DCMAKE_INSTALL_LIBDIR=lib
        -DBUILD_SHARED_LIBS=OFF
        -DBUILD_PROGRAMS=OFF
        -DCMAKE_POSITION_INDEPENDENT_CODE=ON
        -DCMAKE_BUILD_TYPE=${FDK_AAC_BUILD_TYPE}
    )
    if(CMAKE_TOOLCHAIN_FILE)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE})
    endif()
    if(CMAKE_SYSTEM_NAME)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME})
    endif()
    if(CMAKE_SYSTEM_PROCESSOR)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_SYSTEM_PROCESSOR=${CMAKE_SYSTEM_PROCESSOR})
    endif()
    if(CMAKE_C_COMPILER)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER})
    endif()
    if(CMAKE_CXX_COMPILER)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER})
    endif()
    if(CMAKE_C_COMPILER_ARG1)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_C_COMPILER_ARG1=${CMAKE_C_COMPILER_ARG1})
    endif()
    if(CMAKE_CXX_COMPILER_ARG1)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_CXX_COMPILER_ARG1=${CMAKE_CXX_COMPILER_ARG1})
    endif()
    if(CMAKE_C_COMPILER_TARGET)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_C_COMPILER_TARGET=${CMAKE_C_COMPILER_TARGET})
    endif()
    if(CMAKE_CXX_COMPILER_TARGET)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_CXX_COMPILER_TARGET=${CMAKE_CXX_COMPILER_TARGET})
    endif()
    if(CMAKE_LINKER)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_LINKER=${CMAKE_LINKER})
    endif()
    if(CMAKE_MT)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_MT=${CMAKE_MT})
    endif()
    if(CMAKE_RC_COMPILER)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_RC_COMPILER=${CMAKE_RC_COMPILER})
    endif()
    if(APPLE AND CMAKE_OSX_ARCHITECTURES)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_OSX_ARCHITECTURES=${CMAKE_OSX_ARCHITECTURES})
    endif()
    if(APPLE AND CMAKE_OSX_DEPLOYMENT_TARGET)
        list(APPEND FDKAAC_FDK_AAC_CMAKE_ARGS -DCMAKE_OSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET})
    endif()

    ExternalProject_Add(fdk-aac-external
        URL              "${FDKAAC_FDK_AAC_REPOSITORY}/archive/refs/tags/${FDKAAC_FDK_AAC_VERSION}.tar.gz"
        PREFIX           "${CMAKE_BINARY_DIR}/fdk-aac-external"
        CMAKE_ARGS       ${FDKAAC_FDK_AAC_CMAKE_ARGS}
        BUILD_BYPRODUCTS "${FDK_AAC_LIBRARY}"
        INSTALL_DIR      "${FDK_AAC_PREFIX}"
        DOWNLOAD_EXTRACT_TIMESTAMP TRUE
    )

    file(MAKE_DIRECTORY "${FDK_AAC_INCLUDE_DIR}")

    add_library(fdk-aac-static STATIC IMPORTED GLOBAL)
    set_target_properties(fdk-aac-static PROPERTIES
        IMPORTED_LOCATION "${FDK_AAC_LIBRARY}"
        INTERFACE_INCLUDE_DIRECTORIES "${FDK_AAC_INCLUDE_DIR}"
    )

    set(FDK_AAC_LINK_TARGET fdk-aac-static)
else()
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(FDK_AAC REQUIRED IMPORTED_TARGET fdk-aac)
    set(FDK_AAC_LINK_TARGET PkgConfig::FDK_AAC)
endif()

# ---------------------------------------------------------------------------
# platform / feature checks (mirrors configure.ac)
# ---------------------------------------------------------------------------
test_big_endian(WORDS_BIGENDIAN)

check_include_file("stdint.h"    HAVE_STDINT_H)
check_include_file("inttypes.h"  HAVE_INTTYPES_H)
check_include_file("unistd.h"    HAVE_UNISTD_H)
check_include_file("endian.h"    HAVE_ENDIAN_H)
check_include_file("byteswap.h"  HAVE_BYTESWAP_H)
check_include_file("libcharset.h" HAVE_LIBCHARSET_H)
check_include_file("langinfo.h"  HAVE_LANGINFO_H)

check_symbol_exists(sigaction   "signal.h" HAVE_SIGACTION)
check_symbol_exists(getopt_long "getopt.h" HAVE_GETOPT_LONG)
check_function_exists(fseeko    HAVE_FSEEKO)
check_function_exists(fseeko64  HAVE_FSEEKO64)
check_symbol_exists(_vscprintf  "stdio.h"   HAVE__VSCPRINTF)
set(CMAKE_EXTRA_INCLUDE_FILES "sys/timeb.h")
check_type_size("struct __timeb64" STRUCT___TIMEB64)
unset(CMAKE_EXTRA_INCLUDE_FILES)

set(ICONV_LIBRARIES)
if(NOT WIN32)
    find_package(Iconv)
    if(Iconv_FOUND)
        set(HAVE_ICONV TRUE)
        set(ICONV_LIBRARIES ${Iconv_LIBRARIES})
    endif()
    if(HAVE_LIBCHARSET_H)
        find_library(CHARSET_LIBRARY NAMES charset iconv)
        if(CHARSET_LIBRARY)
            list(APPEND ICONV_LIBRARIES ${CHARSET_LIBRARY})
        endif()
    endif()
endif()

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.h.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/config.h"
)

# ---------------------------------------------------------------------------
# sources
# ---------------------------------------------------------------------------
set(FDKAAC_SOURCES
    src/aacenc.c
    src/caf_reader.c
    src/extrapolater.c
    src/limiter.c
    src/lpc.c
    src/m4af.c
    src/main.c
    src/metadata.c
    src/parson.c
    src/pcm_float_converter.c
    src/pcm_native_converter.c
    src/pcm_readhelper.c
    src/pcm_sint16_converter.c
    src/progress.c
    src/wav_reader.c
)

if(WIN32)
    list(APPEND FDKAAC_SOURCES src/compat_win32.c fdkaac.manifest)
    if (MINGW)
        list(APPEND FDKAAC_SOURCES fdkaac.rc)
    endif()
else()
    list(APPEND FDKAAC_SOURCES src/compat_posix.c)
endif()

if(NOT HAVE_GETOPT_LONG)
    list(APPEND FDKAAC_SOURCES missings/getopt.c)
endif()

add_executable(fdkaac ${FDKAAC_SOURCES})

target_include_directories(fdkaac PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}"
    "${CMAKE_CURRENT_BINARY_DIR}"
)
if(NOT HAVE_GETOPT_LONG)
    target_include_directories(fdkaac PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/missings")
endif()

target_compile_definitions(fdkaac PRIVATE HAVE_CONFIG_H)
if(NOT WIN32)
    target_compile_definitions(fdkaac PRIVATE _FILE_OFFSET_BITS=64)
endif()
if(WIN32)
    target_compile_definitions(fdkaac PRIVATE _CRT_SECURE_NO_DEPRECATE _CRT_NONSTDC_NO_DEPRECATE)
    if(MSVC)
        target_compile_definitions(fdkaac PRIVATE inline=_inline)
    endif()
endif()

target_link_libraries(fdkaac PRIVATE ${FDK_AAC_LINK_TARGET} ${ICONV_LIBRARIES})
if(NOT WIN32)
    target_link_libraries(fdkaac PRIVATE m)
endif()

if(FDKAAC_BUILD_FDK_AAC)
    add_dependencies(fdkaac fdk-aac-external)
    # fdk-aac's sources are C++; drive the final link with the C++ compiler
    # so its runtime (libstdc++/libc++) gets linked in automatically instead
    # of leaving unresolved C++ runtime symbols in a plain C link.
    set_target_properties(fdkaac PROPERTIES LINKER_LANGUAGE CXX)
endif()

install(TARGETS fdkaac RUNTIME DESTINATION bin)
install(FILES man/fdkaac.1 DESTINATION share/man/man1)
