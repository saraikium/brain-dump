---
id: async-task-queue
created: 2024-04-07T17:30
updated: 2024-04-07T23:51
tags:
  - nodejs
  - async-task-queue
  - javascript
---

# Definition

An asynchronous task queue is a data structure used for managing and executing asynchronous tasks in a controlled manner. You can think of a task as an async function that performs I/O operations, network requests, or any other asynchronous operation.

## How it's used?

To understand how to build an async task queue, we first need to understand how it's used. We'll start by taking a look at the code that uses such a queue and then we'll focus on building the queue itself.

**So how does our queue work?**
Our queue works by running all the tasks in the scheduled order while also providing concurrency control. We can specify how many tasks to run at the same time using the concurrency parameter of the `AsyncTaskQueue` class constructor. 

```ts
// This queue will run 2 tasks concurrently.
const queue = new AsyncTaskQueue(2)
```

To demonstrate how to use this queue, let's start by creating a helper function `createTask` that takes a task name and returns a function that emulates an async task (like reading from a file, or sending a network request.).

```ts
/**
* @param {string} name - The task name.
*/
function createTask(name: string) {
  const task = (): Promise<string> => {
    return new Promise((resolve) => {
      setTimeout(() => resolve(`${name} done`), 2000);
    });
  };
  return task;
}


```


Now we can use this helper function to create a bunch of tasks. that we can then run using our task queue.

```ts

let tasks = new Array(10)
  .fill(0)
  .map((_, index) => createTask(`Task: ${index + 1}`));
```

Now that we have our tasks array ready, we can iterate over this task queue and run the tasks. You'll notice that when you run this code, tasks complete in the batches of two.

```ts
const queue = new AsyncTaskQueue(2);
tasks.forEach((t) => queue.addTask(t).then((r) => console.log(r)));
```

## Implementation

