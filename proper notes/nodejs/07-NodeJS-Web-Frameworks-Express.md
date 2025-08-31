# Node.js Web Frameworks - Express.js Complete Guide

## Table of Contents
1. [Introduction to Express.js](#introduction-to-expressjs)
2. [Express.js Setup and Basic Server](#expressjs-setup-and-basic-server)
3. [Routing in Express](#routing-in-express)
4. [Middleware Deep Dive](#middleware-deep-dive)
5. [Request and Response Objects](#request-and-response-objects)
6. [Template Engines](#template-engines)
7. [Static Files and Assets](#static-files-and-assets)
8. [Error Handling](#error-handling)
9. [Security Best Practices](#security-best-practices)
10. [Testing Express Applications](#testing-express-applications)
11. [Performance Optimization](#performance-optimization)
12. [Other Node.js Frameworks](#other-nodejs-frameworks)

## Introduction to Express.js

Express.js is the most popular web framework for Node.js, providing a minimal and flexible set of features for web and mobile applications. It's built on top of Node.js's built-in HTTP module and provides a robust set of features for building single-page, multi-page, and hybrid web applications.

### Why Express.js?

- **Minimalist**: Provides essential features without bloat
- **Flexible**: Unopinionated framework that doesn't force architectural decisions
- **Mature**: Battle-tested with a large ecosystem
- **Performance**: Fast and lightweight
- **Community**: Extensive community support and middleware ecosystem

### Express.js Architecture

Express.js follows a middleware-based architecture where requests flow through a series of middleware functions before reaching route handlers.

```javascript
// Basic Express.js flow
// Request → Middleware 1 → Middleware 2 → Route Handler → Response
```

## Express.js Setup and Basic Server

### Installation and Project Setup

```bash
# Initialize new Node.js project
mkdir express-app
cd express-app
npm init -y

# Install Express.js
npm install express

# Install development dependencies
npm install --save-dev nodemon

# Update package.json scripts
```

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "node test.js"
  }
}
```

### Basic Express Server

```javascript
// server.js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// Basic middleware
app.use(express.json()); // Parse JSON bodies
app.use(express.urlencoded({ extended: true })); // Parse URL-encoded bodies

// Basic route
app.get('/', (req, res) => {
    res.json({
        message: 'Welcome to Express.js Server!',
        timestamp: new Date().toISOString(),
        version: '1.0.0'
    });
});

// Health check endpoint
app.get('/health', (req, res) => {
    res.status(200).json({
        status: 'healthy',
        uptime: process.uptime(),
        memory: process.memoryUsage(),
        timestamp: new Date().toISOString()
    });
});

// Start server
app.listen(PORT, () => {
    console.log(`🚀 Server running on http://localhost:${PORT}`);
    console.log(`📊 Health check: http://localhost:${PORT}/health`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('SIGTERM received, shutting down gracefully');
    process.exit(0);
});

process.on('SIGINT', () => {
    console.log('SIGINT received, shutting down gracefully');
    process.exit(0);
});
```

### Express Application Structure

```javascript
// app.js - Application configuration
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
    credentials: true
}));

// Logging middleware
app.use(morgan('combined'));

// Body parsing middleware
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Custom middleware
app.use((req, res, next) => {
    req.requestTime = new Date().toISOString();
    req.requestId = Math.random().toString(36).substr(2, 9);
    next();
});

module.exports = app;

// server.js - Server startup
const app = require('./app');
const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

## Routing in Express

### Basic Routing

```javascript
const express = require('express');
const app = express();

// HTTP Methods
app.get('/users', (req, res) => {
    res.json({ message: 'Get all users' });
});

app.post('/users', (req, res) => {
    res.json({ message: 'Create new user', data: req.body });
});

app.put('/users/:id', (req, res) => {
    res.json({ message: `Update user ${req.params.id}`, data: req.body });
});

app.delete('/users/:id', (req, res) => {
    res.json({ message: `Delete user ${req.params.id}` });
});

// Multiple HTTP methods
app.route('/books')
    .get((req, res) => {
        res.json({ message: 'Get all books' });
    })
    .post((req, res) => {
        res.json({ message: 'Create new book' });
    });

// All HTTP methods
app.all('/secret', (req, res) => {
    res.json({ message: 'Accessing the secret section' });
});
```

### Route Parameters

```javascript
// Route parameters
app.get('/users/:id', (req, res) => {
    const userId = req.params.id;
    res.json({ userId, message: `User ID: ${userId}` });
});

// Multiple parameters
app.get('/users/:userId/posts/:postId', (req, res) => {
    const { userId, postId } = req.params;
    res.json({ userId, postId });
});

// Optional parameters
app.get('/posts/:year/:month?', (req, res) => {
    const { year, month } = req.params;
    res.json({ year, month: month || 'all' });
});

// Wildcard parameters
app.get('/files/*', (req, res) => {
    const filePath = req.params[0];
    res.json({ filePath });
});

// Parameter validation middleware
app.param('id', (req, res, next, id) => {
    // Validate ID format
    if (!/^\d+$/.test(id)) {
        return res.status(400).json({ error: 'Invalid ID format' });
    }
    
    // Add validation logic
    req.validatedId = parseInt(id);
    next();
});

app.get('/validated/:id', (req, res) => {
    res.json({ validatedId: req.validatedId });
});
```

### Query Parameters

```javascript
// Query parameters
app.get('/search', (req, res) => {
    const { q, page = 1, limit = 10, sort = 'name' } = req.query;
    
    res.json({
        query: q,
        pagination: {
            page: parseInt(page),
            limit: parseInt(limit)
        },
        sort,
        filters: req.query
    });
});

// Advanced query handling
app.get('/products', (req, res) => {
    const {
        category,
        minPrice,
        maxPrice,
        inStock,
        tags,
        page = 1,
        limit = 20,
        sort = 'name',
        order = 'asc'
    } = req.query;
    
    // Build query object
    const query = {};
    
    if (category) query.category = category;
    if (minPrice || maxPrice) {
        query.price = {};
        if (minPrice) query.price.$gte = parseFloat(minPrice);
        if (maxPrice) query.price.$lte = parseFloat(maxPrice);
    }
    if (inStock !== undefined) query.inStock = inStock === 'true';
    if (tags) query.tags = { $in: tags.split(',') };
    
    res.json({
        query,
        pagination: { page: parseInt(page), limit: parseInt(limit) },
        sort: { [sort]: order === 'desc' ? -1 : 1 }
    });
});
```

### Express Router

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();

// Middleware specific to this router
router.use((req, res, next) => {
    console.log('User routes middleware');
    next();
});

// Define routes
router.get('/', (req, res) => {
    res.json({ message: 'Get all users' });
});

router.get('/:id', (req, res) => {
    res.json({ message: `Get user ${req.params.id}` });
});

router.post('/', (req, res) => {
    res.json({ message: 'Create user', data: req.body });
});

router.put('/:id', (req, res) => {
    res.json({ message: `Update user ${req.params.id}`, data: req.body });
});

router.delete('/:id', (req, res) => {
    res.json({ message: `Delete user ${req.params.id}` });
});

module.exports = router;

// app.js - Using the router
const userRoutes = require('./routes/users');
app.use('/api/users', userRoutes);

// routes/index.js - Main router
const express = require('express');
const router = express.Router();

const userRoutes = require('./users');
const postRoutes = require('./posts');
const authRoutes = require('./auth');

// Mount sub-routers
router.use('/users', userRoutes);
router.use('/posts', postRoutes);
router.use('/auth', authRoutes);

// API info route
router.get('/', (req, res) => {
    res.json({
        message: 'API v1.0',
        endpoints: {
            users: '/api/users',
            posts: '/api/posts',
            auth: '/api/auth'
        }
    });
});

module.exports = router;
```

### Advanced Routing Patterns

```javascript
// Route patterns with regex
app.get(/.*fly$/, (req, res) => {
    res.json({ message: 'Ends with fly' });
});

// String patterns
app.get('/ab?cd', (req, res) => {
    res.json({ message: 'Matches acd or abcd' });
});

app.get('/ab+cd', (req, res) => {
    res.json({ message: 'Matches abcd, abbcd, abbbcd, etc.' });
});

app.get('/ab*cd', (req, res) => {
    res.json({ message: 'Matches abcd, abxcd, abRANDOMcd, etc.' });
});

// Route handlers array
const middleware1 = (req, res, next) => {
    console.log('Middleware 1');
    next();
};

const middleware2 = (req, res, next) => {
    console.log('Middleware 2');
    next();
};

app.get('/multiple', [middleware1, middleware2], (req, res) => {
    res.json({ message: 'Multiple middlewares' });
});

// Conditional routing
app.get('/conditional/:type', (req, res, next) => {
    if (req.params.type === 'special') {
        res.json({ message: 'Special handling' });
    } else {
        next(); // Pass to next route
    }
});

app.get('/conditional/:type', (req, res) => {
    res.json({ message: 'Default handling', type: req.params.type });
});
```

## Middleware Deep Dive

### Understanding Middleware

Middleware functions are functions that have access to the request object (req), the response object (res), and the next middleware function in the application's request-response cycle.

```javascript
// Basic middleware structure
const myMiddleware = (req, res, next) => {
    // Middleware logic here
    console.log('Middleware executed');
    
    // Call next() to pass control to the next middleware
    next();
    
    // Or send a response and end the cycle
    // res.json({ message: 'Response from middleware' });
    
    // Or pass an error to error handling middleware
    // next(new Error('Something went wrong'));
};

// Using middleware
app.use(myMiddleware);
```

### Types of Middleware

#### Application-level Middleware

```javascript
// Application-level middleware
app.use((req, res, next) => {
    console.log('Time:', Date.now());
    next();
});

// Path-specific middleware
app.use('/users', (req, res, next) => {
    console.log('User routes middleware');
    next();
});

// Method and path specific
app.get('/special', (req, res, next) => {
    console.log('Special GET middleware');
    next();
}, (req, res) => {
    res.json({ message: 'Special route' });
});
```

#### Router-level Middleware

```javascript
const router = express.Router();

// Router-level middleware
router.use((req, res, next) => {
    console.log('Router middleware');
    next();
});

router.use('/protected', (req, res, next) => {
    // Authentication check
    const token = req.headers.authorization;
    if (!token) {
        return res.status(401).json({ error: 'No token provided' });
    }
    next();
});

app.use('/api', router);
```

#### Built-in Middleware

```javascript
// Built-in middleware
app.use(express.json()); // Parse JSON bodies
app.use(express.urlencoded({ extended: true })); // Parse URL-encoded bodies
app.use(express.static('public')); // Serve static files
app.use(express.raw()); // Parse raw bodies
app.use(express.text()); // Parse text bodies
```

#### Third-party Middleware

```javascript
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const compression = require('compression');
const rateLimit = require('express-rate-limit');

// CORS middleware
app.use(cors({
    origin: ['http://localhost:3000', 'https://myapp.com'],
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization']
}));

// Security middleware
app.use(helmet({
    contentSecurityPolicy: {
        directives: {
            defaultSrc: ["'self'"],
            styleSrc: ["'self'", "'unsafe-inline'"],
            scriptSrc: ["'self'"],
            imgSrc: ["'self'", "data:", "https:"]
        }
    }
}));

// Logging middleware
app.use(morgan('combined', {
    stream: {
        write: (message) => {
            console.log(message.trim());
        }
    }
}));

// Compression middleware
app.use(compression({
    filter: (req, res) => {
        if (req.headers['x-no-compression']) {
            return false;
        }
        return compression.filter(req, res);
    }
}));

// Rate limiting
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
    message: 'Too many requests from this IP',
    standardHeaders: true,
    legacyHeaders: false
});

app.use('/api/', limiter);
```

### Custom Middleware Examples

```javascript
// Logging middleware
const requestLogger = (req, res, next) => {
    const start = Date.now();
    
    res.on('finish', () => {
        const duration = Date.now() - start;
        console.log(`${req.method} ${req.originalUrl} - ${res.statusCode} - ${duration}ms`);
    });
    
    next();
};

// Authentication middleware
const authenticate = (req, res, next) => {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
        return res.status(401).json({ error: 'No token provided' });
    }
    
    try {
        // Verify JWT token (pseudo-code)
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({ error: 'Invalid token' });
    }
};

// Authorization middleware
const authorize = (roles = []) => {
    return (req, res, next) => {
        if (!req.user) {
            return res.status(401).json({ error: 'Not authenticated' });
        }
        
        if (roles.length && !roles.includes(req.user.role)) {
            return res.status(403).json({ error: 'Insufficient permissions' });
        }
        
        next();
    };
};

// Validation middleware
const validateBody = (schema) => {
    return (req, res, next) => {
        const { error, value } = schema.validate(req.body);
        
        if (error) {
            return res.status(400).json({
                error: 'Validation error',
                details: error.details.map(d => d.message)
            });
        }
        
        req.body = value; // Use validated/sanitized data
        next();
    };
};

// Cache middleware
const cache = (duration = 300) => {
    const cache = new Map();
    
    return (req, res, next) => {
        const key = req.originalUrl;
        const cached = cache.get(key);
        
        if (cached && Date.now() - cached.timestamp < duration * 1000) {
            return res.json(cached.data);
        }
        
        const originalSend = res.json;
        res.json = function(data) {
            cache.set(key, { data, timestamp: Date.now() });
            originalSend.call(this, data);
        };
        
        next();
    };
};

// Usage examples
app.use(requestLogger);

app.get('/protected', authenticate, (req, res) => {
    res.json({ message: 'Protected route', user: req.user });
});

app.get('/admin', authenticate, authorize(['admin']), (req, res) => {
    res.json({ message: 'Admin only route' });
});

app.post('/users', validateBody(userSchema), (req, res) => {
    res.json({ message: 'User created', data: req.body });
});

app.get('/cached-data', cache(600), (req, res) => {
    // This response will be cached for 10 minutes
    res.json({ data: 'This is cached data', timestamp: new Date() });
});
```

### Error Handling Middleware

```javascript
// Error handling middleware (must have 4 parameters)
const errorHandler = (err, req, res, next) => {
    console.error('Error:', err);
    
    // Default error
    let error = {
        message: 'Internal Server Error',
        status: 500
    };
    
    // Handle different error types
    if (err.name === 'ValidationError') {
        error.message = 'Validation Error';
        error.status = 400;
        error.details = err.details;
    } else if (err.name === 'UnauthorizedError') {
        error.message = 'Unauthorized';
        error.status = 401;
    } else if (err.code === 11000) {
        error.message = 'Duplicate field value';
        error.status = 400;
    } else if (err.name === 'CastError') {
        error.message = 'Invalid ID format';
        error.status = 400;
    }
    
    // Send error response
    res.status(error.status).json({
        error: error.message,
        ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
        ...(error.details && { details: error.details })
    });
};

// 404 handler (should be last)
const notFound = (req, res) => {
    res.status(404).json({
        error: 'Route not found',
        path: req.originalUrl,
        method: req.method
    });
};

// Use error handling middleware
app.use(errorHandler);
app.use(notFound);

// Async error wrapper
const asyncHandler = (fn) => (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
};

// Usage with async routes
app.get('/async-route', asyncHandler(async (req, res) => {
    const data = await someAsyncOperation();
    res.json(data);
}));
```

## Request and Response Objects

### Request Object (req)

```javascript
app.get('/request-demo', (req, res) => {
    // Request properties
    const requestInfo = {
        // Basic properties
        method: req.method,
        url: req.url,
        originalUrl: req.originalUrl,
        path: req.path,
        protocol: req.protocol,
        secure: req.secure,
        
        // Headers
        headers: req.headers,
        userAgent: req.get('User-Agent'),
        contentType: req.get('Content-Type'),
        
        // Parameters
        params: req.params,
        query: req.query,
        body: req.body,
        
        // Network info
        ip: req.ip,
        ips: req.ips,
        hostname: req.hostname,
        
        // Other useful properties
        fresh: req.fresh,
        stale: req.stale,
        xhr: req.xhr,
        
        // Custom properties (added by middleware)
        user: req.user,
        requestTime: req.requestTime
    };
    
    res.json(requestInfo);
});

// Request methods
app.post('/request-methods', (req, res) => {
    // Check content type
    if (req.is('json')) {
        console.log('Request contains JSON');
    }
    
    if (req.is('application/json')) {
        console.log('Request is JSON');
    }
    
    // Check if request accepts certain content types
    if (req.accepts('json')) {
        res.json({ message: 'JSON response' });
    } else if (req.accepts('html')) {
        res.send('<h1>HTML response</h1>');
    } else {
        res.type('txt').send('Text response');
    }
});
```

### Response Object (res)

```javascript
app.get('/response-demo', (req, res) => {
    // Set status code
    res.status(200);
    
    // Set headers
    res.set('X-Custom-Header', 'Custom Value');
    res.set({
        'X-Another-Header': 'Another Value',
        'X-Third-Header': 'Third Value'
    });
    
    // Set content type
    res.type('json');
    // or
    res.contentType('application/json');
    
    // Send response
    res.json({ message: 'Response demo' });
});

// Different response methods
app.get('/response-methods/:type', (req, res) => {
    const { type } = req.params;
    
    switch (type) {
        case 'json':
            res.json({ message: 'JSON response', data: [1, 2, 3] });
            break;
            
        case 'text':
            res.text('Plain text response');
            break;
            
        case 'html':
            res.send('<h1>HTML Response</h1><p>This is HTML</p>');
            break;
            
        case 'redirect':
            res.redirect('/response-methods/json');
            break;
            
        case 'download':
            res.download('./files/sample.pdf', 'downloaded-file.pdf');
            break;
            
        case 'attachment':
            res.attachment('data.csv');
            res.send('id,name,email\n1,John,john@example.com');
            break;
            
        case 'cookie':
            res.cookie('sessionId', '12345', {
                maxAge: 900000,
                httpOnly: true,
                secure: process.env.NODE_ENV === 'production'
            });
            res.json({ message: 'Cookie set' });
            break;
            
        case 'clear-cookie':
            res.clearCookie('sessionId');
            res.json({ message: 'Cookie cleared' });
            break;
            
        default:
            res.status(400).json({ error: 'Invalid response type' });
    }
});

// Response chaining
app.get('/chaining', (req, res) => {
    res
        .status(201)
        .set('X-Custom', 'Value')
        .cookie('test', 'value')
        .json({ message: 'Chained response' });
});

// Conditional responses
app.get('/conditional', (req, res) => {
    const { format } = req.query;
    
    const data = { message: 'Hello World', timestamp: new Date() };
    
    res.format({
        'text/plain': () => {
            res.send(`${data.message} at ${data.timestamp}`);
        },
        
        'text/html': () => {
            res.send(`<h1>${data.message}</h1><p>Time: ${data.timestamp}</p>`);
        },
        
        'application/json': () => {
            res.json(data);
        },
        
        'default': () => {
            res.status(406).send('Not Acceptable');
        }
    });
});
```

### File Handling

```javascript
const multer = require('multer');
const path = require('path');

// Configure multer for file uploads
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, 'uploads/');
    },
    filename: (req, file, cb) => {
        const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
        cb(null, file.fieldname + '-' + uniqueSuffix + path.extname(file.originalname));
    }
});

const upload = multer({
    storage: storage,
    limits: {
        fileSize: 5 * 1024 * 1024 // 5MB limit
    },
    fileFilter: (req, file, cb) => {
        const allowedTypes = /jpeg|jpg|png|gif/;
        const extname = allowedTypes.test(path.extname(file.originalname).toLowerCase());
        const mimetype = allowedTypes.test(file.mimetype);
        
        if (mimetype && extname) {
            return cb(null, true);
        } else {
            cb(new Error('Only images are allowed'));
        }
    }
});

// File upload endpoints
app.post('/upload/single', upload.single('image'), (req, res) => {
    if (!req.file) {
        return res.status(400).json({ error: 'No file uploaded' });
    }
    
    res.json({
        message: 'File uploaded successfully',
        file: {
            filename: req.file.filename,
            originalName: req.file.originalname,
            size: req.file.size,
            mimetype: req.file.mimetype
        }
    });
});

app.post('/upload/multiple', upload.array('images', 5), (req, res) => {
    if (!req.files || req.files.length === 0) {
        return res.status(400).json({ error: 'No files uploaded' });
    }
    
    res.json({
        message: 'Files uploaded successfully',
        files: req.files.map(file => ({
            filename: file.filename,
            originalName: file.originalname,
            size: file.size,
            mimetype: file.mimetype
        }))
    });
});

// File download
app.get('/download/:filename', (req, res) => {
    const filename = req.params.filename;
    const filepath = path.join(__dirname, 'uploads', filename);
    
    // Check if file exists
    if (!fs.existsSync(filepath)) {
        return res.status(404).json({ error: 'File not found' });
    }
    
    res.download(filepath, (err) => {
        if (err) {
            console.error('Download error:', err);
            res.status(500).json({ error: 'Download failed' });
        }
    });
});

// Serve files
app.get('/files/:filename', (req, res) => {
    const filename = req.params.filename;
    const filepath = path.join(__dirname, 'uploads', filename);
    
    res.sendFile(filepath, (err) => {
        if (err) {
            res.status(404).json({ error: 'File not found' });
        }
    });
});
```

## Template Engines

### Setting up Template Engines

```javascript
// Using EJS
app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));

