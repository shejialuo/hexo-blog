---
title: CS149-Assignment2 Building A Task Execution Library
tags:
  - 技术
  - 学习
categories:
  - CS149
date: 2023-02-01 11:36:26
---

## Part A

First, we should understand the file `itasksys.h`. It defines two abstract classes `IRunable` and `ITaskSystem`.

```c++
class IRunable {
public:
  virtual ~IRunnable();
  virtual void runTask(int task_id, int num_total_tasks = 0) = 0;
};
```

It is obvious that the user should define the `IRunable` class. And the `runTask` is provided by the user. And the core class is `ITaskSystem`.

```c++
class ITaskSystem {
public:
  ITaskSystem(int num_threads);
  virtual ~ITaskSystem();
  virtual const char* name() = 0;
  virtual void run(IRunnable* runnable, int num_total_tasks) = 0;
  virtual TaskID runAsyncWithDeps(IRunnable* runnable, int num_total_tasks,
                                  const std::vector<TaskID>& deps) = 0;
  virtual void sync() = 0;
}
```

For `ITaskSystem::run()`, it executes a bulk task launch of `num_total_tasks`. Task execution is synchronous with the calling thread, so it will return only when the execution of all tasks is complete.

For `ITaskSystem::runAsyncWithDeps`, Executes an asynchronous bulk task launch of `num_total_tasks`, but with a dependency on prior launched tasks.

In part A, we do not consider about the `ITaskSystem::runAsyncWithDeps`.

### Serial Program

We first look at class `TaskSystemSerial`. The function `run` is defined as follow.

```c++
void TaskSystemSerial::run(IRunnable* runnable, int num_total_tasks) {
  for (int i = 0; i < num_total_tasks; i++) {
    runnable->runTask(i, num_total_tasks);
  }
}
```

### Step 1

The most simplest answer is just to make the `run` function as the master, and create the threads to do the job. And join the thread at last. (However, I use C++14 for better lambda function).

```c++
void TaskSystemParallelSpawn::run(IRunnable* runnable, int num_total_tasks) {
    auto thread_func = [runnable_ = runnable, num = _num_threads, total = num_total_tasks](int i) {
        while(i < total) {
            runnable_->runTask(i, total);
            i += num;
        }
    };
    std::thread threads[_num_threads];
    for (int i = 0; i < _num_threads; ++i) {
      threads[i] = std::move(std::thread(thread_func, i));
    }
    for (int i = 0; i < _num_threads; ++i) {
      threads[i].join();
    }
}
```

### Step 2

Well, it is not so easy to write a thread loop. There are so many details we need to deal with. The most important thing here is do remember `join` the thread whe the class's lifetime ends.

It may seem that we need to accept every index to dynamic choose what to do. This is a stupid idea. Remember, we should reduce the size of synchronization. So we use the idea of step 1.

And you could see the following code for details. First we should add some private members.

```c++
private:
  int _num_threads; // to store the threads
  std::vector<std::thread> threads; // thread poll
  unsigned int jobs = 0x00; // bitmap value for indicating whether there is a job
  unsigned int bitmap_init_value = 0x00; // initialized bitmap value with 0x1111
  IRunnable* runnable_; // we need to record the runnable
  std::mutex queue_mutex; // the big lock
  bool terminate = false; // Whether we should terminate the thread
  int total_tasks = 0;    // we should record the total task
  void start(int num_threads); // start the thread pool
  void threadLoop(int i); // thread functionaility
  bool busy(); // whether the threads are busy doing their jobs
```

For constructor, we need to initialize the `bitmap_init_value` and start the thread pool.

```c++
TaskSystemParallelThreadPoolSpinning::TaskSystemParallelThreadPoolSpinning(int num_threads)
  : ITaskSystem(num_threads), _num_threads(num_threads) {
  unsigned int init = 0x01;
  for(int i = 0; i < _num_threads; ++i) {
    bitmap_init_value |= init;
    init <<= 1;
  }
  start(_num_threads);
}
```

For `start`, it is easy to understand.

```c++
void TaskSystemParallelThreadPoolSpinning::start(int num_threads) {
  threads.resize(num_threads);
  for(int i = 0; i < num_threads; ++i) {
    threads[i] = std::move(std::thread(&TaskSystemParallelThreadPoolSpinning::threadLoop, this, i));
  }
}
```

Now, we come the most important part. For how to tell whether there is a job for the thread, we use `jobs` as a bit map. And when the job is finished, we make the corresponding to 0.

```c++
void TaskSystemParallelThreadPoolSpinning::threadLoop(int i) {
  while(true && !terminate) {
    bool flag = false;
    {
      std::lock_guard<std::mutex> guard{queue_mutex};
      flag = (jobs >> i) & 0x01;
    }
    if(flag) {
      int taskId = i;
      while(taskId < total_tasks) {
        runnable_->runTask(taskId, total_tasks);
        taskId += _num_threads;
      }
      {
        std::lock_guard<std::mutex> guard{queue_mutex};
        jobs &= ~(0x01 << i);
      }
    }
  }
}
```

