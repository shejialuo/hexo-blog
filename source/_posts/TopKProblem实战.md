---
title: TopKProblem实战
tags:
  - 学习
  - 技术
categories:
  -
date: 2023-09-19 23:06:35
---


最近面试一直被问关于Top K的问题。网上有许多已经有这方面的答案了，然而我认为这些答案仅仅是局限于表面。网上的答案几乎都没有任何的代码实现，我认为理解这个问题需要自己去进行实现。仍然引用费曼的经典话：

> What I cannot create, I do not understand.

本教程的代码位于[topKProblem](https://github.com/shejialuo/topKProblem)仓库中。

## 最经典的Top K问题

最经典的Top K问题忽略任何机器的限制，也就是从一个数组中得到前K大的数。有两种经典的方法：

+ 快速选择
+ 堆排序

### 快速选择

```c++
static int partition(std::vector<int> &data, int start, int end) {
  int pivot = data[end];

  int i = start - 1, j = start;
  while (j < end) {
    if (data[j] > pivot) {
      i++;
      std::swap(data[i], data[j]);
    }
    j++;
  }

  std::swap(data[i + 1], data[end]);
  return i + 1;
}

static int helper(std::vector<int> &data, int k, int start, int end) {
  if (start == end) {
    return start;
  }

  int mid = partition(data, start, end);
  int index = mid - start + 1;

  if (index == k) {
    return mid;
  } else if (index > k) {
    return helper(data, k, start, mid - 1);
  } else {
    return helper(data, k - index, mid + 1, end);
  }
}

std::vector<int> topKUsingPartition(std::vector<int> &data, int k) {
  int index = helper(data, k, 0, data.size() - 1);
  return std::vector<int>{data.cbegin(), data.cbegin() + index + 1};
}
```

### 堆排序

```c++
std::vector<int> topKUsingHeap(const std::vector<int> &data, int k) {
  std::vector<int> ans(k, 0);

  std::priority_queue<int, std::vector<int>, std::greater<int>> heap{};

  for (int i = 0; i < k; i++) {
    heap.push(data[i]);
  }

  for (int i = k; i < data.size(); i++) {
    if (data[i] > heap.top()) {
      heap.pop();
      heap.push(data[i]);
    }
  }

  for (int i = k - 1; i >= 0; i--) {
    ans[i] = heap.top();
    heap.pop();
  }

  return ans;
}
```

## 受限环境下的Top K问题

假设我们的机器只有30M的内存，我们希望处理500M的`int_32`类型的数据，这些数据存储在文件中。我在此处假设`k`的大小是我们能够接受的，也就是`k`的大小对于内存而言可以忽略。

此部分的代码可以直接在[仓库](https://github.com/shejialuo/topKProblem/tree/master/src/topKRestricted)中查看。

### 使用堆排序

最简单的方式可能仍然是使用堆排序，因为我们仅仅只需要在内存中维护$O(k)$的大小，对于大文件我们每次只需要读取20M即可，因此内存永远不会超过。然而我们必须思考一个问题，数字是以什么形式保存在文件中。我在这让假设数字都是通过二进制方式的形式存储的，减少分割等操作。

我采取了如下的思路实现，在Linux系统下通过`setrlimit`限制进程的内存大小，使用c++17的`filesystem`标准库在临时文件中写入数据，进行操作。

### 使用快速选择

对于大文件我们每次只需要读取20M即可，然后选择前`k`大的数。我们就能得到一系列的数组，然后我们直接将数组合并，得到一个新的数组，其大小不可能超过内存，然后再次选择前`k`大的数，从而得出答案。当然我们也可以使用堆来处理，毕竟这属于一个子问题。

### 效率对比

执行的结果如下所示：

```txt
=== Result
Heap average time: 1.4765 s
Select average time: 1.90026 s
```

可以看出，使用堆还是要快一些的，这是因为减少了内存的拷贝。

## 受限条件下基于多线程的快速选择

基于快速选择的方法可以使用多线程进行加速，这是使用堆比不上的优势，现在的处理器基本上都是多核，此处使用多线程进行加速，如何构建多线程是一个问题。首先每个线程即是生产者也是消费者，这样代码写出来就极其的复杂，所以我的一个思路还是确定需要完成的任务数，通过任务数来实现线程之间的同步。

## 问题变种

所谓的问题变种，无非就是通过哈希得到频率，然后根据频率来得到Top K。没有任何本质的区别。

## 思考

实践出真知。
