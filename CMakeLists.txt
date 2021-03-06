# FreeRDP: A Remote Desktop Protocol Implementation
# FreeRDP cmake build script
#
# Copyright 2011 O.S. Systems Software Ltda.
# Copyright 2011 Otavio Salvador <otavio@ossystems.com.br>
# Copyright 2011 Marc-Andre Moreau <marcandre.moreau@gmail.com>
# Copyright 2012 HP Development Company, LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 2.8)

project(FreeRDP C)

set(CMAKE_COLOR_MAKEFILE ON)

# Include our extra modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/)

# Check for cmake compatibility (enable/disable features)
include(CheckCmakeCompat)

# Include cmake modules
include(CheckIncludeFiles)
include(CheckLibraryExists)
include(CheckStructHasMember)
include(CMakeDetermineSystem)
include(FindPkgConfig)
include(TestBigEndian)

include(FindFeature)
include(AutoVersioning)
include(ConfigOptions)
include(ComplexLibrary)
include(FeatureSummary)
include(CheckCCompilerFlag)
include(GNUInstallDirsWrapper)

# Soname versioning
set(FREERDP_VERSION_MAJOR "1")
set(FREERDP_VERSION_MINOR "1")
set(FREERDP_VERSION_REVISION "0")
set(FREERDP_VERSION_SUFFIX "beta1")
set(FREERDP_API_VERSION "${FREERDP_VERSION_MAJOR}.${FREERDP_VERSION_MINOR}")
set(FREERDP_VERSION "${FREERDP_API_VERSION}.${FREERDP_VERSION_REVISION}")
if (FREERDP_VERSION_SUFFIX)
	set(FREERDP_VERSION_FULL "${FREERDP_VERSION}-${FREERDP_VERSION_SUFFIX}")
else()
	set(FREERDP_VERSION_FULL "${FREERDP_VERSION}")
endif()

# Allow to search the host machine for git
if(ANDROID OR IOS)
	SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER, BOTH)
endif(ANDROID OR IOS)
include(GetGitRevisionDescription)
if(ANDROID OR IOS)
	SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER, ONLY)
endif(ANDROID OR IOS)

git_describe(GIT_REVISION --match "[0-9]*" --abbrev=4 --tags --always)
message(STATUS "Git Revision ${GIT_REVISION}")

# Turn on solution folders (2.8.4+)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# Default to release build type
if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE "Release")
endif()

if(NOT DEFINED WITH_CLIENT_INTERFACE)
	set(WITH_CLIENT_INTERFACE ON)
endif()

if(NOT DEFINED WITH_SERVER_INTERFACE)
	set(WITH_SERVER_INTERFACE ON)
endif()

if(NOT DEFINED BUILD_SHARED_LIBS)
	if(ANDROID OR IOS)
		set(BUILD_SHARED_LIBS OFF)
	else()
		set(BUILD_SHARED_LIBS ON)
	endif()
endif()

if(NOT DEFINED EXPORT_ALL_SYMBOLS)
	set(EXPORT_ALL_SYMBOLS TRUE)
endif()

# Configure MSVC Runtime
if(MSVC)
	include(MSVCRuntime)
	if(NOT DEFINED MSVC_RUNTIME)
		set(MSVC_RUNTIME "dynamic")
	endif()
	if(${MSVC_RUNTIME} STREQUAL "static")
		message(STATUS "Use the MSVC static runtime option carefully!")
		message(STATUS "OpenSSL uses /MD by default, and is very picky")
		message(STATUS "Random freeing errors are a common sign of runtime issues")
	endif()
	configure_msvc_runtime()

	if(NOT DEFINED CMAKE_SUPPRESS_REGENERATION)
		set(CMAKE_SUPPRESS_REGENERATION ON)
	endif()
endif()

