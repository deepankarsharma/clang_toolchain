#
# MIT License
#
# Copyright (c) 2018 Joel Winarske
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
#

cmake_minimum_required(VERSION 3.11)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "MinSizeRel" CACHE STRING "Choose the type of build, options are: Debug, Release, or MinSizeRel." FORCE)
    message(STATUS "CMAKE_BUILD_TYPE not set, defaulting to MinSizeRel.")
endif()

set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH}" "${CMAKE_SOURCE_DIR}/cmake")

if(NOT BUILD_NUMBER)
    set(BUILD_NUMBER 0)
endif()
set(CLANG_TOOLCHAIN_VERSION 1.0.${BUILD_NUMBER})

set(PACKAGE_NAME clang-toolchain)
project(${PACKAGE_NAME} VERSION "${CLANG_TOOLCHAIN_VERSION}" LANGUAGES CXX C)

include (ExternalProject)

message(STATUS "Generator .............. ${CMAKE_GENERATOR}")
message(STATUS "Build Type ............. ${CMAKE_BUILD_TYPE}")

include(build_options)

if(NOT THIRD_PARTY_DIR)
    set(THIRD_PARTY_DIR ${CMAKE_SOURCE_DIR}/third_party CACHE PATH "Third party dir")
endif()

if(NOT SDK_ROOT_DIR)
    set(SDK_ROOT_DIR ${CMAKE_SOURCE_DIR})
endif()

if(NOT TOOLCHAIN_DIR)
    set(TOOLCHAIN_DIR ${SDK_ROOT_DIR}/sdk/toolchain CACHE PATH "Toolchain Dir")
endif()

if(NOT TOOLCHAIN_FILE_DIR)
    set(TOOLCHAIN_FILE_DIR ${CMAKE_BINARY_DIR} CACHE PATH "Toolchain File Dir")
endif()

if(NOT TARGET_SYSROOT)
    set(TARGET_SYSROOT ${SDK_ROOT_DIR}/sdk/sysroot CACHE PATH "Sysroot Dir")
endif()

if(NOT TARGET_TRIPLE)
    set(TARGET_TRIPLE arm-linux-gnueabihf CACHE STRING "Target Triple")
endif()

if(NOT LLVM_TARGETS_TO_BUILD)
    set(LLVM_TARGETS_TO_BUILD "X86|ARM|AArch64")
endif()

if(NOT LLVM_VERSION)
    set(LLVM_VERSION tags/RELEASE_701/final/)
endif()

if(NOT LLVM_VER_DIR)
    set(LLVM_VER_DIR 7.0.1)
endif()

set(LLVM_SRC_DIR ${THIRD_PARTY_DIR}/llvm)


set(LLVM_BIN_DIR ${TOOLCHAIN_DIR}/bin)

set(LLVM_CONFIG_PATH ${LLVM_BIN_DIR}/llvm-config)


#
# rsync sysroot
#
find_program(RSYNC rsync)
if(NOT BUILD_PLATFORM_SYSROOT AND RSYNC AND TARGET_HOSTNAME)

    MESSAGE(STATUS "Syncing sysroot from '${TARGET_HOSTNAME}'")

    set(TARGET_SYSROOT_RSYNC 
        mkdir -p ${TARGET_SYSROOT} && cd ${TARGET_SYSROOT} && 
        rsync -avz
            --exclude=firmware
            --exclude=modules
            ${TARGET_HOSTNAME}:/lib/ lib/ && 
        rsync -avz 
            --exclude=lib/chromium-browser/
            --exclude=lib/firefox/
            --exclude=lib/cups/backend/cups-brf # random permission error
            ${TARGET_HOSTNAME}:/usr/ usr/)

    if(BUILD_PLATFORM_RPI)
        set(TARGET_SYSROOT_RSYNC ${TARGET_SYSROOT_RSYNC} && 
            mkdir -p ${TARGET_SYSROOT}/opt && 
            rsync -avz ${TARGET_HOSTNAME}:/opt/vc opt/)
    endif()

    set(BUILD_SYSROOT ON)
    ExternalProject_Add(sysroot
        DOWNLOAD_COMMAND ""
        UPDATE_COMMAND ${TARGET_SYSROOT_RSYNC}
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ""
        INSTALL_COMMAND ""
    )
