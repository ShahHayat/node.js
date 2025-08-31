# JavaScript Advanced Concepts - Complete Guide

## Table of Contents
1. [Prototypes and Inheritance](#prototypes-and-inheritance)
2. [Advanced Closures and Lexical Scoping](#advanced-closures-and-lexical-scoping)
3. [Asynchronous JavaScript](#asynchronous-javascript)
4. [ES6+ Features](#es6-features)
5. [Design Patterns](#design-patterns)
6. [Memory Management](#memory-management)
7. [Performance Optimization](#performance-optimization)
8. [Advanced Function Concepts](#advanced-function-concepts)
9. [Metaprogramming](#metaprogramming)

## Prototypes and Inheritance

### Understanding Prototypes

Every JavaScript object has a prototype, which is another object that it inherits properties and methods from.

```javascript
// Every object has a __proto__ property
let obj = {};
console.log(obj.__proto__); // Object.prototype

// Function prototypes
function Person(name) {
    this.name = name;
}

Person.prototype.greet = function() {
    return `Hello, I'm ${this.name}`;
};

let john = new Person("John");
console.log(john.greet()); // "Hello, I'm John"
console.log(john.__proto__ === Person.prototype); // true
```

### Prototype Chain

```javascript
// Prototype chain example
function Animal(name) {
    this.name = name;
}

Animal.prototype.speak = function() {
    return `${this.name} makes a sound`;
};

function Dog(name, breed) {
    Animal.call(this, name); // Call parent constructor
    this.breed = breed;
}

// Set up inheritance
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

Dog.prototype.bark = function() {
    return `${this.name} barks!`;
};

let buddy = new Dog("Buddy", "Golden Retriever");
console.log(buddy.speak()); // "Buddy makes a sound"
console.log(buddy.bark());  // "Buddy barks!"

// Prototype chain: buddy -> Dog.prototype -> Animal.prototype -> Object.prototype -> null
```

### Modern Class Syntax (ES6)

```javascript
class Animal {
    constructor(name) {
        this.name = name;
    }
    
    speak() {
        return `${this.name} makes a sound`;
    }
    
    // Static method
    static getSpecies() {
        return "Unknown species";
    }
    
    // Getter
    get info() {
        return `Animal: ${this.name}`;
    }
    
    // Setter
    set nickname(value) {
        this._nickname = value;
    }
}

class Dog extends Animal {
    constructor(name, breed) {
        super(name); // Call parent constructor
        this.breed = breed;
    }
    
    speak() {
        return `${this.name} barks!`;
    }
    
    // Private field (ES2022)
    #secretTrick = "roll over";
    
    // Private method
    #performTrick() {
        return this.#secretTrick;
    }
    
    showTrick() {
        return this.#performTrick();
    }
}

let max = new Dog("Max", "Labrador");
console.log(max.speak()); // "Max barks!"
console.log(max.info);    // "Animal: Max"
```

### Advanced Prototype Manipulation

```javascript
// Object.create() for prototype-based inheritance
let animalProto = {
    init(name) {
        this.name = name;
        return this;
    },
    speak() {
        return `${this.name} makes a sound`;
    }
};

let dogProto = Object.create(animalProto);
dogProto.bark = function() {
    return `${this.name} barks!`;
};

let dog = Object.create(dogProto).init("Rex");
console.log(dog.speak()); // "Rex makes a sound"
console.log(dog.bark());  // "Rex barks!"

// Prototype property manipulation
Object.defineProperty(Animal.prototype, 'species', {
    value: 'Animalia',
    writable: false,
    enumerable: true,
    configurable: false
});

// Check prototype relationships
console.log(dog.isPrototypeOf(animalProto)); // false
console.log(animalProto.isPrototypeOf(dog)); // true
console.log(Object.getPrototypeOf(dog) === dogProto); // true
```

## Advanced Closures and Lexical Scoping

### Closure Patterns

#### Module Pattern
```javascript
const Calculator = (function() {
    let history = [];
    
    function add(a, b) {
        const result = a + b;
        history.push(`${a} + ${b} = ${result}`);
        return result;
    }
    
    function getHistory() {
        return [...history]; // Return copy
    }
    
    function clearHistory() {
        history = [];
    }
    
    // Public API
    return {
        add,
        getHistory,
        clearHistory
    };
})();

Calculator.add(5, 3); // 8
console.log(Calculator.getHistory()); // ["5 + 3 = 8"]
```

#### Factory Pattern with Closures
```javascript
function createCounter(initialValue = 0) {
    let count = initialValue;
    
    return {
        increment: (step = 1) => count += step,
        decrement: (step = 1) => count -= step,
        getValue: () => count,
        reset: () => count = initialValue
    };
}

const counter1 = createCounter(10);
const counter2 = createCounter(0);

counter1.increment(5); // 15
counter2.increment();  // 1
```

#### Currying with Closures
```javascript
// Basic currying
function multiply(a) {
    return function(b) {
        return a * b;
    };
}

const multiplyBy2 = multiply(2);
console.log(multiplyBy2(5)); // 10

// Advanced currying
function curry(fn) {
    return function curried(...args) {
        if (args.length >= fn.length) {
            return fn.apply(this, args);
        } else {
            return function(...nextArgs) {
                return curried.apply(this, args.concat(nextArgs));
            };
        }
    };
}

function add(a, b, c) {
    return a + b + c;
}

const curriedAdd = curry(add);
console.log(curriedAdd(1)(2)(3)); // 6
console.log(curriedAdd(1, 2)(3)); // 6
console.log(curriedAdd(1)(2, 3)); // 6
```

### Memory Leaks and Closure Gotchas

```javascript
// Memory leak example
function problematicClosure() {
    let largeData = new Array(1000000).fill('data');
    
    return function() {
        // Even though we don't use largeData, it's kept in memory
        return "Hello World";
    };
}

// Better approach
function optimizedClosure() {
    let largeData = new Array(1000000).fill('data');
    let result = largeData.length; // Extract what we need
    largeData = null; // Help garbage collector
    
    return function() {
        return `Array had ${result} elements`;
    };
}

// Event handler memory leak
function attachListeners() {
    let element = document.getElementById('button');
    let data = new Array(1000000).fill('data');
    
    element.addEventListener('click', function() {
        // data is captured in closure even if not used
        console.log('Clicked');
    });
    
    // Better: remove reference or use weak references
    element.addEventListener('click', function() {
        console.log('Clicked');
    });
    data = null; // Help GC
}
```

## Asynchronous JavaScript

### Callbacks and Callback Hell

```javascript
// Callback pattern
function fetchData(callback) {
    setTimeout(() => {
        callback(null, "Data fetched");
    }, 1000);
}

fetchData((error, data) => {
    if (error) {
        console.error(error);
    } else {
        console.log(data);
    }
});

// Callback hell example
function callbackHell() {
    getData(function(a) {
        getMoreData(a, function(b) {
            getMoreData(b, function(c) {
                getMoreData(c, function(d) {
                    // Pyramid of doom!
                    console.log(d);
                });
            });
        });
    });
}
```

### Promises

```javascript
// Creating promises
function fetchUserData(userId) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (userId > 0) {
                resolve({ id: userId, name: `User ${userId}` });
            } else {
                reject(new Error("Invalid user ID"));
            }
        }, 1000);
    });
}

// Using promises
fetchUserData(1)
    .then(user => {
        console.log("User:", user);
        return fetchUserData(2); // Chain promises
    })
    .then(user => {
        console.log("Second user:", user);
    })
    .catch(error => {
        console.error("Error:", error.message);
    })
    .finally(() => {
        console.log("Promise chain completed");
    });

// Promise.all - Wait for all promises
Promise.all([
    fetchUserData(1),
    fetchUserData(2),
    fetchUserData(3)
]).then(users => {
    console.log("All users:", users);
}).catch(error => {
    console.error("One or more promises failed:", error);
});

// Promise.allSettled - Wait for all promises (ES2020)
Promise.allSettled([
    fetchUserData(1),
    fetchUserData(-1), // This will reject
    fetchUserData(3)
]).then(results => {
    results.forEach((result, index) => {
        if (result.status === 'fulfilled') {
            console.log(`Promise ${index} succeeded:`, result.value);
        } else {
            console.log(`Promise ${index} failed:`, result.reason);
        }
    });
});

// Promise.race - First promise to resolve/reject
Promise.race([
    fetchUserData(1),
    fetchUserData(2)
]).then(user => {
    console.log("First user:", user);
});

// Promise.any - First promise to resolve (ES2021)
Promise.any([
    Promise.reject("Error 1"),
    fetchUserData(2),
    fetchUserData(3)
]).then(user => {
    console.log("First successful user:", user);
});
```

### Async/Await

```javascript
// Basic async/await
async function fetchUsers() {
    try {
        const user1 = await fetchUserData(1);
        const user2 = await fetchUserData(2);
        
        console.log("Users:", user1, user2);
        return [user1, user2];
    } catch (error) {
        console.error("Failed to fetch users:", error);
        throw error;
    }
}

// Parallel execution with async/await
async function fetchUsersParallel() {
    try {
        const [user1, user2, user3] = await Promise.all([
            fetchUserData(1),
            fetchUserData(2),
            fetchUserData(3)
        ]);
        
        return [user1, user2, user3];
    } catch (error) {
        console.error("Error fetching users:", error);
        throw error;
    }
}

// Sequential vs Parallel comparison
async function sequentialExample() {
    const start = Date.now();
    
    const user1 = await fetchUserData(1); // Wait 1 second
    const user2 = await fetchUserData(2); // Wait another 1 second
    
    console.log(`Sequential took: ${Date.now() - start}ms`); // ~2000ms
}

async function parallelExample() {
    const start = Date.now();
    
    const [user1, user2] = await Promise.all([
        fetchUserData(1), // Both start simultaneously
        fetchUserData(2)
    ]);
    
    console.log(`Parallel took: ${Date.now() - start}ms`); // ~1000ms
}

// Error handling patterns
async function errorHandlingPatterns() {
    // Pattern 1: Try-catch for individual operations
    try {
        const user = await fetchUserData(1);
        console.log(user);
    } catch (error) {
        console.error("Failed to fetch user:", error);
    }
    
    // Pattern 2: Multiple try-catch blocks
    let user1, user2;
    
    try {
        user1 = await fetchUserData(1);
    } catch (error) {
        console.error("Failed to fetch user1:", error);
        user1 = null;
    }
    
    try {
        user2 = await fetchUserData(2);
    } catch (error) {
        console.error("Failed to fetch user2:", error);
        user2 = null;
    }
    
    // Pattern 3: Promise-based error handling
    const results = await Promise.allSettled([
        fetchUserData(1),
        fetchUserData(-1)
    ]);
    
    results.forEach((result, index) => {
        if (result.status === 'fulfilled') {
            console.log(`User ${index}:`, result.value);
        } else {
            console.error(`User ${index} error:`, result.reason);
        }
    });
}
```

### Async Iterators and Generators

```javascript
// Generator functions
function* numberGenerator() {
    yield 1;
    yield 2;
    yield 3;
    return "Done";
}

const gen = numberGenerator();
console.log(gen.next()); // {value: 1, done: false}
console.log(gen.next()); // {value: 2, done: false}
console.log(gen.next()); // {value: 3, done: false}
console.log(gen.next()); // {value: "Done", done: true}

// Infinite generator
function* fibonacci() {
    let [a, b] = [0, 1];
    while (true) {
        yield a;
        [a, b] = [b, a + b];
    }
}

const fib = fibonacci();
for (let i = 0; i < 10; i++) {
    console.log(fib.next().value);
}

// Async generators
async function* asyncNumberGenerator() {
    for (let i = 1; i <= 3; i++) {
        await new Promise(resolve => setTimeout(resolve, 1000));
        yield i;
    }
}

async function consumeAsyncGenerator() {
    for await (const num of asyncNumberGenerator()) {
        console.log("Generated:", num);
    }
}

// Custom async iterator
class AsyncRange {
    constructor(start, end) {
        this.start = start;
        this.end = end;
    }
    
    [Symbol.asyncIterator]() {
        let current = this.start;
        const end = this.end;
        
        return {
            async next() {
                if (current <= end) {
                    await new Promise(resolve => setTimeout(resolve, 100));
                    return { value: current++, done: false };
                } else {
                    return { done: true };
                }
            }
        };
    }
}

async function useAsyncRange() {
    for await (const num of new AsyncRange(1, 5)) {
        console.log(num);
    }
}
```

## ES6+ Features

### Destructuring

```javascript
// Array destructuring
const [a, b, ...rest] = [1, 2, 3, 4, 5];
console.log(a, b, rest); // 1 2 [3, 4, 5]

// Object destructuring
const person = { name: "John", age: 30, city: "NYC" };
const { name, age, country = "USA" } = person;
console.log(name, age, country); // John 30 USA

// Nested destructuring
const user = {
    id: 1,
    profile: {
        name: "John",
        contact: {
            email: "john@example.com"
        }
    }
};

const {
    profile: {
        name: userName,
        contact: { email }
    }
} = user;

// Function parameter destructuring
function greetUser({ name, age = 0 }) {
    return `Hello ${name}, you are ${age} years old`;
}

greetUser({ name: "John", age: 30 });

// Swapping variables
let x = 1, y = 2;
[x, y] = [y, x]; // x = 2, y = 1
```

### Template Literals

```javascript
const name = "John";
const age = 30;

// Basic template literal
const greeting = `Hello, my name is ${name} and I'm ${age} years old.`;

// Multi-line strings
const html = `
    <div>
        <h1>${name}</h1>
        <p>Age: ${age}</p>
    </div>
`;

// Tagged template literals
function highlight(strings, ...values) {
    return strings.reduce((result, string, i) => {
        const value = values[i] ? `<mark>${values[i]}</mark>` : '';
        return result + string + value;
    }, '');
}

const highlighted = highlight`Hello ${name}, you are ${age} years old!`;
console.log(highlighted); // Hello <mark>John</mark>, you are <mark>30</mark> years old!

// Raw strings
function raw(strings, ...values) {
    console.log(strings.raw); // Access raw strings
    return String.raw(strings, ...values);
}

const path = raw`C:\Users\${name}\Documents`; // C:\Users\John\Documents
```

### Symbols

```javascript
// Creating symbols
const sym1 = Symbol();
const sym2 = Symbol('description');
const sym3 = Symbol('description');

console.log(sym2 === sym3); // false - symbols are always unique

// Global symbol registry
const globalSym1 = Symbol.for('app.id');
const globalSym2 = Symbol.for('app.id');
console.log(globalSym1 === globalSym2); // true

// Well-known symbols
class MyClass {
    constructor(items) {
        this.items = items;
    }
    
    // Custom iterator
    *[Symbol.iterator]() {
        for (const item of this.items) {
            yield item;
        }
    }
    
    // Custom string representation
    [Symbol.toPrimitive](hint) {
        if (hint === 'number') {
            return this.items.length;
        }
        return this.items.join(', ');
    }
    
    // Custom toString
    [Symbol.toStringTag] = 'MyClass';
}

const myInstance = new MyClass(['a', 'b', 'c']);
for (const item of myInstance) {
    console.log(item); // a, b, c
}

console.log(String(myInstance)); // "a, b, c"
console.log(Number(myInstance)); // 3
console.log(Object.prototype.toString.call(myInstance)); // [object MyClass]

// Private properties with symbols
const _private = Symbol('private');

class SecureClass {
    constructor() {
        this[_private] = 'secret data';
        this.public = 'public data';
    }
    
    getPrivate() {
        return this[_private];
    }
}
```

### Maps and Sets

```javascript
// Map - key-value pairs with any type of key
const map = new Map();

map.set('string', 'value');
map.set(42, 'number key');
map.set(true, 'boolean key');
map.set({id: 1}, 'object key');

console.log(map.get('string')); // 'value'
console.log(map.has(42)); // true
console.log(map.size); // 4

// Iteration
for (const [key, value] of map) {
    console.log(key, value);
}

// WeakMap - garbage-collectable keys
const weakMap = new WeakMap();
let obj = {id: 1};

weakMap.set(obj, 'associated data');
console.log(weakMap.get(obj)); // 'associated data'

obj = null; // Object can be garbage collected

// Set - unique values
const set = new Set([1, 2, 3, 3, 4]);
console.log([...set]); // [1, 2, 3, 4]

set.add(5);
set.delete(1);
console.log(set.has(2)); // true

// Set operations
const set1 = new Set([1, 2, 3]);
const set2 = new Set([3, 4, 5]);

// Union
const union = new Set([...set1, ...set2]); // {1, 2, 3, 4, 5}

// Intersection
const intersection = new Set([...set1].filter(x => set2.has(x))); // {3}

// Difference
const difference = new Set([...set1].filter(x => !set2.has(x))); // {1, 2}

// WeakSet - garbage-collectable values
const weakSet = new WeakSet();
let element = document.querySelector('#my-element');

weakSet.add(element);
console.log(weakSet.has(element)); // true

element = null; // Can be garbage collected
```

### Proxies

```javascript
// Basic proxy
const target = {
    name: 'John',
    age: 30
};

const proxy = new Proxy(target, {
    get(target, property) {
        console.log(`Getting ${property}`);
        return target[property];
    },
    
    set(target, property, value) {
        console.log(`Setting ${property} to ${value}`);
        target[property] = value;
        return true;
    },
    
    has(target, property) {
        console.log(`Checking if ${property} exists`);
        return property in target;
    }
});

console.log(proxy.name); // "Getting name" then "John"
proxy.age = 31; // "Setting age to 31"
console.log('name' in proxy); // "Checking if name exists" then true

// Validation proxy
function createValidatedObject(target, validators) {
    return new Proxy(target, {
        set(target, property, value) {
            if (validators[property]) {
                if (!validators[property](value)) {
                    throw new Error(`Invalid value for ${property}`);
                }
            }
            target[property] = value;
            return true;
        }
    });
}

const user = createValidatedObject({}, {
    age: value => typeof value === 'number' && value >= 0,
    email: value => typeof value === 'string' && value.includes('@')
});

user.age = 25; // OK
user.email = 'john@example.com'; // OK
// user.age = -5; // Error: Invalid value for age

// Array-like proxy
function createArrayLike() {
    return new Proxy({}, {
        get(target, property) {
            if (property === 'length') {
                return Object.keys(target).length;
            }
            return target[property];
        },
        
        set(target, property, value) {
            target[property] = value;
            return true;
        }
    });
}

const arrayLike = createArrayLike();
arrayLike[0] = 'first';
arrayLike[1] = 'second';
console.log(arrayLike.length); // 2
```

## Design Patterns

### Singleton Pattern

```javascript
class Singleton {
    constructor() {
        if (Singleton.instance) {
            return Singleton.instance;
        }
        
        this.data = {};
        Singleton.instance = this;
    }
    
    setData(key, value) {
        this.data[key] = value;
    }
    
    getData(key) {
        return this.data[key];
    }
}

// Usage
const instance1 = new Singleton();
const instance2 = new Singleton();

console.log(instance1 === instance2); // true

// Module-based singleton
const ConfigManager = (function() {
    let instance;
    
    function createInstance() {
        return {
            config: {},
            set(key, value) {
                this.config[key] = value;
            },
            get(key) {
                return this.config[key];
            }
        };
    }
    
    return {
        getInstance() {
            if (!instance) {
                instance = createInstance();
            }
            return instance;
        }
    };
})();
```

### Observer Pattern

```javascript
class EventEmitter {
    constructor() {
        this.events = {};
    }
    
    on(event, callback) {
        if (!this.events[event]) {
            this.events[event] = [];
        }
        this.events[event].push(callback);
    }
    
    off(event, callback) {
        if (this.events[event]) {
            this.events[event] = this.events[event].filter(cb => cb !== callback);
        }
    }
    
    emit(event, data) {
        if (this.events[event]) {
            this.events[event].forEach(callback => callback(data));
        }
    }
    
    once(event, callback) {
        const onceCallback = (data) => {
            callback(data);
            this.off(event, onceCallback);
        };
        this.on(event, onceCallback);
    }
}

// Usage
const emitter = new EventEmitter();

emitter.on('data', (data) => console.log('Received:', data));
emitter.on('data', (data) => console.log('Also received:', data));

emitter.emit('data', 'Hello World');
// Output:
// Received: Hello World
// Also received: Hello World

// Observable pattern
class Observable {
    constructor() {
        this.observers = [];
    }
    
    subscribe(observer) {
        this.observers.push(observer);
        
        // Return unsubscribe function
        return () => {
            this.observers = this.observers.filter(obs => obs !== observer);
        };
    }
    
    notify(data) {
        this.observers.forEach(observer => observer(data));
    }
}

const observable = new Observable();
const unsubscribe = observable.subscribe(data => console.log('Observer 1:', data));
observable.subscribe(data => console.log('Observer 2:', data));

observable.notify('Hello Observers!');
unsubscribe(); // Remove first observer
observable.notify('Hello Remaining Observer!');
```

### Factory Pattern

```javascript
// Simple factory
class Car {
    constructor(make, model) {
        this.make = make;
        this.model = model;
    }
    
    start() {
        return `${this.make} ${this.model} started`;
    }
}

class Motorcycle {
    constructor(make, model) {
        this.make = make;
        this.model = model;
    }
    
    start() {
        return `${this.make} ${this.model} motorcycle started`;
    }
}

class VehicleFactory {
    static createVehicle(type, make, model) {
        switch (type) {
            case 'car':
                return new Car(make, model);
            case 'motorcycle':
                return new Motorcycle(make, model);
            default:
                throw new Error('Unknown vehicle type');
        }
    }
}

const car = VehicleFactory.createVehicle('car', 'Toyota', 'Camry');
const bike = VehicleFactory.createVehicle('motorcycle', 'Honda', 'CBR');

// Abstract factory
class UIFactory {
    createButton() {
        throw new Error('Must implement createButton');
    }
    
    createInput() {
        throw new Error('Must implement createInput');
    }
}

class MaterialUIFactory extends UIFactory {
    createButton() {
        return {
            type: 'material-button',
            render: () => '<button class="material-btn">Click me</button>'
        };
    }
    
    createInput() {
        return {
            type: 'material-input',
            render: () => '<input class="material-input" />'
        };
    }
}

class BootstrapUIFactory extends UIFactory {
    createButton() {
        return {
            type: 'bootstrap-button',
            render: () => '<button class="btn btn-primary">Click me</button>'
        };
    }
    
    createInput() {
        return {
            type: 'bootstrap-input',
            render: () => '<input class="form-control" />'
        };
    }
}
```

### Module Pattern

```javascript
// Revealing module pattern
const Calculator = (function() {
    let result = 0;
    
    function add(x) {
        result += x;
        return this;
    }
    
    function subtract(x) {
        result -= x;
        return this;
    }
    
    function multiply(x) {
        result *= x;
        return this;
    }
    
    function divide(x) {
        if (x !== 0) {
            result /= x;
        }
        return this;
    }
    
    function getResult() {
        return result;
    }
    
    function reset() {
        result = 0;
        return this;
    }
    
    // Public API
    return {
        add,
        subtract,
        multiply,
        divide,
        getResult,
        reset
    };
})();

// Usage
const result = Calculator
    .add(10)
    .multiply(2)
    .subtract(5)
    .getResult(); // 15

// ES6 Module pattern
export class ModernCalculator {
    #result = 0;
    
    add(x) {
        this.#result += x;
        return this;
    }
    
    subtract(x) {
        this.#result -= x;
        return this;
    }
    
    getResult() {
        return this.#result;
    }
    
    reset() {
        this.#result = 0;
        return this;
    }
}
```

## Memory Management

### Garbage Collection

```javascript
// Reference counting (old method)
let obj1 = { name: 'Object 1' };
let obj2 = { name: 'Object 2' };

obj1.ref = obj2; // obj1 references obj2
obj2.ref = obj1; // obj2 references obj1 (circular reference)

// Setting to null doesn't immediately free memory in reference counting
obj1 = null;
obj2 = null; // Circular reference prevents garbage collection

// Mark-and-sweep (modern method)
// Objects are marked as reachable from root, unreachable objects are swept

// Memory leak examples
function memoryLeak1() {
    // Accidental global variable
    leakedVar = 'This creates a global variable';
}

function memoryLeak2() {
    let element = document.getElementById('button');
    
    // Event listener keeps reference to element
    element.addEventListener('click', function() {
        // Even if element is removed from DOM, it stays in memory
        console.log('Clicked');
    });
    
    // Better: remove event listener
    // element.removeEventListener('click', handler);
}

function memoryLeak3() {
    let timer = setInterval(function() {
        // Timer keeps reference to outer scope
        console.log('Timer running');
    }, 1000);
    
    // Don't forget to clear timer
    // clearInterval(timer);
}

// Proper cleanup
class ComponentWithCleanup {
    constructor() {
        this.timer = null;
        this.eventHandler = this.handleEvent.bind(this);
    }
    
    init() {
        this.timer = setInterval(() => {
            this.update();
        }, 1000);
        
        document.addEventListener('click', this.eventHandler);
    }
    
    handleEvent(event) {
        console.log('Event handled');
    }
    
    update() {
        console.log('Component updated');
    }
    
    destroy() {
        if (this.timer) {
            clearInterval(this.timer);
            this.timer = null;
        }
        
        document.removeEventListener('click', this.eventHandler);
    }
}
```

### WeakMap and WeakSet for Memory Management

```javascript
// WeakMap for private data
const privateData = new WeakMap();

class User {
    constructor(name, ssn) {
        this.name = name;
        // Store private data in WeakMap
        privateData.set(this, { ssn });
    }
    
    getSSN() {
        return privateData.get(this).ssn;
    }
}

let user = new User('John', '123-45-6789');
console.log(user.getSSN()); // '123-45-6789'

// When user is set to null, private data is automatically garbage collected
user = null;

// WeakSet for tracking objects
const processedObjects = new WeakSet();

function processObject(obj) {
    if (processedObjects.has(obj)) {
        console.log('Already processed');
        return;
    }
    
    // Process the object
    console.log('Processing object');
    processedObjects.add(obj);
}

let myObj = { id: 1 };
processObject(myObj); // "Processing object"
processObject(myObj); // "Already processed"

myObj = null; // Object and its WeakSet entry can be garbage collected
```

## Performance Optimization

### Debouncing and Throttling

```javascript
// Debouncing - delay execution until after wait time
function debounce(func, wait, immediate = false) {
    let timeout;
    
    return function executedFunction(...args) {
        const later = () => {
            timeout = null;
            if (!immediate) func.apply(this, args);
        };
        
        const callNow = immediate && !timeout;
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
        
        if (callNow) func.apply(this, args);
    };
}

// Usage
const debouncedSearch = debounce(function(query) {
    console.log('Searching for:', query);
    // Perform search API call
}, 300);

// Throttling - limit execution to once per interval
function throttle(func, limit) {
    let inThrottle;
    
    return function executedFunction(...args) {
        if (!inThrottle) {
            func.apply(this, args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}

// Usage
const throttledScroll = throttle(function() {
    console.log('Scroll event handled');
}, 100);

window.addEventListener('scroll', throttledScroll);

// Advanced throttling with leading and trailing options
function advancedThrottle(func, wait, options = {}) {
    let timeout, context, args, result;
    let previous = 0;
    
    const later = function() {
        previous = options.leading === false ? 0 : Date.now();
        timeout = null;
        result = func.apply(context, args);
        if (!timeout) context = args = null;
    };
    
    const throttled = function() {
        const now = Date.now();
        if (!previous && options.leading === false) previous = now;
        const remaining = wait - (now - previous);
        context = this;
        args = arguments;
        
        if (remaining <= 0 || remaining > wait) {
            if (timeout) {
                clearTimeout(timeout);
                timeout = null;
            }
            previous = now;
            result = func.apply(context, args);
            if (!timeout) context = args = null;
        } else if (!timeout && options.trailing !== false) {
            timeout = setTimeout(later, remaining);
        }
        
        return result;
    };
    
    throttled.cancel = function() {
        clearTimeout(timeout);
        previous = 0;
        timeout = context = args = null;
    };
    
    return throttled;
}
```

### Memoization

```javascript
// Simple memoization
function memoize(fn) {
    const cache = new Map();
    
    return function(...args) {
        const key = JSON.stringify(args);
        
        if (cache.has(key)) {
            console.log('Cache hit');
            return cache.get(key);
        }
        
        console.log('Computing result');
        const result = fn.apply(this, args);
        cache.set(key, result);
        return result;
    };
}

// Usage
const expensiveFunction = memoize(function(n) {
    let result = 0;
    for (let i = 0; i < n; i++) {
        result += i;
    }
    return result;
});

console.log(expensiveFunction(1000000)); // Computing result
console.log(expensiveFunction(1000000)); // Cache hit

// Fibonacci with memoization
const fibonacciMemo = memoize(function(n) {
    if (n <= 1) return n;
    return fibonacciMemo(n - 1) + fibonacciMemo(n - 2);
});

console.log(fibonacciMemo(40)); // Much faster than naive recursion

// LRU Cache implementation
class LRUCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.cache = new Map();
    }
    
    get(key) {
        if (this.cache.has(key)) {
            // Move to end (most recently used)
            const value = this.cache.get(key);
            this.cache.delete(key);
            this.cache.set(key, value);
            return value;
        }
        return -1;
    }
    
    put(key, value) {
        if (this.cache.has(key)) {
            // Update existing key
            this.cache.delete(key);
        } else if (this.cache.size >= this.capacity) {
            // Remove least recently used (first item)
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
        }
        
        this.cache.set(key, value);
    }
}

// Memoization with LRU cache
function memoizeWithLRU(fn, capacity = 100) {
    const cache = new LRUCache(capacity);
    
    return function(...args) {
        const key = JSON.stringify(args);
        
        let result = cache.get(key);
        if (result === -1) {
            result = fn.apply(this, args);
            cache.put(key, result);
        }
        
        return result;
    };
}
```

### Lazy Loading and Code Splitting

```javascript
// Lazy loading with dynamic imports
async function loadModule() {
    try {
        const module = await import('./heavy-module.js');
        return module.default;
    } catch (error) {
        console.error('Failed to load module:', error);
    }
}

// Lazy loading with intersection observer
class LazyLoader {
    constructor() {
        this.observer = new IntersectionObserver(
            this.handleIntersection.bind(this),
            { threshold: 0.1 }
        );
    }
    
    observe(element) {
        this.observer.observe(element);
    }
    
    handleIntersection(entries) {
        entries.forEach(entry => {
            if (entry.isIntersecting) {
                this.loadContent(entry.target);
                this.observer.unobserve(entry.target);
            }
        });
    }
    
    async loadContent(element) {
        const src = element.dataset.src;
        if (src) {
            // Load image
            if (element.tagName === 'IMG') {
                element.src = src;
            }
            // Load component
            else if (element.dataset.component) {
                const component = await import(src);
                element.innerHTML = component.render();
            }
        }
    }
}

// Usage
const lazyLoader = new LazyLoader();
document.querySelectorAll('[data-src]').forEach(el => {
    lazyLoader.observe(el);
});

// Virtual scrolling for large lists
class VirtualList {
    constructor(container, items, itemHeight) {
        this.container = container;
        this.items = items;
        this.itemHeight = itemHeight;
        this.visibleCount = Math.ceil(container.clientHeight / itemHeight) + 1;
        this.startIndex = 0;
        
        this.setupContainer();
        this.render();
        this.bindEvents();
    }
    
    setupContainer() {
        this.container.style.height = `${this.items.length * this.itemHeight}px`;
        this.container.style.overflow = 'auto';
        this.container.style.position = 'relative';
    }
    
    render() {
        const fragment = document.createDocumentFragment();
        
        for (let i = 0; i < this.visibleCount && i + this.startIndex < this.items.length; i++) {
            const index = i + this.startIndex;
            const item = this.createItem(this.items[index], index);
            fragment.appendChild(item);
        }
        
        this.container.innerHTML = '';
        this.container.appendChild(fragment);
    }
    
    createItem(data, index) {
        const item = document.createElement('div');
        item.style.position = 'absolute';
        item.style.top = `${index * this.itemHeight}px`;
        item.style.height = `${this.itemHeight}px`;
        item.textContent = data;
        return item;
    }
    
    bindEvents() {
        this.container.addEventListener('scroll', () => {
            const newStartIndex = Math.floor(this.container.scrollTop / this.itemHeight);
            if (newStartIndex !== this.startIndex) {
                this.startIndex = newStartIndex;
                this.render();
            }
        });
    }
}
```

## Advanced Function Concepts

### Function Composition and Piping

```javascript
// Function composition
const compose = (...fns) => (value) => fns.reduceRight((acc, fn) => fn(acc), value);

// Function piping
const pipe = (...fns) => (value) => fns.reduce((acc, fn) => fn(acc), value);

// Example functions
const add = (x) => (y) => x + y;
const multiply = (x) => (y) => x * y;
const square = (x) => x * x;

// Usage
const add5 = add(5);
const multiply2 = multiply(2);

const composedFunction = compose(
    square,
    multiply2,
    add5
);

const pipedFunction = pipe(
    add5,
    multiply2,
    square
);

console.log(composedFunction(3)); // square(multiply2(add5(3))) = square(16) = 256
console.log(pipedFunction(3));    // square(multiply2(add5(3))) = square(16) = 256

// Point-free style
const numbers = [1, 2, 3, 4, 5];

const processNumbers = pipe(
    arr => arr.map(add(1)),
    arr => arr.filter(x => x % 2 === 0),
    arr => arr.reduce((sum, x) => sum + x, 0)
);

console.log(processNumbers(numbers)); // 12
```

### Partial Application and Currying

```javascript
// Partial application
function partial(fn, ...partialArgs) {
    return function(...remainingArgs) {
        return fn(...partialArgs, ...remainingArgs);
    };
}

function greet(greeting, punctuation, name) {
    return `${greeting} ${name}${punctuation}`;
}

const sayHello = partial(greet, 'Hello', '!');
console.log(sayHello('World')); // "Hello World!"

// Auto-currying
function curry(fn) {
    return function curried(...args) {
        if (args.length >= fn.length) {
            return fn.apply(this, args);
        } else {
            return function(...nextArgs) {
                return curried.apply(this, args.concat(nextArgs));
            };
        }
    };
}

const add3 = curry((a, b, c) => a + b + c);

console.log(add3(1)(2)(3)); // 6
console.log(add3(1, 2)(3)); // 6
console.log(add3(1)(2, 3)); // 6

// Practical currying example
const fetch = curry((method, url, options) => {
    return window.fetch(url, { method, ...options });
});

const get = fetch('GET');
const post = fetch('POST');

// Usage
get('/api/users').then(response => response.json());
post('/api/users', { body: JSON.stringify(userData) });
```

### Function Decorators

```javascript
// Timing decorator
function timing(fn) {
    return function(...args) {
        const start = performance.now();
        const result = fn.apply(this, args);
        const end = performance.now();
        console.log(`${fn.name} took ${end - start} milliseconds`);
        return result;
    };
}

// Retry decorator
function retry(maxAttempts, delay = 1000) {
    return function decorator(fn) {
        return async function(...args) {
            let attempts = 0;
            
            while (attempts < maxAttempts) {
                try {
                    return await fn.apply(this, args);
                } catch (error) {
                    attempts++;
                    if (attempts >= maxAttempts) {
                        throw error;
                    }
                    await new Promise(resolve => setTimeout(resolve, delay));
                }
            }
        };
    };
}

// Cache decorator
function cache(ttl = 60000) {
    return function decorator(fn) {
        const cache = new Map();
        
        return function(...args) {
            const key = JSON.stringify(args);
            const cached = cache.get(key);
            
            if (cached && Date.now() - cached.timestamp < ttl) {
                return cached.value;
            }
            
            const result = fn.apply(this, args);
            cache.set(key, { value: result, timestamp: Date.now() });
            
            return result;
        };
    };
}

// Usage
const expensiveOperation = timing(cache(5000)(retry(3)(function(n) {
    if (Math.random() < 0.3) {
        throw new Error('Random failure');
    }
    return n * n;
})));

// Class method decorators (using proposal syntax)
class APIClient {
    @retry(3, 2000)
    @cache(30000)
    async fetchUser(id) {
        const response = await fetch(`/api/users/${id}`);
        if (!response.ok) {
            throw new Error('Failed to fetch user');
        }
        return response.json();
    }
}
```

## Metaprogramming

### Reflect API

```javascript
// Reflect methods
const obj = { name: 'John', age: 30 };

// Reflect.get - get property
console.log(Reflect.get(obj, 'name')); // 'John'

// Reflect.set - set property
Reflect.set(obj, 'city', 'NYC');
console.log(obj.city); // 'NYC'

// Reflect.has - check property existence
console.log(Reflect.has(obj, 'name')); // true

// Reflect.deleteProperty - delete property
Reflect.deleteProperty(obj, 'age');
console.log(obj.age); // undefined

// Reflect.ownKeys - get all keys
console.log(Reflect.ownKeys(obj)); // ['name', 'city']

// Reflect.construct - create instance
function Person(name) {
    this.name = name;
}

const person = Reflect.construct(Person, ['John']);
console.log(person.name); // 'John'

// Reflect.apply - call function
function greet(greeting, punctuation) {
    return `${greeting} ${this.name}${punctuation}`;
}

const result = Reflect.apply(greet, { name: 'World' }, ['Hello', '!']);
console.log(result); // 'Hello World!'
```

### Dynamic Property Access

```javascript
// Dynamic property names
const propName = 'dynamicProp';
const obj = {
    [propName]: 'dynamic value',
    [`computed_${Date.now()}`]: 'timestamp property'
};

// Property descriptors
const descriptor = {
    value: 'controlled value',
    writable: false,
    enumerable: true,
    configurable: false
};

Object.defineProperty(obj, 'controlledProp', descriptor);

// Property getters and setters
class SmartObject {
    constructor() {
        this._data = {};
    }
    
    // Dynamic getter/setter creation
    createProperty(name, initialValue) {
        this._data[name] = initialValue;
        
        Object.defineProperty(this, name, {
            get() {
                console.log(`Getting ${name}`);
                return this._data[name];
            },
            set(value) {
                console.log(`Setting ${name} to ${value}`);
                this._data[name] = value;
            }
        });
    }
}

const smart = new SmartObject();
smart.createProperty('name', 'John');
console.log(smart.name); // "Getting name" then "John"
smart.name = 'Jane'; // "Setting name to Jane"
```

### Code Generation and Evaluation

```javascript
// Function constructor
const dynamicFunction = new Function('a', 'b', 'return a + b');
console.log(dynamicFunction(2, 3)); // 5

// Template-based code generation
function createValidator(rules) {
    const conditions = Object.entries(rules)
        .map(([field, rule]) => {
            switch (rule.type) {
                case 'required':
                    return `if (!obj.${field}) errors.push('${field} is required')`;
                case 'minLength':
                    return `if (obj.${field} && obj.${field}.length < ${rule.value}) errors.push('${field} must be at least ${rule.value} characters')`;
                case 'email':
                    return `if (obj.${field} && !/^[^@]+@[^@]+\.[^@]+$/.test(obj.${field})) errors.push('${field} must be a valid email')`;
                default:
                    return '';
            }
        })
        .filter(Boolean)
        .join(';\n');
    
    const functionBody = `
        const errors = [];
        ${conditions};
        return errors;
    `;
    
    return new Function('obj', functionBody);
}

const validator = createValidator({
    name: { type: 'required' },
    email: { type: 'email' },
    password: { type: 'minLength', value: 8 }
});

console.log(validator({ name: '', email: 'invalid', password: '123' }));
// ['name is required', 'email must be a valid email', 'password must be at least 8 characters']

// AST manipulation (conceptual example)
function transformCode(code) {
    // This would typically use a proper parser like Babel
    return code
        .replace(/console\.log/g, 'logger.info')
        .replace(/var /g, 'let ');
}

const originalCode = `
    var message = 'Hello World';
    console.log(message);
`;

const transformedCode = transformCode(originalCode);
console.log(transformedCode);
// let message = 'Hello World';
// logger.info(message);
```

---

This completes the Advanced JavaScript Concepts section. The content covers prototypes, closures, async programming, ES6+ features, design patterns, memory management, performance optimization, advanced function concepts, and metaprogramming.
