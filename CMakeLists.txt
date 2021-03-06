cmake_minimum_required(VERSION 3.5)
cmake_policy(SET CMP0048 NEW)
project (Bloaty VERSION 1.1)
include(CTest)
set(CMAKE_CXX_STANDARD 17)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)  # Group projects in visual studio

# Options we define for users.
option(BLOATY_ENABLE_ASAN "Enable address sanitizer." OFF)
option(BLOATY_ENABLE_UBSAN "Enable undefined behavior sanitizer." OFF)
option(BLOATY_ENABLE_CMAKETARGETS "Enable installing cmake target files." ON)
option(BLOATY_ENABLE_BUILDID "Enable build id." ON)
option(BLOATY_ENABLE_RE2 "Enable the support for regular expression functions." ON)

if(UNIX)
find_package(PkgConfig)
if(${PKG_CONFIG_FOUND})
if(BLOATY_ENABLE_RE2)
  pkg_search_module(RE2 re2)
endif(BLOATY_ENABLE_RE2)
pkg_search_module(CAPSTONE capstone)
pkg_search_module(PROTOBUF protobuf)
pkg_search_module(ZLIB zlib)
if(BLOATY_ENABLE_RE2)
  if(${RE2_FOUND})
    MESSAGE(STATUS "System re2 found, using")
  else(${RE2_FOUND})
    MESSAGE(STATUS "System re2 not found, using bundled version")
  endif(${RE2_FOUND})
endif(BLOATY_ENABLE_RE2)
if(${CAPSTONE_FOUND})
  MESSAGE(STATUS "System capstone found, using")
else(${CAPSTONE_FOUND})
  MESSAGE(STATUS "System capstone not found, using bundled version")
endif(${CAPSTONE_FOUND})
if(${PROTOBUF_FOUND})
  MESSAGE(STATUS "System protobuf found, using")
else(${PROTOBUF_FOUND})
  MESSAGE(STATUS "System protobuf not found, using bundled version")
endif(${PROTOBUF_FOUND})
if (${ZLIB_FOUND})
  MESSAGE(STATUS "System zlib found, using")
else(${ZLIB_FOUND})
  MESSAGE(STATUS "System zlib not found, using bundled version")
endif(${ZLIB_FOUND})
else(${PKG_CONFIG_FOUND})
  MESSAGE(STATUS "pkg-config not found, using bundled dependencies")
endif(${PKG_CONFIG_FOUND})
endif(UNIX)

# Set default build type.
if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "Setting build type to 'RelWithDebInfo' as none was specified.")
  set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING
      "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel."
      FORCE)
endif()

