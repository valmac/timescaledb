cmake_minimum_required(VERSION 3.11)
include(CheckCCompilerFlag)

# Function to call pg_config and extract values.
#
function(GET_PG_CONFIG var)
  set(_temp)

  # Only call pg_config if the variable didn't already have a value.
  if(NOT ${var})
    execute_process(
      COMMAND ${PG_CONFIG} ${ARGN}
      OUTPUT_VARIABLE _temp
      OUTPUT_STRIP_TRAILING_WHITESPACE)
  endif()

  # On Windows, fields that are not recorded will be given the value
  # "not recorded", so we translate this into <var>-NOTFOUND to make
  # it undefined.
  #
  # It will then also show as, e.g., "PG_LDFLAGS-NOTFOUND" in any
  # string interpolation, making it obvious that it is an undefined
  # CMake variable.
  if("${_temp}" STREQUAL "not recorded")
    set(_temp ${var}-NOTFOUND)
  endif()

  set(${var} ${_temp} PARENT_SCOPE)
endfunction()

configure_file("version.config" "version.config" COPYONLY)
file(READ version.config VERSION_CONFIG)
set(VERSION_REGEX "version[\t ]*=[\t ]*([0-9]+\\.[0-9]+\\.*[0-9]*)([-]([a-z]+[0-9]*))?([-](dev))?\r?\nupdate_from_version[\t ]*=[\t ]*([0-9]+\\.[0-9]+\\.*[0-9]*([-]([a-z]+[0-9]*))?)*(\r?\n)*$")

if (NOT (${VERSION_CONFIG} MATCHES ${VERSION_REGEX}))
  message(FATAL_ERROR "Cannot read version from version.config")
endif ()

# a hack to avoid change of SQL extschema variable
set(extschema "@extschema@")
set(VERSION ${CMAKE_MATCH_1})
set(VERSION_MOD ${CMAKE_MATCH_3})
set(VERSION_DEV ${CMAKE_MATCH_5})
set(UPDATE_FROM_VERSION ${CMAKE_MATCH_6})

if (VERSION_MOD AND VERSION_DEV)
  set(PROJECT_VERSION_MOD ${VERSION}-${VERSION_MOD}-${VERSION_DEV})
elseif (VERSION_MOD)
  set(PROJECT_VERSION_MOD ${VERSION}-${VERSION_MOD})
elseif (VERSION_DEV)
  set(PROJECT_VERSION_MOD ${VERSION}-${VERSION_DEV})
else ()
  set(PROJECT_VERSION_MOD ${VERSION})
endif ()

# Set project name, version, and language. Language needs to be set for compiler checks
project(timescaledb VERSION ${VERSION} LANGUAGES C)

if (NOT CMAKE_BUILD_TYPE)
  # Default to Release builds
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel" FORCE)
endif ()

message(STATUS "TimescaleDB version ${PROJECT_VERSION_MOD}. Can be updated from version ${UPDATE_FROM_VERSION}")
message(STATUS "Build type is ${CMAKE_BUILD_TYPE}")