endif()


#
# create target toolchain file
#
set(PKG_CONFIG_PATH ${TARGET_SYSROOT}/usr/lib/${TARGET_TRIPLE}/pkgconfig:${TARGET_SYSROOT}/usr/share/pkgconfig)

if(BUILD_PLATFORM_RPI)
    set(PKG_CONFIG_PATH ${PKG_CONFIG_PATH}:${TARGET_SYSROOT}/opt/vc/lib/pkgconfig)

    # baseline sysroot
    if(BUILD_PLATFORM_SYSROOT)
        set(BUILD_SYSROOT ON)
        include(rpi/sysroot)
    endif()
endif()

if(NOT BUILD_LIBCXXABI_EXCEPTIONS)
    set(TARGET_CXX_FLAGS "-stdlib=libc++ -fno-exceptions")
else()
    set(TARGET_CXX_FLAGS "-stdlib=libc++")
endif()

if(BUILD_BINUTILS)
    set(TARGET_LINKER gold)
else()
    set(BUILD_LLD ON)
    set(TARGET_LINKER lld)
endif()

set(TARGET_C_LINK_FLAGS   "--rtlib=compiler-rt -fuse-ld=${TARGET_LINKER}")
set(TARGET_CXX_LINK_FLAGS "--rtlib=compiler-rt -fuse-ld=${TARGET_LINKER} -lc++abi -lunwind")

configure_file(${CMAKE_SOURCE_DIR}/cmake/target.clang.toolchain.cmake.in ${TOOLCHAIN_FILE_DIR}/target.toolchain.cmake @ONLY)


#
# built for host
#

