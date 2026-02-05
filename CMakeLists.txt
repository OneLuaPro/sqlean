# MIT License
#
# Copyright (c) 2026 The OneLuaPro project authors
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.15)
project(sqlean C)

# Set C99 standard (required for PCRE2 and some sqlean modules)
set(CMAKE_C_STANDARD 99)
set(CMAKE_VERBOSE_MAKEFILE ON)
if(MSVC)
  # /wd4244: Conversion from 'type1' to 'type2', possible loss of data
  # /wd4267: Conversion from 'size_t' to 'type', possible loss of data
  add_compile_options(/W3 /wd4068 /wd4244 /wd4267) 
endif()
include (GNUInstallDirs)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})
set(INSTALL_LIBDIR
  ${CMAKE_INSTALL_LIBDIR} CACHE PATH "Installation directory for libraries")
set(INSTALL_BINDIR
  ${CMAKE_INSTALL_BINDIR} CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDEDIR
  ${CMAKE_INSTALL_INCLUDEDIR} CACHE PATH "Installation directory for header files")
set(INSTALL_DOCDIR
  ${CMAKE_INSTALL_DOCDIR} CACHE PATH "Installation directory for documentation")
set(INSTALL_MANDIR
  ${CMAKE_INSTALL_MANDIR} CACHE PATH "Installation directory for manpages")
set(INSTALL_DATAROOTDIR
  ${CMAKE_INSTALL_DATAROOTDIR} CACHE PATH "Installation directory for data")

# --- 1. Versioning & Variables ---
set(SQLITE_RELEASE_YEAR "2026")
set(SQLITE_VERSION "3510200") # Version 3.51.2
set(SQLITE_BRANCH "3.51")

