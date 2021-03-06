cmake_minimum_required(VERSION 3.1) # we use target_sources()

option(BOOST "use boost library" OFF)
option(IN_TREE "use the Extempore source dir as EXT_SHARE_DIR - useful for developers" ON)
option(ASIO "use the ASIO Portaudio backend" OFF)
option(JACK "use the Jack Portaudio backend" OFF)
option(PACKAGE "set up install targets for packaging" OFF)
option(BUILD_DEPS "download & build deps for aot_extended" ON)

####################
# option wrangling #
####################

# share directory

# if -DEXT_SHARE_DIR=/path/to/share-dir is provided at the command
# line it will override these values

if(WIN32)
  # Windows doesn't really do "installation", so we'll assume that
  # extempore.exe's always going to be run from the Extempore
  # directory
  set(EXT_SHARE_DIR "." CACHE PATH
    "path to Extempore's shared runtime files")
else()
  if(IN_TREE)
    set(EXT_SHARE_DIR "${CMAKE_SOURCE_DIR}" CACHE PATH
      "path to Extempore's shared runtime files")
  else()
    set(EXT_SHARE_DIR "${CMAKE_INSTALL_PREFIX}/share/extempore" CACHE PATH
      "path to Extempore's shared runtime files")
  endif()
endif()

# packaging (binary distribution)

if(PACKAGE)
  # this needs to be set before project() is called
  set(CMAKE_OSX_SYSROOT macosx)
  set(CMAKE_OSX_DEPLOYMENT_TARGET 10.9)
  message(STATUS "Building Extempore for binary distribution")
  # need deps if we're packaging
  set(BUILD_DEPS ON CACHE BOOL "" FORCE)
  if(IN_TREE)
    set(IN_TREE OFF CACHE BOOL "use the Extempore source dir as EXT_SHARE_DIR - useful for developers" FORCE)
  endif()
endif()

# LLVM

if(DEFINED ENV{EXT_LLVM_DIR})
  # if there's an EXT_LLVM_DIR environment variable, use that
  set(EXT_LLVM_DIR $ENV{EXT_LLVM_DIR})
  set(BUILD_LLVM OFF)
elseif(WIN32)
  set(EXT_LLVM_DIR ${CMAKE_SOURCE_DIR}/llvm)
  set(BUILD_LLVM ON)
else()
  set(EXT_LLVM_DIR ${EXT_SHARE_DIR}/llvm)
  set(BUILD_LLVM ON)
endif()

# building external shared library dependencies

if(BUILD_DEPS)
  set(EXT_DEPS_INSTALL_DIR ${CMAKE_BINARY_DIR}/deps-install)
  if(WIN32)
    set(EXT_PLATFORM_SHLIBS_DIR ${CMAKE_SOURCE_DIR}/libs/platform-shlibs)
  else()
    set(EXT_PLATFORM_SHLIBS_DIR ${EXT_SHARE_DIR}/libs/platform-shlibs)
  endif()
  if(PACKAGE)
    set(EXT_DEPS_C_FLAGS "${CMAKE_C_FLAGS_RELEASE} -mtune=generic")
    set(EXT_DEPS_CXX_FLAGS "${CMAKE_CXX_FLAGS_RELEASE} -mtune=generic")
    message(STATUS "compiler flags for packaging:\nC    ${EXT_DEPS_C_FLAGS}\nCXX  ${EXT_DEPS_CXX_FLAGS}")
  endif()
endif()

project(Extempore VERSION 0.7.0)

if(NOT ${CMAKE_SIZEOF_VOID_P} EQUAL 8)
  message(FATAL_ERROR "Extempore currently only runs on 64-bit platforms.")
endif()

# Set a default build type if none was specified

if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "Building 'Release' configuration")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# # set_target_properties(extempore PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${CMAKE_HOME_DIRECTORY}")
# set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_SOURCE_DIR}/extras/cmake CACHE PATH
#   "path to Extempore's cmake modules")

####################
# platform/version #
####################

# this stuff is handy to make sure that the packages/test platforms
# get sensible names

if(UNIX)
  find_program(UNAME_PROGRAM uname)
  execute_process(COMMAND ${UNAME_PROGRAM} -m
    OUTPUT_VARIABLE UNAME_MACHINE_NAME
    OUTPUT_STRIP_TRAILING_WHITESPACE)
  execute_process(COMMAND ${UNAME_PROGRAM} -r
    OUTPUT_VARIABLE UNAME_OS_RELEASE
    OUTPUT_STRIP_TRAILING_WHITESPACE)
  execute_process(COMMAND ${UNAME_PROGRAM} -s
    OUTPUT_VARIABLE UNAME_OS_NAME
    OUTPUT_STRIP_TRAILING_WHITESPACE)
endif(UNIX)

if(APPLE)
  set(EXTEMPORE_SYSTEM_NAME "osx")
  execute_process(COMMAND sw_vers -productVersion
    OUTPUT_VARIABLE EXTEMPORE_SYSTEM_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE)
  string(REGEX MATCH "^10.[0-9]+" EXTEMPORE_SYSTEM_VERSION ${EXTEMPORE_SYSTEM_VERSION})
  set(EXTEMPORE_SYSTEM_ARCHITECTURE ${UNAME_MACHINE_NAME})
elseif(UNIX)
  # try lsb_release first - better at giving the distro name
  execute_process(COMMAND lsb_release -is
    OUTPUT_VARIABLE EXTEMPORE_SYSTEM_NAME
    OUTPUT_STRIP_TRAILING_WHITESPACE)
  if(NOT EXTEMPORE_SYSTEM_NAME)
    # otherwise use uname output
    set(EXTEMPORE_SYSTEM_NAME ${UNAME_OS_NAME})
  endif()
  set(EXTEMPORE_SYSTEM_VERSION ${UNAME_OS_RELEASE})
  set(EXTEMPORE_SYSTEM_ARCHITECTURE ${UNAME_MACHINE_NAME})
