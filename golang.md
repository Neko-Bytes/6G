# The Beginner's Guide to Go (Golang)

## Module 1: Introduction to Go & How it Runs

Go (often called Golang) is a programming language created by Google. It was designed to be simple, highly readable, and exceptionally fast.

### 1.1 Compilation vs. Interpretation

Computers do not understand English; they only understand binary code (zeros and ones). There are two main ways programming languages translate your instructions:

* **Interpreted Languages (e.g., Python):** The computer reads your code line-by-line, translating and executing as it goes. This is easy to test but slow because the translation happens while the program is running.
* **Compiled Languages (Go, C, C++):** Before you run your program, a tool called a **compiler** translates your entire document into a single, standalone binary file.

Because Go is a **compiled language**, your computer runs the translated file directly. There is no translator active during execution, making Go programs fast and efficient.

## Module 2: Modularity and the Package System

In Go, developers break code into small, self-contained, and reusable pieces. This is called **modularity**. Go enforces this using **Packages**.

### 2.1 What is a Package?

A **package** is simply a directory (a folder) on your computer that contains one or more Go code files. Every file inside that folder must start with a line of code declaring which package it belongs to.

### 2.2 The Capitalization Rule (Enforced Privacy)

Go has a unique, built-in system for protecting code boundaries using capitalization:

* **Uppercase (Public / Exported):** If a variable, function, or structure name starts with a **Capital letter** (e.g., func DisplayMessage()), it is public. Other packages are allowed to import your package and use this function.
* **Lowercase (Private / Unexported):** If a name starts with a **lowercase letter** (e.g., func secretCalculation()), it is private. It is locked inside its home package folder, and no outside code can see or modify it.

### 2.3 The "No Dead Weight" Rule

Go enforces strict rules to keep programs clean:

1. You cannot import an external package folder and not use it.
2. You cannot declare a variable and never use it.
If you do either of these, **Go will refuse to compile your program**.

## Module 3: Structuring Data (No Classes, Just Structs)

Go does not have a class keyword. Instead, Go simplifies data organization using **Structs**.

### 3.1 What is a Struct?

A **struct** (short for structure) is a custom blueprint that defines a collection of fields (variables) grouped under a single name.

```go
type User struct {
    ID       int    // Public field (Capitalized)
    Username string // Public field (Capitalized)
    password string // Private field (lowercase)
}

```

## Module 4: Concurrency and Goroutines

Computers often have to do multiple tasks at the exact same time. This is called **concurrency**.

### 4.1 Traditional Threads vs. Goroutines

Traditionally, operating systems use **OS Threads**. Think of a thread as a heavy, physical lane on a highway. If a C program wants to run 1,000 tasks, it must build 1,000 physical OS lanes.

Go introduces **Goroutines**. A goroutine is a lightweight lane of execution managed by the Go application itself.

| Feature | Traditional OS Thread (C) | Goroutine (Go) |
| --- | --- | --- |
| **Startup Memory** | 1 MB to 2 MB (Heavy) | 2 KB (Ultra-Lightweight) |
| **Managed By** | Operating System Kernel | Go Runtime Scheduler |
| **Creation Cost** | Slow | Instant |

### 4.2 The Go Runtime Scheduler ($M:N$ Multiplexing)

Go manages these massive numbers of tasks using an $M:N$ **Scheduler**. The scheduler takes a large number of lightweight Goroutines ($M$) and dynamically maps (multiplexes) them onto a small, fixed number of heavy physical OS Threads ($N$) matching your physical CPU cores.

### 4.3 Context Switching

When a task pauses (waiting for a file to download), a **context switch** occurs so the CPU can work on something else.

* **OS Context Switch:** The operating system freezes the current thread, saves its huge memory footprint (up to 2 MB) to disk, and reloads a different thread. This is slow.
* **Go Context Switch:** The Go Scheduler handles this inside your application space. It simply slides the paused goroutine off the physical thread and slides a waiting one on. Because it only moves 2 KB of data, this switch is incredibly fast.



