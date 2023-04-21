# Expose an opus interface. If DPP is built without voice support, this interface will literally do nothing.
add_library(dpp_opus INTERFACE)
add_library(dpp::opus ALIAS dpp_opus)

# Only search if we user tried to build DPP with voice support
if (DPP_BUILD_VOICE_SUPPORT)
  # if (BUILD_SHARED_LIBS)
  #   set(OPUS_USE_STATIC_LIBS    OFF)
  # else ()
  #   set(OPUS_USE_STATIC_LIBS    ON)
  # endif ()

  find_package(Opus CONFIG REQUIRED)

  if (DEFINED OPUS_FOUND)
    message(STATUS "-- Found Opus: ${OPUS_INCLUDE_DIR}")

    target_link_libraries(dpp_opus
      INTERFACE
         Opus::opus
    )
  endif ()
endif ()
