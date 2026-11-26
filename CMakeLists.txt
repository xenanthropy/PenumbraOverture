cmake_minimum_required (VERSION 3.16)
project(Overture)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# The engine and audio wrapper live in sibling folders. Give them binary
# folders inside this build tree, so nothing is written into their sources.
add_subdirectory(../OALWrapper ${CMAKE_BINARY_DIR}/OALWrapper)
add_subdirectory(../HPL1Engine ${CMAKE_BINARY_DIR}/HPL1Engine)

set(CMAKE_EXECUTABLE_SUFFIX .bin)

file(GLOB all_sources ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)

add_executable(Overture ${all_sources})

set_target_properties(Overture PROPERTIES
    INSTALL_RPATH_USE_LINK_PATH false
    OUTPUT_NAME overture)

# HPL brings in everything else (OALWrapper, SDL, OpenGL, Cg, Newton,
# AngelScript) through its own link interface.
target_link_libraries(Overture HPL)
