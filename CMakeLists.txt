# Set minimum CMake required version for this project.
cmake_minimum_required(VERSION 3.0 FATAL_ERROR)

# Define a C++ project.
project(RtAudio LANGUAGES CXX)

# Check for Jack (any OS)
find_library(JACK_LIB jack)
find_package(PkgConfig)
pkg_check_modules(jack jack)
if(JACK_LIB OR jack_FOUND)
  set(HAVE_JACK TRUE)
endif()

# Check for Pulse (any OS)
pkg_check_modules(pulse libpulse-simple)

# Check for known non-Linux unix-likes
if (CMAKE_SYSTEM_NAME MATCHES "kNetBSD.*|NetBSD.*")
  message(STATUS "NetBSD detected, using OSS")
  set(xBSD ON)
elseif(UNIX AND NOT APPLE)
  set(LINUX ON)
endif()

# Necessary for Windows
if(WIN32)
  set(CMAKE_DEBUG_POSTFIX "d")
endif()

if(MINGW)
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

# Build Options
option(RTAUDIO_BUILD_SHARED_LIBS "Compile library shared lib." TRUE)
option(RTAUDIO_BUILD_STATIC_LIBS "Compile library static lib." TRUE)
option(RTAUDIO_BUILD_TESTING "Compile test programs." TRUE)
option(RTAUDIO_BUILD_PYTHON "Build PyRtAudio python bindings" OFF)
set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type (Release,Debug)")
set(RTAUDIO_TARGETNAME_UNINSTALL "rtaudio_uninstall" CACHE STRING "Name of 'uninstall' build target")

# API Options
option(RTAUDIO_API_DS "Build DirectSound API" OFF)
option(RTAUDIO_API_ASIO "Build ASIO API" OFF)
option(RTAUDIO_API_WASAPI "Build WASAPI API" ${WIN32})
option(RTAUDIO_API_OSS "Build OSS4 API" ${xBSD})
option(RTAUDIO_API_ALSA "Build ALSA API" ${LINUX})
option(RTAUDIO_API_PULSE "Build PulseAudio API" ${pulse_FOUND})
option(RTAUDIO_API_JACK "Build JACK audio server API" ${HAVE_JACK})
option(RTAUDIO_API_CORE "Build CoreAudio API" ${APPLE})

# Check for functions
include(CheckFunctionExists)
check_function_exists(gettimeofday HAVE_GETTIMEOFDAY)
if (HAVE_GETTIMEOFDAY)
    add_definitions(-DHAVE_GETTIMEOFDAY)
endif ()

# Add -Wall if possible
if (CMAKE_COMPILER_IS_GNUCXX)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
endif (CMAKE_COMPILER_IS_GNUCXX)

# Add debug flags
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
  add_definitions(-D__RTAUDIO_DEBUG__)
  if (CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
  endif (CMAKE_COMPILER_IS_GNUCXX)
endif ()

# Read libtool version info from configure.ac
set(R "m4_define\\(\\[lt_([a-z]+)\\], ([0-9]+)\\)")
file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/configure.ac" CONFIGAC
  REGEX ${R})
foreach(_S ${CONFIGAC})
  string(REGEX REPLACE ${R} "\\1" k ${_S})
  string(REGEX REPLACE ${R} "\\2" v ${_S})
  set(SO_${k} ${v})
endforeach()
math(EXPR SO_current_minus_age "${SO_current} - ${SO_age}")
set(SO_VER "${SO_current_minus_age}")
set(FULL_VER "${SO_current_minus_age}.${SO_revision}.${SO_age}")

# Read package version info from configure.ac
set(R "AC_INIT\\(RtAudio, ([0-9\\.]+),.*\\)")
file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/configure.ac" CONFIGAC
  REGEX ${R})
string(REGEX REPLACE ${R} "\\1" PACKAGE_VERSION ${CONFIGAC})