// Using Handlebars
const exphbs = require('express-handlebars');
app.engine('handlebars', exphbs());
app.set('view engine', 'handlebars');

// Using Pug
app.set('view engine', 'pug');
```

### EJS Templates

```html
<!-- views/layout.ejs -->
<!DOCTYPE html>
<html>
<head>
    <title><%= title %></title>
    <link rel="stylesheet" href="/css/style.css">
</head>
<body>
    <header>
        <nav>
            <a href="/">Home</a>
            <a href="/users">Users</a>
            <% if (user) { %>
                <a href="/profile">Profile</a>
                <a href="/logout">Logout</a>
            <% } else { %>
                <a href="/login">Login</a>
            <% } %>
        </nav>
    </header>
    
    <main>
        <%- body %>
    </main>
    
    <footer>
        <p>&copy; 2024 My App</p>
    </footer>
</body>
</html>

<!-- views/users.ejs -->
<h1>Users List</h1>

<% if (users && users.length > 0) { %>
    <div class="users-grid">
        <% users.forEach(user => { %>
            <div class="user-card">
                <h3><%= user.name %></h3>
                <p>Email: <%= user.email %></p>
                <p>Role: <%= user.role %></p>
                <% if (user.avatar) { %>
                    <img src="/uploads/<%= user.avatar %>" alt="<%= user.name %>">
                <% } %>
                <div class="actions">
                    <a href="/users/<%= user.id %>">View</a>
                    <a href="/users/<%= user.id %>/edit">Edit</a>
                </div>
            </div>
        <% }); %>
    </div>
<% } else { %>
    <p>No users found.</p>
<% } %>

