cmake_minimum_required(VERSION 2.6)
project(flvmeta C)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/modules ${CMAKE_MODULE_PATH})

set(FLVMETA_VERSION "1.2.2")
set(FLVMETA_RELEASE yes)

# check whether we are building a release or a git snapshot
if(NOT FLVMETA_RELEASE)
  # check for git
  find_program(GIT "git")
  mark_as_advanced(GIT)
  # check whether we are in a git repository
  find_file(DOT_GIT ".git" ${CMAKE_SOURCE_DIR})
  mark_as_advanced(DOT_GIT)
  if(GIT AND DOT_GIT)
    # retrieve current revision
    execute_process(
      COMMAND "${GIT}" rev-parse --short HEAD
      WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
      OUTPUT_VARIABLE   GIT_RELEASE
      OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    set(FLVMETA_VERSION "${FLVMETA_VERSION}-dev+g${GIT_RELEASE}")
  endif()
endif()

# generic variables
set(PACKAGE           "flvmeta")
set(PACKAGE_NAME      ${PACKAGE})
set(PACKAGE_BUGREPORT "flvmeta-discussion@googlegroups.com")
set(PACKAGE_VERSION   "${FLVMETA_VERSION}")
set(PACKAGE_STRING    "${PACKAGE_NAME} ${PACKAGE_VERSION}")

# build options
set(
  FLVMETA_USE_SYSTEM_LIBYAML FALSE
  CACHE BOOL "Link flvmeta to the installed version of libyaml"
)

#platform tests
include(CheckFunctionExists)
include(CheckSymbolExists)
include(CheckIncludeFile)
include(CheckTypeSize)
include(TestBigEndian)

check_include_file(sys/types.h  HAVE_SYS_TYPES_H)
check_include_file(stdint.h     HAVE_STDINT_H)
check_include_file(stddef.h     HAVE_STDDEF_H)
check_include_file(inttypes.h   HAVE_INTTYPES_H)

check_type_size("double"      SIZEOF_DOUBLE)
check_type_size("float"       SIZEOF_FLOAT)
check_type_size("long double" SIZEOF_LONG_DOUBLE)
check_type_size("long"        SIZEOF_LONG)
check_type_size("long long"   SIZEOF_LONG_LONG)

# MSVC before VS 2010 did not have stdint.h
if(MSVC AND NOT HAVE_STDINT_H)
  set(int16_t  1)
  set(int32_t  1)
  set(int64_t  1)
  set(int8_t   1)
  set(uint16_t 1)
  set(uint32_t 1)
  set(uint64_t 1)
  set(uint8_t  1)
endif()

test_big_endian(IS_BIGENDIAN)
if(IS_BIGENDIAN)
  set(WORDS_BIGENDIAN 1)
endif()

# C99 isfinite support
check_symbol_exists("isfinite" math.h HAVE_ISFINITE)

# large file support
check_function_exists("fseeko" HAVE_FSEEKO)
if(HAVE_FSEEKO)
  execute_process(
    COMMAND getconf LFS_CFLAGS
    OUTPUT_VARIABLE LFS_CFLAGS
    ERROR_QUIET
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )

  if(LFS_CFLAGS)
    add_definitions(${LFS_CFLAGS})
    set(CMAKE_REQUIRED_FLAGS ${LFS_CFLAGS})
  endif()

  if(WIN32)
    add_definitions(-D_FILE_OFFSET_BITS=64)
    set(CMAKE_REQUIRED_FLAGS -D_FILE_OFFSET_BITS=64)
  endif()

  check_type_size("off_t" SIZEOF_OFF_T)
endif()

# configuration file
configure_file(config-cmake.h.in ${CMAKE_BINARY_DIR}/config.h)
include_directories(${CMAKE_BINARY_DIR})
add_definitions(-DHAVE_CONFIG_H)

# Visual C++ specific configuration
if(MSVC)
  # use static library
  set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MT")

  # C runtime deprecation in Visual C++ 2005 and later
  add_definitions(-D_CRT_SECURE_NO_DEPRECATE -D_CRT_NONSTDC_NO_DEPRECATE)
endif()

# installation
set(CPACK_PACKAGE_VENDOR "Marc Noirot <marc.noirot@gmail.com>")
set(CPACK_PACKAGE_VERSION ${FLVMETA_VERSION})
set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_SOURCE_DIR}/LICENSE.md)
set(CPACK_RESOURCE_FILE_README ${CMAKE_SOURCE_DIR}/README.md)
set(CPACK_SOURCE_IGNORE_FILES "/*build*;/CVS/;/\\\\.svn/;/\\\\.bzr/;/\\\\.hg/;/\\\\.git/;\\\\.swp$;\\\\.#;/#")
set(CPACK_SOURCE_PACKAGE_FILE_NAME "flvmeta-${PACKAGE_VERSION}-src")

if(WIN32)
  if("${CMAKE_SIZEOF_VOID_P}" STREQUAL "8")
    set(EXECUTABLE_TYPE "64")
  else()
    set(EXECUTABLE_TYPE "32")
  endif()

  set(CPACK_PACKAGE_FILE_NAME "flvmeta-${PACKAGE_VERSION}-win${EXECUTABLE_TYPE}")
  set(CPACK_GENERATOR ZIP)
  install(FILES ${CMAKE_SOURCE_DIR}/README.md       DESTINATION . RENAME Readme.txt)
  install(FILES ${CMAKE_SOURCE_DIR}/LICENSE.md      DESTINATION . RENAME License.txt)
  install(FILES ${CMAKE_SOURCE_DIR}/CHANGELOG.md    DESTINATION . RENAME Changelog.txt)
else()
  set(CPACK_PACKAGE_FILE_NAME "flvmeta-${PACKAGE_VERSION}")
endif()

include(CPack)

add_subdirectory(src)
add_subdirectory(man)

# check unit testing framework
find_package(Check)

# Enable unit testing if Check has been found
if(CHECK_FOUND)
  set(CMAKE_REQUIRED_INCLUDES ${CHECK_INCLUDE_DIRS})
  set(CMAKE_REQUIRED_LIBRARIES ${CHECK_LIBRARIES})
  set(CMAKE_REQUIRED_FLAGS -L${CHECK_LIBRARY_DIRS})
  check_symbol_exists(ck_assert_double_eq check.h HAVE_CK_ASSERT_DOUBLE_EQ)

  if(HAVE_CK_ASSERT_DOUBLE_EQ)
    # tests
    enable_testing()
    add_subdirectory(tests)
  endif()
endif()
