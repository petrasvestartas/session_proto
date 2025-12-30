cmake_minimum_required(VERSION 3.22)
project(SessionProto VERSION 1.0 LANGUAGES CXX)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

include(FetchContent)
set(FETCHCONTENT_QUIET OFF)

set(DEPS_DIR "${CMAKE_BINARY_DIR}/deps")
set(GENERATED_DIR "${CMAKE_BINARY_DIR}/generated")
file(MAKE_DIRECTORY ${DEPS_DIR})
file(MAKE_DIRECTORY ${GENERATED_DIR})

set(PROTOC_VERSION "29.0")

# ============ DOWNLOAD PRE-BUILT PROTOC ============
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

# ============ DOWNLOAD PROTOBUF HEADERS (no build) ============
if(NOT EXISTS "${DEPS_DIR}/protobuf/src/google")
    message(STATUS "Downloading protobuf headers...")
    FetchContent_Declare(protobuf
        URL https://github.com/protocolbuffers/protobuf/releases/download/v${PROTOC_VERSION}/protobuf-${PROTOC_VERSION}.zip
        SOURCE_DIR ${DEPS_DIR}/protobuf
    )
    FetchContent_Populate(protobuf)
else()
    message(STATUS "protobuf headers already present")
endif()

# ============ DOWNLOAD ABSEIL HEADERS (no build) ============
if(NOT EXISTS "${DEPS_DIR}/abseil/absl")
    message(STATUS "Downloading abseil headers...")
    FetchContent_Declare(abseil
        GIT_REPOSITORY https://github.com/abseil/abseil-cpp.git
        GIT_TAG 20240722.0
        GIT_SHALLOW TRUE
        SOURCE_DIR ${DEPS_DIR}/abseil
    )
    FetchContent_Populate(abseil)
else()
    message(STATUS "abseil headers already present")
endif()

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
add_custom_target(package_release
    DEPENDS generate_proto
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/include/generated
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/include/google
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/include/absl
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/src
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${GENERATED_DIR} ${CMAKE_BINARY_DIR}/release/include/generated
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${DEPS_DIR}/protobuf/src/google ${CMAKE_BINARY_DIR}/release/include/google
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${DEPS_DIR}/abseil/absl ${CMAKE_BINARY_DIR}/release/include/absl
    COMMENT "Packaging release files"
)

foreach(GEN_CC ${GENERATED_SOURCES})
    get_filename_component(CC_NAME ${GEN_CC} NAME)
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${GEN_CC} ${CMAKE_BINARY_DIR}/release/src/${CC_NAME}
    )
endforeach()
