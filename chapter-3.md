# Chapter 3: Promises

In Chapter 2, we identified two major categories of deficiencies with using callbacks to express program asynchrony and manage concurrency: lack of sequentiality and lack of trustability. Now that we understand the problems more intimately, it's time we turn our attention to patterns that can address them.

The issue we want to address first is the inversion of control, the trust that is so fragilely held and so easily lost. Recall that we wrap up the continuation of our program in a callback function, and hand that callback over to another party (potentially even external code) and just cross our fingers that it will do the right thing with the invocation of the callback. We do this because we want to say, "here's what happens later, after the current step finishes."

But what if we could uninvert that inversion of control? What if instead of handing the continuation of our program to another party, we could expect it to return us a capability to know when its task finishes, and then our code could decide what to do next?

This paradigm is called Promises.

Promises are starting to take the JS world by storm, as developers and specification writers alike desperately seek to untangle the insanity of callback hell in their code/design. In fact, most new async APIs being added to JS/DOM platform are being built on Promises. So it's probably a good idea to dig in and learn them, don't you think!?

> Note: The word "immediately" will be used frequently in this chapter, generally to refer to some Promise resolution action. However, in essentially all cases, "immediately" means in terms of the Job queue behavior (see Chapter 1), not in the strictly synchronous now sense.

---

## What Is a Promise?

When developers decide to learn a new technology or pattern, usually their first step is "Show me the code!" It's quite natural for us to just jump in feet first and learn as we go.

But it turns out that some abstractions get lost on the APIs alone. Promises are one of those tools where it can be painfully obvious from how someone uses it whether they understand what it's for and about versus just learning and using the API.

So before I show the Promise code, I want to fully explain what a Promise really is conceptually. I hope this will then guide you better as you explore integrating Promise theory into your own async flow.

With that in mind, let's look at two different analogies for what a Promise is.

---

### Future Value

Imagine this scenario: I walk up to the counter at a fast-food restaurant, and place an order for a cheeseburger. I hand the cashier $1.47. By placing my order and paying for it, I've made a request for a value back (the cheeseburger). I've started a transaction.

But often, the cheeseburger is not immediately available for me. The cashier hands me something in place of my cheeseburger: a receipt with an order number on it. This order number is an IOU ("I owe you") promise that ensures that eventually, I should receive my cheeseburger.

So I hold onto my receipt and order number. I know it represents my future cheeseburger, so I don't need to worry about it anymore -- aside from being hungry!

While I wait, I can do other things, like send a text message to a friend that says, "Hey, can you come join me for lunch? I'm going to eat a cheeseburger."

I am reasoning about my future cheeseburger already, even though I don't have it in my hands yet. My brain is able to do this because it's treating the order number as a placeholder for the cheeseburger. The placeholder essentially makes the value time independent. It's a future value.

Eventually, I hear, "Order 113!" and I gleefully walk back up to the counter with receipt in hand. I hand my receipt to the cashier, and I take my cheeseburger in return.

In other words, once my future value was ready, I exchanged my value-promise for the value itself.

But there's another possible outcome. They call my order number, but when I go to retrieve my cheeseburger, the cashier regretfully informs me, "I'm sorry, but we appear to be all out of cheeseburgers." Setting aside the customer frustration of this scenario for a moment, we can see an important characteristic of future values: they can either indicate a success or failure.

Every time I order a cheeseburger, I know that I'll either get a cheeseburger eventually, or I'll get the sad news of the cheeseburger shortage, and I'll have to figure out something else to eat for lunch.

> Note: In code, things are not quite as simple, because metaphorically the order number may never be called, in which case we're left indefinitely in an unresolved state. We'll come back to dealing with that case later.

---

### Values Now and Later

This all might sound too mentally abstract to apply to your code. So let's be more concrete.

However, before we can introduce how Promises work in this fashion, we're going to derive in code that we already understand -- callbacks! -- how to handle these future values.

When you write code to reason about a value, such as performing math on a number, whether you realize it or not, you've been assuming something very fundamental about that value, which is that it's a concrete now value already:

```js
var x, y = 2;
console.log(x + y); // NaN <-- because `x` isn't set yet
```

The `x + y` operation assumes both `x` and `y` are already set. In terms we'll expound on shortly, we assume the `x` and `y` values are already resolved.

It would be nonsense to expect that the `+` operator by itself would somehow be magically capable of detecting and waiting around until both `x` and `y` are resolved (aka ready), only then to do the operation. That would cause chaos in the program if different statements finished now and others finished later, right?

How could you possibly reason about the relationships between two statements if either one (or both) of them might not be finished yet? If statement 2 relies on statement 1 being finished, there are just two outcomes: either statement 1 finished right now and everything proceeds fine, or statement 1 didn't finish yet, and thus statement 2 is going to fail.

