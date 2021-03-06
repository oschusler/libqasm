cmake_minimum_required(VERSION 2.8)

enable_testing()

project("libQasm")

find_package(BISON 3.0)
find_package(FLEX 2.6)
find_package(PythonInterp 3 REQUIRED)

if(COMPILER_SUPPORTS_CXX11)
    set(CMAKE_REQUIRED_FLAGS "-std=c++11")
endif()

BISON_TARGET(
    MyParser
    grammar.y ${CMAKE_CURRENT_BINARY_DIR}/grammar.tab.c
    COMPILE_FLAGS "-Wall -d -t -g -r all"
)
FLEX_TARGET(MyScanner lex.l  ${CMAKE_CURRENT_BINARY_DIR}/lex.yy.c)
ADD_FLEX_BISON_DEPENDENCY(MyScanner MyParser)

include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/grammar.tab.c PROPERTIES LANGUAGE CXX)
set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/lex.yy.c PROPERTIES LANGUAGE CXX)

ADD_LIBRARY(lexgramstatic STATIC ${CMAKE_CURRENT_BINARY_DIR}/grammar.tab.c ${CMAKE_CURRENT_BINARY_DIR}/lex.yy.c)
set_target_properties(lexgramstatic PROPERTIES POSITION_INDEPENDENT_CODE ON)
ADD_LIBRARY(lexgram SHARED ${CMAKE_CURRENT_BINARY_DIR}/grammar.tab.c ${CMAKE_CURRENT_BINARY_DIR}/lex.yy.c)

set_property(TARGET lexgramstatic PROPERTY CXX_STANDARD 11)
set_property(TARGET lexgram PROPERTY CXX_STANDARD 11)

## Start of testing
include_directories (${CMAKE_CURRENT_SOURCE_DIR}/../tests/cpp/doctest)
add_compile_options(-std=c++11 -Wall -Wextra)
# Keep test files in a separate source directory called test
file(GLOB TEST_SRCS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} ../tests/cpp/*.cpp)

# Run through each source
foreach(testSrc ${TEST_SRCS})
    # Extract the filename without an extension (NAME_WE)
    get_filename_component(testName ${testSrc} NAME_WE)

    # Add compile target
    add_executable(${testName} ${testSrc})

    add_dependencies(${testName} lexgram)

    set_property(TARGET ${testName} PROPERTY CXX_STANDARD 11)

    # Link to Boost libraries AND targets and dependencies
    target_link_libraries(${testName} lexgram)

    # Move testing binaries into a testBin directory
    set_target_properties(${testName} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY  ${CMAKE_CURRENT_BINARY_DIR}/testBin)

    # Finally add it to test execution
    # Notice the WORKING_DIRECTORY and COMMAND
    file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/../tests/test_data/${testName}.qasm
        DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/testBin)
    add_test(NAME ${testName}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/testBin
        COMMAND ${CMAKE_CURRENT_BINARY_DIR}/testBin/${testName})
endforeach(testSrc)

## Start of SWIG module for the python interface
FIND_PACKAGE(SWIG REQUIRED)
INCLUDE(${SWIG_USE_FILE})

FIND_PACKAGE(PythonLibs REQUIRED)
INCLUDE_DIRECTORIES(${PYTHON_INCLUDE_PATH})

SET_SOURCE_FILES_PROPERTIES(libQasm_swig.i PROPERTIES CPLUSPLUS ON)
SET_PROPERTY(SOURCE libQasm_swig.i PROPERTY SWIG_FLAGS -castmode -modern -keyword)

if (${CMAKE_VERSION} VERSION_LESS "3.8.0")
    SWIG_ADD_MODULE(libQasm python libQasm_swig.i)
else()
    SWIG_ADD_LIBRARY(libQasm LANGUAGE python SOURCES libQasm_swig.i)
endif()

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    SWIG_LINK_LIBRARIES(libQasm lexgram)
    set_target_properties(_libQasm PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
else ()
    SWIG_LINK_LIBRARIES(libQasm ${PYTHON_LIBRARIES} lexgram)
endif()