# --- 2. SQLEAN_VERSION via Git ---
find_package(Git)
if(Git_FOUND)
  execute_process(
    COMMAND ${GIT_EXECUTABLE} tag --points-at HEAD
    OUTPUT_VARIABLE GIT_TAG
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
endif()

if(NOT GIT_TAG)
  set(SQLEAN_VERSION "main")
else()
  set(SQLEAN_VERSION ${GIT_TAG})
endif()

# --- 3. Downloads via FetchContent ---
include(FetchContent)

# Proper URLs with forward slashes and variable expansion
FetchContent_Declare(
  sqlite_source 
  URL "https://www.sqlite.org/${SQLITE_RELEASE_YEAR}/sqlite-amalgamation-${SQLITE_VERSION}.zip"
  DOWNLOAD_EXTRACT_TIMESTAMP TRUE
)
FetchContent_Declare(
  xxhash_header 
  URL "https://github.com/cyan4973/xxhash/raw/v0.8.3/xxhash.h"
  DOWNLOAD_NO_EXTRACT TRUE
)
FetchContent_Declare(
  windirent_header 
  URL "https://github.com/mackyle/sqlite/raw/branch-${SQLITE_BRANCH}/ext/misc/windirent.h" 
  DOWNLOAD_NO_EXTRACT TRUE
)

message(STATUS "Fetching sqlite-amalgamation-${SQLITE_VERSION}.zip ...")
FetchContent_MakeAvailable(sqlite_source)
message(STATUS "Fetching xxhash.h ...")
FetchContent_MakeAvailable(xxhash_header)
message(STATUS "Fetching windirent.h ...")
FetchContent_MakeAvailable(windirent_header)

# --- 4. File Distribution (Setup internal structure) ---
FetchContent_GetProperties(sqlite_source SOURCE_DIR SQLITE_DIR)
FetchContent_GetProperties(xxhash_header SOURCE_DIR XXHASH_DIR)
FetchContent_GetProperties(windirent_header SOURCE_DIR WINDIRENT_DIR)

# Patch prevents windirent.h from ignoring hidden or system files - we need this for
# proper busted testing on lsqlite3 side.
execute_process(
  COMMAND ${GIT_EXECUTABLE} apply -p0 --whitespace=fix --reject
  "${CMAKE_CURRENT_SOURCE_DIR}/src/patches/windirent_no_filter.patch"
  WORKING_DIRECTORY "${WINDIRENT_DIR}"
  RESULT_VARIABLE PATCH_RESULT
)
if(NOT PATCH_RESULT EQUAL 0)
    message(FATAL_ERROR "Patching windirent.h failed or was already applied.")
endif()

# Ensure src directory exists
set(GENERATED_INCLUDE_DIR "${CMAKE_BINARY_DIR}/generated_inc")
file(MAKE_DIRECTORY "${GENERATED_INCLUDE_DIR}/crypto")

# Install necessary files to local src folder
file(INSTALL "${SQLITE_DIR}/sqlite3.h"    DESTINATION "${GENERATED_INCLUDE_DIR}")
file(INSTALL "${SQLITE_DIR}/sqlite3.c"    DESTINATION "${GENERATED_INCLUDE_DIR}")
file(INSTALL "${SQLITE_DIR}/sqlite3ext.h" DESTINATION "${GENERATED_INCLUDE_DIR}")
file(INSTALL "${WINDIRENT_DIR}/windirent.h" DESTINATION "${GENERATED_INCLUDE_DIR}")

# Correct naming for xxhash as expected by sqlean
file(INSTALL "${XXHASH_DIR}/xxhash.h" 
  DESTINATION "${GENERATED_INCLUDE_DIR}/crypto"
  RENAME "xxhash.impl.h"
)

# --- 5. Compiler Flags for Static Building ---
add_definitions(-DSQLEAN_VERSION="${SQLEAN_VERSION}")
add_definitions(-D_CRT_SECURE_NO_WARNINGS)
add_definitions(-DSQLITE_THREADSAFE=1)
add_definitions(-DSQLITE_ENABLE_FTS4)
add_definitions(-DSQLITE_ENABLE_FTS5)
add_definitions(-DSQLITE_ENABLE_RTREE)
add_definitions(-DSQLITE_ENABLE_MATH_FUNCTIONS)

# IMPORTANT: SQLITE_CORE is required when linking SQLite statically 
# to avoid __declspec(dllexport/dllimport) issues in MSVC
add_definitions(-DSQLITE_CORE)

include_directories(src)
include_directories("${GENERATED_INCLUDE_DIR}")

# --- 6. Source Collection ---
# We collect all .c files from all modules to bake them into one .lib
file(GLOB_RECURSE ALL_SRC 
  "src/crypto/*.c" "src/define/*.c" "src/fileio/*.c" "src/fuzzy/*.c" 
  "src/math/*.c" "src/regexp/*.c" "src/regexp/pcre2/*.c" "src/stats/*.c" 
  "src/text/*.c" "src/text/*/*.c" "src/time/*.c" "src/unicode/*.c" 
  "src/uuid/*.c" "src/vsv/*.c"
)

# --- 7. The Static Library Target ---
# This creates sqlean.lib
add_library(sqlean STATIC
  "${GENERATED_INCLUDE_DIR}/sqlite3.c" # Include SQLite itself
  src/sqlite3-sqlean.c                 # Include sqlean entry point
  ${ALL_SRC}                           # Include all modules
)

# MSVC Forced Include for PCRE2 constants
target_compile_options(sqlean PRIVATE "/FI${CMAKE_CURRENT_SOURCE_DIR}/src/regexp/constants.h")

# --- 8. Install Target ---
# Install the static library (.lib)
install(TARGETS sqlean
    ARCHIVE DESTINATION lib
)
# Install necessary headers for development
install(FILES 
    "${GENERATED_INCLUDE_DIR}/sqlite3.h"
    "${GENERATED_INCLUDE_DIR}/sqlite3ext.h"    
    DESTINATION include
)
# Optional: Install sqlean specific entry point header if you created one
# install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/sqlite3-sqlean.h" DESTINATION include)
