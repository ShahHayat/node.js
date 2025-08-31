# JavaScript Fundamentals - Complete Guide

## Table of Contents
1. [Introduction to JavaScript](#introduction-to-javascript)
2. [JavaScript Engine Architecture](#javascript-engine-architecture)
3. [Variables and Data Types](#variables-and-data-types)
4. [Operators](#operators)
5. [Control Structures](#control-structures)
6. [Functions](#functions)
7. [Objects and Arrays](#objects-and-arrays)
8. [Scope and Hoisting](#scope-and-hoisting)
9. [Error Handling](#error-handling)

## Introduction to JavaScript

JavaScript is a high-level, interpreted programming language that was initially created to make web pages interactive. Today, it's one of the most popular programming languages in the world, used for:

- **Frontend Development**: Interactive web applications
- **Backend Development**: Server-side applications (Node.js)
- **Mobile Development**: React Native, Ionic
- **Desktop Applications**: Electron
- **Game Development**: Browser-based games
- **IoT and Embedded Systems**: Johnny-Five, Espruino

### Key Characteristics

- **Interpreted Language**: No compilation step required
- **Dynamic Typing**: Variables can hold different types of values
- **Prototype-based**: Object-oriented programming through prototypes
- **First-class Functions**: Functions are treated as values
- **Event-driven**: Responds to user interactions and system events
- **Non-blocking**: Asynchronous execution model

## JavaScript Engine Architecture

JavaScript engines are responsible for parsing, compiling, and executing JavaScript code. Popular engines include:

- **V8** (Chrome, Node.js)
- **SpiderMonkey** (Firefox)
- **JavaScriptCore** (Safari)
- **Chakra** (Internet Explorer/Edge)

### Engine Components

1. **Parser**: Converts source code into Abstract Syntax Tree (AST)
2. **Interpreter**: Executes code line by line
3. **Compiler**: Optimizes frequently used code (JIT compilation)
4. **Memory Heap**: Stores objects and variables
5. **Call Stack**: Manages function execution context
6. **Garbage Collector**: Automatically manages memory

## Variables and Data Types

### Variable Declarations

```javascript
// var - function-scoped, hoisted, can be redeclared
var name = "John";
var name = "Jane"; // OK

// let - block-scoped, hoisted but not initialized, cannot be redeclared
let age = 25;
// let age = 30; // Error

// const - block-scoped, must be initialized, cannot be reassigned
const PI = 3.14159;
// PI = 3.14; // Error
```

### Primitive Data Types

#### 1. Number
```javascript
let integer = 42;
let float = 3.14;
let scientific = 2.5e6; // 2500000
let binary = 0b1010; // 10
let octal = 0o755; // 493
let hex = 0xFF; // 255

// Special numeric values
let infinity = Infinity;
let negInfinity = -Infinity;
let notANumber = NaN;

// Number methods
Number.isInteger(42); // true
Number.parseFloat("3.14"); // 3.14
Number.parseInt("42px"); // 42
```

#### 2. String
```javascript
let single = 'Hello';
let double = "World";
let template = `Hello ${name}!`;

// String methods
let text = "JavaScript";
text.length; // 10
text.charAt(0); // "J"
text.indexOf("Script"); // 4
text.slice(0, 4); // "Java"
text.substring(4, 10); // "Script"
text.toLowerCase(); // "javascript"
text.toUpperCase(); // "JAVASCRIPT"
text.split(""); // ["J", "a", "v", "a", "S", "c", "r", "i", "p", "t"]
```

#### 3. Boolean
```javascript
let isTrue = true;
let isFalse = false;

// Falsy values in JavaScript
false, 0, -0, 0n, "", null, undefined, NaN

// Truthy values (everything else)
true, 1, "hello", [], {}, function() {}
```

#### 4. Null and Undefined
```javascript
let empty = null; // Intentional absence of value
let notDefined; // undefined (declared but not assigned)

console.log(typeof null); // "object" (historical bug)
console.log(typeof undefined); // "undefined"
```

#### 5. Symbol (ES6)
```javascript
let sym1 = Symbol();
let sym2 = Symbol("description");
let sym3 = Symbol.for("global"); // Global symbol registry

// Symbols are always unique
Symbol() === Symbol(); // false
```

#### 6. BigInt (ES2020)
```javascript
let bigNumber = 123456789012345678901234567890n;
let anotherBig = BigInt("123456789012345678901234567890");

// Cannot mix BigInt with regular numbers
// bigNumber + 10; // Error
bigNumber + 10n; // OK
```

### Non-Primitive Data Types

#### Objects
```javascript
// Object literal
let person = {
    name: "John",
    age: 30,
    city: "New York",
    greet: function() {
        return `Hello, I'm ${this.name}`;
    }
};

// Accessing properties
person.name; // "John"
person["age"]; // 30

// Adding properties
person.email = "john@example.com";

// Deleting properties
delete person.city;
```

#### Arrays
```javascript
let numbers = [1, 2, 3, 4, 5];
let mixed = [1, "hello", true, null, {name: "John"}];

// Array methods
numbers.push(6); // Add to end
numbers.pop(); // Remove from end
numbers.unshift(0); // Add to beginning
numbers.shift(); // Remove from beginning
numbers.slice(1, 3); // Extract portion
numbers.splice(2, 1, "new"); // Remove/add elements
```

## Operators

### Arithmetic Operators
```javascript
let a = 10, b = 3;

a + b; // 13 (Addition)
a - b; // 7 (Subtraction)
a * b; // 30 (Multiplication)
a / b; // 3.333... (Division)
a % b; // 1 (Modulus)
a ** b; // 1000 (Exponentiation)

// Increment/Decrement
++a; // Pre-increment
a++; // Post-increment
--a; // Pre-decrement
a--; // Post-decrement
```

### Comparison Operators
```javascript
let x = 5, y = "5";

x == y;  // true (loose equality, type coercion)
x === y; // false (strict equality, no type coercion)
x != y;  // false
x !== y; // true
x > 3;   // true
x >= 5;  // true
x < 10;  // true
x <= 5;  // true
```

### Logical Operators
```javascript
let p = true, q = false;

p && q; // false (AND)
p || q; // true (OR)
!p;     // false (NOT)

// Short-circuit evaluation
let result = p && "This will execute";
let fallback = q || "Default value";
```

### Assignment Operators
```javascript
let num = 10;

num += 5;  // num = num + 5 (15)
num -= 3;  // num = num - 3 (12)
num *= 2;  // num = num * 2 (24)
num /= 4;  // num = num / 4 (6)
num %= 4;  // num = num % 4 (2)
num **= 3; // num = num ** 3 (8)
```

### Bitwise Operators
```javascript
let a = 5;  // 101 in binary
let b = 3;  // 011 in binary

a & b;  // 1 (001) - AND
a | b;  // 7 (111) - OR
a ^ b;  // 6 (110) - XOR
~a;     // -6 - NOT
a << 1; // 10 (1010) - Left shift
a >> 1; // 2 (10) - Right shift
a >>> 1; // 2 - Unsigned right shift
```

## Control Structures

### Conditional Statements

#### if...else
```javascript
let score = 85;

if (score >= 90) {
    console.log("A grade");
} else if (score >= 80) {
    console.log("B grade");
} else if (score >= 70) {
    console.log("C grade");
} else {
    console.log("Need improvement");
}
```

#### Ternary Operator
```javascript
let age = 20;
let status = age >= 18 ? "Adult" : "Minor";

// Nested ternary
let grade = score >= 90 ? "A" : score >= 80 ? "B" : "C";
```

#### switch Statement
```javascript
let day = "Monday";

switch (day) {
    case "Monday":
        console.log("Start of work week");
        break;
    case "Tuesday":
    case "Wednesday":
    case "Thursday":
        console.log("Midweek");
        break;
    case "Friday":
        console.log("TGIF!");
        break;
    case "Saturday":
    case "Sunday":
        console.log("Weekend!");
        break;
    default:
        console.log("Invalid day");
}
```

### Loops

#### for Loop
```javascript
// Traditional for loop
for (let i = 0; i < 5; i++) {
    console.log(i);
}

// for...in (iterates over object keys)
let obj = {a: 1, b: 2, c: 3};
for (let key in obj) {
    console.log(key, obj[key]);
}

// for...of (iterates over iterable values)
let arr = [1, 2, 3, 4, 5];
for (let value of arr) {
    console.log(value);
}
```

#### while Loop
```javascript
let count = 0;
while (count < 5) {
    console.log(count);
    count++;
}
```

#### do...while Loop
```javascript
let num = 0;
do {
    console.log(num);
    num++;
} while (num < 5);
```

#### Loop Control
```javascript
for (let i = 0; i < 10; i++) {
    if (i === 3) continue; // Skip iteration
    if (i === 7) break;    // Exit loop
    console.log(i);
}
```

## Functions

### Function Declarations
```javascript
// Function declaration (hoisted)
function greet(name) {
    return `Hello, ${name}!`;
}

// Function expression (not hoisted)
const sayHello = function(name) {
    return `Hello, ${name}!`;
};

// Arrow functions (ES6)
const add = (a, b) => a + b;
const square = x => x * x;
const sayHi = () => "Hi!";

// Arrow function with block body
const multiply = (a, b) => {
    const result = a * b;
    return result;
};
```

### Function Parameters

#### Default Parameters
```javascript
function greet(name = "World") {
    return `Hello, ${name}!`;
}

greet(); // "Hello, World!"
greet("John"); // "Hello, John!"
```

#### Rest Parameters
```javascript
function sum(...numbers) {
    return numbers.reduce((total, num) => total + num, 0);
}

sum(1, 2, 3, 4, 5); // 15
```

#### Destructuring Parameters
```javascript
function createUser({name, age, email}) {
    return {
        id: Math.random(),
        name,
        age,
        email,
        createdAt: new Date()
    };
}

createUser({name: "John", age: 30, email: "john@example.com"});
```

### Higher-Order Functions
```javascript
// Function that returns a function
function multiplier(factor) {
    return function(number) {
        return number * factor;
    };
}

const double = multiplier(2);
double(5); // 10

// Function that takes a function as parameter
function processArray(arr, callback) {
    return arr.map(callback);
}

processArray([1, 2, 3], x => x * 2); // [2, 4, 6]
```

### Immediately Invoked Function Expression (IIFE)
```javascript
(function() {
    console.log("IIFE executed!");
})();

// With parameters
(function(name) {
    console.log(`Hello, ${name}!`);
})("World");

// Arrow function IIFE
(() => {
    console.log("Arrow IIFE");
})();
```

## Objects and Arrays

### Object Methods and Properties

#### Object.keys(), Object.values(), Object.entries()
```javascript
let person = {name: "John", age: 30, city: "NYC"};

Object.keys(person);    // ["name", "age", "city"]
Object.values(person);  // ["John", 30, "NYC"]
Object.entries(person); // [["name", "John"], ["age", 30], ["city", "NYC"]]
```

#### Object.assign() and Spread Operator
```javascript
let obj1 = {a: 1, b: 2};
let obj2 = {b: 3, c: 4};

// Object.assign (shallow copy)
let merged1 = Object.assign({}, obj1, obj2); // {a: 1, b: 3, c: 4}

// Spread operator (ES6)
let merged2 = {...obj1, ...obj2}; // {a: 1, b: 3, c: 4}
```

#### Property Descriptors
```javascript
let obj = {};

Object.defineProperty(obj, 'name', {
    value: 'John',
    writable: true,
    enumerable: true,
    configurable: true
});

// Get property descriptor
Object.getOwnPropertyDescriptor(obj, 'name');
```

### Array Methods

#### Mutating Methods
```javascript
let arr = [1, 2, 3];

arr.push(4);        // Add to end: [1, 2, 3, 4]
arr.pop();          // Remove from end: [1, 2, 3]
arr.unshift(0);     // Add to beginning: [0, 1, 2, 3]
arr.shift();        // Remove from beginning: [1, 2, 3]
arr.reverse();      // Reverse: [3, 2, 1]
arr.sort();         // Sort: [1, 2, 3]
arr.splice(1, 1, 'new'); // Remove/add: [1, 'new', 3]
```

#### Non-mutating Methods
```javascript
let numbers = [1, 2, 3, 4, 5];

// map - transform each element
let doubled = numbers.map(x => x * 2); // [2, 4, 6, 8, 10]

// filter - select elements that match condition
let evens = numbers.filter(x => x % 2 === 0); // [2, 4]

// reduce - accumulate values
let sum = numbers.reduce((acc, curr) => acc + curr, 0); // 15

// find - first element that matches
let found = numbers.find(x => x > 3); // 4

// some - check if any element matches
let hasEven = numbers.some(x => x % 2 === 0); // true

// every - check if all elements match
let allPositive = numbers.every(x => x > 0); // true

// includes - check if array contains value
let hasThree = numbers.includes(3); // true

// slice - extract portion
let portion = numbers.slice(1, 3); // [2, 3]

// concat - combine arrays
let combined = numbers.concat([6, 7]); // [1, 2, 3, 4, 5, 6, 7]
```

#### Array Destructuring
```javascript
let arr = [1, 2, 3, 4, 5];

// Basic destructuring
let [first, second] = arr; // first = 1, second = 2

// Skip elements
let [a, , c] = arr; // a = 1, c = 3

// Rest elements
let [head, ...tail] = arr; // head = 1, tail = [2, 3, 4, 5]

// Default values
let [x, y, z = 0] = [1, 2]; // x = 1, y = 2, z = 0
```

## Scope and Hoisting

### Scope Types

#### Global Scope
```javascript
var globalVar = "I'm global";
let globalLet = "I'm also global";

function test() {
    console.log(globalVar); // Accessible
    console.log(globalLet); // Accessible
}
```

#### Function Scope
```javascript
function outer() {
    var functionScoped = "I'm function scoped";
    
    function inner() {
        console.log(functionScoped); // Accessible
    }
    
    inner();
}

// console.log(functionScoped); // Error: not defined
```

#### Block Scope
```javascript
if (true) {
    var varVariable = "var is function scoped";
    let letVariable = "let is block scoped";
    const constVariable = "const is block scoped";
}

console.log(varVariable);    // "var is function scoped"
// console.log(letVariable);    // Error: not defined
// console.log(constVariable);  // Error: not defined
```

### Hoisting

#### Variable Hoisting
```javascript
console.log(hoistedVar); // undefined (not error)
var hoistedVar = "I'm hoisted";

// Equivalent to:
// var hoistedVar;
// console.log(hoistedVar); // undefined
// hoistedVar = "I'm hoisted";

// let and const are hoisted but not initialized
// console.log(letVar); // ReferenceError
let letVar = "I'm let";

// console.log(constVar); // ReferenceError
const constVar = "I'm const";
```

#### Function Hoisting
```javascript
// Function declarations are fully hoisted
console.log(hoistedFunction()); // "I'm hoisted!"

function hoistedFunction() {
    return "I'm hoisted!";
}

// Function expressions are not hoisted
// console.log(notHoisted()); // TypeError
var notHoisted = function() {
    return "I'm not hoisted";
};
```

### Closures
```javascript
function outerFunction(x) {
    // This is the outer function's scope
    
    function innerFunction(y) {
        // Inner function has access to outer function's variables
        return x + y;
    }
    
    return innerFunction;
}

const addFive = outerFunction(5);
console.log(addFive(3)); // 8

// Practical closure example
function counter() {
    let count = 0;
    
    return {
        increment: () => ++count,
        decrement: () => --count,
        getCount: () => count
    };
}

const myCounter = counter();
myCounter.increment(); // 1
myCounter.increment(); // 2
myCounter.getCount();  // 2
```

## Error Handling

### try...catch...finally
```javascript
try {
    // Code that might throw an error
    let result = riskyOperation();
    console.log(result);
} catch (error) {
    // Handle the error
    console.error("An error occurred:", error.message);
} finally {
    // Always executes
    console.log("Cleanup code here");
}
```

### Throwing Custom Errors
```javascript
function divide(a, b) {
    if (b === 0) {
        throw new Error("Division by zero is not allowed");
    }
    return a / b;
}

// Custom error types
class ValidationError extends Error {
    constructor(message) {
        super(message);
        this.name = "ValidationError";
    }
}

function validateAge(age) {
    if (age < 0 || age > 150) {
        throw new ValidationError("Age must be between 0 and 150");
    }
}
```

### Error Types
```javascript
try {
    // Different types of errors
    
    // ReferenceError
    console.log(undefinedVariable);
    
    // TypeError
    null.someMethod();
    
    // SyntaxError (caught at parse time)
    // eval("var x = ;");
    
    // RangeError
    let arr = new Array(-1);
    
} catch (error) {
    console.log(error.name);    // Error type
    console.log(error.message); // Error message
    console.log(error.stack);   // Stack trace
}
```

---

This concludes the JavaScript Fundamentals section. The next sections will cover advanced JavaScript topics, DOM manipulation, and then move into Node.js concepts.
