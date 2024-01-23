cmake_minimum_required(VERSION 3.17.2)
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
set(CMAKE_MODULE_PATH "${CMAKE_INSTALL_PREFIX}/lib/cmake;${CMAKE_INSTALL_PREFIX}/share/cmake;${CMAKE_MODULE_PATH}")

option(BUILD_PARALLEL_LEVEL "Number of parallel build jobs")
if(NOT BUILD_PARALLEL_LEVEL)
    include(ProcessorCount)
    ProcessorCount(N)
    if(N GREATER 3)
        math(EXPR N "${N} - 2")
        set(BUILD_PARALLEL_LEVEL ${N})
    else()
        set(BUILD_PARALLEL_LEVEL 1)
    endif()
endif()
set(ENV{PKG_CONFIG_PATH} "${CMAKE_INSTALL_PREFIX}/lib/pkgconfig;${CMAKE_INSTALL_PREFIX}/share/pkgconfig")

function (build_ext_project target)
    set(PRJ_BUILD_DIR ${CMAKE_BINARY_DIR}/_deps/${target})
    FILE(MAKE_DIRECTORY ${PRJ_BUILD_DIR} ${PRJ_BUILD_DIR}/build)

    set(CMAKE_EXT_BUILD "cmake_minimum_required(VERSION 3.17.2)
    include(ExternalProject)
    ExternalProject_add(${target}
        SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${target}
        PREFIX ${PRJ_BUILD_DIR}
        CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} -DCMAKE_BUILD_TYPE=Release ${ARGN} -DCMAKE_MODULE_PATH=\"${CMAKE_MODULE_PATH}\"
        BUILD_COMMAND cmake --build . --parallel ${BUILD_PARALLEL_LEVEL}
        )
    add_custom_target(build_${target})
    add_dependencies(build_${target} ${target})
    ")
    FILE(WRITE ${PRJ_BUILD_DIR}/CMakeLists.txt "${CMAKE_EXT_BUILD}")

    #if (${target} STREQUAL "spirv_tools")
    #    execute_process(COMMAND python3 utils/git-sync-deps WORKING_DIRECTORY ${PRJ_BUILD_DIR})
    #endif()
    execute_process(COMMAND ${CMAKE_COMMAND} .. WORKING_DIRECTORY ${PRJ_BUILD_DIR}/build)
    execute_process(COMMAND ${CMAKE_COMMAND} --build . WORKING_DIRECTORY ${PRJ_BUILD_DIR}/build)
    execute_process(COMMAND ${CMAKE_COMMAND} --install . WORKING_DIRECTORY ${PRJ_BUILD_DIR}/build)
endfunction()


find_package(PkgConfig REQUIRED)
find_package(Vulkan 1.3.255)

if (NOT Vulkan_FOUND)
    pkg_check_modules(Vulkan vulkan>=1.3.255)
endif()

if (NOT Vulkan_FOUND)
    find_package(robin_hood)

    if (NOT robin_hood_FOUND)
        build_ext_project(robin-hood-hashing https://github.com/martinus/robin-hood-hashing.git 9145f963d80d6a02f0f96a47758050a89184a3ed -DRH_STANDALONE_PROJECT=OFF)
    endif()

    find_package(SPIRV-Headers 1.5.5)
    if (NOT SPIRV-Headers_FOUND)
        pkg_check_modules(SPIRV-Headers SPIRV-Headers>=1.5.5)
    endif()
    if (NOT SPIRV-Headers_FOUND)
        build_ext_project(SPIRV-Headers)
        find_package(SPIRV-Headers 1.5.5)
        if(NOT SPIRV-Headers_FOUND)
            pkg_check_modules(SPIRV-Headers SPIRV-Headers>=1.5.5)
        endif()
        if(NOT SPIRV-Headers_FOUND)
            message(FATAL_ERROR "SPIRV-Headers not found")
        endif()
        build_ext_project(SPIRV-Tools -DSPIRV_TOOLS_BUILD_STATIC=ON -DSPIRV-Headers_SOURCE_DIR=${CMAKE_SOURCE_DIR}/SPIRV-Headers -DINSTALL_GTEST=OFF -DBUILD_TESTING=OFF -DSPIRV_SKIP_TESTS=ON)
        find_package(SPIRV-Tools REQUIRED)
    endif()

    find_package(VulkanHeaders 1.3.268 CONFIG)
    set(ILLIXR_VULKAN_HEADERS_DIR "")
    if (NOT VulkanHeaders_FOUND)
        build_ext_project(Vulkan-Headers)
        find_package(VulkanHeaders 1.3.268 REQUIRED CONFIG)
        set(ILLIXR_VULKAN_HEADERS_DIR "-DVulkanHeaders_DIR=${CMAKE_INSTALL_PREFIX}/share/cmake/VulkanHeaders")

        build_ext_project(Vulkan-Utility-Libraries -DUPDATE_DEPS=ON ${ILLIXR_VULKAN_HEADERS_DIR})

        build_ext_project(Vulkan-ValidationLayers)

        build_ext_project(Vulkan-Loader -DUPDATE_DEPS=ON)
        find_package(VulkanLoader REQUIRED)
    endif()
endif()

find_package(glslang 13.1.1)

if (NOT glslang_FOUND)
    build_ext_project(glslang -DALLOW_EXTERNAL_SPIRV_TOOLS=ON -DSPIRV-Tools-opt_DIR=${CMAKE_INSTALL_PREFIX}/lib/cmake/SPIRV-Tools-opt -DSPIRV-Tools_DIR=${CMAKE_INSTALL_PREFIX}/lib/cmake/SPIRV-Tools)
    find_package(glslang 13.1.1 REQUIRED)
endif()
