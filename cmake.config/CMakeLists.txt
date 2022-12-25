# When building with Visual Studio on Windows, a default configuration for try_compile is Debug
# With CMAKE_TRY_COMPILE_CONFIGURATION, we can choose a specific build configuration.
set(CMAKE_TRY_COMPILE_CONFIGURATION ${CMAKE_BUILD_TYPE})

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