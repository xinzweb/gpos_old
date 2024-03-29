# Copyright (c) 2015, Pivotal Software, Inc. All Rights Reserved.

cmake_minimum_required(VERSION 3.0 FATAL_ERROR)
project(gpos LANGUAGES CXX C)

set(GPOS_VERSION_MAJOR 1)
set(GPOS_VERSION_MINOR 135)
set(GPOS_VERSION_STRING ${GPOS_VERSION_MAJOR}.${GPOS_VERSION_MINOR})

# Whenever an ABI-breaking change is made to GPOS, this should be incremented.
# ABI changes include removing functions, and adding or removing function
# parameters. Because GPOS is a C++ library, there are also several other cases
# that might cause ABI changes, including adding or removing class members,
# and things that might change vtables for classes with virtual methods. If in
# doubt, do the safe thing and increment this number.
set(GPOS_ABI_VERSION 10)

# Check build type
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo")
endif(NOT CMAKE_BUILD_TYPE)
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

# Override CMAKE_SYSTEM_PROCESSOR if it has been explicitly set in a toolchain file.
if (FORCED_CMAKE_SYSTEM_PROCESSOR)
  set(CMAKE_SYSTEM_PROCESSOR ${FORCED_CMAKE_SYSTEM_PROCESSOR})
elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "amd64")
  # Use a standardized name for 64-bit x86.
  set(CMAKE_SYSTEM_PROCESSOR "x86_64")
endif()

# Prevent linking errors for builtins if using an old GCC with the i386
# toolchain file.
if (ENABLE_OLD_GCC_32BIT_MARCH
    AND CMAKE_COMPILER_IS_GNUCXX
    AND (CMAKE_CXX_COMPILER_VERSION VERSION_LESS "4.4.4"))
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=i686")
endif()

# Turn on compiler flags for all warnings if available.
include(CheckCXXCompilerFlag)

check_cxx_compiler_flag("-Wall" COMPILER_HAS_WALL)
if (COMPILER_HAS_WALL)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
endif()

check_cxx_compiler_flag("-Werror" COMPILER_HAS_WERROR)
if (COMPILER_HAS_WERROR)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
endif()

check_cxx_compiler_flag("-Wextra" COMPILER_HAS_WEXTRA)
if (COMPILER_HAS_WEXTRA)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wextra")
endif()

check_cxx_compiler_flag("-pedantic-errors" COMPILER_HAS_PEDANTIC_ERRORS)
if (COMPILER_HAS_PEDANTIC_ERRORS)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic-errors")
endif()

# Turn off warnings about variadic macros if for some reason the C++ dialect is
# not compatible with C99.
check_cxx_compiler_flag("-Wno-variadic-macros" COMPILER_HAS_WNO_VARIADIC_MACROS)
if (COMPILER_HAS_WNO_VARIADIC_MACROS)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-variadic-macros")
endif()

# Turn off warning about tautological compare in CDynamicPtrArray.
# TODO(chasseur): CDynamicPtrArray::UlSafeLength() should be rewritten so that
# it does not trigger this error in Clang.
check_cxx_compiler_flag("-Wno-tautological-undefined-compare"
                        COMPILER_HAS_WNO_TAUTOLOGICAL_UNDEFINED_COMPARE)
if (COMPILER_HAS_WNO_TAUTOLOGICAL_UNDEFINED_COMPARE)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-tautological-undefined-compare")
endif()

# Generate maximum detail for DEBUG information with -g3 if available.
check_cxx_compiler_flag("-g3" COMPILER_HAS_G3)
if (COMPILER_HAS_G3)
  SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g3")
  SET(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} -g3")
endif()

# Do not omit frame pointer. Even with RELEASE builds, it is used for
# backtracing.
check_cxx_compiler_flag("-fno-omit-frame-pointer"
                        COMPILER_HAS_FNO_OMIT_FRAME_POINTER)
if (COMPILER_HAS_FNO_OMIT_FRAME_POINTER)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-omit-frame-pointer")
endif()

# Turn on GPOS_DEBUG define for DEBUG builds.
cmake_policy(SET CMP0043 NEW)
set_property(
    DIRECTORY
    APPEND PROPERTY COMPILE_DEFINITIONS $<$<CONFIG:Debug>:GPOS_DEBUG>)

# Turn on platform-specific defines.
set_property(
    DIRECTORY
    APPEND PROPERTY COMPILE_DEFINITIONS
        GPOS_${CMAKE_SYSTEM_NAME}
        GPOS_${CMAKE_SYSTEM_PROCESSOR})

