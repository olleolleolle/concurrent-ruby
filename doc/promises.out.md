# Basics

## Factory methods

Future and Event are created indirectly with constructor methods in
FactoryMethods. They are not designed for inheritance but rather for
composition.

```ruby
Concurrent::Promises::FactoryMethods.instance_methods(false)
# => [:zip,
#     :create,
#     :delay,
#     :future,
#     :resolvable_future,
#     :resolvable_event,
#     :resolvable_event_on,
#     :resolvable_future_on,
#     :future_on,
#     :resolved_future,
#     :fulfilled_future,
#     :rejected_future,
#     :resolved_event,
#     :delay_on,
#     :schedule,
#     :schedule_on,
#     :zip_futures,
#     :zip_futures_on,
#     :zip_events,
#     :zip_events_on,
#     :any_resolved_future,
#     :any_resolved_future_on,
#     :any,
#     :any_fulfilled_future,
#     :any_fulfilled_future_on,
#     :any_event,
#     :any_event_on]
```

The module can be included or extended where needed.

```ruby
Class.new do
  include Concurrent::Promises::FactoryMethods

  def a_method
    resolvable_event
  end
end.new.a_method
# => <#Concurrent::Promises::ResolvableEvent:0x7fcd849824f8 pending>

mod = Module.new do
  extend Concurrent::Promises::FactoryMethods
end 
mod.resolvable_event
# => <#Concurrent::Promises::ResolvableEvent:0x7fcd8497a690 pending>
```

The default executor can be changed by overriding `default_executor` method
inherited from `Concurrent::Promises::FactoryMethods`.

```ruby
mod = Module.new do
  extend Concurrent::Promises::FactoryMethods
  def self.default_executor
    :fast
  end
end 
mod.future { 1 }.default_executor        # => :fast
Concurrent::Promises.future { 1 }.default_executor
# => :io
```

The module is already extended into {Concurrent::Promises} for convenience.

```ruby
Concurrent::Promises.resolvable_event
# => <#Concurrent::Promises::ResolvableEvent:0x7fcd84922a30 pending>
```

## Asynchronous task

The most basic use-case of the framework is asynchronous processing. A task can
be processed asynchronously by using a `future` factory method. The block will
be executed on an internal thread pool.

Arguments of `future` are passed to the block and evaluation starts immediately.

```ruby
future = Concurrent::Promises.future(0.1) do |duration|
  sleep duration
  :result
end
# => <#Concurrent::Promises::Future:0x7fcd84903860 pending>
```

Asks if the future is resolved, here it will be still in the middle of the
sleep call.

```ruby
future.resolved?                         # => false
```

Retrieving the value will block until the future is resolved.

```ruby
future.value                             # => :result
future.resolved?                         # => true
```

If the task fails we talk about the future being rejected.

```ruby
future = Concurrent::Promises.future { raise 'Boom' }
# => <#Concurrent::Promises::Future:0x7fcd848d9a38 pending>
```

There is no result, the future was rejected with a reason.

```ruby
future.value                             # => nil
future.reason                            # => #<RuntimeError: Boom>
```

It can be forced to raise the reason for rejection when retrieving the value.

```ruby
begin
  future.value! 
rescue => e 
  e
end                                      # => #<RuntimeError: Boom>
```

Which is the same as `future.value! rescue $!` which will be used hereafter.

Or it can be used directly as argument for raise, since it implements exception
method.

```ruby
raise future rescue $!                   # => #<RuntimeError: Boom>
```

## States

Lets define a inspection helper for methods.

```ruby
def inspect_methods(*methods, of:)
  methods.reduce({}) { |h, m| h.update m => of.send(m) }
end 
```

Event has `pending` and `resolved` state. 

```ruby
event = Concurrent::Promises.resolvable_event 
inspect_methods(:state, :pending?, :resolved?, of: event)
# => {:state=>:pending, :pending?=>true, :resolved?=>false}

event.resolve 
inspect_methods(:state, :pending?, :resolved?, of: event)
# => {:state=>:resolved, :pending?=>false, :resolved?=>true}
```

Future's `resolved` state is further specified to be `fulfilled` or `rejected`.

