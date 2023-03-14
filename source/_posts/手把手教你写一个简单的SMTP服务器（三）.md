---
title: 手把手教你写一个简单的SMTP服务器（三）
tags:
  - 技术
  - 教程
  - 学习
categories:
  - 手把手教你写一个简单的SMTP服务器
date: 2023-03-14 19:41:18
---

+ [手把手教你写一个简单的SMTP服务器（一）](https://luolibrary.com/2023/03/12/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E4%B8%80%EF%BC%89/)
+ [手把手教你写一个简单的SMTP服务器（二）](https://luolibrary.com/2023/03/14/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E4%BA%8C%EF%BC%89/)
+ [手把手教你写一个简单的SMTP服务器（三）](https://luolibrary.com/2023/03/14/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E4%B8%89%EF%BC%89/)
+ [手把手教你写一个简单的SMTP服务器（四）](https://luolibrary.com/2023/03/14/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E5%9B%9B%EF%BC%89/)
+ [手把手教你写一个简单的SMTP服务器（五）](https://luolibrary.com/2023/03/14/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E4%BA%94%EF%BC%89/)

---

在教程二中我们已经完成了最重要的工作，构建了最基本的虚基类`State`。回顾在教程二中我们对于状态的定义，我们拟定义如下6个类来表征状态机中的状态：

1. `IdleState`
2. `EhloState`
3. `MailState`
4. `RcptState`
5. `DataStartState`
6. `DataDoneState`

```c++
// miniSMTPServer/context/state.hpp
class IdleState : public State {
public:
  IdleState();
  std::string transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) override;
  ~IdleState() override = default;
};

class EhloState : public State {
public:
  EhloState();
  std::string transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) override;
  ~EhloState() override = default;
};

class MailState : public State {
public:
  MailState();
  std::string transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) override;
  ~MailState() override = default;
};

class RcptState : public State {
public:
  RcptState();
  std::string transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) override;
  ~RcptState() override = default;
};

class DataStartState : public State {
public:
  DataStartState();
  std::string transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) override;
  ~DataStartState() override = default;
};

class DataDoneState : public State {
public:
  DataDoneState();
  std::string transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) override;
  ~DataDoneState() override = default;
};
```

然后我们需要实现其方法，在现在这个阶段，我们当让其做空操作，或者返回默认值。

```c++
// miniSMTPServer/context/state.cpp

// TODO: add implementation for IdleState
IdleState::IdleState() {}
std::string IdleState::transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) { return {}; }

// TODO: add implementation for EhloState
EhloState::EhloState() {}
std::string EhloState::transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) { return {}; }

// TODO: add implementation for MailState
MailState::MailState() {}
std::string MailState::transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) { return {}; }

// TODO: add implementation for RcptState
RcptState::RcptState() {}
std::string RcptState::transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) { return {}; }

// TODO: add implementation for DataStartState
DataStartState::DataStartState() {}
std::string DataStartState::transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) {
  return {};
}

// TODO: add implementation for DataDoneState
DataDoneState::DataDoneState() {}
std::string DataDoneState::transitive(std::vector<std::string> &parameters, std::unique_ptr<State> *&current) {
  return {};
}
```

根据教程二中设计，我们可以知道当服务器端收到客户端的命令时，其会调用`transitive`方法进行状态的转换，一个思路是我们可以使用`make_unique`方法构造一个完整的新类来表述状态，但是这样不是很高效。为了提高效率，我们应该使用全局的生命周期的变量，因此我们可以构造一个新类`States`包含指向6个已经存在的类的指针，我们使用`unique_ptr`管理这些指针，从而避免内存的泄漏。

```c++
// miniSMTPServer/context/state.hpp
struct States {
  static std::unique_ptr<State> idleState;
  static std::unique_ptr<State> ehloState;
  static std::unique_ptr<State> mailState;
  static std::unique_ptr<State> rcptState;
  static std::unique_ptr<State> dataStartState;
  static std::unique_ptr<State> dataDoneState;
};
```

```c++
// miniSMTPServer/context/state.cpp
std::unique_ptr<State> States::idleState = std::make_unique<IdleState>();
std::unique_ptr<State> States::ehloState = std::make_unique<EhloState>();
std::unique_ptr<State> States::mailState = std::make_unique<MailState>();
std::unique_ptr<State> States::rcptState = std::make_unique<RcptState>();
std::unique_ptr<State> States::dataStartState = std::make_unique<DataStartState>();
std::unique_ptr<State> States::dataDoneState = std::make_unique<DataDoneState>();
```

完成了这一步，我们就可以修改`State`类的`transitiveFromQuit`方法：

```c++
// miniSMTPServer/context/state.cpp
std::string State::transitiveFromQuit(std::unique_ptr<State> *&current) {
  current = &States::idleState;
  return "221" + codeToMessages["221"];
}
```

至此，我们完成了我们的状态机的雏形。同时你也可以通过执行命令`git checkout all-states`获得上面的代码。

## IdleState

当状态机一启动时，其应该位于`IdleState`状态，其能够接收4个命令：`RSET`，`NOOP`，`QUIT`以及`EHLO`命令。其中`NOOP`和`QUIT`已经统一处理了，我们并不需要关心。在后面的状态中，我们会忽略这两个命令。对于`RSET`命令来说，其在这个状态没有任何的作用（严格来说，不是没有作用，而是从`IdleState`转化为`IdleState`）。对于`EHLO`命令来说，其应该从`IdleState`状态转化为`EhloState`。在完成这个工作之前，我们首先使用ABNF范式定义`RSET`等命令的请求和响应，为了简单起见，本教程直接设置了一个静态的"127.0.0.1"。

```ABNF
rset-request = "RSET" CRLF
rset-ok-response = "250" SP "Requested mail action okay, completed" CRLF

noop-request = "NOOP" CRLF
noop-ok-response = "250" SP "Requested mail action okay, completed" CRLF

quit-request = "QUIT" CRLF
quit-ok-response = "221" SP "Service closing transmission channel"

ehlo-request = "EHLO" SP "127.0.0.1" CRLF
ehlo-ok-response = "250" SP "Requested mail action okay, completed" CRLF
```

因此，我们首先更新`State`基类中的`isCorrectParameters`方法。其操作很简单，对于`NOOP`，`QUIT`和`RSET`而言，这些命令都不需要任何的参数。而对于`EHLO`命令，我们直接严格按照ABNF定义即可。

```c++
std::optional<std::string> State::isCorrectParameters(std::vector<std::string> &parameters) {
  std::string &command = parameters[0];
  if (command == "NOOP" || command == "QUIT" || command == "RSET") {
    if (parameters.size() != 1) {
      return "501 " + codeToMessages["501"];
    }
  } else if (parameters[0] == "EHLO") {
    if (parameters.size() != 2 || parameters[1] != "127.0.0.1") {
      return "501 " + codeToMessages["501"];
    }
  }

  return std::nullopt;
}
```

你可能觉得这个功能很简单，但是就我个人而言，我认为此处我们应该写一点单元测试保证这个函数的正确性。显然，用cpp写单元测试也是相当的繁琐。首先，你需要在`miniSMTPServer/context`目录下创建一个新的目录`tests`。然后在`tests`目录下添加如下的文件：

+ `CMakeLists.txt`
+ `stateTest.cpp`

对于位于`miniSMTPServer/context/tests`目录下的`CMakeLists.txt`，添加如下的代码：

```cmake
# miniSMTPServer/context/tests/CMakeLists.txt
enable_testing()

add_executable(
  stateTest
  stateTest.cpp
)

target_include_directories(stateTest PRIVATE ../)

target_link_libraries(
  stateTest
  context
  GTest::gtest_main
)

include(GoogleTest)
gtest_discover_tests(stateTest)
```

然后修改位于`miniSMTPServer/context`目录下的`CMakeLists.txt`文件，添加如下的代码：

```cmake
# miniSMTPServer/context/CMakeLists.txt
...
add_subdirectory(./tests)
```

然后我们就可以开始写单元测试了，我们目前的单元测试主要关心`State::isCorrectParameters`方法。

```c++
// miniSMTPServer/context/tests/stateTest.cpp
#include "state.hpp"

#include <gtest/gtest.h>
#include <memory>
#include <unordered_map>
#include <vector>

static std::unordered_map<std::string, std::string> codeToMessages{
    {"220", "Service ready"},
    {"221", "Service closing transmission channel"},
    {"250", "Requested mail action okay, completed"},
    {"354", "Start mail input end <CRLF>.<CRLF>"},
    {"500", "Syntax error, command unrecognized"},
    {"501", "Syntax error in parameters or arguments"},
    {"503", "Bad sequence of commands"},
};

TEST(State, isCorrectParametersNOOP) {
  std::vector<std::vector<std::string>> tests{
      {"NOOP", "param1"},
      {"NOOP", "param1", "param2"},
      {"NOOP", "12"},
      {"NOOP", "3"},
  };

  auto state = std::make_unique<IdleState>();

  for (auto &&test : tests) {
    auto result = state->isCorrectParameters(test);
    ASSERT_TRUE(result.has_value());
    ASSERT_EQ(result.value(), "501 " + codeToMessages["501"]);
  }

  std::vector<std::string> successful{"NOOP"};

  ASSERT_FALSE(state->isCorrectParameters(successful).has_value());
}

TEST(State, isCorrectParametersQUIT) {
  std::vector<std::vector<std::string>> tests{
      {"QUIT", "NOOP"},
      {"QUIT", "NOOP", "EHLO"},
      {"QUIT", "12", "13", "14", "15"},
      {"QUIT", "3", "4", "11111", "22"},
      {"QUIT", "MAIL", "RCPT", "11111", "22"},
  };

  auto state = std::make_unique<IdleState>();

  for (auto &&test : tests) {
    auto result = state->isCorrectParameters(test);
    ASSERT_TRUE(result.has_value());
    ASSERT_EQ(result.value(), "501 " + codeToMessages["501"]);
  }

  std::vector<std::string> successful{"QUIT"};

  ASSERT_FALSE(state->isCorrectParameters(successful).has_value());
}

TEST(State, isCorrectParametersRSET) {
  std::vector<std::vector<std::string>> tests{
      {"RSET", "NOOP"},
      {"RSET", "NOOP", "EHLO"},
      {"RSET", "12", "13", "14", "15"},
      {"RSET", "3", "4", "11111", "22"},
      {"RSET", "MAIL", "RCPT", "11111", "22"},
  };

  auto state = std::make_unique<IdleState>();

  for (auto &&test : tests) {
    auto result = state->isCorrectParameters(test);
    ASSERT_TRUE(result.has_value());
    ASSERT_EQ(result.value(), "501 " + codeToMessages["501"]);
  }

  std::vector<std::string> successful{"RSET"};

  ASSERT_FALSE(state->isCorrectParameters(successful).has_value());
}

TEST(State, isCorrectParametersEHLO) {
  std::vector<std::vector<std::string>> tests{
      {"EHLO"},
      {"EHLO", "127.0.0.2"},
      {"EHLO", "127.0.1.1"},
      {"EHLO", "NOOP", "EHLO"},
      {"EHLO", "12", "13", "14", "15"},
      {"EHLO", "3", "4", "11111", "22"},
      {"EHLO", "MAIL", "RCPT", "11111", "22"},
  };

  auto state = std::make_unique<IdleState>();

  for (auto &&test : tests) {
    auto result = state->isCorrectParameters(test);
    ASSERT_TRUE(result.has_value());
    ASSERT_EQ(result.value(), "501 " + codeToMessages["501"]);
  }

  std::vector<std::string> successful{"EHLO", "127.0.0.1"};

  ASSERT_FALSE(state->isCorrectParameters(successful).has_value());
}
```

上述代码提供的单元测试十分简单。不需要进行讲解，从后面开始，我们会以测试驱动来撰写代码，也就是我们先写测试文件再写相应的功能，当我们的代码能够通过测试后也就证明我们的代码是正确的。

现在我们要开始实现`IdleState`类中的方法。首先我们应该思考`IdleState`中允许存在什么命令，由上面的讲述可知，`IdleState`允许的命令与`State`中的`allowed`一致，所以对其构造函数我们可以不做任何的处理，那么关键的地方就在于`transitive`方法的实现。我们首先实现如下的单元测试的代码：

```c++
// miniSMTPServer/context/tests/stateTest.cpp
#include <utility>
...
TEST(State, IdleStateTransitive) {
  std::vector<std::vector<std::string>> tests{
      {"RSET"},
      {"NOOP"},
      {"QUIT"},
      {"EHLO", "127.0.0.1"},
  };

  std::vector<std::pair<std::string, std::unique_ptr<State> *>> expects{
      {"250 " + codeToMessages["250"], &States::idleState},
      {"250 " + codeToMessages["250"], &States::idleState},
      {"221 " + codeToMessages["221"], &States::idleState},
      {"250 " + codeToMessages["250"], &States::ehloState},
  };

  for (int i = 0; i < tests.size(); ++i) {
    auto state = std::make_unique<IdleState>();
    std::unique_ptr<State> *current = &States::idleState;
    std::string result = state->transitive(tests[0], current);
    EXPECT_EQ(result, expects[i].first);
    EXPECT_EQ(current, expects[i].second);
  }
}
```

测试的代码很简单，你可以编译然后使用`ctest --output-on-failure`可以发现很多错误，那么就让我们来更新`IdleState::transitive`方法，其更新的很简单，首先使用基类的方法`transitiveHelper`。然后只需要处理`EHLO`命令即可，因为`QUIT`和`NOOP`命令已经在`transitiveHelper`方法实现了，而`RSET`方法对于`idleState`没有任何作用。同时我也添加了许多其他测试。

```c++
// miniSMTPServer/context/tests/stateTest.cpp
TEST(State, IdleStateTransitive) {
  std::vector<std::vector<std::string>> tests{
      {"RSET"},
      {"NOOP"},
      {"QUIT"},
      {"EHLO", "127.0.0.1"},
      {"RSTE"},
      {"NOOQ"},
      {"RSET", "NOOP"},
      {"NOOP", "NOOP"},
      {"QUIT", "QUIT"},
      {"EHLO", "127.0.1.1"},
      {"DATA"},
  };

  std::vector<std::pair<std::string, std::unique_ptr<State> *>> expects{
      {"250 " + codeToMessages["250"], &States::idleState},
      {"250 " + codeToMessages["250"], &States::idleState},
      {"221 " + codeToMessages["221"], &States::idleState},
      {"250 " + codeToMessages["250"], &States::ehloState},
      {"500 " + codeToMessages["500"], &States::idleState},
      {"500 " + codeToMessages["500"], &States::idleState},
      {"501 " + codeToMessages["501"], &States::idleState},
      {"501 " + codeToMessages["501"], &States::idleState},
      {"501 " + codeToMessages["501"], &States::idleState},
      {"501 " + codeToMessages["501"], &States::idleState},
      {"503 " + codeToMessages["503"], &States::idleState},
  };

  for (int i = 0; i < tests.size(); ++i) {
    auto state = std::make_unique<IdleState>();
    std::unique_ptr<State> *current = &States::idleState;
    std::string result = state->transitive(tests[i], current);
    EXPECT_EQ(result, expects[i].first);
    EXPECT_EQ(current, expects[i].second);
  }
}
```

再完成了测试代码的编写后，我们开始修改`IdleState::transitive`方法：

```c++
// miniSMTPServer/context/state.cpp
std::optional<std::string> State::isCorrectParameters(std::vector<std::string> &parameters) {
  std::string &command = parameters[0];
  if (command == "NOOP" || command == "QUIT" || command == "RSET") {
    if (parameters.size() != 1) {
      return "501 " + codeToMessages["501"];
    }
  } else if (parameters[0] == "EHLO") {
    if (parameters.size() != 2 || parameters[1] != "127.0.0.1") {
      return "501 " + codeToMessages["501"];
    }
  }

  return std::nullopt;
}
```

然后我们编译代码运行测试:

```sh
cd build && make -j12 && ctest --output-on-failure
```

其结果如下：

```txt
80% tests passed, 1 tests failed out of 5

Total Test time (real) =   0.01 sec

The following tests FAILED:
          5 - State.IdleStateTransitive (Failed)
Errors while running CTest
```

产生这样的结果并不是我们代码的问题，而是由于`commands`变量的定义在教程二中没有完全定义，因此修改其定义如下：

```c++
// miniSMTPServer/context/state.cpp
static std::unordered_set<std::string> commands{"EHLO", "MAIL", "RCPT", "RSET", "NOOP", "QUIT", "DATA", "."};
```

然后我们重新编译代码运行测试:

```sh
cd build && make -j12 && ctest --output-on-failure
```

你会发现我们能够通过所有的测试。

## 小结

本节我们主要完善了我们的状态机并实现了第一个状态`IdleState`的操作，并通过测试驱动的方式写出了我们的代码。在后续的教程中我们将继续完善其他的状态操作。如果你产生了任何的问题，你可以运行如下的命令：

```sh
git checkout idle-state
```