elseif(WIN32)
  set(EXTEMPORE_SYSTEM_NAME "Windows")
  string(REGEX MATCH "^[0-9]+" EXTEMPORE_SYSTEM_VERSION ${CMAKE_SYSTEM_VERSION})
  # deal with Windows version number shenanigans
  if(${EXTEMPORE_SYSTEM_VERSION} LESS 10)
    string(CONCAT ACTUAL_VERSION_EXPRESSION "${EXTEMPORE_SYSTEM_VERSION}" " + 1")
    math(EXPR EXTEMPORE_SYSTEM_VERSION ${ACTUAL_VERSION_EXPRESSION})
  endif()
  set(EXTEMPORE_SYSTEM_ARCHITECTURE ${CMAKE_SYSTEM_PROCESSOR})
else()
  message(FATAL_ERROR "Sorry, Extempore isn't supported on this platform - OSX, Linux & Windows only.")
endif()

########
# PCRE #
########

# current in-tree PCRE version: 8.38

add_library(pcre STATIC
  # headers
  src/pcre/config.h
  src/pcre/pcre.h
  src/pcre/ucp.h
  # source files
  src/pcre/pcre_chartables.c
  src/pcre/pcre_compile.c
  src/pcre/pcre_exec.c
  src/pcre/pcre_globals.c
  src/pcre/pcre_internal.h
  src/pcre/pcre_newline.c
  src/pcre/pcre_tables.c
  )

target_compile_definitions(pcre
  PRIVATE -DHAVE_CONFIG_H
  )

if(PACKAGE)
  target_compile_options(pcre
    PRIVATE -mtune=generic)
endif()

#############
# portaudio #
#############

add_library(portaudio STATIC
  src/portaudio/src/common/pa_allocation.c
  src/portaudio/src/common/pa_allocation.h
  src/portaudio/src/common/pa_converters.c
  src/portaudio/src/common/pa_converters.h
  src/portaudio/src/common/pa_cpuload.c
  src/portaudio/src/common/pa_cpuload.h
  src/portaudio/src/common/pa_debugprint.c
  src/portaudio/src/common/pa_debugprint.h
  src/portaudio/src/common/pa_dither.c
  src/portaudio/src/common/pa_dither.h
  src/portaudio/src/common/pa_endianness.h
  src/portaudio/src/common/pa_front.c
  src/portaudio/src/common/pa_hostapi.h
  src/portaudio/src/common/pa_memorybarrier.h
  src/portaudio/src/common/pa_process.c
  src/portaudio/src/common/pa_process.h
  src/portaudio/src/common/pa_ringbuffer.c
  src/portaudio/src/common/pa_ringbuffer.h
  src/portaudio/src/common/pa_stream.c
  src/portaudio/src/common/pa_stream.h
  src/portaudio/src/common/pa_trace.c
  src/portaudio/src/common/pa_trace.h
  src/portaudio/src/common/pa_types.h
  src/portaudio/src/common/pa_util.h
  )

target_include_directories(portaudio
  PRIVATE src/portaudio/include
  PRIVATE src/portaudio/src/common)

# platform-specific

if(APPLE)
  # use CoreAudio on OSX
  target_sources(portaudio
    PRIVATE src/portaudio/src/hostapi/coreaudio/pa_mac_core.c
    PRIVATE src/portaudio/src/hostapi/coreaudio/pa_mac_core_blocking.c
    # PRIVATE src/portaudio/src/hostapi/coreaudio/pa_mac_core_old.c
    PRIVATE src/portaudio/src/hostapi/coreaudio/pa_mac_core_utilities.c
    PRIVATE src/portaudio/src/os/unix/pa_unix_hostapis.c
    PRIVATE src/portaudio/src/os/unix/pa_unix_util.c
    )
  target_include_directories(portaudio
    PRIVATE src/portaudio/src/os/unix)
  target_compile_definitions(portaudio
    PRIVATE -DPA_USE_COREAUDIO)
  target_compile_options(portaudio
    PRIVATE -Wno-deprecated-declarations)
  target_link_libraries(portaudio
    PRIVATE "-framework CoreAudio"
    PRIVATE "-framework AudioUnit"
    PRIVATE "-framework AudioToolbox")

elseif(UNIX AND NOT APPLE)
  # use ALSA on Linux
  target_sources(portaudio
    PRIVATE src/portaudio/src/hostapi/alsa/pa_linux_alsa.c
    PRIVATE src/portaudio/src/os/unix/pa_unix_hostapis.c
    PRIVATE src/portaudio/src/os/unix/pa_unix_util.c
    )
  target_include_directories(portaudio
    PRIVATE src/portaudio/src/os/unix)
  target_compile_definitions(portaudio
    PRIVATE -DPA_USE_ALSA)
  target_link_libraries(portaudio
    PRIVATE asound)

  if(JACK)
    target_sources(portaudio
      PRIVATE src/portaudio/src/hostapi/jack/pa_jack.c)
    target_compile_definitions(portaudio
      PRIVATE -DPA_USE_JACK)
    target_link_libraries(portaudio
      PRIVATE jack)
  endif()

