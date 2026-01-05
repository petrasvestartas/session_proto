cmake_minimum_required(VERSION 3.22)
project(SessionProto VERSION 1.0 LANGUAGES CXX)

set(CMAKE_BUILD_TYPE Release)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

include(FetchContent)
set(FETCHCONTENT_QUIET OFF)

set(PROTOC_VERSION "33.2")
set(INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install")
set(GENERATED_DIR "${CMAKE_BINARY_DIR}/generated")
file(MAKE_DIRECTORY ${GENERATED_DIR})

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

FetchContent_Declare(protoc
    URL https://github.com/protocolbuffers/protobuf/releases/download/v${PROTOC_VERSION}/protoc-${PROTOC_VERSION}-${PROTOC_PLATFORM}.zip
    SOURCE_DIR ${CMAKE_BINARY_DIR}/protoc
)
FetchContent_MakeAvailable(protoc)
set(PROTOC_BIN "${CMAKE_BINARY_DIR}/protoc/bin/protoc${CMAKE_EXECUTABLE_SUFFIX}")

# ============ FETCH AND BUILD PROTOBUF (official way) ============
set(protobuf_FORCE_FETCH_DEPENDENCIES ON CACHE BOOL "" FORCE)
set(protobuf_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_CONFORMANCE OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_PROTOC_BINARIES OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_LIBPROTOC OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
set(protobuf_WITH_ZLIB OFF CACHE BOOL "" FORCE)
set(protobuf_INSTALL ON CACHE BOOL "" FORCE)
set(ABSL_ENABLE_INSTALL ON CACHE BOOL "" FORCE)
set(ABSL_PROPAGATE_CXX_STD ON CACHE BOOL "" FORCE)
set(BUILD_TESTING OFF CACHE BOOL "" FORCE)
set(CMAKE_INSTALL_PREFIX ${INSTALL_PREFIX} CACHE PATH "" FORCE)
if(MSVC)
    set(protobuf_MSVC_STATIC_RUNTIME ON CACHE BOOL "" FORCE)
    set(ABSL_MSVC_STATIC_RUNTIME ON CACHE BOOL "" FORCE)
endif()

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

# ============ PACKAGE RELEASE (after cmake --install) ============
set(RELEASE_DIR ${CMAKE_BINARY_DIR}/release)

add_custom_target(package_release
    DEPENDS generate_proto libprotobuf
    # Run cmake --install first
    COMMAND ${CMAKE_COMMAND} --install ${CMAKE_BINARY_DIR} --prefix ${INSTALL_PREFIX}
    # Create release structure
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}/src
    # Copy installed files (lib, include from cmake --install)
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${INSTALL_PREFIX}/lib ${RELEASE_DIR}/lib
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${INSTALL_PREFIX}/include ${RELEASE_DIR}/include
    # Add generated proto files
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${GENERATED_DIR} ${RELEASE_DIR}/include/generated
    COMMAND ${CMAKE_COMMAND} -E copy ${GENERATED_SOURCES} ${RELEASE_DIR}/src/
    COMMENT "Packaging release from installed files"
)
