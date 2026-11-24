# This file has been generated with an AI (Mistral Vibe)

cmake_minimum_required(VERSION 3.14)
project(CppBaseWrapper C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 99)

include(FetchContent)

find_package(Threads REQUIRED)
find_package(ZLIB REQUIRED)
find_package(CURL REQUIRED)

# ---------------------------------------------------------------------------
# Dependencies, fetched and built from source via FetchContent
# ---------------------------------------------------------------------------

# --- librdkafka: Kafka C/C++ client, required from the system package ---
find_package(PkgConfig REQUIRED)
pkg_check_modules(PC_RDKAFKA_CPP REQUIRED IMPORTED_TARGET rdkafka++)
message(STATUS "Using system librdkafka ${PC_RDKAFKA_CPP_VERSION} (pkg-config rdkafka++)")
set(RDKAFKA_CPP_TARGET PkgConfig::PC_RDKAFKA_CPP)

# --- cppkafka: C++ wrapper around librdkafka (headers under cppkafka/).
#     Fetched and compiled directly; it links the system librdkafka above. ---
FetchContent_Declare(cppkafka
    GIT_REPOSITORY https://github.com/mfontanini/cppkafka.git
    GIT_TAG        v0.4.1
    GIT_SHALLOW    TRUE)
FetchContent_Populate(cppkafka)
file(GLOB CPPKAFKA_SOURCES
    ${cppkafka_SOURCE_DIR}/src/*.cpp
    ${cppkafka_SOURCE_DIR}/src/utils/*.cpp)
add_library(cppkafka SHARED ${CPPKAFKA_SOURCES})
target_include_directories(cppkafka PUBLIC
    ${cppkafka_SOURCE_DIR}/include
    ${CMAKE_BINARY_DIR}/deps-include   # provides <librdkafka/rdkafka.h>
)
target_include_directories(cppkafka PRIVATE ${cppkafka_SOURCE_DIR}/include/cppkafka)
target_link_libraries(cppkafka PUBLIC ${RDKAFKA_CPP_TARGET})

# --- jansson: JSON library required by avro-c (target: jansson) ---
set(JANSSON_BUILD_DOCS OFF CACHE BOOL "" FORCE)
set(JANSSON_EXAMPLES OFF CACHE BOOL "" FORCE)
set(JANSSON_BUILD_SHARED_LIBS ON CACHE BOOL "" FORCE)
FetchContent_Declare(jansson
    GIT_REPOSITORY https://github.com/akheron/jansson.git
    GIT_TAG        v2.14
    GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(jansson)

# --- avro: fetched only. Avro ships no root CMakeLists and its lang/c++
#     CMake requires Boost components (program_options, regex) that the
#     library itself does not use, so the C and C++ libs are compiled
#     directly from the fetched sources below. ---
FetchContent_Declare(avro
    GIT_REPOSITORY https://github.com/apache/avro.git
    GIT_TAG        release-1.11.3
    GIT_SHALLOW    TRUE)
FetchContent_Populate(avro)

# avro C (required by libserdes); glob the library sources and drop the CLI
# tools, which each have their own main()
set(AVRO_C_DIR ${avro_SOURCE_DIR}/lang/c/src)
file(GLOB AVRO_C_SOURCES ${AVRO_C_DIR}/*.c)
list(REMOVE_ITEM AVRO_C_SOURCES
    ${AVRO_C_DIR}/avroappend.c
    ${AVRO_C_DIR}/avrocat.c
    ${AVRO_C_DIR}/avromod.c
    ${AVRO_C_DIR}/avropipe.c
    ${AVRO_C_DIR}/schema_specific.c   # not part of the library (upstream AVRO_SRC)
)
add_library(avro-c SHARED ${AVRO_C_SOURCES})
# jansson's CMake target exports no include dirs, so add them explicitly
target_include_directories(avro-c PUBLIC
    ${AVRO_C_DIR}
    ${jansson_SOURCE_DIR}/src
    ${jansson_BINARY_DIR}/include
)
target_link_libraries(avro-c PUBLIC jansson ZLIB::ZLIB)

# staging dir so that <avro/Decoder.hh> etc. resolve (in the build tree the
# headers live in lang/c++/api without the avro/ prefix)
file(COPY ${avro_SOURCE_DIR}/lang/c++/api/ DESTINATION ${CMAKE_BINARY_DIR}/deps-include/avro)

# avro C++
set(AVRO_CPP_DIR ${avro_SOURCE_DIR}/lang/c++)
file(GLOB AVRO_CPP_SOURCES
    ${AVRO_CPP_DIR}/impl/*.cc
    ${AVRO_CPP_DIR}/impl/parsing/*.cc
    ${AVRO_CPP_DIR}/impl/json/*.cc
)
# avrogencpp.cc is the schema codegen tool (own main(), needs Boost.ProgramOptions)
list(REMOVE_ITEM AVRO_CPP_SOURCES ${AVRO_CPP_DIR}/impl/avrogencpp.cc)
add_library(avrocpp SHARED ${AVRO_CPP_SOURCES})
# AVRO_DYN_LINK = the shared-library variant of avro's export macros
target_compile_definitions(avrocpp PRIVATE AVRO_DYN_LINK)
target_include_directories(avrocpp PUBLIC
    ${CMAKE_BINARY_DIR}/deps-include
    ${AVRO_CPP_DIR}/api
)
# Boost headers come from the system; libboost_iostreams may only ship as a
# versioned .so without a dev symlink, so fall back to the versioned file
find_library(BOOST_IOSTREAMS_LIBRARY NAMES boost_iostreams)
if(NOT BOOST_IOSTREAMS_LIBRARY)
    file(GLOB BOOST_IOSTREAMS_LIBRARY
        /usr/lib/*/libboost_iostreams.so.*
        /usr/lib/libboost_iostreams.so.*)
    list(GET BOOST_IOSTREAMS_LIBRARY 0 BOOST_IOSTREAMS_LIBRARY)
