# JavaScript DOM and Browser APIs - Complete Guide

## Table of Contents
1. [Document Object Model (DOM)](#document-object-model-dom)
2. [Event Handling](#event-handling)
3. [Browser APIs](#browser-apis)
4. [Web Storage](#web-storage)
5. [Fetch API and AJAX](#fetch-api-and-ajax)
6. [Web Workers](#web-workers)
7. [WebSockets](#websockets)
8. [Geolocation API](#geolocation-api)
9. [File API](#file-api)
10. [Canvas API](#canvas-api)
11. [Intersection Observer](#intersection-observer)
12. [Mutation Observer](#mutation-observer)

## Document Object Model (DOM)

The DOM is a programming interface for HTML documents. It represents the page so that programs can change the document structure, style, and content.

### DOM Structure and Navigation

```javascript
// Document object
console.log(document.title);
console.log(document.URL);
console.log(document.domain);
console.log(document.lastModified);

// Document ready states
console.log(document.readyState); // 'loading', 'interactive', or 'complete'

document.addEventListener('DOMContentLoaded', function() {
    console.log('DOM is fully loaded and parsed');
});

window.addEventListener('load', function() {
    console.log('Page is fully loaded including images, stylesheets, etc.');
});

// Element selection
const elementById = document.getElementById('myId');
const elementsByClass = document.getElementsByClassName('myClass');
const elementsByTag = document.getElementsByTagName('div');
const elementByQuery = document.querySelector('.myClass');
const elementsByQueryAll = document.querySelectorAll('div.myClass');

// Modern selection methods (recommended)
const element = document.querySelector('#myId');
const elements = document.querySelectorAll('.myClass');

// Navigation properties
const parent = element.parentNode;
const parentElement = element.parentElement;
const children = element.children;
const childNodes = element.childNodes; // Includes text nodes
const firstChild = element.firstElementChild;
const lastChild = element.lastElementChild;
const nextSibling = element.nextElementSibling;
const previousSibling = element.previousElementSibling;

// Tree walking
function walkDOM(node, callback) {
    callback(node);
    for (let child of node.children) {
        walkDOM(child, callback);
    }
}

walkDOM(document.body, (node) => {
    console.log(node.tagName);
});
```

### Element Manipulation

```javascript
// Creating elements
const newDiv = document.createElement('div');
const newText = document.createTextNode('Hello World');
const newComment = document.createComment('This is a comment');
const fragment = document.createDocumentFragment();

// Element properties and attributes
newDiv.id = 'newElement';
newDiv.className = 'my-class';
newDiv.classList.add('another-class');
newDiv.classList.remove('my-class');
newDiv.classList.toggle('active');
newDiv.classList.contains('another-class'); // true

// Setting attributes
newDiv.setAttribute('data-id', '123');
newDiv.setAttribute('aria-label', 'New element');
console.log(newDiv.getAttribute('data-id')); // '123'
console.log(newDiv.hasAttribute('data-id')); // true
newDiv.removeAttribute('data-id');

// Dataset API for data attributes
newDiv.dataset.userId = '456';
newDiv.dataset.createdAt = new Date().toISOString();
console.log(newDiv.dataset.userId); // '456'

// Content manipulation
newDiv.textContent = 'Plain text content';
newDiv.innerHTML = '<strong>HTML content</strong>';
newDiv.innerText = 'Text that respects styling';

// Safer HTML insertion
newDiv.insertAdjacentHTML('beforebegin', '<p>Before element</p>');
newDiv.insertAdjacentHTML('afterbegin', '<span>First child</span>');
newDiv.insertAdjacentHTML('beforeend', '<span>Last child</span>');
newDiv.insertAdjacentHTML('afterend', '<p>After element</p>');

// Adding to DOM
document.body.appendChild(newDiv);
document.body.insertBefore(newDiv, document.body.firstChild);
document.body.replaceChild(newDiv, oldElement);
document.body.removeChild(newDiv);

// Modern insertion methods
element.append(newDiv); // Adds to end
element.prepend(newDiv); // Adds to beginning
element.before(newDiv); // Inserts before element
element.after(newDiv); // Inserts after element
element.replaceWith(newDiv); // Replaces element
element.remove(); // Removes element

// Cloning elements
const clonedElement = element.cloneNode(true); // Deep clone
const shallowClone = element.cloneNode(false); // Shallow clone
```

### Styling Elements

```javascript
const element = document.querySelector('#myElement');

// Inline styles
element.style.color = 'red';
element.style.backgroundColor = 'blue';
element.style.fontSize = '16px';
element.style.cssText = 'color: red; background-color: blue; font-size: 16px;';

// CSS properties with dashes become camelCase
element.style.borderRadius = '5px';
element.style.marginTop = '10px';

// Getting computed styles
const computedStyles = window.getComputedStyle(element);
console.log(computedStyles.color);
console.log(computedStyles.getPropertyValue('background-color'));

// CSS classes
element.className = 'class1 class2';
element.classList.add('new-class');
element.classList.remove('old-class');
element.classList.toggle('active');
element.classList.replace('old-class', 'new-class');

// CSS custom properties (CSS variables)
document.documentElement.style.setProperty('--main-color', '#ff0000');
const mainColor = getComputedStyle(document.documentElement)
    .getPropertyValue('--main-color');

// Dynamic stylesheet manipulation
const styleSheet = document.createElement('style');
styleSheet.textContent = `
    .dynamic-class {
        color: red;
        font-weight: bold;
    }
`;
document.head.appendChild(styleSheet);

// CSSOM manipulation
const sheets = document.styleSheets;
const firstSheet = sheets[0];
firstSheet.insertRule('.new-rule { color: blue; }', 0);
firstSheet.deleteRule(0);
```

### Form Handling

```javascript
const form = document.querySelector('#myForm');
const input = document.querySelector('#myInput');
const select = document.querySelector('#mySelect');
const checkbox = document.querySelector('#myCheckbox');

// Getting form values
console.log(input.value);
console.log(select.value);
console.log(select.selectedIndex);
console.log(checkbox.checked);

// Setting form values
input.value = 'New value';
select.selectedIndex = 1;
checkbox.checked = true;

// Form validation
input.setCustomValidity('This field is required');
input.reportValidity();
console.log(input.validity.valid);
console.log(input.validationMessage);

// FormData API
const formData = new FormData(form);
formData.append('extraField', 'extraValue');

// Iterate over form data
for (const [key, value] of formData.entries()) {
    console.log(key, value);
}

// Form submission
form.addEventListener('submit', function(event) {
    event.preventDefault(); // Prevent default form submission
    
    const data = new FormData(this);
    const values = Object.fromEntries(data.entries());
    
    console.log('Form values:', values);
    
    // Custom submission logic
    fetch('/api/submit', {
        method: 'POST',
        body: data
    });
});

// Input events
input.addEventListener('input', function(event) {
    console.log('Input value changed:', event.target.value);
});

input.addEventListener('change', function(event) {
    console.log('Input value committed:', event.target.value);
});

// Focus events
input.addEventListener('focus', function() {
    this.classList.add('focused');
});

input.addEventListener('blur', function() {
    this.classList.remove('focused');
    // Validate input
    if (!this.value) {
        this.setCustomValidity('This field is required');
    } else {
        this.setCustomValidity('');
    }
});
```

## Event Handling

### Event Fundamentals

```javascript
// Event listener methods
element.addEventListener('click', handleClick);
element.removeEventListener('click', handleClick);

// Event handler properties (not recommended for multiple handlers)
element.onclick = handleClick;

// Inline event handlers (not recommended)
// <button onclick="handleClick()">Click me</button>

function handleClick(event) {
    console.log('Event type:', event.type);
    console.log('Target element:', event.target);
    console.log('Current target:', event.currentTarget);
    console.log('Event phase:', event.eventPhase);
    
    // Prevent default behavior
    event.preventDefault();
    
    // Stop event propagation
    event.stopPropagation();
    event.stopImmediatePropagation(); // Stops other listeners on same element
}

// Event options
element.addEventListener('click', handleClick, {
    once: true,      // Remove after first execution
    passive: true,   // Never calls preventDefault()
    capture: true    // Execute during capture phase
});

// Event delegation
document.body.addEventListener('click', function(event) {
    if (event.target.matches('.button')) {
        console.log('Button clicked:', event.target.textContent);
    }
});

// Custom events
const customEvent = new CustomEvent('myCustomEvent', {
    detail: { message: 'Hello from custom event' },
    bubbles: true,
    cancelable: true
});

element.dispatchEvent(customEvent);

element.addEventListener('myCustomEvent', function(event) {
    console.log('Custom event received:', event.detail.message);
});

// Event constructor variations
const clickEvent = new MouseEvent('click', {
    bubbles: true,
    cancelable: true,
    clientX: 100,
    clientY: 200
});

const keyEvent = new KeyboardEvent('keydown', {
    key: 'Enter',
    code: 'Enter',
    keyCode: 13
});
```

### Mouse Events

```javascript
element.addEventListener('mouseenter', function(event) {
    console.log('Mouse entered');
});

element.addEventListener('mouseleave', function(event) {
    console.log('Mouse left');
});

element.addEventListener('mouseover', function(event) {
    console.log('Mouse over (bubbles)');
});

element.addEventListener('mouseout', function(event) {
    console.log('Mouse out (bubbles)');
});

element.addEventListener('mousemove', function(event) {
    console.log(`Mouse position: ${event.clientX}, ${event.clientY}`);
    console.log(`Page position: ${event.pageX}, ${event.pageY}`);
    console.log(`Screen position: ${event.screenX}, ${event.screenY}`);
});

element.addEventListener('mousedown', function(event) {
    console.log('Mouse button pressed:', event.button);
    // 0: left, 1: middle, 2: right
});

element.addEventListener('mouseup', function(event) {
    console.log('Mouse button released');
});

element.addEventListener('contextmenu', function(event) {
    event.preventDefault(); // Prevent context menu
    console.log('Right-click detected');
});

// Drag and drop events
element.draggable = true;

element.addEventListener('dragstart', function(event) {
    event.dataTransfer.setData('text/plain', 'Hello World');
    event.dataTransfer.effectAllowed = 'copy';
});

dropZone.addEventListener('dragover', function(event) {
    event.preventDefault(); // Allow drop
});

dropZone.addEventListener('drop', function(event) {
    event.preventDefault();
    const data = event.dataTransfer.getData('text/plain');
    console.log('Dropped data:', data);
});
```

### Keyboard Events

```javascript
document.addEventListener('keydown', function(event) {
    console.log('Key pressed:', event.key);
    console.log('Key code:', event.code);
    console.log('Char code:', event.charCode);
    console.log('Key code (deprecated):', event.keyCode);
    
    // Modifier keys
    if (event.ctrlKey) console.log('Ctrl key held');
    if (event.shiftKey) console.log('Shift key held');
    if (event.altKey) console.log('Alt key held');
    if (event.metaKey) console.log('Meta key held');
    
    // Common key combinations
    if (event.ctrlKey && event.key === 's') {
        event.preventDefault();
        console.log('Ctrl+S pressed');
    }
    
    if (event.key === 'Escape') {
        console.log('Escape pressed');
    }
    
    if (event.key === 'Enter') {
        console.log('Enter pressed');
    }
});

document.addEventListener('keyup', function(event) {
    console.log('Key released:', event.key);
});

// Input-specific keyboard events
input.addEventListener('keypress', function(event) {
    // Only fires for printable characters
    console.log('Character typed:', event.key);
});

// Keyboard navigation helper
function handleKeyboardNavigation(event) {
    const focusableElements = document.querySelectorAll(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    
    const currentIndex = Array.from(focusableElements).indexOf(event.target);
    
    switch (event.key) {
        case 'ArrowDown':
        case 'ArrowRight':
            event.preventDefault();
            const nextIndex = (currentIndex + 1) % focusableElements.length;
            focusableElements[nextIndex].focus();
            break;
        case 'ArrowUp':
        case 'ArrowLeft':
            event.preventDefault();
            const prevIndex = (currentIndex - 1 + focusableElements.length) % focusableElements.length;
            focusableElements[prevIndex].focus();
            break;
    }
}
```

### Touch Events

```javascript
element.addEventListener('touchstart', function(event) {
    console.log('Touch started');
    console.log('Number of touches:', event.touches.length);
    
    for (let i = 0; i < event.touches.length; i++) {
        const touch = event.touches[i];
        console.log(`Touch ${i}:`, touch.clientX, touch.clientY);
    }
});

element.addEventListener('touchmove', function(event) {
    event.preventDefault(); // Prevent scrolling
    console.log('Touch moved');
});

element.addEventListener('touchend', function(event) {
    console.log('Touch ended');
    console.log('Changed touches:', event.changedTouches.length);
});

// Gesture detection
let startX, startY, startTime;

element.addEventListener('touchstart', function(event) {
    if (event.touches.length === 1) {
        const touch = event.touches[0];
        startX = touch.clientX;
        startY = touch.clientY;
        startTime = Date.now();
    }
});

element.addEventListener('touchend', function(event) {
    if (event.changedTouches.length === 1) {
        const touch = event.changedTouches[0];
        const endX = touch.clientX;
        const endY = touch.clientY;
        const endTime = Date.now();
        
        const deltaX = endX - startX;
        const deltaY = endY - startY;
        const deltaTime = endTime - startTime;
        
        // Swipe detection
        if (Math.abs(deltaX) > 50 && deltaTime < 500) {
            if (deltaX > 0) {
                console.log('Swipe right');
            } else {
                console.log('Swipe left');
            }
        }
        
        // Tap detection
        if (Math.abs(deltaX) < 10 && Math.abs(deltaY) < 10 && deltaTime < 300) {
            console.log('Tap detected');
        }
    }
});

// Pointer events (unified mouse/touch/pen)
element.addEventListener('pointerdown', function(event) {
    console.log('Pointer down:', event.pointerType); // 'mouse', 'touch', 'pen'
});

element.addEventListener('pointermove', function(event) {
    console.log('Pointer move:', event.clientX, event.clientY);
});

element.addEventListener('pointerup', function(event) {
    console.log('Pointer up');
});
```

## Browser APIs

### Window Object

```javascript
// Window properties
console.log(window.innerWidth, window.innerHeight); // Viewport size
console.log(window.outerWidth, window.outerHeight); // Browser window size
console.log(window.screenX, window.screenY); // Window position
console.log(window.devicePixelRatio); // Pixel density

// Window methods
window.alert('Alert message');
window.confirm('Confirm this action?');
const userInput = window.prompt('Enter your name:', 'Default');

// Window navigation
window.open('https://example.com', '_blank');
window.close(); // Only works for windows opened by script
window.print();

// History API
history.pushState({page: 1}, 'Page 1', '/page1');
history.replaceState({page: 2}, 'Page 2', '/page2');
history.back();
history.forward();
history.go(-2); // Go back 2 pages

window.addEventListener('popstate', function(event) {
    console.log('History changed:', event.state);
});

// Location object
console.log(location.href); // Full URL
console.log(location.protocol); // http: or https:
console.log(location.host); // hostname:port
console.log(location.hostname); // hostname
console.log(location.port); // port
console.log(location.pathname); // path
console.log(location.search); // query string
console.log(location.hash); // fragment

location.assign('https://example.com');
location.replace('https://example.com'); // No history entry
location.reload(); // Refresh page

// URL API
const url = new URL('https://example.com/path?param=value#section');
console.log(url.searchParams.get('param')); // 'value'
url.searchParams.set('newParam', 'newValue');
url.searchParams.delete('param');

// URLSearchParams
const params = new URLSearchParams(location.search);
params.forEach((value, key) => {
    console.log(key, value);
});
```

### Performance API

```javascript
// Performance timing
const perfData = performance.getEntriesByType('navigation')[0];
console.log('Page load time:', perfData.loadEventEnd - perfData.navigationStart);
console.log('DOM ready time:', perfData.domContentLoadedEventEnd - perfData.navigationStart);

// Performance marks and measures
performance.mark('start-task');
// ... perform task
performance.mark('end-task');
performance.measure('task-duration', 'start-task', 'end-task');

const measures = performance.getEntriesByType('measure');
console.log('Task duration:', measures[0].duration);

// Resource timing
const resources = performance.getEntriesByType('resource');
resources.forEach(resource => {
    console.log(`${resource.name}: ${resource.duration}ms`);
});

// Observer for performance entries
const observer = new PerformanceObserver(function(list) {
    list.getEntries().forEach(entry => {
        console.log('Performance entry:', entry.name, entry.duration);
    });
});

observer.observe({entryTypes: ['measure', 'navigation', 'resource']});

// Memory usage (Chrome only)
if ('memory' in performance) {
    console.log('Memory usage:', performance.memory);
}
```

### Notification API

```javascript
// Check for permission
if ('Notification' in window) {
    if (Notification.permission === 'granted') {
        showNotification();
    } else if (Notification.permission !== 'denied') {
        Notification.requestPermission().then(function(permission) {
            if (permission === 'granted') {
                showNotification();
            }
        });
    }
}

function showNotification() {
    const notification = new Notification('Hello!', {
        body: 'This is a notification',
        icon: '/icon.png',
        badge: '/badge.png',
        tag: 'unique-id', // Prevents duplicate notifications
        requireInteraction: false,
        silent: false,
        vibrate: [200, 100, 200],
        data: { customData: 'value' }
    });
    
    notification.addEventListener('click', function() {
        window.focus();
        notification.close();
    });
    
    notification.addEventListener('error', function() {
        console.log('Notification error');
    });
    
    // Auto-close after 5 seconds
    setTimeout(() => notification.close(), 5000);
}

// Service Worker notifications (for background notifications)
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js').then(function(registration) {
        registration.showNotification('Background Notification', {
            body: 'This works even when the page is closed',
            icon: '/icon.png'
        });
    });
}
```

### Clipboard API

```javascript
// Modern Clipboard API
async function copyToClipboard(text) {
    try {
        await navigator.clipboard.writeText(text);
        console.log('Text copied to clipboard');
    } catch (err) {
        console.error('Failed to copy text:', err);
        // Fallback to older method
        fallbackCopyTextToClipboard(text);
    }
}

async function readFromClipboard() {
    try {
        const text = await navigator.clipboard.readText();
        console.log('Clipboard content:', text);
        return text;
    } catch (err) {
        console.error('Failed to read clipboard:', err);
    }
}

// Fallback method for older browsers
function fallbackCopyTextToClipboard(text) {
    const textArea = document.createElement('textarea');
    textArea.value = text;
    textArea.style.position = 'fixed';
    textArea.style.left = '-999999px';
    textArea.style.top = '-999999px';
    document.body.appendChild(textArea);
    textArea.focus();
    textArea.select();
    
    try {
        const successful = document.execCommand('copy');
        console.log('Fallback copy successful:', successful);
    } catch (err) {
        console.error('Fallback copy failed:', err);
    }
    
    document.body.removeChild(textArea);
}

// Copy rich content
async function copyRichContent() {
    const data = new ClipboardItem({
        'text/plain': new Blob(['Plain text'], {type: 'text/plain'}),
        'text/html': new Blob(['<b>Rich HTML</b>'], {type: 'text/html'})
    });
    
    try {
        await navigator.clipboard.write([data]);
        console.log('Rich content copied');
    } catch (err) {
        console.error('Failed to copy rich content:', err);
    }
}
```

## Web Storage

### Local Storage and Session Storage

```javascript
// Local Storage (persists until explicitly cleared)
localStorage.setItem('username', 'john_doe');
localStorage.setItem('settings', JSON.stringify({theme: 'dark', lang: 'en'}));

const username = localStorage.getItem('username');
const settings = JSON.parse(localStorage.getItem('settings') || '{}');

localStorage.removeItem('username');
localStorage.clear(); // Remove all items

// Session Storage (cleared when tab closes)
sessionStorage.setItem('tempData', 'temporary value');
const tempData = sessionStorage.getItem('tempData');

// Storage events (fired when storage changes in other tabs)
window.addEventListener('storage', function(event) {
    console.log('Storage changed:');
    console.log('Key:', event.key);
    console.log('Old value:', event.oldValue);
    console.log('New value:', event.newValue);
    console.log('URL:', event.url);
    console.log('Storage area:', event.storageArea);
});

// Storage utility class
class StorageManager {
    constructor(storage = localStorage) {
        this.storage = storage;
    }
    
    set(key, value, expiry = null) {
        const item = {
            value: value,
            expiry: expiry ? Date.now() + expiry : null
        };
        this.storage.setItem(key, JSON.stringify(item));
    }
    
    get(key) {
        const itemStr = this.storage.getItem(key);
        if (!itemStr) return null;
        
        try {
            const item = JSON.parse(itemStr);
            
            if (item.expiry && Date.now() > item.expiry) {
                this.storage.removeItem(key);
                return null;
            }
            
            return item.value;
        } catch (err) {
            return itemStr; // Fallback for non-JSON values
        }
    }
    
    remove(key) {
        this.storage.removeItem(key);
    }
    
    clear() {
        this.storage.clear();
    }
    
    keys() {
        return Object.keys(this.storage);
    }
    
    size() {
        return this.storage.length;
    }
}

const storage = new StorageManager();
storage.set('user', {name: 'John'}, 3600000); // Expires in 1 hour
```

### IndexedDB

```javascript
// IndexedDB for complex client-side storage
class IndexedDBManager {
    constructor(dbName, version = 1) {
        this.dbName = dbName;
        this.version = version;
        this.db = null;
    }
    
    async init() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(this.dbName, this.version);
            
            request.onerror = () => reject(request.error);
            request.onsuccess = () => {
                this.db = request.result;
                resolve(this.db);
            };
            
            request.onupgradeneeded = (event) => {
                const db = event.target.result;
                
                // Create object stores
                if (!db.objectStoreNames.contains('users')) {
                    const userStore = db.createObjectStore('users', {keyPath: 'id', autoIncrement: true});
                    userStore.createIndex('email', 'email', {unique: true});
                    userStore.createIndex('name', 'name', {unique: false});
                }
            };
        });
    }
    
    async add(storeName, data) {
        const transaction = this.db.transaction([storeName], 'readwrite');
        const store = transaction.objectStore(storeName);
        
        return new Promise((resolve, reject) => {
            const request = store.add(data);
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
    }
    
    async get(storeName, key) {
        const transaction = this.db.transaction([storeName], 'readonly');
        const store = transaction.objectStore(storeName);
        
        return new Promise((resolve, reject) => {
            const request = store.get(key);
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
    }
    
    async getAll(storeName) {
        const transaction = this.db.transaction([storeName], 'readonly');
        const store = transaction.objectStore(storeName);
        
        return new Promise((resolve, reject) => {
            const request = store.getAll();
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
    }
    
    async update(storeName, data) {
        const transaction = this.db.transaction([storeName], 'readwrite');
        const store = transaction.objectStore(storeName);
        
        return new Promise((resolve, reject) => {
            const request = store.put(data);
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
    }
    
    async delete(storeName, key) {
        const transaction = this.db.transaction([storeName], 'readwrite');
        const store = transaction.objectStore(storeName);
        
        return new Promise((resolve, reject) => {
            const request = store.delete(key);
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
    }
    
    async query(storeName, indexName, value) {
        const transaction = this.db.transaction([storeName], 'readonly');
        const store = transaction.objectStore(storeName);
        const index = store.index(indexName);
        
        return new Promise((resolve, reject) => {
            const request = index.get(value);
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
    }
}

// Usage
async function useIndexedDB() {
    const dbManager = new IndexedDBManager('MyApp');
    await dbManager.init();
    
    // Add user
    const userId = await dbManager.add('users', {
        name: 'John Doe',
        email: 'john@example.com',
        age: 30
    });
    
    // Get user
    const user = await dbManager.get('users', userId);
    console.log('User:', user);
    
    // Query by index
    const userByEmail = await dbManager.query('users', 'email', 'john@example.com');
    console.log('User by email:', userByEmail);
    
    // Get all users
    const allUsers = await dbManager.getAll('users');
    console.log('All users:', allUsers);
}
```

## Fetch API and AJAX

### Fetch API Basics

```javascript
// Basic GET request
async function fetchData() {
    try {
        const response = await fetch('/api/data');
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const data = await response.json();
        console.log(data);
        return data;
    } catch (error) {
        console.error('Fetch error:', error);
        throw error;
    }
}

// POST request with JSON data
async function postData(userData) {
    try {
        const response = await fetch('/api/users', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': 'Bearer ' + token
            },
            body: JSON.stringify(userData)
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        return await response.json();
    } catch (error) {
        console.error('Post error:', error);
        throw error;
    }
}

// Form data upload
async function uploadFile(file) {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('description', 'File upload');
    
    try {
        const response = await fetch('/api/upload', {
            method: 'POST',
            body: formData // Don't set Content-Type header, let browser set it
        });
        
        return await response.json();
    } catch (error) {
        console.error('Upload error:', error);
        throw error;
    }
}

// Request with timeout
async function fetchWithTimeout(url, options = {}, timeout = 5000) {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeout);
    
    try {
        const response = await fetch(url, {
            ...options,
            signal: controller.signal
        });
        
        clearTimeout(timeoutId);
        return response;
    } catch (error) {
        clearTimeout(timeoutId);
        if (error.name === 'AbortError') {
            throw new Error('Request timed out');
        }
        throw error;
    }
}

// Retry logic
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            const response = await fetch(url, options);
            if (response.ok) {
                return response;
            }
            
            // Don't retry client errors (4xx)
            if (response.status >= 400 && response.status < 500) {
                throw new Error(`Client error: ${response.status}`);
            }
            
            // Retry server errors (5xx)
            if (attempt === maxRetries) {
                throw new Error(`Server error after ${maxRetries} attempts: ${response.status}`);
            }
        } catch (error) {
            if (attempt === maxRetries) {
                throw error;
            }
            
            // Exponential backoff
            const delay = Math.pow(2, attempt - 1) * 1000;
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
}
```

### Advanced Fetch Patterns

```javascript
// Request interceptor pattern
class APIClient {
    constructor(baseURL) {
        this.baseURL = baseURL;
        this.defaultHeaders = {};
    }
    
    setAuthToken(token) {
        this.defaultHeaders['Authorization'] = `Bearer ${token}`;
    }
    
    async request(endpoint, options = {}) {
        const url = `${this.baseURL}${endpoint}`;
        
        const config = {
            headers: {
                'Content-Type': 'application/json',
                ...this.defaultHeaders,
                ...options.headers
            },
            ...options
        };
        
        // Request interceptor
        console.log('Making request:', url, config);
        
        try {
            const response = await fetch(url, config);
            
            // Response interceptor
            if (!response.ok) {
                const errorData = await response.json().catch(() => ({}));
                throw new APIError(response.status, errorData.message || response.statusText);
            }
            
            return response;
        } catch (error) {
            // Error interceptor
            if (error instanceof APIError && error.status === 401) {
                // Handle authentication error
                this.handleAuthError();
            }
            throw error;
        }
    }
    
    async get(endpoint) {
        const response = await this.request(endpoint);
        return response.json();
    }
    
    async post(endpoint, data) {
        const response = await this.request(endpoint, {
            method: 'POST',
            body: JSON.stringify(data)
        });
        return response.json();
    }
    
    async put(endpoint, data) {
        const response = await this.request(endpoint, {
            method: 'PUT',
            body: JSON.stringify(data)
        });
        return response.json();
    }
    
    async delete(endpoint) {
        const response = await this.request(endpoint, {
            method: 'DELETE'
        });
        return response.ok;
    }
    
    handleAuthError() {
        // Redirect to login or refresh token
        window.location.href = '/login';
    }
}

class APIError extends Error {
    constructor(status, message) {
        super(message);
        this.name = 'APIError';
        this.status = status;
    }
}

// Usage
const api = new APIClient('/api');
api.setAuthToken('your-jwt-token');

try {
    const users = await api.get('/users');
    const newUser = await api.post('/users', {name: 'John', email: 'john@example.com'});
} catch (error) {
    if (error instanceof APIError) {
        console.error('API Error:', error.status, error.message);
    } else {
        console.error('Network Error:', error);
    }
}

// Streaming responses
async function streamResponse() {
    const response = await fetch('/api/large-data');
    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    
    while (true) {
        const { done, value } = await reader.read();
        
        if (done) break;
        
        const chunk = decoder.decode(value, { stream: true });
        console.log('Received chunk:', chunk);
        
        // Process chunk
        processChunk(chunk);
    }
}

// Server-Sent Events
function setupEventStream() {
    const eventSource = new EventSource('/api/events');
    
    eventSource.onopen = function() {
        console.log('Event stream opened');
    };
    
    eventSource.onmessage = function(event) {
        console.log('Received event:', event.data);
        const data = JSON.parse(event.data);
        updateUI(data);
    };
    
    eventSource.addEventListener('custom-event', function(event) {
        console.log('Custom event:', event.data);
    });
    
    eventSource.onerror = function(event) {
        console.error('Event stream error:', event);
        
        if (eventSource.readyState === EventSource.CLOSED) {
            console.log('Event stream closed');
        }
    };
    
    // Close connection
    // eventSource.close();
}
```

## Web Workers

### Basic Web Workers

```javascript
// Main thread
if (window.Worker) {
    const worker = new Worker('/js/worker.js');
    
    // Send data to worker
    worker.postMessage({
        command: 'calculate',
        data: [1, 2, 3, 4, 5]
    });
    
    // Receive data from worker
    worker.onmessage = function(event) {
        console.log('Result from worker:', event.data);
    };
    
    worker.onerror = function(error) {
        console.error('Worker error:', error);
    };
    
    // Terminate worker
    // worker.terminate();
}

// worker.js
self.onmessage = function(event) {
    const { command, data } = event.data;
    
    switch (command) {
        case 'calculate':
            const result = heavyCalculation(data);
            self.postMessage(result);
            break;
        
        case 'process':
            const processed = processData(data);
            self.postMessage(processed);
            break;
        
        default:
            self.postMessage({ error: 'Unknown command' });
    }
};

function heavyCalculation(numbers) {
    // Simulate heavy computation
    let sum = 0;
    for (let i = 0; i < 1000000; i++) {
        sum += numbers.reduce((acc, num) => acc + num * Math.random(), 0);
    }
    return sum;
}

function processData(data) {
    // Process data without blocking main thread
    return data.map(item => ({
        ...item,
        processed: true,
        timestamp: Date.now()
    }));
}

// Import scripts in worker
self.importScripts('/js/utils.js', '/js/calculations.js');

// Error handling in worker
self.onerror = function(error) {
    console.error('Worker internal error:', error);
};
```

### Shared Workers

```javascript
// Main thread 1
const sharedWorker = new SharedWorker('/js/shared-worker.js');
const port = sharedWorker.port;

port.start();

port.postMessage({
    type: 'subscribe',
    channel: 'notifications'
});

port.onmessage = function(event) {
    console.log('Shared worker message:', event.data);
};

// Main thread 2 (different tab)
const sharedWorker2 = new SharedWorker('/js/shared-worker.js');
const port2 = sharedWorker2.port;

port2.start();

port2.postMessage({
    type: 'broadcast',
    channel: 'notifications',
    message: 'Hello from tab 2'
});

// shared-worker.js
const connections = [];
const channels = new Map();

self.onconnect = function(event) {
    const port = event.ports[0];
    connections.push(port);
    
    port.onmessage = function(event) {
        const { type, channel, message } = event.data;
        
        switch (type) {
            case 'subscribe':
                if (!channels.has(channel)) {
                    channels.set(channel, []);
                }
                channels.get(channel).push(port);
                break;
                
            case 'broadcast':
                const subscribers = channels.get(channel) || [];
                subscribers.forEach(subscriberPort => {
                    if (subscriberPort !== port) {
                        subscriberPort.postMessage({
                            type: 'broadcast',
                            channel,
                            message
                        });
                    }
                });
                break;
        }
    };
    
    port.start();
};
```

### Service Workers

```javascript
// Register service worker
if ('serviceWorker' in navigator) {
    window.addEventListener('load', function() {
        navigator.serviceWorker.register('/sw.js')
            .then(function(registration) {
                console.log('SW registered: ', registration);
                
                registration.addEventListener('updatefound', function() {
                    const newWorker = registration.installing;
                    newWorker.addEventListener('statechange', function() {
                        if (newWorker.state === 'installed') {
                            if (navigator.serviceWorker.controller) {
                                // New update available
                                showUpdateNotification();
                            }
                        }
                    });
                });
            })
            .catch(function(registrationError) {
                console.log('SW registration failed: ', registrationError);
            });
    });
    
    // Listen for messages from service worker
    navigator.serviceWorker.addEventListener('message', function(event) {
        console.log('Message from SW:', event.data);
    });
}

// sw.js (Service Worker)
const CACHE_NAME = 'my-app-v1';
const urlsToCache = [
    '/',
    '/css/style.css',
    '/js/app.js',
    '/images/icon.png'
];

// Install event
self.addEventListener('install', function(event) {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then(function(cache) {
                return cache.addAll(urlsToCache);
            })
    );
});

// Fetch event
self.addEventListener('fetch', function(event) {
    event.respondWith(
        caches.match(event.request)
            .then(function(response) {
                // Return cached version or fetch from network
                return response || fetch(event.request);
            })
    );
});

// Activate event
self.addEventListener('activate', function(event) {
    event.waitUntil(
        caches.keys().then(function(cacheNames) {
            return Promise.all(
                cacheNames.map(function(cacheName) {
                    if (cacheName !== CACHE_NAME) {
                        return caches.delete(cacheName);
                    }
                })
            );
        })
    );
});

// Background sync
self.addEventListener('sync', function(event) {
    if (event.tag === 'background-sync') {
        event.waitUntil(doBackgroundSync());
    }
});

// Push notifications
self.addEventListener('push', function(event) {
    const options = {
        body: event.data.text(),
        icon: '/images/icon.png',
        badge: '/images/badge.png'
    };
    
    event.waitUntil(
        self.registration.showNotification('Push Notification', options)
    );
});
```

## WebSockets

### Basic WebSocket Usage

```javascript
class WebSocketClient {
    constructor(url) {
        this.url = url;
        this.ws = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
        this.reconnectDelay = 1000;
        this.listeners = new Map();
    }
    
    connect() {
        try {
            this.ws = new WebSocket(this.url);
            
            this.ws.onopen = (event) => {
                console.log('WebSocket connected');
                this.reconnectAttempts = 0;
                this.emit('open', event);
            };
            
            this.ws.onmessage = (event) => {
                try {
                    const data = JSON.parse(event.data);
                    this.emit('message', data);
                    
                    // Handle specific message types
                    if (data.type) {
                        this.emit(data.type, data);
                    }
                } catch (error) {
                    console.error('Failed to parse WebSocket message:', error);
                    this.emit('error', error);
                }
            };
            
            this.ws.onclose = (event) => {
                console.log('WebSocket closed:', event.code, event.reason);
                this.emit('close', event);
                
                if (!event.wasClean && this.reconnectAttempts < this.maxReconnectAttempts) {
                    this.reconnect();
                }
            };
            
            this.ws.onerror = (error) => {
                console.error('WebSocket error:', error);
                this.emit('error', error);
            };
            
        } catch (error) {
            console.error('Failed to create WebSocket connection:', error);
            this.emit('error', error);
        }
    }
    
    reconnect() {
        this.reconnectAttempts++;
        const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
        
        console.log(`Attempting to reconnect in ${delay}ms (attempt ${this.reconnectAttempts})`);
        
        setTimeout(() => {
            this.connect();
        }, delay);
    }
    
    send(data) {
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(data));
        } else {
            console.warn('WebSocket is not open. Message not sent:', data);
            this.emit('error', new Error('WebSocket is not open'));
        }
    }
    
    close() {
        if (this.ws) {
            this.ws.close(1000, 'Client closing connection');
        }
    }
    
    // Event emitter methods
    on(event, callback) {
        if (!this.listeners.has(event)) {
            this.listeners.set(event, []);
        }
        this.listeners.get(event).push(callback);
    }
    
    off(event, callback) {
        if (this.listeners.has(event)) {
            const callbacks = this.listeners.get(event);
            const index = callbacks.indexOf(callback);
            if (index > -1) {
                callbacks.splice(index, 1);
            }
        }
    }
    
    emit(event, data) {
        if (this.listeners.has(event)) {
            this.listeners.get(event).forEach(callback => {
                try {
                    callback(data);
                } catch (error) {
                    console.error('Error in WebSocket event callback:', error);
                }
            });
        }
    }
    
    // Connection state
    get readyState() {
        return this.ws ? this.ws.readyState : WebSocket.CLOSED;
    }
    
    get isConnected() {
        return this.readyState === WebSocket.OPEN;
    }
}

// Usage
const wsClient = new WebSocketClient('ws://localhost:8080');

wsClient.on('open', () => {
    console.log('Connected to WebSocket server');
    wsClient.send({ type: 'auth', token: 'user-token' });
});

wsClient.on('message', (data) => {
    console.log('Received message:', data);
});

wsClient.on('chat-message', (data) => {
    displayChatMessage(data.user, data.message);
});

wsClient.on('user-joined', (data) => {
    showNotification(`${data.username} joined the chat`);
});

wsClient.on('error', (error) => {
    console.error('WebSocket error:', error);
});

wsClient.on('close', (event) => {
    console.log('Connection closed:', event.code, event.reason);
});

wsClient.connect();

// Send messages
document.getElementById('sendButton').addEventListener('click', () => {
    const input = document.getElementById('messageInput');
    wsClient.send({
        type: 'chat-message',
        message: input.value,
        timestamp: Date.now()
    });
    input.value = '';
});
```

### Advanced WebSocket Patterns

```javascript
// WebSocket with heartbeat
class HeartbeatWebSocket extends WebSocketClient {
    constructor(url, heartbeatInterval = 30000) {
        super(url);
        this.heartbeatInterval = heartbeatInterval;
        this.heartbeatTimer = null;
        this.lastHeartbeat = null;
    }
    
    connect() {
        super.connect();
        
        this.on('open', () => {
            this.startHeartbeat();
        });
        
        this.on('close', () => {
            this.stopHeartbeat();
        });
        
        this.on('pong', () => {
            this.lastHeartbeat = Date.now();
        });
    }
    
    startHeartbeat() {
        this.heartbeatTimer = setInterval(() => {
            if (this.isConnected) {
                this.send({ type: 'ping', timestamp: Date.now() });
            }
        }, this.heartbeatInterval);
    }
    
    stopHeartbeat() {
        if (this.heartbeatTimer) {
            clearInterval(this.heartbeatTimer);
            this.heartbeatTimer = null;
        }
    }
}

// WebSocket message queue
class QueuedWebSocket extends HeartbeatWebSocket {
    constructor(url) {
        super(url);
        this.messageQueue = [];
    }
    
    send(data) {
        if (this.isConnected) {
            super.send(data);
        } else {
            // Queue message for later
            this.messageQueue.push(data);
        }
    }
    
    connect() {
        super.connect();
        
        this.on('open', () => {
            // Send queued messages
            while (this.messageQueue.length > 0) {
                const message = this.messageQueue.shift();
                super.send(message);
            }
        });
    }
}

// WebSocket room manager
class WebSocketRoomManager {
    constructor(wsClient) {
        this.wsClient = wsClient;
        this.currentRooms = new Set();
        
        this.wsClient.on('room-joined', (data) => {
            this.currentRooms.add(data.room);
            console.log(`Joined room: ${data.room}`);
        });
        
        this.wsClient.on('room-left', (data) => {
            this.currentRooms.delete(data.room);
            console.log(`Left room: ${data.room}`);
        });
    }
    
    joinRoom(roomId) {
        if (!this.currentRooms.has(roomId)) {
            this.wsClient.send({
                type: 'join-room',
                room: roomId
            });
        }
    }
    
    leaveRoom(roomId) {
        if (this.currentRooms.has(roomId)) {
            this.wsClient.send({
                type: 'leave-room',
                room: roomId
            });
        }
    }
    
    sendToRoom(roomId, message) {
        this.wsClient.send({
            type: 'room-message',
            room: roomId,
            message: message
        });
    }
    
    getRooms() {
        return Array.from(this.currentRooms);
    }
}

// Usage
const wsClient = new QueuedWebSocket('ws://localhost:8080');
const roomManager = new WebSocketRoomManager(wsClient);

wsClient.connect();

// Join rooms
roomManager.joinRoom('general');
roomManager.joinRoom('development');

// Send message to specific room
roomManager.sendToRoom('general', 'Hello everyone!');
```

## Geolocation API

```javascript
// Check if geolocation is supported
if ('geolocation' in navigator) {
    // Get current position
    navigator.geolocation.getCurrentPosition(
        function(position) {
            const { latitude, longitude, accuracy } = position.coords;
            console.log(`Location: ${latitude}, ${longitude}`);
            console.log(`Accuracy: ${accuracy} meters`);
            
            // Additional properties
            console.log(`Altitude: ${position.coords.altitude}`);
            console.log(`Heading: ${position.coords.heading}`);
            console.log(`Speed: ${position.coords.speed}`);
            console.log(`Timestamp: ${new Date(position.timestamp)}`);
        },
        function(error) {
            switch (error.code) {
                case error.PERMISSION_DENIED:
                    console.error('User denied geolocation request');
                    break;
                case error.POSITION_UNAVAILABLE:
                    console.error('Location information unavailable');
                    break;
                case error.TIMEOUT:
                    console.error('Location request timed out');
                    break;
                default:
                    console.error('Unknown geolocation error');
                    break;
            }
        },
        {
            enableHighAccuracy: true,
            timeout: 10000,
            maximumAge: 60000 // Cache position for 1 minute
        }
    );
    
    // Watch position changes
    const watchId = navigator.geolocation.watchPosition(
        function(position) {
            updateLocationOnMap(position.coords);
        },
        function(error) {
            console.error('Geolocation watch error:', error);
        },
        {
            enableHighAccuracy: true,
            timeout: 5000,
            maximumAge: 30000
        }
    );
    
    // Stop watching
    // navigator.geolocation.clearWatch(watchId);
    
} else {
    console.error('Geolocation is not supported by this browser');
}

// Geolocation utility class
class LocationService {
    constructor() {
        this.watchId = null;
        this.lastKnownPosition = null;
    }
    
    async getCurrentPosition(options = {}) {
        return new Promise((resolve, reject) => {
            if (!navigator.geolocation) {
                reject(new Error('Geolocation not supported'));
                return;
            }
            
            navigator.geolocation.getCurrentPosition(
                (position) => {
                    this.lastKnownPosition = position;
                    resolve(position);
                },
                reject,
                {
                    enableHighAccuracy: true,
                    timeout: 10000,
                    maximumAge: 300000, // 5 minutes
                    ...options
                }
            );
        });
    }
    
    startWatching(callback, options = {}) {
        if (!navigator.geolocation) {
            throw new Error('Geolocation not supported');
        }
        
        this.watchId = navigator.geolocation.watchPosition(
            (position) => {
                this.lastKnownPosition = position;
                callback(position);
            },
            (error) => {
                console.error('Geolocation watch error:', error);
            },
            {
                enableHighAccuracy: true,
                timeout: 5000,
                maximumAge: 60000,
                ...options
            }
        );
        
        return this.watchId;
    }
    
    stopWatching() {
        if (this.watchId !== null) {
            navigator.geolocation.clearWatch(this.watchId);
            this.watchId = null;
        }
    }
    
    calculateDistance(lat1, lon1, lat2, lon2) {
        const R = 6371; // Earth's radius in kilometers
        const dLat = this.degreesToRadians(lat2 - lat1);
        const dLon = this.degreesToRadians(lon2 - lon1);
        
        const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                  Math.cos(this.degreesToRadians(lat1)) * Math.cos(this.degreesToRadians(lat2)) *
                  Math.sin(dLon / 2) * Math.sin(dLon / 2);
        
        const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        return R * c; // Distance in kilometers
    }
    
    degreesToRadians(degrees) {
        return degrees * (Math.PI / 180);
    }
    
    getLastKnownPosition() {
        return this.lastKnownPosition;
    }
}

// Usage
const locationService = new LocationService();

// Get current location
try {
    const position = await locationService.getCurrentPosition();
    console.log('Current location:', position.coords);
} catch (error) {
    console.error('Failed to get location:', error);
}

// Start tracking location
locationService.startWatching((position) => {
    console.log('Location updated:', position.coords);
    
    // Calculate distance from starting point
    if (startingPosition) {
        const distance = locationService.calculateDistance(
            startingPosition.latitude,
            startingPosition.longitude,
            position.coords.latitude,
            position.coords.longitude
        );
        console.log(`Distance traveled: ${distance.toFixed(2)} km`);
    }
});
```

## File API

```javascript
// File input handling
const fileInput = document.getElementById('fileInput');
const dropZone = document.getElementById('dropZone');

fileInput.addEventListener('change', handleFileSelect);
dropZone.addEventListener('drop', handleFileDrop);
dropZone.addEventListener('dragover', handleDragOver);

function handleFileSelect(event) {
    const files = event.target.files;
    processFiles(files);
}

function handleFileDrop(event) {
    event.preventDefault();
    const files = event.dataTransfer.files;
    processFiles(files);
}

function handleDragOver(event) {
    event.preventDefault();
    event.dataTransfer.dropEffect = 'copy';
}

function processFiles(files) {
    Array.from(files).forEach(file => {
        console.log('File info:');
        console.log('Name:', file.name);
        console.log('Size:', file.size, 'bytes');
        console.log('Type:', file.type);
        console.log('Last modified:', new Date(file.lastModified));
        
        // Process different file types
        if (file.type.startsWith('image/')) {
            handleImageFile(file);
        } else if (file.type === 'text/plain') {
            handleTextFile(file);
        } else if (file.type === 'application/json') {
            handleJSONFile(file);
        } else {
            handleGenericFile(file);
        }
    });
}

// Image file handling
function handleImageFile(file) {
    const reader = new FileReader();
    
    reader.onload = function(event) {
        const img = new Image();
        img.onload = function() {
            console.log(`Image dimensions: ${img.width}x${img.height}`);
            
            // Display image
            const imgElement = document.createElement('img');
            imgElement.src = event.target.result;
            imgElement.style.maxWidth = '300px';
            document.body.appendChild(imgElement);
            
            // Create thumbnail
            createThumbnail(img, 150, 150);
        };
        img.src = event.target.result;
    };
    
    reader.onerror = function() {
        console.error('Error reading image file');
    };
    
    reader.readAsDataURL(file);
}

// Text file handling
function handleTextFile(file) {
    const reader = new FileReader();
    
    reader.onload = function(event) {
        const text = event.target.result;
        console.log('Text content:', text);
        
        // Display text
        const pre = document.createElement('pre');
        pre.textContent = text;
        document.body.appendChild(pre);
    };
    
    reader.readAsText(file);
}

// JSON file handling
function handleJSONFile(file) {
    const reader = new FileReader();
    
    reader.onload = function(event) {
        try {
            const json = JSON.parse(event.target.result);
            console.log('JSON data:', json);
            
            // Display formatted JSON
            const pre = document.createElement('pre');
            pre.textContent = JSON.stringify(json, null, 2);
            document.body.appendChild(pre);
        } catch (error) {
            console.error('Invalid JSON file:', error);
        }
    };
    
    reader.readAsText(file);
}

// Generic file handling
function handleGenericFile(file) {
    const reader = new FileReader();
    
    reader.onload = function(event) {
        const arrayBuffer = event.target.result;
        console.log('File loaded as ArrayBuffer:', arrayBuffer.byteLength, 'bytes');
        
        // Convert to different formats as needed
        const uint8Array = new Uint8Array(arrayBuffer);
        console.log('First 10 bytes:', Array.from(uint8Array.slice(0, 10)));
    };
    
    reader.readAsArrayBuffer(file);
}

// Create thumbnail
function createThumbnail(img, maxWidth, maxHeight) {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    
    // Calculate thumbnail dimensions
    let { width, height } = img;
    const aspectRatio = width / height;
    
    if (width > height) {
        width = maxWidth;
        height = width / aspectRatio;
    } else {
        height = maxHeight;
        width = height * aspectRatio;
    }
    
    canvas.width = width;
    canvas.height = height;
    
    // Draw thumbnail
    ctx.drawImage(img, 0, 0, width, height);
    
    // Convert to blob
    canvas.toBlob(function(blob) {
        console.log('Thumbnail created:', blob.size, 'bytes');
        
        // Display thumbnail
        const thumbnailUrl = URL.createObjectURL(blob);
        const thumbnailImg = document.createElement('img');
        thumbnailImg.src = thumbnailUrl;
        thumbnailImg.style.border = '1px solid #ccc';
        document.body.appendChild(thumbnailImg);
        
        // Clean up object URL
        thumbnailImg.onload = () => URL.revokeObjectURL(thumbnailUrl);
    }, 'image/jpeg', 0.8);
}

// File upload with progress
async function uploadFileWithProgress(file) {
    const formData = new FormData();
    formData.append('file', file);
    
    try {
        const response = await fetch('/api/upload', {
            method: 'POST',
            body: formData
        });
        
        if (!response.ok) {
            throw new Error(`Upload failed: ${response.statusText}`);
        }
        
        const result = await response.json();
        console.log('Upload successful:', result);
        return result;
    } catch (error) {
        console.error('Upload error:', error);
        throw error;
    }
}

// Chunked file upload for large files
class ChunkedUploader {
    constructor(file, chunkSize = 1024 * 1024) { // 1MB chunks
        this.file = file;
        this.chunkSize = chunkSize;
        this.totalChunks = Math.ceil(file.size / chunkSize);
        this.uploadedChunks = 0;
    }
    
    async upload(onProgress) {
        const uploadId = this.generateUploadId();
        
        for (let chunkIndex = 0; chunkIndex < this.totalChunks; chunkIndex++) {
            const start = chunkIndex * this.chunkSize;
            const end = Math.min(start + this.chunkSize, this.file.size);
            const chunk = this.file.slice(start, end);
            
            await this.uploadChunk(chunk, chunkIndex, uploadId);
            this.uploadedChunks++;
            
            if (onProgress) {
                onProgress({
                    loaded: this.uploadedChunks,
                    total: this.totalChunks,
                    percentage: (this.uploadedChunks / this.totalChunks) * 100
                });
            }
        }
        
        return this.finalizeUpload(uploadId);
    }
    
    async uploadChunk(chunk, chunkIndex, uploadId) {
        const formData = new FormData();
        formData.append('chunk', chunk);
        formData.append('chunkIndex', chunkIndex);
        formData.append('uploadId', uploadId);
        formData.append('totalChunks', this.totalChunks);
        formData.append('fileName', this.file.name);
        
        const response = await fetch('/api/upload-chunk', {
            method: 'POST',
            body: formData
        });
        
        if (!response.ok) {
            throw new Error(`Chunk upload failed: ${response.statusText}`);
        }
        
        return response.json();
    }
    
    async finalizeUpload(uploadId) {
        const response = await fetch('/api/finalize-upload', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                uploadId,
                fileName: this.file.name,
                fileSize: this.file.size,
                totalChunks: this.totalChunks
            })
        });
        
        if (!response.ok) {
            throw new Error(`Upload finalization failed: ${response.statusText}`);
        }
        
        return response.json();
    }
    
    generateUploadId() {
        return Date.now() + '-' + Math.random().toString(36).substr(2, 9);
    }
}

// Usage
const uploader = new ChunkedUploader(file);
uploader.upload((progress) => {
    console.log(`Upload progress: ${progress.percentage.toFixed(1)}%`);
    updateProgressBar(progress.percentage);
}).then(result => {
    console.log('Upload completed:', result);
}).catch(error => {
    console.error('Upload failed:', error);
});
```

## Canvas API

```javascript
// Basic canvas setup
const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');

// Set canvas size
canvas.width = 800;
canvas.height = 600;

// Basic drawing operations
ctx.fillStyle = 'red';
ctx.fillRect(10, 10, 100, 100);

ctx.strokeStyle = 'blue';
ctx.lineWidth = 2;
ctx.strokeRect(150, 10, 100, 100);

// Drawing paths
ctx.beginPath();
ctx.moveTo(300, 50);
ctx.lineTo(350, 10);
ctx.lineTo(400, 50);
ctx.lineTo(375, 100);
ctx.lineTo(325, 100);
ctx.closePath();
ctx.fillStyle = 'green';
ctx.fill();
ctx.stroke();

// Drawing circles
ctx.beginPath();
ctx.arc(500, 60, 40, 0, 2 * Math.PI);
ctx.fillStyle = 'purple';
ctx.fill();

// Text drawing
ctx.font = '24px Arial';
ctx.fillStyle = 'black';
ctx.fillText('Hello Canvas!', 10, 150);

ctx.strokeStyle = 'red';
ctx.strokeText('Outlined Text', 10, 180);

// Image drawing
const img = new Image();
img.onload = function() {
    ctx.drawImage(img, 10, 200, 100, 100);
    
    // Draw part of image
    ctx.drawImage(img, 0, 0, 50, 50, 150, 200, 100, 100);
};
img.src = '/path/to/image.jpg';

// Canvas animation class
class CanvasAnimation {
    constructor(canvas) {
        this.canvas = canvas;
        this.ctx = canvas.getContext('2d');
        this.animationId = null;
        this.lastTime = 0;
        this.objects = [];
    }
    
    addObject(obj) {
        this.objects.push(obj);
    }
    
    removeObject(obj) {
        const index = this.objects.indexOf(obj);
        if (index > -1) {
            this.objects.splice(index, 1);
        }
    }
    
    start() {
        this.animate(0);
    }
    
    stop() {
        if (this.animationId) {
            cancelAnimationFrame(this.animationId);
            this.animationId = null;
        }
    }
    
    animate(currentTime) {
        const deltaTime = currentTime - this.lastTime;
        this.lastTime = currentTime;
        
        // Clear canvas
        this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
        
        // Update and draw objects
        this.objects.forEach(obj => {
            obj.update(deltaTime);
            obj.draw(this.ctx);
        });
        
        this.animationId = requestAnimationFrame((time) => this.animate(time));
    }
}

// Animated ball class
class Ball {
    constructor(x, y, radius, color, vx = 0, vy = 0) {
        this.x = x;
        this.y = y;
        this.radius = radius;
        this.color = color;
        this.vx = vx;
        this.vy = vy;
        this.gravity = 0.5;
        this.bounce = 0.8;
    }
    
    update(deltaTime) {
        // Apply gravity
        this.vy += this.gravity;
        
        // Update position
        this.x += this.vx;
        this.y += this.vy;
        
        // Bounce off walls
        if (this.x + this.radius > canvas.width || this.x - this.radius < 0) {
            this.vx *= -this.bounce;
            this.x = Math.max(this.radius, Math.min(canvas.width - this.radius, this.x));
        }
        
        if (this.y + this.radius > canvas.height || this.y - this.radius < 0) {
            this.vy *= -this.bounce;
            this.y = Math.max(this.radius, Math.min(canvas.height - this.radius, this.y));
        }
    }
    
    draw(ctx) {
        ctx.beginPath();
        ctx.arc(this.x, this.y, this.radius, 0, 2 * Math.PI);
        ctx.fillStyle = this.color;
        ctx.fill();
        ctx.strokeStyle = 'black';
        ctx.stroke();
    }
}

// Usage
const animation = new CanvasAnimation(canvas);

// Add some balls
for (let i = 0; i < 10; i++) {
    const ball = new Ball(
        Math.random() * canvas.width,
        Math.random() * canvas.height,
        10 + Math.random() * 20,
        `hsl(${Math.random() * 360}, 70%, 50%)`,
        (Math.random() - 0.5) * 10,
        (Math.random() - 0.5) * 10
    );
    animation.addObject(ball);
}

animation.start();

// Mouse interaction
canvas.addEventListener('click', function(event) {
    const rect = canvas.getBoundingClientRect();
    const x = event.clientX - rect.left;
    const y = event.clientY - rect.top;
    
    const ball = new Ball(x, y, 15, 'red', 0, -10);
    animation.addObject(ball);
});

// Canvas utilities
class CanvasUtils {
    static getImageData(canvas) {
        const ctx = canvas.getContext('2d');
        return ctx.getImageData(0, 0, canvas.width, canvas.height);
    }
    
    static putImageData(canvas, imageData) {
        const ctx = canvas.getContext('2d');
        ctx.putImageData(imageData, 0, 0);
    }
    
    static applyFilter(canvas, filterFunction) {
        const ctx = canvas.getContext('2d');
        const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        const data = imageData.data;
        
        for (let i = 0; i < data.length; i += 4) {
            const pixel = {
                r: data[i],
                g: data[i + 1],
                b: data[i + 2],
                a: data[i + 3]
            };
            
            const filtered = filterFunction(pixel);
            
            data[i] = filtered.r;
            data[i + 1] = filtered.g;
            data[i + 2] = filtered.b;
            data[i + 3] = filtered.a;
        }
        
        ctx.putImageData(imageData, 0, 0);
    }
    
    static grayscale(pixel) {
        const gray = 0.299 * pixel.r + 0.587 * pixel.g + 0.114 * pixel.b;
        return { r: gray, g: gray, b: gray, a: pixel.a };
    }
    
    static sepia(pixel) {
        return {
            r: Math.min(255, 0.393 * pixel.r + 0.769 * pixel.g + 0.189 * pixel.b),
            g: Math.min(255, 0.349 * pixel.r + 0.686 * pixel.g + 0.168 * pixel.b),
            b: Math.min(255, 0.272 * pixel.r + 0.534 * pixel.g + 0.131 * pixel.b),
            a: pixel.a
        };
    }
    
    static saveAsImage(canvas, filename = 'canvas.png') {
        const link = document.createElement('a');
        link.download = filename;
        link.href = canvas.toDataURL();
        link.click();
    }
}

// Apply filters
document.getElementById('grayscaleBtn').addEventListener('click', () => {
    CanvasUtils.applyFilter(canvas, CanvasUtils.grayscale);
});

document.getElementById('sepiaBtn').addEventListener('click', () => {
    CanvasUtils.applyFilter(canvas, CanvasUtils.sepia);
});

document.getElementById('saveBtn').addEventListener('click', () => {
    CanvasUtils.saveAsImage(canvas, 'my-canvas.png');
});
```

## Intersection Observer

```javascript
// Basic Intersection Observer
const observer = new IntersectionObserver(function(entries, observer) {
    entries.forEach(entry => {
        if (entry.isIntersecting) {
            console.log('Element is visible:', entry.target);
            
            // Add animation class
            entry.target.classList.add('animate-in');
            
            // Stop observing this element
            observer.unobserve(entry.target);
        }
    });
}, {
    root: null, // Use viewport as root
    rootMargin: '0px',
    threshold: 0.1 // Trigger when 10% visible
});

// Observe elements
document.querySelectorAll('.animate-on-scroll').forEach(el => {
    observer.observe(el);
});

// Lazy loading images
class LazyImageLoader {
    constructor() {
        this.imageObserver = new IntersectionObserver(
            this.handleImageIntersection.bind(this),
            {
                rootMargin: '50px 0px', // Load images 50px before they come into view
                threshold: 0
            }
        );
        
        this.init();
    }
    
    init() {
        const lazyImages = document.querySelectorAll('img[data-src]');
        lazyImages.forEach(img => {
            this.imageObserver.observe(img);
        });
    }
    
    handleImageIntersection(entries) {
        entries.forEach(entry => {
            if (entry.isIntersecting) {
                const img = entry.target;
                const src = img.dataset.src;
                
                if (src) {
                    img.src = src;
                    img.removeAttribute('data-src');
                    img.classList.add('loaded');
                    
                    img.onload = () => {
                        img.classList.add('fade-in');
                    };
                    
                    this.imageObserver.unobserve(img);
                }
            }
        });
    }
}

// Initialize lazy loading
const lazyLoader = new LazyImageLoader();

// Infinite scroll
class InfiniteScroll {
    constructor(containerSelector, loadMoreCallback) {
        this.container = document.querySelector(containerSelector);
        this.loadMoreCallback = loadMoreCallback;
        this.loading = false;
        this.page = 1;
        
        this.sentinel = document.createElement('div');
        this.sentinel.className = 'scroll-sentinel';
        this.container.appendChild(this.sentinel);
        
        this.observer = new IntersectionObserver(
            this.handleSentinelIntersection.bind(this),
            {
                root: null,
                rootMargin: '100px',
                threshold: 0
            }
        );
        
        this.observer.observe(this.sentinel);
    }
    
    async handleSentinelIntersection(entries) {
        const entry = entries[0];
        
        if (entry.isIntersecting && !this.loading) {
            this.loading = true;
            
            try {
                const hasMore = await this.loadMoreCallback(this.page);
                
                if (hasMore) {
                    this.page++;
                } else {
                    // No more content, stop observing
                    this.observer.unobserve(this.sentinel);
                    this.sentinel.remove();
                }
            } catch (error) {
                console.error('Failed to load more content:', error);
            } finally {
                this.loading = false;
            }
        }
    }
    
    destroy() {
        this.observer.disconnect();
        if (this.sentinel.parentNode) {
            this.sentinel.remove();
        }
    }
}

// Usage
const infiniteScroll = new InfiniteScroll('#content-container', async (page) => {
    try {
        const response = await fetch(`/api/posts?page=${page}`);
        const data = await response.json();
        
        // Add new content to container
        data.posts.forEach(post => {
            const postElement = createPostElement(post);
            document.getElementById('content-container').appendChild(postElement);
        });
        
        return data.hasMore;
    } catch (error) {
        console.error('Failed to load posts:', error);
        return false;
    }
});

// Visibility tracking for analytics
class VisibilityTracker {
    constructor() {
        this.observer = new IntersectionObserver(
            this.handleVisibilityChange.bind(this),
            {
                threshold: [0, 0.25, 0.5, 0.75, 1.0]
            }
        );
        
        this.visibilityData = new Map();
    }
    
    track(element, trackingId) {
        element.dataset.trackingId = trackingId;
        this.observer.observe(element);
        
        this.visibilityData.set(trackingId, {
            element,
            firstSeen: null,
            lastSeen: null,
            maxVisibility: 0,
            totalTime: 0,
            currentVisibility: 0
        });
    }
    
    handleVisibilityChange(entries) {
        entries.forEach(entry => {
            const trackingId = entry.target.dataset.trackingId;
            const data = this.visibilityData.get(trackingId);
            
            if (!data) return;
            
            const now = Date.now();
            data.currentVisibility = entry.intersectionRatio;
            data.maxVisibility = Math.max(data.maxVisibility, entry.intersectionRatio);
            
            if (entry.isIntersecting) {
                if (!data.firstSeen) {
                    data.firstSeen = now;
                }
                data.lastSeen = now;
            }
            
            // Send analytics event for significant visibility changes
            if (entry.intersectionRatio > 0.5 && data.maxVisibility <= 0.5) {
                this.sendAnalyticsEvent('element_viewed', {
                    trackingId,
                    visibility: entry.intersectionRatio
                });
            }
        });
    }
    
    sendAnalyticsEvent(eventType, data) {
        // Send to analytics service
        console.log('Analytics event:', eventType, data);
        
        // Example: send to Google Analytics
        // gtag('event', eventType, data);
    }
    
    getVisibilityData(trackingId) {
        return this.visibilityData.get(trackingId);
    }
    
    stopTracking(trackingId) {
        const data = this.visibilityData.get(trackingId);
        if (data) {
            this.observer.unobserve(data.element);
            this.visibilityData.delete(trackingId);
        }
    }
}

// Usage
const visibilityTracker = new VisibilityTracker();

document.querySelectorAll('.track-visibility').forEach((element, index) => {
    visibilityTracker.track(element, `element-${index}`);
});
```

## Mutation Observer

```javascript
// Basic Mutation Observer
const targetNode = document.getElementById('container');

const observer = new MutationObserver(function(mutationsList, observer) {
    mutationsList.forEach(mutation => {
        switch (mutation.type) {
            case 'childList':
                console.log('Child nodes changed');
                mutation.addedNodes.forEach(node => {
                    if (node.nodeType === Node.ELEMENT_NODE) {
                        console.log('Added element:', node);
                    }
                });
                mutation.removedNodes.forEach(node => {
                    if (node.nodeType === Node.ELEMENT_NODE) {
                        console.log('Removed element:', node);
                    }
                });
                break;
                
            case 'attributes':
                console.log(`Attribute ${mutation.attributeName} changed on`, mutation.target);
                break;
                
            case 'characterData':
                console.log('Text content changed:', mutation.target.textContent);
                break;
        }
    });
});

// Start observing
observer.observe(targetNode, {
    childList: true,
    attributes: true,
    attributeOldValue: true,
    characterData: true,
    characterDataOldValue: true,
    subtree: true // Observe all descendants
});

// Stop observing
// observer.disconnect();

// DOM change tracker
class DOMChangeTracker {
    constructor() {
        this.observers = new Map();
        this.changeHandlers = new Map();
    }
    
    track(element, options = {}) {
        const elementId = this.generateId();
        
        const observer = new MutationObserver((mutations) => {
            this.handleMutations(elementId, mutations);
        });
        
        const observerOptions = {
            childList: true,
            attributes: true,
            characterData: true,
            subtree: true,
            attributeOldValue: true,
            characterDataOldValue: true,
            ...options
        };
        
        observer.observe(element, observerOptions);
        
        this.observers.set(elementId, {
            observer,
            element,
            options: observerOptions
        });
        
        return elementId;
    }
    
    onChanged(elementId, handler) {
        if (!this.changeHandlers.has(elementId)) {
            this.changeHandlers.set(elementId, []);
        }
        this.changeHandlers.get(elementId).push(handler);
    }
    
    handleMutations(elementId, mutations) {
        const handlers = this.changeHandlers.get(elementId) || [];
        
        handlers.forEach(handler => {
            try {
                handler(mutations);
            } catch (error) {
                console.error('Error in mutation handler:', error);
            }
        });
    }
    
    stopTracking(elementId) {
        const trackedElement = this.observers.get(elementId);
        if (trackedElement) {
            trackedElement.observer.disconnect();
            this.observers.delete(elementId);
            this.changeHandlers.delete(elementId);
        }
    }
    
    generateId() {
        return 'tracker-' + Date.now() + '-' + Math.random().toString(36).substr(2, 9);
    }
    
    getTrackedElements() {
        return Array.from(this.observers.keys());
    }
}

// Usage
const changeTracker = new DOMChangeTracker();

const trackerId = changeTracker.track(document.body);

changeTracker.onChanged(trackerId, (mutations) => {
    mutations.forEach(mutation => {
        console.log('DOM changed:', mutation.type, mutation.target);
        
        if (mutation.type === 'childList') {
            // Handle added/removed elements
            mutation.addedNodes.forEach(node => {
                if (node.nodeType === Node.ELEMENT_NODE) {
                    initializeNewElement(node);
                }
            });
        }
    });
});

// Auto-initialize components
class ComponentAutoInitializer {
    constructor() {
        this.componentInitializers = new Map();
        this.observer = new MutationObserver(this.handleMutations.bind(this));
        
        this.observer.observe(document.body, {
            childList: true,
            subtree: true
        });
    }
    
    register(selector, initializer) {
        this.componentInitializers.set(selector, initializer);
        
        // Initialize existing elements
        document.querySelectorAll(selector).forEach(element => {
            if (!element.dataset.initialized) {
                this.initializeComponent(element, initializer);
            }
        });
    }
    
    handleMutations(mutations) {
        mutations.forEach(mutation => {
            if (mutation.type === 'childList') {
                mutation.addedNodes.forEach(node => {
                    if (node.nodeType === Node.ELEMENT_NODE) {
                        this.initializeElementAndDescendants(node);
                    }
                });
            }
        });
    }
    
    initializeElementAndDescendants(element) {
        // Check the element itself
        this.componentInitializers.forEach((initializer, selector) => {
            if (element.matches(selector) && !element.dataset.initialized) {
                this.initializeComponent(element, initializer);
            }
        });
        
        // Check descendants
        this.componentInitializers.forEach((initializer, selector) => {
            element.querySelectorAll(selector).forEach(descendant => {
                if (!descendant.dataset.initialized) {
                    this.initializeComponent(descendant, initializer);
                }
            });
        });
    }
    
    initializeComponent(element, initializer) {
        try {
            initializer(element);
            element.dataset.initialized = 'true';
        } catch (error) {
            console.error('Failed to initialize component:', error);
        }
    }
    
    destroy() {
        this.observer.disconnect();
        this.componentInitializers.clear();
    }
}

// Usage
const autoInitializer = new ComponentAutoInitializer();

// Register component initializers
autoInitializer.register('.tooltip', (element) => {
    // Initialize tooltip
    element.addEventListener('mouseenter', showTooltip);
    element.addEventListener('mouseleave', hideTooltip);
});

autoInitializer.register('.dropdown', (element) => {
    // Initialize dropdown
    const trigger = element.querySelector('.dropdown-trigger');
    const menu = element.querySelector('.dropdown-menu');
    
    trigger.addEventListener('click', () => {
        menu.classList.toggle('show');
    });
});

autoInitializer.register('.modal-trigger', (element) => {
    // Initialize modal trigger
    element.addEventListener('click', (e) => {
        e.preventDefault();
        const modalId = element.dataset.modalTarget;
        const modal = document.getElementById(modalId);
        if (modal) {
            showModal(modal);
        }
    });
});

// Form validation observer
class FormValidationObserver {
    constructor(form) {
        this.form = form;
        this.observer = new MutationObserver(this.handleFormChanges.bind(this));
        
        this.observer.observe(form, {
            childList: true,
            attributes: true,
            attributeFilter: ['required', 'pattern', 'min', 'max', 'minlength', 'maxlength'],
            subtree: true
        });
        
        this.setupInitialValidation();
    }
    
    handleFormChanges(mutations) {
        let shouldRevalidate = false;
        
        mutations.forEach(mutation => {
            if (mutation.type === 'childList') {
                // New form fields added
                mutation.addedNodes.forEach(node => {
                    if (node.nodeType === Node.ELEMENT_NODE) {
                        const inputs = node.querySelectorAll('input, textarea, select');
                        inputs.forEach(input => this.setupFieldValidation(input));
                        shouldRevalidate = true;
                    }
                });
            } else if (mutation.type === 'attributes') {
                // Validation attributes changed
                this.setupFieldValidation(mutation.target);
                shouldRevalidate = true;
            }
        });
        
        if (shouldRevalidate) {
            this.validateForm();
        }
    }
    
    setupInitialValidation() {
        const inputs = this.form.querySelectorAll('input, textarea, select');
        inputs.forEach(input => this.setupFieldValidation(input));
    }
    
    setupFieldValidation(input) {
        // Remove existing listeners
        input.removeEventListener('blur', this.validateField);
        input.removeEventListener('input', this.validateField);
        
        // Add validation listeners
        input.addEventListener('blur', () => this.validateField(input));
        input.addEventListener('input', () => this.validateField(input));
    }
    
    validateField(input) {
        const isValid = input.checkValidity();
        const errorElement = input.parentNode.querySelector('.error-message');
        
        if (isValid) {
            input.classList.remove('invalid');
            if (errorElement) {
                errorElement.remove();
            }
        } else {
            input.classList.add('invalid');
            if (!errorElement) {
                const error = document.createElement('div');
                error.className = 'error-message';
                error.textContent = input.validationMessage;
                input.parentNode.appendChild(error);
            }
        }
    }
    
    validateForm() {
        const inputs = this.form.querySelectorAll('input, textarea, select');
        inputs.forEach(input => this.validateField(input));
    }
    
    destroy() {
        this.observer.disconnect();
    }
}

// Usage
const form = document.getElementById('dynamic-form');
const formValidator = new FormValidationObserver(form);

---

This completes the comprehensive JavaScript DOM and Browser APIs section. The content covers DOM manipulation, event handling, various browser APIs, storage solutions, modern web technologies, and advanced patterns for web development.
