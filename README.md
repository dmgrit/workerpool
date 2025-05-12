# Dynamic Worker Pool

This package provides a dynamic worker pool that allows you to reconfigure the number of workers at runtime.  
It reads messages from a Go channel and spawns a configurable number of goroutines to process them. 
* The pool uses a **token-based** mechanism: each goroutine must acquire a token from a limited pool before starting. Once it completes, the goroutine returns the token to the pool.   
This ensures that the number of active goroutines never exceeds the configured maximum.     
* Reconfiguring the number of workers is non-blocking, can be called multiple times, and sets a desired state that the pool will converge toward automatically.
* Setting the number of workers to 0 is allowed. In this case, the worker pool pauses processing until the worker count is set to a value greater than zero.

The pool is closed in any of the following cases:
* The `Shutdown()` method is called.
* The context provided during initialization is canceled.
* The input channel passed to the pool is closed.

## Installation

```shell
go get github.com/dmgrit/workerpool
```

## Example

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/dmgrit/workerpool"
)

func main() {
    // Create a context with cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Define a processing function
    processFn := func(item int) {
        fmt.Printf("Processing item: %d\n", item)
        time.Sleep(500 * time.Millisecond) // Simulate work
    }

    // Create a new DynamicWorkerPool with 3 workers
    pool, err := workerpool.NewDynamicWorkerPool(ctx, processFn, 3)
    if err != nil {
        fmt.Printf("Error creating worker pool: %v\n", err)
        return
    }

    // Create a channel to send items to the pool
    items := make(chan int)

    // Start the worker pool
    err = pool.Process(items)
    if err != nil {
        fmt.Printf("Error starting processing items: %v\n", err)
        return
    }

    // Send items to the pool
    go func() {
        for i := 1; i <= 100; i++ {
            items <- i
        }
        close(items) // Close the channel when done
    }()

    // Dynamically update the number of workers
    time.Sleep(2 * time.Second)
    fmt.Println("Updating workers to 5")
    if err := pool.UpdateWorkersNum(5); err != nil {
        fmt.Printf("Error updating workers: %v\n", err)
    }

    // Wait for all tasks to complete
    <-pool.Done()
    fmt.Println("All tasks completed")
}
```

