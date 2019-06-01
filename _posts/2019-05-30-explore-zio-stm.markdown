---
layout: post
author: freskog
title: Exploring the STM functionality in ZIO
tags: ZIO STM
---

## Introduction

If you've been following the Scala community lately, you've definitely seen [ZIO][zio-github] getting more and more attention.
ZIO bills itself as:

>ZIO — A type-safe, composable library for asynchronous and concurrent programming in Scala

This post is an attempt at summarizing my learnings of using the STM (Software Transactional Memory) feature in ZIO.

STM is a tool for writing concurrent programs. It does this by bringing the idea of transactions into concurrent programming. Full disclaimer
here, I'm certainly no expert on this topic so please be gentle brave keyboard warriors. 

To get an idea of how to use the STM functionality, I decided to implement a use case around partitioning workloads. Let's say you
have a queue of incoming payloads, each payload needs to be processed using some user provided function. The processing must be
done sequentially for all payloads that belong to the same partition, but two payloads belonging to different partitions can be
processed concurrently.

I'm going to assume some pre-requisite knowledge in order to keep the length of this text manageable. Specifically, I'll assume
that you are familiar with the standard Scala Future and also Cats IO or Scalaz Task.

## Requirements

Rather than trying (and probably failing) to come up with a real world use case to justify all the requirements, I'm going
to just unceremoniously list them here

* All payloads belonging to the same partition are processed sequentially, by a user provided function
* In case of a defect/configurable timeout in the user function, we want to log the cause and resume processing payloads
* If a partition is idle for a configurable period a timeout is triggered, and  all resources allocated to it must be freed
* We need to have a mechanism to prevent one partition from stealing all the available capacity (limit max pending work)
* All timeouts and max capacity per partition must be configurable by the caller

Combining all of those with a bit of trial and error led me to the following design

{% highlight scala %}

type PartId = String

case class Config(userTTL:Duration, idleTTL:Duration, maxPending:Int)

def partition[A](config: Config, partIdOf: A => PartId, action: A => UIO[Unit]): ZIO[Any, Nothing, A => UIO[Boolean]]

{% endhighlight %}


This function is what the end user will interact with, so we'll break it down into its parts.

| Name     | Type                                                | Meaning                                                          |
| -------- | --------------------------------------------------- | ---------------------------------------------------------------- |
|          | A                                                   | The type of the incoming payloads                                |
|          | PartId                                              | A type alias for String, representing a partition id             |
|          | UIO[_]                                              | Unfallible IO, an effect type that has no expected failure modes |
| config   | Config                                              | User provided configuration for timeouts & back pressure         |
| partIdOf | A => PartId                                         | Function that determines what partition a payload belongs to     |
| action   | A => UIO[Unit]                                      | Function that performs an unfallible effect and returns unit     |
|          | ZIO[Any, Nothing, A => UIO[Boolean]]                | The return type, a producer function wrapped inside of an effect |

I think most of the types involved here are fairly straight forward, except for the return type. It might be worth taking a moment
to try to understand it better. It consists of three parts, the resouce part, error part and the value part.

**Any** is the resource part, if you've ever used a DI framework, this is the ZIO
equivalent. We can see that this function doesn't have any external dependencies. If it had said **Console with Clock**, then
we would need to *provide* an environment that contains the implementation for both of these services before running the effect.
I won't explain in detail how this works, but it's essentially a baked in reader monad. If that means nothing to you, then
think of it as having easy access to your Spring Context inside of the effect, and to run the effect you need to call
it with its Spring Context.

**Nothing**, just by looking at this return type it's clear that this effect has no expected failures. That doesn't mean it
won't have defects though. In ZIO there's a big distinction between failures we expect to handle, and defects that we either
didn't anticipate, or failures that we can't reasonably handle (i.e., database was eaten by gnomes). I generally try to handle
failures close to where they occur (perhaps attempt a retry, try an alternative strategy etc), and to let defects propagate
to a common point in the system, where all defects that happened as part of processing are logged together.

**A => UIO[Boolean]**, this is the function the user must call to insert work into the system, aka the producer function, which
when called tries to accept a payload of type A for processing. If we've hit the maximum of pending work for this partition, the
function will return false (wrapped inside of UIO[_], since the act of accepting work for processing is an effect).

#### Why do we need to have two layers of effects?

The effect layers correspond to two different actions. The outer most layer is saying that it will generate a producer function
in a manner that requires an effect. The inner layer is say that the generated producer function is itself effectful. Being
effectful here means that they describe side-effects like using mutable state and spawn new fibers.

