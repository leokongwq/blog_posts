---
layout: post
comments: true
title: Fibers-and-Threads-in-node
date: 2016-10-17 11:26:39
tags:
- node
categories:
- nodejs
---

I like node.js, and I’m not the only one, obviously! I like it primarily for two things: it is simple and it is very fast. I already said it many times but one more won’t hurt.Before working with node, I had spent many years working with threaded application servers. This was fun sometimes but it was also often frustrating: so many APIs to learn, so much code to write, so many risks with critical sections and deadlocks, such a waste of costly system resources (stacks, locks), etc. Node came as a breath of fresh air: a simple event loop and callbacks. You can do a lot with so little, and it really flies!But it does not look like we managed to eradicate threads. They keep coming back. At the beginning of last year Marcel Laverdet opened Pandora’s box by releasing node-fibers: his threads are a little greener than our old ones but they have some similarities with them. And this week the box got wide open as Jorge Chamorro Bieling released threads_a_gogo, an implementation of real threads for node.js.Isn’t that awful? We were perfectly happy with the event loop and callbacks, and now we have to deal with threads and all their complexities again. Why on earth? Can’t we stop the thread cancer before it kills us!Well. First, things aren’t so bad because fibers and threads did not make it into node’s core. The core is still relying only on the event loop and callbacks. And it is probably better this way.And then maybe we need to overcome our natural aversion for threads and their complexities. Maybe these new threads aren’t so complex after all. And maybe they solve real problems. This is what I’m going to explore in this post.

<!-- more -->

### Threads and Fibers

The main difference between fibers and real threads is on the scheduling side: threads use implicit, preemptive scheduling while fibers use explicit, non-preemptive scheduling. This means that threaded code may be interrupted at any point, even in the middle of evaluating an expression, to give CPU cycles to code running in another thread. With fibers, these interruptions and context switches don’t happen randomly; they are into the hands of the programmer who decides where his code is going to yield and give CPU cycles to other fibers.

The big advantage of fiber’s explicit yielding is that the programmer does not need to protect critical code sections as long as they don’t yield. Any piece of code that does not yield cannot be interrupted by other fibers. This means a lot less synchronization overhead.

But there is a flip side to the coin: threads are fair; fibers are not. If a fiber runs a long computation without yielding, it prevents other fibers from getting CPU cycles. A phenomenon known as starvation, and which is not new in node.js: it is inherent to node’s event loop model; if a callback starts a long computation, it blocks the event loop and prevents other events from getting their chance to run.

Also, threads take advantage of multiple cores. If four threads compete for CPU on a quad-core processor, each thread gets 100% (or close) of a core. With fibers there is no real parallelism; at one point in time, there is only one fiber that runs on one of the cores and the other fibers only get a chance to run at the next yielding point.

Fibers – What for?

So, it looks like fibers don’t bring much to the plate. They don’t allow node modules to take advantage of multiple cores and they have the same starvation/fairness issues as the basic event loop. What’s the deal then?

Fibers were introduced and are getting some love primarily because they solve one of node’s big programming pain points: the so called callback pyramid of doom. The problem is best demonstrated by an example:

    function archiveOrders(date, cb) {
  db.connect(function(err, conn) {
    if (err) return cb(err);
    conn.query("selectom orders where date < ?",  
               [date], function(err, orders) {
      if (err) return cb(err);
      helper.each(orders, function(order, next) {
        conn.execute("insert into archivedOrders ...", 
                     [order.id, ...], function(err) {
          if (err) return cb(err);
          conn.execute("delete from orders where id=?", 
                       [order.id], function(err) {
            if (err) return cb(err);
            next();
          });
        });
      }, function() {
        console.log("orders been archived");
        cb();
      });
    });
  });
}

This is a very simple piece of business logic but we already see the pyramid forming. Also, the code is polluted by lots of callback noise. And things get worse as the business logic gets more complex, with more tests and loops.

Fibers, with Marcel’s futures library, let you rewrite this code as:

var archiveOrders = (function(date) {
  var conn = db.connect().wait();
  conn.query("selectom orders where date < ?",  
             [date]).wait().forEach(function(order) {
    conn.execute("insert into archivedOrders ...", 
                 [order.id, ...]).wait();
    conn.execute("delete from orders where id=?", 
                 [order.id]).wait();
  });
  console.log("orders been archived");
}).future();
The callback pyramid is gone; the signal to noise ratio is higher, asynchronous calls can be chained (for example query(...).wait().forEach(...)), etc. And things don’t get worse when the business logic gets more complex. You just write normal code with the usual control flow keywords (if, while, etc.) and built-in functions (forEach). You can even use classical try/catch exception handling and you get complete and meaningful stack traces.

Less code. Easier to read. Easier to modify. Easier to debug. Fibers clearly give the programmer a better comfort zone.

