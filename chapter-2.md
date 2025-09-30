# Chapter 2: Callbacks

Callbacks are the most fundamental way to express asynchrony in JavaScript. In fact, every single async program you've ever written or used in JS, from the simplest one-liner to the most complex, is built on callbacks.

---

## Continuations and the "Gap"

A callback is a function that you pass to another function, to be "called back" later when an async event completes. In other words, the callback represents the continuation of your program.

Whenever you introduce a callback, you introduce a gap—a place in your code where the flow "jumps" from one part to another, often after some asynchronous event.

Example:

```js
// A
setTimeout(function(){
  // C
}, 1000);
// B
```

Here, "A" runs, then "B" runs, and after 1 second, "C" runs. The "gap" is the time between B and C.

---

## The Sequential Brain vs. Async Code

Humans naturally plan sequentially: do A, then B, then C. Synchronous code works the same way.

But with asynchrony, our brains struggle. We have to plan around the fact that code may run "out of order" or later than expected.

Example (synchronous swap):

```js
z = x;
x = y;
y = z;
```

Async code with callbacks makes this sort of sequential reasoning difficult, especially as your program grows.

---

## Callback Hell (Pyramid of Doom)

Callback hell is what happens when you chain callbacks deeply, making code hard to read and reason about. It's not just about indentation—it's about the loss of clarity in program flow.

Example:

```js
listen("click", function handler(evt){
  setTimeout(function request(){
    ajax("http://some.url.1", function response(text){
      if (text == "hello") handler();
      else if (text == "world") request();
    });
  }, 500);
});
```

Even if you name your callbacks and flatten them, the logical flow is still hard to follow.

---

## Inversion of Control and Trust Issues

When you pass a callback to another party (a library, an API), you give up control over how and when your function is called. This is called **inversion of control**.

You have to trust the other party to:
- Call your callback at the correct time (not too early, not too late, not never)
- Call it the correct number of times (not zero, not more than once)
- Pass the correct parameters/environment
- Not swallow errors/exceptions

---

## Real-World Trust Hazards

Suppose a library calls your callback multiple times by accident. For example:

```js
analytics.trackPurchase(purchaseData, function(){
  chargeCreditCard();
  displayThankyouPage();
});
```

If the library calls your callback 5 times, a customer may be charged 5 times! You can add a latch:

```js
var tracked = false;
analytics.trackPurchase(purchaseData, function(){
  if (!tracked) {
    tracked = true;
    chargeCreditCard();
    displayThankyouPage();
  }
});
```

But what if the callback is never called? Or called with the wrong arguments? Or an error is swallowed?

---

## Defensive Patterns for Callbacks

Defensive programming is needed to guard against these hazards. Examples:
- Type validation
- Conversion/casting
- Latches, gates, and timeouts

It's not just for third-party code—you should do this even in your own code.

---

## Callback API Patterns and Their Drawbacks

### Split Callbacks (Success/Error)

```js
function success(data) { ... }
function failure(err) { ... }
ajax("url", success, failure);
```
- Verbose; does not guarantee correct behavior.

### Error-First Callbacks (Node.js/Errback Style)

```js
function response(err, data) { ... }
ajax("url", response);
```
- If error, first argument is set; otherwise, it's falsy.

### Timeoutify Utility

```js
function timeoutify(fn, delay) {
  var intv = setTimeout(function(){
    intv = null;
    fn(new Error("Timeout!"));
  }, delay);
  return function() {
    if (intv) {
      clearTimeout(intv);
      fn.apply(this, arguments);
    }
  };
}
ajax("url", timeoutify(foo, 500));
```
- Ensures callback is called after a delay, if not already.

### Asyncify Utility (Preventing Zalgo)

```js
function asyncify(fn) {
  var orig_fn = fn,
      intv = setTimeout(function(){
        intv = null;
        if (fn) fn();
      }, 0);
  fn = null;
  return function() {
    if (intv) {
      fn = orig_fn.bind.apply(
        orig_fn,
        [this].concat([].slice.call(arguments))
      );
    } else {
      orig_fn.apply(this, arguments);
    }
  };
}
ajax("url", asyncify(result));
```
- Ensures callback is always async, regardless of underlying event source.

---

## The Zalgo Problem

Some APIs may call your callback synchronously or asynchronously depending on context (cache hit vs. network). This unpredictability is called "releasing Zalgo" and is a major source of bugs. Always async your callbacks to ensure predictable async flow.

---

## Real-World Example: The Tale of Five Callbacks

A real bug: an analytics utility calls the callback 5 times, charging a customer 5 times.

Defensive code (latch) helps, but you also have to worry about:
- Callback never called
- Callback called too early/late
- Callback called too many/few times
- Environment/parameters not passed
- Errors/exceptions swallowed

---

## Limitations of Callbacks

- Repetitive, error-prone defensive code is required everywhere.
- Patterns like split callbacks, error-first, timeoutify, asyncify help, but are verbose and don't solve all problems.
- You can't even fully trust your own code.

---

## Callbacks and Sequential Brain Planning

Our brains plan tasks as a sequence ("to-do" lists). When forced to plan for asynchrony, our thinking (and code) becomes messy and nonlinear. Callback hell is not just about nesting—it's about the loss of clarity and predictability in mapping planned steps to code and code to execution.

---

## Why Callbacks Are Not Enough

- Callbacks break our natural, sequential reasoning, making code harder to reason about, debug, and maintain.
- Inversion of control leads to trust issues and repetitive, defensive code.
- Ad hoc utilities (split callbacks, error-first, timeoutify, asyncify) add boilerplate and don't address all issues.

---

## The Need for Something Better

Promises (next chapter) are the answer: a more robust, standardized way to express asynchrony, sequentiality, and trust in JS code.

---
