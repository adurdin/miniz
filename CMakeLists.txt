PROJECT(miniz C)
cmake_minimum_required(VERSION 3.4)

set(Upstream_VERSION 2.1.0)

if(CMAKE_BUILD_TYPE STREQUAL "")
  # CMake defaults to leaving CMAKE_BUILD_TYPE empty. This screws up
  # differentiation between debug and release builds.
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING
    "Choose the type of build, options are: None (CMAKE_CXX_FLAGS or \
CMAKE_C_FLAGS used) Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif ()

option(BUILD_EXAMPLES "Build examples" ON)
option(AMALGAMATE_SOURCES "Amalgamate sources into miniz.h/c" OFF)
option(BUILD_HEADER_ONLY "Build a header-only version" OFF)
option(BUILD_SHARED_LIBS "Build shared library instead of static" ON)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin)

if(BUILD_HEADER_ONLY)
  set(AMALGAMATE_SOURCES ON CACHE BOOL "Build a header-only version" FORCE)
endif(BUILD_HEADER_ONLY)

if(AMALGAMATE_SOURCES)
  # Amalgamate
  file(COPY miniz.h DESTINATION ${CMAKE_BINARY_DIR}/amalgamation/)
  file(READ miniz.h MINIZ_H)
  file(READ miniz_common.h MINIZ_COMMON_H)
  file(READ miniz_tdef.h MINIZ_TDEF_H)
  file(READ miniz_tinfl.h MINIZ_TINFL_H)
  file(READ miniz_zip.h MINIZ_ZIP_H)
  file(APPEND ${CMAKE_BINARY_DIR}/amalgamation/miniz.h
     "${MINIZ_COMMON_H} ${MINIZ_TDEF_H} ${MINIZ_TINFL_H} ${MINIZ_ZIP_H}")

  file(COPY miniz.c DESTINATION ${CMAKE_BINARY_DIR}/amalgamation/)
  file(READ miniz_tdef.c MINIZ_TDEF_C)
  file(READ miniz_tinfl.c MINIZ_TINFL_C)
  file(READ miniz_zip.c MINIZ_ZIP_C)
  file(APPEND ${CMAKE_BINARY_DIR}/amalgamation/miniz.c
     "${MINIZ_TDEF_C} ${MINIZ_TINFL_C} ${MINIZ_ZIP_C}")

  file(READ ${CMAKE_BINARY_DIR}/amalgamation/miniz.h AMAL_MINIZ_H)
  file(READ ${CMAKE_BINARY_DIR}/amalgamation/miniz.c AMAL_MINIZ_C)
  foreach(REPLACE_STRING miniz;miniz_common;miniz_tdef;miniz_tinfl;miniz_zip)
    string(REPLACE "#include \"${REPLACE_STRING}.h\"" "" AMAL_MINIZ_H "${AMAL_MINIZ_H}")
    string(REPLACE "#include \"${REPLACE_STRING}.h\"" "" AMAL_MINIZ_C "${AMAL_MINIZ_C}")
  endforeach()
  if(BUILD_HEADER_ONLY)
    string(CONCAT AMAL_MINIZ_H "${AMAL_MINIZ_H}" "\n#ifndef MINIZ_HEADER_FILE_ONLY\n"
             "${AMAL_MINIZ_C}" "\n#endif // MINIZ_HEADER_FILE_ONLY\n")
    file(WRITE ${CMAKE_BINARY_DIR}/amalgamation/miniz.h "${AMAL_MINIZ_H}")
    add_library(${PROJECT_NAME} INTERFACE)
    set_property(TARGET ${PROJECT_NAME} APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES
      $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/amalgamation>
      $<INSTALL_INTERFACE:include>
    )
  else(BUILD_HEADER_ONLY)
    string(CONCAT AMAL_MINIZ_C "#include \"miniz.h\"\n" "${AMAL_MINIZ_C}")
    file(WRITE ${CMAKE_BINARY_DIR}/amalgamation/miniz.h "${AMAL_MINIZ_H}")
    file(WRITE ${CMAKE_BINARY_DIR}/amalgamation/miniz.c "${AMAL_MINIZ_C}")
    set(miniz_SOURCE ${CMAKE_BINARY_DIR}/amalgamation/miniz.h
                     ${CMAKE_BINARY_DIR}/amalgamation/miniz.c)
    add_library(${PROJECT_NAME} ${miniz_SOURCE})
    target_include_directories(${PROJECT_NAME} PUBLIC
      $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/amalgamation>
      $<INSTALL_INTERFACE:include>
    )
  endif(BUILD_HEADER_ONLY)
  set(INSTALL_HEADERS ${CMAKE_BINARY_DIR}/amalgamation/miniz.h)
