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

```go
var counter

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
  
  - as a synchronisation mecanism !

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
    <-ready // Wait for it!
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

### Timers

Instead of 

```
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

consider

---

```
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

```
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

## TODO

  - Mention Actor model ("Subject" vs "Object")

---

![bg](images/nasa-wAkLQnT2TC0-unsplash.jpg)

<!-- _color: white -->

# Where is the ISS?

![](images/ISS-position-browser.png)

---

`app.go`

```go
package main

import (
    "io"
    "net/http"
)

func PrintISSPosition() {
    resp, _ := http.Get("http://api.open-notify.org/iss-now.json")
    body := resp.Body
    bytes, _ := io.ReadAll(body)
    fmt.Println(string(bytes))
}

func main() {
    PrintISSPosition()
}
```

---


```
$ go run app.go 
{"message": "success", "timestamp": 1672155304, 
"iss_position": {"longitude": "-13.8055", "latitude": "-37.7661"}}
```

---

```go
import (
    "fmt"
    "time"
)

...

func Compute() {
    for i := 1; i <= 10; i++ {
    time.Sleep(time.Second / 10)
    fmt.Print(i, " ")
    }
    fmt.Println("")
}

func main() {
    PrintISSPosition()
    Compute()
}

```

---

```
$ time go run app.go 
{"message": "success", "timestamp": 1672155760, 
"iss_position": {"longitude": "21.8250", "latitude": "-50.7057"}}
1 2 3 4 5 6 7 8 9 10 

real    0m1,710s
user    0m0,394s
sys     0m0,125s
```

---

## Start a GoRoutine

```go

func main() {
    go PrintISSPosition()
    Compute()
}

```

---

```
$ time go run app.go 
1 2 3 4 {"message": "success", "timestamp": 1672156095, 
"iss_position": {"longitude": "54.5906", "latitude": "-49.9492"}}
5 6 7 8 9 10 

real    0m1,318s
user    0m0,400s
sys     0m0,163s
```