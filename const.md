# Const

A variable can be defined as `const`, so it's not a *variable* anymore. Basically, a const variable can not be re-assigned and a const object (which the const variable implies) can not be called with a non-const method (it's not a strict requirement, though).

```ruby
foo = 2
foo = 3 # Ok

const bar = 2
bar = 3 # Compilation-time error

qux = [1, 2]
qux << 3 # Ok

const quux = [1, 2]
quux << 3 # Compilation-time error
```

You can mix variable and object constant-ness, which would mean different things:

```ruby
const foo = const [1, 2]
foo = 42 # Cannot reassign const variable
foo << 3 # Cannot call a non-const method on a const object
bar = foo
bar << 3 # Error, because bar references const object

foo = const [1, 2]
foo = 42 # Ok
foo << 3 # Error
bar = foo
bar << 3 # Error

const foo = [1, 2]
foo = 42 # Error
foo << 3 # Error
bar = foo
bar << 3 # Ok, because bar references [1, 2], which is non-const by itself
```

Object methods can be declared as `const` which means that they would not modify the object itself (including modifying both instance and static variables). It implies a restriction on calling another non-const methods implicitly from within it. However, it can be overcome with the explicit `variable` keyword.

```ruby
class Array
  const def [](index)
    get(index)
  end

  def []=(index, value)
    set(index, value)
  end

  const def poison(value)
    self.[rand] = value # Compilation-time error; []= method is non-const (i.e. variable)
    variable self.[rand] = value # Ok, if you really want to
  end
end
```

Function arguments can be declared `const` as well, which would imply constant-ness in the current scope only.

```ruby
def foo(const ary : Array(Number))
  ary << 43 # Compilation-time error
end

ary = [1, 2]
ary << 42 # Ok
foo(ary)
ary << 44 # Also Ok
```

You can call constant methods on variables using the same `const` keyword:

```ruby
ary = [1, 2]
const ary << 43 # Nope
ary << 44 # Ok
```

Nested objects constant-ness propagation example:

```ruby
class Post
  const @user : User
  getter user

  property content : String
end

class User
  property id : Int32
  property settings : Settings?
end

class Settings
  property foo : String
end

post = Post.new(
  user: User.new(
    id: 42,
    settings: Setting.new(
      foo: "bar"
    )
  ),
  content: "Hello World!")

post.content = "Bye World!" # Legit
post.user = User.new(id: 43) # Illegal, basically because there is no setter method
post.user.id = 43 # Still illegal, because @user is const, and `id =` is variable
variable post.user.id = 43 # Legal, but requires explicitness
post.user.settings.foo = "baz" # Ok, as we don't change the @user itself
```