# Autodetect bit-width if not already set by toolchain file.
if (NOT GPOS_ARCH_BITS)
  # Autodetect bit-width.
  if (${CMAKE_SIZEOF_VOID_P} EQUAL 8)
    set(GPOS_ARCH_BITS 64)
  elseif (${CMAKE_SIZEOF_VOID_P} EQUAL 4)
    set(GPOS_ARCH_BITS 32)
    if (${CMAKE_SYSTEM_PROCESSOR} MATCHES "x86_64")
      message(WARNING "Detected x86-64 processor, but pointer size is 32 bits. "
                      "Compilation may fail because of this inconsistency (using "
                      "one of the toolchain files to explicitly target a specific "
                      "CPU architecture may avoid this problem).")
    endif()
  else()
    message(FATAL_ERROR "Could not detect 32-bit OR 64-bit architecture")
  endif()
endif()

set_property(
    DIRECTORY
    APPEND PROPERTY COMPILE_DEFINITIONS
        GPOS_${GPOS_ARCH_BITS}BIT)

# Need pthreads.
find_package(Threads REQUIRED)
if (NOT CMAKE_USE_PTHREADS_INIT)
  message(FATAL_ERROR "Found a threading library, but it is not pthreads.")
endif()

# Try to find atomic operations.
include(FindAtomics.cmake)

if ((NOT (COMPILER_HAS_FETCH_ADD_32 AND COMPILER_HAS_FETCH_ADD_64
          AND COMPILER_HAS_CAS_32 AND COMPILER_HAS_CAS_64))
    AND (NOT (${CMAKE_SYSTEM_NAME} MATCHES "SunOS")))
  message(FATAL_ERROR "Could not find GCC-style atomic built-ins or Solaris "
                      "atomic headers. GPOS will fail to build. Please try "
                      "using a recent g++ or clang++ compiler.")
endif()


include_directories(libgpos/include)

# Generate version-number header.
configure_file(version.h.in
               ${PROJECT_BINARY_DIR}/libgpos/include/gpos/version.h)

# Compile .cpp source files under libgpos/src into monolithic gpos library.
add_library(gpos SHARED
            libgpos/src/_api.cpp
            libgpos/src/utils.cpp
            libgpos/src/common/CAutoTimer.cpp
            libgpos/src/common/CBitSet.cpp
            libgpos/src/common/CBitSetIter.cpp
            libgpos/src/common/CBitVector.cpp
            libgpos/src/common/CHeapObject.cpp
            libgpos/src/common/clibwrapper.cpp
            libgpos/src/common/CMainArgs.cpp
            libgpos/src/common/CRandom.cpp
            libgpos/src/common/CStackDescriptor.cpp
            libgpos/src/common/CStackObject.cpp
            libgpos/src/common/CTimerUser.cpp
            libgpos/src/common/CWallClock.cpp
            libgpos/src/common/pthreadwrapper.cpp
            libgpos/src/common/syslibwrapper.cpp
            libgpos/src/error/CAutoExceptionStack.cpp
            libgpos/src/error/CAutoTrace.cpp
            libgpos/src/error/CErrorContext.cpp
            libgpos/src/error/CErrorHandlerStandard.cpp
            libgpos/src/error/CException.cpp
            libgpos/src/error/CFSimulator.cpp
            libgpos/src/error/CLogger.cpp
            libgpos/src/error/CLoggerFile.cpp
            libgpos/src/error/CLoggerStream.cpp
            libgpos/src/error/CLoggerSyslog.cpp
            libgpos/src/error/CMessage.cpp
            libgpos/src/error/CMessageRepository.cpp
            libgpos/src/error/CMessageTable.cpp
            libgpos/src/error/CMiniDumper.cpp
            libgpos/src/error/CSerializable.cpp
            libgpos/src/error/ILogger.cpp
            libgpos/src/io/CFileDescriptor.cpp
            libgpos/src/io/CFileReader.cpp
            libgpos/src/io/CFileWriter.cpp
            libgpos/src/io/COstream.cpp
            libgpos/src/io/COstreamBasic.cpp
            libgpos/src/io/COstreamFile.cpp
            libgpos/src/io/COstreamString.cpp
            libgpos/src/io/ioutils.cpp
            libgpos/src/memory/CAutoMemoryPool.cpp
            libgpos/src/memory/CCacheFactory.cpp
            libgpos/src/memory/CMemoryPool.cpp
            libgpos/src/memory/CMemoryPoolInjectFault.cpp
            libgpos/src/memory/CMemoryPoolManager.cpp
            libgpos/src/memory/CMemoryPoolSlab.cpp
            libgpos/src/memory/CMemoryPoolStack.cpp
            libgpos/src/memory/CMemoryPoolTracker.cpp
            libgpos/src/memory/CMemoryVisitorPrint.cpp
            libgpos/src/memory/CSlab.cpp
            libgpos/src/memory/CSlabCache.cpp
            libgpos/src/memory/IMemoryPool.cpp
            libgpos/src/net/CSocket.cpp
            libgpos/src/net/CSocketConnectorUDS.cpp
            libgpos/src/net/CSocketListener.cpp
            libgpos/src/net/CSocketListenerUDS.cpp
            libgpos/src/net/netutils.cpp
            libgpos/src/string/CStringStatic.cpp
            libgpos/src/string/CWString.cpp
            libgpos/src/string/CWStringBase.cpp
            libgpos/src/string/CWStringConst.cpp
            libgpos/src/string/CWStringDynamic.cpp
            libgpos/src/string/CWStringStatic.cpp
            libgpos/src/sync/atomic.cpp
            libgpos/src/sync/CAutoMutex.cpp
            libgpos/src/sync/CAutoSpinlock.cpp
            libgpos/src/sync/CEvent.cpp
            libgpos/src/task/CAutoSuspendAbort.cpp
            libgpos/src/task/CAutoTaskProxy.cpp
            libgpos/src/task/CAutoTraceFlag.cpp
            libgpos/src/task/CTask.cpp
            libgpos/src/task/CTaskContext.cpp
            libgpos/src/task/CTaskLocalStorage.cpp
            libgpos/src/task/CTaskSchedulerFifo.cpp
            libgpos/src/task/CThreadManager.cpp
            libgpos/src/task/CWorker.cpp
            libgpos/src/task/CWorkerId.cpp
            libgpos/src/task/CWorkerPoolManager.cpp
            libgpos/src/task/ITask.cpp
            libgpos/src/task/IWorker.cpp
            libgpos/src/test/CFSimulatorTestExt.cpp
            libgpos/src/test/CTimeSliceTest.cpp
            libgpos/src/test/CUnittest.cpp)
