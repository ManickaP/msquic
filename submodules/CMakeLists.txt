# Copyright (c) Microsoft Corporation.
# Licensed under the MIT License.

cmake_minimum_required(VERSION 3.16)

# This is a helper project to build OpenSSL as part of the CMake "superbuild"
# pattern, which sidesteps issues managing a dependency graph within a single
# CMake project and enables smoother developer workflows. Currently, this only
# supports the Win32 platform

# This file is intended to be included in the parent msquic project via FetchContent
project(OpenSSLQuic)

option(QUIC_BUILD_DIR "QUIC build directory" ${CMAKE_CURRENT_BINARY_DIR})
option(OPENSSL_DIR "OpenSSL build directory" ${QUIC_BUILD_DIR}/openssl)

set(LIBSSL_DEBUG_PATH ${OPENSSL_DIR}/debug/lib/libssl${CMAKE_STATIC_LIBRARY_SUFFIX})
set(LIBCRYPTO_DEBUG_PATH ${OPENSSL_DIR}/debug/lib/libcrypto${CMAKE_STATIC_LIBRARY_SUFFIX})
set(LIBSSL_PATH ${OPENSSL_DIR}/release/lib/libssl${CMAKE_STATIC_LIBRARY_SUFFIX})
set(LIBCRYPTO_PATH ${OPENSSL_DIR}/release/lib/libcrypto${CMAKE_STATIC_LIBRARY_SUFFIX})

if(NOT WIN32)
    message(FATAL_ERROR "This subproject does not yet support non-Win32 platforms")
endif()

if (QUIC_UWP_BUILD)
    # Translate target architecture into corresponding OpenSSL build flag
    if (${SYSTEM_PROCESSOR} STREQUAL "arm64")
        set(QUIC_OPENSSL_WIN_ARCH "VC-WIN64-ARM")
    elseif (${SYSTEM_PROCESSOR} STREQUAL "arm")
        set(QUIC_OPENSSL_WIN_ARCH "VC-WIN32-ARM")
    elseif (${SYSTEM_PROCESSOR} STREQUAL "win32")
        set(QUIC_OPENSSL_WIN_ARCH "VC-WIN32-ONECORE")
    elseif (${SYSTEM_PROCESSOR} STREQUAL "x64" OR ${SYSTEM_PROCESSOR} STREQUAL "amd64")
        set(QUIC_OPENSSL_WIN_ARCH "VC-WIN64A-ONECORE")
    else()
        message(FATAL_ERROR "Unknown Generator Platform ${SYSTEM_PROCESSOR}")
    endif()
else()
    # Translate target architecture into corresponding OpenSSL build flag
    if (${SYSTEM_PROCESSOR} STREQUAL "arm64")
        set(QUIC_OPENSSL_WIN_ARCH "VC-WIN64-ARM")
    elseif (${SYSTEM_PROCESSOR} STREQUAL "arm")
        set(QUIC_OPENSSL_WIN_ARCH "VC-WIN32-ARM")
    elseif (${SYSTEM_PROCESSOR} STREQUAL "win32")
        set(QUIC_OPENSSL_WIN_ARCH "VC-WIN32")
    elseif (${SYSTEM_PROCESSOR} STREQUAL "x64" OR ${SYSTEM_PROCESSOR} STREQUAL "amd64")
        set(QUIC_OPENSSL_WIN_ARCH "VC-WIN64A")
    else()
        message(FATAL_ERROR "Unknown Generator Platform ${SYSTEM_PROCESSOR}")
    endif()
endif()

set(OPENSSL_CONFIG_FLAGS
    enable-tls1_3 no-makedepend no-dgram no-ssl3 no-psk no-srp

    #
    # The following line is needed for the 3.0 branch.
    #
    # no-uplink no-cmp no-acvp_tests no-fips no-padlockeng no-siv
    no-zlib no-egd no-idea no-rc5 no-rc4 no-afalgeng
    no-comp no-cms no-ct no-srp no-srtp no-ts no-gost no-dso no-ec2m
    no-tls1 no-tls1_1 no-tls1_2 no-dtls no-dtls1 no-dtls1_2 no-ssl
    no-ssl3-method no-tls1-method no-tls1_1-method no-tls1_2-method no-dtls1-method no-dtls1_2-method
    no-siphash no-whirlpool no-aria no-bf no-blake2 no-sm2 no-sm3 no-sm4 no-camellia no-cast no-md4 no-mdc2 no-ocb no-rc2 no-rmd160 no-scrypt
    no-weak-ssl-ciphers no-shared no-tests ${QUIC_OPENSSL_WIN_ARCH})

