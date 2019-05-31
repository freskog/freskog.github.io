---
layout: post
author: freskog
title: Exploring the STM functionality in ZIO
---

## Introduction

If you've been following the Scala community lately, you've definitely seen ZIO getting more and more attention.
ZIO bills itself as:

>ZIO — A type-safe, composable library for asynchronous and concurrent programming in Scala

This post is a summary my learnings of using the STM (Software Transactional Memory) feature in ZIO.

STM is a tool for writing concurrent programs. It does this by bringing the idea of transactions into concurrent programming. Full disclaimer
here, I'm certainly no expert on this topic so please be gentle brave keyboard warriors. 

To get an idea of how to use the STM functionality, I decided to implement a use case around partitioning workloads. Let's say you
have a queue of incoming payloads, each payload needs to be processed using some function. The processing must be done sequentially for
all payloads that belong to the same partition, but two payloads belonging to different partitions can be processed concurrently.

I'm going to assume some pre-requisite knowledge in order to keep the length of this text manageable. Specifically, I'll assume
that you are familiar with the standard Scala Future and also Cats IO or Scalaz Task.

## Requirements

Rather than coming up with a detailed use case, I'm just going to list the non functional requirements we need to fulfill

* All payloads belonging to the same partition are processed sequentially, by a user provided function
* In case of a defect/timeout in the user function, we want to log the cause and resume processing payloads
* If a partition is idle for too long, then all resources allocated to it must be freed
* We need to have a mechanism to prevent one partition from stealing all the available capacity (limit max pending work)
* All timeouts and max capacity per user must be configurable by the caller

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
|          | PartId                                              | A type alias for String, which represents a partition            |
|          | UIO[_]                                              | Unfallible IO, an effect type that has no expected failure modes |
| config   | Config                                              | User provided configuration for timeouts & back pressure         |
| partIdOf | A => PartId                                         | Function that determines what partition a payload belongs to     |
| action   | A => UIO[Unit]                                      | Function that performs an unfallible effect and returns unit     |
|          | ZIO[Any, Nothing, A => UIO[Boolean]]                | The return type, a producer function wrapped inside of an effect |

I think most of the types involved here are fairly straight forward, except for the return type. It might be worth taking a moment
to try to understand it better. It consists of three parts, the resouce part, error part and the value part.

**Any** is the resource part, if you've ever used a DI framework, this is the ZIO
equivalent. We can see that this function doesn't have any external dependencies. If it had said **Console with Clock**, then
we would need to provide an environment that contains the implementation for both of these services before running the effect.

**Nothing**, just by looking at this return type it's clear that this effect has no expected failures. That doesn't mean it
won't have defects though. In ZIO there's a big distinction between failures we expect to handle, and defects that we either
didn't anticipate, or we just can reasonably handle (i.e., database was eaten by gnomes). I generally try to handle
failures close to where they occur (perhaps a retry etc), and to let defects propagate to the entry point of the system, where
all defects that happened are logged together.

**A => UIO[Boolean]**, this is the function the user must call to insert work into the system, aka the producer function. Notice
that this function is itself effectful, which makes sense. It's worth nothing that **UIO[A]** is a type alias for **ZIO[Any,Nothing,A]**.

Why do we need to have two layers of effects?

As it turns out, we need to share some state between multiple calls to the producer function, and doing that requires an effect. 
The producer function does return an effect, but multiple calls to the same function can't interact without breaking
some of the fundamental assumptions of how ZIO works.

If you want to learn more about why that is, I recommend this [talk][systemfw-state-in-fp] by Fabio Labella.


## What is STM?

Before getting into the implementation details, I'd like to summarize how STM works, from an end user perspective. 
Take the following with a generous pinch of salt, as I'm fairly new to this topic.

According to the ZIO scaladoc, a value of type STM[E,A] 
>... represents an effect that can be performed transactionally, resulting in a failure `E` or a value `A`

Transactionally here means that we have isolation from other transactions when reading and writing transactional entities. Take for instance
the classic problem of transferring money from bank account 1 to bank account 2. 