If this sort of thing sounds familiar from Chapter 1, good!

Let's go back to our `x + y` math operation. Imagine if there was a way to say, "Add x and y, but if either of them isn't ready yet, just wait until they are. Add them as soon as you can."

Your brain might have just jumped to callbacks. OK, so...

```js
function add(getX,getY,cb) {
  var x, y;
  getX(function(xVal){
    x = xVal;
    // both are ready?
    if (y != undefined) {
      cb(x + y); // send along sum
    }
  });
  getY(function(yVal){
    y = yVal;
    // both are ready?
    if (x != undefined) {
      cb(x + y); // send along sum
    }
  });
}
// `fetchX()` and `fetchY()` are sync or async
// functions
add(fetchX, fetchY, function(sum){
  console.log(sum); // that was easy, huh?
});
```

Take just a moment to let the beauty (or lack thereof) of that snippet sink in (whistles patiently).

While the ugliness is undeniable, there's something very important to notice about this async pattern.

In that snippet, we treated `x` and `y` as future values, and we express an operation `add(..)` that (from the outside) does not care whether `x` or `y` or both are available right away or not. In other words, it normalizes the now and later, such that we can rely on a predictable outcome of the `add(..)` operation.

By using an `add(..)` that is temporally consistent -- it behaves the same across now and later times -- the async code is much easier to reason about.

To put it more plainly: to consistently handle both now and later, we make both of them later: all operations become async.

Of course, this rough callbacks-based approach leaves much to be desired. It's just a first tiny step toward realizing the benefits of reasoning about future values without worrying about the time aspect of when it's available or not.

---

### Promise Value

We'll definitely go into a lot more detail about Promises later in the chapter -- so don't worry if some of this is confusing -- but let's just briefly glimpse at how we can express the `x + y` example via Promises:

```js
function add(xPromise,yPromise) {
  // `Promise.all([ .. ])` takes an array of promises,
  // and returns a new promise that waits on them
  // all to finish
  return Promise.all([xPromise, yPromise])
    // when that promise is resolved, let's take the
    // received `X` and `Y` values and add them together.
    .then(function(values){
      // `values` is an array of the messages from the
      // previously resolved promises
      return values[0] + values[1];
    });
}
// `fetchX()` and `fetchY()` return promises for
// their respective values, which may be ready
// *now* or *later*.
add(fetchX(), fetchY())
  // we get a promise back for the sum of those
  // two numbers.
  // now we chain-call `then(..)` to wait for the
  // resolution of that returned promise.
  .then(function(sum){
    console.log(sum); // that was easier!
  });
```

There are two layers of Promises in this snippet. `fetchX()` and `fetchY()` are called directly, and the values they return (promises!) are passed into `add(..)`. The underlying values those promises represent may be ready now or later, but each promise normalizes the behavior to be the same regardless. We reason about X and Y values in a time-independent way. They are future values.

The second layer is the promise that `add(..)` creates (via `Promise.all([ .. ])`) and returns, which we wait on by calling `then(..)`. When the `add(..)` operation completes, our `then(..)` at the end of the snippet -- is actually operating on that second promise returned, rather than the first one created by `Promise.all([ .. ])`.

> Note: Inside `add(..)`, the logic for waiting on the X and Y future values is hidden inside of `Promise.all([ .. ])`. Also, though we are not chaining off the end of that second `then(..)`, it too has created another promise, had we chosen to observe/use it. This Promise chaining stuff will be explained in much greater detail later in this chapter.

Just like with cheeseburger orders, it's possible that the resolution of a Promise is rejection instead of fulfillment. Unlike a fulfilled Promise, where the value is always programmatic, a rejection value -- commonly called a "rejection reason" -- can either be set directly by the program logic, or it can result implicitly from a runtime exception.

With Promises, the `then(..)` call can actually take two functions, the first for fulfillment (as shown earlier), and the second for rejection:

```js
add(fetchX(), fetchY())
  .then(
    // fullfillment handler
    function(sum) {
      console.log(sum);
    },
    // rejection handler
    function(err) {
      console.error(err); // bummer!
    }
  );
```

If something went wrong getting X or Y, or something somehow failed during the addition, the promise that `add(..)` returns is rejected, and the second callback error handler passed to `then(..)` will receive the rejection value from the promise.

Because Promises encapsulate the time-dependent state -- waiting on the fulfillment or rejection of the underlying value -- from the outside, the Promise itself is time-independent, and thus Promises can be composed (combined) in predictable ways regardless of the timing or outcome underneath.

Moreover, once a Promise is resolved, it stays that way forever -- it becomes an immutable value at that point -- and can then be observed as many times as necessary.