# Compiler-specific flags
if(CMAKE_COMPILER_IS_GNUCC)
	if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64" OR CMAKE_SYSTEM_PROCESSOR MATCHES "i686")
		if(NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=i686")
		endif()
	endif()
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")
	CHECK_C_COMPILER_FLAG (-Wno-unused-result Wno-unused-result)
	if(Wno-unused-result)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unused-result")
	endif()
	CHECK_C_COMPILER_FLAG (-Wno-unused-but-set-variable Wno-unused-but-set-variable)
	if(Wno-unused-but-set-variable)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unused-but-set-variable")
	endif()
	CHECK_C_COMPILER_FLAG(-Wno-deprecated-declarations Wno-deprecated-declarations)
	if(Wno-deprecated-declarations)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-deprecated-declarations")
	endif()
	if(NOT EXPORT_ALL_SYMBOLS)
		message(STATUS "GCC default symbol visibility: hidden")
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
	endif()
	if(BUILD_TESTING)
		CHECK_C_COMPILER_FLAG(-Wno-format Wno-format)
		if(Wno-format)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-format")
		endif()
	endif()
	if(CMAKE_BUILD_TYPE STREQUAL "Release")
		set(CMAKE_C_FLAGS_RELEASE "-DNDEBUG")
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2")
	endif()
	if(${CMAKE_VERSION} VERSION_LESS 2.8.8)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
	endif()
	if(WITH_SSE2)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse2")
	endif()
endif()

if(MSVC)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /Gd")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /MT")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /O2")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /Ob2")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /W2")
	if(CMAKE_SIZEOF_VOID_P EQUAL 8)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_AMD64_")
	else()
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_X86_")
	endif()
	SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR})
	SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR})
endif()

if(WIN32)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DUNICODE -D_UNICODE")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_WIN32_WINNT=0x0501")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_CRT_SECURE_NO_WARNINGS")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DWIN32_LEAN_AND_MEAN")	
endif()

if(IOS)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -isysroot ${CMAKE_IOS_SDK_ROOT} -g")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DWINPR_EXPORTS")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DFREERDP_EXPORTS")

# Include files
if(NOT IOS)
check_include_files(fcntl.h HAVE_FCNTL_H)
check_include_files(unistd.h HAVE_UNISTD_H)
check_include_files(stdint.h HAVE_STDINT_H)
check_include_files(inttypes.h HAVE_INTTYPES_H)
check_include_files(sys/modem.h HAVE_SYS_MODEM_H)
check_include_files(sys/filio.h HAVE_SYS_FILIO_H)
check_include_files(sys/strtio.h HAVE_SYS_STRTIO_H)
else()
set(HAVE_FCNTL_H 1)
set(HAVE_UNISTD_H 1)
set(HAVE_STDINT_H 1)
set(HAVE_INTTYPES_H 1)
set(HAVE_SYS_FILIO_H 1)
endif()

if(NOT IOS)
check_struct_has_member("struct tm" tm_gmtoff time.h HAVE_TM_GMTOFF)
else()
set(HAVE_TM_GMTOFF 1)
endif()

# Mac OS X
if(APPLE)
        if(IOS)
                if (NOT FREERDP_IOS_EXTERNAL_SSL_PATH)
                        message(STATUS "FREERDP_IOS_EXTERNAL_SSL_PATH not set! Required if openssl is not found in the iOS SDK (which usually isn't")
                endif()
                set(CMAKE_FIND_ROOT_PATH ${CMAKE_FIND_ROOT_PATH} ${FREERDP_IOS_EXTERNAL_SSL_PATH})
        	set_property(GLOBAL PROPERTY XCODE_ATTRIBUTE_SKIP_INSTALL YES)
	else()
                if(IS_DIRECTORY /opt/local/include)
                        include_directories(/opt/local/include)
                        link_directories(/opt/local/lib)
                endif()
                set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mmacosx-version-min=10.4")
        endif()
        if(WITH_CLANG)
                set(CMAKE_C_COMPILER "clang")
        endif()
endif()