If you want to learn more about managing mutable state using an effect system, I recommend this [talk][systemfw-state-in-fp] by Fabio Labella.

## What is STM?

Before getting into the implementation details, I'd like to summarize how STM works, well from an end user perspective anyway.
Take the following with a generous pinch of salt, as I'm fairly new to this topic.

According to the ZIO scaladoc, a value of type **STM[E,A]** 
>STM[E,A] represents an effect that can be performed transactionally, resulting in a failure `E` or a value `A`

Transactionally here means that we have isolation from other transactions when reading and writing transactional entities. Take for instance
the classic problem of transferring money from bank account 1 to bank account 2. 

Transfer 5 monies from account 1 to 2:
1. Check that account 1 has a balance of at least 5, otherwise abort transfer
2. Account 1 is credited 5
3. Account 2 is debited 5

If two transfers were started at the same time, we could end up with a negative balance in account 1. 

STM solves this problem by tracking all reads and writes, so if at the end of a transaction any of the transactional values involved
were changed by another transaction committing, the current transaction is retried.

In this example, the two transactions would both see the same values, and perform the same writes up until the point the first one commits.
When the other transaction sees that values it read/wrote were modified in another transaction, an automatic retry is triggered.
Since the retry will see the updated value of bank account 1, there is no risk of us ending up with a negative balance.

Because losing transactions are retried, we can't perform any IO as part of a transaction. Imagine if we had a non-idempotent side-effect
like calling a wallet service to update the balances of the bank accounts, and that happend as part of our transaction.

That might sound like a show-stopper. Turns out that with ZIO we're not actually performing any IO, we're just building a program consisting
of descriptions of the IO actions we'd like to perform. This approach is actually powerful enough to solve many problems, 
our partitioning use case included.


## Implementing our use case 

#### Publisher

The producer function is what the *partition[A]* function returns, more specifically it's the **A => UIO[Boolean]** wrapped 
inside of **ZIO[Any,Nothing,A => UIO[Boolean]]**.

The **Boolean** is a way of letting the caller of the producer function know if the submitted work was accepted or not. This is
our way of implementing back pressure, force the caller to decide what to do if they exceed their limit. If you imagine the caller being a web service 
processing incoming requests on behalf of different users, we might return an error asking the user to slow down. We could also just log the error,
or halt processing entirely (I would avoid that).

The standard way of decoupling a producer from a consumer is to put a message onto a queue, and have a separate fiber act as a consumer. 
STM provides a queue that can participate in transactions, it's called **TQueue[A]**. It's API is quite straightforward, it has
methods for publishing (*offer*), consuming (*take*), and for checking how many items are currently in the queue (*size*) and its
maximum capacity (*capacity*).


{% highlight scala %}

def publish[A](queue:TQueue[A], a:A):STM[Nothing, Boolean] =
 queue.size.flatMap(size => if(size == queue.capacity) STM.succeed(false) else queue.offer(a) *> STM.succeed(true))

{% endhighlight %}

Our *publish[A]* function checks that we have spare capacity before attempting to publish to the queue. If we didn't have this check, we could end up
suspending the fiber trying to publish which we don't want in this case. 

If you're not familiar with the {% highlight scala %} fa *> fb {% endhighlight %} operator it's essentially the the same thing as writing 

{% highlight scala %}

for {
 _ <- fa
 b <- fb
} yield b

{% endhighlight %}

Because we're using STM, we don't have to worry about the number of queued items changing between checking the remaining capacity and the call to 
*queue.offer(a)*. If another transaction commits, and dequeues/enqueues a message onto the queue this transaction will be retried. A retry obviously
doesn't mean that we publish the message twice, as the first publish would have been rolledback.

Note that we're not done with our producer function yet, this is just the publishing part of our producer. We'll return to it once we've 
seen how to build a consumer.

#### Consumer

Our consumer was defined as a function of type **A => UIO[Unit]**. We need to listen to messages from the queue, and then build the appropriate
action to take once the transaction commits.

This consumer needs to run in it's own fiber. I've referred to fibers a few times without explaining what they are. Just like a value of type **ZIO[R,E,A]**
describes a program which will fail with an **E** or succeed with an **A** given an environment **R**, a **Fiber[E,A]** is a value representing a running computation 
which can either fail with an **E** or succeed with an **A**. The distinction is very important, a value of **ZIO[R,E,A]** can be rerun as many times as you require
as it's the description of a program. You can't do the same with a Fiber[E,A], because it represents something that is already running.

