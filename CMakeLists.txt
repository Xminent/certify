cmake_minimum_required(VERSION 3.8)
project(
  certify
  VERSION 0.1
  LANGUAGES CXX)

list(INSERT CMAKE_MODULE_PATH 0 ${PROJECT_SOURCE_DIR}/CMakeModules)

option(CERTIFY_BUILD_TESTS "Build tests" OFF)
option(CERTIFY_INSTALL "Install certify" OFF)

if(CMAKE_VERSION VERSION_LESS 3.11)
  # Latest FindBoost.cmake has likely been updated to detect Boost version not
  # yet released
  if(NOT EXISTS "${CMAKE_BINARY_DIR}/cmake/FindBoost.cmake")
    message(
      STATUS
        "Downloading FindBoost.cmake from https://gitlab.kitware.com/cmake/ release branch"
    )
    file(
      DOWNLOAD
      "https://gitlab.kitware.com/cmake/cmake/raw/release/Modules/FindBoost.cmake"
      "${CMAKE_BINARY_DIR}/cmake/FindBoost.cmake")
  endif()
  list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_BINARY_DIR}/cmake)
endif()

find_package(
  Boost 1.67
  COMPONENTS system filesystem date_time
  REQUIRED)

find_package(Threads)

find_package(OpenSSL REQUIRED)

add_library(core INTERFACE)

add_library(certify::core ALIAS core)

target_compile_features(core INTERFACE cxx_std_11)

target_include_directories(
  core INTERFACE $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
                 $<INSTALL_INTERFACE:include>)

if(MSVC)
  target_link_libraries(core INTERFACE Crypt32.lib)
endif()

if(APPLE)
  target_link_libraries(core INTERFACE "-framework CoreFoundation"
                                       "-framework Security")
  set_target_properties(core PROPERTIES LINK_FLAGS "-Wl,-F/Library/Frameworks")
endif()

target_link_libraries(
  core INTERFACE Boost::system Boost::filesystem Boost::date_time
                 Threads::Threads OpenSSL::SSL OpenSSL::Crypto)

include(CMakePackageConfigHelpers)
write_basic_package_version_file("certifyConfigVersion.cmake"
                                 COMPATIBILITY AnyNewerVersion)

if(CERTIFY_INSTALL)
  install(FILES "certifyConfig.cmake"
                "${CMAKE_CURRENT_BINARY_DIR}/certifyConfigVersion.cmake"
          DESTINATION lib/cmake/certify)

  install(
    DIRECTORY ${CMAKE_SOURCE_DIR}/include/
    DESTINATION include
    FILES_MATCHING
    PATTERN "*.hpp")

  install(
    TARGETS core
    EXPORT certifyTargets
    INCLUDES
    DESTINATION include)

  install(
    EXPORT certifyTargets
    FILE certifyTargets.cmake
    NAMESPACE certify::
    DESTINATION lib/cmake/certify)
endif()

include(CTest)

if(CERTIFY_BUILD_TESTS)
  find_package(Boost 1.67 REQUIRED COMPONENTS unit_test_framework)

  enable_testing()
  add_subdirectory(tests)
endif()
