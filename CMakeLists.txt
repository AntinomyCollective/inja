cmake_minimum_required(VERSION 3.1)


##
## PROJECT
##
project(inja LANGUAGES CXX VERSION 1.0.1)
set(INJA_VERSION ${PROJECT_VERSION})


##
## OPTIONS
##
option(BUILD_UNIT_TESTS "Build the unit tests" ON)
option(BUILD_BENCHMARK "Build the inja benchmark" OFF)
option(HUNTER_ENABLED "Use hunter to manage dependencies" OFF)


##
## HUNTER
##
if(HUNTER_ENABLED)
  include("cmake/HunterGate.cmake")
  HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.19.156.tar.gz"
    SHA1 "8d5e4635b137365e0d1ade4d60accf4e2bb41f0d"
  )
endif()


##
## CONFIGURATION
##
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
set(INJA_INCLUDE_DIR include)
set(INJA_SINGLE_INCLUDE_DIR single_include)
set(INJA_HEADER_INSTALL_DIR include)

if(WIN32 AND MSVC AND MSVC_VERSION LESS 1900)
  message(FATAL_ERROR "[${PROJECT_NAME}] Visual Studio versions prior to 2015 do not support the noexcept keyword, which is used in the JSON library.")
endif()


##
## TESTS
## create and configure the unit test target
##
if(BUILD_UNIT_TESTS)
  enable_testing()
  add_subdirectory(test)
endif()


##
## AMALGAMATE
## amalgamate header files into single_include
##
execute_process(COMMAND python3 amalgamate/amalgamate.py -c amalgamate/config.json -s include
  WORKING_DIRECTORY ${PROJECT_SOURCE_DIR})


##
## TARGETS
## Build targets for the interface library
##
add_library(inja INTERFACE)
target_include_directories(inja INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${INJA_INCLUDE_DIR}>
  $<INSTALL_INTERFACE:${INJA_HEADER_INSTALL_DIR}>
)


add_library(inja_single INTERFACE)
target_include_directories(inja_single INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${INJA_SINGLE_INCLUDE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${INJA_INCLUDE_DIR}>
  $<INSTALL_INTERFACE:${INJA_HEADER_INSTALL_DIR}>
)


if(HUNTER_ENABLED) # Use Hunter to manage dependencies
  # Add JSON package
  hunter_add_package(nlohmann_json)
  find_package(nlohmann_json CONFIG REQUIRED)
  # Add dependencies to target
  target_link_libraries(inja INTERFACE nlohmann_json)
endif()


##
## INSTALL
## install header files, generate and install cmake config files for find_package()
##
set(include_install_dir ${INJA_HEADER_INSTALL_DIR})
set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")
include(CMakePackageConfigHelpers)


write_basic_package_version_file(
  "${version_config}" COMPATIBILITY SameMajorVersion
)
configure_package_config_file(
  "cmake/Config.cmake.in"
  "${project_config}"
  INSTALL_DESTINATION "${config_install_dir}"
)

install(
  TARGETS inja
  EXPORT "${TARGETS_EXPORT_NAME}"
  LIBRARY DESTINATION "lib"
  ARCHIVE DESTINATION "lib"
  RUNTIME DESTINATION "bin"
  INCLUDES DESTINATION "${include_install_dir}"
)

install(
  FILES ${INJA_INCLUDE_DIR}/inja.hpp
  DESTINATION "${include_install_dir}"
)

install(
  FILES "${project_config}" "${version_config}"
  DESTINATION "${config_install_dir}"
)

install(
  EXPORT "${TARGETS_EXPORT_NAME}"
  NAMESPACE "${namespace}"
  DESTINATION "${config_install_dir}"
)
