cmake_minimum_required(VERSION 2.8.11)

# TODO:
#   - Would prefer renaming config.h.in
#   - uriparser::liburiparser or uriparser::uriparser
#   - preprocess.sh does not work on Windows
#   - Changes to liburiparser.pc.in and Doxyfile means it no longer build
#     with autotools
#   - CI will need updating as well

project(uriparser)

include(CheckSymbolExists)
include(CTest)
include(GNUInstallDirs)

# Check if uriparser is the top-level project (standalone), or a subproject
set(uriparser_standalone FALSE)
get_directory_property(uriparser_parent_directory PARENT_DIRECTORY)
if(uriparser_parent_directory STREQUAL "")
  set(uriparser_standalone TRUE)
endif()
unset(uriparser_parent_directory)

#
# Options
#

# URIPARSER_BUILD_SHARED controls if uriparser libraries are built as shared
# or static
#
# It defaults to the value of BUILD_SHARED_LIBS if set, and in most cases
# that should be used instead. The purpose of URIPARSER_BUILD_SHARED is to allow
# overriding it when built as a subproject.
set(uriparser_shared_default OFF)
if(DEFINED BUILD_SHARED_LIBS)
  set(uriparser_shared_default ${BUILD_SHARED_LIBS})
endif()
option(URIPARSER_BUILD_SHARED "Build uriparser with shared libraries" ${uriparser_shared_default})
unset(uriparser_shared_default)

# URIPARSER_BUILD_TESTING controls if uriparser adds testing support
#
# When built standalone, it defaults to the value of BUILD_TESTING if set.
# An optional prefix for the test names can be set with URIPARSER_TEST_PREFIX.
set(uriparser_testing_default ON)
if(uriparser_standalone)
  if(DEFINED BUILD_TESTING)
    set(uriparser_testing_default ${BUILD_TESTING})
  endif()
else()
  set(uriparser_testing_default OFF)
endif()
option(URIPARSER_BUILD_TESTING "Add testing support" ${uriparser_testing_default})
unset(uriparser_testing_default)

mark_as_advanced(URIPARSER_TEST_PREFIX)

# URIPARSER_BUILD_INSTALL controls if uriparser adds an install target to the
# build
#
# When built standalone or as a shared library subproject, the default is ON,
# and for static library subproject the default is OFF.
if(uriparser_standalone OR URIPARSER_BUILD_SHARED)
  option(URIPARSER_BUILD_INSTALL "Add uriparser install target" ON)
else()
  option(URIPARSER_BUILD_INSTALL "Add uriparser install target" OFF)
endif()

# URIPARSER_BUILD_DOC controls if uriparser adds a doc target to the build
#
# When built standalone the default is ON, and for subproject the default is
# OFF.
if(uriparser_standalone)
  option(URIPARSER_BUILD_DOC "Add uriparser doc target" ON)
else()
  option(URIPARSER_BUILD_DOC "Add uriparser doc target" OFF)
endif()

# URIPARSER_DEFAULT_RELEASE enables changing empty build type to Release
#
# Make based single-configuration generators default to an empty build type,
# which might be surprising, but could be useful if you want full control over
# compiler and linker flags. When URIPARSER_DEFAULT_RELEASE is ON, change an
# empty default build type to Release.
option(URIPARSER_DEFAULT_RELEASE "If CMAKE_BUILD_TYPE is empty, default to Release" ON)

if(uriparser_standalone AND URIPARSER_DEFAULT_RELEASE)
  if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "CMAKE_BUILD_TYPE empty, defaulting to Release")
    set(CMAKE_BUILD_TYPE Release)
  endif()
endif()

# URIPARSER_ENABLE_ANSI and URIPARSER_ENABLE_UNICODE enable respectively the
# char and wchar_t interfaces
#
# They can be used to disable one of these if only the other is needed.
option(URIPARSER_ENABLE_ANSI "Enable char interface (ANSI)" ON)
option(URIPARSER_ENABLE_UNICODE "Enable wchar_t interface (UNICODE)" ON)

if(NOT URIPARSER_ENABLE_ANSI AND NOT URIPARSER_ENABLE_UNICODE)
  message(FATAL_ERROR "Must enable at least one of URIPARSER_ENABLE_ANSI and URIPARSER_ENABLE_UNICODE")
endif()
if(URIPARSER_BUILD_TESTING AND NOT URIPARSER_ENABLE_ANSI OR NOT URIPARSER_ENABLE_UNICODE)
  message(FATAL_ERROR "Tests require both URIPARSER_ENABLE_ANSI and URIPARSER_ENABLE_UNICODE")
endif()

#
# Library version
#
set(URIPARSER_VERSION_MAJOR 0)
set(URIPARSER_VERSION_MINOR 9)
set(URIPARSER_VERSION_PATCH 0)
set(URIPARSER_VERSION ${URIPARSER_VERSION_MAJOR}.${URIPARSER_VERSION_MINOR}.${URIPARSER_VERSION_PATCH})

set(URIPARSER_SOVERSION_CURRENT 1)
set(URIPARSER_SOVERSION_REVISION 23)
set(URIPARSER_SOVERSION_AGE 0)
math(EXPR URIPARSER_SOVERSION "${URIPARSER_SOVERSION_CURRENT} - ${URIPARSER_SOVERSION_AGE}")

