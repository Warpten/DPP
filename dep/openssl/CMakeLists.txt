# if (BUILD_SHARED_LIBS)
#     set(OPENSSL_USE_STATIC_LIBS OFF)
# else ()
#     set(OPENSSL_USE_STATIC_LIBS ON)
# endif ()

find_package(OpenSSL 1.1 REQUIRED COMPONENTS Crypto SSL)

add_library(dpp_openssl INTERFACE)

target_link_libraries(dpp_openssl
  INTERFACE
    OpenSSL::Crypto
    OpenSSL::SSL
)

add_library(dpp::openssl ALIAS dpp_openssl)