# Check out Git submodules.
if (EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.gitmodules")
  execute_process (COMMAND git submodule update --init --recursive
                   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
endif()

# Add third_party libraries, disabling as much as we can of their builds.

add_definitions(-D_LIBCXXABI_FUNC_VIS=)  # For Demumble.
if(BLOATY_ENABLE_RE2)
  add_definitions(-DUSE_RE2)
endif(BLOATY_ENABLE_RE2)

# Set MSVC runtime before including thirdparty libraries
if(MSVC)
  # Link also the runtime library statically so that MSVCR*.DLL is not required at runtime.
  # https://msdn.microsoft.com/en-us/library/2kzt1wy3.aspx
  # This is achieved by replacing msvc option /MD with /MT and /MDd with /MTd
  # http://www.cmake.org/Wiki/CMake_FAQ#How_can_I_build_my_MSVC_application_with_a_static_runtime.3F
  foreach(flag_var
      CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
      CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
    if(${flag_var} MATCHES "/MD")
      string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
    endif(${flag_var} MATCHES "/MD")
  endforeach(flag_var)
endif()

if(UNIX)
  if(BLOATY_ENABLE_RE2)
    if(${RE2_FOUND})
      include_directories(${RE2_INCLUDE_DIRS})
    else(${RE2_FOUND})
      set(RE2_BUILD_TESTING OFF CACHE BOOL "enable testing for RE2" FORCE)
      add_subdirectory(third_party/re2)
      include_directories(third_party/re2)
    endif(${RE2_FOUND})
  endif(BLOATY_ENABLE_RE2)
  if(${CAPSTONE_FOUND})
    include_directories(${CAPSTONE_INCLUDE_DIRS})
  else(${CAPSTONE_FOUND})
    set(CAPSTONE_BUILD_SHARED OFF CACHE BOOL "Build shared library" FORCE)
    set(CAPSTONE_BUILD_TESTS OFF CACHE BOOL "Build tests" FORCE)
    add_subdirectory(third_party/capstone)
    include_directories(third_party/capstone/include)
  endif(${CAPSTONE_FOUND})
  if(${PROTOBUF_FOUND})
    include_directories(${PROTOBUF_INCLUDE_DIRS})
  else(${PROTOBUF_FOUND})
    set(protobuf_BUILD_TESTS OFF CACHE BOOL "enable tests for proto2" FORCE)
    set(protobuf_BUILD_SHARED_LIBS OFF CACHE BOOL "enable shared libs for proto2" FORCE)
    add_subdirectory(third_party/protobuf/cmake)
    include_directories(SYSTEM third_party/protobuf/src)
  endif(${PROTOBUF_FOUND})
  if(${ZLIB_FOUND})
    include_directories(${ZLIB_INCLUDE_DIRS})
  else(${ZLIB_FOUND})
    add_subdirectory(third_party/zlib)
    include_directories(third_party/zlib)
  endif(${ZLIB_FOUND})
else(UNIX)
  if(BLOATY_ENABLE_RE2)
    set(RE2_BUILD_TESTING OFF CACHE BOOL "enable testing for RE2" FORCE)
    add_subdirectory(third_party/re2)
    include_directories(third_party/re2)
    set_property(TARGET re2 PROPERTY FOLDER "third_party")
  endif(BLOATY_ENABLE_RE2)

  set(CAPSTONE_BUILD_SHARED OFF CACHE BOOL "Build shared library" FORCE)
  set(CAPSTONE_BUILD_TESTS OFF CACHE BOOL "Build tests" FORCE)
  add_subdirectory(third_party/capstone)
  include_directories(third_party/capstone/include)
  set_property(TARGET capstone-static PROPERTY FOLDER "third_party")

  set(protobuf_BUILD_TESTS OFF CACHE BOOL "enable tests for proto2" FORCE)
  set(protobuf_BUILD_SHARED_LIBS OFF CACHE BOOL "enable shared libs for proto2" FORCE)
  add_subdirectory(third_party/protobuf/cmake)
  include_directories(SYSTEM third_party/protobuf/src)

  add_subdirectory(third_party/zlib)
  include_directories(third_party/zlib)
  include_directories(${CMAKE_CURRENT_BINARY_DIR}/third_party/zlib)
  set_property(TARGET example PROPERTY FOLDER "third_party")
  set_property(TARGET minigzip PROPERTY FOLDER "third_party")
  set_property(TARGET zlib PROPERTY FOLDER "third_party")
  set_property(TARGET zlibstatic PROPERTY FOLDER "third_party")
  set_property(TARGET libprotobuf PROPERTY FOLDER "third_party")
  set_property(TARGET libprotobuf-lite PROPERTY FOLDER "third_party")
  set_property(TARGET libprotoc PROPERTY FOLDER "third_party")
  set_property(TARGET protoc PROPERTY FOLDER "third_party")
endif(UNIX)

include_directories(.)
include_directories(src)
include_directories(third_party/abseil-cpp)
include_directories("${CMAKE_CURRENT_BINARY_DIR}/src")

# Baseline build flags.
if(MSVC)
  set(CMAKE_CXX_FLAGS "/EHsc /wd4018 /D_CRT_SECURE_NO_WARNINGS /DNOMINMAX")
else(MSVC)
  set(CMAKE_CXX_FLAGS "-std=c++11 -W -Wall -Wno-sign-compare")
  set(CMAKE_CXX_FLAGS_DEBUG "-g1")
  set(CMAKE_CXX_FLAGS_RELEASE "-O2")
  set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g1")
  set_source_files_properties(third_party/demumble/third_party/libcxxabi/cxa_demangle.cpp PROPERTIES COMPILE_FLAGS -Wno-implicit-fallthrough)
endif()

if(APPLE)
elseif(UNIX)
  if(BLOATY_ENABLE_BUILDID)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--build-id")
  endif(BLOATY_ENABLE_BUILDID)
endif()

# When using Ninja, compiler output won't be colorized without this.
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG(-fdiagnostics-color=always SUPPORTS_COLOR_ALWAYS)
if(SUPPORTS_COLOR_ALWAYS)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdiagnostics-color=always")
endif()

# Implement ASAN/UBSAN options
if(BLOATY_ENABLE_ASAN)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
  set(CMAKE_LINKER_FLAGS_DEBUG "${CMAKE_LINKER_FLAGS_DEBUG} -fsanitize=address")
endif()

if(BLOATY_ENABLE_UBSAN)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined")
  set(CMAKE_LINKER_FLAGS_DEBUG "${CMAKE_LINKER_FLAGS_DEBUG} -fsanitize=undefined")
endif()

if(DEFINED ENV{CXXFLAGS})
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} $ENV{CXXFLAGS}")
endif()

file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/src)
if(${PROTOC_FOUND})
add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/src/bloaty.pb.cc
  DEPENDS protoc ${CMAKE_CURRENT_SOURCE_DIR}/src/bloaty.proto
  COMMAND protoc ${CMAKE_CURRENT_SOURCE_DIR}/src/bloaty.proto
      --cpp_out=${CMAKE_CURRENT_BINARY_DIR}/src
      -I${CMAKE_CURRENT_SOURCE_DIR}/src
)
else(${PROTOC_FOUND})
add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/src/bloaty.pb.cc
  COMMAND protoc ${CMAKE_CURRENT_SOURCE_DIR}/src/bloaty.proto
      --cpp_out=${CMAKE_CURRENT_BINARY_DIR}/src
      -I${CMAKE_CURRENT_SOURCE_DIR}/src
)
endif(${PROTOC_FOUND})

