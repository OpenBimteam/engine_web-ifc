# set up cmake to download the external projects
include(FetchContent)
cmake_policy(SET CMP0054 NEW)
cmake_minimum_required(VERSION 3.18)

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
	message("Building In Debug Mode")
	set(RELEASE false)
	set(DEBUG_FLAG "-g -O3")
else()
	set(DEBUG_FLAG "-O3")
	set(RELEASE true)
endif()

project(web-ifc LANGUAGES CXX)
enable_testing()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# collect source files
file(GLOB_RECURSE web-ifc-source web-ifc/*.cpp)
file(GLOB_RECURSE web-ifc-wasm wasm/*.cpp)

# download the external projects and save their paths
Message("Downloading FastFloat")
FetchContent_Declare(fastfloat GIT_REPOSITORY "https://github.com/fastfloat/fast_float" GIT_TAG "2b2395f9ac836ffca6404424bcc252bff7aa80e4" SOURCE_SUBDIR "../")
FetchContent_MakeAvailable(fastfloat)

Message("Downloading TinyNURBS")
FetchContent_Declare(tinynurbs GIT_REPOSITORY "https://github.com/QuimMoya/tinynurbs" GIT_TAG "47115cd9b6e922b27bbc4ab01fdeac2e9ea597a4" SOURCE_SUBDIR "../")
FetchContent_MakeAvailable(tinynurbs)

Message("Downloading TinyCPPTest")
FetchContent_Declare(tinycpptest GIT_REPOSITORY "https://github.com/kovacsv/TinyCppTest" GIT_TAG "12e42c8ac6e032ce450fb3f772ebdfd1ddc6008c" SOURCE_SUBDIR "../")
FetchContent_MakeAvailable(tinycpptest)

Message("Downloading GLM")
FetchContent_Declare(glm GIT_REPOSITORY "https://github.com/g-truc/glm" GIT_TAG "8d1fd52e5ab5590e2c81768ace50c72bae28f2ed" SOURCE_SUBDIR "../")
FetchContent_MakeAvailable(glm)

Message("Downloading Earcut")
FetchContent_Declare(earcut GIT_REPOSITORY "https://github.com/mapbox/earcut.hpp" GIT_TAG "4811a2b69b91f6127a75e780de6e2113609ddabb" SOURCE_SUBDIR "../")
FetchContent_MakeAvailable(earcut)

Message("Downloading CDT")
FetchContent_Declare(cdt GIT_REPOSITORY "https://github.com/artem-ogre/CDT" GIT_TAG "4d0c9026b8ec846fe544897e7111f8f9080d5f8a" SOURCE_SUBDIR "../")
FetchContent_MakeAvailable(cdt)

Message("Downloading spdlog")
FetchContent_Declare(spdlog GIT_REPOSITORY "https://github.com/gabime/spdlog" GIT_TAG "f355b3d58f7067eee1706ff3c801c2361011f3d5" SOURCE_SUBDIR "../")
FetchContent_MakeAvailable(spdlog)

Message("Downloading StdUUID")
FetchContent_Declare(stduuid GIT_REPOSITORY "https://github.com/mariusbancila/stduuid" GIT_TAG "3afe7193facd5d674de709fccc44d5055e144d7a" SOURCE_SUBDIR "../")
FetchContent_MakeAvailable(stduuid)


Message("Downloads Finished")

function(PARAM_SETTER THE_EXECUTABLE)
	# Private libraries
	target_include_directories(${THE_EXECUTABLE} PRIVATE
		$<BUILD_INTERFACE:${tinynurbs_SOURCE_DIR}/include>
		$<BUILD_INTERFACE:${fastfloat_SOURCE_DIR}/include>
		$<BUILD_INTERFACE:${cdt_SOURCE_DIR}/CDT/include>
		$<BUILD_INTERFACE:${earcut_SOURCE_DIR}/include>
		$<BUILD_INTERFACE:${spdlog_SOURCE_DIR}/include>
		$<BUILD_INTERFACE:${stduuid_SOURCE_DIR}/include>
	)

	# Public libraries
	target_include_directories(${THE_EXECUTABLE} PUBLIC
		$<BUILD_INTERFACE:${glm_SOURCE_DIR}/>
		$<BUILD_INTERFACE:${glm_SOURCE_DIR}/glm>
	)

	if(NOT MSVC)
		target_compile_options(${THE_EXECUTABLE} PUBLIC "-Wall" "-Wextra" "-Wpedantic" "-pedantic")
		if(RELEASE)
			target_compile_options(${THE_EXECUTABLE} PUBLIC "-O3")
		endif()
	else()
		target_compile_options(${THE_EXECUTABLE} PUBLIC "/utf-8")
	endif()

	if(EMSCRIPTEN)
		target_compile_options(${THE_EXECUTABLE} PUBLIC "-fexperimental-library")
	endif()
endfunction()

if(EMSCRIPTEN)
	# build parameters for web-ifc
	add_executable(web-ifc ${web-ifc-source} ${web-ifc-wasm})
	param_setter(web-ifc)
	set_target_properties(web-ifc PROPERTIES LINK_FLAGS "${DEBUG_FLAG} --bind -flto -s ALLOW_MEMORY_GROWTH=1 -s DYNAMIC_EXECUTION=0 -s MAXIMUM_MEMORY=4GB -sSTACK_SIZE=5MB -s EXPORT_NAME=WebIFCWasm -s MODULARIZE=1 -s ENVIRONMENT=web  -sASSERTIONS=1 -s EXPORTED_RUNTIME_METHODS=\"['HEAPU8','HEAPU32','HEAPF32']\"")

	add_executable(web-ifc-node ${web-ifc-source} ${web-ifc-wasm})
	param_setter(web-ifc-node)
	set_target_properties(web-ifc-node PROPERTIES LINK_FLAGS "${DEBUG_FLAG} --bind -flto -s ALLOW_MEMORY_GROWTH=1 -s DYNAMIC_EXECUTION=0 -s MAXIMUM_MEMORY=4GB -sSTACK_SIZE=5MB -s EXPORT_NAME=WebIFCWasm -s MODULARIZE=1 -s EXPORTED_RUNTIME_METHODS=\"['HEAPU8','HEAPU32','HEAPF32']\"")

	# multi-treaded versions
	add_executable(web-ifc-mt ${web-ifc-source} ${web-ifc-wasm})
	param_setter(web-ifc-mt)
	target_compile_options(web-ifc-mt PUBLIC "-pthread")
	set_target_properties(web-ifc-mt PROPERTIES LINK_FLAGS "${DEBUG_FLAG} -pthread -s PTHREAD_POOL_SIZE=navigator.hardwareConcurrency --bind -flto --define-macro=REAL_T_IS_DOUBLE -sSTACK_SIZE=5MB -s ALLOW_MEMORY_GROWTH=1 -s DYNAMIC_EXECUTION=0 -s MAXIMUM_MEMORY=4GB -s EXPORT_NAME=WebIFCWasm -s MODULARIZE=1 -s ENVIRONMENT=web,worker")
endif()

if(NOT EMSCRIPTEN)
	include(GNUInstallDirs)
	include(CMakePackageConfigHelpers)

	# build web-ifc as a shared library
	add_library(web-ifc-library SHARED ${web-ifc-source})
	param_setter(web-ifc-library)

	# Export symbols on Windows
	set_target_properties(web-ifc-library PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)

	# Add generator expressions for install support
	target_include_directories(web-ifc-library PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/web-ifc>
		$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
		$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/web-ifc>
	)

	# build parameters for web-ifc-test
	add_executable(web-ifc-test ${web-ifc-source} "./test/encoding_test.cpp" "./test/main.cpp" "./test/io_helpers.cpp")
	param_setter(web-ifc-test)
	target_include_directories(web-ifc-test PUBLIC ${tinycpptest_SOURCE_DIR}/Sources)

	add_test(web-ifc-test web-ifc-test)
	set_tests_properties(web-ifc-test PROPERTIES LABELS "web-ifc")

	# build parameters for web-ifc in testing environment
	add_executable(web-ifc ${web-ifc-source} "./test/web-ifc-test.cpp" "./test/io_helpers.cpp")
	param_setter(web-ifc)
	target_include_directories(web-ifc PUBLIC ${tinycpptest_SOURCE_DIR}/Sources)

	# comment these to prevent debug files being generated
	if(NOT RELEASE)
		target_compile_options(web-ifc PUBLIC "-DCSG_DEBUG_OUTPUT")
		target_compile_options(web-ifc PUBLIC "-DDEBUG_DUMP_SVG")
	endif()

	# Installation rules
	# Install the shared library
	install(TARGETS web-ifc-library
		EXPORT web-ifc-targets
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		RUNTIME DESTINATION ${CMAKE_INSTALL_LIBDIR}
	)

	# Install web-ifc headers. The excluded files are internal-only headers
	# that pull in bundled deps we intentionally don't install (tinynurbs,
	# spdlog, mapbox/earcut, CDT -- see "Install public dep. API" below); they
	# aren't reachable from the public ModelManager.h/IfcLoader.h surface, so
	# installing them without their dependencies would leave dangling,
	# uncompilable headers in the install tree.
	install(DIRECTORY web-ifc/
		DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/web-ifc
		FILES_MATCHING
		PATTERN "*.h"
		PATTERN "mesh_utils.h" EXCLUDE
		PATTERN "geometryutils.h" EXCLUDE
		PATTERN "shared-position.h" EXCLUDE
		PATTERN "utils.h" EXCLUDE
	)

	# Install version.h
	install(FILES version.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/web-ifc)

	# Install public dep. API
	install(DIRECTORY ${glm_SOURCE_DIR}/glm DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

	# Export targets with namespace
	install(EXPORT web-ifc-targets
		FILE web-ifc-targets.cmake
		NAMESPACE web-ifc::
		DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/web-ifc
	)

	# Generate and install CMake config file
	configure_package_config_file(
		${CMAKE_CURRENT_SOURCE_DIR}/cmake/web-ifc-config.cmake.in
		${CMAKE_CURRENT_BINARY_DIR}/web-ifc-config.cmake
		INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/web-ifc
	)

	install(FILES ${CMAKE_CURRENT_BINARY_DIR}/web-ifc-config.cmake
		DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/web-ifc
	)
endif()
