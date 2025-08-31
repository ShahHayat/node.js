# Node.js Modules and Package Management - Complete Guide

## Table of Contents
1. [Module System Overview](#module-system-overview)
2. [CommonJS Modules](#commonjs-modules)
3. [ES6 Modules (ESM)](#es6-modules-esm)
4. [Built-in Modules](#built-in-modules)
5. [npm (Node Package Manager)](#npm-node-package-manager)
6. [Package.json Deep Dive](#packagejson-deep-dive)
7. [Dependency Management](#dependency-management)
8. [Publishing Packages](#publishing-packages)
9. [Alternative Package Managers](#alternative-package-managers)
10. [Module Resolution](#module-resolution)
11. [Best Practices](#best-practices)

## Module System Overview

Node.js uses a modular architecture where code is organized into reusable modules. There are three types of modules:

1. **Built-in modules**: Core modules that come with Node.js
2. **Local modules**: Modules you create in your application
3. **Third-party modules**: Modules installed via npm

### Module Types

```javascript
// Built-in module
const fs = require('fs');
const path = require('path');

// Local module
const myModule = require('./my-module');
const utils = require('../utils/helpers');

// Third-party module
const express = require('express');
const lodash = require('lodash');
```

## CommonJS Modules

CommonJS is the traditional module system used in Node.js. It uses `require()` to import modules and `module.exports` or `exports` to export them.

### Basic Module Creation and Usage

```javascript
// math.js - Module definition
const PI = 3.14159;

function add(a, b) {
    return a + b;
}

function subtract(a, b) {
    return a - b;
}

function multiply(a, b) {
    return a * b;
}

function divide(a, b) {
    if (b === 0) {
        throw new Error('Division by zero');
    }
    return a / b;
}

// Export individual functions
exports.add = add;
exports.subtract = subtract;

// Or export as object
module.exports = {
    PI,
    add,
    subtract,
    multiply,
    divide
};

// app.js - Module usage
const math = require('./math');

console.log(math.add(5, 3)); // 8
console.log(math.PI); // 3.14159

// Destructuring import
const { add, subtract, PI } = require('./math');
console.log(add(10, 5)); // 15
console.log(subtract(10, 5)); // 5
```

### exports vs module.exports

```javascript
// Understanding the difference

// This works - adding properties to exports
exports.method1 = function() { return 'method1'; };
exports.method2 = function() { return 'method2'; };

// This doesn't work as expected - reassigning exports
exports = {
    method3: function() { return 'method3'; }
};

// This works - reassigning module.exports
module.exports = {
    method4: function() { return 'method4'; }
};

// This also works - adding to module.exports after reassignment
module.exports.method5 = function() { return 'method5'; };

// Explanation:
// exports is just a reference to module.exports
// When you reassign exports, you break the reference
// module.exports is what actually gets exported
```

### Advanced Module Patterns

```javascript
// 1. Constructor Pattern
// user.js
function User(name, email) {
    this.name = name;
    this.email = email;
    this.createdAt = new Date();
}

User.prototype.getInfo = function() {
    return `${this.name} (${this.email})`;
};

User.prototype.isActive = function() {
    const daysSinceCreation = (Date.now() - this.createdAt.getTime()) / (1000 * 60 * 60 * 24);
    return daysSinceCreation < 30;
};

module.exports = User;

// Usage
const User = require('./user');
const user = new User('John Doe', 'john@example.com');
console.log(user.getInfo());

// 2. Factory Pattern
// logger.js
function createLogger(level = 'info') {
    const levels = {
        error: 0,
        warn: 1,
        info: 2,
        debug: 3
    };
    
    const currentLevel = levels[level];
    
    return {
        error: (message) => {
            if (currentLevel >= levels.error) {
                console.error(`[ERROR] ${new Date().toISOString()}: ${message}`);
            }
        },
        warn: (message) => {
            if (currentLevel >= levels.warn) {
                console.warn(`[WARN] ${new Date().toISOString()}: ${message}`);
            }
        },
        info: (message) => {
            if (currentLevel >= levels.info) {
                console.info(`[INFO] ${new Date().toISOString()}: ${message}`);
            }
        },
        debug: (message) => {
            if (currentLevel >= levels.debug) {
                console.debug(`[DEBUG] ${new Date().toISOString()}: ${message}`);
            }
        }
    };
}

module.exports = createLogger;

// Usage
const createLogger = require('./logger');
const logger = createLogger('debug');
logger.info('Application started');

// 3. Singleton Pattern
// database.js
class Database {
    constructor() {
        if (Database.instance) {
            return Database.instance;
        }
        
        this.connection = null;
        this.isConnected = false;
        Database.instance = this;
    }
    
    connect(connectionString) {
        if (!this.isConnected) {
            // Simulate database connection
            this.connection = { url: connectionString };
            this.isConnected = true;
            console.log('Database connected');
        }
        return this.connection;
    }
    
    disconnect() {
        if (this.isConnected) {
            this.connection = null;
            this.isConnected = false;
            console.log('Database disconnected');
        }
    }
    
    query(sql) {
        if (!this.isConnected) {
            throw new Error('Database not connected');
        }
        console.log(`Executing: ${sql}`);
        return { result: 'mock data' };
    }
}

module.exports = new Database(); // Export instance, not class

// Usage
const db = require('./database');
db.connect('mongodb://localhost:27017');
db.query('SELECT * FROM users');

// 4. Module with initialization
// config.js
let config = null;

function initialize(environment = 'development') {
    if (config) {
        return config;
    }
    
    const configs = {
        development: {
            port: 3000,
            database: 'mongodb://localhost:27017/dev',
            debug: true
        },
        production: {
            port: process.env.PORT || 8080,
            database: process.env.DATABASE_URL,
            debug: false
        },
        test: {
            port: 3001,
            database: 'mongodb://localhost:27017/test',
            debug: false
        }
    };
    
    config = configs[environment] || configs.development;
    return config;
}

function get(key) {
    if (!config) {
        throw new Error('Config not initialized. Call initialize() first.');
    }
    return config[key];
}

module.exports = {
    initialize,
    get
};

// Usage
const config = require('./config');
config.initialize(process.env.NODE_ENV);
console.log('Port:', config.get('port'));
```

### Module Caching

```javascript
// counter.js
let count = 0;

function increment() {
    return ++count;
}

function getCount() {
    return count;
}

module.exports = { increment, getCount };

// app.js
const counter1 = require('./counter');
const counter2 = require('./counter');

console.log(counter1.increment()); // 1
console.log(counter2.increment()); // 2 (same instance due to caching)
console.log(counter1.getCount()); // 2

// Verify they're the same instance
console.log(counter1 === counter2); // true

// Clear cache (useful for testing)
delete require.cache[require.resolve('./counter')];
const counter3 = require('./counter');
console.log(counter3.getCount()); // 0 (fresh instance)

// Module cache inspection
console.log('Cached modules:');
Object.keys(require.cache).forEach(module => {
    console.log(module);
});
```

### Circular Dependencies

```javascript
// a.js
console.log('Loading module A');
const b = require('./b');

function funcA() {
    console.log('Function A called');
    b.funcB();
}

module.exports = { funcA };

// b.js
console.log('Loading module B');
const a = require('./a');

function funcB() {
    console.log('Function B called');
    // a might be undefined or incomplete here
    if (a && a.funcA) {
        console.log('Module A is available');
    } else {
        console.log('Module A is not yet complete');
    }
}

module.exports = { funcB };

// Better approach - lazy loading
// b-improved.js
console.log('Loading module B (improved)');

function funcB() {
    console.log('Function B called');
    // Lazy load to avoid circular dependency issues
    const a = require('./a');
    if (a && a.funcA) {
        console.log('Module A is available');
    }
}

module.exports = { funcB };
```

## ES6 Modules (ESM)

ES6 modules are the standardized module system for JavaScript, supported in Node.js 12+.

### Enabling ES6 Modules

```json
// package.json
{
  "type": "module"
}
```

Or use `.mjs` extension for individual files.

### Basic ES6 Module Syntax

```javascript
// math.mjs
export const PI = 3.14159;

export function add(a, b) {
    return a + b;
}

export function subtract(a, b) {
    return a - b;
}

// Default export
export default function multiply(a, b) {
    return a * b;
}

// app.mjs
import multiply, { add, subtract, PI } from './math.mjs';
import * as math from './math.mjs';

console.log(add(5, 3)); // 8
console.log(multiply(4, 5)); // 20
console.log(math.PI); // 3.14159

// Dynamic imports
async function loadMath() {
    const mathModule = await import('./math.mjs');
    console.log(mathModule.add(10, 5));
}

loadMath();
```

### Advanced ES6 Module Patterns

```javascript
// user.mjs
export class User {
    constructor(name, email) {
        this.name = name;
        this.email = email;
        this.id = Math.random().toString(36).substr(2, 9);
    }
    
    getProfile() {
        return {
            id: this.id,
            name: this.name,
            email: this.email
        };
    }
}

export class AdminUser extends User {
    constructor(name, email, permissions = []) {
        super(name, email);
        this.permissions = permissions;
        this.role = 'admin';
    }
    
    hasPermission(permission) {
        return this.permissions.includes(permission);
    }
}

export const USER_ROLES = {
    ADMIN: 'admin',
    USER: 'user',
    GUEST: 'guest'
};

export function createUser(userData) {
    if (userData.role === USER_ROLES.ADMIN) {
        return new AdminUser(userData.name, userData.email, userData.permissions);
    }
    return new User(userData.name, userData.email);
}

// Re-exports
export { validateEmail } from './validators.mjs';
export { hashPassword } from './security.mjs';

// app.mjs
import { User, AdminUser, USER_ROLES, createUser } from './user.mjs';

const admin = createUser({
    name: 'Admin User',
    email: 'admin@example.com',
    role: USER_ROLES.ADMIN,
    permissions: ['read', 'write', 'delete']
});

console.log(admin.getProfile());
```

### Conditional Imports

```javascript
// feature-loader.mjs
export async function loadFeature(featureName) {
    switch (featureName) {
        case 'analytics':
            const analytics = await import('./features/analytics.mjs');
            return analytics.default;
        
        case 'payment':
            const payment = await import('./features/payment.mjs');
            return payment.default;
        
        case 'chat':
            const chat = await import('./features/chat.mjs');
            return chat.default;
        
        default:
            throw new Error(`Unknown feature: ${featureName}`);
    }
}

// app.mjs
import { loadFeature } from './feature-loader.mjs';

async function initializeApp() {
    const features = ['analytics', 'payment'];
    
    for (const featureName of features) {
        try {
            const feature = await loadFeature(featureName);
            await feature.initialize();
            console.log(`${featureName} feature loaded`);
        } catch (error) {
            console.error(`Failed to load ${featureName}:`, error);
        }
    }
}

initializeApp();
```

### Mixed Module Systems

```javascript
// Using CommonJS in ES6 modules
import { createRequire } from 'module';
const require = createRequire(import.meta.url);

const fs = require('fs'); // CommonJS module
const config = require('./config.json'); // JSON file

// Using ES6 modules in CommonJS
// commonjs-file.js
(async () => {
    const { default: fetch } = await import('node-fetch');
    const response = await fetch('https://api.example.com/data');
    const data = await response.json();
    console.log(data);
})();
```

## Built-in Modules

Node.js comes with many built-in modules that provide core functionality.

### File System (fs)

```javascript
const fs = require('fs');
const path = require('path');
const { promisify } = require('util');

// Synchronous operations
try {
    const data = fs.readFileSync('example.txt', 'utf8');
    console.log(data);
} catch (error) {
    console.error('Error reading file:', error);
}

// Asynchronous with callbacks
fs.readFile('example.txt', 'utf8', (error, data) => {
    if (error) {
        console.error('Error:', error);
        return;
    }
    console.log(data);
});

// Promises-based API
const fsPromises = fs.promises;

async function readFileAsync() {
    try {
        const data = await fsPromises.readFile('example.txt', 'utf8');
        console.log(data);
    } catch (error) {
        console.error('Error:', error);
    }
}

// File operations
async function fileOperations() {
    try {
        // Write file
        await fsPromises.writeFile('output.txt', 'Hello World', 'utf8');
        
        // Append to file
        await fsPromises.appendFile('output.txt', '\nAppended text', 'utf8');
        
        // Check if file exists
        try {
            await fsPromises.access('output.txt');
            console.log('File exists');
        } catch {
            console.log('File does not exist');
        }
        
        // Get file stats
        const stats = await fsPromises.stat('output.txt');
        console.log('File size:', stats.size);
        console.log('Is file:', stats.isFile());
        console.log('Is directory:', stats.isDirectory());
        
        // Copy file
        await fsPromises.copyFile('output.txt', 'backup.txt');
        
        // Rename/move file
        await fsPromises.rename('backup.txt', 'renamed.txt');
        
        // Delete file
        await fsPromises.unlink('renamed.txt');
        
    } catch (error) {
        console.error('File operation error:', error);
    }
}

// Directory operations
async function directoryOperations() {
    try {
        // Create directory
        await fsPromises.mkdir('test-dir', { recursive: true });
        
        // Read directory
        const files = await fsPromises.readdir('.');
        console.log('Files in current directory:', files);
        
        // Read directory with file types
        const entries = await fsPromises.readdir('.', { withFileTypes: true });
        entries.forEach(entry => {
            console.log(`${entry.name} - ${entry.isFile() ? 'File' : 'Directory'}`);
        });
        
        // Remove directory
        await fsPromises.rmdir('test-dir');
        
    } catch (error) {
        console.error('Directory operation error:', error);
    }
}
```

### Path Module

```javascript
const path = require('path');

// Path components
const filePath = '/users/john/documents/file.txt';

console.log('Directory name:', path.dirname(filePath)); // /users/john/documents
console.log('Base name:', path.basename(filePath)); // file.txt
console.log('Extension:', path.extname(filePath)); // .txt
console.log('Name without extension:', path.basename(filePath, '.txt')); // file

// Path construction
const fullPath = path.join('/users', 'john', 'documents', 'file.txt');
console.log('Joined path:', fullPath);

// Resolve absolute path
const absolutePath = path.resolve('documents', 'file.txt');
console.log('Absolute path:', absolutePath);

// Relative path
const relativePath = path.relative('/users/john', '/users/john/documents/file.txt');
console.log('Relative path:', relativePath); // documents/file.txt

// Parse path
const parsedPath = path.parse(filePath);
console.log('Parsed path:', parsedPath);
// {
//   root: '/',
//   dir: '/users/john/documents',
//   base: 'file.txt',
//   ext: '.txt',
//   name: 'file'
// }

// Format path
const formattedPath = path.format({
    dir: '/users/john/documents',
    name: 'file',
    ext: '.txt'
});
console.log('Formatted path:', formattedPath);

// Platform-specific separators
console.log('Path separator:', path.sep); // '/' on Unix, '\\' on Windows
console.log('Path delimiter:', path.delimiter); // ':' on Unix, ';' on Windows

// Normalize path
const messyPath = '/users/john/../john/./documents//file.txt';
console.log('Normalized:', path.normalize(messyPath)); // /users/john/documents/file.txt
```

### OS Module

```javascript
const os = require('os');

// System information
console.log('Platform:', os.platform());
console.log('Architecture:', os.arch());
console.log('CPU cores:', os.cpus().length);
console.log('Total memory:', Math.round(os.totalmem() / 1024 / 1024 / 1024), 'GB');
console.log('Free memory:', Math.round(os.freemem() / 1024 / 1024 / 1024), 'GB');
console.log('Uptime:', Math.round(os.uptime() / 60 / 60), 'hours');

// User information
console.log('Username:', os.userInfo().username);
console.log('Home directory:', os.homedir());
console.log('Temp directory:', os.tmpdir());

// Network interfaces
const networkInterfaces = os.networkInterfaces();
Object.keys(networkInterfaces).forEach(interface => {
    console.log(`${interface}:`, networkInterfaces[interface]);
});

// System constants
console.log('EOL character:', JSON.stringify(os.EOL));
console.log('Load average:', os.loadavg());
```

### URL Module

```javascript
const url = require('url');
const { URL, URLSearchParams } = require('url');

// Parse URL (legacy API)
const parsedUrl = url.parse('https://example.com:8080/path?query=value#fragment');
console.log('Parsed URL:', parsedUrl);

// Format URL (legacy API)
const formattedUrl = url.format({
    protocol: 'https:',
    hostname: 'example.com',
    port: '8080',
    pathname: '/path',
    query: { query: 'value' },
    hash: '#fragment'
});
console.log('Formatted URL:', formattedUrl);

// Modern URL API
const myUrl = new URL('https://example.com:8080/path?query=value&foo=bar#fragment');

console.log('Protocol:', myUrl.protocol);
console.log('Hostname:', myUrl.hostname);
console.log('Port:', myUrl.port);
console.log('Pathname:', myUrl.pathname);
console.log('Search:', myUrl.search);
console.log('Hash:', myUrl.hash);

// URL search parameters
console.log('Query parameter:', myUrl.searchParams.get('query'));
myUrl.searchParams.set('newParam', 'newValue');
myUrl.searchParams.delete('foo');
console.log('Updated URL:', myUrl.toString());

// URLSearchParams
const params = new URLSearchParams('query=value&foo=bar&query=another');
console.log('All query values:', params.getAll('query'));

params.forEach((value, key) => {
    console.log(`${key}: ${value}`);
});

// Convert to object
const paramsObject = Object.fromEntries(params.entries());
console.log('Params object:', paramsObject);
```

### Crypto Module

```javascript
const crypto = require('crypto');

// Generate random data
const randomBytes = crypto.randomBytes(16);
console.log('Random bytes:', randomBytes.toString('hex'));

// Hash data
const hash = crypto.createHash('sha256');
hash.update('Hello World');
const digest = hash.digest('hex');
console.log('SHA256 hash:', digest);

// HMAC
const hmac = crypto.createHmac('sha256', 'secret-key');
hmac.update('Hello World');
const hmacDigest = hmac.digest('hex');
console.log('HMAC:', hmacDigest);

// Encryption/Decryption
function encrypt(text, password) {
    const algorithm = 'aes-256-gcm';
    const key = crypto.scryptSync(password, 'salt', 32);
    const iv = crypto.randomBytes(16);
    
    const cipher = crypto.createCipher(algorithm, key);
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    return {
        encrypted,
        iv: iv.toString('hex')
    };
}

function decrypt(encryptedData, password) {
    const algorithm = 'aes-256-gcm';
    const key = crypto.scryptSync(password, 'salt', 32);
    
    const decipher = crypto.createDecipher(algorithm, key);
    let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
}

// Usage
const originalText = 'This is a secret message';
const password = 'my-secret-password';

const encrypted = encrypt(originalText, password);
console.log('Encrypted:', encrypted);

const decrypted = decrypt(encrypted, password);
console.log('Decrypted:', decrypted);

// Generate key pairs
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

console.log('Public key:', publicKey);
console.log('Private key length:', privateKey.length);
```

## npm (Node Package Manager)

npm is the default package manager for Node.js, providing access to the largest ecosystem of open source libraries.

### Basic npm Commands

```bash
# Initialize a new project
npm init
npm init -y  # Skip questions

# Install packages
npm install <package-name>
npm install <package-name>@<version>
npm install <package-name> --save-dev
npm install <package-name> --global

# Install from package.json
npm install
npm ci  # Clean install (faster, for production)

# Update packages
npm update
npm update <package-name>

# Remove packages
npm uninstall <package-name>
npm uninstall <package-name> --save-dev

# List installed packages
npm list
npm list --depth=0
npm list --global

# Search packages
npm search <keyword>

# Get package information
npm info <package-name>
npm view <package-name> versions

# Run scripts
npm run <script-name>
npm start
npm test

# Audit packages for vulnerabilities
npm audit
npm audit fix

# Clean cache
npm cache clean --force
```

### Package Installation Examples

```javascript
// package.json
{
  "name": "my-app",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.0",
    "lodash": "~4.17.21",
    "moment": "2.29.4"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "nodemon": "^2.0.20",
    "eslint": "^8.0.0"
  }
}

// Version ranges
// ^4.18.0 - Compatible with 4.18.0, allows 4.x.x but not 5.0.0
// ~4.17.21 - Reasonably close to 4.17.21, allows 4.17.x but not 4.18.0
// 2.29.4 - Exact version
// * - Latest version (not recommended)
// >4.0.0 - Greater than 4.0.0
// >=4.0.0 <5.0.0 - Range
```

### npm Scripts

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "build": "webpack --mode=production",
    "test": "jest",
    "test:watch": "jest --watch",
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "clean": "rm -rf dist/",
    "prebuild": "npm run clean",
    "postbuild": "npm run test",
    "deploy": "npm run build && npm run deploy:prod",
    "deploy:prod": "rsync -avz dist/ user@server:/var/www/",
    "db:migrate": "node scripts/migrate.js",
    "db:seed": "node scripts/seed.js"
  }
}
```

```bash
# Run scripts
npm run dev
npm run build
npm test

# Pre/post hooks automatically run
npm run build  # Runs prebuild, build, postbuild
```

### npx - Package Runner

```bash
# Run package without installing globally
npx create-react-app my-app
npx eslint src/
npx jest --version

# Run local package
npx nodemon server.js

# Run package from URL
npx https://gist.github.com/username/gist-id

# Run specific version
npx jest@28 --version
```

### npm Configuration

```bash
# View configuration
npm config list
npm config list -l  # All config

# Set configuration
npm config set registry https://registry.npmjs.org/
npm config set init-author-name "Your Name"
npm config set init-author-email "you@example.com"
npm config set init-license "MIT"

# Delete configuration
npm config delete proxy

# Edit config file
npm config edit
```

```ini
# .npmrc file
registry=https://registry.npmjs.org/
save-exact=true
init-author-name=Your Name
init-author-email=you@example.com
init-license=MIT
```

## Package.json Deep Dive

The `package.json` file is the heart of any Node.js project, containing metadata and configuration.

### Complete Package.json Structure

```json
{
  "name": "my-awesome-package",
  "version": "1.2.3",
  "description": "A comprehensive example package.json",
  "keywords": ["node", "javascript", "example"],
  "homepage": "https://github.com/username/my-awesome-package#readme",
  "bugs": {
    "url": "https://github.com/username/my-awesome-package/issues",
    "email": "support@example.com"
  },
  "license": "MIT",
  "author": {
    "name": "Your Name",
    "email": "you@example.com",
    "url": "https://yourwebsite.com"
  },
  "contributors": [
    {
      "name": "Contributor Name",
      "email": "contributor@example.com"
    }
  ],
  "funding": {
    "type": "patreon",
    "url": "https://patreon.com/username"
  },
  "main": "index.js",
  "module": "index.mjs",
  "types": "index.d.ts",
  "exports": {
    ".": {
      "import": "./index.mjs",
      "require": "./index.js",
      "types": "./index.d.ts"
    },
    "./utils": {
      "import": "./utils/index.mjs",
      "require": "./utils/index.js"
    }
  },
  "bin": {
    "my-cli": "./bin/cli.js"
  },
  "files": [
    "lib/",
    "bin/",
    "index.js",
    "index.mjs",
    "README.md"
  ],
  "directories": {
    "lib": "./lib",
    "bin": "./bin",
    "doc": "./docs",
    "test": "./test"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/username/my-awesome-package.git"
  },
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "build": "webpack --mode=production",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/",
    "format": "prettier --write src/",
    "prepare": "husky install",
    "prepublishOnly": "npm test && npm run lint",
    "preversion": "npm run lint",
    "version": "npm run format && git add -A src",
    "postversion": "git push && git push --tags"
  },
  "dependencies": {
    "express": "^4.18.0",
    "lodash": "^4.17.21"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "eslint": "^8.0.0",
    "prettier": "^2.7.0",
    "husky": "^8.0.0",
    "webpack": "^5.74.0"
  },
  "peerDependencies": {
    "react": ">=16.0.0"
  },
  "peerDependenciesMeta": {
    "react": {
      "optional": true
    }
  },
  "optionalDependencies": {
    "fsevents": "^2.3.2"
  },
  "bundledDependencies": [
    "lodash"
  ],
  "engines": {
    "node": ">=14.0.0",
    "npm": ">=6.0.0"
  },
  "os": [
    "linux",
    "darwin"
  ],
  "cpu": [
    "x64",
    "arm64"
  ],
  "private": false,
  "publishConfig": {
    "registry": "https://registry.npmjs.org/",
    "access": "public"
  },
  "workspaces": [
    "packages/*"
  ],
  "config": {
    "port": "3000"
  },
  "browserslist": [
    "> 1%",
    "last 2 versions"
  ],
  "eslintConfig": {
    "extends": ["eslint:recommended"]
  },
  "jest": {
    "testEnvironment": "node"
  }
}
```

### Package.json Scripts Lifecycle

```json
{
  "scripts": {
    "preinstall": "echo 'Before install'",
    "install": "echo 'During install'",
    "postinstall": "echo 'After install'",
    
    "prepack": "echo 'Before pack'",
    "pack": "echo 'During pack'",
    "postpack": "echo 'After pack'",
    
    "prepublish": "echo 'Before publish (deprecated)'",
    "prepare": "echo 'After install and before publish'",
    "prepublishOnly": "echo 'Before publish only'",
    "preversion": "echo 'Before version bump'",
    "version": "echo 'During version bump'",
    "postversion": "echo 'After version bump'",
    
    "pretest": "echo 'Before test'",
    "test": "jest",
    "posttest": "echo 'After test'",
    
    "prestop": "echo 'Before stop'",
    "stop": "echo 'Stop script'",
    "poststop": "echo 'After stop'",
    
    "prestart": "echo 'Before start'",
    "start": "node server.js",
    "poststart": "echo 'After start'",
    
    "prerestart": "echo 'Before restart'",
    "restart": "echo 'Restart script'",
    "postrestart": "echo 'After restart'"
  }
}
```

### Environment Variables in Scripts

```json
{
  "scripts": {
    "dev": "NODE_ENV=development nodemon server.js",
    "prod": "NODE_ENV=production node server.js",
    "test": "NODE_ENV=test jest",
    "build:dev": "NODE_ENV=development webpack",
    "build:prod": "NODE_ENV=production webpack",
    "cross-platform": "cross-env NODE_ENV=production node server.js"
  },
  "devDependencies": {
    "cross-env": "^7.0.3"
  }
}
```

## Dependency Management

### Semantic Versioning (SemVer)

```
MAJOR.MINOR.PATCH

1.2.3
│ │ │
│ │ └── PATCH: Bug fixes (backwards compatible)
│ └──── MINOR: New features (backwards compatible)
└────── MAJOR: Breaking changes (not backwards compatible)

Version Ranges:
^1.2.3 := >=1.2.3 <2.0.0 (Compatible with 1.2.3)
~1.2.3 := >=1.2.3 <1.3.0 (Reasonably close to 1.2.3)
1.2.3 := Exactly 1.2.3
>1.2.3 := Greater than 1.2.3
>=1.2.3 := Greater than or equal to 1.2.3
<1.2.3 := Less than 1.2.3
<=1.2.3 := Less than or equal to 1.2.3
1.2.3 - 2.3.4 := >=1.2.3 <=2.3.4
1.2.3 || >=2.5.0 || 5.0.0 - 7.2.3 := Any of these versions
* := Any version
```

### Lock Files

```javascript
// package-lock.json structure (simplified)
{
  "name": "my-app",
  "version": "1.0.0",
  "lockfileVersion": 2,
  "requires": true,
  "packages": {
    "": {
      "name": "my-app",
      "version": "1.0.0",
      "dependencies": {
        "express": "^4.18.0"
      }
    },
    "node_modules/express": {
      "version": "4.18.2",
      "resolved": "https://registry.npmjs.org/express/-/express-4.18.2.tgz",
      "integrity": "sha512-...",
      "dependencies": {
        "accepts": "~1.3.8",
        "array-flatten": "1.1.1"
      }
    }
  }
}

// Benefits of lock files:
// 1. Reproducible installs across environments
// 2. Exact dependency tree
// 3. Security - prevents malicious package substitution
// 4. Performance - faster installs
```

### Dependency Types

```javascript
// Production dependencies
// These are required for your app to run
"dependencies": {
  "express": "^4.18.0",
  "mongoose": "^6.6.0",
  "lodash": "^4.17.21"
}

// Development dependencies
// Only needed during development
"devDependencies": {
  "jest": "^29.0.0",
  "eslint": "^8.0.0",
  "nodemon": "^2.0.20",
  "webpack": "^5.74.0"
}

// Peer dependencies
// Required by your package but should be provided by the consumer
"peerDependencies": {
  "react": ">=16.0.0",
  "react-dom": ">=16.0.0"
}

// Optional dependencies
// Nice to have but not required
"optionalDependencies": {
  "fsevents": "^2.3.2"  // Only available on macOS
}

// Bundled dependencies
// Included in your package when published
"bundledDependencies": [
  "private-module"
]
```

### Dependency Resolution

```javascript
// How Node.js resolves modules
// 1. Core modules (fs, path, http)
// 2. File modules (./file, ../file, /absolute/path)
// 3. node_modules traversal

// Resolution algorithm for require('express'):
// 1. Check if 'express' is a core module
// 2. Look for node_modules/express in current directory
// 3. Look for node_modules/express in parent directory
// 4. Continue up the directory tree
// 5. Check global node_modules
// 6. Throw MODULE_NOT_FOUND error

// Module resolution debugging
const Module = require('module');
console.log('Module paths:', Module.globalPaths);
console.log('Resolve express:', require.resolve('express'));
console.log('Resolve paths for express:', Module._resolveFilename('express', module));
```

### Managing Dependencies

```javascript
// dependency-manager.js
const fs = require('fs');
const { execSync } = require('child_process');

class DependencyManager {
    constructor(packageJsonPath = './package.json') {
        this.packageJsonPath = packageJsonPath;
        this.packageJson = JSON.parse(fs.readFileSync(packageJsonPath, 'utf8'));
    }
    
    // Check for outdated packages
    checkOutdated() {
        try {
            const result = execSync('npm outdated --json', { encoding: 'utf8' });
            return JSON.parse(result);
        } catch (error) {
            // npm outdated returns non-zero exit code when packages are outdated
            if (error.stdout) {
                return JSON.parse(error.stdout);
            }
            return {};
        }
    }
    
    // Check for security vulnerabilities
    async auditPackages() {
        try {
            const result = execSync('npm audit --json', { encoding: 'utf8' });
            return JSON.parse(result);
        } catch (error) {
            if (error.stdout) {
                return JSON.parse(error.stdout);
            }
            throw error;
        }
    }
    
    // List unused dependencies
    findUnusedDependencies() {
        const dependencies = Object.keys(this.packageJson.dependencies || {});
        const devDependencies = Object.keys(this.packageJson.devDependencies || {});
        const allDependencies = [...dependencies, ...devDependencies];
        
        const used = new Set();
        
        // Simple check - scan files for require/import statements
        const scanDirectory = (dir) => {
            const files = fs.readdirSync(dir, { withFileTypes: true });
            
            files.forEach(file => {
                if (file.isDirectory() && file.name !== 'node_modules') {
                    scanDirectory(`${dir}/${file.name}`);
                } else if (file.name.match(/\.(js|mjs|ts|jsx|tsx)$/)) {
                    const content = fs.readFileSync(`${dir}/${file.name}`, 'utf8');
                    
                    // Find require statements
                    const requireMatches = content.match(/require\(['"`]([^'"`]+)['"`]\)/g);
                    if (requireMatches) {
                        requireMatches.forEach(match => {
                            const module = match.match(/require\(['"`]([^'"`]+)['"`]\)/)[1];
                            if (!module.startsWith('.') && !module.startsWith('/')) {
                                used.add(module.split('/')[0]); // Handle scoped packages
                            }
                        });
                    }
                    
                    // Find import statements
                    const importMatches = content.match(/import .+ from ['"`]([^'"`]+)['"`]/g);
                    if (importMatches) {
                        importMatches.forEach(match => {
                            const module = match.match(/from ['"`]([^'"`]+)['"`]/)[1];
                            if (!module.startsWith('.') && !module.startsWith('/')) {
                                used.add(module.split('/')[0]);
                            }
                        });
                    }
                }
            });
        };
        
        scanDirectory('.');
        
        return allDependencies.filter(dep => !used.has(dep));
    }
    
    // Generate dependency report
    generateReport() {
        const outdated = this.checkOutdated();
        const unused = this.findUnusedDependencies();
        
        console.log('=== Dependency Report ===\n');
        
        console.log('Outdated packages:');
        Object.entries(outdated).forEach(([name, info]) => {
            console.log(`  ${name}: ${info.current} → ${info.latest}`);
        });
        
        console.log('\nUnused packages:');
        unused.forEach(dep => {
            console.log(`  ${dep}`);
        });
        
        console.log(`\nTotal dependencies: ${Object.keys(this.packageJson.dependencies || {}).length}`);
        console.log(`Total dev dependencies: ${Object.keys(this.packageJson.devDependencies || {}).length}`);
    }
}

// Usage
const depManager = new DependencyManager();
depManager.generateReport();
```

## Publishing Packages

### Preparing for Publication

```javascript
// 1. Create .npmignore file
/*
node_modules/
*.log
.env
.DS_Store
coverage/
test/
src/
.git/
.gitignore
.eslintrc.js
webpack.config.js
*/

// 2. Set up package.json for publishing
{
  "name": "@username/package-name",  // Scoped package
  "version": "1.0.0",
  "description": "A useful package",
  "main": "lib/index.js",
  "module": "lib/index.mjs",
  "types": "lib/index.d.ts",
  "files": [
    "lib/",
    "README.md",
    "LICENSE"
  ],
  "keywords": ["utility", "helper"],
  "author": "Your Name <you@example.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/username/package-name.git"
  },
  "bugs": {
    "url": "https://github.com/username/package-name/issues"
  },
  "homepage": "https://github.com/username/package-name#readme",
  "engines": {
    "node": ">=14.0.0"
  },
  "scripts": {
    "build": "webpack",
    "test": "jest",
    "prepublishOnly": "npm test && npm run build"
  }
}
```

### Publishing Workflow

```bash
# 1. Login to npm
npm login

# 2. Test package locally
npm pack  # Creates .tgz file
npm install -g ./package-name-1.0.0.tgz  # Test installation

# 3. Publish package
npm publish

# 4. Publish scoped package
npm publish --access public

# 5. Update version and publish
npm version patch  # 1.0.0 → 1.0.1
npm version minor  # 1.0.1 → 1.1.0
npm version major  # 1.1.0 → 2.0.0

# 6. Publish with tag
npm publish --tag beta
npm publish --tag next

# 7. Unpublish (only within 72 hours)
npm unpublish package-name@1.0.0
```

### Publishing Script

```javascript
// scripts/publish.js
const { execSync } = require('child_process');
const fs = require('fs');

class Publisher {
    constructor() {
        this.packageJson = JSON.parse(fs.readFileSync('./package.json', 'utf8'));
    }
    
    async publish(versionType = 'patch') {
        try {
            console.log('🔍 Running pre-publish checks...');
            
            // Run tests
            console.log('Running tests...');
            execSync('npm test', { stdio: 'inherit' });
            
            // Run linting
            console.log('Running linter...');
            execSync('npm run lint', { stdio: 'inherit' });
            
            // Build package
            console.log('Building package...');
            execSync('npm run build', { stdio: 'inherit' });
            
            // Bump version
            console.log(`Bumping ${versionType} version...`);
            execSync(`npm version ${versionType}`, { stdio: 'inherit' });
            
            // Publish
            console.log('Publishing to npm...');
            execSync('npm publish', { stdio: 'inherit' });
            
            console.log('✅ Package published successfully!');
            
            // Push changes to git
            console.log('Pushing changes to git...');
            execSync('git push && git push --tags', { stdio: 'inherit' });
            
        } catch (error) {
            console.error('❌ Publish failed:', error.message);
            process.exit(1);
        }
    }
    
    async publishBeta() {
        try {
            // Bump prerelease version
            execSync('npm version prerelease --preid=beta', { stdio: 'inherit' });
            
            // Publish with beta tag
            execSync('npm publish --tag beta', { stdio: 'inherit' });
            
            console.log('✅ Beta version published!');
        } catch (error) {
            console.error('❌ Beta publish failed:', error.message);
            process.exit(1);
        }
    }
}

// Usage
const publisher = new Publisher();
const command = process.argv[2] || 'patch';

if (command === 'beta') {
    publisher.publishBeta();
} else {
    publisher.publish(command);
}
```

### Package Documentation

```markdown
# Package Name

Brief description of what your package does.

## Installation

```bash
npm install package-name
```

## Usage

```javascript
const packageName = require('package-name');

// Example usage
const result = packageName.doSomething();
```

## API

### `doSomething(param1, param2)`

Description of the function.

**Parameters:**
- `param1` (string): Description
- `param2` (number): Description

**Returns:** Description of return value

**Example:**
```javascript
const result = packageName.doSomething('hello', 42);
```

## License

MIT
```

## Alternative Package Managers

### Yarn

```bash
# Install Yarn
npm install -g yarn

# Initialize project
yarn init

# Install dependencies
yarn add express
yarn add --dev jest
yarn add --peer react

# Install from package.json
yarn install
yarn install --frozen-lockfile  # Production install

# Remove packages
yarn remove express

# Update packages
yarn upgrade
yarn upgrade express

# Run scripts
yarn start
yarn test

# Workspace management
yarn workspaces run build
```

### pnpm

```bash
# Install pnpm
npm install -g pnpm

# Initialize project
pnpm init

# Install dependencies
pnpm add express
pnpm add -D jest
pnpm add -P react

# Install from package.json
pnpm install
pnpm install --frozen-lockfile

# Remove packages
pnpm remove express

# Update packages
pnpm update
pnpm update express

# Run scripts
pnpm start
pnpm test

# Workspace management
pnpm -r run build
```

### Comparison

```javascript
// Performance comparison (approximate)
// Installation speed: pnpm > yarn > npm
// Disk usage: pnpm (symlinks) < yarn < npm
// Network usage: pnpm < yarn ≈ npm

// Lock file comparison
// npm: package-lock.json
// yarn: yarn.lock
// pnpm: pnpm-lock.yaml

// Workspace support
// npm: workspaces (v7+)
// yarn: workspaces (v1+)
// pnpm: workspaces (built-in)
```

## Module Resolution

### Node.js Module Resolution Algorithm

```javascript
// require(X) from module at path Y
// 1. If X is a core module, return the core module
// 2. If X begins with '/', return LOAD_AS_FILE(X)
// 3. If X begins with './' or '../', return LOAD_AS_FILE(Y + X)
// 4. Return LOAD_NODE_MODULES(X, dirname(Y))

// LOAD_AS_FILE(X)
// 1. If X is a file, load X as JavaScript text
// 2. If X.js is a file, load X.js as JavaScript text
// 3. If X.json is a file, parse X.json to a JavaScript Object
// 4. If X.node is a file, load X.node as binary addon

// LOAD_AS_DIRECTORY(X)
// 1. If X/package.json is a file, parse and use "main" field
// 2. If X/index.js is a file, load X/index.js as JavaScript text
// 3. If X/index.json is a file, parse X/index.json to JavaScript object
// 4. If X/index.node is a file, load X/index.node as binary addon

// Module resolution debugging
const Module = require('module');

// Override require.resolve to add logging
const originalResolve = Module._resolveFilename;
Module._resolveFilename = function(request, parent, isMain) {
    console.log(`Resolving: ${request} from ${parent.filename}`);
    return originalResolve.call(this, request, parent, isMain);
};

// Test resolution
require('express');
require('./local-module');
```

### Custom Module Loader

```javascript
// custom-loader.js
const Module = require('module');
const fs = require('fs');
const path = require('path');

// Custom file extension handler
Module._extensions['.config'] = function(module, filename) {
    const content = fs.readFileSync(filename, 'utf8');
    const config = JSON.parse(content);
    
    // Transform config
    config.loaded = true;
    config.loadedAt = new Date().toISOString();
    
    module.exports = config;
};

// Custom require hook
const originalRequire = Module.prototype.require;
Module.prototype.require = function(id) {
    console.log(`Loading module: ${id}`);
    const start = Date.now();
    
    const result = originalRequire.call(this, id);
    
    const duration = Date.now() - start;
    console.log(`Loaded ${id} in ${duration}ms`);
    
    return result;
};

// Usage
require('./custom-loader');
const config = require('./app.config'); // Will use custom loader
```

## Best Practices

### Project Structure

```
my-node-app/
├── src/                    # Source code
│   ├── controllers/        # Route controllers
│   ├── models/            # Data models
│   ├── services/          # Business logic
│   ├── utils/             # Utility functions
│   ├── middleware/        # Express middleware
│   └── config/            # Configuration files
├── tests/                 # Test files
│   ├── unit/              # Unit tests
│   ├── integration/       # Integration tests
│   └── fixtures/          # Test data
├── docs/                  # Documentation
├── scripts/               # Build/deployment scripts
├── public/                # Static files (if web app)
├── lib/                   # Compiled/built files
├── bin/                   # Executable files
├── .env.example           # Environment variables template
├── .gitignore             # Git ignore rules
├── .npmignore             # npm ignore rules
├── package.json           # Package configuration
├── package-lock.json      # Dependency lock file
├── README.md              # Project documentation
└── LICENSE                # License file
```

### Module Design Patterns

```javascript
// 1. Single Responsibility Principle
// Good: Each module has one responsibility
// user-service.js
class UserService {
    async createUser(userData) { /* ... */ }
    async getUserById(id) { /* ... */ }
    async updateUser(id, userData) { /* ... */ }
    async deleteUser(id) { /* ... */ }
}

// email-service.js
class EmailService {
    async sendEmail(to, subject, body) { /* ... */ }
    async sendWelcomeEmail(user) { /* ... */ }
}

// 2. Dependency Injection
// config.js
module.exports = {
    database: {
        url: process.env.DATABASE_URL,
        options: { useNewUrlParser: true }
    },
    email: {
        apiKey: process.env.EMAIL_API_KEY,
        from: process.env.EMAIL_FROM
    }
};

// user-service.js
class UserService {
    constructor(database, emailService, config) {
        this.database = database;
        this.emailService = emailService;
        this.config = config;
    }
    
    async createUser(userData) {
        const user = await this.database.users.create(userData);
        await this.emailService.sendWelcomeEmail(user);
        return user;
    }
}

module.exports = UserService;

// app.js
const UserService = require('./user-service');
const EmailService = require('./email-service');
const Database = require('./database');
const config = require('./config');

const database = new Database(config.database);
const emailService = new EmailService(config.email);
const userService = new UserService(database, emailService, config);

// 3. Factory Pattern for Module Creation
// service-factory.js
class ServiceFactory {
    constructor(config) {
        this.config = config;
        this.services = new Map();
    }
    
    createService(serviceName) {
        if (this.services.has(serviceName)) {
            return this.services.get(serviceName);
        }
        
        let service;
        switch (serviceName) {
            case 'user':
                const UserService = require('./services/user-service');
                service = new UserService(this.config);
                break;
            case 'email':
                const EmailService = require('./services/email-service');
                service = new EmailService(this.config);
                break;
            default:
                throw new Error(`Unknown service: ${serviceName}`);
        }
        
        this.services.set(serviceName, service);
        return service;
    }
}

module.exports = ServiceFactory;

// 4. Plugin Architecture
// plugin-manager.js
class PluginManager {
    constructor() {
        this.plugins = new Map();
    }
    
    async loadPlugin(name, options = {}) {
        try {
            const PluginClass = require(`./plugins/${name}`);
            const plugin = new PluginClass(options);
            
            if (typeof plugin.init === 'function') {
                await plugin.init();
            }
            
            this.plugins.set(name, plugin);
            console.log(`Plugin ${name} loaded successfully`);
        } catch (error) {
            console.error(`Failed to load plugin ${name}:`, error);
        }
    }
    
    getPlugin(name) {
        return this.plugins.get(name);
    }
    
    async unloadPlugin(name) {
        const plugin = this.plugins.get(name);
        if (plugin && typeof plugin.destroy === 'function') {
            await plugin.destroy();
        }
        this.plugins.delete(name);
    }
}

// plugins/auth-plugin.js
class AuthPlugin {
    constructor(options) {
        this.options = options;
    }
    
    async init() {
        console.log('Auth plugin initialized');
    }
    
    authenticate(token) {
        // Authentication logic
    }
    
    async destroy() {
        console.log('Auth plugin destroyed');
    }
}

module.exports = AuthPlugin;
```

### Error Handling Best Practices

```javascript
// 1. Custom Error Classes
// errors/base-error.js
class BaseError extends Error {
    constructor(message, statusCode = 500, isOperational = true) {
        super(message);
        this.name = this.constructor.name;
        this.statusCode = statusCode;
        this.isOperational = isOperational;
        
        Error.captureStackTrace(this, this.constructor);
    }
}

// errors/validation-error.js
class ValidationError extends BaseError {
    constructor(message, field) {
        super(message, 400);
        this.field = field;
    }
}

// errors/not-found-error.js
class NotFoundError extends BaseError {
    constructor(resource) {
        super(`${resource} not found`, 404);
        this.resource = resource;
    }
}

// 2. Centralized Error Handler
// middleware/error-handler.js
function errorHandler(err, req, res, next) {
    // Log error
    console.error(err.stack);
    
    // Operational errors - send to client
    if (err.isOperational) {
        return res.status(err.statusCode).json({
            error: {
                message: err.message,
                ...(err.field && { field: err.field })
            }
        });
    }
    
    // Programming errors - don't leak details
    res.status(500).json({
        error: {
            message: 'Internal server error'
        }
    });
}

// 3. Async Error Wrapper
function asyncHandler(fn) {
    return (req, res, next) => {
        Promise.resolve(fn(req, res, next)).catch(next);
    };
}

// Usage
app.get('/users/:id', asyncHandler(async (req, res) => {
    const user = await userService.getUserById(req.params.id);
    if (!user) {
        throw new NotFoundError('User');
    }
    res.json(user);
}));

app.use(errorHandler);
```

### Performance Best Practices

```javascript
// 1. Module Loading Optimization
// Lazy loading
function getHeavyModule() {
    if (!this._heavyModule) {
        this._heavyModule = require('./heavy-module');
    }
    return this._heavyModule;
}

// Conditional loading
if (process.env.NODE_ENV === 'development') {
    require('./dev-tools');
}

// 2. Caching Strategies
// Simple in-memory cache
class Cache {
    constructor(ttl = 60000) {
        this.cache = new Map();
        this.ttl = ttl;
    }
    
    set(key, value) {
        this.cache.set(key, {
            value,
            expires: Date.now() + this.ttl
        });
    }
    
    get(key) {
        const item = this.cache.get(key);
        if (!item) return null;
        
        if (Date.now() > item.expires) {
            this.cache.delete(key);
            return null;
        }
        
        return item.value;
    }
    
    clear() {
        this.cache.clear();
    }
}

// 3. Resource Management
class ResourceManager {
    constructor() {
        this.resources = new Set();
        
        // Cleanup on process exit
        process.on('SIGINT', () => this.cleanup());
        process.on('SIGTERM', () => this.cleanup());
    }
    
    addResource(resource) {
        this.resources.add(resource);
    }
    
    async cleanup() {
        console.log('Cleaning up resources...');
        
        for (const resource of this.resources) {
            try {
                if (typeof resource.close === 'function') {
                    await resource.close();
                }
            } catch (error) {
                console.error('Error closing resource:', error);
            }
        }
        
        this.resources.clear();
    }
}
```

---

This completes the comprehensive Node.js Modules and Package Management section. The content covers CommonJS and ES6 modules, built-in modules, npm usage, package.json configuration, dependency management, publishing packages, alternative package managers, module resolution, and best practices.