```ruby
future = Concurrent::Promises.resolvable_future 
inspect_methods(:state, :pending?, :resolved?, :fulfilled?, :rejected?, 
    of: future)
# => {:state=>:pending,
#     :pending?=>true,
#     :resolved?=>false,
#     :fulfilled?=>false,
#     :rejected?=>false}

future.fulfill :value 
inspect_methods(:state, :pending?, :resolved?, :fulfilled?, :rejected?,
    :result, :value, :reason, of: future)
# => {:state=>:fulfilled,
#     :pending?=>false,
#     :resolved?=>true,
#     :fulfilled?=>true,
#     :rejected?=>false,
#     :result=>[true, :value, nil],
#     :value=>:value,
#     :reason=>nil}

future = Concurrent::Promises.rejected_future StandardError.new 
inspect_methods(:state, :pending?, :resolved?, :fulfilled?, :rejected?, 
    :result, :value, :reason, of: future)
# => {:state=>:rejected,
#     :pending?=>false,
#     :resolved?=>true,
#     :fulfilled?=>false,
#     :rejected?=>true,
#     :result=>[false, nil, #<StandardError: StandardError>],
#     :value=>nil,
#     :reason=>#<StandardError: StandardError>}
```

## Direct creation of resolved futures

When an existing value has to wrapped in a future it does not have to go
through evaluation as follows.

```ruby
Concurrent::Promises.future { :value }
# => <#Concurrent::Promises::Future:0x7fcd84290c80 pending>
```

Instead it can be created directly.

```ruby
Concurrent::Promises.fulfilled_future(:value)
# => <#Concurrent::Promises::Future:0x7fcd84250fb8 fulfilled>
Concurrent::Promises.rejected_future(StandardError.new('Ups'))
# => <#Concurrent::Promises::Future:0x7fcd84220340 rejected>
Concurrent::Promises.resolved_future(true, :value, nil)
# => <#Concurrent::Promises::Future:0x7fcd84173dc0 fulfilled>
Concurrent::Promises.resolved_future(false, nil, StandardError.new('Ups'))
# => <#Concurrent::Promises::Future:0x7fcd84208628 rejected>
```

## Chaining

Big advantage of promises is ability to chain tasks together without blocking
current thread.

```ruby
Concurrent::Promises.
    future(2) { |v| v.succ }.
    then(&:succ).
    value!                               # => 4
```

As `future` factory method takes argument, `then` method takes as well. Any
supplied arguments are passed to the block, and the library ensures that they
are visible to the block.

```ruby
Concurrent::Promises.
    future('3') { |s| s.to_i }.
    then(2) { |v, arg| v + arg }.
    value                                # => 5
Concurrent::Promises.
    fulfilled_future('3').
    then(&:to_i).
    then(2, &:+).
    value                                # => 5
Concurrent::Promises.
    fulfilled_future(1).
    chain(2) { |fulfilled, value, reason, arg| value + arg }.
    value                                # => 3
```

Passing the arguments in (similarly as for a thread `Thread.new(arg) { |arg|
do_stuff arg }`) is **required**, both following examples may break.

```ruby
arg = 1                                  # => 1
Thread.new { do_stuff arg }
# => #<Thread:0x007fcd84115bf8@promises.in.md:203 run>
Concurrent::Promises.future { do_stuff arg }
# => <#Concurrent::Promises::Future:0x7fcd841624a8 pending>
```

## Branching, and zipping

Besides chaining it can also be branched.

```ruby
head    = Concurrent::Promises.fulfilled_future -1 
branch1 = head.then(&:abs) 
branch2 = head.then(&:succ).then(&:succ) 

branch1.value!                           # => 1
branch2.value!                           # => 1
```

It can be combined back to one future by zipping (`zip`, `&`).

```ruby
branch1.zip(branch2).value!              # => [1, 1]
(branch1 & branch2).
    then { |a, b| a + b }.
    value!                               # => 2
(branch1 & branch2).
    then(&:+).
    value!                               # => 2
Concurrent::Promises.
    zip(branch1, branch2, branch1).
    then { |*values| values.reduce(&:+) }.
    value!                               # => 3
```

Instead of zipping only the first one can be taken if needed.

```ruby
Concurrent::Promises.any(branch1, branch2).value!
# => 1
(branch1 | branch2).value!               # => 1
```

## Blocking methods

In these examples we have used blocking methods like `value` extensively for
their convenience, however in practice is better to avoid them and continue
chaining.

If they need to be used (e.g. when integrating with threads), `value!` is a
better option over `value` when rejections are not dealt with differently.
Otherwise the rejection are not handled and probably silently forgotten.

## Error handling

When one of the tasks in the chain fails, the rejection propagates down the
chain without executing the tasks created with `then`.

```ruby
Concurrent::Promises.
    fulfilled_future(Object.new).
    then(&:succ).
    then(&:succ).
    result
# => [false,
#     nil,
#     #<NoMethodError: undefined method `succ' for #<Object:0x007fcd84be6d20>>]
```

As `then` chained tasks execute only on fulfilled futures, there is a `rescue`
method which chains a task which is executed only when the future is rejected. 
It can be used to recover from rejection.

Using rescue to fulfill to 0 instead of the error.

```ruby
Concurrent::Promises.
    fulfilled_future(Object.new).
    then(&:succ).
    then(&:succ).
    rescue { |err| 0 }.
    result                               # => [true, 0, nil]
