# Session Proto

Protobuf schemas for Session geometry kernel.

## Linking in CMake

Download release and extract to `deps/session_proto/`, then:

```cmake
set(PROTO_DIR "${CMAKE_SOURCE_DIR}/deps/session_proto")

file(GLOB PROTO_SOURCES "${PROTO_DIR}/src/*.cc")
add_library(proto-objects OBJECT ${PROTO_SOURCES})
target_include_directories(proto-objects PUBLIC
    ${PROTO_DIR}/include/generated
    ${PROTO_DIR}/include/protobuf
    ${PROTO_DIR}/include/abseil
)

# Link to your target
target_link_libraries(your_target PRIVATE
    $<TARGET_OBJECTS:proto-objects>
    ${PROTO_DIR}/lib/libprotobuf.a
    ${PROTO_DIR}/lib/libutf8_range.a
    ${PROTO_DIR}/lib/libutf8_validity.a
    ${PROTO_DIR}/lib/libabsl_all.a
)
```

## Local Build

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --target package_release -j8
```

Output in `build/release/`.
