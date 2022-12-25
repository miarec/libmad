cmake_minimum_required (VERSION 3.8)

option(BUILD_SHARED_LIBS "Build a shared library" OFF)

if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE "Release")
   message(STATUS "Build type not specified: default is Release")
endif()

#-------------------------------------------------------------------------------
# Read version number from version.h file
#-------------------------------------------------------------------------------
file(STRINGS "version.h" VERSION)

string(REGEX MATCH "MAD_VERSION_MAJOR[ \t]+([0-9]+)" _ ${VERSION})
set(VERSION_MAJOR ${CMAKE_MATCH_1})

string(REGEX MATCH "MAD_VERSION_MINOR[ \t]+([0-9]+)" _ ${VERSION})
set(VERSION_MINOR ${CMAKE_MATCH_1})

string(REGEX MATCH "MAD_VERSION_PATCH[ \t]+([0-9]+)" _ ${VERSION})
set(VERSION_PATCH ${CMAKE_MATCH_1})

message(STATUS "Building version ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")

#-------------------------------------------------------------------------------
# Project definition
#-------------------------------------------------------------------------------
project (mad 
  VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH} 
  LANGUAGES C
)

set(PROJECT_URL "https://github.com/miarec/libmad")

add_library(${PROJECT_NAME})

set_target_properties(${PROJECT_NAME}
	PROPERTIES VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}
	SOVERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}
  POSITION_INDEPENDENT_CODE ON
)

target_sources(${PROJECT_NAME}
	PRIVATE
		bit.c
		decoder.c
		fixed.c
		frame.c
		huffman.c
		layer12.c
		layer3.c
		stream.c
		synth.c
		timer.c
		version.c
)

# Generate config.h file
add_subdirectory(cmake.config)
add_compile_definitions(HAVE_CONFIG_H)

# Location of "config.h" file
target_include_directories( ${PROJECT_NAME} 
  PRIVATE 
    "${CMAKE_CURRENT_BINARY_DIR}/cmake.config"
  )


# ---------------------------------------------------------------------
# Generate "mad.h" header file
# ---------------------------------------------------------------------
# List of headers files that will be merged into "mad.h"
set(EXPORTED_HEADERS
  ${CMAKE_CURRENT_BINARY_DIR}/cmake.config/fpm.h
	version.h 
	fixed.h 
	bit.h 
	timer.h 
	stream.h 
	frame.h
	synth.h
	decoder.h
)

set (MAD_H "${CMAKE_BINARY_DIR}/include/mad.h")
configure_file( mad.h.header.in "${MAD_H}" )

foreach( header_file ${EXPORTED_HEADERS} )
   file( READ ${header_file} HEADER_DATA )
   string( REPLACE "# include" "// # include" HEADER_DATA_REPLACED "${HEADER_DATA}")
   file( APPEND ${MAD_H} "// \"${header_file}\"\n\n${HEADER_DATA_REPLACED}\n" )
endforeach()

file( APPEND ${MAD_H} "# ifdef __cplusplus\n}\n# endif\n#endif\n" )

target_include_directories(${PROJECT_NAME}
    PRIVATE
        # where the library itself will look for its internal headers
        ${CMAKE_CURRENT_SOURCE_DIR}
    PUBLIC
        # where top-level project will look for the library's public headers
        $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/include>
        # where external projects will look for the library's public headers
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

if(WIN32 AND BUILD_SHARED_LIBS)
    set_target_properties(${PROJECT_NAME} PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()


# ----------------------------------------------------------------------
# Example application
# ----------------------------------------------------------------------

include(CMakeDependentOption)
cmake_dependent_option(EXAMPLE "Build example executable" ON "HAVE_UNISTD_H;HAVE_SYS_STAT_H;HAVE_SYS_MMAN_H" OFF)
if(EXAMPLE)
  add_executable(mad_example minimad.c)
  target_link_libraries(mad_example PRIVATE ${PROJECT_NAME})
endif()

# ----------------------------------------------------------------------
# Installation
# ----------------------------------------------------------------------
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Library files
install(TARGETS ${PROJECT_NAME}
  EXPORT madTargets
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
  INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

# Header files
install(
  FILES "${MAD_H}"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

# ----------------------------------------------------------------------
# pkgconfig
# ----------------------------------------------------------------------
if(IS_ABSOLUTE "${CMAKE_INSTALL_LIBDIR}")
  set(PKGCONFIG_LIBDIR "${CMAKE_INSTALL_LIBDIR}")
else()
  set(PKGCONFIG_LIBDIR "\${prefix}/${CMAKE_INSTALL_LIBDIR}")
endif()
if(IS_ABSOLUTE "${CMAKE_INSTALL_INCLUDEDIR}")
  set(PKGCONFIG_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}")
else()
  set(PKGCONFIG_INCLUDEDIR "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")
endif()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/packaging.cmake/libmad.pc.in
  ${CMAKE_CURRENT_BINARY_DIR}/packaging.cmake/libmad.pc @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/packaging.cmake/libmad.pc DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")

# ----------------------------------------------------------------------
# CMake config
# ----------------------------------------------------------------------
set(MAD_INSTALL_CMAKEDIR "${CMAKE_INSTALL_LIBDIR}/cmake/mad")
install(
  EXPORT madTargets
  FILE madTargets.cmake
  NAMESPACE mad::
  DESTINATION "${MAD_INSTALL_CMAKEDIR}"
)
configure_package_config_file(packaging.cmake/madConfig.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/packaging.cmake/madConfig.cmake"
  INSTALL_DESTINATION "${MAD_INSTALL_CMAKEDIR}"
)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/packaging.cmake/madConfigVersion.cmake"
  VERSION "${CMAKE_PROJECT_VERSION}"
  COMPATIBILITY SameMajorVersion
)

install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/packaging.cmake/madConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/packaging.cmake/madConfigVersion.cmake"
  DESTINATION "${MAD_INSTALL_CMAKEDIR}"
)

