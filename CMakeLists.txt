##
# Main CMake project file
# Contains global options and definitions
##

cmake_minimum_required(VERSION 2.8)

project(colobot C CXX)

##
# Project version
##
set(COLOBOT_VERSION_CODENAME "Gold")
set(COLOBOT_VERSION_MAJOR    0)
set(COLOBOT_VERSION_MINOR    1)
set(COLOBOT_VERSION_REVISION 5)

# Used on official releases
set(COLOBOT_VERSION_RELEASE_CODENAME "-alpha")
# Used on unreleased, development builds
#set(COLOBOT_VERSION_UNRELEASED "+alpha")

# Append git characteristics to version
if(DEFINED COLOBOT_VERSION_UNRELEASED)
    if(EXISTS "${CMAKE_SOURCE_DIR}/.git")
        find_package(Git)
        execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --abbrev-ref HEAD
                        WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
                        OUTPUT_VARIABLE GIT_BRANCH
                        OUTPUT_STRIP_TRAILING_WHITESPACE)
        execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
                        WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
                        OUTPUT_VARIABLE GIT_REVISION
                        OUTPUT_STRIP_TRAILING_WHITESPACE)
        set(COLOBOT_VERSION_UNRELEASED "${COLOBOT_VERSION_UNRELEASED}-git-${GIT_BRANCH}~r${GIT_REVISION}")
        set(COLOBOT_VERSION_DISPLAY "git-${GIT_BRANCH}~r${GIT_REVISION}")
    else()
        set(COLOBOT_VERSION_DISPLAY "${COLOBOT_VERSION_CODENAME}-${COLOBOT_VERSION_UNRELEASED}")
    endif()
else()
    set(COLOBOT_VERSION_DISPLAY "${COLOBOT_VERSION_MAJOR}.${COLOBOT_VERSION_MINOR}.${COLOBOT_VERSION_REVISION}${COLOBOT_VERSION_RELEASE_CODENAME}")
endif()

set(COLOBOT_VERSION_FULL "${COLOBOT_VERSION_MAJOR}.${COLOBOT_VERSION_MINOR}.${COLOBOT_VERSION_REVISION}${COLOBOT_VERSION_UNRELEASED}${COLOBOT_VERSION_RELEASE_CODENAME}")
message(STATUS "Building Colobot \"${COLOBOT_VERSION_CODENAME}\" (${COLOBOT_VERSION_FULL})")


##
# Platform detection and some related checks
##
if("${CMAKE_SYSTEM_NAME}" MATCHES "Windows")
    message(STATUS "Build for Windows system")
    set(PLATFORM_WINDOWS 1)
    set(PLATFORM_GNU     0)
    set(PLATFORM_LINUX   0)
    set(PLATFORM_MACOSX  0)
    set(PLATFORM_OTHER   0)

    # Platform-dependent implementation of system.h
    set(SYSTEM_CPP_MODULE "system_windows.cpp")
elseif("${CMAKE_SYSTEM_NAME}" MATCHES "Linux")
    message(STATUS "Build for Linux system")
    set(PLATFORM_WINDOWS 0)
    set(PLATFORM_LINUX   1)
    set(PLATFORM_GNU     1)
    set(PLATFORM_MACOSX  0)
    set(PLATFORM_OTHER   0)

    # Platform-dependent implementation of system.h
    set(SYSTEM_CPP_MODULE "system_linux.cpp")
elseif("${CMAKE_SYSTEM_NAME}" MATCHES "kFreeBSD" OR "${CMAKE_SYSTEM_NAME}" STREQUAL "GNU")
    message(STATUS "Build for kFreeBSD system")
    set(PLATFORM_WINDOWS 0)
    set(PLATFORM_LINUX   0)
    set(PLATFORM_GNU     1)
    set(PLATFORM_MACOSX  0)
    set(PLATFORM_OTHER   0)

    # Platform-dependent implementation of system.h
    set(SYSTEM_CPP_MODULE "system_other.cpp")
elseif("${CMAKE_SYSTEM_NAME}" MATCHES "Darwin")
    message(STATUS "Build for Mac OSX system")
    set(PLATFORM_WINDOWS 0)
    set(PLATFORM_LINUX   0)
    set(PLATFORM_GNU     0)
    set(PLATFORM_MACOSX  1)
    set(PLATFORM_OTHER   0)

    # Platform-dependent implementation of system.h
    set(SYSTEM_CPP_MODULE "system_macosx.cpp")

    set(USE_SDL_MAIN 1) # fixes SDL_main
