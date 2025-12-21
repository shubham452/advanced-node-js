

### **Part 1: The Internals of Node**

#### **001 – 003: Foundations (V8 & Libuv)**
Node.js relies on two critical open-source dependencies to function:
1.  **V8:** Google’s JavaScript engine (70% C++). It allows JS to execute outside the browser.
2.  **libuv:** A C++ library that handles the operating system's underlying file system, networking, and concurrency.

**Why Node?**
Node acts as a bridge. It provides a consistent JavaScript API (wrappers) so developers don't have to write C++ to access these low-level features.

#### **004: The Basics of Threads**
*   **Process:** An instance of a computer program.
*   **Thread:** A "todo list" of instructions for the CPU to execute.
*   **Scheduling:** The OS decides which thread to process. It can pause a thread waiting for I/O (like reading a file) to let another thread run.

#### **005 – 007: The Event Loop (Pseudo-Code)**
The video uses "fake" code to explain the lifecycle of a Node process. The Event Loop is a control structure that decides what the single thread should do at any given time.

**The Pseudo-Code (`loop.js`):**
```javascript
// 1. Run the file contents
myFile.runContents();

// 2. Arrays that track pending work
const pendingTimers = [];
const pendingOSTasks = [];
const pendingOperations = [];

// 3. The Event Loop "Tick"
while(shouldContinue()) {
    // Step 1: Check pendingTimers (setTimeout, setInterval)
    
    // Step 2: Check pendingOSTasks (Networking) and pendingOperations (FS)
    
    // Step 3: PAUSE execution. 
    // Node sits here and waits for an event (timer expiry, file read finish) 
    // instead of spinning the CPU.
    
    // Step 4: Check pendingTimers for setImmediate
    
    // Step 5: Handle 'close' events (cleanup)
}
// Exit back to terminal
```
**Key Concept:** Step 3 is crucial. Node pauses and waits for tasks to complete, which is efficient.

#### **008 – 009: Is Node Single Threaded?**
The Event Loop itself is single-threaded, but the standard library utilizes **threads** outside of that loop for specific tasks. This is tested using the Crypto module.

**The Test Code (`threads.js`):**
```javascript
const crypto = require('crypto');

const start = Date.now();

// Intentionally expensive function (hashing)
crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
    console.log('1:', Date.now() - start);
});

// Run a second one immediately
crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
    console.log('2:', Date.now() - start);
});
```
**Result:** Both complete at roughly the same time (e.g., ~1 second). If Node were purely single-threaded, the second would finish at 2 seconds. This proves parallel execution.

#### **010 – 013: The Libuv Thread Pool**
The concurrency observed above is due to **libuv's Thread Pool**.
*   **Default Size:** 4 threads.
*   **Mechanism:** When you call `pbkdf2`, Node offloads the math to the C++ side, which uses a thread from the pool.
*   **Limitation:** If you run 5 calls, the first 4 run in parallel. The 5th waits for a thread to free up.

**Changing Thread Pool Size:**
You can customize this via an environment variable before running the app (or at the top of the file on Mac/Linux):
```javascript
process.env.UV_THREADPOOL_SIZE = 2; // Restrict to 2 threads
```
If set to 2, two calls run, then the next two, doubling the total time.

#### **014 – 016: OS Operations (Networking)**
Not all async code uses the thread pool. Networking functions delegate directly to the OS kernel.

**The Code (`async.js`):**
```javascript
const https = require('https');
const start = Date.now();

function doRequest() {
    https.request('https://www.google.com', res => {
        res.on('data', () => {});
        res.on('end', () => {
            console.log(Date.now() - start);
        });
    }).end();
}

// Call 6 times
doRequest(); doRequest(); doRequest(); 
doRequest(); doRequest(); doRequest();
```
**Result:** All 6 finish at nearly the same time. `https.request` is handled by the OS, not the 4-thread pool. It is non-blocking and infinite in scalability relative to the thread pool.

#### **017 – 019: The "Multitask" Interview Question**
This section combines `https`, `fs`, and `crypto` to show how the Event Loop, Thread Pool, and OS interact.

