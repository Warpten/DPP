find_package(OpenSSL 1.1 REQUIRED COMPONENTS Crypto SSL)

add_library(dpp_openssl INTERFACE IMPORTED)

target_link_libraries(dpp_openssl
  INTERFACE
    OpenSSL::Crypto
    OpenSSL::SSL
)

add_library(dpp::openssl ALIAS dpp_openssl)
