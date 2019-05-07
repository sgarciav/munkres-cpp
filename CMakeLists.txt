# Common variables.
cmake_minimum_required  (VERSION 2.8)
project                 (munkres-cpp)
set (PROJECT_VERSION "0.1.0-dev")
set (munkres-cpp_VERSION_MAJOR 2)
set (munkres-cpp_VERSION_MINOR 0)

set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
set (CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_CXX_FLAGS} -O0 -ggdb3 -DDEBUG")
set (CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS} -O3")

include(GNUInstallDirs)

include_directories (${PROJECT_SOURCE_DIR}/src)
include_directories (${PROJECT_SOURCE_DIR}/src/adapters)

# Sources.
set (
    MunkresCppLib_SOURCES
    ${PROJECT_SOURCE_DIR}/src/munkres.cpp
)

# Headers.
set (
    MunkresCppLib_HEADERS
    ${PROJECT_SOURCE_DIR}/src/matrix.h
    ${PROJECT_SOURCE_DIR}/src/matrix.cpp
    ${PROJECT_SOURCE_DIR}/src/munkres.h
)

add_subdirectory(${PROJECT_SOURCE_DIR}/src/adapters)

###############################################################################
# Create the munkres library
###############################################################################
set(LIBRARY_NAME munkres)
add_library (
    ${LIBRARY_NAME} STATIC
    ${MunkresCppLib_SOURCES}
)

###############################################################################
# Installation targets
###############################################################################
install (TARGETS ${LIBRARY_NAME}
  # IMPORTANT: Add the library to the "export-set"
  EXPORT ${PROJECT_NAME}-targets
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/${DEB_HOST_MULTIARCH}
  PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ)

install (FILES ${MunkresCppLib_HEADERS}
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/munkres
  PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ)

# Binary example
set (MunkresCppBin_SOURCES ${PROJECT_SOURCE_DIR}/examples/main.cpp)
add_executable (munkres.bin EXCLUDE_FROM_ALL ${MunkresCppBin_SOURCES})
target_link_libraries (munkres.bin munkres)
add_custom_target (example)
add_dependencies  (example munkres.bin)


#############################################################################
# Generate the cmake configuration files for the install tree
#############################################################################
include(CMakePackageConfigHelpers)

set(LIB_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR} CACHE PATH "Library install dir")
# set(CMAKE_CONFIG_DEST "${LIB_INSTALL_DIR}/${PROJECT_NAME}/cmake")
set(CMAKE_CONFIG_DEST "${LIB_INSTALL_DIR}/${DEB_HOST_MULTIARCH}/${PROJECT_NAME}/cmake")

configure_package_config_file(
  ${PROJECT_SOURCE_DIR}/cmake/Modules/project-config.cmake.in
  ${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}-config.cmake
  INSTALL_DESTINATION ${CMAKE_CONFIG_DEST}
  )

write_basic_package_version_file(
  ${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}-config-version.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion
  )

# Install cmake config files
install(FILES ${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}-config.cmake
  ${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}-config-version.cmake
  DESTINATION ${CMAKE_CONFIG_DEST})

# Export project export set to the install tree
install(
  EXPORT ${PROJECT_NAME}-targets
  DESTINATION ${CMAKE_CONFIG_DEST}
  )

# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry ~/.cmake)
export(PACKAGE ${PROJECT_NAME})


#############################################################################
# Configure project in development mode.
#############################################################################
option(MUNKRESCPP_DEVEL_MODE "Configure project in development mode." OFF)
if (MUNKRESCPP_DEVEL_MODE)
    # Enable the ExternalProject_Add directive.
    # Which used for getting external tools for the Project.
    include (ExternalProject)


    # Testing
    add_subdirectory (tests)


    # Benchmarking
    add_subdirectory (benchmarks)


    # Custom target to build everything.
    add_custom_target  (full)
    add_dependencies (
        full
        munkres
        example
        tests
        benchmarks
    )

    # celero needs curses library
    #find_package(Curses)
    #include_directories(${CURSES_INCLUDE_DIR})

    # Static code analyse.
    set (CppCheck_REPORT ${PROJECT_BINARY_DIR}/cppcheck.report)
    add_custom_command (
        OUTPUT  ${CppCheck_REPORT}
        COMMAND cppcheck ${MunkresCppLib_SOURCES} ${MunkresCppBin_SOURCES} -I${PROJECT_SOURCE_DIR}/src -I${PROJECT_SOURCE_DIR} --enable=all --force --inconclusive > cppcheck.report 2>&1
    )
    add_custom_target  (cppcheck DEPENDS ${CppCheck_REPORT})
    set_directory_properties (PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES ${CppCheck_REPORT})
endif (MUNKRESCPP_DEVEL_MODE)