elseif(WIN32)
  # use everything except for ASIO on Windows by default, but you can
  # use asio with the cmake ASIO option (see below)
  target_sources(portaudio
    PRIVATE src/portaudio/src/hostapi/wdmks/pa_win_wdmks.c
    # PRIVATE src/portaudio/src/hostapi/wmme/pa_win_wmme.c
    PRIVATE src/portaudio/src/hostapi/dsound/pa_win_ds.c
    PRIVATE src/portaudio/src/hostapi/dsound/pa_win_ds_dynlink.c
    # PRIVATE src/portaudio/src/hostapi/wasapi/pa_win_wasapi.c
    PRIVATE src/portaudio/src/os/win/pa_win_coinitialize.c
    PRIVATE src/portaudio/src/os/win/pa_win_hostapis.c
    PRIVATE src/portaudio/src/os/win/pa_win_util.c
    PRIVATE src/portaudio/src/os/win/pa_win_waveformat.c
    PRIVATE src/portaudio/src/os/win/pa_win_wdmks_utils.c
    PRIVATE src/portaudio/src/os/win/pa_x86_plain_converters.c
    )
  target_include_directories(portaudio
    PRIVATE src/portaudio/src/os/win)
  target_compile_definitions(portaudio
    PRIVATE -DPA_USE_WDMKS
    # PRIVATE -DPA_USE_WMME
    # PRIVATE -DPA_USE_WASAPI
    PRIVATE -DPA_USE_DS)
endif()

if(ASIO)
  # if you want to build with ASIO on Windows, you need the ASIO SDK.
  # See src/portaudio/src/hostapi/asio/ASIO-README.txt for
  # instructions on how to get it

  # then, copy it into the src/portaudio/src/hostapi/asio directory in
  # the Extempore source folder
  target_sources(portaudio
    PRIVATE src/portaudio/src/hostapi/asio/pa_asio.cpp
    PRIVATE src/portaudio/src/hostapi/asio/ASIOSDK/common/asio.cpp
    PRIVATE src/portaudio/src/hostapi/asio/ASIOSDK/host/asiodrivers.cpp
    PRIVATE src/portaudio/src/hostapi/asio/ASIOSDK/host/pc/asiolist.cpp
    )
  target_include_directories(portaudio
    PRIVATE src/portaudio/src/hostapi/asio/ASIOSDK/common
    PRIVATE src/portaudio/src/hostapi/asio/ASIOSDK/host
    PRIVATE src/portaudio/src/hostapi/asio/ASIOSDK/host/pc)
  target_compile_definitions(portaudio
    PRIVATE -DPA_USE_ASIO)
endif()

if(PACKAGE)
  target_compile_options(portaudio
    PRIVATE -mtune=generic)
endif()

##############
# LLVM 3.7.0 #
##############

# if you need to build LLVM by hand, the command will be something like
# cmake .. -DLLVM_TARGETS_TO_BUILD=X86 -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_TERMINFO=OFF -DLLVM_ENABLE_ZLIB=OFF -DCMAKE_INSTALL_PREFIX=c:/Users/ben/Code/extempore/llvm-3.8.0-release

if(BUILD_LLVM)
  include(ExternalProject)
  find_program(PATCH_PROGRAM patch)

  if(NOT PATCH_PROGRAM)
    message(WARNING "Cannot find \"patch.exe\", which is required to patch LLVM")
  endif()

  if(PACKAGE)
    ExternalProject_Add(LLVM
      PREFIX llvm
      URL http://www.llvm.org/releases/3.7.0/llvm-3.7.0.src.tar.xz
      URL_MD5 b98b9495e5655a672d6cb83e1a180f8e
      PATCH_COMMAND patch -p0 < ${CMAKE_SOURCE_DIR}/extras/extempore-llvm-3.7.0.patch
      CMAKE_ARGS
      -DCMAKE_OSX_SYSROOT=macosx
      -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
      -DLLVM_TARGETS_TO_BUILD=X86
      -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      -DLLVM_ENABLE_TERMINFO=OFF
      -DLLVM_ENABLE_ZLIB=OFF
      -DCMAKE_C_FLAGS=${EXT_DEPS_C_FLAGS}
      -DCMAKE_CXX_FLAGS=${EXT_DEPS_CXX_FLAGS}
      -DCMAKE_INSTALL_PREFIX=${EXT_LLVM_DIR})
  else()
    ExternalProject_Add(LLVM
      PREFIX llvm
      URL http://www.llvm.org/releases/3.7.0/llvm-3.7.0.src.tar.xz
      URL_MD5 b98b9495e5655a672d6cb83e1a180f8e
      PATCH_COMMAND patch -p0 < ${CMAKE_SOURCE_DIR}/extras/extempore-llvm-3.7.0.patch
      CMAKE_ARGS
      -DLLVM_TARGETS_TO_BUILD=X86
      -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      -DLLVM_ENABLE_TERMINFO=OFF
      -DLLVM_ENABLE_ZLIB=OFF
      -DCMAKE_INSTALL_PREFIX=${EXT_LLVM_DIR})
  endif()
  if(WIN32)
    add_custom_target(LLVM_install
      COMMENT "This is necessary to get CMake to install LLVM (needed to build Extempore) before the Extempore install step"
      DEPENDS LLVM
      COMMAND ${CMAKE_COMMAND} -P ${CMAKE_BINARY_DIR}/llvm/src/LLVM-build/cmake_install.cmake)
  else()
    ExternalProject_Add_StepTargets(LLVM install)
  endif()
endif()

# the ordering of these libs matters, especially with the gcc linker.
# Check the output of "llvm-config --libnames" to be sure
set(EXT_LLVM_LIBRARIES "LLVMLTO;LLVMObjCARCOpts;LLVMLinker;LLVMBitWriter;LLVMIRReader;LLVMAsmParser;LLVMOption;LLVMTableGen;LLVMX86Disassembler;LLVMX86AsmParser;LLVMX86CodeGen;LLVMSelectionDAG;LLVMAsmPrinter;LLVMX86Desc;LLVMMCDisassembler;LLVMX86Info;LLVMX86AsmPrinter;LLVMX86Utils;LLVMMCJIT;LLVMipo;LLVMVectorize;LLVMInterpreter;LLVMExecutionEngine;LLVMRuntimeDyld;LLVMCodeGen;LLVMTarget;LLVMScalarOpts;LLVMObject;LLVMMCParser;LLVMBitReader;LLVMInstCombine;LLVMInstrumentation;LLVMTransformUtils;LLVMipa;LLVMMC;LLVMAnalysis;LLVMCore;LLVMSupport")

