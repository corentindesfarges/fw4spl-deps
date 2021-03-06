# From https://github.com/davidsansome/python-cmake-buildsystem

cmake_minimum_required(VERSION 2.6)

set(LIBPYTHON_MAJOR 2)
set(LIBPYTHON_MINOR 7)
set(LIBPYTHON_RELEASE 0)
set(LIBPYTHON_NAME python${LIBPYTHON_MAJOR}${LIBPYTHON_MINOR})
set(LIBPYTHON_VERSION ${LIBPYTHON_MAJOR}.${LIBPYTHON_MINOR})
set(LIBPYTHON ${LIBPYTHON_NAME})

cmake_policy(SET CMP0011 NEW)

# Include helper functions
include(cmake/Assembler.cmake)
include(cmake/ParseArguments.cmake)
include(cmake/Extensions.cmake)

# Options
option(ENABLE_SHARED "Build a shared libpython library" ON)
option(ENABLE_STATIC "Build a static libpython library" OFF)
option(USE_LIB64 "Search for dependencies and install to prefix/lib64 instead of prefix/lib" OFF)

if(NOT ENABLE_SHARED AND NOT ENABLE_STATIC)
    message(FATAL_ERROR "One or both of ENABLE_SHARED or ENABLE_STATIC must be set")
endif(NOT ENABLE_SHARED AND NOT ENABLE_STATIC)

if(USE_LIB64)
    set_property(GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS ON)
    set(LIBDIR "lib64")
else(USE_LIB64)
    set(LIBDIR "lib")
endif(USE_LIB64)

set(EXTRA_PYTHONPATH "" CACHE STRING
    "A colon (:) separated list of extra paths to add to the PYTHONPATH")

# Find any dependencies
find_package(BZip2)
find_package(Curses)
find_package(EXPAT)
find_package(OpenSSL)
find_package(TCL)
find_package(Threads)
find_package(ZLIB REQUIRED)

find_path(DB_INCLUDE_PATH db.h)
find_library(DB_LIBRARY NAMES db-4.8)

find_path(GDBM_INCLUDE_PATH gdbm-ndbm.h)
find_library(GDBM_LIBRARY gdbm)
find_library(GDBM_COMPAT_LIBRARY gdbm_compat)

find_path(READLINE_INCLUDE_PATH readline/readline.h)
find_library(READLINE_LIBRARY readline)

find_path(SQLITE3_INCLUDE_PATH sqlite3.h)
find_library(SQLITE3_LIBRARY sqlite3)

# Set include directories
include_directories(${CMAKE_SOURCE_DIR})
include_directories(${CMAKE_SOURCE_DIR}/cmake)
include_directories(${CMAKE_SOURCE_DIR}/Include)
include_directories(${CMAKE_SOURCE_DIR}/Python)

if(UNIX)
    configure_file(${CMAKE_SOURCE_DIR}/cmake/config-unix/pyconfig.h.in
                   ${CMAKE_BINARY_DIR}/cmake/config-unix/pyconfig.h)
    include_directories(${CMAKE_BINARY_DIR}/cmake/config-unix)
elseif(WIN32)
    include_directories(${CMAKE_SOURCE_DIR}/PC)
endif(UNIX)

# Set cflags used by all components
#set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-strict-aliasing -fwrapv -Wall -Wstrict-prototypes")

if(NOT WIN32)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
endif(NOT WIN32)

# Useful additional variables that extensions can use.
if(UNIX AND NOT APPLE)
    set(LINUX ON)
else(UNIX AND NOT APPLE)
    set(LINUX OFF)
endif(UNIX AND NOT APPLE)

set(PYTHONHOME "${LIBDIR}/${LIBPYTHON}")

if(WIN32)
    set(DYNLOAD "${LIBDIR}/${LIBPYTHON}/")
else(WIN32)
    set(DYNLOAD "${LIBDIR}/${LIBPYTHON}/lib-dynload")
endif(WIN32)

add_subdirectory(cmake/libpython)

# Add the other subdirectories
add_subdirectory(cmake/pgen)
add_subdirectory(cmake/python)
add_subdirectory(cmake/include)
add_subdirectory(cmake/lib)

# Add extension modules
set(builtin_extensions "" CACHE INTERNAL "" FORCE)
set(builtin_source "" CACHE INTERNAL "" FORCE)
set(builtin_link_libraries "" CACHE INTERNAL "" FORCE)
set(builtin_includedirs "" CACHE INTERNAL "" FORCE)
set(builtin_definitions "" CACHE INTERNAL "" FORCE)
set(extensions_enabled "" CACHE INTERNAL "" FORCE)
set(extensions_disabled "" CACHE INTERNAL "" FORCE)
add_subdirectory(cmake/extensions)
show_extension_summary()

# Install some configuration information
configure_file(cmake/makefile-variables.in
               ${CMAKE_BINARY_DIR}/makefile-variables)
install(FILES ${CMAKE_BINARY_DIR}/makefile-variables
        DESTINATION ${PYTHONHOME}/config/
        RENAME Makefile)
if(UNIX)
    install(FILES ${CMAKE_BINARY_DIR}/cmake/config-unix/pyconfig.h
            DESTINATION include/)
elseif(WIN32)
    install(FILES PC/pyconfig.h
            DESTINATION include/python2.7/)
endif(UNIX)