# Android
if(ANDROID)
	if (NOT FREERDP_ANDROID_EXTERNAL_SSL_PATH)
		if(IS_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/external/openssl")
			set(FREERDP_ANDROID_EXTERNAL_SSL_PATH "${CMAKE_CURRENT_SOURCE_DIR}/external/openssl")
		else()
			message(STATUS "FREERDP_ANDROID_EXTERNAL_SSL_PATH not set! - Needs to be set if openssl is not found in the android NDK (which usually isn't)")
		endif()
	endif()
	set(CMAKE_FIND_ROOT_PATH ${CMAKE_FIND_ROOT_PATH} ${FREERDP_ANDROID_EXTERNAL_SSL_PATH})
	set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/client/Android/FreeRDPCore/libs/${ANDROID_ABI})
	CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/scripts/regenerate_jni_headers.sh.cmake ${CMAKE_CURRENT_SOURCE_DIR}/scripts/regenerate_jni_headers.sh @ONLY)
endif()

set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
if(NOT IOS AND NOT ANDROID)
	find_package(Threads REQUIRED)
endif()

if(UNIX OR CYGWIN)
	check_include_files(sys/eventfd.h HAVE_EVENTFD_H)
	set(X11_FEATURE_TYPE "RECOMMENDED")
else()
	set(X11_FEATURE_TYPE "DISABLED")
endif()

set(X11_FEATURE_PURPOSE "X11")
set(X11_FEATURE_DESCRIPTION "X11 client and server")

set(DIRECTFB_FEATURE_TYPE "OPTIONAL")
set(DIRECTFB_FEATURE_PURPOSE "DirectFB")
set(DIRECTFB_FEATURE_DESCRIPTION "DirectFB client")

set(ZLIB_FEATURE_TYPE "REQUIRED")
set(ZLIB_FEATURE_PURPOSE "compression")
set(ZLIB_FEATURE_DESCRIPTION "data compression")

set(OPENSSL_FEATURE_TYPE "REQUIRED")
set(OPENSSL_FEATURE_PURPOSE "cryptography")
set(OPENSSL_FEATURE_DESCRIPTION "encryption, certificate validation, hashing functions")

set(ALSA_FEATURE_TYPE "RECOMMENDED")
set(ALSA_FEATURE_PURPOSE "sound")
set(ALSA_FEATURE_DESCRIPTION "audio input, audio output and multimedia redirection")

set(PULSE_FEATURE_TYPE "OPTIONAL")
set(PULSE_FEATURE_PURPOSE "sound")
set(PULSE_FEATURE_DESCRIPTION "audio input, audio output and multimedia redirection")

set(CUPS_FEATURE_TYPE "OPTIONAL")
set(CUPS_FEATURE_PURPOSE "printing")
set(CUPS_FEATURE_DESCRIPTION "printer device redirection")

set(PCSC_FEATURE_TYPE "OPTIONAL")
set(PCSC_FEATURE_PURPOSE "smart card")
set(PCSC_FEATURE_DESCRIPTION "smart card device redirection")

set(FFMPEG_FEATURE_TYPE "RECOMMENDED")
set(FFMPEG_FEATURE_PURPOSE "multimedia")
set(FFMPEG_FEATURE_DESCRIPTION "multimedia redirection, audio and video playback")

set(GSTREAMER_FEATURE_TYPE "RECOMMENDED")
set(GSTREAMER_FEATURE_PURPOSE "multimedia")
set(GSTREAMER_FEATURE_DESCRIPTION "multimedia redirection, audio and video playback")

set(IPP_FEATURE_TYPE "OPTIONAL")
set(IPP_FEATURE_PURPOSE "performance")
set(IPP_FEATURE_DESCRIPTION "Intel Integrated Performance Primitives library")

if(WIN32)
	set(X11_FEATURE_TYPE "DISABLED")
	set(ZLIB_FEATURE_TYPE "DISABLED")
	set(DIRECTFB_FEATURE_TYPE "DISABLED")
	set(ALSA_FEATURE_TYPE "DISABLED")
	set(PULSE_FEATURE_TYPE "DISABLED")
	set(CUPS_FEATURE_TYPE "DISABLED")
	set(PCSC_FEATURE_TYPE "DISABLED")
	set(FFMPEG_FEATURE_TYPE "DISABLED")
	set(GSTREAMER_FEATURE_TYPE "DISABLED")