> Note: Because a Promise is externally immutable once resolved, it's now safe to pass that value around to any party and know that it cannot be modified accidentally or maliciously. This is especially true in relation to multiple parties observing the resolution of a Promise. It is not possible for one party to affect another party's ability to observe Promise resolution. Immutability may sound like an academic topic, but it's actually one of the most fundamental and important aspects of Promise design, and shouldn't be casually passed over.

That's one of the most powerful and important concepts to understand about Promises. With a fair amount of work, you could ad hoc create the same effects with nothing but ugly callback composition, but that's not really an effective strategy, especially because you have to do it over and over again.

Promises are an easily repeatable mechanism for encapsulating and composing future values.

---

## Completion Event

As we just saw, an individual Promise behaves as a future value. But there's another way to think of the resolution of a Promise: as a flow-control mechanism -- a temporal this-then-that -- for two or more steps in an asynchronous task.

Let's imagine calling a function `foo(..)` to perform some task. We don't know about any of its details, nor do we care. It may complete the task right away, or it may take a while.

We just simply need to know when `foo(..)` finishes so that we can move on to our next task. In other words, we'd like a way to be notified of `foo(..)`'s completion so that we can continue.

In typical JavaScript fashion, if you need to listen for a notification, you'd likely think of that in terms of events. So we could reframe our need for notification as a need to listen for a completion (or continuation) event emitted by `foo(..)`.

> Note: Whether you call it a "completion event" or a "continuation event" depends on your perspective. Is the focus more on what happens with `foo(..)`, or what happens after `foo(..)` finishes? Both perspectives are accurate and useful. The event notification tells us that `foo(..)` has completed, but also that it's OK to continue with the next step. Indeed, the callback you pass to be called for the event notification is itself what we've previously called a continuation. Because completion event is a bit more focused on the `foo(..)`, which more has our attention at present, we slightly favor completion event for the rest of this text.

With callbacks, the "notification" would be our callback invoked by the task (`foo(..)`). But with Promises, we turn the relationship around, and expect that we can listen for an event from `foo(..)`, and when notified, proceed accordingly.

First, consider some pseudocode:

```
foo(x){
  // start doing something that could take a while
}
foo(42)
on(foo "completion") {
  // now we can do the next step!
}
on(foo "error") {
  // oops, something went wrong in `foo(..)`
}
```

We call `foo(..)` and then we set up two event listeners, one for "completion" and one for "error" -- the two possible final outcomes of the `foo(..)` call. In essence, `foo(..)` doesn't even appear to be aware that the calling code has subscribed to these events, which makes for a very nice separation of concerns.

Unfortunately, such code would require some "magic" of the JS environment that doesn't exist (and would likely be a bit impractical). Here's the more natural way we could express that in JS:

```js
function foo(x) {
  // start doing something that could take a while
  // make a `listener` event notification
  // capability to return
  return listener;
}
var evt = foo(42);
evt.on("completion", function(){
  // now we can do the next step!
});
evt.on("failure", function(err){
  // oops, something went wrong in `foo(..)`
});
```

`foo(..)` expressly creates an event subscription capability to return back, and the calling code receives and registers the two event handlers against it.

The inversion from normal callback-oriented code should be obvious, and it's intentional. Instead of passing the callbacks to `foo(..)`, it returns an event capability we call `evt`, which receives the callbacks.

But if you recall from Chapter 2, callbacks themselves represent an inversion of control. So inverting the callback pattern is actually an inversion of inversion, or an uninversion of control -- restoring control back to the calling code where we wanted it to be in the first place.

One important benefit is that multiple separate parts of the code can be given the event listening capability, and they can all independently be notified of when `foo(..)` completes to perform subsequent steps after its completion:

```js
var evt = foo(42);
// let `bar(..)` listen to `foo(..)`'s completion
bar(evt);
// also, let `baz(..)` listen to `foo(..)`'s completion
baz(evt);
```

Uninversion of control enables a nicer separation of concerns, where how `foo(..)` is called. Similarly, `bar(..)` and `baz(..)` don't need to be involved in `foo(..)`'s implementation or internal logic.

---

## Promise "Events"

As you may have guessed by now, the `evt` event listening capability is an analogy for a Promise.

In a Promise-based approach, the previous snippet would have `foo(..)` creating and returning a Promise instance, and that promise would then be passed to `bar(..)` and `baz(..)`.

> Note: The Promise resolution "events" we listen for aren't strictly events (though they certainly behave like events for these purposes), and they're not typically called "completion" or "error". Instead, we use `then(..)` to register a "then" event. Or perhaps more precisely, `then(..)` registers "fulfillment" and/or "rejection" event(s), though we don't see those terms used explicitly in the code.