**Fiber[E,A]** is in many ways the equivalent of a **Future[A]**, except it explicitly tracks how it can fail. Another really important distinction is that a 
**Fiber[E,A]** when interpreted by the ZIO runtime environment runs as a green thread, whereas a Future will run directly on a OS thread. The ability to
have multiple green threads running concurrently on a single OS thread lets us save a lot of resources (especially if all of our IO uses non-blocking
operations).

A **Fiber[E,A]** is created by calling *fork* on a value of type **ZIO[R,E,A]**, this tells the runtime environment to run the program described
by the **ZIO[R,E,A]** value on a new fiber.

Now that we know how to make our consumer run on its own fiber, let's go through the actions our consumer need to perform
1. Take a message from the queue or timeout if there are no more messages, 
2. Perform user action, if timeout / defect swallow and log it.
3. Repeat forever until we are interrupted, or there's a timeout in step 1.
4. If the Fiber is terminated we need to perform any related clean up action.

{% highlight scala %}

def debug(cause: Exit.Cause[String]): ZIO[Console, Nothing, Unit] =
  putStrLn(cause.prettyPrint)

def takeNextMessageOrTimeout[A](id: PartId, queue: TQueue[A]): ZIO[Clock with Conf, String, A] =
  idleTTL flatMap queue.take.commit.timeoutFail(s"$id consumer expired")

def safelyPerformAction[A](id: PartId, action: A => UIO[Unit])(a:A): ZIO[PartEnv, Nothing, Unit] =
  (userTTL flatMap (action(a).timeoutFail(s"$id action timed out")(_))).sandbox.catchAll(debug)

def startConsumer[A](id:PartId, queue: TQueue[A], cleanup:UIO[Unit], action: A => UIO[Unit]): ZIO[PartEnv, Nothing, Unit] =
  (takeNextMessageOrTimeout(id, queue) flatMap safelyPerformAction(id, action)).forever.option.ensuring(cleanup).fork.unit

{% endhighlight %}

For people used to working primarily with Futures, it's probably surprising to see the call to *timeoutFail* after we've called *commit*. If you think about 
this code as a series of descriptions it's easier to understand what's going on. When we call *commit*, we've got a **ZIO[Any,Nothing,A]**, and calling
*timeoutFail* on that value is going to produce a value of type **ZIO[R,String,A]**. Because we're dealing with descriptions that makes sense. 

There's a little bit of subtlety when *flatMap* is involved, because of it's signature. We need to provide a function with the signature **A => ZIO[R,E,B]**,
and this means that any timeout set on the **ZIO[R,E,B]** inside of the *flatMap* will only have an effect on the instructions included inside that value. 
In practice this means that anything before the *flatMap* and after is unaffected by a call to timeout inside of the *flatMap*. 

To make this a little clearer, let's go through the takeNextMessageOrTimeout method in more detail

| Expression      | Type before           | Type after                      | Effect                                                                        |
|-----------------|-----------------------|---------------------------------|-------------------------------------------------------------------------------|
| idleTTL         |                       | ZIO[Conf, Nothing, Duration]    | Will return the timeout value for how long we can wait for a message          |
| queue           |                       | TQueue[A]                       |                                                                               |
| take            | TQueue[A]             | STM[Nothing,A]                  | Part of a transaction that takes a message of type A from the queue           |
| commit          | STM[Nothing,A]        | ZIO[Conf, Nothing, A]           | A program using Conf, producing an A from the committed transaction           |
| timeoutFail(..) | ZIO[Conf, Nothing, A] | ZIO[Clock with Conf, String, A] | A program using Clock & Conf, either failing with String or succeeding with A |


The final signature tells us quite a bit, this function needs a Clock and a Conf provided to it before it can be run, and when it is run it will either
fail with a String or succeed with an A. 

Another surprising thing might be the return type of safelyPerformAction, where the return type indicates that it can not fail, eventhough there's a call
to *timeoutFail*. This is because of the call to sandbox, which will lift both expected failures and defects into a special data structure called **Exit.Cause[E]**.
We do this for two reasons, one is to catch any timeouts from the user provided action, but also to catch any potential defects that might lurk in the user
defined action. If we didn't use sandbox, we'd risk that any error/defect in the provided action would terminate the fiber, which is not what we want. 

