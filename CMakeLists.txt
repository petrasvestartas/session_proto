cmake_minimum_required(VERSION 3.22)
project(SessionProto VERSION 1.0 LANGUAGES CXX)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED True)

include(ExternalProject)

set(INSTALL_DIR "${CMAKE_BINARY_DIR}/install")
set(GENERATED_DIR "${CMAKE_BINARY_DIR}/generated")
file(MAKE_DIRECTORY ${GENERATED_DIR})

# Build Abseil
ExternalProject_Add(abseil_external
    GIT_REPOSITORY https://github.com/abseil/abseil-cpp.git
    GIT_TAG 20240722.0
    UPDATE_COMMAND ""
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR}/abseil
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DABSL_ENABLE_INSTALL=ON
        -DABSL_PROPAGATE_CXX_STD=ON
        -DCMAKE_CXX_STANDARD=17
        -DBUILD_SHARED_LIBS=OFF
)

# Create monolithic Abseil archive
if(APPLE)
    add_custom_command(
        OUTPUT ${INSTALL_DIR}/abseil/lib/libabsl_all.a
        COMMAND sh -c "libtool -static -o ${INSTALL_DIR}/abseil/lib/libabsl_all.a ${INSTALL_DIR}/abseil/lib/libabsl_*.a"
        DEPENDS abseil_external
    )
elseif(WIN32)
    add_custom_command(
        OUTPUT ${INSTALL_DIR}/abseil/lib/absl_all.lib
        COMMAND powershell -Command "lib.exe /OUT:${INSTALL_DIR}/abseil/lib/absl_all.lib ${INSTALL_DIR}/abseil/lib/absl_*.lib"
        DEPENDS abseil_external
    )
else()
    add_custom_command(
        OUTPUT ${INSTALL_DIR}/abseil/lib/libabsl_all.a
        COMMAND sh -c "ar -rcT ${INSTALL_DIR}/abseil/lib/libabsl_all.a ${INSTALL_DIR}/abseil/lib/libabsl_*.a"
        DEPENDS abseil_external
    )
endif()

if(WIN32)
    add_custom_target(abseil_monolith DEPENDS ${INSTALL_DIR}/abseil/lib/absl_all.lib)
else()
    add_custom_target(abseil_monolith DEPENDS ${INSTALL_DIR}/abseil/lib/libabsl_all.a)
endif()

# Build protobuf
ExternalProject_Add(protobuf_external
    DEPENDS abseil_external
    GIT_REPOSITORY https://github.com/protocolbuffers/protobuf.git
    GIT_TAG v29.0
    UPDATE_COMMAND ""
    CMAKE_ARGS
        -Dprotobuf_BUILD_TESTS=OFF
        -Dprotobuf_BUILD_EXAMPLES=OFF
        -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR}/protobuf
        -Dprotobuf_BUILD_SHARED_LIBS=OFF
        -Dprotobuf_ABSL_PROVIDER=package
        -DCMAKE_PREFIX_PATH=${INSTALL_DIR}/abseil
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
)

# Generate C++ from .proto files
file(GLOB PROTO_FILES "${CMAKE_CURRENT_SOURCE_DIR}/*.proto")

add_custom_target(generate_proto ALL DEPENDS protobuf_external)

foreach(PROTO_FILE ${PROTO_FILES})
    get_filename_component(PROTO_NAME ${PROTO_FILE} NAME_WE)
    add_custom_command(TARGET generate_proto POST_BUILD
        COMMAND ${INSTALL_DIR}/protobuf/bin/protoc
            --cpp_out=${GENERATED_DIR}
            --proto_path=${CMAKE_CURRENT_SOURCE_DIR}
            ${PROTO_FILE}
    )
endforeach()

# Package release
add_custom_target(package_release
    DEPENDS abseil_monolith protobuf_external generate_proto
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/lib
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/include/generated
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/include/protobuf
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/include/abseil
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/release/src
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${GENERATED_DIR} ${CMAKE_BINARY_DIR}/release/include/generated
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${INSTALL_DIR}/protobuf/include ${CMAKE_BINARY_DIR}/release/include/protobuf
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${INSTALL_DIR}/abseil/include ${CMAKE_BINARY_DIR}/release/include/abseil
)

if(APPLE OR UNIX)
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND sh -c "cp ${GENERATED_DIR}/*.cc ${CMAKE_BINARY_DIR}/release/src/"
        COMMAND sh -c "cp ${INSTALL_DIR}/protobuf/lib/*.a ${CMAKE_BINARY_DIR}/release/lib/"
        COMMAND sh -c "cp ${INSTALL_DIR}/abseil/lib/libabsl_all.a ${CMAKE_BINARY_DIR}/release/lib/"
    )
else()
    add_custom_command(TARGET package_release POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy ${INSTALL_DIR}/protobuf/lib/*.lib ${CMAKE_BINARY_DIR}/release/lib/
        COMMAND ${CMAKE_COMMAND} -E copy ${INSTALL_DIR}/abseil/lib/absl_all.lib ${CMAKE_BINARY_DIR}/release/lib/
    )
endif()
