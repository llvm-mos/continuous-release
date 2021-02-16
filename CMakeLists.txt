# Clone and build the llvm-mos compiler, linker and assembler
cmake_minimum_required(VERSION 3.10)
project(build-llvm-mos-compiler-dist
    LANGUAGES C CXX)

# Set constants
set(LLVM_MOS_GIT_URL https://github.com/llvm-mos/llvm-mos.git
    CACHE STRING "URL of git repository containing the llvm-mos compilers")
set(LLVM_MOS_GIT_TAG main
    CACHE STRING "Tag, branch or hash of git repository to pull")
set(LLVM_MOS_RELEASE_SLUG ${LLVM_MOS_GIT_TAG}
    CACHE STRING "The short, human-readable, no-slashes name of the release to create (i.e. feature-do-linking)")
set(LLVM_MOS_CMAKE_CACHE_FILE_NAME LLVM-cache.cmake)
set(LLVM_MOS_CMAKE_CACHE_FILE_PATH ${DOWNLOAD_DIR}/${LLVM_MOS_CMAKE_CACHE_FILE_NAME})
set(LLVM_MOS_CMAKE_CACHE_URL 
    https://raw.githubusercontent.com/llvm-mos/llvm-mos-sdk/main/cmake/llvm-mos/${LLVM_MOS_CMAKE_CACHE_FILE_NAME}
    CACHE STRING "URL of CMake cache file")
set(LLVM_MOS_GH_RELEASE_CREATE_FLAGS --prerelease 
    CACHE STRING "Flags to pass to Github CLI's release create command")
set(CMAKE_BUILD_TYPE Release
    CACHE STRING "Build type for compilers")
set(LLVM_MOS_DISABLE_TEST_SUITE YES 
    CACHE BOOL "Should this build skip the test suite for the llvm-mos compilers?")
set(LLVM_MOS_INSTALL_SUBPATH install)

# Find whatever local host compiler we have
include(CMakeDetermineCXXCompiler)

# Make adjustments for MSVC
set(LLVM_MOS_C_FLAGS "")
if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    list(APPEND LLVM_MOS_C_FLAGS /FS)
    list(REMOVE_DUPLICATES LLVM_MOS_C_FLAGS)
endif()

# Convince cmake that we really just want to download a file in this step
set(DO_NOTHING ${CMAKE_COMMAND} --version)

include(ExternalProject)
ExternalProject_Add( download-cache
    URL ${LLVM_MOS_CMAKE_CACHE_URL}
    DOWNLOAD_NAME ${LLVM_MOS_CMAKE_CACHE_FILE_NAME}
    DOWNLOAD_NO_EXTRACT YES 
    UPDATE_COMMAND ${DO_NOTHING}
    CONFIGURE_COMMAND ${DO_NOTHING}
    BUILD_COMMAND ${DO_NOTHING}
    INSTALL_COMMAND ${DO_NOTHING}
)

ExternalProject_Get_Property(download-cache DOWNLOAD_DIR)
set(LLVM_MOS_CMAKE_CACHE_FILE ${DOWNLOAD_DIR}/${LLVM_MOS_CMAKE_CACHE_FILE_NAME})

set(LLVM_MOS_CMAKE_ARGS         
    -C ${LLVM_MOS_CMAKE_CACHE_FILE}
    <SOURCE_DIR>/llvm
    -DCMAKE_C_FLAGS:STRING=${LLVM_MOS_C_FLAGS}
    -DCMAKE_CXX_FLAGS:STRING=${LLVM_MOS_C_FLAGS}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    )
# Use LLVM compiler on Windows
if (WIN32)
    list(APPEND LLVM_MOS_CMAKE_ARGS "-Tllvm")
    list(REMOVE_DUPLICATES LLVM_MOS_CMAKE_ARGS)
endif()

set(LLVM_MOS_CMAKE_CACHE_ARGS "")

# Actually build the llvm-mos compiler
ExternalProject_Add( build
    GIT_REPOSITORY ${LLVM_MOS_GIT_URL}
    GIT_TAG ${LLVM_MOS_GIT_TAG}
    GIT_PROGRESS 1
    CMAKE_ARGS ${LLVM_MOS_CMAKE_ARGS}
    CMAKE_CACHE_ARGS ${LLVM_MOS_CMAKE_CACHE_ARGS} 
        -DCMAKE_INSTALL_PREFIX:STRING=<INSTALL_DIR>/${LLVM_MOS_INSTALL_SUBPATH}
    INSTALL_COMMAND cmake --build . --target distribution
    COMMAND cmake --build . --target install
    TEST_COMMAND cmake --build . --target check-all
    TEST_AFTER_INSTALL YES
    TEST_EXCLUDE_FROM_MAIN ${LLVM_MOS_DISABLE_TEST_SUITE}
    USES_TERMINAL_DOWNLOAD 1
    USES_TERMINAL_UPDATE 1
    USES_TERMINAL_CONFIGURE 1
    USES_TERMINAL_BUILD 1
    USES_TERMINAL_INSTALL 1
    USES_TERMINAL_TEST 1
)
ExternalProject_Get_Property(build INSTALL_DIR)
add_dependencies(build download-cache)

string(TOLOWER ${CMAKE_SYSTEM_NAME} LLVM_MOS_PLATFORM_NAME)
set(LLVM_MOS_INSTALL_DIR ${INSTALL_DIR}/${LLVM_MOS_INSTALL_SUBPATH})
set(LLVM_MOS_RELEASE_NAME llvm-mos-${LLVM_MOS_PLATFORM_NAME}-${LLVM_MOS_RELEASE_SLUG})
set(LLVM_MOS_PACKAGE_PATH  
    ${LLVM_MOS_INSTALL_DIR}/../${LLVM_MOS_RELEASE_NAME}.zip)

find_program(LLVM_MOS_ZIP zip REQUIRED)
add_custom_target(package
    COMMAND 
        ${LLVM_MOS_ZIP} -FSr -9 -v
            ${LLVM_MOS_PACKAGE_PATH}
            ./
    WORKING_DIRECTORY ${LLVM_MOS_INSTALL_DIR}
    BYPRODUCTS ${LLVM_MOS_PACKAGE_PATH}
    COMMENT Compressing release package ${LLVM_MOS_PACKAGE_PATH}
    DEPENDS build
)

find_program(LLVM_MOS_GIT git REQUIRED)
find_program(LLVM_MOS_GH gh REQUIRED)
execute_process(
    COMMAND
        ${LLVM_MOS_GIT} rev-parse HEAD
    OUTPUT_VARIABLE
        LLVM_MOS_GIT_HASH
)

add_custom_target(ship
# Ignore the gh release command if it fails by throwing in a git command
# that always succeeds.
    COMMAND
        ${LLVM_MOS_GH} release delete ${LLVM_MOS_RELEASE_NAME} -y ||
           echo "No previous release of ${LLVM_MOS_RELEASE_NAME} to delete" 
    COMMAND
        ${LLVM_MOS_GH} release create 
            ${LLVM_MOS_RELEASE_NAME} 
            ${LLVM_MOS_GH_RELEASE_CREATE_FLAGS}
            ${LLVM_MOS_PACKAGE_PATH} 
    DEPENDS package
)