Fibers make this possible because they solve a tricky topological problem with callbacks. I’ll try to explain this problem on a very simple example:

db.connect(function(err, conn) {
  if (err) return cb(err);
  // conn is available in this block
  doSomething(conn);
});
// Would be nice to be able to assign conn to a variable 
// in this scope so that we could resume execution here 
// rather than in the block above.
// But, unfortunately, this is impossible, at least if we 
// stick to vanilla JS (without fibers).
The topological issue is that the conn value is only accessible in the callback scope. If we could transfer it to the outer scope, we could continue execution at the top level and avoid the pyramid of doom. Naively we would like to do the following:

var c;
db.connect(function(err, conn) {
  if (err) return cb(err);
  c = conn;
});
// conn is now in c (???)
doSomething(c);
But it does not work because the callback is invoked asynchronously. So c is still undefined when execution reaches doSomething(c). The c variable gets assigned much later, when the asynchronous connect completes.

Fibers make this possible, though, because they provide a yield function that allows the code to wait for an answer from the callback. The code becomes:

var fiber = Fiber.current;
db.connect(function(err, conn) {
  if (err) return fiber.throwInto(err);
  fiber.run(conn);
});
// Next line will yield until fiber.throwInto 
// or fiber.run are called
var c = Fiber.yield();
// If fiber.throwInto was called we don't reach this point 
// because the previous line throws.
// So we only get here if fiber.run was called and then 
// c receives the conn value.
doSomething(c);
// Problem solved! 
Things are slightly more complex in real life because you also need to create a Fiber to make it work.

But the key point is that the yield/run/throwInto combination makes it possible to transfer the conn value from the inner scope to the outer scope, which was impossible before.

Here, I dived into the low level fiber primitives. I don’t want this to be taken as an encouragement to write code with these primitives because this can be very error prone. On the other hand, Marcel’s futures library provides the right level of abstraction and safety.

And, to be complete, it would be unfair to say that fibers solve just this problem. They also enable powerful programming abstractions like generators. But my sense is that the main reason why they get so much attention in node.js is because they provide a very elegant and efficient solution to the pyramid of doom problem.

Sponsored Ad

The pyramid of doom problem can be solved in a different way, by applying a CPS transformation to the code. This is what my own tool, streamline.js, does. It leads to code which is very similar to what you’d write with fiber’s futures library:

function archiveOrders(date, _) {
  var conn = db.connect(_);
  flows.each(_, conn.query("selectom orders where date < ?",  
                           [date], _), function(_, order) {
    conn.execute("insert into archivedOrders ...", 
                 [order.id, ...], _);
    conn.execute("delete from orders where id=?", 
                 [order.id], _);
  });
  console.log("orders been archived");
}
The signal to noise ratio is even slightly better as the wait() and future() calls have been eliminated.

And streamline gives you the choice between transforming the code into pure callback code, or into code that takes advantage of the node-fibers library. If you choose the second option, the transformation is much simpler and preserves line numbers. And the best part is that I did not even have to write the fibers transformation, Marcel offered it on a silver plate.

Wrapping up on fibers

In summary, fibers don’t really change the execution model of node.js. Execution is still single-threaded and the scheduling of fibers is non-preemptive, just like the scheduling of events and callbacks in node’s event loop. Fibers don’t really bring much help with fairness/starvation issues caused by CPU intensive tasks either.

But, on the other hand, fibers solve the callback pyramid of doom problem and can provide a great relief to developers, especially those who have thick layers of logic to write.

Threads – What for?

As I said in the intro, threads landed into node this week, with Jorge’s thread_a_gogo implementation (and I had a head start on them because Jorge asked me to help with beta testing and packaging). What do they bring to the plate? And this time we are talking about real threads, not the green kind. Shouldn’t we be concerned that these threads will drag us into the classical threading issues that we had avoided so far?

Well, the answer is loud and clear: there is nothing to be worried about! These threads aren’t disruptive in any way. They won’t create havoc in what we have. But they will fill an important gap, as they will allow us to handle CPU intensive operations very cleanly and efficiently in node. In short, all we get here is bonus!

Sounds too good to be true! Why would these threads be so good when we had so many issues with threads before? The answer is simple: because we had the wrong culprit! The problems that we had were not due to the threads themselves, they were due to the fact that we had SHARED MUTABLE STATE!

When you are programming with threads in Java or .NET or other similar environments, any object which is directly or indirectly accessible from a global variable, or from a reference that you pass from one thread to another, is shared by several threads. If this object is immutable, there is no real problem because no thread can alter it. But if the object is mutable, you have to introduce synchronization to ensure that the object’s state is changed and read in a disciplined way. If you don’t, some thread may access the object in an inconsistent state because another thread was interrupted in the middle of a modification on the object. And then things usually get really bad: incorrect values, crashes because data structures are corrupted, etc.

