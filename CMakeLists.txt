cmake_minimum_required(VERSION 2.6)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/share)

project(abcToA)

set(ABCTOA_MAJOR_VERSION 3)
set(ABCTOA_MINOR_VERSION 0)
set(ABCTOA_PATCH_VERSION 2)
set(ABCTOA_VERSION "${ABCTOA_MAJOR_VERSION}.${ABCTOA_MINOR_VERSION}.${ABCTOA_PATCH_VERSION}")

IF(NOT MTOA_VERSION AND NOT $ENV{MTOA_VERSION} STREQUAL "")
  SET(MTOA_VERSION $ENV{MTOA_VERSION})
ENDIF()


set(CMAKE_BUILD_TYPE RELEASE)
set(CMAKE_VERBOSE_MAKEFILE FALSE)
set(CMAKE_SKIP_RPATH TRUE)

# Fix to stop crash on OSX < 10.9
set(CMAKE_OSX_DEPLOYMENT_TARGET 10.8)

#variable helpers
# Cmake system specific flags
SET(WINDOWS FALSE)
IF ("${CMAKE_SYSTEM_NAME}" MATCHES "Windows")
    SET(WINDOWS TRUE)
ENDIF()

SET(DARWIN FALSE)
IF ("${CMAKE_SYSTEM_NAME}" MATCHES "Darwin")
    SET(DARWIN TRUE)
    # suppress rpath warning
    IF (POLICY CMP0042)
        CMAKE_POLICY(SET CMP0042 OLD)
    ENDIF()
ENDIF()



SET(LINUX FALSE)
IF ("${CMAKE_SYSTEM_NAME}" MATCHES "Linux")
    SET(LINUX TRUE)
ENDIF()

