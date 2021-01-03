# Clone and build the llvm-mos compiler, linker and assembler
cmake_minimum_required(VERSION 3.17)
project(build-llvm-mos-compiler-dist
    LANGUAGES ASM C CXX)

# Set constants
set(LLVM_MOS_GIT_URL https://github.com/johnwbyrd/llvm-mos.git
    CACHE STRING "URL of git repository containing the llvm-mos compilers")
set(LLVM_MOS_GIT_TAG master
    CACHE STRING "Tag, branch or hash of git repository to pull")
set(LLVM_MOS_CMAKE_CACHE_FILE_NAME LLVM-cache.cmake)
set(LLVM_MOS_CMAKE_CACHE_FILE_PATH ${DOWNLOAD_DIR}/${LLVM_MOS_CMAKE_CACHE_FILE_NAME})
set(LLVM_MOS_CMAKE_CACHE_URL 
    https://raw.githubusercontent.com/llvm-mos/llvm-mos-sdk/main/cmake/llvm-mos/${LLVM_MOS_CMAKE_CACHE_FILE_NAME}
    CACHE STRING "URL of CMake cache file")
set(LLVM_MOS_INSTALL_SUBPATH install)

include(CMakeDetermineCXXCompiler)

# Make adjustments for MSVC
set(LLVM_MOS_C_FLAGS "")
if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    list(APPEND LLVM_MOS_C_FLAGS /FS)
endif()

# Convince cmake that we really just want to download a file in this step
set(DO_NOTHING ${CMAKE_COMMAND} --version)
file(DOWNLOAD LLVM_MOS_CMAKE_CACHE_URL)
include(ExternalProject)
ExternalProject_Add( llvm-mos-download-cache
    URL ${LLVM_MOS_CMAKE_CACHE_URL}
    DOWNLOAD_NAME ${LLVM_MOS_CMAKE_CACHE_FILE_NAME}
    DOWNLOAD_NO_EXTRACT YES 
    CONFIGURE_COMMAND ${DO_NOTHING}
    BUILD_COMMAND ${DO_NOTHING}
    INSTALL_COMMAND ${DO_NOTHING}
)
ExternalProject_Get_Property(llvm-mos-download-cache DOWNLOAD_DIR)
set(LLVM_MOS_CMAKE_CACHE_FILE ${DOWNLOAD_DIR}/LLVM-cache.cmake)

# Actually build the llvm-mos compiler
ExternalProject_Add( llvm-mos
    GIT_REPOSITORY ${LLVM_MOS_GIT_URL}
    GIT_TAG ${LLVM_MOS_GIT_TAG}
    GIT_PROGRESS 1
    CMAKE_ARGS
        -C ${LLVM_MOS_CMAKE_CACHE_FILE}
        <SOURCE_DIR>/llvm
        -DCMAKE_C_FLAGS:STRING=${LLVM_MOS_C_FLAGS}
        -DCMAKE_CXX_FLAGS:STRING=${LLVM_MOS_C_FLAGS}
    CMAKE_CACHE_ARGS 
        -DCMAKE_INSTALL_PREFIX:STRING=<INSTALL_DIR>/${LLVM_MOS_INSTALL_SUBPATH}
        -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    TEST_COMMAND cmake --build . --target distribution
    PATCH_COMMAND
    TEST_BEFORE_INSTALL 1
    USES_TERMINAL_DOWNLOAD 1
    USES_TERMINAL_UPDATE 1
    USES_TERMINAL_CONFIGURE 1
    USES_TERMINAL_BUILD 1
    USES_TERMINAL_INSTALL 1
    USES_TERMINAL_TEST 1
)
ExternalProject_Get_Property(llvm-mos INSTALL_DIR)
add_dependencies(llvm-mos llvm-mos-download-cache)

string(TOLOWER ${CMAKE_SYSTEM_NAME} LLVM_MOS_PLATFORM_NAME)
set(LLVM_MOS_INSTALL_DIR ${INSTALL_DIR}/${LLVM_MOS_INSTALL_SUBPATH})
set(LLVM_MOS_PACKAGE_PATH  
    ${LLVM_MOS_INSTALL_DIR}/../llvm-mos-${LLVM_MOS_PLATFORM_NAME}-${LLVM_MOS_GIT_TAG}.zip)

add_custom_target(llvm-mos-package
    ALL
    COMMAND 
        zip -FSr -9 -v
            ${LLVM_MOS_PACKAGE_PATH}
            ./
    WORKING_DIRECTORY ${LLVM_MOS_INSTALL_DIR}
    BYPRODUCTS ${LLVM_MOS_PACKAGE_PATH}
    COMMENT Compressing release package ${LLVM_MOS_PACKAGE_PATH}
    DEPENDS llvm-mos
)
    
add_dependencies(llvm-mos-package llvm-mos)