Consider:

```js
function foo(x) {
  // start doing something that could take a while
  // construct and return a promise
  return new Promise(function(resolve, reject){
    // eventually, call `resolve(..)` or `reject(..)`,
    // which are the resolution callbacks for
    // the promise.
  });
}
var p = foo(42);
bar(p);
baz(p);
```

> Note: The pattern shown with `new Promise(function(..){ .. })` is generally called the "revealing constructor". The function passed in is executed immediately (not async deferred, as callbacks to `then(..)` are), and it's provided two parameters, which in this case we've named `resolve` and `reject`. These are the resolution functions for the promise. `resolve(..)` generally signals fulfillment, and `reject(..)` signals rejection.

You can probably guess what the internals of `bar(..)` and `baz(..)` might look like:

```js
function bar(fooPromise) {
  // listen for `foo(..)` to complete
  fooPromise.then(
    function(){
      // `foo(..)` has now finished, so
      // do `bar(..)`'s task
    },
    function(){
      // oops, something went wrong in `foo(..)`
    }
  );
}
// ditto for `baz(..)`
```

Promise resolution doesn't necessarily need to involve sending along a message, as it did when we were examining Promises as future values. It can just be a flow-control signal, as used in the previous snippet.

Another way to approach this is:

```js
function bar() {
  // `foo(..)` has definitely finished, so
  // do `bar(..)`'s task
}
function oopsBar() {
  // oops, something went wrong in `foo(..)`,
  // so `bar(..)` didn't run
}
// ditto for `baz()` and `oopsBaz()`

var p = foo(42);
p.then(bar, oopsBar);
p.then(baz, oopsBaz);
```

> Note: If you've seen Promise-based coding before, you might be tempted to believe that the last two lines of that code could be written as `p.then(..).then(..)`, using chaining, rather than `p.then(..); p.then(..)`. That would have an entirely different behavior, so be careful! The difference might not be clear right now, but it's actually a different async pattern than we've seen thus far: splitting/forking. Don't worry! We'll come back to this point later in this chapter.

Instead of passing the `p` promise to `bar(..)` and `baz(..)`, we use the promise to control when `bar(..)` and `baz(..)` will get executed, if ever. The primary difference is in the error handling.

In the first snippet's approach, `bar(..)` is called regardless of whether `foo(..)` succeeds or fails, and it handles its own fallback logic if it's notified that `foo(..)` failed. The same is true for `baz(..)`, obviously.

In the second snippet, `bar(..)` only gets called if `foo(..)` succeeds, and otherwise `oopsBar(..)` gets called. Ditto for `baz(..)`.

Neither approach is correct per se. There will be cases where one is preferred over the other.

In either case, the promise `p` that comes back from `foo(..)` is used to control what happens next.

Moreover, the fact that both snippets end up calling `then(..)` twice against the same promise `p` illustrates the point made earlier, which is that Promises (once resolved) retain their same resolution (fulfillment or rejection) forever, and can subsequently be observed as many times as necessary.

Whenever `p` is resolved, the next step will always be the same, both now and later.

---

## Thenable Duck Typing

In Promises-land, an important detail is how to know for sure if some value is a genuine Promise or not. Or more directly, is it a value that will behave like a Promise?

Given that Promises are constructed by the `new Promise(..)` syntax, you might think that `p instanceof Promise` would be an acceptable check. But unfortunately, there are a number of reasons that's not totally sufficient.

Mainly, you can receive a Promise value from another browser window (iframe, etc.), which would have its own Promise different from the one in the current window/frame, and that check would fail to identify the Promise instance.

Moreover, a library or framework may choose to vend its own Promises and not use the native ES6 Promise implementation to do so. In fact, you may very well be using Promises with libraries in older browsers that have no Promise at all.

When we discuss Promise resolution processes later in this chapter, it will become more obvious why a non-genuine-but Promise-like value would still be very important to be able to recognize and assimilate. But for now, just take my word for it that it's a critical piece of the puzzle.

As such, it was decided that the way to recognize a Promise (or something that behaves like a Promise) would be to define something called a "thenable" as any object or function which has a `then(..)` method on it. It is assumed that any such value is a Promise-conforming thenable.

The general term for "type checks" that make assumptions about a value's "type" based on its shape (what properties are present) is called "duck typing" -- "If it looks like a duck, and quacks like a duck, it must be a duck" (see the Types & Grammar title of this book series). So the duck typing check for a thenable would roughly be:

```js
if (
  p !== null &&
  (
    typeof p === "object" ||
    typeof p === "function"
  ) &&
  typeof p.then === "function"
) {
  // assume it's a thenable!
}
else {
  // not a thenable
}
```

