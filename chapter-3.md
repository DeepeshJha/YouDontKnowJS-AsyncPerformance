# Chapter 3: Promises — Detailed Notes

---

## 1. Why Promises? (Problems With Callbacks)

- **Callback drawbacks**:
  - Lack of sequentiality: hard to express "do this, then that" in a readable way.
  - Lack of trust: inversion of control—your continuation is handed to another party, who may misuse or mishandle it.
- **Promises** address these by:
  - Letting you listen for completion rather than hand over control.
  - Returning an object (the promise) you can use to know when something finishes, and then decide what to do next.

---

## 2. What Is a Promise? (Conceptual Overview)

- **Analogy 1: Future Value (Cheeseburger Order)**
  - You pay, receive an order number (promise), and wait. The receipt is a placeholder for the future value (the burger).
  - When ready, you redeem it for the burger—or find out they’re out (failure).
  - Promises can be fulfilled (success) or rejected (failure).
  - Sometimes, the order is never called (the promise stays pending/unresolved).

- **Analogy 2: Completion Event**
  - Promises can be thought of as flow-control signals for when an async task finishes.
  - Promises invert the usual callback model: you register to be notified, rather than hand off your code.

---

## 3. Promises in Code

- **Old pattern with callbacks**:
  ```js
  function add(getX, getY, cb) {
    var x, y;
    getX(function(xVal) {
      x = xVal;
      if (y !== undefined) cb(x + y);
    });
    getY(function(yVal) {
      y = yVal;
      if (x !== undefined) cb(x + y);
    });
  }
  add(fetchX, fetchY, function(sum) { console.log(sum); });
  ```
  - Normalizes both now and later, but is clunky and error-prone.

- **Promise pattern**:
  ```js
  function add(xPromise, yPromise) {
    return Promise.all([xPromise, yPromise])
      .then(function(values) {
        return values[0] + values[1];
      });
  }
  add(fetchX(), fetchY()).then(function(sum) {
    console.log(sum);
  });
  ```
  - `Promise.all` waits for all promises to finish.
  - Promises let you reason about future values in a consistent, time-independent way.

**Rejection:**
```js
add(fetchX(), fetchY())
.then(
  function(sum) { console.log(sum); },
  function(err) { console.error(err); }
);
```
- `then` can take two functions: one for fulfill, one for reject.

**Note:**  
Once a promise is resolved (fulfilled or rejected), it becomes immutable and can be observed multiple times.

---

## 4. Event-Like Behavior and Uninversion of Control

- Promises let you subscribe to their completion, not hand over your code.
- Multiple listeners can independently subscribe.
- The promise acts as a neutral third-party between producer and consumer.

**Example:**
```js
function foo(x) {
  return new Promise(function(resolve, reject) {
    // do something async, then:
    // resolve() or reject()
  });
}
var p = foo(42);
bar(p);
baz(p);
```
- Both `bar` and `baz` can independently listen to the outcome.

---

## 5. Thenable Duck Typing

- Any object or function with a `.then(..)` method is considered "thenable" (Promise-like).
- Duck typing can lead to accidental misidentification if objects have a `then` property unintentionally.
- **Example hazard:** Adding `then` to `Object.prototype` or `Array.prototype` causes all objects/arrays to be seen as thenables by Promise logic.

**Note:**  
Duck typing is a compromise—be careful to avoid accidental thenables.

---

## 6. Trust Issues Solved by Promises

Promises are designed to guarantee:

- **Callbacks called too early:**  
  Promises always call `.then()` handlers asynchronously (never synchronously).
- **Callbacks called too late or never:**  
  `.then()` handlers are scheduled when `resolve`/`reject` is called.
- **Never calling the callback:**  
  Use `Promise.race` with a timeout to ensure a signal is always sent.
- **Too few/many calls:**  
  A promise resolves only once; further calls are ignored.
- **Parameter passing:**  
  A promise resolves with a single value (object/array if you need multiple).
- **Swallowing errors:**  
  Errors in promise logic or handlers cause the promise to reject.

**Example:**
```js
var p = new Promise(function(resolve, reject){
  foo.bar(); // throws error
  resolve(42); // never runs
});
p.then(
  function fulfilled(){},
  function rejected(err){ /* err is the exception object */ }
);
```

**Uncaught errors in `.then()` are handled by the next promise in the chain.**

---

## 7. Trusting Promises

- Use `Promise.resolve()` to normalize thenables and values into real, trustable promises.
- `Promise.resolve` unwraps thenables until it gets a non-Promise, non-thenable value.

**Example:**
```js
Promise.resolve(foo(42)).then(function(v) {
  console.log(v);
});
```
- This ensures `.then()` always returns a real, trustable promise.

---

## 8. Chaining Promises (Promise Chain Flow)

- Each `.then()` call returns a new promise, enabling chaining.
- Returning a value from a `.then()` handler fulfills the next promise.
- Returning a promise from a handler causes the next step to wait for that promise.

**Example:**
```js
Promise.resolve(21)
  .then(function(v) {
    return new Promise(function(resolve) {
      setTimeout(function() { resolve(v * 2); }, 100);
    });
  })
  .then(function(v) {
    console.log(v); // 42 after 100ms
  });
```

**Error Propagation:**
- Errors thrown in handlers propagate and reject the next promise in the chain.
- `.catch()` is a shortcut for handling rejections.

**Example:**
```js
request("url1")
  .then(function(response1) {
    return request("url2?v=" + response1);
  })
  .then(function(response2) {
    console.log(response2);
  })
  .catch(function(err) {
    console.error(err);
  });
```

---

## 9. Promise API & Patterns

- **Promise.all([..])**: Waits for all promises to fulfill; rejects if any reject. Returns array of values.
- **Promise.race([..])**: Resolves/rejects as soon as the first promise does.
- **Other patterns:**  
  - `none([..])`: All must reject to fulfill.
  - `any([..])`: Fulfills if any fulfill.
  - `first([..])`: Fulfills as soon as the first fulfills.

**Iteration with Promises:**
- `Promise.map` can be implemented to run a function against all items, returning a promise of results.

---

## 10. Error Handling (and "Pit of Despair")

- Synchronous `try..catch` does not catch async errors.
- Promise error handling is split: one for fulfill, one for reject.
- If you forget to attach a rejection handler, errors may be silently swallowed.
- Best practice: always end chains with `.catch(...)` to avoid unhandled errors.

**Uncaught Handling:**
- Some environments provide "global unhandled rejection" handlers.
- You can polyfill or use libraries for better error reporting.

---

## 11. Promise Limitations

- **Single value:** Only one fulfillment/rejection value per promise.
- **Single resolution:** Promises can't be resolved more than once (not suitable for event streams).
- **Error propagation:** Errors in a chain can be swallowed if not handled properly.
- **Inertia:** Refactoring old callback code to use promises can be challenging.
- **Uncancelable:** Native Promises cannot be canceled; cancellation needs higher-level abstractions.
- **Performance:** Promises add a slight overhead but provide trust and composability.

---

## 12. Review

- Promises are a powerful abstraction for asynchrony, solving trust and control issues that plague callbacks.
- They enable more readable, maintainable, and reason-able async code.
- Promises orchestrate callbacks through a trustable intermediary.
- For advanced needs (cancellation, multi-value, event streams), use promise-based libraries or higher abstractions.

---