foreach(llvm_lib ${EXT_LLVM_LIBRARIES})
  get_filename_component(LLVM_LIB_FULLPATH "${EXT_LLVM_DIR}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}${llvm_lib}${CMAKE_STATIC_LIBRARY_SUFFIX}" ABSOLUTE)
  list(APPEND LLVM_LIBRARIES ${LLVM_LIB_FULLPATH})
endforeach()

#########
# Boost #
#########

# *must* have Boost on Windows
if(WIN32)
  set(BOOST ON)
  if(NOT BOOST_DIR)
    # if BOOST_DIR has been set elsewhere, assume it's all present and
    # correct and go with that. Otherwise, download boost with nuget
    # commandline tool and stick it in ${CMAKE_SOURCE_DIR}/boost
    set(BOOST_DIR ${CMAKE_SOURCE_DIR}/boost)
    # check nuget commandline tool is installed
    find_program(NUGET nuget)
    if(NOT NUGET)
      message(FATAL "Cannot find nuget command line tool.\nEither install it with e.g. choco install nuget.commandline, or get boost some other way and set the -DBOOST_DIR variable")
    else()
      if(NOT IS_DIRECTORY ${BOOST_DIR})
        # if directory exists, assume this step has already been done
        message(STATUS "Downloading boost components into ${CMAKE_SOURCE_DIR}/boost with \"nuget install\"")
        file(MAKE_DIRECTORY ${BOOST_DIR})
        execute_process(
          COMMAND ${NUGET} install -Version 1.59 boost-vc140
          WORKING_DIRECTORY ${BOOST_DIR})
      endif()
    endif()
  endif()
endif()

#############
# extempore #
#############

# source files

add_executable(extempore src/Extempore.cpp
  src/AudioDevice.cpp
  src/EXTCondition.cpp
  src/EXTLLVM.cpp
  src/EXTMonitor.cpp
  src/EXTMutex.cpp
  src/EXTThread.cpp
  src/Extempore.cpp
  src/OSC.cpp
  src/Scheme.cpp
  src/SchemeFFI.cpp
  src/SchemeProcess.cpp
  src/SchemeREPL.cpp
  src/TaskScheduler.cpp
  src/UNIV.cpp
  )

if(WIN32)
  target_sources(extempore PRIVATE src/extempore.def)
endif()

target_include_directories(extempore PRIVATE include)

# suppress the warning about the opcode switch statement
if(UNIX)
  set_source_files_properties(src/Scheme.cpp PROPERTIES COMPILE_FLAGS -Wno-switch)
endif()

# dependencies

add_dependencies(extempore pcre portaudio)

if(BUILD_LLVM)
  if(WIN32)
    add_dependencies(extempore LLVM_install)
  else()
    add_dependencies(extempore LLVM)
  endif()
endif()

target_include_directories(extempore
  PRIVATE src/pcre
  PRIVATE src/portaudio/include)

target_link_libraries(extempore
  PRIVATE pcre
  PRIVATE portaudio)

target_include_directories(extempore PRIVATE ${EXT_LLVM_DIR}/include)
target_link_libraries(extempore PRIVATE ${LLVM_LIBRARIES})

# compiler options

if(PACKAGE)
  target_compile_definitions(extempore
    PRIVATE -DEXT_SHARE_DIR=".")
  target_compile_options(extempore
    PRIVATE -mtune=generic)
else()
  target_compile_definitions(extempore
    PRIVATE -DEXT_SHARE_DIR="${EXT_SHARE_DIR}")
endif()

# platform-specific config

if(UNIX)
  target_compile_definitions(extempore
    PRIVATE -D_GNU_SOURCE
	  PRIVATE -D__STDC_CONSTANT_MACROS
	  PRIVATE -D__STDC_FORMAT_MACROS
	  PRIVATE -D__STDC_LIMIT_MACROS)
  target_compile_options(extempore
    PRIVATE -std=c++11
    PRIVATE -fvisibility-inlines-hidden
    # PRIVATE -fno-exceptions
    PRIVATE -fno-rtti
    PRIVATE -fno-common
    PRIVATE -Woverloaded-virtual
    # PRIVATE -Wcast-qual
    PRIVATE -Wno-unused-result)
  target_link_libraries(extempore PRIVATE pthread)
endif()

if(WIN32)
  target_compile_definitions(extempore
    PRIVATE -DPCRE_STATIC
    PRIVATE -D_CRT_SECURE_NO_WARNINGS)
elseif(APPLE) # OSX
  # use clang++ by default
  set(CMAKE_C_COMPILER clang)
  set(CMAKE_CXX_COMPILER clang++)
  # tell the compiler about the few ObjC++ source files on OSX
  set_source_files_properties(
    src/Extempore.cpp
    src/SchemeFFI.cpp
    src/UNIV.cpp
    PROPERTIES
    COMPILE_FLAGS "-x objective-c++")
  # frameworks
  target_link_libraries(extempore
    PRIVATE "-framework Cocoa"
    PRIVATE "-framework CoreAudio"
    PRIVATE "-framework AudioUnit"
    PRIVATE "-framework AudioToolbox")

elseif(UNIX AND NOT APPLE) # Linux
  set_property(TARGET extempore PROPERTY POSITION_INDEPENDENT_CODE ON)
  # target_link_libraries(extempore PRIVATE --export-dynamic)
  target_link_libraries(extempore PRIVATE dl)
