cmake_minimum_required (VERSION 3.10)

project(cmock VERSION 2.4.6)

#-----------------------------------------------------------------------------
# Include cmake modules
#-----------------------------------------------------------------------------
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
include(CMakeDependentOption)

#-----------------------------------------------------------------------------
# Options
#-----------------------------------------------------------------------------
# When cross compiling CMake tools are only needed on the host
option(INSTALL_CMAKE_TOOLS "Install CMock CMake functions" ON)
# For packaging by tools like bitbake, shared and static libs should be build
# at once
CMAKE_DEPENDENT_OPTION(BUILD_STATIC_LIBS "Build static libs" OFF
	"BUILD_SHARED_LIBS" OFF)


#-----------------------------------------------------------------------------
# Compiler flags
#-----------------------------------------------------------------------------
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -pedantic -Wextra")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -pedantic -Wextra")

#-----------------------------------------------------------------------------
# Target unity
#-----------------------------------------------------------------------------
set(TARGET_UNITY_NAME unity)
set(TARGET_UNITY_NAMES ${TARGET_UNITY_NAME})

set(UNITY_FLOAT UNITY_INCLUDE_FLOAT)
set(UNITY_DOUBLE UNITY_INCLUDE_DOUBLE)
set(UNITY_EXCLUDE_FLOAT_PRINT UNITY_INCLUDE_FLOAT_PRINT)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/unity_config.h.in
    ${CMAKE_CURRENT_SOURCE_DIR}/vendor/unity/src/unity_config.h @ONLY
)

set(TARGET_UNITY_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/vendor/unity/src/unity.c)
add_library(${TARGET_UNITY_NAME} ${TARGET_UNITY_SOURCES})

if(BUILD_SHARED_LIBS AND BUILD_STATIC_LIBS)
    add_library(${TARGET_UNITY_NAME}_static STATIC ${TARGET_UNITY_SOURCES})
    set_target_properties(${TARGET_UNITY_NAME}_static PROPERTIES
        POSITION_INDEPENDENT_CODE FALSE)
    list(APPEND TARGET_UNITY_NAMES ${TARGET_UNITY_NAME}_static)
endif()

foreach(TARGET_NAME ${TARGET_UNITY_NAMES})
    if(NOT TARGET unity::${TARGET_NAME})
        add_library(unity::${TARGET_NAME} ALIAS ${TARGET_NAME})
    endif()

    target_include_directories(${TARGET_NAME} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/vendor/unity/src>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}>
    )

    set_target_properties(${TARGET_NAME} PROPERTIES C_STANDARD 99)
    set_target_properties(${TARGET_NAME} PROPERTIES VERSION ${PROJECT_VERSION})
    set_target_properties(${TARGET_NAME} PROPERTIES SOVERSION ${PROJECT_VERSION})

    target_compile_definitions(${TARGET_NAME} PUBLIC -DUNITY_INCLUDE_CONFIG_H)
endforeach()

list(APPEND HEADER_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/vendor/unity/src/unity.h
    ${CMAKE_CURRENT_SOURCE_DIR}/vendor/unity/src/unity_internals.h
    ${CMAKE_CURRENT_SOURCE_DIR}/vendor/unity/src/unity_config.h
)

#-----------------------------------------------------------------------------
# Target cmock
#-----------------------------------------------------------------------------
set(TARGET_CMOCK_NAMES ${PROJECT_NAME})
set(TARGET_CMOCK_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/src/cmock.c)

add_library(${PROJECT_NAME} ${TARGET_CMOCK_SOURCES})

target_link_libraries(${PROJECT_NAME} ${TARGET_UNITY_NAME})

if(BUILD_SHARED_LIBS AND BUILD_STATIC_LIBS)
     add_library(${PROJECT_NAME}_static STATIC ${TARGET_CMOCK_SOURCES})
     target_link_libraries(${PROJECT_NAME}_static ${TARGET_UNITY_NAME}_static)
     set_target_properties(${PROJECT_NAME}_static PROPERTIES
          POSITION_INDEPENDENT_CODE FALSE)
     list(APPEND TARGET_CMOCK_NAMES ${PROJECT_NAME}_static)
