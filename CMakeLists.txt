cmake_minimum_required (VERSION 3.16)
project(Overture)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# The engine and audio wrapper live in sibling folders. Give them binary
# folders inside this build tree, so nothing is written into their sources.
add_subdirectory(../OALWrapper ${CMAKE_BINARY_DIR}/OALWrapper)
add_subdirectory(../HPL1Engine ${CMAKE_BINARY_DIR}/HPL1Engine)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/../dependencies/include
    ${CMAKE_CURRENT_SOURCE_DIR}/../HPL1Engine/include
    ${CMAKE_CURRENT_SOURCE_DIR}/../OALWrapper/include
)

link_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/../dependencies/lib/linux)

set(CMAKE_EXECUTABLE_SUFFIX .bin)

file(GLOB all_sources ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)

add_executable(Overture ${all_sources})

set_target_properties(Overture PROPERTIES
    INSTALL_RPATH_USE_LINK_PATH false
    OUTPUT_NAME overture)

target_link_libraries(Overture HPL OALWrapper Newton angelscript fltk GL GLU Cg CgGL ogg vorbis vorbisfile openal alut SDL SDL_ttf SDL_image)