endif()

if(APPLE)
	set(DIRECTFB_FEATURE_TYPE "DISABLED")
	set(FFMPEG_FEATURE_TYPE "OPTIONAL")
	set(GSTREAMER_FEATURE_TYPE "OPTIONAL")
	if(IOS)
		set(X11_FEATURE_TYPE "DISABLED")
		set(ALSA_FEATURE_TYPE "DISABLED")
		set(PULSE_FEATURE_TYPE "DISABLED")
		set(CUPS_FEATURE_TYPE "DISABLED")
		set(PCSC_FEATURE_TYPE "DISABLED")
		set(GSTREAMER_FEATURE_TYPE "DISABLED")
	endif()
endif()

if(ANDROID)
	set(X11_FEATURE_TYPE "DISABLED")
	set(DIRECTFB_FEATURE_TYPE "DISABLED")
	set(ALSA_FEATURE_TYPE "DISABLED")
	set(PULSE_FEATURE_TYPE "DISABLED")
	set(CUPS_FEATURE_TYPE "DISABLED")
	set(PCSC_FEATURE_TYPE "DISABLED")
	set(FFMPEG_FEATURE_TYPE "DISABLED")
	set(GSTREAMER_FEATURE_TYPE "DISABLED")
endif()


find_feature(X11 ${X11_FEATURE_TYPE} ${X11_FEATURE_PURPOSE} ${X11_FEATURE_DESCRIPTION})
find_feature(DirectFB ${DIRECTFB_FEATURE_TYPE} ${DIRECTFB_FEATURE_PURPOSE} ${DIRECTFB_FEATURE_DESCRIPTION})

find_feature(ZLIB ${ZLIB_FEATURE_TYPE} ${ZLIB_FEATURE_PURPOSE} ${ZLIB_FEATURE_DESCRIPTION})
find_feature(OpenSSL ${OPENSSL_FEATURE_TYPE} ${OPENSSL_FEATURE_PURPOSE} ${OPENSSL_FEATURE_DESCRIPTION})

find_feature(ALSA ${ALSA_FEATURE_TYPE} ${ALSA_FEATURE_PURPOSE} ${ALSA_FEATURE_DESCRIPTION})
find_feature(Pulse ${PULSE_FEATURE_TYPE} ${PULSE_FEATURE_PURPOSE} ${PULSE_FEATURE_DESCRIPTION})

find_feature(Cups ${CUPS_FEATURE_TYPE} ${CUPS_FEATURE_PURPOSE} ${CUPS_FEATURE_DESCRIPTION})
find_feature(PCSC ${PCSC_FEATURE_TYPE} ${PCSC_FEATURE_PURPOSE} ${PCSC_FEATURE_DESCRIPTION})

find_feature(FFmpeg ${FFMPEG_FEATURE_TYPE} ${FFMPEG_FEATURE_PURPOSE} ${FFMPEG_FEATURE_DESCRIPTION})
find_feature(Gstreamer ${GSTREAMER_FEATURE_TYPE} ${GSTREAMER_FEATURE_PURPOSE} ${GSTREAMER_FEATURE_DESCRIPTION})

if(TARGET_ARCH MATCHES "x86|x64")
	if (NOT APPLE)
		# Intel Performance Primitives
		find_feature(IPP ${IPP_FEATURE_TYPE} ${IPP_FEATURE_PURPOSE} ${IPP_FEATURE_DESCRIPTION})
	endif()
endif()

# Installation Paths
if(WIN32)
	set(CMAKE_INSTALL_BINDIR ".")
	set(CMAKE_INSTALL_LIBDIR ".")
endif()

# Path to put FreeRDP data
set(FREERDP_DATA_PATH "${CMAKE_INSTALL_PREFIX}/share/freerdp")

# Path to put plugins

set(FREERDP_LIBRARY_PATH "${CMAKE_INSTALL_LIBDIR}")

