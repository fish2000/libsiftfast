# exact C++ implementation of lowe's sift program
# author: zerofrog(@gmail.com), Sep 2008
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU Lesser General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# Lesser GNU General Public License for more details.
#
# You should have received a copy of the GNU Lesser General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
cmake_minimum_required (VERSION 2.4.7)
project (libsiftfast)
set( CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS TRUE )
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}")

# Differences between CMake 2.4 and 2.6
if("${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}" GREATER 2.4)
   message(STATUS "Using cmake version ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}" )
   # some developers may be using an cmake cvs version which didn't have set_property() yet
   # Tell them that a more recent version is required.
   if(NOT COMMAND SET_PROPERTY)
      message(FATAL_ERROR "You are using an old version of CMake from cvs, please update to CMake >= 2.6.0 or cvs at least from Feb 20th, 2008")
   endif(NOT COMMAND SET_PROPERTY)

   # CMP0003: add the link paths to the link command as with cmake 2.4
   cmake_policy(SET CMP0003 OLD)
endif()

# Add the automatically determined parts of the RPATH
# which point to directories outside the build tree to the install RPATH
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

include(CheckCXXSourceRuns)

if( CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX )
  set(CMAKE_CXX_FLAGS_RELEASE "-O3 -g")
  add_definitions(" -Wall -fPIC -v ")

  # check for x86-64 system
  check_cxx_source_runs("
  int main()
  {
    int a = 0;
    int*pa = &a;
    asm(\".intel_syntax\\\\n\"
	\"mov %%rax, %0\\\\n\"
    \"mov %%eax, [%%rax]\\\\n\"
    \".att_syntax\\\\n\"
    : : \"r\"(pa) : \"%rax\");
    return 0;
  }"
  IS_X86_64)

  if( IS_X86_64 )
    add_definitions("-D__x86_64__")
  endif()
else()
  set(IS_X86_64 0)
endif()

include(CheckIncludeFile)
include(CheckLibraryExists)
include(CheckCXXSourceRuns)
include(CheckCXXCompilerFlag)

if( UNIX OR CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX )
  set(STDC_LIBRARY stdc++)
else()
  set(STDC_LIBRARY)
endif()

# check for SSE extensions
if( CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX )
  set(SSE_FLAGS)

  set(CMAKE_REQUIRED_FLAGS "-msse2")
  check_cxx_source_runs("
    #include <emmintrin.h>

    int main()
    {
        __m128d a, b;
        double vals[2] = {0};
        a = _mm_loadu_pd(vals);
        b = _mm_add_pd(a,a);
        _mm_storeu_pd(vals,b);
        return 0;
     }"
     HAS_SSE2_EXTENSIONS)

  set(CMAKE_REQUIRED_FLAGS "-msse")
  check_cxx_source_runs("
    #include <xmmintrin.h>
    int main()
    {
        __m128 a, b;
        float vals[4] = {0};
        a = _mm_loadu_ps(vals);
        b = a;
        b = _mm_add_ps(a,b);
        _mm_storeu_ps(vals,b);
        return 0;
    }"
    HAS_SSE_EXTENSIONS)

  set(CMAKE_REQUIRED_FLAGS)

  if(HAS_SSE2_EXTENSIONS)
    message(STATUS "Using SSE2 extensions")
    set(SSE_FLAGS "-msse2 -mfpmath=sse")
  elseif(HAS_SSE_EXTENSIONS)
    message(STATUS "Using SSE extensions")
    set(SSE_FLAGS "-msse -mfpmath=sse")
  endif()

  add_definitions(${SSE_FLAGS})
elseif(MSVC)
  check_cxx_source_runs("
    #include <emmintrin.h>

    int main()
    {
        __m128d a, b;
        double vals[2] = {0};
        a = _mm_loadu_pd(vals);
        b = _mm_add_pd(a,a);
        _mm_storeu_pd(vals,b);
        return 0;
     }"
     HAS_SSE2_EXTENSIONS)
  if( HAS_SSE2_EXTENSIONS )
    message(STATUS "Using SSE2 extensions")
    add_definitions( "/arch:SSE2 /fp:fast -D__SSE__ -D__SSE2__" )
  endif()
endif()

set(Boost_ADDITIONAL_VERSIONS "1.40" "1.39" "1.38" "1.37.0" "1.37" "1.35.0" "1.34.1" "1.34.0" "1.34" "1.33.1" "1.33.0" "1.33")
if( NOT $ENV{BOOST_INCLUDEDIR} STREQUAL "" )
  set(Boost_INCLUDE_DIR $ENV{BOOST_INCLUDEDIR})
endif()
if( NOT $ENV{BOOST_LIBRARYDIR} STREQUAL "" )
  set(Boost_LIBRARY_DIRS $ENV{BOOST_LIBRARYDIR})
endif()

find_package(Boost
    COMPONENTS python
)

if( Boost_FOUND )
  message(STATUS "found boost version: ${Boost_VERSION}")
else()
  message(STATUS "Could not find boost libraries!")
endif()


#find_library(
#    CPPUNIT_LIBRARY_RELEASE
#            NAMES
#            PATHS   /usr/local/Cellar/cppunit/1.12.1/lib
#                    /usr/lib
#                    /usr/lib64
#                    /usr/local/lib
#                    /usr/local/lib64
#)

add_library(libsiftfast SHARED libsiftfast.cpp)
add_library(libsiftfast-static STATIC libsiftfast.cpp)




if( CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX )
  target_link_libraries(libsiftfast m stdc++)
endif()

target_link_libraries(libsiftfast m stdc++ dl python2.7)

set_target_properties(libsiftfast PROPERTIES OUTPUT_NAME  siftfast CLEAN_DIRECT_OUTPUT 1)
if( MSVC )
  set(LIBSIFTFAST_NAME libsiftfast-s)
else()
  set(LIBSIFTFAST_NAME siftfast)
endif()
set_target_properties(libsiftfast-static PROPERTIES OUTPUT_NAME ${LIBSIFTFAST_NAME} CLEAN_DIRECT_OUTPUT 1)

#if(UNIX AND CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
#  set(CMAKE_INSTALL_PREFIX "/usr/local" CACHE PATH "libsiftfast install prefix" FORCE )
#endif()

message(STATUS "installing to ${CMAKE_INSTALL_PREFIX}")

# compile without depending on libsiftfast
include_directories(${CMAKE_CURRENT_SOURCE_DIR})
add_executable(siftfast libsiftfast.cpp siftfast.cpp)# profiler.cpp)
#set_target_properties(siftfast PROPERTIES COMPILE_FLAGS "-DDVPROFILE")
target_link_libraries(siftfast libsiftfast m stdc++ dl python2.7)

#
# generate python bindings via boost-python
#
set(BUILD_SIFTFASTPY)
if( Boost_PYTHON_FOUND )
  find_package(PythonLibs)

  if( PYTHONLIBS_FOUND OR PYTHON_LIBRARIES )

    find_package(PythonInterp)
    if( NOT PYTHON_EXECUTABLE )
      # look specifically for 2.6
      FIND_PROGRAM(PYTHON_EXECUTABLE
        NAMES python2.7 python
        PATHS [HKEY_LOCAL_MACHINE\\SOFTWARE\\Python\\PythonCore\\2.6\\InstallPath])
    endif()

    if( PYTHON_EXECUTABLE )
      # get the site-packages directory
      execute_process(
        COMMAND ${PYTHON_EXECUTABLE} -c "from distutils.sysconfig import get_python_lib; print get_python_lib()"
        OUTPUT_VARIABLE _python_sitepackage
        RESULT_VARIABLE _python_failed)
      if( ${_python_failed} EQUAL 0 )
        string(REGEX REPLACE "[\r\n]" "" _python_sitepackage "${_python_sitepackage}")
        set(PYTHON_INCLUDE_PATH ${PYTHON_INCLUDE_PATH} ${_python_sitepackage}/numpy/core/include)
      else()
        message(STATUS "failed to get python site-package directory")
      endif()
    endif()

    add_definitions(${Boost_CFLAGS})
    set(CMAKE_REQUIRED_INCLUDES ${PYTHON_INCLUDE_PATH} ${Boost_INCLUDE_DIR} )
    set(CMAKE_REQUIRED_LIBRARIES ${PYTHON_LIBRARIES} ${Boost_PYTHON_LIBRARY} ${Boost_THREAD_LIBRARY})
    set(CMAKE_REQUIRED_FLAGS ${Boost_CFLAGS})

    # check if all header files can be included
    check_cxx_source_runs("
    #include <Python.h>
    #include <boost/thread/thread.hpp>
    #include <boost/thread/mutex.hpp>
    #include <boost/thread/condition.hpp>
    #include <boost/shared_ptr.hpp>
    #define PY_ARRAY_UNIQUE_SYMBOL PyArrayHandle
    #include <boost/python.hpp>
    #include <pyconfig.h>
    #include <numpy/arrayobject.h>
    int main()
    {
      return 0;
    }"
    HAVE_ALL_PYTHON_HEADERS)

    if( HAVE_ALL_PYTHON_HEADERS )
      message(STATUS "python and boost-python found")
      include_directories(${PYTHON_INCLUDE_PATH} ${Boost_INCLUDE_DIRS})
      link_directories(${Boost_LIBRARY_DIRS})
      add_library(siftfastpy SHARED siftfastpy.cpp libsiftfast.cpp)
      # stdc++ has to be included before opengl libraries due to some ATI bug (http://wiki.fifengine.de/Segfault_in_cxa_allocate_exception#Workaround)
      target_link_libraries(siftfastpy ${STDC_LIBRARY} ${PYTHON_LIBRARIES} ${Boost_PYTHON_LIBRARY})

      set_target_properties(siftfastpy PROPERTIES PREFIX "")
      if( WIN32 )
        set_target_properties(siftfastpy PROPERTIES SUFFIX ".pyd")
      endif()
      set(BUILD_SIFTFASTPY 1)
      install(TARGETS siftfastpy DESTINATION lib/python2.7/site-packages PERMISSIONS
        OWNER_READ OWNER_WRITE OWNER_EXECUTE
        GROUP_READ GROUP_EXECUTE
        WORLD_READ WORLD_EXECUTE)
    else()
      message(STATUS "failed to use boost python libraries, check if python-numpy is installed.")
    endif()
  else()
    message(STATUS "failed to find python-dev please install it")
  endif()
else()
  message(STATUS "failed to find boost-python, please install it")
endif()

# check for OpenMP
if( NOT DEFINED USE_OPENMP OR USE_OPENMP  )

  if( WIN32 )
    CHECK_INCLUDE_FILE(omp.h HAVE_OMP_H)
    if( HAVE_OMP_H )
      message(STATUS "Using OpenMP")
      check_cxx_compiler_flag(/openmp HAVE_OPENMP)

      if( HAVE_OPENMP )
        message(STATUS "compiling with openmp support")
        add_definitions("/openmp")
      endif()
    endif()

  elseif(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)

    # check if compilers supports -fopenmp
    INCLUDE(CheckCCompilerFlag)
    check_cxx_compiler_flag(-fopenmp HAVE_OPENMP)
    check_library_exists(gomp omp_get_num_threads "" HAS_GOMP_LIB)

    if( HAVE_OPENMP AND HAS_GOMP_LIB )
      message(STATUS "compiling with openmp support")
      add_definitions("-fopenmp")
      target_link_libraries(siftfast gomp)
      target_link_libraries(libsiftfast gomp)
      target_link_libraries(libsiftfast-static gomp)
      if( BUILD_SIFTFASTPY )
        target_link_libraries(siftfastpy gomp)
      endif()
      set(OPENMP_LFLAGS "-lgomp")
    endif()
  endif()
endif()

#
# generate mex files
#

set(MATLAB MATLAB-NOTFOUND)
if( WIN32 )
  FIND_PROGRAM(MATLAB NAME "mex.bat" PATHS )
else()
  FIND_PROGRAM(MATLAB NAME "mex" PATHS )
endif()

set(MEX_LIBS)
if( CMAKE_COMPILER_IS_GNUC OR CMAKE_COMPILER_IS_GNUCXX )
  if( WIN32 )
    set(MEX_CXXFLAGS "${OPENMP_LFLAGS} ")
  else()
    # check for lapack
    check_library_exists(lapack _init "" HAS_LAPACK_LIB)
    if( HAS_LAPACK_LIB )
      set(MEX_LIBS "-llapack")
    endif()
    set(MEX_CXXFLAGS "-Wl,-rpath,${CMAKE_INSTALL_PREFIX}/lib ${OPENMP_LFLAGS}")
  endif()
elseif(MSVC)
  set(MEX_CXXFLAGS "-L\"${libsiftfast_BINARY_DIR}/Release\" -L\"${libsiftfast_BINARY_DIR}/Debug\"")
else()
  set(MEX_CXXFLAGS)
endif()

if(MATLAB)
  # check if the mex file is actually matlab (can be confused with latex)
  EXEC_PROGRAM(${MATLAB} ARGS "-version" OUTPUT_VARIABLE MEX_TEST_OUT RETURN_VALUE MEX_TEST_RETURN)
  #message(STATUS "mex result: ${MEX_TEST_OUT}")
  string(REGEX MATCH "MATLAB" IS_MATLAB "${MEX_TEST_OUT}")

  if( IS_MATLAB )
    set(USE_MATLAB 1)
  else()
    set(USE_MATLAB)
  endif()
endif()

if(USE_MATLAB)

  if( DARWIN OR APPLE )
    set(MEXEXT "mexmac")
  elseif( UNIX )
    if( IS_X86_64 )
      set(MEXEXT "mexa64")
    else()
      set(MEXEXT "mexglx")
    endif()
  elseif( WIN64 )
    set(MEXEXT "mexw64")
  elseif( WIN32 OR CYGWIN OR WINDOWS )
    set(MEXEXT "mexw32")
  else()
    set(MEXEXT "mex")
  endif()

  set(MATLAB_MEX "siftfast.${MEXEXT}")

  if( MSVC )
    set(MATLAB_MEX_OUT "${CMAKE_CURRENT_BINARY_DIR}/matlab")
  else()
    set(MATLAB_MEX_OUT ${CMAKE_CURRENT_BINARY_DIR})
  endif()

  add_custom_command(
    OUTPUT ${MATLAB_MEX_OUT}/${MATLAB_MEX}
    COMMAND mex
    ARGS -I\"${CMAKE_SOURCE_DIR}\" -L\"${libsiftfast_BINARY_DIR}\" ${MEX_LIBS} ${MEX_CXXFLAGS} -lsiftfast -outdir \"${MATLAB_MEX_OUT}\" -output \"${MATLAB_MEX}\" \"${CMAKE_CURRENT_SOURCE_DIR}/siftmex.cpp\"
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/siftmex.cpp libsiftfast libsiftfast-static
  )
  add_custom_target(siftfast_matlab ALL DEPENDS ${MATLAB_MEX_OUT}/${MATLAB_MEX})
  install(FILES ${MATLAB_MEX_OUT}/${MATLAB_MEX} siftfast.m DESTINATION share/siftfast/matlab)
else()
message(STATUS "MATLAB installation not found")
endif(USE_MATLAB)


# include(${CMAKE_SOURCE_DIR}/FindOctave.cmake)
# if(OCTAVE_FOUND)
#   if( MSVC )
#     # need _mex since mkoctfile generates a siftfast.lib which messes with windows stuff
#     set(OCTAVE_MEX ${CMAKE_CURRENT_BINARY_DIR}/octave/siftfast_mex.mex)
#   else()
#     set(OCTAVE_MEX ${CMAKE_CURRENT_BINARY_DIR}/siftfast.mex)
#   endif()
#
#   add_custom_command(
#     OUTPUT ${OCTAVE_MEX}
#     COMMAND ${MKOCTFILE_EXECUTABLE}
#     ARGS --mex -I${CMAKE_SOURCE_DIR} -L\"${libsiftfast_BINARY_DIR}\" ${MEX_LIBS} ${MEX_CXXFLAGS} -lsiftfast -o \"${OCTAVE_MEX}\" \"${CMAKE_CURRENT_SOURCE_DIR}/siftmex.cpp\"
#     DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/siftmex.cpp libsiftfast libsiftfast-static
#   )
#
#   add_custom_target(siftfast_octave ALL DEPENDS ${OCTAVE_MEX})
#   if( MSVC )
#     add_custom_command(
#       OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/octave/siftfast.mex
#       COMMAND ${CMAKE_COMMAND} -E copy
#       ARGS \"${OCTAVE_MEX}\" \"${CMAKE_CURRENT_BINARY_DIR}/octave/siftfast.mex\"
#       DEPENDS ${OCTAVE_MEX}
#     )
#     add_custom_target(siftfast_octave2 ALL DEPENDS DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/octave/siftfast.mex)
#   endif()
#
#   install(FILES ${OCTAVE_MEX} siftfast.m DESTINATION share/siftfast/octave)
#
# else(OCTAVE_FOUND)
# message(STATUS "Octave installation not found")
# endif(OCTAVE_FOUND)

install(FILES siftfast.h DESTINATION include/siftfast)
install(TARGETS siftfast DESTINATION bin)
if( MSVC )
  install(TARGETS libsiftfast RUNTIME DESTINATION bin LIBRARY DESTINATION bin ARCHIVE DESTINATION lib)
else()
  install(TARGETS libsiftfast DESTINATION lib PERMISSIONS
                  OWNER_READ OWNER_WRITE OWNER_EXECUTE
                  GROUP_READ GROUP_EXECUTE
                  WORLD_READ WORLD_EXECUTE)
endif()

file(GLOB jpg_files ${CMAKE_CURRENT_SOURCE_DIR}/examples/*.jpg)
install(FILES ${jpg_files} DESTINATION share/siftfast/examples)
install(PROGRAMS ${CMAKE_CURRENT_SOURCE_DIR}/examples/testsiftfast.py DESTINATION share/siftfast/examples)

# add make uninstall capability
CONFIGURE_FILE(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
  IMMEDIATE @ONLY)

ADD_CUSTOM_TARGET(uninstall
  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
