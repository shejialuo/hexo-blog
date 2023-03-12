---
title: 手把手教你写一个简单的SMTP服务器（一）
tags:
  - 技术
  - 教程
  - 学习
categories:
  - 手把手教你写一个简单的SMTP服务器
date: 2023-03-12 17:15:26
---


本系列的教程拟使用现代C++一步一步地实现一个简单的SMTP服务器，其主要目的在于国内目前计算机网络教学多是针对于协议本身的教学而忽略了实践。[Simple Mail Transfer Protocol (SMTP)](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol)是应用层中较为简单的协议，十分适合用来进行实践。在这里，借用费曼的话：

> What I cannot create, I do not understand.

然而，完整的SMTP协议也是相当繁琐的。故本教程会对完整的SMTP协议进行简化，并假设SMTP服务器运行在本地的环境，不需要扩展以支持通过Internet的TCP连接进而能用在现实世界中。服务器所有的通信都应包含在本地回环（local loopback）中。

在开始之前，你应该对SMTP协议有一些基本的了解，并对C++的现代特性有所了解，如果你对此没有把握，你可以参考互联网的资料进行学习。

## 假设

为了让教程聚焦于核心的部分，本教程做出了以下的前提条件：

+ SMTP服务器仅只需要运行在本地，所有的通信都应包含在本地回环中。
+ 对于每一次客户端的请求，SMTP服务器仅只产生一个响应。且只有客户端接收了服务器的相应，才能够再次发送请求。
+ SMTP服务器仅支持以下的命令：
  + `EHLO`
  + `MAIL`
  + `RCPT`
  + `DATA`
  + `RSET`
  + `NOOP`
  + `QUIT`

## 基础知识

