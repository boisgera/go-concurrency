---
marp: true
title: Go Concurrency
author: Sébastien Boisgérault
theme: uncover
---

# Go Concurrency

---

## Goroutines 

💻 [A Tour of Go: Goroutines](https://go.dev/tour/concurrency/1)

```go
package main

import (
    "fmt"
    "time"
)

func say(s string) {
    for i := 0; i < 5; i++ {
        time.Sleep(100 * time.Millisecond)
        fmt.Println(s)
    }
}
```

----

### Serial execution

🚀 Execute

```go
func main() {
    say("world")
    say("hello")
}
```

---

### Concurrent Execution

🚀 Execute

```go
func main() {
    go say("world")
    say("hello")
}
```

---

`go says("world")`:

  - is a **non-blocking** call: it returns "immediately",
    and doesn't wait for the completion of the function.

  - its gets executed: 
  
      - **in the background**,

      - in its own **lightweight process**

      - with its own **execution thread**


  - **concurrently** with `say("hello")`

---

### 🧪 Experiment

🚀 Run the same code several times.

What's going on?

---

![bg left:25%](images/tim-mossholder-QbHF_M47FeY-unsplash.jpg)

### ⚠️ Pitfall

  - 🎲 Concurrency $\to$ **non-determinism**

  - 🤔 Programs are harder to understand

  - 🪲 Beware the [Heisenbugs](Heisenbug)!

---

### 🧪 Experiment

🚀 Execute

```go
func main() {
    go say("world")
    go say("hello")
}
```

What is going on ? Do you have a (dirty) fix ?

---
![bg right](images/uriel-soberanes-L1bAGEWYCtk-unsplash.jpg)

### ⚠️ Data races

```go
var counter = 0

func add(i int) {
    counter += i
}

func display() {
    for {
        fmt.Println(counter)
        time.Sleep(time.Second)
    }
}

func main() {
    go display() // 📈
    for {
        time.Sleep(time.Second) // 😴
        // 🔨 Hammer the counter!
        for i := 0; i < 1000; i++ {
            go add(1)
            go add(-1)
        }
    }
}
```

---

### 🤦 Ooops

![bg contain](images/counter.svg)

---

## Shared Variables

  - In C/C++, you would use a 🔒 **lock** (mutex) to ensure that at most one process can access the `counter` variable at any given moment.

---

This also works in Go, but it's not idiomatic. Instead:

  > Don't communicate by sharing memory; 
  > share memory by communicating.

The core communication device is a **channel**.

---

![bg](images/christophe-dion-3KA1M16PuoE-unsplash.jpg)

## Channels

---

### Create

Channels are typed and (optionally) buffered:

```go
message := make(chan string, 1)

numbers := make(chan int, 10)

ready := make(chan bool) // same as make(chan bool, 0)
```

---

### Read & Write

```go
message <- "Hello world!"
s := <-message
fmt.Println(s)
```

```go
t := <-message // ⏳ blocked
```

```go
message <- "Hello"
message <- "world! // ⏳ blocked
```

---

```go
numbers <- 0
numbers <- 1
numbers <- 2
fmt.Println(<-numbers)
fmt.Println(<-numbers)
fmt.Println(<-numbers)
fmt.Println(<-numbers) // ⏳ blocked
```

---

Unbuffered channels seem useless at first sight

```go
ready <- true // ⏳
```

```go
status := <-ready // ⏳
```



---

... but they actually shine 

  - in a concurrent setting 
  
  - as a synchronisation mecanism!

----

### We're not ready yet

```go
var ready = make(chan bool)

func Greetings() {
    for i:=0; i<10; i++ {
        fmt.Println("Hello world!")
    }
    ready <- true
} 

func main() {
    fmt.Println("begin")
    go Greetings()
    <-ready // Wait for the end!
    fmt.Println("end")
}

```

---

### Parallel Computations

```go
var c = make(chan int)

func sum(s []int) {
    sum := 0
    for _, v := range s {
        sum += v
    }
    c <- sum
}

func main() {
    s := []int{7, 2, 8, -9, 4, 0}
    go sum(s[:len(s)/2], c)
    go sum(s[len(s)/2:], c)
    x, y := <-c, <-c
    fmt.Println(x, y, x+y)
}
```

---

### Throttling Process

```go
func throttled(input chan string) chan string {
    output := make(chan string)
    go func() {
        for {
            output <- <-input
            time.Sleep(3 * time.Second)
        }
    }()
    return output
}

func main() {
    input := make(chan string)
    output := throttled(input)
    for i:=0; i<10; i++ {
        input <- "Hello world!"
        fmt.Println(<-output)
    }
}
```

---


### ✌️ No more data races

![bg left:33%](images/hello-i-m-nik-MAgPyHRO0AA-unsplash.jpg)

Channels: safe to use concurrently

```go
var counter = 0
var ch = make(chan int, 100)

func add(i int) {
    ch <- i
}

func counterHandler() {
    func() {
        for {
            counter += <-ch
        }
    }()
}
```

---

```go
func main() {
    go counterHandler() // ⚡ New!
    go display() // 📈
    for {
        time.Sleep(time.Second) // 😴
        // 🔨 Hammer the counter!
        for i := 0; i < 1000; i++ {
            go add(1)
            go add(-1)
        }
    }
}
```

---

### Timers

Sequential implementation of two concurrent processes:

```go
for {
    for i:=0; i < 10; i++ {
        // Executed at ~10Hz
        fmt.Println("FAST")
        time.Sleep(time.Second / 10) 
    }
    // Executed at ~1Hz
    fmt.Println("SLOW") 
}
```

---

Consider instead

```go
func Printer(m string, d time.Duration) {
    for {
        fmt.Print(m)
        time.Sleep(d)
    }
}

func main() {
    go Printer("FAST", time.Second / 10)
    Printer("SLOW", time.Second)
}
```

---

Even better

```go
func Printer(m string, d time.Duration) {
    for {
        wait := time.After(d)
        fmt.Println(m)
        <-wait
    }
}

func main() {
    go Printer("FAST", time.Second / 10)
    Printer("SLOW", time.Second)
}
```

or use the [`time` module](https://pkg.go.dev/time)  `Ticker` & `Timer` API!

---

![bg left:33%](images/tim-zankert-gm3M-CsuynI-unsplash.jpg)

```go
func HappyNewYear(year int) chan struct{} {
    trigger := make(chan struct{})
    newYear := time.Date(
        year, time.January, 
        0, 0, 0, 0, 0, time.UTC)

    go func() {
        for {
            if time.Now().After(newYear) {
                fmt.Println("🥳 Happy New Year!")
                trigger <- struct{}{}
                return
            }
            time.Sleep(time.Second)
        }
    }()

    return trigger
}

func main() {
    <-HappyNewYear(2023)
}
```

<!--

## TODO

  - Mention Actor model ("Subject" vs "Object")

-->