The question is what to do with any potential failures? In this particular scenario I decided that the best thing to do was to simply log them to the console.
The latest ZIO (1.0-RC5) includes support for monadic tracing, which is very similar to a stack trace, awesome feature which I'll show some samples of later.

We don't use sandbox to swallow the timeout that can happen while we're taking from the queue. This is intentional, if a consumer hasn't received
any messages for a while we assume it's safe to stop processing messages for the relevant consumer. The timeout is how we achieve that, as the *forever*
effect will not repeat the effect in case of errors. To prevent spamming the output with stack traces, we add the *option* call. It will move errors
into the result and ensure a clean termination of the fiber after the cleanup action has been invoked (*ensuring* is like a finalizer).

Our little consumer program is nearly done, we just need to add an instruction to say that all of the above should happen in a dedicated fiber, by calling *fork*.

Finally, to make the return type a little prettier, we also call *unit* (as we don't need to interact with the forked fiber, we can ignore it).

#### Tying it all together

We have our publisher, and we have our consumer. Now all we need is a way to tie all these parts together. 

When the *producer* function is invoked we need to
1. check if there are any existing consumers for the relevant partition
2. if not, then we need to create a new consumer for the partition
3. fetch the right queue for the partition
4. publish the incoming message to it's consumer
5. return the result of the publish (it will be true if the message was accepted by the queue, otherwise false), and the consumer
6. take the result and consumer from the committed STM transansaction and run them

Because the consumers can come and go, we need to make sure that the map of partition ids to queues (**Map[PartId,TQueue[A]]**), can participate in transactions. 
This means we need to wrap it in a transactional reference, the **TRef[A]** type.

To make the following code a little more readable, I've introduced two type aliases
- **Queues[A]** is an alias for **TRef[Map[PartId,TQueue[A]]]**
- **PartEnv** is an alias for Clock with Console with Conf, 

{% highlight scala %}

type Queues[A] = TRef[Map[PartId,TQueue[A]]]
type PartEnv   = Clock with Console with Conf

def hasConsumer[A](queues:Queues[A], id:PartId): STM[Nothing, Boolean] =
  queues.get.map(_.contains(id))

def removeConsumerFor[A](queues:Queues[A], id: PartId): UIO[Unit] =
  queues.update(_ - id).unit.commit

def getWorkQueueFor[A](queues:Queues[A], id: PartId): STM[Nothing, TQueue[A]] =
  queues.get.map(_(id))

def setWorkQueueFor[A](queues:Queues[A], id:PartId, queue:TQueue[A]): STM[Nothing, Unit] =
  queues.update(_.updated(id, queue)).unit

def createConsumer[A](queues:Queues[A], id:PartId, maxPending:Int, action: A => UIO[Unit]): STM[Nothing, ZIO[PartEnv, Nothing, Unit]] =
  for {
    queue <- TQueue.make[A](maxPending)
    _     <- setWorkQueueFor(queues, id, queue)
  } yield startConsumer(id, queue, removeConsumerFor(queues, id), action)

def producer[A](queues:Queues[A], partIdOf:A => PartId, action: A => UIO[Unit])(a:A): ZIO[PartEnv, Nothing, Boolean] =
  maxPending >>= { maxPending:Int =>
    STM.atomically {
      for {
           exists <- hasConsumer(queues, partIdOf(a))
               id  = partIdOf(a)
         consumer <- if (exists) STM.succeed(ZIO.unit) else createConsumer(queues, id, maxPending, action)
            queue <- getWorkQueueFor(queues, partIdOf(a))
        published <- publish(queue, a)
      } yield ZIO.succeed(published) <* consumer
    }.flatten
  }
{% endhighlight %} 

The inner for comprehension results in a value of **STM[Nothing, ZIO[PartEnv, Nothing, Boolean]]**, this value represents a transaction that will 
result in a program that will start consuming from a queue (or not, if there's already an active consumer) and yield a value indicating whether the publish succeeded. 

the *STM.atomically* block takes a value of type **STM[E,A]** and turns it into a **ZIO[Any,E,A]**, it's the exact same thing as calling *commit*, just
with different syntax. In this case, we get **ZIO[Any,Nothing,ZIO[PartEnv,Nothing,Boolean]]**. Our goal is to return a **ZIO[PartEnv, Nothing, Boolean]**, and
the easiest way to do that is to *flatten* it. 

We're now almost feature complete, we just need to hook our implementation up with the API we defined.

#### The final piece of the puzzle

There are only some parts that I haven't showed of the implementation of the *partition* function. Let's see the missing parts now

{% highlight scala %}

def partition[A](config: Config, partIdOf: A => PartId, action: A => UIO[Unit]): ZIO[Any, Nothing, A => UIO[Boolean]] =
  TRef.make(Map.empty[PartId, TQueue[A]]).commit.map(
    queues => producer(queues, partIdOf, action)(_).provide(buildEnv(config, env))
  )

trait Conf {
  def userTTL: Duration
  def idleTTL: Duration
  def maxPending: Int
}

def buildEnv(conf:Config, env:Clock with Console):PartEnv =
  new Conf with Clock with Console {
    override def userTTL: Duration = conf.userTTL
    override def idleTTL: Duration = conf.idleTTL
    override def maxPending: Int = conf.maxPending

    override val clock:Clock.Service[Any] = env.clock
    override val scheduler:Scheduler.Service[Any] = env.scheduler
    override val console:Console.Service[Any] = env.console
  }

val userTTL:ZIO[Conf, Nothing, Duration] =
  ZIO.access[Conf](_.userTTL)

val idleTTL:ZIO[Conf, Nothing, Duration] =
  ZIO.access[Conf](_.idleTTL)

val maxPending:ZIO[Conf, Nothing, Int] =
  ZIO.access[Conf](_.maxPending)

{% endhighlight %}

I won't go into much detail here, but the *userTTL*, *idleTTL* and *maxPending* values are utilizing the ZIO approach for doing dependency injection. The
*buildEnv* function is what builds the actual implementations that our functions will use. To plug them in we need to call *provide*. If we have a value
of type **ZIO[R,E,A]**, then we need a value of type R to call *provide*, and that will result in a new value of **ZIO[Any,E,A]**. 

In the [accompanying source code][github-repo-link] you can see some more details around how everything is wired together (maybe a topic for another blog post?).

## Testing it

I wanted to get a feel for how we can test this code, so I wrote some basic tests (far from what I would consider exhaustive :). I also wrote a short
demo "app" to show some of the behaviors. 

Testing pure functions is remarkably easy, as all the required dependencies are right there in the signature of the method being tested. There's no need
to jump through hoops to do mocking. The most pleasant tests that I wrote were those for the *publish[A]* function.

{% highlight scala %}

class PublishTests extends BaseTests {

  behavior of "a publisher"

  it should "return true when publishing to an empty TQueue" in {
    runSTM {
      (TQueue.make[Int](1) >>= (publish(_, 1))) map (published => assert(published))
    }
  }

  it should "return false when publishing to a full TQueue" in {
    runSTM(
      (TQueue.make[Int](0) >>= (publish(_, 1))) map (published => assert(!published))
    )
  }
}

{% endhighlight %}

I ended up having to write a little helper function called runSTM, which takes a value of STM[Nothing,Assertion] and calls commit and then runs it using
the runtime. 

Unfortunately not everything was quite as nice as the above tests. I tried to use the TestClock and TestConsole provided by scalaz-zio-testkit to unit test
the timeouts, and that turns out to be impossible. Nonetheless, we can still run the tests using the real non-deterministic runtime and test them that way.

{% highlight scala %}

  val config =
    Config(
      userTTL = Duration(100, MILLISECONDS),
      idleTTL = Duration(100, MILLISECONDS),
      maxPending = 1
    )

  behavior of "a consumer"

  it should "always successfully process a value on the queue" in {
    runReal(
      for {
        env     <- partEnv(config)
        queue   <- TQueue.make[String](1).commit
        promise <- Promise.make[Nothing,String]
        _       <- startConsumer("p1", queue, UIO.unit, promise.succeed(_:String).unit).provide(env)
        _       <- queue.offer("published").commit
        result  <- promise.await.timeoutFail("not published")(Duration(150,MILLISECONDS)).fold(identity,identity)
      } yield assert(result == "published")
    )
  }


{% endhighlight %}

This is a pattern that repeats, so if this were a real application I would add a helper function/fixture for abstracting over
the promise-publish-consume-await pattern. The real problem though isn't that there's a little bit of boilerplate, but rather
that we're running on the real runtime. This means that we have to set all the timeouts to cater for the slowest machine
that will run our build. Maybe that's a small wart, but a little bit of a wart nonetheless.


### The Demo App

I decided to write a little sample program what will show case some the features we've implemented. It's just a silly program that
will take a list of numbers, and for each number we have a short delay and print a message to the console.

{% highlight scala %}

object PartitioningDemo extends App {

  val config:Config = Config(userTTL = Duration(3, SECONDS), idleTTL = Duration(2, SECONDS), maxPending = 3)

  def brokenUserFunction(startTs:Long, counts:Ref[Map[Int,Int]])(n:Int): ZIO[Console with Clock, Nothing, Unit] =
    ZIO.descriptorWith( desc =>
      for {
        now <- sleep(Duration(100 * n, MILLISECONDS)) *> currentTime (MILLISECONDS)
        m   <- counts.update(m => m.updated(n, m.getOrElse(n, 0) + 1))
        msg = s"Offset: ${now - startTs}ms Fiber: ${desc.id}, n = $n (call #${m(n)})"
        _   <- if (n == 0 && m(n) == 1) throw new IllegalArgumentException(msg) else putStrLn(msg)
      } yield ()
    )

  val workItems: List[Int] = List.range(0,11) ::: List.range(0,11) ::: List.range(0, 31)

  val program: ZIO[Environment with Partition, Nothing, Int] =
    for {
          now <- clock.currentTime(TimeUnit.MILLISECONDS)
      counter <- Ref.make(Map.empty[Int,Int])
          env <- ZIO.environment[Console with Clock]
      process <- partition[Int](config, _.toString, brokenUserFunction(now,counter)(_).provide(env))
      results <- ZIO.foreach(workItems)(process)
            _ <- console.putStrLn(s"Published ${results.count(identity)} out of ${results.length}")
            _ <- ZIO.sleep(Duration.fromScala(10.seconds))
    } yield 0

  override def run(args: List[String]): ZIO[Environment, Nothing, Int] =
    program.provideSome[Environment]( env =>
      new Clock with Console with System with Random with Blocking with Partition.Live {
        override val  blocking:  Blocking.Service[Any] = env.blocking
        override val     clock:     Clock.Service[Any] = env.clock
        override val   console:   Console.Service[Any] = env.console
        override val    random:    Random.Service[Any] = env.random
        override val    system:    System.Service[Any] = env.system
      })

}

{% endhighlight %} 

I think the interesting bit to notice here is that we will blow up with an exception the first time *brokenFunction* is called for partition id 0. 
There will also be a timeout triggered for the last message (30 times 100 will put it at the limit of 3 seconds for userTTL). Let's have a look at the 
output from running this.

{% highlight bash %}

Published 53 out of 53
Fiber failed.
An unchecked error was produced.
java.lang.IllegalArgumentException: Offset: 623ms Fiber: 265, n = 0 (call #1)
    at freskog.concurrency.app.PartitioningDemo$.$anonfun$brokenUserFunction$7(PartitioningDemo.scala:27)
    at ...

Fiber:265 was supposed to continue to:
  at ...

Fiber:265 execution trace:
  at freskog.concurrency.app.PartitioningDemo$.brokenUserFunction(PartitioningDemo.scala:25)
  at freskog.concurrency.app.PartitioningDemo$.brokenUserFunction(PartitioningDemo.scala:25)
  at freskog.concurrency.app.PartitioningDemo$.brokenUserFunction(PartitioningDemo.scala:24)
  at freskog.concurrency.app.PartitioningDemo$.brokenUserFunction(PartitioningDemo.scala:24)
  at freskog.concurrency.app.PartitioningDemo$.brokenUserFunction(PartitioningDemo.scala:24)
  at ...

Fiber:265 was spawned by:

Fiber:2 was supposed to continue to:
  a future continuation at freskog.concurrency.partition.Partition$.safelyPerformAction(Partition.scala:71)
  at ...

Fiber:2 execution trace:
  at freskog.concurrency.partition.Partition$.safelyPerformAction(Partition.scala:71)
  at freskog.concurrency.partition.Partition$.startConsumer(Partition.scala:74)
  at ...

Fiber:2 was spawned by:

Fiber:1 was supposed to continue to:
  a future continuation at freskog.concurrency.app.PartitioningDemo$.program(PartitioningDemo.scala:40)
  at ...

Fiber:1 execution trace:
  at freskog.concurrency.partition.Partition$.producer(Partition.scala:103)
  at freskog.concurrency.partition.Partition$.producer(Partition.scala:95)
  at freskog.concurrency.app.PartitioningDemo$.program(PartitioningDemo.scala:39)
  at freskog.concurrency.partition.Partition$Live$$anon$1.partition(Partition.scala:28)
  at ...

Fiber:1 was spawned by:

Fiber:0 was supposed to continue to:
  a future continuation at scalaz.zio.App.main(App.scala:57)

Fiber:0 ZIO Execution trace: <empty trace>

Fiber:0 was spawned by: <empty trace>
Offset: 667ms Fiber: 285, n = 0 (call #2)
Offset: 671ms Fiber: 293, n = 0 (call #3)
Offset: 723ms Fiber: 210, n = 1 (call #1)
Offset: 825ms Fiber: 209, n = 2 (call #1)
Offset: 825ms Fiber: 305, n = 1 (call #2)
Offset: 926ms Fiber: 181, n = 3 (call #1)
Offset: 927ms Fiber: 318, n = 1 (call #3)
Offset: 1026ms Fiber: 193, n = 4 (call #1)
Offset: 1027ms Fiber: 317, n = 2 (call #2)
Offset: 1121ms Fiber: 157, n = 5 (call #1)
Offset: 1226ms Fiber: 221, n = 6 (call #1)
Offset: 1229ms Fiber: 331, n = 3 (call #2)
Offset: 1229ms Fiber: 349, n = 2 (call #3)
Offset: 1325ms Fiber: 159, n = 7 (call #1)
Offset: 1426ms Fiber: 261, n = 8 (call #1)
Offset: 1428ms Fiber: 341, n = 4 (call #2)
Offset: 1525ms Fiber: 238, n = 9 (call #1)
Offset: 1531ms Fiber: 377, n = 3 (call #3)
Offset: 1623ms Fiber: 251, n = 10 (call #1)
Offset: 1624ms Fiber: 357, n = 5 (call #2)
Offset: 1722ms Fiber: 188, n = 11 (call #1)
Offset: 1824ms Fiber: 245, n = 12 (call #1)
Offset: 1829ms Fiber: 365, n = 6 (call #2)
Offset: 1829ms Fiber: 401, n = 4 (call #3)
Offset: 1925ms Fiber: 215, n = 13 (call #1)
Offset: 2022ms Fiber: 226, n = 14 (call #1)
Offset: 2027ms Fiber: 385, n = 7 (call #2)
Offset: 2124ms Fiber: 223, n = 15 (call #1)
Offset: 2126ms Fiber: 429, n = 5 (call #3)
Offset: 2224ms Fiber: 185, n = 16 (call #1)
Offset: 2228ms Fiber: 393, n = 8 (call #2)
Offset: 2325ms Fiber: 248, n = 17 (call #1)
Offset: 2422ms Fiber: 171, n = 18 (call #1)
Offset: 2427ms Fiber: 409, n = 9 (call #2)
Offset: 2431ms Fiber: 449, n = 6 (call #3)
Offset: 2523ms Fiber: 247, n = 19 (call #1)
Offset: 2626ms Fiber: 233, n = 20 (call #1)
Offset: 2626ms Fiber: 425, n = 10 (call #2)
Offset: 2726ms Fiber: 232, n = 21 (call #1)
Offset: 2730ms Fiber: 465, n = 7 (call #3)
Offset: 2831ms Fiber: 269, n = 22 (call #1)
Offset: 2923ms Fiber: 172, n = 23 (call #1)
Offset: 3023ms Fiber: 201, n = 24 (call #1)
Offset: 3029ms Fiber: 485, n = 8 (call #3)
Offset: 3127ms Fiber: 277, n = 25 (call #1)
Offset: 3225ms Fiber: 194, n = 26 (call #1)
Offset: 3323ms Fiber: 169, n = 27 (call #1)
Offset: 3329ms Fiber: 501, n = 9 (call #3)
Offset: 3425ms Fiber: 205, n = 28 (call #1)
Offset: 3528ms Fiber: 273, n = 29 (call #1)
Fiber failed.
A checked error was not handled.
30 action timed out

Fiber:102 was supposed to continue to:
  a future continuation at freskog.concurrency.partition.Partition$.safelyPerformAction(Partition.scala:71)
  at ...

Fiber:102 execution trace:
  at freskog.concurrency.partition.Partition$.safelyPerformAction(Partition.scala:71)
  at freskog.concurrency.partition.Partition$.startConsumer(Partition.scala:74)
  at freskog.concurrency.partition.Partition$.takeNextMessageOrTimeout(Partition.scala:68)
  at ...

Fiber:102 was spawned by:

Fiber:1 was supposed to continue to:
  a future continuation at ...

Fiber:1 execution trace:
  at freskog.concurrency.partition.Partition$.producer(Partition.scala:103)
  at freskog.concurrency.partition.Partition$.producer(Partition.scala:95)
  at ...

Fiber:1 was spawned by:

Fiber:0 was supposed to continue to:
  a future continuation at scalaz.zio.App.main(App.scala:57)

Fiber:0 ZIO Execution trace: <empty trace>

Fiber:0 was spawned by: <empty trace>
Offset: 3628ms Fiber: 521, n = 10 (call #3)

Process finished with exit code 0
{% endhighlight %}

For brevity, I've snipped some of the traces (the parts from the inside of the ZIO library itself) and replaced them with "at ...".

It's good to see that both the user defect and the timeout are logged as expected. We also see that all the messages are processed in the
expected order, with parallelism between the different partitions. Again, we can see that having an error for the first message
for partition id 0, didn't prevent subsequent messages from being processed correctly in that partition.

I haven't shown that resources are being freed, but I'll leave that as an excercise for the reader :)

One last interesting thing to note is that we saw all messages being published. If we wanted to simulate back pressure kicking in, we could run the program again
with a max pending of just 1. Let's see what happens!

{% highlight bash %}

Published 31 out of 53
...
Offset: 806ms Fiber: 273, n = 1 (call #1)
Offset: 902ms Fiber: 165, n = 2 (call #1)
Offset: 1004ms Fiber: 269, n = 3 (call #1)
Offset: 1104ms Fiber: 238, n = 4 (call #1)
Offset: 1201ms Fiber: 221, n = 5 (call #1)
Offset: 1303ms Fiber: 159, n = 6 (call #1)
Offset: 1403ms Fiber: 171, n = 7 (call #1)
Offset: 1502ms Fiber: 259, n = 8 (call #1)
Offset: 1603ms Fiber: 190, n = 9 (call #1)
Offset: 1715ms Fiber: 277, n = 10 (call #1)
Offset: 1804ms Fiber: 194, n = 11 (call #1)
Offset: 1903ms Fiber: 243, n = 12 (call #1)
Offset: 2002ms Fiber: 157, n = 13 (call #1)
Offset: 2103ms Fiber: 226, n = 14 (call #1)
Offset: 2203ms Fiber: 237, n = 15 (call #1)
Offset: 2303ms Fiber: 186, n = 16 (call #1)
Offset: 2402ms Fiber: 203, n = 17 (call #1)
Offset: 2501ms Fiber: 225, n = 18 (call #1)
Offset: 2602ms Fiber: 254, n = 19 (call #1)
Offset: 2705ms Fiber: 167, n = 20 (call #1)
Offset: 2803ms Fiber: 178, n = 21 (call #1)
Offset: 2903ms Fiber: 232, n = 22 (call #1)
Offset: 3003ms Fiber: 212, n = 23 (call #1)
Offset: 3102ms Fiber: 158, n = 24 (call #1)
Offset: 3204ms Fiber: 249, n = 25 (call #1)
Offset: 3300ms Fiber: 253, n = 26 (call #1)
Offset: 3398ms Fiber: 209, n = 27 (call #1)
Offset: 3502ms Fiber: 177, n = 28 (call #1)
Offset: 3604ms Fiber: 211, n = 29 (call #1)
Offset: 3699ms Fiber: 265, n = 30 (call #1)
...
{% endhighlight %}

I've snipped out the monadic traces entirely this time, and 
31 out of 53 work items were successfully published, in this silly app we simply ignore the failed ones. In this case it looks like we managed to publish one message
for each partition. Notice that the publish succeeded for n = 0, but we throw an exception so it doesn't show up with the other messages.

## Conclusions

I have to say it was a great experience trying out STM and ZIO in general. It's extremely well thought out, and my only gripe so far has been that I can't unit test
my timeouts in a deterministic fashion. There's an open ticket [here][timeout-bug] which hopefully will get some attention soon. Other than that, I have to
say I can't find any faults from an end user perspective.

One thing which I haven't covered in this post is performance. The maintainers of ZIO itself keep pretty good benchmarks, and it's worth trying those yourself.

That said for IO heavy loads, I would be very comfortable using ZIO in production as it is, assuming that the team felt comfortable with the library.

If you want to play around with the code, I've prepared a [github project][github-repo-link] with all the code I've shown here.

/Fred

[systemfw-state-in-fp]: https://vimeo.com/294736344
[github-repo-link]: https://github.com/freskog/stm-partitioning
[timeout-bug]: https://github.com/zio/zio/issues/925
[zio-github]: https://github.com/zio/zio
