# Node.js Internal Architecture: Event Loop, Queues, and Processing Flow

Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient. Let me break down the internal workings with conceptual diagrams and flow explanations.

## 1. Core Components Overview

```
┌───────────────────────┐
│                       │
│       JavaScript      │
│      (Your Code)      │
│                       │
└───────────┬───────────┘
            │
┌───────────▼───────────┐
│                       │
│      Event Loop       │
│                       │
└───────────┬───────────┘
            │
┌───────────▼───────────┐
│                       │
│   Thread Pool (libuv) │
│   (for heavy tasks)   │
│                       │
└───────────────────────┘
```

```
![image](https://github.com/ShahHayat/node.js/attachments/architecture.png)

```

## 2. Detailed Event Loop Phases

The event loop has several phases, each with its own queue:

```
┌───────────────────────┐
│        timers         │ <── setTimeout, setInterval
└───────────┬───────────┘
            │
┌───────────▼───────────┐
│   pending callbacks   │ <── I/O callbacks deferred to next loop
└───────────┬───────────┘
            │
┌───────────▼───────────┐
│     idle, prepare     │ <── internal use only
└───────────┬───────────┘
            │
┌───────────▼───────────┐
│        poll           │ <── retrieve new I/O events
└───────────┬───────────┘
            │
┌───────────▼───────────┐
│        check          │ <── setImmediate callbacks
└───────────┬───────────┘
            │
┌───────────▼───────────┐
│     close callbacks   │ <── socket.on('close', ...)
└───────────────────────┘
```

## 3. Task Queue Flow Diagram

Here's how different tasks get processed:

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐    │
│  │             │    │             │    │                 │    │
│  │  Next Tick  │    │  Microtask  │    │  Macrotask      │    │
│  │  Queue      │    │  Queue      │    │  Queue          │    │
│  │ (process.   │    │ (Promises)  │    │ (Timers, I/O,   │    │
│  │  nextTick)  │    │             │    │  setImmediate)  │    │
│  │             │    │             │    │                 │    │
│  └──────┬──────┘    └──────┬──────┘    └────────┬────────┘    │
│         │                  │                    │             │
│         └────────┬─────────┘                    │             │
│                  │                              │             │
│         ┌────────▼───────────────┐      ┌───────▼───────┐     │
│         │                        │      │               │     │
│         │      Event Loop        │      │  Thread Pool  │     │
│         │                        │      │  (for fs,     │     │
│         └────────┬───────────────┘      │  crypto, etc) │     │
│                  │                      │               │     │
│         ┌────────▼───────────────┐      └───────┬───────┘     │
│         │                        │              │             │
│         │   Completed I/O        ◄──────────────┘             │
│         │   Operations           │                            │
│         └────────────────────────┘                            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

## 4. Sequence of Operations

1. **Timers Phase**: Executes callbacks scheduled by `setTimeout()` and `setInterval()`
2. **Pending Callbacks**: Executes I/O callbacks deferred from previous loop
3. **Idle/Prepare**: Internal housekeeping
4. **Poll Phase**:
   - Retrieves new I/O events
   - Executes I/O related callbacks
   - May block here waiting for I/O
5. **Check Phase**: `setImmediate()` callbacks are executed here
6. **Close Callbacks**: Execute close event callbacks (e.g., `socket.on('close')`)

## 5. Practical Example Flow

Consider this code:

```javascript
const fs = require('fs');

console.log('Start');

setTimeout(() => console.log('Timeout 1'), 0);
setImmediate(() => console.log('Immediate 1'));

fs.readFile('file.txt', () => {
  console.log('File read');
  setTimeout(() => console.log('Timeout 2'), 0);
  setImmediate(() => console.log('Immediate 2'));
});

process.nextTick(() => console.log('Next Tick'));

Promise.resolve().then(() => console.log('Promise'));

console.log('End');
```

Execution flow:

```
┌───────────────────────────────┐
│          Call Stack           │
├───────────────────────────────┤
│ console.log('Start')          │
│ console.log('End')            │
└───────────────┬───────────────┘
                │
┌───────────────▼───────────────┐
│        Next Tick Queue        │
├───────────────────────────────┤
│ process.nextTick callback     │
└───────────────┬───────────────┘
                │
┌───────────────▼───────────────┐
│      Microtask Queue          │
├───────────────────────────────┤
│ Promise callback              │
└───────────────┬───────────────┘
                │
┌───────────────▼───────────────┐
│        Timers Phase           │
├───────────────────────────────┤
│ setTimeout(0) callback        │
└───────────────┬───────────────┘
                │
┌───────────────▼───────────────┐
│        Check Phase            │
├───────────────────────────────┤
│ setImmediate callback         │
└───────────────┬───────────────┘
                │
┌───────────────▼───────────────┐
│   I/O Callback (file read)    │
├───────────────────────────────┤
│ fs.readFile callback          │
│   - console.log('File read')  │
│   - Adds new timer            │
│   - Adds new immediate        │
└───────────────┬───────────────┘
                │
┌───────────────▼───────────────┐
│        Next Tick Queue        │
├───────────────────────────────┤
│ (empty)                       │
└───────────────┬───────────────┘
                │
┌───────────────▼───────────────┐
│      Microtask Queue          │
├───────────────────────────────┤
│ (empty)                       │
└───────────────┬───────────────┘
                │
┌───────────────▼───────────────┐
│        Timers Phase           │
├───────────────────────────────┤
│ setTimeout(0) from file cb    │
└───────────────┬───────────────┘
                │
┌───────────────▼───────────────┐
│        Check Phase            │
├───────────────────────────────┤
│ setImmediate from file cb     │
└───────────────────────────────┘
```

Output order:
```
Start
End
Next Tick
Promise
Timeout 1
Immediate 1
File read
Immediate 2
Timeout 2
```

## 6. Key Takeaways

1. **Priority Order**:
   - Next tick queue (highest priority)
   - Microtask queue (Promise callbacks)
   - Macrotask queue (timers, I/O, setImmediate)

2. **Event Loop**:
   - Processes one phase at a time
   - Moves to next phase only when current queue is empty
   - Between each phase, processes nextTick and microtasks

3. **Thread Pool**:
   - Used for operations that can't be done asynchronously at OS level
   - Default size is 4, can be changed with `UV_THREADPOOL_SIZE`

4. **Non-blocking I/O**:
   - When possible, uses OS async interfaces (epoll, kqueue, IOCP)
   - Falls back to thread pool when OS async isn't available

This architecture allows Node.js to handle thousands of concurrent connections with a single thread, making it highly efficient for I/O-bound applications.
