#  Copyright (c) 2018, Facebook, Inc.
#  All rights reserved.

cmake_minimum_required(VERSION 3.0.2)

project("fizz" VERSION 1.0.0 LANGUAGES CXX)

add_compile_options(-std=c++14)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/)

# When installing Folly & Fizz in a non-default prefix, this will let
# projects linking against libfizz.so to find libfolly.so automatically.
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(INCLUDE_INSTALL_DIR include CACHE STRING
    "The subdirectory where header files should be installed")
set(LIB_INSTALL_DIR lib CACHE STRING
    "The subdirectory where libraries should be installed")
set(CMAKE_INSTALL_DIR lib/cmake/fizz CACHE STRING
    "The subdirectory where CMake package config files should be installed")

# Try finding folly via its installed CMake configuration file first.
# This should work if folly was built with CMake.
find_package(folly CONFIG)
if (NOT folly_FOUND)
  # Look for folly using our FindFolly.cmake module.  This may work if the
  # folly was built with its older autoconf build files rather than with CMake.
  find_package(Folly MODULE REQUIRED)
endif()

find_package(Boost REQUIRED COMPONENTS system thread filesystem regex context)
find_package(OpenSSL REQUIRED)
find_package(Glog REQUIRED)
find_package(Gflags REQUIRED)
find_package(Libevent REQUIRED)
find_package(DoubleConversion REQUIRED)
find_package(Threads REQUIRED)
if (UNIX AND NOT APPLE)
  find_package(Librt)
endif()

include(CheckAtomic)

find_library(SODIUM sodium)
if(SODIUM-NOTFOUND)
  MESSAGE (FATAL_ERROR "libsodium is required")
ENDIF()


set(FIZZ_HEADER_DIRS
  base
  client
  crypto
  crypto/aead
  crypto/exchange
  crypto/signature
  crypto/openssl
  extensions/secretlogging
  extensions/tokenbinding
  protocol
  record
  server
)

set(FIZZ_TEST_HEADER_DIRS
  client/test
  crypto/aead/test
  crypto/exchange/test
  crypto/test
  protocol/test
  record/test
  server/test
  test
)