Transfer 5 monies from account 1 to 2:
1. Check that account 1 has a balance of at least 5, otherwise abort transfer
2. Account 1 is credited 5
3. Account 2 is debited 5

If two transfers were started at the same time, we could end up with a negative balance in account 1. 

STM solves this problem by tracking all reads and writes, so if at the end of a transaction any of the transactional values involved
were changed by another transaction, the current transaction is retried.

In this example, the two transactions would both see the same values, and perform the same writes up until the point the first one commits.
When the other transaction sees that values it read/wrote were modified in another transaction, an automatic retry is triggered.
Since the retry will see the updated value of bank account 1, there is no risk of us ending up with a negative balance.

Because losing transactions are retried, we can't perform any IO as part of a transaction. Imagine if we had a non-idempotent side-effect
like calling a wallet service to update the balances of the bank accounts, and that happend as part of our transaction.

The way ZIO solves this is build a description of what IO actions to perform as the end value. I'll show one way of doing this.

## Implementing our use case 

#### Publisher

A producer is what the **partition** function returns wrapped inside of ZIO, and as we recall the inner type was **A => UIO[Boolean]**.
The standard way of decoupling a producer from a consumer is to put a message onto a queue. We'll do exactly that here as well.
STM provides a queue that can participate in transactions, it's called TQueue. We can easily write a function that will publish to
a queue if it's not already full.

{% highlight scala %}

def publish[A](queue:TQueue[A], a:A):STM[Nothing, Boolean] =
 queue.size.flatMap(size => if(size == queue.capacity) STM.succeed(false) else queue.offer(a) *> STM.succeed(true))

{% endhighlight %}

Because we're using STM, we don't have to worry about the number of queued items changing between checking the remaining capacity and the call to 
queue.offer(a).

Note that we're not done with our producer function yet, this is just the publishing part of our producer. We'll return to it once we've 
seen how to build a consumer.

#### Consumer

Our consumer was defined as a function of type **A => UIO[Unit]**. We need to listen to messages from the queue, and then build the appropriate
action to take once the transaction commits.

This consumer needs to run in it's own Fiber (a fiber in ZIO models a running computation, much like how Future does,
 but the ZIO runtime will use green threads instead of actual threads). 

Basically the workflow for the consumer is
1. Take a message from the queue or timeout if there are no more messages, 
2. Perform user action, if timeout / defect swallow and log it.
3. Repeat forever until we are interrupted, or there's a timeout in step 1.
4. If the Fiber is terminated we need to perform any related clean up action.

{% highlight scala %}

def debug(cause: Exit.Cause[String]): ZIO[Console, Nothing, Unit] =
 console.putStrLn(cause.failures.mkString("\n\t") + 
   cause.defects.mkString("\n\t"))
                
def takeNextMessageOrTimeout(id: PartId, queue: TQueue[A]): ZIO[Clock, String, A] =
 queue.take.commit.timeoutFail(s"$id was retired")(Duration(2, SECONDS))

def safelyPerformAction(id: PartId, action: A => UIO[Unit]): A => ZIO[Console with Clock, Nothing, Unit] =
 action(_).timeoutFail(s"$id action timed out")(Duration(3, SECONDS))
  .sandbox.catchAll(debug)

def startConsumer(id:PartId, queue: TQueue[A], cleanup:UIO[Unit], action: A => UIO[Unit]): ZIO[Console with Clock, Nothing, Unit] =
 (takeNextMessageOrTimeout(id, queue) flatMap safelyPerformAction(id, action))
  .forever.ensuring(cleanup).fork.unit

{% endhighlight %}

For people used to working primarily with Futures, it's probably surprising to see the call to timeout after we've called commit. With ZIO
we're dealing with descriptions of programs rather than running programs (the closest equivalent to a Future is as mentioned previously a Fiber in ZIO).

When we're working with STM[E,A] or ZIO[R,E,A]/IO[E,A]/UIO[A] we're always just manipulating a value. That value is a description of a 
transaction/program.

To make this a little clearer, let's go through the takeNextMessageOrTimeout method in more detail

