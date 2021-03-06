cmake_minimum_required(VERSION 2.8)

project(FlatBuffers)

# NOTE: Code coverage only works on Linux & OSX.
option(FLATBUFFERS_CODE_COVERAGE "Enable the code coverage build option." OFF)
option(FLATBUFFERS_BUILD_TESTS "Enable the build of tests and samples." ON)
option(FLATBUFFERS_INSTALL "Enable the installation of targets." ON)
option(FLATBUFFERS_BUILD_FLATLIB "Enable the build of the flatbuffers library"
       ON)
option(FLATBUFFERS_BUILD_FLATC "Enable the build of the flatbuffers compiler"
       ON)
option(FLATBUFFERS_BUILD_FLATHASH "Enable the build of flathash" ON)
option(FLATBUFFERS_BUILD_GRPCTEST "Enable the build of grpctest" OFF)
option(FLATBUFFERS_BUILD_SHAREDLIB
       "Enable the build of the flatbuffers shared library"
       OFF)

if(NOT FLATBUFFERS_BUILD_FLATC AND FLATBUFFERS_BUILD_TESTS)
    message(WARNING
    "Cannot build tests without building the compiler. Tests will be disabled.")
    set(FLATBUFFERS_BUILD_TESTS OFF)
endif()

set(FlatBuffers_Library_SRCS
  include/flatbuffers/code_generators.h
  include/flatbuffers/base.h
  include/flatbuffers/flatbuffers.h
  include/flatbuffers/hash.h
  include/flatbuffers/idl.h
  include/flatbuffers/util.h
  include/flatbuffers/reflection.h
  include/flatbuffers/reflection_generated.h
  include/flatbuffers/flexbuffers.h
  include/flatbuffers/registry.h
  src/code_generators.cpp
  src/idl_parser.cpp
  src/idl_gen_text.cpp
  src/reflection.cpp
  src/util.cpp
)

set(FlatBuffers_Compiler_SRCS
  ${FlatBuffers_Library_SRCS}
  src/idl_gen_cpp.cpp
  src/idl_gen_general.cpp
  src/idl_gen_go.cpp
  src/idl_gen_js.cpp
  src/idl_gen_php.cpp
  src/idl_gen_python.cpp
  src/idl_gen_fbs.cpp
  src/idl_gen_grpc.cpp
  src/flatc.cpp
  src/flatc_main.cpp
  grpc/src/compiler/schema_interface.h
  grpc/src/compiler/cpp_generator.h
  grpc/src/compiler/cpp_generator.cc
  grpc/src/compiler/go_generator.h
  grpc/src/compiler/go_generator.cc
)

set(FlatHash_SRCS
  include/flatbuffers/hash.h
  src/flathash.cpp
)

set(FlatBuffers_Tests_SRCS
  ${FlatBuffers_Library_SRCS}
  src/idl_gen_fbs.cpp
  tests/test.cpp
  # file generate by running compiler on tests/monster_test.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/monster_test_generated.h
)

set(FlatBuffers_Sample_Binary_SRCS
  include/flatbuffers/flatbuffers.h
  samples/sample_binary.cpp
  # file generated by running compiler on samples/monster.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/samples/monster_generated.h
)

set(FlatBuffers_Sample_Text_SRCS
  ${FlatBuffers_Library_SRCS}
  samples/sample_text.cpp
  # file generated by running compiler on samples/monster.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/samples/monster_generated.h
)

set(FlatBuffers_GRPCTest_SRCS
  include/flatbuffers/flatbuffers.h
  include/flatbuffers/grpc.h
  tests/monster_test.grpc.fb.h
  tests/monster_test.grpc.fb.cc
  grpc/tests/grpctest.cpp
  # file generated by running compiler on samples/monster.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/samples/monster_generated.h
)

if(BIICODE)
  include(biicode/cmake/biicode.cmake)
  return()
endif()

# Interface library that installs headers
add_library(flatbuffers_headers INTERFACE)

target_include_directories(flatbuffers_headers
  INTERFACE
    $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/grpc>
    $<INSTALL_INTERFACE:include>
)

