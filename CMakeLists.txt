cmake_minimum_required(VERSION 3.10)
project(EPlayerServer LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 99)
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

# ---- dependencies -----------------------------------------------------------
find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED)            # Crypto::MD5

find_path(MYSQL_INCLUDE_DIR mysql/mysql.h
  PATHS /usr/include /usr/local/include /usr/include/mariadb)
find_library(MYSQL_LIBRARY NAMES mysqlclient mariadb
  PATHS /usr/lib /usr/local/lib /usr/lib/x86_64-linux-gnu)
if(NOT MYSQL_INCLUDE_DIR OR NOT MYSQL_LIBRARY)
  message(FATAL_ERROR "libmysqlclient not found. On Debian/Ubuntu: sudo apt install libmysqlclient-dev")
endif()

# ---- third party (vendored) -------------------------------------------------
add_library(http_parser STATIC third_party/http_parser/http_parser.c)
target_include_directories(http_parser PUBLIC third_party/http_parser)

add_library(sqlite3 STATIC third_party/sqlite3/sqlite3.c)
target_include_directories(sqlite3 PUBLIC third_party)
target_link_libraries(sqlite3 PUBLIC Threads::Threads ${CMAKE_DL_LIBS})
target_compile_options(sqlite3 PRIVATE -w)

add_library(jsoncpp STATIC
  third_party/jsoncpp/json_reader.cpp
  third_party/jsoncpp/json_value.cpp
  third_party/jsoncpp/json_writer.cpp)
target_include_directories(jsoncpp PUBLIC third_party)

# ---- server -----------------------------------------------------------------
file(GLOB SERVER_SOURCES CONFIGURE_DEPENDS src/*.cpp)
add_executable(${PROJECT_NAME} ${SERVER_SOURCES})
target_include_directories(${PROJECT_NAME}
  PRIVATE include third_party ${MYSQL_INCLUDE_DIR})
target_link_libraries(${PROJECT_NAME}
  PRIVATE http_parser sqlite3 jsoncpp
          Threads::Threads OpenSSL::Crypto ${MYSQL_LIBRARY} ${CMAKE_DL_LIBS} m)
target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wno-unused-result)

# the logger writes to ./log relative to the working directory
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/log)

install(TARGETS ${PROJECT_NAME} RUNTIME DESTINATION bin)
