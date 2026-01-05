# Session Proto

Protobuf schemas for Session geometry kernel.

## Build

```bash
mkdir -p build && cd build
cmake ..
cmake --build . -j8
```

Generated files in `build/generated/` (`.pb.cc` and `.pb.h`).

## Package Release

```bash
cmake --build . --target package_release -j8
```

Output in `build/release/` with:
- `lib/` - static libraries (protobuf, abseil)
- `include/` - headers (protobuf, abseil, generated)
- `src/` - generated `.pb.cc` files

## Usage in CMake

Download release and extract to `deps/session_proto/`:

```cmake
set(PROTO_DIR "${CMAKE_SOURCE_DIR}/deps/session_proto")

file(GLOB PROTO_SOURCES "${PROTO_DIR}/src/*.cc")
add_library(proto-objects OBJECT ${PROTO_SOURCES})
target_include_directories(proto-objects PUBLIC
    ${PROTO_DIR}/include/generated
    ${PROTO_DIR}/include/protobuf
    ${PROTO_DIR}/include/abseil
)

target_link_libraries(your_target PRIVATE
    $<TARGET_OBJECTS:proto-objects>
    ${PROTO_DIR}/lib/libprotobuf.a
    ${PROTO_DIR}/lib/libutf8_range.a
    ${PROTO_DIR}/lib/libutf8_validity.a
    ${PROTO_DIR}/lib/libabsl_all.a
)
```
