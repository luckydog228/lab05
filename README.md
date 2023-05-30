# lab05
[![Coverage Status](https://coveralls.io/repos/github/luckydog228/lab05/badge.svg?branch=refs/heads/main)](https://coveralls.io/github/luckydog228/lab05?branch=refs/heads/main)
##Task1

connect gtest
```
$ mkdir third-party
$ git submodule add https://github.com/google/googletest third-party/gtest
$ cd third-party/gtest && git checkout release-1.8.1 && cd ../..
```
```
$ git add third-party
$ git commit -m "added gtest framework"
$ git push origin main
```

CMakeList for banking

```
project(banking_lib)

if (NOT TARGET libbanking)
    add_library(libbanking STATIC
        /Account.cpp
        /Transaction.cpp
    )

    install(TARGETS libbanking
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
    )
endif(NOT TARGET libbanking)

include_directories()
```
push it

CMakeLists for test
```
cmake_minimum_required(VERSION 3.4)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TEST "Build tests" OFF)

project(banking)

add_library(banking STATIC banking/Account.cpp banking/Transaction.cpp)
target_include_directories(banking PUBLIC banking/)

if(BUILD_TESTS)
  enable_testing()
  add_subdirectory(third-party/gtest)
  file(GLOB BANKING_TEST_SOURCES tests/tests.cpp)
  add_executable(check tests/tests.cpp)
  target_link_libraries(check banking gtest_main gmock_main)
  add_test(NAME check COMMAND check)
endif()
```

push it

##Task2

tests.cpp
```
#include "Account.h"
#include "Transaction.h"

#include <gtest/gtest.h>
#include <gmock/gmock.h>

class AccountMock : public Account {
public:
    AccountMock(int id, int balance) : Account(id, balance) {}
    MOCK_CONST_METHOD0(GetBalance, int());
    MOCK_METHOD1(ChangeBalance, void(int diff));
    MOCK_METHOD0(Lock, void());
    MOCK_METHOD0(Unlock, void());
};
class TransactionMock : public Transaction {
public:
    MOCK_METHOD3(Make, bool(Account& from, Account& to, int sum));
};

TEST(Account, Mock) {
    AccountMock acc(1, 100);
    EXPECT_CALL(acc, GetBalance()).Times(1);
    EXPECT_CALL(acc, ChangeBalance(testing::_)).Times(2);
    EXPECT_CALL(acc, Lock()).Times(2);
    EXPECT_CALL(acc, Unlock()).Times(1);
    acc.GetBalance();
    acc.ChangeBalance(100); // throw
    acc.Lock();
    acc.ChangeBalance(100);
    acc.Lock(); // throw
    acc.Unlock();
}

TEST(Account, SimpleTest) {
    Account acc(1, 100);
    EXPECT_EQ(acc.id(), 1);
    EXPECT_EQ(acc.GetBalance(), 100);
    EXPECT_THROW(acc.ChangeBalance(100), std::runtime_error);
    EXPECT_NO_THROW(acc.Lock());
    acc.ChangeBalance(100);
    EXPECT_EQ(acc.GetBalance(), 200);
    EXPECT_THROW(acc.Lock(), std::runtime_error);
}

TEST(Transaction, Mock) {
    TransactionMock tr;
    Account ac1(1, 50);
    Account ac2(2, 500);
    EXPECT_CALL(tr, Make(testing::_, testing::_, testing::_))
    .Times(6);
    tr.set_fee(100);
    tr.Make(ac1, ac2, 199);
    tr.Make(ac2, ac1, 500);
    tr.Make(ac2, ac1, 300);
    tr.Make(ac1, ac1, 0); // throw
    tr.Make(ac1, ac2, -1); // throw
    tr.Make(ac1, ac2, 99); // throw
}

TEST(Transaction, SimpleTest) {
    Transaction tr;
    Account ac1(1, 50);
    Account ac2(2, 500);
    tr.set_fee(100);
    EXPECT_EQ(tr.fee(), 100);
    EXPECT_THROW(tr.Make(ac1, ac1, 0), std::logic_error);
    EXPECT_THROW(tr.Make(ac1, ac2, -1), std::invalid_argument);
    EXPECT_THROW(tr.Make(ac1, ac2, 99), std::logic_error);
    EXPECT_FALSE(tr.Make(ac1, ac2, 199));
    EXPECT_FALSE(tr.Make(ac2, ac1, 500));
    EXPECT_FALSE(tr.Make(ac2, ac1, 300));
}
```

push it

Action.yml

```
name: CMake

on:
 push:
  branches: [main]
 pull_request:
  branches: [main]

jobs:
 build_Linux:

  runs-on: ubuntu-latest

  steps:
  - uses: actions/checkout@v3

  - name: Adding gtest
    run: git clone https://github.com/google/googletest.git third-party/gtest -b release-1.11.0

  - name: Install lcov
    run: sudo apt-get install -y lcov

  - name: Config banking with tests
    run: cmake -H. -B ${{github.workspace}}/build -DBUILD_TESTS=ON

  - name: Build banking
    run: cmake --build ${{github.workspace}}/build

  - name: Run tests
    run: |
      build/check
      cmake --build ${{github.workspace}}/build --target test -- ARGS=--verbose
```
push it

##Task3

CMakeLists.txt new

```
cmake_minimum_required(VERSION 3.4)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TEST "Build tests" OFF)

if(BUILD_TESTS)
  add_compile_options(--coverage)
endif()

option(COVERAGE "Check coverage" ON)

project(banking)

add_library(banking STATIC banking/Account.cpp banking/Transaction.cpp)
target_include_directories(banking PUBLIC banking/)

target_link_libraries(banking gcov)

if(BUILD_TESTS)
  enable_testing()
  add_subdirectory(third-party/gtest)
  file(GLOB BANKING_TEST_SOURCES tests/tests.cpp)
  add_executable(check tests/tests.cpp)
  target_link_libraries(check banking gtest_main gmock_main)
  add_test(NAME check COMMAND check)
endif()

if (COVERAGE)
	target_compile_options(check PRIVATE --coverage)
	target_link_libraries(check --coverage)
endif()
```
push it

Action.yml new
```
name: CMake

on:
 push:
  branches: [main]
 pull_request:
  branches: [main]

jobs:
 build_Linux:

  runs-on: ubuntu-latest

  steps:
  - uses: actions/checkout@v3

  - name: Adding gtest
    run: git clone https://github.com/google/googletest.git third-party/gtest -b release-1.11.0

  - name: Install lcov
    run: sudo apt-get install -y lcov

  - name: Config banking with tests
    run: cmake -H. -B ${{github.workspace}}/build -DBUILD_TESTS=ON

  - name: Build banking
    run: cmake --build ${{github.workspace}}/build

  - name: Run tests
    run: |
      build/check
      cmake --build ${{github.workspace}}/build --target test -- ARGS=--verbose

  - name: Do lcov stuff
    run: lcov -c -d build/CMakeFiles/banking.dir/banking/ --include *.cpp --output-file ./coverage/lcov.info

  - name: Publish to coveralls.io
    uses: coverallsapp/github-action@v1.1.2
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
```
push it

Registering on the website https://coveralls.io