endif()

# on Windows, put the created extempore.exe straight into the source
# directory, and the .lib file into libs/platform-shlibs
if(WIN32)
  set_target_properties(extempore
    PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_SOURCE_DIR}
    RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_SOURCE_DIR}
    LIBRARY_OUTPUT_DIRECTORY_DEBUG ${CMAKE_SOURCE_DIR}
    LIBRARY_OUTPUT_DIRECTORY_RELEASE ${CMAKE_SOURCE_DIR}
    ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${CMAKE_SOURCE_DIR}/libs/platform-shlibs
    ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${CMAKE_SOURCE_DIR}/libs/platform-shlibs)
endif()

if(BOOST)
  target_include_directories(extempore
    PRIVATE ${BOOST_DIR}/boost.1.59.0.0/lib/native/include)
  target_compile_definitions(extempore
    PRIVATE -DEXT_BOOST -DBOOST_EXCEPTION_DISABLE)
  target_link_libraries(extempore
    PRIVATE ${BOOST_DIR}/boost_date_time-vc140.1.59.0.0/lib/native/address-model-64/lib/libboost_date_time-vc140-mt$<$<CONFIG:Debug>:-gd>-1_59.lib
    PRIVATE ${BOOST_DIR}/boost_regex-vc140.1.59.0.0/lib/native/address-model-64/lib/libboost_regex-vc140-mt$<$<CONFIG:Debug>:-gd>-1_59.lib
    PRIVATE ${BOOST_DIR}/boost_system-vc140.1.59.0.0/lib/native/address-model-64/lib/libboost_system-vc140-mt$<$<CONFIG:Debug>:-gd>-1_59.lib)
endif()

##########
# assets #
##########

add_custom_target(assets
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  COMMAND curl -O http://extempore.moso.com.au/extras/assets.tgz
  COMMAND ${CMAKE_COMMAND} -E tar xf assets.tgz
  COMMAND ${CMAKE_COMMAND} -E remove assets.tgz
  COMMENT "Downloading assets from http://extempore.moso.com.au/extras/assets.tgz")

###########
# install #
###########

if(IN_TREE)
  # if we're developing in-tree, installation just involves moving the
  # binary into the toplevel source directory
  install(TARGETS extempore
    RUNTIME
    DESTINATION bin)
else()
  # otherwise, install all the things
  if(PACKAGE)
    set(EXT_INSTALL_PREFIX extempore)
    install(TARGETS extempore
      RUNTIME
      DESTINATION ${EXT_INSTALL_PREFIX})
  else()
    set(EXT_INSTALL_PREFIX share/extempore)
    install(TARGETS extempore
      RUNTIME
      DESTINATION bin)
  endif()
  # install the rest of the necessary files into the "share" directory
  install(DIRECTORY runtime libs examples tests extras
    DESTINATION ${EXT_INSTALL_PREFIX}
    PATTERN ".DS_Store" EXCLUDE
    PATTERN "extras/cmake" EXCLUDE
    PATTERN "extras/extempore-emacs-mode" EXCLUDE)
  # assets are optional
  install(DIRECTORY assets
    OPTIONAL
    DESTINATION ${EXT_INSTALL_PREFIX}
    PATTERN ".DS_Store" EXCLUDE)
endif()

###################
# AOT compilation #
###################

if(WIN32)

  configure_file(
    ${CMAKE_SOURCE_DIR}/extras/cmake/aot.cmake.in
    ${CMAKE_SOURCE_DIR}/extras/cmake/aot.cmake
    @ONLY)

  configure_file(
    ${CMAKE_SOURCE_DIR}/extras/cmake/aot_extended.cmake.in
    ${CMAKE_SOURCE_DIR}/extras/cmake/aot_extended.cmake
    @ONLY)

  add_custom_target(aot
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_SOURCE_DIR}/extras/cmake/aot.cmake
    COMMENT "Ahead-of-time compiling the standard library..."
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

  add_custom_target(aot_extended ALL
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_SOURCE_DIR}/extras/cmake/aot_extended.cmake
    COMMENT "Ahead-of-time compiling the standard library..."
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

