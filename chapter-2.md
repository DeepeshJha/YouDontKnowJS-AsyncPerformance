# Chapter 2: Callbacks — Detailed Notes

---

## 1. Callbacks: The Foundation of JS Asynchrony

- Callbacks are functions "called back" by the event loop when async events are ready.
- The most fundamental async pattern in JavaScript.
- Most JS programs, even complex ones, are built on callbacks.

---

## 2. Continuations

- Callbacks encapsulate the **continuation** of your program—the "rest" that runs after an async step.
- **Example:**
  ```js
  // A
  setTimeout(function() {
    // C
  }, 1000);
  // B
  ```
  - A and B run now; C runs later (after 1 second).

**Key observation:**  
Introducing a callback creates a subtle but significant divergence between how our brains expect code to run (sequentially) and how it actually does (nonlinear, evented).

---

## 3. The Sequential Brain

- Humans naturally **plan** tasks sequentially (A, then B, then C).
- Our brains actually operate much like an event loop: single-tasking, but rapidly context-switching.
- Synchronous code matches our mental planning; async code with callbacks does not.

**Example:**
```js
z = x;
x = y;
y = z;
```
- Each statement waits for the previous to finish.

But async code is more like:
- A, then (maybe B, maybe C, maybe D), maybe in unpredictable order.

---

## 4. Callback Hell: Nested/Chained Callbacks

- **Callback hell** (a.k.a. "pyramid of doom") refers to nested async code, not just indentation but confusion in flow.
- **Example:**
  ```js
  listen("click", function handler(evt) {
    setTimeout(function request() {
      ajax("http://some.url.1", function response(text) {
        if (text == "hello") handler();
        else if (text == "world") request();
      });
    }, 500);
  });
  ```
- Even if you "flatten" code by naming callbacks, you still have to jump around to follow the flow.

**Problem:**  
Linking steps together with callbacks is brittle, hard to generalize, and error-prone—especially when you need error handling, retries, or forking logic.

---

## 5. Inversion of Control and Trust Issues

- When you give your callback to a utility (especially third-party), you lose control—**inversion of control**.
- The utility can:
  - Call your callback too early.
  - Call it too late (or never).
  - Call it too many or too few times.
  - Fail to pass necessary parameters.
  - Swallow errors/exceptions.

**Example:**
```js
analytics.trackPurchase(purchaseData, function() {
  chargeCreditCard();
  displayThankyouPage();
});
```
- If the utility calls your callback five times, your customer might get charged five times!

**Mitigation:**
```js
var tracked = false;
analytics.trackPurchase(purchaseData, function() {
  if (!tracked) {
    tracked = true;
    chargeCreditCard();
    displayThankyouPage();
  }
});
```
- But now, what if they *never* call your callback? More ad hoc logic required.

**List of possible misbehaviors:**
- Call too early
- Call too late/never
- Call too few/many times
- Fail to pass parameters
- Swallow errors

---

## 6. Defensive Coding (Even With Your Own Code)

- Defensive programming for callbacks is like input validation for functions.
- Even your own utilities may need checks, as you can't always guarantee code will behave.
- Defensive patterns:
  - Type checking
  - Conversions
  - Latches, gates, timeouts, etc.

---

## 7. Callback Patterns and Their Limits

### Split Callbacks (Success/Error)
```js
function success(data) { ... }
function failure(err) { ... }
ajax("url", success, failure);
```
- More verbose, but still doesn't prevent too many/few calls or both/no calls.

### Error-First Callbacks (Node.js Style)
```js
function response(err, data) { ... }
ajax("url", response);
```
- If error, first argument is set; otherwise it's falsy and remaining arguments are success data.

### Timeoutify Utility
```js
function timeoutify(fn, delay) { ... }
ajax("url", timeoutify(foo, 500));
```
- Prevents "never called" by making sure the callback is invoked after a delay, if it hasn't been already.

### Asyncify Utility
```js
function asyncify(fn) { ... }
ajax("url", asyncify(result));
```
- Ensures callback is always invoked asynchronously (prevents Zalgo).

---

## 8. The Zalgo Problem

- Some APIs may call your callback synchronously or asynchronously depending on conditions (e.g., cache hit/miss).
- This unpredictability ("releasing Zalgo") leads to hard-to-find bugs.
- Always async your callbacks to ensure predictable async flow.

---

## 9. Review & Key Takeaways

- **Callbacks** are the foundation of JS asynchrony, but:
  1. They break our natural sequential reasoning, making code harder to write, debug, and maintain.
  2. They suffer from inversion of control, leading to trust issues, bugs, and repetitive defensive patterns.
- Patterns like split/error-first callbacks, timeoutify, and asyncify help, but are verbose and don't solve all issues.
- As JS matures, we need a more robust, reusable solution for expressing asynchrony in a way that matches how we think and plan code.
- **Promises (next chapter)** are the next step in solving these issues.

---
