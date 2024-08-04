---
title: 'MIT6.824: MapReduce'
tags:
  - 技术
  - 学习
categories:
  - MIT6.824
date: 2024-08-04 15:13:57
---


This is my record for finish MIT 6.824(spring 2022) first lab [MapReduce](http://nil.csail.mit.edu/6.824/2022/labs/lab-mr.html).

## Sequence Version

Before we dive into how to solve this problem, we first need to understand the sequence version. First, we look at how `Map` operates.

```go
intermediate := []mr.KeyValue{}
for _ filename := range os Args[2:] {
  file, err := os.Open(filename)
  if err != nil {
    log.Fatalf("cannot open %v", filename)
  }
  content, err := io.ReadAll(file)
  if err != nil {
    log.Fatalf("cannot read %v", filename)
  }
  file.Close()
  kva := mapf(filename, string(content))
  intermediate = append(intermediate, kva...)
}
```

The idea for above code is simple. It just calls the user-provided `Map` function to generate the key-value pair struct `mr.KeyValue` and put the results into `intermediate`.

Next we need to partitioning the `intermediate` to make it into groups. Actually, we can just sort the results. This is also what the code does.

```go
sort.Sort(ByKey(intermediate))
```

Now next we need to call user-provided `Reduce` function. This is also easy. We have already sorted the `intermediate`. Now we just handle the groups one by one.

```go
i := 0
for i < len(intermediate) {
  j := i + 1
  for j < len(intermediate) && intermediate[j].key == intermediate[i].key {
    j++
  }
  values := []string{}
  for k := i; k < j; k++ {
    values = append(values, intermediate[k].Value)
  }
  output := reducef(intermediate[i].Key, values)
  fmt.Fprintf(ofile, "%v %v\n", intermediate[i].Key, output)
  i = j
}
```

## Need Analysis

### Coordinator

For MapReduce, there should be a coordinator which allocates the tasks. And the coordinator should use data structure to hold the status. The coordinator could not record the result file of the `Map` operation, because we have used some rules to make a pattern match.

### Worker

However, for a worker, things would be complicated. In the code, worker could do both `Map` and `Reduce`. I don't think it is a good idea to make them exclusively. So for a worker, it should start two goroutines, one for `Map` and another for `Reduce`.

For `Map` operation, we should achieve the following functionality:

+ Divide the intermediate keys into buckets for `nReduce` reduce tasks where `nReduce` is the number of reduce tasks.
+ The intermediate files name should be `mr-X-Y`, where `X` is the Map task number, and `Y` is the reduce task number.
+ Use `encoding/json` package to write the key/value pairs in JSON format to intermediate files.

For `Reduce` operation, we should achieve the following functionality:

+ Use `encoding/json` package to read the key/value paris in JSON format from intermediate files.
+ Put the output of the Xth reduce task in the file `mr-out-X`.

## Data Structure Design

There are the following things we need to consider:

+ We need to use the semantic words to represent the states both for the coordinator and workers. However, golang doesn't provide `enum` type like C/C++. So I decide to use `const` to emulate the `enum` type like the following:

  ```go
  type WorkingStatus int32

  const (
    idle       WorkingStatus = 0
    processing WorkingStatus = 1
    failed     WorkingStatus = 2
    terminated WorkingStatus = 3
  )
  ```

+ We need to store the tasks we need to handle in the coordinator and also we need to allocate this task for worker. So we need a way to represent the `Map` and `Reduce` task.

  ```go
  type MapTask struct {
    ID       int    // the current map task id
    Filename string // the file which the map task is processing
  }

  type ReduceTask struct {
    ID           int // the current reduce task id
    MapTaskTotal int // the current total map task num for the reduce task
  }
  ```

+ We need to store the worker information, because the worker may crash. If we do not store this state, we cannot do any crash recovery. Because a worker will handle `Map`and `Reduce` task, so we need to also create states to hold task status. I create a `WorkerState` to combine the `MapWorker` and `ReduceWorker`.

  ```go
  type MapWorker struct {
  status    WorkingStatus // status
  task      MapTask       // current task
  startTime time.Time     // map task start time
  }

  type ReduceWorker struct {
    status    WorkingStatus // status
    task      ReduceTask    // current task
    startTime time.Time     // reduce task start time
  }

  type WorkerState struct {
    id           int          // Worker identifier
    mapWorker    MapWorker    // the map worker
    reduceWorker ReduceWorker // the reduce worker
  }
  ```

And we could the following data structure for coordinator:

```go
type Coordinator struct {
  worker                []WorkerState // To store the status of the workers
  mapTask               []MapTask     // The map tasks to be done
  reduceTask            []ReduceTask  // The reduce tasks to be done
  mapTaskNum            int           // The total number of map task
  reduceTaskNum         int           // The total number of reduce task
  mapTaskFinishedNum    int           // The number of mapTask finished
  reduceTaskFinishedNum int           // The number of reduceTask finished
  allMapFinished        bool          // To indicate whether all Map operations are finished
  allReduceFinished     bool          // To indicate whether all Reduce operations are finished
  status                WorkingStatus // The status of the coordinator
}
```

## RPC Operations

Next, we need to handle the rpc operations where we need to consider the synchronization between workers and coordinator.

### Register

When a worker is initialized, it should call `Register` RPC. The coordinator should lock the mutex and update the `worker` field and return the id and the reduce bucket to the worker like the following:

```go
type RegisterRequest struct{}

type RegisterResponse struct {
  ID      int // the worker identifier
  NReduce int // the reduce bucket
}

func (c *Coordinator) Register(request *RegisterRequest, response *RegisterResponse) error {
  mutex.Lock()
  defer mutex.Unlock()

  newId := len(c.worker)

  newWorker := WorkerState{
    id: newId,
    mapWorker: MapWorker{
      status: idle,
    },
    reduceWorker: ReduceWorker{
      status: idle,
    },
  }
  c.worker = append(c.worker, newWorker)

  response.ID = newId
  response.NReduce = c.reduceTaskNum

  return nil
}
```

### Request Map Task

When the workers want to request map task, it should call `MapRequest` RPC. However, when there is no task available allocated to the worker, we should block the process in RPC call. You could see the following code to understand the detail.

```go
type MapTaskRequest struct {
  ID int // the worker identifier
}

type MapTaskResponse struct {
  Task     MapTask
  Shutdown bool
}

func (c *Coordinator) MapRequest(
  request *MapTaskRequest,
  response *MapTaskResponse) error {

  condMap.L.Lock()
  defer condMap.L.Unlock()

  // Here, we use `len(c.filenames)` to indicate whether there is
  // an available task for `Map` operation.
  for len(c.mapTask) == 0 {

    // If all of the `Map` operation is finished, we should
    // set the `Shutdown` field to `true` and return.
    if c.allMapFinished {
      response.Shutdown = true
      return nil
    }

    condMap.Wait()
  }

  // We should return map task back to the `Map`
  response.Task = c.mapTask[0]

  // Here, we should store the information of the worker
  c.worker[request.ID].mapWorker.task = c.mapTask[0]
  c.worker[request.ID].mapWorker.status = processing
  c.worker[request.ID].mapWorker.startTime = time.Now()

  c.mapTask = c.mapTask[1:]

  return nil
}
```

### Finish Map Task

When the workers finish the `Map` operation, it should call `MapFinish` to indicate the coordinator that "I have finished this task". However, there are so many details we need to consider:

1. There are some tasks running too much time which we will consider they are failed. If later the coordinator receives the `MapFinish` request, we should simply indicate the workers that they should be shutdown.
2. Because the workers may crash, we should never write the real name here. Instead workers should create temp file name. The coordinator should rename this in the RPC call. So we need to add a new field `Info` representing the mapping between the temp file name and the original file name.

```go
type MapTaskRequest struct {
  ...
  Info map[string]string
}

func (c *Coordinator) MapFinish(
  request *MapTaskRequest,
  response *MapTaskResponse) error {

  mutex.Lock()
  defer mutex.Unlock()

  if c.worker[request.ID].mapWorker.status == failed ||
    c.allMapFinished {
    response.Shutdown = true
    return nil
  }

  if c.worker[request.ID].mapWorker.status == processing && !c.allMapFinished {

    c.mapTaskFinishedNum++

    c.worker[request.ID].mapWorker.status = idle

    for temp, original := range request.Info {
      os.Rename(temp, original)
    }

    // When all the `Map` operation is finished we should broadcast for
    // `Reduce` operation. Pay attention that `len(c.filenames) == 0`
    if c.mapTaskFinishedNum == c.mapTaskNum {
      c.allMapFinished = true
      condReduce.Broadcast()
    }
  }

  return nil

}

```

The `ReduceRequest` and `ReduceFinish` functions are like `MapRequest` and `MapFinish`. I omit the detail here.

## Coordinator Operations

There are two operations we need to consider for coordinator, one is that coordinator needs to check the liveness of the each worker for executing crash recovery. I define a goroutine `checkLiveness` here.

```go
func (c *Coordinator) checkLiveness() {

  for {

    done := c.Done()
    if done {
      break
    }

    mutex.Lock()

    mapRestartTime := 0
    reduceRestartTime := 0

    for i := range c.worker {

      if c.worker[i].mapWorker.status == processing && time.Since(c.worker[i].mapWorker.startTime).Seconds() > 10 {
        mapRestartTime++
        c.mapTask = append(c.mapTask, MapTask{
          ID:       c.worker[i].mapWorker.task.ID,
          Filename: c.worker[i].mapWorker.task.Filename,
        })
        c.worker[i].mapWorker.status = failed
      }

      if c.worker[i].reduceWorker.status == processing && time.Since(c.worker[i].reduceWorker.startTime).Seconds() > 10 {
        reduceRestartTime++
        c.reduceTask = append(c.reduceTask, ReduceTask{
          MapTaskTotal: c.worker[i].reduceWorker.task.MapTaskTotal,
          ID:           c.worker[i].reduceWorker.task.ID,
        })
        c.worker[i].reduceWorker.status = failed
      }
    }

    for i := 0; i < mapRestartTime; i++ {
      condMap.Signal()
    }

    for i := 0; i < reduceRestartTime; i++ {
      condReduce.Signal()
    }

    mutex.Unlock()

    time.Sleep(5 * time.Second)
  }
}
```

## Worker Operations

The entrypoint is the `Worker` function, we will start two goroutine in this function, one is used for handling `mapProcess` and the other is used for handling `reduceProcess`.

```go
func Worker(mapf func(string, string) []KeyValue,
  reducef func(string, []string) string) {

  // When there is a new worker, we should call RPC
  // `RegisterRPC`.
  response := RegisterResponse{}
  request := RegisterRequest{}
  mutex = sync.Mutex{}
  if !RegisterRPC(&request, &response) {
    return
  }
  wg.Add(2)

  go mapProcess(mapf, response.ID, response.NReduce)
  go reduceProcess(reducef, response.ID, response.NReduce)

  wg.Wait()
}
```

`mapProcess` goroutine will handle the following things:

1. Firstly uses `MapRequestRPC` to request one job from coordinator, and the most two important things got from coordinator is the `MapTask` structure.
2. We need to maintain the mapping between the temp file name and the original filename.

```go
func mapProcess(mapf func(string, string) []KeyValue, id int, nReduce int) {
  defer wg.Done()

  for {

    request := MapTaskRequest{ID: id}
    response := MapTaskResponse{}

    if !MapRequestRPC(&request, &response) {
      return
    }

    // If the coordinator is terminated, we should terminate
    // the `mapProcess`
    if response.Shutdown {
      return
    }

    tempToOriginal := make(map[string]string)

    intermediate := []KeyValue{}
    filename := response.Task.Filename
    file, _ := os.Open(filename)
    content, _ := io.ReadAll(file)
    file.Close()
    kva := mapf(filename, string(content))
    intermediate = append(intermediate, kva...)

    outputNamePrefix := fmt.Sprintf("mr-%d-", response.Task.ID)

    outputFile := []*os.File{}
    enc := []*json.Encoder{}
    for i := 1; i <= nReduce; i++ {
      outputName := fmt.Sprintf("%s%d", outputNamePrefix, i)
      tempFile, _ := os.CreateTemp(".", "tempfile-")
      tempToOriginal[tempFile.Name()] = outputName

      encTemp := json.NewEncoder(tempFile)
      outputFile = append(outputFile, tempFile)
      enc = append(enc, encTemp)
    }

    for _, kv := range intermediate {
      reduceNum := ihash(kv.Key) % nReduce
      enc[reduceNum].Encode(kv)
    }

    for _, file := range outputFile {
      file.Close()
    }

    request.Info = tempToOriginal

    if !MapFinishRPC(&request, &response) {
      return
    }

    for k := range tempToOriginal {
      delete(tempToOriginal, k)
    }
  }
}
```

`reduceProcess` is just the same as `mapProcess`. Actually, from my perspective, there is nothing different.

```go
func reduceProcess(reducef func(string, []string) string, id int, nReduce int) {
  defer wg.Done()

  for {

    request := ReduceTaskRequest{ID: id}
    response := ReduceTaskResponse{}

    if !ReduceRequestRPC(&request, &response) {
      return
    }

    if response.Shutdown {
      return
    }

    intermediate := []KeyValue{}
    for i := 1; i <= response.Task.MapTaskTotal; i++ {
      inputName := fmt.Sprintf("mr-%d-%d", i, response.Task.ID)
      file, _ := os.Open(inputName)
      dec := json.NewDecoder(file)
      for {
        var temp KeyValue
        if err := dec.Decode(&temp); err != nil {
          break
        }
        intermediate = append(intermediate, temp)
      }
      file.Close()
    }

    sort.Sort(ByKey(intermediate))

    temporaryToOriginal := make(map[string]string)

    outputName := fmt.Sprintf("mr-out-%d", response.Task.ID)
    tempFile, _ := os.CreateTemp(".", "tempfile-")
    temporaryToOriginal[tempFile.Name()] = outputName
    i := 0
    for i < len(intermediate) {
      j := i + 1
      for j < len(intermediate) && intermediate[j].Key == intermediate[i].Key {
        j++
      }
      values := []string{}
      for k := i; k < j; k++ {
        values = append(values, intermediate[k].Value)
      }
      output := reducef(intermediate[i].Key, values)
      fmt.Fprintf(tempFile, "%v %v\n", intermediate[i].Key, output)
      i = j
    }
    tempFile.Close()
    request.Info = temporaryToOriginal

    if !ReduceFinishRPC(&request, &response) {
      return
    }
  }
}
```
