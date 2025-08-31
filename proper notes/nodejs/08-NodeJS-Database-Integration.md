# Node.js Database Integration - Complete Guide

## Table of Contents
1. [Database Overview](#database-overview)
2. [MongoDB with Mongoose](#mongodb-with-mongoose)
3. [SQL Databases with Sequelize](#sql-databases-with-sequelize)
4. [Native Database Drivers](#native-database-drivers)
5. [Database Connection Management](#database-connection-management)
6. [Query Builders and ORMs](#query-builders-and-orms)
7. [Database Migrations and Seeds](#database-migrations-and-seeds)
8. [Caching Strategies](#caching-strategies)
9. [Database Security](#database-security)
10. [Performance Optimization](#performance-optimization)
11. [Testing with Databases](#testing-with-databases)

## Database Overview

Node.js can work with virtually any database system. The choice depends on your application requirements:

### Database Types

#### NoSQL Databases
- **MongoDB**: Document-based, flexible schema
- **Redis**: In-memory key-value store
- **Cassandra**: Wide-column store for big data
- **CouchDB**: Document-oriented with HTTP API

#### SQL Databases
- **PostgreSQL**: Advanced relational database
- **MySQL**: Popular relational database
- **SQLite**: Lightweight file-based database
- **SQL Server**: Microsoft's enterprise database

#### NewSQL Databases
- **CockroachDB**: Distributed SQL database
- **TiDB**: Hybrid transactional/analytical processing

## MongoDB with Mongoose

Mongoose is an Object Document Mapping (ODM) library for MongoDB and Node.js.

### Setup and Connection

```javascript
const mongoose = require('mongoose');

// Database configuration
const dbConfig = {
    development: {
        url: 'mongodb://localhost:27017/myapp_dev',
        options: {
            useNewUrlParser: true,
            useUnifiedTopology: true
        }
    },
    production: {
        url: process.env.MONGODB_URI,
        options: {
            useNewUrlParser: true,
            useUnifiedTopology: true,
            maxPoolSize: 10,
            serverSelectionTimeoutMS: 5000,
            socketTimeoutMS: 45000
        }
    }
};

class DatabaseConnection {
    constructor() {
        this.connection = null;
        this.isConnected = false;
    }
    
    async connect(environment = 'development') {
        try {
            const config = dbConfig[environment];
            
            this.connection = await mongoose.connect(config.url, config.options);
            this.isConnected = true;
            
            console.log(`✅ Connected to MongoDB (${environment})`);
            
            // Connection event handlers
            mongoose.connection.on('error', (err) => {
                console.error('❌ MongoDB connection error:', err);
            });
            
            mongoose.connection.on('disconnected', () => {
                console.log('📡 MongoDB disconnected');
                this.isConnected = false;
            });
            
            mongoose.connection.on('reconnected', () => {
                console.log('🔄 MongoDB reconnected');
                this.isConnected = true;
            });
            
            return this.connection;
        } catch (error) {
            console.error('❌ Failed to connect to MongoDB:', error);
            throw error;
        }
    }
    
    async disconnect() {
        try {
            await mongoose.disconnect();
            this.isConnected = false;
            console.log('📡 Disconnected from MongoDB');
        } catch (error) {
            console.error('❌ Error disconnecting from MongoDB:', error);
            throw error;
        }
    }
    
    getStatus() {
        return {
            isConnected: this.isConnected,
            readyState: mongoose.connection.readyState,
            host: mongoose.connection.host,
            port: mongoose.connection.port,
            name: mongoose.connection.name
        };
    }
}

// Usage
const db = new DatabaseConnection();
db.connect(process.env.NODE_ENV || 'development');
```

### Mongoose Schemas and Models

```javascript
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');

// User Schema
const userSchema = new mongoose.Schema({
    username: {
        type: String,
        required: [true, 'Username is required'],
        unique: true,
        trim: true,
        minlength: [3, 'Username must be at least 3 characters'],
        maxlength: [30, 'Username cannot exceed 30 characters']
    },
    
    email: {
        type: String,
        required: [true, 'Email is required'],
        unique: true,
        lowercase: true,
        validate: {
            validator: function(v) {
                return /^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/.test(v);
            },
            message: 'Please enter a valid email'
        }
    },
    
    password: {
        type: String,
        required: [true, 'Password is required'],
        minlength: [8, 'Password must be at least 8 characters'],
        select: false // Don't include in queries by default
    },
    
    profile: {
        firstName: { type: String, required: true },
        lastName: { type: String, required: true },
        avatar: { type: String },
        bio: { type: String, maxlength: 500 },
        dateOfBirth: { type: Date },
        location: {
            city: String,
            country: String,
            coordinates: {
                type: [Number], // [longitude, latitude]
                index: '2dsphere'
            }
        }
    },
    
    role: {
        type: String,
        enum: ['user', 'moderator', 'admin'],
        default: 'user'
    },
    
    preferences: {
        notifications: {
            email: { type: Boolean, default: true },
            push: { type: Boolean, default: true },
            sms: { type: Boolean, default: false }
        },
        theme: {
            type: String,
            enum: ['light', 'dark', 'auto'],
            default: 'light'
        },
        language: {
            type: String,
            default: 'en'
        }
    },
    
    isActive: {
        type: Boolean,
        default: true
    },
    
    lastLogin: {
        type: Date
    },
    
    loginAttempts: {
        type: Number,
        default: 0
    },
    
    lockUntil: {
        type: Date
    }
}, {
    timestamps: true, // Adds createdAt and updatedAt
    versionKey: false, // Disable __v field
    toJSON: { virtuals: true }, // Include virtuals in JSON
    toObject: { virtuals: true }
});

// Virtual properties
userSchema.virtual('fullName').get(function() {
    return `${this.profile.firstName} ${this.profile.lastName}`;
});

userSchema.virtual('isLocked').get(function() {
    return !!(this.lockUntil && this.lockUntil > Date.now());
});

// Indexes
userSchema.index({ email: 1 });
userSchema.index({ username: 1 });
userSchema.index({ 'profile.location.coordinates': '2dsphere' });
userSchema.index({ createdAt: -1 });
userSchema.index({ lastLogin: -1 });

// Pre-save middleware
userSchema.pre('save', async function(next) {
    // Only hash password if it's modified
    if (!this.isModified('password')) return next();
    
    try {
        const salt = await bcrypt.genSalt(12);
        this.password = await bcrypt.hash(this.password, salt);
        next();
    } catch (error) {
        next(error);
    }
});

// Instance methods
userSchema.methods.comparePassword = async function(candidatePassword) {
    try {
        return await bcrypt.compare(candidatePassword, this.password);
    } catch (error) {
        throw error;
    }
};

userSchema.methods.incrementLoginAttempts = function() {
    // If we have a previous lock that has expired, restart at 1
    if (this.lockUntil && this.lockUntil < Date.now()) {
        return this.updateOne({
            $set: { loginAttempts: 1 },
            $unset: { lockUntil: 1 }
        });
    }
    
    const updates = { $inc: { loginAttempts: 1 } };
    
    // If we're at max attempts and not locked, lock account
    if (this.loginAttempts + 1 >= 5 && !this.isLocked) {
        updates.$set = { lockUntil: Date.now() + 2 * 60 * 60 * 1000 }; // 2 hours
    }
    
    return this.updateOne(updates);
};

userSchema.methods.resetLoginAttempts = function() {
    return this.updateOne({
        $unset: { loginAttempts: 1, lockUntil: 1 }
    });
};

// Static methods
userSchema.statics.findByEmail = function(email) {
    return this.findOne({ email: email.toLowerCase() });
};

userSchema.statics.findActiveUsers = function() {
    return this.find({ isActive: true });
};

userSchema.statics.findByLocation = function(coordinates, maxDistance = 1000) {
    return this.find({
        'profile.location.coordinates': {
            $near: {
                $geometry: {
                    type: 'Point',
                    coordinates: coordinates
                },
                $maxDistance: maxDistance
            }
        }
    });
};

// Query helpers
userSchema.query.byRole = function(role) {
    return this.where({ role });
};

userSchema.query.active = function() {
    return this.where({ isActive: true });
};

// Create model
const User = mongoose.model('User', userSchema);

module.exports = User;
```

### Advanced Mongoose Operations

```javascript
const User = require('./models/User');

class UserService {
    // Create user
    async createUser(userData) {
        try {
            const user = new User(userData);
            await user.save();
            
            // Return user without password
            return await User.findById(user._id);
        } catch (error) {
            if (error.code === 11000) {
                throw new Error('User already exists');
            }
            throw error;
        }
    }
    
    // Find users with pagination
    async findUsers(filters = {}, options = {}) {
        const {
            page = 1,
            limit = 10,
            sort = { createdAt: -1 },
            populate = []
        } = options;
        
        const skip = (page - 1) * limit;
        
        let query = User.find(filters);
        
        // Apply population
        if (populate.length > 0) {
            populate.forEach(field => {
                query = query.populate(field);
            });
        }
        
        // Execute query with pagination
        const [users, total] = await Promise.all([
            query.sort(sort).skip(skip).limit(limit),
            User.countDocuments(filters)
        ]);
        
        return {
            users,
            pagination: {
                page,
                limit,
                total,
                pages: Math.ceil(total / limit),
                hasNext: page < Math.ceil(total / limit),
                hasPrev: page > 1
            }
        };
    }
    
    // Update user
    async updateUser(id, updateData) {
        try {
            const user = await User.findByIdAndUpdate(
                id,
                { $set: updateData },
                { 
                    new: true, // Return updated document
                    runValidators: true // Run schema validators
                }
            );
            
            if (!user) {
                throw new Error('User not found');
            }
            
            return user;
        } catch (error) {
            throw error;
        }
    }
    
    // Delete user (soft delete)
    async deleteUser(id) {
        try {
            const user = await User.findByIdAndUpdate(
                id,
                { isActive: false, deletedAt: new Date() },
                { new: true }
            );
            
            if (!user) {
                throw new Error('User not found');
            }
            
            return user;
        } catch (error) {
            throw error;
        }
    }
    
    // Search users
    async searchUsers(searchTerm, options = {}) {
        const {
            page = 1,
            limit = 10,
            fields = ['username', 'email', 'profile.firstName', 'profile.lastName']
        } = options;
        
        const skip = (page - 1) * limit;
        
        // Build search query
        const searchQuery = {
            $and: [
                { isActive: true },
                {
                    $or: fields.map(field => ({
                        [field]: { $regex: searchTerm, $options: 'i' }
                    }))
                }
            ]
        };
        
        const [users, total] = await Promise.all([
            User.find(searchQuery)
                .select('-password')
                .sort({ createdAt: -1 })
                .skip(skip)
                .limit(limit),
            User.countDocuments(searchQuery)
        ]);
        
        return {
            users,
            search: {
                term: searchTerm,
                fields,
                total,
                page,
                limit
            }
        };
    }
    
    // Aggregate users by role
    async getUserStats() {
        const stats = await User.aggregate([
            { $match: { isActive: true } },
            {
                $group: {
                    _id: '$role',
                    count: { $sum: 1 },
                    avgLoginAttempts: { $avg: '$loginAttempts' },
                    lastLogin: { $max: '$lastLogin' }
                }
            },
            { $sort: { count: -1 } }
        ]);
        
        const totalUsers = await User.countDocuments({ isActive: true });
        const totalInactive = await User.countDocuments({ isActive: false });
        
        return {
            byRole: stats,
            total: {
                active: totalUsers,
                inactive: totalInactive,
                all: totalUsers + totalInactive
            }
        };
    }
    
    // Bulk operations
    async bulkUpdateUsers(updates) {
        const bulkOps = updates.map(update => ({
            updateOne: {
                filter: { _id: update.id },
                update: { $set: update.data },
                upsert: false
            }
        }));
        
        const result = await User.bulkWrite(bulkOps);
        
        return {
            matched: result.matchedCount,
            modified: result.modifiedCount,
            upserted: result.upsertedCount
        };
    }
}

// Usage in Express routes
const userService = new UserService();

app.get('/api/users', async (req, res) => {
    try {
        const { page, limit, role, search } = req.query;
        
        let filters = {};
        if (role) filters.role = role;
        
        let result;
        if (search) {
            result = await userService.searchUsers(search, { page, limit });
        } else {
            result = await userService.findUsers(filters, { page, limit });
        }
        
        res.json(result);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

app.post('/api/users', async (req, res) => {
    try {
        const user = await userService.createUser(req.body);
        res.status(201).json(user);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

app.get('/api/users/stats', async (req, res) => {
    try {
        const stats = await userService.getUserStats();
        res.json(stats);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});
```

### Mongoose Relationships

```javascript
// Post Schema with references
const postSchema = new mongoose.Schema({
    title: {
        type: String,
        required: true,
        trim: true,
        maxlength: 200
    },
    
    content: {
        type: String,
        required: true
    },
    
    author: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'User',
        required: true
    },
    
    categories: [{
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Category'
    }],
    
    tags: [{
        type: String,
        lowercase: true,
        trim: true
    }],
    
    status: {
        type: String,
        enum: ['draft', 'published', 'archived'],
        default: 'draft'
    },
    
    publishedAt: {
        type: Date
    },
    
    views: {
        type: Number,
        default: 0
    },
    
    likes: [{
        user: {
            type: mongoose.Schema.Types.ObjectId,
            ref: 'User'
        },
        createdAt: {
            type: Date,
            default: Date.now
        }
    }],
    
    comments: [{
        author: {
            type: mongoose.Schema.Types.ObjectId,
            ref: 'User',
            required: true
        },
        content: {
            type: String,
            required: true,
            maxlength: 1000
        },
        createdAt: {
            type: Date,
            default: Date.now
        },
        replies: [{
            author: {
                type: mongoose.Schema.Types.ObjectId,
                ref: 'User'
            },
            content: String,
            createdAt: {
                type: Date,
                default: Date.now
            }
        }]
    }],
    
    metadata: {
        readingTime: Number, // in minutes
        wordCount: Number,
        excerpt: String,
        featuredImage: String,
        seoTitle: String,
        seoDescription: String
    }
}, {
    timestamps: true,
    toJSON: { virtuals: true },
    toObject: { virtuals: true }
});

// Virtual for comment count
postSchema.virtual('commentCount').get(function() {
    return this.comments.length;
});

// Virtual for like count
postSchema.virtual('likeCount').get(function() {
    return this.likes.length;
});

// Pre-save middleware
postSchema.pre('save', function(next) {
    if (this.isModified('content')) {
        // Calculate reading time and word count
        const words = this.content.split(/\s+/).length;
        this.metadata.wordCount = words;
        this.metadata.readingTime = Math.ceil(words / 200); // Assume 200 words per minute
        
        // Generate excerpt
        this.metadata.excerpt = this.content.substring(0, 200) + '...';
    }
    
    if (this.isModified('status') && this.status === 'published' && !this.publishedAt) {
        this.publishedAt = new Date();
    }
    
    next();
});

// Static methods
postSchema.statics.findPublished = function() {
    return this.find({ status: 'published' }).sort({ publishedAt: -1 });
};

postSchema.statics.findByAuthor = function(authorId) {
    return this.find({ author: authorId }).populate('author', 'username profile.firstName profile.lastName');
};

postSchema.statics.findPopular = function(limit = 10) {
    return this.find({ status: 'published' })
        .sort({ views: -1, likes: -1 })
        .limit(limit)
        .populate('author', 'username profile.firstName profile.lastName');
};

// Instance methods
postSchema.methods.addComment = function(authorId, content) {
    this.comments.push({
        author: authorId,
        content: content
    });
    return this.save();
};

postSchema.methods.toggleLike = function(userId) {
    const existingLike = this.likes.find(like => 
        like.user.toString() === userId.toString()
    );
    
    if (existingLike) {
        this.likes.pull({ _id: existingLike._id });
    } else {
        this.likes.push({ user: userId });
    }
    
    return this.save();
};

postSchema.methods.incrementViews = function() {
    return this.updateOne({ $inc: { views: 1 } });
};

const Post = mongoose.model('Post', postSchema);

// Category Schema
const categorySchema = new mongoose.Schema({
    name: {
        type: String,
        required: true,
        unique: true,
        trim: true
    },
    
    slug: {
        type: String,
        unique: true,
        lowercase: true
    },
    
    description: {
        type: String,
        maxlength: 500
    },
    
    parent: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Category'
    },
    
    isActive: {
        type: Boolean,
        default: true
    }
}, {
    timestamps: true
});

// Generate slug from name
categorySchema.pre('save', function(next) {
    if (this.isModified('name')) {
        this.slug = this.name.toLowerCase().replace(/[^a-z0-9]+/g, '-');
    }
    next();
});

const Category = mongoose.model('Category', categorySchema);

module.exports = { User, Post, Category };
```

### Complex Queries and Aggregations

```javascript
class PostService {
    // Advanced search with filters
    async searchPosts(searchOptions = {}) {
        const {
            query = '',
            category,
            author,
            tags = [],
            dateFrom,
            dateTo,
            minViews,
            status = 'published',
            page = 1,
            limit = 10,
            sort = { publishedAt: -1 }
        } = searchOptions;
        
        // Build MongoDB query
        const mongoQuery = { status };
        
        // Text search
        if (query) {
            mongoQuery.$or = [
                { title: { $regex: query, $options: 'i' } },
                { content: { $regex: query, $options: 'i' } },
                { tags: { $in: [new RegExp(query, 'i')] } }
            ];
        }
        
        // Category filter
        if (category) {
            mongoQuery.categories = category;
        }
        
        // Author filter
        if (author) {
            mongoQuery.author = author;
        }
        
        // Tags filter
        if (tags.length > 0) {
            mongoQuery.tags = { $in: tags };
        }
        
        // Date range filter
        if (dateFrom || dateTo) {
            mongoQuery.publishedAt = {};
            if (dateFrom) mongoQuery.publishedAt.$gte = new Date(dateFrom);
            if (dateTo) mongoQuery.publishedAt.$lte = new Date(dateTo);
        }
        
        // Minimum views filter
        if (minViews) {
            mongoQuery.views = { $gte: minViews };
        }
        
        const skip = (page - 1) * limit;
        
        // Execute query
        const [posts, total] = await Promise.all([
            Post.find(mongoQuery)
                .populate('author', 'username profile.firstName profile.lastName profile.avatar')
                .populate('categories', 'name slug')
                .sort(sort)
                .skip(skip)
                .limit(limit)
                .lean(), // Use lean() for better performance when you don't need full documents
            Post.countDocuments(mongoQuery)
        ]);
        
        return {
            posts,
            pagination: {
                page,
                limit,
                total,
                pages: Math.ceil(total / limit)
            },
            filters: searchOptions
        };
    }
    
    // Analytics aggregation
    async getPostAnalytics(timeframe = 'month') {
        let dateFilter;
        const now = new Date();
        
        switch (timeframe) {
            case 'day':
                dateFilter = new Date(now.setDate(now.getDate() - 1));
                break;
            case 'week':
                dateFilter = new Date(now.setDate(now.getDate() - 7));
                break;
            case 'month':
                dateFilter = new Date(now.setMonth(now.getMonth() - 1));
                break;
            case 'year':
                dateFilter = new Date(now.setFullYear(now.getFullYear() - 1));
                break;
            default:
                dateFilter = new Date(0); // All time
        }
        
        const analytics = await Post.aggregate([
            {
                $match: {
                    status: 'published',
                    publishedAt: { $gte: dateFilter }
                }
            },
            {
                $group: {
                    _id: null,
                    totalPosts: { $sum: 1 },
                    totalViews: { $sum: '$views' },
                    totalLikes: { $sum: { $size: '$likes' } },
                    totalComments: { $sum: { $size: '$comments' } },
                    avgViews: { $avg: '$views' },
                    avgReadingTime: { $avg: '$metadata.readingTime' },
                    mostViewedPost: { $max: '$views' },
                    leastViewedPost: { $min: '$views' }
                }
            }
        ]);
        
        // Posts by category
        const postsByCategory = await Post.aggregate([
            {
                $match: {
                    status: 'published',
                    publishedAt: { $gte: dateFilter }
                }
            },
            { $unwind: '$categories' },
            {
                $lookup: {
                    from: 'categories',
                    localField: 'categories',
                    foreignField: '_id',
                    as: 'categoryInfo'
                }
            },
            { $unwind: '$categoryInfo' },
            {
                $group: {
                    _id: '$categoryInfo.name',
                    count: { $sum: 1 },
                    totalViews: { $sum: '$views' }
                }
            },
            { $sort: { count: -1 } }
        ]);
        
        // Posts by month
        const postsByMonth = await Post.aggregate([
            {
                $match: {
                    status: 'published',
                    publishedAt: { $gte: dateFilter }
                }
            },
            {
                $group: {
                    _id: {
                        year: { $year: '$publishedAt' },
                        month: { $month: '$publishedAt' }
                    },
                    count: { $sum: 1 },
                    totalViews: { $sum: '$views' }
                }
            },
            { $sort: { '_id.year': -1, '_id.month': -1 } }
        ]);
        
        // Top authors
        const topAuthors = await Post.aggregate([
            {
                $match: {
                    status: 'published',
                    publishedAt: { $gte: dateFilter }
                }
            },
            {
                $group: {
                    _id: '$author',
                    postCount: { $sum: 1 },
                    totalViews: { $sum: '$views' },
                    totalLikes: { $sum: { $size: '$likes' } }
                }
            },
            {
                $lookup: {
                    from: 'users',
                    localField: '_id',
                    foreignField: '_id',
                    as: 'authorInfo'
                }
            },
            { $unwind: '$authorInfo' },
            {
                $project: {
                    author: {
                        id: '$authorInfo._id',
                        username: '$authorInfo.username',
                        fullName: '$authorInfo.fullName'
                    },
                    postCount: 1,
                    totalViews: 1,
                    totalLikes: 1,
                    avgViewsPerPost: { $divide: ['$totalViews', '$postCount'] }
                }
            },
            { $sort: { postCount: -1 } },
            { $limit: 10 }
        ]);
        
        return {
            overview: analytics[0] || {},
            byCategory: postsByCategory,
            byMonth: postsByMonth,
            topAuthors: topAuthors,
            timeframe: timeframe
        };
    }
}

// Usage
const postService = new PostService();

app.get('/api/posts/analytics', async (req, res) => {
    try {
        const { timeframe = 'month' } = req.query;
        const analytics = await postService.getPostAnalytics(timeframe);
        res.json(analytics);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});
```

## SQL Databases with Sequelize

Sequelize is a popular Object-Relational Mapping (ORM) library for Node.js.

### Setup and Configuration

```javascript
const { Sequelize, DataTypes } = require('sequelize');

// Database configuration
const dbConfig = {
    development: {
        username: 'root',
        password: 'password',
        database: 'myapp_dev',
        host: 'localhost',
        dialect: 'postgres',
        logging: console.log
    },
    production: {
        use_env_variable: 'DATABASE_URL',
        dialect: 'postgres',
        logging: false,
        pool: {
            max: 20,
            min: 0,
            acquire: 30000,
            idle: 10000
        }
    }
};

class DatabaseManager {
    constructor(environment = 'development') {
        this.environment = environment;
        this.sequelize = null;
        this.models = {};
    }
    
    async initialize() {
        try {
            const config = dbConfig[this.environment];
            
            if (config.use_env_variable) {
                this.sequelize = new Sequelize(process.env[config.use_env_variable], config);
            } else {
                this.sequelize = new Sequelize(
                    config.database,
                    config.username,
                    config.password,
                    config
                );
            }
            
            // Test connection
            await this.sequelize.authenticate();
            console.log('✅ Database connection established successfully');
            
            // Define models
            this.defineModels();
            
            // Set up associations
            this.setupAssociations();
            
            // Sync database (be careful in production)
            if (this.environment === 'development') {
                await this.sequelize.sync({ alter: true });
                console.log('📊 Database synchronized');
            }
            
            return this.sequelize;
        } catch (error) {
            console.error('❌ Unable to connect to database:', error);
            throw error;
        }
    }
    
    defineModels() {
        // User model
        this.models.User = this.sequelize.define('User', {
            id: {
                type: DataTypes.UUID,
                defaultValue: DataTypes.UUIDV4,
                primaryKey: true
            },
            
            username: {
                type: DataTypes.STRING(50),
                allowNull: false,
                unique: true,
                validate: {
                    len: [3, 50],
                    isAlphanumeric: true
                }
            },
            
            email: {
                type: DataTypes.STRING(100),
                allowNull: false,
                unique: true,
                validate: {
                    isEmail: true
                }
            },
            
            password: {
                type: DataTypes.STRING,
                allowNull: false,
                validate: {
                    len: [8, 100]
                }
            },
            
            firstName: {
                type: DataTypes.STRING(50),
                allowNull: false
            },
            
            lastName: {
                type: DataTypes.STRING(50),
                allowNull: false
            },
            
            avatar: {
                type: DataTypes.STRING,
                validate: {
                    isUrl: true
                }
            },
            
            role: {
                type: DataTypes.ENUM('user', 'moderator', 'admin'),
                defaultValue: 'user'
            },
            
            isActive: {
                type: DataTypes.BOOLEAN,
                defaultValue: true
            },
            
            lastLogin: {
                type: DataTypes.DATE
            },
            
            loginAttempts: {
                type: DataTypes.INTEGER,
                defaultValue: 0
            },
            
            lockUntil: {
                type: DataTypes.DATE
            }
        }, {
            timestamps: true,
            paranoid: true, // Soft deletes
            indexes: [
                { fields: ['email'] },
                { fields: ['username'] },
                { fields: ['createdAt'] },
                { fields: ['lastLogin'] }
            ]
        });
        
        // Post model
        this.models.Post = this.sequelize.define('Post', {
            id: {
                type: DataTypes.UUID,
                defaultValue: DataTypes.UUIDV4,
                primaryKey: true
            },
            
            title: {
                type: DataTypes.STRING(200),
                allowNull: false
            },
            
            slug: {
                type: DataTypes.STRING(250),
                unique: true
            },
            
            content: {
                type: DataTypes.TEXT,
                allowNull: false
            },
            
            excerpt: {
                type: DataTypes.STRING(500)
            },
            
            status: {
                type: DataTypes.ENUM('draft', 'published', 'archived'),
                defaultValue: 'draft'
            },
            
            publishedAt: {
                type: DataTypes.DATE
            },
            
            views: {
                type: DataTypes.INTEGER,
                defaultValue: 0
            },
            
            readingTime: {
                type: DataTypes.INTEGER // in minutes
            },
            
            wordCount: {
                type: DataTypes.INTEGER
            },
            
            featuredImage: {
                type: DataTypes.STRING
            }
        }, {
            timestamps: true,
            paranoid: true,
            indexes: [
                { fields: ['status'] },
                { fields: ['publishedAt'] },
                { fields: ['authorId'] },
                { fields: ['slug'] },
                { fields: ['views'] }
            ]
        });
        
        // Comment model
        this.models.Comment = this.sequelize.define('Comment', {
            id: {
                type: DataTypes.UUID,
                defaultValue: DataTypes.UUIDV4,
                primaryKey: true
            },
            
            content: {
                type: DataTypes.TEXT,
                allowNull: false,
                validate: {
                    len: [1, 1000]
                }
            },
            
            isApproved: {
                type: DataTypes.BOOLEAN,
                defaultValue: false
            }
        }, {
            timestamps: true,
            paranoid: true
        });
        
        // Like model
        this.models.Like = this.sequelize.define('Like', {
            id: {
                type: DataTypes.UUID,
                defaultValue: DataTypes.UUIDV4,
                primaryKey: true
            }
        }, {
            timestamps: true,
            indexes: [
                { unique: true, fields: ['userId', 'postId'] }
            ]
        });
    }
    
    setupAssociations() {
        const { User, Post, Comment, Like } = this.models;
        
        // User associations
        User.hasMany(Post, { foreignKey: 'authorId', as: 'posts' });
        User.hasMany(Comment, { foreignKey: 'authorId', as: 'comments' });
        User.hasMany(Like, { foreignKey: 'userId', as: 'likes' });
        
        // Post associations
        Post.belongsTo(User, { foreignKey: 'authorId', as: 'author' });
        Post.hasMany(Comment, { foreignKey: 'postId', as: 'comments' });
        Post.hasMany(Like, { foreignKey: 'postId', as: 'likes' });
        
        // Comment associations
        Comment.belongsTo(User, { foreignKey: 'authorId', as: 'author' });
        Comment.belongsTo(Post, { foreignKey: 'postId', as: 'post' });
        
        // Like associations
        Like.belongsTo(User, { foreignKey: 'userId', as: 'user' });
        Like.belongsTo(Post, { foreignKey: 'postId', as: 'post' });
        
        // Self-referencing association for comment replies
        Comment.hasMany(Comment, { foreignKey: 'parentId', as: 'replies' });
        Comment.belongsTo(Comment, { foreignKey: 'parentId', as: 'parent' });
    }
    
    getModels() {
        return this.models;
    }
    
    async close() {
        await this.sequelize.close();
    }
}

// Usage
const dbManager = new DatabaseManager(process.env.NODE_ENV);
await dbManager.initialize();
const { User, Post, Comment, Like } = dbManager.getModels();
```

### Sequelize Operations

```javascript
class PostService {
    constructor(models) {
        this.Post = models.Post;
        this.User = models.User;
        this.Comment = models.Comment;
        this.Like = models.Like;
    }
    
    // Create post with transaction
    async createPost(authorId, postData) {
        const t = await this.Post.sequelize.transaction();
        
        try {
            // Generate slug
            const slug = postData.title
                .toLowerCase()
                .replace(/[^a-z0-9]+/g, '-')
                .replace(/^-|-$/g, '');
            
            // Calculate reading time
            const wordCount = postData.content.split(/\s+/).length;
            const readingTime = Math.ceil(wordCount / 200);
            
            const post = await this.Post.create({
                ...postData,
                authorId,
                slug,
                wordCount,
                readingTime,
                excerpt: postData.content.substring(0, 200) + '...'
            }, { transaction: t });
            
            await t.commit();
            
            // Fetch post with associations
            return await this.Post.findByPk(post.id, {
                include: [
                    { model: this.User, as: 'author', attributes: ['username', 'firstName', 'lastName'] }
                ]
            });
        } catch (error) {
            await t.rollback();
            throw error;
        }
    }
    
    // Complex query with joins
    async getPostsWithStats(options = {}) {
        const {
            page = 1,
            limit = 10,
            authorId,
            status = 'published',
            includeStats = true
        } = options;
        
        const offset = (page - 1) * limit;
        
        const whereClause = { status };
        if (authorId) whereClause.authorId = authorId;
        
        const includeOptions = [
            {
                model: this.User,
                as: 'author',
                attributes: ['id', 'username', 'firstName', 'lastName', 'avatar']
            }
        ];
        
        if (includeStats) {
            includeOptions.push(
                {
                    model: this.Comment,
                    as: 'comments',
                    attributes: ['id', 'content', 'createdAt'],
                    include: [
                        {
                            model: this.User,
                            as: 'author',
                            attributes: ['username', 'firstName', 'lastName']
                        }
                    ]
                },
                {
                    model: this.Like,
                    as: 'likes',
                    attributes: ['id', 'userId', 'createdAt']
                }
            );
        }
        
        const { rows: posts, count: total } = await this.Post.findAndCountAll({
            where: whereClause,
            include: includeOptions,
            order: [['publishedAt', 'DESC']],
            limit,
            offset,
            distinct: true // Important when using includes
        });
        
        // Add computed fields
        const postsWithStats = posts.map(post => {
            const postData = post.toJSON();
            
            if (includeStats) {
                postData.stats = {
                    commentCount: post.comments.length,
                    likeCount: post.likes.length,
                    engagementRate: (post.comments.length + post.likes.length) / Math.max(post.views, 1)
                };
            }
            
            return postData;
        });
        
        return {
            posts: postsWithStats,
            pagination: {
                page,
                limit,
                total,
                pages: Math.ceil(total / limit)
            }
        };
    }
    
    // Raw SQL queries
    async getAdvancedAnalytics() {
        const [results] = await this.Post.sequelize.query(`
            SELECT 
                DATE_TRUNC('month', "publishedAt") as month,
                COUNT(*) as post_count,
                SUM(views) as total_views,
                AVG(views) as avg_views,
                COUNT(DISTINCT "authorId") as unique_authors
            FROM "Posts" 
            WHERE status = 'published' 
                AND "publishedAt" >= NOW() - INTERVAL '12 months'
                AND "deletedAt" IS NULL
            GROUP BY DATE_TRUNC('month', "publishedAt")
            ORDER BY month DESC
        `);
        
        return results;
    }
    
    // Bulk operations
    async bulkUpdateViews(postIds) {
        return await this.Post.update(
            { views: this.Post.sequelize.literal('views + 1') },
            { where: { id: postIds } }
        );
    }
    
    // Complex search with full-text search (PostgreSQL)
    async fullTextSearch(searchTerm, options = {}) {
        const { page = 1, limit = 10 } = options;
        const offset = (page - 1) * limit;
        
        const posts = await this.Post.findAll({
            where: this.Post.sequelize.literal(`
                to_tsvector('english', title || ' ' || content) 
                @@ plainto_tsquery('english', :searchTerm)
            `),
            replacements: { searchTerm },
            include: [
                {
                    model: this.User,
                    as: 'author',
                    attributes: ['username', 'firstName', 'lastName']
                }
            ],
            order: [
                [this.Post.sequelize.literal(`
                    ts_rank(to_tsvector('english', title || ' ' || content), 
                            plainto_tsquery('english', '${searchTerm}'))
                `), 'DESC']
            ],
            limit,
            offset
        });
        
        return posts;
    }
}

// Model hooks
User.beforeCreate(async (user) => {
    const salt = await bcrypt.genSalt(12);
    user.password = await bcrypt.hash(user.password, salt);
});

Post.beforeCreate((post) => {
    if (!post.slug) {
        post.slug = post.title
            .toLowerCase()
            .replace(/[^a-z0-9]+/g, '-')
            .replace(/^-|-$/g, '');
    }
});

// Instance methods
User.prototype.comparePassword = async function(candidatePassword) {
    return await bcrypt.compare(candidatePassword, this.password);
};

Post.prototype.incrementViews = function() {
    return this.increment('views');
};

// Class methods
User.findByEmail = function(email) {
    return this.findOne({ where: { email: email.toLowerCase() } });
};

Post.findPublished = function() {
    return this.findAll({
        where: { status: 'published' },
        order: [['publishedAt', 'DESC']]
    });
};
```

### Advanced Sequelize Features

```javascript
// Migrations
// migrations/20240101000000-create-users.js
module.exports = {
    async up(queryInterface, Sequelize) {
        await queryInterface.createTable('Users', {
            id: {
                type: Sequelize.UUID,
                defaultValue: Sequelize.UUIDV4,
                primaryKey: true
            },
            username: {
                type: Sequelize.STRING(50),
                allowNull: false,
                unique: true
            },
            email: {
                type: Sequelize.STRING(100),
                allowNull: false,
                unique: true
            },
            password: {
                type: Sequelize.STRING,
                allowNull: false
            },
            firstName: {
                type: Sequelize.STRING(50),
                allowNull: false
            },
            lastName: {
                type: Sequelize.STRING(50),
                allowNull: false
            },
            role: {
                type: Sequelize.ENUM('user', 'moderator', 'admin'),
                defaultValue: 'user'
            },
            isActive: {
                type: Sequelize.BOOLEAN,
                defaultValue: true
            },
            createdAt: {
                type: Sequelize.DATE,
                allowNull: false
            },
            updatedAt: {
                type: Sequelize.DATE,
                allowNull: false
            },
            deletedAt: {
                type: Sequelize.DATE
            }
        });
        
        // Add indexes
        await queryInterface.addIndex('Users', ['email']);
        await queryInterface.addIndex('Users', ['username']);
        await queryInterface.addIndex('Users', ['createdAt']);
    },
    
    async down(queryInterface, Sequelize) {
        await queryInterface.dropTable('Users');
    }
};

// Seeders
// seeders/20240101000000-demo-users.js
module.exports = {
    async up(queryInterface, Sequelize) {
        const bcrypt = require('bcrypt');
        
        const users = [];
        for (let i = 1; i <= 100; i++) {
            const salt = await bcrypt.genSalt(12);
            const hashedPassword = await bcrypt.hash('password123', salt);
            
            users.push({
                id: require('uuid').v4(),
                username: `user${i}`,
                email: `user${i}@example.com`,
                password: hashedPassword,
                firstName: `First${i}`,
                lastName: `Last${i}`,
                role: i <= 5 ? 'admin' : i <= 20 ? 'moderator' : 'user',
                createdAt: new Date(),
                updatedAt: new Date()
            });
        }
        
        await queryInterface.bulkInsert('Users', users);
    },
    
    async down(queryInterface, Sequelize) {
        await queryInterface.bulkDelete('Users', null, {});
    }
};

// Run migrations and seeds
// npm run migrate
// npm run seed
```

## Native Database Drivers

### MongoDB Native Driver

```javascript
const { MongoClient } = require('mongodb');

class MongoDBClient {
    constructor(url) {
        this.url = url;
        this.client = null;
        this.db = null;
    }
    
    async connect(dbName) {
        try {
            this.client = new MongoClient(this.url, {
                useNewUrlParser: true,
                useUnifiedTopology: true,
                maxPoolSize: 10,
                serverSelectionTimeoutMS: 5000
            });
            
            await this.client.connect();
            this.db = this.client.db(dbName);
            
            console.log('✅ Connected to MongoDB');
            return this.db;
        } catch (error) {
            console.error('❌ MongoDB connection failed:', error);
            throw error;
        }
    }
    
    async disconnect() {
        if (this.client) {
            await this.client.close();
            console.log('📡 Disconnected from MongoDB');
        }
    }
    
    // Collection operations
    collection(name) {
        return this.db.collection(name);
    }
    
    // User operations
    async createUser(userData) {
        const users = this.collection('users');
        const result = await users.insertOne({
            ...userData,
            createdAt: new Date(),
            updatedAt: new Date()
        });
        
        return await users.findOne({ _id: result.insertedId });
    }
    
    async findUsers(filter = {}, options = {}) {
        const users = this.collection('users');
        const {
            page = 1,
            limit = 10,
            sort = { createdAt: -1 },
            projection = {}
        } = options;
        
        const skip = (page - 1) * limit;
        
        const cursor = users.find(filter, { projection })
            .sort(sort)
            .skip(skip)
            .limit(limit);
        
        const [results, total] = await Promise.all([
            cursor.toArray(),
            users.countDocuments(filter)
        ]);
        
        return {
            users: results,
            pagination: {
                page,
                limit,
                total,
                pages: Math.ceil(total / limit)
            }
        };
    }
    
    async updateUser(id, updateData) {
        const users = this.collection('users');
        const { ObjectId } = require('mongodb');
        
        const result = await users.findOneAndUpdate(
            { _id: new ObjectId(id) },
            { 
                $set: { 
                    ...updateData, 
                    updatedAt: new Date() 
                } 
            },
            { returnDocument: 'after' }
        );
        
        return result.value;
    }
    
    // Aggregation pipeline
    async getUserStatistics() {
        const users = this.collection('users');
        
        const pipeline = [
            {
                $group: {
                    _id: '$role',
                    count: { $sum: 1 },
                    avgLoginAttempts: { $avg: '$loginAttempts' }
                }
            },
            {
                $sort: { count: -1 }
            }
        ];
        
        return await users.aggregate(pipeline).toArray();
    }
    
    // Text search
    async searchUsers(searchTerm) {
        const users = this.collection('users');
        
        // Create text index if not exists
        await users.createIndex({
            username: 'text',
            email: 'text',
            'profile.firstName': 'text',
            'profile.lastName': 'text'
        });
        
        return await users.find({
            $text: { $search: searchTerm }
        }, {
            score: { $meta: 'textScore' }
        }).sort({ score: { $meta: 'textScore' } }).toArray();
    }
    
    // Geospatial queries
    async findNearbyUsers(longitude, latitude, maxDistance = 1000) {
        const users = this.collection('users');
        
        return await users.find({
            'location.coordinates': {
                $near: {
                    $geometry: {
                        type: 'Point',
                        coordinates: [longitude, latitude]
                    },
                    $maxDistance: maxDistance
                }
            }
        }).toArray();
    }
}

// Usage
const mongoClient = new MongoDBClient('mongodb://localhost:27017');
await mongoClient.connect('myapp');
```

### PostgreSQL with pg

```javascript
const { Pool, Client } = require('pg');

class PostgreSQLClient {
    constructor(config) {
        this.pool = new Pool({
            host: config.host || 'localhost',
            port: config.port || 5432,
            database: config.database,
            user: config.user,
            password: config.password,
            max: config.maxConnections || 20,
            idleTimeoutMillis: 30000,
            connectionTimeoutMillis: 2000
        });
        
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        this.pool.on('connect', (client) => {
            console.log('📡 New client connected to PostgreSQL');
        });
        
        this.pool.on('error', (err) => {
            console.error('❌ PostgreSQL pool error:', err);
        });
        
        this.pool.on('remove', (client) => {
            console.log('📡 Client removed from PostgreSQL pool');
        });
    }
    
    // Query with automatic connection handling
    async query(text, params = []) {
        const start = Date.now();
        
        try {
            const result = await this.pool.query(text, params);
            const duration = Date.now() - start;
            
            console.log(`📊 Query executed in ${duration}ms`);
            return result;
        } catch (error) {
            console.error('❌ Query error:', error);
            throw error;
        }
    }
    
    // Transaction support
    async transaction(callback) {
        const client = await this.pool.connect();
        
        try {
            await client.query('BEGIN');
            const result = await callback(client);
            await client.query('COMMIT');
            
            console.log('✅ Transaction committed');
            return result;
        } catch (error) {
            await client.query('ROLLBACK');
            console.log('↩️ Transaction rolled back');
            throw error;
        } finally {
            client.release();
        }
    }
    
    // Prepared statements
    async preparedQuery(name, text, values = []) {
        const client = await this.pool.connect();
        
        try {
            // Prepare statement if not exists
            await client.query(`PREPARE ${name} AS ${text}`);
            
            // Execute prepared statement
            const result = await client.query(`EXECUTE ${name}`, values);
            return result;
        } catch (error) {
            // If statement already exists, just execute
            if (error.code === '42P05') {
                const result = await client.query(`EXECUTE ${name}`, values);
                return result;
            }
            throw error;
        } finally {
            client.release();
        }
    }
    
    async close() {
        await this.pool.end();
        console.log('📡 PostgreSQL pool closed');
    }
}

// User repository with PostgreSQL
class UserRepository {
    constructor(dbClient) {
        this.db = dbClient;
    }
    
    async createUser(userData) {
        const query = `
            INSERT INTO users (username, email, password, first_name, last_name, role)
            VALUES ($1, $2, $3, $4, $5, $6)
            RETURNING id, username, email, first_name, last_name, role, created_at
        `;
        
        const values = [
            userData.username,
            userData.email,
            userData.password,
            userData.firstName,
            userData.lastName,
            userData.role || 'user'
        ];
        
        const result = await this.db.query(query, values);
        return result.rows[0];
    }
    
    async findUsers(filters = {}, options = {}) {
        const { page = 1, limit = 10, sort = 'created_at DESC' } = options;
        const offset = (page - 1) * limit;
        
        let whereClause = 'WHERE is_active = true';
        const values = [];
        let paramCount = 0;
        
        // Build dynamic where clause
        if (filters.role) {
            paramCount++;
            whereClause += ` AND role = $${paramCount}`;
            values.push(filters.role);
        }
        
        if (filters.email) {
            paramCount++;
            whereClause += ` AND email ILIKE $${paramCount}`;
            values.push(`%${filters.email}%`);
        }
        
        const query = `
            SELECT 
                id, username, email, first_name, last_name, 
                role, is_active, last_login, created_at, updated_at,
                COUNT(*) OVER() as total_count
            FROM users 
            ${whereClause}
            ORDER BY ${sort}
            LIMIT $${paramCount + 1} OFFSET $${paramCount + 2}
        `;
        
        values.push(limit, offset);
        
        const result = await this.db.query(query, values);
        const users = result.rows;
        const total = users.length > 0 ? parseInt(users[0].total_count) : 0;
        
        // Remove total_count from results
        users.forEach(user => delete user.total_count);
        
        return {
            users,
            pagination: {
                page,
                limit,
                total,
                pages: Math.ceil(total / limit)
            }
        };
    }
    
    async updateUser(id, updateData) {
        const fields = Object.keys(updateData);
        const values = Object.values(updateData);
        
        const setClause = fields.map((field, index) => 
            `${field} = $${index + 1}`
        ).join(', ');
        
        const query = `
            UPDATE users 
            SET ${setClause}, updated_at = NOW()
            WHERE id = $${fields.length + 1}
            RETURNING id, username, email, first_name, last_name, role, updated_at
        `;
        
        values.push(id);
        
        const result = await this.db.query(query, values);
        return result.rows[0];
    }
    
    async deleteUser(id) {
        // Soft delete
        const query = `
            UPDATE users 
            SET is_active = false, deleted_at = NOW()
            WHERE id = $1
            RETURNING id
        `;
        
        const result = await this.db.query(query, [id]);
        return result.rows[0];
    }
    
    // Complex join query
    async getUsersWithPostStats() {
        const query = `
            SELECT 
                u.id, u.username, u.email, u.first_name, u.last_name,
                COUNT(p.id) as post_count,
                SUM(p.views) as total_views,
                AVG(p.views) as avg_views,
                MAX(p.published_at) as last_post_date
            FROM users u
            LEFT JOIN posts p ON u.id = p.author_id AND p.status = 'published'
            WHERE u.is_active = true
            GROUP BY u.id, u.username, u.email, u.first_name, u.last_name
            ORDER BY post_count DESC, total_views DESC
        `;
        
        const result = await this.db.query(query);
        return result.rows;
    }
}

// Usage
const pgClient = new PostgreSQLClient({
    host: 'localhost',
    database: 'myapp',
    user: 'postgres',
    password: 'password'
});

const userRepo = new UserRepository(pgClient);
```

## Database Connection Management

### Connection Pooling

```javascript
class ConnectionPoolManager {
    constructor() {
        this.pools = new Map();
        this.healthCheckInterval = null;
    }
    
    // MongoDB connection pool
    createMongoPool(name, config) {
        const mongoose = require('mongoose');
        
        const connection = mongoose.createConnection(config.url, {
            ...config.options,
            maxPoolSize: config.maxPoolSize || 10,
            minPoolSize: config.minPoolSize || 1,
            maxIdleTimeMS: config.maxIdleTimeMS || 30000,
            serverSelectionTimeoutMS: config.serverSelectionTimeoutMS || 5000
        });
        
        this.pools.set(name, {
            type: 'mongodb',
            connection,
            config
        });
        
        return connection;
    }
    
    // PostgreSQL connection pool
    createPostgresPool(name, config) {
        const { Pool } = require('pg');
        
        const pool = new Pool({
            host: config.host,
            port: config.port,
            database: config.database,
            user: config.user,
            password: config.password,
            max: config.max || 20,
            min: config.min || 2,
            idleTimeoutMillis: config.idleTimeoutMillis || 30000,
            connectionTimeoutMillis: config.connectionTimeoutMillis || 2000
        });
        
        this.pools.set(name, {
            type: 'postgresql',
            pool,
            config
        });
        
        return pool;
    }
    
    // Redis connection pool
    createRedisPool(name, config) {
        const Redis = require('ioredis');
        
        const redis = new Redis({
            host: config.host,
            port: config.port,
            password: config.password,
            db: config.db || 0,
            retryDelayOnFailover: 100,
            maxRetriesPerRequest: 3,
            lazyConnect: true,
            keepAlive: 30000
        });
        
        this.pools.set(name, {
            type: 'redis',
            client: redis,
            config
        });
        
        return redis;
    }
    
    // Get connection pool
    getPool(name) {
        const pool = this.pools.get(name);
        if (!pool) {
            throw new Error(`Connection pool '${name}' not found`);
        }
        return pool;
    }
    
    // Health checks
    async checkHealth(name) {
        const pool = this.getPool(name);
        
        try {
            switch (pool.type) {
                case 'mongodb':
                    await pool.connection.db.admin().ping();
                    return { status: 'healthy', type: 'mongodb' };
                    
                case 'postgresql':
                    const client = await pool.pool.connect();
                    await client.query('SELECT 1');
                    client.release();
                    return { status: 'healthy', type: 'postgresql' };
                    
                case 'redis':
                    await pool.client.ping();
                    return { status: 'healthy', type: 'redis' };
                    
                default:
                    throw new Error('Unknown pool type');
            }
        } catch (error) {
            return { 
                status: 'unhealthy', 
                type: pool.type, 
                error: error.message 
            };
        }
    }
    
    // Start health monitoring
    startHealthMonitoring(interval = 30000) {
        this.healthCheckInterval = setInterval(async () => {
            for (const [name, pool] of this.pools) {
                const health = await this.checkHealth(name);
                console.log(`Health check ${name}:`, health);
                
                if (health.status === 'unhealthy') {
                    // Implement reconnection logic here
                    console.warn(`⚠️ Pool ${name} is unhealthy, attempting reconnection...`);
                }
            }
        }, interval);
    }
    
    // Stop health monitoring
    stopHealthMonitoring() {
        if (this.healthCheckInterval) {
            clearInterval(this.healthCheckInterval);
            this.healthCheckInterval = null;
        }
    }
    
    // Close all connections
    async closeAll() {
        this.stopHealthMonitoring();
        
        for (const [name, pool] of this.pools) {
            try {
                switch (pool.type) {
                    case 'mongodb':
                        await pool.connection.close();
                        break;
                    case 'postgresql':
                        await pool.pool.end();
                        break;
                    case 'redis':
                        await pool.client.quit();
                        break;
                }
                console.log(`✅ Closed connection pool: ${name}`);
            } catch (error) {
                console.error(`❌ Error closing pool ${name}:`, error);
            }
        }
        
        this.pools.clear();
    }
    
    // Get pool statistics
    getPoolStats(name) {
        const pool = this.getPool(name);
        
        switch (pool.type) {
            case 'postgresql':
                return {
                    totalCount: pool.pool.totalCount,
                    idleCount: pool.pool.idleCount,
                    waitingCount: pool.pool.waitingCount
                };
                
            case 'mongodb':
                return {
                    readyState: pool.connection.readyState,
                    host: pool.connection.host,
                    port: pool.connection.port
                };
                
            case 'redis':
                return {
                    status: pool.client.status,
                    connectedClients: pool.client.connector?.connecting ? 1 : 0
                };
                
            default:
                return { type: pool.type };
        }
    }
}

// Usage
const poolManager = new ConnectionPoolManager();

// Create connection pools
const mongoPool = poolManager.createMongoPool('main', {
    url: 'mongodb://localhost:27017/myapp',
    options: { useNewUrlParser: true, useUnifiedTopology: true }
});

const pgPool = poolManager.createPostgresPool('main', {
    host: 'localhost',
    database: 'myapp',
    user: 'postgres',
    password: 'password'
});

const redisPool = poolManager.createRedisPool('cache', {
    host: 'localhost',
    port: 6379
});

// Start monitoring
poolManager.startHealthMonitoring();

// Graceful shutdown
process.on('SIGTERM', async () => {
    console.log('🔄 Closing database connections...');
    await poolManager.closeAll();
    process.exit(0);
});
```

## Caching Strategies

### Redis Caching

```javascript
const Redis = require('ioredis');

class CacheManager {
    constructor(redisConfig) {
        this.redis = new Redis(redisConfig);
        this.defaultTTL = 3600; // 1 hour
        
        this.redis.on('connect', () => {
            console.log('✅ Connected to Redis');
        });
        
        this.redis.on('error', (err) => {
            console.error('❌ Redis error:', err);
        });
    }
    
    // Basic cache operations
    async get(key) {
        try {
            const value = await this.redis.get(key);
            return value ? JSON.parse(value) : null;
        } catch (error) {
            console.error('Cache get error:', error);
            return null;
        }
    }
    
    async set(key, value, ttl = this.defaultTTL) {
        try {
            const serialized = JSON.stringify(value);
            await this.redis.setex(key, ttl, serialized);
            return true;
        } catch (error) {
            console.error('Cache set error:', error);
            return false;
        }
    }
    
    async del(key) {
        try {
            await this.redis.del(key);
            return true;
        } catch (error) {
            console.error('Cache delete error:', error);
            return false;
        }
    }
    
    // Cache with automatic refresh
    async getOrSet(key, fetchFunction, ttl = this.defaultTTL) {
        try {
            // Try to get from cache first
            let value = await this.get(key);
            
            if (value === null) {
                // Cache miss - fetch data
                value = await fetchFunction();
                await this.set(key, value, ttl);
            }
            
            return value;
        } catch (error) {
            console.error('Cache getOrSet error:', error);
            // Fallback to direct fetch
            return await fetchFunction();
        }
    }
    
    // Cache with tags for invalidation
    async setWithTags(key, value, tags = [], ttl = this.defaultTTL) {
        const pipeline = this.redis.pipeline();
        
        // Set the main cache entry
        pipeline.setex(key, ttl, JSON.stringify(value));
        
        // Add to tag sets
        tags.forEach(tag => {
            pipeline.sadd(`tag:${tag}`, key);
            pipeline.expire(`tag:${tag}`, ttl + 60); // Tag expires slightly later
        });
        
        await pipeline.exec();
        return true;
    }
    
    async invalidateByTag(tag) {
        try {
            // Get all keys with this tag
            const keys = await this.redis.smembers(`tag:${tag}`);
            
            if (keys.length > 0) {
                const pipeline = this.redis.pipeline();
                
                // Delete all keys
                keys.forEach(key => pipeline.del(key));
                
                // Delete the tag set
                pipeline.del(`tag:${tag}`);
                
                await pipeline.exec();
                console.log(`🗑️ Invalidated ${keys.length} cache entries for tag: ${tag}`);
            }
            
            return keys.length;
        } catch (error) {
            console.error('Cache invalidation error:', error);
            return 0;
        }
    }
    
    // Distributed locking
    async acquireLock(lockKey, ttl = 10) {
        const lockValue = Math.random().toString(36);
        const result = await this.redis.set(
            `lock:${lockKey}`,
            lockValue,
            'EX',
            ttl,
            'NX'
        );
        
        return result === 'OK' ? lockValue : null;
    }
    
    async releaseLock(lockKey, lockValue) {
        const script = `
            if redis.call("GET", KEYS[1]) == ARGV[1] then
                return redis.call("DEL", KEYS[1])
            else
                return 0
            end
        `;
        
        const result = await this.redis.eval(script, 1, `lock:${lockKey}`, lockValue);
        return result === 1;
    }
    
    // Cache statistics
    async getStats() {
        const info = await this.redis.info('memory');
        const keyspace = await this.redis.info('keyspace');
        
        return {
            memory: this.parseRedisInfo(info),
            keyspace: this.parseRedisInfo(keyspace),
            connections: await this.redis.info('clients')
        };
    }
    
    parseRedisInfo(infoString) {
        const lines = infoString.split('\r\n');
        const info = {};
        
        lines.forEach(line => {
            if (line && !line.startsWith('#')) {
                const [key, value] = line.split(':');
                info[key] = value;
            }
        });
        
        return info;
    }
}

// Cache middleware for Express
function cacheMiddleware(cacheManager, options = {}) {
    return async (req, res, next) => {
        const {
            ttl = 300,
            keyGenerator = (req) => `route:${req.method}:${req.originalUrl}`,
            condition = () => true,
            tags = []
        } = options;
        
        // Check if caching should be applied
        if (!condition(req)) {
            return next();
        }
        
        const cacheKey = keyGenerator(req);
        
        try {
            // Try to get cached response
            const cached = await cacheManager.get(cacheKey);
            
            if (cached) {
                console.log(`📊 Cache hit: ${cacheKey}`);
                return res.json(cached);
            }
            
            // Cache miss - intercept response
            const originalJson = res.json;
            res.json = function(data) {
                // Cache the response
                cacheManager.setWithTags(cacheKey, data, tags, ttl)
                    .catch(err => console.error('Cache set error:', err));
                
                console.log(`💾 Cached response: ${cacheKey}`);
                return originalJson.call(this, data);
            };
            
            next();
        } catch (error) {
            console.error('Cache middleware error:', error);
            next();
        }
    };
}

// Usage
const cacheManager = new CacheManager({
    host: 'localhost',
    port: 6379
});

// Cache user list for 5 minutes
app.get('/api/users', 
    cacheMiddleware(cacheManager, {
        ttl: 300,
        tags: ['users'],
        condition: (req) => !req.query.search // Don't cache search results
    }),
    async (req, res) => {
        const users = await userService.findUsers();
        res.json(users);
    }
);

// Invalidate cache when user is created/updated
app.post('/api/users', async (req, res) => {
    try {
        const user = await userService.createUser(req.body);
        
        // Invalidate users cache
        await cacheManager.invalidateByTag('users');
        
        res.status(201).json(user);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});
```

---

This covers comprehensive database integration including MongoDB with Mongoose, SQL databases with Sequelize, native drivers, connection management, and caching strategies. The content includes practical examples, advanced patterns, and production-ready code. Would you like me to continue with the final section on Advanced Node.js topics?