if (QUIC_UWP_BUILD)
    list(APPEND OPENSSL_CONFIG_FLAGS no-async)
endif()

# Create working and output directories as needed
file(MAKE_DIRECTORY ${OPENSSL_DIR}/debug/include)
file(MAKE_DIRECTORY ${OPENSSL_DIR}/release/include)
file(MAKE_DIRECTORY ${QUIC_BUILD_DIR}/submodules/openssl/debug)
file(MAKE_DIRECTORY ${QUIC_BUILD_DIR}/submodules/openssl/release)

# Configure steps for debug and release variants
add_custom_command(
    WORKING_DIRECTORY ${QUIC_BUILD_DIR}/submodules/openssl/debug
    OUTPUT ${QUIC_BUILD_DIR}/submodules/openssl/debug/makefile
    COMMAND perl ${CMAKE_CURRENT_SOURCE_DIR}/openssl/Configure ${OPENSSL_CONFIG_FLAGS} --debug --prefix=${OPENSSL_DIR}/debug
    COMMENT "OpenSSL debug configure"
)
add_custom_command(
    WORKING_DIRECTORY ${QUIC_BUILD_DIR}/submodules/openssl/release
    OUTPUT ${QUIC_BUILD_DIR}/submodules/openssl/release/makefile
    COMMAND perl ${CMAKE_CURRENT_SOURCE_DIR}/openssl/Configure ${OPENSSL_CONFIG_FLAGS} --prefix=${OPENSSL_DIR}/release
    COMMENT "OpenSSL release configure"
)

# Compile/install commands for debug and release variants
add_custom_command(
    OUTPUT ${LIBSSL_DEBUG_PATH}
    BYPRODUCTS ${LIBCRYPTO_DEBUG_PATH}
    DEPENDS ${QUIC_BUILD_DIR}/submodules/openssl/debug/makefile
    WORKING_DIRECTORY ${QUIC_BUILD_DIR}/submodules/openssl/debug
    COMMAND nmake install_dev
    COMMENT "OpenSSL debug build"
)
add_custom_command(
    OUTPUT ${LIBSSL_PATH}
    BYPRODUCTS ${LIBCRYPTO_PATH}
    DEPENDS ${QUIC_BUILD_DIR}/submodules/openssl/release/makefile
    WORKING_DIRECTORY ${QUIC_BUILD_DIR}/submodules/openssl/release
    COMMAND nmake install_dev
    COMMENT "OpenSSL release build"
)

# Named target depending on the final lib artifacts produced by custom commands
add_custom_target(
    OpenSSL_Target
    DEPENDS
    $<$<CONFIG:Debug>:${LIBSSL_DEBUG_PATH}>
    $<$<NOT:$<CONFIG:Debug>>:${LIBSSL_PATH}>
)
set_property(TARGET OpenSSL_Target PROPERTY FOLDER "${QUIC_FOLDER_PREFIX}helpers")

# Target to export to parent project
add_library(OpenSSLQuic INTERFACE)
add_dependencies(OpenSSLQuic OpenSSL_Target)
target_include_directories(
    OpenSSLQuic
    INTERFACE
    $<$<CONFIG:Debug>:${OPENSSL_DIR}/debug/include>
    $<$<NOT:$<CONFIG:Debug>>:${OPENSSL_DIR}/release/include>
)
target_link_libraries(
    OpenSSLQuic
    INTERFACE
    $<$<CONFIG:Debug>:${LIBSSL_DEBUG_PATH}>
    $<$<CONFIG:Debug>:${LIBCRYPTO_DEBUG_PATH}>
    $<$<NOT:$<CONFIG:Debug>>:${LIBSSL_PATH}>
    $<$<NOT:$<CONFIG:Debug>>:${LIBCRYPTO_PATH}>
)
add_library(OpenSSLQuic::OpenSSLQuic ALIAS OpenSSLQuic)

