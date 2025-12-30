cmake_minimum_required(VERSION 3.22)
project(SessionProto VERSION 1.0 LANGUAGES CXX)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# MSVC: Use static runtime for all targets
if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

include(FetchContent)
set(FETCHCONTENT_QUIET OFF)

set(GENERATED_DIR "${CMAKE_BINARY_DIR}/generated")
file(MAKE_DIRECTORY ${GENERATED_DIR})

# Abseil configuration
set(ABSL_PROPAGATE_CXX_STD ON CACHE BOOL "" FORCE)
set(ABSL_ENABLE_INSTALL OFF CACHE BOOL "" FORCE)
set(BUILD_TESTING OFF CACHE BOOL "" FORCE)
if(MSVC)
    set(ABSL_MSVC_STATIC_RUNTIME ON CACHE BOOL "" FORCE)
endif()

# Protobuf configuration
set(protobuf_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_CONFORMANCE OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_LIBPROTOC ON CACHE BOOL "" FORCE)
set(protobuf_ABSL_PROVIDER "module" CACHE STRING "" FORCE)
set(protobuf_WITH_ZLIB OFF CACHE BOOL "" FORCE)
set(utf8_range_ENABLE_TESTS OFF CACHE BOOL "" FORCE)
set(utf8_range_ENABLE_INSTALL OFF CACHE BOOL "" FORCE)
if(MSVC)
    set(protobuf_MSVC_STATIC_RUNTIME ON CACHE BOOL "" FORCE)
endif()

# Fetch Abseil
FetchContent_Declare(abseil
    GIT_REPOSITORY https://github.com/abseil/abseil-cpp.git
    GIT_TAG 20240722.0
    GIT_SHALLOW TRUE
)

# Fetch Protobuf
FetchContent_Declare(protobuf
    GIT_REPOSITORY https://github.com/protocolbuffers/protobuf.git
    GIT_TAG v29.0
    GIT_SHALLOW TRUE
)

FetchContent_MakeAvailable(abseil protobuf)

# Generate C++ from .proto files
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
        COMMAND protobuf::protoc
            --cpp_out=${GENERATED_DIR}
            --proto_path=${CMAKE_CURRENT_SOURCE_DIR}
            ${PROTO_FILE}
        DEPENDS protobuf::protoc ${PROTO_FILE}
        COMMENT "Generating ${PROTO_NAME}.pb.cc/h"
        VERBATIM
    )
endforeach()

add_custom_target(generate_proto ALL DEPENDS ${GENERATED_SOURCES} ${GENERATED_HEADERS})

# Package release - collect headers and generated code
add_custom_target(package_release
    DEPENDS generate_proto
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/include/generated
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/include/google/protobuf
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/include/absl
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/src
    # Copy generated headers
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${GENERATED_DIR} ${CMAKE_BINARY_DIR}/release/include/generated
    # Copy protobuf headers from FetchContent source
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${protobuf_SOURCE_DIR}/src/google ${CMAKE_BINARY_DIR}/release/include/google
    # Copy abseil headers from FetchContent source
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${abseil_SOURCE_DIR}/absl ${CMAKE_BINARY_DIR}/release/include/absl
    COMMENT "Packaging release files"
)

# Copy generated .cc files to release/src
foreach(GEN_CC ${GENERATED_SOURCES})
    get_filename_component(CC_NAME ${GEN_CC} NAME)
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${GEN_CC} ${CMAKE_BINARY_DIR}/release/src/${CC_NAME}
    )
endforeach()
