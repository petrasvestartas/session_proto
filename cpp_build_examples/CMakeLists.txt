cmake_minimum_required(VERSION 3.22)

#=============================================================================
# CompasPb Library - Pure protobuf code
#=============================================================================

set(COMPAS_PB_LIB CompasPb)

# Collect protobuf generated files
file(GLOB_RECURSE CompasPbFiles
    compas_pb/generated/*.pb.h
    compas_pb/generated/*.pb.cc
    compas_kumiki_pb/generated/*.pb.h
    compas_kumiki_pb/generated/*.pb.cc
)

# Collect protobuf core files
file(GLOB CompasPbCoreFiles
    compas_pb/core.h
    compas_pb/conversion.h
    compas_pb/conversion.cpp
)

# Create protobuf library
add_library(${COMPAS_PB_LIB} STATIC
    ${CompasPbFiles}
    ${CompasPbCoreFiles}
)

target_compile_options(${COMPAS_PB_LIB} PRIVATE
    /Zc:wchar_t                          # Protobuf wchar_t setting
    /W1 /wd4005 /wd4996 /wd4244 /wd4267  # Suppress protobuf warnings
)

target_include_directories(${COMPAS_PB_LIB} PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/compas_pb/
    ${CMAKE_CURRENT_SOURCE_DIR}/compas_pb/generated/
    ${CMAKE_CURRENT_SOURCE_DIR}/compas_kumiki_pb/generated/
)

target_link_libraries(${COMPAS_PB_LIB} PUBLIC
    protobuf::libprotobuf
)

#=============================================================================
# CompasArchicad Library
#=============================================================================

set(COMPAS_ARCHICAD_LIB CompasArchicad)

# Collect ArchiCAD transporter files and converter files
file(GLOB TransporterFiles Transporter/*.h Transporter/*.cpp)
# file(GLOB ConverterFiles Converter/*.h Converter/*.cpp)

add_library(${COMPAS_ARCHICAD_LIB} STATIC
    ${TransporterFiles}
    # ${ConverterFiles}
)

target_link_libraries(${COMPAS_ARCHICAD_LIB} PUBLIC
    ${COMPAS_PB_LIB}
)

# CRITICAL: Add protobuf compiler settings to match CompasPb
target_compile_options(${COMPAS_ARCHICAD_LIB} PRIVATE
    /Zc:wchar_t                          # Match protobuf wchar_t setting
    /W1 /wd4005 /wd4996 /wd4244 /wd4267  # Suppress protobuf warnings
)

target_include_directories(${COMPAS_ARCHICAD_LIB} PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    # ${CMAKE_CURRENT_SOURCE_DIR}/Converter/
    ${CMAKE_CURRENT_SOURCE_DIR}/Transporter/
    ${CMAKE_CURRENT_SOURCE_DIR}/compas_pb/
    ${CMAKE_CURRENT_SOURCE_DIR}/compas_pb/generated/
    ${CMAKE_CURRENT_SOURCE_DIR}/compas_kumiki_pb/generated/
    ${AC_API_DEVKIT_DIR}/Support/Inc
    ${LIB_INCLUDE_DIRS}
)

# ArchiCAD modules
file(GLOB ModuleFolders ${AC_API_DEVKIT_DIR}/Support/Modules/*)
target_include_directories(${COMPAS_ARCHICAD_LIB} PUBLIC ${ModuleFolders})

if(WIN32)
    file(GLOB LibFilesInFolder ${AC_API_DEVKIT_DIR}/Support/Modules/*/*/*.lib)
    target_link_libraries(${COMPAS_ARCHICAD_LIB} PUBLIC ${LibFilesInFolder})
endif()

target_compile_definitions(${COMPAS_ARCHICAD_LIB} PUBLIC
    AC${AC_VERSION}
    ACExtension
)

source_group("Transporter" FILES ${TransporterFiles})
# source_group("Converter" FILES ${ConverterFiles})

set_target_properties(${COMPAS_ARCHICAD_LIB} PROPERTIES FOLDER Libraries)