```

Rescue not executed when there is no rejection.

```ruby
Concurrent::Promises.
    fulfilled_future(1).
    then(&:succ).
    then(&:succ).
    rescue { |e| 0 }. 
    result                               # => [true, 3, nil]
```

Tasks added with `chain` are evaluated always.

```ruby
Concurrent::Promises.
    fulfilled_future(1).
    chain { |fulfilled, value, reason| fulfilled ? value : reason }.
    value!                               # => 1
Concurrent::Promises.
    rejected_future(StandardError.new('Ups')).
    chain { |fulfilled, value, reason| fulfilled ? value : reason }.
    value!                               # => #<StandardError: Ups>
```

Zip is rejected if any of the zipped futures is.

```ruby
rejected_zip = Concurrent::Promises.zip(
    Concurrent::Promises.fulfilled_future(1),
    Concurrent::Promises.rejected_future(StandardError.new('Ups')))
# => <#Concurrent::Promises::Future:0x7fcd84b94430 rejected>
rejected_zip.result
# => [false, [1, nil], [nil, #<StandardError: Ups>]]
rejected_zip.
    rescue { |reason1, reason2| (reason1 || reason2).message }.
    value                                # => "Ups"
```

## Delayed futures

Delayed futures will not evaluate until asked by `touch` or other method
requiring resolution. 

```ruby
future = Concurrent::Promises.delay { sleep 0.1; 'lazy' }
# => <#Concurrent::Promises::Future:0x7fcd84b7e888 pending>
sleep 0.1 
future.resolved?                         # => false
future.touch
# => <#Concurrent::Promises::Future:0x7fcd84b7e888 pending>
sleep 0.2 
future.resolved?                         # => true
```

All blocking methods like `wait`, `value` call `touch` and trigger evaluation.

```ruby
Concurrent::Promises.delay { :value }.value
# => :value
```

It propagates trough chain up allowing whole or partial lazy chains.

```ruby
head    = Concurrent::Promises.delay { 1 } 
branch1 = head.then(&:succ) 
branch2 = head.delay.then(&:succ) 
join    = branch1 & branch2 

sleep 0.1 
```

Nothing resolves.

```ruby
[head, branch1, branch2, join].map(&:resolved?)
# => [false, false, false, false]
```

Force `branch1` evaluation.

```ruby
branch1.value                            # => 2
sleep 0.1 
[head, branch1, branch2, join].map(&:resolved?)
# => [true, true, false, false]
```

Force evaluation of both by calling `value` on `join`.

```ruby
join.value                               # => [2, 2]
[head, branch1, branch2, join].map(&:resolved?)
# => [true, true, true, true]
```

## Flatting

Sometimes it is needed to wait for a inner future. Apparent solution is to wait
inside the future `Concurrent::Promises.future { Concurrent::Promises.future { 1+1 }.value }.value`
however as mentioned before, `value` calls should be **avoided** to avoid
blocking threads. Therefore there is a flat method which is a correct solution
in this situation and does not block any thread.

```ruby
Concurrent::Promises.future { Concurrent::Promises.future { 1+1 } }.flat.value!
# => 2
```

A more complicated example.
```ruby
Concurrent::Promises.
    future { Concurrent::Promises.future { Concurrent::Promises.future { 1 + 1 } } }.
    flat(1).
    then { |future| future.then(&:succ) }.
    flat(1).
    value!                               # => 3
```

## Scheduling

Tasks can be planned to be executed with a time delay.

Schedule task to be executed in 0.1 seconds.

```ruby
scheduled = Concurrent::Promises.schedule(0.1) { 1 }
# => <#Concurrent::Promises::Future:0x7fcd84b0d7f0 pending>
scheduled.resolved?                      # => false
```

Value will become available after 0.1 seconds. 

```ruby
scheduled.value                          # => 1
```

It can be used in the chain as well, where the delay is counted form a moment
its parent resolves. Therefore following future will be resolved in 0.2 seconds.

```ruby
future = Concurrent::Promises.
    future { sleep 0.1; :result }.
    schedule(0.1).
    then(&:to_s).
    value!                               # => "result"