target_link_libraries(gpos ${CMAKE_THREAD_LIBS_INIT} ${CMAKE_DL_LIBS})

# Extra system libs needed on Solaris.
if (${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
  find_library(RT_LIBRARY NAMES rt librt)
  if (${RT_LIBRARY-NOTFOUND})
    message(WARNING "rt library not found. Linking may fail.")
  else()
    target_link_libraries(gpos ${RT_LIBRARY})
  endif()

  find_library(SOCKET_LIBRARY NAMES socket libsocket)
  if (${SOCKET_LIBRARY-NOTFOUND})
    message(WARNING "socket library not found. Linking may fail.")
  else()
    target_link_libraries(gpos ${SOCKET_LIBRARY})
  endif()
endif()

set_target_properties(gpos PROPERTIES
                      SOVERSION ${GPOS_ABI_VERSION}
                      VERSION ${GPOS_VERSION_STRING})

# Tests reside in the 'server' subdirectory.
enable_testing()
add_subdirectory(server)

# Installation.
option(VERBOSE_INSTALL_PATH "Install in a subdirectory path that includes GPOS version, CPU architecture, and bit width" OFF)
if (VERBOSE_INSTALL_PATH)
  set(installpath "libgpos/${GPOS_VERSION_STRING}/${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}/${CMAKE_BUILD_TYPE}")
  string(TOLOWER ${installpath} installpath)
  get_filename_component(full_install_name_dir "${installpath}/lib" ABSOLUTE)
  install(TARGETS gpos DESTINATION "${installpath}/lib")
  install(DIRECTORY libgpos/include/gpos DESTINATION "${installpath}/include")
  install(FILES "${PROJECT_BINARY_DIR}/libgpos/include/gpos/version.h"
          DESTINATION "${installpath}/include/gpos")
else()
  get_filename_component(full_install_name_dir "${CMAKE_INSTALL_PREFIX}/lib" ABSOLUTE)
  install(TARGETS gpos DESTINATION lib)
  install(DIRECTORY libgpos/include/gpos DESTINATION include)
  install(FILES "${PROJECT_BINARY_DIR}/libgpos/include/gpos/version.h"
          DESTINATION include/gpos)
endif()

# Mac OS X handles searching for dynamic libraries differently from other
# unices. It does not merely search for the leaf filename of a .dylib in a list
# of paths known to the linker, but instead looks up libraries based on either
# their full absolute path, or a relative path. We set the INSTALL_NAME_DIR
# property here to force the former behavior so that anything linking against
# gpos will look for the shared library in its absolute install path.
set_target_properties(gpos PROPERTIES
                      INSTALL_NAME_DIR "${full_install_name_dir}")

# Uninstallation.
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)

add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
