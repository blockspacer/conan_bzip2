cmake_minimum_required(VERSION 3.4)

project(bzip2)

include(GNUInstallDirs)

option(COMPILE_WITH_LLVM_TOOLS
  "Enable clang from llvm_tools (conan package)" OFF)

# see https://github.com/Ericsson/codechecker/blob/master/tools/report-converter/README.md#undefined-behaviour-sanitizer
# NOTE: Compile with -g and -fno-omit-frame-pointer
# to get proper debug information in your binary.
# NOTE: Run your program with environment variable UBSAN_OPTIONS=print_stacktrace=1.
# see https://github.com/google/sanitizers/wiki/SanitizerCommonFlags
option(ENABLE_UBSAN
  "Enable Undefined Behaviour Sanitizer" OFF)

# see https://github.com/google/sanitizers/wiki/AddressSanitizerLeakSanitizer
# see https://github.com/Ericsson/codechecker/blob/master/tools/report-converter/README.md#address-sanitizer
# NOTE: Compile with -g and -fno-omit-frame-pointer
# to get proper debug information in your binary.
# NOTE: use ASAN_OPTIONS=detect_leaks=1 LSAN_OPTIONS=suppressions=suppr.txt
# NOTE: You need the ASAN_OPTIONS=symbolize=1
# to turn on resolving addresses in object code
# to source code line numbers and filenames.
# This option is implicit for Clang but it won't do any harm.
# see https://github.com/google/sanitizers/wiki/SanitizerCommonFlags
option(ENABLE_ASAN
  "Enable Address Sanitizer" OFF)

# see https://github.com/Ericsson/codechecker/blob/master/tools/report-converter/README.md#memory-sanitizer
# NOTE: Compile with -g and -fno-omit-frame-pointer
# to get proper debug information in your binary.
option(ENABLE_MSAN
  "Enable Memory Sanitizer" OFF)

# see https://github.com/Ericsson/codechecker/blob/master/tools/report-converter/README.md#thread-sanitizer
# NOTE: Compile with -g
# to get proper debug information in your binary.
option(ENABLE_TSAN
  "Enable Thread Sanitizer" OFF)

if(COMPILE_WITH_LLVM_TOOLS)
  message(STATUS
    "Using COMPILE_WITH_LLVM_TOOLS")
  # do not check compile in conanbuildinfo
  # cause we will switch to other compiler after conan install
  set(CONAN_DISABLE_CHECK_COMPILER ON)
else(COMPILE_WITH_LLVM_TOOLS)
  message(STATUS
    "Disabled COMPILE_WITH_LLVM_TOOLS")
endif(COMPILE_WITH_LLVM_TOOLS)

include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
conan_basic_setup()

if(ENABLE_MSAN OR ENABLE_TSAN OR ENABLE_ASAN OR ENABLE_UBSAN)
  find_package(cmake_sanitizers REQUIRED)
endif()

find_package(cmake_helper_utils REQUIRED)

# prefer ASCII for folder names
force_latin_paths() # from cmake_helper_utils (conan package)

# out dirs (CMAKE_*_OUTPUT_DIRECTORY) must be not empty
validate_out_dirs() # from cmake_helper_utils (conan package)

# In-source builds not allowed
validate_out_source_build(WARNING) # from cmake_helper_utils (conan package)

if(ENABLE_MSAN)
  add_msan_flags()
endif(ENABLE_MSAN)

if(ENABLE_TSAN)
  add_tsan_flags()
endif(ENABLE_TSAN)

if(ENABLE_ASAN)
  add_asan_flags()
endif(ENABLE_ASAN)

if(ENABLE_UBSAN)
  add_ubsan_flags()
endif(ENABLE_UBSAN)

if(COMPILE_WITH_LLVM_TOOLS)
  message(STATUS
    "Using COMPILE_WITH_LLVM_TOOLS")
  # force change CMAKE_*_COMPILER and CMAKE_LINKER to clang from conan
  compile_with_llvm_tools() # from cmake_helper_utils (conan package)
else(COMPILE_WITH_LLVM_TOOLS)
  message(STATUS
    "Disabled COMPILE_WITH_LLVM_TOOLS")
endif(COMPILE_WITH_LLVM_TOOLS)

if(MSVC OR MSVC90 OR MSVC10)
    set(MSVC ON)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

set(SOURCE_SUBFOLDER ${CMAKE_CURRENT_SOURCE_DIR}/source_subfolder)
set(BZ2_LIBRARY bz2)
set(BZ2_NAMESPACE BZip2)
set(BZ2_CONFIG ${BZ2_NAMESPACE}Config)

option(BZ2_BUILD_EXE OFF)

add_library(${BZ2_LIBRARY}  ${SOURCE_SUBFOLDER}/blocksort.c
                            ${SOURCE_SUBFOLDER}/bzlib.c
                            ${SOURCE_SUBFOLDER}/compress.c
                            ${SOURCE_SUBFOLDER}/crctable.c
                            ${SOURCE_SUBFOLDER}/decompress.c
                            ${SOURCE_SUBFOLDER}/huffman.c
                            ${SOURCE_SUBFOLDER}/randtable.c
                            ${SOURCE_SUBFOLDER}/bzlib.h
                            ${SOURCE_SUBFOLDER}/bzlib_private.h)