endif()
target_link_libraries(avrocpp PUBLIC ${BOOST_IOSTREAMS_LIBRARY} ZLIB::ZLIB Threads::Threads)

# --- libserdes: Confluent Avro serdes (autotools-only upstream, so its
#     sources are compiled directly here; targets: serdes, serdes-cpp) ---
FetchContent_Declare(libserdes
    GIT_REPOSITORY https://github.com/confluentinc/libserdes.git
    GIT_TAG        v8.4.1
    GIT_SHALLOW    TRUE)
FetchContent_Populate(libserdes)
# serdes_int.h includes "../config.h", which autoconf would generate
file(WRITE ${libserdes_SOURCE_DIR}/config.h "/* empty config.h for the CMake build */\n")

# staging dirs so that <libserdes/serdes.h> and <libserdes/serdescpp.h> resolve
file(COPY ${libserdes_SOURCE_DIR}/src/ DESTINATION ${CMAKE_BINARY_DIR}/deps-include-serdes-c/libserdes)
file(COPY ${libserdes_SOURCE_DIR}/src-cpp/ DESTINATION ${CMAKE_BINARY_DIR}/deps-include-serdes-cpp/libserdes)

file(GLOB SERDES_C_SOURCES ${libserdes_SOURCE_DIR}/src/*.c)
add_library(serdes SHARED ${SERDES_C_SOURCES})
target_include_directories(serdes PUBLIC
    ${libserdes_SOURCE_DIR}/src
    ${CMAKE_BINARY_DIR}/deps-include-serdes-c
)
target_link_libraries(serdes PUBLIC avro-c CURL::libcurl Threads::Threads)

add_library(serdes-cpp SHARED ${libserdes_SOURCE_DIR}/src-cpp/Serdes.cpp)
target_include_directories(serdes-cpp PUBLIC
    ${libserdes_SOURCE_DIR}/src-cpp
    ${libserdes_SOURCE_DIR}/src
    ${CMAKE_BINARY_DIR}/deps-include-serdes-cpp
    ${CMAKE_BINARY_DIR}/deps-include
)
target_link_libraries(serdes-cpp PUBLIC serdes avrocpp)

# ---------------------------------------------------------------------------
# CppBaseWrapper
# ---------------------------------------------------------------------------

file(GLOB SOURCES
    "src/api/Traffic/*.cpp"
    "src/*/*.cpp"
    "src/*/*/*.cpp"
)

add_library(CppBaseWrapper SHARED ${SOURCES})

target_include_directories(CppBaseWrapper PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_SOURCE_DIR}/src/datamodel
    ${CMAKE_BINARY_DIR}/deps-include
    ${CMAKE_BINARY_DIR}/deps-include-serdes-c
    ${CMAKE_BINARY_DIR}/deps-include-serdes-cpp
    ${libserdes_SOURCE_DIR}/src
    ${libserdes_SOURCE_DIR}/src-cpp
)

target_link_libraries(CppBaseWrapper PUBLIC
    cppkafka
    ${RDKAFKA_CPP_TARGET}
    avrocpp
    avro-c
    serdes
    serdes-cpp
    ${CMAKE_DL_LIBS}
)
