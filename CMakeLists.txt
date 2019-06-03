cmake_minimum_required( VERSION 2.8.12 )

# This has to be initialized before the project() command appears
# Set the default of CMAKE_BUILD_TYPE to be release, unless user specifies with -D.  MSVC_IDE does not use CMAKE_BUILD_TYPE
if( NOT MSVC_IDE AND NOT CMAKE_BUILD_TYPE )
    set( CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." )
endif()

# Default installation path
if(WIN32)
    set(CMAKE_INSTALL_PREFIX "/opt/rocm/x86_64-w64-mingw32" CACHE PATH "")
else()
    set(CMAKE_INSTALL_PREFIX "/opt/rocm" CACHE PATH "")
endif()

project (RPP C CXX )

enable_testing()

find_package(ROCM REQUIRED PATHS /opt/rocm)

include(ROCMInstallTargets)
include(ROCMPackageConfigHelpers)
include(ROCMSetupVersion)
include(ROCMInstallSymlinks)
include(ROCMCreatePackage)
include(CheckCXXCompilerFlag)

# Strip symbols for release
if(NOT WIN32 AND NOT APPLE)
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -s")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -s")
endif()

rocm_setup_version(VERSION 1.8.1)

list( APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake )

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "5.3")
        message(FATAL_ERROR "RPP requires at least gcc 5.3")
    endif()
endif()

############################################################
# require C++14
add_compile_options(-std=c++14)

############################################################
# OPTION - MIOpen Backend
# - OpenCL
# - HCC
if(CMAKE_CXX_COMPILER MATCHES ".*hcc" OR CMAKE_CXX_COMPILER MATCHES ".*hipcc")
    set(RPP_DEFAULT_BACKEND "HIP")
else()
    set(RPP_DEFAULT_BACKEND "OpenCL")
endif()

if(WIN32 AND CMAKE_CROSSCOMPILING)
    set(WINE_CMD "wine")
else()
    set(WINE_CMD)
endif()

option(ENABLE_HIP_WORKAROUNDS Off)
if(ENABLE_HIP_WORKAROUNDS)
    # Add these to CMAKE_PREFIX_PATH to workaround installation problems with hip and hcc
    list(APPEND CMAKE_PREFIX_PATH /opt/rocm/hcc /opt/rocm/hip)
endif()

set( RPP_BACKEND ${RPP_DEFAULT_BACKEND} CACHE STRING
    "Which of Rpps's backends to use?" )
set_property( CACHE RPP_BACKEND PROPERTY STRINGS
    OpenCL HIP HIPOC )
# OpenCL 1.2
if( RPP_BACKEND STREQUAL "OpenCL")
    set(RPP_BACKEND_OPENCL 1)
    find_package( OpenCL REQUIRED )
endif()

# HIP
if( RPP_BACKEND STREQUAL "HIP" OR RPP_BACKEND STREQUAL "HIPOC")
    list(APPEND CMAKE_PREFIX_PATH /opt/rocm /opt/rocm/hip /opt/rocm/hcc)
    set(RPP_BACKEND_HIP 1)
    find_program(HIP_OC_COMPILER clang-ocl
        PATH_SUFFIXES bin
        PATHS /opt/rocm
    )
    if(HIP_OC_COMPILER)
        message(STATUS "hip compiler: ${HIP_OC_COMPILER}")
        set(HIP_OC_COMPILER "${HIP_OC_COMPILER}")
    else()
        message(FATAL_ERROR "clang-ocl not found")
    endif()
    find_package(hip REQUIRED)
    link_libraries(stdc++)

    # A hack to make this work without the device enumerator
    link_libraries(-amdgpu-target=gfx803 -amdgpu-target=gfx900 -Wno-unused-command-line-argument)
    CHECK_CXX_COMPILER_FLAG(-amdgpu-target=gfx906 -Werror, GFX_906_FLAG_VALID)
    if(GFX_906_FLAG_VALID)
        link_libraries(-amdgpu-target=gfx906)
    endif()
    # end hack

    # rocblas
    set(RPP_USE_ROCBLAS ON CACHE BOOL "")
    if(RPP_USE_ROCBLAS)
        find_package(rocblas REQUIRED PATHS /opt/rocm)
        message(STATUS "Build with rocblas")
    else()
        message(STATUS "Build without rocblas")
    endif()
endif()
message( STATUS "${RPP_BACKEND} backend selected." )


option( BUILD_SHARED_LIBS "Build as a shared library" ON )

option( BUILD_DEV "Build for development only" OFF)

