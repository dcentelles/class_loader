cmake_minimum_required(VERSION 2.8.3)
set(CMAKE_VERBOSE_MAKEFILE ON)

#SET PROJECT NAME
#NOTE: replace "foo" by your project name
SET(PROJECT_NAME_ORIG "class_loader" )
message("Building ${PROJECT_NAME_ORIG}...")

#BEGIN RESOLVE GIT REVISION
#https://bravenewmethod.com/2017/07/14/git-revision-as-compiler-definition-in-build-with-cmake/
message(STATUS "Resolving GIT Version...")
set(_build_version "unknown")
find_package(Git)
if(GIT_FOUND)
  execute_process(
    COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
    OUTPUT_VARIABLE _build_version
    ERROR_QUIET
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  message( STATUS "GIT hash: ${_build_version}")
else()
  message(STATUS "GIT not found")
endif()
string(TIMESTAMP _time_stamp)
#END RESOLVE GIT REVISION

#DECLARE PROJECT
SET(PROJECT_NAME_GIT "${PROJECT_NAME_ORIG}_${_build_version}")
string(TOUPPER ${PROJECT_NAME_ORIG} TMP0)
SET(${TMP0}_GIT "${PROJECT_NAME_GIT}" PARENT_SCOPE)

if (TARGET ${PROJECT_NAME_GIT})
    message(WARNING "${PROJECT_NAME_GIT} has already been built!")
    return()
endif()

project (${PROJECT_NAME_GIT} CXX)
message(STATUS "PROJECT NAME: ${PROJECT_NAME}")

find_package(Boost REQUIRED COMPONENTS thread system)

set(CATKIN_DISABLED true CACHE BOOL "Disable the catkin build, useful if catkin is present but a build outside of ros is done")
if(NOT CATKIN_DISABLED)
  find_package(catkin QUIET)
endif()

find_package(console_bridge REQUIRED)

if(${catkin_FOUND})
  find_package(catkin REQUIRED COMPONENTS cmake_modules)
  find_package(Poco REQUIRED COMPONENTS Foundation)
  catkin_package(
    INCLUDE_DIRS include
    LIBRARIES ${PROJECT_NAME} ${Poco_LIBRARIES}
    DEPENDS Boost Poco console_bridge
    CFG_EXTRAS class_loader-extras.cmake
  )
else()
  SET(BUILDTYPE ${CMAKE_BUILD_TYPE})
  SET(CMAKE_BUILD_TYPE "Release")
  set(Poco_DIR cmake)
  find_package(Poco REQUIRED COMPONENTS Foundation)
  SET(CMAKE_BUILD_TYPE ${BUILDTYPE})
  set(CATKIN_GLOBAL_BIN_DESTINATION bin)
  set(CATKIN_GLOBAL_LIB_DESTINATION lib)
  set(CATKIN_GLOBAL_LIBEXEC_DESTINATION lib)
  set(CATKIN_GLOBAL_INCLUDE_DESTINATION include)
  set(CATKIN_PACKAGE_LIB_DESTINATION ${CATKIN_GLOBAL_LIB_DESTINATION})
  set(CATKIN_PACKAGE_BIN_DESTINATION ${CATKIN_GLOBAL_LIBEXEC_DESTINATION}/${PROJECT_NAME})
  set(CATKIN_PACKAGE_INCLUDE_DESTINATION ${CATKIN_GLOBAL_INCLUDE_DESTINATION}/${PROJECT_NAME})
endif()

include_directories(include ${console_bridge_INCLUDE_DIRS} ${Boost_INCLUDE_DIRS} ${Poco_INCLUDE_DIRS})

set(${PROJECT_NAME}_SRCS
  src/class_loader.cpp
  src/class_loader_core.cpp
  src/meta_object.cpp
  src/multi_library_class_loader.cpp
)
set(${PROJECT_NAME}_HDRS
  include/class_loader/class_loader.h
  include/class_loader/class_loader_core.h
  include/class_loader/class_loader_exceptions.h
  include/class_loader/class_loader_register_macro.h
  include/class_loader/console_bridge_compatibility.h
  include/class_loader/meta_object.h
  include/class_loader/multi_library_class_loader.h
)
add_definitions(-fPIC)
add_library(${PROJECT_NAME} ${${PROJECT_NAME}_SRCS} ${${PROJECT_NAME}_HDRS})
target_link_libraries(${PROJECT_NAME} ${Boost_LIBRARIES} ${console_bridge_LIBRARIES} ${Poco_LIBRARIES})

install(TARGETS ${PROJECT_NAME}
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_GLOBAL_BIN_DESTINATION})
install(DIRECTORY include/class_loader/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION})

install(PROGRAMS scripts/class_loader_headers_update.py
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION})

if(CATKIN_ENABLE_TESTING)
  add_subdirectory(test)
endif()

if(NOT catkin_FOUND)
  set(TARGET_NAME ${PROJECT_NAME})
  set(PKGCONFIG_LIBS
    ${Boost_LIBRARIES}
    ${console_bridge_LIBRARIES}
    ${Poco_LIBRARIES}
  )
  message(STATUS "POCO LIBS: ${Poco_LIBRARIES}")
  SET(${PROJECT_NAME}_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/include/ PARENT_SCOPE)
  SET(${PROJECT_NAME}_LIBRARIES ${PROJECT_NAME} PARENT_SCOPE)
  # Prepare and install necessary files to support finding of the library
  # using pkg-config
  #configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/${TARGET_NAME}.pc.in
  #  ${CMAKE_CURRENT_BINARY_DIR}/${TARGET_NAME}.pc @ONLY)
  #
  #install(FILES ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.pc DESTINATION lib/pkgconfig)
endif()
