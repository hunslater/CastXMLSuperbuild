cmake_minimum_required(VERSION 2.8)

project(CastXMLSuperbuild NONE)
# Set a default build type if none was specified
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()
enable_language(C)
enable_language(CXX)


# Use a static runtime with MSVC
if(MSVC)
  foreach(flag
    CMAKE_CXX_FLAGS
    CMAKE_CXX_FLAGS_RELEASE
    CMAKE_C_FLAGS
    CMAKE_C_FLAGS_RELEASE
    )
    if(${flag} MATCHES "/MD")
      string(REGEX REPLACE "/MD" "/MT" ${flag} "${${flag}}")
    endif()
  endforeach()
endif()


if(APPLE)
  set(osx_args
    -DCMAKE_OSX_DEPLOYMENT_TARGET:STRING=10.9
    -DCMAKE_OSX_ARCHITECTURES:STRING=x86_64
    )
endif()


include(ExternalProject)

find_program(NINJA_EXECUTABLE ninja)
if(NINJA_EXECUTABLE)
  set(CMAKE_GENERATOR "Ninja")
  # Make Ninja verbose so CircleCI does not think the build has halted.
  set(verbose_command BUILD_COMMAND ${NINJA_EXECUTABLE} ${BUILD_FLAGS} -v)
endif()

option(USE_SYSTEM_LLVM "Use the system LLVM instead of building it." OFF)
if(USE_SYSTEM_LLVM)
  find_package(LLVM REQUIRED NO_MODULE)
  set(castxml_deps)
else()

  set(llvm_version 5.0.0)
  set(llvm_md5 e3782305ff4a60ad9bbc3a926425786f)
  ExternalProject_Add(llvm
    # This is the upstream source code repackaged in a .tar.gz for
    # compatibility with older CMake. Also the tests/ directory
    # removed to save space.
    URL https://midas3.kitware.com/midas/api/rest?method=midas.bitstream.download&checksum=${llvm_md5}&name=llvm-${llvm_version}.src.tar.gz
    URL_MD5 ${llvm_md5}
    CMAKE_ARGS -Wno-dev
    CMAKE_GENERATOR "${CMAKE_GENERATOR}"
    CMAKE_CACHE_ARGS
      -DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}
      "-DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS} -w"
      "-DCMAKE_CXX_FLAGS_RELEASE:STRING=${CMAKE_CXX_FLAGS_RELEASE}"
      -DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}
      "-DCMAKE_C_FLAGS:STRING=${CMAKE_C_FLAGS} -w"
      "-DCMAKE_C_FLAGS_RELEASE:STRING=${CMAKE_C_FLAGS_RELEASE}"
      -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
      -DLLVM_USE_CRT_RELEASE:STRING=MT
      -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
      -DLLVM_ENABLE_TERMINFO:BOOL=OFF
      -DLLVM_INCLUDE_TESTS:BOOL=OFF
      -DLLVM_INCLUDE_EXAMPLES:BOOL=OFF
      -DLLVM_INCLUDE_DOCS:BOOL=OFF
      ${osx_args}
    ${verbose_command}
    LOG_BUILD 0
    INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/llvm
    )


  set(clang_md5 79eddb9b1f6e6ee8e9eeff1245d3f359)
  ExternalProject_Add(clang
    # This is the upstream source code repackaged in a .tar.gz for
    # compatibility with older CMake. Also the tests/ directory
    # removed to save space.
    URL https://midas3.kitware.com/midas/api/rest?method=midas.bitstream.download&checksum=${clang_md5}&name=cfe-${llvm_version}.src.tar.gz
    URL_MD5 ${clang_md5}
    DEPENDS llvm
    SOURCE_DIR ${CMAKE_CURRENT_BINARY_DIR}/cfe-${llvm_version}
    CMAKE_ARGS -Wno-dev
    CMAKE_GENERATOR "${CMAKE_GENERATOR}"
    CMAKE_CACHE_ARGS
      -DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}
      "-DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS} -w"
      "-DCMAKE_CXX_FLAGS_RELEASE:STRING=${CMAKE_CXX_FLAGS_RELEASE}"
      -DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}
      "-DCMAKE_C_FLAGS:STRING=${CMAKE_C_FLAGS} -w"
      "-DCMAKE_C_FLAGS_RELEASE:STRING=${CMAKE_C_FLAGS_RELEASE}"
      -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
      -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
      -DCLANG_INCLUDE_DOCS:BOOL=OFF
      -DCLANG_INCLUDE_TESTS:BOOL=OFF
      -DLLVM_CONFIG:PATH=${CMAKE_CURRENT_BINARY_DIR}/llvm/bin/llvm-config
      ${osx_args}
    ${verbose_command}
    LOG_BUILD 0
    INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/llvm
    )

  set(castxml_deps llvm clang)
  set(LLVM_DIR ${CMAKE_CURRENT_BINARY_DIR}/llvm/lib/cmake/llvm/)
endif()


# Use a static C++ library on GCC/Linux
if(CMAKE_COMPILER_IS_GNUCXX AND CMAKE_SYSTEM_NAME MATCHES "Linux")
  set(extra_flags "-static-libstdc++")
endif()
# 2017-11-06
set(CastXML_GIT_TAG 3a81f3cc3060be86064ba2e666d30b0c358cc847 CACHE STRING "CastXML Git revision.")
ExternalProject_Add(castxml
  GIT_REPOSITORY https://github.com/CastXML/CastXML.git
  GIT_TAG ${CastXML_GIT_TAG}
  DEPENDS ${castxml_deps}
  CMAKE_ARGS -Wno-dev
  CMAKE_GENERATOR "${CMAKE_GENERATOR}"
  CMAKE_CACHE_ARGS
    -DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}
    "-DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS} -w ${extra_flags}"
    "-DCMAKE_CXX_FLAGS_RELEASE:STRING=${CMAKE_CXX_FLAGS_RELEASE}"
    -DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}
    "-DCMAKE_C_FLAGS:STRING=${CMAKE_C_FLAGS} -w"
    "-DCMAKE_C_FLAGS_RELEASE:STRING=${CMAKE_C_FLAGS_RELEASE}"
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
    -DLLVM_DIR:PATH=${LLVM_DIR}
    ${osx_args}
  ${verbose_command}
  LOG_BUILD 0
  INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/castxml
  )