file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/src/bloaty_package.bloaty
     DESTINATION ${CMAKE_CURRENT_BINARY_DIR})

add_library(libbloaty STATIC
    src/bloaty.cc
    src/bloaty.h
    src/disassemble.cc
    ${CMAKE_CURRENT_BINARY_DIR}/src/bloaty.pb.cc
    src/dwarf.cc
    src/dwarf_constants.h
    src/elf.cc
    src/macho.cc
    src/pe.cc
    third_party/lief_pe/pe_structures.h
    src/range_map.cc
    src/range_map.h
    src/re.h
    src/util.cc
    src/util.h
    src/webassembly.cc
    # Until Abseil has a proper CMake build system
    third_party/abseil-cpp/absl/base/internal/raw_logging.cc # Grrrr...
    third_party/abseil-cpp/absl/base/internal/throw_delegate.cc
    third_party/abseil-cpp/absl/debugging/internal/demangle.cc
    third_party/abseil-cpp/absl/numeric/int128.cc
    third_party/abseil-cpp/absl/strings/ascii.cc
    third_party/abseil-cpp/absl/strings/charconv.cc
    third_party/abseil-cpp/absl/strings/escaping.cc
    third_party/abseil-cpp/absl/strings/internal/charconv_bigint.cc
    third_party/abseil-cpp/absl/strings/internal/charconv_parse.cc
    third_party/abseil-cpp/absl/strings/internal/escaping.cc
    third_party/abseil-cpp/absl/strings/internal/memutil.cc
    third_party/abseil-cpp/absl/strings/internal/utf8.cc
    third_party/abseil-cpp/absl/strings/match.cc
    third_party/abseil-cpp/absl/strings/numbers.cc
    third_party/abseil-cpp/absl/strings/str_cat.cc
    third_party/abseil-cpp/absl/strings/string_view.cc
    third_party/abseil-cpp/absl/strings/str_split.cc
    third_party/abseil-cpp/absl/strings/substitute.cc
    third_party/abseil-cpp/absl/types/bad_optional_access.cc
    # One source file, no special build system needed.
    third_party/demumble/third_party/libcxxabi/cxa_demangle.cpp
    )
set_property(TARGET libbloaty PROPERTY FOLDER "bloaty")

if(UNIX)
  set(LIBBLOATY_LIBS libbloaty)
  if(${PROTOBUF_FOUND})
    set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} ${PROTOBUF_LIBRARIES})
  else(${PROTOBUF_FOUND})
    set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} libprotoc)
  endif(${PROTOBUF_FOUND})
  if(BLOATY_ENABLE_RE2)
    if(${RE2_FOUND})
      set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} ${RE2_LIBRARIES})
    else(${RE2_FOUND})
      set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} re2)
    endif(${RE2_FOUND})
  endif(BLOATY_ENABLE_RE2)
  if(${CAPSTONE_FOUND})
    set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} ${CAPSTONE_LIBRARIES})
  else(${CAPSTONE_FOUND})
    set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} capstone-static)
  endif(${CAPSTONE_FOUND})
  if(${ZLIB_FOUND})
    set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} ${ZLIB_LIBRARIES})
  else(${ZLIB_FOUND})
    set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} zlib)
  endif(${ZLIB_FOUND})
else(UNIX)
  set(LIBBLOATY_LIBS libbloaty libprotoc capstone-static)
  if(BLOATY_ENABLE_RE2)
    set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} re2)
  endif(BLOATY_ENABLE_RE2)
  set(LIBBLOATY_LIBS ${LIBBLOATY_LIBS} zlibstatic)
endif(UNIX)