endif()

foreach(TARGET_NAME ${TARGET_CMOCK_NAMES})
    if(NOT TARGET unity::${TARGET_NAME})
        add_library(unity::${TARGET_NAME} ALIAS ${TARGET_NAME})
    endif()

    target_include_directories(${TARGET_NAME} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
	$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${TARGET_NAME}>
    )

    set_target_properties(${TARGET_NAME} PROPERTIES C_STANDARD 99)
    set_target_properties(${TARGET_NAME} PROPERTIES VERSION ${PROJECT_VERSION})
    set_target_properties(${TARGET_NAME} PROPERTIES SOVERSION ${PROJECT_VERSION})
endforeach()

list(APPEND HEADER_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/src/cmock_internals.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/cmock.h
)

#-----------------------------------------------------------------------------
# Install/export targets and files
#-----------------------------------------------------------------------------
set(CONFIG_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
set(PROJECT_VERSION_FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake")
set(PROJECT_CONFIG_FILE "${PROJECT_NAME}-config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(TARGETS_EXPORT_FILE "${TARGETS_EXPORT_NAME}.cmake")
set(CMOCK_TOOLS_FILE "${PROJECT_NAME}-tools.cmake")
set(CMOCK_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/${PROJECT_NAME}")

# generate package version file
write_basic_package_version_file(
    ${PROJECT_VERSION_FILE}
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_CONFIG_FILE}.in
    ${CMAKE_CURRENT_BINARY_DIR}/install-${PROJECT_CONFIG_FILE}
    INSTALL_DESTINATION ${CONFIG_INSTALL_DIR}
    PATH_VARS CONFIG_INSTALL_DIR CMOCK_INSTALL_DIR
    INSTALL_PREFIX ${CMAKE_BINARY_DIR}
)

# generate a package configuration just for build tree export
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/build-tree-${PROJECT_CONFIG_FILE}.in
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_CONFIG_FILE}
    @ONLY
)

if(INSTALL_CMAKE_TOOLS)
	# configure cmock tools module
	configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/${CMOCK_TOOLS_FILE}.in ${CMOCK_TOOLS_FILE}
	  @ONLY
	)
endif()

# install targets
install(TARGETS ${TARGET_UNITY_NAMES} ${TARGET_CMOCK_NAMES}
        EXPORT ${TARGETS_EXPORT_NAME}
        LIBRARY DESTINATION ${CMOCK_INSTALL_DIR}
        ARCHIVE DESTINATION ${CMOCK_INSTALL_DIR}
)

# install headers
install(FILES ${HEADER_FILES}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}
)

# install scripts directory
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/scripts
        DESTINATION  ${CMOCK_INSTALL_DIR}
)

# install lib directory
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/lib
        DESTINATION  ${CMOCK_INSTALL_DIR}
)

# install config directory
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/config
        DESTINATION  ${CMOCK_INSTALL_DIR}
)

# install vendor/unity/auto directory
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/vendor/unity/auto
        DESTINATION  ${CMOCK_INSTALL_DIR}/vendor/unity
)

# install version file
install(FILES ${PROJECT_VERSION_FILE}
        DESTINATION ${CONFIG_INSTALL_DIR}
)

# build directory package config
export(EXPORT ${TARGETS_EXPORT_NAME}
       FILE ${TARGETS_EXPORT_FILE}
       NAMESPACE "unity::"
)

# installed package config
install(EXPORT ${TARGETS_EXPORT_NAME}
        DESTINATION ${CONFIG_INSTALL_DIR}
        FILE ${TARGETS_EXPORT_FILE}
        NAMESPACE "unity::"
)

# install package config
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/install-${PROJECT_CONFIG_FILE}
        DESTINATION ${CONFIG_INSTALL_DIR}
        RENAME ${PROJECT_CONFIG_FILE}
)

if(INSTALL_CMAKE_TOOLS)
	# install tools module
	install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${CMOCK_TOOLS_FILE}
	        DESTINATION ${CONFIG_INSTALL_DIR}
	)
endif()

# add to CMake registry
export(PACKAGE ${PROJECT_NAME})