**The Code (`multitask.js`):**
```javascript
const https = require('https');
const crypto = require('crypto');
const fs = require('fs');

const start = Date.now();

function doRequest() {
    https.request('https://www.google.com', res => {
        res.on('data', () => {});
        res.on('end', () => {
            console.log('HTTP:', Date.now() - start);
        });
    }).end();
}

function doHash() {
    crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
        console.log('Hash:', Date.now() - start);
    });
}

doRequest(); // 1. HTTP
fs.readFile('multitask.js', 'utf8', () => { // 2. FS
    console.log('FS:', Date.now() - start);
});
doHash(); doHash(); doHash(); doHash(); // 3. Four Hashes
```
**The Result & Explanation:**
1.  **HTTP finishes first:** OS handles it, no thread pool blocking.
2.  **Hash finishes:** The thread pool (4 threads) fills up with the FS call and 3 Hash calls.
3.  **FS finishes last (Unexpected):** The FS call takes a thread, requests file stats from the hard drive, and *pauses*. Because it paused, the thread is released and picks up the 4th Hash call. The FS task must wait until a thread becomes free again to finish reading.

---

### **Part 2: Enhancing Node Performance**

#### **020 – 022: Blocking the Event Loop**
Since the main Event Loop is single-threaded, a calculation in the JS code blocks *everything*.

**The Blocking Code (`index.js`):**
```javascript
const express = require('express');
const app = express();

function doWork(duration) {
    const start = Date.now();
    // Spin-wait (blocks the loop)
    while(Date.now() - start < duration) {}
}

app.get('/', (req, res) => {
    doWork(5000); // Freezes server for 5 seconds
    res.send('Hi there');
});

app.listen(3000);
```
**Impact:** If Tab A loads `/`, Tab B (even if requesting a different fast route) cannot be served until Tab A finishes.

#### **023 – 025: Clustering in Theory & Action**
To solve blocking, we use **Clustering**. This creates multiple instances (processes) of the app.
*   **Cluster Manager:** Starts workers, monitors health.
*   **Worker:** Instances that actually handle requests.

**The Code (`index.js` with Cluster):**
```javascript
const cluster = require('cluster');

// Is this the first time executing index.js?
if (cluster.isMaster) {
    // Create a child instance
    cluster.fork(); 
    cluster.fork(); 
} else {
    // Child mode: Act like a normal server
    const express = require('express');
    const app = express();
    
    // ... server routes ...

    app.listen(3000);
}
```
**Result:** Requests can be handled in parallel because there are multiple Event Loops running.

#### **026 – 028: Benchmarking & Diminishing Returns**
Using **Apache Benchmark (`ab`)** to test performance.
*   **Command:** `ab -c 50 -n 500 localhost:3000/fast` (50 concurrent requests, 500 total).
*   **Diminishing Returns:** If you `fork()` 6 times on a dual-core machine, performance drops. The CPU spends too much time "context switching" between threads rather than doing work. You should generally match forks to your physical/logical core count.

#### **029 – 030: PM2 Configuration**
In production, use **PM2** instead of manual `cluster` code. It handles restarts and cluster management automatically.

**Key Commands:**
*   `npm install -g pm2`
*   `pm2 start index.js -i 0` (0 = auto-detect number of cores and launch that many instances).
*   `pm2 list`, `pm2 show index`, `pm2 monit` (dashboard).
*   `pm2 delete index` (stop).

#### **031 – 033: Webworker Threads**
Uses the `webworker-threads` library to spawn a thread specifically for JavaScript logic (not I/O).

**The Code (`index.js`):**
```javascript
const Worker = require('webworker-threads').Worker;

const worker = new Worker(function() {
    // Inside the Worker Thread
    this.onmessage = function() {
        let counter = 0;
        while (counter < 1e9) { counter++; } // Heavy work
        postMessage(counter); // Send back to main app
    };
});

// Inside Main App
worker.onmessage = function(message) {
    console.log(message.data);
};

worker.postMessage(); // Trigger the worker
```
**Conclusion:** This is useful for heavy JS math, but usually, Clustering (PM2) is the preferred method for general server performance.
