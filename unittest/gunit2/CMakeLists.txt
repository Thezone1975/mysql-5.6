# Copyright (c) 2010, 2016, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301  USA

# We want release-1.8.0.zip in order to build these unit tests.
# If you have already downloaded it,
# invoke cmake with -DWITH_GMOCK=/path/to/release-1.8.0.zip
#                or -DWITH_GMOCK=/path/to
#
# Alternatively, set an environment variable
# export WITH_GMOCK=/path/to/release-1.8.0.zip
#
# You can also do cmake -DENABLE_DOWNLOADS=1
# and we will download it from https://github.com/google/googletest/archive/
#
# Either way: we will unpack the zip, compile gmock-all.cc and gtest-all.cc
# and link them into the executables.

# Disable googletest for Solaris Studio < 12.4 since it doesn't compile.
IF(CMAKE_C_COMPILER_ID MATCHES "SunPro")
  IF(${CC_MINOR_VERSION} LESS 13)
    MESSAGE(WARNING "Googletest requires Solaris Studio 12.4 or newer.")
    UNSET(GMOCK_FOUND)
    UNSET(GMOCK_FOUND CACHE)
    RETURN()
  ENDIF()
ENDIF()

# Default location for where to download and build gmock/gtest.
IF(NOT DOWNLOAD_ROOT)
  SET(DOWNLOAD_ROOT ${CMAKE_SOURCE_DIR}/source_downloads)
ENDIF()

# We want googletest version 1.8, which also contains googlemock.
SET(GMOCK_PACKAGE_NAME "release-1.8.0")

IF (DEFINED ENV{WITH_GMOCK} AND NOT DEFINED WITH_GMOCK)
  FILE(TO_CMAKE_PATH "$ENV{WITH_GMOCK}" WITH_GMOCK)
ENDIF()

IF(LOCAL_GMOCK_ZIP
   AND NOT ${LOCAL_GMOCK_ZIP} MATCHES ".*${GMOCK_PACKAGE_NAME}\\.zip")
 SET(LOCAL_GMOCK_ZIP 0)
ENDIF()

IF (WITH_GMOCK)
  ## Did we get a full path name, including file name?
  IF (${WITH_GMOCK} MATCHES ".*\\.zip")
    GET_FILENAME_COMPONENT(GMOCK_DIR ${WITH_GMOCK} PATH)
    GET_FILENAME_COMPONENT(GMOCK_ZIP ${WITH_GMOCK} NAME)
    FIND_FILE(LOCAL_GMOCK_ZIP
              NAMES ${GMOCK_ZIP}
              PATHS ${GMOCK_DIR}
              NO_DEFAULT_PATH
             )
  ELSE()
    ## Did we get a path name to the directory of the .zip file?
    ## Check for both release-x.y.z.zip and googletest-release-x.y.z.zip
    FIND_FILE(LOCAL_GMOCK_ZIP
              NAMES "${GMOCK_PACKAGE_NAME}.zip"
              "googletest-${GMOCK_PACKAGE_NAME}.zip"
              PATHS ${WITH_GMOCK}
              NO_DEFAULT_PATH
              )
    ## If WITH_GMOCK is a directory, use it for download.
    SET(DOWNLOAD_ROOT ${WITH_GMOCK})
  ENDIF()
  MESSAGE(STATUS "Local gmock zip ${LOCAL_GMOCK_ZIP}")
ENDIF()

IF(NOT EXISTS DOWNLOAD_ROOT)
  MAKE_DIRECTORY(${DOWNLOAD_ROOT})
ENDIF()

IF(NOT GMOCK_SOURCE_DIR AND NOT GTEST_SOURCE_DIR)
  SET(GMOCK_SOURCE_DIR
    ${DOWNLOAD_ROOT}/googletest-${GMOCK_PACKAGE_NAME}/googlemock)
  SET(GTEST_SOURCE_DIR
    ${DOWNLOAD_ROOT}/googletest-${GMOCK_PACKAGE_NAME}/googletest)
ENDIF()
# We may have downloaded gmock/gtest already, building in a different directory.
IF(EXISTS ${GMOCK_SOURCE_DIR} OR EXISTS ${LOCAL_GMOCK_ZIP})
  MESSAGE(STATUS "GMOCK_SOURCE_DIR:${GMOCK_SOURCE_DIR}")
  SET(GMOCK_DOWNLOADED 1 CACHE INTERNAL "")
  SET(GMOCK_FOUND 1 CACHE INTERNAL "")
# If source dir does not exist, reset dependent variables (might be set from
# before).
ELSE()
  SET(LOCAL_GMOCK_ZIP 0 CACHE INTERNAL "")
  SET(GMOCK_DOWNLOADED 0 CACHE INTERNAL "")
  SET(GMOCK_FOUND 0 CACHE INTERNAL "")
  SET(GMOCK_INCLUDE_DIRS 0 CACHE INTERNAL "")