```

Time can be used as well.

```ruby
Concurrent::Promises.schedule(Time.now + 10) { :val }
# => <#Concurrent::Promises::Future:0x7fcd84ae7960 pending>
```

## Resolvable Future and Event:

Sometimes it is required to resolve a future externally, in these cases
`resolvable_future` and `resolvable_event` factory methods can be uses. See
{Concurrent::Promises::ResolvableFuture} and
{Concurrent::Promises::ResolvableEvent}.

```ruby
future = Concurrent::Promises.resolvable_future
# => <#Concurrent::Promises::ResolvableFuture:0x7fcd84adff30 pending>
```

The thread will be blocked until the future is resolved

```ruby
thread = Thread.new { future.value } 
future.fulfill 1
# => <#Concurrent::Promises::ResolvableFuture:0x7fcd84adff30 fulfilled>
thread.value                             # => 1
```

Future can be resolved only once.

```ruby
future.fulfill 1 rescue $!
# => #<Concurrent::MultipleAssignmentError: Future can be resolved only once. It's [true, 1, nil], trying to set [true, 1, nil]. {:current_result=>[true, 1, nil], :new_result=>[true, 1, nil]}>
future.fulfill 2, false                  # => false
```

## How are promises executed?

Promises use global pools to execute the tasks. Therefore each task may run on
different thread which implies that users have to be careful not to depend on
Thread local variables (or they have to set at the begging of the task and
cleaned up at the end of the task).

Since the tasks are running on may different threads of the thread pool, it's
better to follow following rules:

-   Use only data passed in through arguments or values of parent futures, to 
    have better control over what are futures accessing.
-   The data passed in and out of futures are easier to deal with if they are 
    immutable or at least treated as such.
-   Any mutable and mutated object accessed by more than one threads or futures 
    must be thread safe, see {Concurrent::Array}, {Concurrent::Hash}, and 
    {Concurrent::Map}. (Value of a future may be consumed by many futures.)
-   Futures can access outside objects, but they has to be thread-safe.

> *TODO: This part to be extended*

# Advanced

## Callbacks

```ruby
queue  = Queue.new                       # => #<Thread::Queue:0x007fcd84abf320>
future = Concurrent::Promises.delay { 1 + 1 }
# => <#Concurrent::Promises::Future:0x7fcd84abe3a8 pending>

future.on_fulfillment { queue << 1 } # evaluated asynchronously
future.on_fulfillment! { queue << 2 } # evaluated on resolving thread

queue.empty?                             # => true
future.value                             # => 2
queue.pop                                # => 2
queue.pop                                # => 1
```

## Using executors

Factory methods, chain, and callback methods have all other version of them
which takes executor argument.

It takes an instance of an executor or a symbol which is a shortcuts for the
two global pools in concurrent-ruby. `fast` for short and non-blocking tasks
and `:io` for blocking and long tasks.

```ruby
Concurrent::Promises.future_on(:fast) { 2 }.
    then_on(:io) { File.read __FILE__ }.
    value.size                           # => 26689
```

## Run (simulated process)

Similar to flatting is running. When `run` is called on a future it will flat
indefinitely as long the future fulfils into a `Future` value. It can be used
to simulate a thread like processing without actually occupying the thread.

```ruby
count = lambda do |v|
  v += 1
  v < 5 ? Concurrent::Promises.future_on(:fast, v, &count) : v
end
# => #<Proc:0x007fcd84a9c730@promises.in.md:511 (lambda)>
400.times.
    map { Concurrent::Promises.future_on(:fast, 0, &count).run.value! }.
    all? { |v| v == 5 }                  # => true
```

Therefore the above example finished fine on the the `:fast` thread pool even
though it has much less threads than there is the simulated process.

# Interoperability

## Actors

Create an actor which takes received numbers and returns the number squared. 

```ruby
actor = Concurrent::Actor::Utils::AdHoc.spawn :square do
  -> v { v ** 2 }
end
# => #<Concurrent::Actor::Reference:0x7fcd8419ab78 /square (Concurrent::Actor::Utils::AdHoc)>
```

Send result of `1+1` to the actor, and add 2 to the result send back from the
actor.

```ruby
Concurrent::Promises.
    future { 1 + 1 }.
    then_ask(actor).
    then { |v| v + 2 }.
    value!                               # => 6
