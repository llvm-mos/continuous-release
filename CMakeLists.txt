# Clone and build the llvm-mos compiler, linker and assembler
# cmake 3.15 permits us to build multiple targets in one go
cmake_minimum_required(VERSION 3.15)
project(build-llvm-mos-compiler-dist
    LANGUAGES C CXX)

# Set constants
if(NOT DEFINED LLVM_MOS_GIT_URL)
    set(LLVM_MOS_GIT_URL https://github.com/llvm-mos/llvm-mos.git
    CACHE STRING "URL of git repository containing the llvm-mos compilers")
endif()
if(NOT DEFINED LLVM_MOS_GIT_TAG)
    set(LLVM_MOS_GIT_TAG main
    CACHE STRING "Tag, branch or hash of git repository to pull")
endif()
if(NOT DEFINED LLVM_MOS_GIT_RELEASE_SLUG)
    set(LLVM_MOS_RELEASE_SLUG ${LLVM_MOS_GIT_TAG}
        CACHE STRING "The short, human-readable, no-slashes name of the release to create (i.e. feature-do-linking)")
endif()
if(NOT DEFINED CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE MinSizeRel
        CACHE STRING "Build type for compilers")
endif()
set(LLVM_MOS_CMAKE_CACHE_FILE clang/cmake/caches/MOS.cmake
    CACHE STRING "Path from root of project to the MOS cmake cache file")
set(LLVM_MOS_GH_RELEASE_CREATE_FLAGS --prerelease 
    CACHE STRING "Flags to pass to Github CLI's release create command")
set(LLVM_MOS_TEST_SUITE_ENABLED No
    CACHE BOOL "Should this build run the test suite for the llvm-mos compilers?")
set(LLVM_MOS_TEST_SUITE_TARGET check-llvm 
    CACHE STRING "The target to cmake in order to test")
set(LLVM_MOS_INSTALL_SUBPATH dist/llvm-mos)

set(LLVM_MOS_TEST_SUITE_DISABLED (NOT ${LLVM_MOS_TEST_SUITE_ENABLED}))

# Find whatever local host compiler we have
include(CMakeDetermineCXXCompiler)

# Make adjustments for MSVC
set(LLVM_MOS_C_FLAGS "")
if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    list(APPEND LLVM_MOS_C_FLAGS /FS)
    list(REMOVE_DUPLICATES LLVM_MOS_C_FLAGS)
endif()

include(ExternalProject)

set(LLVM_MOS_CMAKE_ARGS         
    -C <SOURCE_DIR>/${LLVM_MOS_CMAKE_CACHE_FILE}
    <SOURCE_DIR>/llvm
    -DCMAKE_C_FLAGS:STRING=${LLVM_MOS_C_FLAGS}
    -DCMAKE_CXX_FLAGS:STRING=${LLVM_MOS_C_FLAGS}
    )

# Any Windows specific flags go here
if (WIN32)
    list(APPEND LLVM_MOS_CMAKE_ARGS "")
    list(REMOVE_DUPLICATES LLVM_MOS_CMAKE_ARGS)
endif()

set(LLVM_MOS_CMAKE_CACHE_ARGS "")

# Download and build the llvm-mos compiler
ExternalProject_Add( build
    GIT_REPOSITORY ${LLVM_MOS_GIT_URL}
    GIT_TAG ${LLVM_MOS_GIT_TAG}
    GIT_PROGRESS 1
    GIT_SHALLOW 0
    CMAKE_ARGS ${LLVM_MOS_CMAKE_ARGS}
    CMAKE_CACHE_ARGS ${LLVM_MOS_CMAKE_CACHE_ARGS} 
        -DCMAKE_INSTALL_PREFIX:STRING=<INSTALL_DIR>/${LLVM_MOS_INSTALL_SUBPATH}
    INSTALL_COMMAND cmake -E echo "Skipping install step"
    COMMAND cmake --build . --target install
    TEST_COMMAND cmake --build . --target ${LLVM_MOS_TEST_SUITE_TARGET}
    TEST_AFTER_INSTALL YES
    TEST_EXCLUDE_FROM_MAIN ${LLVM_MOS_TEST_SUITE_DISABLED}
    USES_TERMINAL_DOWNLOAD 1
    USES_TERMINAL_UPDATE 1
    USES_TERMINAL_CONFIGURE 1
    USES_TERMINAL_BUILD 1
    USES_TERMINAL_INSTALL 1
    USES_TERMINAL_TEST 1
)
ExternalProject_Get_Property(build INSTALL_DIR)
ExternalProject_Get_Property(build SOURCE_DIR)