set(LLVM_CHECKOUT cd ${THIRD_PARTY_DIR} &&
    svn co -q http://llvm.org/svn/llvm-project/llvm/${LLVM_VERSION} llvm &&
    svn co -q http://llvm.org/svn/llvm-project/cfe/${LLVM_VERSION} clang)

set(LLVM_ENABLE_PROJECTS "clang")

if(BUILD_LLD)
    set(LLVM_CHECKOUT ${LLVM_CHECKOUT} && cd ${THIRD_PARTY_DIR} &&
        svn co -q http://llvm.org/svn/llvm-project/lld/${LLVM_VERSION} lld)
    set(LLVM_ENABLE_PROJECTS "${LLVM_ENABLE_PROJECTS}|lld")
endif()

if(BUILD_LLDB)
    set(LLVM_CHECKOUT ${LLVM_CHECKOUT} && cd ${THIRD_PARTY_DIR} &&
        svn co -q http://llvm.org/svn/llvm-project/lldb/${LLVM_VERSION} lldb)
    set(LLVM_ENABLE_PROJECTS "${LLVM_ENABLE_PROJECTS}|lldb")
endif()

if(BUILD_COMPILER_RT)
    set(LLVM_CHECKOUT ${LLVM_CHECKOUT} && cd ${THIRD_PARTY_DIR} &&
    svn co -q http://llvm.org/svn/llvm-project/compiler-rt/${LLVM_VERSION} compiler-rt)
    set(LLVM_ENABLE_PROJECTS "${LLVM_ENABLE_PROJECTS}|compiler-rt")
endif()

if(BUILD_LIBUNWIND)
    set(LLVM_CHECKOUT ${LLVM_CHECKOUT} && cd ${THIRD_PARTY_DIR} &&
    svn co -q http://llvm.org/svn/llvm-project/libunwind/${LLVM_VERSION} libunwind)
    set(LLVM_ENABLE_PROJECTS "${LLVM_ENABLE_PROJECTS}|libunwind")
endif()

if(BUILD_LIBCXXABI)
    set(LLVM_CHECKOUT ${LLVM_CHECKOUT} && cd ${THIRD_PARTY_DIR} &&
        svn co -q http://llvm.org/svn/llvm-project/libcxxabi/${LLVM_VERSION} libcxxabi)
    set(LLVM_ENABLE_PROJECTS "${LLVM_ENABLE_PROJECTS}|libcxxabi")
endif()

if(BUILD_LIBCXX)
    set(LLVM_CHECKOUT ${LLVM_CHECKOUT} && cd ${THIRD_PARTY_DIR} &&
        svn co -q http://llvm.org/svn/llvm-project/libcxx/${LLVM_VERSION} libcxx)
    set(LLVM_ENABLE_PROJECTS "${LLVM_ENABLE_PROJECTS}|libcxx")
endif()

ExternalProject_Add(clang
    DOWNLOAD_COMMAND
        ${CMAKE_COMMAND} -E make_directory ${THIRD_PARTY_DIR} && ${LLVM_CHECKOUT}
    SOURCE_DIR ${LLVM_SRC_DIR}
    UPDATE_COMMAND ""
    BUILD_IN_SOURCE 0
    LIST_SEPARATOR |
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${TOOLCHAIN_DIR}
        -DCMAKE_BUILD_TYPE=MinSizeRel
        -DCMAKE_VERBOSE_MAKEFILE=${CMAKE_VERBOSE_MAKEFILE}
        -DLLVM_DEFAULT_TARGET_TRIPLE=${TARGET_TRIPLE}
        -DLLVM_TARGETS_TO_BUILD=${LLVM_TARGETS_TO_BUILD}
        -DLLVM_ENABLE_ASSERTIONS=ON
        -DLLVM_BUILD_DOCS=OFF
        -DLLVM_INCLUDE_DOCS=OFF
        -DLLVM_ENABLE_DOXYGEN=OFF
        -DLLVM_ENABLE_SPHINX=OFF
        -DLLVM_ENABLE_OCAMLDOC=OFF
        -DLLVM_ENABLE_BINDINGS=OFF
        -DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON
        -DLLVM_ENABLE_PROJECTS=${LLVM_ENABLE_PROJECTS}
        -DLLVM_BUILD_RUNTIME=OFF
        -DLLVM_USE_HOST_TOOLS=ON
)

if(BUILD_BINUTILS)

    if(${TARGET_TRIPLE} MATCHES "arm-none-eabi")
        set(BINUTILS_TAG users/ARM/embedded-binutils-master-2018q4)
    else()
        set(BINUTILS_TAG binutils-2_31_1)
    endif()
    
    # Clang patch
    if (CMAKE_CXX_COMPILER_ID MATCHES ".*[cC]lang")
        message(STATUS "Patching for Clang")
        set(BINUTILS_PATCH 
            git checkout gold/gold-threads.cc && 
            git apply ${CMAKE_SOURCE_DIR}/cmake/patches/binutils/gold.patch)
    else()
        set(BINUTILS_PATCH
            git checkout gold/gold-threads.cc)
    endif()

    ExternalProject_Add(binutils
        GIT_REPOSITORY git://sourceware.org/git/binutils-gdb.git
        GIT_TAG ${BINUTILS_TAG}
        GIT_SHALLOW ON
        PATCH_COMMAND ${BINUTILS_PATCH}
        BUILD_IN_SOURCE 0
        SOURCE_DIR ${THIRD_PARTY_DIR}/binutils
        UPDATE_COMMAND ""
        CONFIGURE_COMMAND
            PATH=${LLVM_BIN_DIR}:$ENV{PATH}
            ${THIRD_PARTY_DIR}/binutils/configure
                --prefix=${TOOLCHAIN_DIR}
                --target=${TARGET_TRIPLE}
                --enable-gold
                --enable-ld
                --enable-lto
                --disable-werror
                #--enable-64-bit-bfd
    )
    add_dependencies(binutils clang)
endif()

#
# cross compile for target
#

configure_file(cmake/clang.toolchain.cmake.in ${CMAKE_BINARY_DIR}/toolchain.cmake @ONLY)

if(BUILD_NEWLIB)
    include(newlib)
endif()

if(BUILD_COMPILER_RT)
    
    ExternalProject_Add(compiler-rt
        DOWNLOAD_COMMAND ""
        SOURCE_DIR ${THIRD_PARTY_DIR}/compiler-rt
        BUILD_IN_SOURCE 0
        UPDATE_COMMAND ""
        CMAKE_ARGS
            -DCMAKE_TOOLCHAIN_FILE=${CMAKE_BINARY_DIR}/toolchain.cmake
            -DCMAKE_INSTALL_PREFIX=${TOOLCHAIN_DIR}/lib/clang/${LLVM_VER_DIR}
            -DCMAKE_BUILD_TYPE=MinSizeRel
            -DCMAKE_VERBOSE_MAKEFILE=ON
            -DLLVM_CONFIG_PATH=${LLVM_BIN_DIR}/llvm-config
            -DCOMPILER_RT_HAS_FPIC_FLAG=ON
            -DCOMPILER_RT_BUILD_BUILTINS=ON
            -DCOMPILER_RT_BUILD_SANITIZERS=ON
            -DCOMPILER_RT_BUILD_XRAY=ON
            -DCOMPILER_RT_BUILD_LIBFUZZER=OFF
            -DCOMPILER_RT_BUILD_PROFILE=OFF
            -DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=OFF # LLVM BUG: for arm-linux-eabihf this flags causes install to be armhf-linux-eabihf
    )
    add_dependencies(compiler-rt clang)
    if(BUILD_BINUTILS)
        add_dependencies(compiler-rt binutils)
    endif()
    if(BUILD_SYSROOT)
        add_dependencies(compiler-rt sysroot)
    endif()

endif()

if(BUILD_LIBCXXABI)
    ExternalProject_Add(libcxxabi
        DOWNLOAD_COMMAND ""
        SOURCE_DIR ${THIRD_PARTY_DIR}/libcxxabi
        BUILD_IN_SOURCE 0
        UPDATE_COMMAND ""
        CMAKE_ARGS
            -DCMAKE_TOOLCHAIN_FILE=${CMAKE_BINARY_DIR}/toolchain.cmake
            -DCMAKE_INSTALL_PREFIX=${TOOLCHAIN_DIR}/lib/clang/${LLVM_VER_DIR}/${TARGET_TRIPLE}
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_VERBOSE_MAKEFILE=${CMAKE_VERBOSE_MAKEFILE}
            -DLLVM_CONFIG_PATH=${LLVM_BIN_DIR}/llvm-config
            -DLIBCXXABI_SYSROOT=${TARGET_SYSROOT}
            -DLIBCXXABI_TARGET_TRIPLE=${TARGET_TRIPLE}
            -DLIBCXXABI_ENABLE_SHARED=ON
            -DLIBCXXABI_USE_COMPILER_RT=${BUILD_COMPILER_RT}
            -DLIBCXXABI_USE_LLVM_UNWINDER=${BUILD_LIBUNWIND}
            -DLIBCXXABI_ENABLE_STATIC_UNWINDER=${BUILD_LIBUNWIND}
            -DLIBCXXABI_ENABLE_EXCEPTIONS=${BUILD_LIBCXXABI_EXCEPTIONS}
            -DLIBCXXABI_ENABLE_ASSERTIONS=${BUILD_LIBCXXABI_ASSERTIONS}
            -DLIBCXXABI_ENABLE_PEDANTIC=ON
    )
    add_dependencies(libcxxabi clang)
    if(BUILD_BINUTILS)
        add_dependencies(libcxxabi binutils)
    endif()
    if(BUILD_COMPILER_RT)
        add_dependencies(libcxxabi compiler-rt)
    endif()
    if(BUILD_SYSROOT)
        add_dependencies(libcxxabi sysroot)
    endif()        
endif()

if(BUILD_LIBUNWIND)
    ExternalProject_Add(libunwind
        DOWNLOAD_COMMAND ""
        SOURCE_DIR ${THIRD_PARTY_DIR}/libunwind
        BUILD_IN_SOURCE 0
        UPDATE_COMMAND ""
        CMAKE_ARGS
            -DCMAKE_TOOLCHAIN_FILE=${CMAKE_BINARY_DIR}/toolchain.cmake
            -DCMAKE_INSTALL_PREFIX=${TOOLCHAIN_DIR}/lib/clang/${LLVM_VER_DIR}/${TARGET_TRIPLE}
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_VERBOSE_MAKEFILE=${CMAKE_VERBOSE_MAKEFILE}
            -DLLVM_CONFIG_PATH=${LLVM_BIN_DIR}/llvm-config
            -DLIBUNWIND_STANDALONE_BUILD=ON
            -DLIBUNWIND_TARGET_TRIPLE=${TARGET_TRIPLE}
            -DLIBUNWIND_SYSROOT=${TARGET_SYSROOT}
            -DLIBUNWIND_ENABLE_SHARED=ON
            -ULIBUNWIND_USE_COMPILER_RT
    )
    add_dependencies(libunwind clang)
    if(BUILD_BINUTILS)
        add_dependencies(libunwind binutils)
    endif()
    if(BUILD_COMPILER_RT)
        add_dependencies(libunwind compiler-rt)
    endif()
    if(BUILD_LIBCXXABI AND BUILD_LIBUNWIND)
        add_dependencies(libcxxabi libunwind)
    endif()
    if(BUILD_SYSROOT)
        add_dependencies(libunwind sysroot)
    endif()        
endif()

if(BUILD_LIBCXX)
    ExternalProject_Add(libcxx
        DOWNLOAD_COMMAND ""
        SOURCE_DIR ${THIRD_PARTY_DIR}/libcxx
        BUILD_IN_SOURCE 0
        UPDATE_COMMAND ""
        CMAKE_ARGS
            -DCMAKE_TOOLCHAIN_FILE=${CMAKE_BINARY_DIR}/toolchain.cmake
            -DCMAKE_INSTALL_PREFIX=${TOOLCHAIN_DIR}/lib/clang/${LLVM_VER_DIR}/${TARGET_TRIPLE}
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DCMAKE_VERBOSE_MAKEFILE=${CMAKE_VERBOSE_MAKEFILE}
            -DLLVM_CONFIG_PATH=${LLVM_BIN_DIR}/llvm-config
            -DLIBCXX_STANDALONE_BUILD=OFF
            -DLIBCXX_SYSROOT=${TARGET_SYSROOT}
            -DLIBCXX_TARGET_TRIPLE=${TARGET_TRIPLE}
            -DLIBCXX_USE_COMPILER_RT=${BUILD_COMPILER_RT}
            -DLIBCXX_ENABLE_SHARED=ON
            -DLIBCXXABI_USE_LLVM_UNWINDER=${BUILD_LIBUNWIND}
            -DLIBCXXABI_ENABLE_STATIC_UNWINDER=${BUILD_LIBUNWIND}
    )
    add_dependencies(libcxx libcxxabi)
    if(BUILD_BINUTILS)
        add_dependencies(libcxx binutils)
    endif()
   if(BUILD_COMPILER_RT)
        add_dependencies(libcxx compiler-rt)
    endif()
    if(BUILD_SYSROOT)
        add_dependencies(libcxx sysroot)
    endif()
endif()

#
# Platform Apps
#
if(BUILD_PLATFORM_RPI)
    include(rpi/apps)
endif()

if(BUILD_WAYLAND)
    include(wayland/apps)
endif()

#
# Common
#
if(BUILD_MRAA AND BUILD_LIBCXXABI AND BUILD_LIBCXX)
    ExternalProject_Add(mraa
        GIT_REPOSITORY https://github.com/intel-iot-devkit/mraa.git
        GIT_TAG master
        GIT_SHALLOW ON
        SOURCE_DIR ${THIRD_PARTY_DIR}/mraa
        BUILD_IN_SOURCE 0
        UPDATE_COMMAND ""
        CMAKE_ARGS
            -DCMAKE_TOOLCHAIN_FILE=${TOOLCHAIN_FILE_DIR}/target.toolchain.cmake
            -DCMAKE_INSTALL_PREFIX=${TARGET_SYSROOT}
            -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
            -DBUILDSWIG=OFF
            -DBUILDSWIGPYTHON=OFF
            -DBUILDSWIGNODE=OFF
            -DJSONPLAT=OFF
            -DBUILDTESTS=OFF
    )
    add_dependencies(mraa libcxxabi libcxx)
    if(BUILD_BINUTILS)
        add_dependencies(mraa binutils)
    endif()
    if(BUILD_SYSROOT)
        add_dependencies(mraa sysroot)
    endif()
endif()