<a href="/users/new" class="btn btn-primary">Add New User</a>
```

```javascript
// Using templates in routes
app.get('/', (req, res) => {
    res.render('index', {
        title: 'Home Page',
        user: req.user,
        message: 'Welcome to our application!'
    });
});

app.get('/users', async (req, res) => {
    try {
        const users = await User.find();
        res.render('users', {
            title: 'Users',
            users: users,
            user: req.user
        });
    } catch (error) {
        res.status(500).render('error', {
            title: 'Error',
            error: error.message
        });
    }
});

// Template helpers/filters
app.locals.formatDate = (date) => {
    return new Date(date).toLocaleDateString();
};

app.locals.truncate = (str, length = 100) => {
    return str.length > length ? str.substring(0, length) + '...' : str;
};
```

## Static Files and Assets

### Serving Static Files

```javascript
// Basic static file serving
app.use(express.static('public'));

// Multiple static directories
app.use(express.static('public'));
app.use(express.static('files'));
app.use(express.static('uploads'));

// Virtual path prefix
app.use('/static', express.static('public'));
app.use('/assets', express.static('assets'));

// Static file options
app.use('/static', express.static('public', {
    maxAge: '1d', // Cache for 1 day
    etag: false,
    index: false, // Disable directory indexing
    dotfiles: 'ignore'
}));
```

### Advanced Static File Handling

```javascript
const path = require('path');
const fs = require('fs');