ENDIF()


IF(LOCAL_GMOCK_ZIP AND NOT EXISTS ${GMOCK_SOURCE_DIR})
  # Unpack tarball
  EXECUTE_PROCESS(
    COMMAND ${CMAKE_COMMAND} -E tar xfz  "${LOCAL_GMOCK_ZIP}"
    WORKING_DIRECTORY "${DOWNLOAD_ROOT}"
    RESULT_VARIABLE tar_result
  )
  IF (tar_result MATCHES 0)
    SET(GMOCK_FOUND 1 CACHE INTERNAL "")
  ENDIF()
ENDIF()


OPTION(ENABLE_DOWNLOADS
  "Download and build 3rd party source code components, e.g. googletest"
  OFF)

# While experimenting, use local URL rather than google.
SET(GMOCK_TARBALL "googletest-${GMOCK_PACKAGE_NAME}.zip")
SET(GMOCK_DOWNLOAD_URL
  "https://github.com/google/googletest/archive/${GMOCK_PACKAGE_NAME}.zip"
  )

MACRO(HTTP_PROXY_HINT)
  MESSAGE(STATUS
    "If you are inside a firewall, you may need to use an https proxy: "
    "export https_proxy=http://example.com:80")
ENDMACRO()


IF(NOT GMOCK_FOUND)
  IF(NOT ENABLE_DOWNLOADS)
    # Give warning
    MESSAGE(STATUS
      "Googletest was not found. gtest-based unit tests will be disabled. "
      "You can run cmake . -DENABLE_DOWNLOADS=1 to automatically download "
      "and build required components from source.")
    HTTP_PROXY_HINT()
    RETURN()
  ENDIF()

  # Download googletest source
  IF(NOT EXISTS ${GMOCK_SOURCE_DIR})
    IF(NOT EXISTS ${DOWNLOAD_ROOT}/${GMOCK_TARBALL})
      # Download the tarball
      # Use CMake builtin download capabilities
      FILE(DOWNLOAD ${GMOCK_DOWNLOAD_URL}
        ${DOWNLOAD_ROOT}/${GMOCK_TARBALL} TIMEOUT 30 STATUS ERR)
      IF(ERR EQUAL 0)
        SET(DOWNLOAD_SUCCEEDED 1)
      ELSE()
        MESSAGE(STATUS "Download failed, error: ${ERR}")
        # A failed DOWNLOAD leaves an empty file, remove it
        FILE(REMOVE ${DOWNLOAD_ROOT}/${GMOCK_TARBALL})
      ENDIF()

      IF (DOWNLOAD_SUCCEEDED)
        MESSAGE(STATUS
          "Successfully downloaded ${GMOCK_DOWNLOAD_URL} to ${DOWNLOAD_ROOT}")
      ELSE()
        MESSAGE(STATUS
          "To enable googletest, please download ${GMOCK_DOWNLOAD_URL} "
          "to the directory ${DOWNLOAD_ROOT}")
        HTTP_PROXY_HINT()
        RETURN()
      ENDIF()
    ENDIF()

    # Unpack tarball
    EXECUTE_PROCESS(
      COMMAND ${CMAKE_COMMAND} -E tar xfz  "${DOWNLOAD_ROOT}/${GMOCK_TARBALL}"
        WORKING_DIRECTORY "${DOWNLOAD_ROOT}"
      )
  ENDIF()
  IF(EXISTS ${GMOCK_SOURCE_DIR})
    SET(GMOCK_DOWNLOADED 1 CACHE INTERNAL "")
    SET(GMOCK_FOUND 1 CACHE INTERNAL "")
  ENDIF()
ENDIF()


IF(NOT GMOCK_FOUND)
  RETURN()
ENDIF()

SET(GMOCK_INCLUDE_DIRS
  ${GMOCK_SOURCE_DIR}
  ${GMOCK_SOURCE_DIR}/include
  ${GTEST_SOURCE_DIR}
  ${GTEST_SOURCE_DIR}/include
  CACHE INTERNAL "")

# Build gmock/gtest libraries.
INCLUDE_DIRECTORIES(
  ${GMOCK_SOURCE_DIR}
  ${GMOCK_SOURCE_DIR}/include
  ${GTEST_SOURCE_DIR}
  ${GTEST_SOURCE_DIR}/include
)
ADD_LIBRARY(gmock STATIC ${GMOCK_SOURCE_DIR}/src/gmock-all.cc)
ADD_LIBRARY(gtest STATIC ${GTEST_SOURCE_DIR}/src/gtest-all.cc)
SET(GTEST_LIBRARIES gmock gtest)

INCLUDE_DIRECTORIES(
  ${CMAKE_SOURCE_DIR}/include
  ${CMAKE_SOURCE_DIR}/regex
  ${CMAKE_SOURCE_DIR}/sql
  ${CMAKE_SOURCE_DIR}/storage/example
)