string(TOLOWER ${CMAKE_SYSTEM_NAME} LLVM_MOS_PLATFORM_NAME)

set(LLVM_MOS_INSTALL_DIR ${INSTALL_DIR}/${LLVM_MOS_INSTALL_SUBPATH})
set(LLVM_MOS_SOURCE_DIR ${SOURCE_DIR})
set(LLVM_MOS_RELEASE_NAME llvm-mos-${LLVM_MOS_PLATFORM_NAME}-${LLVM_MOS_RELEASE_SLUG})
set(LLVM_MOS_PACKAGE_PATH  
    ${LLVM_MOS_INSTALL_DIR}/../../${LLVM_MOS_RELEASE_NAME}.zip)

set(extra_zip_flags)
if (NOT WIN32)
    set(extra_zip_flags -y)
endif()

find_program(LLVM_MOS_ZIP zip REQUIRED)
add_custom_target(package
    COMMAND 
        ${LLVM_MOS_ZIP} -FSr ${extra_zip_flags} -9 -v
            ${LLVM_MOS_PACKAGE_PATH}
            ./
    WORKING_DIRECTORY ${LLVM_MOS_INSTALL_DIR}/..
    BYPRODUCTS ${LLVM_MOS_PACKAGE_PATH}
    COMMENT Compressing release package ${LLVM_MOS_PACKAGE_PATH}
    DEPENDS build
)

find_program(LLVM_MOS_GIT git REQUIRED)
find_program(LLVM_MOS_GH gh REQUIRED)

string(TIMESTAMP LLVM_MOS_DATE_TIME UTC)

add_custom_target(ship
# Ignore the gh release command if it fails by throwing in a git command
# that always succeeds.
    COMMAND 
        cd
    COMMAND
        ${LLVM_MOS_GH} release delete ${LLVM_MOS_RELEASE_NAME} -y ||
           echo "No previous release ${LLVM_MOS_RELEASE_NAME} to delete" 
    COMMAND
        git push --delete origin ${LLVM_MOS_RELEASE_NAME} ||
            echo "No previous tag ${LLVM_MOS_RELEASE_NAME} to delete"
    COMMAND
        echo "This is an automated build of the ${LLVM_MOS_RELEASE_SLUG} branch for ${CMAKE_SYSTEM_NAME}." > git.log
    COMMAND
        echo "Repository: ${LLVM_MOS_GIT_URL}" >> git.log
    COMMAND
        echo "Branch: ${LLVM_MOS_RELEASE_SLUG}" >> git.log
    COMMAND
        echo "Tag: ${LLVM_MOS_GIT_TAG}" >> git.log
    COMMAND
        echo "Target: ${CMAKE_SYSTEM_NAME}" >> git.log
    COMMAND
        echo "Time: ${LLVM_MOS_DATE_TIME}" >> git.log
    COMMAND
        ${LLVM_MOS_GH} release create 
            ${LLVM_MOS_RELEASE_NAME} 
            --notes-file git.log
            --title ${LLVM_MOS_RELEASE_NAME}
            --target ${LLVM_MOS_GIT_TAG}
            ${LLVM_MOS_GH_RELEASE_CREATE_FLAGS}
            ${LLVM_MOS_PACKAGE_PATH} 
    DEPENDS package
    WORKING_DIRECTORY ${LLVM_MOS_SOURCE_DIR}
    COMMENT Compressing release package ${LLVM_MOS_PACKAGE_PATH}
)