set(FREERDP_PLUGIN_PATH "${CMAKE_INSTALL_LIBDIR}/freerdp")
set(FREERDP_ADDIN_PATH "${FREERDP_PLUGIN_PATH}")

# Path to put extensions
set(FREERDP_EXTENSION_PATH "${CMAKE_INSTALL_FULL_LIBDIR}/freerdp/extensions")

# Include directories
include_directories(${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

# Configure files
add_definitions("-DHAVE_CONFIG_H")
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_SOURCE_DIR}/config.h)

# RPATH configuration
set(CMAKE_SKIP_BUILD_RPATH FALSE)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
set(CMAKE_INSTALL_RPATH "\$ORIGIN/../${CMAKE_INSTALL_LIBDIR}:\$ORIGIN/..")

# Unit Tests

include(CTest)

if(BUILD_TESTING)	
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DFREERDP_TEST_EXPORTS")
	
	enable_testing()
	
	if(MSVC)
		set(TESTING_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}")
	else()
		set(TESTING_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/Testing")
	endif()
endif()

# WinPR
set(WINPR_INCLUDE_DIR "${CMAKE_SOURCE_DIR}/winpr/include")
include_directories(${WINPR_INCLUDE_DIR})

add_subdirectory(winpr)

# Generate pkg-config
if(NOT MSVC)
	configure_file(${CMAKE_CURRENT_SOURCE_DIR}/freerdp.pc.in ${CMAKE_CURRENT_BINARY_DIR}/freerdp.pc @ONLY)
	install(FILES ${CMAKE_CURRENT_BINARY_DIR}/freerdp.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif()

if(WITH_CUNIT)
	message(FATAL_ERROR "cunit (WITH_CUNIT) is deprecated please use BUILD_TESTING to build ctest tests.
The cunit directory contains the old tests and is kept until all tests are converted.")
endif()

# Sub-directories

if(WITH_THIRD_PARTY)
	add_subdirectory(third-party)
	if (NOT "${THIRD_PARTY_INCLUDES}" STREQUAL "")
		include_directories(${THIRD_PARTY_INCLUDES})
	endif()
endif()

add_subdirectory(include)

add_subdirectory(libfreerdp)

if(WITH_CHANNELS)
	add_subdirectory(channels)
endif()

if (IOS)
set(CMAKE_OSX_DEPLOYMENT_TARGET "")
set(CMAKE_OSX_SYSROOT "iphoneos")
endif()
if(WITH_CLIENT)
	add_subdirectory(client)
endif()

if(WITH_SERVER)
	add_subdirectory(server)
endif()

# Packaging

SET(CPACK_BINARY_ZIP "ON")

set(CPACK_SOURCE_IGNORE_FILES "/\\\\.git/;/\\\\.gitignore;/CMakeCache.txt")

if(NOT WIN32)
    if(APPLE AND (NOT IOS))
    #set(CPACK_PACKAGE_EXECUTABLES "mfreerdp")

        if(WITH_SERVER)
            set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_EXECUTABLES} "mfreerdp-server")
        endif()
	endif()
    
    if(WITH_X11)
        set(CPACK_PACKAGE_EXECUTABLES "xfreerdp")

        if(WITH_SERVER)
            set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_EXECUTABLES} "xfreerdp-server")
        endif()
    endif()


endif()

set(CPACK_SYSTEM_NAME "${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")
set(CPACK_TOPLEVEL_TAG "${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")

string(TOLOWER ${CMAKE_PROJECT_NAME} CMAKE_PROJECT_NAME_lower)
set(CPACK_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME_lower}-${FREERDP_VERSION_FULL}-${CPACK_SYSTEM_NAME}")
set(CPACK_SOURCE_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME_lower}-${FREERDP_VERSION_FULL}-${CPACK_SYSTEM_NAME}")

