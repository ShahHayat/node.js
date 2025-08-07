# Node.js Architecture: In-Depth Technical Notes

## Table of Contents
1. [Introduction & Overview](#introduction--overview)
2. [Core Architecture Components](#core-architecture-components)
3. [Event Loop Deep Dive](#event-loop-deep-dive)
4. [V8 JavaScript Engine](#v8-javascript-engine)
5. [libuv Library](#libuv-library)
6. [Node.js Core Modules](#nodejs-core-modules)
7. [Memory Management](#memory-management)
8. [Threading Model](#threading-model)
9. [File System Operations](#file-system-operations)
10. [Network Architecture](#network-architecture)
11. [Stream Architecture](#stream-architecture)
12. [Module System](#module-system)
13. [Process & Child Processes](#process--child-processes)
14. [Clustering & Load Balancing](#clustering--load-balancing)
15. [Performance Optimization](#performance-optimization)
16. [Security Architecture](#security-architecture)
17. [Debugging & Profiling](#debugging--profiling)
18. [Best Practices & Patterns](#best-practices--patterns)
19. [🚀 Request Lifecycle: From Client to Response](#request-lifecycle-from-client-to-response)

---

## Introduction & Overview

### What is Node.js?
Node.js is a JavaScript runtime built on Chrome's V8 JavaScript engine that allows JavaScript to be executed on the server-side. It uses an event-driven, non-blocking I/O model that makes it lightweight and efficient.

### Key Characteristics
- **Single-threaded**: Uses a single main thread for JavaScript execution
- **Event-driven**: Built around an event loop mechanism
- **Non-blocking I/O**: Operations don't block the main thread
- **Asynchronous**: Heavy use of callbacks, promises, and async/await
- **Cross-platform**: Runs on Windows, macOS, Linux, and other platforms

### Architecture Stack
```
┌─────────────────────────────────────┐
│        JavaScript Application       │
├─────────────────────────────────────┤
│          Node.js Core APIs          │
├─────────────────────────────────────┤
│          Node.js Bindings           │
├─────────────────────────────────────┤
│     V8 Engine    │     libuv        │
├──────────────────┼──────────────────┤
│   C/C++ Core     │ Operating System │
└─────────────────────────────────────┘
```

---

## Core Architecture Components

### 1. V8 JavaScript Engine
- **Purpose**: Compiles and executes JavaScript code
- **Key Features**:
  - Just-In-Time (JIT) compilation
  - Hidden class optimization
  - Inline caching
  - Garbage collection
  - Memory management

### 2. libuv Library
- **Purpose**: Cross-platform asynchronous I/O library
- **Responsibilities**:
  - Event loop implementation
  - Thread pool management
  - File system operations
  - Network operations
  - Timer management

### 3. Node.js Bindings
- **Purpose**: Bridge between JavaScript and C/C++ code
- **Components**:
  - C++ addons
  - Native modules
  - Buffer management
  - Error handling

### 4. Core Modules
- **Built-in modules**: fs, http, crypto, path, os, etc.
- **Native modules**: Written in C/C++ for performance
- **JavaScript modules**: Higher-level abstractions

---

## Event Loop Deep Dive

### Event Loop Phases

The event loop is the heart of Node.js architecture, consisting of 6 main phases:

```
   ┌───────────────────────────┐
┌─>│           timers          │  ── Executes setTimeout() and setInterval() callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  ── Executes I/O callbacks deferred to the next loop iteration
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  ── Only used internally
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  ── Fetch new I/O events; execute I/O related callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  ── setImmediate() callbacks are invoked here
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  ── e.g. socket.on('close', ...)
   └───────────────────────────┘
```

### Phase Details

#### 1. Timers Phase
```javascript
// Example: Timer callbacks execution
setTimeout(() => {
    console.log('Timer callback executed');
}, 0);

setInterval(() => {
    console.log('Interval callback executed');
}, 1000);
```

#### 2. Pending Callbacks Phase
- Executes callbacks for some system operations (TCP errors, etc.)
- Handles callbacks deferred from previous iterations

#### 3. Idle, Prepare Phase
- Internal use only
- Prepares for polling

#### 4. Poll Phase
```javascript
// Example: I/O operations
const fs = require('fs');

fs.readFile('file.txt', (err, data) => {
    // This callback executes in the poll phase
    console.log('File read complete');
});
```

#### 5. Check Phase
```javascript
// setImmediate callbacks execute here
setImmediate(() => {
    console.log('setImmediate callback');
});
```

#### 6. Close Callbacks Phase
```javascript
// Connection close callbacks
server.on('close', () => {
    console.log('Server closed');
});
```

### Process.nextTick() and Promises

```javascript
// Execution order demonstration
console.log('Start');

setTimeout(() => console.log('Timer'), 0);
setImmediate(() => console.log('Immediate'));

process.nextTick(() => console.log('nextTick 1'));
Promise.resolve().then(() => console.log('Promise 1'));

process.nextTick(() => console.log('nextTick 2'));
Promise.resolve().then(() => console.log('Promise 2'));

console.log('End');

// Output:
// Start
// End
// nextTick 1
// nextTick 2
// Promise 1
// Promise 2
// Timer
// Immediate
```

---

## V8 JavaScript Engine

### Architecture Components

#### 1. Parser
- Converts JavaScript source code into Abstract Syntax Tree (AST)
- Performs syntax validation
- Handles ES6+ features

#### 2. Ignition Interpreter
- Bytecode interpreter introduced in V8 5.9
- Generates bytecode from AST
- Reduces memory usage

#### 3. TurboFan Compiler
- Optimizing compiler for hot code paths
- Performs advanced optimizations
- Can deoptimize code when assumptions are violated

```javascript
// Example of optimization
function add(a, b) {
    return a + b; // V8 optimizes this for specific types
}

// First few calls - interpreted
add(1, 2);
add(3, 4);

// After many calls with integers - optimized
for (let i = 0; i < 10000; i++) {
    add(i, i + 1); // TurboFan optimizes for integer addition
}

// If called with different types - deoptimized
add("hello", "world"); // Causes deoptimization
```

### Memory Layout

```
┌─────────────────────────────────────┐
│              Heap                   │
├─────────────────────────────────────┤
│           New Space                 │  ── Young generation (short-lived objects)
│  ┌─────────────┬─────────────────┐  │
│  │    From     │       To        │  │
│  │   Space     │     Space       │  │
│  └─────────────┴─────────────────┘  │
├─────────────────────────────────────┤
│           Old Space                 │  ── Old generation (long-lived objects)
├─────────────────────────────────────┤
│           Code Space                │  ── Compiled code
├─────────────────────────────────────┤
│           Map Space                 │  ── Hidden classes/maps
├─────────────────────────────────────┤
│         Large Object Space          │  ── Large objects (>512KB)
└─────────────────────────────────────┘
```

### Garbage Collection

#### Generational Collection
- **Minor GC**: Scavenger for new space (fast, frequent)
- **Major GC**: Mark-and-sweep for old space (slower, less frequent)

```javascript
// Memory management example
function createObjects() {
    let objects = [];
    for (let i = 0; i < 1000000; i++) {
        objects.push({ id: i, data: 'some data' });
    }
    return objects; // Objects may be promoted to old space
}

// Force garbage collection (for testing only)
if (global.gc) {
    global.gc();
}
```

---

## libuv Library

### Architecture Overview

libuv is the C library that provides Node.js with:
- Event loop
- Asynchronous I/O operations
- Thread pool management
- Platform abstraction

### Key Components

#### 1. Event Loop Implementation
```c
// Simplified event loop structure
typedef struct uv_loop_s {
    void* data;
    unsigned int active_handles;
    void* handle_queue[2];
    void* active_reqs[2];
    unsigned int stop_flag;
    // ... more fields
} uv_loop_t;
```

#### 2. Handle Types
- **uv_tcp_t**: TCP streams
- **uv_udp_t**: UDP sockets
- **uv_pipe_t**: Named pipes
- **uv_timer_t**: Timers
- **uv_fs_event_t**: File system events

#### 3. Request Types
- **uv_fs_t**: File system operations
- **uv_getaddrinfo_t**: DNS resolution
- **uv_work_t**: Thread pool work

### Thread Pool

```javascript
// Thread pool usage examples
const fs = require('fs');
const crypto = require('crypto');

// File operations use thread pool
fs.readFile('large-file.txt', (err, data) => {
    console.log('File read completed');
});

// CPU-intensive operations use thread pool
crypto.pbkdf2('password', 'salt', 100000, 64, 'sha512', (err, key) => {
    console.log('Key derivation completed');
});

// Configure thread pool size
process.env.UV_THREADPOOL_SIZE = 8; // Default is 4
```

---

## Node.js Core Modules

### Built-in Module Categories

#### 1. File System (fs)
```javascript
const fs = require('fs');
const path = require('path');

// Asynchronous operations
fs.readFile(path.join(__dirname, 'file.txt'), 'utf8', (err, data) => {
    if (err) throw err;
    console.log(data);
});

// Synchronous operations (blocking)
try {
    const data = fs.readFileSync('file.txt', 'utf8');
    console.log(data);
} catch (err) {
    console.error(err);
}

// Promise-based operations
const fsPromises = require('fs').promises;

async function readFileAsync() {
    try {
        const data = await fsPromises.readFile('file.txt', 'utf8');
        console.log(data);
    } catch (err) {
        console.error(err);
    }
}
```

#### 2. HTTP Module
```javascript
const http = require('http');
const url = require('url');

// Create HTTP server
const server = http.createServer((req, res) => {
    const parsedUrl = url.parse(req.url, true);
    
    res.writeHead(200, {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    });
    
    res.end(JSON.stringify({
        method: req.method,
        path: parsedUrl.pathname,
        query: parsedUrl.query
    }));
});

server.listen(3000, () => {
    console.log('Server listening on port 3000');
});

// HTTP client
const options = {
    hostname: 'api.example.com',
    port: 443,
    path: '/data',
    method: 'GET',
    headers: {
        'Authorization': 'Bearer token'
    }
};

const req = http.request(options, (res) => {
    let data = '';
    
    res.on('data', (chunk) => {
        data += chunk;
    });
    
    res.on('end', () => {
        console.log(JSON.parse(data));
    });
});

req.on('error', (err) => {
    console.error(err);
});

req.end();
```

#### 3. Crypto Module
```javascript
const crypto = require('crypto');

// Hashing
const hash = crypto.createHash('sha256');
hash.update('data to hash');
const hashDigest = hash.digest('hex');

// Symmetric encryption
const algorithm = 'aes-256-cbc';
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);

function encrypt(text) {
    const cipher = crypto.createCipher(algorithm, key, iv);
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return encrypted;
}

function decrypt(encryptedText) {
    const decipher = crypto.createDecipher(algorithm, key, iv);
    let decrypted = decipher.update(encryptedText, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
}

// Asymmetric encryption
const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
    modulusLength: 2048,
    publicKeyEncoding: {
        type: 'spki',
        format: 'pem'
    },
    privateKeyEncoding: {
        type: 'pkcs8',
        format: 'pem'
    }
});
```

---

## Memory Management

### Heap Structure and Management

#### Memory Regions
1. **Stack**: Function calls, local variables, primitive types
2. **Heap**: Objects, arrays, closures
3. **Code Space**: Compiled JavaScript code
4. **Native Memory**: C++ objects, buffers

```javascript
// Memory monitoring
const process = require('process');

function getMemoryUsage() {
    const usage = process.memoryUsage();
    
    console.log({
        rss: `${Math.round(usage.rss / 1024 / 1024)} MB`,      // Resident Set Size
        heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)} MB`,
        heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)} MB`,
        external: `${Math.round(usage.external / 1024 / 1024)} MB`
    });
}

// Memory leak detection
function potentialMemoryLeak() {
    const leaks = [];
    
    setInterval(() => {
        // This creates a memory leak
        leaks.push(new Array(1000000).fill('leak'));
        getMemoryUsage();
    }, 1000);
}

// Buffer management
const buffer1 = Buffer.alloc(1024);        // Initialized with zeros
const buffer2 = Buffer.allocUnsafe(1024);  // Uninitialized (faster)
const buffer3 = Buffer.from('hello');      // From string
```

### Garbage Collection Strategies

```javascript
// Weak references for cache implementation
const cache = new WeakMap();

function cacheData(obj, data) {
    cache.set(obj, data);
}

function getCachedData(obj) {
    return cache.get(obj);
}

// Object will be garbage collected even if in WeakMap
let key = { id: 1 };
cacheData(key, { expensive: 'computation' });
key = null; // Object can now be garbage collected
```

---

## Threading Model

### Single-Threaded Event Loop with Thread Pool

Node.js uses a hybrid threading model:

```
Main Thread (JavaScript)
├── Event Loop
├── Call Stack
└── Callback Queue

Thread Pool (libuv)
├── File I/O
├── DNS Resolution
├── CPU-intensive tasks
└── Custom C++ operations
```

### Worker Threads (Node.js 10+)

```javascript
// main.js
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
    // Main thread
    console.log('Main thread starting');
    
    const worker = new Worker(__filename, {
        workerData: { start: 0, end: 1000000 }
    });
    
    worker.on('message', (result) => {
        console.log('Result from worker:', result);
    });
    
    worker.on('error', (err) => {
        console.error('Worker error:', err);
    });
    
    worker.on('exit', (code) => {
        console.log(`Worker exited with code ${code}`);
    });
} else {
    // Worker thread
    const { start, end } = workerData;
    
    function fibonacci(n) {
        if (n < 2) return n;
        return fibonacci(n - 1) + fibonacci(n - 2);
    }
    
    let sum = 0;
    for (let i = start; i < end; i++) {
        sum += fibonacci(i % 40); // CPU-intensive task
    }
    
    parentPort.postMessage(sum);
}
```

### Cluster Module

```javascript
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
    console.log(`Master ${process.pid} is running`);
    
    // Fork workers
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }
    
    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died`);
        cluster.fork(); // Restart worker
    });
    
    cluster.on('online', (worker) => {
        console.log(`Worker ${worker.process.pid} is online`);
    });
} else {
    // Worker process
    http.createServer((req, res) => {
        res.writeHead(200);
        res.end(`Hello from worker ${process.pid}\n`);
    }).listen(8000);
    
    console.log(`Worker ${process.pid} started`);
}
```

---

## File System Operations

### File System Architecture

```javascript
const fs = require('fs');
const path = require('path');
const { pipeline } = require('stream');

// Different approaches to file operations

// 1. Callback-based (traditional)
fs.readFile('input.txt', 'utf8', (err, data) => {
    if (err) {
        console.error('Error reading file:', err);
        return;
    }
    
    // Process data
    const processedData = data.toUpperCase();
    
    fs.writeFile('output.txt', processedData, (err) => {
        if (err) {
            console.error('Error writing file:', err);
            return;
        }
        console.log('File written successfully');
    });
});

// 2. Promise-based
const fsPromises = require('fs').promises;

async function processFile() {
    try {
        const data = await fsPromises.readFile('input.txt', 'utf8');
        const processedData = data.toUpperCase();
        await fsPromises.writeFile('output.txt', processedData);
        console.log('File processed successfully');
    } catch (err) {
        console.error('Error processing file:', err);
    }
}

// 3. Stream-based (for large files)
function processLargeFile() {
    const readStream = fs.createReadStream('large-input.txt', { encoding: 'utf8' });
    const writeStream = fs.createWriteStream('large-output.txt');
    
    pipeline(
        readStream,
        require('stream').Transform({
            transform(chunk, encoding, callback) {
                this.push(chunk.toString().toUpperCase());
                callback();
            }
        }),
        writeStream,
        (err) => {
            if (err) {
                console.error('Pipeline failed:', err);
            } else {
                console.log('Pipeline succeeded');
            }
        }
    );
}
```

### File System Watching

```javascript
const fs = require('fs');
const path = require('path');

// Watch file changes
fs.watchFile('config.json', (curr, prev) => {
    console.log('File modified:', {
        previous: prev.mtime,
        current: curr.mtime
    });
});

// Watch directory changes (more efficient)
fs.watch('.', { recursive: true }, (eventType, filename) => {
    if (filename) {
        console.log(`File ${filename} was ${eventType}`);
    }
});

// More robust file watching with error handling
const watcher = fs.watch('./src', { recursive: true });

watcher.on('change', (eventType, filename) => {
    console.log(`File ${filename} changed: ${eventType}`);
});

watcher.on('error', (err) => {
    console.error('Watcher error:', err);
});

// Close watcher when done
process.on('SIGINT', () => {
    watcher.close();
    process.exit(0);
});
```

---

## Network Architecture

### HTTP/HTTPS Implementation

```javascript
const http = require('http');
const https = require('https');
const http2 = require('http2');
const fs = require('fs');

// HTTP/1.1 Server
const httpServer = http.createServer((req, res) => {
    // Handle request
    let body = '';
    
    req.on('data', chunk => {
        body += chunk.toString();
    });
    
    req.on('end', () => {
        res.writeHead(200, {
            'Content-Type': 'application/json',
            'Connection': 'keep-alive'
        });
        
        res.end(JSON.stringify({
            method: req.method,
            url: req.url,
            headers: req.headers,
            body: body
        }));
    });
});

// HTTPS Server
const httpsOptions = {
    key: fs.readFileSync('private-key.pem'),
    cert: fs.readFileSync('certificate.pem')
};

const httpsServer = https.createServer(httpsOptions, (req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Secure connection established');
});

// HTTP/2 Server
const http2Server = http2.createSecureServer(httpsOptions);

http2Server.on('stream', (stream, headers) => {
    stream.respond({
        'content-type': 'text/html',
        ':status': 200
    });
    
    stream.end('<h1>Hello HTTP/2</h1>');
});

// TCP Server
const net = require('net');

const tcpServer = net.createServer((socket) => {
    console.log('Client connected');
    
    socket.on('data', (data) => {
        console.log('Received:', data.toString());
        socket.write('Echo: ' + data);
    });
    
    socket.on('close', () => {
        console.log('Client disconnected');
    });
    
    socket.on('error', (err) => {
        console.error('Socket error:', err);
    });
});

// UDP Server
const dgram = require('dgram');

const udpServer = dgram.createSocket('udp4');

udpServer.on('message', (msg, rinfo) => {
    console.log(`Received ${msg} from ${rinfo.address}:${rinfo.port}`);
    
    udpServer.send(
        `Echo: ${msg}`,
        rinfo.port,
        rinfo.address,
        (err) => {
            if (err) console.error('Send error:', err);
        }
    );
});
```

### Advanced Networking Features

```javascript
// Keep-alive connections
const http = require('http');
const { Agent } = require('http');

const keepAliveAgent = new Agent({
    keepAlive: true,
    keepAliveMsecs: 1000,
    maxSockets: 50,
    maxFreeSockets: 10,
    timeout: 60000
});

// WebSocket implementation
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws, req) => {
    console.log('WebSocket connection established');
    
    ws.on('message', (message) => {
        console.log('Received:', message);
        
        // Broadcast to all clients
        wss.clients.forEach((client) => {
            if (client !== ws && client.readyState === WebSocket.OPEN) {
                client.send(message);
            }
        });
    });
    
    ws.on('close', () => {
        console.log('WebSocket connection closed');
    });
});
```

---

## Stream Architecture

### Stream Types and Implementation

Node.js provides four fundamental stream types:

```javascript
const { Readable, Writable, Transform, Duplex } = require('stream');

// 1. Readable Stream
class NumberGenerator extends Readable {
    constructor(max) {
        super({ objectMode: true });
        this.current = 0;
        this.max = max;
    }
    
    _read() {
        if (this.current < this.max) {
            this.push({ number: this.current++ });
        } else {
            this.push(null); // End stream
        }
    }
}

// 2. Writable Stream
class NumberLogger extends Writable {
    constructor() {
        super({ objectMode: true });
    }
    
    _write(chunk, encoding, callback) {
        console.log(`Logging: ${chunk.number}`);
        callback();
    }
}

// 3. Transform Stream
class NumberDoubler extends Transform {
    constructor() {
        super({ objectMode: true });
    }
    
    _transform(chunk, encoding, callback) {
        this.push({ number: chunk.number * 2 });
        callback();
    }
}

// 4. Duplex Stream
class EchoStream extends Duplex {
    constructor() {
        super({ objectMode: true });
        this.buffer = [];
    }
    
    _read() {
        if (this.buffer.length > 0) {
            this.push(this.buffer.shift());
        }
    }
    
    _write(chunk, encoding, callback) {
        this.buffer.push(chunk);
        callback();
    }
}

// Usage example
const generator = new NumberGenerator(5);
const doubler = new NumberDoubler();
const logger = new NumberLogger();

generator
    .pipe(doubler)
    .pipe(logger);
```

### Stream Pipeline and Error Handling

```javascript
const { pipeline } = require('stream');
const fs = require('fs');
const zlib = require('zlib');

// Safe pipeline with error handling
pipeline(
    fs.createReadStream('input.txt'),
    zlib.createGzip(),
    fs.createWriteStream('output.txt.gz'),
    (err) => {
        if (err) {
            console.error('Pipeline failed:', err);
        } else {
            console.log('Pipeline succeeded');
        }
    }
);

// Async pipeline
const { pipeline: asyncPipeline } = require('stream/promises');

async function compressFile() {
    try {
        await asyncPipeline(
            fs.createReadStream('input.txt'),
            zlib.createGzip(),
            fs.createWriteStream('output.txt.gz')
        );
        console.log('File compressed successfully');
    } catch (err) {
        console.error('Compression failed:', err);
    }
}
```

---

## Module System

### CommonJS Module System

```javascript
// math.js - Module definition
const PI = 3.14159;

function add(a, b) {
    return a + b;
}

function multiply(a, b) {
    return a * b;
}

class Calculator {
    constructor() {
        this.history = [];
    }
    
    calculate(operation, a, b) {
        let result;
        switch (operation) {
            case 'add':
                result = add(a, b);
                break;
            case 'multiply':
                result = multiply(a, b);
                break;
            default:
                throw new Error('Unknown operation');
        }
        
        this.history.push({ operation, a, b, result });
        return result;
    }
}

// Export patterns
module.exports = {
    PI,
    add,
    multiply,
    Calculator
};

// Alternative export patterns
// module.exports.add = add;
// exports.PI = PI;
```

```javascript
// app.js - Module consumption
const math = require('./math');
const { add, Calculator } = require('./math');

console.log(math.PI);
console.log(math.add(5, 3));

const calc = new Calculator();
console.log(calc.calculate('add', 10, 20));
```

### ES Modules (ESM)

```javascript
// math.mjs or math.js with "type": "module" in package.json
export const PI = 3.14159;

export function add(a, b) {
    return a + b;
}

export function multiply(a, b) {
    return a * b;
}

export default class Calculator {
    constructor() {
        this.history = [];
    }
    
    calculate(operation, a, b) {
        // Implementation
    }
}
```

```javascript
// app.mjs
import Calculator, { add, PI } from './math.mjs';
import * as math from './math.mjs';

console.log(PI);
console.log(add(5, 3));

const calc = new Calculator();
```

### Module Resolution Algorithm

```
require('./math')
├── 1. Check if './math' is a file
│   ├── ./math.js
│   ├── ./math.json
│   └── ./math.node
├── 2. Check if './math' is a directory
│   ├── ./math/package.json (main field)
│   └── ./math/index.js
└── 3. Throw MODULE_NOT_FOUND error
```

### Module Caching

```javascript
// singleton.js
console.log('Module loaded'); // Only runs once

let instance = null;

class Singleton {
    constructor() {
        if (instance) {
            return instance;
        }
        instance = this;
        this.data = Math.random();
    }
}

module.exports = Singleton;

// Usage
const Singleton1 = require('./singleton');
const Singleton2 = require('./singleton');

const s1 = new Singleton1();
const s2 = new Singleton2();

console.log(s1 === s2); // true (same instance)
console.log(s1.data === s2.data); // true
```

---

## Process & Child Processes

### Process Object

```javascript
// Process information
console.log({
    pid: process.pid,
    ppid: process.ppid,
    platform: process.platform,
    arch: process.arch,
    version: process.version,
    argv: process.argv,
    env: process.env,
    cwd: process.cwd(),
    uptime: process.uptime()
});

// Process events
process.on('exit', (code) => {
    console.log(`Process exiting with code: ${code}`);
});

process.on('uncaughtException', (err) => {
    console.error('Uncaught Exception:', err);
    process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection at:', promise, 'reason:', reason);
    process.exit(1);
});

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('Received SIGTERM, shutting down gracefully');
    server.close(() => {
        process.exit(0);
    });
});

process.on('SIGINT', () => {
    console.log('Received SIGINT, shutting down gracefully');
    process.exit(0);
});
```

### Child Processes

```javascript
const { spawn, exec, execFile, fork } = require('child_process');

// 1. spawn - Low-level process creation
const ls = spawn('ls', ['-la'], {
    cwd: '/tmp',
    env: { ...process.env, CUSTOM_VAR: 'value' }
});

ls.stdout.on('data', (data) => {
    console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
    console.error(`stderr: ${data}`);
});

ls.on('close', (code) => {
    console.log(`Child process exited with code ${code}`);
});

// 2. exec - Execute shell commands
exec('cat /etc/passwd | grep root', (error, stdout, stderr) => {
    if (error) {
        console.error(`exec error: ${error}`);
        return;
    }
    console.log(`stdout: ${stdout}`);
    console.error(`stderr: ${stderr}`);
});

// 3. execFile - Execute specific files
execFile('node', ['--version'], (error, stdout, stderr) => {
    if (error) {
        console.error(`execFile error: ${error}`);
        return;
    }
    console.log(`Node version: ${stdout}`);
});

// 4. fork - Create child Node.js processes
const worker = fork('./worker.js', ['arg1', 'arg2']);

worker.send({ type: 'task', data: 'process this' });

worker.on('message', (message) => {
    console.log('Message from worker:', message);
});

worker.on('exit', (code) => {
    console.log(`Worker exited with code ${code}`);
});
```

```javascript
// worker.js - Child process implementation
process.on('message', (message) => {
    console.log('Received from parent:', message);
    
    if (message.type === 'task') {
        // Process the task
        const result = message.data.toUpperCase();
        
        process.send({
            type: 'result',
            data: result
        });
    }
});

// Signal when ready
process.send({ type: 'ready' });
```

---

## Clustering & Load Balancing

### Cluster Module Deep Dive

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

const numWorkers = os.cpus().length;

if (cluster.isMaster) {
    console.log(`Master process ${process.pid} is running`);
    console.log(`Forking ${numWorkers} workers`);
    
    // Fork workers
    for (let i = 0; i < numWorkers; i++) {
        const worker = cluster.fork();
        
        worker.on('online', () => {
            console.log(`Worker ${worker.process.pid} is online`);
        });
        
        worker.on('exit', (code, signal) => {
            console.log(`Worker ${worker.process.pid} died with code ${code} and signal ${signal}`);
            console.log('Starting a new worker');
            cluster.fork();
        });
    }
    
    // Handle worker messages
    cluster.on('message', (worker, message) => {
        console.log(`Message from worker ${worker.process.pid}:`, message);
    });
    
    // Graceful shutdown
    process.on('SIGTERM', () => {
        console.log('Master received SIGTERM, shutting down workers');
        
        for (const id in cluster.workers) {
            cluster.workers[id].kill();
        }
    });
    
} else {
    // Worker process
    const server = http.createServer((req, res) => {
        // Simulate some work
        const start = Date.now();
        
        while (Date.now() - start < 100) {
            // CPU intensive task
        }
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
            worker: process.pid,
            timestamp: new Date().toISOString(),
            uptime: process.uptime()
        }));
    });
    
    server.listen(3000, () => {
        console.log(`Worker ${process.pid} listening on port 3000`);
    });
    
    // Report to master
    process.send({ type: 'worker-status', pid: process.pid, status: 'ready' });
}
```

### Load Balancing Strategies

```javascript
// Round-robin load balancer (manual implementation)
class LoadBalancer {
    constructor(workers) {
        this.workers = workers;
        this.current = 0;
    }
    
    getNextWorker() {
        const worker = this.workers[this.current];
        this.current = (this.current + 1) % this.workers.length;
        return worker;
    }
    
    addWorker(worker) {
        this.workers.push(worker);
    }
    
    removeWorker(worker) {
        const index = this.workers.indexOf(worker);
        if (index > -1) {
            this.workers.splice(index, 1);
            if (this.current >= this.workers.length) {
                this.current = 0;
            }
        }
    }
}

// Health checking
class HealthChecker {
    constructor(workers) {
        this.workers = workers;
        this.healthyWorkers = new Set(workers);
    }
    
    async checkHealth() {
        for (const worker of this.workers) {
            try {
                // Send health check message
                worker.send({ type: 'health-check' });
                
                // Wait for response with timeout
                const isHealthy = await this.waitForHealthResponse(worker, 5000);
                
                if (isHealthy) {
                    this.healthyWorkers.add(worker);
                } else {
                    this.healthyWorkers.delete(worker);
                }
            } catch (err) {
                console.error(`Health check failed for worker ${worker.process.pid}:`, err);
                this.healthyWorkers.delete(worker);
            }
        }
    }
    
    waitForHealthResponse(worker, timeout) {
        return new Promise((resolve, reject) => {
            const timer = setTimeout(() => {
                resolve(false);
            }, timeout);
            
            const handler = (message) => {
                if (message.type === 'health-response') {
                    clearTimeout(timer);
                    worker.removeListener('message', handler);
                    resolve(true);
                }
            };
            
            worker.on('message', handler);
        });
    }
    
    getHealthyWorkers() {
        return Array.from(this.healthyWorkers);
    }
}
```

---

## Performance Optimization

### Profiling and Monitoring

```javascript
// Performance monitoring
const performanceHooks = require('perf_hooks');

// Performance observer
const obs = new performanceHooks.PerformanceObserver((list) => {
    const entries = list.getEntries();
    entries.forEach((entry) => {
        console.log(`${entry.name}: ${entry.duration}ms`);
    });
});

obs.observe({ entryTypes: ['measure', 'mark'] });

// Mark performance points
performanceHooks.performance.mark('start-operation');

// Simulate some work
setTimeout(() => {
    performanceHooks.performance.mark('end-operation');
    performanceHooks.performance.measure(
        'operation-duration',
        'start-operation',
        'end-operation'
    );
}, 1000);

// Function timing decorator
function timed(target, propertyKey, descriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = function(...args) {
        const start = process.hrtime.bigint();
        const result = originalMethod.apply(this, args);
        const end = process.hrtime.bigint();
        
        console.log(`${propertyKey} took ${Number(end - start) / 1000000}ms`);
        return result;
    };
    
    return descriptor;
}

class DataProcessor {
    @timed
    processData(data) {
        // CPU intensive operation
        return data.map(item => item * 2);
    }
}
```

### Memory Optimization

```javascript
// Object pooling
class ObjectPool {
    constructor(createFn, resetFn, initialSize = 10) {
        this.createFn = createFn;
        this.resetFn = resetFn;
        this.pool = [];
        
        // Pre-populate pool
        for (let i = 0; i < initialSize; i++) {
            this.pool.push(this.createFn());
        }
    }
    
    acquire() {
        if (this.pool.length > 0) {
            return this.pool.pop();
        }
        return this.createFn();
    }
    
    release(obj) {
        this.resetFn(obj);
        this.pool.push(obj);
    }
}

// Usage
const bufferPool = new ObjectPool(
    () => Buffer.alloc(1024),
    (buffer) => buffer.fill(0),
    50
);

function processData(data) {
    const buffer = bufferPool.acquire();
    
    try {
        // Use buffer for processing
        buffer.write(data);
        // ... processing logic
        
        return buffer.toString();
    } finally {
        bufferPool.release(buffer);
    }
}

// Stream optimization for large datasets
const { Transform } = require('stream');

class BatchProcessor extends Transform {
    constructor(batchSize = 100) {
        super({ objectMode: true });
        this.batch = [];
        this.batchSize = batchSize;
    }
    
    _transform(chunk, encoding, callback) {
        this.batch.push(chunk);
        
        if (this.batch.length >= this.batchSize) {
            this.processBatch();
        }
        
        callback();
    }
    
    _flush(callback) {
        if (this.batch.length > 0) {
            this.processBatch();
        }
        callback();
    }
    
    processBatch() {
        // Process batch efficiently
        const processedBatch = this.batch.map(item => {
            // Processing logic
            return item.toUpperCase();
        });
        
        this.push(processedBatch);
        this.batch = [];
    }
}
```

### CPU Optimization

```javascript
// Event loop lag monitoring
function measureEventLoopLag() {
    const start = process.hrtime();
    
    setImmediate(() => {
        const lag = process.hrtime(start);
        const lagMs = lag[0] * 1000 + lag[1] * 1e-6;
        
        if (lagMs > 10) {
            console.warn(`Event loop lag: ${lagMs.toFixed(2)}ms`);
        }
        
        // Schedule next measurement
        setTimeout(measureEventLoopLag, 1000);
    });
}

measureEventLoopLag();

// CPU-intensive task optimization
function optimizedFibonacci(n, memo = {}) {
    if (n in memo) return memo[n];
    if (n <= 2) return 1;
    
    memo[n] = optimizedFibonacci(n - 1, memo) + optimizedFibonacci(n - 2, memo);
    return memo[n];
}

// Break up CPU-intensive tasks
function processLargeArray(array, processor, batchSize = 1000) {
    return new Promise((resolve, reject) => {
        const results = [];
        let index = 0;
        
        function processBatch() {
            const end = Math.min(index + batchSize, array.length);
            
            try {
                for (let i = index; i < end; i++) {
                    results.push(processor(array[i]));
                }
                
                index = end;
                
                if (index < array.length) {
                    setImmediate(processBatch);
                } else {
                    resolve(results);
                }
            } catch (err) {
                reject(err);
            }
        }
        
        processBatch();
    });
}
```

---

## Security Architecture

### Security Best Practices

```javascript
// Input validation and sanitization
const validator = require('validator');
const DOMPurify = require('isomorphic-dompurify');

function validateAndSanitizeInput(input) {
    // Validate input format
    if (!input || typeof input !== 'string') {
        throw new Error('Invalid input format');
    }
    
    // Length validation
    if (input.length > 1000) {
        throw new Error('Input too long');
    }
    
    // Sanitize HTML
    const sanitized = DOMPurify.sanitize(input);
    
    // Additional validation
    if (!validator.isLength(sanitized, { min: 1, max: 1000 })) {
        throw new Error('Invalid input length');
    }
    
    return sanitized;
}

// Rate limiting
class RateLimiter {
    constructor(maxRequests = 100, windowMs = 60000) {
        this.maxRequests = maxRequests;
        this.windowMs = windowMs;
        this.requests = new Map();
    }
    
    isAllowed(identifier) {
        const now = Date.now();
        const windowStart = now - this.windowMs;
        
        if (!this.requests.has(identifier)) {
            this.requests.set(identifier, []);
        }
        
        const userRequests = this.requests.get(identifier);
        
        // Remove old requests
        while (userRequests.length > 0 && userRequests[0] < windowStart) {
            userRequests.shift();
        }
        
        if (userRequests.length >= this.maxRequests) {
            return false;
        }
        
        userRequests.push(now);
        return true;
    }
    
    reset(identifier) {
        this.requests.delete(identifier);
    }
}

// Secure headers middleware
function securityHeaders(req, res, next) {
    // Prevent clickjacking
    res.setHeader('X-Frame-Options', 'DENY');
    
    // XSS protection
    res.setHeader('X-XSS-Protection', '1; mode=block');
    
    // Content type sniffing protection
    res.setHeader('X-Content-Type-Options', 'nosniff');
    
    // Referrer policy
    res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
    
    // Content Security Policy
    res.setHeader('Content-Security-Policy', 
        "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'");
    
    // HSTS (for HTTPS)
    if (req.secure) {
        res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    }
    
    next();
}

// Authentication and authorization
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

class AuthService {
    constructor(secretKey) {
        this.secretKey = secretKey;
        this.saltRounds = 12;
    }
    
    async hashPassword(password) {
        return bcrypt.hash(password, this.saltRounds);
    }
    
    async verifyPassword(password, hash) {
        return bcrypt.compare(password, hash);
    }
    
    generateToken(payload, expiresIn = '1h') {
        return jwt.sign(payload, this.secretKey, { expiresIn });
    }
    
    verifyToken(token) {
        try {
            return jwt.verify(token, this.secretKey);
        } catch (err) {
            throw new Error('Invalid token');
        }
    }
    
    // Middleware for protecting routes
    authenticate(req, res, next) {
        const authHeader = req.headers.authorization;
        
        if (!authHeader || !authHeader.startsWith('Bearer ')) {
            return res.status(401).json({ error: 'No token provided' });
        }
        
        const token = authHeader.substring(7);
        
        try {
            const decoded = this.verifyToken(token);
            req.user = decoded;
            next();
        } catch (err) {
            return res.status(401).json({ error: 'Invalid token' });
        }
    }
    
    authorize(roles) {
        return (req, res, next) => {
            if (!req.user || !roles.includes(req.user.role)) {
                return res.status(403).json({ error: 'Insufficient permissions' });
            }
            next();
        };
    }
}
```

### Encryption and Data Protection

```javascript
const crypto = require('crypto');

class EncryptionService {
    constructor() {
        this.algorithm = 'aes-256-gcm';
        this.keyLength = 32;
        this.ivLength = 16;
        this.tagLength = 16;
    }
    
    generateKey() {
        return crypto.randomBytes(this.keyLength);
    }
    
    encrypt(plaintext, key) {
        const iv = crypto.randomBytes(this.ivLength);
        const cipher = crypto.createCipher(this.algorithm, key, iv);
        
        let encrypted = cipher.update(plaintext, 'utf8', 'hex');
        encrypted += cipher.final('hex');
        
        const tag = cipher.getAuthTag();
        
        return {
            encrypted,
            iv: iv.toString('hex'),
            tag: tag.toString('hex')
        };
    }
    
    decrypt(encryptedData, key) {
        const { encrypted, iv, tag } = encryptedData;
        
        const decipher = crypto.createDecipher(this.algorithm, key, Buffer.from(iv, 'hex'));
        decipher.setAuthTag(Buffer.from(tag, 'hex'));
        
        let decrypted = decipher.update(encrypted, 'hex', 'utf8');
        decrypted += decipher.final('utf8');
        
        return decrypted;
    }
    
    // Key derivation
    deriveKey(password, salt) {
        return crypto.pbkdf2Sync(password, salt, 100000, this.keyLength, 'sha512');
    }
    
    // Digital signatures
    sign(data, privateKey) {
        const sign = crypto.createSign('SHA256');
        sign.update(data);
        sign.end();
        return sign.sign(privateKey, 'hex');
    }
    
    verify(data, signature, publicKey) {
        const verify = crypto.createVerify('SHA256');
        verify.update(data);
        verify.end();
        return verify.verify(publicKey, signature, 'hex');
    }
}
```

---

## Debugging & Profiling

### Debugging Techniques

```javascript
// Built-in debugging
const util = require('util');

// Custom debug utility
function debug(namespace) {
    const enabled = process.env.DEBUG && process.env.DEBUG.includes(namespace);
    
    return function(...args) {
        if (enabled) {
            const timestamp = new Date().toISOString();
            console.log(`[${timestamp}] ${namespace}:`, ...args);
        }
    };
}

const dbDebug = debug('app:database');
const httpDebug = debug('app:http');

// Usage
dbDebug('Connecting to database');
httpDebug('Incoming request:', { method: 'GET', url: '/api/users' });

// Error tracking
class ErrorTracker {
    constructor() {
        this.errors = [];
        this.setupGlobalHandlers();
    }
    
    setupGlobalHandlers() {
        process.on('uncaughtException', (err) => {
            this.logError('uncaughtException', err);
            process.exit(1);
        });
        
        process.on('unhandledRejection', (reason, promise) => {
            this.logError('unhandledRejection', reason, { promise });
            process.exit(1);
        });
    }
    
    logError(type, error, context = {}) {
        const errorInfo = {
            type,
            message: error.message,
            stack: error.stack,
            timestamp: new Date().toISOString(),
            process: {
                pid: process.pid,
                memory: process.memoryUsage(),
                uptime: process.uptime()
            },
            context
        };
        
        this.errors.push(errorInfo);
        
        // Send to logging service
        this.sendToLoggingService(errorInfo);
    }
    
    async sendToLoggingService(errorInfo) {
        // Implementation for external logging service
        console.error('Error logged:', errorInfo);
    }
}

// Performance profiling
class Profiler {
    constructor() {
        this.profiles = new Map();
    }
    
    start(name) {
        this.profiles.set(name, {
            startTime: process.hrtime.bigint(),
            startMemory: process.memoryUsage()
        });
    }
    
    end(name) {
        const profile = this.profiles.get(name);
        if (!profile) {
            throw new Error(`No profile found for: ${name}`);
        }
        
        const endTime = process.hrtime.bigint();
        const endMemory = process.memoryUsage();
        
        const result = {
            name,
            duration: Number(endTime - profile.startTime) / 1000000, // Convert to ms
            memoryDelta: {
                rss: endMemory.rss - profile.startMemory.rss,
                heapTotal: endMemory.heapTotal - profile.startMemory.heapTotal,
                heapUsed: endMemory.heapUsed - profile.startMemory.heapUsed,
                external: endMemory.external - profile.startMemory.external
            }
        };
        
        this.profiles.delete(name);
        return result;
    }
    
    // Async profiling wrapper
    async profile(name, asyncFn) {
        this.start(name);
        try {
            const result = await asyncFn();
            const profileResult = this.end(name);
            console.log(`Profile ${name}:`, profileResult);
            return result;
        } catch (err) {
            this.end(name);
            throw err;
        }
    }
}

// Usage
const profiler = new Profiler();

async function expensiveOperation() {
    return profiler.profile('database-query', async () => {
        // Simulate database query
        await new Promise(resolve => setTimeout(resolve, 100));
        return { data: 'result' };
    });
}
```

### Advanced Debugging Tools

```javascript
// Async hooks for tracking async operations
const async_hooks = require('async_hooks');

class AsyncTracker {
    constructor() {
        this.asyncOperations = new Map();
        this.hook = async_hooks.createHook({
            init: this.init.bind(this),
            before: this.before.bind(this),
            after: this.after.bind(this),
            destroy: this.destroy.bind(this)
        });
    }
    
    enable() {
        this.hook.enable();
    }
    
    disable() {
        this.hook.disable();
    }
    
    init(asyncId, type, triggerAsyncId, resource) {
        this.asyncOperations.set(asyncId, {
            type,
            triggerAsyncId,
            created: Date.now(),
            stack: new Error().stack
        });
    }
    
    before(asyncId) {
        const op = this.asyncOperations.get(asyncId);
        if (op) {
            op.started = Date.now();
        }
    }
    
    after(asyncId) {
        const op = this.asyncOperations.get(asyncId);
        if (op && op.started) {
            op.duration = Date.now() - op.started;
        }
    }
    
    destroy(asyncId) {
        this.asyncOperations.delete(asyncId);
    }
    
    getStats() {
        const stats = {
            total: this.asyncOperations.size,
            byType: {}
        };
        
        for (const [id, op] of this.asyncOperations) {
            if (!stats.byType[op.type]) {
                stats.byType[op.type] = 0;
            }
            stats.byType[op.type]++;
        }
        
        return stats;
    }
}

// CPU profiling
const { Session } = require('inspector');

class CPUProfiler {
    constructor() {
        this.session = new Session();
        this.session.connect();
    }
    
    async startProfiling() {
        await this.session.post('Profiler.enable');
        await this.session.post('Profiler.start');
    }
    
    async stopProfiling() {
        const { profile } = await this.session.post('Profiler.stop');
        await this.session.post('Profiler.disable');
        return profile;
    }
    
    async profileFunction(fn) {
        await this.startProfiling();
        try {
            const result = await fn();
            const profile = await this.stopProfiling();
            return { result, profile };
        } catch (err) {
            await this.stopProfiling();
            throw err;
        }
    }
}
```

---

## Best Practices & Patterns

### Design Patterns

```javascript
// 1. Singleton Pattern
class Database {
    constructor() {
        if (Database.instance) {
            return Database.instance;
        }
        
        this.connection = null;
        Database.instance = this;
    }
    
    async connect() {
        if (!this.connection) {
            // Simulate database connection
            this.connection = { connected: true };
        }
        return this.connection;
    }
}

// 2. Factory Pattern
class LoggerFactory {
    static createLogger(type, options = {}) {
        switch (type) {
            case 'console':
                return new ConsoleLogger(options);
            case 'file':
                return new FileLogger(options);
            case 'remote':
                return new RemoteLogger(options);
            default:
                throw new Error(`Unknown logger type: ${type}`);
        }
    }
}

class ConsoleLogger {
    log(message) {
        console.log(`[${new Date().toISOString()}] ${message}`);
    }
}

class FileLogger {
    constructor(options) {
        this.filename = options.filename || 'app.log';
    }
    
    log(message) {
        require('fs').appendFileSync(this.filename, 
            `[${new Date().toISOString()}] ${message}\n`);
    }
}

// 3. Observer Pattern
class EventEmitter {
    constructor() {
        this.events = {};
    }
    
    on(event, listener) {
        if (!this.events[event]) {
            this.events[event] = [];
        }
        this.events[event].push(listener);
    }
    
    emit(event, data) {
        if (this.events[event]) {
            this.events[event].forEach(listener => listener(data));
        }
    }
    
    off(event, listenerToRemove) {
        if (this.events[event]) {
            this.events[event] = this.events[event].filter(
                listener => listener !== listenerToRemove
            );
        }
    }
}

// 4. Strategy Pattern
class PaymentProcessor {
    constructor(strategy) {
        this.strategy = strategy;
    }
    
    processPayment(amount) {
        return this.strategy.process(amount);
    }
    
    setStrategy(strategy) {
        this.strategy = strategy;
    }
}

class CreditCardStrategy {
    process(amount) {
        return `Processing $${amount} via Credit Card`;
    }
}

class PayPalStrategy {
    process(amount) {
        return `Processing $${amount} via PayPal`;
    }
}

// 5. Command Pattern
class Command {
    execute() {
        throw new Error('Execute method must be implemented');
    }
    
    undo() {
        throw new Error('Undo method must be implemented');
    }
}

class FileOperationCommand extends Command {
    constructor(operation, filename, content) {
        super();
        this.operation = operation;
        this.filename = filename;
        this.content = content;
        this.previousContent = null;
    }
    
    execute() {
        const fs = require('fs');
        
        if (this.operation === 'write') {
            if (fs.existsSync(this.filename)) {
                this.previousContent = fs.readFileSync(this.filename, 'utf8');
            }
            fs.writeFileSync(this.filename, this.content);
        }
    }
    
    undo() {
        const fs = require('fs');
        
        if (this.operation === 'write') {
            if (this.previousContent !== null) {
                fs.writeFileSync(this.filename, this.previousContent);
            } else {
                fs.unlinkSync(this.filename);
            }
        }
    }
}
```

### Error Handling Patterns

```javascript
// Result pattern for error handling
class Result {
    constructor(value, error) {
        this.value = value;
        this.error = error;
    }
    
    static success(value) {
        return new Result(value, null);
    }
    
    static failure(error) {
        return new Result(null, error);
    }
    
    isSuccess() {
        return this.error === null;
    }
    
    isFailure() {
        return this.error !== null;
    }
    
    map(fn) {
        if (this.isFailure()) {
            return this;
        }
        
        try {
            return Result.success(fn(this.value));
        } catch (error) {
            return Result.failure(error);
        }
    }
    
    flatMap(fn) {
        if (this.isFailure()) {
            return this;
        }
        
        try {
            return fn(this.value);
        } catch (error) {
            return Result.failure(error);
        }
    }
}

// Usage
async function fetchUserData(userId) {
    try {
        const user = await database.getUser(userId);
        return Result.success(user);
    } catch (error) {
        return Result.failure(error);
    }
}

// Async error handling wrapper
function asyncWrapper(fn) {
    return async (req, res, next) => {
        try {
            await fn(req, res, next);
        } catch (error) {
            next(error);
        }
    };
}

// Circuit breaker pattern
class CircuitBreaker {
    constructor(options = {}) {
        this.failureThreshold = options.failureThreshold || 5;
        this.resetTimeout = options.resetTimeout || 60000;
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
        this.failureCount = 0;
        this.lastFailureTime = null;
    }
    
    async execute(fn) {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime >= this.resetTimeout) {
                this.state = 'HALF_OPEN';
            } else {
                throw new Error('Circuit breaker is OPEN');
            }
        }
        
        try {
            const result = await fn();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }
    
    onSuccess() {
        this.failureCount = 0;
        this.state = 'CLOSED';
    }
    
    onFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();
        
        if (this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
        }
    }
}
```

### Configuration Management

```javascript
// Configuration manager
class ConfigManager {
    constructor() {
        this.config = {};
        this.watchers = [];
        this.loadConfig();
    }
    
    loadConfig() {
        // Load from environment variables
        this.config.port = process.env.PORT || 3000;
        this.config.nodeEnv = process.env.NODE_ENV || 'development';
        this.config.database = {
            host: process.env.DB_HOST || 'localhost',
            port: process.env.DB_PORT || 5432,
            name: process.env.DB_NAME || 'app',
            user: process.env.DB_USER || 'user',
            password: process.env.DB_PASSWORD || 'password'
        };
        
        // Load from config file if exists
        try {
            const configFile = require('./config.json');
            this.config = { ...this.config, ...configFile };
        } catch (err) {
            // Config file doesn't exist or is invalid
        }
        
        this.notifyWatchers();
    }
    
    get(key, defaultValue = undefined) {
        const keys = key.split('.');
        let value = this.config;
        
        for (const k of keys) {
            if (value && typeof value === 'object' && k in value) {
                value = value[k];
            } else {
                return defaultValue;
            }
        }
        
        return value;
    }
    
    set(key, value) {
        const keys = key.split('.');
        let current = this.config;
        
        for (let i = 0; i < keys.length - 1; i++) {
            const k = keys[i];
            if (!(k in current) || typeof current[k] !== 'object') {
                current[k] = {};
            }
            current = current[k];
        }
        
        current[keys[keys.length - 1]] = value;
        this.notifyWatchers();
    }
    
    watch(callback) {
        this.watchers.push(callback);
    }
    
    notifyWatchers() {
        this.watchers.forEach(callback => callback(this.config));
    }
}

// Usage
const config = new ConfigManager();

config.watch((newConfig) => {
    console.log('Configuration updated:', newConfig);
});

// Application startup
class Application {
    constructor() {
        this.config = new ConfigManager();
        this.server = null;
    }
    
    async start() {
        const port = this.config.get('port');
        const nodeEnv = this.config.get('nodeEnv');
        
        console.log(`Starting application in ${nodeEnv} mode on port ${port}`);
        
        // Initialize database
        await this.initDatabase();
        
        // Start HTTP server
        this.server = http.createServer((req, res) => {
            // Handle requests
        });
        
        this.server.listen(port, () => {
            console.log(`Server listening on port ${port}`);
        });
    }
    
    async initDatabase() {
        const dbConfig = this.config.get('database');
        // Initialize database connection
    }
    
    async shutdown() {
        if (this.server) {
            this.server.close();
        }
        // Close database connections
        // Clean up resources
    }
}
```

---

## Conclusion

This comprehensive guide covers the fundamental and advanced aspects of Node.js architecture. Understanding these concepts is crucial for building scalable, performant, and maintainable Node.js applications.

### Key Takeaways

1. **Event Loop**: The heart of Node.js's non-blocking architecture
2. **Single-threaded**: JavaScript execution is single-threaded, but I/O is handled asynchronously
3. **Performance**: Optimize for the event loop, manage memory carefully, and use appropriate patterns
4. **Security**: Implement proper validation, authentication, and authorization
5. **Monitoring**: Use profiling and debugging tools to identify bottlenecks
6. **Patterns**: Apply appropriate design patterns for maintainable code

### Recommended Reading

- Node.js Official Documentation
- libuv Documentation
- V8 Engine Documentation
- Node.js Best Practices Guide
- Performance Optimization Guides

---

## 🚀 Request Lifecycle: From Client to Response

This section provides an in-depth exploration of how Node.js handles incoming requests, tracing the journey from client arrival to response delivery. We'll examine every component involved in this process with detailed illustrations and real-world examples.

### Overview: The Complete Request Journey

When a client sends a request to a Node.js application, it triggers a complex orchestration of components working together. Let's visualize this entire process:

```
🌐 Client Request Journey Through Node.js Architecture
════════════════════════════════════════════════════════════════════════════════════

                            ┌─────────────────┐
                            │      CLIENT     │
                            │   (Browser/App) │
                            └─────────┬───────┘
                                      │ HTTP Request
                                      ▼
                            ┌─────────────────┐
                            │     NETWORK     │
                            │  (TCP/IP Stack) │
                            └─────────┬───────┘
                                      │
                                      ▼
    ┌─────────────────────────────────────────────────────────────────────────────────┐
    │                              NODE.JS RUNTIME                                    │
    │                                                                                 │
    │  ┌─────────────────┐    ┌──────────────────────────────────────────────────┐    │
    │  │      libuv      │    │              V8 JavaScript Engine                │    │
    │  │                 │    │                                                  │    │
    │  │  ┌──────────────┤    │  ┌──────────────────┐  ┌─────────────────────┐   │    │
    │  │  │ Event Loop   │◄───┼──│   Call Stack     │  │     Heap            │   │    │
    │  │  │              │    │  │                  │  │   (Objects)         │   │    │
    │  │  │ ┌──────────┐ │    │  │ ┌──────────────┐ │  │                     │   │    │
    │  │  │ │  Timer   │ │    │  │ │main()        │ │  │ ┌─────────────────┐ │   │    │
    │  │  │ │  Phase   │ │    │  │ │requestHandler│ │  │ │  New Space      │ │   │    │
    │  │  │ └──────────┘ │    │  │ │callback()    │ │  │ │  (Young Gen)    │ │   │    │
    │  │  │ ┌──────────┐ │    │  │ └──────────────┘ │  │ └─────────────────┘ │   │    │
    │  │  │ │ Pending  │ │    │  └──────────────────┘  │ ┌─────────────────┐ │   │    │
    │  │  │ │Callbacks │ │    │                        │ │  Old Space      │ │   │    │
    │  │  │ └──────────┘ │    │  ┌─────────────────┐   │ │  (Old Gen)      │ │   │    │
    │  │  │ ┌──────────┐ │    │  │  Callback Queue │   │ └─────────────────┘ │   │    │
    │  │  │ │   Poll   │ │◄───┼──│                 │   └─────────────────────┘   │    │
    │  │  │ │  Phase   │ │    │  │ ┌─────────────┐ │                             │    │
    │  │  │ └──────────┘ │    │  │ │  Microtask  │ │                             │    │
    │  │  │ ┌──────────┐ │    │  │ │    Queue    │ │                             │    │
    │  │  │ │  Check   │ │    │  │ └─────────────┘ │                             │    │
    │  │  │ │  Phase   │ │    │  │ ┌─────────────┐ │                             │    │
    │  │  │ └──────────┘ │    │  │ │   Timer     │ │                             │    │
    │  │  └──────────────┘    │  │ │   Queue     │ │                             │    │
    │  │                      │  │ └─────────────┘ │                             │    │
    │  │  ┌──────────────┐    │  └─────────────────┘                             │    │
    │  │  │ Thread Pool  │    └──────────────────────────────────────────────────┘    │
    │  │  │              │                                                            │
    │  │  │ ┌──────────┐ │                                                            │
    │  │  │ │Thread 1  │ │    ┌─────────────────────────────────────────────────┐     │
    │  │  │ │(File I/O)│ │    │             Node.js Core Modules                │     │
    │  │  │ └──────────┘ │    │                                                 │     │
    │  │  │ ┌──────────┐ │    │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────┐  │     │
    │  │  │ │Thread 2  │ │    │  │   http  │ │   fs    │ │ crypto  │ │ path  │  │     │
    │  │  │ │(DNS)     │ │    │  └─────────┘ └─────────┘ └─────────┘ └───────┘  │     │
    │  │  │ └──────────┘ │    │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────┐  │     │
    │  │  │ ┌──────────┐ │    │  │   net   │ │ stream  │ │ buffer  │ │ util  │  │     │
    │  │  │ │Thread 3  │ │    │  └─────────┘ └─────────┘ └─────────┘ └───────┘  │     │
    │  │  │ │(Crypto)  │ │    └─────────────────────────────────────────────────┘     │
    │  │  │ └──────────┘ │                                                            │
    │  │  │ ┌──────────┐ │                                                            │
    │  │  │ │Thread 4  │ │                                                            │
    │  │  │ │(Custom)  │ │                                                            │
    │  │  │ └──────────┘ │                                                            │
    │  │  └──────────────┘                                                            │ 
    │  └─────────────────┘                                                            │
    └─────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │  DATABASE       │
                                  │  FILE SYSTEM    │
                                  │  EXTERNAL API   │
                                  └─────────────────┘
```

### Phase 1: Request Arrival and Network Handling

#### 1.1 Network Layer Reception

When a client sends an HTTP request, it first reaches the operating system's network stack:

```
📡 Network Request Reception Flow
═══════════════════════════════════════════════════════════════════════════════

    ┌─────────────────┐      TCP SYN      ┌──────────────────────────────────────────┐
    │     CLIENT      │ ────────────────► │           OPERATING SYSTEM               │
    │                 │                   │                                          │
    │  Browser/App    │◄────────────────  │  ┌────────────────────────────────────┐  │
    │  IP: 192.168.1.5│    TCP SYN-ACK    │  │         Network Stack              │  │
    │  Port: 54321    │                   │  │                                    │  │
    └─────────────────┘                   │  │  ┌───────────────────────────────┐ │  │
                                          │  │  │     TCP/IP Layer              │ │  │
           ▼                              │  │  │                               │ │  │
                                          │  │  │  ┌──────────────────────────┐ │ │  │
    HTTP GET /api/users                   │  │  │  │     Socket Layer         │ │ │  │
    Host: localhost:3000                  │  │  │  │                          │ │ │  │
    User-Agent: Mozilla/5.0               │  │  │  │  fd: 12 (file descriptor)│ │ │  │
    Accept: application/json              │  │  │  │  Port: 3000              │ │ │  │
    Connection: keep-alive                │  │  │  │  State: ESTABLISHED      │ │ │  │
                                          │  │  │  └──────────────────────────┘ │ │  │
                                          │  │  └───────────────────────────────┘ │  │
                                          │  └────────────────────────────────────┘  │
                                          └──────────────────────────────────────────┘
                                                           │
                                                           ▼
                                          ┌──────────────────────────────────────┐
                                          │           libuv (Event I/O)          │
                                          │                                      │
                                          │  ┌─────────────────────────────────┐ │
                                          │  │      I/O Watcher                │ │
                                          │  │                                 │ │
                                          │  │  Socket fd: 12                  │ │
                                          │  │  Event: READABLE                │ │
                                          │  │  Callback: onConnection         │ │
                                          │  └─────────────────────────────────┘ │
                                          └──────────────────────────────────────┘
```

#### 1.2 libuv I/O Watchers

libuv continuously monitors file descriptors (including network sockets) for I/O events:

```javascript
// Simplified representation of what happens internally in libuv
const server = http.createServer((req, res) => {
    // This callback will be registered with libuv
    console.log('Request received!');
});

// When server.listen() is called:
server.listen(3000, () => {
    console.log('Server listening on port 3000');
    
    // Internally, libuv creates:
    // 1. A TCP socket
    // 2. Binds it to port 3000
    // 3. Sets it to listening mode
    // 4. Registers an I/O watcher for the socket file descriptor
});

// libuv I/O watcher pseudo-code
class IOWatcher {
    constructor(fd, eventType, callback) {
        this.fd = fd;              // File descriptor (e.g., 12)
        this.eventType = eventType; // READABLE, WRITABLE, etc.
        this.callback = callback;   // Function to call when event occurs
    }
    
    // Called by libuv when the watched event occurs
    onEvent() {
        // Add callback to the event loop's callback queue
        eventLoop.addCallback(this.callback);
    }
}
```

### Phase 2: Event Loop Processing

#### 2.1 Event Loop Phases in Detail

When the I/O watcher detects an incoming connection, it queues a callback for the event loop. Let's trace through each phase:

```
🔄 Event Loop Phases During Request Processing
═══════════════════════════════════════════════════════════════════════════════

 ┌─────────────────────────────────────────────────────────────────────────────┐
 │                               EVENT LOOP                                    │
 │                                                                             │
 │  ┌─────────────────────┐         Phase 1: TIMERS                            │
 │  │    Timer Queue      │         • Execute setTimeout() callbacks           │
 │  │                     │         • Execute setInterval() callbacks          │
 │  │ ┌─────────────────┐ │         • Process expired timers                   │
 │  │ │setTimeout(cb,0) │ │ ──────► • Example: Rate limiting timers            │
 │  │ └─────────────────┘ │                                                    │
 │  │ ┌─────────────────┐ │                                                    │
 │  │ │setInterval(cb)  │ │                                                    │
 │  │ └─────────────────┘ │                                                    │
 │  └─────────────────────┘                                                    │
 │              │                                                              │
 │              ▼                                                              │
 │  ┌─────────────────────┐         Phase 2: PENDING CALLBACKS                 │
 │  │  Pending Callbacks  │         • Execute I/O callbacks from previous      │
 │  │      Queue          │           iterations that were deferred            │
 │  │                     │         • TCP error callbacks                      │
 │  │ ┌─────────────────┐ │ ──────► • UDP error callbacks                      │
 │  │ │TCP error cb     │ │         • Rarely used in HTTP servers              │
 │  │ └─────────────────┘ │                                                    │
 │  └─────────────────────┘                                                    │
 │              │                                                              │
 │              ▼                                                              │
 │  ┌─────────────────────┐         Phase 3: IDLE, PREPARE                     │
 │  │    Internal Use     │         • Internal libuv operations                │
 │  │                     │         • Prepare for polling                      │
 │  │   (libuv only)      │ ──────► • Not directly accessible from JS          │
 │  │                     │         • Housekeeping tasks                       │
 │  └─────────────────────┘                                                    │
 │              │                                                              │
 │              ▼                                                              │
 │  ┌─────────────────────┐         Phase 4: POLL (Most Important!)            │
 │  │     Poll Queue      │         • Fetch new I/O events                     │
 │  │                     │         • Execute I/O callbacks                    │
 │  │ ┌─────────────────┐ │         • HTTP request callbacks                   │
 │  │ │HTTP req handler │ │ ──────► • Database query callbacks                 │
 │  │ └─────────────────┘ │         • File read/write callbacks                │
 │  │ ┌─────────────────┐ │         • Network operation callbacks              │
 │  │ │DB query cb      │ │         • This is where most app logic runs!       │
 │  │ └─────────────────┘ │                                                    │
 │  │ ┌─────────────────┐ │         Poll Phase Behavior:                       │
 │  │ │File read cb     │ │         • If poll queue not empty: execute all     │
 │  │ └─────────────────┘ │         • If poll queue empty: wait for events     │
 │  └─────────────────────┘         • Timeout if timers scheduled              │
 │              │                                                              │
 │              ▼                                                              │
 │  ┌─────────────────────┐         Phase 5: CHECK                             │
 │  │    Check Queue      │         • Execute setImmediate() callbacks         │
 │  │                     │         • Run after poll phase completes           │
 │  │ ┌─────────────────┐ │ ──────► • Higher priority than setTimeout(0)       │
 │  │ │setImmediate(cb) │ │         • Useful for deferring execution           │
 │  │ └─────────────────┘ │                                                    │
 │  └─────────────────────┘                                                    │
 │              │                                                              │
 │              ▼                                                              │
 │  ┌─────────────────────┐         Phase 6: CLOSE CALLBACKS                   │
 │  │  Close Callbacks    │         • Execute close event callbacks            │
 │  │      Queue          │         • socket.on('close', callback)             │
 │  │                     │ ──────► • server.on('close', callback)             │
 │  │ ┌─────────────────┐ │         • Clean up resources                       │
 │  │ │socket.close cb  │ │                                                    │
 │  │ └─────────────────┘ │                                                    │
 │  └─────────────────────┘                                                    │
 │              │                                                          ▲   │
 │              └──────────────────── Back to Phase 1 ─────────────────────┘   │
 └─────────────────────────────────────────────────────────────────────────────┘

     ⚡ Between each phase: Microtask Processing
     ┌─────────────────────────────────────────────────────────────────────┐
     │  1. Process.nextTick() callbacks (highest priority)                 │
     │  2. Promise.then() callbacks (resolved promises)                    │
     │  3. queueMicrotask() callbacks                                      │
     └─────────────────────────────────────────────────────────────────────┘
```

#### 2.2 Request Processing in Poll Phase

The poll phase is where HTTP request handling happens. Let's see the detailed flow:

```javascript
// Real-world HTTP server with detailed logging
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
    console.log('🎯 [POLL PHASE] HTTP request callback executing');
    console.log(`📍 Event Loop Phase: POLL`);
    console.log(`🔗 Request: ${req.method} ${req.url}`);
    
    // This entire function runs in the POLL phase
    const startTime = process.hrtime.bigint();
    
    // Synchronous operations (run immediately in poll phase)
    const headers = req.headers;
    const method = req.method;
    const url = req.url;
    
    if (url === '/api/users' && method === 'GET') {
        // Simulate async database query
        console.log('💾 [POLL PHASE] Initiating database query...');
        
        // This will be handled by thread pool
        queryDatabase('SELECT * FROM users', (err, results) => {
            console.log('📊 [POLL PHASE] Database callback executing');
            
            if (err) {
                res.writeHead(500, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({ error: 'Database error' }));
                return;
            }
            
            // Process data (synchronous - still in poll phase)
            const processedData = results.map(user => ({
                id: user.id,
                name: user.name,
                email: user.email
            }));
            
            // Send response (I/O operation)
            res.writeHead(200, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify(processedData));
            
            const endTime = process.hrtime.bigint();
            console.log(`⏱️  Request processed in ${Number(endTime - startTime) / 1000000}ms`);
        });
        
    } else if (url === '/api/file' && method === 'GET') {
        console.log('📁 [POLL PHASE] Reading file...');
        
        // File operation - will use thread pool
        fs.readFile('./large-file.txt', 'utf8', (err, data) => {
            console.log('📄 [POLL PHASE] File read callback executing');
            
            if (err) {
                res.writeHead(404, { 'Content-Type': 'text/plain' });
                res.end('File not found');
                return;
            }
            
            res.writeHead(200, { 'Content-Type': 'text/plain' });
            res.end(data);
        });
        
    } else {
        // Immediate response (no async operations)
        console.log('⚡ [POLL PHASE] Sending immediate response');
        res.writeHead(404, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Not found' }));
    }
});

// Custom database query function (simulates async DB operation)
function queryDatabase(query, callback) {
    // This operation will be handled by libuv's thread pool
    console.log('🔄 [THREAD POOL] Database query started');
    
    // Simulate database work in thread pool
    setImmediate(() => {
        console.log('🔄 [CHECK PHASE] setImmediate callback - simulating DB response');
        
        // Simulate processing time
        setTimeout(() => {
            console.log('⏰ [TIMER PHASE] Database query completed');
            
            // Mock database results
            const mockResults = [
                { id: 1, name: 'John Doe', email: 'john@example.com' },
                { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
            ];
            
            // Call callback with results
            callback(null, mockResults);
        }, 100);
    });
}

server.listen(3000, () => {
    console.log('🚀 Server listening on port 3000');
    console.log('📍 Event Loop Phase: POLL (waiting for connections)');
});
```

### Phase 3: Thread Pool Operations

#### 3.1 Thread Pool Architecture

When the request handler needs to perform I/O operations, libuv's thread pool comes into play:

```
🧵 Thread Pool Operations During Request Processing
══════════════════════════════════════════════════════════════════════════════

                      ┌─────────────────────────────────────────────────────┐
                      │                     MAIN THREAD                     │
                      │                                                     │
                      │  ┌──────────────────────────────────────────────┐   │
                      │  │               Event Loop                     │   │
                      │  │                                              │   │
                      │  │  ┌─────────────────────────────────────┐     │   │
                      │  │  │            POLL PHASE               │     │   │
                      │  │  │                                     │     │   │
                      │  │  │  fs.readFile('data.txt', cb)        │     │   │
                      │  │  │  db.query('SELECT * FROM users',cb) │     │   │
                      │  │  │  crypto.pbkdf2(password, salt, cb)  │     │   │
                      │  │  └─────────────┬───────────────────────┘     │   │
                      │  └───────────────┬┴─────────────────────────────┘   │
                      └─────────────────┬┴──────────────────────────────────┘
                                        │
                      ┌─────────────────▼────────────────────────────────────┐
                      │                libuv Work Queue                      │
                      │                                                      │
                      │  ┌────────────────────────────────────────────┐      │
                      │  │             Work Items Queue               │      │
                      │  │                                            │      │
                      │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │      │
                      │  │  │File I/O  │  │DB Query  │  │Crypto    │  │      │
                      │  │  │Request   │  │Request   │  │Operation │  │      │
                      │  │  │          │  │          │  │          │  │      │
                      │  │  │file.txt  │  │SELECT *  │  │pbkdf2()  │  │      │
                      │  │  │callback_1│  │callback_2│  │callback_3│  │      │
                      │  │  └──────────┘  └──────────┘  └──────────┘  │      │
                      │  └────────────────────┬───────────────────────       │
                      └────────────────────┬──┴───────────────────────────┬──┘
                                           │                              │
                      ┌────────────────────▼──────────────────────────────▼──────┐
                      │                  THREAD POOL (libuv)                     │
                      │                                                          │
                      │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
                      │  │Thread 1  │  │Thread 2  │  │Thread 3  │  │Thread 4  │  │
                      │  │          │  │          │  │          │  │          │  │
                      │  │   File   │  │  DNS     │  │ Crypto   │  │  DB      │  │
                      │  │   I/O    │  │Resolution│  │Operation │  │Operation │  │
                      │  │          │  │          │  │          │  │          │  │
                      │  │Status:   │  │Status:   │  │Status:   │  │Status:   │  │
                      │  │BUSY      │  │IDLE      │  │BUSY      │  │BUSY      │  │
                      │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
                      └───────┼───────────┬─┴──────────┬──┴───────────┬─┴────────┘
                              │           │            │              │
                      ┌───────▼───────────▼────────────▼──────────────▼───┐
                      │                Completion Queue                   │
                      │                                                   │
                      │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
                      │  │callback_1│  │callback_2│  │callback_3│         │
                      │  │(file)    │  │(query)   │  │(hash)    │         │
                      │  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
                      └───────┼───────────┬─┴──────────┬──┴───────────────┘
                              │           │            │
                      ┌───────▼───────────▼────────────▼──────────────┐
                      │            Event Loop (Next Iteration)        │
                      │                                               │
                      │  ┌────────────────────────────────────────┐   │
                      │  │             POLL PHASE                 │   │
                      │  │                                        │   │
                      │  │ • Execute callback_1 (file complete)   │   │
                      │  │ • Execute callback_2 (query complete)  │   │
                      │  │ • Execute callback_3 (crypto complete) │   │
                      │  └────────────────────────────────────────┘   │
                      └───────────────────────────────────────────────┘
```

#### 3.2 Thread Pool Configuration and Monitoring

```javascript
// Configure thread pool size (must be set before any I/O operations)
process.env.UV_THREADPOOL_SIZE = 16; // Default is 4

const fs = require('fs');
const crypto = require('crypto');
const { performance } = require('perf_hooks');

// Thread pool monitor
class ThreadPoolMonitor {
    constructor() {
        this.activeOperations = new Map();
        this.completedOperations = 0;
        this.totalOperations = 0;
    }
    
    startOperation(type, id) {
        this.totalOperations++;
        this.activeOperations.set(id, {
            type,
            startTime: performance.now(),
            id
        });
        
        console.log(`🧵 [THREAD POOL] Started ${type} operation (ID: ${id})`);
        console.log(`📊 Active operations: ${this.activeOperations.size}`);
    }
    
    completeOperation(id) {
        const operation = this.activeOperations.get(id);
        if (operation) {
            const duration = performance.now() - operation.startTime;
            this.completedOperations++;
            this.activeOperations.delete(id);
            
            console.log(`✅ [THREAD POOL] Completed ${operation.type} (ID: ${id}) in ${duration.toFixed(2)}ms`);
            console.log(`📊 Active operations: ${this.activeOperations.size}`);
            console.log(`📈 Completion rate: ${this.completedOperations}/${this.totalOperations}`);
        }
    }
    
    getStats() {
        return {
            active: this.activeOperations.size,
            completed: this.completedOperations,
            total: this.totalOperations,
            activeOperations: Array.from(this.activeOperations.values())
        };
    }
}

const monitor = new ThreadPoolMonitor();

// HTTP server with thread pool monitoring
const http = require('http');

const server = http.createServer(async (req, res) => {
    const requestId = Math.random().toString(36).substr(2, 9);
    console.log(`🌐 [REQUEST ${requestId}] ${req.method} ${req.url}`);
    
    if (req.url === '/api/heavy-computation') {
        // CPU-intensive operation using thread pool
        const operationId = `crypto-${requestId}`;
        monitor.startOperation('CRYPTO', operationId);
        
        crypto.pbkdf2('password', 'salt', 100000, 64, 'sha512', (err, derivedKey) => {
            monitor.completeOperation(operationId);
            
            if (err) {
                res.writeHead(500);
                res.end('Crypto operation failed');
                return;
            }
            
            res.writeHead(200, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({
                key: derivedKey.toString('hex'),
                requestId
            }));
        });
        
    } else if (req.url === '/api/file-operation') {
        // File I/O operation using thread pool
        const operationId = `file-${requestId}`;
        monitor.startOperation('FILE_READ', operationId);
        
        fs.readFile('./package.json', 'utf8', (err, data) => {
            monitor.completeOperation(operationId);
            
            if (err) {
                res.writeHead(404);
                res.end('File not found');
                return;
            }
            
            res.writeHead(200, { 'Content-Type': 'application/json' });
            res.end(data);
        });
        
    } else if (req.url === '/api/multiple-operations') {
        // Multiple concurrent operations
        const fileOpId = `file-${requestId}`;
        const cryptoOpId = `crypto-${requestId}`;
        
        let completedOps = 0;
        const results = {};
        
        // File operation
        monitor.startOperation('FILE_READ', fileOpId);
        fs.readFile('./package.json', 'utf8', (err, data) => {
            monitor.completeOperation(fileOpId);
            results.fileData = err ? null : JSON.parse(data);
            
            if (++completedOps === 2) {
                sendResponse();
            }
        });
        
        // Crypto operation
        monitor.startOperation('CRYPTO', cryptoOpId);
        crypto.randomBytes(32, (err, buffer) => {
            monitor.completeOperation(cryptoOpId);
            results.randomData = err ? null : buffer.toString('hex');
            
            if (++completedOps === 2) {
                sendResponse();
            }
        });
        
        function sendResponse() {
            res.writeHead(200, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({
                results,
                requestId,
                threadPoolStats: monitor.getStats()
            }));
        }
        
    } else if (req.url === '/api/stats') {
        // Return thread pool statistics
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
            threadPoolStats: monitor.getStats(),
            processStats: {
                memoryUsage: process.memoryUsage(),
                cpuUsage: process.cpuUsage(),
                uptime: process.uptime()
            }
        }));
        
    } else {
        res.writeHead(404);
        res.end('Not found');
    }
});

server.listen(3000, () => {
    console.log('🚀 Server listening on port 3000');
    console.log(`🧵 Thread pool size: ${process.env.UV_THREADPOOL_SIZE || 4}`);
});
```

### Phase 4: Timers and Scheduling

#### 4.1 Timer Management in Request Context

Timers play a crucial role in request handling, especially for timeouts, rate limiting, and scheduled operations:

```
⏰ Timer Management During Request Processing
═══════════════════════════════════════════════════════════════════════════════

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                            REQUEST HANDLER                              │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐    │
    │  │                    Request Processing                           │    │
    │  │                                                                 │    │ 
    │  │  1. Request arrives                                             │    │
    │  │  2. Set request timeout ──────────────────────────┐             │    │
    │  │  3. Start processing                              │             │    │
    │  │  4. Initiate I/O operations                       │             │    │
    │  │  5. Set rate limiting timer ──────────────────────┼─────────┐   │    │
    │  │                                                   │         │   │    │
    │  └───────────────────────────────────────────────────┼─────────┼───┘    │
    └──────────────────────────────────────────────────────┼─────────┼─────────┘
                                                           │         │
                         ┌─────────────────────────────────┘         │
                         │                                           │
                         ▼                                           ▼
    ┌─────────────────────────────────────┐        ┌─────────────────────────────┐
    │           TIMER QUEUE               │        │       TIMER QUEUE           │
    │                                     │        │                             │
    │  ┌─────────────────────────────────┐│        │ ┌─────────────────────────┐ │
    │  │     Request Timeout Timer       ││        │ │   Rate Limit Timer      │ │
    │  │                                 ││        │ │                         │ │
    │  │  setTimeout(() => {             ││        │ │ setTimeout(() => {      │ │
    │  │    if (!responsesSent) {        ││        │ │   resetRateLimit();     │ │
    │  │      res.writeHead(408);        ││        │ │ }, 60000);              │ │
    │  │      res.end('Request Timeout');││        │ │                         │ │
    │  │    }                            ││        │ │ Purpose: Reset rate     │ │
    │  │  }, 30000);                     ││        │ │ limiting counters       │ │
    │  │                                 ││        │ │                         │ │
    │  │  Purpose: Prevent hanging       ││        │ └─────────────────────────┘ │
    │  │  requests and resource leaks    ││        └─────────────────────────────┘
    │  └─────────────────────────────────┘│
    └─────────────────────────────────────┘
                         │
                         ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        EVENT LOOP - TIMER PHASE                         │
    │                                                                         │
    │  Every event loop iteration:                                            │
    │                                                                         │
    │  1. Check timer heap for expired timers                                 │
    │  2. Execute callbacks for expired timers                                │
    │  3. Update timer heap                                                   │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐    │
    │  │               Timer Heap (Min-Heap)                             │    │
    │  │                                                                 │    │
    │  │             [Timer 1: expires in 100ms]                         │    │
    │  │         /                               \                       │    │
    │  │    [Timer 2: 500ms]             [Timer 3: 300ms]                │    │
    │  │     /          \                   /          \                 │    │
    │  │ [Timer 4]  [Timer 5]         [Timer 6]    [Timer 7]             │    │
    │  │                                                                 │    │
    │  │  When Timer 1 expires:                                          │    │
    │  │  • Execute callback                                             │    │
    │  │  • Rebalance heap                                               │    │
    │  │  • Continue to next phase                                       │    │
    │  └─────────────────────────────────────────────────────────────────┘    │
    └─────────────────────────────────────────────────────────────────────────┘
```

#### 4.2 Advanced Timer Usage in HTTP Servers

```javascript
const http = require('http');
const { performance } = require('perf_hooks');

class AdvancedHTTPServer {
    constructor() {
        this.activeRequests = new Map();
        this.rateLimiters = new Map();
        this.requestTimeout = 30000; // 30 seconds
        this.rateLimitWindow = 60000; // 1 minute
        this.rateLimitMax = 100; // 100 requests per minute per IP
    }
    
    createServer() {
        return http.createServer((req, res) => {
            const requestId = this.generateRequestId();
            const clientIP = req.connection.remoteAddress;
            const startTime = performance.now();
            
            console.log(`🌐 [${requestId}] Request started: ${req.method} ${req.url}`);
            
            // Set up request tracking
            this.activeRequests.set(requestId, {
                req,
                res,
                startTime,
                clientIP,
                completed: false
            });
            
            // 1. Set request timeout
            const timeoutTimer = setTimeout(() => {
                this.handleRequestTimeout(requestId);
            }, this.requestTimeout);
            
            // 2. Check rate limiting
            if (!this.checkRateLimit(clientIP)) {
                clearTimeout(timeoutTimer);
                this.sendRateLimitResponse(res, requestId);
                return;
            }
            
            // 3. Set up response completion handler
            const originalEnd = res.end;
            res.end = (...args) => {
                clearTimeout(timeoutTimer);
                this.completeRequest(requestId, startTime);
                originalEnd.apply(res, args);
            };
            
            // 4. Handle the actual request
            this.handleRequest(req, res, requestId);
        });
    }
    
    handleRequest(req, res, requestId) {
        const { url, method } = req;
        
        if (url === '/api/fast') {
            // Fast response - no timers needed
            res.writeHead(200, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({ 
                message: 'Fast response',
                requestId,
                timestamp: Date.now()
            }));
            
        } else if (url === '/api/slow') {
            // Simulate slow processing with setTimeout
            console.log(`⏳ [${requestId}] Starting slow operation...`);
            
            setTimeout(() => {
                console.log(`✅ [${requestId}] Slow operation completed`);
                
                if (this.isRequestActive(requestId)) {
                    res.writeHead(200, { 'Content-Type': 'application/json' });
                    res.end(JSON.stringify({ 
                        message: 'Slow response completed',
                        requestId,
                        timestamp: Date.now()
                    }));
                }
            }, 5000); // 5 second delay
            
        } else if (url === '/api/scheduled') {
            // Demonstrate multiple timers
            console.log(`📅 [${requestId}] Setting up scheduled responses...`);
            
            let responseCount = 0;
            const responses = [];
            
            // Schedule multiple responses
            [1000, 2000, 3000].forEach((delay, index) => {
                setTimeout(() => {
                    responseCount++;
                    responses.push(`Response ${responseCount} at ${Date.now()}`);
                    
                    console.log(`📨 [${requestId}] Scheduled response ${responseCount}`);
                    
                    // Send final response after all scheduled operations
                    if (responseCount === 3 && this.isRequestActive(requestId)) {
                        res.writeHead(200, { 'Content-Type': 'application/json' });
                        res.end(JSON.stringify({
                            message: 'All scheduled responses completed',
                            responses,
                            requestId
                        }));
                    }
                }, delay);
            });
            
        } else if (url === '/api/stats') {
            // Return server statistics
            res.writeHead(200, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({
                activeRequests: this.activeRequests.size,
                rateLimiters: Object.fromEntries(this.rateLimiters),
                uptime: process.uptime(),
                memoryUsage: process.memoryUsage()
            }));
            
        } else {
            res.writeHead(404);
            res.end('Not found');
        }
    }
    
    checkRateLimit(clientIP) {
        const now = Date.now();
        
        if (!this.rateLimiters.has(clientIP)) {
            this.rateLimiters.set(clientIP, {
                requests: 1,
                windowStart: now,
                resetTimer: setTimeout(() => {
                    this.resetRateLimit(clientIP);
                }, this.rateLimitWindow)
            });
            return true;
        }
        
        const limiter = this.rateLimiters.get(clientIP);
        
        if (now - limiter.windowStart >= this.rateLimitWindow) {
            // Reset the window
            clearTimeout(limiter.resetTimer);
            limiter.requests = 1;
            limiter.windowStart = now;
            limiter.resetTimer = setTimeout(() => {
                this.resetRateLimit(clientIP);
            }, this.rateLimitWindow);
            return true;
        }
        
        limiter.requests++;
        console.log(`🚦 [RATE LIMIT] ${clientIP}: ${limiter.requests}/${this.rateLimitMax}`);
        
        return limiter.requests <= this.rateLimitMax;
    }
    
    resetRateLimit(clientIP) {
        console.log(`🔄 [RATE LIMIT] Reset for ${clientIP}`);
        this.rateLimiters.delete(clientIP);
    }
    
    handleRequestTimeout(requestId) {
        const request = this.activeRequests.get(requestId);
        if (request && !request.completed) {
            console.log(`⏰ [${requestId}] Request timeout after ${this.requestTimeout}ms`);
            
            request.res.writeHead(408, { 'Content-Type': 'application/json' });
            request.res.end(JSON.stringify({
                error: 'Request timeout',
                timeout: this.requestTimeout,
                requestId
            }));
            
            this.completeRequest(requestId, request.startTime);
        }
    }
    
    completeRequest(requestId, startTime) {
        const request = this.activeRequests.get(requestId);
        if (request) {
            request.completed = true;
            const duration = performance.now() - startTime;
            
            console.log(`✅ [${requestId}] Request completed in ${duration.toFixed(2)}ms`);
            this.activeRequests.delete(requestId);
        }
    }
    
    isRequestActive(requestId) {
        const request = this.activeRequests.get(requestId);
        return request && !request.completed;
    }
    
    generateRequestId() {
        return Math.random().toString(36).substr(2, 9);
    }
}

// Usage
const server = new AdvancedHTTPServer();
const httpServer = server.createServer();

httpServer.listen(3000, () => {
    console.log('🚀 Advanced HTTP Server listening on port 3000');
    console.log('📊 Features enabled:');
    console.log('   - Request timeouts (30s)');
    console.log('   - Rate limiting (100 req/min per IP)');
    console.log('   - Request tracking');
    console.log('   - Scheduled operations');
    
    // Server-level timer for cleanup
    setInterval(() => {
        console.log(`📈 Server stats: ${server.activeRequests.size} active requests`);
        
        // Cleanup old rate limiters (failsafe)
        const now = Date.now();
        for (const [ip, limiter] of server.rateLimiters) {
            if (now - limiter.windowStart > server.rateLimitWindow * 2) {
                console.log(`🧹 Cleaning up old rate limiter for ${ip}`);
                clearTimeout(limiter.resetTimer);
                server.rateLimiters.delete(ip);
            }
        }
    }, 10000); // Every 10 seconds
});

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('🛑 Received SIGTERM, shutting down gracefully...');
    
    httpServer.close(() => {
        console.log('✅ HTTP server closed');
        process.exit(0);
    });
});
```

### Phase 5: Memory Management During Request Processing

#### 5.1 Memory Flow and Garbage Collection

```
🧠 Memory Management During Request Lifecycle
═══════════════════════════════════════════════════════════════════════════════

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                           V8 HEAP STRUCTURE                             │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐    │
    │  │                        NEW SPACE (Young Generation)             │    │
    │  │                                                                 │    │
    │  │  ┌─────────────────┐              ┌─────────────────────────┐   │    │
    │  │  │   From Space    │              │      To Space           │   │    │
    │  │  │                 │              │                         │   │    │
    │  │  │   req object    │   Scavenge   │    Surviving objects    │   │    │
    │  │  │   res object    │  ──────────► │    Processed data       │   │    │
    │  │  │   temp data     │              │    Active callbacks     │   │    │
    │  │  │   callbacks     │              │                         │   │    │
    │  │  │   short-lived   │              │                         │   │    │
    │  │  └─────────────────┘              └─────────────────────────┘   │    │
    │  └─────────────────────────────────────────────────────────────────┘    │
    │                                    │                                    │
    │                                    ▼ Promotion (after multiple GCs)     │
    │  ┌─────────────────────────────────────────────────────────────────┐    │
    │  │                      OLD SPACE (Old Generation)                 │    │
    │  │                                                                 │    │
    │  │    Server instance             Connection pools                 │    │
    │  │    Module cache                Authentication tokens            │    │
    │  │    Route handlers              Session data                     │    │
    │  │    Long-lived closures         Cached responses                 │    │
    │  │                                                                 │    │
    │  │              Mark-and-Sweep Garbage Collection                  │    │
    │  │              (Runs less frequently, more expensive)             │    │
    │  └─────────────────────────────────────────────────────────────────┘    │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐    │
    │  │                        LARGE OBJECT SPACE                       │    │
    │  │                                                                 │    │
    │  │    Large file buffers (>512KB)                                  │    │
    │  │    Image/video data                                             │    │
    │  │    Large JSON responses                                         │    │
    │  │    Database result sets                                         │    │
    │  └─────────────────────────────────────────────────────────────────┘    │
    └─────────────────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────────────┐
    │               MEMORY USAGE DURING REQUEST PROCESSING                    │
    │                                                                         │
    │  Memory Usage                                                           │
    │      ▲                                                                  │
    │      │     🔴 Peak Usage                                                │
    │      │    ╱ ╲   (Request processing)                                    │
    │      │   ╱   ╲                                                          │
    │      │  ╱     ╲  🟡 GC Triggered                                        │
    │      │ ╱       ╲╱                                                       │
    │      │╱         ╲                                                       │
    │      │           ╲ 🟢 Memory Released                                   │
    │      │            ╲                                                     │
    │      │             ╲___________________                                 │
    │      └─────────────────────────────────────► Time                       │
    │       Request    Processing    GC      Idle                             │
    │       Start      Peak          Run     State                            │
    │                                                                         │
    │  Phases:                                                                │
    │  1. Request Start: Allocate req/res objects in New Space                │
    │  2. Processing: Create temporary objects, buffers, data structures      │
    │  3. Peak Usage: Maximum memory consumption during processing            │
    │  4. GC Trigger: Garbage collection initiated (automatic)                │
    │  5. Cleanup: Temporary objects collected, memory freed                  │
    │  6. Idle: Base memory usage, long-lived objects remain                  │
    └─────────────────────────────────────────────────────────────────────────┘
```

#### 5.2 Memory-Efficient Request Handling

```javascript
const http = require('http');
const stream = require('stream');
const { performance } = require('perf_hooks');

class MemoryEfficientServer {
    constructor() {
        this.requestCount = 0;
        this.setupMemoryMonitoring();
    }
    
    setupMemoryMonitoring() {
        // Monitor memory usage every 5 seconds
        setInterval(() => {
            const usage = process.memoryUsage();
            console.log('📊 Memory Usage:', {
                rss: `${Math.round(usage.rss / 1024 / 1024)}MB`,
                heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)}MB`,
                heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)}MB`,
                external: `${Math.round(usage.external / 1024 / 1024)}MB`,
                activeRequests: this.requestCount
            });
            
            // Force garbage collection if heap usage is high (for monitoring only)
            if (global.gc && usage.heapUsed > 100 * 1024 * 1024) { // 100MB
                console.log('🗑️ Forcing garbage collection...');
                global.gc();
            }
        }, 5000);
    }
    
    createServer() {
        return http.createServer((req, res) => {
            this.requestCount++;
            const requestId = `req_${this.requestCount}`;
            const startMemory = process.memoryUsage();
            
            console.log(`🌐 [${requestId}] Started - Heap: ${Math.round(startMemory.heapUsed / 1024 / 1024)}MB`);
            
            // Clean up tracking when request completes
            const originalEnd = res.end;
            res.end = (...args) => {
                this.requestCount--;
                const endMemory = process.memoryUsage();
                const memoryDelta = endMemory.heapUsed - startMemory.heapUsed;
                
                console.log(`✅ [${requestId}] Completed - Memory delta: ${Math.round(memoryDelta / 1024)}KB`);
                originalEnd.apply(res, args);
            };
            
            this.handleRequest(req, res, requestId);
        });
    }
    
    handleRequest(req, res, requestId) {
        const { url } = req;
        
        if (url === '/api/memory-efficient') {
            this.handleMemoryEfficientRoute(req, res, requestId);
        } else if (url === '/api/memory-heavy') {
            this.handleMemoryHeavyRoute(req, res, requestId);
        } else if (url === '/api/streaming') {
            this.handleStreamingRoute(req, res, requestId);
        } else if (url === '/api/buffer-pool') {
            this.handleBufferPoolRoute(req, res, requestId);
        } else {
            res.writeHead(404);
            res.end('Not found');
        }
    }
    
    handleMemoryEfficientRoute(req, res, requestId) {
        console.log(`💚 [${requestId}] Memory-efficient processing`);
        
        // Use minimal objects, avoid closures with large scope
        const responseData = this.createMinimalResponse(requestId);
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify(responseData));
        
        // Explicitly nullify references (helps GC in some cases)
        // responseData = null; // Not needed in modern V8, but shown for educational purposes
    }
    
    handleMemoryHeavyRoute(req, res, requestId) {
        console.log(`🔴 [${requestId}] Memory-heavy processing`);
        
        // Intentionally create large objects to demonstrate memory usage
        const largeArray = new Array(1000000).fill(0).map((_, i) => ({
            id: i,
            data: `Item ${i}`,
            timestamp: Date.now(),
            metadata: {
                processed: true,
                version: '1.0',
                tags: ['heavy', 'memory', 'demo']
            }
        }));
        
        // Simulate processing that creates temporary objects
        const processedData = largeArray
            .filter(item => item.id % 2 === 0)
            .map(item => ({
                ...item,
                processed_at: new Date().toISOString()
            }))
            .slice(0, 1000); // Take only first 1000 items
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
            message: 'Heavy processing completed',
            count: processedData.length,
            requestId,
            sample: processedData.slice(0, 5) // Only send sample to reduce response size
        }));
        
        // Large objects will be garbage collected after this function exits
    }
    
    handleStreamingRoute(req, res, requestId) {
        console.log(`🌊 [${requestId}] Streaming response`);
        
        res.writeHead(200, { 
            'Content-Type': 'application/json',
            'Transfer-Encoding': 'chunked'
        });
        
        // Create a readable stream that generates data on-demand
        const dataStream = new stream.Readable({
            objectMode: true,
            read() {
                // Generate data chunks without keeping large amounts in memory
                static chunkCount = 0;
                
                if (chunkCount < 1000) {
                    this.push({
                        chunk: chunkCount++,
                        data: `Generated data ${chunkCount}`,
                        timestamp: Date.now()
                    });
                } else {
                    this.push(null); // End stream
                }
            }
        });
        
        // Transform stream to JSON
        const jsonTransform = new stream.Transform({
            objectMode: true,
            transform(chunk, encoding, callback) {
                if (chunk === null) {
                    this.push(']}');
                } else if (chunk.chunk === 0) {
                    this.push('{"data":[' + JSON.stringify(chunk));
                } else {
                    this.push(',' + JSON.stringify(chunk));
                }
                callback();
            }
        });
        
        // Pipe streams together
        dataStream
            .pipe(jsonTransform)
            .pipe(res);
        
        console.log(`🌊 [${requestId}] Streaming started - memory efficient`);
    }
    
    handleBufferPoolRoute(req, res, requestId) {
        console.log(`🏊 [${requestId}] Using buffer pool`);
        
        // Simulate buffer pool usage (in real apps, use a proper buffer pool)
        const bufferSize = 64 * 1024; // 64KB
        const buffer = Buffer.allocUnsafe(bufferSize);
        
        try {
            // Simulate filling buffer with data
            const data = `Response data for ${requestId}`;
            buffer.write(data, 0, 'utf8');
            
            res.writeHead(200, { 
                'Content-Type': 'text/plain',
                'Content-Length': data.length
            });
            res.end(buffer.slice(0, data.length));
            
        } finally {
            // In a real buffer pool, we would return the buffer to the pool
            // buffer.fill(0); // Clear sensitive data
            console.log(`🏊 [${requestId}] Buffer reused - memory efficient`);
        }
    }
    
    createMinimalResponse(requestId) {
        // Create minimal object without unnecessary properties
        return {
            id: requestId,
            timestamp: Date.now(),
            status: 'success'
        };
    }
}

// Object pooling example for high-traffic scenarios
class ObjectPool {
    constructor(createFn, resetFn, maxSize = 100) {
        this.createFn = createFn;
        this.resetFn = resetFn;
        this.pool = [];
        this.maxSize = maxSize;
        this.created = 0;
        this.reused = 0;
    }
    
    acquire() {
        if (this.pool.length > 0) {
            this.reused++;
            return this.pool.pop();
        }
        
        this.created++;
        return this.createFn();
    }
    
    release(obj) {
        if (this.pool.length < this.maxSize) {
            this.resetFn(obj);
            this.pool.push(obj);
        }
        // If pool is full, let object be garbage collected
    }
    
    getStats() {
        return {
            poolSize: this.pool.length,
            created: this.created,
            reused: this.reused,
            reuseRatio: this.reused / (this.created + this.reused)
        };
    }
}

// Response object pool
const responsePool = new ObjectPool(
    () => ({ data: null, status: null, timestamp: null }),
    (obj) => { obj.data = null; obj.status = null; obj.timestamp = null; },
    50
);

// Usage
const server = new MemoryEfficientServer();
const httpServer = server.createServer();

httpServer.listen(3000, () => {
    console.log('🚀 Memory-efficient server listening on port 3000');
    console.log('📋 Available endpoints:');
    console.log('   GET /api/memory-efficient - Minimal memory usage');
    console.log('   GET /api/memory-heavy - High memory usage demo');
    console.log('   GET /api/streaming - Streaming response');
    console.log('   GET /api/buffer-pool - Buffer pool usage');
});

// Memory pressure handling
process.on('warning', (warning) => {
    if (warning.name === 'MaxListenersExceededWarning' || warning.name === 'DeprecationWarning') {
        console.warn('⚠️ Memory/Performance Warning:', warning.message);
    }
});
```

### Phase 6: Complete Request Lifecycle Example

Let's put it all together with a comprehensive example that demonstrates the complete request lifecycle:

```javascript
// Complete Node.js Request Lifecycle Demonstration
const http = require('http');
const fs = require('fs');
const crypto = require('crypto');
const { performance } = require('perf_hooks');

class RequestLifecycleDemo {
    constructor() {
        this.requestCounter = 0;
        this.setupLifecycleLogging();
    }
    
    setupLifecycleLogging() {
        // Log event loop lag
        const start = process.hrtime();
        setInterval(() => {
            const lag = process.hrtime(start);
            const lagMs = lag[0] * 1000 + lag[1] * 1e-6;
            if (lagMs > 10) {
                console.log(`⚠️ Event loop lag: ${lagMs.toFixed(2)}ms`);
            }
        }, 1000);
    }
    
    createServer() {
        return http.createServer((req, res) => {
            const requestId = ++this.requestCounter;
            this.traceCompleteRequestLifecycle(req, res, requestId);
        });
    }
    
    async traceCompleteRequestLifecycle(req, res, requestId) {
        const lifecycle = {
            requestId,
            url: req.url,
            method: req.method,
            startTime: performance.now(),
            phases: []
        };
        
        const logPhase = (phase, detail) => {
            const timestamp = performance.now();
            lifecycle.phases.push({
                phase,
                detail,
                timestamp,
                timeSinceStart: timestamp - lifecycle.startTime,
                memoryUsage: process.memoryUsage().heapUsed
            });
            console.log(`🔍 [${requestId}] ${phase}: ${detail} (+${(timestamp - lifecycle.startTime).toFixed(2)}ms)`);
        };
        
        try {
            // Phase 1: Request Arrival (already in POLL phase)
            logPhase('REQUEST_ARRIVAL', 'HTTP request callback executing in POLL phase');
            
            // Phase 2: Header Processing (synchronous - still in POLL phase)
            logPhase('HEADER_PROCESSING', 'Parsing headers and URL synchronously');
            const headers = req.headers;
            const userAgent = headers['user-agent'] || 'unknown';
            
            // Phase 3: Request Body Handling (if applicable)
            if (req.method === 'POST' || req.method === 'PUT') {
                logPhase('BODY_READING', 'Starting body data collection');
                const body = await this.readRequestBody(req, requestId, logPhase);
                lifecycle.bodySize = body.length;
            }
            
            // Phase 4: Business Logic Processing
            logPhase('BUSINESS_LOGIC', 'Starting application logic');
            
            if (req.url === '/api/complete-demo') {
                await this.handleCompleteDemoRequest(req, res, requestId, lifecycle, logPhase);
            } else if (req.url === '/api/lifecycle-stats') {
                this.handleLifecycleStatsRequest(res, requestId, lifecycle, logPhase);
            } else {
                logPhase('ROUTE_NOT_FOUND', '404 response preparation');
                res.writeHead(404, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({ error: 'Not found', requestId }));
            }
            
        } catch (error) {
            logPhase('ERROR_HANDLING', `Error occurred: ${error.message}`);
            res.writeHead(500, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({ error: 'Internal server error', requestId }));
        } finally {
            // Phase N: Request Completion
            const totalTime = performance.now() - lifecycle.startTime;
            logPhase('REQUEST_COMPLETE', `Total processing time: ${totalTime.toFixed(2)}ms`);
            
            // Log complete lifecycle summary
            console.log(`📋 [${requestId}] Lifecycle Summary:`, {
                totalTime: `${totalTime.toFixed(2)}ms`,
                phases: lifecycle.phases.length,
                url: lifecycle.url,
                method: lifecycle.method
            });
        }
    }
    
    async handleCompleteDemoRequest(req, res, requestId, lifecycle, logPhase) {
        // Phase 4a: Authentication simulation
        logPhase('AUTHENTICATION', 'Simulating token validation');
        await this.simulateAsync(50, 'auth validation');
        
        // Phase 4b: Database query simulation (uses thread pool)
        logPhase('DATABASE_QUERY', 'Initiating database query - moving to thread pool');
        const dbResults = await this.simulateDatabaseQuery(requestId, logPhase);
        
        // Phase 4c: File system operation (uses thread pool)
        logPhase('FILE_OPERATION', 'Reading configuration file - using thread pool');
        const configData = await this.readConfigFile(requestId, logPhase);
        
        // Phase 4d: CPU-intensive operation (uses thread pool)
        logPhase('CRYPTO_OPERATION', 'Performing encryption - using thread pool');
        const encryptedData = await this.performCryptoOperation(requestId, logPhase);
        
        // Phase 4e: External API call simulation
        logPhase('EXTERNAL_API', 'Simulating external API call');
        const apiData = await this.simulateExternalAPI(requestId, logPhase);
        
        // Phase 4f: Data processing (synchronous - back in POLL phase)
        logPhase('DATA_PROCESSING', 'Processing and combining all data synchronously');
        const processedData = {
            requestId,
            dbResults,
            config: configData,
            encrypted: encryptedData,
            externalData: apiData,
            processingTime: performance.now() - lifecycle.startTime,
            timestamp: new Date().toISOString()
        };
        
        // Phase 4g: Response serialization
        logPhase('RESPONSE_SERIALIZATION', 'Converting data to JSON');
        const responseBody = JSON.stringify(processedData, null, 2);
        
        // Phase 4h: HTTP response headers
        logPhase('RESPONSE_HEADERS', 'Setting HTTP response headers');
        res.writeHead(200, {
            'Content-Type': 'application/json',
            'X-Request-ID': requestId.toString(),
            'X-Processing-Time': (performance.now() - lifecycle.startTime).toFixed(2)
        });
        
        // Phase 4i: Response body transmission
        logPhase('RESPONSE_TRANSMISSION', 'Sending response body to client');
        res.end(responseBody);
    }
    
    handleLifecycleStatsRequest(res, requestId, lifecycle, logPhase) {
        logPhase('STATS_GENERATION', 'Generating lifecycle statistics');
        
        const stats = {
            requestId,
            processStats: {
                uptime: process.uptime(),
                memoryUsage: process.memoryUsage(),
                cpuUsage: process.cpuUsage()
            },
            eventLoopStats: {
                // Note: These would require additional monitoring setup in production
                activeHandles: process._getActiveHandles().length,
                activeRequests: process._getActiveRequests().length
            },
            currentPhase: 'STATS_GENERATION',
            timestamp: new Date().toISOString()
        };
        
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify(stats, null, 2));
    }
    
    // Utility methods for different types of operations
    
    async readRequestBody(req, requestId, logPhase) {
        return new Promise((resolve, reject) => {
            let body = '';
            
            req.on('data', chunk => {
                body += chunk.toString();
                logPhase('BODY_CHUNK', `Received ${chunk.length} bytes`);
            });
            
            req.on('end', () => {
                logPhase('BODY_COMPLETE', `Total body size: ${body.length} bytes`);
                resolve(body);
            });
            
            req.on('error', reject);
        });
    }
    
    async simulateDatabaseQuery(requestId, logPhase) {
        return new Promise((resolve) => {
            logPhase('DB_THREAD_START', 'Database operation started in thread pool');
            
            // Simulate database work in thread pool
            setImmediate(() => {
                logPhase('DB_CALLBACK', 'Database callback executing in CHECK phase');
                
                setTimeout(() => {
                    logPhase('DB_COMPLETE', 'Database query completed in TIMER phase');
                    resolve([
                        { id: 1, name: 'User 1', created: new Date().toISOString() },
                        { id: 2, name: 'User 2', created: new Date().toISOString() }
                    ]);
                }, 100);
            });
        });
    }
    
    async readConfigFile(requestId, logPhase) {
        return new Promise((resolve, reject) => {
            logPhase('FILE_THREAD_START', 'File read operation queued for thread pool');
            
            // This will actually use the thread pool
            fs.readFile('./package.json', 'utf8', (err, data) => {
                if (err) {
                    logPhase('FILE_ERROR', `File read error: ${err.message}`);
                    resolve({ error: err.message });
                    return;
                }
                
                logPhase('FILE_COMPLETE', 'File read completed, callback in POLL phase');
                try {
                    const parsed = JSON.parse(data);
                    resolve({ name: parsed.name, version: parsed.version });
                } catch (parseErr) {
                    resolve({ error: 'Invalid JSON' });
                }
            });
        });
    }
    
    async performCryptoOperation(requestId, logPhase) {
        return new Promise((resolve, reject) => {
            logPhase('CRYPTO_THREAD_START', 'Crypto operation queued for thread pool');
            
            // This will use the thread pool
            crypto.pbkdf2(`password-${requestId}`, 'salt', 10000, 32, 'sha512', (err, derivedKey) => {
                if (err) {
                    logPhase('CRYPTO_ERROR', `Crypto error: ${err.message}`);
                    reject(err);
                    return;
                }
                
                logPhase('CRYPTO_COMPLETE', 'Crypto operation completed, callback in POLL phase');
                resolve(derivedKey.toString('hex'));
            });
        });
    }
    
    async simulateExternalAPI(requestId, logPhase) {
        return new Promise((resolve) => {
            logPhase('API_REQUEST', 'External API request initiated');
            
            // Simulate network delay
            setTimeout(() => {
                logPhase('API_RESPONSE', 'External API response received in TIMER phase');
                resolve({
                    data: `External data for request ${requestId}`,
                    timestamp: Date.now(),
                    source: 'external-api'
                });
            }, 200);
        });
    }
    
    async simulateAsync(delay, operation) {
        return new Promise(resolve => {
            setTimeout(() => resolve(`${operation} completed`), delay);
        });
    }
}

// Create and start the server
const demo = new RequestLifecycleDemo();
const server = demo.createServer();

server.listen(3000, () => {
    console.log('🚀 Request Lifecycle Demo Server listening on port 3000');
    console.log('');
    console.log('📋 Available endpoints:');
    console.log('   GET  /api/complete-demo    - Complete request lifecycle demonstration');
    console.log('   POST /api/complete-demo    - Same as above, but with body parsing');
    console.log('   GET  /api/lifecycle-stats  - Request processing statistics');
    console.log('');
    console.log('🔍 Watch the console for detailed phase-by-phase logging');
    console.log('');
    console.log('Example request:');
    console.log('curl -X GET http://localhost:3000/api/complete-demo');
    console.log('curl -X POST http://localhost:3000/api/complete-demo -d "{"test":"data"}"');
});

// Graceful shutdown handling
process.on('SIGTERM', () => {
    console.log('🛑 Received SIGTERM signal, shutting down gracefully...');
    
    server.close(() => {
        console.log('✅ HTTP server closed');
        process.exit(0);
    });
});
```

This comprehensive section provides an in-depth exploration of how Node.js handles requests from arrival to response, complete with detailed illustrations and practical examples. The documentation covers:

1. **Network Layer Reception** - How requests arrive at the OS level
2. **Event Loop Processing** - Detailed breakdown of all 6 phases
3. **Thread Pool Operations** - How I/O operations are handled
4. **Timer Management** - Request timeouts, rate limiting, and scheduling
5. **Memory Management** - Heap structure and garbage collection during requests
6. **Complete Lifecycle Demo** - A comprehensive example tracing every step

Each section includes both theoretical explanations with visual diagrams and practical code examples that you can run to see the concepts in action.

---

*This document serves as a comprehensive reference for Node.js architecture. Regular updates and practical examples help maintain relevance with the evolving Node.js ecosystem.* 
