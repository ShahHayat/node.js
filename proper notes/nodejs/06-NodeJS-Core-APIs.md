# Node.js Core APIs - Complete Guide

## Table of Contents
1. [HTTP Module](#http-module)
2. [File System (fs)](#file-system-fs)
3. [Streams](#streams)
4. [Events](#events)
5. [URL and Query String](#url-and-query-string)
6. [Crypto](#crypto)
7. [Child Process](#child-process)
8. [Cluster](#cluster)
9. [Utilities](#utilities)
10. [Performance Hooks](#performance-hooks)

## HTTP Module

The HTTP module is one of the most important core modules in Node.js, enabling you to create HTTP servers and clients.

### Basic HTTP Server

```javascript
const http = require('http');
const url = require('url');
const querystring = require('querystring');

// Create a basic HTTP server
const server = http.createServer((req, res) => {
    const parsedUrl = url.parse(req.url, true);
    const path = parsedUrl.pathname;
    const query = parsedUrl.query;
    const method = req.method.toLowerCase();
    
    // Set CORS headers
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type');
    
    // Handle different routes
    if (path === '/' && method === 'get') {
        res.writeHead(200, { 'Content-Type': 'text/html' });
        res.end(`
            <h1>Welcome to Node.js Server</h1>
            <p>Current time: ${new Date().toISOString()}</p>
            <p>Your IP: ${req.connection.remoteAddress}</p>
        `);
    } else if (path === '/api/users' && method === 'get') {
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
            users: [
                { id: 1, name: 'John Doe', email: 'john@example.com' },
                { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
            ],
            query: query
        }));
    } else if (path === '/api/users' && method === 'post') {
        let body = '';
        
        req.on('data', chunk => {
            body += chunk.toString();
        });
        
        req.on('end', () => {
            try {
                const userData = JSON.parse(body);
                res.writeHead(201, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({
                    message: 'User created successfully',
                    user: { id: Date.now(), ...userData }
                }));
            } catch (error) {
                res.writeHead(400, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({ error: 'Invalid JSON' }));
            }
        });
    } else {
        res.writeHead(404, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Not Found' }));
    }
});

// Server configuration
const PORT = process.env.PORT || 3000;
const HOST = process.env.HOST || 'localhost';

server.listen(PORT, HOST, () => {
    console.log(`Server running at http://${HOST}:${PORT}/`);
});

// Handle server errors
server.on('error', (err) => {
    if (err.code === 'EADDRINUSE') {
        console.error(`Port ${PORT} is already in use`);
        process.exit(1);
    } else {
        console.error('Server error:', err);
    }
});

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('SIGTERM received, shutting down gracefully');
    server.close(() => {
        console.log('Server closed');
        process.exit(0);
    });
});
```

### Advanced HTTP Server Features

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');
const zlib = require('zlib');

class HTTPServer {
    constructor(options = {}) {
        this.port = options.port || 3000;
        this.host = options.host || 'localhost';
        this.routes = new Map();
        this.middlewares = [];
        this.staticDir = options.staticDir || 'public';
        
        this.server = http.createServer(this.handleRequest.bind(this));
        this.setupErrorHandling();
    }
    
    // Middleware support
    use(middleware) {
        this.middlewares.push(middleware);
    }
    
    // Route registration
    get(path, handler) {
        this.addRoute('GET', path, handler);
    }
    
    post(path, handler) {
        this.addRoute('POST', path, handler);
    }
    
    put(path, handler) {
        this.addRoute('PUT', path, handler);
    }
    
    delete(path, handler) {
        this.addRoute('DELETE', path, handler);
    }
    
    addRoute(method, path, handler) {
        const key = `${method}:${path}`;
        this.routes.set(key, handler);
    }
    
    async handleRequest(req, res) {
        try {
            // Parse request
            this.parseRequest(req);
            
            // Apply middlewares
            for (const middleware of this.middlewares) {
                await this.runMiddleware(middleware, req, res);
                if (res.headersSent) return;
            }
            
            // Route matching
            const routeKey = `${req.method}:${req.parsedUrl.pathname}`;
            const handler = this.routes.get(routeKey);
            
            if (handler) {
                await handler(req, res);
            } else {
                // Try static file serving
                await this.serveStaticFile(req, res);
            }
            
        } catch (error) {
            this.handleError(error, req, res);
        }
    }
    
    parseRequest(req) {
        const url = require('url');
        req.parsedUrl = url.parse(req.url, true);
        req.query = req.parsedUrl.query;
        req.params = {};
    }
    
    async runMiddleware(middleware, req, res) {
        return new Promise((resolve, reject) => {
            middleware(req, res, (err) => {
                if (err) reject(err);
                else resolve();
            });
        });
    }
    
    async serveStaticFile(req, res) {
        const filePath = path.join(this.staticDir, req.parsedUrl.pathname);
        
        try {
            const stats = await fs.promises.stat(filePath);
            
            if (stats.isFile()) {
                const ext = path.extname(filePath);
                const contentType = this.getContentType(ext);
                
                res.writeHead(200, {
                    'Content-Type': contentType,
                    'Content-Length': stats.size,
                    'Cache-Control': 'public, max-age=3600'
                });
                
                // Stream file with compression
                const readStream = fs.createReadStream(filePath);
                
                if (req.headers['accept-encoding']?.includes('gzip')) {
                    res.setHeader('Content-Encoding', 'gzip');
                    readStream.pipe(zlib.createGzip()).pipe(res);
                } else {
                    readStream.pipe(res);
                }
            } else {
                this.send404(res);
            }
        } catch (error) {
            this.send404(res);
        }
    }
    
    getContentType(ext) {
        const mimeTypes = {
            '.html': 'text/html',
            '.css': 'text/css',
            '.js': 'text/javascript',
            '.json': 'application/json',
            '.png': 'image/png',
            '.jpg': 'image/jpeg',
            '.gif': 'image/gif',
            '.svg': 'image/svg+xml',
            '.ico': 'image/x-icon'
        };
        return mimeTypes[ext] || 'application/octet-stream';
    }
    
    send404(res) {
        res.writeHead(404, { 'Content-Type': 'text/html' });
        res.end('<h1>404 - Not Found</h1>');
    }
    
    handleError(error, req, res) {
        console.error('Server error:', error);
        if (!res.headersSent) {
            res.writeHead(500, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({ error: 'Internal Server Error' }));
        }
    }
    
    setupErrorHandling() {
        this.server.on('error', (err) => {
            console.error('Server error:', err);
        });
    }
    
    listen(callback) {
        this.server.listen(this.port, this.host, callback);
    }
    
    close(callback) {
        this.server.close(callback);
    }
}

// Usage
const app = new HTTPServer({ port: 3000, staticDir: './public' });

// Middleware
app.use((req, res, next) => {
    console.log(`${new Date().toISOString()} - ${req.method} ${req.url}`);
    next();
});

app.use((req, res, next) => {
    res.json = (data) => {
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify(data));
    };
    next();
});

// Routes
app.get('/', (req, res) => {
    res.json({ message: 'Welcome to custom HTTP server!' });
});

app.get('/api/time', (req, res) => {
    res.json({ 
        timestamp: new Date().toISOString(),
        timezone: Intl.DateTimeFormat().resolvedOptions().timeZone
    });
});

app.post('/api/echo', async (req, res) => {
    let body = '';
    for await (const chunk of req) {
        body += chunk;
    }
    
    res.json({
        method: req.method,
        url: req.url,
        headers: req.headers,
        body: body
    });
});

app.listen(() => {
    console.log('Custom server running on http://localhost:3000');
});
```

### HTTP Client

```javascript
const http = require('http');
const https = require('https');
const url = require('url');

class HTTPClient {
    constructor(options = {}) {
        this.timeout = options.timeout || 5000;
        this.defaultHeaders = options.headers || {};
    }
    
    async request(options) {
        return new Promise((resolve, reject) => {
            const parsedUrl = typeof options === 'string' ? url.parse(options) : options;
            const isHttps = parsedUrl.protocol === 'https:';
            const client = isHttps ? https : http;
            
            const requestOptions = {
                hostname: parsedUrl.hostname,
                port: parsedUrl.port || (isHttps ? 443 : 80),
                path: parsedUrl.path,
                method: parsedUrl.method || 'GET',
                headers: {
                    ...this.defaultHeaders,
                    ...parsedUrl.headers
                }
            };
            
            const req = client.request(requestOptions, (res) => {
                let data = '';
                
                res.on('data', chunk => {
                    data += chunk;
                });
                
                res.on('end', () => {
                    const response = {
                        statusCode: res.statusCode,
                        statusMessage: res.statusMessage,
                        headers: res.headers,
                        data: data
                    };
                    
                    // Parse JSON if content-type is application/json
                    if (res.headers['content-type']?.includes('application/json')) {
                        try {
                            response.data = JSON.parse(data);
                        } catch (error) {
                            // Keep as string if JSON parsing fails
                        }
                    }
                    
                    resolve(response);
                });
            });
            
            req.on('error', reject);
            
            // Set timeout
            req.setTimeout(this.timeout, () => {
                req.destroy();
                reject(new Error('Request timeout'));
            });
            
            // Send request body if provided
            if (parsedUrl.body) {
                if (typeof parsedUrl.body === 'object') {
                    req.setHeader('Content-Type', 'application/json');
                    req.write(JSON.stringify(parsedUrl.body));
                } else {
                    req.write(parsedUrl.body);
                }
            }
            
            req.end();
        });
    }
    
    async get(url, headers = {}) {
        return this.request({ ...url, method: 'GET', headers });
    }
    
    async post(url, body, headers = {}) {
        return this.request({ 
            ...url, 
            method: 'POST', 
            body, 
            headers: {
                'Content-Type': 'application/json',
                ...headers
            }
        });
    }
    
    async put(url, body, headers = {}) {
        return this.request({ 
            ...url, 
            method: 'PUT', 
            body, 
            headers: {
                'Content-Type': 'application/json',
                ...headers
            }
        });
    }
    
    async delete(url, headers = {}) {
        return this.request({ ...url, method: 'DELETE', headers });
    }
}

// Usage
const client = new HTTPClient({ timeout: 10000 });

async function makeRequests() {
    try {
        // GET request
        const getResponse = await client.get('https://jsonplaceholder.typicode.com/posts/1');
        console.log('GET Response:', getResponse.data);
        
        // POST request
        const postResponse = await client.post('https://jsonplaceholder.typicode.com/posts', {
            title: 'My New Post',
            body: 'This is the content of my post',
            userId: 1
        });
        console.log('POST Response:', postResponse.data);
        
    } catch (error) {
        console.error('Request failed:', error.message);
    }
}

makeRequests();
```

## File System (fs)

The File System module provides an API for interacting with the file system.

### File Operations

```javascript
const fs = require('fs').promises;
const path = require('path');
const { createReadStream, createWriteStream } = require('fs');

class FileManager {
    constructor(baseDir = './') {
        this.baseDir = path.resolve(baseDir);
    }
    
    // Read file with different encodings
    async readFile(filePath, encoding = 'utf8') {
        try {
            const fullPath = path.join(this.baseDir, filePath);
            const data = await fs.readFile(fullPath, encoding);
            return data;
        } catch (error) {
            throw new Error(`Failed to read file ${filePath}: ${error.message}`);
        }
    }
    
    // Write file with automatic directory creation
    async writeFile(filePath, data, options = {}) {
        try {
            const fullPath = path.join(this.baseDir, filePath);
            const dir = path.dirname(fullPath);
            
            // Create directory if it doesn't exist
            await fs.mkdir(dir, { recursive: true });
            
            await fs.writeFile(fullPath, data, {
                encoding: 'utf8',
                ...options
            });
            
            return fullPath;
        } catch (error) {
            throw new Error(`Failed to write file ${filePath}: ${error.message}`);
        }
    }
    
    // Append to file
    async appendFile(filePath, data) {
        try {
            const fullPath = path.join(this.baseDir, filePath);
            await fs.appendFile(fullPath, data, 'utf8');
            return fullPath;
        } catch (error) {
            throw new Error(`Failed to append to file ${filePath}: ${error.message}`);
        }
    }
    
    // Copy file
    async copyFile(source, destination) {
        try {
            const sourcePath = path.join(this.baseDir, source);
            const destPath = path.join(this.baseDir, destination);
            const destDir = path.dirname(destPath);
            
            await fs.mkdir(destDir, { recursive: true });
            await fs.copyFile(sourcePath, destPath);
            
            return destPath;
        } catch (error) {
            throw new Error(`Failed to copy file from ${source} to ${destination}: ${error.message}`);
        }
    }
    
    // Move/rename file
    async moveFile(source, destination) {
        try {
            const sourcePath = path.join(this.baseDir, source);
            const destPath = path.join(this.baseDir, destination);
            const destDir = path.dirname(destPath);
            
            await fs.mkdir(destDir, { recursive: true });
            await fs.rename(sourcePath, destPath);
            
            return destPath;
        } catch (error) {
            throw new Error(`Failed to move file from ${source} to ${destination}: ${error.message}`);
        }
    }
    
    // Delete file
    async deleteFile(filePath) {
        try {
            const fullPath = path.join(this.baseDir, filePath);
            await fs.unlink(fullPath);
            return true;
        } catch (error) {
            if (error.code === 'ENOENT') {
                return false; // File doesn't exist
            }
            throw new Error(`Failed to delete file ${filePath}: ${error.message}`);
        }
    }
    
    // Check if file exists
    async exists(filePath) {
        try {
            const fullPath = path.join(this.baseDir, filePath);
            await fs.access(fullPath);
            return true;
        } catch {
            return false;
        }
    }
    
    // Get file stats
    async getStats(filePath) {
        try {
            const fullPath = path.join(this.baseDir, filePath);
            const stats = await fs.stat(fullPath);
            
            return {
                size: stats.size,
                isFile: stats.isFile(),
                isDirectory: stats.isDirectory(),
                created: stats.birthtime,
                modified: stats.mtime,
                accessed: stats.atime,
                permissions: stats.mode
            };
        } catch (error) {
            throw new Error(`Failed to get stats for ${filePath}: ${error.message}`);
        }
    }
    
    // Watch file for changes
    watchFile(filePath, callback) {
        const fullPath = path.join(this.baseDir, filePath);
        
        const watcher = fs.watch(fullPath, (eventType, filename) => {
            callback(eventType, filename, fullPath);
        });
        
        return watcher;
    }
    
    // Stream large file read
    createReadStream(filePath, options = {}) {
        const fullPath = path.join(this.baseDir, filePath);
        return createReadStream(fullPath, options);
    }
    
    // Stream large file write
    createWriteStream(filePath, options = {}) {
        const fullPath = path.join(this.baseDir, filePath);
        const dir = path.dirname(fullPath);
        
        // Create directory synchronously for streams
        require('fs').mkdirSync(dir, { recursive: true });
        
        return createWriteStream(fullPath, options);
    }
}

// Usage examples
async function fileOperations() {
    const fileManager = new FileManager('./data');
    
    try {
        // Write a file
        await fileManager.writeFile('test.txt', 'Hello, World!');
        console.log('File written successfully');
        
        // Read the file
        const content = await fileManager.readFile('test.txt');
        console.log('File content:', content);
        
        // Append to file
        await fileManager.appendFile('test.txt', '\nAppended text');
        
        // Get file stats
        const stats = await fileManager.getStats('test.txt');
        console.log('File stats:', stats);
        
        // Copy file
        await fileManager.copyFile('test.txt', 'backup/test-copy.txt');
        
        // Check if file exists
        const exists = await fileManager.exists('backup/test-copy.txt');
        console.log('Backup exists:', exists);
        
        // Watch file for changes
        const watcher = fileManager.watchFile('test.txt', (eventType, filename) => {
            console.log(`File ${filename} changed: ${eventType}`);
        });
        
        // Stop watching after 10 seconds
        setTimeout(() => watcher.close(), 10000);
        
    } catch (error) {
        console.error('File operation failed:', error.message);
    }
}

fileOperations();
```

### Directory Operations

```javascript
const fs = require('fs').promises;
const path = require('path');

class DirectoryManager {
    constructor(baseDir = './') {
        this.baseDir = path.resolve(baseDir);
    }
    
    // Create directory
    async createDirectory(dirPath) {
        try {
            const fullPath = path.join(this.baseDir, dirPath);
            await fs.mkdir(fullPath, { recursive: true });
            return fullPath;
        } catch (error) {
            throw new Error(`Failed to create directory ${dirPath}: ${error.message}`);
        }
    }
    
    // List directory contents
    async listDirectory(dirPath = '.', options = {}) {
        try {
            const fullPath = path.join(this.baseDir, dirPath);
            const entries = await fs.readdir(fullPath, { 
                withFileTypes: true,
                ...options
            });
            
            return entries.map(entry => ({
                name: entry.name,
                isFile: entry.isFile(),
                isDirectory: entry.isDirectory(),
                isSymlink: entry.isSymbolicLink(),
                path: path.join(dirPath, entry.name)
            }));
        } catch (error) {
            throw new Error(`Failed to list directory ${dirPath}: ${error.message}`);
        }
    }
    
    // Remove directory
    async removeDirectory(dirPath) {
        try {
            const fullPath = path.join(this.baseDir, dirPath);
            await fs.rmdir(fullPath, { recursive: true });
            return true;
        } catch (error) {
            if (error.code === 'ENOENT') {
                return false; // Directory doesn't exist
            }
            throw new Error(`Failed to remove directory ${dirPath}: ${error.message}`);
        }
    }
    
    // Copy directory recursively
    async copyDirectory(source, destination) {
        try {
            const sourcePath = path.join(this.baseDir, source);
            const destPath = path.join(this.baseDir, destination);
            
            await this.copyDirectoryRecursive(sourcePath, destPath);
            return destPath;
        } catch (error) {
            throw new Error(`Failed to copy directory from ${source} to ${destination}: ${error.message}`);
        }
    }
    
    async copyDirectoryRecursive(source, destination) {
        const stats = await fs.stat(source);
        
        if (stats.isDirectory()) {
            await fs.mkdir(destination, { recursive: true });
            const entries = await fs.readdir(source);
            
            for (const entry of entries) {
                const sourcePath = path.join(source, entry);
                const destPath = path.join(destination, entry);
                await this.copyDirectoryRecursive(sourcePath, destPath);
            }
        } else {
            await fs.copyFile(source, destination);
        }
    }
    
    // Find files matching pattern
    async findFiles(pattern, dirPath = '.') {
        const results = [];
        const fullPath = path.join(this.baseDir, dirPath);
        
        await this.findFilesRecursive(fullPath, pattern, results, dirPath);
        return results;
    }
    
    async findFilesRecursive(currentPath, pattern, results, relativePath) {
        try {
            const entries = await fs.readdir(currentPath, { withFileTypes: true });
            
            for (const entry of entries) {
                const entryPath = path.join(currentPath, entry.name);
                const relativeEntryPath = path.join(relativePath, entry.name);
                
                if (entry.isDirectory()) {
                    await this.findFilesRecursive(entryPath, pattern, results, relativeEntryPath);
                } else if (entry.isFile()) {
                    if (this.matchesPattern(entry.name, pattern)) {
                        const stats = await fs.stat(entryPath);
                        results.push({
                            name: entry.name,
                            path: relativeEntryPath,
                            size: stats.size,
                            modified: stats.mtime
                        });
                    }
                }
            }
        } catch (error) {
            // Skip directories we can't read
            console.warn(`Cannot read directory ${currentPath}: ${error.message}`);
        }
    }
    
    matchesPattern(filename, pattern) {
        if (typeof pattern === 'string') {
            return filename.includes(pattern);
        } else if (pattern instanceof RegExp) {
            return pattern.test(filename);
        }
        return false;
    }
    
    // Get directory size
    async getDirectorySize(dirPath = '.') {
        let totalSize = 0;
        const fullPath = path.join(this.baseDir, dirPath);
        
        await this.calculateSizeRecursive(fullPath, (size) => {
            totalSize += size;
        });
        
        return totalSize;
    }
    
    async calculateSizeRecursive(currentPath, callback) {
        try {
            const stats = await fs.stat(currentPath);
            
            if (stats.isFile()) {
                callback(stats.size);
            } else if (stats.isDirectory()) {
                const entries = await fs.readdir(currentPath);
                
                for (const entry of entries) {
                    const entryPath = path.join(currentPath, entry);
                    await this.calculateSizeRecursive(entryPath, callback);
                }
            }
        } catch (error) {
            // Skip files/directories we can't access
            console.warn(`Cannot access ${currentPath}: ${error.message}`);
        }
    }
    
    // Clean empty directories
    async cleanEmptyDirectories(dirPath = '.') {
        const fullPath = path.join(this.baseDir, dirPath);
        return await this.cleanEmptyDirectoriesRecursive(fullPath);
    }
    
    async cleanEmptyDirectoriesRecursive(currentPath) {
        try {
            const entries = await fs.readdir(currentPath);
            let isEmpty = true;
            
            for (const entry of entries) {
                const entryPath = path.join(currentPath, entry);
                const stats = await fs.stat(entryPath);
                
                if (stats.isDirectory()) {
                    const wasEmpty = await this.cleanEmptyDirectoriesRecursive(entryPath);
                    if (!wasEmpty) {
                        isEmpty = false;
                    }
                } else {
                    isEmpty = false;
                }
            }
            
            if (isEmpty && currentPath !== this.baseDir) {
                await fs.rmdir(currentPath);
                console.log(`Removed empty directory: ${currentPath}`);
                return true;
            }
            
            return isEmpty;
        } catch (error) {
            console.warn(`Cannot process directory ${currentPath}: ${error.message}`);
            return false;
        }
    }
}

// Usage examples
async function directoryOperations() {
    const dirManager = new DirectoryManager('./workspace');
    
    try {
        // Create directories
        await dirManager.createDirectory('projects/my-app/src');
        await dirManager.createDirectory('projects/my-app/tests');
        console.log('Directories created');
        
        // List directory contents
        const contents = await dirManager.listDirectory('projects');
        console.log('Directory contents:', contents);
        
        // Find JavaScript files
        const jsFiles = await dirManager.findFiles(/\.js$/, 'projects');
        console.log('JavaScript files:', jsFiles);
        
        // Get directory size
        const size = await dirManager.getDirectorySize('projects');
        console.log(`Directory size: ${size} bytes`);
        
        // Copy directory
        await dirManager.copyDirectory('projects/my-app', 'backups/my-app-backup');
        console.log('Directory copied');
        
        // Clean empty directories
        await dirManager.cleanEmptyDirectories();
        console.log('Empty directories cleaned');
        
    } catch (error) {
        console.error('Directory operation failed:', error.message);
    }
}

directoryOperations();
```

## Streams

Streams are a fundamental concept in Node.js for handling data that comes in chunks.

### Stream Types and Usage

```javascript
const { Readable, Writable, Transform, pipeline } = require('stream');
const fs = require('fs');
const zlib = require('zlib');
const crypto = require('crypto');

// Custom Readable Stream
class NumberStream extends Readable {
    constructor(options = {}) {
        super(options);
        this.current = options.start || 1;
        this.max = options.max || 100;
    }
    
    _read() {
        if (this.current <= this.max) {
            this.push(`${this.current}\n`);
            this.current++;
        } else {
            this.push(null); // End of stream
        }
    }
}

// Custom Writable Stream
class LogStream extends Writable {
    constructor(options = {}) {
        super(options);
        this.logFile = options.logFile || 'output.log';
    }
    
    _write(chunk, encoding, callback) {
        const timestamp = new Date().toISOString();
        const logEntry = `[${timestamp}] ${chunk.toString()}`;
        
        fs.appendFile(this.logFile, logEntry, (err) => {
            if (err) {
                callback(err);
            } else {
                console.log(`Logged: ${chunk.toString().trim()}`);
                callback();
            }
        });
    }
}

// Custom Transform Stream
class UpperCaseTransform extends Transform {
    _transform(chunk, encoding, callback) {
        const upperCased = chunk.toString().toUpperCase();
        callback(null, upperCased);
    }
}

class JSONParseTransform extends Transform {
    constructor(options = {}) {
        super({ objectMode: true, ...options });
        this.buffer = '';
    }
    
    _transform(chunk, encoding, callback) {
        this.buffer += chunk.toString();
        const lines = this.buffer.split('\n');
        
        // Keep the last incomplete line in buffer
        this.buffer = lines.pop();
        
        for (const line of lines) {
            if (line.trim()) {
                try {
                    const obj = JSON.parse(line);
                    this.push(obj);
                } catch (error) {
                    this.emit('error', new Error(`Invalid JSON: ${line}`));
                    return;
                }
            }
        }
        
        callback();
    }
    
    _flush(callback) {
        if (this.buffer.trim()) {
            try {
                const obj = JSON.parse(this.buffer);
                this.push(obj);
            } catch (error) {
                this.emit('error', new Error(`Invalid JSON: ${this.buffer}`));
                return;
            }
        }
        callback();
    }
}

// Stream utilities
class StreamUtils {
    // Convert stream to promise
    static streamToPromise(stream) {
        return new Promise((resolve, reject) => {
            const chunks = [];
            
            stream.on('data', chunk => chunks.push(chunk));
            stream.on('end', () => resolve(Buffer.concat(chunks)));
            stream.on('error', reject);
        });
    }
    
    // Convert buffer/string to readable stream
    static bufferToStream(buffer) {
        const readable = new Readable({
            read() {}
        });
        
        readable.push(buffer);
        readable.push(null);
        
        return readable;
    }
    
    // Split stream by delimiter
    static createSplitStream(delimiter = '\n') {
        return new Transform({
            objectMode: true,
            transform(chunk, encoding, callback) {
                const lines = chunk.toString().split(delimiter);
                
                for (const line of lines) {
                    if (line.trim()) {
                        this.push(line);
                    }
                }
                
                callback();
            }
        });
    }
    
    // Batch stream items
    static createBatchStream(batchSize = 10) {
        let batch = [];
        
        return new Transform({
            objectMode: true,
            transform(chunk, encoding, callback) {
                batch.push(chunk);
                
                if (batch.length >= batchSize) {
                    this.push(batch);
                    batch = [];
                }
                
                callback();
            },
            flush(callback) {
                if (batch.length > 0) {
                    this.push(batch);
                }
                callback();
            }
        });
    }
    
    // Rate limit stream
    static createRateLimitStream(itemsPerSecond = 10) {
        const interval = 1000 / itemsPerSecond;
        let lastTime = 0;
        
        return new Transform({
            objectMode: true,
            transform(chunk, encoding, callback) {
                const now = Date.now();
                const timeSinceLastItem = now - lastTime;
                
                if (timeSinceLastItem >= interval) {
                    lastTime = now;
                    this.push(chunk);
                    callback();
                } else {
                    const delay = interval - timeSinceLastItem;
                    setTimeout(() => {
                        lastTime = Date.now();
                        this.push(chunk);
                        callback();
                    }, delay);
                }
            }
        });
    }
}

// Advanced stream processing examples
async function streamExamples() {
    try {
        console.log('=== Basic Stream Example ===');
        
        // Create streams
        const numberStream = new NumberStream({ start: 1, max: 10 });
        const upperCaseTransform = new UpperCaseTransform();
        const logStream = new LogStream({ logFile: 'numbers.log' });
        
        // Pipeline streams
        await new Promise((resolve, reject) => {
            pipeline(
                numberStream,
                upperCaseTransform,
                logStream,
                (err) => {
                    if (err) reject(err);
                    else resolve();
                }
            );
        });
        
        console.log('Basic stream pipeline completed');
        
        console.log('\n=== File Processing Example ===');
        
        // Process large CSV file
        const csvProcessor = new Transform({
            objectMode: true,
            transform(chunk, encoding, callback) {
                const lines = chunk.toString().split('\n');
                
                for (const line of lines) {
                    if (line.trim()) {
                        const columns = line.split(',');
                        const record = {
                            id: columns[0],
                            name: columns[1],
                            email: columns[2],
                            processedAt: new Date().toISOString()
                        };
                        this.push(JSON.stringify(record) + '\n');
                    }
                }
                
                callback();
            }
        });
        
        // Create sample CSV data
        const csvData = `1,John Doe,john@example.com
2,Jane Smith,jane@example.com
3,Bob Johnson,bob@example.com`;
        
        const csvStream = StreamUtils.bufferToStream(csvData);
        const outputStream = fs.createWriteStream('processed.json');
        
        await new Promise((resolve, reject) => {
            pipeline(
                csvStream,
                csvProcessor,
                outputStream,
                (err) => {
                    if (err) reject(err);
                    else resolve();
                }
            );
        });
        
        console.log('CSV processing completed');
        
        console.log('\n=== Compression Example ===');
        
        // Compress and decompress data
        const data = 'This is some data that will be compressed and then decompressed using streams.'.repeat(100);
        const inputStream = StreamUtils.bufferToStream(data);
        const compressStream = zlib.createGzip();
        const decompressStream = zlib.createGunzip();
        
        // Compress
        const compressedData = await new Promise((resolve, reject) => {
            const chunks = [];
            
            pipeline(
                inputStream,
                compressStream,
                new Writable({
                    write(chunk, encoding, callback) {
                        chunks.push(chunk);
                        callback();
                    }
                }),
                (err) => {
                    if (err) reject(err);
                    else resolve(Buffer.concat(chunks));
                }
            );
        });
        
        console.log(`Original size: ${data.length} bytes`);
        console.log(`Compressed size: ${compressedData.length} bytes`);
        console.log(`Compression ratio: ${(compressedData.length / data.length * 100).toFixed(2)}%`);
        
        // Decompress
        const decompressedData = await new Promise((resolve, reject) => {
            const chunks = [];
            
            pipeline(
                StreamUtils.bufferToStream(compressedData),
                decompressStream,
                new Writable({
                    write(chunk, encoding, callback) {
                        chunks.push(chunk);
                        callback();
                    }
                }),
                (err) => {
                    if (err) reject(err);
                    else resolve(Buffer.concat(chunks).toString());
                }
            );
        });
        
        console.log(`Decompressed correctly: ${data === decompressedData}`);
        
    } catch (error) {
        console.error('Stream example failed:', error);
    }
}

streamExamples();
```

### Stream Monitoring and Error Handling

```javascript
const { EventEmitter } = require('events');
const { pipeline, Transform } = require('stream');

class StreamMonitor extends EventEmitter {
    constructor() {
        super();
        this.streams = new Map();
        this.stats = {
            totalBytes: 0,
            totalChunks: 0,
            errors: 0,
            startTime: Date.now()
        };
    }
    
    monitorStream(stream, name) {
        const streamStats = {
            name,
            bytes: 0,
            chunks: 0,
            errors: 0,
            startTime: Date.now()
        };
        
        this.streams.set(stream, streamStats);
        
        stream.on('data', (chunk) => {
            streamStats.bytes += chunk.length;
            streamStats.chunks++;
            this.stats.totalBytes += chunk.length;
            this.stats.totalChunks++;
            
            this.emit('data', { stream: name, chunk: chunk.length });
        });
        
        stream.on('end', () => {
            streamStats.endTime = Date.now();
            streamStats.duration = streamStats.endTime - streamStats.startTime;
            
            this.emit('end', { stream: name, stats: streamStats });
        });
        
        stream.on('error', (error) => {
            streamStats.errors++;
            this.stats.errors++;
            
            this.emit('error', { stream: name, error });
        });
        
        return stream;
    }
    
    getStats() {
        const duration = Date.now() - this.stats.startTime;
        const throughput = this.stats.totalBytes / (duration / 1000); // bytes per second
        
        return {
            ...this.stats,
            duration,
            throughput: Math.round(throughput),
            streams: Array.from(this.streams.values())
        };
    }
}

// Error handling transform stream
class ErrorHandlingTransform extends Transform {
    constructor(options = {}) {
        super(options);
        this.retryAttempts = options.retryAttempts || 3;
        this.retryDelay = options.retryDelay || 1000;
        this.onError = options.onError || ((error, chunk) => {
            console.error('Transform error:', error.message);
        });
    }
    
    async _transform(chunk, encoding, callback) {
        let attempts = 0;
        
        while (attempts <= this.retryAttempts) {
            try {
                const result = await this.processChunk(chunk);
                callback(null, result);
                return;
            } catch (error) {
                attempts++;
                
                if (attempts > this.retryAttempts) {
                    this.onError(error, chunk);
                    callback(); // Skip this chunk
                    return;
                }
                
                // Wait before retry
                await new Promise(resolve => setTimeout(resolve, this.retryDelay));
            }
        }
    }
    
    async processChunk(chunk) {
        // Override this method in subclasses
        return chunk;
    }
}

// Circuit breaker for streams
class CircuitBreakerTransform extends Transform {
    constructor(options = {}) {
        super(options);
        this.failureThreshold = options.failureThreshold || 5;
        this.resetTimeout = options.resetTimeout || 30000;
        this.monitoringWindow = options.monitoringWindow || 60000;
        
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
        this.failures = [];
        this.lastFailureTime = null;
    }
    
    _transform(chunk, encoding, callback) {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime > this.resetTimeout) {
                this.state = 'HALF_OPEN';
                console.log('Circuit breaker: HALF_OPEN');
            } else {
                // Skip processing when circuit is open
                callback();
                return;
            }
        }
        
        this.processWithCircuitBreaker(chunk)
            .then(result => {
                this.onSuccess();
                callback(null, result);
            })
            .catch(error => {
                this.onFailure(error);
                callback(); // Skip chunk on failure
            });
    }
    
    async processWithCircuitBreaker(chunk) {
        // Override this method in subclasses
        return chunk;
    }
    
    onSuccess() {
        if (this.state === 'HALF_OPEN') {
            this.state = 'CLOSED';
            this.failures = [];
            console.log('Circuit breaker: CLOSED');
        }
    }
    
    onFailure(error) {
        const now = Date.now();
        this.failures.push(now);
        this.lastFailureTime = now;
        
        // Remove old failures outside monitoring window
        this.failures = this.failures.filter(
            time => now - time < this.monitoringWindow
        );
        
        if (this.failures.length >= this.failureThreshold) {
            this.state = 'OPEN';
            console.log('Circuit breaker: OPEN');
        }
        
        console.error('Stream processing error:', error.message);
    }
}

// Usage example
async function monitoredStreamProcessing() {
    const monitor = new StreamMonitor();
    
    // Set up monitoring
    monitor.on('data', ({ stream, chunk }) => {
        console.log(`${stream}: processed ${chunk} bytes`);
    });
    
    monitor.on('end', ({ stream, stats }) => {
        console.log(`${stream}: completed in ${stats.duration}ms`);
        console.log(`  Processed: ${stats.chunks} chunks, ${stats.bytes} bytes`);
    });
    
    monitor.on('error', ({ stream, error }) => {
        console.error(`${stream}: error - ${error.message}`);
    });
    
    // Create streams with monitoring
    const inputStream = monitor.monitorStream(
        fs.createReadStream('large-file.txt'),
        'input'
    );
    
    const processingStream = monitor.monitorStream(
        new Transform({
            transform(chunk, encoding, callback) {
                // Simulate processing
                const processed = chunk.toString().toUpperCase();
                callback(null, processed);
            }
        }),
        'processing'
    );
    
    const outputStream = monitor.monitorStream(
        fs.createWriteStream('processed-file.txt'),
        'output'
    );
    
    try {
        await new Promise((resolve, reject) => {
            pipeline(
                inputStream,
                processingStream,
                outputStream,
                (err) => {
                    if (err) reject(err);
                    else resolve();
                }
            );
        });
        
        console.log('\nFinal Stats:', monitor.getStats());
        
    } catch (error) {
        console.error('Pipeline failed:', error);
    }
}

// Run if file exists
if (require('fs').existsSync('large-file.txt')) {
    monitoredStreamProcessing();
} else {
    console.log('Create a file named "large-file.txt" to run the monitoring example');
}
```

## Events

The Events module provides an event emitter pattern for handling asynchronous events.

### EventEmitter Basics

```javascript
const EventEmitter = require('events');

// Basic EventEmitter usage
class MyEmitter extends EventEmitter {}

const myEmitter = new MyEmitter();

// Add listeners
myEmitter.on('event', (data) => {
    console.log('Event received:', data);
});

myEmitter.on('event', (data) => {
    console.log('Second listener:', data);
});

// Emit events
myEmitter.emit('event', { message: 'Hello World' });

// One-time listener
myEmitter.once('special', (data) => {
    console.log('This will only run once:', data);
});

myEmitter.emit('special', 'First time');
myEmitter.emit('special', 'Second time'); // Won't trigger listener

// Remove listeners
const listener = (data) => console.log('Removable listener:', data);
myEmitter.on('removable', listener);
myEmitter.emit('removable', 'Before removal');
myEmitter.removeListener('removable', listener);
myEmitter.emit('removable', 'After removal'); // Won't trigger
```

### Advanced EventEmitter Patterns

```javascript
const EventEmitter = require('events');

// Custom EventEmitter with enhanced features
class EnhancedEventEmitter extends EventEmitter {
    constructor(options = {}) {
        super();
        this.maxListeners = options.maxListeners || 10;
        this.enableLogging = options.enableLogging || false;
        this.eventHistory = [];
        this.listenerStats = new Map();
        
        if (this.enableLogging) {
            this.setupLogging();
        }
    }
    
    setupLogging() {
        const originalEmit = this.emit.bind(this);
        const originalOn = this.on.bind(this);
        const originalOff = this.removeListener.bind(this);
        
        this.emit = function(eventName, ...args) {
            const timestamp = new Date().toISOString();
            this.eventHistory.push({
                event: eventName,
                timestamp,
                args: args.length,
                listeners: this.listenerCount(eventName)
            });
            
            console.log(`[${timestamp}] Emitting: ${eventName}`);
            return originalEmit(eventName, ...args);
        };
        
        this.on = function(eventName, listener) {
            if (!this.listenerStats.has(eventName)) {
                this.listenerStats.set(eventName, { added: 0, removed: 0 });
            }
            this.listenerStats.get(eventName).added++;
            
            console.log(`Added listener for: ${eventName}`);
            return originalOn(eventName, listener);
        };
        
        this.removeListener = function(eventName, listener) {
            if (this.listenerStats.has(eventName)) {
                this.listenerStats.get(eventName).removed++;
            }
            
            console.log(`Removed listener for: ${eventName}`);
            return originalOff(eventName, listener);
        };
    }
    
    // Emit with delay
    emitAsync(eventName, ...args) {
        return new Promise((resolve) => {
            setImmediate(() => {
                const result = this.emit(eventName, ...args);
                resolve(result);
            });
        });
    }
    
    // Emit with timeout
    emitWithTimeout(eventName, timeout, ...args) {
        return new Promise((resolve, reject) => {
            const timer = setTimeout(() => {
                reject(new Error(`Event ${eventName} timed out after ${timeout}ms`));
            }, timeout);
            
            const cleanup = () => clearTimeout(timer);
            
            this.once(eventName + ':response', (response) => {
                cleanup();
                resolve(response);
            });
            
            this.emit(eventName, ...args);
        });
    }
    
    // Get event statistics
    getStats() {
        return {
            eventHistory: this.eventHistory.slice(-10), // Last 10 events
            listenerStats: Object.fromEntries(this.listenerStats),
            currentListeners: this.eventNames().map(name => ({
                event: name,
                count: this.listenerCount(name)
            }))
        };
    }
    
    // Wait for event
    waitFor(eventName, timeout = 5000) {
        return new Promise((resolve, reject) => {
            const timer = setTimeout(() => {
                this.removeListener(eventName, listener);
                reject(new Error(`Timeout waiting for event: ${eventName}`));
            }, timeout);
            
            const listener = (...args) => {
                clearTimeout(timer);
                resolve(args.length === 1 ? args[0] : args);
            };
            
            this.once(eventName, listener);
        });
    }
}

// Event-driven application example
class Application extends EnhancedEventEmitter {
    constructor() {
        super({ enableLogging: true });
        this.state = 'stopped';
        this.modules = new Map();
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        this.on('app:start', this.handleStart.bind(this));
        this.on('app:stop', this.handleStop.bind(this));
        this.on('module:register', this.handleModuleRegister.bind(this));
        this.on('module:error', this.handleModuleError.bind(this));
    }
    
    async handleStart() {
        if (this.state === 'running') {
            this.emit('app:warning', 'Application already running');
            return;
        }
        
        this.state = 'starting';
        this.emit('app:state-change', { from: 'stopped', to: 'starting' });
        
        try {
            // Initialize modules
            for (const [name, module] of this.modules) {
                await module.start();
                this.emit('module:started', name);
            }
            
            this.state = 'running';
            this.emit('app:state-change', { from: 'starting', to: 'running' });
            this.emit('app:ready');
            
        } catch (error) {
            this.state = 'error';
            this.emit('app:error', error);
        }
    }
    
    async handleStop() {
        if (this.state === 'stopped') {
            this.emit('app:warning', 'Application already stopped');
            return;
        }
        
        this.state = 'stopping';
        this.emit('app:state-change', { from: 'running', to: 'stopping' });
        
        try {
            // Stop modules in reverse order
            const moduleEntries = Array.from(this.modules.entries()).reverse();
            
            for (const [name, module] of moduleEntries) {
                await module.stop();
                this.emit('module:stopped', name);
            }
            
            this.state = 'stopped';
            this.emit('app:state-change', { from: 'stopping', to: 'stopped' });
            
        } catch (error) {
            this.emit('app:error', error);
        }
    }
    
    handleModuleRegister({ name, module }) {
        this.modules.set(name, module);
        this.emit('app:info', `Module ${name} registered`);
    }
    
    handleModuleError({ module, error }) {
        this.emit('app:error', `Module ${module} error: ${error.message}`);
    }
    
    // Public methods
    async start() {
        return this.emitAsync('app:start');
    }
    
    async stop() {
        return this.emitAsync('app:stop');
    }
    
    registerModule(name, module) {
        this.emit('module:register', { name, module });
    }
    
    getState() {
        return this.state;
    }
}

// Example module
class DatabaseModule extends EventEmitter {
    constructor(config) {
        super();
        this.config = config;
        this.connected = false;
    }
    
    async start() {
        this.emit('connecting');
        
        // Simulate connection
        await new Promise(resolve => setTimeout(resolve, 1000));
        
        if (Math.random() > 0.8) {
            const error = new Error('Database connection failed');
            this.emit('error', error);
            throw error;
        }
        
        this.connected = true;
        this.emit('connected');
    }
    
    async stop() {
        this.emit('disconnecting');
        
        // Simulate disconnection
        await new Promise(resolve => setTimeout(resolve, 500));
        
        this.connected = false;
        this.emit('disconnected');
    }
}

// Usage example
async function eventDrivenApp() {
    const app = new Application();
    
    // Set up application event listeners
    app.on('app:ready', () => {
        console.log('✅ Application is ready!');
    });
    
    app.on('app:error', (error) => {
        console.error('❌ Application error:', error.message);
    });
    
    app.on('app:state-change', ({ from, to }) => {
        console.log(`🔄 State change: ${from} → ${to}`);
    });
    
    app.on('module:started', (name) => {
        console.log(`📦 Module started: ${name}`);
    });
    
    // Register modules
    const dbModule = new DatabaseModule({ host: 'localhost', port: 5432 });
    app.registerModule('database', dbModule);
    
    // Set up module event listeners
    dbModule.on('connecting', () => console.log('🔌 Connecting to database...'));
    dbModule.on('connected', () => console.log('✅ Database connected'));
    dbModule.on('error', (error) => console.error('❌ Database error:', error.message));
    
    try {
        // Start application
        await app.start();
        
        // Wait for some time
        await new Promise(resolve => setTimeout(resolve, 3000));
        
        // Stop application
        await app.stop();
        
        // Show statistics
        console.log('\n📊 Event Statistics:');
        console.log(JSON.stringify(app.getStats(), null, 2));
        
    } catch (error) {
        console.error('Application failed:', error.message);
    }
}

eventDrivenApp();
```

### Event-driven Architecture Patterns

```javascript
const EventEmitter = require('events');

// Publisher-Subscriber Pattern
class EventBus extends EventEmitter {
    constructor() {
        super();
        this.topics = new Map();
        this.subscribers = new Map();
    }
    
    subscribe(topic, subscriber, handler) {
        if (!this.topics.has(topic)) {
            this.topics.set(topic, new Set());
        }
        
        if (!this.subscribers.has(subscriber)) {
            this.subscribers.set(subscriber, new Set());
        }
        
        this.topics.get(topic).add(subscriber);
        this.subscribers.get(subscriber).add(topic);
        
        this.on(topic, handler);
        
        console.log(`Subscriber ${subscriber} subscribed to ${topic}`);
    }
    
    unsubscribe(topic, subscriber) {
        if (this.topics.has(topic)) {
            this.topics.get(topic).delete(subscriber);
        }
        
        if (this.subscribers.has(subscriber)) {
            this.subscribers.get(subscriber).delete(topic);
        }
        
        console.log(`Subscriber ${subscriber} unsubscribed from ${topic}`);
    }
    
    publish(topic, data) {
        console.log(`Publishing to ${topic}:`, data);
        this.emit(topic, data);
    }
    
    getTopics() {
        return Array.from(this.topics.keys());
    }
    
    getSubscribers(topic) {
        return Array.from(this.topics.get(topic) || []);
    }
}

// Command Pattern with Events
class CommandBus extends EventEmitter {
    constructor() {
        super();
        this.handlers = new Map();
        this.middleware = [];
    }
    
    registerHandler(commandType, handler) {
        this.handlers.set(commandType, handler);
        console.log(`Handler registered for command: ${commandType}`);
    }
    
    addMiddleware(middleware) {
        this.middleware.push(middleware);
    }
    
    async execute(command) {
        const { type, ...payload } = command;
        
        this.emit('command:received', { type, payload });
        
        try {
            // Apply middleware
            let context = { command, payload };
            
            for (const middleware of this.middleware) {
                context = await middleware(context);
            }
            
            // Execute command
            const handler = this.handlers.get(type);
            if (!handler) {
                throw new Error(`No handler registered for command: ${type}`);
            }
            
            const result = await handler(context.payload);
            
            this.emit('command:success', { type, payload, result });
            return result;
            
        } catch (error) {
            this.emit('command:error', { type, payload, error });
            throw error;
        }
    }
}

// Saga Pattern for Complex Workflows
class SagaManager extends EventEmitter {
    constructor() {
        super();
        this.sagas = new Map();
        this.runningInstances = new Map();
    }
    
    registerSaga(name, sagaDefinition) {
        this.sagas.set(name, sagaDefinition);
        console.log(`Saga registered: ${name}`);
    }
    
    async startSaga(sagaName, initialData) {
        const sagaDefinition = this.sagas.get(sagaName);
        if (!sagaDefinition) {
            throw new Error(`Saga not found: ${sagaName}`);
        }
        
        const instanceId = this.generateId();
        const instance = {
            id: instanceId,
            name: sagaName,
            data: initialData,
            currentStep: 0,
            completedSteps: [],
            compensations: [],
            status: 'running'
        };
        
        this.runningInstances.set(instanceId, instance);
        this.emit('saga:started', { instanceId, sagaName, data: initialData });
        
        try {
            await this.executeSaga(instance, sagaDefinition);
        } catch (error) {
            await this.compensateSaga(instance);
            throw error;
        }
        
        return instanceId;
    }
    
    async executeSaga(instance, sagaDefinition) {
        const steps = sagaDefinition.steps;
        
        for (let i = instance.currentStep; i < steps.length; i++) {
            const step = steps[i];
            instance.currentStep = i;
            
            this.emit('saga:step-started', {
                instanceId: instance.id,
                stepIndex: i,
                stepName: step.name
            });
            
            try {
                const result = await step.execute(instance.data);
                
                instance.completedSteps.push({
                    index: i,
                    name: step.name,
                    result,
                    timestamp: new Date()
                });
                
                if (step.compensate) {
                    instance.compensations.unshift({
                        stepIndex: i,
                        compensate: step.compensate,
                        data: result
                    });
                }
                
                // Update instance data with step result
                instance.data = { ...instance.data, ...result };
                
                this.emit('saga:step-completed', {
                    instanceId: instance.id,
                    stepIndex: i,
                    stepName: step.name,
                    result
                });
                
            } catch (error) {
                this.emit('saga:step-failed', {
                    instanceId: instance.id,
                    stepIndex: i,
                    stepName: step.name,
                    error
                });
                throw error;
            }
        }
        
        instance.status = 'completed';
        this.emit('saga:completed', { instanceId: instance.id });
    }
    
    async compensateSaga(instance) {
        instance.status = 'compensating';
        this.emit('saga:compensating', { instanceId: instance.id });
        
        for (const compensation of instance.compensations) {
            try {
                await compensation.compensate(compensation.data);
                this.emit('saga:compensation-completed', {
                    instanceId: instance.id,
                    stepIndex: compensation.stepIndex
                });
            } catch (error) {
                this.emit('saga:compensation-failed', {
                    instanceId: instance.id,
                    stepIndex: compensation.stepIndex,
                    error
                });
            }
        }
        
        instance.status = 'compensated';
        this.emit('saga:compensated', { instanceId: instance.id });
    }
    
    generateId() {
        return Math.random().toString(36).substr(2, 9);
    }
}

// Usage examples
async function eventPatternExamples() {
    console.log('=== Publisher-Subscriber Pattern ===');
    
    const eventBus = new EventBus();
    
    // Subscribers
    eventBus.subscribe('user.created', 'email-service', (user) => {
        console.log(`📧 Sending welcome email to ${user.email}`);
    });
    
    eventBus.subscribe('user.created', 'analytics-service', (user) => {
        console.log(`📊 Recording user signup: ${user.id}`);
    });
    
    eventBus.subscribe('order.placed', 'inventory-service', (order) => {
        console.log(`📦 Updating inventory for order: ${order.id}`);
    });
    
    // Publish events
    eventBus.publish('user.created', { id: 1, email: 'user@example.com' });
    eventBus.publish('order.placed', { id: 101, items: ['item1', 'item2'] });
    
    console.log('\n=== Command Pattern ===');
    
    const commandBus = new CommandBus();
    
    // Middleware
    commandBus.addMiddleware(async (context) => {
        console.log(`🔍 Validating command: ${context.command.type}`);
        return context;
    });
    
    commandBus.addMiddleware(async (context) => {
        console.log(`📝 Logging command: ${context.command.type}`);
        return context;
    });
    
    // Command handlers
    commandBus.registerHandler('create-user', async (payload) => {
        console.log(`👤 Creating user: ${payload.name}`);
        return { id: Date.now(), ...payload };
    });
    
    commandBus.registerHandler('send-email', async (payload) => {
        console.log(`📧 Sending email to: ${payload.to}`);
        return { messageId: Math.random().toString(36) };
    });
    
    // Execute commands
    try {
        const user = await commandBus.execute({
            type: 'create-user',
            name: 'John Doe',
            email: 'john@example.com'
        });
        
        await commandBus.execute({
            type: 'send-email',
            to: user.email,
            subject: 'Welcome!',
            body: 'Welcome to our platform!'
        });
        
    } catch (error) {
        console.error('Command failed:', error.message);
    }
    
    console.log('\n=== Saga Pattern ===');
    
    const sagaManager = new SagaManager();
    
    // Define order processing saga
    sagaManager.registerSaga('process-order', {
        steps: [
            {
                name: 'validate-order',
                execute: async (data) => {
                    console.log('🔍 Validating order...');
                    if (!data.items || data.items.length === 0) {
                        throw new Error('Order has no items');
                    }
                    return { validated: true };
                },
                compensate: async (data) => {
                    console.log('↩️ Reverting order validation...');
                }
            },
            {
                name: 'reserve-inventory',
                execute: async (data) => {
                    console.log('📦 Reserving inventory...');
                    return { reservationId: 'res-123' };
                },
                compensate: async (data) => {
                    console.log('↩️ Releasing inventory reservation...');
                }
            },
            {
                name: 'process-payment',
                execute: async (data) => {
                    console.log('💳 Processing payment...');
                    if (Math.random() < 0.3) {
                        throw new Error('Payment failed');
                    }
                    return { transactionId: 'tx-456' };
                },
                compensate: async (data) => {
                    console.log('↩️ Refunding payment...');
                }
            },
            {
                name: 'fulfill-order',
                execute: async (data) => {
                    console.log('🚚 Fulfilling order...');
                    return { trackingNumber: 'track-789' };
                },
                compensate: async (data) => {
                    console.log('↩️ Canceling fulfillment...');
                }
            }
        ]
    });
    
    // Saga event listeners
    sagaManager.on('saga:started', ({ instanceId, sagaName }) => {
        console.log(`🚀 Saga started: ${sagaName} (${instanceId})`);
    });
    
    sagaManager.on('saga:completed', ({ instanceId }) => {
        console.log(`✅ Saga completed: ${instanceId}`);
    });
    
    sagaManager.on('saga:compensated', ({ instanceId }) => {
        console.log(`↩️ Saga compensated: ${instanceId}`);
    });
    
    // Execute saga
    try {
        const instanceId = await sagaManager.startSaga('process-order', {
            orderId: 'order-123',
            items: ['item1', 'item2'],
            total: 99.99
        });
        
        console.log(`Saga instance created: ${instanceId}`);
        
    } catch (error) {
        console.error('Saga failed:', error.message);
    }
}

eventPatternExamples();
```

---

This covers the core HTTP, File System, Streams, and Events modules with comprehensive examples and advanced patterns. The content demonstrates practical usage scenarios and best practices for each module. Would you like me to continue with the remaining core APIs (URL, Crypto, Child Process, Cluster, Utilities, and Performance Hooks)?