else(AMALGAMATE_SOURCES)
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
  set(miniz_SOURCE miniz.c miniz_zip.c miniz_tinfl.c miniz_tdef.c)
  add_library(${PROJECT_NAME} ${miniz_SOURCE})

  set_property(TARGET ${PROJECT_NAME} PROPERTY VERSION ${Upstream_VERSION})
  set_property(TARGET ${PROJECT_NAME} PROPERTY SOVERSION 2)
  set_property(TARGET ${PROJECT_NAME} PROPERTY
    INTERFACE_${PROJECT_NAME}_MAJOR_VERSION 2)
  set_property(TARGET ${PROJECT_NAME} APPEND PROPERTY
    COMPATIBLE_INTERFACE_STRING ${PROJECT_NAME}_MAJOR_VERSION
  )

  # target_include_directories(${PROJECT_NAME} PUBLIC
  #   $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  #   $<INSTALL_INTERFACE:include>
  # )
  file(GLOB INSTALL_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/*.h)

  install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}Targets
    RUNTIME  DESTINATION bin
    ARCHIVE  DESTINATION lib
    LIBRARY  DESTINATION lib
    INCLUDES DESTINATION include
    )

  include(CMakePackageConfigHelpers)
  write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${Upstream_VERSION}
    COMPATIBILITY AnyNewerVersion
  )

  export(EXPORT ${PROJECT_NAME}Targets
    FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}Targets.cmake"
    NAMESPACE ${PROJECT_NAME}::
  )
  configure_file(Config.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}Config.cmake"
    @ONLY
  )

  set(ConfigPackageLocation lib/cmake/${PROJECT_NAME})
  install(EXPORT ${PROJECT_NAME}Targets
    FILE
      ${PROJECT_NAME}Targets.cmake
    NAMESPACE
      ${PROJECT_NAME}::
    DESTINATION
      ${ConfigPackageLocation}
  )
  install(
    FILES
      "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}Config.cmake"
      "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}ConfigVersion.cmake"
    DESTINATION
      ${ConfigPackageLocation}
    COMPONENT
      Devel
  )
endif(AMALGAMATE_SOURCES)

if(BUILD_EXAMPLES)
  set(EXAMPLE1_SRC_LIST "${CMAKE_CURRENT_SOURCE_DIR}/examples/example1.c")
  set(EXAMPLE2_SRC_LIST "${CMAKE_CURRENT_SOURCE_DIR}/examples/example2.c")
  set(EXAMPLE3_SRC_LIST "${CMAKE_CURRENT_SOURCE_DIR}/examples/example3.c")
  set(EXAMPLE4_SRC_LIST "${CMAKE_CURRENT_SOURCE_DIR}/examples/example4.c")
  set(EXAMPLE5_SRC_LIST "${CMAKE_CURRENT_SOURCE_DIR}/examples/example5.c")
  set(EXAMPLE6_SRC_LIST "${CMAKE_CURRENT_SOURCE_DIR}/examples/example6.c")
  set(MINIZ_TESTER_SRC_LIST
      "${CMAKE_CURRENT_SOURCE_DIR}/tests/miniz_tester.cpp"
      "${CMAKE_CURRENT_SOURCE_DIR}/tests/timer.cpp")

  add_executable(example1 ${EXAMPLE1_SRC_LIST})
  target_link_libraries(example1 miniz)
  add_executable(example2 ${EXAMPLE2_SRC_LIST})
  target_link_libraries(example2 miniz)
  add_executable(example3 ${EXAMPLE3_SRC_LIST})
  target_link_libraries(example3 miniz)
  add_executable(example4 ${EXAMPLE4_SRC_LIST})
  target_link_libraries(example4 miniz)
  add_executable(example5 ${EXAMPLE5_SRC_LIST})
  target_link_libraries(example5 miniz)
  add_executable(example6 ${EXAMPLE6_SRC_LIST})
  target_link_libraries(example6 miniz)
  if(${UNIX})
      target_link_libraries(example6 m)
  endif()

  # add_executable(miniz_tester ${MINIZ_TESTER_SRC_LIST})
  # target_link_libraries(miniz_tester miniz)
endif(BUILD_EXAMPLES)

set(INCLUDE_INSTALL_DIR "include")

install(FILES ${INSTALL_HEADERS} DESTINATION ${INCLUDE_INSTALL_DIR}/${PROJECT_NAME})