If you have shared mutable state, you need synchronization. And synchronization is a difficult and risky art. If your locks are too coarse you get very low throughput because your threads spend most of their time waiting on locks. If they are too granular, you run the risk of missing some edge cases in your locking strategy or of letting deadlocks creep in. And, even if you get your synchronization right, you pay a price for it because locks are not free and don’t scale well.

But threads a gogo (I’ll call them TAGG from now on) don’t share mutable state. Each thread runs in its own isolate, which means that it has its own copy of the Javascript code, its own global variables, its own heap and stack. Also, the API does not let you pass a reference to a mutable Javascript object from one thread to another. You can only pass strings (which are immutable in Javascript) (*). So you are on the safe side, you don’t run the risk of having one thread modify something that another thread is accessing at the same time. And you don’t need synchronization, at least not the kind you needed around shared mutable objects.

(*) it would be nice to be able to share frozen objects across threads. This is not available in the first version of TAGG but this may become possible in the future. TAGG may also support passing buffers across thread boundaries at some point (note that this may introduce a limited, but acceptable, form of shared state).

I hope that I have reassured the skeptics at this point. As Jorge puts it, these threads aren’t evil. And actually, they solve an important problem which was dramatized in a blog post a few months ago: node breaks on CPU intensive tasks. The blog post that I’m referring to was really trashy and derogative and it was making a huge fuss about a problem that most node applications won’t have. But it cannot be dismissed completely: some applications need to make expensive computations, and, without threads, node does not handle this well, to say the least, because any long running computation blocks the event loop. This is where TAGG comes to the rescue.

If you have a function that uses a lot of CPU, TAGG lets you create a worker thread and load your function into it. The API is straightforwards:

var TAGG = require('threads_a_gogo');

// our CPU intensive function
function fibo(n) { 
  return n > 1 ? fibo(n - 1) + fibo(n - 2) : 1;
}

// create a worker thread
var t = TAGG.create();
// load our function into the worker thread
t.eval(fibo);
Once you have loaded your function, you can call it. Here also the API is simple:

t.eval("fibo(30)", function(err, result) {
  console.log("fibo(30 result);
});
The function is executed in a separate thread, running in its own isolate. It runs in parallel with the main thread. So, if you have more than one core the computation will run at full speed in a spare core, without any impact on the main thread, which will continue to dispatch and process events at full speed.

When the function completes, its result is transferred to the main thread and dispatched to the callback of the t.eval call. So, from the main thread, the fibo computation behaves like an ordinary asynchronous operation: it is initiated by the t.eval call and the result comes back through a callback.

Often you’ll have several requests that need expensive computations. So TAGG comes with a simple pool API that lets you allocate several threads and dispatch requests to the first available one. For example:

var pool = TAGG.createPool(16);
// load the function in all 16 threads
pool.all.eval(fibo);
// dispatch the request to one of the threads
pool.any.eval("fibo(30unction(err, result) {
  console.log("fibo(30 result);
});
TAGG also provides support for events. You can exchange events in both directions between the main thread and worker threads. And, as you probably guessed at this point, the API is naturally aligned on node’s Emitter API. I won’t give more details but the TAGG module contains several examples.

A slight word of caution though: this is a first release so TAGG may lack a few usability features. The one that comes first to mind is a module system to make it easy to load complex functions with code split in several source files. And there are still a lot of topics to explore, like passing frozen objects or buffers. But the implementation is very clean, very simple and performance is awesome.

Wrapping up on threads

Of course, I’m a bit biased because Jorge involved me in the TAGG project before the release. But I find TAGG really exciting. It removes one of node’s main limitations, its inability to deal with intensive computations. And it does it with a very simple API which is completely aligned on node’s fundamentals.

Actually, threads are not completely new to node and you could already write addons that delegate complex functions to threads, but you had to do it in C/C++. Now, you can do it in Javascript. A very different proposition for people like me who invested a lot on Javascript recently, and not much on C/C++.

The problem could also be solved by delegating long computations to child processes but this is costlier and slower.

From a more academic standpoint, TAGG brings a first bit of Erlang’s concurrency model, based on share nothing threads and message passing, into node. An excellent move.

Putting it all together

I thought that I was going to write a short post, for once, but it looks like I got overboard, as usual. So I’ll quickly recap by saying that fibers and threads are different beasts and play different roles in node.

Fibers introduce powerful programming abstractions like generators and fix the callback pyramid problem. They address a usability issue.

Threads, on the other hand, fix a hole in node’s story, its inability to deal with CPU intensive operations, without having to dive into C/C++. They address a performance issue.

And the two blend well together (and — sponsored ad — they also blend with streamline.js), as this last example shows:

var pool = TAGG.createPool(16);
pool.all.eval(fibo);
console.log("fibo(30 pool.any.eval("fibo(30));
Kudos to Marcel and Jorge for making these amazing technologies available to the community.
                    
                    