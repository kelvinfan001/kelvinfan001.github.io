---
title: Concurrent Applications with Actix
date: 2021-04-23
tags: []
description: Experiments in asynchronous Rust with Actix and the actor model.
tag:
  - async
  - Rust
  - actors
  - Actix
---

<img width="auto" src="/assets/img/actix/logo-large.png" alt="Actix Logo"/>

# "OS-scars" awarded by actors

Let me start by thanking the [Zincati update agent](https://coreos.github.io/zincati/) --- you the real MVP. When I first started working on the automatic updates agent for the Fedora CoreOS operating system, I was in the middle of learning a whole new programming language; on top of that, I had to brush up on my asynchronous programming knowledge, get familiar with a new conceptual model of concurrent computation, and learn a whole new Rust framework.

Learning asynchronous Rust, the actor model, and Actix all at once left me scars, scars of knowledge that I will be able to recall forever... until I forget. So yeah, let me try to write some of this down in a blog post while it's still fresh in my head.

# A short-ish, grossly simplified summary of asynchronous Rust

### OS Threads

In a basic operating systems course, we at least learn about OS threads. While they definitely have less overhead than full on processes, there's actually still quite a bit of overhead in this model of concurrency. This mainly stems from the fact that the kernel can interrupt at any time.

While this makes it slightly easier to program, at every context switch, the operating system needs to save the value of all CPU registers and the entire call stack of the thread since it has no idea about the progress of the task running in the thread; for example, the interrupt could have happened in the middle of a huge recursive computation.

### Cooperative Multitasking

Imagine if we can context switch only when convenient, such as when we're waiting for something anyway, and only save exactly what we need to. This, however, requires tasks to cooperate and yield to other tasks voluntarily, since we don't rely on the kernel to police the tasks and force tasks to yield.

If we can allow cooperation, then we can achieve much more efficient context switches and save a bunch more memory.

### Back to the `Future`s

One way to achieve this is by using futures. Basically, a future is a task that might not be fully finished yet. In some other languages, like JavaScript, they're called Promises. In Rust, they are represented by the `Future` trait. Any struct can implement it. The most crucial component is the `poll` method that returns a `Poll` enum that can be of the two variants `Ready(T)` and `Pending`. With this `poll` method, a task can indicate whether they want to yield and give up the CPU, i.e. it yields if the task returns `Pending` when polled. Additionally, a struct that implements the `Future` trait stores only the minimum amount of information it needs to store in between `poll` calls.

Under the hood, `Future`s in Rust are finite state machines generated by the compiler from `async/await` or "future combinators" (e.g. `.then()`). It's actually super interesting how state machines can perfectly implement `Future`s efficiently, but I really want to talk about Actix, so I'll take a shortcut and just provide a link [here](https://os.phil-opp.com/async-await/#state-machine-transformation).

So now we have a mechanism for tasks to store (just enough) information about their progress and signal their willingness to yield at convenient times, we need something to run the futures, call `poll` on them, schedule them, etc.

### The executor
An executor is essentially a scheduler. In Rust, the Tokio crate is commonly used --- it contains the runtime components necessary for building an asynchronous application. One major component is a scheduler for asynchronous "tasks".

The scheduler is responsible for actually executing the futures that get spawned as "tasks" onto it. In Tokio, tasks are green threads, or user level threads, as they are completely managed by the Tokio library in userland and the kernel is unaware of such threads. The scheduler is in charge of making sure these tasks run concurrently and potentially on multiple OS threads.

In order to execute a future, we can "spawn" a task to run that future and submit it to the scheduler, which makes sure that the task executes _when it has work to do_. In Tokio, tasks only require a single allocation and 64 bytes of memory. As we can see, we can make huge savings over OS threads. With a scehduler, we can spawn multiple tasks to run multiple futures concurrently. We have achieved asynchrony!

# Actix

Asynchrony is perfect for when we have an application that needs to handle multiple somewhat indepedent tasks (in the sense of "job", not a Tokio "task") at the same time. The actor model "is a conceptual model to deal with concurrent computation". Brian Storti wrote a great [blog post](https://www.brianstorti.com/the-actor-model/) that summarizes the actor model very well. [Actix](https://docs.rs/actix/0.11.1/actix/) is an asynchronous Rust framework built for the actor model.

For example, Zincati (at the time of writing) has four major things to do, all of which should conceptually run at the same time. These four major tasks can run on four separate "actors". It has an "update agent" actor containing the core logic of the agent and manages the agent's finite state machine, an "[RPM-OSTree](https://coreos.github.io/rpm-ostree/#rpm-ostree-a-true-hybrid-imagepackage-system) client" actor responsible for communicating with the RPM-OSTree daemon, a metrics service for serving client requests on the local metrics endpoint, and a D-Bus server for inter-process communication. 

As we can see, Zincati works very well with Actix.

### Single actor running in the Actix system

Getting started with Actix is actually not too bad. Let's start by building an actor. (Also import some crates that we'll be using later)

{% highlight rust %}
use actix::prelude::*;
use anyhow::Result;
use futures::prelude::*;
use tokio::time::{sleep, Duration};

struct MyActor {}

impl Actor for MyActor {
    type Context = Context<Self>;
}
{% endhighlight %}

We simply need to implement actix's `Actor` trait for a struct. Notably, we need to define a `Context` that defines its actor's execution state. Here, we've defined the actor's context to be `Context<A>`, which is an _asynchronous_ context (some actors could run in synchronous contexts). We'll talk more about execution contexts in the next bit. First, let's find a way to somehow run our actor.

{% highlight rust %}
fn main() -> Result<()> {
    let sys = actix::System::new();

    sys.block_on(async {
        let _addr = MyActor {}.start();
    });
    sys.run()?;

    Ok(())
}
{% endhighlight %}

Here, we've created an Actix "system". This system is built on top of Tokio and creates an asynchronous execution context for our actors. Here, the system runs on a single thread. We start `MyActor` simply by calling `.start()` on it, and then we run our system. Our single actor is now running on our system. With `sys.run()`, the system will be in charge of executing the futures that are spawned by actors.

Note: we're not using the address returned by `.start()`. Normally, we can use this address to send different actors messages from different actors, but here, we're only going to have the actors send messages to themselves.

Now let's get our actor to do stuff. Remember that actors respond to messages in order to do stuff, so let's create a `Message`.

{% highlight rust %}
struct SmallTask {}

impl Message for SmallTask {
    type Result = ();
}
{% endhighlight %}

Just like our actor, all we needed to do was implement the `Message` trait for our `SmallTask` struct. We've defined the expected result of handling this message to be the empty tuple. Next, let's teach our actor to handle `SmallTask` messages.

{% highlight rust %}
impl Handler<SmallTask> for MyActor {
    type Result = ();

    fn handle(&mut self, _msg: SmallTask, ctx: &mut Context<Self>) -> Self::Result {
        let fut = Box::pin(async {
            println!("Easy task done!");
        });

        // Convert a normal future to an `ActorFuture`.
        // This part is not too important, just know that `ctx.spawn()`
        // has to take in an `ActorFuture`.
        let actor_future = fut.into_actor(self);

        // Spawns a future into the context.
        ctx.spawn(actor_future); 
    }
}
{% endhighlight %}

Let's look at what's happening here. Actors in Actix can add futures to its execution context. When the `MyActor` receives a `SmallTask` message, it'll handle it by creating a future. This future immediately resolves since all it does is print a line. Then, it'll spawn a task into _`MyActor`'s own execution context_ to run that future.

Let's get things going by getting `MyActor` to send itself a `SmallTask` message once it is started.

{% highlight rust %}
impl Actor for MyActor {
    type Context = Context<Self>;

    // Called when an actor gets polled the first time.
    fn started(&mut self, ctx: &mut Self::Context) {
        // Sends the message to self.
        ctx.notify(SmallTask {});
    }
}
{% endhighlight %}

Our entire program so far:

{% highlight rust %}
use actix::prelude::*;
use anyhow::Result;
use futures::prelude::*;
use tokio::time::{sleep, Duration};

struct MyActor {}

impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        ctx.notify(SmallTask {});
    }
}

