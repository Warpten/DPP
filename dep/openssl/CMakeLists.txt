# TODO: Why is this even necessary? This feels like a CI-specific hack
if (APPLE AND CMAKE_APPLE_SILICON_PROCESSOR EQUAL arm64)
	set(OPENSSL_ROOT_DIR "/opt/homebrew/opt/openssl")
else ()
	set(OPENSSL_ROOT_DIR "/usr/local/opt/openssl")
endif ()

find_package(OpenSSL 1.1 REQUIRED COMPONENTS Crypto SSL)

add_library(dpp_openssl INTERFACE)

target_link_libraries(dpp_openssl
  INTERFACE
    OpenSSL::Crypto
    OpenSSL::SSL
)

add_library(dpp::openssl ALIAS dpp_openssl)