Yuck! Setting aside the fact that this logic is a bit ugly to implement in various places, there's something deeper and more troubling going on.

If you try to fulfill a Promise with any object/function value that happens to have a `then(..)` function on it, but you weren't intending it to be treated as a Promise/thenable, you're out of luck, because it will automatically be recognized as thenable and treated with special rules (see later in the chapter).

This is even true if you didn't realize the value has a `then(..)` on it. For example:

```js
var o = { then: function(){} };
// make `v` be `[[Prototype]]`-linked to `o`
var v = Object.create(o);
v.someStuff = "cool";
v.otherStuff = "not so cool";
v.hasOwnProperty("then"); // false
```

`v` doesn't look like a Promise or thenable at all. It's just a plain object with some properties on it. You're probably just intending to send that value around like any other object.

But unknown to you, `v` is also `[[Prototype]]`-linked (see the this & Object Prototypes title of this book series) to another object `o`, which happens to have a `then(..)` on it. So the thenable duck typing checks will think and assume `v` is a thenable. Uh oh.

It doesn't even need to be something as directly intentional as that:

```js
Object.prototype.then = function(){};
Array.prototype.then = function(){};
var v1 = { hello: "world" };
var v2 = [ "Hello", "World" ];
```

Both then(..) to `v1` and `v2` will be assumed to be thenables. You can't control or predict if any other code accidentally or maliciously adds `then(..)` to Object.prototype, Array.prototype, or any of the other native prototypes. And if what's specified is a function that doesn't call either of its parameters as callbacks, then any Promise resolved with such a value will just silently hang forever! Crazy.

Sound implausible or unlikely? Perhaps.

But keep in mind that there were several well-known non-Promise libraries preexisting in the community prior to ES6 that happened to already have a method on them called `then(..)`. Some of those libraries chose to rename their own methods to avoid collision (that sucks!). Others have simply been relegated to the unfortunate status of "incompatible with Promise-based coding" in reward for their inability to change to get out of the way.

The standards decision to hijack the previously nonreserved -- and completely general-purpose sounding -- `then` property name means that no value (or any of its delegates), either past, present, or future, can have a `then(..)` function present, either on purpose or by accident, or that value will be confused for a thenable in Promises systems, which will probably create bugs that are really hard to track down.

> Warning: I do not like how we ended up with duck typing of thenables for Promise recognition. There were other options, such as "branding" or even "anti-branding"; what we got seems like a worst-case compromise. But it's not all doom and gloom. Thenable duck typing can be helpful, as we'll see later. Just beware that thenable duck typing can be hazardous if it incorrectly identifies something as a Promise that isn't.

---

---

## Promise Trust

We've now seen two strong analogies that explain different aspects of what Promises can do for our async code. But if we stop there, we've missed perhaps the single most important characteristic that the Promise pattern establishes: trust.

Whereas the future values and completion events analogies play out explicitly in the code patterns we've explored, it won't be entirely obvious why or how Promises are designed to solve all of the inversion of control trust issues we laid out in the "Trust Issues" section of Chapter 2. But with a little digging, we can uncover some important guarantees that restore the confidence in async coding that Chapter 2 tore down!

Let's start by reviewing the trust issues with callbacks-only coding. When you pass a callback to a utility foo(..), it might:
- Call the callback too early
- Call the callback too late (or never)
- Call the callback too few or too many times
- Fail to pass along any necessary environment/parameters
- Swallow any errors/exceptions that may happen

The characteristics of Promises are intentionally designed to provide useful, repeatable answers to all these concerns.

---

### Calling Too Early

Primarily, this is a concern of whether code can introduce Zalgo-like effects (see Chapter 2), where sometimes a task finishes synchronously and sometimes asynchronously, which can lead to race conditions.

Promises by definition cannot be susceptible to this concern, because even an immediately fulfilled Promise (like Promise(function(resolve){ resolve(42); })) cannot be observed synchronously.

That is, when you call new then(..) on a Promise, even if that Promise was already resolved, the callback you provide to then(..) will always be called asynchronously (for more on this, refer back to "Jobs" in Chapter 1).

No more need to insert your own setTimeout(..,0) hacks. Promises prevent Zalgo automatically.

---

### Calling Too Late

Similar to the previous point, a Promise's then(..) registered observation callbacks are automatically scheduled when either resolve(..) or reject(..) are called by the Promise creation capability. Those scheduled callbacks will predictably be fired at the next asynchronous moment (see "Jobs" in Chapter 1).

It's not possible for synchronous observation, so it's not possible for a synchronous chain of tasks to run in such a way to in effect "delay" another callback from happening as expected. That is, when a Promise is resolved, all then(..) registered callbacks on it will be called, in order, immediately at the next asynchronous opportunity (again, see "Jobs" in Chapter 1), and nothing that happens inside of one of those callbacks can affect/delay the calling of the other callbacks.

