# Expose a sodium interface. If DPP is built without voice support, this interface will literally do nothing.
add_library(dpp_sodium INTERFACE)

# Only search if we user tried to build DPP with voice support
if (DPP_BUILD_VOICE_SUPPORT)
  if (BUILD_SHARED_LIBS)
    set(sodium_USE_STATIC_LIBS    OFF)
  else ()
    set(sodium_USE_STATIC_LIBS    ON)
  endif ()

  find_package(unofficial-sodium CONFIG REQUIRED)

  if (unofficial-sodium_FOUND)
    message(STATUS "-- Found Sodium.")

    target_link_libraries(dpp_sodium
      INTERFACE
         unofficial-sodium::sodium
    )
  endif ()
endif ()

add_library(dpp::sodium ALIAS dpp_sodium)
