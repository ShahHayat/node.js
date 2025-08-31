# Node.js Advanced Topics - Complete Guide

## Table of Contents
1. [Clustering and Scaling](#clustering-and-scaling)
2. [Worker Threads](#worker-threads)
3. [Child Processes](#child-processes)
4. [Performance Optimization](#performance-optimization)
5. [Security Best Practices](#security-best-practices)
6. [Testing Strategies](#testing-strategies)
7. [Deployment and DevOps](#deployment-and-devops)
8. [Monitoring and Logging](#monitoring-and-logging)
9. [Memory Management](#memory-management)
10. [Microservices Architecture](#microservices-architecture)
11. [Real-time Applications](#real-time-applications)
12. [Production Best Practices](#production-best-practices)

## Clustering and Scaling

Node.js runs on a single thread, but you can utilize multiple CPU cores using the cluster module.

### Basic Clustering

```javascript
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;
const process = require('process');

if (cluster.isMaster) {
    console.log(`Master ${process.pid} is running`);
    
    // Fork workers
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }
    
    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died`);
        console.log('Starting a new worker');
        cluster.fork();
    });
    
} else {
    // Worker process
    const express = require('express');
    const app = express();
    
    app.get('/', (req, res) => {
        res.json({
            message: 'Hello from Node.js cluster!',
            pid: process.pid,
            worker: cluster.worker.id
        });
    });
    
    const PORT = process.env.PORT || 3000;
    app.listen(PORT, () => {
        console.log(`Worker ${process.pid} started on port ${PORT}`);
    });
}
```

### Advanced Cluster Management

```javascript
const cluster = require('cluster');
const os = require('os');
const EventEmitter = require('events');

class ClusterManager extends EventEmitter {
    constructor(options = {}) {
        super();
        this.numWorkers = options.numWorkers || os.cpus().length;
        this.workers = new Map();
        this.restartDelay = options.restartDelay || 1000;
        this.maxRestarts = options.maxRestarts || 5;
        this.workerScript = options.workerScript || './worker.js';
        this.gracefulShutdownTimeout = options.gracefulShutdownTimeout || 30000;
    }
    
    start() {
        if (!cluster.isMaster) {
            throw new Error('ClusterManager should only run in master process');
        }
        
        console.log(`🚀 Master ${process.pid} starting with ${this.numWorkers} workers`);
        
        // Fork workers
        for (let i = 0; i < this.numWorkers; i++) {
            this.forkWorker();
        }
        
        this.setupEventHandlers();
        this.setupGracefulShutdown();
        
        this.emit('started', { masterPid: process.pid, numWorkers: this.numWorkers });
    }
    
    forkWorker() {
        const worker = cluster.fork();
        
        this.workers.set(worker.id, {
            worker,
            restarts: 0,
            startTime: Date.now(),
            lastRestart: null
        });
        
        worker.on('message', (message) => {
            this.handleWorkerMessage(worker, message);
        });
        
        worker.on('exit', (code, signal) => {
            this.handleWorkerExit(worker, code, signal);
        });
        
        worker.on('error', (error) => {
            console.error(`❌ Worker ${worker.id} error:`, error);
        });
        
        this.emit('worker:started', { workerId: worker.id, pid: worker.process.pid });
        
        return worker;
    }
    
    handleWorkerMessage(worker, message) {
        switch (message.type) {
            case 'health':
                this.emit('worker:health', { 
                    workerId: worker.id, 
                    health: message.data 
                });
                break;
                
            case 'metrics':
                this.emit('worker:metrics', {
                    workerId: worker.id,
                    metrics: message.data
                });
                break;
                
            case 'error':
                this.emit('worker:error', {
                    workerId: worker.id,
                    error: message.data
                });
                break;
                
            default:
                this.emit('worker:message', { workerId: worker.id, message });
        }
    }
    
    handleWorkerExit(worker, code, signal) {
        const workerInfo = this.workers.get(worker.id);
        
        if (!workerInfo) return;
        
        console.log(`💀 Worker ${worker.id} (PID: ${worker.process.pid}) died`);
        console.log(`   Exit code: ${code}, Signal: ${signal}`);
        
        this.workers.delete(worker.id);
        
        // Check if we should restart
        if (workerInfo.restarts < this.maxRestarts) {
            setTimeout(() => {
                console.log(`🔄 Restarting worker ${worker.id}`);
                const newWorker = this.forkWorker();
                
                // Update restart count
                const newWorkerInfo = this.workers.get(newWorker.id);
                newWorkerInfo.restarts = workerInfo.restarts + 1;
                newWorkerInfo.lastRestart = Date.now();
                
            }, this.restartDelay);
        } else {
            console.error(`❌ Worker ${worker.id} exceeded max restarts (${this.maxRestarts})`);
            this.emit('worker:failed', { workerId: worker.id });
        }
        
        this.emit('worker:exit', { 
            workerId: worker.id, 
            code, 
            signal, 
            restarts: workerInfo.restarts 
        });
    }
    
    setupEventHandlers() {
        cluster.on('fork', (worker) => {
            console.log(`👶 Worker ${worker.id} forked (PID: ${worker.process.pid})`);
        });
        
        cluster.on('online', (worker) => {
            console.log(`✅ Worker ${worker.id} is online`);
        });
        
        cluster.on('listening', (worker, address) => {
            console.log(`👂 Worker ${worker.id} listening on ${address.address}:${address.port}`);
        });
        
        cluster.on('disconnect', (worker) => {
            console.log(`📡 Worker ${worker.id} disconnected`);
        });
    }
    
    setupGracefulShutdown() {
        const shutdown = (signal) => {
            console.log(`🔄 Received ${signal}, shutting down gracefully`);
            
            this.emit('shutdown:start', { signal });
            
            // Stop accepting new connections
            Object.values(cluster.workers).forEach(worker => {
                worker.send({ type: 'shutdown' });
            });
            
            // Wait for workers to finish
            setTimeout(() => {
                console.log('⏰ Shutdown timeout reached, forcing exit');
                Object.values(cluster.workers).forEach(worker => {
                    worker.kill();
                });
                process.exit(0);
            }, this.gracefulShutdownTimeout);
            
            // Wait for all workers to exit
            let workersAlive = Object.keys(cluster.workers).length;
            
            cluster.on('exit', () => {
                workersAlive--;
                if (workersAlive === 0) {
                    console.log('✅ All workers exited, master shutting down');
                    this.emit('shutdown:complete');
                    process.exit(0);
                }
            });
        };
        
        process.on('SIGTERM', shutdown);
        process.on('SIGINT', shutdown);
    }
    
    // Send message to all workers
    broadcast(message) {
        Object.values(cluster.workers).forEach(worker => {
            worker.send(message);
        });
    }
    
    // Send message to specific worker
    sendToWorker(workerId, message) {
        const worker = cluster.workers[workerId];
        if (worker) {
            worker.send(message);
        }
    }
    
    // Get cluster statistics
    getStats() {
        const workers = Object.values(cluster.workers).map(worker => {
            const workerInfo = this.workers.get(worker.id);
            return {
                id: worker.id,
                pid: worker.process.pid,
                state: worker.state,
                uptime: Date.now() - workerInfo.startTime,
                restarts: workerInfo.restarts,
                lastRestart: workerInfo.lastRestart
            };
        });
        
        return {
            masterPid: process.pid,
            numWorkers: workers.length,
            workers
        };
    }
    
    // Reload workers (zero-downtime deployment)
    async reload() {
        console.log('🔄 Reloading all workers...');
        
        const workers = Object.values(cluster.workers);
        
        for (const worker of workers) {
            await this.reloadWorker(worker);
            // Wait a bit before reloading next worker
            await new Promise(resolve => setTimeout(resolve, 2000));
        }
        
        console.log('✅ All workers reloaded');
        this.emit('reload:complete');
    }
    
    async reloadWorker(worker) {
        return new Promise((resolve) => {
            const newWorker = this.forkWorker();
            
            newWorker.once('listening', () => {
                // Gracefully shutdown old worker
                worker.send({ type: 'shutdown' });
                
                setTimeout(() => {
                    if (!worker.isDead()) {
                        worker.kill();
                    }
                    resolve();
                }, 5000);
            });
        });
    }
}

// worker.js - Worker process code
if (!cluster.isMaster) {
    const express = require('express');
    const app = express();
    
    // Health monitoring
    let isShuttingDown = false;
    const startTime = Date.now();
    
    // Middleware
    app.use(express.json());
    
    // Health check endpoint
    app.get('/health', (req, res) => {
        const memUsage = process.memoryUsage();
        
        res.json({
            status: isShuttingDown ? 'shutting-down' : 'healthy',
            pid: process.pid,
            workerId: cluster.worker.id,
            uptime: Date.now() - startTime,
            memory: {
                rss: Math.round(memUsage.rss / 1024 / 1024) + 'MB',
                heapUsed: Math.round(memUsage.heapUsed / 1024 / 1024) + 'MB',
                heapTotal: Math.round(memUsage.heapTotal / 1024 / 1024) + 'MB'
            }
        });
    });
    
    // API routes
    app.get('/api/data', (req, res) => {
        res.json({
            message: 'Data from worker',
            pid: process.pid,
            timestamp: new Date().toISOString()
        });
    });
    
    // CPU-intensive route (demonstrates load distribution)
    app.get('/api/cpu-intensive', (req, res) => {
        const iterations = 1000000;
        let result = 0;
        
        for (let i = 0; i < iterations; i++) {
            result += Math.sqrt(i);
        }
        
        res.json({
            result,
            pid: process.pid,
            iterations
        });
    });
    
    const server = app.listen(3000, () => {
        console.log(`Worker ${cluster.worker.id} (PID: ${process.pid}) listening on port 3000`);
    });
    
    // Handle messages from master
    process.on('message', (message) => {
        if (message.type === 'shutdown') {
            console.log(`Worker ${cluster.worker.id} received shutdown signal`);
            isShuttingDown = true;
            
            // Stop accepting new connections
            server.close(() => {
                console.log(`Worker ${cluster.worker.id} server closed`);
                process.exit(0);
            });
        }
    });
    
    // Send health updates to master
    setInterval(() => {
        process.send({
            type: 'health',
            data: {
                pid: process.pid,
                memory: process.memoryUsage(),
                uptime: process.uptime()
            }
        });
    }, 10000);
}

// Usage
if (cluster.isMaster) {
    const clusterManager = new ClusterManager({
        numWorkers: 4,
        restartDelay: 2000,
        maxRestarts: 3
    });
    
    clusterManager.on('worker:started', ({ workerId, pid }) => {
        console.log(`✅ Worker ${workerId} started with PID ${pid}`);
    });
    
    clusterManager.on('worker:exit', ({ workerId, code, signal }) => {
        console.log(`💀 Worker ${workerId} exited with code ${code}, signal ${signal}`);
    });
    
    clusterManager.start();
    
    // API to get cluster stats
    const express = require('express');
    const masterApp = express();
    
    masterApp.get('/cluster/stats', (req, res) => {
        res.json(clusterManager.getStats());
    });
    
    masterApp.post('/cluster/reload', async (req, res) => {
        await clusterManager.reload();
        res.json({ message: 'Cluster reloaded successfully' });
    });
    
    masterApp.listen(3001, () => {
        console.log('Master API listening on port 3001');
    });
}
```

### Load Balancing Strategies

```javascript
const cluster = require('cluster');

class LoadBalancer extends EventEmitter {
    constructor() {
        super();
        this.workers = new Map();
        this.currentWorker = 0;
        this.strategies = {
            'round-robin': this.roundRobin.bind(this),
            'least-connections': this.leastConnections.bind(this),
            'cpu-usage': this.cpuUsage.bind(this),
            'memory-usage': this.memoryUsage.bind(this)
        };
        this.currentStrategy = 'round-robin';
    }
    
    addWorker(worker) {
        this.workers.set(worker.id, {
            worker,
            connections: 0,
            cpuUsage: 0,
            memoryUsage: 0,
            responseTime: 0,
            lastUpdate: Date.now()
        });
        
        worker.on('message', (message) => {
            if (message.type === 'stats') {
                this.updateWorkerStats(worker.id, message.data);
            }
        });
    }
    
    updateWorkerStats(workerId, stats) {
        const workerInfo = this.workers.get(workerId);
        if (workerInfo) {
            workerInfo.connections = stats.connections || 0;
            workerInfo.cpuUsage = stats.cpuUsage || 0;
            workerInfo.memoryUsage = stats.memoryUsage || 0;
            workerInfo.responseTime = stats.responseTime || 0;
            workerInfo.lastUpdate = Date.now();
        }
    }
    
    // Round-robin strategy
    roundRobin() {
        const workers = Array.from(this.workers.values());
        if (workers.length === 0) return null;
        
        const worker = workers[this.currentWorker % workers.length];
        this.currentWorker++;
        
        return worker.worker;
    }
    
    // Least connections strategy
    leastConnections() {
        let selectedWorker = null;
        let minConnections = Infinity;
        
        for (const workerInfo of this.workers.values()) {
            if (workerInfo.connections < minConnections) {
                minConnections = workerInfo.connections;
                selectedWorker = workerInfo.worker;
            }
        }
        
        return selectedWorker;
    }
    
    // CPU usage strategy
    cpuUsage() {
        let selectedWorker = null;
        let minCpuUsage = Infinity;
        
        for (const workerInfo of this.workers.values()) {
            if (workerInfo.cpuUsage < minCpuUsage) {
                minCpuUsage = workerInfo.cpuUsage;
                selectedWorker = workerInfo.worker;
            }
        }
        
        return selectedWorker;
    }
    
    // Memory usage strategy
    memoryUsage() {
        let selectedWorker = null;
        let minMemoryUsage = Infinity;
        
        for (const workerInfo of this.workers.values()) {
            if (workerInfo.memoryUsage < minMemoryUsage) {
                minMemoryUsage = workerInfo.memoryUsage;
                selectedWorker = workerInfo.worker;
            }
        }
        
        return selectedWorker;
    }
    
    // Get next worker based on current strategy
    getNextWorker() {
        const strategy = this.strategies[this.currentStrategy];
        return strategy();
    }
    
    setStrategy(strategyName) {
        if (this.strategies[strategyName]) {
            this.currentStrategy = strategyName;
            console.log(`🔄 Load balancing strategy changed to: ${strategyName}`);
        } else {
            throw new Error(`Unknown strategy: ${strategyName}`);
        }
    }
    
    getStats() {
        const workers = Array.from(this.workers.values()).map(workerInfo => ({
            id: workerInfo.worker.id,
            pid: workerInfo.worker.process.pid,
            connections: workerInfo.connections,
            cpuUsage: workerInfo.cpuUsage,
            memoryUsage: workerInfo.memoryUsage,
            responseTime: workerInfo.responseTime,
            uptime: Date.now() - workerInfo.lastUpdate
        }));
        
        return {
            strategy: this.currentStrategy,
            totalWorkers: workers.length,
            workers
        };
    }
}

// Usage in master process
if (cluster.isMaster) {
    const loadBalancer = new LoadBalancer();
    
    // Create workers
    for (let i = 0; i < os.cpus().length; i++) {
        const worker = cluster.fork();
        loadBalancer.addWorker(worker);
    }
    
    // Monitor load balancer
    loadBalancer.on('worker:started', ({ workerId, pid }) => {
        console.log(`⚖️ Worker ${workerId} added to load balancer`);
    });
    
    // Periodically log stats
    setInterval(() => {
        console.log('📊 Load balancer stats:', loadBalancer.getStats());
    }, 30000);
}
```

## Worker Threads

Worker threads allow you to run JavaScript in parallel, useful for CPU-intensive tasks.

### Basic Worker Threads

```javascript
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
    // Main thread
    class WorkerPool {
        constructor(workerScript, poolSize = 4) {
            this.workerScript = workerScript;
            this.poolSize = poolSize;
            this.workers = [];
            this.queue = [];
            this.currentWorker = 0;
            
            this.initializeWorkers();
        }
        
        initializeWorkers() {
            for (let i = 0; i < this.poolSize; i++) {
                this.createWorker();
            }
        }
        
        createWorker() {
            const worker = new Worker(__filename); // Use current file as worker
            
            worker.on('error', (error) => {
                console.error('Worker error:', error);
                this.replaceWorker(worker);
            });
            
            worker.on('exit', (code) => {
                if (code !== 0) {
                    console.error(`Worker stopped with exit code ${code}`);
                    this.replaceWorker(worker);
                }
            });
            
            this.workers.push({
                worker,
                busy: false,
                tasksCompleted: 0
            });
        }
        
        replaceWorker(deadWorker) {
            const index = this.workers.findIndex(w => w.worker === deadWorker);
            if (index !== -1) {
                this.workers.splice(index, 1);
                this.createWorker();
            }
        }
        
        async execute(data) {
            return new Promise((resolve, reject) => {
                const task = { data, resolve, reject };
                
                const availableWorker = this.workers.find(w => !w.busy);
                
                if (availableWorker) {
                    this.runTask(availableWorker, task);
                } else {
                    this.queue.push(task);
                }
            });
        }
        
        runTask(workerInfo, task) {
            workerInfo.busy = true;
            
            const messageHandler = (result) => {
                workerInfo.busy = false;
                workerInfo.tasksCompleted++;
                workerInfo.worker.off('message', messageHandler);
                workerInfo.worker.off('error', errorHandler);
                
                if (result.error) {
                    task.reject(new Error(result.error));
                } else {
                    task.resolve(result.data);
                }
                
                // Process next task in queue
                if (this.queue.length > 0) {
                    const nextTask = this.queue.shift();
                    this.runTask(workerInfo, nextTask);
                }
            };
            
            const errorHandler = (error) => {
                workerInfo.busy = false;
                workerInfo.worker.off('message', messageHandler);
                workerInfo.worker.off('error', errorHandler);
                task.reject(error);
            };
            
            workerInfo.worker.once('message', messageHandler);
            workerInfo.worker.once('error', errorHandler);
            workerInfo.worker.postMessage(task.data);
        }
        
        getStats() {
            return {
                poolSize: this.poolSize,
                queueLength: this.queue.length,
                workers: this.workers.map(w => ({
                    busy: w.busy,
                    tasksCompleted: w.tasksCompleted
                }))
            };
        }
        
        async terminate() {
            await Promise.all(
                this.workers.map(w => w.worker.terminate())
            );
            this.workers = [];
            this.queue = [];
        }
    }
    
    // Example usage
    async function main() {
        const pool = new WorkerPool(__filename, 4);
        
        // CPU-intensive tasks
        const tasks = [];
        for (let i = 0; i < 10; i++) {
            tasks.push(pool.execute({
                type: 'fibonacci',
                number: 35 + i
            }));
        }
        
        console.time('Parallel execution');
        const results = await Promise.all(tasks);
        console.timeEnd('Parallel execution');
        
        console.log('Results:', results);
        console.log('Pool stats:', pool.getStats());
        
        await pool.terminate();
    }
    
    main().catch(console.error);
    
} else {
    // Worker thread
    parentPort.on('message', async (data) => {
        try {
            let result;
            
            switch (data.type) {
                case 'fibonacci':
                    result = fibonacci(data.number);
                    break;
                    
                case 'prime-check':
                    result = isPrime(data.number);
                    break;
                    
                case 'hash':
                    const crypto = require('crypto');
                    result = crypto.createHash('sha256').update(data.text).digest('hex');
                    break;
                    
                default:
                    throw new Error(`Unknown task type: ${data.type}`);
            }
            
            parentPort.postMessage({ data: result });
        } catch (error) {
            parentPort.postMessage({ error: error.message });
        }
    });
    
    function fibonacci(n) {
        if (n <= 1) return n;
        return fibonacci(n - 1) + fibonacci(n - 2);
    }
    
    function isPrime(n) {
        if (n <= 1) return false;
        if (n <= 3) return true;
        if (n % 2 === 0 || n % 3 === 0) return false;
        
        for (let i = 5; i * i <= n; i += 6) {
            if (n % i === 0 || n % (i + 2) === 0) {
                return false;
            }
        }
        
        return true;
    }
}
```

### Shared Array Buffer and Atomics

```javascript
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
    // Main thread - Shared memory example
    class SharedCounterManager {
        constructor(numWorkers = 4) {
            this.numWorkers = numWorkers;
            this.workers = [];
            
            // Create shared memory (4 bytes for counter)
            this.sharedBuffer = new SharedArrayBuffer(16);
            this.sharedArray = new Int32Array(this.sharedBuffer);
            
            // Initialize shared data
            Atomics.store(this.sharedArray, 0, 0); // counter
            Atomics.store(this.sharedArray, 1, 0); // total operations
            
            this.createWorkers();
        }
        
        createWorkers() {
            for (let i = 0; i < this.numWorkers; i++) {
                const worker = new Worker(__filename, {
                    workerData: {
                        sharedBuffer: this.sharedBuffer,
                        workerId: i
                    }
                });
                
                worker.on('message', (message) => {
                    console.log(`Worker ${i}: ${message}`);
                });
                
                worker.on('error', (error) => {
                    console.error(`Worker ${i} error:`, error);
                });
                
                this.workers.push(worker);
            }
        }
        
        getCounterValue() {
            return Atomics.load(this.sharedArray, 0);
        }
        
        getTotalOperations() {
            return Atomics.load(this.sharedArray, 1);
        }
        
        async terminate() {
            await Promise.all(this.workers.map(worker => worker.terminate()));
        }
    }
    
    // Usage
    async function demonstrateSharedMemory() {
        const manager = new SharedCounterManager(4);
        
        // Let workers run for 5 seconds
        await new Promise(resolve => setTimeout(resolve, 5000));
        
        console.log('Final counter value:', manager.getCounterValue());
        console.log('Total operations:', manager.getTotalOperations());
        
        await manager.terminate();
    }
    
    demonstrateSharedMemory();
    
} else {
    // Worker thread
    const { sharedBuffer, workerId } = workerData;
    const sharedArray = new Int32Array(sharedBuffer);
    
    // Simulate work with shared counter
    function doWork() {
        // Atomically increment counter
        const oldValue = Atomics.load(sharedArray, 0);
        const newValue = oldValue + 1;
        
        // Use compare and exchange for thread-safe increment
        const success = Atomics.compareExchange(sharedArray, 0, oldValue, newValue);
        
        if (success === oldValue) {
            // Successfully incremented
            Atomics.add(sharedArray, 1, 1); // Increment operation counter
            parentPort.postMessage(`Incremented counter to ${newValue}`);
        } else {
            // Another worker modified the counter, try again
            setImmediate(doWork);
        }
    }
    
    // Start working
    const interval = setInterval(doWork, 100);
    
    // Stop after some time
    setTimeout(() => {
        clearInterval(interval);
        parentPort.postMessage(`Worker ${workerId} finished`);
    }, 4000);
}
```

## Child Processes

Child processes allow you to spawn new processes and communicate with them.

### Spawn, Exec, and Fork

```javascript
const { spawn, exec, execFile, fork } = require('child_process');
const path = require('path');

class ProcessManager {
    constructor() {
        this.processes = new Map();
    }
    
    // Spawn process (streaming output)
    async spawnProcess(command, args = [], options = {}) {
        return new Promise((resolve, reject) => {
            const process = spawn(command, args, {
                stdio: ['pipe', 'pipe', 'pipe'],
                ...options
            });
            
            let stdout = '';
            let stderr = '';
            
            process.stdout.on('data', (data) => {
                stdout += data.toString();
                console.log(`📤 ${command}: ${data}`);
            });
            
            process.stderr.on('data', (data) => {
                stderr += data.toString();
                console.error(`❌ ${command}: ${data}`);
            });
            
            process.on('close', (code) => {
                if (code === 0) {
                    resolve({ stdout, stderr, code });
                } else {
                    reject(new Error(`Process exited with code ${code}: ${stderr}`));
                }
            });
            
            process.on('error', (error) => {
                reject(error);
            });
            
            // Store process reference
            this.processes.set(process.pid, {
                process,
                command,
                args,
                startTime: Date.now()
            });
        });
    }
    
    // Execute command (buffer output)
    async execCommand(command, options = {}) {
        return new Promise((resolve, reject) => {
            exec(command, {
                maxBuffer: 1024 * 1024, // 1MB buffer
                timeout: 30000,
                ...options
            }, (error, stdout, stderr) => {
                if (error) {
                    reject(error);
                } else {
                    resolve({ stdout, stderr });
                }
            });
        });
    }
    
    // Fork Node.js process
    forkNodeProcess(scriptPath, args = [], options = {}) {
        const childProcess = fork(scriptPath, args, {
            silent: false,
            ...options
        });
        
        this.processes.set(childProcess.pid, {
            process: childProcess,
            type: 'fork',
            script: scriptPath,
            startTime: Date.now()
        });
        
        return childProcess;
    }
    
    // Execute shell script
    async executeScript(scriptPath, args = []) {
        try {
            const result = await this.spawnProcess('node', [scriptPath, ...args]);
            console.log(`✅ Script ${scriptPath} completed successfully`);
            return result;
        } catch (error) {
            console.error(`❌ Script ${scriptPath} failed:`, error.message);
            throw error;
        }
    }
    
    // Execute system command
    async executeSystemCommand(command) {
        try {
            console.log(`🔧 Executing: ${command}`);
            const result = await this.execCommand(command);
            console.log(`✅ Command completed: ${command}`);
            return result;
        } catch (error) {
            console.error(`❌ Command failed: ${command}`, error.message);
            throw error;
        }
    }
    
    // Kill process
    killProcess(pid, signal = 'SIGTERM') {
        const processInfo = this.processes.get(pid);
        if (processInfo) {
            processInfo.process.kill(signal);
            this.processes.delete(pid);
            return true;
        }
        return false;
    }
    
    // Kill all processes
    killAll(signal = 'SIGTERM') {
        for (const [pid, processInfo] of this.processes) {
            processInfo.process.kill(signal);
        }
        this.processes.clear();
    }
    
    // Get process statistics
    getProcessStats() {
        const stats = [];
        
        for (const [pid, processInfo] of this.processes) {
            stats.push({
                pid,
                command: processInfo.command || processInfo.script,
                args: processInfo.args,
                uptime: Date.now() - processInfo.startTime,
                type: processInfo.type || 'spawn'
            });
        }
        
        return stats;
    }
}

// Example usage
async function processExamples() {
    const processManager = new ProcessManager();
    
    try {
        console.log('=== Spawn Example ===');
        
        // List directory contents
        const lsResult = await processManager.spawnProcess('ls', ['-la']);
        console.log('Directory listing:', lsResult.stdout);
        
        console.log('\n=== Exec Example ===');
        
        // Execute shell command
        const dateResult = await processManager.execCommand('date');
        console.log('Current date:', dateResult.stdout.trim());
        
        // Git information
        const gitResult = await processManager.execCommand('git log --oneline -5');
        console.log('Recent commits:', gitResult.stdout);
        
        console.log('\n=== System Information ===');
        
        // System information commands
        const commands = [
            'uname -a',
            'df -h',
            'free -m',
            'ps aux | head -10'
        ];
        
        for (const command of commands) {
            try {
                const result = await processManager.executeSystemCommand(command);
                console.log(`\n${command}:`);
                console.log(result.stdout);
            } catch (error) {
                console.log(`Command failed: ${command}`);
            }
        }
        
    } catch (error) {
        console.error('Process example error:', error);
    } finally {
        processManager.killAll();
    }
}

// Child process communication example
// worker-script.js
const { parentPort } = require('worker_threads');

if (parentPort) {
    parentPort.on('message', async (data) => {
        try {
            const result = await processLargeDataset(data);
            parentPort.postMessage({ success: true, result });
        } catch (error) {
            parentPort.postMessage({ success: false, error: error.message });
        }
    });
}

async function processLargeDataset(data) {
    // Simulate CPU-intensive work
    const { numbers, operation } = data;
    
    switch (operation) {
        case 'sum':
            return numbers.reduce((sum, num) => sum + num, 0);
            
        case 'average':
            return numbers.reduce((sum, num) => sum + num, 0) / numbers.length;
            
        case 'sort':
            return numbers.sort((a, b) => a - b);
            
        case 'prime-count':
            return numbers.filter(isPrime).length;
            
        default:
            throw new Error('Unknown operation');
    }
}

function isPrime(n) {
    if (n <= 1) return false;
    if (n <= 3) return true;
    if (n % 2 === 0 || n % 3 === 0) return false;
    
    for (let i = 5; i * i <= n; i += 6) {
        if (n % i === 0 || n % (i + 2) === 0) {
            return false;
        }
    }
    
    return true;
}

// Usage in main application
async function demonstrateWorkerThreads() {
    const pool = new WorkerPool('./worker-script.js', 4);
    
    const largeDataset = Array.from({ length: 100000 }, () => 
        Math.floor(Math.random() * 1000000)
    );
    
    console.time('Worker pool processing');
    
    const tasks = [
        { numbers: largeDataset.slice(0, 25000), operation: 'sum' },
        { numbers: largeDataset.slice(25000, 50000), operation: 'average' },
        { numbers: largeDataset.slice(50000, 75000), operation: 'sort' },
        { numbers: largeDataset.slice(75000), operation: 'prime-count' }
    ];
    
    const results = await Promise.all(
        tasks.map(task => pool.execute(task))
    );
    
    console.timeEnd('Worker pool processing');
    console.log('Results:', results);
    
    await pool.terminate();
}

if (isMainThread) {
    processExamples();
    demonstrateWorkerThreads();
}
```

## Performance Optimization

### Profiling and Monitoring

```javascript
const v8 = require('v8');
const fs = require('fs');

class PerformanceProfiler {
    constructor() {
        this.metrics = {
            requests: 0,
            errors: 0,
            totalResponseTime: 0,
            slowRequests: 0,
            memoryLeaks: []
        };
        
        this.slowRequestThreshold = 1000; // 1 second
        this.startTime = Date.now();
    }
    
    // Middleware for performance monitoring
    performanceMiddleware() {
        return (req, res, next) => {
            const startTime = process.hrtime.bigint();
            
            res.on('finish', () => {
                const endTime = process.hrtime.bigint();
                const responseTime = Number(endTime - startTime) / 1000000; // Convert to milliseconds
                
                this.metrics.requests++;
                this.metrics.totalResponseTime += responseTime;
                
                if (responseTime > this.slowRequestThreshold) {
                    this.metrics.slowRequests++;
                    console.warn(`🐌 Slow request: ${req.method} ${req.url} - ${responseTime.toFixed(2)}ms`);
                }
                
                if (res.statusCode >= 400) {
                    this.metrics.errors++;
                }
                
                // Log request details
                console.log(`📊 ${req.method} ${req.url} - ${res.statusCode} - ${responseTime.toFixed(2)}ms`);
            });
            
            next();
        };
    }
    
    // CPU profiling
    startCPUProfiling(duration = 30000) {
        console.log('🔍 Starting CPU profiling...');
        
        v8.getHeapSnapshot().pipe(fs.createWriteStream('heap-snapshot-before.heapsnapshot'));
        
        setTimeout(() => {
            v8.getHeapSnapshot().pipe(fs.createWriteStream('heap-snapshot-after.heapsnapshot'));
            console.log('✅ CPU profiling completed');
        }, duration);
    }
    
    // Memory monitoring
    startMemoryMonitoring(interval = 5000) {
        setInterval(() => {
            const memUsage = process.memoryUsage();
            const heapUsed = memUsage.heapUsed / 1024 / 1024;
            
            console.log(`🧠 Memory usage: ${heapUsed.toFixed(2)} MB`);
            
            // Detect potential memory leaks
            if (heapUsed > 100) { // 100MB threshold
                this.metrics.memoryLeaks.push({
                    timestamp: new Date(),
                    heapUsed: heapUsed,
                    rss: memUsage.rss / 1024 / 1024
                });
                
                console.warn(`⚠️ High memory usage detected: ${heapUsed.toFixed(2)} MB`);
            }
        }, interval);
    }
    
    // Event loop lag monitoring
    startEventLoopMonitoring(interval = 1000) {
        setInterval(() => {
            const start = process.hrtime.bigint();
            
            setImmediate(() => {
                const lag = Number(process.hrtime.bigint() - start) / 1000000;
                console.log(`⚡ Event loop lag: ${lag.toFixed(2)}ms`);
                
                if (lag > 100) {
                    console.warn(`🐌 High event loop lag detected: ${lag.toFixed(2)}ms`);
                }
            });
        }, interval);
    }
    
    // Generate performance report
    getPerformanceReport() {
        const uptime = Date.now() - this.startTime;
        const avgResponseTime = this.metrics.totalResponseTime / this.metrics.requests;
        const errorRate = (this.metrics.errors / this.metrics.requests) * 100;
        const slowRequestRate = (this.metrics.slowRequests / this.metrics.requests) * 100;
        
        return {
            uptime: uptime,
            requests: {
                total: this.metrics.requests,
                errors: this.metrics.errors,
                errorRate: errorRate.toFixed(2) + '%',
                slowRequests: this.metrics.slowRequests,
                slowRequestRate: slowRequestRate.toFixed(2) + '%'
            },
            performance: {
                averageResponseTime: avgResponseTime.toFixed(2) + 'ms',
                requestsPerSecond: (this.metrics.requests / (uptime / 1000)).toFixed(2),
                memoryLeaks: this.metrics.memoryLeaks.length
            },
            memory: process.memoryUsage(),
            eventLoop: {
                activeHandles: process._getActiveHandles().length,
                activeRequests: process._getActiveRequests().length
            }
        };
    }
    
    // Heap snapshot
    takeHeapSnapshot(filename) {
        const snapshotPath = filename || `heap-${Date.now()}.heapsnapshot`;
        v8.getHeapSnapshot().pipe(fs.createWriteStream(snapshotPath));
        console.log(`📸 Heap snapshot saved to ${snapshotPath}`);
        return snapshotPath;
    }
    
    // Force garbage collection (requires --expose-gc flag)
    forceGC() {
        if (global.gc) {
            global.gc();
            console.log('🗑️ Garbage collection forced');
        } else {
            console.warn('⚠️ Garbage collection not available. Run with --expose-gc flag');
        }
    }
}

// Usage
const profiler = new PerformanceProfiler();

app.use(profiler.performanceMiddleware());

// Start monitoring
profiler.startMemoryMonitoring();
profiler.startEventLoopMonitoring();

// Performance endpoints
app.get('/performance/report', (req, res) => {
    res.json(profiler.getPerformanceReport());
});

app.post('/performance/heap-snapshot', (req, res) => {
    const filename = profiler.takeHeapSnapshot();
    res.json({ message: 'Heap snapshot taken', filename });
});

app.post('/performance/gc', (req, res) => {
    profiler.forceGC();
    res.json({ message: 'Garbage collection triggered' });
});
```

### Optimization Techniques

```javascript
// Object pooling for frequently created objects
class ObjectPool {
    constructor(createFn, resetFn, maxSize = 100) {
        this.createFn = createFn;
        this.resetFn = resetFn;
        this.maxSize = maxSize;
        this.pool = [];
        this.created = 0;
        this.acquired = 0;
        this.released = 0;
    }
    
    acquire() {
        let obj;
        
        if (this.pool.length > 0) {
            obj = this.pool.pop();
        } else {
            obj = this.createFn();
            this.created++;
        }
        
        this.acquired++;
        return obj;
    }
    
    release(obj) {
        if (this.pool.length < this.maxSize) {
            this.resetFn(obj);
            this.pool.push(obj);
            this.released++;
        }
    }
    
    getStats() {
        return {
            poolSize: this.pool.length,
            created: this.created,
            acquired: this.acquired,
            released: this.released,
            inUse: this.acquired - this.released
        };
    }
}

// Example: Response object pool
const responsePool = new ObjectPool(
    () => ({ data: null, status: 200, headers: {} }),
    (obj) => {
        obj.data = null;
        obj.status = 200;
        obj.headers = {};
    }
);

// Memoization for expensive operations
class MemoCache {
    constructor(maxSize = 1000, ttl = 3600000) {
        this.cache = new Map();
        this.maxSize = maxSize;
        this.ttl = ttl;
        this.hits = 0;
        this.misses = 0;
    }
    
    memoize(fn) {
        return (...args) => {
            const key = JSON.stringify(args);
            const cached = this.cache.get(key);
            
            if (cached && Date.now() - cached.timestamp < this.ttl) {
                this.hits++;
                return cached.value;
            }
            
            this.misses++;
            const result = fn(...args);
            
            // Manage cache size
            if (this.cache.size >= this.maxSize) {
                const firstKey = this.cache.keys().next().value;
                this.cache.delete(firstKey);
            }
            
            this.cache.set(key, {
                value: result,
                timestamp: Date.now()
            });
            
            return result;
        };
    }
    
    clear() {
        this.cache.clear();
    }
    
    getStats() {
        return {
            size: this.cache.size,
            hits: this.hits,
            misses: this.misses,
            hitRate: this.hits / (this.hits + this.misses) * 100
        };
    }
}

// Example: Memoized expensive calculation
const memoCache = new MemoCache();

const expensiveCalculation = memoCache.memoize((n) => {
    console.log(`Computing fibonacci(${n})`);
    if (n <= 1) return n;
    return expensiveCalculation(n - 1) + expensiveCalculation(n - 2);
});

// Performance optimization middleware
function optimizationMiddleware() {
    return (req, res, next) => {
        // Enable gzip compression for responses
        if (req.headers['accept-encoding']?.includes('gzip')) {
            res.setHeader('Content-Encoding', 'gzip');
        }
        
        // Set cache headers for static content
        if (req.url.match(/\.(css|js|png|jpg|gif|ico|svg)$/)) {
            res.setHeader('Cache-Control', 'public, max-age=31536000'); // 1 year
        }
        
        // Prevent memory leaks in long-running requests
        req.on('close', () => {
            // Cleanup request-specific resources
            if (req.cleanup) {
                req.cleanup();
            }
        });
        
        next();
    };
}
```

## Security Best Practices

### Authentication and Authorization

```javascript
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const rateLimit = require('express-rate-limit');
const helmet = require('helmet');

class SecurityManager {
    constructor(options = {}) {
        this.jwtSecret = options.jwtSecret || process.env.JWT_SECRET;
        this.jwtExpiry = options.jwtExpiry || '24h';
        this.saltRounds = options.saltRounds || 12;
        this.maxLoginAttempts = options.maxLoginAttempts || 5;
        this.lockoutDuration = options.lockoutDuration || 15 * 60 * 1000; // 15 minutes
    }
    
    // Password hashing
    async hashPassword(password) {
        try {
            const salt = await bcrypt.genSalt(this.saltRounds);
            return await bcrypt.hash(password, salt);
        } catch (error) {
            throw new Error('Password hashing failed');
        }
    }
    
    async comparePassword(password, hashedPassword) {
        try {
            return await bcrypt.compare(password, hashedPassword);
        } catch (error) {
            throw new Error('Password comparison failed');
        }
    }
    
    // JWT token management
    generateToken(payload) {
        return jwt.sign(payload, this.jwtSecret, {
            expiresIn: this.jwtExpiry,
            issuer: 'myapp',
            audience: 'myapp-users'
        });
    }
    
    verifyToken(token) {
        try {
            return jwt.verify(token, this.jwtSecret, {
                issuer: 'myapp',
                audience: 'myapp-users'
            });
        } catch (error) {
            throw new Error('Invalid token');
        }
    }
    
    generateRefreshToken() {
        const crypto = require('crypto');
        return crypto.randomBytes(32).toString('hex');
    }
    
    // Rate limiting configurations
    createRateLimiter(options = {}) {
        return rateLimit({
            windowMs: options.windowMs || 15 * 60 * 1000, // 15 minutes
            max: options.max || 100, // requests per window
            message: options.message || 'Too many requests, please try again later',
            standardHeaders: true,
            legacyHeaders: false,
            handler: (req, res) => {
                res.status(429).json({
                    error: 'Rate limit exceeded',
                    retryAfter: Math.round(options.windowMs / 1000)
                });
            }
        });
    }
    
    // Security middleware
    securityMiddleware() {
        return [
            // Helmet for security headers
            helmet({
                contentSecurityPolicy: {
                    directives: {
                        defaultSrc: ["'self'"],
                        styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
                        fontSrc: ["'self'", "https://fonts.gstatic.com"],
                        scriptSrc: ["'self'"],
                        imgSrc: ["'self'", "data:", "https:"],
                        connectSrc: ["'self'"],
                        frameSrc: ["'none'"],
                        objectSrc: ["'none'"],
                        upgradeInsecureRequests: []
                    }
                },
                hsts: {
                    maxAge: 31536000,
                    includeSubDomains: true,
                    preload: true
                }
            }),
            
            // Request sanitization
            (req, res, next) => {
                // Sanitize query parameters
                for (const key in req.query) {
                    if (typeof req.query[key] === 'string') {
                        req.query[key] = this.sanitizeInput(req.query[key]);
                    }
                }
                
                // Sanitize request body
                if (req.body && typeof req.body === 'object') {
                    req.body = this.sanitizeObject(req.body);
                }
                
                next();
            }
        ];
    }
    
    sanitizeInput(input) {
        return input
            .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
            .replace(/javascript:/gi, '')
            .replace(/on\w+\s*=/gi, '');
    }
    
    sanitizeObject(obj) {
        const sanitized = {};
        
        for (const [key, value] of Object.entries(obj)) {
            if (typeof value === 'string') {
                sanitized[key] = this.sanitizeInput(value);
            } else if (typeof value === 'object' && value !== null) {
                sanitized[key] = this.sanitizeObject(value);
            } else {
                sanitized[key] = value;
            }
        }
        
        return sanitized;
    }
    
    // Authentication middleware
    authenticate() {
        return (req, res, next) => {
            const authHeader = req.headers.authorization;
            
            if (!authHeader || !authHeader.startsWith('Bearer ')) {
                return res.status(401).json({ error: 'No token provided' });
            }
            
            const token = authHeader.substring(7);
            
            try {
                const decoded = this.verifyToken(token);
                req.user = decoded;
                next();
            } catch (error) {
                res.status(401).json({ error: 'Invalid token' });
            }
        };
    }
    
    // Authorization middleware
    authorize(roles = [], permissions = []) {
        return (req, res, next) => {
            if (!req.user) {
                return res.status(401).json({ error: 'Not authenticated' });
            }
            
            // Check roles
            if (roles.length > 0 && !roles.includes(req.user.role)) {
                return res.status(403).json({ error: 'Insufficient role' });
            }
            
            // Check permissions
            if (permissions.length > 0) {
                const userPermissions = req.user.permissions || [];
                const hasPermission = permissions.some(permission => 
                    userPermissions.includes(permission)
                );
                
                if (!hasPermission) {
                    return res.status(403).json({ error: 'Insufficient permissions' });
                }
            }
            
            next();
        };
    }
    
    // CSRF protection
    csrfProtection() {
        const crypto = require('crypto');
        const tokens = new Map();
        
        return (req, res, next) => {
            if (req.method === 'GET') {
                // Generate CSRF token for GET requests
                const token = crypto.randomBytes(32).toString('hex');
                tokens.set(req.sessionID || req.ip, token);
                res.locals.csrfToken = token;
                next();
            } else {
                // Validate CSRF token for other methods
                const sessionToken = tokens.get(req.sessionID || req.ip);
                const requestToken = req.headers['x-csrf-token'] || req.body._csrf;
                
                if (!sessionToken || sessionToken !== requestToken) {
                    return res.status(403).json({ error: 'Invalid CSRF token' });
                }
                
                next();
            }
        };
    }
}

// Usage
const security = new SecurityManager({
    jwtSecret: process.env.JWT_SECRET,
    jwtExpiry: '24h',
    saltRounds: 12
});

// Apply security middleware
app.use(security.securityMiddleware());

// Rate limiting
app.use('/api/', security.createRateLimiter({
    windowMs: 15 * 60 * 1000,
    max: 100
}));

app.use('/api/auth/', security.createRateLimiter({
    windowMs: 15 * 60 * 1000,
    max: 5 // Stricter limit for auth endpoints
}));

// Authentication routes
app.post('/api/auth/login', async (req, res) => {
    try {
        const { email, password } = req.body;
        
        // Find user (pseudo-code)
        const user = await User.findByEmail(email);
        if (!user) {
            return res.status(401).json({ error: 'Invalid credentials' });
        }
        
        // Check password
        const isValidPassword = await security.comparePassword(password, user.password);
        if (!isValidPassword) {
            return res.status(401).json({ error: 'Invalid credentials' });
        }
        
        // Generate tokens
        const accessToken = security.generateToken({
            userId: user.id,
            email: user.email,
            role: user.role
        });
        
        const refreshToken = security.generateRefreshToken();
        
        // Save refresh token (pseudo-code)
        await user.update({ refreshToken });
        
        res.json({
            accessToken,
            refreshToken,
            user: {
                id: user.id,
                email: user.email,
                role: user.role
            }
        });
        
    } catch (error) {
        res.status(500).json({ error: 'Login failed' });
    }
});

// Protected routes
app.get('/api/profile', 
    security.authenticate(), 
    (req, res) => {
        res.json({ user: req.user });
    }
);

app.get('/api/admin', 
    security.authenticate(),
    security.authorize(['admin']),
    (req, res) => {
        res.json({ message: 'Admin access granted' });
    }
);
```

### Input Validation and Sanitization

```javascript
const joi = require('joi');
const validator = require('validator');

class ValidationManager {
    constructor() {
        this.schemas = new Map();
    }
    
    // Define validation schemas
    defineSchema(name, schema) {
        this.schemas.set(name, schema);
    }
    
    // Validation middleware
    validate(schemaName) {
        return (req, res, next) => {
            const schema = this.schemas.get(schemaName);
            
            if (!schema) {
                return res.status(500).json({ error: 'Validation schema not found' });
            }
            
            const { error, value } = schema.validate(req.body, {
                abortEarly: false,
                stripUnknown: true
            });
            
            if (error) {
                return res.status(400).json({
                    error: 'Validation failed',
                    details: error.details.map(detail => ({
                        field: detail.path.join('.'),
                        message: detail.message
                    }))
                });
            }
            
            req.body = value;
            next();
        };
    }
    
    // Custom sanitization
    sanitizeInput(input, options = {}) {
        if (typeof input !== 'string') return input;
        
        let sanitized = input;
        
        // Remove HTML tags
        if (options.stripTags !== false) {
            sanitized = sanitized.replace(/<[^>]*>/g, '');
        }
        
        // Escape HTML entities
        if (options.escapeHtml !== false) {
            sanitized = validator.escape(sanitized);
        }
        
        // Trim whitespace
        if (options.trim !== false) {
            sanitized = sanitized.trim();
        }
        
        // Remove SQL injection patterns
        if (options.preventSQLInjection !== false) {
            sanitized = sanitized.replace(/('|(\\')|(;)|(\\;)|(\|)|(\\|)|(\*)|(\\\*))/g, '');
        }
        
        return sanitized;
    }
}

// Define validation schemas
const validationManager = new ValidationManager();

validationManager.defineSchema('user', joi.object({
    username: joi.string().alphanum().min(3).max(30).required(),
    email: joi.string().email().required(),
    password: joi.string().min(8).pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/).required(),
    firstName: joi.string().min(2).max(50).required(),
    lastName: joi.string().min(2).max(50).required(),
    age: joi.number().integer().min(13).max(120),
    phone: joi.string().pattern(/^\+?[1-9]\d{1,14}$/),
    address: joi.object({
        street: joi.string().max(100),
        city: joi.string().max(50),
        state: joi.string().max(50),
        zipCode: joi.string().max(10),
        country: joi.string().max(50)
    })
}));

validationManager.defineSchema('post', joi.object({
    title: joi.string().min(5).max(200).required(),
    content: joi.string().min(10).required(),
    category: joi.string().required(),
    tags: joi.array().items(joi.string().max(20)).max(10),
    status: joi.string().valid('draft', 'published', 'archived').default('draft'),
    featuredImage: joi.string().uri(),
    publishedAt: joi.date().iso()
}));

// Usage
app.post('/api/users', 
    validationManager.validate('user'),
    async (req, res) => {
        try {
            const user = await User.create(req.body);
            res.status(201).json(user);
        } catch (error) {
            res.status(400).json({ error: error.message });
        }
    }
);
```

## Testing Strategies

### Unit Testing with Jest

```javascript
// tests/unit/user.service.test.js
const UserService = require('../../src/services/UserService');
const User = require('../../src/models/User');

// Mock the User model
jest.mock('../../src/models/User');

describe('UserService', () => {
    let userService;
    
    beforeEach(() => {
        userService = new UserService();
        jest.clearAllMocks();
    });
    
    describe('createUser', () => {
        it('should create a user successfully', async () => {
            const userData = {
                username: 'testuser',
                email: 'test@example.com',
                password: 'password123'
            };
            
            const mockUser = { id: 1, ...userData };
            User.create.mockResolvedValue(mockUser);
            
            const result = await userService.createUser(userData);
            
            expect(User.create).toHaveBeenCalledWith(userData);
            expect(result).toEqual(mockUser);
        });
        
        it('should handle duplicate email error', async () => {
            const userData = {
                username: 'testuser',
                email: 'existing@example.com',
                password: 'password123'
            };
            
            const error = new Error('Email already exists');
            error.code = 11000;
            User.create.mockRejectedValue(error);
            
            await expect(userService.createUser(userData))
                .rejects.toThrow('Email already exists');
        });
    });
    
    describe('getUserById', () => {
        it('should return user when found', async () => {
            const userId = '123';
            const mockUser = { id: userId, username: 'testuser' };
            
            User.findById.mockResolvedValue(mockUser);
            
            const result = await userService.getUserById(userId);
            
            expect(User.findById).toHaveBeenCalledWith(userId);
            expect(result).toEqual(mockUser);
        });
        
        it('should return null when user not found', async () => {
            const userId = '999';
            User.findById.mockResolvedValue(null);
            
            const result = await userService.getUserById(userId);
            
            expect(result).toBeNull();
        });
    });
});
```

### Integration Testing

```javascript
// tests/integration/user.routes.test.js
const request = require('supertest');
const app = require('../../src/app');
const { setupTestDB, teardownTestDB } = require('../helpers/db');

describe('User Routes', () => {
    let authToken;
    let testUser;
    
    beforeAll(async () => {
        await setupTestDB();
    });
    
    afterAll(async () => {
        await teardownTestDB();
    });
    
    beforeEach(async () => {
        // Create test user and get auth token
        const userData = {
            username: 'testuser',
            email: 'test@example.com',
            password: 'password123',
            firstName: 'Test',
            lastName: 'User'
        };
        
        const response = await request(app)
            .post('/api/auth/register')
            .send(userData);
        
        testUser = response.body.user;
        authToken = response.body.token;
    });
    
    describe('GET /api/users', () => {
        it('should return list of users', async () => {
            const response = await request(app)
                .get('/api/users')
                .set('Authorization', `Bearer ${authToken}`)
                .expect(200);
            
            expect(response.body).toHaveProperty('users');
            expect(Array.isArray(response.body.users)).toBe(true);
            expect(response.body).toHaveProperty('pagination');
        });
        
        it('should require authentication', async () => {
            await request(app)
                .get('/api/users')
                .expect(401);
        });
        
        it('should support pagination', async () => {
            const response = await request(app)
                .get('/api/users?page=1&limit=5')
                .set('Authorization', `Bearer ${authToken}`)
                .expect(200);
            
            expect(response.body.pagination.page).toBe(1);
            expect(response.body.pagination.limit).toBe(5);
        });
    });
    
    describe('POST /api/users', () => {
        it('should create a new user', async () => {
            const newUser = {
                username: 'newuser',
                email: 'new@example.com',
                password: 'password123',
                firstName: 'New',
                lastName: 'User'
            };
            
            const response = await request(app)
                .post('/api/users')
                .set('Authorization', `Bearer ${authToken}`)
                .send(newUser)
                .expect(201);
            
            expect(response.body).toHaveProperty('id');
            expect(response.body.username).toBe(newUser.username);
            expect(response.body.email).toBe(newUser.email);
            expect(response.body).not.toHaveProperty('password');
        });
        
        it('should validate required fields', async () => {
            const invalidUser = {
                username: 'test'
                // Missing required fields
            };
            
            const response = await request(app)
                .post('/api/users')
                .set('Authorization', `Bearer ${authToken}`)
                .send(invalidUser)
                .expect(400);
            
            expect(response.body).toHaveProperty('error');
            expect(response.body).toHaveProperty('details');
        });
    });
    
    describe('PUT /api/users/:id', () => {
        it('should update user', async () => {
            const updateData = {
                firstName: 'Updated',
                lastName: 'Name'
            };
            
            const response = await request(app)
                .put(`/api/users/${testUser.id}`)
                .set('Authorization', `Bearer ${authToken}`)
                .send(updateData)
                .expect(200);
            
            expect(response.body.firstName).toBe(updateData.firstName);
            expect(response.body.lastName).toBe(updateData.lastName);
        });
        
        it('should not allow updating another user', async () => {
            // Create another user
            const anotherUser = await User.create({
                username: 'anotheruser',
                email: 'another@example.com',
                password: 'password123'
            });
            
            await request(app)
                .put(`/api/users/${anotherUser.id}`)
                .set('Authorization', `Bearer ${authToken}`)
                .send({ firstName: 'Hacked' })
                .expect(403);
        });
    });
});
```

### Testing Utilities

```javascript
// tests/helpers/db.js
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');

let mongod;

async function setupTestDB() {
    try {
        mongod = await MongoMemoryServer.create();
        const uri = mongod.getUri();
        
        await mongoose.connect(uri, {
            useNewUrlParser: true,
            useUnifiedTopology: true
        });
        
        console.log('✅ Test database connected');
    } catch (error) {
        console.error('❌ Test database setup failed:', error);
        throw error;
    }
}

async function teardownTestDB() {
    try {
        await mongoose.connection.dropDatabase();
        await mongoose.connection.close();
        await mongod.stop();
        
        console.log('✅ Test database cleaned up');
    } catch (error) {
        console.error('❌ Test database cleanup failed:', error);
        throw error;
    }
}

async function clearTestDB() {
    const collections = mongoose.connection.collections;
    
    for (const key in collections) {
        const collection = collections[key];
        await collection.deleteMany({});
    }
}

module.exports = {
    setupTestDB,
    teardownTestDB,
    clearTestDB
};

// tests/helpers/factories.js
const faker = require('faker');

class TestDataFactory {
    static createUser(overrides = {}) {
        return {
            username: faker.internet.userName(),
            email: faker.internet.email(),
            password: 'password123',
            firstName: faker.name.firstName(),
            lastName: faker.name.lastName(),
            role: 'user',
            ...overrides
        };
    }
    
    static createPost(authorId, overrides = {}) {
        return {
            title: faker.lorem.sentence(),
            content: faker.lorem.paragraphs(3),
            author: authorId,
            status: 'published',
            tags: faker.lorem.words(3).split(' '),
            ...overrides
        };
    }
    
    static async createUserInDB(overrides = {}) {
        const userData = this.createUser(overrides);
        return await User.create(userData);
    }
    
    static async createPostInDB(authorId, overrides = {}) {
        const postData = this.createPost(authorId, overrides);
        return await Post.create(postData);
    }
    
    static async createTestDataSet() {
        // Create test users
        const users = await Promise.all([
            this.createUserInDB({ role: 'admin' }),
            this.createUserInDB({ role: 'moderator' }),
            ...Array.from({ length: 10 }, () => this.createUserInDB())
        ]);
        
        // Create test posts
        const posts = [];
        for (const user of users.slice(0, 5)) {
            const userPosts = await Promise.all([
                this.createPostInDB(user._id),
                this.createPostInDB(user._id, { status: 'draft' }),
                this.createPostInDB(user._id, { status: 'archived' })
            ]);
            posts.push(...userPosts);
        }
        
        return { users, posts };
    }
}

module.exports = TestDataFactory;
```

### Load Testing

```javascript
// tests/load/load-test.js
const autocannon = require('autocannon');
const app = require('../../src/app');

class LoadTester {
    constructor() {
        this.server = null;
        this.results = [];
    }
    
    async startServer() {
        return new Promise((resolve) => {
            this.server = app.listen(0, () => {
                const port = this.server.address().port;
                console.log(`🚀 Test server started on port ${port}`);
                resolve(port);
            });
        });
    }
    
    async stopServer() {
        if (this.server) {
            await new Promise((resolve) => {
                this.server.close(resolve);
            });
            console.log('🛑 Test server stopped');
        }
    }
    
    async runLoadTest(options = {}) {
        const defaultOptions = {
            duration: 30, // 30 seconds
            connections: 10,
            pipelining: 1,
            headers: {
                'content-type': 'application/json'
            }
        };
        
        const testOptions = { ...defaultOptions, ...options };
        
        console.log(`🔥 Starting load test: ${JSON.stringify(testOptions)}`);
        
        const result = await autocannon(testOptions);
        
        this.results.push({
            timestamp: new Date(),
            options: testOptions,
            result: {
                requests: result.requests,
                latency: result.latency,
                throughput: result.throughput,
                errors: result.errors,
                timeouts: result.timeouts
            }
        });
        
        return result;
    }
    
    async runComprehensiveTest(baseUrl) {
        const tests = [
            {
                name: 'GET /health',
                url: `${baseUrl}/health`,
                method: 'GET'
            },
            {
                name: 'GET /api/users',
                url: `${baseUrl}/api/users`,
                method: 'GET',
                headers: {
                    'Authorization': 'Bearer test-token'
                }
            },
            {
                name: 'POST /api/users',
                url: `${baseUrl}/api/users`,
                method: 'POST',
                headers: {
                    'content-type': 'application/json',
                    'Authorization': 'Bearer test-token'
                },
                body: JSON.stringify({
                    username: 'loadtest',
                    email: 'load@test.com',
                    password: 'password123'
                })
            }
        ];
        
        const results = [];
        
        for (const test of tests) {
            console.log(`\n🧪 Running test: ${test.name}`);
            
            try {
                const result = await this.runLoadTest({
                    url: test.url,
                    method: test.method,
                    headers: test.headers,
                    body: test.body,
                    duration: 10,
                    connections: 5
                });
                
                results.push({
                    name: test.name,
                    success: true,
                    ...result
                });
                
                console.log(`✅ ${test.name} completed`);
                console.log(`   Requests/sec: ${result.requests.average}`);
                console.log(`   Latency avg: ${result.latency.average}ms`);
                
            } catch (error) {
                console.error(`❌ ${test.name} failed:`, error.message);
                results.push({
                    name: test.name,
                    success: false,
                    error: error.message
                });
            }
        }
        
        return results;
    }
    
    generateReport() {
        const report = {
            timestamp: new Date(),
            totalTests: this.results.length,
            summary: {
                avgRequestsPerSec: 0,
                avgLatency: 0,
                totalErrors: 0
            },
            tests: this.results
        };
        
        if (this.results.length > 0) {
            const validResults = this.results.filter(r => r.result);
            
            report.summary.avgRequestsPerSec = validResults.reduce(
                (sum, r) => sum + r.result.requests.average, 0
            ) / validResults.length;
            
            report.summary.avgLatency = validResults.reduce(
                (sum, r) => sum + r.result.latency.average, 0
            ) / validResults.length;
            
            report.summary.totalErrors = validResults.reduce(
                (sum, r) => sum + r.result.errors, 0
            );
        }
        
        return report;
    }
}

// Usage
async function runLoadTests() {
    const loadTester = new LoadTester();
    
    try {
        const port = await loadTester.startServer();
        const baseUrl = `http://localhost:${port}`;
        
        await loadTester.runComprehensiveTest(baseUrl);
        
        const report = loadTester.generateReport();
        console.log('\n📊 Load Test Report:');
        console.log(JSON.stringify(report, null, 2));
        
    } finally {
        await loadTester.stopServer();
    }
}

if (require.main === module) {
    runLoadTests().catch(console.error);
}
```

## Deployment and DevOps

### Docker Configuration

```dockerfile
# Dockerfile
FROM node:18-alpine AS base

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Copy application code
COPY --chown=nodejs:nodejs . .

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# Start application
CMD ["node", "server.js"]

# Multi-stage build for development
FROM base AS development
USER root
RUN npm install
USER nodejs
CMD ["npm", "run", "dev"]

# Production build
FROM base AS production
ENV NODE_ENV=production
CMD ["npm", "start"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: ${NODE_ENV:-development}
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=${NODE_ENV:-development}
      - DATABASE_URL=mongodb://mongo:27017/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - mongo
      - redis
    volumes:
      - .:/app
      - /app/node_modules
    networks:
      - app-network
    restart: unless-stopped
    
  mongo:
    image: mongo:5
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    networks:
      - app-network
    restart: unless-stopped
    
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    restart: unless-stopped
    
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    networks:
      - app-network
    restart: unless-stopped

volumes:
  mongo-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

### CI/CD Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    services:
      mongodb:
        image: mongo:5
        ports:
          - 27017:27017
      redis:
        image: redis:7
        ports:
          - 6379:6379
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linting
      run: npm run lint
    
    - name: Run tests
      run: npm test
      env:
        NODE_ENV: test
        DATABASE_URL: mongodb://localhost:27017/test
        REDIS_URL: redis://localhost:6379
    
    - name: Run integration tests
      run: npm run test:integration
    
    - name: Generate coverage report
      run: npm run test:coverage
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
  
  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run security audit
      run: npm audit --audit-level moderate
    
    - name: Run Snyk security scan
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  
  build:
    needs: [test, security]
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          myapp:latest
          myapp:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Deploy to production
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          docker pull myapp:${{ github.sha }}
          docker stop myapp || true
          docker rm myapp || true
          docker run -d \
            --name myapp \
            --restart unless-stopped \
            -p 3000:3000 \
            -e NODE_ENV=production \
            -e DATABASE_URL=${{ secrets.DATABASE_URL }} \
            -e JWT_SECRET=${{ secrets.JWT_SECRET }} \
            myapp:${{ github.sha }}
```

### Production Deployment with PM2

```javascript
// ecosystem.config.js
module.exports = {
    apps: [{
        name: 'myapp',
        script: './server.js',
        instances: 'max', // Use all CPU cores
        exec_mode: 'cluster',
        env: {
            NODE_ENV: 'development',
            PORT: 3000
        },
        env_production: {
            NODE_ENV: 'production',
            PORT: 3000
        },
        error_file: './logs/err.log',
        out_file: './logs/out.log',
        log_file: './logs/combined.log',
        time: true,
        max_memory_restart: '1G',
        node_args: '--max-old-space-size=1024',
        watch: false,
        ignore_watch: ['node_modules', 'logs'],
        max_restarts: 10,
        min_uptime: '10s',
        kill_timeout: 5000,
        wait_ready: true,
        listen_timeout: 10000
    }],
    
    deploy: {
        production: {
            user: 'deploy',
            host: ['server1.example.com', 'server2.example.com'],
            ref: 'origin/main',
            repo: 'git@github.com:username/myapp.git',
            path: '/var/www/myapp',
            'post-deploy': 'npm install && pm2 reload ecosystem.config.js --env production',
            'pre-setup': 'apt update && apt install git -y'
        }
    }
};
```

```bash
# PM2 commands
pm2 start ecosystem.config.js --env production
pm2 reload myapp
pm2 stop myapp
pm2 delete myapp
pm2 logs myapp
pm2 monit
pm2 status

# Deploy
pm2 deploy production setup
pm2 deploy production
```

---

This covers comprehensive advanced Node.js topics including clustering, worker threads, child processes, performance optimization, security, testing strategies, and deployment. The final section would cover monitoring, logging, and production best practices. Would you like me to complete the remaining content?