```

So `(1 + 1)**2 + 2 = 6`.

The `ask` method returns future.

```ruby
actor.ask(2).then(&:succ).value!         # => 5
```

## Channel

There is an implementation of channel as well. Lets start by creating a
channel with capacity 2 messages.

```ruby
ch1 = Concurrent::Promises::Channel.new 2
# => <#Concurrent::Promises::Channel:0x7fcd84c15dc8 size:2>
```

We push 3 messages, it can be observed that the last future representing the
push is not fulfilled since the capacity prevents it. When the work which fills
the channel depends on the futures created by push it can be used to create
back pressure – the filling work is delayed until the channel has space for
more messages.

```ruby
pushes = 3.times.map { |i| ch1.push i }
# => [<#Concurrent::Promises::Future:0x7fcd84c0e1e0 fulfilled>,
#     <#Concurrent::Promises::Future:0x7fcd84c0de20 fulfilled>,
#     <#Concurrent::Promises::Future:0x7fcd84c0d4e8 pending>]
ch1.pop.value!                           # => 0
pushes
# => [<#Concurrent::Promises::Future:0x7fcd84c0e1e0 fulfilled>,
#     <#Concurrent::Promises::Future:0x7fcd84c0de20 fulfilled>,
#     <#Concurrent::Promises::Future:0x7fcd84c0d4e8 fulfilled>]
```

A selection over channels can be created with select_channel factory method. It
will be fulfilled with a first message available in any of the channels. It
returns a pair to be able to find out which channel had the message available.

```ruby
ch2    = Concurrent::Promises::Channel.new 2
# => <#Concurrent::Promises::Channel:0x7fcd84bef218 size:2>
result = Concurrent::Promises.select_channel(ch1, ch2)
# => <#Concurrent::Promises::ResolvableFuture:0x7fcd84bee390 fulfilled>
result.value!
# => [<#Concurrent::Promises::Channel:0x7fcd84c15dc8 size:2>, 1]

Concurrent::Promises.future { 1+1 }.then_push_channel(ch1)
# => <#Concurrent::Promises::Future:0x7fcd84be5808 pending>
result = (
    Concurrent::Promises.fulfilled_future('%02d') &      
        Concurrent::Promises.select_channel(ch1, ch2)).
    then { |format, (channel, value)| format format, value }
# => <#Concurrent::Promises::Future:0x7fcd84bdd810 pending>
result.value!                            # => "02"
```

## ProcessingActor

There is also a new implementation of actors based on the Channel and the
ability of promises to simulate process. The actor runs as a process but also
does not occupy a thread per actor as previous Concurrent::Actor
implementation. This implementation is close to Erlang actors, therefore OTP
can be ported for this actors (and it's planned).

The simplest actor is a one which just computes without even receiving a
message.

```ruby
actor = Concurrent::ProcessingActor.act(an_argument = 2) do |actor, number|
  number ** 3
end
# => <#Concurrent::ProcessingActor:0x7fcd84bcefe0 termination:pending>
actor.termination.value!                 # => 8
```
Let's receive some messages though.

```ruby
add_2_messages = Concurrent::ProcessingActor.act do |actor|
  # Receive two messages then terminate normally with the sum.
  (actor.receive & actor.receive).then do |a, b|
    a + b
  end
end
add_2_messages.tell 1
# => <#Concurrent::Promises::Future:0x7fcd84baceb8 pending>
add_2_messages.termination.resolved?     # => false
add_2_messages.tell 3
# => <#Concurrent::Promises::Future:0x7fcd84ba6450 pending>
add_2_messages.termination.value!        # => 4
```

Actors can also be used to apply back pressure to a producer. Let's start by
defining an actor which a mailbox of size 2.

```ruby
slow_counter = -> (actor, count) do
  actor.receive.then do |command, number|
    sleep 0.1
    case command
    when :add
      slow_counter.call actor, count + number
    when :done
      # terminate
      count
    end
  end
end

actor = Concurrent::ProcessingActor.act_listening( 
    Concurrent::Promises::Channel.new(2), 
    0,
    &slow_counter)
# => <#Concurrent::ProcessingActor:0x7fcd84b7d640 termination:pending>
```

Now we can create a producer which will push messages only when there is a
space available in the mailbox. We use promises to free a thread during waiting
on a free space in the mailbox.

```ruby
produce = -> receiver, i do
  if i < 10
    receiver.
        # send a message to the actor, resolves only after the message is 
        # accepted by the actor's mailbox
        tell([:add, i]).
        # send incremented message when the above message is accepted 
        then(i+1, &produce)
  else
    receiver.tell(:done)
    # do not continue 
  end
end

Concurrent::Promises.future(actor, 0, &produce).run.wait!
# => <#Concurrent::Promises::Future:0x7fcd84b4fab0 fulfilled>