else(WIN32)

  # this approach requires specifying the inter-lib dependencies by hand, but allows us to do AOT compilation in parallell

  function(aotcompile_lib lib group port) # deps are optional, and go at the end
    get_filename_component(basename ${lib} NAME_WE)
    set(targetname aot_${basename})
    if(PACKAGE)
      add_custom_target(${targetname}
        COMMAND extempore --nobase --noaudio --mcpu=generic --attr=none --port=${port}
        --eval "(impc:aot:compile-xtm-file \"${lib}\" #t #t)"
        BYPRODUCTS libs/aot-cache/xtm${basename}${CMAKE_SHARED_LIBRARY_SUFFIX}
        WORKING_DIRECTORY ${EXT_SHARE_DIR}
        VERBATIM)
    else(PACKAGE)
      add_custom_target(${targetname}
        COMMAND extempore --nobase --noaudio --port=${port}
        --eval "(impc:aot:compile-xtm-file \"${lib}\" #t #t)"
        BYPRODUCTS libs/aot-cache/xtm${basename}${CMAKE_SHARED_LIBRARY_SUFFIX}
        WORKING_DIRECTORY ${EXT_SHARE_DIR}
        VERBATIM)
    endif(PACKAGE)
  add_dependencies(${targetname} extempore)
    add_dependencies(${group} ${targetname})
    foreach(dep ${ARGN})
      add_dependencies(${targetname} aot_${dep})
    endforeach()
  endfunction(aotcompile_lib)

  # core
  add_custom_target(aot)
  aotcompile_lib(libs/base/base.xtm aot 17099) # no lib dependency for base.xtm
  aotcompile_lib(libs/core/math.xtm aot 17097 base)
  aotcompile_lib(libs/core/audiobuffer.xtm aot 17087 base)
  aotcompile_lib(libs/core/audio_dsp.xtm aot 17095 base math audiobuffer)
  aotcompile_lib(libs/core/instruments.xtm aot 17093 base math audio_dsp)

  # extended
  add_custom_target(aot_extended ALL)
  aotcompile_lib(libs/external/fft.xtm aot_extended 17091 base math)
  aotcompile_lib(libs/external/sndfile.xtm aot_extended 17089 base)
  aotcompile_lib(libs/external/audio_dsp_ext.xtm aot_extended 17085 base fft sndfile)
  aotcompile_lib(libs/external/instruments_ext.xtm aot_extended 17083 base sndfile instruments)
  aotcompile_lib(libs/external/portmidi.xtm aot_extended 17081 base)
  aotcompile_lib(libs/external/stb_image.xtm aot_extended 17077 base)
  aotcompile_lib(libs/external/glfw3.xtm aot_extended 17075 base)
  if(WIN32)
    set(GL_BIND_METHOD getprocaddress)
  else()
    set(GL_BIND_METHOD directbind)
  endif()
  aotcompile_lib(libs/external/gl/glcore-${GL_BIND_METHOD}.xtm aot_extended 17073 base)
  aotcompile_lib(libs/external/gl/gl-objects.xtm aot_extended 17071 base glcore-${GL_BIND_METHOD} stb_image)
  aotcompile_lib(libs/external/nanovg.xtm aot_extended 17069 base glcore-${GL_BIND_METHOD})
  aotcompile_lib(libs/external/gl/glcompat-${GL_BIND_METHOD}.xtm aot_extended 17067 base)
  aotcompile_lib(libs/external/graphics-pipeline.xtm aot_extended 17065 base glcompat-${GL_BIND_METHOD})
  if(NOT WIN32)
    # these ones don't currently work on Windows
    # aotcompile_lib(libs/external/glib.xtm aot_extended 17079 base)
    aotcompile_lib(libs/external/assimp.xtm aot_extended 17063 base stb_image graphics-pipeline)
  endif()

endif(WIN32)

add_dependencies(aot extempore)
add_dependencies(aot_extended extempore)

# uninstall only AOT-compiled libs
add_custom_target(clean_aot
  COMMAND ${CMAKE_COMMAND} -E remove_directory ${EXT_SHARE_DIR}/libs/aot-cache
  COMMENT "Removing AOT-compiled libs")

#########################
# deps for AOT-extended #
#########################