OPTION(USE_HDF5 "Include HDF5 stuff" OFF)
OPTION(USE_ILMBASE_STATIC_LIBS "Build with static ilmBase libs" ON)
OPTION(USE_STATIC_HDF5 "Build with static HDF5 libs" OFF)
OPTION(ALEMBIC_SHARED_LIBS "Use shared libraries" OFF)
OPTION(ALEMBIC_ILMBASE_LINK_STATIC "Use static IlmBase libraries" ON)
OPTION(BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(USE_NSIS "Use NSIS generator to produce installer" OFF)

# Custom target for packaging.
if(USE_NSIS)
  set(CPACK_GENERATOR "NSIS")
else(USE_NSIS)
  set(CPACK_GENERATOR "ZIP")
endif(USE_NSIS)



set(CPACK_PACKAGE_VERSION "${ABCTOA_MAJOR_VERSION}")
set(CPACK_PACKAGE_VERSION_PATCH "${ABCTOA_PATCH_VERSION}")
set(CPACK_PACKAGE_VENDOR "Nozon")
set(CPACK_PACKAGE_INSTALL_REGISTRY_KEY "abcToA")

set(CPACK_SOURCE_GENERATOR "TGZ")


IF (LINUX OR DARWIN)
	INCLUDE(CheckCXXCompilerFlag)
	CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
	CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
	IF (COMPILER_SUPPORTS_CXX11)
		SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
	ELSEIF (COMPILER_SUPPORTS_CXX0X)
		SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
	ENDIF()
ENDIF()

# Compiler flags
if (WINDOWS)
    # Disable some of the bullshit warnings MSVC wants to barf
    add_definitions( "-W3 -MP -D_CRT_SECURE_NO_WARNINGS -wd4005 -wd4996 -wd4305 -wd4244 -nologo" )
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
ELSEIF (DARWIN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
ENDIF()


IF (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
ENDIF()

# Set some debug vs opt flags
IF ("${CMAKE_BUILD_TYPE}" MATCHES "Debug")
    ADD_DEFINITIONS(-DDEBUG=1 -UNDEBUG)
    IF (NOT WINDOWS)
        ADD_DEFINITIONS(-Wall -Werror -Wextra -Wno-unused-parameter)
    ENDIF()
ELSEIF ("${CMAKE_BUILD_TYPE}" MATCHES "Release")
    ADD_DEFINITIONS(-DNDEBUG=1 -UDEBUG)
    IF (NOT WINDOWS)
        ADD_DEFINITIONS(-O3)
    ENDIF()
ENDIF()

IF (NOT WINDOWS)
    SET(EXTERNAL_MATH_LIBS "-lm")
ELSE()
    SET(EXTERNAL_MATH_LIBS "")
ENDIF()

IF (WINDOWS)
    ADD_DEFINITIONS(-DPLATFORM_WINDOWS -DPLATFORM=WINDOWS)
ELSEIF (DARWIN)
    ADD_DEFINITIONS(-DPLATFORM_DARWIN -DPLATFORM=DARWIN)
ELSE()
    ADD_DEFINITIONS(-DPLATFORM_LINUX -DPLATFORM=LINUX -D_LINUX)
ENDIF()

# check if we have a local cmake include file and include it if we do
# this is useful for setting our arnold location as an alternative to
# environment variables
if(EXISTS ${CMAKE_SOURCE_DIR}/local.cmake)
    message(STATUS "Using local.cmake")
    include(${CMAKE_SOURCE_DIR}/local.cmake)
endif()

# Find Arnold SDK
find_package(Arnold REQUIRED)

find_package(Mtoa REQUIRED)

# Find maya
find_package(Maya REQUIRED)

# Find HDF5
if(USE_HDF5)
	find_package(AlembicHDF5 REQUIRED)
	find_package(AlembicZlib REQUIRED)
	find_package(SZIP)
endif()


configure_file(
    "${PROJECT_SOURCE_DIR}/exportArnoldDefinitions.in.py"
    "${PROJECT_SOURCE_DIR}/exportArnoldDefinitions.py"
)

if (NOT DEFINED INSTALL_DIR)
	set(INSTALL_DIR "abcToA-${ABCTOA_VERSION}")
else()
    message("Installing to:\n\t${INSTALL_DIR}")
endif()

# Set up installation paths
set(DSO_INSTALL_DIR ${INSTALL_DIR}/procedurals)
set(TRANSLATOR_INSTALL_DIR ${INSTALL_DIR}/extensions)
set(SHADERS_INSTALL_DIR ${INSTALL_DIR}/shaders)
set(PLUGINS_INSTALL_DIR ${INSTALL_DIR}/plug-ins)
set(PYSIDE_INSTALL_DIR ${INSTALL_DIR}/pyside)

message("DSO binaries will be installed to:\n\t${DSO_INSTALL_DIR}")
message("Shaders will be installed to: \n\t${SHADERS_INSTALL_DIR}")
message("Maya Plug-ins will be installed to: \n\t${PLUGINS_INSTALL_DIR}")
message("MtoA Translators will be installed to: \n\t${TRANSLATOR_INSTALL_DIR}")
message("Icons will be installed to: \n\t${INSTALL_DIR}/icons")
message("Scripts will be installed to: \n\t${INSTALL_DIR}/scripts")
message("PySide will be installed to: \n\t${PYSIDE_INSTALL_DIR}")
message("Samples will be installed to: \n\t${INSTALL_DIR}/samples")

add_subdirectory(alembic EXCLUDE_FROM_ALL) 
add_subdirectory(ThirdParty EXCLUDE_FROM_ALL)

# Set the list of subdirectories to recurse into to find stuff to build
set(SUBDIRECTORIES 
		arnold/procedurals/alembicProcedural
		#arnold/shaders/abcShader
		arnold/utility/assShadersToAbc
		maya/alembicHolder
		mtoa/ABCViewer
		#mtoa/abcShader
)

# loop over subdirectories
foreach(SUBDIR ${SUBDIRECTORIES})
    add_subdirectory(${SUBDIR})
endforeach()


# abcMayaShader won't compile on darwin for the moment.
IF(NOT DARWIN)
	add_subdirectory(maya/abcMayaShader)
ENDIF()

# add top-level files
INSTALL(FILES README INSTALL DESTINATION ${INSTALL_DIR})

# add scripts & icons
INSTALL(DIRECTORY icons DESTINATION ${INSTALL_DIR})
INSTALL(DIRECTORY maya/scripts DESTINATION ${INSTALL_DIR})
INSTALL(DIRECTORY maya/samples DESTINATION ${INSTALL_DIR})


# configure other files with version numbers etc.

configure_file (
  "${CMAKE_SOURCE_DIR}/maya/alembicHolder/version.h.in"
  "${CMAKE_BINARY_DIR}/maya/alembicHolder/version.h"
  )

configure_file (
"${CMAKE_SOURCE_DIR}/abcToA.mod.in"
"${CMAKE_BINARY_DIR}/abcToA.mod"
)

configure_file(
    "${CMAKE_SOURCE_DIR}/INSTALL.in"
    "${CMAKE_BINARY_DIR}/INSTALL"
)
  
# 


