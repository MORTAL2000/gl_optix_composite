cmake_minimum_required(VERSION 2.8)
set(PROJNAME gl_optix_composite)
Project(${PROJNAME})
Message(STATUS "-------------------------------")
Message(STATUS "Processing Project ${PROJNAME}:")

#####################################################################################
# offer the choice of having shared_sources as a sub-folder... good for packaging a sample
#
# if BASE_DIRECTORY not defined, it means this cmake file was called as the first entry point and not included

if(NOT BASE_DIRECTORY)
  if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/shared_sources)
    Message(STATUS "found sub-folder shared_sources")
    SET(BASE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
    INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/shared_sources/CMakeLists_include.txt)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/shared_sources ${CMAKE_BINARY_DIR}/shared_sources)
  elseif(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/../../shared_sources)
    SET(BASE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/../..)
    INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/../../shared_sources/CMakeLists_include.txt)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../../shared_sources ${CMAKE_BINARY_DIR}/shared_sources)
  else()
    SET(BASE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/..)
    INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/../shared_sources/CMakeLists_include.txt)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../shared_sources ${CMAKE_BINARY_DIR}/shared_sources)
  endif()
endif()

_add_project_definitions(${PROJNAME})

#####################################################################################
# additions from packages needed for this sample
# add refs  in LIBRARIES_OPTIMIZED
# add refs  in LIBRARIES_DEBUG
# add files in PACKAGE_SOURCE_FILES
#
unset(USE_OPTIX CACHE)

# use a custom FindCUDA module instead of the default CMake version
set(CMAKE_MODULE_PATH "${BASE_DIRECTORY}/shared_sources/cmake/private;${CMAKE_MODULE_PATH}")
_add_package_Cuda ()

if ( CUDA_FOUND )
   _add_package_Optix ()   
   if ( OPTIX_FOUND )        
	   SET(USE_OPTIX ON CACHE BOOL "Use OPTIX")	   
	   add_definitions(-DBUILD_OPTIX)	   
	   add_definitions(-DBUILD_OPTIX_PRIME)
	else ()
	   SET(USE_OPTIX OFF CACHE BOOL "Use Optix")       
	endif()
else ()
   SET(USE_OPTIX OFF CACHE BOOL "Use Optix")   
endif()

#####################################################################################
# Source files for this project
#
file(GLOB SOURCE_FILES src/*.cpp src/*.hpp src/*.inl src/*.h src/*.c)

#####################################################################################
# additional files for tools/helpers
#
SET(COMMON_SOURCE_FILES ${GLEW_SOURCE}   
)

#####################################################################################
# Custom Build for OptiX CUDA files
#####################################################################################

# Get list of .cu file
file(GLOB CUDA_FILES "shaders/*.cu" )

# Compile to PTX as build step
NVCUDA_COMPILE_PTX( SOURCES ${CUDA_FILES} TARGET_PATH "${CMAKE_BINARY_DIR}/ptx" GENERATED_FILES PTX_SOURCES NVCC_OPTIONS "-arch=sm_20" "-I${OPTIX_ROOT_DIR}/include" )

#if ( USE_OPTIX )
#	file(GLOB CUDA_LIST "shaders/*.cu")
#   unset(CUDA_TARGETS CACHE)
#	foreach (CUFILE ${CUDA_LIST} )
#	  get_filename_component ( CUDIR "${CUFILE}" PATH )
#	  get_filename_component ( CUROOT "${CUFILE}" NAME_WE )
#	  message ( STATUS "Compiling CUDA file: ${CUDIR}/${CUROOT}.cu" )
#	  add_custom_command (         
#		 OUTPUT ${CUROOT}.ptx
#		 COMMAND "${CUDA_ROOT_DIR}\\bin\\nvcc" --use_fast_math --use-local-env --cl-version 2010 -Xcompiler "/Gm,/Zi" ${CUFILE} -I${OPTIX_ROOT_DIR}/include -I../src -ptx -o ${CUROOT}.ptx
#		 DEPENDS ${CUFILE} 
#		 COMMAND ${CMAKE_COMMAND} ARGS -E copy_if_different ${CUROOT}.ptx $<TARGET_FILE_DIR:${PROJNAME}>
#		 VERBATIM
#		 )    
#	  add_custom_target ( ${CUROOT} ALL DEPENDS ${CUROOT}.ptx )
#	  list(APPEND CUDA_TARGETS "${CUROOT}.ptx")  
#	endforeach()
#	message ( STATUS "Targets: ${CUDA_TARGETS}" )
# endif()

#####################################################################################
# Executable
#
if(WIN32 AND NOT GLUT_FOUND)
  add_definitions(/wd4267) #remove size_t to int warning
  add_definitions(/wd4996) #remove printf warning
  add_definitions(/wd4244) #remove double to float conversion warning
  add_definitions(/wd4305) #remove double to float truncation warning
  add_executable(${PROJNAME} WIN32 ${SOURCE_FILES} ${COMMON_SOURCE_FILES} ${PACKAGE_SOURCE_FILES} ${CUDA_FILES} ${PTX_SOURCES})
else()
  add_executable(${PROJNAME} ${SOURCE_FILES} ${COMMON_SOURCE_FILES} ${PACKAGE_SOURCE_FILES} ${CUDA_FILES} ${PTX_SOURCES})
endif()

#####################################################################################
# common source code needed for this sample
#
source_group(common FILES 
  ${COMMON_SOURCE_FILES}
)
source_group(CUDA FILES 
  ${CUDA_FILES}
)
source_group(ptx FILES 
  ${PTX_SOURCES}
)

#####################################################################################
# Linkage
#
target_link_libraries(${PROJNAME} optimized
    ${LIBRARIES_OPTIMIZED}
    ${PLATFORM_LIBRARIES} 
)
target_link_libraries(${PROJNAME} debug
    ${LIBRARIES_DEBUG}
    ${PLATFORM_LIBRARIES} 
)
message ( STATUS "Libraries used: ${LIBRARIES_DEBUG}" )




#####################################################################################
# copies binaries that need to be put next to the exe files (ZLib, etc.)
#
_copy_binaries_to_target( ${PROJNAME} )


