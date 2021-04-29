cmake_minimum_required(VERSION 3.11.4)

project(
  Nccl
  VERSION 1.0
  LANGUAGES C CXX)

option(NCCL_BUILD_SHARED "Build as shared library" ON)

enable_language(CUDA)

if(${CMAKE_MINOR_VERSION} LESS_EQUAL 17)
  find_package(CUDA REQUIRED)
  set(NCCL_CUDA_LIBRARIES "${CUDA_LIBRARIES}")
  set(NCCL_CURAND_LINK_LIBRARY ${CUDA_curand_LIBRARY})
  set(NCCL_CUDA_INCLUDE_DIRECTORIES ${CUDA_INCLUDE_DIRS})
  set(NCCL_CUDA_VERSION_MAJOR ${CUDA_VERSION_MAJOR})
else()
  find_package(CUDAToolkit REQUIRED)
  set(NCCL_CUDA_VERSION_MAJOR ${CUDAToolkit_VERSION_MAJOR})
  set(NCCL_CUDA_LIBRARIES "CUDA::cudart_static")
  set(NCCL_CUDA_INCLUDE_DIRECTORIES ${CUDAToolkit_INCLUDE_DIRS})
  set(NCCL_CURAND_LINK_LIBRARY "CUDA::curand")
endif()

if(POLICY CMP0074)
  cmake_policy(SET CMP0074 NEW)
endif()

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE
      "Release"
      CACHE STRING "Choose the build type, e.g. Debug." FORCE)
endif()

file(GLOB NCCL_INCLUDE_FILES src/*.h)
file(GLOB NCCL_SOURCE_FILES src/*.cu)

if(NCCL_BUILD_SHARED)
  add_library(Nccl SHARED ${NCCL_SOURCE_FILES} ${NCCL_INCLUDE_FILES})
else()
  add_library(Nccl STATIC ${NCCL_SOURCE_FILES} ${NCCL_INCLUDE_FILES})
endif()

add_library(Nccl::Nccl ALIAS Nccl)

target_compile_definitions(Nccl PRIVATE COMPILING_NCCL)

if(NOT NCCL_BUILD_SHARED)
  target_compile_definitions(Nccl PUBLIC NCCL_STATIC_LIB)
endif()

target_include_directories(
  Nccl
  PUBLIC $<BUILD_INTERFACE:${Nccl_BINARY_DIR}/src>
         $<BUILD_INTERFACE:${Nccl_SOURCE_DIR}/src> $<INSTALL_INTERFACE:include>)

add_executable(all_gather_test test/single/all_gather_test.cu)
target_include_directories(all_gather_test PRIVATE test/include)
target_link_libraries(all_gather_test PRIVATE Nccl ${NCCL_CURAND_LINK_LIBRARY})
list(APPEND NCCL_TESTS all_gather_test)

add_executable(all_reduce_test test/single/all_reduce_test.cu)
target_include_directories(all_reduce_test PRIVATE test/include)
target_link_libraries(all_reduce_test PRIVATE Nccl ${NCCL_CURAND_LINK_LIBRARY})
list(APPEND NCCL_TESTS all_reduce_test)

add_executable(broadcast_test test/single/broadcast_test.cu)
target_include_directories(broadcast_test PRIVATE test/include)
target_link_libraries(broadcast_test PRIVATE Nccl ${NCCL_CURAND_LINK_LIBRARY})
list(APPEND NCCL_TESTS broadcast_test)

add_executable(reduce_test test/single/reduce_test.cu)
target_include_directories(reduce_test PRIVATE test/include)
target_link_libraries(reduce_test PRIVATE Nccl ${NCCL_CURAND_LINK_LIBRARY})
list(APPEND NCCL_TESTS reduce_test)

add_executable(reduce_scatter_test test/single/reduce_scatter_test.cu)
target_include_directories(reduce_scatter_test PRIVATE test/include)
target_link_libraries(reduce_scatter_test PRIVATE Nccl ${NCCL_CURAND_LINK_LIBRARY})
list(APPEND NCCL_TESTS reduce_scatter_test)

configure_file(${Nccl_SOURCE_DIR}/NcclConfig.cmake.in
               ${Nccl_BINARY_DIR}/NcclConfig.cmake @ONLY)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "NcclConfigVersion.cmake"
  VERSION ${Nccl_VERSION}
  COMPATIBILITY SameMajorVersion)

install(
  TARGETS Nccl ${NCCL_TESTS}
  EXPORT NcclTargets
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  RUNTIME DESTINATION bin
  INCLUDES
  DESTINATION include)
install(FILES ${NCCL_INCLUDE_FILES} DESTINATION include)

# CMake package file for find_package(Nccl::Nccl)
install(
  EXPORT NcclTargets
  FILE NcclTargets.cmake
  NAMESPACE Nccl::
  DESTINATION lib/cmake/Nccl)
install(FILES ${Nccl_BINARY_DIR}/NcclConfig.cmake
              ${Nccl_BINARY_DIR}/NcclConfigVersion.cmake
        DESTINATION lib/cmake/Nccl)

# Status Message for Build Options ############################################
#
message("")
message("Nccl build configuration:")
message("  Nccl Version: ${Nccl_VERSION}")
message("  C++ Compiler : ${CMAKE_CXX_COMPILER_ID} "
        "${CMAKE_CXX_COMPILER_VERSION} " "${CMAKE_CXX_COMPILER_WRAPPER}")
message("    ${CMAKE_CXX_COMPILER}")
message("  CUDA Compiler : ${CMAKE_CUDA_COMPILER} ")
message("")
message("  Installation prefix: ${CMAKE_INSTALL_PREFIX}")
message("")
message("  Build Type: ${CMAKE_BUILD_TYPE}")
