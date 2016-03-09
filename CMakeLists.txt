cmake_minimum_required (VERSION 2.8)
project(Boost.Config LANGUAGES C CXX)

set(Boost.Config_VERSION 1.61.0)

add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND} -VV -C ${CMAKE_CFG_INTDIR})

macro(add_test_executable TEST_NAME)
    add_executable (${TEST_NAME} EXCLUDE_FROM_ALL ${ARGN})
    if(WIN32)
        add_test(NAME ${TEST_NAME} WORKING_DIRECTORY ${LIBRARY_OUTPUT_PATH} COMMAND ${TEST_NAME}${CMAKE_EXECUTABLE_SUFFIX})
    else()
        add_test(NAME ${TEST_NAME} COMMAND ${TEST_NAME})
    endif()
    add_dependencies(check ${TEST_NAME})
    set_tests_properties(${TEST_NAME} PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")
endmacro(add_test_executable)

macro(add_test_compile TEST_NAME)
    add_library (${TEST_NAME} EXCLUDE_FROM_ALL ${ARGN})
    add_dependencies(check ${TEST_NAME})
endmacro(add_test_compile)

macro(add_test_compile_fail TEST_NAME)
    add_executable (${TEST_NAME} EXCLUDE_FROM_ALL ${ARGN})
    add_test(NAME ${TEST_NAME}
         COMMAND ${CMAKE_COMMAND} --build . --target ${TEST_NAME} --config $<CONFIGURATION>
         WORKING_DIRECTORY ${CMAKE_BINARY_DIR})
    set_tests_properties(${TEST_NAME} PROPERTIES WILL_FAIL TRUE)
endmacro(add_test_compile_fail)

include(CTest)

include_directories(include)

install (DIRECTORY include/boost DESTINATION include)
configure_file(boost_config.pc.in boost_config.pc)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/boost_config.pc DESTINATION lib/pkgconfig)

include(CheckLibraryExists)
include(CheckCXXCompilerFlag)
set(LIBS_TO_CHECK atomic pthread rt)
set(THREADED_LIBS "")
foreach(lib ${LIBS_TO_CHECK})
    check_library_exists(LIBRARY ${lib} VARIABLE HAS_LIB${lib})
    if(HAS_LIB${lib})
        set(THREADED_LIBS "${THREADED_LIBS} ${lib}")
    endif()
endforeach()

macro(add_threaded_test TEST_NAME TEST_FILE)
    add_test_executable(${TEST_NAME} ${TEST_FILE})
    target_link_libraries(${TEST_NAME} ${THREADED_LIBS})
endmacro(add_threaded_test)

add_test_executable(config_test_c test/config_test_c.c)
add_test_executable(config_info test/config_info.cpp)
add_test_executable(config_test test/config_test.cpp)
add_threaded_test(config_test_threaded test/config_test.cpp)

set(DISABLE_RTTI -fno-rtti)
if(MSVC)
    set(DISABLE_RTTI /GR-)
endif(MSVC)
add_threaded_test(config_test_no_rtti test/config_test.cpp)
set_target_properties(config_test_no_rtti PROPERTIES COMPILE_FLAGS ${DISABLE_RTTI})
add_test_executable(config_info_no_rtti test/config_info.cpp)
set_target_properties(config_info_no_rtti PROPERTIES COMPILE_FLAGS ${DISABLE_RTTI})

set(DISABLE_EXCEPTIONS -fno-exceptions)
if(MSVC)
    set(DISABLE_EXCEPTIONS "/EHs-c-")
endif(MSVC)
add_threaded_test(config_test_no_exceptions test/config_test.cpp)
set_target_properties(config_test_no_exceptions PROPERTIES COMPILE_FLAGS ${DISABLE_EXCEPTIONS})
add_test_executable(config_info_no_exceptions test/config_info.cpp)
set_target_properties(config_info_no_exceptions PROPERTIES COMPILE_FLAGS ${DISABLE_EXCEPTIONS})

add_test_compile_fail(test_thread_fail1 test/threads/test_thread_fail1.cpp)
add_test_compile_fail(test_thread_fail2 test/threads/test_thread_fail2.cpp)

add_test_executable(abi_test test/abi/abi_test.cpp test/abi/main.cpp)

set(IMPLICIT_FALLTHROUGH_FLAG "-std=c++11 -Wimplicit-fallthrough -Wall -Werror")
check_cxx_compiler_flag(${IMPLICIT_FALLTHROUGH_FLAG} HAS_IMPLICIT_FALLTHROUGH)
add_test_compile(boost_fallthrough_test test/boost_fallthrough_test.cpp)
if(HAS_IMPLICIT_FALLTHROUGH)
    set_target_properties(boost_fallthrough_test PROPERTIES COMPILE_FLAGS ${IMPLICIT_FALLTHROUGH_FLAG})
endif(HAS_IMPLICIT_FALLTHROUGH)
add_test_compile(helper_macro_test test/helper_macro_test.cpp)

add_test_compile(cstdint_include_test test/cstdint_include_test.cpp)
