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

With `go`, `says("world")`:

  - is a **non-blocking** call (with no return value),

  - gets executed **in the background**,

  - **concurrently** with `say("hello")`

---

### 🧪 Experiment

🚀 Run the same code several times.

What's going on?


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

## Pitfalls of Concurrency

⚠️ Concurrent process & Shared Variables

---

```go
func main() {
    counter := 0
    // 📈
    go func() {
        for {
            fmt.Println(counter)
            time.Sleep(time.Second)
        }
    }()
    for {
        // 😴
        time.Sleep(time.Second)
        // 🔨
        for i := 0; i < 1000; i++ {
            go func() {
                counter = counter + 1
            }()
            go func() {
                counter = counter - 1
            }()
        }
    }
}
```

---

### 🤦 Ooops

![bg contain](images/counter.svg)

---

  - In C/C++, you would use a 🔒 **lock** (mutex) to ensure that at most one process can access the `counter` variable at any given moment.

---

This also works in Go, but it's not idiomatic. 

In Go:

  > Don't communicate by sharing memory; 
  > share memory by communicating.

The core communication device is a **channel**.

---


## Channels

---

**TODO:**

Channels: (sync) signals & data.

  - signal ("sync on") end of goroutine,
    more generally, sync messages.

  - "emulate" return value (blocking or non-blocking)

  - multiple input values

---

## Patterns

  - Parallel execution

  - Throttling (ex 1 req / 5 sec)

  - Channel Duplication

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