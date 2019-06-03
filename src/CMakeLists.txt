cmake_minimum_required( VERSION 2.8.12 )

cmake_policy(SET CMP0057 NEW)

#include (GenerateExportHeader)

#configure_file("${PROJECT_SOURCE_DIR}/include/miopen/config.h.in" "${PROJECT_BINARY_DIR}/include/miopen/config.h")

# configure a header file to pass the CMake version settings to the source, and package the header files in the output archive
#configure_file( "${PROJECT_SOURCE_DIR}/include/miopen/version.h.in" "${PROJECT_BINARY_DIR}/include/miopen/version.h" )

#message( STATUS "MIOpen_VERSION= ${MIOpen_VERSION}" )
#message( STATUS "CMAKE_BUILD_TYPE= ${CMAKE_BUILD_TYPE}" )

# This is incremented when the ABI to the library changes
#set( MIOpen_SOVERSION 1 )

file(GLOB Rpp_Source
     "${CMAKE_CURRENT_LIST_DIR}/imgaug/*.cpp"
    )

message("rpp source" ${Rpp_Source})
if( RPP_BACKEND STREQUAL "OpenCL" )
add_definitions(-DMOD_CL_PATH="${CMAKE_CURRENT_LIST_DIR}/kernels/")
file(GLOB Rpp_Source_cl "${CMAKE_CURRENT_LIST_DIR}/ocl/*.cpp")
    list(APPEND Rpp_Source ${Rpp_Source_cl})
endif()

# build library
add_library( RPP
    ${Rpp_Source}
    )

set_target_properties(RPP PROPERTIES SOVERSION 1)

clang_tidy_check(RPP)

function(target_internal_library TARGET)
    target_link_libraries(${TARGET} PRIVATE ${ARGN})
    set(PASS_ARGS debug optimized)
    set(DEPS)
    foreach(DEP ${ARGN})
        if(DEP IN_LIST PASS_ARGS)
            list(APPEND DEPS ${DEP})
        else()
            list(APPEND DEPS $<BUILD_INTERFACE:${DEP}>)
        endif()
    endforeach()
    target_link_libraries(${TARGET} INTERFACE ${DEPS})
endfunction()

target_include_directories(RPP PUBLIC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src/include>
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src/include/rpp>
)


#generate_export_header(MIOpen
#    EXPORT_FILE_NAME ${PROJECT_BINARY_DIR}/include/miopen/export.h
#)
set(PACKAGE_DEPENDS)
############################################################
# RPP depends on OpenCL
if( RPP_BACKEND STREQUAL "OpenCL")
    MESSAGE( STATUS "RPP linking OpenCL: ${OPENCL_INCLUDE_DIRS}" )
    target_include_directories(RPP SYSTEM PUBLIC ${OPENCL_INCLUDE_DIRS} )
    target_link_libraries( RPP PUBLIC ${OPENCL_LIBRARIES} )
    list(APPEND PACKAGE_DEPENDS PACKAGE OpenCL)
    set(BACKEND_PACKAGE "OpenCL")
elseif(RPP_BACKEND STREQUAL "HIPOC" OR RPP_BACKEND STREQUAL "HIP")
    target_link_libraries( RPP PRIVATE hip::hip_hcc hip::device )
    target_link_libraries( RPP INTERFACE hip::hip_hcc )
    if(ENABLE_HIP_WORKAROUNDS)
        # Workaround hip not setting its usage requirements correctly
        target_compile_definitions( RPP PRIVATE -D__HIP_PLATFORM_HCC__=1 )
    endif()
    # This is helpful for the tests
    target_link_libraries( RPP INTERFACE $<BUILD_INTERFACE:${hip_LIBRARIES}> )
    list(APPEND PACKAGE_DEPENDS PACKAGE hip)
    set(BACKEND_PACKAGE "hip")
endif()

############################################################
if(rocblas_FOUND)
    target_link_libraries( RPP INTERFACE $<BUILD_INTERFACE:roc::rocblas> )
    target_link_libraries( RPP PRIVATE roc::rocblas )
endif()

if(WIN32 AND NOT MSVC)
    if(BUILD_DEV)
        target_link_libraries(RPP PUBLIC -Wl,-export-all-symbols -Wl,-exclude-symbols=_Unwind_Resume)
    endif()
    target_link_libraries(RPP PUBLIC -Wl,--whole-archive -lgcc -lstdc++-6 -Wl,--no-whole-archive -Wl,--allow-multiple-definition)
endif()

target_include_directories(RPP PUBLIC $<BUILD_INTERFACE:${Boost_INCLUDE_DIRS}>)
#target_internal_library(RPP
#    optimized ${Boost_FILESYSTEM_LIBRARY_RELEASE}
#    optimized ${Boost_SYSTEM_LIBRARY_RELEASE}
#    debug ${Boost_FILESYSTEM_LIBRARY_DEBUG}
#    debug ${Boost_SYSTEM_LIBRARY_DEBUG}
#)

############################################################
# Installation
rocm_install_targets(
  TARGETS RPP
  INCLUDE
    ${PROJECT_SOURCE_DIR}/include
    ${PROJECT_BINARY_DIR}/include
  PREFIX ${RPP_INSTALL_DIR}
 )

rocm_export_targets(
  TARGETS RPP
  PREFIX ${RPP_INSTALL_DIR}
  DEPENDS
    ${PACKAGE_DEPENDS}
)



rocm_install_symlink_subdir(${RPP_INSTALL_DIR})