# Turn off some warning flags when compiling GUnit
MY_CHECK_CXX_COMPILER_FLAG("-Wno-null-dereference" HAVE_NO_NULL)
IF(HAVE_NO_NULL)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-null-dereference")
ENDIF()
MY_CHECK_CXX_COMPILER_FLAG("-Wno-unused-local-typedefs" HAVE_NO_UNUSED_TYPEDEFS)
IF(HAVE_NO_UNUSED_TYPEDEFS)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-local-typedefs")
ENDIF()
MY_CHECK_CXX_COMPILER_FLAG("-Wno-compare-distinct-pointer-types"
                           HAVE_NO_COMPARE_DISTINCT_POINTER_TYPES)
IF(HAVE_NO_COMPARE_DISTINCT_POINTER_TYPES)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-compare-distinct-pointer-types")
ENDIF()

MY_CHECK_CXX_COMPILER_FLAG("-fno-builtin-memcmp" HAVE_NO_BUILTIN_MEMCMP)

# main-wrapper libraries (with tap-compatible option).
ADD_LIBRARY(gunit_small STATIC
  ../gunit/gunit_test_main.cc
  ../gunit/tap_event_listener.cc
  ../gunit/thread_utils.cc)
ADD_DEPENDENCIES(gunit_small GenError)
TARGET_LINK_LIBRARIES(gunit_small
  mysys mysys_ssl mytap dbug strings ${GTEST_LIBRARIES})
MESSAGE(STATUS "GTEST_LIBRARIES:${GTEST_LIBRARIES}")

# Add some defines.
ADD_DEFINITIONS(-DMYSQL_SERVER)
ADD_DEFINITIONS(-DERRMSG_DIR="${PROJECT_BINARY_DIR}/sql/share")
ADD_DEFINITIONS(-DDATA_DIR="${CMAKE_CURRENT_BINARY_DIR}")

# Add tests (link them with gunit/gmock libraries)
SET(TESTS
  ""
  )

## Merging tests into fewer executables saves *a lot* of
## link time and disk space ...
OPTION(MERGE_UNITTESTS "Merge tests into one executable" OFF)
IF (MERGE_UNITTESTS)
  SET(HHWHEELTEST ${CMAKE_CURRENT_BINARY_DIR}/hhwheeltest.cc)
  SET_SOURCE_FILES_PROPERTIES(HHWHEELTEST
    PROPERTIES GENERATED 1)

  INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})

  FILE(WRITE ${HHWHEELTEST} "// Merging small unit tests\n")
  FOREACH(test ${TESTS})
    FILE(APPEND ${HHWHEELTEST} "#include \"${test}-t.cc\"\n")
  ENDFOREACH()

  ADD_EXECUTABLE(hhwheeltest-t ${HHWHEELTEST})
  IF(WIN32)
    LIST(APPEND SRC_FILES ../../sql/nt_servc.cc)
  ENDIF()
  LIST(APPEND SRC_FILES ../../storage/example/ha_example.cc)
  SET_SOURCE_FILES_PROPERTIES(
    ../../storage/example/ha_example.cc
    PROPERTIES COMPILE_FLAGS "-DDISABLE_DTRACE"
  )

  TARGET_LINK_LIBRARIES(hhwheeltest-t
    gunit_small sqlgunitlib strings dbug regex)
  TARGET_LINK_LIBRARIES(hhwheeltest-t sql)

  # Necessary to make sure that we can use the jemalloc API calls.
  IF(NOT PREV_LINK_FLAGS)
    SET(PREV_LINK_FLAGS)
  ENDIF()

  ADD_TEST(hhwheeltest hhwheeltest-t)

ENDIF(MERGE_UNITTESTS)

  FOREACH(test ${TESTS})
    ADD_EXECUTABLE(${test}-t ${test}-t.cc)
    TARGET_LINK_LIBRARIES(${test}-t gunit_small sqlgunitlib strings dbug regex)
    IF(MERGE_UNITTESTS)
      SET_PROPERTY(TARGET ${test}-t PROPERTY EXCLUDE_FROM_ALL TRUE)
      IF(WIN32)
        SET_PROPERTY(TARGET ${test}-t PROPERTY EXCLUDE_FROM_DEFAULT_BUILD TRUE)
      ENDIF()
    ELSE()
      ADD_TEST(${test} ${test}-t)
    ENDIF()
  ENDFOREACH()

add_executable(fast_int2str-t fast_int2str_test.cc)
target_link_libraries(fast_int2str-t gtest strings)
add_test(fast_int2str fast_int2str-t)

## Most executables depend on libeay32.dll (through mysys_ssl).
COPY_OPENSSL_DLLS(copy_openssl_gunit)
