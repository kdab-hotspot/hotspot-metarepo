cmake_minimum_required(VERSION 3.1)

project(hotspot-perfparser)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

get_filename_component(__cmake_path ${CMAKE_COMMAND} PATH)
find_program(CPACK_COMMAND cpack ${__cmake_path})

set(PACKAGES_DIR ${CMAKE_BINARY_DIR}/packages)

if (("x${CMAKE_BUILD_PARALLEL_LEVEL}" STREQUAL "x") AND UNIX)
include(ProcessorCount)
ProcessorCount(NCORES)
message(STATUS "Defaulting to ${NCORES} parallel jobs")
set(CMAKE_BUILD_PARALLEL_LEVEL ${NCORES})
endif()

if ("x${CMAKE_BUILD_TYPE}" STREQUAL "x")
set(CMAKE_BUILD_TYPE "Release")
endif()
string(TOLOWER ${CMAKE_BUILD_TYPE} BUILD_TYPE)

##################################################

macro(add_component COMPONENT TOOLCHAIN)

# Cannot use ARGN directly with list() command,
# so copy it to a variable first.
set(EXTRA_MACRO_ARGS ${ARGN})
set(EXTRA_PATH "")

# Did we get any optional args?
list(LENGTH EXTRA_MACRO_ARGS NUM_EXTRA_ARGS)
if (${NUM_EXTRA_ARGS} GREATER 0)
    list(GET EXTRA_MACRO_ARGS 0 EXTRA_PATH)
    list(REMOVE_ITEM EXTRA_MACRO_ARGS "${EXTRA_PATH}")
endif ()
if ("x${EXTRA_PATH}" STREQUAL "xNONE")
set(EXTRA_PATH "")
endif()

# Treat the rest of macro args as CMake extra args
set(CMAKE_EXTRA_ARGS ${EXTRA_MACRO_ARGS})

string(TOLOWER ${CMAKE_BUILD_TYPE} BUILD_TYPE)
string(TOLOWER ${CMAKE_SYSTEM_PROCESSOR} ARCH)
string(TOLOWER ${CMAKE_SYSTEM_NAME} OS)
set(COMPONENT_PACKAGE "${COMPONENT}-${ARCH}-${OS}-${BUILD_TYPE}.zip")
if ("x${EXTRA_PATH}" STREQUAL "x")
set(COMPONENT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/${COMPONENT})
else()
set(COMPONENT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/${COMPONENT}/${EXTRA_PATH})
endif()

add_custom_command(
	OUTPUT ${COMPONENT_PATH}/build
	COMMAND ${CMAKE_COMMAND} -E make_directory ${COMPONENT_PATH}/build)

if ("${TOOLCHAIN}" STREQUAL "OPENSOURCE")
add_custom_command(
	OUTPUT ${PACKAGES_DIR}/${COMPONENT_PACKAGE}
	DEPENDS ${COMPONENT_PATH}/build
	COMMAND ${CMAKE_COMMAND} -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCMAKE_INSTALL_PREFIX=${PACKAGES_DIR} ${CMAKE_EXTRA_ARGS} ${COMPONENT_PATH}
	COMMAND ${CMAKE_COMMAND} --build ${COMPONENT_PATH}/build -- -j${CMAKE_BUILD_PARALLEL_LEVEL}
	COMMAND ${CPACK_COMMAND} -C ${CMAKE_BUILD_TYPE}
	COMMENT "Building ${COMPONENT} bundle"
	WORKING_DIRECTORY ${COMPONENT_PATH}/build)
elseif ("${TOOLCHAIN}" STREQUAL "MSVC")
add_custom_command(
	OUTPUT ${PACKAGES_DIR}/${COMPONENT_PACKAGE}
	DEPENDS ${COMPONENT_PATH}/build
	COMMAND ${CMAKE_COMMAND} -G "Visual Studio 16 2019" -A Win32 -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCMAKE_INSTALL_PREFIX=${PACKAGES_DIR} ${CMAKE_EXTRA_ARGS} ${COMPONENT_PATH}
	COMMAND ${CMAKE_COMMAND} --build ${COMPONENT_PATH}/build --config ${CMAKE_BUILD_TYPE}
	COMMAND ${CPACK_COMMAND} -C ${CMAKE_BUILD_TYPE}
	COMMENT "Building ${COMPONENT} bundle"
	WORKING_DIRECTORY ${COMPONENT_PATH}/build)
endif()

add_custom_target(${COMPONENT}_component ALL
	DEPENDS ${PACKAGES_DIR}/${COMPONENT_PACKAGE})

endmacro(add_component)

##################################################

