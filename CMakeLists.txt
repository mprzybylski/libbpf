CMAKE_MINIMUM_REQUIRED(VERSION 3.23.1)

project(libbpf
        VERSION 1.4.1
        DESCRIPTION "CMake wrapper for libbpf Makefile build"
        HOMEPAGE_URL https://github.com/libbpf/libbpf
        LANGUAGES C
)

option(BUILD_SHARED_LIBS "Build libbpf shared-object libraries" ON)

include(ExternalProject)

# NOTES on finding dependencies:
#   - The libbpf Makefile defaults to pkg-config to find its external
#     dependencies including header file and library locations. See the
#     environment variables listed in the pkg-config man page if it is
#     necessary to adjust pkg-config's search paths.
#   - Alternately, one could define the NO_PKG_CONFIG environment variable
#     and then define additional -I... arguments in the EXTRA_CFLAGS
#     environment variable and additional -L... and -l... arguments in the
#     EXTRA_LDFLAGS environment variable.

set(LIBBPF_COMMON_MAKEVARS PREFIX=${CMAKE_INSTALL_PREFIX})

# vcpkg sets CMAKE_INSTALL_LIBDIR.  See...
# https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_cmake_configure
if(DEFINED CMAKE_INSTALL_LIBDIR)
    list(APPEND LIBBPF_COMMON_MAKEVARS LIBDIR=${CMAKE_INSTALL_LIBDIR})
endif()

if(NOT BUILD_SHARED_LIBS)
    set(LIBBPF_BUILD_MAKEVARS BUILD_STATIC_ONLY=true)
endif()

set(LIBBPF_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/libbpf-prefix)
ExternalProject_Add(libbpf
        SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src
        # There's no configure script AND the libbpf Makefile was written for an in-source build,
        # so we'll rsync the source to our build directory before building.
        CONFIGURE_COMMAND rsync -a --delete ${CMAKE_CURRENT_SOURCE_DIR}/ ${LIBBPF_PREFIX}/src/libbpf-build/
        BUILD_COMMAND cd src && ${LIBBPF_COMMON_MAKEVARS} ${LIBBPF_BUILD_MAKEVARS} make
        BUILD_ALWAYS true
        # Stage to external project INSTALL_DIR
        INSTALL_COMMAND cd src && DESTDIR=${LIBBPF_PREFIX} ${LIBBPF_COMMON_MAKEVARS} make install
)

if(CMAKE_INSTALL_LIBDIR)
    install(DIRECTORY ${LIBBPF_PREFIX}/${CMAKE_INSTALL_PREFIX}/include
            DESTINATION ${CMAKE_INSTALL_PREFIX})
    cmake_path(GET CMAKE_INSTALL_FULL_LIBDIR PARENT_PATH LIBDIR_DESTINATION)
    install(DIRECTORY ${LIBBPF_PREFIX}/${CMAKE_INSTALL_LIBDIR} DESTINATION ${LIBDIR_DESTINATION})
else()
    install(DIRECTORY ${LIBBPF_PREFIX}/${CMAKE_INSTALL_PREFIX}/ DESTINATION ${CMAKE_INSTALL_PREFIX}
        PATTERN "lib*"
        PATTERN "include")
endif()