set(CPACK_PACKAGE_NAME "FreeRDP")
set(CPACK_PACKAGE_VENDOR "FreeRDP")
set(CPACK_PACKAGE_VERSION ${FREERDP_VERSION_FULL})
set(CPACK_PACKAGE_VERSION_MAJOR ${FREERDP_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${FREERDP_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${FREERDP_VERSION_REVISION})
SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY "FreeRDP: A Remote Desktop Protocol Implementation")

set(CPACK_PACKAGE_CONTACT "Marc-Andre Moreau")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "marcandre.moreau@gmail.com")
set(CPACK_DEBIAN_ARCHITECTURE ${CMAKE_SYSTEM_PROCESSOR})

set(CPACK_PACKAGE_INSTALL_DIRECTORY "FreeRDP")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_SOURCE_DIR}/LICENSE")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_SOURCE_DIR}/LICENSE")

set(CPACK_NSIS_MODIFY_PATH ON)
set(CPACK_PACKAGE_ICON "${CMAKE_SOURCE_DIR}/resources\\\\FreeRDP_Install.bmp")
set(CPACK_NSIS_MUI_ICON "${CMAKE_SOURCE_DIR}/resources\\\\FreeRDP_Icon_96px.ico")
set(CPACK_NSIS_MUI_UNICON "${CMAKE_SOURCE_DIR}/resource\\\\FreeRDP_Icon_96px.ico")

set(CPACK_COMPONENTS_ALL client server libraries headers)

if(MSVC)
	if(MSVC_RUNTIME STREQUAL "dynamic")
		set(CMAKE_INSTALL_SYSTEM_RUNTIME_LIBS_SKIP TRUE)
		include(InstallRequiredSystemLibraries)
		install(PROGRAMS ${CMAKE_INSTALL_SYSTEM_RUNTIME_LIBS}
			DESTINATION ${CMAKE_INSTALL_BINDIR}
			COMPONENT libraries)
    endif()
			
    set(EXTRA_DATA_DIR "extra/")
    file(GLOB EXTRA_FILES "${CMAKE_SOURCE_DIR}/extra/*")
    install(FILES ${EXTRA_FILES}
        DESTINATION ${EXTRA_DATA_DIR}
        COMPONENT extra)
			
    set(REV_DATA_DIR "REV/")
    file(GLOB REV_FILES "${CMAKE_SOURCE_DIR}/REV/*")
    install(FILES ${REV_FILES}
        DESTINATION ${REV_DATA_DIR}
        COMPONENT rev)
	
    set(CPACK_COMPONENTS_AL ${CPACK_COMPONENTS_ALL} extra rev)
endif()


set(CPACK_COMPONENT_CLIENT_DISPLAY_NAME "Client")
set(CPACK_COMPONENT_CLIENT_GROUP "Applications")

set(CPACK_COMPONENT_SERVER_DISPLAY_NAME "Server")
set(CPACK_COMPONENT_SERVER_GROUP "Applications")

set(CPACK_COMPONENT_LIBRARIES_DISPLAY_NAME "Libraries")
set(CPACK_COMPONENT_LIBRARIES_GROUP "Runtime")

set(CPACK_COMPONENT_HEADERS_DISPLAY_NAME "Headers")
set(CPACK_COMPONENT_HEADERS_GROUP "Development")

set(CPACK_COMPONENT_GROUP_RUNTIME_DESCRIPTION "Runtime")
set(CPACK_COMPONENT_GROUP_APPLICATIONS_DESCRIPTION "Applications")
set(CPACK_COMPONENT_GROUP_DEVELOPMENT_DESCRIPTION "Development")

# Workaround to remove c++ compiler macros and defines for Eclipse.
# If c++ macros/defines are set __cplusplus is also set which causes
# problems when compiling freerdp/jni. To prevent this problem
# we set the macros to "".
if (ANDROID AND CMAKE_EXTRA_GENERATOR STREQUAL "Eclipse CDT4")
  set(CMAKE_EXTRA_GENERATOR_CXX_SYSTEM_DEFINED_MACROS "")
  message(STATUS "Disabled CXX system defines for eclipse (workaround).")
endif()



include(CPack)