if(URIPARSER_BUILD_SHARED)
  set(uriparser_library_type SHARED)
else()
  set(uriparser_library_type STATIC)
endif()

#
# config.h
#
check_symbol_exists(wprintf wchar.h HAVE_WPRINTF)
check_symbol_exists(reallocarray stdlib.h HAVE_REALLOCARRAY)

configure_file(src/config.h.in config.h)

#
# liburiparser
#
add_library(liburiparser ${uriparser_library_type}
  src/UriCommon.c
  src/UriCommon.h
  src/UriCompare.c
  src/UriEscape.c
  src/UriFile.c
  src/UriIp4.c
  src/UriIp4Base.c
  src/UriIp4Base.h
  src/UriMemory.c
  src/UriMemory.h
  src/UriNormalize.c
  src/UriNormalizeBase.c
  src/UriNormalizeBase.h
  src/UriParse.c
  src/UriParseBase.c
  src/UriParseBase.h
  src/UriQuery.c
  src/UriRecompose.c
  src/UriResolve.c
  src/UriShorten.c
)

if(NOT URIPARSER_ENABLE_ANSI)
  target_compile_definitions(liburiparser PUBLIC URI_NO_ANSI)
endif()

if(NOT URIPARSER_ENABLE_UNICODE)
  target_compile_definitions(liburiparser PUBLIC URI_NO_UNICODE)
endif()

target_compile_definitions(liburiparser PRIVATE HAVE_CONFIG_H)

target_include_directories(liburiparser
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  PRIVATE
    ${CMAKE_CURRENT_BINARY_DIR}
)

set_target_properties(liburiparser PROPERTIES
  OUTPUT_NAME uriparser
  VERSION ${URIPARSER_SOVERSION}.${URIPARSER_SOVERSION_AGE}.${URIPARSER_SOVERSION_REVISION}
  SOVERSION ${URIPARSER_SOVERSION}
)

if(WIN32)
  target_link_libraries(liburiparser ws2_32)
endif()

# MSVC does not export symbols by default when building a DLL, this is a
# workaround for recent versions of CMake
if(MSVC AND URIPARSER_BUILD_SHARED)
  if(CMAKE_VERSION VERSION_LESS 3.4)
    message(WARNING "Automatic export of all symbols to DLL not supported until CMake 3.4")
  else()
    set_target_properties(liburiparser PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)
  endif()
endif()

#
# uriparse
#
add_executable(uriparse tool/uriparse.c)
target_link_libraries(uriparse liburiparser)

#
# Doc
#
if(URIPARSER_BUILD_DOC)
  add_subdirectory(doc)
endif()

#
# Tests
#
if(URIPARSER_BUILD_TESTING)
  find_package(GTest 1.8.1 REQUIRED)

  add_executable(test_uriparser
    test/FourSuite.cpp
    test/MemoryManagerSuite.cpp
    test/VersionSuite.cpp
    test/test.cpp
  )

  if(CMAKE_VERSION VERSION_LESS 3.5)
    target_include_directories(test_uriparser PRIVATE ${GTEST_INCLUDE_DIRS})
    target_link_libraries(test_uriparser ${GTEST_BOTH_LIBRARIES})
  else()
    target_link_libraries(test_uriparser GTest::GTest GTest::Main)
  endif()

  target_compile_definitions(test_uriparser PRIVATE HAVE_CONFIG_H)
  target_include_directories(test_uriparser PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

  target_link_libraries(test_uriparser liburiparser)

  if(MSVC)
    target_compile_definitions(test_uriparser PRIVATE _CRT_SECURE_NO_WARNINGS)
  endif()

  add_test("${URIPARSER_TEST_PREFIX}uriparser" test_uriparser)
endif()

# Create aliases
#
# Makes targets available to projects using uriparser as a subproject using
# the same names as in the config file package.
if(NOT CMAKE_VERSION VERSION_LESS 3.0)
  add_library(uriparser::liburiparser ALIAS liburiparser)
  add_executable(uriparser::uriparse ALIAS uriparse)
endif()

#
# Install
#
if(URIPARSER_BUILD_INSTALL)
  # Install binaries, libraries, and headers
  install(TARGETS liburiparser uriparse
    EXPORT uriparserTargets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )
  install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    FILES_MATCHING
    PATTERN *.h
  )

  # Install config file package
  #
  # This allows CMake based projects to use the installed libraries with
  # find_package(uriparser).
  if(NOT CMAKE_VERSION VERSION_LESS 3.0)
    include(CMakePackageConfigHelpers)
    write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/uriparserConfigVersion.cmake
      VERSION ${URIPARSER_VERSION}
      COMPATIBILITY SameMajorVersion
    )
    # Since we have no dependencies, use export file directly as config file
    install(EXPORT uriparserTargets
      DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/uriparser
      NAMESPACE uriparser::
      FILE uriparserConfig.cmake
    )
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/uriparserConfigVersion.cmake
      DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/uriparser
    )
  endif()

  # Install pkg-config file
  configure_file(liburiparser.pc.in liburiparser.pc @ONLY)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/liburiparser.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
  )
endif()
