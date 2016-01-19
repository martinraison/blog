---
layout: post
title:  "Clojure & core.async: go blocks everywhere?"
date:   2015-07-27 19:26:37
categories: clojure
---

While not part of the Clojure core library, [core.async][core-async]'s `go` blocks are very appealing for writing asynchronous code. In fact, it is very tempting to use`go` and `<!` as replacements for the `async` and `await` primitives for coroutines found in other programming languages such as [C#][c-sharp], [Hack][hack] or (soon) [Python][python].

Here is a toy example in Python:

{% highlight python %}
async def f(db, query):
    data = await db.fetch(query)
    return data + 10
{% endhighlight %}

and what could be the corresponding Clojure code:

{% highlight clojure %}
(defn f [db query]
  (go (let [data (<! (fetch db query))]
        (+ data 10))))
{% endhighlight %}

While this pattern makes it simple to reason about asynchronous code, it is also pervasive. Any function that calls `f` will likely have to use the same pattern:

{% highlight python %}
async def g(db, query):
    data = await f(db, query)
    return data * 2
{% endhighlight %}

And in Clojure:

{% highlight clojure %}
(defn g [db, query]
  (go (let [data (<! (f db query))]
        (* data 2))))
{% endhighlight %}

Syntactically, this remains quite convenient. However in the case of Clojure, the overhead of using a `go` block is high. As the codebase grows, function calls start nesting, `go` blocks proliferate, and the execution becomes inefficient. It also discourages good coding practices, since factoring out logic in simple functions incurs a performance penalty.

Thankfully, this can be mostly avoided by introducing a slightly different flavor of `go` blocks (simply dubbed `async` blocks in this post).

# The async macro

First, let's look at the implementation of the `go` macro to understand where the overhead is coming from (no need to understand everything):

