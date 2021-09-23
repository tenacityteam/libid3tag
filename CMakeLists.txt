cmake_minimum_required(VERSION 3.1.0)
project(id3tag VERSION 0.16.1)

option(BUILD_SHARED_LIBS "Build dynamic library" ON)

include(GNUInstallDirs)

#
# Build
#
add_custom_command(
    COMMAND sed -n -f genre.dat.sed genre.dat.in > ${CMAKE_CURRENT_BINARY_DIR}/genre.dat
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/genre.dat
    MAIN_DEPENDENCY genre.dat.in
    DEPENDS genre.dat.sed
)

find_program(GPERF NAMES gperf)

add_custom_command(
    COMMAND ${GPERF} -tCcTonD --output-file=${CMAKE_CURRENT_BINARY_DIR}/compat.c -K id -N id3_compat_lookup -s -3 -k '*' ${CMAKE_CURRENT_SOURCE_DIR}/compat.gperf
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/compat.c
    MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/compat.gperf
)

add_custom_command(
    COMMAND ${GPERF} -tCcTonD --output-file=${CMAKE_CURRENT_BINARY_DIR}/frametype.c -K id -N id3_frametype_lookup -s -3 -k '*' ${CMAKE_CURRENT_SOURCE_DIR}/frametype.gperf
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/frametype.c
    MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/frametype.gperf
)

add_library(id3tag
    ${CMAKE_CURRENT_BINARY_DIR}/compat.c
    crc.c
    debug.c
    field.c
    file.c
    frame.c
    ${CMAKE_CURRENT_BINARY_DIR}/frametype.c
    genre.c
    ${CMAKE_CURRENT_BINARY_DIR}/genre.dat
    latin1.c
    parse.c
    render.c
    tag.c
    ucs4.c
    utf16.c
    utf8.c
    util.c
    version.c
)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/id3tag.h.in ${CMAKE_CURRENT_BINARY_DIR}/id3tag.h @ONLY)
target_include_directories(id3tag PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
if(WIN32 AND BUILD_SHARED_LIBS)
    set_target_properties(id3tag PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

set_target_properties(id3tag PROPERTIES
  VERSION ${CMAKE_PROJECT_VERSION}
  SOVERSION ${CMAKE_PROJECT_VERSION}
)

include(CheckIncludeFile)
check_include_file(sys/stat.h HAVE_SYS_STAT_H)
if(HAVE_SYS_STAT_H)
  target_compile_definitions(id3tag PRIVATE HAVE_SYS_STAT_H)
endif()

check_include_file(unistd.h HAVE_UNISTD_H)
if(HAVE_UNISTD_H)
  target_compile_definitions(id3tag PRIVATE HAVE_UNISTD_H)
endif()

check_include_file(assert.h HAVE_ASSERT_H)
if(HAVE_ASSERT_H)
  target_compile_definitions(id3tag PRIVATE HAVE_ASSERT_H)
endif()

include(CheckFunctionExists)
check_function_exists(ftruncate HAVE_FTRUNCATE)
if(HAVE_FTRUNCATE)
  target_compile_definitions(id3tag PRIVATE HAVE_FTRUNCATE)
endif()

find_package(ZLIB REQUIRED)
target_link_libraries(id3tag PUBLIC ZLIB::ZLIB)

#
# Installation
#

include(CMakePackageConfigHelpers)

# Library files
install(TARGETS id3tag
  EXPORT id3tagTargets
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
  INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

# Header files
install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/id3tag.h"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

# pkgconfig
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/packaging/id3tag.pc.in
  ${CMAKE_CURRENT_BINARY_DIR}/packaging/id3tag.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/packaging/id3tag.pc DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")

# CMake config
set(ID3TAG_INSTALL_CMAKEDIR "${CMAKE_INSTALL_LIBDIR}/cmake/id3tag")
install(
  EXPORT id3tagTargets
  FILE id3tagTargets.cmake
  NAMESPACE id3tag::
  DESTINATION "${ID3TAG_INSTALL_CMAKEDIR}"
)
configure_package_config_file(packaging/id3tagConfig.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/packaging/id3tagConfig.cmake"
  INSTALL_DESTINATION "${ID3TAG_INSTALL_CMAKEDIR}"
)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/packaging/id3tagConfigVersion.cmake"
  VERSION "${CMAKE_PROJECT_VERSION}"
  COMPATIBILITY SameMajorVersion
)
install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/packaging/id3tagConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/packaging/id3tagConfigVersion.cmake"
  DESTINATION "${ID3TAG_INSTALL_CMAKEDIR}"
)