target_include_directories(${BZ2_LIBRARY} PRIVATE ${SOURCE_SUBFOLDER})


if(ENABLE_MSAN OR ENABLE_TSAN OR ENABLE_ASAN OR ENABLE_UBSAN)
  message(STATUS
    "Using sanitizers")

  if(cmake_build_type_tolower MATCHES "release" )
    message(WARNING
      "Disable sanitizers in Release builds")
  endif()

  # use llvm_tools from conan
  find_program_helper(llvm-symbolizer
    PATHS
      #${CONAN_BIN_DIRS}
      ${CONAN_BIN_DIRS_LLVM_TOOLS}
    NO_SYSTEM_ENVIRONMENT_PATH
    NO_CMAKE_SYSTEM_PATH
    ${ARGUMENTS_UNPARSED_ARGUMENTS}
    REQUIRED
    OUT_VAR LLVM_SYMBOLIZER_PROGRAM
    VERBOSE TRUE
  )
  check_sanitizer_options(
    ENABLE_TSAN ${ENABLE_TSAN}
    ENABLE_ASAN ${ENABLE_ASAN}
    ENABLE_MSAN ${ENABLE_MSAN}
    ENABLE_UBSAN ${ENABLE_UBSAN}
    LLVM_SYMBOLIZER_PROGRAM ${LLVM_SYMBOLIZER_PROGRAM}
  )
else()
  message(STATUS
    "Disabled sanitizers")
endif()

if(ENABLE_MSAN)
  message(STATUS "enabling MSAN on ${BZ2_LIBRARY}")
  add_msan_static_link(${BZ2_LIBRARY})
  add_msan_definitions(${BZ2_LIBRARY})
  add_msan_flags()
else(ENABLE_MSAN)
  message(STATUS
    "Disabled ENABLE_MSAN")
endif(ENABLE_MSAN)

if(ENABLE_TSAN)
  message(STATUS "enabling TSAN on ${BZ2_LIBRARY}")
  add_tsan_static_link(${BZ2_LIBRARY})
  add_tsan_flags(${BZ2_LIBRARY})
  add_tsan_definitions(${BZ2_LIBRARY})
  add_tsan_flags()
else(ENABLE_TSAN)
  message(STATUS
    "Disabled ENABLE_TSAN")
endif(ENABLE_TSAN)

if(ENABLE_ASAN)
  message(STATUS "enabling ASAN on ${BZ2_LIBRARY}")
  add_asan_static_link(${BZ2_LIBRARY})
  add_asan_definitions(${BZ2_LIBRARY})
  add_asan_flags()
else(ENABLE_ASAN)
  message(STATUS
    "Disabled ENABLE_ASAN")
endif(ENABLE_ASAN)

if(ENABLE_UBSAN)
  message(STATUS "enabling UBSAN on ${BZ2_LIBRARY}")
  add_ubsan_static_link(${BZ2_LIBRARY})
  add_ubsan_definitions(${BZ2_LIBRARY})
  add_ubsan_flags()
else(ENABLE_UBSAN)
  message(STATUS
    "Disabled ENABLE_UBSAN")
endif(ENABLE_UBSAN)

if(BZ2_BUILD_EXE)
    add_executable(${CMAKE_PROJECT_NAME} ${SOURCE_SUBFOLDER}/bzip2.c)
    target_link_libraries(${CMAKE_PROJECT_NAME} ${BZ2_LIBRARY})
    target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ${SOURCE_SUBFOLDER})
endif()

set_target_properties(${BZ2_LIBRARY} PROPERTIES VERSION ${BZ2_VERSION_STRING} SOVERSION ${BZ2_VERSION_MAJOR})

export(TARGETS ${BZ2_LIBRARY}
       NAMESPACE ${BZ2_NAMESPACE}::
       FILE "${CMAKE_CURRENT_BINARY_DIR}/${BZ2_CONFIG}.cmake")

install(
  TARGETS ${BZ2_LIBRARY}
  EXPORT ${BZ2_CONFIG}
  INCLUDES DESTINATION
    ${CMAKE_INSTALL_INCLUDEDIR}
  PUBLIC_HEADER DESTINATION
    ${CMAKE_INSTALL_INCLUDEDIR}
  RUNTIME DESTINATION
    ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION
    ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION
    ${CMAKE_INSTALL_LIBDIR})

if(BZ2_BUILD_EXE)
    install(
      TARGETS ${CMAKE_PROJECT_NAME}
      EXPORT ${BZ2_CONFIG}
      INCLUDES DESTINATION
        ${CMAKE_INSTALL_INCLUDEDIR}
      PUBLIC_HEADER DESTINATION
        ${CMAKE_INSTALL_INCLUDEDIR}
      RUNTIME DESTINATION
        ${CMAKE_INSTALL_BINDIR}
      LIBRARY DESTINATION
        ${CMAKE_INSTALL_LIBDIR}
      ARCHIVE DESTINATION
        ${CMAKE_INSTALL_LIBDIR})
endif()

install(FILES ${SOURCE_SUBFOLDER}/bzlib.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(EXPORT ${BZ2_CONFIG}
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${CMAKE_PROJECT_NAME}"
        NAMESPACE ${BZ2_LIBRARY}::)