+ [扩充巴科斯范式（ABNF）](https://zh.wikipedia.org/wiki/%E6%89%A9%E5%85%85%E5%B7%B4%E7%A7%91%E6%96%AF%E8%8C%83%E5%BC%8F)：本教程将采用ABNF来定义通信的协议。其思路很容易理解，读者可参考链接有一个初步的认识。
+ Socket网络编程基础：本教程拟在Linux环境下实现，读者需掌握Socket的基本使用。

## 初始代码

不幸的是，`cpp`的编译构建系统远不如其他语言方便，为了减少读者的心智负担，本教程提供了最初的代码。读者可以通过以下的命令下载初始代码：

```sh
git clone https://github.com/shejialuo/miniSMTPServer

git checkout start-code
```

用你最喜欢的编辑器或者IDE打开，你应该看到如下的目录结构：

```txt
├── CMakeLists.txt
├── miniSMTPServer
│   ├── CMakeLists.txt
│   └── miniSMTPServer.cpp
└── README.md
```

然后你可以执行如下的命令进行编译并生成可执行文件（在这个过程中，你需要使用代理下载第三方库）：

```sh
mkdir build && cd build && cmake .. && cmake --build .
```

当你执行完后，你会在主目录下看到生成了一个名为`miniSMTP`的可执行文件。在终端运行`./miniSMTP`，产生如下的结果：

```txt
Hello, This is a simple SMTP server
```

## 封装TCP Server

实际上，本教程可以直接通过`socket`编程进行实现，然而这样的实现过于地丑陋。故本教程拟对TCP Server进行一个简单地封装。其核心的内容的思路来源于[CS144](https://github.com/CS144/sponge)，如果你对此部分不感兴趣或者感到困难的话，可以直接跳过本部分，毕竟此处不是教程的重点。

### 封装系统调用

首先我们在`miniSMTPServer`目录下创建一个名为`util`的目录，然后在`util`目录下创建`util.hpp`，`util.cpp`和`CMakeLists.txt`。 我们只需要实现一个功能，就是封装系统调用，并提供相应的错误检测机制。其实现逻辑十分简单，此处不赘述。

```cmake
# miniSMTPServer/util/CMakeLists.txt
add_library(util STATIC util.cpp)
```

```c
// miniSMTPServer/util/util.hpp
#pragma once

#include <cerrno>
#include <string>
#include <system_error>

/**
 * @brief std::system_error plus the name of what was being
 * attempted.
 *
 */
class tagged_error : public std::system_error {
private:
  std::string attempt_and_error;  //!< what was attempted, and what happened

public:
  /**
   * @brief Construct from a category, an attempt, and an error code.
   *
   * @param[in] category is the category of error
   * @param[in] attempt is what was supposed to happen
   * @param[in] error_code is the resulting error
   */
  tagged_error(const std::error_category &category, const std::string &attempt, const int error_code)
      : std::system_error{error_code, category}, attempt_and_error{attempt + ": " + std::system_error::what()} {}

  /**
   * @brief Returns a C string describing the error
   *
   */
  const char *what() const noexcept override { return attempt_and_error.c_str(); }
};

/**
 * @brief A tagged_error for syscalls
 *
 */
class unix_error : public tagged_error {
public:
  /**
   * @brief  Construct from a syscall name and the resulting errno
   *
   * @param[in] attempt is the name of the syscall attempted
   * @param[in] error is the [errno(3)](\ref man3::errno) that resulted
   */
  explicit unix_error(const std::string &attempt, const int error = errno)
      : tagged_error{std::system_category(), attempt, error} {}
};

/**
 * @brief Error-checking wrapper for most syscalls
 *
 */
int SystemCall(const char *attempt, const int return_value, const int errno_mask = 0);

/**
 * @brief Version of SystemCall that takes a C++ std::string
 *
 */
int SystemCall(const std::string &attempt, const int return_value, const int errno_mask = 0);
```

```c
// miniSMTPServer/util/util.cpp
#include "util.hpp"

#include <string>

int SystemCall(const char *attempt, const int return_value, const int errno_mask) {
  if (return_value >= 0 || errno == errno_mask) {
    return return_value;
  }

  throw unix_error(attempt);
}

int SystemCall(const std::string &attempt, const int return_value, const int errno_mask) {
  return SystemCall(attempt.c_str(), return_value, errno_mask);
}
```

最后，需要在`miniSMTPServer`的`CMakeLists.txt`的末尾添加如下的语句：

```cmake
# miniSMTPServer/CMakeLists.txt
...
add_subdirectory(./util)
```

你也可以直接使用如下的命令获取到本小节的代码：

```sh
git checkout system-call
```

### 封装TCPSocket

如何封装一个`TCPSocket`呢？我们可以分为一下三步走：

1. 封装文件描述符类`FileDescriptor`
2. 封装地址类`Address`。
3. 继承`FileDescriptor`得到`TCPSocket`。

首先我们在`util`目录下创建`socket.hpp`和`socket.cpp`，并修改`CMakeLists.txt`将`socket.cpp`加入编译单元中。

```cmake
# miniSMTPServer/util/CMakeLists.txt
add_library(util STATIC util.cpp socket.cpp)
```

首先，我们需要构造一个`FileDescriptor`类，实际上对于该类的封装的思路可以很简单，也可以很复杂，如果我们不考虑其复制等生命周期问题，我们完全可以直接使用一个`int`类型的变量然后添加一些操作文件描述符的方法。在此处，我们采用更为复杂的设计模式：

1. `FileDescriptor`包含一个私有的`FDWrapper`，其包含了真正的文件描述符的信息。`FDWrapper`不能被复制也不能被移动。
2. 由于`FDWrapper`不能被复制也不能被移动，故`FileDescriptor`包含变量`internal_fd`的智能指针，表示`FDWrapper`被引用的数量，通过`shared_ptr`表示`FDWrapper`的生命周期。

于是，我们就可以给出如下的代码：

```c++
// miniSMTPServer/util/socket.hpp
class FileDescriptor {
  /**
   * @brief  A handle on a kernel file descriptor.
   *
   * @details FileDescriptor objects contain a std::shared_ptr to a FDWrapper.
   */
  class FDWrapper {
  public:
    int fd;               //!< The file descriptor number returned by the kernel
    bool eof = false;     //!< Flag indicating whether FDWrapper::_fd is at EOF
    bool closed = false;  //!< Flag indicating whether FDWrapper::_fd has been closed

    /**
     * @brief Construct from a file descriptor number returned by the kernel
     *
     */
    explicit FDWrapper(const int fd);
    /**
     * @brief Closes the file descriptor upon destruction
     *
     */
    ~FDWrapper();
    /**
     * @brief Calls [close(2)](\ref man2::close) on FDWrapper::fd
     *
     */
    void close();

    FDWrapper(const FDWrapper &other) = delete;
    FDWrapper &operator=(const FDWrapper &other) = delete;
    FDWrapper(FDWrapper &&other) = delete;
    FDWrapper &operator=(FDWrapper &&other) = delete;
  };

  //! A reference-counted handle to a shared FDWrapper
  std::shared_ptr<FDWrapper> internal_fd;

  // private constructor used to duplicate the FileDescriptor (increase the reference count)
  explicit FileDescriptor(std::shared_ptr<FDWrapper> other_shared_ptr);

public:
  //! Construct from a file descriptor number returned by the kernel
  explicit FileDescriptor(const int fd);

  //! Free the std::shared_ptr; the FDWrapper destructor calls close() when the refcount goes to zero.
  ~FileDescriptor() = default;

  //! Read up to `limit` bytes
  std::string read(const size_t limit = std::numeric_limits<size_t>::max());

  //! Read up to `limit` bytes into `str` (caller can allocate storage)
  void read(std::string &str, const size_t limit = std::numeric_limits<size_t>::max());

  //! Write a string, possibly blocking until all is written
  size_t write(const char *str);

  //! Write a string, possibly blocking until all is written
  size_t write(const std::string &str);

  //! Close the underlying file descriptor
  void close() { internal_fd->close(); }

  //! Copy a FileDescriptor explicitly, increasing the FDWrapper refcount
  FileDescriptor duplicate() const;

  int fd_num() const { return internal_fd->fd; }

  bool eof() const { return internal_fd->eof; }

  bool closed() const { return internal_fd->closed; }

  FileDescriptor(const FileDescriptor &other) = delete;
  FileDescriptor &operator=(const FileDescriptor &other) = delete;
  FileDescriptor(FileDescriptor &&other) = default;
  FileDescriptor &operator=(FileDescriptor &&other) = default;
};
```

```c++
// miniSMTPServer/util/socket.cpp
FileDescriptor::FDWrapper::FDWrapper(const int f) : fd{f} {
  if (fd < 0) {
    throw std::runtime_error("invalid fd number:" + std::to_string(fd));
  }
}

void FileDescriptor::FDWrapper::close() {
  SystemCall("close", ::close(fd));
  eof = closed = true;
}

FileDescriptor::FDWrapper::~FDWrapper() {
  try {
    if (closed) {
      return;
    }
    close();
  } catch (const std::exception &e) {
    std::cerr << "Exception destructing FileWrapper: " << e.what() << std::endl;
  }
}

FileDescriptor::FileDescriptor(const int fd) : internal_fd{std::make_shared<FDWrapper>(fd)} {}

FileDescriptor::FileDescriptor(std::shared_ptr<FDWrapper> other) : internal_fd{std::move(other)} {}

FileDescriptor FileDescriptor::duplicate() const { return FileDescriptor(internal_fd); }

void FileDescriptor::read(std::string &str, const size_t limit) {
  constexpr size_t BUFFER_SIZE = 1024 * 1024;
  const size_t size_to_read = std::min(BUFFER_SIZE, limit);
  str.resize(size_to_read);

  ssize_t bytes_read = SystemCall("read", ::read(fd_num(), str.data(), size_to_read));
  if (limit > 0 && bytes_read == 0) {
    internal_fd->eof = true;
  }

  if (bytes_read > static_cast<ssize_t>(size_to_read)) {
    throw std::runtime_error("read() read more than requested");
  }
  str.resize(bytes_read);
}

std::string FileDescriptor::read(const size_t limit) {
  std::string ret;
  read(ret, limit);
  return ret;
}

size_t FileDescriptor::write(const char *str) {
  size_t bytes_written = 0;
  bytes_written = SystemCall("write", ::write(fd_num(), str, strlen(str)));

  if (bytes_written == 0 && strlen(str) != 0) {
    throw std::runtime_error("write() returned 0 given non-empty input");
  }

  if (bytes_written > strlen(str)) {
    throw std::runtime_error("write() wrote more than requested");
  }

  return bytes_written;
}

size_t FileDescriptor::write(const std::string &str) { return write(str.c_str());
```

如果为了更好的扩展性，我们应该去新建一个`Address`类表示地址，但是此处我偷懒了，因为我们的重心并不在此处。`TCPSocket`的操作就比较简单了，只需要在`FileDescriptor`的基础上，增加以下的方法：

+ `bind`
+ `listen`
+ `accept`
+ `set_reuseaddr`

```c++
// miniSMTPServer/util/socket.hpp
class TCPSocket : public FileDescriptor {
public:
  //! Construct via [socket(2)](\ref man2::socket)
  TCPSocket();

  explicit TCPSocket(FileDescriptor &&fd);

  //! Wrapper around [setsockopt(2)](\ref man2::setsockopt)
  template <typename option_type>
  void setsockopt(const int level, const int option, const option_type &option_value);

  //! Bind a socket to a specified address with [bind(2)](\ref man2::bind), usually for listen/accept
  void bind(int port = 9400);

  //! Mark a socket as listening for incoming connections
  void listen(const int backlog = 16);

  //! Accept a new incoming connection
  TCPSocket accept();

  //! Allow local address to be reused sooner via [SO_REUSEADDR](\ref man7::socket)
  void set_reuseaddr();
};
```

```c++
// miniSMTPServer/util/socket.cpp
TCPSocket::TCPSocket() : FileDescriptor{SystemCall("socket", ::socket(AF_INET, SOCK_STREAM, 0))} {}

TCPSocket::TCPSocket(FileDescriptor &&fd) : FileDescriptor(std::move(fd)) {}

template <typename option_type>
void TCPSocket::setsockopt(const int level, const int option, const option_type &option_value) {
  SystemCall("setsockopt", ::setsockopt(fd_num(), level, option, &option_value, sizeof(option_value)));
}

void TCPSocket::set_reuseaddr() { setsockopt(SOL_SOCKET, SO_REUSEADDR, int(true)); }

void TCPSocket::bind(int port) {
  struct sockaddr_in address;

  address.sin_family = AF_INET;
  address.sin_port = htons(port);
  inet_pton(AF_INET, "127.0.0.1", &address.sin_addr);
  SystemCall("bind", ::bind(fd_num(), (struct sockaddr *)&address, sizeof(address)));
}

void TCPSocket::listen(const int backlog) { SystemCall("listen", ::listen(fd_num(), backlog)); }

TCPSocket TCPSocket::accept() {
  return TCPSocket(FileDescriptor(SystemCall("accept", ::accept(fd_num(), nullptr, nullptr))));
}
```

你也可以直接使用以下的命令得到上述的代码：

```sh
git checkout tcp-socket
```

你可以发现，我们其实绕了一大圈最终也只是为了实现这些命令，但是这些类的建立为我们写代码提供了良好的抽象能力。

## 测试

在完成了`TCPSocket`类的实现后，我们就可以开始写一个简单的服务器了，在这个简单的服务器中，我们不使用任何的线程，不使用任何的I/O复用，我们就简单地一个一个处理到来的请求。毕竟我们的重点并不在于服务器高性能的实现。

在`miniSMTPServer/miniSMTPServer.cpp`中编写如下的代码：

```c++
#include "socket.hpp"

#include <iostream>
#include <string>

int main() {
  std::cout << "Hello, This is a simple SMTP server\n";

  TCPSocket socket{};
  socket.bind();
  socket.listen();

  std::string result{};

  while (true) {
    auto s = socket.accept();

    while (true) {
      s.read(result);
      if (result.size() == 0) {
        std::cout << "S: Connection lost\n";
        s.close();
        break;
      }
      s.write("Hello, this is the server\n");
    }
  }

  return 0;
}
```

同时，我们需要修改`miniSMTPServer/CMakeLists.txt`为其添加相应的头文件和库：

```cmake
...

target_include_directories(miniSMTP PRIVATE ./util)
target_link_libraries(miniSMTP util)

...
```

同样地，你也可以使用下面的命令得到如上所示的代码：

```sh
git checkout tcp-server
```

编译代码运行`./miniSMTP`，然后执行命令`telnet 127.0.0.1 9400`，你应该能够得到如下图所示的执行的结果。

![Simple Server Test Result](https://s2.loli.net/2023/03/12/TnurykNBVoLYeXx.png)
