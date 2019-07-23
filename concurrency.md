# Concurrency

Onyx has coroutines model inspired by Go and Crystal. This implies absence of explicit thread control in user programs. Any coroutine spawned can be stealed by another thread unless declared with the `local` keyword.

`async` and `await` keywords complement coroutines to build control flows conveniently.

There can be race conditions in user programs. To avoid them, concepts of mutexes and atomics exist.

## Couroutines

Coroutines are also called *fibers*. Affected local variable would be promoted to heap upon spawning a fiber. A fiber can be stealed by another thread. Unlike as in Crystal, a fiber is spawned using the `async` keyword.

```ruby
a = 3
&a.stack? # => true

async do
  a = 4
end

&a.stack? # => false
puts a # => 3

yield # Yield the control allowing the same or another thread execute the coroutine

puts a # => 4
```

As mentioned above, you can restrict a fiber to run within the same thread only using the `local` keyword:

```ruby
local async do
  # Will be executed within the same thread, granting some race guarantees
end
```

`yield` is guaranteed to wait exactly until all the previously spawned fibers in this very thread have yielded or ended — in the same order.

Just like in Javascript, an `async` call returns a `Promise(T)`, which would be *resolved* at some moment in the future. Calling the `#await` method on a promise (or using the `await` keyword, which is kind of sugar here) would block until the promise is resolved and return its return value.

```ruby
class HTTP::Client
  async static def get(url : String)
    await @socket.gets_to_end # Example
  end

  # A sync overload
  static def get(url : String)
    @socket.gets_to_end
  end
end

promise = HTTP::Client.get("https://google.com")
puts promise.type # Promise(String)

response = await promise # Or `promise.await`; would block the thread
puts response
```

## Select

The `select` operator can be used to wait for the **first** promise resolved from the list:

```ruby
select
when response = HTTP::Client.get("https://google.com")
  puts "Google won!"
when response = HTTP::Client.get("https://yahoo.com")
  puts "Yahoo won!"
when Timer.new(30.seconds) # async static def new(time : Time::Span); sleep(time); end
  raise "Timeout!"
end
```

## Parallel assignment

Sweet sugar coming straight from ES6:

```ruby
google, yahoo = await HTTP::Client.get("https://google.com"), HTTP::Client.get("https://yahoo.com")
```

```ruby
promises = [HTTP::Client.get("https://google.com"), HTTP::Client.get("https://yahoo.com")]
google, yahoo = await promises # A bit unsafer in the runtime (i.e. array size mismatch)
```

## Mutexes

To deal with race conditions, a developer can use mutexes. MutEx means Mutual Exclusion — it guarantees that only the current thread would access something. Essentially, to proceed, a thread has to acquire a mutex lock. If another thread is currently holding this lock, then the current one has to wait unless the lock is released.

```ruby
mutex = Mutex.new
a = 42

2.times do
  async do
    # Lock guarantees that in both cases
    # the result would be equal to 42
    mutex.lock do
      a += 5
      a -= 5
      puts a
    end
  end
end
```

## Atomics

A complementary way to deal with race conditions is to mark a variable or an object as `atomic`. Contextually this means that all operations applied to this object are guaranteed to be atomic (*sic!*). Depending on the type, such operations are usually expected to return the former (meaningful) value.

Just like with the `const` keyword, an `atomic` method overload is attempted to be called on an atomic object, raising in compilation-time if not found. A non-atomic method is called `volatile`. An atomic method would still be called for a non-atomic object if volatile alternative is absent. However, you can explicitly call a volatile method if you apply the `volatile` modifier for the call (e.g. `@buffer.(volatile)push(x)`).

Within methods of an atomic object, the atomicity property is forwarded to instance variables method calls, recursively.

Atomicity is a compiler feature. It only affects which methods are called in which scope. The atomicity property is not stored anywhere in the runtime. It does not affect the size of runtime objects.

Always remember that atomicity usually has a performance cost depending on the implementation. You should carefully choose between racing guarantees and speed.

Here goes an example of an atomic variable:

```ruby
atomic a = 42

2.times do
  async do
    puts(a += 1) # Puts "42", then "43"
  end
end

yield

puts a # => 44
```

An atomic operation on a numeric variable returns its old value. Therefore, `a += 1` is expected to be a volatile operation, because it expands to `a = a + 1`, which is not thread-safe? However, it's possible to overload the `+=` method. In this case, atomic numbers could (and do) overload this method to modify the variable pointer, still returning the old value. Another example demonstrating it:

```ruby
atomic a = 42

# A single atomic operation happens — the assignment.
# a + 1 returns 43 in non-atomic way,
# therefore it equals to a = 43, which is atomic op.
# It can be seen that the expression is not really atomic itself
b = (a = a + 1)
puts a # => 43
puts b # => 42

# In this case, the `+=` method is truly atomic
b = (a += 1)
puts a # => 44 (the new value)
puts b # => 43 (the old value)
```

Another example with a boolean variable:

```ruby
atomic b = true

c = (b = false)
puts c # => true (the old value)

c = (b = false)
puts c # => false
```

Apart from variables, objects can be marked atomic as well. However, neither primitves nor structs can have this property themselves, as they're copied by value. Otherwise, the atomicity property is acceptable for classes. You also can apply both `atomic` and `const` to a class instance, though it does not always make sense.

```ruby
a = atomic 42 # Numbers are passed by value, you cannot mark them as atomic
atomic a = 42 # Legit

struct Foo; property bar : String; end
f = atomic Foo.new("baz") # Nope
atomic f = Foo.new("baz") # Ok

b = atomic const [1, 2] # Doesn't make much sense in this context, but allowed
c = atomic const Stack.new(3) # Non-resizeable atomic stack — perfectly legit

b = atomic [1, 2]
b << 3 # An atomic push, which would atomically resize the array if needed

h = atomic {"foo" => "bar"}
h.delete("foo") # Atomically delete the "foo" key

n = atomic {foo: "bar"} # That's a named tuple, not a hash. Deny!
atomic n = {foo: "bar"} # Ok, but useless
```

You should always think that object modifiers are applied to the expression result itself, not the intermediate chain calls. You can imagine `atomic const` being some magic method call which applies these properties to its argument. In fact, you actually can wrap the expression in parenthesis:

```ruby
stack = atomic const (Stack.new(3))
```

Now it should be clear that the `new` method call is neither atomic nor const itself! Under the hood, the `volatile mutable static def new` is called. If you really need to call exactly the atomic `new`, use the call modifiers:

```ruby
stack = atomic const (Stack.(atomic const)new(3))
stack.push(42) # Atomic
```

That's a bit cumbersome, indeed, but for greater good. It also can be avoided, for example, by keeping in mind that marking the *variable* atomic would imply atomic calls everywhere. So the example above should practically be the same as:

```ruby
atomic const stack = Stack.new(3)
stack.push(42) # Still atomic

# Unless the stack object itself is assigned to another variable
foo = stack
foo.push(42) # Not atomic anymore
```

You can mix variable and its value atomicity (not applicable for neither primitives nor structs, as mentioned above):

```ruby
a = [1]
a.push(2) # Non-atomic operation

atomic b = atomic a
c = b # Copies the atomic reference to a

b.push(3) # Atomic operation
c.push(4) # Atomic operation

d = (b = [0])
puts d # [1, 2, 3, 4] (old value)
puts b # [0] (new value)

e = (c = [0])
puts e # [0] (new value, because c variable is not atomic)
puts c # [0] (new value)
```

```ruby
a = [1]

b = atomic a
c = b # Copies the atomic reference to a

b.push(2) # Atomic operation
c.push(3) # Atomic operation

d = (b = [0])
puts d # [0] (new value, because b variable is not atomic)
puts b # [0] (new value)

e = (c = [0])
puts e # [0] (new value)
puts c # [0] (new value)
```

```ruby
a = [1]

atomic b = a
c = b # Copies the non-atomic reference to a

b.push(2) # Atomic operation
c.push(3) # Non-atomic operation

d = (b = [0])
puts d # [1, 2, 3] (old value, because b variable is atomic)
puts b # [0] (new value)

e = (c = [0])
puts e # [0] (new value)
puts c # [0] (new value)
```

You can mark an object definition itself as `atomic` making it implicitly atomic (and therefore thread-safe) by default. A `Mutex` class itself is a good example. However, it is possible to define such an object as volatile:

```ruby
atomic class Mutex
  # Implicitly `atomic def lock`.
  def lock
  end
end

m1 = Mutex.new # Implicitly atomic
m1.lock # Atomic

m2 = volatile Mutex.new # Would try to call volatile methods from now
m2.lock # Lookup for `volatile def lookup`, but still call the atomic variant instead
```

Function arguments can be atomic as well, which would imply atomicity in current scope only. Once an argument is out of the function scope, it stops being atomic:

```ruby
ary = [0]

def fill(atomic ary : Array(Int32), number : UInt)
  number.times do
    async do
      ary.push(rand(100)) # Atomic push
    end
  end

  yield
end

ary << 1 # Non-atomic push
fill(ary)
ary << 2 # Also non-atomic push
```

Just a reminder, you can call an atomic method on a non-atomic object using the same `atomic` keyword as a call modifier:

```ruby
ary = [1, 2]
puts ary.(atomic)push # => [1, 2, 3]
```

### Stack example

Atomic stacks can be used as replacement for Crystal's `Channel(T)` class.

```ruby
# `const` is contextual constant-ness. In this case it means that Stack is of fixed size
stack = atomic const Stack(Int32).new(3)

3.times do |i|
  async do
    stack.push(i) # Thread-safe push (because stack is `atomic`)
  end
end

3.times do
  # `await` means to use the `async` `#pop` method
  value = await stack.pop # Thread-safe pop (because stack is `atomic`)
  puts value
end
```

```ruby
class Stack(T)
  # The instance variable of the buffer is always constant,
  # because you can not change the buffer of a stack.
  #
  # The buffer itself is always mutable, because we don't
  # want constant Array when the stack is constant.
  #
  # The atomicity of the `@buffer` variable is
  # inherited from the current stack atomicity.
  const @buffer : mutable Array(T)

  def initialize(capacity)
    @buffer = Array(T).new(capacity)
  end

  # Calling sync `#push` on a `const Stack` raises if it's currently full.
  atomic const def push(value : T)
    # `atomic def Array#lock?` is a mutex lock wrapper
    lock = @buffer.lock?(@buffer.size < @buffer.capacity)

    if lock
      # We've already aquired the buffer lock,
      # no need to call atomic push here;
      # using `volatile` modifier then.
      #
      # In fact, attempting to re-aquire the lock
      # would result in deadlock
      @buffer.(volatile)push(value)
      lock.release
    else
      raise "Is full"
    end
  end

  # Async variant waits for free space instead of raising.
  async atomic const def push(value : T)
    until lock = @buffer.lock?(@buffer.size < @buffer.capacity)
      yield
    end

    @buffer.(volatile)push(value)
    lock.release
  end

  # Implicitly volatile const push, which could lead to
  # unwanted resize in a multi-threaded programs.
  const def push(value : T)
    raise "Is full" if @buffer.size == @buffer.capacity
    @buffer.push(value)
  end

  # ditto
  async const def push(value : T)
    until @buffer.size == @buffer.capacity
      yield
    end

    @buffer.push(value)
  end

  # Mutable push implementation, allowing to resize if needed.
  # This definition covers both atomic and non-atomic variants.
  atomic def push(value : T) # The `mutable` modificator is implicit
    # An atomic or non-atomic `Array#push` is called
    # depending on the actual call atomicity
    @buffer.push(value)
  end

  # `#pop` doesn't contextually change the stack itself, so it's always const.
  atomic const def pop
    result = @buffer.pop?
    raise "Is empty" unless result
    return result
  end

  # Wait until a value is popped.
  async atomic const def pop
    until result = @buffer.pop?
      yield
    end

    return result
  end
end
```

## Threads

By default, there is no direct access to threads in an Onyx program. Parallelism is achieved via scheduler which distributes coroutines among the threads. Onyx has an abstract scheduler implementation, which is single-threaded by default.

To enable multi-threading, a user has to require a desired threaded scheduler implementation. Luckily, it is implicit in common toolchains (e.g. Ubuntu). Depending on the target and/or stdlib selected, a default threads implementation may be selected by the compiler.

However, a user is free to provide their own custom multi-threaded scheduler implementation if needed.

```ruby
# Explicit requirement in the code,
# which would rely on the `pthread.h` header,
# requiring the C standard library.
# It's implicit on most POSIX and also MinGW-w64 targets
require "threading/pthread"

# Implicit on Windows MSVC targets; relies upon the `win32thread.h` header
require "threading/win32thread"

# If want to use custom threading, an Onyx file is required,
# which may include C lib calls
require "./my/custom/threading"

# If want to explicitly disable threading
require "threading/none"
```

It is also possible to pass the `--threading` flag to the compiler to force-rewrite threading implementation. It can be `pthread`, `win32thread` or `none`. For custom implementations, it must be an object file in format understandable by the target:

```sh
$ onyx build main.nx --threading=./my-threading.elf.o
```