# Init variables
set(rtaudio_SOURCES RtAudio.cpp RtAudio.h rtaudio_c.cpp rtaudio_c.h)
set(LINKLIBS)
set(PKGCONFIG_REQUIRES)
set(API_DEFS)
set(API_LIST)

# Tweak API-specific configuration.

# Jack
if (RTAUDIO_API_JACK AND jack_FOUND)
  set(NEED_PTHREAD ON)
  list(APPEND PKGCONFIG_REQUIRES "jack")
  list(APPEND API_DEFS "-D__UNIX_JACK__")
  list(APPEND API_LIST "jack")
  if(jack_FOUND)
    list(APPEND LINKLIBS ${jack_LIBRARIES})
    list(APPEND INCDIRS ${jack_INCLUDEDIR})
  else()
    list(APPEND LINKLIBS ${JACK_LIB})
  endif()
endif()

# ALSA
if (RTAUDIO_API_ALSA)
  set(NEED_PTHREAD ON)
  find_package(ALSA)
  if (NOT ALSA_FOUND)
    message(FATAL_ERROR "ALSA API requested but no ALSA dev libraries found")
  endif()
  list(APPEND INCDIRS ${ALSA_INCLUDE_DIR})
  list(APPEND LINKLIBS ${ALSA_LIBRARY})
  list(APPEND PKGCONFIG_REQUIRES "alsa")
  list(APPEND API_DEFS "-D__LINUX_ALSA__")
  list(APPEND API_LIST "alsa")
endif()

# OSS
if (RTAUDIO_OSS)
  set(NEED_PTHREAD ON)
  find_library(OSSAUDIO_LIB ossaudio)
  if (OSSAUDIO_LIB)
    list(APPEND LINKLIBS ossaudio)
    # Note: not an error on some systems
  endif()
  list(APPEND API_DEFS "-D__LINUX_OSS__")
  list(APPEND API_LIST "oss")
endif()

# Pulse
if (RTAUDIO_API_PULSE)
  set(NEED_PTHREAD ON)
  find_library(PULSE_LIB pulse)
  find_library(PULSESIMPLE_LIB pulse-simple)
  list(APPEND LINKLIBS ${PULSE_LIB} ${PULSESIMPLE_LIB})
  list(APPEND PKGCONFIG_REQUIRES "libpulse-simple")
  list(APPEND API_DEFS "-D__LINUX_PULSE__")
  list(APPEND API_LIST "pulse")
endif()

# CoreAudio
if (RTAUDIO_API_CORE)
  find_library(COREAUDIO_LIB CoreAudio)
  find_library(COREFOUNDATION_LIB CoreFoundation)
  list(APPEND LINKLIBS ${COREAUDIO_LIB} ${COREFOUNDATION_LIB})
  list(APPEND API_DEFS "-D__MACOSX_CORE__")
  list(APPEND API_LIST "core")
endif()

# ASIO
if (RTAUDIO_API_ASIO)
  set(NEED_WIN32LIBS ON)
  include_directories(include)
  list(APPEND rtaudio_SOURCES
    include/asio.cpp
    include/asiodrivers.cpp
    include/asiolist.cpp
    include/iasiothiscallresolver.cpp)
  list(APPEND API_DEFS "-D__WINDOWS_ASIO__")
  list(APPEND API_LIST "asio")
endif()

# DSound
if (RTAUDIO_API_DS)
  set(NEED_WIN32LIBS ON)
  list(APPEND LINKLIBS dsound)
  list(APPEND API_DEFS "-D__WINDOWS_DS__")
  list(APPEND API_LIST "ds")
endif()

# WASAPI
if (RTAUDIO_API_WASAPI)
  include_directories(include)
  set(NEED_WIN32LIBS ON)
  list(APPEND LINKLIBS ksuser mfplat mfuuid wmcodecdspuuid)
  list(APPEND API_DEFS "-D__WINDOWS_WASAPI__")
  list(APPEND API_LIST "wasapi")
endif()

# Windows libs
if (NEED_WIN32LIBS)
  list(APPEND LINKLIBS winmm ole32)
