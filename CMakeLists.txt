# CMake project for tsproc

# based in part on an example found at:
# http://lagrange.mechse.illinois.edu/mwest/partmc/partmc-1.2.0/CMakeLists.txt

message("Processing top-level CMakelists.txt for project tsproc")

#------------------------------------------------------------------------------------#
#                           Build directory check                                    #
#------------------------------------------------------------------------------------#
if(${CMAKE_SOURCE_DIR}/src STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "Cannot be built in the source directory. Use
  out-of-source build instead.")
  message(FATAL_ERROR "  cd /some/work/dir/build")
  message(FATAL_ERROR "  cmake -i /path/to/source")
endif(${CMAKE_SOURCE_DIR}/src STREQUAL ${CMAKE_BINARY_DIR})

#------------------------------------------------------------------------
# Set basic project settings
#------------------------------------------------------------------------
project (tsproc Fortran C)
enable_language (Fortran C)

SET(CPACK_GENERATOR "TGZ")
include(CPack)

set(PACKAGE_BUGREPORT "smwesten@usgs.gov")
set(PACKAGE_NAME "tsproc")
set(PACKAGE_STRING "tsproc 2.0")
set(PACKAGE_TARNAME "tsproc")
set(PACKAGE_VERSION "2.0")

set(CMAKE_INCLUDE_CURRENT_DIR ON)

cmake_minimum_required(VERSION 2.8)

option (MAKEFILE_VERBOSE
      "Produce verbose makefiles?" OFF)

option (TARGET__TSPROC_EXECUTABLE
      "Compile the main TSPROC executable?" ON)

option (TARGET__TSPROC_LIBRARY
      "Compile TSPROC as a library?" ON)

option (OPTION__UNROLL_CONTROL_FILE
      "Compile the code with support for control file unrolling?" OFF)
if(OPTION__UNROLL_CONTROL_FILE)
  set ( PREPROCESSOR_DEFINES ${PREPROCESSOR_DEFINES} "UNROLL_CONTROL_FILE")
endif()

#------------------------------------------------------------------------
# Restrict CMAKE_BUILD_TYPE to "Release" or "Debug" (GUI); set build type
#------------------------------------------------------------------------
#set_property(GLOBAL CMAKE_BUILD_TYPE
#          PROPERTY STRINGS "Release" "Debug")

set( CMAKE_BUILD_TYPE "Release" CACHE STRING
       "Compile in DEBUG or RELEASE mode" )

#------------------------------------------------------------------------
# What Fortran compiler are we using?
#------------------------------------------------------------------------
get_filename_component (Fortran_COMPILER_NAME ${CMAKE_Fortran_COMPILER} NAME)

#------------------------------------------------------------------------
# Locate utility programs
#------------------------------------------------------------------------

find_program( R_SCRIPT Rscript.exe Rscript
			  HINTS
			  ENV R_HOME
              PATHS
			  "c:/Program Files/R"
			  "c:/Program Files (x86)/R"
)

message("Rscript.exe is defined as: " ${R_SCRIPT})


#------------------------------------------------------------------------
# Set platform-specific options
#------------------------------------------------------------------------
if(CMAKE_HOST_WIN32)

    if (Fortran_COMPILER_NAME MATCHES "gfortran[.a-z]*")

      message("")
      message(" ***** setting options for gfortran on MinGW *****")
	  message("  PLATFORM_TYPE = " ${PLATFORM_TYPE} )
	  message("  MINGWDIR = " ${MINGWBASE} )

      set(CMAKE_INSTALL_PREFIX "d:/DOS" CACHE STRING
          "Location for installation of the executable")

      FIND_LIBRARY(GCC_LIB libgcc.a
	    ENV
		${LIBRARY_PATH}
	    HINTS
		${MINGWBASE}
      )

      FIND_LIBRARY(GFORTRAN_LIB libgfortran.a
	    ENV
		${LIBRARY_PATH}
	    HINTS
		${MINGWBASE}
      )

      set (CMAKE_Fortran_FLAGS_RELEASE "-O2 -mtune=native -ffree-line-length-none -static-libgcc -static-libgfortran -finit-local-zero")
      set (CMAKE_Fortran_FLAGS_DEBUG "-O0 -g -Wall -Wextra -fcheck=all -fexceptions -ffree-line-length-none -finit-local-zero")

      SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static -static-libgcc -static-libgfortran")
      set(EXTRA_LIBS ${EXTRA_LIBS} ${GCC_LIB} ${GFORTRAN_LIB})

      set_property(CACHE CMAKE_BUILD_TYPE
              PROPERTY STRINGS "Release" "Debug")

      add_custom_target(copy ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/src/tsproc.exe ${CMAKE_INSTALL_PREFIX}/tsproc.exe )

   endif ()

elseif(CMAKE_HOST_UNIX)

    if (Fortran_COMPILER_NAME MATCHES "^gfortran[.a-z]*")

      message("")
      message(" ***** setting options for gfortran on UNIX like OS *****")

      set (CMAKE_Fortran_FLAGS_RELEASE "-Ofast -fwhole-file -ffree-line-length-none")
      set (CMAKE_Fortran_FLAGS_DEBUG "-O0 -g -pg -fbounds-check -Wall -Wextra -fexceptions -ffree-line-length-none")

    endif ()

    # set options specific to the Intel compiler on Linux
    if (Fortran_COMPILER_NAME MATCHES "ifort[.a-z]*")

      message("")
      message(" ***** setting options for Intel fortran on Linux *****")

      set(CMAKE_INSTALL_PREFIX "/usr/local/bin" CACHE STRING
          "Location for installation of the executable")

      set (CMAKE_Fortran_FLAGS_RELEASE "-fast -arch ")
      set (CMAKE_Fortran_FLAGS_DEBUG "-warn all -traceback ")

      SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-intel -static-libgcc")
      set(EXTRA_LIBS "${EXTRA_LIBS} libgcc.a libgfortran.a"} )
      add_custom_target(copy ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/tsproc ${CMAKE_INSTALL_PREFIX}/tsproc )

    endif()

   # If not Linux and ifort, accept CMAKE defaults...
   # I think compiler choice is keyed off of the FC environment variable.

else()

   message(FATAL_ERROR "Windows (32/64)/gfortran/MinGW or Linux-64/Intel environments were not detected!")

endif()

message(" *****")
message("")

enable_testing()
add_subdirectory(src)
add_subdirectory(tests/io)
add_subdirectory(tests/calcs)
