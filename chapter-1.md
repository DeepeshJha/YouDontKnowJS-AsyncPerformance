# Chapter 1: Asynchrony: Now & Later — Detailed Notes

---

## 1. Introduction: The Nature of Asynchrony in JavaScript

- **Asynchrony** is about expressing and managing code that runs "now" and code that runs "later" with a gap in between.
- This is deeper than just time taken by a for-loop; it's about non-blocking behaviors, where the program is not actively executing in the "gap."
- **Examples of asynchrony**: user input, file/database access, network requests, animation intervals.
- Key point: You have to manage program state across these time gaps.
- *Metaphor:* "Mind the gap" (as in the London subway chasm).

---

## 2. The Fundamental Problem

- Relationship between "now" and "later" is at the heart of asynchronous programming.
- JS started with callbacks as the main async primitive, but as JS has grown, managing asynchrony with callbacks has become increasingly painful.
- There is a need for better, more reason-able approaches.

---

## 3. Programs in Chunks

- JS code is written in files, but functionally it's made of "chunks" (functions).
- Only one chunk runs "now," the rest are scheduled for "later."
- **The problem:** "later" does not always immediately follow "now."

### Example:
```js
var data = ajax("http://some.url.1");
console.log(data); // Oops! `data` generally won't have the Ajax results
```
- Standard Ajax is async, so the result isn’t available immediately.
- If Ajax were blocking, `data` would have the response, but that's not how things work.

**Callback solution:**
```js
ajax("http://some.url.1", function myCallbackFunction(data){
  console.log(data); // Yay, I gots me some `data`!
});
```

**Note:**  
While synchronous Ajax requests are technically possible, they should **never** be used in browsers because they lock the UI and prevent user interaction.

---

## 4. Now and Later Chunks — Explicit Example

```js
function now() { return 21; }
function later() { answer = answer * 2; console.log("Meaning of life:", answer); }
var answer = now();
setTimeout(later, 1000); // "later" chunk
```
- "Now" chunk: function definitions, variable assignment, `setTimeout` registration.
- "Later" chunk: the body of `later()` is executed after 1000ms.

**Explicit breakdown:**
- Now:
  - function now() { return 21; }
  - function later() { .. }
  - var answer = now();
  - setTimeout(later, 1000);
- Later:
  - answer = answer * 2;
  - console.log("Meaning of life:", answer);

**Key point:**  
Any time you wrap code into a function and specify it should be executed in response to an event (timer, click, Ajax, etc.), you are creating a "later" chunk and introducing asynchrony.

---

## 5. Async Console

- `console.*` methods are not part of official JavaScript; they are provided by the hosting environment (browser, Node.js, etc.).
- In some browsers, `console.log(..)` may defer output for performance reasons, especially for expensive I/O.
- **Example:**
  ```js
  var a = { index: 1 };
  console.log(a); // ??
  a.index++;
  ```
  - Normally, you'd expect `{index: 1}` in the console, but in rare cases, you might see `{index: 2}` if the output was deferred.

**Note:**  
- Use breakpoints in your debugger for reliable state snapshots.
- Or, serialize objects to strings (e.g., `console.log(JSON.stringify(a))`) for an immutable snapshot.

---

## 6. Event Loop

- Up until ES6, the core JS engine had **no direct built-in asynchrony**. All asynchrony is managed by the *hosting environment* (browser, Node.js, etc.) via the **event loop**.
- The event loop is a queue (FIFO) of events/callbacks to be processed one at a time.

**Pseudocode:**
```js
var eventLoop = [];
var event;
while (true) {
  if (eventLoop.length > 0) {
    event = eventLoop.shift();
    try { event(); }
    catch (err) { reportError(err); }
  }
}
```
- `setTimeout` only schedules a timer; when it expires, the environment puts the callback into the event queue.
- If 20 items are in the queue, your callback waits its turn; timers are not perfectly accurate in JS.

**Note:**  
With ES6, the event loop queue is now formally specified, especially for Promises.

---

## 7. Parallelism vs. Asynchrony

- **Asynchrony** is about "now" and "later" execution (non-blocking, not immediate).
- **Parallelism** is about things happening at the same instant (threads, processes).
- JS is **single-threaded**: code inside a function runs to completion without interruption from other JS code.

### Example:
```js
var a = 20;
function foo() { a = a + 1; }
function bar() { a = a * 2; }
ajax("http://some.url.1", foo);
ajax("http://some.url.2", bar);
```
- If `foo()` runs before `bar()`, result is `a = 42`. If `bar()` runs before `foo()`, result is `a = 41`.

**Note:**  
In a parallel system, shared memory between threads can lead to unpredictable results. In JS, because of single-threading and no shared memory across threads, this level of nondeterminism is avoided.

---

## 8. Run-to-Completion

