cmake_minimum_required(VERSION 3.5)

if(POLICY CMP0066)
  cmake_policy(SET CMP0066 OLD)
endif()
if(POLICY CMP0067)
  cmake_policy(SET CMP0067 NEW)
endif()

include(cmake/Mac.cmake)

project(fish)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# We are C++11.
set(CMAKE_CXX_STANDARD 11)
set(DEFAULT_BUILD_TYPE "RelWithDebInfo")

# Generate Xcode schemas (but not for tests).
set(CMAKE_XCODE_GENERATE_SCHEME 1)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "Setting build type to default '${DEFAULT_BUILD_TYPE}'")
    set(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}")
endif()

# Error out when linking statically, it doesn't work.
if (CMAKE_EXE_LINKER_FLAGS MATCHES ".*-static.*")
    message(FATAL_ERROR "Fish does not support static linking")
endif()

# Force colored warnings in Ninja's output, if the compiler has -fdiagnostics-color support.
# Rationale in https://github.com/ninja-build/ninja/issues/814
if (CMAKE_GENERATOR STREQUAL "Ninja" AND
    ((CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9) OR
     (CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 3.5) OR
     (CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang" AND NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 6.0)))
      add_compile_options(-fdiagnostics-color=always)
endif()

# Enable a whole bunch of warnings, but turn off:
# - comment because we use a bunch of those, and they're not really all that harmful.
# - address, because that occurs for our mkostemp check (weak-linking requires us to compare `&mkostemp == nullptr`).
add_compile_options(-Wall -Wextra -Wno-comment -Wno-address)

if ((CMAKE_CXX_COMPILER_ID STREQUAL "Clang") OR (CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang"))
  add_compile_options(-Wunused-template -Wunused-local-typedef -Wunused-macros)
endif()

# Disable exception handling.
add_compile_options(-fno-exceptions)

# Undefine NDEBUG to keep assert() in release builds.
add_definitions(-UNDEBUG)

# Hide the CMake Rules directories in Xcode projects.
source_group("CMake Rules" REGULAR_EXPRESSION "^$")

# Put source and header files at top level under targets.
source_group("Source Files" REGULAR_EXPRESSION ".*\\.cpp")
source_group("Header Files" REGULAR_EXPRESSION ".*\\.h")
source_group("Builtins" "builtins/")

# Support folders.
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# Work around issue where archive-built libs go in the wrong place.
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR})

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
  set(FISH_IN_TREE_BUILD TRUE)
else()
  set(FISH_IN_TREE_BUILD FALSE)
endif()

# NetBSD does weird things with finding libraries,
# making the tests fail by failing to find pcre.
#
# Keep the rpath used to build.
if(CMAKE_SYSTEM_NAME STREQUAL NetBSD)
  set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
endif()

# List of sources for builtin functions.
set(FISH_BUILTIN_SRCS
    src/builtin.cpp src/builtins/argparse.cpp
    src/builtins/bg.cpp src/builtins/bind.cpp src/builtins/block.cpp
    src/builtins/builtin.cpp src/builtins/cd.cpp src/builtins/command.cpp
    src/builtins/commandline.cpp src/builtins/complete.cpp src/builtins/contains.cpp
    src/builtins/disown.cpp src/builtins/echo.cpp src/builtins/emit.cpp
    src/builtins/eval.cpp src/builtins/exit.cpp src/builtins/fg.cpp
    src/builtins/function.cpp src/builtins/functions.cpp src/builtins/history.cpp
    src/builtins/jobs.cpp src/builtins/math.cpp src/builtins/printf.cpp src/builtins/path.cpp
    src/builtins/pwd.cpp src/builtins/random.cpp src/builtins/read.cpp
    src/builtins/realpath.cpp src/builtins/return.cpp src/builtins/set.cpp
    src/builtins/set_color.cpp src/builtins/source.cpp src/builtins/status.cpp
    src/builtins/string.cpp src/builtins/test.cpp src/builtins/type.cpp src/builtins/ulimit.cpp
    src/builtins/wait.cpp)

# List of other sources.
set(FISH_SRCS
    src/ast.cpp src/autoload.cpp src/color.cpp src/common.cpp src/complete.cpp src/env.cpp
    src/env_dispatch.cpp src/env_universal_common.cpp src/event.cpp src/exec.cpp
    src/expand.cpp src/fallback.cpp src/fd_monitor.cpp src/fish_version.cpp
    src/flog.cpp src/function.cpp src/future_feature_flags.cpp src/highlight.cpp
    src/history.cpp src/history_file.cpp src/input.cpp src/input_common.cpp
    src/io.cpp src/iothread.cpp src/job_group.cpp src/kill.cpp
    src/null_terminated_array.cpp src/operation_context.cpp src/output.cpp
    src/pager.cpp src/parse_execution.cpp src/parse_tree.cpp src/parse_util.cpp
    src/parser.cpp src/parser_keywords.cpp src/path.cpp src/postfork.cpp
    src/proc.cpp src/re.cpp src/reader.cpp src/redirection.cpp src/screen.cpp
    src/signal.cpp src/termsize.cpp src/timer.cpp src/tinyexpr.cpp
    src/tokenizer.cpp src/topic_monitor.cpp src/trace.cpp src/utf8.cpp src/util.cpp
    src/wait_handle.cpp src/wcstringutil.cpp src/wgetopt.cpp src/wildcard.cpp 
    src/wutil.cpp src/fds.cpp
)

# Header files are just globbed.
file(GLOB FISH_HEADERS src/*.h)

# Set up config.h
include(cmake/ConfigureChecks.cmake)
include(cmake/gettext.cmake)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config_cmake.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

# Set up standard directories.
include(GNUInstallDirs)
add_definitions(-D_UNICODE=1
                -DLOCALEDIR="${CMAKE_INSTALL_FULL_LOCALEDIR}"
                -DPREFIX=L"${CMAKE_INSTALL_PREFIX}"
                -DDATADIR=L"${CMAKE_INSTALL_FULL_DATADIR}"
                -DSYSCONFDIR=L"${CMAKE_INSTALL_FULL_SYSCONFDIR}"
                -DBINDIR=L"${CMAKE_INSTALL_FULL_BINDIR}"
                -DDOCDIR=L"${CMAKE_INSTALL_FULL_DOCDIR}")

# Set up the machinery around FISH-BUILD-VERSION-FILE
# This defines the FBVF variable.
include(Version)

# Let fish pick up when we're running out of the build directory without installing
get_filename_component(REAL_CMAKE_BINARY_DIR "${CMAKE_BINARY_DIR}" REALPATH)
get_filename_component(REAL_CMAKE_SOURCE_DIR "${CMAKE_SOURCE_DIR}" REALPATH)
add_definitions(-DCMAKE_BINARY_DIR="${REAL_CMAKE_BINARY_DIR}")
add_definitions(-DCMAKE_SOURCE_DIR="${REAL_CMAKE_SOURCE_DIR}")

# Teach fish_version.o to rebuild when FBVF changes.
# The standard C++ include detection machinery misses this.
set_source_files_properties(src/fish_version.cpp
                            PROPERTIES OBJECT_DEPENDS
                            ${CMAKE_CURRENT_BINARY_DIR}/${FBVF})

# Enable thread-safe errno on Solaris (#5611)
add_definitions(-D_REENTRANT)

# Set up PCRE2
include(cmake/PCRE2.cmake)

# Define a function to link dependencies.
function(FISH_LINK_DEPS_AND_SIGN target)
  target_link_libraries(${target} fishlib)
  codesign_on_mac(${target})
endfunction(FISH_LINK_DEPS_AND_SIGN)

# Define libfish.a.
add_library(fishlib STATIC ${FISH_SRCS} ${FISH_BUILTIN_SRCS})
target_sources(fishlib PRIVATE ${FISH_HEADERS})
target_link_libraries(fishlib
  ${CURSES_LIBRARY} ${CURSES_EXTRA_LIBRARY} Threads::Threads ${CMAKE_DL_LIBS}
  ${PCRE2_LIB} ${Intl_LIBRARIES} ${ATOMIC_LIBRARY})
target_include_directories(fishlib PRIVATE
  ${CURSES_INCLUDE_DIRS})

# Define fish.
add_executable(fish src/fish.cpp)
fish_link_deps_and_sign(fish)

# Define fish_indent.
add_executable(fish_indent
               src/fish_indent.cpp src/print_help.cpp)
fish_link_deps_and_sign(fish_indent)

# Define fish_key_reader.
add_executable(fish_key_reader
               src/fish_key_reader.cpp src/print_help.cpp)
fish_link_deps_and_sign(fish_key_reader)

# Set up the docs.
include(cmake/Docs.cmake)

# A helper for running tests.
add_executable(fish_test_helper src/fish_test_helper.cpp)

# Set up tests.
include(cmake/Tests.cmake)

# Benchmarking support.
include(cmake/Benchmark.cmake)

# Set up install.
include(cmake/Install.cmake)

# Mac app.
include(cmake/MacApp.cmake)

# ThreadSanitizer likes to muck with signal handlers, which interferes
# with fish_test_helper printing the ignored signal mask.
# Ensure fish_test_helper does not use TSan.
# Note the environment var is CXXFLAGS, but the CMake var is CMAKE_CXX_FLAGS.
if (CMAKE_CXX_FLAGS MATCHES ".*-fsanitize=thread.*")
  target_compile_options(fish_test_helper PRIVATE "-fno-sanitize=all")
  target_link_libraries(fish_test_helper "-fno-sanitize=all")
endif()

# Lint targets
# This could be implemented as target properties, but the script has the useful feature of only
# checking the currently-staged commands
# The generator expressions below rebuild the command line for the fishlib targets
# CMake does not support the "iquote" flag - https://gitlab.kitware.com/cmake/cmake/issues/15491
set(LINT_ARGS "-D$<JOIN:$<TARGET_PROPERTY:fishlib,COMPILE_DEFINITIONS>, -D>" "-I$<JOIN:$<TARGET_PROPERTY:fishlib,INCLUDE_DIRECTORIES>, -I>")
add_custom_target(lint
    COMMAND build_tools/lint.fish -p ${CMAKE_BINARY_DIR} -- ${LINT_ARGS}
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
)
add_custom_target(lint-all
    COMMAND build_tools/lint.fish --all -p ${CMAKE_BINARY_DIR} -- ${LINT_ARGS}
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
)

include(FeatureSummary)
feature_summary(WHAT ALL)
