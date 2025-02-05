cmake_minimum_required(VERSION 3.22)
set(CMAKE_VERBOSE_MAKEFILE True)

set(PACKAGE_STRING "ILLIXR-vulkan 1.3.268")
set(PACKAGE_VERSION_MAJOR "1")
set(PACKAGE_VERSION_MINOR "3")
set(PACKAGE_VERSION_PATCH "268")
set(PACKAGE_VERSION_LABEL "")

if(PACKAGE_VERSION_LABEL)
    set(PACKAGE_VERSION "${PACKAGE_VERSION_MAJOR}.${PACKAGE_VERSION_MINOR}.${PACKAGE_VERSION_PATCH}-${PACKAGE_VERSION_LABEL}")
else()
    set(PACKAGE_VERSION "${PACKAGE_VERSION_MAJOR}.${PACKAGE_VERSION_MINOR}.${PACKAGE_VERSION_PATCH}")
endif()

project(ILLIXR_vulkan VERSION 1.3.268)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_MODULE_PATH "${CMAKE_INSTALL_PREFIX}/lib/cmake")

option(BUILD_PARALLEL_LEVEL "Number of parallel build jobs")
if(NOT BUILD_PARALLEL_LEVEL)
    include(ProcessorCount)
    ProcessorCount(N)
    if(N GREATER 3)
        math(EXPR N "${N} / 2")
        set(BUILD_PARALLEL_LEVEL ${N})
    else()
        set(BUILD_PARALLEL_LEVEL 1)
    endif()
endif()
set(ENV{PKG_CONFIG_PATH} "${CMAKE_INSTALL_PREFIX}/lib/pkgconfig;${CMAKE_INSTALL_PREFIX}/share/pkgconfig")

function(build_ext_project target)
    set(PRJ_BUILD_DIR ${CMAKE_BINARY_DIR}/_deps/${target})
    FILE(MAKE_DIRECTORY ${PRJ_BUILD_DIR} ${PRJ_BUILD_DIR}/build)

    set(CMAKE_EXT_BUILD "cmake_minimum_required(VERSION 3.22)
project(${target})
include(ExternalProject)
ExternalProject_add(${target}
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${target}
    PREFIX ${PRJ_BUILD_DIR}
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} -DCMAKE_BUILD_TYPE=Release ${ARGN} -DCMAKE_MODULE_PATH=${CMAKE_MODULE_PATH}
    BUILD_COMMAND cmake --build . --parallel ${BUILD_PARALLEL_LEVEL}
    )
add_custom_target(build_${target})
add_dependencies(build_${target} ${target})"
    )
    FILE(WRITE ${PRJ_BUILD_DIR}/CMakeLists.txt "${CMAKE_EXT_BUILD}")

    execute_process(COMMAND ${CMAKE_COMMAND} .. WORKING_DIRECTORY ${PRJ_BUILD_DIR}/build)
    execute_process(COMMAND ${CMAKE_COMMAND} --build . WORKING_DIRECTORY ${PRJ_BUILD_DIR}/build)
    execute_process(COMMAND ${CMAKE_COMMAND} --install . WORKING_DIRECTORY ${PRJ_BUILD_DIR}/build)
endfunction()


find_package(PkgConfig REQUIRED)

find_package(robin_hood QUIET)

if (NOT robin_hood_FOUND)
    build_ext_project(robin-hood-hashing -DRH_STANDALONE_PROJECT=OFF)
endif()

find_package(SPIRV-Headers 1.5.5 QUIET)
if (NOT SPIRV-Headers_FOUND)
    pkg_check_modules(SPIRV-Headers SPIRV-Headers>=1.5.5)
endif()

if (NOT SPIRV-Headers_FOUND)
    build_ext_project(SPIRV-Headers)
    find_package(SPIRV-Headers 1.5.5 QUIET)
    if(NOT SPIRV-Headers_FOUND)
        pkg_check_modules(SPIRV-Headers REQUIRED SPIRV-Headers>=1.5.5)
    endif()

    build_ext_project(SPIRV-Tools -DSPIRV_TOOLS_BUILD_STATIC=ON -DSPIRV-Headers_SOURCE_DIR=${CMAKE_SOURCE_DIR}/SPIRV-Headers -DINSTALL_GTEST=OFF -DBUILD_TESTING=OFF -DSPIRV_SKIP_TESTS=ON)
    find_package(SPIRV-Tools REQUIRED)
endif()

build_ext_project(Vulkan-Headers)

build_ext_project(Vulkan-Utility-Libraries -DUPDATE_DEPS=ON "-DVulkanHeaders_DIR=${CMAKE_INSTALL_PREFIX}/share/cmake/VulkanHeaders")

build_ext_project(Vulkan-ValidationLayers)

build_ext_project(Vulkan-Loader -DUPDATE_DEPS=ON)
#find_package(VulkanLoader REQUIRED)

find_package(glslang 13.1.1)

if (NOT glslang_FOUND)
    build_ext_project(glslang -DALLOW_EXTERNAL_SPIRV_TOOLS=ON -DSPIRV-Tools-opt_DIR=${CMAKE_INSTALL_PREFIX}/lib/cmake/SPIRV-Tools-opt -DSPIRV-Tools_DIR=${CMAKE_INSTALL_PREFIX}/lib/cmake/SPIRV-Tools)
    #find_package(glslang 13.1.1 REQUIRED)
endif()