- JS functions are atomic in execution: once a function starts running, it finishes before any other JS event handler or function can run (run-to-completion).
- This limits nondeterminism to the ordering of entire function executions, not individual statements.

### Example:
```js
var a = 1, b = 2;
function foo() { a++; b = b * a; a = b + 3; }
function bar() { b--; a = 8 + b; b = a * 2; }
ajax("http://some.url.1", foo);
ajax("http://some.url.2", bar);
```
- Only two possible outcomes, depending on which handler runs first.

---

## 9. Concurrency and Race Conditions

- **Concurrency**: Multiple chains of events ("processes") interleave over time. At any instant, only one event is executing, but from a high level, several processes seem to be running at once.

### Example:
- Process 1: Responds to `onscroll` events.
- Process 2: Handles Ajax responses.
- Even if both are ready to run at the same instant, only one can proceed at a time in the event loop.

#### Event Loop Queue Visualization

- All events, regardless of origin, are merged into the single event loop queue.
- "Process 1" and "Process 2" run concurrently (task-level parallelism), but their individual events are handled sequentially.

---

## 10. Noninteracting vs Interacting Concurrency

- **Noninteracting:** If two processes don't interact (e.g., update different objects), their relative event order is irrelevant.
  ```js
  var res = {};
  function foo(results) { res.foo = results; }
  function bar(results) { res.bar = results; }
  ajax("url1", foo);
  ajax("url2", bar);
  ```
- **Interacting:** If the order matters (e.g., updating the same array), you must coordinate to prevent bugs (race conditions).
  ```js
  var res = [];
  function response(data) { res.push(data); }
  ajax("url1", response);
  ajax("url2", response);
  // res[0] may not correspond to url1!
  ```

**Fix for race condition:**
```js
function response(data) {
  if (data.url == "url1") res[0] = data;
  else if (data.url == "url2") res[1] = data;
}
```

- **Gate:** Wait for all needed values before continuing.
  ```js
  var a, b;
  function foo(x) { a = x * 2; if (a && b) baz(); }
  function bar(y) { b = y * 2; if (a && b) baz(); }
  function baz() { console.log(a + b); }
  ```
- **Latch:** Only the first callback "wins."
  ```js
  var a;
  function foo(x) { if (!a) { a = x * 2; baz(); } }
  function bar(x) { if (!a) { a = x / 2; baz(); } }
  function baz() { console.log(a); }
  ```

---

## 11. Cooperative Concurrency

- For long-running tasks (e.g., processing a huge array), break them into smaller async chunks and "yield" between them so the event loop can handle other events.

### Example:
```js
function response(data) {
  var chunk = data.splice(0, 1000);
  res = res.concat(chunk.map(val => val * 2));
  if (data.length > 0) setTimeout(() => response(data), 0);
}
```
- This makes the site more responsive by avoiding long, blocking operations.

**Note:**  
Order of results may not be preserved without extra coordination.

---

## 12. The ES6 Job Queue

- ES6 introduced the "Job queue" (microtasks), used by Promises and other async features.
- Job queue sits at the end of every event loop tick; jobs are executed before the next event loop tick starts.
- **Metaphor:** The event loop queue is like an amusement park ride; the job queue is a shortcut to get back on the ride immediately.
- Jobs can add more jobs, so an infinite job loop could "starve" the event loop.

---

## 13. Statement Ordering & Compilation

- JS engines may optimize by reordering statements as long as the observable result is unchanged.
- Side effects (like `console.log` or getters) prevent unsafe reordering.

### Example:
```js
var a, b;
a = 10; b = 30; a = a + 1; b = b + 1; console.log(a + b); // 42
```
- The engine may safely reorder for performance, as long as the result is 42.

**But not:**
```js
var a, b;
a = 10; b = 30; console.log(a * b); // 300
a = a + 1; b = b + 1; console.log(a + b); // 42
```
- Here, the order matters for the observable results.

---

## 14. Review & Key Takeaways

- JS programs are split into now/later chunks, sharing scope and state.
- The event loop runs events one at a time ("tick").
- User interaction, I/O, and timers enqueue events.
- Only one event is processed at a time.
- Concurrency is about interleaving event chains; parallelism is simultaneous execution (not present in standard JS).
- Coordination (gates, latches, batching) is crucial for correct async/concurrent code.
- ES6 introduced the Job queue (Promises).
- Code order isn't always execution order due to optimizations.

---

## **Extra: Noteworthy Notes and Warnings**

- **Never use synchronous Ajax** in the browser—it blocks the UI.
- **console.log** output may not be immediate; use serialization or breakpoints to reliably inspect values.
- **setTimeout(..0)** does not guarantee exact event ordering; timer drift may occur.
- **Not all nondeterminism is bad**—sometimes it's intentional or irrelevant.
- **Be wary of assumptions** about the order of async responses; always coordinate when needed.
- **Compiler statement reordering** should never be observably incorrect; if you see it, it's a bug in the JS engine.

---