{% highlight clojure %}
(defmacro go
  [& body]
  `(let [c# (chan 1)
         captured-bindings# (clojure.lang.Var/getThreadBindingFrame)]
     (dispatch/run
      (fn []
        (let [f# ~(ioc/state-machine `(do ~@body) 1 (keys &env) ioc/async-custom-terminators)
              state# (-> (f#)
                         (ioc/aset-all! ioc/USER-START-IDX c#
                                        ioc/BINDINGS-IDX captured-bindings#))]
          (ioc/run-state-machine-wrapped state#))))
     c#))
{% endhighlight %}

Two key points:

* The body of the go block is dispatched to a thread pool via `dispatch/run`
* The result of the computation is passed to the current thread via a core.async channel created with `(chan 1)`

`dispatch/run` is used to run multiple asynchronous operations concurrently. In the following example, both calls to `f` are happening at the same time:

{% highlight clojure %}
(defn h [db query1 query2]
  (go (let [c1 (f db query1)
            c2 (f db query2)]
        (+ (<! c1) (<! c2)))))
{% endhighlight %}

However, let's have another look at the `g` function above:

{% highlight clojure %}
(defn g [db, query]
  (go (let [data (<! (f db query))]
        (* data 2))))
{% endhighlight %}

While `f` is running, `g` doesn't do anything concurrently. If we wanted to get rid of `f`, we could rewrite `g` as follows:

{% highlight clojure %}
(defn g [db, query]
  (go (let [data (<! (fetch db query))]
        (* (+ data 10) 2))))
{% endhighlight %}

Then instead of having `g` waiting on `f` waiting on the db fetch, we'd only have `g` waiting on the db fetch, and the scheduler would have one less lightweight thread to worry about. In other words, there is no point in dispatching `f`'s computation to a different thread - we should just keep running our logic in `g`'s thread.

To do this, let's slightly change the `go` macro:

{% highlight clojure %}
(defmacro go'
  [& body]
  `(let [c# (chan 1)
         captured-bindings# (clojure.lang.Var/getThreadBindingFrame)]
     (let [f# ~(ioc/state-machine `(do ~@body) 1 (keys &env) ioc/async-custom-terminators)
          state# (-> (f#)
                     (ioc/aset-all! ioc/USER-START-IDX c#
                                    ioc/BINDINGS-IDX captured-bindings#))]
       (ioc/run-state-machine-wrapped state#))
     c#))
{% endhighlight %}

The only thing we changed is that we run the computation directly in the calling thread instead of dispatching it.

Now that we're not using the dispatcher, we can optimize this even more. Since the body runs in the same thread, we don't need a core.async channel to communicate its result back to us. Instead, we can use a simple stateful thread-local variable. To avoid messing around too much with the core.async implementation, a quick and dirty way to do this is to provide implementations of core.async's `ReadPort`, `WritePort` and `Channel` protocols for the java `ThreadLocal` class:

{% highlight clojure %}
(require [clojure.core.async.impl.protocols :as impl])
(import [clojure.lang IDeref])

(extend-type ThreadLocal
  impl/ReadPort
  (take! [this _]
    (reify IDeref
      (deref [_] (.get this))))
  impl/WritePort
  (put! [this val _]
    (.set this val)
    (reify IDeref
      (deref [this] true)))
  impl/Channel
  (close! [this])
  (closed? [this] (.get this)))
{% endhighlight %}

Now we can use `ThreadLocal` instances like core.async channels, except that they don't provide any concurrency guarantees, only work with a single message, and don't support callbacks for the `take!` and `put!` methods. However, they are sufficient for our needs and considerably lighter, since they don't need queues for puts and takes, or an internal buffer.

Replacing `(chan 1)` with `ThreadLocal` completes our `async` macro:

{% highlight clojure %}
(defmacro async
  [& body]
  `(let [c# (ThreadLocal.)
         captured-bindings# (clojure.lang.Var/getThreadBindingFrame)]
     (let [f# ~(ioc/state-machine `(do ~@body) 1 (keys &env) ioc/async-custom-terminators)
          state# (-> (f#)
                     (ioc/aset-all! ioc/USER-START-IDX c#
                                    ioc/BINDINGS-IDX captured-bindings#))]
       (ioc/run-state-machine-wrapped state#))
     c#))
{% endhighlight %}

# Benchmark

To simulate deeply nested asynchronous function calls, we use a simple asynchronous function that calls itself recursively multiple times, terminating with a call to the "root" asynchronous operation `result`. In real-life, `result` could be our database fetch, but here we're only interested in measuring the core.async code overhead, so we just return a closed channel with one value.

{% highlight clojure %}
(defn result []
  (let [c (chan 1)]
    (put! c :done)
    (close! c)
    c))

(defn go-test [n]
  (go (if (> n 0)
        (<! (go-test (dec n)))
        (<! (result)))))

(defn async-test [n]
  (async (if (> n 0)
           (<! (async-test (dec n)))
           (<! (result)))))

(defn base-test [n]
  (if (> n 0)
    (base-test (dec n))
    (result)))
{% endhighlight %}

`go-test` uses core.async's `go` block and `async-test` is the same function but using `async` instead. For reference, the `base-test` function removes the asynchronous overhead completely by simply passing the result channel down the stack (in typical code, each nested function call may transform the result contained inside the channel, so this simplification is not possible).

Finally, we define the two following benchmark functions, to test sequential and parallel behaviors:

{% highlight clojure %}
(defn sequential [nesting-factor num-calls]
  (time (dotimes [_ num-calls] (<!! (go-test nesting-factor))))
  (time (dotimes [_ num-calls] (<!! (async-test nesting-factor))))
  (time (dotimes [_ num-calls] (<!! (base-test nesting-factor)))))

(defn parallel [nesting-factor num-calls]
  (time (dorun (pmap (fn [_] (<!! (go-test nesting-factor))) (range num-calls))))
  (time (dorun (pmap (fn [_] (<!! (async-test nesting-factor))) (range num-calls))))
  (time (dorun (pmap (fn [_] (<!! (base-test nesting-factor))) (range num-calls)))))
{% endhighlight %}

On my laptop, with clojure 1.7.0 and after some JVM warmup, I get the following results:

{% highlight clojure %}
=> (sequential 20 1e5)
"Elapsed time: 11933.357949 msecs"
"Elapsed time: 878.814996 msecs"
"Elapsed time: 42.607629 msecs"

=> (parallel 20 1e5)
"Elapsed time: 2711.289935 msecs"
"Elapsed time: 499.933361 msecs"
"Elapsed time: 314.250108 msecs"
{% endhighlight %}

The code using the `async` macro is more than 10x faster than the equivalent code using `go` in the sequential case, and 5x faster in the parallel case. We shouldn't take these factors too literally since they depend heavily on the level of nesting of the code, but the performance win is clearly significant. Note that the non-async version remains much faster, which is expected.

# Should I just drop go entirely and use async instead?

`go` exists for a good reason. You should probably keep using it if your function is running CPU-intensive tasks in the calling thread (or doing IO, but you should never block on IO in a core.async thread).

Imagine we modify `f` slightly:

{% highlight clojure %}
(defn f [db query]
  (async (let [data (<! (fetch db query))]
           (expensive-cpu-intensive-task data))))
{% endhighlight %}

Here we use `async` instead of `go`, and also do some expensive computation to transform the fetched data. Now if we call the function `h` defined previously, both calls to `f` cause the expensive computation to be run in `h`'s thread, so no parallelism is possible. On the other hand, if we had used the original version of `go`, the dispatcher would have been able to schedule each call to `f` on a different thread.

However, in real-life asynchronous code, most functions only exist to make the code better organized, and concurrent execution is only needed for a small number of specific operations (database fetches, network calls, etc). As long as these operations are not run in the calling thread, everything is fine. For example, using `async` in `f` still allows the two database fetches to happen concurrently when we call `h`, because there is one more indirection between the call to `f` and the actual database operation.

# Last word

The suggested `async` macro is designed to be as simple and loosely coupled with the core.async implementation as possible. Including `async` as part of the core.async code may enable further optimizations.


[core-async]:  https://github.com/clojure/core.async
[c-sharp]:     https://msdn.microsoft.com/en-us/library/hh191443.aspx
[hack]:        http://docs.hhvm.com/manual/en/hack.async.php
[python]:      https://www.python.org/dev/peps/pep-0492/

