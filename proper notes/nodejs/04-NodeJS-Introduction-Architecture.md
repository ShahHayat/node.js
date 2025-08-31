# Node.js Introduction and Architecture - Complete Guide

## Table of Contents
1. [What is Node.js?](#what-is-nodejs)
2. [Node.js Architecture](#nodejs-architecture)
3. [Event Loop](#event-loop)
4. [Installation and Setup](#installation-and-setup)
5. [Your First Node.js Application](#your-first-nodejs-application)
6. [Global Objects](#global-objects)
7. [Process Object](#process-object)
8. [Buffer](#buffer)
9. [Timers](#timers)
10. [Error Handling in Node.js](#error-handling-in-nodejs)

## What is Node.js?

Node.js is a **JavaScript runtime environment** built on Chrome's V8 JavaScript engine. It allows you to run JavaScript on the server-side, outside of a web browser.

### Key Characteristics

- **Server-side JavaScript**: Execute JavaScript code on servers
- **Event-driven**: Based on events and callbacks
- **Non-blocking I/O**: Asynchronous operations by default
- **Single-threaded**: Main event loop runs on a single thread
- **Cross-platform**: Runs on Windows, macOS, Linux
- **Package ecosystem**: Vast npm (Node Package Manager) ecosystem

### What Node.js is NOT

- **Not a programming language**: It's a runtime for JavaScript
- **Not a framework**: It's a platform (though frameworks like Express.js run on it)
- **Not multi-threaded**: Main execution is single-threaded (though it uses thread pools internally)

### Use Cases

#### Ideal for:
- **Real-time applications**: Chat applications, live updates
- **API servers**: RESTful APIs, GraphQL servers
- **Microservices**: Lightweight, scalable services
- **I/O intensive applications**: File operations, database operations
- **Single Page Applications (SPAs)**: Server-side rendering
- **Command-line tools**: Build tools, utilities
- **IoT applications**: Lightweight runtime for devices

#### Not ideal for:
- **CPU-intensive tasks**: Heavy computational work
- **Applications requiring high security**: Financial systems (though not impossible)
- **Applications with heavy server-side logic**: Complex business logic might be better in other languages

## Node.js Architecture

### High-Level Architecture

Node.js architecture consists of several layers:

```
┌─────────────────────────────────────┐
│           Node.js Application       │
├─────────────────────────────────────┤
│              Node.js API            │
├─────────────────────────────────────┤
│               Node.js               │
│            (C++ Bindings)           │
├─────────────────────────────────────┤
│     V8 Engine      │     libuv      │
│   (JavaScript      │   (Event Loop, │
│    Execution)      │   Thread Pool) │
├─────────────────────────────────────┤
│            Operating System         │
└─────────────────────────────────────┘
```

### Core Components

#### 1. V8 JavaScript Engine
- **Purpose**: Compiles and executes JavaScript code
- **Features**: 
  - Just-in-time (JIT) compilation
  - Memory management and garbage collection
  - Optimized JavaScript execution

```javascript
// V8 compiles this JavaScript to machine code
function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

console.log(fibonacci(10)); // V8 optimizes this execution
```

#### 2. libuv
- **Purpose**: Provides asynchronous I/O operations
- **Features**:
  - Event loop implementation
  - Thread pool for blocking operations
  - Platform abstraction layer

#### 3. Node.js Bindings
- **Purpose**: Bridge between JavaScript and C++
- **Components**:
  - Built-in modules (fs, http, crypto, etc.)
  - Native addons support

#### 4. Node.js Standard Library
- **Built-in modules**: File system, HTTP, streams, etc.
- **No compilation required**: Ready to use

### Memory Architecture

```javascript
// Memory usage inspection
console.log('Memory Usage:', process.memoryUsage());
// Output:
// {
//   rss: 24576000,      // Resident Set Size
//   heapTotal: 6144000, // Total heap allocated
//   heapUsed: 4321456,  // Heap actually used
//   external: 8192      // External memory usage
// }

// Heap structure in V8
// ┌─────────────────────────────────┐
// │              Heap               │
// ├─────────────────────────────────┤
// │         Young Generation        │
// │  ┌─────────────┬─────────────┐  │
// │  │   From      │     To      │  │
// │  │   Space     │   Space     │  │
// │  └─────────────┴─────────────┘  │
// ├─────────────────────────────────┤
// │         Old Generation          │
// │  ┌─────────────┬─────────────┐  │
// │  │   Old       │    Large    │  │
// │  │  Pointer    │   Object    │  │
// │  │   Space     │   Space     │  │
// │  └─────────────┴─────────────┘  │
// └─────────────────────────────────┘
```

## Event Loop

The Event Loop is the heart of Node.js's non-blocking I/O model. It's what allows Node.js to perform non-blocking I/O operations despite JavaScript being single-threaded.

### Event Loop Phases

```
   ┌───────────────────────────┐
┌─>│           timers          │  ← setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  ← I/O callbacks deferred
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  ← Internal use only
│  └─────────────┬─────────────┘      
│  ┌─────────────┴─────────────┐
│  │           poll            │  ← Fetch new I/O events
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  ← setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  ← close event callbacks
   └───────────────────────────┘
```

### Event Loop Example

```javascript
const fs = require('fs');

console.log('Start');

// Timer phase
setTimeout(() => {
    console.log('Timer 1');
}, 0);

// Check phase
setImmediate(() => {
    console.log('Immediate 1');
});

// I/O operation
fs.readFile(__filename, () => {
    console.log('File read');
    
    // These will execute in the next iteration
    setTimeout(() => {
        console.log('Timer 2');
    }, 0);
    
    setImmediate(() => {
        console.log('Immediate 2');
    });
});

// Process.nextTick has highest priority
process.nextTick(() => {
    console.log('Next tick 1');
});

// Promise callbacks are in microtask queue
Promise.resolve().then(() => {
    console.log('Promise 1');
});

console.log('End');

// Typical output:
// Start
// End
// Next tick 1
// Promise 1
// Timer 1
// Immediate 1
// File read
// Immediate 2
// Timer 2
```

### Microtasks vs Macrotasks

```javascript
// Microtasks (higher priority)
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('Promise'));

// Macrotasks (lower priority)
setTimeout(() => console.log('setTimeout'), 0);
setImmediate(() => console.log('setImmediate'));

// Output:
// nextTick
// Promise
// setTimeout
// setImmediate
```

### Event Loop Monitoring

```javascript
// Monitor event loop lag
function measureEventLoopLag() {
    const start = process.hrtime.bigint();
    
    setImmediate(() => {
        const lag = process.hrtime.bigint() - start;
        console.log(`Event loop lag: ${Number(lag) / 1000000}ms`);
    });
}

// Check event loop lag every second
setInterval(measureEventLoopLag, 1000);

// Block the event loop (bad practice)
function blockEventLoop() {
    const start = Date.now();
    while (Date.now() - start < 1000) {
        // Blocking operation
    }
}

// This will show increased lag
setTimeout(blockEventLoop, 2000);
```

## Installation and Setup

### Installing Node.js

#### Option 1: Official Installer
1. Visit [nodejs.org](https://nodejs.org)
2. Download LTS version (recommended for production)
3. Run installer and follow instructions

#### Option 2: Package Managers

```bash
# macOS with Homebrew
brew install node

# Ubuntu/Debian
sudo apt update
sudo apt install nodejs npm

# Windows with Chocolatey
choco install nodejs

# Arch Linux
sudo pacman -S nodejs npm
```

#### Option 3: Node Version Manager (Recommended)

```bash
# Install nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Restart terminal or source profile
source ~/.bashrc

# Install latest LTS Node.js
nvm install --lts

# Install specific version
nvm install 18.17.0

# Use specific version
nvm use 18.17.0

# List installed versions
nvm list

# Set default version
nvm alias default 18.17.0
```

### Verifying Installation

```bash
# Check Node.js version
node --version
# or
node -v

# Check npm version
npm --version
# or
npm -v

# Check installation paths
which node
which npm

# Node.js REPL (Read-Eval-Print Loop)
node
> console.log('Hello Node.js!');
Hello Node.js!
> .exit
```

### Development Environment Setup

#### 1. Code Editor Extensions
- **VS Code**: Node.js Extension Pack
- **Sublime Text**: Node.js packages
- **Vim**: Node.js plugins

#### 2. Essential Global Packages

```bash
# Nodemon - Auto-restart during development
npm install -g nodemon

# PM2 - Production process manager
npm install -g pm2

# HTTP Server - Quick static file server
npm install -g http-server

# ESLint - Code linting
npm install -g eslint

# TypeScript (if using TypeScript)
npm install -g typescript ts-node
```

## Your First Node.js Application

### Hello World

```javascript
// hello.js
console.log('Hello, Node.js!');
console.log('Node.js version:', process.version);
console.log('Platform:', process.platform);
```

```bash
# Run the application
node hello.js
```

### Simple HTTP Server

```javascript
// server.js
const http = require('http');

// Create HTTP server
const server = http.createServer((req, res) => {
    // Set response header
    res.writeHead(200, { 'Content-Type': 'text/html' });
    
    // Send response
    res.end(`
        <h1>Hello from Node.js Server!</h1>
        <p>Request URL: ${req.url}</p>
        <p>Request Method: ${req.method}</p>
        <p>Current Time: ${new Date().toISOString()}</p>
    `);
});

// Start server
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`Server running at http://localhost:${PORT}/`);
});

// Handle server errors
server.on('error', (err) => {
    console.error('Server error:', err);
});

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('Received SIGTERM, shutting down gracefully');
    server.close(() => {
        console.log('Server closed');
        process.exit(0);
    });
});
```

```bash
# Run the server
node server.js

# Visit http://localhost:3000 in browser
```

### File-based Routing

```javascript
// file-server.js
const http = require('http');
const fs = require('fs');
const path = require('path');

const server = http.createServer((req, res) => {
    let filePath = path.join(__dirname, 'public', req.url === '/' ? 'index.html' : req.url);
    
    // Get file extension
    const extname = String(path.extname(filePath)).toLowerCase();
    
    // MIME types
    const mimeTypes = {
        '.html': 'text/html',
        '.js': 'text/javascript',
        '.css': 'text/css',
        '.json': 'application/json',
        '.png': 'image/png',
        '.jpg': 'image/jpg',
        '.gif': 'image/gif',
        '.svg': 'image/svg+xml',
        '.wav': 'audio/wav',
        '.mp4': 'video/mp4',
        '.woff': 'application/font-woff',
        '.ttf': 'application/font-ttf',
        '.eot': 'application/vnd.ms-fontobject',
        '.otf': 'application/font-otf',
        '.wasm': 'application/wasm'
    };
    
    const contentType = mimeTypes[extname] || 'application/octet-stream';
    
    // Read and serve file
    fs.readFile(filePath, (err, data) => {
        if (err) {
            if (err.code === 'ENOENT') {
                // File not found
                res.writeHead(404, { 'Content-Type': 'text/html' });
                res.end('<h1>404 - File Not Found</h1>');
            } else {
                // Server error
                res.writeHead(500, { 'Content-Type': 'text/html' });
                res.end(`<h1>500 - Server Error</h1><p>${err.code}</p>`);
            }
        } else {
            // Success
            res.writeHead(200, { 'Content-Type': contentType });
            res.end(data, 'utf-8');
        }
    });
});

const PORT = 3000;
server.listen(PORT, () => {
    console.log(`File server running at http://localhost:${PORT}/`);
});
```

### Package.json Setup

```json
{
  "name": "my-node-app",
  "version": "1.0.0",
  "description": "My first Node.js application",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": ["node", "javascript", "server"],
  "author": "Your Name",
  "license": "MIT",
  "dependencies": {},
  "devDependencies": {
    "nodemon": "^3.0.1"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

```bash
# Initialize package.json
npm init -y

# Install dependencies
npm install express
npm install --save-dev nodemon

# Run development server
npm run dev
```

## Global Objects

Node.js provides several global objects that are available in all modules without requiring them.

### Global Object

```javascript
// Global object (similar to window in browsers)
console.log(global);

// Add global variables (not recommended)
global.myGlobalVar = 'Hello Global';
console.log(myGlobalVar); // Accessible anywhere

// Check if running in Node.js
if (typeof global !== 'undefined') {
    console.log('Running in Node.js');
} else {
    console.log('Running in browser');
}
```

### __dirname and __filename

```javascript
// Current directory name
console.log('Directory:', __dirname);

// Current file name with full path
console.log('Filename:', __filename);

// Parse path components
const path = require('path');
console.log('Base name:', path.basename(__filename));
console.log('Extension:', path.extname(__filename));
console.log('Directory name:', path.dirname(__filename));

// Relative path resolution
const configPath = path.join(__dirname, 'config', 'database.json');
console.log('Config path:', configPath);
```

### require and module

```javascript
// Current module information
console.log('Module:', module);
console.log('Module filename:', module.filename);
console.log('Module loaded:', module.loaded);
console.log('Module parent:', module.parent);

// Require cache
console.log('Require cache:', require.cache);

// Clear require cache (useful for testing)
delete require.cache[require.resolve('./some-module')];

// Require resolve
console.log('Module path:', require.resolve('express'));
```

### exports vs module.exports

```javascript
// math.js
exports.add = (a, b) => a + b;
exports.subtract = (a, b) => a - b;

// This won't work as expected
// exports = { multiply: (a, b) => a * b };

// This works
module.exports = {
    multiply: (a, b) => a * b,
    divide: (a, b) => a / b
};

// Or combine both
module.exports.power = (base, exp) => Math.pow(base, exp);

// main.js
const math = require('./math');
console.log(math.multiply(5, 3)); // 15
console.log(math.power(2, 3)); // 8
```

### console

```javascript
// Basic logging
console.log('Info message');
console.info('Info message');
console.warn('Warning message');
console.error('Error message');

// Formatted output
console.log('User: %s, Age: %d', 'John', 30);
console.log('Object: %j', { name: 'John', age: 30 });

// Timing
console.time('operation');
// Some operation
setTimeout(() => {
    console.timeEnd('operation');
}, 1000);

// Counting
console.count('requests');
console.count('requests');
console.countReset('requests');

// Stack trace
console.trace('Trace point');

// Table output
const users = [
    { name: 'John', age: 30 },
    { name: 'Jane', age: 25 }
];
console.table(users);

// Assert
console.assert(1 === 2, 'This will show an error');

// Clear console (in supporting terminals)
console.clear();

// Group output
console.group('User Details');
console.log('Name: John');
console.log('Age: 30');
console.groupEnd();
```

## Process Object

The `process` object provides information about and control over the current Node.js process.

### Process Information

```javascript
// Process information
console.log('Process ID:', process.pid);
console.log('Parent Process ID:', process.ppid);
console.log('Node.js version:', process.version);
console.log('Node.js versions:', process.versions);
console.log('Platform:', process.platform);
console.log('Architecture:', process.arch);
console.log('Current working directory:', process.cwd());
console.log('Executable path:', process.execPath);
console.log('Execution arguments:', process.execArgv);

// Environment variables
console.log('Environment:', process.env);
console.log('NODE_ENV:', process.env.NODE_ENV || 'development');

// Command line arguments
console.log('Arguments:', process.argv);
// node app.js --port 3000 --env production
// process.argv[0] = node path
// process.argv[1] = script path
// process.argv[2] = '--port'
// process.argv[3] = '3000'
// etc.
```

### Command Line Argument Parsing

```javascript
// Simple argument parsing
function parseArgs() {
    const args = {};
    for (let i = 2; i < process.argv.length; i++) {
        const arg = process.argv[i];
        if (arg.startsWith('--')) {
            const key = arg.slice(2);
            const value = process.argv[i + 1];
            args[key] = value;
            i++; // Skip next argument
        }
    }
    return args;
}

const args = parseArgs();
console.log('Parsed arguments:', args);

// Advanced argument parsing with validation
class ArgumentParser {
    constructor() {
        this.definitions = new Map();
        this.parsed = {};
    }
    
    define(name, options = {}) {
        this.definitions.set(name, {
            type: options.type || 'string',
            required: options.required || false,
            default: options.default,
            description: options.description || ''
        });
        return this;
    }
    
    parse(argv = process.argv) {
        const args = argv.slice(2);
        
        for (let i = 0; i < args.length; i++) {
            const arg = args[i];
            
            if (arg.startsWith('--')) {
                const key = arg.slice(2);
                const definition = this.definitions.get(key);
                
                if (!definition) {
                    throw new Error(`Unknown argument: --${key}`);
                }
                
                let value = args[i + 1];
                
                if (definition.type === 'boolean') {
                    value = true;
                } else if (definition.type === 'number') {
                    value = Number(value);
                    if (isNaN(value)) {
                        throw new Error(`Invalid number for --${key}: ${args[i + 1]}`);
                    }
                    i++; // Skip next argument
                } else {
                    if (!value || value.startsWith('--')) {
                        throw new Error(`Missing value for --${key}`);
                    }
                    i++; // Skip next argument
                }
                
                this.parsed[key] = value;
            }
        }
        
        // Check required arguments
        for (const [name, definition] of this.definitions) {
            if (definition.required && !(name in this.parsed)) {
                throw new Error(`Required argument missing: --${name}`);
            }
            
            // Set defaults
            if (!(name in this.parsed) && definition.default !== undefined) {
                this.parsed[name] = definition.default;
            }
        }
        
        return this.parsed;
    }
    
    help() {
        console.log('Usage: node script.js [options]');
        console.log('Options:');
        
        for (const [name, definition] of this.definitions) {
            const required = definition.required ? ' (required)' : '';
            const defaultValue = definition.default !== undefined ? ` [default: ${definition.default}]` : '';
            console.log(`  --${name}  ${definition.description}${required}${defaultValue}`);
        }
    }
}

// Usage
const parser = new ArgumentParser();
parser
    .define('port', { type: 'number', default: 3000, description: 'Server port' })
    .define('host', { type: 'string', default: 'localhost', description: 'Server host' })
    .define('env', { type: 'string', required: true, description: 'Environment' })
    .define('debug', { type: 'boolean', description: 'Enable debug mode' });

try {
    const config = parser.parse();
    console.log('Configuration:', config);
} catch (error) {
    console.error('Error:', error.message);
    parser.help();
    process.exit(1);
}
```

### Process Events and Exit

```javascript
// Exit event
process.on('exit', (code) => {
    console.log(`Process exiting with code: ${code}`);
    // Synchronous cleanup only
});

// Before exit event
process.on('beforeExit', (code) => {
    console.log('Process before exit with code:', code);
    // Can perform asynchronous operations
});

// Uncaught exception
process.on('uncaughtException', (error) => {
    console.error('Uncaught Exception:', error);
    // Log error and exit gracefully
    process.exit(1);
});

// Unhandled promise rejection
process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection at:', promise, 'reason:', reason);
    // Log error and exit gracefully
    process.exit(1);
});

// Signal handling
process.on('SIGINT', () => {
    console.log('Received SIGINT (Ctrl+C), shutting down gracefully');
    // Cleanup and exit
    process.exit(0);
});

process.on('SIGTERM', () => {
    console.log('Received SIGTERM, shutting down gracefully');
    // Cleanup and exit
    process.exit(0);
});

// Manual exit
// process.exit(0); // Success
// process.exit(1); // Error

// Change working directory
try {
    process.chdir('/tmp');
    console.log('Changed directory to:', process.cwd());
} catch (err) {
    console.error('Failed to change directory:', err);
}
```

### Process Memory and CPU

```javascript
// Memory usage
function logMemoryUsage() {
    const usage = process.memoryUsage();
    console.log('Memory Usage:');
    console.log(`  RSS: ${Math.round(usage.rss / 1024 / 1024)} MB`);
    console.log(`  Heap Total: ${Math.round(usage.heapTotal / 1024 / 1024)} MB`);
    console.log(`  Heap Used: ${Math.round(usage.heapUsed / 1024 / 1024)} MB`);
    console.log(`  External: ${Math.round(usage.external / 1024 / 1024)} MB`);
    
    if (usage.arrayBuffers) {
        console.log(`  Array Buffers: ${Math.round(usage.arrayBuffers / 1024 / 1024)} MB`);
    }
}

// CPU usage
function logCPUUsage() {
    const usage = process.cpuUsage();
    console.log('CPU Usage:');
    console.log(`  User: ${usage.user} microseconds`);
    console.log(`  System: ${usage.system} microseconds`);
}

// High-resolution time
function measureExecutionTime() {
    const start = process.hrtime.bigint();
    
    // Some operation
    for (let i = 0; i < 1000000; i++) {
        Math.sqrt(i);
    }
    
    const end = process.hrtime.bigint();
    const duration = Number(end - start) / 1000000; // Convert to milliseconds
    
    console.log(`Execution time: ${duration.toFixed(3)}ms`);
}

// Resource usage monitoring
function startMonitoring() {
    setInterval(() => {
        console.log('\n--- Resource Monitor ---');
        logMemoryUsage();
        logCPUUsage();
        console.log(`Uptime: ${Math.round(process.uptime())} seconds`);
    }, 5000);
}

// Force garbage collection (if --expose-gc flag is used)
if (global.gc) {
    console.log('Garbage collection available');
    global.gc();
    console.log('Garbage collection forced');
} else {
    console.log('Garbage collection not available. Run with --expose-gc');
}
```

## Buffer

Buffer is a Node.js global object for handling binary data. It's similar to arrays but specifically designed for binary data.

### Creating Buffers

```javascript
// Create empty buffer
const buf1 = Buffer.alloc(10); // 10 bytes, filled with zeros
console.log(buf1); // <Buffer 00 00 00 00 00 00 00 00 00 00>

// Create buffer with unsafe allocation (faster but may contain old data)
const buf2 = Buffer.allocUnsafe(10);
console.log(buf2); // Contains random data

// Create buffer from string
const buf3 = Buffer.from('Hello World', 'utf8');
console.log(buf3); // <Buffer 48 65 6c 6c 6f 20 57 6f 72 6c 64>

// Create buffer from array
const buf4 = Buffer.from([72, 101, 108, 108, 111]); // "Hello"
console.log(buf4.toString()); // "Hello"

// Create buffer with fill value
const buf5 = Buffer.alloc(10, 'a');
console.log(buf5.toString()); // "aaaaaaaaaa"
```

### Buffer Operations

```javascript
const buffer = Buffer.from('Hello World');

// Length
console.log('Length:', buffer.length); // 11

// Convert to string
console.log('String:', buffer.toString()); // "Hello World"
console.log('Hex:', buffer.toString('hex')); // "48656c6c6f20576f726c64"
console.log('Base64:', buffer.toString('base64')); // "SGVsbG8gV29ybGQ="

// Access individual bytes
console.log('First byte:', buffer[0]); // 72 ('H')
console.log('As char:', String.fromCharCode(buffer[0])); // 'H'

// Modify buffer
buffer[0] = 74; // 'J'
console.log('Modified:', buffer.toString()); // "Jello World"

// Copy buffer
const copy = Buffer.alloc(buffer.length);
buffer.copy(copy);
console.log('Copy:', copy.toString()); // "Jello World"

// Slice buffer (creates a view, not a copy)
const slice = buffer.slice(0, 5);
console.log('Slice:', slice.toString()); // "Jello"

// Concatenate buffers
const buf1 = Buffer.from('Hello ');
const buf2 = Buffer.from('World');
const combined = Buffer.concat([buf1, buf2]);
console.log('Combined:', combined.toString()); // "Hello World"
```

### Buffer Encoding

```javascript
const text = 'Hello 世界 🌍';

// Different encodings
const utf8Buffer = Buffer.from(text, 'utf8');
const utf16Buffer = Buffer.from(text, 'utf16le');
const asciiBuffer = Buffer.from(text, 'ascii');

console.log('UTF-8:', utf8Buffer.length, 'bytes');
console.log('UTF-16:', utf16Buffer.length, 'bytes');
console.log('ASCII:', asciiBuffer.toString()); // Lossy conversion

// Supported encodings
const encodings = ['ascii', 'utf8', 'utf16le', 'ucs2', 'base64', 'latin1', 'binary', 'hex'];
encodings.forEach(encoding => {
    if (Buffer.isEncoding(encoding)) {
        console.log(`${encoding}: supported`);
    }
});
```

### Binary Data Manipulation

```javascript
// Working with binary data
class BinaryDataHandler {
    constructor() {
        this.buffer = Buffer.alloc(0);
    }
    
    // Write different data types
    writeInt8(value) {
        const buf = Buffer.allocUnsafe(1);
        buf.writeInt8(value, 0);
        this.buffer = Buffer.concat([this.buffer, buf]);
        return this;
    }
    
    writeInt16BE(value) {
        const buf = Buffer.allocUnsafe(2);
        buf.writeInt16BE(value, 0);
        this.buffer = Buffer.concat([this.buffer, buf]);
        return this;
    }
    
    writeInt32BE(value) {
        const buf = Buffer.allocUnsafe(4);
        buf.writeInt32BE(value, 0);
        this.buffer = Buffer.concat([this.buffer, buf]);
        return this;
    }
    
    writeFloatBE(value) {
        const buf = Buffer.allocUnsafe(4);
        buf.writeFloatBE(value, 0);
        this.buffer = Buffer.concat([this.buffer, buf]);
        return this;
    }
    
    writeString(str, encoding = 'utf8') {
        const buf = Buffer.from(str, encoding);
        // Write length first (4 bytes)
        const lengthBuf = Buffer.allocUnsafe(4);
        lengthBuf.writeInt32BE(buf.length, 0);
        this.buffer = Buffer.concat([this.buffer, lengthBuf, buf]);
        return this;
    }
    
    getBuffer() {
        return this.buffer;
    }
    
    // Read data
    static readFrom(buffer) {
        const reader = new BinaryDataReader(buffer);
        return reader;
    }
}

class BinaryDataReader {
    constructor(buffer) {
        this.buffer = buffer;
        this.offset = 0;
    }
    
    readInt8() {
        const value = this.buffer.readInt8(this.offset);
        this.offset += 1;
        return value;
    }
    
    readInt16BE() {
        const value = this.buffer.readInt16BE(this.offset);
        this.offset += 2;
        return value;
    }
    
    readInt32BE() {
        const value = this.buffer.readInt32BE(this.offset);
        this.offset += 4;
        return value;
    }
    
    readFloatBE() {
        const value = this.buffer.readFloatBE(this.offset);
        this.offset += 4;
        return value;
    }
    
    readString(encoding = 'utf8') {
        const length = this.readInt32BE();
        const str = this.buffer.toString(encoding, this.offset, this.offset + length);
        this.offset += length;
        return str;
    }
}

// Usage
const writer = new BinaryDataHandler();
writer
    .writeInt8(42)
    .writeInt16BE(1000)
    .writeInt32BE(100000)
    .writeFloatBE(3.14159)
    .writeString('Hello World');

const binaryData = writer.getBuffer();
console.log('Binary data:', binaryData);
console.log('Hex representation:', binaryData.toString('hex'));

// Read back the data
const reader = BinaryDataHandler.readFrom(binaryData);
console.log('Int8:', reader.readInt8());
console.log('Int16BE:', reader.readInt16BE());
console.log('Int32BE:', reader.readInt32BE());
console.log('FloatBE:', reader.readFloatBE());
console.log('String:', reader.readString());
```

### Buffer Utilities

```javascript
// Buffer comparison
const buf1 = Buffer.from('ABC');
const buf2 = Buffer.from('ABC');
const buf3 = Buffer.from('BCD');

console.log('buf1 equals buf2:', buf1.equals(buf2)); // true
console.log('buf1 compare buf3:', buf1.compare(buf3)); // -1 (buf1 < buf3)

// Buffer includes
const haystack = Buffer.from('Hello World');
const needle = Buffer.from('World');
console.log('Includes World:', haystack.includes(needle)); // true
console.log('Index of World:', haystack.indexOf(needle)); // 6

// Buffer swap (byte order)
const buf = Buffer.from([0x01, 0x02, 0x03, 0x04]);
console.log('Original:', buf);
buf.swap16(); // Swap every 2 bytes
console.log('Swapped 16:', buf);

// JSON serialization
const buffer = Buffer.from('Hello');
const json = JSON.stringify(buffer);
console.log('JSON:', json); // {"type":"Buffer","data":[72,101,108,108,111]}

const restored = Buffer.from(JSON.parse(json).data);
console.log('Restored:', restored.toString()); // "Hello"

// Buffer pool information
console.log('Buffer pool size:', Buffer.poolSize);
```

## Timers

Node.js provides several timer functions for scheduling code execution.

### setTimeout and setInterval

```javascript
// setTimeout - execute once after delay
const timeoutId = setTimeout(() => {
    console.log('This runs after 2 seconds');
}, 2000);

// Clear timeout
clearTimeout(timeoutId);

// setTimeout with arguments
setTimeout((name, age) => {
    console.log(`Hello ${name}, you are ${age} years old`);
}, 1000, 'John', 30);

// setInterval - execute repeatedly
const intervalId = setInterval(() => {
    console.log('This runs every second');
}, 1000);

// Clear after 5 seconds
setTimeout(() => {
    clearInterval(intervalId);
    console.log('Interval cleared');
}, 5000);
```

### setImmediate and process.nextTick

```javascript
// setImmediate - execute in next iteration of event loop
setImmediate(() => {
    console.log('setImmediate callback');
});

// process.nextTick - execute before next event loop iteration
process.nextTick(() => {
    console.log('nextTick callback');
});

console.log('Synchronous operation');

// Output order:
// Synchronous operation
// nextTick callback
// setImmediate callback
```

### Advanced Timer Patterns

```javascript
// Timer with cleanup
class Timer {
    constructor() {
        this.timers = new Set();
    }
    
    setTimeout(callback, delay, ...args) {
        const timerId = setTimeout(() => {
            this.timers.delete(timerId);
            callback(...args);
        }, delay);
        
        this.timers.add(timerId);
        return timerId;
    }
    
    setInterval(callback, interval, ...args) {
        const timerId = setInterval(callback, interval, ...args);
        this.timers.add(timerId);
        return timerId;
    }
    
    clear(timerId) {
        if (this.timers.has(timerId)) {
            clearTimeout(timerId);
            clearInterval(timerId);
            this.timers.delete(timerId);
        }
    }
    
    clearAll() {
        this.timers.forEach(timerId => {
            clearTimeout(timerId);
            clearInterval(timerId);
        });
        this.timers.clear();
    }
}

// Usage
const timer = new Timer();

const id1 = timer.setTimeout(() => console.log('Timer 1'), 1000);
const id2 = timer.setInterval(() => console.log('Timer 2'), 500);

// Clear all timers after 3 seconds
setTimeout(() => {
    timer.clearAll();
    console.log('All timers cleared');
}, 3000);

// Debounced function
function debounce(func, delay) {
    let timeoutId;
    
    return function(...args) {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => func.apply(this, args), delay);
    };
}

// Throttled function
function throttle(func, limit) {
    let inThrottle;
    
    return function(...args) {
        if (!inThrottle) {
            func.apply(this, args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}

// Usage
const debouncedLog = debounce(console.log, 300);
const throttledLog = throttle(console.log, 300);

// Retry with exponential backoff
async function retryWithBackoff(operation, maxRetries = 3, baseDelay = 1000) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            return await operation();
        } catch (error) {
            if (attempt === maxRetries) {
                throw error;
            }
            
            const delay = baseDelay * Math.pow(2, attempt - 1);
            console.log(`Attempt ${attempt} failed, retrying in ${delay}ms`);
            
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
}

// Usage
async function unreliableOperation() {
    if (Math.random() < 0.7) {
        throw new Error('Operation failed');
    }
    return 'Success!';
}

retryWithBackoff(unreliableOperation)
    .then(result => console.log('Result:', result))
    .catch(error => console.error('Final error:', error));
```

## Error Handling in Node.js

### Synchronous Error Handling

```javascript
// Try-catch for synchronous code
try {
    const data = JSON.parse('invalid json');
} catch (error) {
    console.error('JSON parse error:', error.message);
}

// Custom error classes
class ValidationError extends Error {
    constructor(message, field) {
        super(message);
        this.name = 'ValidationError';
        this.field = field;
    }
}

class DatabaseError extends Error {
    constructor(message, code) {
        super(message);
        this.name = 'DatabaseError';
        this.code = code;
    }
}

// Throwing custom errors
function validateUser(user) {
    if (!user.name) {
        throw new ValidationError('Name is required', 'name');
    }
    
    if (!user.email) {
        throw new ValidationError('Email is required', 'email');
    }
    
    if (!/\S+@\S+\.\S+/.test(user.email)) {
        throw new ValidationError('Invalid email format', 'email');
    }
}

// Using custom errors
try {
    validateUser({ name: 'John' });
} catch (error) {
    if (error instanceof ValidationError) {
        console.error(`Validation error in ${error.field}: ${error.message}`);
    } else {
        console.error('Unexpected error:', error);
    }
}
```

### Asynchronous Error Handling

```javascript
const fs = require('fs').promises;

// Async/await error handling
async function readFileWithErrorHandling(filename) {
    try {
        const data = await fs.readFile(filename, 'utf8');
        return data;
    } catch (error) {
        if (error.code === 'ENOENT') {
            console.error('File not found:', filename);
        } else if (error.code === 'EACCES') {
            console.error('Permission denied:', filename);
        } else {
            console.error('Unexpected error:', error);
        }
        throw error; // Re-throw if needed
    }
}

// Promise error handling
function readFileWithPromises(filename) {
    return fs.readFile(filename, 'utf8')
        .then(data => {
            console.log('File read successfully');
            return data;
        })
        .catch(error => {
            console.error('Error reading file:', error.message);
            throw error;
        });
}

// Callback error handling (Node.js convention)
const fsCallback = require('fs');

function readFileWithCallback(filename, callback) {
    fsCallback.readFile(filename, 'utf8', (error, data) => {
        if (error) {
            return callback(error);
        }
        callback(null, data);
    });
}

// Usage
readFileWithCallback('example.txt', (error, data) => {
    if (error) {
        console.error('Callback error:', error.message);
        return;
    }
    console.log('File content:', data);
});
```

### Error Handling Patterns

```javascript
// Error-first callback pattern
function asyncOperation(input, callback) {
    setImmediate(() => {
        if (!input) {
            return callback(new Error('Input is required'));
        }
        
        if (typeof input !== 'string') {
            return callback(new TypeError('Input must be a string'));
        }
        
        // Success
        callback(null, `Processed: ${input}`);
    });
}

// Promisify callback-based function
const { promisify } = require('util');

const asyncOperationPromise = promisify(asyncOperation);

// Usage
asyncOperationPromise('test')
    .then(result => console.log(result))
    .catch(error => console.error(error));

// Error handling middleware pattern
class ErrorHandler {
    constructor() {
        this.handlers = [];
    }
    
    use(handler) {
        this.handlers.push(handler);
    }
    
    async handle(error, context = {}) {
        for (const handler of this.handlers) {
            try {
                const handled = await handler(error, context);
                if (handled) {
                    return;
                }
            } catch (handlerError) {
                console.error('Error in error handler:', handlerError);
            }
        }
        
        // Default handling
        console.error('Unhandled error:', error);
    }
}

// Error handlers
const errorHandler = new ErrorHandler();

errorHandler.use(async (error, context) => {
    if (error instanceof ValidationError) {
        console.log('Validation error handled:', error.message);
        return true; // Handled
    }
    return false; // Not handled
});

errorHandler.use(async (error, context) => {
    if (error.code === 'ENOENT') {
        console.log('File not found error handled');
        return true;
    }
    return false;
});

// Circuit breaker pattern for error handling
class CircuitBreaker {
    constructor(operation, options = {}) {
        this.operation = operation;
        this.failureThreshold = options.failureThreshold || 5;
        this.resetTimeout = options.resetTimeout || 60000;
        this.monitoringPeriod = options.monitoringPeriod || 10000;
        
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
        this.failureCount = 0;
        this.lastFailureTime = null;
        this.successCount = 0;
    }
    
    async execute(...args) {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime > this.resetTimeout) {
                this.state = 'HALF_OPEN';
                this.successCount = 0;
            } else {
                throw new Error('Circuit breaker is OPEN');
            }
        }
        
        try {
            const result = await this.operation(...args);
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }
    
    onSuccess() {
        this.failureCount = 0;
        
        if (this.state === 'HALF_OPEN') {
            this.successCount++;
            if (this.successCount >= 3) {
                this.state = 'CLOSED';
            }
        }
    }
    
    onFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();
        
        if (this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
        }
    }
    
    getState() {
        return this.state;
    }
}

// Usage
async function unreliableService() {
    if (Math.random() < 0.7) {
        throw new Error('Service unavailable');
    }
    return 'Service response';
}

const circuitBreaker = new CircuitBreaker(unreliableService, {
    failureThreshold: 3,
    resetTimeout: 5000
});

// Test circuit breaker
async function testCircuitBreaker() {
    for (let i = 0; i < 10; i++) {
        try {
            const result = await circuitBreaker.execute();
            console.log(`Attempt ${i + 1}: ${result}`);
        } catch (error) {
            console.log(`Attempt ${i + 1}: ${error.message} (State: ${circuitBreaker.getState()})`);
        }
        
        await new Promise(resolve => setTimeout(resolve, 1000));
    }
}

testCircuitBreaker();
```

---

This completes the comprehensive Node.js Introduction and Architecture section. The content covers the fundamentals of Node.js, its architecture, event loop, installation, basic applications, global objects, process management, buffers, timers, and error handling patterns.