endif()

# pthread
if (NEED_PTHREAD)
  find_package(Threads REQUIRED
    CMAKE_THREAD_PREFER_PTHREAD
    THREADS_PREFER_PTHREAD_FLAG)
  list(APPEND LINKLIBS Threads::Threads)
endif()

# Create library targets.
cmake_policy(SET CMP0042 OLD)
set(LIB_TARGETS)
if(RTAUDIO_BUILD_SHARED_LIBS)
  add_library(rtaudio SHARED ${rtaudio_SOURCES})
  list(APPEND LIB_TARGETS rtaudio)

  # Add headers destination for install rule.
  set_target_properties(rtaudio PROPERTIES PUBLIC_HEADER RtAudio.h
    SOVERSION ${SO_VER}
    VERSION ${FULL_VER})

  # Set include paths, populate target interface.
  target_include_directories(rtaudio PRIVATE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    ${INCDIRS})

  # Set compile-time definitions
  target_compile_definitions(rtaudio PRIVATE ${API_DEFS})
  target_compile_definitions(rtaudio PRIVATE RTAUDIO_EXPORT)

  target_link_libraries(rtaudio ${LINKLIBS})
endif()

if(RTAUDIO_BUILD_STATIC_LIBS)
  add_library(rtaudio_static STATIC ${rtaudio_SOURCES})
  list(APPEND LIB_TARGETS rtaudio_static)

  # Add headers destination for install rule.
  set_target_properties(rtaudio_static PROPERTIES PUBLIC_HEADER RtAudio.h
    SOVERSION ${SO_VER}
    VERSION ${FULL_VER})

  # Set include paths, populate target interface.
  target_include_directories(rtaudio_static PRIVATE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    ${INCDIRS})

  # Set compile-time definitions
  target_compile_definitions(rtaudio_static PRIVATE ${API_DEFS})

  target_link_libraries(rtaudio_static ${LINKLIBS})
endif()

# Set standard installation directories.
include(GNUInstallDirs)

# Subdirs
if (RTAUDIO_BUILD_TESTING)
  include(CTest)
  add_subdirectory(tests)
endif (RTAUDIO_BUILD_TESTING)

# Message
string(REPLACE ";" " " apilist "${API_LIST}")
message(STATUS "Compiling with support for: ${apilist}")

# PkgConfig file
string(REPLACE ";" " " req "${PKGCONFIG_REQUIRES}")
string(REPLACE ";" " " api "${API_DEFS}")
set(prefix ${CMAKE_INSTALL_PREFIX})
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/rtaudio.pc.in" "rtaudio.pc" @ONLY)

# Add install rule.
install(TARGETS ${LIB_TARGETS}
        EXPORT RtAudioTargets
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# Store the package in the user registry.
export(PACKAGE RtAudio)

# Set installation path for CMake files.
if(WIN32)
    set(RTAUDIO_CMAKE_DESTINATION cmake)
else()
    set(RTAUDIO_CMAKE_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/RtAudio)
endif()

# Create CMake configuration export file.
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/RtAudioConfig.cmake "include(\${CMAKE_CURRENT_LIST_DIR}/RtAudioTargets.cmake)")

# Install CMake configuration export file.
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/RtAudioConfig.cmake
        DESTINATION ${RTAUDIO_CMAKE_DESTINATION})

# Export library target (build-tree).
export(EXPORT RtAudioTargets
       NAMESPACE RtAudio::)

# Export library target (install-tree).
install(EXPORT RtAudioTargets
        DESTINATION ${RTAUDIO_CMAKE_DESTINATION}
        NAMESPACE RtAudio::)

# Configure uninstall target.
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/RtAudioConfigUninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/RtAudioConfigUninstall.cmake" @ONLY)

# Create uninstall target.
add_custom_target(${RTAUDIO_TARGETNAME_UNINSTALL}
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/RtAudioConfigUninstall.cmake)

install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/rtaudio.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