For example:

```js
p.then(function(){
  p.then(function(){
    console.log("C");
  });
  console.log("A");
});
p.then(function(){
  console.log("B");
});
// A B C
```

Here, "C" cannot interrupt and precede "B", by virtue of how Promises are defined to operate.

---

### Promise Scheduling Quirks

It's important to note, though, that there are lots of nuances of scheduling where the relative ordering between callbacks chained off two separate Promises is not reliably predictable.

If two promises p1 and p2 are both already resolved, it should be true that calling the callback(s) for p1 before the ones for p2 would end up calling the callback(s) for p1 before the ones for p2. But there are subtle cases where that might not be true, such as the following:

```js
var p3 = new Promise(function(resolve,reject){
  resolve("B");
});
var p1 = new Promise(function(resolve,reject){
  resolve(p3);
});
var p2 = new Promise(function(resolve,reject){
  resolve("A");
});
p1.then(function(v){
  console.log(v);
});
p2.then(function(v){
  console.log(v);
});
// A B  <-- not B A as you might expect
```

We'll cover this more later, but as you can see, p1 is resolved not with an immediate value, but with another promise p3 which is itself resolved with the value "B". The specified behavior is to unwrap p3 into p1, but asynchronously, so p1's callback(s) are behind p2's callback(s) in the asynchronous Job queue (see Chapter 1).

To avoid such nuanced nightmares, you should never rely on anything about the ordering/scheduling of callbacks across Promises. In fact, a good practice is not to code in such a way where the ordering of multiple callbacks matters at all. Avoid that if you can.

---

### Never Calling the Callback

This is a very common concern. It's addressable in several ways with Promises.