When the other calls `run`, it first initialize `jobs` to `bitmap_init_value`. And set the corresponding `runnable_` and the number of tasks. And if the `jobs` becomes 0, all the threads have competed their jobs. Thus, we can return.

```c++
void TaskSystemParallelThreadPoolSpinning::run(IRunnable* runnable, int num_total_tasks) {
  total_tasks = num_total_tasks;
  runnable_ = runnable;
  {
    std::lock_guard<std::mutex> guard{queue_mutex};
    jobs = bitmap_init_value;
  }
  while(busy());
}
```

Do remember join the threads at the destructor:

```c++
TaskSystemParallelThreadPoolSpinning::~TaskSystemParallelThreadPoolSpinning() {
  /*
    * Here, we don't need to synchronize the code, because
    * the thread will never write `terminate`. No matter
    * the thread may read some corrupted value, this doesn't matter.
  */
  terminate = true;
  for(int i = 0; i < _num_threads; ++i) {
    threads[i].join();
  }
}
```

### Step 3

In the step 2, we have pushed all the threads and `run` spin, which is inefficient. So we should make them sleep. The idea here is simple. We just use condition variables to achieve that. It is just consumer and producer problem. So I omit detail here.

### Conclusion for Part A

I wanna say sometimes spin is better than sleep. Because sleep would cause context switch, which may be inefficient when cpu speed is high.

## Part B

For part B, the most interesting thing is how should we solve the dependency.

When the user calls `runAsyncWithDeps`, it will pass a bunch of task ids. So there is an important question: how can we find an efficient data structure to represent the dependency.

For every task, it will have dependencies, so I use `unordered_map<TaskID, unordered_set<Task*>>` to represent dependencies for the following several reasons:

1. We can find the dependencies of a specified task.
2. Because the dependencies are represented as `unordered_set`, it is efficient to insert or delete.

Because there are different tasks, I define a helper class `Task`:

```c++
class Task {
public:
  TaskID id;
  IRunnable* runnable;
  int processing = 0;
  int finished = 0;
  int total_tasks;
  size_t dependencies;
  std::mutex task_mutex;
  Task(TaskID id_, IRunnable* runnable_, int total_tasks_, size_t deps)
    :id(id_), runnable(runnable_), total_tasks(total_tasks_), dependencies(deps) {}
};
```

As you can see, the `processing` filed is to indicate the next job for this task we should handle, the `finished` field is used to indicate how many tasks we have finished. We need to protect the variable, so for each task, there is a `mutex`.

If a task's dependencies (`Task::dependencies`) are not zero, we should not run this task. So I use the following data structures:

+ `vector<Task*> ready`: the tasks which are ready, so we can handle.
+ `unordered_set<Task*> blocked`: the tasks which should not run at now.

The whole idea is when user calls `runAsyncWithDeps`, we should update the `dependency` and just sends the task to `blocked`. And in the thread loop, we first check whether there is a task in the `ready`. If so, we random choose one task of the `ready` to handle, if the task is all finished, we should update the `dependency` again. When all tasks are finished, we should terminate.

It may sound easy, however the correct implementation is hard.

```c++
class TaskSystemParallelThreadPoolSleeping: public ITaskSystem {
private:
  bool terminate = false; // To indicate whether to stop the thread pool
  int _num_threads = 0; // To indicate how many threads
  int sleepThreadNum = 0; // The number of thread which is sleeping
  std::unordered_map<TaskID, Task*> finished {}; // To record the finished task
  std::vector<Task*> ready {}; // The task is ready to be processed
  std::unordered_set<Task*> blocked {}; // The task is blocked
  std::vector<std::thread> threads;
  std::unordered_map<TaskID, std::unordered_set<Task*>> depencency {}; // The depencency information
  TaskID id = 0;
  std::mutex queue_mutex;
  std::condition_variable consumer;
  std::condition_variable producer;
  void start(int num_threads);
  void threadLoop(int index);
  void deleteFinishedTask(Task* task);
  void moveBlockTaskToReady();
  void signalSync();
public:
  TaskSystemParallelThreadPoolSleeping(int num_threads);
  ~TaskSystemParallelThreadPoolSleeping();
  const char* name();
  void run(IRunnable* runnable, int num_total_tasks);
  TaskID runAsyncWithDeps(IRunnable* runnable, int num_total_tasks,
                          const std::vector<TaskID>& deps);
  void sync();
};
```

Then we first look at `runAsyncWithDeps`:

```c++
/**
 * For simplicity and easy-handling, we just make the new task to the `blocked`, and record
 * the dependency information and notify all the producers, and immediately return to the
 * user for async operation. And also we make the implementation more easily.
 */
TaskID TaskSystemParallelThreadPoolSleeping::runAsyncWithDeps(IRunnable* runnable, int num_total_tasks,
                                                    const std::vector<TaskID>& deps) {
  Task* task = new Task(id, runnable, num_total_tasks, deps.size());
  {
    std::unique_lock<std::mutex> guard{queue_mutex};
    // We just simply add the task to the blocked.
    blocked.insert(task);

    // Record dependency information for later processing
    for (TaskID dep : deps) {
      if (depencency.count(dep)) {
        depencency[dep].insert(task);
      } else {
        depencency[dep] = std::unordered_set<Task*>{task};
      }
    }
    // We should notify the producer to continue processing
    producer.notify_all();
  }
  return id++;
}
```

The main functionality is in the `threadLoop` function:

```c++
void TaskSystemParallelThreadPoolSleeping::threadLoop(int id_) {
  while(true) {
    int index = -1;
    Task* task = nullptr;
    {
      std::unique_lock<std::mutex> guard{queue_mutex};
      if(ready.empty()) {
        if(!blocked.empty()) {
          // We should check to move the blocked to the ready.
          moveBlockTaskToReady();
        }
        // If ready is still empty, we should sleep the thread.
        if(ready.empty()) {
            sleepThreadNum++;
            producer.wait(guard);
            sleepThreadNum--;
          }
      }
      /*
        * Here, we must tell whether the ready is empty,
        * when ready.size() == 0, rand() % 0 will cause
        * float point exception. It sucks.
      */
      if(!ready.empty()) {index = rand() % ready.size();
        // Here, we use random to choose the task for each thread
        // for simplicity.
        task = ready[index];
      };
    }
    if(terminate) {
      return;
    }
    if(task == nullptr) continue;
    int processing = -1, finished = -1;
    {
      std::unique_lock<std::mutex> guard{task->task_mutex};
      processing = task->processing;
      // There are some situations `processing` will exceed
      // the total number, because we don't know when the
      // `deleteFinishedTask` is finished. We may choose the
      // task which is actually finished (or just only one)
      if(processing >= task->total_tasks) continue;
      task->processing++;
    }
    if(processing < task->total_tasks) {
      task->runnable->runTask(processing, task->total_tasks);
      std::unique_lock<std::mutex> guard{task->task_mutex};
      task->finished++;
      finished = task->finished;
    }
    if(finished == task->total_tasks) {
      std::unique_lock<std::mutex> guard{queue_mutex};
      deleteFinishedTask(task);
      // When we signalSync, there are may be some threads which
      // are processing useless. So it may just return to the
      // destructor. So in the destructor we must wait for all
      // the thread going to sleep. And we call `notify_all` to
      // make all the threads stop. The design here should be
      // optimized. However, I don't have enough time...
      signalSync();
    }
  }
}
```

And the other part is some auxiliary functions:


```c++
void TaskSystemParallelThreadPoolSleeping::deleteFinishedTask(Task* task) {
  size_t i = 0;
  for (;i < ready.size(); ++i) {
    if(ready[i] == task) break;
  }
  finished.insert({ready[i]->id ,ready[i]});
  ready.erase(ready.begin() + i);

  if(depencency.count(task->id)) {
    for(auto t: depencency[task->id]) {
      t->dependencies--;
    }
  }
}

/**
 * Move blocked task to the ready when the task's dependency is
 * all finished.
 */
void TaskSystemParallelThreadPoolSleeping::moveBlockTaskToReady() {
  std::vector<Task*> moved {};
  for(auto task : blocked) {
    if(task->dependencies == 0) {
      ready.push_back(task);
      moved.push_back(task);
    }
  }
  for(auto task: moved) {
    blocked.erase(task);
  }
}

/**
 * When all the tasks are finished, which means `ready` and `blocked`
 * are are empty, we could signal the ONLY ONE consumer.
 */
void TaskSystemParallelThreadPoolSleeping::signalSync() {
  if(ready.empty() && blocked.empty()) {
    consumer.notify_one();
  }
}
```

And for the destructor:

```c++
TaskSystemParallelThreadPoolSleeping::~TaskSystemParallelThreadPoolSleeping() {

  terminate = true;

  // It may seem why here we need to use spin to test the sleepThreadNum
  // Because of the design, there may be some threads who is not sleeping at
  // this time, in order to make there is no dead-lock. See the `threadLoop`
  // for more detail.
  while(true) {
    std::unique_lock<std::mutex> guard{queue_mutex};
    if(sleepThreadNum == _num_threads) break;
  }

  // We should notify all the threads to return
  producer.notify_all();

  for(int i = 0; i < _num_threads; i++) {
    threads[i].join();
  }

  // We should free the memory
  for(auto task : finished) {
   delete task.second;
  }
}
```