actor.termination.value!                 # => 45
```


# Use-cases

## Simple background processing
  
```ruby
Concurrent::Promises.future { do_stuff }
# => <#Concurrent::Promises::Future:0x7fcd84aecc30 pending>
```

## Parallel background processing

```ruby
tasks = 4.times.map { |i| Concurrent::Promises.future(i) { |i| i*2 } }
# => [<#Concurrent::Promises::Future:0x7fcd84adec70 pending>,
#     <#Concurrent::Promises::Future:0x7fcd84addf00 pending>,
#     <#Concurrent::Promises::Future:0x7fcd84ad7a38 pending>,
#     <#Concurrent::Promises::Future:0x7fcd84ad6fc0 pending>]
Concurrent::Promises.zip(*tasks).value!
# => [0, 2, 4, 6]
```

## Actor background processing

Actors are mainly keep and isolate state, they should stay responsive not being
blocked by a longer running computations. It desirable to offload the work to
stateless promises.

Lets define an actor which will process jobs, while staying responsive, and
tracking the number of tasks being processed.

```ruby
class Computer < Concurrent::Actor::RestartingContext
  def initialize
    super()
    @jobs = {}
  end

  def on_message(msg)
    command, *args = msg
    case command
    # new job to process
    when :run
      job        = args[0]
      @jobs[job] = envelope.future
      # Process asynchronously and send message back when done.
      Concurrent::Promises.future(&job).chain(job) do |fulfilled, value, reason, job|
        self.tell [:done, job, fulfilled, value, reason]
      end
      # Do not make return value of this method to be answer of this message.
      # We are answering later in :done by resolving the future kept in @jobs.
      Concurrent::Actor::Behaviour::MESSAGE_PROCESSED
    when :done
      job, fulfilled, value, reason = *args
      future                        = @jobs.delete job
      # Answer the job's result.
      future.resolve fulfilled, value, reason
    when :status
      { running_jobs: @jobs.size }
    else
      # Continue to fail with unknown message.
      pass 
    end
  end
end
```

Create the computer actor and send it 3 jobs.

```ruby
computer = Concurrent::Actor.spawn Computer, :computer
# => #<Concurrent::Actor::Reference:0x7fcd84a3deb0 /computer (Computer)>
results = 3.times.map { computer.ask [:run, -> { sleep 0.1; :result }] }
# => [<#Concurrent::Promises::Future:0x7fcd849e0030 pending>,
#     <#Concurrent::Promises::Future:0x7fcd849d80b0 pending>,
#     <#Concurrent::Promises::Future:0x7fcd849d1418 pending>]
computer.ask(:status).value!             # => {:running_jobs=>3}
results.map(&:value!)                    # => [:result, :result, :result]
```
## Solving the Thread count limit by thread simulation

Sometimes an application requires to process a lot of tasks concurrently. If
the number of concurrent tasks is high enough than it is not possible to create
a Thread for each of them. A partially satisfactory solution could be to use
Fibers, but that solution locks the application on MRI since other Ruby
implementations are using threads for each Fiber.

This library provides a {Concurrent::Promises::Future#run} method on a future
to simulate threads without actually accepting one all the time. The run method
is similar to {Concurrent::Promises::Future#flat} but it will keep flattening
until it's fulfilled with non future value, then the value is taken as a result
of the process simulated by `run`.

```ruby
body = lambda do |v|
  # Some computation step of the process    
  new_v = v + 1
  # Is the process finished?
  if new_v < 5
    # Continue computing with new value, does not have to be recursive.
    # It just has to return a future.
    Concurrent::Promises.future(new_v, &body)
  else
    # The process is finished, fulfill the final value with `new_v`.
    new_v
  end
end
Concurrent::Promises.future(0, &body).run.value! # => 5
```

This solution works well an any Ruby implementation.

> *TODO: More examples to be added.*

## Cancellation

### Simple

Lets have two processes which will count until cancelled.

```ruby
source, token = Concurrent::Cancellation.create
# => [<#Concurrent::Cancellation:0x7fcd848b9260 canceled:false>,
#     <#Concurrent::Cancellation::Token:0x7fcd848b3d38 canceled:false>]

count_until_cancelled = -> token, count do
  if token.canceled?
    count
  else
    Concurrent::Promises.future token, count+1, &count_until_cancelled
  end
end 

futures = Array.new(2) do
  Concurrent::Promises.future(token, 0, &count_until_cancelled).run
end
# => [<#Concurrent::Promises::Future:0x7fcd84880370 pending>,
#     <#Concurrent::Promises::Future:0x7fcd848737b0 pending>]

sleep 0.01 
source.cancel                            # => true
futures.map(&:value!)                    # => [83, 79]
```

Cancellation can also be used as event or future to log or plan re-execution.

```ruby
token.to_event.chain do
  # log cancellation
  # plane re-execution
end
```

### Parallel background processing with cancellation

Each task tries to count to 1000 but there is a randomly failing test. The
tasks share a cancellation, when one of them fails it cancels the others.

```ruby
source, token = Concurrent::Cancellation.create
# => [<#Concurrent::Cancellation:0x7fcd84b15428 canceled:false>,
#     <#Concurrent::Cancellation::Token:0x7fcd84b14f50 canceled:false>]
tasks = 4.times.map do |i|
  Concurrent::Promises.future(source, token, i) do |source, token, i|
    count = 0
    1000.times do
      break count = :cancelled if token.canceled?
      count += 1
      sleep 0.01
      if rand > 0.95
        source.cancel
        raise 'random error'
      end
      count
    end
  end
