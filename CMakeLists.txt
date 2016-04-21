#
# xUnit++ test framework
#
cmake_minimum_required(VERSION 3.1)
project(xUnitpp)

set(USE_ASAN 0 CACHE BOOL "Use address sanitizer")
set(USE_MSAN 0 CACHE BOOL "Use memory sanitizer")
set(USE_TSAN 0 CACHE BOOL "Use thread sanitizer")
set(USE_COVERAGE 0 CACHE BOOL "Use code coverage")

# Helpers
# -------------------------------------------------------------------------------------------------
function(globrecurse_c_cpp_sources result folder)
	file(GLOB_RECURSE res
		"${folder}/*.c"
		"${folder}/*.cpp"
		"${folder}/*.h"
		"${folder}/*.hpp")
	set(${result} ${res} PARENT_SCOPE)
endfunction()

function(xunit_compiler_options target)
	# Compiler-specific options
	if(WIN32)
		# Fast compilation
		target_compile_options(${target} PRIVATE "/MP")

		# No warning about not secure CRT functions
		target_compile_definitions(${target} PRIVATE _CRT_SECURE_NO_WARNINGS)
	endif()

	if(MINGW OR CMAKE_COMPILER_IS_GNUCXX OR ${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang")
		# Remove __stdcall
		target_compile_definitions(${target} PUBLIC "__stdcall=")

		if (USE_ASAN)
			target_compile_options(${target} PRIVATE "-fsanitize=address")
			target_link_libraries(${target} PRIVATE "-fsanitize=address")
		endif()
		if (USE_MSAN)
			target_compile_options(${target} PRIVATE "-fsanitize=memory")
			target_link_libraries(${target} PRIVATE "-fsanitize=memory")
		endif()
		if (USE_TSAN)
			target_compile_options(${target} PRIVATE "-fsanitize=thread")
			target_link_libraries(${target} PRIVATE "-fsanitize=thread")
		endif()
		if (USE_COVERAGE)
			target_compile_options(${target} PRIVATE "-coverage")
			target_link_libraries(${target} PRIVATE "-fprofile-arcs")
		endif()
		# On linux 64bit we should use position independent code, as we may get linked in a .so
		if(${CMAKE_SIZEOF_VOID_P} EQUAL 8)
			set_target_properties(${target} PROPERTIES POSITION_INDEPENDENT_CODE ON)
		endif()
	endif()
endfunction()

# Folders
# -------------------------------------------------------------------------------------------------
set(XUNITPP_ROOT_DIR "${CMAKE_CURRENT_LIST_DIR}")

# Core target
# -------------------------------------------------------------------------------------------------
# Sources
globrecurse_c_cpp_sources(XUNITPP_SRC_FILES_CORE ${XUNITPP_ROOT_DIR}/xUnit++)
source_group(Source FILES ${XUNITPP_SRC_FILES_CORE})

# Static library, but ideally should be shared
add_library(xunitpp_core STATIC ${XUNITPP_SRC_FILES_CORE})
target_include_directories(xunitpp_core
	SYSTEM PUBLIC ${XUNITPP_ROOT_DIR}/xUnit++
	PRIVATE ${XUNITPP_ROOT_DIR}/xUnit++/xUnit++)
if(NOT MSVC)
	target_link_libraries(xunitpp_core PRIVATE pthread m stdc++)
endif()
set_target_properties(xunitpp_core PROPERTIES CXX_STANDARD 14 CXX_STANDARD_REQUIRED ON)
xunit_compiler_options(xunitpp_core)

# Utility target
# -------------------------------------------------------------------------------------------------
# Sources
globrecurse_c_cpp_sources(XUNITPP_SRC_FILES_UTIL ${XUNITPP_ROOT_DIR}/xUnit++.Utility)
source_group(Source FILES ${XUNITPP_SRC_FILES_UTIL})

# Library
add_library(xunitpp_util STATIC ${XUNITPP_SRC_FILES_UTIL})
target_include_directories(xunitpp_util SYSTEM PUBLIC ${XUNITPP_ROOT_DIR}/xUnit++.Utility)
target_link_libraries(xunitpp_util PRIVATE xunitpp_core)
set_target_properties(xunitpp_util PROPERTIES CXX_STANDARD 14 CXX_STANDARD_REQUIRED ON)
xunit_compiler_options(xunitpp_util)

# Console target
# -------------------------------------------------------------------------------------------------
# Sources
globrecurse_c_cpp_sources(XUNITPP_SRC_FILES_CONSOLE ${XUNITPP_ROOT_DIR}/xUnit++.console)
source_group(Source FILES ${XUNITPP_SRC_FILES_CONSOLE})

# Executable
add_executable(xunitpp_console ${XUNITPP_SRC_FILES_CONSOLE})
target_include_directories(xunitpp_console PRIVATE ${XUNITPP_ROOT_DIR}/xUnit++.console)
target_link_libraries(xunitpp_console PRIVATE xunitpp_core xunitpp_util)
if(UNIX)
	target_link_libraries(xunitpp_console PRIVATE dl)
endif()
set_target_properties(xunitpp_console PROPERTIES CXX_STANDARD 14 CXX_STANDARD_REQUIRED ON)
xunit_compiler_options(xunitpp_console)

