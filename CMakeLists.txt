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

target_compile_definitions(${PROJECT_NAME} 
   PRIVATE
      HAVE_CONFIG_H
)
target_include_directories( ${PROJECT_NAME} 
  PRIVATE 
    "${CMAKE_CURRENT_BINARY_DIR}"
    # "${CMAKE_BINARY_DIR}/include" 
  )

#
# Compile definitions
#
option(OPT_ACCURACY "Favour accuracy" ON)
option(OPT_SPEED "Favour speed" OFF)
option(OPT_SSO "Enable a fast subband synthesis approximation optimization" OFF)

option(ASO "Enable CPU Architecture Specific Optimizations (x86, ARM, and MIPS only)" ON)
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  message(STATUS "Using 64 bit fixed point math")
  option(FPM_64BIT "64 bit fixed point math" ON)
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86" OR "i386")
  message(STATUS "Using x86 fixed point math")
  option (FPM_INTEL "x86 fixed point math" ON)
  if(ASO)
    target_compile_definitions(${PROJECT_NAME} PRIVATE ASO_ZEROCHECK)
  endif()
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES ".*(arm|ARM).*")
  message(STATUS "Using ARM fixed point math")
  option (FPM_ARM "ARM fixed point math" ON)
  if(ASO)
    enable_language(ASM)
    target_compile_definitions(${PROJECT_NAME} PRIVATE ASO_INTERLEAVE1 ASO_IMDCT)
    target_sources(${PROJECT_NAME} PRIVATE imdct_l_arm.S)
  endif()
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES ".*(mips|MIPS).*")
  message(STATUS "Using MIPS fixed point math")
  option(FPM_MIPS "MIPS fixed point math" ON)
  if(ASO)
    target_compile_definitions(${PROJECT_NAME} PRIVATE ASO_INTERLEAVE2 ASO_ZEROCHECK)
  endif()
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES ".*(sparc|SPARC).*")
  message(STATUS "Using SPARC fixed point math")
  option(FPM_SPARC "SPARC fixed point math" ON)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES ".(ppc|PPC|powerpc).*")
  message(STATUS "Using PowerPC fixed point math")
  option(FPM_PPC "PowerPC fixed point math" ON)
else()
  message(WARNING "Target CPU architecture not detected. Fixed-point math will yield limited accuracy.")
  option(FPM_DEFAULT "Generic fixed-point math" ON)
endif()


include(CheckTypeSize)
check_type_size("int" SIZEOF_INT BUILTIN_TYPES_ONLY LANGUAGE C)
check_type_size("long" SIZEOF_LONG BUILTIN_TYPES_ONLY LANGUAGE C)
check_type_size("long long" SIZEOF_LONG_LONG BUILTIN_TYPES_ONLY LANGUAGE C)


include(CheckIncludeFile)
check_include_file(assert.h HAVE_ASSERT_H)
check_include_file(dlfcn.h HAVE_DLFCN_H)
check_include_file(errno.h HAVE_ERRNO_H)
check_include_file(fcntl HAVE_FCNTL)
check_include_file(fcntl.h HAVE_FCNTL_H)
check_include_file(inttypes.h HAVE_INTTYPES_H)
check_include_file(limits.h HAVE_LIMITS_H)

option(MADD16_ASM "Enable if your MIPS CPU supports a 2-operand MADD16 instruction." OFF)
if(MADD16_ASM)
  option(HAVE_MADD16_ASM "MIPS CPU supports a 2-operand MADD16 instruction" ON)
endif()

option(MADD_ASM "Enable if your MIPS CPU supports a 2-operand MADD instruction." OFF)
if(MADD_ASM)
  option(HAVE_MADD_ASM "MIPS CPU supports a 2-operand MADD instruction" ON)
endif()

check_include_file(memory.h HAVE_MEMORY_H)
check_include_file(stdint.h HAVE_STDINT_H)
check_include_file(stdlib.h HAVE_STDLIB_H)
check_include_file(strings.h HAVE_STRINGS_H)
check_include_file(string.h HAVE_STRING_H)
check_include_file(sys/stat.h HAVE_SYS_STAT_H)
check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(sys/wait.h HAVE_SYS_WAIT_H)
check_include_file(unistd.h HAVE_UNISTD_H)
check_include_file(sys/mman.h HAVE_SYS_MMAN_H)

include(CheckFunctionExists)
check_function_exists(fork HAVE_FORK)
check_function_exists(pipe HAVE_PIPE)
check_function_exists(waitpid HAVE_WAITPID)
check_function_exists(ftruncate HAVE_FTRUNCATE)


# ---------------------------------------------------------------------
# Generate "fpm.h" header file
# ---------------------------------------------------------------------
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/fpm.h.in ${CMAKE_CURRENT_BINARY_DIR}/fpm.h @ONLY)

# ---------------------------------------------------------------------
# Generate "config.h" header file
# ---------------------------------------------------------------------
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h @ONLY)

# ---------------------------------------------------------------------
# Generate "mad.h" header file
# ---------------------------------------------------------------------
# List of headers files that will be merged into "mad.h"
set(EXPORTED_HEADERS
  ${CMAKE_CURRENT_BINARY_DIR}/fpm.h
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

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/packaging/libmad.pc.in
  ${CMAKE_CURRENT_BINARY_DIR}/packaging/libmad.pc @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/packaging/libmad.pc DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")

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
configure_package_config_file(packaging/madConfig.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/packaging/madConfig.cmake"
  INSTALL_DESTINATION "${MAD_INSTALL_CMAKEDIR}"
)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/packaging/madConfigVersion.cmake"
  VERSION "${CMAKE_PROJECT_VERSION}"
  COMPATIBILITY SameMajorVersion
)

install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/packaging/madConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/packaging/madConfigVersion.cmake"
  DESTINATION "${MAD_INSTALL_CMAKEDIR}"
)

