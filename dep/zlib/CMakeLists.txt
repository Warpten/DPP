add_library(dpp_zlib INTERFACE IMPORTED)

# Only search if we user tried to build DPP with voice support
find_package(ZLIB REQUIRED)

target_link_libraries(dpp_zlib
  INTERFACE
    ZLIB::ZLIB
)

add_library(dpp::zlib ALIAS dpp_zlib)