end
# => [<#Concurrent::Promises::Future:0x7fcd84b0c0f8 pending>,
#     <#Concurrent::Promises::Future:0x7fcd84b06c20 pending>,
#     <#Concurrent::Promises::Future:0x7fcd84b055f0 pending>,
#     <#Concurrent::Promises::Future:0x7fcd84b043a8 pending>]
Concurrent::Promises.zip(*tasks).result
# => [false,
#     [:cancelled, nil, :cancelled, :cancelled],
#     [nil, #<RuntimeError: random error>, nil, nil]]
```

Without the randomly failing part it produces following.

```ruby
source, token = Concurrent::Cancellation.create
# => [<#Concurrent::Cancellation:0x7fcd84ac43c0 canceled:false>,
#     <#Concurrent::Cancellation::Token:0x7fcd84abfd98 canceled:false>]
tasks = 4.times.map do |i|
  Concurrent::Promises.future(source, token, i) do |source, token, i|
    count = 0
    1000.times do
      break count = :cancelled if token.canceled?
      count += 1
      # sleep 0.01
      # if rand > 0.95
      #   source.cancel
      #   raise 'random error'
      # end
    end
    count
  end
end
Concurrent::Promises.zip(*tasks).result
# => [true, [1000, 1000, 1000, 1000], nil]
```

## Throttling concurrency

By creating an actor managing the resource we can control how many threads is
accessing the resource. In this case one at the time.

```ruby
data      = Array.new(10) { |i| '*' * i }
# => ["",
#     "*",
#     "**",
#     "***",
#     "****",
#     "*****",
#     "******",
#     "*******",
#     "********",
#     "*********"]
DB = Concurrent::Actor::Utils::AdHoc.spawn :db, data do |data|
  lambda do |message|
    # pretending that this queries a DB
    data[message]
  end
end

concurrent_jobs = 11.times.map do |v|
  DB.
      # ask the DB with the `v`, only one at the time, rest is parallel
      ask(v).
      # get size of the string, rejects for 11
      then(&:size).
      # translate error to a value (message of the exception)
      rescue { |reason| reason.message } 
end 

Concurrent::Promises.zip(*concurrent_jobs).value!
# => [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, "undefined method `size' for nil:NilClass"]
```

Often there is more then one DB connections, then the pool can be used.

```ruby
pool_size = 5                            # => 5

DB_POOL = Concurrent::Actor::Utils::Pool.spawn!('DB-pool', pool_size) do |index|
  # DB connection constructor
  Concurrent::Actor::Utils::AdHoc.spawn(
      name: "connection-#{index}", 
      args: [data]) do |data|
    lambda do |message|
      # pretending that this queries a DB
      data[message]
    end
  end
end

concurrent_jobs = 11.times.map do |v|
  DB_POOL.
      # ask the DB with the `v`, only one at the time, rest is parallel
      ask(v).
      # get size of the string, rejects for 11
      then(&:size).
      # translate error to a value (message of the exception)
      rescue { |reason| reason.message } 
end 

Concurrent::Promises.zip(*concurrent_jobs).value!
# => [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, "undefined method `size' for nil:NilClass"]
```

In other cases the DB adapter maintains its internal connection pool and we
just need to limit concurrent access to the DB's API to avoid the calls being
blocked.

Lets pretend that the `#[]` method on `DB_INTERNAL_POOL` is using the internal
pool of size 3. We create throttle with the same size

```ruby
DB_INTERNAL_POOL = Concurrent::Array.new data 
# => ["",
#     "*",
#     "**",
#     "***",
#     "****",
#     "*****",
#     "******",
#     "*******",
#     "********",
#     "*********"]

max_tree = Concurrent::Throttle.new 3
# => <#Concurrent::Throttle:0x7fcd84bfcc10 limit:3 can_run:3>

futures = 11.times.map do |i|
  max_tree.
      # throttled tasks, at most 3 simultaneous calls of [] on the database
      throttled_future { DB_INTERNAL_POOL[i] }.
      # un-throttled tasks, unlimited concurrency
      then { |starts| starts.size }.
      rescue { |reason| reason.message }
end 

futures.map(&:value!)
# => [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, "undefined method `size' for nil:NilClass"]
```

## Long stream of tasks, applying back pressure

Lets assume that we queuing an API for a data and the queries can be faster
than we are able to process them. This example shows how to use channel as a
buffer and how to apply back pressure to slow down the queries. 

```ruby
require 'json' 

