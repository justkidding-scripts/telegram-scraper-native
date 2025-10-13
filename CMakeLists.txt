cmake_minimum_required(VERSION 3.20)
project(telegram_scraper_native CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Compiler optimizations for release
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG -flto -march=native -mtune=native")
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -fsanitize=address")

# Find required packages
find_package(PkgConfig REQUIRED)
find_package(Threads REQUIRED)

# SQLite3
find_path(SQLITE3_INCLUDE_DIR sqlite3.h)
find_library(SQLITE3_LIBRARIES sqlite3)

# Check for nlohmann/json
find_path(NLOHMANN_JSON_INCLUDE_DIR nlohmann/json.hpp
 PATHS /usr/include /usr/local/include /opt/homebrew/include
)

if(NOT NLOHMANN_JSON_INCLUDE_DIR)
 message(STATUS "nlohmann/json not found, will download...")
 include(FetchContent)
 FetchContent_Declare(
 nlohmann_json
 GIT_REPOSITORY https/github.com/nlohmann/json.git
 GIT_TAG v3.11.3
 )
 FetchContent_MakeAvailable(nlohmann_json)
endif()

# Build Rust library first
set(RUST_TARGET_DIR "${CMAKE_CURRENT_SOURCE_DIR}/target/release")
set(RUST_LIB "${RUST_TARGET_DIR}/libtelegram_scraper_core.a")

# Custom command to build Rust library
add_custom_command(
 OUTPUT ${RUST_LIB}
 COMMAND cargo build --release
 WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
 COMMENT "Building Rust library..."
 VERBATIM
)

add_custom_target(rust_lib ALL DEPENDS ${RUST_LIB})

# Create executable
add_executable(telegram_scraper_native src/main.cpp)

# Add dependencies
add_dependencies(telegram_scraper_native rust_lib)

# Include directories
target_include_directories(telegram_scraper_native PRIVATE
 ${CMAKE_CURRENT_SOURCE_DIR}/src
 ${SQLITE3_INCLUDE_DIR}
)

if(NLOHMANN_JSON_INCLUDE_DIR)
 target_include_directories(telegram_scraper_native PRIVATE ${NLOHMANN_JSON_INCLUDE_DIR})
else()
 target_link_libraries(telegram_scraper_native PRIVATE nlohmann_json::nlohmann_json)
endif()

# Link libraries
target_link_libraries(telegram_scraper_native PRIVATE
 ${RUST_LIB}
 ${SQLITE3_LIBRARIES}
 Threads::Threads
 ${CMAKE_DL_LIBS}
 m # Math library
)

# Platform-specific libraries
if(UNIX AND NOT APPLE)
 target_link_libraries(telegram_scraper_native PRIVATE rt)
endif()

# Install target
install(TARGETS telegram_scraper_native
 RUNTIME DESTINATION bin
)

# Create build script
configure_file(
 "${CMAKE_CURRENT_SOURCE_DIR}/build.sh.in"
 "${CMAKE_CURRENT_SOURCE_DIR}/build.sh"
 @ONLY
)

# Package configuration
set(CPACK_PACKAGE_NAME "TelegramScraperNative")
set(CPACK_PACKAGE_VERSION "2.0.0")
set(CPACK_PACKAGE_DESCRIPTION "High-performance Telegram scraper - C++/Rust hybrid")
set(CPACK_GENERATOR "DEB;RPM;TGZ")
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libsqlite3-0")
set(CPACK_RPM_PACKAGE_REQUIRES "sqlite")

include(CPack)