if(BUILD_DEPS)

  include(ExternalProject)

  ExternalProject_Add(portmidi
    PREFIX portmidi
    URL https://github.com/extemporelang/portmidi/archive/217.zip
    CMAKE_ARGS
    -DCMAKE_OSX_SYSROOT=macosx
    -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_C_FLAGS=${EXT_DEPS_C_FLAGS}
    -DCMAKE_CXX_FLAGS=${EXT_DEPS_CXX_FLAGS}
    -DCMAKE_INSTALL_PREFIX=${EXT_DEPS_INSTALL_DIR})

  ExternalProject_Add(nanovg
    PREFIX nanovg
    URL https://github.com/extemporelang/nanovg/archive/master.zip
    CMAKE_ARGS
    -DCMAKE_OSX_SYSROOT=macosx
    -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_C_FLAGS=${EXT_DEPS_C_FLAGS}
    -DCMAKE_CXX_FLAGS=${EXT_DEPS_CXX_FLAGS}
    -DCMAKE_INSTALL_PREFIX=${EXT_DEPS_INSTALL_DIR})

  ExternalProject_Add(kiss_fft
    PREFIX kiss_fft
    URL https://github.com/extemporelang/kiss_fft/archive/master.zip
    CMAKE_ARGS
    -DCMAKE_OSX_SYSROOT=macosx
    -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_C_FLAGS=${EXT_DEPS_C_FLAGS}
    -DCMAKE_CXX_FLAGS=${EXT_DEPS_CXX_FLAGS}
    -DCMAKE_INSTALL_PREFIX=${EXT_DEPS_INSTALL_DIR})

  ExternalProject_Add(stb_image
    PREFIX stb_image
    URL https://github.com/extemporelang/stb/archive/master.zip
    CMAKE_ARGS
    -DCMAKE_OSX_SYSROOT=macosx
    -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_C_FLAGS=${EXT_DEPS_C_FLAGS}
    -DCMAKE_CXX_FLAGS=${EXT_DEPS_CXX_FLAGS}
    -DCMAKE_INSTALL_PREFIX=${EXT_DEPS_INSTALL_DIR})

  ExternalProject_Add(glfw3
    PREFIX glfw3
    URL https://github.com/glfw/glfw/archive/3.1.2.zip
    CMAKE_ARGS
    -DCMAKE_OSX_SYSROOT=macosx
    -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_C_FLAGS=${EXT_DEPS_C_FLAGS}
    -DCMAKE_CXX_FLAGS=${EXT_DEPS_CXX_FLAGS}
    -DBUILD_SHARED_LIBS=ON
    -DGLFW_BUILD_EXAMPLES=OFF
    -DGLFW_BUILD_TESTS=OFF
    -DCMAKE_INSTALL_PREFIX=${EXT_DEPS_INSTALL_DIR})

  ExternalProject_Add(assimp
    PREFIX assimp
    URL https://github.com/assimp/assimp/archive/v3.2.zip
    CMAKE_ARGS
    -DCMAKE_OSX_SYSROOT=macosx
    -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_C_FLAGS=${EXT_DEPS_C_FLAGS}
    -DCMAKE_CXX_FLAGS=${EXT_DEPS_CXX_FLAGS}
    -DASSIMP_BUILD_ASSIMP_TOOLS=OFF
    -DASSIMP_BUILD_SAMPLES=OFF
    -DASSIMP_BUILD_TESTS=OFF
    -DCMAKE_INSTALL_PREFIX=${EXT_DEPS_INSTALL_DIR})

  if(WIN32)
    # on Windows, we need expat for apr
    ExternalProject_Add(expat
      PREFIX expat
      URL http://downloads.sourceforge.net/project/expat/expat/2.1.0/expat-2.1.0.tar.gz
      CMAKE_ARGS
      -DBUILD_tools=OFF
      -DBUILD_examples=OFF
      -DBUILD_tests=OFF
      -DCMAKE_OSX_SYSROOT=macosx
      -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
      -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      -DCMAKE_C_FLAGS=${EXT_DEPS_C_FLAGS}
      -DCMAKE_CXX_FLAGS=${EXT_DEPS_CXX_FLAGS}
      -DCMAKE_INSTALL_PREFIX=${EXT_DEPS_INSTALL_DIR})

    # need to tell CMake where to find libexpat
    set(CMAKE_MODULE_PATH ${EXT_DEPS_INSTALL_DIR})
    ExternalProject_Add(apr
      PREFIX apr
      URL http://mirror.ventraip.net.au/apache/apr/apr-1.5.2.tar.gz
      CMAKE_ARGS
      -DAPR_HAVE_ODBC=OFF
      -DINSTALL_PDB=OFF
      -DCMAKE_OSX_SYSROOT=macosx
      -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
      -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      -DCMAKE_C_FLAGS=${EXT_DEPS_C_FLAGS}
      -DCMAKE_CXX_FLAGS=${EXT_DEPS_CXX_FLAGS}
      -DCMAKE_INSTALL_PREFIX=${EXT_DEPS_INSTALL_DIR})

    add_dependencies(apr expat)

    # aargh! sndfile on Windows still requires sh, sed, etc. and
    # doesn't work. so we'll just grab the pre-built one manually

    # ExternalProject_Add(sndfile
    #   PREFIX libsndfile
    #   URL https://github.com/erikd/libsndfile/archive/master.zip
    #   CONFIGURE_COMMAND ${CMAKE_BINARY_DIR}/libsndfile/src/sndfile/autogen.sh cmake && ${CMAKE_COMMAND} ${CMAKE_SOURCE_DIR}
    #   CMAKE_ARGS
    #   -DCMAKE_OSX_SYSROOT=macosx
    #   -DCMAKE_OSX_DEPLOYMENT_TARGET=10.9
    #   -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    #   -DCMAKE_INSTALL_PREFIX=${EXT_DEPS_INSTALL_DIR})
  else()
    # build with as few deps as we can get away with
    ExternalProject_Add(sndfile
      PREFIX libsndfile
      URL http://www.mega-nerd.com/libsndfile/files/libsndfile-1.0.26.tar.gz
      CONFIGURE_COMMAND sh configure --prefix=${EXT_DEPS_INSTALL_DIR} --disable-static --disable-cpu-clip --disable-external-libs --disable-sqlite --disable-octave
      BUILD_COMMAND make CFLAGS=${EXT_DEPS_C_FLAGS}
      INSTALL_COMMAND make install
      BUILD_IN_SOURCE ON)

    ExternalProject_Add(apr
      PREFIX apr
      URL http://mirror.ventraip.net.au/apache/apr/apr-1.5.2.tar.gz
      CONFIGURE_COMMAND sh configure --prefix=${EXT_DEPS_INSTALL_DIR} --enable-threads --disable-static
      BUILD_COMMAND make
      INSTALL_COMMAND make install
      BUILD_IN_SOURCE ON)
  endif()

  # wrap the whole thing in a target

  if(UNIX)
    add_custom_target(extended_deps
      COMMENT "moving shared libs into ${EXT_PLATFORM_SHLIBS_DIR}"
      DEPENDS sndfile assimp glfw3 stb_image kiss_fft nanovg portmidi LLVM sndfile apr
      COMMAND ${CMAKE_COMMAND} -E make_directory ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy libassimp${CMAKE_SHARED_LIBRARY_SUFFIX} ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy libkiss_fft${CMAKE_SHARED_LIBRARY_SUFFIX} ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy libnanovg${CMAKE_SHARED_LIBRARY_SUFFIX} ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy libportmidi${CMAKE_SHARED_LIBRARY_SUFFIX} ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy libsndfile${CMAKE_SHARED_LIBRARY_SUFFIX} ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy libstb_image${CMAKE_SHARED_LIBRARY_SUFFIX} ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy libglfw${CMAKE_SHARED_LIBRARY_SUFFIX} ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy libapr-1${CMAKE_SHARED_LIBRARY_SUFFIX} ${EXT_PLATFORM_SHLIBS_DIR}
      WORKING_DIRECTORY ${EXT_DEPS_INSTALL_DIR}/lib)
  elseif(WIN32)
    add_custom_target(extended_deps
      COMMENT "moving .dll and .lib files into ${EXT_PLATFORM_SHLIBS_DIR}"
      DEPENDS sndfile assimp glfw3 stb_image kiss_fft nanovg portmidi LLVM sndfile apr
      COMMAND ${CMAKE_COMMAND} -E make_directory ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy bin/assimp-vc130-mt.dll ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/kiss_fft.dll ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/nanovg.dll ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/portmidi.dll ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/stb_image.dll ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/glfw3.dll ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy bin/libapr-1.dll ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/assimp-vc130-mt.lib ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/kiss_fft.lib ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/nanovg.lib ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/portmidi.lib ${EXT_PLATFORM_SHLIBS_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy lib/stb_image.lib ${EXT_PLATFORM_SHLIBS_DIR}
      # glfw3 has different base names for the .dll and .lib, so fix that
      COMMAND ${CMAKE_COMMAND} -E copy lib/glfw3dll.lib ${EXT_PLATFORM_SHLIBS_DIR}/glfw3.lib
      COMMAND ${CMAKE_COMMAND} -E copy lib/libapr-1.lib ${EXT_PLATFORM_SHLIBS_DIR}
      WORKING_DIRECTORY ${EXT_DEPS_INSTALL_DIR})
  endif()

endif(BUILD_DEPS)

add_dependencies(aot_extended extended_deps)

#########
# tests #
#########

include(CTest)

if(BUILD_TESTING)

  # tests - core
  add_test(NAME system
    COMMAND extempore --noaudio --port=17099 --eval "(xtmtest-run-tests \"tests/core/system.xtm\" #t #t)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME adt
    COMMAND extempore --noaudio --port=17097 --eval "(xtmtest-run-tests \"tests/core/adt.xtm\" #t #t)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME math
    COMMAND extempore --noaudio --port=17095 --eval "(xtmtest-run-tests \"tests/core/math.xtm\" #t #t)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME std
    COMMAND extempore --noaudio --port=17093 --eval "(xtmtest-run-tests \"tests/core/std.xtm\" #t #t)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME xtlang
    COMMAND extempore --noaudio --port=17091 --eval "(xtmtest-run-tests \"tests/core/xtlang.xtm\" #t #t)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME generics
    COMMAND extempore --noaudio --port=17089 --eval "(xtmtest-run-tests \"tests/core/generics.xtm\" #t #t)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})

  # tests - external
  add_test(NAME fft
    COMMAND extempore --noaudio --port=17087 --eval "(xtmtest-run-tests \"tests/external/fft.xtm\" #t #t)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})

  # examples - core
  add_test(NAME fmsynth
    COMMAND extempore --port=17085 --eval "(sys:load-then-quit \"examples/core/fmsynth.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME xthread
    COMMAND extempore --port=17083 --eval "(sys:load-then-quit \"examples/core/xthread.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME scheduler
    COMMAND extempore --port=17081 --eval "(sys:load-then-quit \"examples/core/scheduler.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME nbody_lang_shootout
    COMMAND extempore --port=17079 --eval "(sys:load-then-quit \"examples/core/nbody_lang_shootout.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})

  # examples - external
  add_test(NAME electrofunk
    COMMAND extempore --port=17077 --eval "(sys:load-then-quit \"examples/external/electrofunk.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME triangle
    COMMAND extempore --port=17069 --eval "(sys:load-then-quit \"examples/external/shader-tutorials/triangle.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME raymarcher
    COMMAND extempore --port=17075 --eval "(sys:load-then-quit \"examples/external/raymarcher.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME spectrogram
    COMMAND extempore --port=17073 --eval "(sys:load-then-quit \"examples/external/spectrogram.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME gl-compatibility
    COMMAND extempore --port=17067 --eval "(sys:load-then-quit \"examples/external/gl-compatibility.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME xtmrender3
    COMMAND extempore --port=17071 --eval "(sys:load-then-quit \"examples/external/xtmrender3.xtm\" 10)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
  add_test(NAME going-native
    COMMAND extempore --port=17065 --eval "(sys:load-then-quit \"examples/external/going-native.xtm\" 60)"
    WORKING_DIRECTORY ${EXT_SHARE_DIR})
endif()

##########
# xtmdoc #
##########

add_custom_target(xtmdoc
  COMMAND extempore
           --port 17095
           --eval "(begin (sys:load \"libs/core/audio_dsp.xtm\") (sys:load \"libs/core/instruments.xtm\") (sys:load \"libs/core/math.xtm\") (sys:load \"libs/base/base.xtm\") (sys:load \"libs/external/fft.xtm\") (sys:load \"libs/external/gl.xtm\") (sys:load \"libs/external/glfw3.xtm\") (sys:load \"libs/external/instruments_ext.xtm\") (sys:load \"libs/external/nanovg.xtm\") (sys:load \"libs/external/sndfile.xtm\") (sys:load \"libs/external/stb_image.xtm\") (xtmdoc-export-caches-to-json \"/tmp/xtmdoc.json\" #f) (quit 0))"
  COMMENT "Generating xtmdoc output in /tmp/xtmdoc.json"
  VERBATIM)

add_dependencies(xtmdoc extempore)

#########
# cpack #
#########

# cpack is cmake's tool for providing distributable
# binaries/installers on various platforms.

set(CPACK_PACKAGE_NAME "Extempore")
set(CPACK_PACKAGE_VENDOR "Andrew Sorensen")
set(CPACK_PACKAGE_CONTACT "Ben Swift")
set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})

# give the zipball a nice name on each platform
if(APPLE)
  set(CPACK_PACKAGE_FILE_NAME Extempore-${PROJECT_VERSION}-osx10.9+)
elseif(UNIX)
  set(CPACK_PACKAGE_FILE_NAME Extempore-${PROJECT_VERSION}-${EXTEMPORE_SYSTEM_NAME})
elseif(WIN32)
  set(CPACK_PACKAGE_FILE_NAME Extempore-${PROJECT_VERSION}-win${EXTEMPORE_SYSTEM_VERSION})
endif()

set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "The Extempore programming environment (http://extempore.moso.com.au)")

if(APPLE)
  set(CPACK_GENERATOR DragNDrop)
elseif(UNIX AND NOT APPLE)
  set(CPACK_GENERATOR TGZ)
elseif(WIN32)
  set(CPACK_GENERATOR ZIP)
  # set(CPACK_GENERATOR NSIS64)
endif()

include(CPack)
