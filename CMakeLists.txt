cmake_minimum_required(VERSION 3.22)
project(SessionProto VERSION 1.0 LANGUAGES CXX)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED True)

if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

include(FetchContent)
set(FETCHCONTENT_QUIET OFF)

set(GENERATED_DIR "${CMAKE_BINARY_DIR}/generated")
set(DEPS_DIR "${CMAKE_BINARY_DIR}/deps")
file(MAKE_DIRECTORY ${GENERATED_DIR})
file(MAKE_DIRECTORY ${DEPS_DIR})

set(PROTOC_VERSION "29.0")

# ============ DOWNLOAD PRE-BUILT PROTOC (for code generation only) ============
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|arm64")
        set(PROTOC_PLATFORM "linux-aarch_64")
    else()
        set(PROTOC_PLATFORM "linux-x86_64")
    endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm64")
        set(PROTOC_PLATFORM "osx-aarch_64")
    else()
        set(PROTOC_PLATFORM "osx-x86_64")
    endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(PROTOC_PLATFORM "win64")
endif()

set(PROTOC_BIN "${DEPS_DIR}/protoc/bin/protoc")
if(WIN32)
    set(PROTOC_BIN "${DEPS_DIR}/protoc/bin/protoc.exe")
endif()

if(NOT EXISTS "${DEPS_DIR}/protoc/bin")
    message(STATUS "Downloading protoc ${PROTOC_VERSION} for ${PROTOC_PLATFORM}...")
    FetchContent_Declare(protoc
        URL https://github.com/protocolbuffers/protobuf/releases/download/v${PROTOC_VERSION}/protoc-${PROTOC_VERSION}-${PROTOC_PLATFORM}.zip
        SOURCE_DIR ${DEPS_DIR}/protoc
    )
    FetchContent_Populate(protoc)
else()
    message(STATUS "protoc already present")
endif()

# ============ BUILD ABSEIL (for static libraries) ============
set(ABSL_PROPAGATE_CXX_STD ON CACHE BOOL "" FORCE)
set(ABSL_ENABLE_INSTALL OFF CACHE BOOL "" FORCE)
set(BUILD_TESTING OFF CACHE BOOL "" FORCE)
if(MSVC)
    set(ABSL_MSVC_STATIC_RUNTIME ON CACHE BOOL "" FORCE)
endif()

FetchContent_Declare(abseil
    GIT_REPOSITORY https://github.com/abseil/abseil-cpp.git
    GIT_TAG 20240722.0
    GIT_SHALLOW TRUE
)
FetchContent_MakeAvailable(abseil)