set(PROJECT_INSTALL_METHOD source CACHE STRING "Specify what install platform this binary
is built for")
message(STATUS "Install method is '${PROJECT_INSTALL_METHOD}'")

# Build compilation database by default
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Code coverage is optional and OFF by default
option(CODECOVERAGE "Enable code coverage for the build" OFF)

if (CMAKE_BUILD_TYPE MATCHES Debug)
  # CMAKE_BUILD_TYPE is set at CMake configuration type. But usage of CMAKE_C_FLAGS_DEBUG is
  # determined at build time by running cmake --build . --config Debug (at least on Windows).
  # Therefore, we only set these flags if the configuration-time CMAKE_BUILD_TYPE is set to
  # Debug. Then Debug enabled builds will only happen on Windows if both the configuration-
  # and build-time settings are Debug.
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -DDEBUG=1 -DTS_DEBUG=1")
endif (CMAKE_BUILD_TYPE MATCHES Debug)

set(SUPPORTED_COMPILERS "GNU" "Clang" "AppleClang" "MSVC")

# Check for a supported compiler
if (NOT CMAKE_C_COMPILER_ID IN_LIST SUPPORTED_COMPILERS)
   message(FATAL_ERROR "Unsupported compiler ${CMAKE_C_COMPILER_ID}. Supported compilers are: ${SUPPORTED_COMPILERS}")
endif ()

# Option to treat warnings as errors when compiling (default on for
# debug builds, off for all other build types)
if (CMAKE_BUILD_TYPE STREQUAL Debug)
  message(STATUS "CMAKE_BUILD_TYPE matches Debug")
  option(WARNINGS_AS_ERRORS "Make compiler warnings into errors (default ON)" ON)
else()
  message(STATUS "CMAKE_BUILD_TYPE does not match Debug")
  option(WARNINGS_AS_ERRORS "Make compiler warnings into errors (default ON)" OFF)
endif()

if (WARNINGS_AS_ERRORS)
  if (CMAKE_C_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
    add_compile_options(-Werror)
  elseif (CMAKE_C_COMPILER_ID MATCHES "MSVC")
    add_compile_options(/WX)
  endif ()
endif (WARNINGS_AS_ERRORS)

if(CMAKE_C_COMPILER_ID MATCHES "GNU|AppleClang|Clang")
  # These two flags generate too many errors currently, but we
  # probably want these optimizations enabled.
  #
  # -fdelete-null-pointer-checks -Wnull-dereference

  # This flag avoid some subtle bugs related to standard conversions,
  # but currently does not compile because we are using too many
  # implicit conversions that potentially lose precision.
  #
  # -Wconversions

  # These flags are supported on all compilers.
  add_compile_options(
    -Wempty-body -Wvla -Wall -Wmissing-prototypes -Wpointer-arith
    -Werror=vla -Wendif-labels
    -fno-strict-aliasing -fno-omit-frame-pointer)

  # These flags are just supported on some of the compilers, so we
  # check them before adding them.
  check_c_compiler_flag(-Wno-format-truncation CC_SUPPORTS_NO_FORMAT_TRUNCATION)
  if(CC_SUPPORTS_NO_FORMAT_TRUNCATION)
    add_compile_options(-Wno-format-truncation)
  else()
    message(STATUS "Compiler does not support -Wno-format-truncation")
  endif()

  check_c_compiler_flag(-Wno-stringop-truncation CC_NO_STRINGOP_TRUNCATION)
  if(CC_NO_STRINGOP_TRUNCATION)
    add_compile_options(-Wno-stringop-truncation)
  else()
    message(STATUS "Compiler does not support -Wno-stringop-truncation")
  endif()

  check_c_compiler_flag(-Wimplicit-fallthrough CC_SUPPORTS_IMPLICIT_FALLTHROUGH)
  if(CC_SUPPORTS_IMPLICIT_FALLTHROUGH)
    add_compile_options( -Wimplicit-fallthrough)
  else()
    message(STATUS "Compiler does not support -Wimplicit-fallthrough")
  endif()

  # On UNIX, the compiler needs to support -fvisibility=hidden to hide symbols by default
  check_c_compiler_flag(-fvisibility=hidden CC_SUPPORTS_VISIBILITY_HIDDEN)

  if (NOT CC_SUPPORTS_VISIBILITY_HIDDEN)
    message(FATAL_ERROR "The compiler ${CMAKE_C_COMPILER_ID} does not support -fvisibility=hidden")
  endif (NOT CC_SUPPORTS_VISIBILITY_HIDDEN)
endif()

# On Windows, default to only include Release builds so MSBuild.exe 'just works'
if (WIN32 AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_CONFIGURATION_TYPES Release CACHE STRING "Semicolon separated list of supported configuration types, only supports Debug, Release, MinSizeRel, and RelWithDebInfo, anything else will be ignored." FORCE)
endif ()

message(STATUS "Using compiler ${CMAKE_C_COMPILER_ID}")

if (ENABLE_OPTIMIZER_DEBUG)
  message(STATUS "Enabling OPTIMIZER_DEBUG. Make sure that ${PG_SOURCE_DIR} is installed and built with OPTIMIZER_DEBUG.")
  add_definitions(-DOPTIMIZER_DEBUG)
endif()

# Search paths for Postgres binaries
if(WIN32)
  find_path(PG_PATH
    postgres.exe
    PATHS "C:/PostgreSQL" "C:/Program Files/PostgreSQL"
    PATH_SUFFIXES bin 12/bin 11/bin
    DOC "The path to a PostgreSQL installation")
elseif(UNIX)
  find_path(PG_PATH
    postgres
    PATHS $ENV{HOME} /opt/local/pgsql /usr/local/pgsql /usr/lib/postgresql
    PATH_SUFFIXES bin 12/bin 11/bin
    DOC "The path to a PostgreSQL installation")
endif()

find_program(PG_CONFIG pg_config
  HINTS ${PG_PATH}
  PATH_SUFFIXES bin
  DOC "The path to the pg_config of the PostgreSQL version to compile against")

if (NOT PG_CONFIG)
  message(FATAL_ERROR "Unable to find 'pg_config'")
endif ()

find_package(Git)

if(GIT_FOUND)
  # We use "git describe" to generate the tag. It will find the latest
  # tag and also add some additional information if we are not on the
  # tag.
  execute_process(
    COMMAND ${GIT_EXECUTABLE} describe --dirty --always --tags
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE EXT_GIT_COMMIT_TAG
    RESULT_VARIABLE _describe_RESULT
    OUTPUT_STRIP_TRAILING_WHITESPACE)

  # Fetch the commit HASH of head (short version) using rev-parse
  execute_process(
    COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE EXT_GIT_COMMIT_HASH
    RESULT_VARIABLE _revparse_RESULT
    OUTPUT_STRIP_TRAILING_WHITESPACE)

  # Fetch the date of the head commit
  execute_process(
    COMMAND ${GIT_EXECUTABLE} log -1 --format=%aI
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE EXT_GIT_COMMIT_TIME
    RESULT_VARIABLE _log_RESULT
    OUTPUT_STRIP_TRAILING_WHITESPACE)

  # Results are non-zero if there were an error
  if(_describe_RESULT OR _revparse_RESULT OR _log_RESULT)
    message(STATUS "Unable to get git commit information")
  endif()
endif()

message(STATUS "Using pg_config ${PG_CONFIG}")

# Check PostgreSQL version
execute_process(
  COMMAND ${PG_CONFIG} --version
  OUTPUT_VARIABLE PG_VERSION_STRING
  OUTPUT_STRIP_TRAILING_WHITESPACE)

if (NOT ${PG_VERSION_STRING} MATCHES "^PostgreSQL[ ]+([0-9]+)(\\.([0-9]+)|devel|rc[0-9]+)")
  message(FATAL_ERROR "Could not parse PostgreSQL version ${PG_VERSION_STRING}")
endif ()

set(PG_VERSION_MAJOR ${CMAKE_MATCH_1})
if (${CMAKE_MATCH_COUNT} GREATER "2" )
  set(PG_VERSION_MINOR ${CMAKE_MATCH_3})
else()
  set(PG_VERSION_MINOR 0)
endif ()
set(PG_VERSION "${PG_VERSION_MAJOR}.${PG_VERSION_MINOR}")

message(STATUS "Compiling against PostgreSQL version ${PG_VERSION}")

# Ensure that PostgreSQL version is supported and consistent
# with src/compat.h version check
if ((${PG_VERSION_MAJOR} LESS "11") OR
    (${PG_VERSION_MAJOR} GREATER "13"))
  message(FATAL_ERROR "TimescaleDB only supports PostgreSQL 11, 12 and 13")
endif()

# Get PostgreSQL configuration from pg_config
get_pg_config(PG_INCLUDEDIR --includedir)
get_pg_config(PG_INCLUDEDIR_SERVER --includedir-server)
get_pg_config(PG_LIBDIR --libdir)
get_pg_config(PG_PKGLIBDIR --pkglibdir)
get_pg_config(PG_SHAREDIR --sharedir)
get_pg_config(PG_BINDIR --bindir)
get_pg_config(PG_CPPFLAGS --cppflags)
get_pg_config(PG_LDFLAGS --ldflags)
get_pg_config(PG_LIBS --libs)

find_path(PG_SOURCE_DIR
  src/include/pg_config.h.in
  HINTS
  $ENV{HOME}
  $ENV{HOME}/projects
  $ENV{HOME}/Projects
  $ENV{HOME}/development
  $ENV{HOME}/Development
  $ENV{HOME}/workspace
  PATH_SUFFIXES
  postgres
  postgresql
  pgsql
  DOC
  "The path to the PostgreSQL source tree")

if (PG_SOURCE_DIR)
  message(STATUS "Found PostgreSQL source in ${PG_SOURCE_DIR}")
endif (PG_SOURCE_DIR)

set(EXT_CONTROL_FILE ${PROJECT_NAME}.control)
configure_file(${EXT_CONTROL_FILE}.in ${EXT_CONTROL_FILE})

install(
  FILES ${CMAKE_CURRENT_BINARY_DIR}/${EXT_CONTROL_FILE}
  DESTINATION "${PG_SHAREDIR}/extension")

# We look for specific versions installed by distributions before the
# default name since distros that have installed clang-format-9 will
# not work and the user need to install an earlier version, which will
# then be named "clang-format-N".
#
# This breaks the CMake convention of using the "default" name first
# to handle local installs. If this turns out to be something that we
# want to support, we need to look specifically for "clang-format" in
# the local installation paths before looking for the versioned names
# in standard installation paths.
find_program(CLANG_FORMAT
  NAMES clang-format-8 clang-format-7 clang-format
  PATHS
  /usr/bin
  /usr/local/bin
  /usr/local/opt/
  /usr/local/opt/llvm/bin
  /opt/bin
  DOC "The path to clang-format")

if (CLANG_FORMAT)
  execute_process(
    COMMAND ${CLANG_FORMAT} --version
    OUTPUT_VARIABLE CLANG_FORMAT_VERSION_OUTPUT
    OUTPUT_STRIP_TRAILING_WHITESPACE)

  if (NOT ${CLANG_FORMAT_VERSION_OUTPUT} MATCHES "version[ ]+([0-9]+)\\.([0-9]+)(\\.([0-9]+))*")
    message(FATAL_ERROR "Could not parse clang-format version ${CLANG_FORMAT_VERSION_OUTPUT}")
  endif ()

  if((${CMAKE_MATCH_1} LESS "7") OR (${CMAKE_MATCH_1} GREATER "8"))
    message(WARNING "clang-format version 7 or 8 required")
    set(CLANG_FORMAT False)
  endif()
endif ()

if (NOT CLANG_FORMAT)
  find_program(DOCKER docker DOC "The path to docker")

  if(NOT DOCKER)
    message(WARNING "clang-format is disabled (can't find clang-format or docker)")
  else ()
    message(STATUS "Using docker based clang-format")
    add_custom_target(format
      COMMAND docker run --rm -it --user=`id -u`:`id -g` --volume=${PROJECT_SOURCE_DIR}:/timescaledb timescaledev/postgres-dev-clang:clang7-pg11.1 /timescaledb/scripts/clang_format_all.sh
    )
  endif()
else()
  message(STATUS "Using local clang-format")
  add_custom_target(format
    COMMAND ${CMAKE_COMMAND} -E env CLANG_FORMAT=${CLANG_FORMAT} ${PROJECT_SOURCE_DIR}/scripts/clang_format_all.sh
  )
endif ()

if(CMAKE_C_COMPILER_ID MATCHES "Clang|AppleClang")
  set(LINTER_DEFAULT ON)
else()
  set(LINTER_DEFAULT OFF)
endif()

# Linter support via clang-tidy. Enabled when using clang as compiler
option(LINTER "Enable linter support using clang-tidy (ON when using clang)" ${LINTER_DEFAULT})

if (LINTER)
  find_program(CLANG_TIDY
    clang-tidy
    PATHS
    /usr/bin
    /usr/local/bin
    /usr/local/opt/
    /usr/local/opt/llvm/bin
    /opt/bin
    DOC "The path to the clang-tidy linter")

  if (CLANG_TIDY)
    message(STATUS "Linter support (clang-tidy) enabled")
    set(CMAKE_C_CLANG_TIDY "${CLANG_TIDY};--quiet")
  else ()
    message(STATUS "Install clang-tidy to enable code linting")
  endif (CLANG_TIDY)
endif (LINTER)

option(USE_OPENSSL "Enable use of OpenSSL if available" ON)
option(SEND_TELEMETRY_DEFAULT "The default value for whether to send telemetry" ON)
option(REGRESS_CHECKS "PostgreSQL regress checks through installcheck" ON)
option(ENABLE_OPTIMIZER_DEBUG "Enable OPTIMIZER_DEBUG when building. Requires Postgres server to be built with OPTIMIZER_DEBUG." OFF)

# Option to enable assertions. Note that if we include headers from a
# PostgreSQL build that has assertions enabled, we might inherit that
# setting without explicitly enabling assertions via the ASSERTIONS
# option defined here. Thus, this option is mostly useful to enable
# assertions when the PostgreSQL we compile against has it disabled.
option(ASSERTIONS "Compile with assertion checks (default OFF)" OFF)

if (NOT EXISTS ${PG_INCLUDEDIR}/pg_config.h)
  message(FATAL_ERROR "Could not find pg_config.h in ${PG_INCLUDEDIR}. "
    "Make sure PG_PATH points to a valid PostgreSQL installation that includes development headers.")
endif ()

file(READ ${PG_INCLUDEDIR}/pg_config.h PG_CONFIG_H)
string(REGEX MATCH "#define USE_ASSERT_CHECKING 1" PG_USE_ASSERT_CHECKING ${PG_CONFIG_H})

if (PG_USE_ASSERT_CHECKING AND NOT ASSERTIONS)
  message("Assertion checks are OFF although enabled in PostgreSQL build (pg_config.h). "
    "The PostgreSQL setting for assertions will take precedence.")
elseif (ASSERTIONS)
  message(STATUS "Assertion checks are ON")
  add_compile_definitions(USE_ASSERT_CHECKING=1)
elseif (CMAKE_BUILD_TYPE MATCHES Debug)
  message("Assertion checks are OFF in Debug build. Set -DASSERTIONS=ON to enable assertions.")
else ()
  message(STATUS "Assertion checks are OFF")
endif ()

# Check if PostgreSQL has OpenSSL enabled by inspecting pg_config --configure.
# Right now, a Postgres header will redefine an OpenSSL function if Postgres is not installed --with-openssl,
# so in order for TimescaleDB to compile correctly with OpenSSL, Postgres must also have OpenSSL enabled.
execute_process(
  COMMAND ${PG_CONFIG} --configure
  OUTPUT_VARIABLE PG_CONFIGURE_FLAGS
  OUTPUT_STRIP_TRAILING_WHITESPACE)
string(REGEX MATCH "--with-openssl" PG_USE_OPENSSL "${PG_CONFIGURE_FLAGS}")

if (USE_OPENSSL AND (NOT PG_USE_OPENSSL))
    message(FATAL_ERROR "PostgreSQL was built without OpenSSL support, which TimescaleDB needs for full compatibility. Please rebuild PostgreSQL using `--with-openssl` or if you want to continue without OpenSSL, re-run bootstrap with `-DUSE_OPENSSL=0`")
endif (USE_OPENSSL AND (NOT PG_USE_OPENSSL))

if (USE_OPENSSL)
  # Try to find a local OpenSSL installation
  find_package(OpenSSL)

  if (NOT OPENSSL_FOUND)
    message(FATAL_ERROR "TimescaleDB requires OpenSSL but it wasn't found. If you want to continue without OpenSSL, re-run bootstrap with `-DUSE_OPENSSL=0`")
  endif(NOT OPENSSL_FOUND)

  if (${OPENSSL_VERSION} VERSION_LESS "1.0")
    message(FATAL_ERROR "TimescaleDB requires OpenSSL version 1.0 or greater")
  endif ()

  if(CMAKE_BUILD_TYPE STREQUAL "Debug" AND MSVC)
    set(_libraries)
    foreach(_path ${OPENSSL_LIBRARIES})
      get_filename_component(_dir ${_path} DIRECTORY)
      get_filename_component(_name ${_path} NAME_WE)
      string(REGEX REPLACE "[Dd]$" "" _fixed ${_name})
      get_filename_component(_ext ${_path} EXT)
      list(APPEND _libraries "${_dir}/${_fixed}${_ext}")
    endforeach()
    set(OPENSSL_LIBRARIES ${_libraries})
  endif()
  message(STATUS "OPENSSL_LIBRARIES: ${OPENSSL_LIBRARIES}")
  message(STATUS "Using OpenSSL version ${OPENSSL_VERSION}")
endif (USE_OPENSSL)

if (CODECOVERAGE)
  message(STATUS "Code coverage is enabled.")
  # Note that --coverage is synonym for the necessary compiler and
  # linker flags for the given compiler.  For example, with GCC,
  # --coverage translates to -fprofile-arcs -ftest-coverage when
  # compiling and -lgcov when linking
  add_compile_options(--coverage -O0)
  add_link_options(--coverage)
endif (CODECOVERAGE)

if (UNIX)
  add_subdirectory(scripts)
endif (UNIX)

add_subdirectory(sql)
add_subdirectory(test)
add_subdirectory(src)

option(APACHE_ONLY "only compile apache code" off)

if(NOT APACHE_ONLY)
  add_subdirectory(tsl)
endif()

add_custom_target(licensecheck
  COMMAND ${PROJECT_SOURCE_DIR}/scripts/check_license_all.sh
  )

# This needs to be the last subdirectory so that other targets are
# already defined
if (CODECOVERAGE)
  add_subdirectory(codecov)
endif ()

if (IS_DIRECTORY ${PROJECT_SOURCE_DIR}/.git)
  configure_file(${PROJECT_SOURCE_DIR}/scripts/githooks/commit_msg.py ${PROJECT_SOURCE_DIR}/.git/hooks/commit-msg COPYONLY)
endif()
