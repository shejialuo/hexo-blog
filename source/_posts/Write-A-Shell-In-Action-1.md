---
title: Write A Shell In Action (1)
tags:
  - 技术
  - 教程
categories:
  - Write a Shell In Action
date: 2023-12-24 23:23:03
---


Why should we write a shell? Well, I think the most important reason is that it is fun. Learning how to write a shell will improve the system programming experience for the UNIX environment. Well, as the Feynman said:

> What I cannot create, I do not understand

## The Minimal Setup

In this tutorial, we will use the c++ as the language to write the shell. So at we should use `cmake` to construct the build environment. If you are not familiar with `cmake`, don't worry about it. It's easy to understand. For convenience, I hope you can just use the following commands to initialize the repository.

```sh
mkdir miniShell && cd miniShell
git init
git remote add upstream https://github.com/shejialuo/miniShell
git pull upstream start-code
```

At this time, you will find the `miniShell.cpp` under the root tree, the functionality of the `miniShell.cpp` is simple enough. Because this is tutorial, we just simplify the program. We assume `miniShell` would have only two modes like any shell:

+ Interactive
+ Batch mode

The idea is actually the same. However, at now. We do not implement any functionality.

```c++
#include <iostream>

void printUsage() { std::cout << "Usage: ./miniShell [file]\n"; }

int main(int argc, char *argv[]) {
  if (argc != 1 && argc != 2) {
    printUsage();
  }

  if (argc == 1) {
    std::cout << "This is a mini shell\n";
  } else {
    std::cout << "This is a mini shell called with a file " << argv[1] << '\n';
  }

  return 0;
}
```

If you are interested in the `CMakeLists.txt`. You could just provide its content to the ChatGPT, it will give you a wonderful explanation. At now, you could build this program:

```sh
mkdir build && cd build && cmake .. && cmake --build .
```

Under the `build` directory, you will find the `miniShell` executable.

## The Parser

We have already set up a minimal development environment, now we need to parse the shell script. Actually, writing a parser could be another theme. So in this tutorial, we will just use native way to parse the shell script.

So we should first make sure the functionality of the shell. I don't think it's a good idea to support the condition or loop control. It's not the point. We just want to learn the system programming. Below is the spec:

1. The operations for variables.

   ```sh
   variable=5
   echo $variable # 5
   echo ${variable} # 5
   echo '${variable}' # ${variable}
   echo "${variable}" # 5
   ```

2. Command substitution

   ```sh
   variable=`echo 5` # variable=5
   variable=$(echo 5) # variable=5
   ```

3. Redirection
   ```sh
   echo '5' > /tmp/txt
   echo '55' >> /tmp/txt
   cat < /tmp/txt
   ```
4. Pipe
   ```sh
   command1 | command2 | command3 | command4
   ```
5. Job control
   ```sh
   sleep 100 &
   bg %1
   ```

As you can see, actually we could simplify the question. We first split the script line by line. For each line, we split the line by `|`. And we will generate the following classes for further process:

1. `Variable`: handle the `a=3`.
2. `Command`: handle the `<command> <argument1> <argument2>`.
3. `PipeCommand`: handle the `Command1 | Command2 | Command3`.

I will not dive into this part, because it would be annoying and far away from the theme of this tutorial. Here, I give a snippet:

```c++
class Command : public CommandBase {
private:
  std::string command;
  std::vector<std::string> arguments;
  std::optional<std::string> redirectFile;
  RedirectStatus redirectStatus;

public:
  Command() = delete;
  Command(std::string &&c, std::vector<std::string> &&ar, std::optional<std::string> &&rf, RedirectStatus status)
      : CommandBase{}
      , command{std::move(c)}
      , arguments{std::move(ar)}
      , redirectFile{std::move(rf)}
      , redirectStatus{status} {}

  virtual void execute() final;
  virtual std::string inspect() final {
    std::string result = command;
    for (const auto &argument : arguments) {
      result += " " + argument;
    }
    return result;
  }
  virtual ~Command() = default;
};
```

From the above snippet, all we need to do is define the `execute` function. For each line, we will call the `execute` function. You could use the following command to pull the latest code:

```sh
git pull upstream add-parser
```

Below is the code hierarchy, if you are interested about the detail, you could see the `parser` directory.

```txt
.
├── CMakeLists.txt
├── command
│   ├── CMakeLists.txt
│   ├── command.cpp
│   └── command.hpp
├── miniShell.cpp
├── parser
│   ├── CMakeLists.txt
│   ├── parser.cpp
│   ├── parser.hpp
│   └── tests
│       ├── CMakeLists.txt
│       └── parserTest.cpp
└── README.md
```

Look at the `miniShell.cpp`:

```c++
#include "parser/parser.hpp"

#include <fstream>
#include <iostream>
#include <stdlib.h>
#include <string>

void printUsage() { std::cout << "Usage: ./miniShell [file]\n"; }

void start(std::istream &is, bool isFile = false) {
  Parser parser{};
  while (true) {
    if (!isFile) {
      std::cout << "$ ";
      std::cout.flush();
    }
    std::string script{};
    std::getline(is, script);

    if (script.empty()) {
      return;
    }

    if (!script.empty() && script.back() == '\n') {
      script.erase(script.length() - 1);
    }

    auto command = parser.parseScript(script);
    if (command != nullptr) {
      command->execute();
    }
  }
}

int main(int argc, char *argv[]) {
  if (argc != 1 && argc != 2) {
    printUsage();
  }

  if (argc == 1) {
    start(std::cin);
  } else {
    std::ifstream ifs{argv[1]};
    if (!ifs.is_open()) {
      std::cout << "Cannot open file " << argv[1] << '\n';
      exit(1);
    }
    start(ifs, true);
  }

  return 0;
}
```

Due to the nice abstraction from cpp, the file stream and standard input stream are all the `istream`. So we define a `start` command to abstract the process. The most important function you could see is the `execute` function.

## A Simple Start

Now we try a simple command `/usr/bin/ls`, we will write the method `Command::execute`. We will use `fork` system call to create a child process and waits for its action which would use `exec` system call for `/usr/bin/ls`. If you do not understand these concepts, you could read the man page of the `fork` or just read [Advanced Programming in the UNIX Environment](https://en.wikipedia.org/wiki/Advanced_Programming_in_the_Unix_Environment) chapter 8.

```c++
void Command::execute() {
  pid_t pid = fork();
  if (pid == -1) {
    std::cout << "[fork error]: cannot fork\n";
    exit(EXIT_FAILURE);
  } else if (pid == 0) {
    std::vector<char *> argv{};
    argv.resize(arguments.size() + 2);

    argv[0] = const_cast<char *>(command.c_str());
    for (size_t i = 0; i < arguments.size(); i++) {
      argv[i + 1] = const_cast<char *>(arguments[i].c_str());
    }

    argv.back() = nullptr;

    execv(command.c_str(), argv.data());
  } else {
    wait(NULL);
  }
}
```

However, the above code is clear. What we do is simply pass the command and arguments to the `execv` system call. Like the following figure illustrates, when we type `/usr/bin/ls`, the result is OK. However, when we type `ls`, there is no result. This is because we need to search the `$PATH` environment. We will improve our code for part 2.

![ls test result](https://s2.loli.net/2023/12/24/DjPidQtCuGmUHvb.png)

As usual, you could use the following command to get the code:

```sh
git pull upstream simple-ls
```
