# Const

`const`, just like `atomic`, is a compilation-time modifier, which can be applied both to variables and their pointees. The modifier applies the constant-ness property to an entity. Essentially, this property restricts calling non-constant methods on neither the object itself nor its instance variables. An antonym of being constant is being `mutable`.

Here go some examples:

```ruby
const foo = 2
foo = 3 # Error, cannot reassign const variable
```

```ruby
const foo = [1, 2]
foo << 3 # Error, cannot call a mutable method on a const variable
```

```ruby
const foo = const [1, 2]
foo = 42 # Error, cannot reassign const variable
foo << 3 # Error, cannot call a mutable method on a const variable

bar = foo
bar << 3 # Error, because bar still references the const object
```

```ruby
foo = const [1, 2]
foo << 3 # Error, cannot call a mutable method on a const variable

bar = foo
bar << 3 # Still error

foo = 42 # Ok, because foo variable is not const itself
```

```ruby
const foo = [1, 2]
foo = 42 # Error, because foo variable is const
foo << 3 # Ok, the array is mutable itself

bar = foo
bar << 3 # Also ok
```

The constant-ness aims to help a developer to avoid occasionally changing a contextually constant value. However, it is absolutely possible to overcome the constant-ness property explicitly using the notorius `mutable` keyword.

Object methods can be declared as `const` which means that they're permitted to neither:

  * Call mutable instance methods on the object itself,
    unless the call is modified with the `mutable` modifier

    ```ruby
    class Foo
      # An implicitly mutable method.
      def bar
      end

      const def baz
        bar # Compilation-time error, because `#bar` is mutable
        (mutable)bar # Ok
        self.(mutable)bar # Ok
      end
    end
    ```

  * Re-assign instance variables, unless they (instance variables) are marked `mutable`

    ```ruby
    class Foo
      @bar : String

      const def baz
        @bar = "baz" # Compilation-time error
        @bar.(mutable)=("baz") # Ok
        @bar.(mutable) = "baz" # Ok
      end
    end
    ```

    ```ruby
    class Foo
      mutable @bar : String

      const def baz
        @bar = "baz" # Ok
      end
    end
    ```

  * Call mutable methods on instance variables, unless their pointees are marked `mutable`

    ```ruby
    class Foo
      @bar : Array(Int32)

      const def baz
        @bar[0] *= 2 # Compilation-time error
        @bar.(mutable)[0] = @bar[0] * 2 # Ok
      end
    end
    ```

    ```ruby
    class Foo
      @bar : mutable Array(Int32)

      const def baz
        @bar[0] *= 2 # Ok
      end
    end
    ```

Just like with `atomic`, a const method would be called on a mutable object if its mutable variant is absent. Likewise, an whole object can be marked `const`, so its method are implicitly const by default, carrying the same restrictions. However, `atomic` does not imply `const`!

Function arguments can be declared `const` as well, which would imply constant-ness in the current scope only:

```ruby
def foo(const ary : Array(Number))
  ary << 43 # Compilation-time error, cannot call mutable method const array
  ary.(mutable) << 43 # Would work
end

def bar(const ary : Array(Number))
  # Do nothing mutable; ary is a constant here
end

ary = [1, 2]
ary << 42 # Ok
bar(ary) # The array would be constant within the method
ary << 44 # Also ok, the array is still mutable outside
```

However, you cannot imply an explicit mutability to a const argument:

```ruby
def foo(ary : Array(Number))
  ary << 43
end

def bar(mutable ary : Array(Number))
  ary << 44
end

ary = const [1, 2]
foo(ary) # Error, because foo sees a const array, so << cannot be called on it
bar(ary) # Error, because bar expects mutable arrays only
bar(mutable ary) # Ok
```
