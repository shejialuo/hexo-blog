---
title: 手把手教你写一个简单的SMTP服务器（五）
tags:
  - 技术
  - 教程
  - 学习
categories:
  - 手把手教你写一个简单的SMTP服务器
date: 2023-03-14 19:41:26
---

+ [手把手教你写一个简单的SMTP服务器（一）](https://luolibrary.com/2023/03/12/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E4%B8%80%EF%BC%89/)
+ [手把手教你写一个简单的SMTP服务器（二）](https://luolibrary.com/2023/03/14/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E4%BA%8C%EF%BC%89/)
+ [手把手教你写一个简单的SMTP服务器（三）](https://luolibrary.com/2023/03/14/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E4%B8%89%EF%BC%89/)
+ [手把手教你写一个简单的SMTP服务器（四）](https://luolibrary.com/2023/03/14/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E5%9B%9B%EF%BC%89/)
+ [手把手教你写一个简单的SMTP服务器（五）](https://luolibrary.com/2023/03/14/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%86%99%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84SMTP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%88%E4%BA%94%EF%BC%89/)

---

## Context

在教程四中，我们已经实现了状态机，接下来我们定义一个`Context`类，其仅仅简单地封装一下，维持一个`current`变量指向当前的状态即可。我们需要在`miniSMTPServer/context`目录下创建`context.hpp`和`context.cpp`文件并修改`CMakeLists.txt`。

```c++
// miniSMTPServer/context/context.hpp
#pragma once

#include "state.hpp"

#include <memory>

class Context {
private:
  std::unique_ptr<State> *current;

public:
  Context();
  std::string transitive(std::vector<std::string> &parameters);
  ~Context() = default;
};
```

```c++
// miniSMTPServer/context/context.cpp
#include "context.hpp"

#include "state.hpp"

Context::Context() { current = &States::idleState; }
std::string Context::transitive(std::vector<std::string> &parameters) {
  return (*current)->transitive(parameters, current);
}
```

```cmake
# miniSMTPServer/context/CMakeLists.txt
add_library(context STATIC state.cpp context.cpp)
...
```

可以看出`Context`类十分的简单，这是因为大部分工作都已经实现了。

## SMTP Server

现在，我们终于可以开始写我们的Server了，我们只需要完成一个简单得不能再简单的工作：按照空格分割字符串。然而你可能会想到最直接的方法就是定义一个`istringstream`。然而这是不正确的，因为可能会存在如下的情况：

```txt
MAIL shejialuo@gmail.com OK
MAIL  shejialuo@gmail.com NOT OK
```

ABNF中严格地定义了只允许拥有一个空格。而所有命令的参数最多不超过1个。我们采取的策略就可以非常简单了，直接以一个空格分割字符串即可。但是值得注意的是，我们必须删除接收到的字符串的最后两个，根据协议其最后两个字符为`CRLF`，即`\r\n`。

```c++
// miniSMTPServer/miniSMTPServer.cpp
std::vector<std::string> getParameters(std::string &request) {
  request.pop_back();
  request.pop_back();
  int split = 0;
  for (; split < request.size(); split++) {
    if (request[split] == ' ') {
      break;
    }
  }

  std::string command = request.substr(0, split);
  if (split != request.size()) {
    std::string parameter = request.substr(split + 1, request.size() - split - 1);
    return {command, parameter};
  }

  return {command};
}
```

最后我们只需要修改主函数就可以了，其逻辑十分的简单：

```c++
// miniSMTPServer/miniSMTPServer.cpp
int main() {
  std::cout << "Hello, This is a simple SMTP server\n";

  TCPSocket socket{};
  socket.set_reuseaddr();
  socket.bind();
  socket.listen();

  std::string request{};
  Context context{};

  while (true) {
    auto s = socket.accept();

    bool isDone = false;
    while (!isDone) {
      s.read(request);
      if (request.empty()) {
        std::cout << "S: Connection lost\n";
        s.close();
        break;
      }
      std::cout << "C: " << request;
      std::string parameter{};
      std::vector<std::string> parameters = getParameters(request);

      std::string result = context.transitive(parameters);

      s.write(result + "\r\n");
      std::cout << "S: " << result << std::endl;
      if (result.substr(0, 3) == "221") {
        s.close();
        isDone = true;
      }
    }
  }

  return 0;
}
```

最后我们需要修改`miniSMTPServer/CMakeLists.txt`文件如下所示：

```cmake
# miniSMTPServer/CMakeLists.txt
add_executable(miniSMTP miniSMTPServer.cpp)

set_target_properties(miniSMTP PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/)

target_include_directories(miniSMTP PRIVATE ./util ./context)

target_link_libraries(miniSMTP util context)

add_subdirectory(./util)
add_subdirectory(./context)
```

然后进行编译最终运行`./miniSMTP`。你可以尝试许多的输入输出，如果你发现了Bug，欢迎给仓库提交[miniSMTPServer](https://github.com/shejialuo/miniSMTPServer)

```sh
git checkout finish-all
```

## 总结

你可能会发现我们没有对邮件进行任何的处理，之所以我不打算做这样的处理是因为这并不是我们应该做的重点。如果你想保存用户发送的邮件，是一个非常简单的事情。你可能会想是不是需要在`transitive`函数中做这样的处理。实际上最合理的方式是新增加一个`doAction`的虚函数。对于每一个状态其存在一个处理函数：

+ `IdleState`：清空邮件信息的缓存。
+ `EhloState`：清空邮件信息的缓存。
+ `MailState`：记录邮件的发送人。
+ `RcptState`：记录邮件的接收人。
+ `DataStartState`：记录发送的信息。
+ `DataStartDone`：如果收到了`QUIT`命令，将邮件保存到硬盘。

希望能够阅读到这儿的你，或多或少能有所收获。这是我第一次尝试写一个教程，终于明白了写一个教程的艰辛。希望你能有一天用自己的知识帮助到他人。