# Helper function to add private and public interface properties
function (configure_cxx_flags_and_headers TARGET_NAME)
  if(EXISTS "${CMAKE_TOOLCHAIN_FILE}")
    # do not apply any global settings if the toolchain
    # is being configured externally
  else()
    set_target_properties(${TARGET_NAME}
        PROPERTIES
          CXX_STANDARD 11
          CXX_STANDARD_REQUIRED YES
          CXX_EXTENSIONS NO
    )
    target_compile_options(${TARGET_NAME}
      PRIVATE
        # Apple
        $<$<PLATFORM_ID:Darwin>:-Wall -pedantic -Werror -Wextra -Wno-unused-parameter>

        # GNU
        $<$<AND:$<CXX_COMPILER_ID:GNU>,$<NOT:$<PLATFORM_ID:Cygwin>>>:-Wall -pedantic -Werror -Wextra -Werror=shadow>
        $<$<AND:$<CXX_COMPILER_ID:GNU>,$<VERSION_GREATER:$<CXX_COMPILER_VERSION>,4.4>>:-Wunused-result -Werror=unused-result -Wunused-parameter -Werror=unused-parameter>
        $<$<CXX_COMPILER_ID:GNU>:-fsigned-char>

        # Clang
        $<$<CXX_COMPILER_ID:Clang>:-Wall -pedantic -Werror -Wextra -Wno-unused-parameter -fsigned-char>

        # MSVC
        $<$<CXX_COMPILER_ID:MSVC>:/W4 /WX /wd4512 /wd4316>
    )
  endif()
  target_link_libraries(${TARGET_NAME}
    PUBLIC
        flatbuffers_headers
  )
endfunction(configure_cxx_flags_and_headers)

if(FLATBUFFERS_BUILD_FLATLIB)
  add_library(flatbuffers STATIC ${FlatBuffers_Library_SRCS})
  configure_cxx_flags_and_headers(flatbuffers)
endif()

# Helper function to add code coverage flags
function(configure_executables TARGET_NAME)
  configure_cxx_flags_and_headers(${TARGET_NAME})
  target_compile_options(flatc
    PRIVATE
      $<$<BOOL:${FLATBUFFERS_CODE_COVERAGE}>:-g -fprofile-arcs -ftest-coverage>
      $<$<CXX_COMPILER_ID:MSVC>:/MT>
  )
endfunction(configure_executables)

if(FLATBUFFERS_BUILD_FLATC)
  add_executable(flatc ${FlatBuffers_Compiler_SRCS})
  configure_executables(flatc)

  if(NOT FLATBUFFERS_FLATC_EXECUTABLE)
    set(FLATBUFFERS_FLATC_EXECUTABLE $<TARGET_FILE:flatc>)
  endif()
endif()

if(FLATBUFFERS_BUILD_FLATHASH)
  add_executable(flathash ${FlatHash_SRCS})
  configure_executables(flathash)
endif()

if(FLATBUFFERS_BUILD_SHAREDLIB)
  add_library(flatbuffers_shared SHARED ${FlatBuffers_Library_SRCS})
  set_target_properties(flatbuffers_shared PROPERTIES OUTPUT_NAME flatbuffers)
  configure_cxx_flags_and_headers(flatbuffers_shared)
endif()

function(compile_flatbuffers_schema_to_cpp SRC_FBS)
  get_filename_component(SRC_FBS_DIR ${SRC_FBS} PATH)
  string(REGEX REPLACE "\\.fbs$" "_generated.h" GEN_HEADER ${SRC_FBS})
  add_custom_command(
    OUTPUT ${GEN_HEADER}
    COMMAND "${FLATBUFFERS_FLATC_EXECUTABLE}" -c --no-includes --gen-mutable
            --gen-object-api -o "${SRC_FBS_DIR}"
            -I "${CMAKE_CURRENT_SOURCE_DIR}/tests/include_test"
            "${CMAKE_CURRENT_SOURCE_DIR}/${SRC_FBS}"
    DEPENDS flatc)
endfunction()

function(compile_flatbuffers_schema_to_binary SRC_FBS)
  get_filename_component(SRC_FBS_DIR ${SRC_FBS} PATH)
  string(REGEX REPLACE "\\.fbs$" ".bfbs" GEN_BINARY_SCHEMA ${SRC_FBS})
  add_custom_command(
    OUTPUT ${GEN_BINARY_SCHEMA}
    COMMAND "${FLATBUFFERS_FLATC_EXECUTABLE}" -b --schema -o "${SRC_FBS_DIR}"
            "${CMAKE_CURRENT_SOURCE_DIR}/${SRC_FBS}"
    DEPENDS flatc)
