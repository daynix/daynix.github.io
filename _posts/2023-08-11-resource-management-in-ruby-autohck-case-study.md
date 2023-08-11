---
layout: post
title:  "Resource Management in Ruby - AutoHCK Case Study"
date:   2023-08-11 20:00:00 +0900
author:
  name: Akihiko Odaki
  url: https://github.com/akihikodaki
---

{% include 2023-08-11-resource-management-in-ruby-autohck-case-study/ruby.svg %}

## Resource management is a solved problem in Ruby... or is it?

In C language, you need to call `free()` whenever an allocated memory is no
longer used. If you fail to call `free()`, you will get out of memory. If you
try to use memory already freed, it results in a hard-to-debug bug and security
vulnerabilities. And the code gets cluttered with so frequent `free()` calls.

But such `free()` calls are unnecessary on Ruby. Ruby has a garbage collector so
the memory associated with an object that is no longer referenced by anybody
will be freed automatically. Resource management is a solved problem in Ruby...
or is it?

Unfortunately, the garbage collector only works with resources managed by Ruby.
A notable example of resources *not* managed by Ruby is an external process
created with `spawn`. In this post, we discuss why the garbage collector is
*not* a one-size-fits-for-all approach, and how to *safely* manage resources
without the garbage collector by illustrating
[a new resource management mechanism I'm proposing for our Ruby product, AutoHCK](https://github.com/HCK-CI/AutoHCK/pull/227).

## What is AutoHCK?

[AutoHCK](https://github.com/HCK-CI/AutoHCK) is a tool to automate Windows driver
testing. AutoHCK runs
[the Windows Hardware Lab Kit (Windows HLK)](https://learn.microsoft.com/en-us/windows-hardware/test/hlk/),
a testing framework for Windows drivers, on QEMU virtual machines. It is
particularly useful when you want to test with para-virtualized devices
implemented in QEMU or when you want to run Windows HLK on a Linux host.

AutoHCK is responsible to prepare files required by test runs, to manage virtual
machines, and to collect test results. These features are implemented in Ruby.

## Problem with garbage collector

As described earlier, resource management is not a solved problem even for Ruby,
which comes with a garbage collector.

The problem with a garbage collector is that it is highly _asynchronous_. Its
asynchronous nature benefits when managing pure-Ruby objects. For example,
consider of freeing a long linked list of Ruby objects:

~~~ Ruby
S = Struct.new(:next)
l = 10000000.times.reduce(S.new) { |sum| S.new(sum) }
~~~

If you naively try to free something like `l` on a platform that lacks garbage
collector (e.g. C++ and Rust), it will freeze the process for a while as it will
need 10 million objects at once, but Ruby can do that when it is idle and avoid
freezing.

<fig style="text-align: center">
{% comment %}
~~~ mermaid
sequenceDiagram
    Note over Application: Delete reference to "l"
    Note over Application: Free an object
    Note over Application: Free an object
    Note over Application: Free an object
    Note over Application: Free an object
    Note over Application: ...
~~~
{% endcomment %}
{% include 2023-08-11-resource-management-in-ruby-autohck-case-study/synchronous.svg %}
<figcaption>Synchronous resource deallocation (naive C++/Rust)</figcaption>
</fig>

<fig style="text-align: center">
{% comment %}
~~~ mermaid
sequenceDiagram
    Note over Application: Delete reference to "l"
    activate Application
    Note over Application: Start another useful work...
    Garbage Collector->>Application: Inspect memory
    Application->>Garbage Collector: Retrieve stale "l"
    Note over Garbage Collector: Free an object
    Note over Garbage Collector: Free an object
    Note over Garbage Collector: Free an object
    Note over Garbage Collector: Free an object
    Note over Garbage Collector: ...
    deactivate Application
~~~
{% endcomment %}
{% include 2023-08-11-resource-management-in-ruby-autohck-case-study/asynchronous.svg %}
<figcaption>
Asynchronous resource deallocation by Ruby garbage collector
</figcaption>
</fig>

However, it relies on the fact that there is plenty of resources available;
nobody cares even if the garbage collector increases the memory usage by a few
megabytes. It is untrue for scarce resources. Such an example in AutoHCK is
access to a disk image. AutoHCK sometimes performs a sequential run of two QEMU
processes using a disk image; the earlier one sets up a virtual machine and the
later one does the actual work. It will corrupt the disk image if the later
reads it while the earlier is still writing. This means only one virtual machine
can use a disk image at a time so access to the image is very scarce. In such a
case, _synchronous_ resource management is mandatory, and a garbage collector is
a poor option for that.

<fig style="text-align: center">
{% comment %}
~~~ mermaid
sequenceDiagram
    Note over Application: Finishes the setup process (but doesn't stop it!)
    rect rgb(249, 117, 131)
        activate Application
        Note over Application: Start the test process...
        Garbage Collector->>Application: Inspect memory
        Note over Application, Garbage Collector: Concurrent disk image accesses!
        Application->>Garbage Collector: Retrieve finished the setup process
        Note over Garbage Collector: Stop the setup process
        deactivate Application
    end
~~~
{% endcomment %}
{% include 2023-08-11-resource-management-in-ruby-autohck-case-study/hazard.svg %}
<figcaption>
Asynchronous resource deallocation with mutually exclusive access hazard (red)
</figcaption>
</fig>

## Alternatives

As shown above, the garbage collector is not a silver bullet. This section
presents alternative options for resource management.

### Reference counting

The idea behind reference counting is simple; remember how many references are
out there and release the resource when the count reaches zero. The resource
deallocation timing in a reference counting system is more predictable than in
the garbage collector as the deallocation happens exactly when the references
are deleted.

The obvious downside is that a stale reference may keep a resource occupied. In
the earlier example of a sequential run of two QEMU processes, if the earlier
process keeps the stale reference to the disk image, the later process will
never have access to the image, which makes AutoHCK hang. In such a case, you
will want to mark the stale reference invalid instead of waiting forever till it
disappears.

### Resource scope

The resource scope is an inherent concept in many programming languages. In such
languages, a variable becomes available when the execution enters a _scope_, and
it will be released when the execution leaves the scope.

The resource scope mechanism works neatly when the execution does not involve
persistent states. For example, if a system works like a mathematical function
that simply computes the output from the input, obviously intermediate values
computed in the system can be released after the execution.

The mechanism will not work well for applications like daemon. For example,
consider a key-value storage that accepts _set_ and _get_ operations. A pair of
a key and a value needs to persist after exiting the scope of the _set_
operation. In other words, the pair of a key and a value needs to _escape_ the
scope.

## Introducing AutoHCK::ResourceScope

So how can we complement the garbage collector for resource management in
AutoHCK? Determining the best method for resource management requires an
understanding of the workload. In the case of AutoHCK, there are two important
facts:

- AutoHCK is not a daemon but more function-like. It consumes a Windows image
  and test configurations, outputs the test results, and terminates.

- AutoHCK has essential resources that require mutually-exclusive access:
  disk images. Disk images are needed for virtual machines and virtual machines
  are needed for... literally everything AutoHCK does.

They make a clear case for introducing a resource management mechanism using
scope.

### The interface

Below is an example usage of `AutoHCK::ResourceScope`:

~~~ Ruby
ResourceScope.open do |scope|
  a = File.new('a.txt')
  scope << a
  a.read
end
~~~

This code opens a file, reads it, and closes it. `ResourceScope` can be applied
to any resources that can be freed by calling its `close` method.

For this simple example,
[you may just give a block to `File.open`](https://docs.ruby-lang.org/en/3.2/File.html#method-c-open)
to get the identical result, but the strength of `AutoHCK::ResourceScope` is
that it is _composable_.

The composability relies on the fact that it can be nested.

~~~ Ruby
ResourceScope.open do |a|
  file = ResourceScope.open do |b|
    path_file = File.new('path.txt')
    b << path_file
    File.new(path_file.read)
  end

  a << file
  file.read
end
~~~

This way, you can have function-like structures nested. Moreover,
`ResourceScope` has a special method named `transaction`:

~~~ Ruby
ResourceScope.open do |a|
  do_something_with a.transaction do |b|
    file = File.new('a.txt')
    b << file
    raise 'unexpected header' if file.readline != 'magic'
    file.rewind
    file
  end
end
~~~

Resources attached to a transaction scope will be freed if an exception occurs.
Otherwise, they will escape from the transaction scope to the original scope.
It is known as _move_ semantics in C++ and Rust. This makes sure resources will
not leak due to exceptions during resource allocation.

Finally, a scope object can be passed to methods.

~~~ Ruby
class C
  def new(scope)
    @file = File.new('a.txt')
    scope << @file
  end

  def read
    @file.read
  end
end

ResourceScope.open do |scope|
  c = C.new(scope)
  c.read
end
~~~

A scope parameter of a method forces the programmer to consider what scope is
appropriate to manage the allocated resource.

Nested scopes, transactions, and scope parameters make it possible to compose
a complex procedure from simple ones involving resource allocation and usage.

### Interrupts

A major pitfall when managing resources without the garbage collector in Ruby is
interrupt handling. In Ruby, an interrupt can happen anywhere in the code so a
resource will leak if an interrupt happens after it is allocated and before it
is freed.

{% comment %}
~~~ mermaid
sequenceDiagram
    Note over Application: Start resource cleanup...
    activate Application
    Interrupt Handler->>Application: Deliver an interrupt
    Note over Application: Raise an interrupt
    deactivate Application
~~~
{% endcomment %}
{% include 2023-08-11-resource-management-in-ruby-autohck-case-study/unmasked.svg %}

`Thread.handle_interrupt` allows to mask interrupts but its use is tedious.
`AutoHCK::ResourceScope.open` automatically use the method and make it simple.

{% comment %}
~~~ mermaid
sequenceDiagram
    Note over Application: Mask AutoHCKInterrupt
    Note over Application: Start resource cleanup...
    activate Application
    Interrupt Handler->>Application: Deliver an interrupt
    Note over Application: Finish resource cleanup
    deactivate Application
    Note over Application: Unmask AutoHCKInterrupt
    Note over Application: Raise AutoHCKInterrupt
~~~
{% endcomment %}
{% include 2023-08-11-resource-management-in-ruby-autohck-case-study/masked.svg %}

Below is the actual implementation of `AutoHCK::ResourceScope.open`:

~~~ Ruby
def self.open(resources = [])
  scope = new(resources)
  Thread.handle_interrupt(AutoHCKInterrupt => :never) do
    Thread.handle_interrupt(AutoHCKInterrupt => :on_blocking) do
      yield scope
    end
  ensure
    resources.reverse_each(&:close)
  end
end
~~~

`Thread.handle_interrupt(AutoHCKInterrupt => :never)` ensures an interrupt will
not occur when freeing resources.

`Thread.handle_interrupt(AutoHCKInterrupt => :on_blocking)` ensures an interrupt
will not occur after a resource is allocated and before it is assigned to the
scope.

`AutoHCK::ResourceScope.open` use a specialized exception class
(`AutoHCKInterrupt`) so that third-party code that creates a child thread and
interrupt it works. A typical example is
[timeout](https://github.com/ruby/timeout). As timeout works by creating a
thread and interrupting it after a specified period, masking generic interrupts
prevents it from functioning.

{% comment %}
~~~ mermaid
sequenceDiagram
    Note over Application: Mask all interrupts
    Note over Application: Start resource cleanup...
    activate Application
    Application->>Timeout Thread: Create a thread
    Application->>Timer: Arm
    Note over Timeout Thread: Start a task
    activate Timeout Thread
    Note over Timeout Thread: Hang!
    Timer->>Application: Fire
    Application->>Timeout Thread: Deliver an interrupt
    Note over Timeout Thread: ...
    Note over Application, Timer: System hangs!
    deactivate Timeout Thread
~~~
{% endcomment %}
{% include 2023-08-11-resource-management-in-ruby-autohck-case-study/timeout.svg %}

This also means `AutoHCK::ResourceScope` cannot be modified in a thread that can
have an interrupt other than `AutoHCKInterrupt`.

## Conclusion

We discussed the resource scope as an alternative to the garbage collector, and
how to implement it in Ruby. There is no single universal way to manage
resources. For example, Chromium, a large program written in C++, uses all
of [a garbage collector](https://chromium.googlesource.com/chromium/src/+/master/third_party/blink/renderer/platform/heap/BlinkGCAPIReference.md),
[reference counting and resource scope](https://chromium.googlesource.com/chromium/src/+/master/base/memory/).
Sometimes you may find something missing in the programming platform and you may
need to implement it by yourself. In such a case, carefully design your own
solution and also watch out for platform-specific pitfalls.