if (UNIX)
add_component(zlib OPENSOURCE)
add_component(pcre2 OPENSOURCE NONE -DPCRE2_BUILD_PCRE2_16:BOOL=ON)
add_component(freetype2 OPENSOURCE)
add_component(libpng OPENSOURCE)
if (NOT APPLE)
set(QT_REQUIRE_GUI_FEATURE -DQT_FEATURE_xcb:BOOL=ON)
endif()
add_component(qtbase OPENSOURCE NONE -DCMAKE_PREFIX_PATH:STRING=${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build ${QT_REQUIRE_GUI_FEATURE} -DQT_FEATURE_network:BOOL=ON -DBUILD_EXAMPLES:BOOL=OFF -DUSE_ORIGIN_RPATH:BOOL=ON)
add_component(hotspot-perfparser OPENSOURCE NONE -DCMAKE_PREFIX_PATH:STRING=${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/hotspot-perfparser/build)
else()
add_component(zlib MSVC)
add_component(pcre2 MSVC NONE -DPCRE2_BUILD_PCRE2_16:BOOL=ON)
add_component(freetype2 MSVC)
add_component(libpng MSVC NONE -DCMAKE_PREFIX_PATH:STRING=${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/libpng/build)
add_component(qtbase MSVC NONE -DCMAKE_PREFIX_PATH:STRING=${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build -DQT_FEATURE_network:BOOL=ON -DBUILD_EXAMPLES:BOOL=OFF -DUSE_ORIGIN_RPATH:BOOL=ON)
add_component(hotspot-perfparser MSVC NONE -DCMAKE_PREFIX_PATH:STRING=${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/hotspot-perfparser/build)
endif()

add_custom_command(
	OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/libpng/build
	COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/libpng/build)

add_custom_command(
	OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build
	COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build)

add_custom_command(
        OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/hotspot-perfparser/build
        COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/hotspot-perfparser/build)

# Load ZLIB.

set(HOTSPOT_ZLIB_ARCHIVE "${PACKAGES_DIR}/zlib-${ARCH}-${OS}-${BUILD_TYPE}.zip")

add_custom_target(zlib_bundle1
	COMMAND ${CMAKE_COMMAND} -E tar xf ${HOTSPOT_ZLIB_ARCHIVE}
	DEPENDS zlib_component ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/libpng/build
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/libpng/build
	COMMENT "Decompressing precompiled bundle ${HOTSPOT_ZLIB_ARCHIVE}")

add_custom_target(zlib_bundle2
	COMMAND ${CMAKE_COMMAND} -E tar xf ${HOTSPOT_ZLIB_ARCHIVE}
	DEPENDS zlib_component ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build
	COMMENT "Decompressing precompiled bundle ${HOTSPOT_ZLIB_ARCHIVE}")

add_dependencies(libpng_component zlib_bundle1)
add_dependencies(qtbase_component zlib_bundle2)

# Load PCRE2.

set(HOTSPOT_PCRE2_ARCHIVE "${PACKAGES_DIR}/pcre2-${ARCH}-${OS}-${BUILD_TYPE}.zip")

add_custom_target(pcre2_bundle
	COMMAND ${CMAKE_COMMAND} -E tar xf ${HOTSPOT_PCRE2_ARCHIVE}
	DEPENDS pcre2_component ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build
	COMMENT "Decompressing precompiled bundle ${HOTSPOT_PCRE2_ARCHIVE}")

add_dependencies(qtbase_component pcre2_bundle)

# Load FreeType.

set(HOTSPOT_FREETYPE2_ARCHIVE "${PACKAGES_DIR}/freetype2-${ARCH}-${OS}-${BUILD_TYPE}.zip")

add_custom_target(freetype2_bundle
	COMMAND ${CMAKE_COMMAND} -E tar xf ${HOTSPOT_FREETYPE2_ARCHIVE}
	DEPENDS freetype2_component ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build
	COMMENT "Decompressing precompiled bundle ${HOTSPOT_FREETYPE2_ARCHIVE}")

add_dependencies(qtbase_component freetype2_bundle)

# Load FreeType.

set(HOTSPOT_LIBPNG_ARCHIVE "${PACKAGES_DIR}/libpng-${ARCH}-${OS}-${BUILD_TYPE}.zip")

add_custom_target(libpng_bundle
	COMMAND ${CMAKE_COMMAND} -E tar xf ${HOTSPOT_LIBPNG_ARCHIVE}
	DEPENDS libpng_component ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/qtbase/build
	COMMENT "Decompressing precompiled bundle ${HOTSPOT_LIBPNG_ARCHIVE}")

add_dependencies(qtbase_component libpng_bundle)

# Load Qt6 bundle.

set(HOTSPOT_QTBASE_ARCHIVE "${PACKAGES_DIR}/QtBase-${ARCH}-${OS}-${BUILD_TYPE}.zip")

add_custom_target(qtbase_bundle
	COMMAND ${CMAKE_COMMAND} -E tar xf ${HOTSPOT_QTBASE_ARCHIVE}
	DEPENDS qtbase_component ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/hotspot-perfparser/build
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/hotspot-perfparser/build
	COMMENT "Decompressing precompiled bundle ${HOTSPOT_QTBASE_ARCHIVE}")

set(HOTSPOT_ARCHIVE "${PACKAGES_DIR}/hotspot-perfparser-${ARCH}-${OS}-${BUILD_TYPE}.zip")

# Make Hotspot bundle to depend on Qt6 runtime bundle.

add_dependencies(hotspot-perfparser_component qtbase_bundle)

##################################################

add_custom_command(
        OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/install
        COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/install)

add_custom_target(qtbase_install ALL
	COMMAND ${CMAKE_COMMAND} -E tar xf ${HOTSPOT_QTBASE_ARCHIVE}
	DEPENDS qtbase_component ${CMAKE_CURRENT_SOURCE_DIR}/install
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/install
	COMMENT "Installing ${HOTSPOT_QTBASE_ARCHIVE}")

add_custom_target(hotspot-perfparser_install ALL
	COMMAND ${CMAKE_COMMAND} -E tar xf ${HOTSPOT_ARCHIVE}
	DEPENDS hotspot-perfparser_component ${CMAKE_CURRENT_SOURCE_DIR}/install
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/install
	COMMENT "Installing ${HOTSPOT_ARCHIVE}")

