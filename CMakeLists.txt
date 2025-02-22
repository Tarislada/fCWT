cmake_minimum_required(VERSION 3.10)

if (NOT (UNIX OR WIN32))
  message(FATAL_ERROR "unsupported target platform")
endif()

#Build options
option(BUILD_SHARED_LIBS "Build shared library?" ON)
option(BUILD_MATLAB "Build MATLAB MEX-files" OFF)

if(NOT DEFINED CMAKE_BUILD_TYPE OR NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING
      "Build type: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()
message("Build type is ${CMAKE_BUILD_TYPE}")

#Installation options
if(UNIX)
  set(FCWT_MATLAB_DIR "../MATLAB" CACHE
   STRING "Name of the build directory for the MATLAB MEX-files")
  set(FCWT_CMAKE_INSTALL_DIR "share/fcwt/cmake" CACHE
    STRING "Name of the installation directory for the CMake stuff relative to ${CMAKE_INSTALL_PREFIX}")
  set(FCWT_LIB_INSTALL_DIR "lib" CACHE
    STRING "Name of the installation directory for the libraries relative to ${CMAKE_INSTALL_PREFIX}")
  set(FCWT_INCLUDE_INSTALL_DIR "include" CACHE
    STRING "Name of the installation directory for the includes relative to ${CMAKE_INSTALL_PREFIX}")
elseif(WIN32)
  # On Windows, CMAKE_INSTALL_PREFIX is $ENV{ProgramFiles}\\${CMAKE_PROJECT_NAME} by default. In turn:
  #   - on AMD64: ProgramFiles=%ProgramFiles%
  #   - on   x86: ProgramFiles=%ProgramFiles(x86)%
  # See: https://msdn.microsoft.com/en-us/library/aa384274.aspx
  set(FCWT_MATLAB_DIR "../MATLAB" CACHE
   STRING "Name of the build directory for the MATLAB MEX-files")
  set(FCWT_CMAKE_INSTALL_DIR "cmake" CACHE
    STRING "Name of share directory relative to ${CMAKE_INSTALL_PREFIX}")
  set(FCWT_LIB_INSTALL_DIR "lib" CACHE
    STRING "Name of the installation directory for the libraries relative to ${CMAKE_INSTALL_PREFIX}")
   set(FCWT_INCLUDE_INSTALL_DIR "include" CACHE
    STRING "Name of the installation directory for the includes relative to ${CMAKE_INSTALL_PREFIX}")
endif()

#Use gcc version 10. If your system has a different version, please change these two lines.
set(CMAKE_C_COMPILER "gcc-10")
set(CMAKE_CXX_COMPILER "g++-10")

# set the project name
project(fCWT VERSION 1.0 DESCRIPTION "Highly optimized implementation of the Continuous Wavelet Transform")

# specify the C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED True)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")

if(UNIX)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx2 -O2 -lm")
elseif(WIN32)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++17 /arch:AVX2 /O2 /W1")
endif()

find_package(OpenMP REQUIRED)
if (OPENMP_FOUND)
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
endif()

link_directories(${PROJECT_SOURCE_DIR})

set(libraryheaders 
    "./src/fcwt.h"
)
set(implementationheaders 
    "./src/main.h" 
    "./src/rwave.h" 
    "./src/wavelib.h" 
) 
set(librarysources 
    "./src/fcwt.cpp"
)
set(implementationsources 
    "./src/main.cpp" 
    "./src/rwave.cpp"
    "./src/wavelib.cpp"
)

set(headers ${libraryheaders} ${implementationheaders})
set(sources ${librarysources} ${implementationsources})

# targets 
set(fcwt_library_target "fCWT")

#library
if(BUILD_SHARED_LIBS)
    message("Building of shared library is enabled.")
    add_definitions(-DFCWT_LIBRARY_DLL_BUILDING)
else()
    message("Building of static library is enabled.")
endif()

add_library(${fcwt_library_target} ${libraryheaders} ${librarysources})

set_target_properties(${fcwt_library_target}
    PROPERTIES
    LINKER_LANGUAGE "CXX"
    VERSION ${PROJECT_VERSION}
)


# add the executable
add_executable(fCWT_example ${headers} ${sources}) 
include_directories("${PROJECT_SOURCE_DIR}/src/libwavelib")
include_directories("${PROJECT_SOURCE_DIR}/src/fftw")
include_directories("${PROJECT_SOURCE_DIR}/src/MEX")

find_library (
    FFTW
    NAMES fftw3f libfftw3f # what to look for
    HINTS "${PROJECT_SOURCE_DIR}/src/fftw" # where to look
    NO_DEFAULT_PATH # do not search system default paths
    REQUIRED
)
if(UNIX)
	find_library (
		FFTW_OMP
		NAMES fftw3f_omp libfftw3f_omp # what to look for
		HINTS "${PROJECT_SOURCE_DIR}/src/fftw" # where to look
		NO_DEFAULT_PATH # do not search system default paths
		REQUIRED
	)
endif()
find_library (
    WAVELIB
    NAMES wavelib libwavelib # what to look for
    HINTS "${PROJECT_SOURCE_DIR}/src/libwavelib" # where to look
    NO_DEFAULT_PATH # do not search system default paths
    REQUIRED
)

# check if we found the library
message(${PROJECT_SOURCE_DIR})
message("STATUS OF LIBRARY: FFTW: [${FFTW}]")
if(UNIX)
	message("STATUS OF LIBRARY: FFTW_OMP: [${FFTW_OMP}]")
endif()
message("STATUS OF LIBRARY: WAVELIB: [${WAVELIB}]")

if(FFTW)
    target_link_libraries(fCWT_example PRIVATE ${FFTW})
    target_link_libraries(${fcwt_library_target} PRIVATE ${FFTW})
else()
    message(SEND_ERROR "Did not find lib fftw3f")
endif()

if(UNIX)
	if(FFTW_OMP)
		target_link_libraries(fCWT_example PRIVATE ${FFTW_OMP})
		target_link_libraries(${fcwt_library_target} PRIVATE ${FFTW_OMP})
	else()
		message(SEND_ERROR "Did not find lib fftw3f_omp")
	endif()
endif()

if (NOT WAVELIB)
    message(SEND_ERROR "Did not find lib wavelib")
else()
    target_link_libraries(fCWT_example PRIVATE ${WAVELIB})
endif()

if(WIN32)
	file(COPY "${PROJECT_SOURCE_DIR}/src/fftw/fftw3f.dll" DESTINATION ${PROJECT_BINARY_DIR}/Debug)
	file(COPY "${PROJECT_SOURCE_DIR}/src/fftw/fftw3f.lib" DESTINATION ${PROJECT_BINARY_DIR}/Debug)
	file(COPY "${PROJECT_SOURCE_DIR}/src/fftw/fftw3f.dll" DESTINATION ${PROJECT_BINARY_DIR}/Release)
	file(COPY "${PROJECT_SOURCE_DIR}/src/fftw/fftw3f.lib" DESTINATION ${PROJECT_BINARY_DIR}/Release)
	file(COPY ${WAVELIB} DESTINATION ${PROJECT_BINARY_DIR}/Debug)
	file(COPY ${WAVELIB} DESTINATION ${PROJECT_BINARY_DIR}/Release)
endif()


if(BUILD_MATLAB)
    set(MATLAB_FIND_DEBUG)
    
    #Add MATLAB 2021a as possible export
    set(MATLAB_ADDITIONAL_VERSIONS "R2021a=9.10")
    
    find_package(Matlab)
    
    matlab_add_mex(
        NAME fCWTmex
        SRC src/MEX/fcwtmex.cpp src/fcwt.h src/fcwt.cpp
        OUTPUT_NAME ${FCWT_MATLAB_DIR}/fCWT
        LINK_TO ${FFTW} ${FFTW_OMP}
        R2018a
    )
    
    matlab_add_mex(
        NAME fCWTmexplan
        SRC src/MEX/fcwtplan.cpp src/fcwt.h src/fcwt.cpp
        OUTPUT_NAME ${FCWT_MATLAB_DIR}/fCWT_create_plan
        LINK_TO ${FFTW} ${FFTW_OMP}
        R2018a
    )
endif()


# Installing

install(FILES ${libraryheaders}
    DESTINATION ${FCWT_INCLUDE_INSTALL_DIR})

install(TARGETS ${fcwt_library_target}
    EXPORT ${fcwt_library_target}_export
    ARCHIVE  DESTINATION ${FCWT_LIB_INSTALL_DIR}
    LIBRARY  DESTINATION ${FCWT_LIB_INSTALL_DIR}
    RUNTIME  DESTINATION ${FCWT_LIB_INSTALL_DIR}
    INCLUDES DESTINATION ${FCWT_INCLUDE_INSTALL_DIR})

install(EXPORT ${fcwt_library_target}_export
  DESTINATION ${FCWT_CMAKE_INSTALL_DIR}
  FILE ${fcwt_library_target}-config.cmake)