foreach(dir ${FIZZ_HEADER_DIRS})
  file(GLOB_RECURSE headers ${dir}/*.h)
  set(FIZZ_HEADERS
    ${FIZZ_HEADERS}
    ${headers})
endforeach()

foreach(dir ${FIZZ_TEST_HEADER_DIRS})
  file(GLOB_RECURSE headers ${dir}/*.h)
  set(FIZZ_TEST_HEADERS
    ${FIZZ_TEST_HEADERS}
    ${headers})
endforeach()

set(FIZZ_SOURCES
  crypto/Utils.cpp
  crypto/exchange/X25519.cpp
  crypto/aead/OpenSSLEVPCipher.cpp
  crypto/aead/IOBufUtil.cpp
  crypto/signature/Signature.cpp
  crypto/Sha256.cpp
  crypto/Sha384.cpp
  crypto/openssl/OpenSSLKeyUtils.cpp
  record/Types.cpp
  record/RecordLayer.cpp
  record/EncryptedRecordLayer.cpp
  record/PlaintextRecordLayer.cpp
  server/ServerProtocol.cpp
  server/CertManager.cpp
  server/State.cpp
  server/FizzServer.cpp
  server/TicketCodec.cpp
  server/CookieCipher.cpp
  server/ReplayCache.cpp
  protocol/AsyncFizzBase.cpp
  protocol/Types.cpp
  protocol/Exporter.cpp
  protocol/DefaultCertificateVerifier.cpp
  protocol/Events.cpp
  protocol/KeyScheduler.cpp
  protocol/Certificate.cpp
  extensions/secretlogging/LoggingKeyScheduler.cpp
  extensions/tokenbinding/Types.cpp
  extensions/tokenbinding/TokenBindingConstructor.cpp
  extensions/tokenbinding/TokenBindingClientExtension.cpp
  extensions/tokenbinding/Validator.cpp
  client/State.cpp
  client/ClientProtocol.cpp
  client/SynchronizedLruPskCache.cpp
  client/EarlyDataRejectionPolicy.cpp
)

add_library(fizz
  ${FIZZ_HEADERS}
  ${FIZZ_SOURCES}
)

if (BUILD_SHARED_LIBS)
  set_target_properties(fizz
    PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION 1)
endif()

get_filename_component(FIZZ_BASE_DIR ${CMAKE_SOURCE_DIR}/.. ABSOLUTE)

target_include_directories(
  fizz
  PUBLIC
    $<BUILD_INTERFACE:${FIZZ_BASE_DIR}>
    $<INSTALL_INTERFACE:${INCLUDE_INSTALL_DIR}>
    ${FOLLY_INCLUDE_DIR}
    ${Boost_INCLUDE_DIR}
    ${OPENSSL_INCLUDE_DIR}
  PRIVATE
    ${GLOG_INCLUDE_DIRS}
    ${GFLAGS_INCLUDE_DIRS}
    ${LIBEVENT_INCLUDE_DIRS}
    ${DOUBLE_CONVERSION_INCLUDE_DIRS}
)


target_link_libraries(fizz
  PUBLIC
    ${FOLLY_LIBRARIES}
    ${Boost_LIBRARIES}
    ${OPENSSL_LIBRARIES}
    ${SODIUM}
  PRIVATE
    ${GLOG_LIBRARIES}
    ${GFLAGS_LIBRARIES}
    ${LIBEVENT_LIBRARIES}
    ${DOUBLE_CONVERSION_LIBRARIES}
    ${CMAKE_DL_LIBS}
    ${LIBRT_LIBRARIES})

install(
  TARGETS fizz
  EXPORT fizz-exports
  DESTINATION ${LIB_INSTALL_DIR}
)

# We unfortunately cannot install fizz's headers with the install()
# statement above.  install(TARGETS) appears to only support installing
# PUBLIC_HEADER in a flat include directory, and not a deeper tree.
foreach(dir ${FIZZ_HEADER_DIRS})
  get_filename_component(PARENT_DIR "/${dir}" DIRECTORY)
  install(DIRECTORY ${dir} DESTINATION "${INCLUDE_INSTALL_DIR}/fizz${PARENT_DIR}"
          FILES_MATCHING PATTERN "*.h"
          PATTERN "test" EXCLUDE)
endforeach()

# Install CMake package configuration files for fizz
include(CMakePackageConfigHelpers)
configure_package_config_file(
  cmake/fizz-config.cmake.in
  fizz-config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_DIR}
  PATH_VARS
    INCLUDE_INSTALL_DIR
    CMAKE_INSTALL_DIR
)
install(
  FILES ${CMAKE_CURRENT_BINARY_DIR}/fizz-config.cmake
  DESTINATION ${CMAKE_INSTALL_DIR}
)
install(EXPORT fizz-exports
        FILE fizz-targets.cmake
        NAMESPACE fizz::
        DESTINATION ${CMAKE_INSTALL_DIR})

IF(CMAKE_CROSSCOMPILING)
   option(BUILD_TESTS "BUILD_TESTS" OFF)
ELSE(CMAKE_CROSSCOMPILING)
   option(BUILD_TESTS "BUILD_TESTS" ON)
ENDIF(CMAKE_CROSSCOMPILING)

if(BUILD_TESTS)
  enable_testing()

  find_package(GMock REQUIRED)
  if(NOT LIBGMOCK_FOUND)
    include(ExternalProject)

    # Download and install GoogleMock
    ExternalProject_Add(
        gtest
        GIT_REPOSITORY https://github.com/google/googletest.git
        GIT_TAG release-1.8.0
        PREFIX gtest
        # Disable install step
        INSTALL_COMMAND ""
        LOG_DOWNLOAD ON
        LOG_UPDATE 1
        LOG_CONFIGURE ON
        LOG_BUILD ON
        LOG_TEST 1
        LOG_INSTALL 1
    )

    # Create a libgmock target to be used as a dependency by test programs
    add_library(libgmock IMPORTED STATIC GLOBAL)
    add_dependencies(libgmock gtest)
    add_library(libgmock_main IMPORTED STATIC GLOBAL)
    add_dependencies(libgmock_main gtest)

    # Set gmock properties
    ExternalProject_Get_Property(gtest source_dir binary_dir)
    set_target_properties(libgmock PROPERTIES
        "IMPORTED_LOCATION" "${binary_dir}/googlemock/libgmock.a"
        "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
    )
    set_target_properties(libgmock_main PROPERTIES
        "IMPORTED_LOCATION" "${binary_dir}/googlemock/libgmock_main.a"
        "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
    )
    set(LIBGMOCK_LIBRARIES libgmock libgmock_main)
    set(LIBGMOCK_INCLUDE_DIR "${source_dir}/googlemock/include")
    set(LIBGTEST_INCLUDE_DIR "${source_dir}/googletest/include")
    set(LIBGTEST_INCLUDE_DIR "${source_dir}/googletest/include")
  endif()

  add_library(fizz_test_support
    crypto/aead/test/TestUtil.cpp
    crypto/test/TestUtil.cpp
    ${FIZZ_TEST_HEADERS})

  target_link_libraries(fizz_test_support
    PUBLIC
      fizz
  )

  add_library(fizz_test_main
    test/CMakeTestMain.cpp)

  target_link_libraries(fizz_test_main
    PUBLIC
      fizz
      fizz_test_support
      ${LIBGMOCK_LIBRARIES})

  target_include_directories(fizz_test_main
    PUBLIC
    ${LIBGMOCK_INCLUDE_DIR}
    ${LIBGTEST_INCLUDE_DIR})

  macro(add_gtest test_source test_name)
  add_executable(${test_name} ${test_source})
  target_include_directories(
    ${test_name} PUBLIC ${LIBGMOCK_INCLUDE_DIR} ${LIBGTEST_INCLUDE_DIR})
  target_link_libraries(${test_name}
    fizz
    fizz_test_support
    fizz_test_main
    ${LIBGMOCK_LIBRARIES})

  # GMOCK_MOCK_METHOD() will complain otherwise
  target_compile_options(${test_name}
    PRIVATE "-Wno-inconsistent-missing-override")

  add_test(${test_name} bin/${test_name})
  endmacro(add_gtest)

  add_gtest(client/test/SynchronizedLruPskCacheTest.cpp SyncronizedLruPskCacheTest)
  add_gtest(client/test/AsyncFizzClientTest.cpp AsyncFizzClientTest)
  add_gtest(client/test/ClientProtocolTest.cpp ClientProtocolTest)
  add_gtest(client/test/FizzClientTest.cpp FizzClientTest)
  add_gtest(crypto/aead/test/OpenSSLEVPCipherTest.cpp OpenSSLEVPCipherTest)
  add_gtest(crypto/aead/test/IOBufUtilTest.cpp IOBufUtilTest)
  add_gtest(crypto/exchange/test/X25519KeyExchangeTest.cpp X25519KeyExchangeTest)
  add_gtest(crypto/exchange/test/ECKeyExchangeTest.cpp ECKeyExchangeTest)
  add_gtest(crypto/openssl/test/OpenSSLKeyUtilsTest.cpp OpenSSLKeyUtilsTest)
  add_gtest(crypto/signature/test/RSAPSSSignatureTest.cpp RSAPSSSignatureTest)
  add_gtest(crypto/signature/test/ECSignatureTest.cpp ECSignatureTest)
  add_gtest(crypto/test/HkdfTest.cpp HkdfTest)
  add_gtest(crypto/test/KeyDerivationTest.cpp KeyDerivationTest)
  add_gtest(crypto/test/RandomGeneratorTest.cpp RandomGeneratorTest)
  add_gtest(crypto/test/UtilsTest.cpp UtilsTest)
  add_gtest(extensions/tokenbinding/test/TokenBindingConstructorTest.cpp TokenBindingConstructorTest)
  add_gtest(extensions/tokenbinding/test/ValidatorTest.cpp ValidatorTest)
  add_gtest(extensions/tokenbinding/test/TokenBindingServerExtensionTest.cpp TokenBindingServerExtensionTest)
  add_gtest(extensions/tokenbinding/test/TokenBindingTest.cpp TokenBindingTest)
  add_gtest(extensions/tokenbinding/test/TokenBindingClientExtensionTest.cpp TokenBindingClientExtensionTest)
  add_gtest(protocol/test/CertTest.cpp CertTest)
  add_gtest(protocol/test/FizzBaseTest.cpp FizzBaseTest)
  add_gtest(protocol/test/KeySchedulerTest.cpp KeySchedulerTest)
  add_gtest(protocol/test/DefaultCertificateVerifierTest.cpp DefaultCertificateVerifierTest)
  add_gtest(protocol/test/HandshakeContextTest.cpp HandshakeContextTest)
  add_gtest(protocol/test/ExporterTest.cpp ExporterTest)
  add_gtest(record/test/ExtensionsTest.cpp ExtensionsTest)
  add_gtest(record/test/EncryptedRecordTest.cpp EncryptedRecordTest)
  add_gtest(record/test/TypesTest.cpp TypesTest)
  add_gtest(record/test/HandshakeTypesTest.cpp HandshakeTypesTest)
  add_gtest(record/test/RecordTest.cpp RecordTest)
  add_gtest(record/test/PlaintextRecordTest.cpp PlaintextRecordTest)
  add_gtest(server/test/CertManagerTest.cpp CertManagerTest)
  add_gtest(server/test/CookieCipherTest.cpp CookieCipherTest)
  add_gtest(server/test/AeadTicketCipherTest.cpp AeadTicketCipherTest)
  add_gtest(server/test/AsyncFizzServerTest.cpp AsyncFizzServerTest)
  add_gtest(server/test/AeadCookieCipherTest.cpp AeadCookieCipherTest)
  add_gtest(server/test/TicketCodecTest.cpp TicketCodecTest)
  add_gtest(server/test/ServerProtocolTest.cpp ServerProtocolTest)
  add_gtest(server/test/NegotiatorTest.cpp NegotiatorTest)
  add_gtest(server/test/FizzServerTest.cpp FizzServerTest)
  add_gtest(test/AsyncFizzBaseTest.cpp AsyncFizzBaseTest)
  add_gtest(test/HandshakeTest.cpp HandshakeTest)
endif()

option(BUILD_EXAMPLES "BUILD_EXAMPLES" ON)

if(BUILD_EXAMPLES)
  add_executable(ClientSocket client/test/ClientSocket.cpp)
  target_link_libraries(ClientSocket fizz)
  add_executable(ServerSocket server/test/ServerSocket.cpp)
  target_link_libraries(ServerSocket fizz)
  add_executable(BogoShim test/BogoShim.cpp)
  target_link_libraries(BogoShim fizz)
endif()
