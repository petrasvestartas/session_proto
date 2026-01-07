cmake_minimum_required(VERSION 3.22)
project(SessionProto VERSION 1.0 LANGUAGES CXX)

set(CMAKE_BUILD_TYPE Release)
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

include(FetchContent)
set(FETCHCONTENT_QUIET OFF)

set(PROTOC_VERSION "33.2")
set(GENERATED_DIR "${CMAKE_BINARY_DIR}/generated")
file(MAKE_DIRECTORY ${GENERATED_DIR})

# ============ DOWNLOAD PROTOC (must match library version) ============
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

FetchContent_Declare(protoc
    URL https://github.com/protocolbuffers/protobuf/releases/download/v${PROTOC_VERSION}/protoc-${PROTOC_VERSION}-${PROTOC_PLATFORM}.zip
    SOURCE_DIR ${CMAKE_BINARY_DIR}/protoc
)
FetchContent_MakeAvailable(protoc)
set(PROTOC_BIN "${CMAKE_BINARY_DIR}/protoc/bin/protoc${CMAKE_EXECUTABLE_SUFFIX}")
message(STATUS "Using protoc v${PROTOC_VERSION}: ${PROTOC_BIN}")

# ============ FETCH AND BUILD PROTOBUF (official way) ============
set(protobuf_FORCE_FETCH_DEPENDENCIES ON CACHE BOOL "" FORCE)
set(protobuf_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_CONFORMANCE OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_PROTOC_BINARIES OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_LIBPROTOC OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
set(protobuf_WITH_ZLIB OFF CACHE BOOL "" FORCE)
set(protobuf_INSTALL OFF CACHE BOOL "" FORCE)
set(ABSL_ENABLE_INSTALL OFF CACHE BOOL "" FORCE)
set(ABSL_PROPAGATE_CXX_STD ON CACHE BOOL "" FORCE)
set(BUILD_TESTING OFF CACHE BOOL "" FORCE)
set(ABSL_BUILD_TESTING OFF CACHE BOOL "" FORCE)

FetchContent_Declare(protobuf
    GIT_REPOSITORY https://github.com/protocolbuffers/protobuf.git
    GIT_TAG v${PROTOC_VERSION}
    GIT_SHALLOW TRUE
)
FetchContent_MakeAvailable(protobuf)

# ============ GENERATE PROTO FILES ============
file(GLOB PROTO_FILES "${CMAKE_CURRENT_SOURCE_DIR}/*.proto")

foreach(PROTO_FILE ${PROTO_FILES})
    get_filename_component(PROTO_NAME ${PROTO_FILE} NAME_WE)
    list(APPEND GENERATED_SOURCES "${GENERATED_DIR}/${PROTO_NAME}.pb.cc")
    list(APPEND GENERATED_HEADERS "${GENERATED_DIR}/${PROTO_NAME}.pb.h")

    add_custom_command(
        OUTPUT "${GENERATED_DIR}/${PROTO_NAME}.pb.cc" "${GENERATED_DIR}/${PROTO_NAME}.pb.h"
        COMMAND ${PROTOC_BIN} --cpp_out=${GENERATED_DIR} --proto_path=${CMAKE_CURRENT_SOURCE_DIR} ${PROTO_FILE}
        DEPENDS ${PROTO_FILE}
        COMMENT "Generating ${PROTO_NAME}.pb.cc/h"
    )
endforeach()

add_custom_target(generate_proto ALL DEPENDS ${GENERATED_SOURCES})

# ============ PACKAGE RELEASE ============
set(RELEASE_DIR ${CMAKE_BINARY_DIR}/release)
set(PROTOBUF_SRC ${CMAKE_BINARY_DIR}/_deps/protobuf-src)
set(ABSL_SRC ${CMAKE_BINARY_DIR}/_deps/absl-src)

add_custom_target(package_release
    DEPENDS generate_proto libprotobuf
    # Create release structure
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/lib
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/src
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/include/generated
    # Copy generated proto files
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${GENERATED_DIR} ${RELEASE_DIR}/include/generated
    COMMAND ${CMAKE_COMMAND} -E copy ${GENERATED_SOURCES} ${RELEASE_DIR}/src/
    # Copy protobuf headers
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${PROTOBUF_SRC}/src/google ${RELEASE_DIR}/include/google
    # Copy abseil headers
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${ABSL_SRC}/absl ${RELEASE_DIR}/include/absl
    # Copy utf8_range headers
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${PROTOBUF_SRC}/third_party/utf8_range ${RELEASE_DIR}/include/utf8_range
    COMMENT "Packaging release"
)

# Copy static libraries (platform-specific paths)
if(MSVC)
    file(WRITE ${CMAKE_BINARY_DIR}/copy_libs.ps1
"Get-ChildItem -Path '${CMAKE_BINARY_DIR}/_deps/protobuf-build' -Recurse -Filter '*.lib' | Where-Object { $_.FullName -match 'Release' } | Copy-Item -Destination '${RELEASE_DIR}/lib/' -Force
Get-ChildItem -Path '${CMAKE_BINARY_DIR}/_deps/absl-build' -Recurse -Filter '*.lib' | Where-Object { $_.FullName -match 'Release' } | Copy-Item -Destination '${RELEASE_DIR}/lib/' -Force
")
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND powershell -ExecutionPolicy Bypass -File ${CMAKE_BINARY_DIR}/copy_libs.ps1
        COMMENT "Copying Windows libraries (flattened)"
    )
else()
    file(WRITE ${CMAKE_BINARY_DIR}/copy_libs.sh
"#!/bin/sh
find ${CMAKE_BINARY_DIR}/_deps/protobuf-build -name '*.a' -exec cp {} ${RELEASE_DIR}/lib/ \\;
find ${CMAKE_BINARY_DIR}/_deps/absl-build -name '*.a' -exec cp {} ${RELEASE_DIR}/lib/ \\;
")
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND chmod +x ${CMAKE_BINARY_DIR}/copy_libs.sh
        COMMAND ${CMAKE_BINARY_DIR}/copy_libs.sh
        COMMENT "Copying Unix libraries"
    )
endif()