endfunction()

if(FLATBUFFERS_BUILD_TESTS)
  enable_testing()

  include(CMakeParseArguments)
  function(flatbuffers_add_test)
    cmake_parse_arguments(
      PARSED_ARGS
      ""
      "TARGET;SCHEMA"
      "SOURCES"
      ${ARGN}
    )
    compile_flatbuffers_schema_to_cpp(${PARSED_ARGS_SCHEMA})
    add_executable(${PARSED_ARGS_TARGET} ${PARSED_ARGS_SOURCES})
    configure_executables(${PARSED_ARGS_TARGET})
    set_property(TARGET ${PARSED_ARGS_TARGET}
      PROPERTY
        COMPILE_DEFINITIONS FLATBUFFERS_TRACK_VERIFIER_BUFFER_SIZE FLATBUFFERS_DEBUG_VERIFICATION_FAILURE=1
    )
  endfunction()

  flatbuffers_add_test(
    TARGET flattests
    SCHEMA tests/monster_test.fbs
    SOURCES ${FlatBuffers_Tests_SRCS}
  )

  file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/tests" DESTINATION
       "${CMAKE_CURRENT_BINARY_DIR}")
  add_test(NAME flattests COMMAND flattests)

  flatbuffers_add_test(
    TARGET flatsamplebinary
    SCHEMA samples/monster.fbs
    SOURCES ${FlatBuffers_Sample_Binary_SRCS}
  )

  add_executable(flatsampletext ${FlatBuffers_Sample_Text_SRCS})
  configure_executables(flatsampletext)
endif()

if(FLATBUFFERS_BUILD_GRPCTEST)
  add_executable(grpctest ${FlatBuffers_GRPCTest_SRCS})
  target_compile_options(grpctest PRIVATE -Wno-unused-parameter -Wno-shadow)
  target_link_libraries(grpctest grpc++_unsecure pthread dl)
  configure_executables(grpctest)
endif()

if(FLATBUFFERS_INSTALL)
  set(FLATBUFFERS_TARGETS_TO_INSTALL "flatbuffers_headers")
  if (FLATBUFFERS_BUILD_FLATLIB)
      list(APPEND FLATBUFFERS_TARGETS_TO_INSTALL "flatbuffers")
  endif()
  if (FLATBUFFERS_BUILD_FLATC)
      list(APPEND FLATBUFFERS_TARGETS_TO_INSTALL "flatc")
  endif()
  if (FLATBUFFERS_BUILD_SHAREDLIB)
      list(APPEND FLATBUFFERS_TARGETS_TO_INSTALL "flatbuffers_shared")
  endif()

  install(
    TARGETS ${FLATBUFFERS_TARGETS_TO_INSTALL} EXPORT "${PROJECT_NAME}Targets"
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
  )

  install(DIRECTORY include/flatbuffers DESTINATION include)

  install(EXPORT "${PROJECT_NAME}Targets"
      FILE "${PROJECT_NAME}Targets.cmake"
      NAMESPACE "${PROJECT_NAME}::"
      DESTINATION lib/cmake/flatbuffers
  )

  include(CMakePackageConfigHelpers)

  configure_package_config_file(
    "cmake/Config.cmake.in" "${PROJECT_NAME}Config.cmake"
    INSTALL_DESTINATION lib/cmake/flatbuffers
  )

  EXECUTE_PROCESS(
    COMMAND git describe
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  string(REGEX REPLACE "^v([0-9\\.]+)-.*" "\\1" VERSION "${GIT_VERSION}")
  
  write_basic_package_version_file("${PROJECT_NAME}ConfigVersion.cmake"
      VERSION "${VERSION}"
      COMPATIBILITY AnyNewerVersion
  )

  install(FILES
      "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
      "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    DESTINATION lib/cmake/flatbuffers
  )
endif()

include(CMake/BuildFlatBuffers.cmake)

if(FLATBUFFERS_PACKAGE_DEBIAN)
    include(CMake/PackageDebian.cmake)
endif()