// Custom static file middleware
const serveStatic = (directory, options = {}) => {
    return (req, res, next) => {
        const filePath = path.join(directory, req.path);
        
        // Security check - prevent directory traversal
        if (!filePath.startsWith(path.resolve(directory))) {
            return res.status(403).json({ error: 'Access denied' });
        }
        
        fs.stat(filePath, (err, stats) => {
            if (err || !stats.isFile()) {
                return next();
            }
            
            // Set cache headers
            if (options.maxAge) {
                res.set('Cache-Control', `max-age=${options.maxAge}`);
            }
            
            // Set content type
            const ext = path.extname(filePath);
            const contentType = getContentType(ext);
            res.set('Content-Type', contentType);
            
            // Stream file
            const stream = fs.createReadStream(filePath);
            stream.pipe(res);
        });
    };
};

function getContentType(ext) {
    const types = {
        '.html': 'text/html',
        '.css': 'text/css',
        '.js': 'text/javascript',
        '.json': 'application/json',
        '.png': 'image/png',
        '.jpg': 'image/jpeg',
        '.gif': 'image/gif',
        '.svg': 'image/svg+xml'
    };
    return types[ext] || 'application/octet-stream';
}

// Asset pipeline with compression
const compression = require('compression');
app.use(compression({
    filter: (req, res) => {
        if (req.headers['x-no-compression']) {
            return false;
        }
        return compression.filter(req, res);
    }
}));

// Serve compressed assets
app.use('/assets', (req, res, next) => {
    const acceptsGzip = req.headers['accept-encoding']?.includes('gzip');
    
    if (acceptsGzip) {
        const gzipPath = req.path + '.gz';
        const fullGzipPath = path.join('public/assets', gzipPath);
        
        if (fs.existsSync(fullGzipPath)) {
            res.set('Content-Encoding', 'gzip');
            req.url = gzipPath;
        }
    }
    
    next();
}, express.static('public/assets'));
```

---

This covers the comprehensive Express.js framework guide. The content includes setup, routing, middleware, request/response handling, templates, static files, and advanced patterns. Would you like me to continue with the remaining sections (Error Handling, Security, Testing, Performance, and Other Frameworks)?
