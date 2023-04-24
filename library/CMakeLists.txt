# Check for various C functions
check_cxx_symbol_exists(prctl "sys/prctl.h" HAVE_PRCTL)
check_cxx_symbol_exists(pthread_setname_np "pthread.h" HAVE_PTHREAD_SETNAME_NP)

# Identify the valid signature of pthread_setname_np, because different systems have different forms of this function.
if (HAVE_PTHREAD_SETNAME_NP)
  include(CheckCXXSourceCompiles)

  check_cxx_source_compiles("
#include <thread>

int main() {
  pthread_setname_np(\"ThreadName\");
  return 0;
}" HAVE_SINGLE_PARAMETER_SETNAME_NP)

  check_cxx_source_compiles("
#include <thread>

int main() {
  pthread_setname_np(pthread_self(), \"ThreadName\");
  return 0;
}" HAVE_TWO_PARAMETER_SETNAME_NP)
endif ()

# ###################################################################################################################
# Consumed packages.
find_package(PkgConfig QUIET)

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
find_package(Git QUIET)

# Look for ldconfig on Unix or MSYS if required
if ((UNIX OR MSYS) AND DPP_RUN_LDCONFIG)
  find_program(LDCONFIG_EXECUTABLE "ldconfig")
endif ()

# ###################################################################################################################
# Declaring the project now
if (NOT GIT_FOUND AND EXISTS "${CMAKE_SOURCE_DIR}/.git")
  message(FATAL_ERROR "You are using a git version of D++ but do not have git installed. Install git (not 'gh') and try again.")
endif ()

if (WIN32 AND NOT MINGW AND BUILD_SHARED_LIBS)
    message(STATUS "-- Configuring .rc resource script.")

    configure_file("${PROJECT_SOURCE_DIR}/src/dpp/dpp.rc.in"
      "${CMAKE_BINARY_DIR}/resources/dpp.rc"
      NEWLINE_STYLE WIN32
    )
endif ()

file(GLOB DPP_SUBDIRECTORY_LIST "${PROJECT_SOURCE_DIR}/src/dpp")

foreach (DPP_FULL_MODULE_NAME ${DPP_SUBDIRECTORY_LIST})
  get_filename_component(DPP_MODULE_NAME "${DPP_FULL_MODULE_NAME}" NAME)
  file(GLOB DPP_MODULE_SOURCES
    "${PROJECT_SOURCE_DIR}/src/dpp/*.cpp"
    "${PROJECT_SOURCE_DIR}/src/dpp/events/*.cpp"
    "${PROJECT_SOURCE_DIR}/src/dpp/cluster/*.cpp"
    "${CMAKE_BINARY_DIR}/resources/*.rc"
  )

  # This will be either STATIC or SHARED depending on BUILD_SHARED_LIBS.
  add_library("${DPP_MODULE_NAME}"
    ${DPP_MODULE_SOURCES}
  )

  set_target_properties("${DPP_MODULE_NAME}"
    PROPERTIES
    VERSION ${CMAKE_PROJECT_VERSION}
    SOVERSION ${CMAKE_PROJECT_VERSION}
    POSITION_INDEPENDENT_CODE ${BUILD_SHARED_LIBS}
  )

  target_include_directories("${DPP_MODULE_NAME}"
    PUBLIC
      $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
    PRIVATE
      $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/dpp>
  )

  add_dependencies("${DPP_MODULE_NAME}"
    dpp::openssl
    dpp::zlib
	dpp::sodium
	dpp::opus
  )

  target_link_libraries("${DPP_MODULE_NAME}"
    PUBLIC
	  Threads::Threads
    PRIVATE
      dpp::openssl
      dpp::zlib
      dpp::opus
      dpp::sodium
  )
endforeach ()

target_compile_features(dpp
  PUBLIC
    # Coroutines require C++20
    "$<IF:$<BOOL:DPP_EXPERIMENTAL_CORO>,cxx_std_20,cxx_std_17>"
  PRIVATE
    cxx_constexpr
    cxx_auto_type
    cxx_defaulted_functions
    cxx_deleted_functions
    cxx_final
    cxx_lambdas
    cxx_override
    cxx_thread_local
    cxx_variadic_templates
    cxx_attribute_deprecated
)

# Setup compiler flags.
# Explicitely not done with generator expressions to maintain readability.
if (NOT MSVC)
  target_compile_options(dpp
    PRIVATE
      -Wall
      -Wno-unused-private-field
      -Wno-psabi
      -Wempty-body
      -Wignored-qualifiers
      -Wimplicit-fallthrough
      -Wmissing-field-initializers
      -Wsign-compare
      -Wtype-limits
      -Wuninitialized
      -Wshift-negative-value
  )
else ()
  target_compile_options(dpp
    PRIVATE
      $<$<CONFIG:Debug>:/Zi>
      /sdl
      /MP
      /DFD_SETSIZE=1024
      /Zc:preprocessor
      /Zc:__cplusplus
      /permissive-
      /bigobj
  )
endif ()

# Coroutines require a bit more setup.
if (DPP_EXPERIMENTAL_CORO)
  if (MSVC AND NOT MINGW)
    target_compile_options(dpp PRIVATE /await:strict)
  else ()
    target_compile_options(dpp PRIVATE -fcoroutines)
  endif ()

  target_compile_definitions(dpp PUBLIC DPP_CORO)

  execute_process(WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/.."
    COMMAND php buildtools/make_struct.php "\\Dpp\\Generator\\CoroGenerator"
  )
endif ()

if (WIN32 AND NOT MINGW)
  target_compile_definitions(dpp
    PRIVATE
      OPENSSL_SYS_WIN32
      WIN32_LEAN_AND_MEAN
      _CRT_NONSTDC_NO_DEPRECATE
      _CRT_SECURE_NO_WARNINGS
      _WINSOCK_DEPRECATED_NO_WARNINGS
  )
endif()

target_compile_definitions(dpp
  PRIVATE
    DPP_BUILD
    DPP_OS=${CMAKE_SYSTEM_NAME}
)

# For some reason, these don't work properly with generator expressions ?!
if (HAVE_PRCTL)
  target_compile_definitions(dpp PRIVATE HAVE_PRCTL)
endif()

if (HAVE_PTHREAD_SETNAME_NP)
  target_compile_definitions(dpp PRIVATE HAVE_PTHREAD_SETNAME_NP)
endif ()

if (DPP_USE_EXTERNAL_JSON)
  # Use external JSON if need be. Expect it to be on the include path.
  # no questions asked.
  target_compile_definitions(dpp PRIVATE DPP_USE_EXTERNAL_JSON)
endif ()

if (NOT BUILD_SHARED_LIBS)
  target_compile_definitions(dpp PRIVATE DPP_STATIC_BUILD)
endif ()

set_target_properties(dpp
  PROPERTIES
    OUTPUT_NAME           "dpp"
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS        OFF
)

# ###################################################################################################################
# Set up unit tests

if (DPP_BUILD_TEST_TARGET)
  enable_testing("${PROJECT_SOURCE_DIR}")
  file(GLOB DPP_TEST_NAME_LIST "${PROJECT_SOURCE_DIR}/src/*" NAME)
  foreach (DPP_FULL_TEST_NAME ${DPP_TEST_NAME_LIST})
    get_filename_component(DPP_TEST_NAME ${DPP_FULL_TEST_NAME} NAME)

    if (NOT "${DPP_TEST_NAME}" STREQUAL "dpp")
      message(STATUS "-- Configuring test ${DPP_TEST_NAME} with source: ${DPP_FULL_TEST_NAME}/*.cpp")

      file(GLOB DPP_TEST_SRC "${DPP_FULL_TEST_NAME}/*.cpp")

      add_executable("${DPP_TEST_NAME}" ${DPP_TEST_SRC})

      target_link_libraries("${DPP_TEST_NAME}"
        PUBLIC
          dpp
      )
    endif ()
  endforeach ()

  add_test(NAME unittests
    COMMAND library/unittest
    WORKING_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/library
  )
endif ()

# ###################################################################################################################
# CPack support

if (DPP_INSTALL_TARGET)
  if (NOT WIN32)
    include(GNUInstallDirs)
    install(TARGETS dpp LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})
    install(DIRECTORY "${PROJECT_SOURCE_DIR}/include" DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

    install(CODE "execute_process(COMMAND ${CMAKE_COMMAND} -DDPP_RUN_LDCONFIG=${DPP_RUN_LDCONFIG} -DLDCONFIG_EXECUTABLE=${LDCONFIG_EXECUTABLE} -P ${PROJECT_SOURCE_DIR}/cmake/PostInstall.cmake)")

    configure_file("${PROJECT_SOURCE_DIR}/dpp.pc.in" "${CMAKE_BINARY_DIR}/dpp.pc" @ONLY)
    install(FILES "${CMAKE_BINARY_DIR}/dpp.pc" DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
    
    message(STATUS "-- Library install directory located at ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}")
    message(STATUS "-- Library include directory located at ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_INCLUDEDIR}")
  elseif (MINGW)
    install(TARGETS dpp LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)
    install(DIRECTORY "${PROJECT_SOURCE_DIR}/include" DESTINATION "${CMAKE_INSTALL_PREFIX}/include")

    message(STATUS "-- Library install directory located at ${CMAKE_INSTALL_PREFIX}/lib")
    message(STATUS "-- Library include directory located at ${CMAKE_INSTALL_PREFIX}/include")
  endif()

  # Setup information for packaging and distribution
  include("${PROJECT_SOURCE_DIR}/cmake/CPackSetup.cmake")
  include(CPack)
endif ()