Let's take a look at the class definition.
```ts

export class AsyncTaskQueue {
  private taskQueue: Task<any>[] = [];
  private consumerQueue: ResolveFunc<Task<any>>[] = [];

}
```

 As you can see our class contains two internal arrays (which we'll be using as queues). 
- One for storing the tasks called the `taskQueue` . A task is defined as
```ts
// A task is an async function that returns a Promise containing value of type T
type Task<T> = () => Promise<T>;
```

- Our second queue will be used for storing the consumers. Consumers are functions that consume the value returned by running a task. (As you'll see soon the consumers will be `resolve` functions that are used inside the promise constructor). Let's define our Consumer type as well.

```ts
type ResolveFunc<T> = (value: T | PromiseLike<T>) => void;
```

## Constructor

We take `concurrency` as the input parameter of the constructor which controls how many tasks we can run concurrently. If the `concurrency` is less than `0`, we throw an error, otherwise, we spawn n runners where n is equal to concurrency. 
```ts
export class AsyncTaskQueue {
	...
	...
  constructor(concurrency: number) {
    if (concurrency <= 0) {
      throw new Error("Concurrency must be greater than 0")
    }
    for (let i = 0; i < concurrency; i++) {
      this.runner();
    }
  }
	...
	...
}
```

## Running the tasks

`runner` is the method we were calling inside the loop in our constructor.It retrieves the next task from the `taskQueue`  asynchronously and runs it. Since it's getting the next task asynchronously using `this.getNextTask`, the function execution will suspend until there is a task available to run. If there's no task, this function simply sleeps, so we have no wasted CPU cycles.

```ts
  /**
   *  this is function that contains an infinite loop. Inside the loop,
   * it gets the next task in the queue and runs it. 
   */
  private async runner() {
    while (true) {
      try {
        // getNextTask returns  Promise<Task<T>>
        // The await here unwraps the outer promise and we get Task<T>
		// and suspends the execution so we're not wasting CPU cycles. 
		// If the queue is empty, our runner simply sleeps.
		// Even though JavaScript never sleeps ;-)
        const task = await this.getNextTask();
        // The we run the task
        await task();
      } catch (err) {
        console.error(err);
      }
    }
  }

```


## Getting the next task

As we saw our runner runs an infinite loop and inside the loop, calls the `this.getNextTask` method to get the next task. Getting the next task works like this
- First thing to note is that our `getNextTask` returns a promise that contains the task. 
- Inside the promise constructor we check if the `taskQueue` has a task available. 
- If the task is available, we immediately resolve the promise with the available task.
- But if the queue is empty, we simply wait for the task to be added by pushing the resolve function inside the `consumerQueue`. You'll see how that works in the next section. Then soon as the next task is added to our `AsyncTaskQueue`, this promise will be resolved with the value of that task.

```ts
  private getNextTask<T>(): Promise<Task<T>> {
    return new Promise((resolve) => {
      // Check if there's a task available
      const nextTask = this.taskQueue.shift();
      if (nextTask) {
        //if a task is available resolve the promise with the task.
        resolve(nextTask);
      } else {
        // If there's no task available at the moment,
        // we wait for the next task to be added.
        this.consumerQueue.push(resolve);
      }
    });
  }

```

## Adding a task 

And now for the final part, how do we add a task to our `AsyncTaskQueue`. It looks something like this.

I have put numbered comments in the code so I can explain this easily.
```ts
  public addTask<T>(task: Task<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      const taskWrapper = () => {    // ---> 1
        const taskPromise = task(); // ----> 2
        taskPromise.then(resolve, reject); // ----> 3
        return taskPromise;
      };

      const consumer = this.consumerQueue.shift(); // ----> 4
      if (consumer) {
        consumer(taskWrapper); // ------> 5
      } else {
        this.taskQueue.push(taskWrapper); // ---->6
      }
    });
  }

```

This function takes a task as an input and returns a promise. This promise will resolve when the task has run successfully and the it'll contain the value obtained by running the given task. If the task throws an error, this promise will reject with that error so the users can handle both the success and error cases. like we saw above.

```ts
tasks.forEach((t) => queue.addTask(t).then((r) => console.log(r)));
```

Let's see what happens inside the promise constructor
1. We create a wrapper function `taskWrapper`. We'll push this wrapper to the `taskQueue` instead of pushing the task directly because we want the user to be able to handle the success or failure cases of running the task. 
2. We call the `task` and get a promise back. Notice that we're not using the await here so the promise won't be evaluated yet.
3. Next thing this `taskWrapper` does is evaluate the task promise by calling `taskPromise.then(resolve, reject)` and passing it the `resolve` and `reject` from arguments of the outer promise that we are returning to the user. This way, outer promise will either be fulfilled with the success value of running the task successfully or rejected with the failure reason of the said task.
4. Now that we have created our `taskWrapper` inside the promise constructor, we need to do something about it. As we saw in the `getNextTask` method, if there were no tasks available, we said we would wait? This was the line 
```ts
	...
	// If there's no task available at the moment,
	// we wait for the next task to be added.
	this.consumerQueue.push(resolve);
	...
```
well, now we check the `consumerQueue` and see if there's a consumer waiting. 
5. If we have a consumer waiting, we simply pass the `taskWrapper` to the waiting consumer so it can be run immediately.
6. If there's no consumer waiting, push the task wrapper to the `taskQueue`, where it'll eventually be picked up by a consumer.


So putting it all together here's what our `AsyncTaskQueue` class looks like

```ts
// queue.ts
type ResolveFunc<T> = (value: T | PromiseLike<T>) => void;
type Task<T> = () => Promise<T>;

export class AsyncTaskQueue {
  // taskQueue is used for storing tasks / taskWrappers
  private taskQueue: Task<any>[] = [];
  // consumerQueue is used for storing the resolve parameter of the Promise constructor.
  // You'll soon see why
  private consumerQueue: ResolveFunc<Task<any>>[] = [];

  constructor(concurrency: number) {
    if (concurrency <= 0) {
      throw new Error("Concurrency must be greater than 0")
    }
    for (let i = 0; i < concurrency; i++) {
      this.runner();
    }
  }

  /**
   *  this is function that contains an infinite loop. Inside the loop,
   * it gets the next task in the queue and runs it. 
   */
  private async runner() {
    while (true) {
      try {
        // getNextTask returns  Promise<Task<T>>
        // The await here unwraps the outer promise and we get Task<T>
        // and suspends the execution so we're not wasting CPU cycles. 
        // If the queue is empty, our runner simply sleeps.
        // Even though JavaScript never sleeps ;-)
        const task = await this.getNextTask();
        // The we run the task which 
        await task();
      } catch (err) {
        console.error(err);
      }
    }
  }

  private getNextTask<T>(): Promise<Task<T>> {
    return new Promise((resolve) => {
      // Check if there's a task available
      const nextTask = this.taskQueue.shift();
      if (nextTask) {
        //if a task is available resolve the promise with the task.
        resolve(nextTask);
      } else {
        // If there's no task available at the moment, we wait for the next task to be added.
        this.consumerQueue.push(resolve);
      }
    });
  }

  public addTask<T>(task: Task<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      // We'll push this wrapper function to the taskQueue instead of
      // pushing the task directly,
      // because we want to resolve or reject the promise we're returning 
      // from the addTask method.
      const taskWrapper = () => {
        // Get the promise
        const taskPromise = task();
        // Evaluate the promise and pass it to the 
        // resolve and reject handler of the promise we're
        // returning from the addTask method. 
        // This way when the task runs successfully,
        // our returned promise will be fulfilled with 
        // the success value, or the error reason of running the task.
        taskPromise.then(resolve, reject);
        return taskPromise;
      };

      // Remember we pushed a `resolve` to this.consumerQueue 
      // in the getNextTask method if no taks were available?
      const consumer = this.consumerQueue.shift();
      // If the consumer queue does have a runner waiting for a task, then 
      if (consumer) {
        // We simply give our taskWrapper to the runner directly
        consumer(taskWrapper);
      } else {
        // Otherwise we'll add it to the taskQueue 
        // where it'll be picked up by a runner
        this.taskQueue.push(taskWrapper);
      }
    });
  }
}

```

Here's how we use it.
```ts
// index.ts
import { AsyncTaskQueue } from "./queue.ts"

function createTask(name: string) {
  const task = (): Promise<string> => {
    return new Promise((resolve) => {
      setTimeout(() => resolve(`${name} done`), 2000);
    });
  };
  return task;
}

let tasks = new Array(10)
  .fill(0)
  .map((_, index) => createTask(`Task: ${index + 1}`));

const queue = new AsyncTaskQueue(2);

tasks.forEach((t) => queue.addTask(t).then((r) => console.log(r)));

```