First, nothing (not even a JS error) can prevent a Promise from notifying you of its resolution (if it's resolved). If you register both fulfillment and rejection callbacks for a Promise, and the Promise gets resolved, one of the two callbacks will always be called.

Of course, if your callbacks themselves have JS errors, you may not see the outcome you expect, but the callback will in fact have been called. We'll cover later how to be notified of an error in your callback, because even those don't get swallowed.

But what if the Promise itself never gets resolved either way? Even that is a condition that Promises provide an answer for, using a higher level abstraction called a "race":

```js
// a utility for timing out a Promise
function timeoutPromise(delay) {
  return new Promise(function(resolve,reject){
    setTimeout(function(){
      reject("Timeout!");
    }, delay);
  });
}
// setup a timeout for `foo()`
Promise.race([
  foo(),                 // attempt `foo()`
  timeoutPromise(3000)   // give it 3 seconds
])
.then(
  function(){
    // `foo(..)` fulfilled in time!
  },
  function(err){
    // either `foo()` rejected, or it just
    // didn't finish in time, so inspect
    // `err` to know which
  }
);
```

There are more details to consider with this Promise timeout pattern, but we'll come back to it later.

Importantly, we can ensure a signal as to the outcome of foo(), to prevent it from hanging our program indefinitely.

---

### Calling Too Few or Too Many Times

By definition, one is the appropriate number of times for the callback to be called. The "too few" case would be zero calls, which is the same as the "never" case we just examined.

The "too many" case is easy to explain. Promises are defined so that they can only be resolved once. If for some reason the Promise creation code tries to call resolve(..) or reject(..) multiple times, or tries to call both, the Promise will accept only the first resolution, and will silently ignore any subsequent attempts.

Because a Promise can only be resolved once, any then(..) registered callbacks will only ever be called once (each).

Of course, if you register the same callback more than once, (e.g., p.then(f); p.then(f);), it'll be called as many times as it was registered. The guarantee that a response function is called only once does not prevent you from shooting yourself in the foot.

---

### Failing to Pass Along Any Parameters/Environment

Promises can have, at most, one resolution value (fulfillment or rejection).

If you don't explicitly resolve with a value either way, the value is undefined, as is typical in JS. But whatever the value, it will always be passed to all registered (and appropriate: fulfillment or rejection) callbacks, either now or in the future.

Something to be aware of: If you call resolve(..) or reject(..) with multiple parameters, all subsequent parameters beyond the first will be silently ignored. Although that might seem a violation of the guarantee we just described, it's not exactly, because it constitutes an invalid usage of the Promise mechanism. Other invalid usages of the API (such as calling resolve(..) multiple times) are similarly protected, so the Promise behavior here is consistent (if not a tiny bit frustrating).

If you want to pass along multiple values, you must wrap them in another single value that you pass, such as an array or an object.

As for environment, functions in JS always retain their closure of the scope in which they're defined (see the Scope & Closures title of this series), so they of course would continue to have access to whatever surrounding state you provide. Of course, the same is true of callbacks-only design, so this isn't a specific augmentation of benefit from Promises -- but it's a guarantee we can rely on nonetheless.

---

### Swallowing Any Errors/Exceptions

In the base sense, this is a restatement of the previous point. If you reject a Promise with a reason (aka error message), that value is passed to the rejection callback(s).

But there's something much bigger at play here. If at any point in the creation of a Promise, or in the observation of its resolution, a JS exception error occurs, such as a TypeError or ReferenceError, that exception will be caught, and it will force the Promise in question to become rejected.

For example:

```js
var p = new Promise(function(resolve,reject){
  foo.bar(); // `foo` is not defined, so error!
  resolve(42); // never gets here :(
});
p.then(
  function fulfilled(){
    // never gets here :(
  },
  function rejected(err){
    // `err` will be a `TypeError` exception object
    // from the `foo.bar()` line.
  }
);
```

The JS exception that occurs from foo.bar() becomes a Promise rejection that you can catch and respond to.

This is an important detail, because it effectively solves another potential Zalgo moment, which is that errors could create a synchronous reaction whereas nonerrors would be asynchronous. Promises turn even JS exceptions into asynchronous behavior, thereby reducing the race condition chances greatly.

But what happens if a Promise is fulfilled, but there's a JS exception error during the observation (in a then(..) registered callback)? Even those aren't lost, but you may find how they're handled a bit surprising, until you dig in a little deeper:

```js
var p = new Promise(function(resolve,reject){
  resolve(42);
});
p.then(
  function fulfilled(msg){
    foo.bar();
    console.log(msg); // never gets here :(
  },
  function rejected(err){
    // never gets here either :(
  }
);
```

Wait, that makes it seem like the exception from foo.bar() really did get swallowed. Never fear, it didn't. But something deeper is wrong, which is that we've failed to listen for it. The p.then(..) call itself returns another promise, and it's that promise that will be rejected with the TypeError exception.

Why couldn't it just call the error handler we have defined there? Seems like a logical behavior on the surface. But it would violate the fundamental principle that Promises are immutable once resolved. p was already fulfilled to the value 42, so it can't later be changed to a rejection just because there's an error in observing p's resolution.

Besides the principle violation, such behavior could wreak havoc, if say there were multiple then(..) registered callbacks on the promise p, because some would get called and others wouldn't, and it would be very opaque as to why.

---

---

## Trustable Promise?

Promises are now a first-class citizen in JS (as of ES6), and the Promise constructor provides the "revealing constructor" pattern we've already discussed.

But here's an important question: if you receive a Promise from somewhere, how do you know it's actually trustworthy? For instance, it might be an object that looks like a Promise—maybe it has a `then(..)` method—but it's not a genuine, spec-conformant, trustable Promise. Maybe it's a thenable from a third-party library that doesn't fulfill all the ES6 Promise behaviors.

So how do you normalize a value into a real, trustworthy Promise? That's where `Promise.resolve(..)` comes in.

---

### Promise.resolve

The `Promise.resolve(..)` utility takes any value and wraps it in a real, genuine, trustable ES6 Promise. If the value is already a real Promise, it returns it unchanged. If the value is a thenable, it assimilates ("unwraps") it recursively until a non-thenable value or a real Promise is found.

```js
Promise.resolve(42) === Promise.resolve(Promise.resolve(42)); // true
```

This means, you should always use `Promise.resolve(..)` on any value (especially any thenable) before you try to use it as a Promise in your code.

```js
Promise.resolve(foo(42)).then(function(v){
  console.log(v);
});
```

Regardless of what `foo(42)` returns—immediate value, thenable, or real Promise—`.then(..)` will work as expected.

---

## Chaining Flow

One of the most powerful features of Promises is chaining.

Every time you call `.then(..)` on a Promise, it creates and returns a new Promise, which can then be observed by another `.then(..)`, and so on.

```js
Promise.resolve(21)
.then(function(v){
  // fulfilled with 21
  return v * 2;
})
.then(function(v){
  // fulfilled with 42
  console.log(v); // 42
});
```

If you return a value from a `.then(..)` handler, the next `.then(..)` in the chain receives that value as its fulfillment.

If you return a promise or thenable, the next `.then(..)` waits for it to resolve before continuing.

```js
Promise.resolve(21)
.then(function(v){
  return new Promise(function(resolve,reject){
    setTimeout(function(){
      resolve(v * 2);
    }, 100);
  });
})
.then(function(v){
  // fulfilled with 42 after 100ms
  console.log(v); // 42
});
```

If you throw an error, or return a promise that rejects, the next rejection handler in the chain will be called.

```js
Promise.resolve(21)
.then(function(v){
  return new Promise(function(resolve,reject){
    throw "Uh oh!";
  });
})
.then(
  function(v){
    // never gets here :(
  },
  function(err){
    // err is "Uh oh!"
  }
);
```

If you omit a handler (e.g., `.then(null, rejectionHandler)`), a default pass-through handler is substituted. This means errors can "bubble" until caught.

---

## .catch(..)

The `.catch(..)` method is a shorthand for `.then(null, rejectionHandler)`.

```js
Promise.resolve(21)
.then(function(v){
  foo.bar(); // error!
  return v * 2;
})
.catch(function(err){
  // err is a TypeError exception object
});
```

You can also "recover" from errors by returning a value from a `.catch(..)` handler, which resumes the chain as fulfilled.

---

## Chaining and Forking

Each `.then(..)` call returns a new promise, so you can "fork" a promise chain:

```js
var p = Promise.resolve(21);
p.then(function(v){
  return v * 2;
});
p.then(function(v){
  return v * 3;
});
```

Each chain proceeds independently. This is fundamentally different than chaining `.then(..)` calls one after another, which is sequential.

---

## Orchestration Patterns

### Promise.all(..): Gate

`Promise.all([ .. ])` takes an array (or iterable) of promises (or values). It returns a new promise that waits for all input promises to fulfill. If any input rejects, it rejects immediately.

```js
var p1 = request( "http://some.url.1" );
var p2 = request( "http://some.url.2" );

Promise.all( [p1, p2] )
.then(function(msgs){
  // both requests fulfilled
  return request( "http://some.url.3/?v=" + msgs.join(",") );
})
.then(function(msg){
  // handle result from third request
});
```

If any input rejects, the returned promise rejects with the first rejection reason.

---

### Promise.race(..): Latch

`Promise.race([ .. ])` returns a promise that settles as soon as any input settles (fulfill or reject).

```js
Promise.race( [p1, p2] )
.then(function(msg){
  // first to fulfill
}, function(err){
  // first to reject
});
```

If the input array is empty, `Promise.race` never settles.

---

### Promise.allSettled(..): Waiting for All Outcomes

`Promise.allSettled([ .. ])` returns a promise that fulfills after all inputs settle, regardless of whether they fulfill or reject. The result is an array of objects describing the outcome of each input.

```js
Promise.allSettled([p1, p2])
.then(function(results){
  results.forEach(function(result){
    if (result.status === "fulfilled") {
      // handle result.value
    } else {
      // handle result.reason
    }
  });
});
```

---

### Promise.any(..): First Fulfillment

`Promise.any([ .. ])` fulfills as soon as any input fulfills, or rejects if all inputs reject (with an AggregateError).

---

#### Custom Patterns

- **none([ .. ])**: fulfills only if all inputs reject.
- **first([ .. ])**: fulfills as soon as the first input fulfills (ignores rejections).
- **last([ .. ])**: only the last fulfillment wins.

These are not built in, but can be implemented with Promise combinator logic.

---

### Promise.map(..): Concurrent Iteration

A common pattern is to perform an async operation on each item in a collection and collect all results.

```js
Promise.all( arr.map(asyncFn) )
.then(function(results){
  // results in order
});
```

---

## Error Handling Patterns and Pitfalls

- Always use `.catch(..)` at the end of a promise chain to avoid the "pit of despair" (silent swallowed errors).
- Errors thrown in executor functions or `.then(..)` handlers become rejections.
- If a promise is rejected and never handled, browsers/Node.js may warn about "unhandled promise rejection".

---

## .finally(..) and Promise.observe

- `.finally(..)` runs a handler regardless of fulfillment or rejection.
- Custom patterns like `Promise.observe` can be used to "peek" at a promise's outcome for logging or cleanup.

---

## Polyfills and Promisification

- Use `Promise.resolve(..)` to normalize third-party thenables.
- Use helper functions to "promisify" callback-based APIs.

```js
function promisify(fn) {
  return function(...args){
    return new Promise(function(resolve, reject){
      fn(...args, function(err, val){
        if (err) reject(err);
        else resolve(val);
      });
    });
  };
}
```

---

## Limitations

- Promises are single-value, single-resolution; not for event streams.
- Promises cannot be cancelled (natively); use higher abstractions for cancellation.
- Promises add some overhead compared to bare callbacks, but provide much more safety and composability.

---

## Final Notes

Promises are a foundational abstraction for async JS programming. They solve callback hell, uninvert control, guarantee trust, and enable powerful composition and orchestration patterns. For advanced needs—streams, cancellation, multi-value—look to further abstractions or libraries. But for most code, promises are the right tool.

---