# ============ FAKE ABSL PACKAGE (for protobuf to find our abseil) ============
file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/fake_absl_package)
file(WRITE ${CMAKE_BINARY_DIR}/fake_absl_package/abslConfig.cmake "# Abseil already loaded via FetchContent")
file(WRITE ${CMAKE_BINARY_DIR}/fake_absl_package/abslConfigVersion.cmake
"set(PACKAGE_VERSION \"20240722.0\")
set(PACKAGE_VERSION_COMPATIBLE TRUE)
set(PACKAGE_VERSION_EXACT FALSE)
")
set(absl_DIR ${CMAKE_BINARY_DIR}/fake_absl_package CACHE PATH "" FORCE)

# ============ BUILD PROTOBUF LIBRARY (without submodules, uses our abseil) ============
set(protobuf_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_CONFORMANCE OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_LIBPROTOC OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_PROTOC_BINARIES OFF CACHE BOOL "" FORCE)
set(protobuf_INSTALL OFF CACHE BOOL "" FORCE)
set(protobuf_ABSL_PROVIDER "package" CACHE STRING "" FORCE)
set(protobuf_WITH_ZLIB OFF CACHE BOOL "" FORCE)
set(utf8_range_ENABLE_TESTS OFF CACHE BOOL "" FORCE)
set(utf8_range_ENABLE_INSTALL OFF CACHE BOOL "" FORCE)
if(MSVC)
    set(protobuf_MSVC_STATIC_RUNTIME ON CACHE BOOL "" FORCE)
endif()

FetchContent_Declare(protobuf
    GIT_REPOSITORY https://github.com/protocolbuffers/protobuf.git
    GIT_TAG v${PROTOC_VERSION}
    GIT_SHALLOW TRUE
    GIT_SUBMODULES ""
)
FetchContent_MakeAvailable(protobuf)

# ============ GENERATE PROTO FILES ============
file(GLOB PROTO_FILES "${CMAKE_CURRENT_SOURCE_DIR}/*.proto")

set(GENERATED_SOURCES "")
set(GENERATED_HEADERS "")

foreach(PROTO_FILE ${PROTO_FILES})
    get_filename_component(PROTO_NAME ${PROTO_FILE} NAME_WE)
    set(GEN_CC "${GENERATED_DIR}/${PROTO_NAME}.pb.cc")
    set(GEN_H "${GENERATED_DIR}/${PROTO_NAME}.pb.h")
    list(APPEND GENERATED_SOURCES ${GEN_CC})
    list(APPEND GENERATED_HEADERS ${GEN_H})

    add_custom_command(
        OUTPUT ${GEN_CC} ${GEN_H}
        COMMAND ${PROTOC_BIN}
            --cpp_out=${GENERATED_DIR}
            --proto_path=${CMAKE_CURRENT_SOURCE_DIR}
            ${PROTO_FILE}
        DEPENDS ${PROTO_FILE}
        COMMENT "Generating ${PROTO_NAME}.pb.cc/h"
        VERBATIM
    )
endforeach()

add_custom_target(generate_proto ALL DEPENDS ${GENERATED_SOURCES} ${GENERATED_HEADERS})

# ============ PACKAGE RELEASE ============
set(RELEASE_DIR ${CMAKE_BINARY_DIR}/release)

add_custom_target(package_release
    DEPENDS generate_proto libprotobuf
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/lib
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/include/generated
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/include/google
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/include/absl
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/src
    # Copy headers
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${GENERATED_DIR} ${RELEASE_DIR}/include/generated
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${protobuf_SOURCE_DIR}/src/google ${RELEASE_DIR}/include/google
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${abseil_SOURCE_DIR}/absl ${RELEASE_DIR}/include/absl
    COMMENT "Packaging release files"
)

# Copy generated .cc to src
foreach(GEN_CC ${GENERATED_SOURCES})
    get_filename_component(CC_NAME ${GEN_CC} NAME)
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${GEN_CC} ${RELEASE_DIR}/src/${CC_NAME}
    )
endforeach()

# Copy built libraries
if(WIN32)
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:libprotobuf> ${RELEASE_DIR}/lib/
        COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:utf8_validity> ${RELEASE_DIR}/lib/
        COMMENT "Copying protobuf libraries"
    )
else()
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:protobuf::libprotobuf> ${RELEASE_DIR}/lib/
        COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:utf8_validity> ${RELEASE_DIR}/lib/
        COMMENT "Copying protobuf libraries"
    )
endif()

# Copy abseil libraries (create a combined archive)
if(APPLE)
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND libtool -static -o ${RELEASE_DIR}/lib/libabsl_all.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/base/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/strings/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/hash/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/container/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/time/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/numeric/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/synchronization/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/debugging/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/status/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/types/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/crc/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/log/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/flags/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/random/libabsl_*.a ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/profiling/libabsl_*.a
        COMMENT "Creating monolithic abseil library"
    )
elseif(WIN32)
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND cmd /c "dir /s /b \"${CMAKE_BINARY_DIR}\\_deps\\abseil-build\\absl\\*.lib\" > \"${RELEASE_DIR}\\lib\\abseil_libs.txt\""
        COMMAND lib.exe /OUT:${RELEASE_DIR}/lib/absl_all.lib @${RELEASE_DIR}/lib/abseil_libs.txt
        COMMAND ${CMAKE_COMMAND} -E remove ${RELEASE_DIR}/lib/abseil_libs.txt
        COMMENT "Creating monolithic abseil library"
    )
else()
    # Note: Must NOT use -T (thin archive) - it creates references that break when shipped
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/lib/absl_tmp
        COMMAND bash -c "cd ${RELEASE_DIR}/lib/absl_tmp && for lib in ${CMAKE_BINARY_DIR}/_deps/abseil-build/absl/*/libabsl_*.a; do ar -x \"$$lib\"; done && ar -rcs ../libabsl_all.a *.o && cd .. && rm -rf absl_tmp"
        COMMENT "Creating monolithic abseil library"
    )
endif()
