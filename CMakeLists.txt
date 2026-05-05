cmake_minimum_required(VERSION 3.10)
project(lab05)

option(BUILD_TESTS "Build tests" OFF)

add_subdirectory(banking)

if(BUILD_TESTS)
    enable_testing()
    
    # Подключаем GTest напрямую
    add_subdirectory(third-party/gtest)
    
    # Создаем тесты
    add_executable(check tests/test_account.cpp tests/test_transaction.cpp)
    target_link_libraries(check banking gtest gtest_main pthread)
    target_include_directories(check PRIVATE banking third-party/gtest/googletest/include)
    
    add_test(NAME check COMMAND check)
endif()

include(CPackConfig.cmake)