if(UNIX)
  if(BLOATY_ENABLE_RE2)
    if(${RE2_FOUND})
      link_directories(${RE2_LIBRARY_DIRS})
    endif(${RE2_FOUND})
  endif(BLOATY_ENABLE_RE2)
  if(${CAPSTONE_FOUND})
    link_directories(${CAPSTONE_LIBRARY_DIRS})
  endif(${CAPSTONE_FOUND})
  if(${PROTOBUF_FOUND})
    link_directories(${PROTOBUF_LIBRARY_DIRS})
  endif(${PROTOBUF_FOUND})
  if(${ZLIB_FOUND})
    link_directories(${ZLIB_LIBRARY_DIRS})
  endif(${ZLIB_FOUND})
endif(UNIX)

if(DEFINED ENV{LIB_FUZZING_ENGINE})
  message("LIB_FUZZING_ENGINE set, building fuzz_target instead of Bloaty")
  add_executable(fuzz_target tests/fuzz_target.cc)
  target_link_libraries(fuzz_target "${LIBBLOATY_LIBS}" "${CMAKE_THREAD_LIBS_INIT}" $ENV{LIB_FUZZING_ENGINE})
else()
  add_executable(bloaty src/main.cc)
  target_link_libraries(bloaty "${LIBBLOATY_LIBS}")

  set_property(TARGET bloaty PROPERTY FOLDER "bloaty")

  # All of this is to add -pthread, which is required by re2 (not us).
  find_package(Threads REQUIRED)
  if(THREADS_HAVE_PTHREAD_ARG)
    set_property(TARGET bloaty PROPERTY COMPILE_OPTIONS "-pthread")
    set_property(TARGET bloaty PROPERTY INTERFACE_COMPILE_OPTIONS "-pthread")
  endif()
  if(CMAKE_THREAD_LIBS_INIT)
    target_link_libraries(bloaty "${CMAKE_THREAD_LIBS_INIT}")
  endif()

  if(BLOATY_ENABLE_CMAKETARGETS)
    install(
      TARGETS bloaty
      EXPORT ${PROJECT_NAME}Targets
      RUNTIME DESTINATION bin
    )
  else(BLOATY_ENABLE_CMAKETARGETS)
    install(
      TARGETS bloaty
      RUNTIME DESTINATION bin
    )
  endif(BLOATY_ENABLE_CMAKETARGETS)

  if (IS_DIRECTORY "${PROJECT_SOURCE_DIR}/tests")
    enable_testing()

    if(BUILD_TESTING)
      option(INSTALL_GTEST "" OFF)
      add_subdirectory(third_party/googletest)
      include_directories(third_party/googletest/googletest/include)
      include_directories(third_party/googletest/googlemock/include)

      set(TEST_TARGETS
          bloaty_test
          bloaty_test_pe
          bloaty_misc_test
          range_map_test
          )

      foreach(target ${TEST_TARGETS})
        add_executable(${target} tests/${target}.cc)
        target_link_libraries(${target} "${LIBBLOATY_LIBS}" gtest_main gmock "${CMAKE_THREAD_LIBS_INIT}")
        set_property(TARGET ${target} PROPERTY FOLDER "tests")
      endforeach(target)

      add_executable(fuzz_test tests/fuzz_target.cc tests/fuzz_driver.cc)
      target_link_libraries(fuzz_test "${LIBBLOATY_LIBS}" "${CMAKE_THREAD_LIBS_INIT}")
      set_property(TARGET fuzz_test PROPERTY FOLDER "tests")

      foreach(testlib gmock gmock_main gtest gtest_main)
        set_property(TARGET ${testlib} PROPERTY FOLDER "tests/libs")
      endforeach(testlib)

      file(GLOB fuzz_corpus tests/testdata/fuzz_corpus/*)

      add_test(NAME range_map_test COMMAND range_map_test)
      add_test(NAME bloaty_test_x86-64 COMMAND bloaty_test WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests/testdata/linux-x86_64)
      add_test(NAME bloaty_test_x86 COMMAND bloaty_test WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests/testdata/linux-x86)
      add_test(NAME bloaty_test_pe_x64 COMMAND bloaty_test_pe WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests/testdata/PE/x64)
      add_test(NAME bloaty_test_pe_x86 COMMAND bloaty_test_pe WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests/testdata/PE/x86)
      add_test(NAME bloaty_misc_test COMMAND bloaty_misc_test WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests/testdata/misc)
      add_test(NAME fuzz_test COMMAND fuzz_test ${fuzz_corpus} WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests/testdata/fuzz_corpus)
    endif()
  endif()

  if(BLOATY_ENABLE_CMAKETARGETS)
    install(EXPORT ${PROJECT_NAME}Targets NAMESPACE ${PROJECT_NAME} DESTINATION lib/${PROJECT_NAME})
  endif(BLOATY_ENABLE_CMAKETARGETS)
endif()
