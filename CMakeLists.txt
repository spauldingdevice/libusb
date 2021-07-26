cmake_minimum_required(VERSION 3.10)
project(libusb C)

include(CheckCXXCompilerFlag)
include(CheckFunctionExists)
include(CheckSymbolExists)
include(CheckStructHasMember)
include(CheckIncludeFiles)
include(CheckTypeSize)

if(APPLE)
	set(OS_SOURCES
			libusb/os/darwin_usb.h
			libusb/os/darwin_usb.c
			)
	find_library(COREFOUNDATION_LIBRARY CoreFoundation)
	find_library(IOKIT_LIBRARY IOKit)
	find_library(OBJC_LIBRARY objc)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "HAIKU")
	set(OS_SOURCES
			libusb/os/haiku_usb.h
			libusb/os/haiku_usb_backend.cpp
			libusb/os/haiku_usb_raw.h
			libusb/os/haiku_usb_raw.cpp
			)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
	set(OS_SOURCES
			libusb/os/linux_usbfs.h
			libusb/os/linux_usbfs.c
			)
	set(_GNU_SOURCE 1 CACHE INTERNAL "" FORCE)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
	set(OS_SOURCES
			libusb/os/netbsd_usb.c
			)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD")
	set(OS_SOURCES
			libusb/os/openbsd_usb.c
			)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
	set(OS_SOURCES
			libusb/os/sunos_usb.h
			libusb/os/sunos_usb.c
			)
elseif(WIN32 OR CYGWIN)
	set(OS_SOURCES
			libusb/libusb-1.0.def
			libusb/libusb-1.0.rc
			libusb/os/windows_common.h
			libusb/os/windows_common.c
			libusb/os/windows_usbdk.h 
			libusb/os/windows_usbdk.c
			libusb/os/windows_winusb.h
			libusb/os/windows_winusb.c
			)
else()
	set(OS_SOURCES
			libusb/os/null_usb.c
			)
endif()

if(UNIX)
	set(PLATFORM_SOURCES
			libusb/os/events_posix.h 
			libusb/os/events_posix.c
			libusb/os/threads_posix.h 
			libusb/os/threads_posix.c
			)
	set(PLATFORM_POSIX 1 CACHE INTERNAL "" FORCE)
	check_cxx_compiler_flag("-fvisibility=hidden" HAVE_VISIBILITY)
	find_package(Threads REQUIRED)
elseif(WIN32)
	set(PLATFORM_SOURCES
			libusb/os/events_windows.h 
			libusb/os/events_windows.c
			libusb/os/threads_windows.h 
			libusb/os/threads_windows.c
			)
	set(PLATFORM_WINDOWS 1 CACHE INTERNAL "" FORCE)
endif()

if(HAVE_VISIBILITY)
	set(DEFAULT_VISIBILITY "__attribute__((visibility(\"default\")))" CACHE INTERNAL "visibility attribute to function decl" FORCE)
else()
	set(DEFAULT_VISIBILITY "" CACHE INTERNAL "visibility attribute to function decl" FORCE)
endif()

option(ENABLE_DEBUG_LOGGING "Enable debug message logging" ON)
option(ENABLE_LOGGING "Enable message logging" ON)
option(USE_SYSTEM_LOGGING_FACILITY "Output logging messages to the systemwide log" OFF)

check_include_files(asm/types.h HAVE_ASM_TYPES_H)
check_function_exists(clock_gettime HAVE_CLOCK_GETTIME)
check_function_exists(eventfd HAVE_EVENTFD)
check_function_exists(pipe2 HAVE_PIPE2)
check_function_exists(pthread_condattr_setclock HAVE_PTHREAD_CONDATTR_SETCLOCK)
check_function_exists(pthread_setname_np HAVE_PTHREAD_SETNAME_NP)
check_function_exists(pthread_threadid_np HAVE_PTHREAD_THREADID_NP)
check_include_files(string.h HAVE_STRING_H)
check_struct_has_member("struct timespec" tv_sec time.h HAVE_STRUCT_TIMESPEC)
check_function_exists(syslog HAVE_SYSLOG)
check_include_files(sys/time.h HAVE_SYS_TIME_H)
check_symbol_exists(timerfd_create "sys/timerfd.h" HAVE_TIMERFD) 

configure_file(libusb/config.h.in "${CMAKE_CURRENT_BINARY_DIR}/config.h" @ONLY)

set(libusb_SOURCES
		libusb/core.c
		libusb/descriptor.c
		libusb/hotplug.c
		libusb/io.c
		libusb/libusb.h
		libusb/libusbi.h
		libusb/strerror.c
		libusb/sync.c
		libusb/version.h
		libusb/version_nano.h
		)

add_library(libusb ${libusb_SOURCES} ${OS_SOURCES} ${PLATFORM_SOURCES})
target_link_libraries(libusb
		PUBLIC
		$<$<BOOL:${PLATFORM_POSIX}>:Threads::Threads>
		$<$<PLATFORM_ID:Darwin>:${COREFOUNDATION_LIBRARY} ${IOKIT_LIBRARY} ${OBJC_LIBRARY}>
		)
target_compile_definitions(libusb PRIVATE 
		$<$<PLATFORM_ID:Windows>:_WIN32_WINNT=0x0600>
		)
target_compile_options(libusb
		PUBLIC
		$<$<CXX_COMPILER_ID:MSVC>:/wd4127 /wd4200 /wd4201 /wd4324 /wd4996> #/analyze: /wd28719 /wd28125
		)
target_include_directories(libusb
		PRIVATE
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libusb/os>
		PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libusb>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
		${CMAKE_CURRENT_BINARY_DIR}
		)
