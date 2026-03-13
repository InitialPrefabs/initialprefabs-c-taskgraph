---
title: Home
layout: simple
og_type: website
---

# Table Of Contents
  * [C TaskGraph](#c-taskgraph)
  * [Preface and some history](#preface-and-some-history)
  * [Handles vs Async Await](#handles-vs-async-await)
  * [Unity Support](#unity-support)
  * [The workflow](#the-workflow)
  

# C TaskGraph

C TaskGraph is a small light weight taskgraph that allows you to create `Task` structs to run on separate threads written in C99 std. The initial
target platforms for support are
  * Windows (in progress)
  * Linux (in progress)

with future integration with
  * Mac
  * IOS
  * Android
  * Xbox
  * Playstation
  * Nintendo Switch

and planned engine integration with
  * Unity (in progress)
  * Unreal Engine

# Preface and some history
I really enjoyed how Unity's Job System API was easy to use for developers and in my mind made sense. If you wanted to schedule a task, you would 
define your own struct implementing `IJob`, `IJobParallelFor`, or `IJobFor` and use their `Schedule` extension method. Internally, Unity would handle 
everything for you.

After attempting to create a similar API with C# strictly, I found the performance of using `Task` objects worse as there were many gaps in between 
multiple tasks scheduled at once. So...as a result I wrote this in C99 with the intention to support multiple languages such as

* C# (in progress)
* C++
* Rust

## Handles vs Async Await
Async await is useful! For personal preference and readability, I definitely prefer the api where you receive a `Handle` to your task that is a minimal 
abstraction. This way, you can schedule all of your tasks at once and execute them later versus, launching a thread to run a C# `Task` immediately and 
using the `await` keyword for the promise of a result. This can become a little cumbersome to write if you need to launch multiple tasks all at once. 
DotNET does have a `Task.WhenAll` api where you can await **multiple** tasks, but writing this out is pretty cumbersome.

## Unity support
> With the intended Unity support, what is the difference between this and the Unity Job System?

Ultimately, I'm not trying to replace the Unity Job System. I want my version of the TaskGraph to be flexible supporting blittable and managed objects.

# The workflow
```c
// Define your task structs that you want to do the work
typedef struct MoveAlongXAxisTask {
  float* x_data;
  float speed;
  float delta_time;
} ExampleTask;
// Define function you want to execute on this task
void execute_move_along_x_axis(void* task, uint32_t index) {
  MoveAlongXAxis* task_data = (MoveAlongXAxis*)task;
  task_data->x_data[index] += speed * delta_time;
}

typedef struct SinAlongYAxisTask {
  float* y_data;
  float frequency;
  float delta_time;
} SinAlongYAxisTask;
void execute_sin_along_y_axis(void* task, uint32_t index) {
  SinAlongYAxisTask* task_data = (SinAlongYAxis*)task;
  task_data->y_data[index] = sin(frequency * delta_time);
}

typedef struct DoSomethingWithPostionTask {
  float* x_data;
  float* y_data;
} DoSomethingWithPositionTask;
void execute_do_something_task(void* task, uint32_t index) {
  // Do something with the xy data
}

TaskGraph graph;

void init() {
  // Initialize the taskgraph
  taskgraph_init(&graph, SOME_MAXIMUM_TASK_COUNT, SOME_MAXIMUM_TASK_DATA_CAPACITY);
}

void shutdown() {
  // Shutting down a taskgraph attempts to flush all remaining work and safely shutdown the worker queue.
  taskgraph_free(&graph);
}

void update() {
  MoveAlongXAxis task_a = { .x_data = YOUR_X_DATA, .speed = 1.0f, .delta_time = YOUR_ENGINE_DELTA_TIME };
  // Schedule this task without any dependencies 
  TaskHandle handle_a = taskgraph_schedule(&graph, create_task_desc(MoveAlongXAxis, &task_a, &execute_move_along_x_axis));

  SinAlongYAxis task_b = { .y_data = YOUR_Y_DATA, .frequency = 4.0f, .delta_time = YOUR_ENGINE_DELTA_TIME };
  // Schedule this task without any dependencies 
  TaskHandle handle_b = taskgraph_schedule(&graph, create_task_desc(SinAlongBAxis, &task_b, &execute_sin_along_y_axis));

  // This means that task_a and task_b will execute in parallel together at relatively the same time.
  
  // Now if we we want to make a task rely on task_a and task_b completing first, we need to add dependencies
  // to task_a and task_b.
  // Additionally, if you want a task to run on multiple threads, where each thread operates on a chunk, use the 
  // create_schedule_metadata macro to create a ScheduleMetadata struct with a valid batch_size that is <= total 
  // # of elements.
  DoSomethingWithPositionTask task_c = { .x_data = YOUR_X_DATA, .y_data = YOUR_Y_DATA };
  TaskHandle handle_c = taskgraph_schedule(
    &graph, 
    create_task_desc(DoSomethingWithPositionTask, &task_c, &execute_do_something_task), 
    create_schedule_metadata(YOUR_BATCH_SIZE, YOUR_MAX_NUMBER_OF_ELEMENTS, handle_a, handle_b));

  // Sort and execute the taskgraph
  taskgraph_sort(&graph);
  // When executing the taskgraph, all queued work will be scheduled on N threads, where N 
  // is the max # of logical processors your pc has.
  // Any worker thread that finishes its queue first will attempt to look at another thread's
  // local worker queue and steal that work to execute.
  taskgraph_execute(&graph);
}
```