else()
    message(STATUS "Build for other system")
    set(PLATFORM_WINDOWS 0)
    set(PLATFORM_LINUX   0)
    set(PLATFORM_GNU     0)
    set(PLATFORM_MACOSX  0)
    set(PLATFORM_OTHER   1)

    # Platform-dependent implementation of system.h
    set(SYSTEM_CPP_MODULE "system_other.cpp")
endif()


##
# Build options
##

# Build targets should be placed in the root build directory
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})

# Include cmake directory with some additional scripts
set(CMAKE_MODULE_PATH "${colobot_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})

# Compiler detection
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    execute_process(
        COMMAND ${CMAKE_CXX_COMPILER} -dumpversion OUTPUT_VARIABLE GCC_VERSION)
    if (GCC_VERSION VERSION_GREATER 4.7 OR GCC_VERSION VERSION_EQUAL 4.7)
        message(STATUS "Detected GCC version 4.7+")
        set(CXX11_FLAGS "-std=gnu++11")
        add_definitions(-DOVERRIDE=override)
    elseif (GCC_VERSION VERSION_GREATER 4.6 OR GCC_VERSION VERSION_EQUAL 4.6)
        message(STATUS "Detected GCC version 4.6+")
        set(CXX11_FLAGS "-std=c++0x")
        add_definitions(-DOVERRIDE=)
    else()
        message(FATAL_ERROR "${PROJECT_NAME} requires GCC 4.6 or greater.")
    endif()
    set(NORMAL_CXX_FLAGS "-Wall -Wold-style-cast")
    set(RELEASE_CXX_FLAGS "-O2")
    set(DEBUG_CXX_FLAGS "-g -O0")
    set(TEST_CXX_FLAGS "-pthread")
    add_definitions(-DNOEXCEPT=noexcept)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    message(STATUS "Detected Clang compiler")
    set(CXX11_FLAGS "-std=c++11")
    set(NORMAL_CXX_FLAGS "-Wall -Wold-style-cast")
    set(RELEASE_CXX_FLAGS "-O2")
    set(DEBUG_CXX_FLAGS "-g -O0")
    set(TEST_CXX_FLAGS "-pthread")
    add_definitions(-DOVERRIDE=override)
    add_definitions(-DNOEXCEPT=noexcept)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    message(STATUS "Detected MSVC compiler")
    set(CXX11_FLAGS "")
    set(NORMAL_CXX_FLAGS "/wd\"4244\" /wd\"4309\" /wd\"4800\" /wd\"4996\"") # disable some useless warnings
    set(RELEASE_CXX_FLAGS "")
    set(DEBUG_CXX_FLAGS "")
    set(TEST_CXX_FLAGS "")
    add_definitions(-DNOEXCEPT=)
    add_definitions(-DOVERRIDE=override)
else()
    message(FATAL_ERROR "Your C++ compiler doesn't seem to be supported.")
endif()


# Global compile flags
# These are specific to GCC/MinGW/clang; for other compilers, change as necessary
# The flags are used throughout src/ and test/ subdirs

# Special flags for boost
add_definitions(-DBOOST_NO_SCOPED_ENUMS -DBOOST_NO_CXX11_SCOPED_ENUMS)

set(COLOBOT_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${NORMAL_CXX_FLAGS} ${CXX11_FLAGS}")
set(COLOBOT_CXX_FLAGS_RELEASE "${RELEASE_CXX_FLAGS}")
set(COLOBOT_CXX_FLAGS_DEBUG "${DEBUG_CXX_FLAGS}")

# Flags for gtest
set(COLOBOT_GTEST_CXX_FLAGS "${TEST_CXX_FLAGS}")


# Asserts can be enabled/disabled regardless of build type
option(ASSERTS "Enable assert()s" ON)

# Development build can be enabled/disabled regardless of build type
option(DEV_BUILD "Enable development build (enables some debugging tools, local setting paths, etc.)" OFF)

# Portable build - load all data from current directory
option(PORTABLE "Portable build" OFF)

# Building tests can be enabled/disabled
option(TESTS "Build tests" OFF)

# Building tool programs can be enabled/disabled
option(TOOLS "Build tool programs" OFF)

# CBot can also be a static library
option(CBOT_STATIC "Build CBot as static libary" OFF)

# Generate desktop files, manpage, etc.
option(DESKTOP "Generate desktop files, manpages, etc" ON)

# Doxygen docs are optional for installation
option(INSTALL_DOCS "Install Doxygen-generated documentation" OFF)

# Build OpenAL sound support
option(OPENAL_SOUND "Build OpenAL sound support" ON)

# This is useful in case you want to use static boost libraries
option(BOOST_STATIC "Link with static boost libraries" OFF)

# This is useful on Windows, if linking against standard GLEW dll fails
option(GLEW_STATIC "Link statically with GLEW" OFF)

# Sometimes helpful if there is a different version of gtest installed on system vs bundled
option(FORCE_BUNDLED_GTEST "Force the use of bundled gtest" OFF)

# Default build type if not given is debug
if(NOT DEFINED CMAKE_BUILD_TYPE)
    message(STATUS "Build type not specified - assuming debug")
    set(CMAKE_BUILD_TYPE debug)
endif()

# Warn about development build
if(DEV_BUILD)
    message("Building with development extensions")
endif()


##
# Searching for packages
##

find_package(OpenGL 1.4 REQUIRED)
find_package(SDL 1.2.10 REQUIRED)
find_package(SDL_image 1.2 REQUIRED)
find_package(SDL_ttf 2.0 REQUIRED)
find_package(PNG 1.2 REQUIRED)
find_package(Gettext REQUIRED)
find_package(PhysFS REQUIRED)

set(Boost_USE_STATIC_LIBS ${BOOST_STATIC})
set(Boost_USE_MULTITHREADED ON)
set(Boost_USE_STATIC_RUNTIME OFF)

set(Boost_ADDITIONALVERSION "1.51" "1.51.0")
find_package(Boost COMPONENTS system filesystem regex REQUIRED)

find_package(GLEW REQUIRED)

if (OPENAL_SOUND)
    find_package(OpenAL REQUIRED)
    include_directories(${OPENAL_INCLUDE_DIR})
    find_package(LibSndFile REQUIRED)
endif()


if(NOT ASSERTS)
    add_definitions(-DNDEBUG)
endif()

if(TESTS)
    add_definitions(-DTESTS -DTEST_VIRTUAL=virtual)
else()
    add_definitions(-DTEST_VIRTUAL=)
endif()

if(DEV_BUILD)
    add_definitions(-DDEV_BUILD)
endif()

##
# Additional settings to use when cross-compiling with MXE (http://mxe.cc/)
##

include("${colobot_SOURCE_DIR}/cmake/mxe.cmake")

##
# Additional settings for MSYS
##
include("${colobot_SOURCE_DIR}/cmake/msys.cmake")


##
# MSVC specific settings
##
set(WINGETOPT 0)
if(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    message(STATUS "Adding MSVC-specific options")

    set(CBOT_STATIC 1) # only this works for some reason
    set(USE_SDL_MAIN 1) # fixes SDL_main
    set(WINGETOPT 1) # use wingetopt library
endif()

##
# Clipboard support needs X11 libraries
##
if(PLATFORM_GNU)
    find_package(X11 REQUIRED)
endif()


##
# Clipboard support
##
set(CLIPBOARD_INCLUDE_DIR ${colobot_SOURCE_DIR}/lib/clipboard/include)
add_subdirectory(lib/clipboard)


##
# Localename
##
set(LOCALENAME_INCLUDE_DIR ${colobot_SOURCE_DIR}/lib/localename)
add_subdirectory(lib/localename)


##
# Wingetopt
##
if(WINGETOPT)
    set(WINGETOPT_INCLUDE_DIR ${colobot_SOURCE_DIR}/lib/wingetopt/src)
    add_subdirectory(lib/wingetopt)
endif()


##
# Doxygen docs
##

find_package(Doxygen)

if(DOXYGEN_FOUND AND DOXYGEN_DOT_FOUND)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)
    add_custom_target(doc
                      ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
                      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                      COMMENT "Generating API documentation with Doxygen" VERBATIM)
endif()


##
# Targets
##

# Installation paths defined before compiling sources
if(PLATFORM_WINDOWS)
    if(MXE)
        # We need to use STRING because PATH doesn't accept relative paths
        set(COLOBOT_INSTALL_BIN_DIR ./ CACHE STRING "Colobot binary directory")
        set(COLOBOT_INSTALL_LIB_DIR ./ CACHE STRING "Colobot libraries directory")
        set(COLOBOT_INSTALL_DATA_DIR ./data CACHE STRING "Colobot shared data directory")
        set(COLOBOT_INSTALL_I18N_DIR ./lang CACHE STRING "Colobot translations directory")
        set(COLOBOT_INSTALL_DOC_DIR ./doc CACHE STRING "Colobot documentation directory")
    else()
        set(COLOBOT_INSTALL_BIN_DIR ${CMAKE_INSTALL_PREFIX}/ CACHE PATH "Colobot binary directory")
        set(COLOBOT_INSTALL_LIB_DIR ${CMAKE_INSTALL_PREFIX}/ CACHE PATH "Colobot libraries directory")
        set(COLOBOT_INSTALL_DATA_DIR ${CMAKE_INSTALL_PREFIX}/data CACHE PATH "Colobot shared data directory")
        set(COLOBOT_INSTALL_I18N_DIR ${CMAKE_INSTALL_PREFIX}/lang CACHE PATH "Colobot translations directory")
        set(COLOBOT_INSTALL_DOC_DIR ${CMAKE_INSTALL_PREFIX}/doc CACHE PATH "Colobot documentation directory")
    endif()
elseif(PLATFORM_MACOSX)
    set(COLOBOT_INSTALL_BIN_DIR ../MacOS CACHE STRING "Colobot binary directory")
    set(COLOBOT_INSTALL_LIB_DIR ../MacOS CACHE STRING "Colobot libraries directory")
    set(COLOBOT_INSTALL_DATA_DIR . CACHE STRING "Colobot shared data directory")
    set(COLOBOT_INSTALL_I18N_DIR i18n CACHE SRING "Colobot translations directory")
    set(COLOBOT_INSTALL_DOC_DIR doc CACHE STRING "Colobot documentation directory")
else()
    set(COLOBOT_INSTALL_BIN_DIR ${CMAKE_INSTALL_PREFIX}/games CACHE PATH "Colobot binary directory")
    set(COLOBOT_INSTALL_LIB_DIR ${CMAKE_INSTALL_PREFIX}/lib/colobot CACHE PATH "Colobot libraries directory")
    set(COLOBOT_INSTALL_DATA_DIR ${CMAKE_INSTALL_PREFIX}/share/games/colobot CACHE PATH "Colobot shared data directory")
    set(COLOBOT_INSTALL_I18N_DIR ${CMAKE_INSTALL_PREFIX}/share/locale CACHE PATH "Colobot translations directory")
    set(COLOBOT_INSTALL_DOC_DIR ${CMAKE_INSTALL_PREFIX}/share/doc/colobot CACHE PATH "Colobot documentation directory")
endif()

# Subdirectory with sources
add_subdirectory(src)

add_subdirectory(po)

if(DESKTOP)
    add_subdirectory(desktop)
endif()

if(TESTS)
    # Google Test library
    find_path(GTEST_SRC_DIR NAMES src/gtest.cc src/gtest-all.cc PATHS /usr/src PATH_SUFFIXES gtest)
    find_path(GTEST_INCLUDE_DIR gtest/gtest.h PATHS /usr/include)
    if(NOT(FORCE_BUNDLED_GTEST) AND GTEST_SRC_DIR AND GTEST_INCLUDE_DIR)
        message(STATUS "Using system gtest library in ${GTEST_SRC_DIR}")
    else()
        message(STATUS "Using bundled gtest library")
        set(GTEST_SRC_DIR ${colobot_SOURCE_DIR}/lib/gtest)
        set(GTEST_INCLUDE_DIR ${colobot_SOURCE_DIR}/lib/gtest/include)
    endif()

    add_subdirectory(${GTEST_SRC_DIR} lib/gtest)

    # Hippomocks library
    set(HIPPOMOCKS_INCLUDE_DIR ${colobot_SOURCE_DIR}/lib/hippomocks)

    # Tests targets
    enable_testing()
    add_subdirectory(test)

endif()


##
# Installation
##

# Data: check if the submodule handles its own installation
if(EXISTS "${CMAKE_SOURCE_DIR}/data/CMakeLists.txt")
    message(STATUS "Data directory will install itself.")
    add_subdirectory(data)
else()
    message(WARNING "Data directory is not available; make sure colobot-data is installed in ${COLOBOT_INSTALL_DATA_DIR}.")
endif()

# Documentation
if(INSTALL_DOCS AND DOXYGEN_FOUND AND DOXYGEN_DOT_FOUND)
    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc/ DESTINATION ${COLOBOT_INSTALL_DOC_DIR} OPTIONAL)
endif()