channel       = Concurrent::Promises::Channel.new 6
# => <#Concurrent::Promises::Channel:0x7fcd84a2d6c8 size:6>
source, token = Concurrent::Cancellation.create
# => [<#Concurrent::Cancellation:0x7fcd84a08080 canceled:false>,
#     <#Concurrent::Cancellation::Token:0x7fcd84a035f8 canceled:false>]

def query_random_text(token, channel)
  Concurrent::Promises.future do
    # for simplicity the query is omitted
    # url = 'some api'
    # Net::HTTP.get(URI(url))
    sleep 0.1
    { 'message' => 
        'Lorem ipsum rhoncus scelerisque vulputate diam inceptos' 
    }.to_json
  end.then(token) do |value, token|
    # The push to channel is fulfilled only after the message is successfully
    # published to the channel, therefore it will not continue querying until 
    # current message is pushed.
    channel.push(value) | 
        # It could wait on the push indefinitely if the token is not checked 
        # here with `or` (the pipe).
        token.to_future
  end.flat_future.then(token) do |_, token|
    # query again after the message is pushed to buffer
    query_random_text(token, channel) unless token.canceled?
  end
end

words          = []                      # => []
words_throttle = Concurrent::Throttle.new 1
# => <#Concurrent::Throttle:0x7fcd849d0400 limit:1 can_run:1>

def count_words_in_random_text(token, channel, words, words_throttle)
  channel.pop.then do |response|
    string = JSON.load(response)['message']
    # processing is slower than querying
    sleep 0.2
    words_count = string.scan(/\w+/).size
  end.then_throttled_by(words_throttle, words) do |words_count, words|
    # safe since throttled to only 1 task at a time
    words << words_count
  end.then(token) do |_, token|
    # count words in next message
    unless token.canceled?
      count_words_in_random_text(token, channel, words, words_throttle)
    end
  end
end

query_processes = 3.times.map do
  Concurrent::Promises.future(token, channel, &method(:query_random_text)).run
end
# => [<#Concurrent::Promises::Future:0x7fcd849a8180 pending>,
#     <#Concurrent::Promises::Future:0x7fcd84991f98 pending>,
#     <#Concurrent::Promises::Future:0x7fcd84981080 pending>]

word_counter_processes = 2.times.map do
  Concurrent::Promises.future(token, channel, words, words_throttle, 
      &method(:count_words_in_random_text)).run
end
# => [<#Concurrent::Promises::Future:0x7fcd8496acb8 pending>,
#     <#Concurrent::Promises::Future:0x7fcd8495bce0 pending>]

sleep 0.5                                # => 1
```

Let it run for a while then cancel it and ensure that the runs all fulfilled
(therefore ended) after the cancellation. Finally print the result.

```ruby
source.cancel                            # => true
query_processes.map(&:wait!) 
# => [<#Concurrent::Promises::Future:0x7fcd849a8180 fulfilled>,
#     <#Concurrent::Promises::Future:0x7fcd84991f98 fulfilled>,
#     <#Concurrent::Promises::Future:0x7fcd84981080 fulfilled>]
word_counter_processes.map(&:wait!)
# => [<#Concurrent::Promises::Future:0x7fcd8496acb8 fulfilled>,
#     <#Concurrent::Promises::Future:0x7fcd8495bce0 fulfilled>]
words                                    # => [7, 7, 7, 7]
```

Compared to using threads directly this is highly configurable and compostable
solution.


## Periodic task

By combining `schedule`, `run` and `Cancellation` periodically executed task
can be easily created.

```ruby
repeating_scheduled_task = -> interval, token, task do
  Concurrent::Promises.
      # Schedule the task.
      schedule(interval, token, &task).
      # If successful schedule again. 
      # Alternatively use chain to schedule always.
      then { repeating_scheduled_task.call(interval, token, task) }
end

cancellation, token = Concurrent::Cancellation.create
# => [<#Concurrent::Cancellation:0x7fcd84bfdb88 canceled:false>,
#     <#Concurrent::Cancellation::Token:0x7fcd84bfd6b0 canceled:false>]

task = -> token do
  5.times do
    token.raise_if_canceled
    # do stuff
    print '.'
    sleep 0.01
  end
end

result = Concurrent::Promises.future(0.1, token, task, &repeating_scheduled_task).run
# => <#Concurrent::Promises::Future:0x7fcd84bec2c0 pending>
sleep 0.2                                # => 0
cancellation.cancel                      # => true
result.result
# => [false,
#     nil,
#     #<Concurrent::CancelledOperationError: Concurrent::CancelledOperationError>]
```