struct SmallTask {}

impl Message for SmallTask {
    type Result = ();
}

impl Handler<SmallTask> for MyActor {
    type Result = ();

    fn handle(&mut self, _msg: SmallTask, ctx: &mut Context<Self>) -> Self::Result {
        let fut = Box::pin(async {
            println!("Easy task done!");
        });

        let actor_future = fut.into_actor(self); // Convert a normal future to an `ActorFuture`

        // Spawns a future into the context.
        ctx.spawn(actor_future);
    }
}

fn main() -> Result<()> {
    let sys = actix::System::new();

    sys.block_on(async {
        let _addr = MyActor {}.start();
    });
    sys.run()?;

    Ok(())
}
{% endhighlight %}

If we run this, we'll get:

```
$ cargo run
Easy task done!
```

### Single actor handling messages asynchronously

To demonstrate Actix's asynchronous powers, we'll have to do something slightly more interesting than that. Let's create a message that needs to be handled by running a slightly more complex future. This time, we'll simulate some work that needs to wait a bit, perhaps we're fetching something over the internet or doing some IO task.

{% highlight rust %}
struct ComplexTask {}

impl Message for ComplexTask {
    type Result = ();
}

impl Handler<ComplexTask> for MyActor {
    type Result = ();

    fn handle(&mut self, _msg: ComplexTask, ctx: &mut Context<Self>) -> Self::Result {
        let fut = Box::pin({
            sleep(Duration::from_secs(3)).then(|_| async {
                println!("Complex task done!");
            })
        });

        let actor_fut = fut.into_actor(self);

        ctx.spawn(actor_fut);
    }
}
{% endhighlight %}