set( RPP_INSTALL_DIR rpp)
set( DATA_INSTALL_DIR ${RPP_INSTALL_DIR}/${CMAKE_INSTALL_DATAROOTDIR}/rpp )



set(CPACK_DEBIAN_PACKAGE_DEPENDS "rocm-opencl-dev, rocm-utils, hip_hcc")
set(CPACK_RPM_PACKAGE_REQUIRES "rocm-opencl-devel, rocm-utils, hip_hcc")

rocm_create_package(
    NAME RPP-${RPP_BACKEND}
    DESCRIPTION "AMD's Optimized CV and DataAugmentation Library"
    MAINTAINER "lokesh <lokeswara@multicorewareinc.com>"
    LDCONFIG
    # DEPENDS rocm-opencl rocm-utils hip_hcc tinygemm
)

include(EnableCompilerWarnings)
if(CMAKE_CXX_COMPILER MATCHES ".*hcc" OR CMAKE_CXX_COMPILER MATCHES ".*hipcc" )
    set(CLANG_TIDY_CHECKS -modernize-use-override -readability-non-const-parameter)
endif()
include(ClangTidy)
enable_clang_tidy(
    CHECKS
        *
        -cert-env33-c
        -android-cloexec-fopen
        # Yea we shouldn't be using rand()
        -cert-msc30-c
        -cert-msc50-cpp
        -clang-analyzer-alpha.core.CastToStruct
        -clang-analyzer-optin.performance.Padding
        -clang-diagnostic-deprecated-declarations
        -clang-diagnostic-extern-c-compat
        -clang-diagnostic-unused-command-line-argument
        -cppcoreguidelines-pro-bounds-array-to-pointer-decay
        -cppcoreguidelines-pro-bounds-constant-array-index
        -cppcoreguidelines-pro-bounds-pointer-arithmetic
        -cppcoreguidelines-pro-type-member-init
        -cppcoreguidelines-pro-type-reinterpret-cast
        -cppcoreguidelines-pro-type-union-access
        -cppcoreguidelines-pro-type-vararg
        -cppcoreguidelines-special-member-functions
        -fuchsia-*
        -google-explicit-constructor
        -google-readability-braces-around-statements
        -google-readability-todo
        -google-runtime-int
        -google-runtime-references
        -hicpp-braces-around-statements
        -hicpp-explicit-conversions
        -hicpp-no-array-decay
        # We really shouldn't use bitwise operators with signed integers, but
        # opencl leaves us no choice
        -hicpp-signed-bitwise
        -hicpp-special-member-functions
        -hicpp-use-equals-default
        -hicpp-use-override
        -llvm-header-guard
        -llvm-include-order
        -misc-misplaced-const
        -modernize-pass-by-value
        -modernize-use-default-member-init
        -modernize-use-equals-default
        -modernize-use-transparent-functors
        -performance-unnecessary-value-param
        -readability-braces-around-statements
        -readability-else-after-return
        -readability-named-parameter
        -hicpp-use-auto
	    -modernize-use-auto
        -cppcoreguidelines-avoid-magic-numbers
        -readability-magic-numbers
        -cppcoreguidelines-macro-usage
        -misc-non-private-member-variables-in-classes
        -cppcoreguidelines-non-private-member-variables-in-classes
        -readability-isolate-declaration
        -cppcoreguidelines-avoid-c-arrays
        -hicpp-avoid-c-arrays
        -modernize-avoid-c-arrays
        -readability-uppercase-literal-suffix
        -hicpp-uppercase-literal-suffix
        -cert-msc32-c
        -cert-msc51-cpp
        -bugprone-exception-escape
        
        ${CLANG_TIDY_CHECKS}
    ERRORS
        *
        -readability-inconsistent-declaration-parameter-name
    HEADER_FILTER
        ".*hpp"
    EXTRA_ARGS
        -DRPP_USE_CLANG_TIDY

)
include(CppCheck)
enable_cppcheck(
    CHECKS
        all
    SUPPRESS
        ConfigurationNotChecked
        unmatchedSuppression
        unusedFunction
        # constStatement
        noExplicitConstructor
        passedByValue
        # preprocessorErrorDirective
        unusedStructMember
        # Ignore initializer lists in the tests
        useInitializationList:*test/*.cpp
        *:*.cl
        *:*src/kernels/*.h
    FORCE
    SOURCES
        addkernels/
        # driver/
        include/
        src/
        test/
    INCLUDE
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${CMAKE_CURRENT_BINARY_DIR}/include
        ${CMAKE_CURRENT_SOURCE_DIR}/src/include
    DEFINE
        RPP_USE_RPPGEMM=1
)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)

add_subdirectory(src)