| Expression      | Type before           | Type after            | Effect                                                                        |
|-----------------|-----------------------|-----------------------|-------------------------------------------------------------------------------|
| queue           |                       | TQueue[A]             |                                                                               |
| take            | TQueue[A]             | STM[Nothing,A]        | Part of a transaction that takes a message of type A from the queue           |
| commit          | STM[Nothing,A]        | UIO[A]                | A program producing a value of type A from the committed transaction          |
| timeoutFail(..) | UIO[A]                | ZIO[Clock, String, A] | A program using Clock, which either fails with a String or succeeds with an A |


The final signature tells us quite a bit, this function needs a Clock provided to it before it can be run, and when it is run it will either
fail with a String or succeed with an A. We can use a different implementation of the Clock service for testing (there is a test kit for ZIO
that contains a suitable implementation).

Another surprising thing might be the return type of safelyPerformAction, where the return type indicates that it can not fail. This is because of
us *sandbox*ing all failures and defects, which we then just log using our debug method. The end result still requires a Clock.

Finally, we need to ensure that we repeat the program that takes the next message with a timeout, and then safely performs an action forever.
We don't swallow the errors for the timeout that can happen while we're taking from the queue. This is intentional, if a consumer hasn't received
any messages for a while we assume it's safe to terminate it. Should the program be terminated, we want to ensure that the user provided
cleanup action is performed. The only thing left is to instruct the runtime to fork this program and run it on a separate fiber. To make
the return type a little prettier, we also call unit (as we don't need to interact with the forked fiber).


#### Tying it all together

We have our publisher, and we have our consumer. Now all we need is a way to tie all these parts together. We want our producer to
create a queue, start a consumer and publish a message for each new partition it detects. We need track the currently active partitions
as a **Map[PartId,TQueue[A]]**. Because the consumers can come and go, we need to make sure that this map can participate in transactions.
This means we need to introduce the **TRef[A]** type.

We'll ask who ever calls into us to provide a premade workQueues of type **TRef[Map[PartId,TQueue[A]]**, and to supply an env
of type  **Console with Clock**.

Let's look at the code

{% highlight scala %}

def hasConsumer(id:PartId):STM[Nothing, Boolean] =
 workQueues.get.map(_.contains(id))

def removeConsumerFor(id: PartId): IO[Nothing, Unit] =
 workQueues.update(_ - id).unit.commit

def getWorkQueueFor(id: PartId):STM[Nothing, TQueue[A]] =
 workQueues.get.map(_(id))

def setWorkQueueFor(id:PartId, queue:TQueue[A]):STM[Nothing, Unit] =
 workQueues.update(_.updated(id, queue)).unit

def createConsumer(id:PartId, action: A => UIO[Unit]): STM[Nothing, ZIO[Console with Clock, Nothing, Unit]] =
 for {
  queue <- TQueue.make[A](10)
      _ <- setWorkQueueFor(id, queue)
 } yield startConsumer(id, queue, removeConsumerFor(id), action)

def producerSTM(a:A): STM[Nothing, ZIO[Console with Clock, Nothing, Boolean]] =
 for {
     exists <- hasConsumer(partIdOf(a))
   consumer <- if(exists) STM.succeed(ZIO.unit) else createConsumer(partIdOf(a), action)
      queue <- getWorkQueueFor(partIdOf(a))
  published <- publish(queue,a)
 } yield ZIO.succeed(published) <* consumer

def producer(a:A):UIO[Boolean] =
 producerSTM(a).commit.flatten.provide(env)

{% endhighlight %} 

Some of these methods are quite straightforward, like *hasConsumer*, *removeConsumerFor* and *setWorkQueueFor*. The remaining three methods
are more interesting. 

For instance, in *createConsumer* we're both creating the work queue for a new consumer, and building the program
that will consume from said queue. Notice that we're building the program, as opposed to running it. We can't actually
(well we shouldn't at least) perform any IO inside of a transaction. ZIO makes that very explicit through the type system.

We don't want to create a consumer every time the producer function is called though, so we need a function that can check for existing
consumers, and call createConsumer only when needed and then publish the message. This is the job of *producerSTM*. Again, we're not
allowed to actually run any IO inside of a transaction, so all we can do is to build new programs that when run will do what we've
asked. In this method we're just combining the consumer and the result of the publish into a program that will when run consume
messages in a separate fiber. 

There's a small wart here, in that we end up having to create a dummy consumer in case one has already been defined. This dummy
consumer is actually just an effect that returns Unit.

The last method *producer*, is what builds the final effect returned to the user. We convert the producerSTM(..) call into a ZIO effect
by calling commit. Interestingly, this will give us an effect of the type **UIO[ZIO[Console with Clock, Nothing, Boolean]]**. To
meet our api specification, we just need to call *flatten*, and provide the passed in *env* on the effect value.

#### The final piece of the puzzle

There are only some parts that I haven't showed of the implementation of the *partition* function. Let's see the missing parts now

{% highlight scala %}
type PartId = String

def partition[A](partIdOf:A => PartId, action:A => UIO[Unit]):ZIO[Clock with Console, Nothing, A => UIO[Boolean]] =
 for {
  workQueues <- TRef.make(Map.empty[PartId,TQueue[A]]).commit
  env        <- ZIO.environment[Clock with Console]
  worker     <- ZIO.effectTotal {

  def publish(queue:TQueue[A], a:A):STM[Nothing, Boolean] =
    queue.size flatMap (size => if(size == queue.capacity) STM.succeed(false) else queue.offer(a) *> STM.succeed(true))

  def debug(cause: Exit.Cause[String]): ZIO[Console, Nothing, Unit] =
    console.putStrLn(cause.failures.mkString("\n\t") + cause.defects.mkString("\n\t"))

  def takeNextMessageOrTimeout(id: PartId, queue: TQueue[A]): ZIO[Clock, String, A] =
    queue.take.commit.timeoutFail(s"$id was retired")(Duration(2, SECONDS))

  def safelyPerformAction(id: PartId, action: A => UIO[Unit]): A => ZIO[Console with Clock, Nothing, Unit] =
    action(_).timeoutFail(s"$id action timed out")(Duration(3, SECONDS)).sandbox.catchAll(debug)

  def startConsumer(id:PartId, queue: TQueue[A], cleanup:UIO[Unit], action: A => UIO[Unit]): ZIO[Console with Clock, Nothing, Unit] =
    (takeNextMessageOrTimeout(id, queue) flatMap safelyPerformAction(id, action)).forever.ensuring(cleanup).fork.unit

  def hasConsumer(id:PartId):STM[Nothing, Boolean] =
    workQueues.get.map(_.contains(id))

  def removeConsumerFor(id: PartId): IO[Nothing, Unit] =
    workQueues.update(_ - id).unit.commit

  def getWorkQueueFor(id: PartId):STM[Nothing, TQueue[A]] =
    workQueues.get.map(_(id))

  def setWorkQueueFor(id:PartId, queue:TQueue[A]):STM[Nothing, Unit] =
    workQueues.update(_.updated(id, queue)).unit

  def createConsumer(id:PartId, action: A => UIO[Unit]): STM[Nothing, ZIO[Console with Clock, Nothing, Unit]] =
    for {
      queue <- TQueue.make[A](10)
          _ <- setWorkQueueFor(id, queue)
    } yield startConsumer(id, queue, removeConsumerFor(id), action)

  def producerSTM(a:A): STM[Nothing, ZIO[Console with Clock, Nothing, Boolean]] =
    for {
        exists <- hasConsumer(partIdOf(a))
      consumer <- if(exists) STM.succeed(ZIO.unit) else createConsumer(partIdOf(a), action)
         queue <- getWorkQueueFor(partIdOf(a))
     published <- publish(queue,a)
    } yield ZIO.succeed(published) <* consumer

  def producer(a:A):UIO[Boolean] =
    producerSTM(a).commit.flatten.provide(env)

  producer(_)

 }
} yield worker

{% endhighlight %}

That is the full implementation. In a real application this would perhaps have been built as a service instead of just a function like I did here.

## Experiments

## Conclusions

[systemfw-state-in-fp]: https://vimeo.com/294736344