We first create a future that sleeps for 3 seconds, simulating some work where we can wait, then we chain on another future that just prints a line using the future combinator `.then()`. We also could have used the newer `async/await` syntax. Either way, the compiler knows to convert our nice human-readable syntax into a FSM that implements `Future`.

Let's get `MyActor` to send itself an `ComplexTask` message before it sends itself a `SmallTask` message. 

{% highlight rust %}
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        ctx.notify(ComplexTask {}); // `ComplexTask` FIRST!
        ctx.notify(SmallTask {});
    }
}
{% endhighlight %}

If we run this:

```
$ cargo run
Easy task done!
<3 seconds passes...>
Complex task done!
```

Lo and behold! Even though `MyActor` sent itself an `ComplexTask` first, the `SmallTask` was completed first. Our system's execution context asynchronously ran the two futures to completion. While `ComplexTask` was doing its waiting, it yielded the CPU back to the executor, and so it was able to execute our small task first.

### Single actor handling messages "synchronously" internally.

Sometimes, we might want our actors to handle messages _to completion_ sequentially. Actix lets us do that:

{% highlight rust %}
impl Handler<ComplexTask> for MyActor {
    type Result = ();

    fn handle(&mut self, _msg: ComplexTask, ctx: &mut Context<Self>) -> Self::Result {
        let fut = Box::pin({
            sleep(Duration::from_secs(3)).then(|_| async {
                println!("Complex task done!");
            })
        });

        let actor_fut = fut.into_actor(self);

        // Like `spawn`, this spawns into the context a task to run the future,
        // but ALSO waits for it to resolve.
        // This stops the actor from processing any incoming events until the future resolves.
        ctx.wait(actor_fut);
    }
}

impl Handler<SmallTask> for MyActor {
    type Result = ();

    fn handle(&mut self, _msg: SmallTask, ctx: &mut Context<Self>) -> Self::Result {
        let fut = Box::pin(async {
            println!("Easy task done!");
        });

        let actor_future = fut.into_actor(self);

        // Also use `wait` here.
        ctx.wait(actor_future);
    }
}
{% endhighlight %}

Using the `wait()` method of `MyActor`'s own execution context, we can tell `MyActor` to wait until the future is resolved before doing anything else.

```
$ cargo run
<3 seconds passes...>
Complex task done!
Easy task done!
```

The above example isn't anything too amazing on its own, but consider multiple actors...

### Two actors handling messages "synchronously" only in their own context

What we did with `ctx.wait()` in the above example is sort of like using `.await`; we're only waiting within the actor. In other words, by telling `MyActor`'s _own_ execution context to `wait()`, we don't block the entire thread; instead, `MyActor` yields the CPU whenever it can while the future is resolving. Mutliple actors on the system will still run concurrently. Let's verify that this is true.

Let's do this by introducing another actor, `MyOtherActor`, to send itself a `SmallTask` message once started, and let `MyActor` only send itself an `ComplexTask` message once started.

{% highlight rust %}
struct MyOtherActor {}

impl Actor for MyOtherActor {
    type Context = Context<Self>;

    // Called when an actor gets polled the first time.
    fn started(&mut self, ctx: &mut Self::Context) {
        ctx.notify(SmallTask {});
    }
}

impl Handler<SmallTask> for MyOtherActor {
    type Result = ();

    fn handle(&mut self, _msg: SmallTask, ctx: &mut Context<Self>) -> Self::Result {
        let fut = Box::pin(async {
            println!("Easy task done!");
        });

        let actor_future = fut.into_actor(self);

        // Still using `wait` here.
        ctx.wait(actor_future);
    }
}
{% endhighlight %}

Modify our `main()` function to start `MyOtherActor`:

