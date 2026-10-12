cmake_minimum_required(VERSION 3.7)

project(libxslt_vendor)

find_package(ament_cmake REQUIRED)

# libxslt is the GNOME XSLT library; it depends on libxml2. lyrical targets
# Ubuntu 26.04 (resolute), whose libxslt1-dev is 1.1.45 — pin the upstream
# GNOME tag to match. libxml2 must be built first (see package.xml depend).

option(FORCE_BUILD_VENDOR_PKG
  "Build libxslt from source, even if a system-installed package is available"
  OFF)

if(NOT FORCE_BUILD_VENDOR_PKG)
  find_package(LibXslt QUIET)
endif()

macro(build_libxslt)
  set(extra_cmake_args)
  if(DEFINED CMAKE_BUILD_TYPE)
    list(APPEND extra_cmake_args -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})
  endif()
  if(DEFINED BUILD_SHARED_LIBS)
    list(APPEND extra_cmake_args -DBUILD_SHARED_LIBS=${BUILD_SHARED_LIBS})
  else()
    list(APPEND extra_cmake_args -DBUILD_SHARED_LIBS=ON)
  endif()
  if(DEFINED CMAKE_TOOLCHAIN_FILE)
    list(APPEND extra_cmake_args "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}")
  endif()
  list(APPEND extra_cmake_args "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}")
  # libxslt's inner CMake does find_package(LibXml2 CONFIG). The package.xml
  # <depend>libxml2_vendor</depend> makes colcon put libxml2's install root on
  # CMAKE_PREFIX_PATH — but the VxWorks toolchain sets
  # CMAKE_FIND_ROOT_PATH_MODE_PACKAGE=ONLY, so find_package searches ONLY
  # CMAKE_FIND_ROOT_PATH and ignores CMAKE_PREFIX_PATH / *_ROOT hints. Hence the
  # dep's install root (the outer CMAKE_PREFIX_PATH, == export/root) must be
  # forwarded into the inner build's find-root.
  list(APPEND extra_cmake_args
    "-DCMAKE_FIND_ROOT_PATH=${CMAKE_FIND_ROOT_PATH}"
    -DLIBXSLT_WITH_PYTHON=OFF
    -DLIBXSLT_WITH_CRYPTO=OFF
    -DLIBXSLT_WITH_TESTS=OFF
    -DLIBXSLT_WITH_PROGRAMS=OFF)

  include(ExternalProject)

  # VxWorks source patch (carried from pkg/libxslt/files/0001-add-VxWorks-support.patch).
  externalproject_add(libxslt_external
    GIT_REPOSITORY https://github.com/GNOME/libxslt.git
    GIT_TAG v1.1.45
    GIT_CONFIG advice.detachedHead=false
    UPDATE_COMMAND ""
    TIMEOUT 600
    PATCH_COMMAND
      patch -p1 -N -r - < ${CMAKE_CURRENT_SOURCE_DIR}/patches/0001-add-VxWorks-support.patch || true
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_install
      ${extra_cmake_args}
  )

  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_install/
    DESTINATION
      ${CMAKE_INSTALL_PREFIX}
    USE_SOURCE_PERMISSIONS
  )
endmacro()

if(NOT LibXslt_FOUND)
  message(STATUS "Did not find libxslt installation, building from source")
  build_libxslt()
endif()

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
