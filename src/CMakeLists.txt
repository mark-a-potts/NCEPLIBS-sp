include("list_of_files.cmake")

if(CMAKE_Fortran_COMPILER_ID MATCHES "^(Intel)$")
  set(CMAKE_Fortran_FLAGS
    "-g -traceback -auto -convert big_endian -assume byterecl -fp-model strict -fpp -fPIC"
    )
  set(CMAKE_Fortran_FLAGS_RELEASE "-O3")
  set(fortran_d_flags "-i4 -r8")
  set(fortran_8_flags "-i8 -r8")
elseif(CMAKE_Fortran_COMPILER_ID MATCHES "^(GNU|Clang|AppleClang)$")
  set(CMAKE_Fortran_FLAGS "-g -fbacktrace -fconvert=big-endian -fPIC")
  set(CMAKE_Fortran_FLAGS_RELEASE "-O3")
  set(fortran_d_flags "-fdefault-real-8")
  set(fortran_8_flags "-fdefault-integer-8 -fdefault-real-8")
endif()

if(APPLE)
  add_compile_definitions(APPLE)
elseif(UNIX)
  add_compile_definitions(LINUX)
endif()

if(OPENMP)
  find_package(OpenMP REQUIRED COMPONENTS Fortran)
endif()
if(OpenMP_FOUND)
  add_compile_definitions(OPENMP)
endif()

set(kinds "4" "8" "d")
foreach(kind ${kinds})
  set(lib_name ${PROJECT_NAME}_${kind})
  set(module_dir ${CMAKE_CURRENT_BINARY_DIR}/include_${kind})

  set(BUILD_FLAGS "${fortran_${kind}_flags}")
  if(OpenMP_Fortran_FOUND)
    set(BUILD_FLAGS "${BUILD_FLAGS} ${OpenMP_Fortran_FLAGS}")
  endif()

  add_library(${lib_name} STATIC ${fortran_src})
  add_library(${PROJECT_NAME}::${lib_name} ALIAS ${lib_name})
  
  set_target_properties(${lib_name} PROPERTIES COMPILE_FLAGS "${BUILD_FLAGS}")
  set_target_properties(${lib_name} PROPERTIES Fortran_MODULE_DIRECTORY ${module_dir})
  set_target_properties(${lib_name} PROPERTIES INTERFACE_LINK_LIBRARIES ${lib_name})
  target_include_directories(${lib_name} PUBLIC
    $<BUILD_INTERFACE:${module_dir}>
    $<INSTALL_INTERFACE:include_${kind}>)

  list(APPEND LIB_TARGETS ${lib_name})
  install(DIRECTORY ${module_dir} DESTINATION ${CMAKE_INSTALL_PREFIX})
endforeach()

install(
  TARGETS ${LIB_TARGETS}
  EXPORT ${PROJECT_NAME}Exports
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)

### Package config
include(CMakePackageConfigHelpers)
set(CONFIG_INSTALL_DESTINATION lib/cmake/${PROJECT_NAME})

export(EXPORT ${PROJECT_NAME}Exports
  NAMESPACE ${PROJECT_NAME}::
  FILE ${PROJECT_NAME}-targets.cmake)

configure_package_config_file(
  ${CMAKE_SOURCE_DIR}/cmake/PackageConfig.cmake.in ${CMAKE_BINARY_DIR}/${PROJECT_NAME}-config.cmake
  INSTALL_DESTINATION ${CONFIG_INSTALL_DESTINATION})
install(FILES ${CMAKE_BINARY_DIR}/${PROJECT_NAME}-config.cmake
  DESTINATION ${CONFIG_INSTALL_DESTINATION})

write_basic_package_version_file(
  ${CMAKE_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion)
install(FILES ${CMAKE_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
  DESTINATION ${CONFIG_INSTALL_DESTINATION})

install(EXPORT ${PROJECT_NAME}Exports
  NAMESPACE ${PROJECT_NAME}::
  FILE ${PROJECT_NAME}-targets.cmake
  DESTINATION ${CONFIG_INSTALL_DESTINATION})