{% highlight rust %}
fn main() -> Result<()> {
    let sys = actix::System::new();

    sys.block_on(async {
        let _my_actor_addr = MyActor {}.start();
        sleep(Duration::from_secs(2)).await; // Ensure that `MyOtherActor` is started AFTER
        let _my_other_actor_addr = MyOtherActor {}.start();
    });
    sys.run()?;

    Ok(())
}
{% endhighlight %}

Note that we wait two seconds before starting `MyOtherActor` to ensure that `MyOtherActor` (which will handle `SmallTask`) will be started _after_ `MyActor`.

Have `MyActor` send itself the `ComplexTask`, only:

{% highlight rust %}
impl Actor for MyActor {
    type Context = Context<Self>;

    // Called when an actor gets polled the first time.
    fn started(&mut self, ctx: &mut Self::Context) {
        // Sends the message to self.
        ctx.notify(ComplexTask {});
    }
}
{% endhighlight %}

Remember, we are still using `ctx.wait()` in the `handle()` methods of both messages, but when we run this:

```
$ cargo run
<2 seconds passes...>
Easy task done!
<1 second passes...>
Complex task done!
```

Even though `MyActor` was started 2 seconds earlier, `SmallTask` (which was handled by `MyOtherActor`) completed first! The actors in an Actix system truly do run concurrently.

This is awesome, no matter what we do in each actor in the system, as long as the work they do are reasonably asynchronous (we have `.await`s/future combinators here and there), we can be sure that many actors on the system will be running concurrently.

### What about CPU-bound synchronous workloads?

Wait... but what if we have actors whose work can only be done synchronously? Not all work can be interrupted at convenient places, for example, an actor might need to do some heavy computation for a long time that really cannot be interrupted at any point. Another example, using Zincati, is its D-Bus server. Because the Rust implementation of D-Bus that we're using in Zincati doesn't support async, Zincati's D-Bus server handles requests in a blocking loop.

If we let a regular "async" actor do this type of synchronous, CPU-bound work on the system's thread, it'll block the system's event loop, and that would be disastrous. Let's see the disaster unfold by modifying `ComplexTask` to simulate some synchronous, intensive work by calculating the 40th fibonnaci number.

{% highlight rust %}
impl Handler<ComplexTask> for MyActor {
    type Result = ();

    fn handle(&mut self, _msg: ComplexTask, _ctx: &mut Context<Self>) -> Self::Result {
        fibonacci(40);
        println!("Complex task done!");
    }
}

fn fibonacci(n: u32) -> u32 {
    match n {
        0 => 1,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}
{% endhighlight %}

Also comment out the bit in `main()` where we wait 2 seconds before starting `MyOtherActor`, so we start `MyOtherActor` as early as possible, but still after `MyActor`.

{% highlight rust %}
sys.block_on(async {
    let _my_actor_addr = MyActor {}.start();
    // sleep(Duration::from_secs(2)).await; // Ensure that `MyOtherActor` is started AFTER
    let _my_other_actor_addr = MyOtherActor {}.start();
});
{% endhighlight %}

```
$ cargo run
<wait a good amount of time, depending on how beefy your computer is>
Complex task done!
Easy task done!
```

We see that now, `SmallTask` _always_ finishes only after `ComplexTask` is done. Essentially, while computing our Fibonacci number, we never yielded the CPU, so our whole system is blocked on `MyActor` trying to handle our `ComplexTask`.

### `SyncArbiter` to the rescue

Luckily, Actix allows us to offload actors that perform such blocking or intensive synchronous tasks to other OS threads, via `SyncArbiter`s. This way, these dirty synchronous actors don't run on the system's thread and pollute our nice asynchronous system. We essentially outsource the context switching to the kernel for heavy synchronous work. In other words, since such synchronous work don't have convenient times to get interrupted anyway, we'll just leave it up to the OS to interrupt them whenever it wants. An added bonus is that if your computer as multiple cores, our original system can potentially run in parallel with these heavy synchronous tasks.

The syntax for starting actors on `SyncArbiter`s is slightly different, and since this post is already getting much longer than I expected, I'll just leave some links. Zincati has some [good examples](https://github.com/coreos/zincati/blob/master/src/cli/agent.rs), though. If you're interested, check out how the `RpmOstreeClient` and `DBusService` actors are started in Zincati.

# Conclusion

As we saw, Actix can help us create concurrent applications under the actor model, making use of both green threads and OS threads.

# References
- https://os.phil-opp.com/async-await/#implementation
- https://rust-lang.github.io/async-book/
- https://tokio.rs/tokio/tutorial/
- https://actix.rs/book/actix/sec-0-quick-start.html
- https://www.reddit.com/r/rust/comments/frxg9t/is_async_just_sugar_for_threading_in_certain_cases/
