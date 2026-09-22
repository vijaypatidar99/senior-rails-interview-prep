# Senior Ruby on Rails Interview Prep Guide

A comprehensive, scroll-through interview handbook for a **Senior Ruby on Rails / Full-Stack Engineer** — Ruby and Rails internals, PostgreSQL, Redis, JavaScript, API design, system design, testing, Docker, AWS, DevOps, security, Git, and behavioral prep, all in one place.

This guide is deliberately **Rails-focused**. It intentionally skips MongoDB, React, Node/Express, and GraphQL internals — if your loop is a MERN stack instead of (or alongside) Rails, this isn't that guide.

**How to use this guide:** for every question, write your own answer in your own words *before* reading the one here. If you can't produce a real code example or a concrete story from a project you've actually built, that's your real study gap, not the question. Come back to the [Prep Checklist](#prep-checklist) at the end once you've been through a section, to self-check what actually stuck.

**Format:** most questions follow **Short Answer** (what you'd say in the first 10 seconds) → **Simple Explanation** (the reasoning behind it) → **Example** (real code). System design questions use **How to think about it** instead, since the point there is the reasoning process, not a memorized answer. Behavioral questions use **Framework** + **What a strong answer covers**, since nobody can hand you your own professional experience — use these to structure your *real* stories, not to memorize someone else's.

---

## Table of Contents

- [Ruby](#ruby) — 38 questions
- [Ruby on Rails](#ruby-on-rails) — 72 questions
- [PostgreSQL / SQL](#postgresql--sql) — 37 questions
- [Redis](#redis) — 16 questions
- [JavaScript](#javascript) — 21 questions
- [APIs](#apis) — 26 questions
- [System Design](#system-design) — 31 questions
- [Testing / RSpec](#testing--rspec) — 30 questions
- [Docker](#docker) — 19 questions
- [AWS / Cloud](#aws--cloud) — 22 questions
- [DevOps / CI-CD](#devops--ci-cd) — 30 questions
- [Security](#security) — 30 questions
- [Git](#git) — 19 questions
- [Behavioral / Senior Engineer Questions](#behavioral--senior-engineer-questions) — 15 questions


---


## Ruby

**— Language Basics & OOP —**

### 1. What's the difference between a block, a Proc, and a lambda?

**Short Answer**

All three are "a chunk of code you can pass around", but they differ in two ways: how strict they are about arguments, and what `return` does.

- **Block** — code attached to a method call with `{ }` or `do...end`. It isn't an object on its own.
- **Proc** — a real object. Relaxed about arguments. `return` inside it returns from the method where the Proc was *written*.
- **Lambda** — also a Proc object, but strict about arguments, and `return` only exits the lambda itself.

**Simple Explanation**

Two differences matter in real code.

**1. Argument checking (arity).** A Proc (and a block) is forgiving. Pass too few arguments and the missing ones become `nil`. Pass too many and the extras are dropped. A lambda behaves like a normal method and raises `ArgumentError` if the count is wrong.

**2. What `return` does.** `return` in a lambda just exits the lambda, like a normal method. `return` in a Proc tries to return from the method where that Proc was defined. If that method already finished, you get a `LocalJumpError`.

Rule of thumb: use a lambda when you're passing a callback around, because it behaves like a normal method. Use blocks/Procs for `each`-style code where jumping out of the method is actually what you want.

**Example**

```ruby
my_proc   = Proc.new { |a, b, c| p [a, b, c] }   # lenient arity
my_lambda = lambda   { |a, b| p [a, b] }         # strict arity

my_proc.call(1, 2)          # => [1, 2, nil]  -- missing arg silently becomes nil
my_lambda.call(1, 2)        # => [1, 2]
my_lambda.call(1)           # raises ArgumentError: wrong number of arguments (given 1, expected 2)

def proc_return
  p = Proc.new { return 10 }  # `return` exits the ENCLOSING method (proc_return)
  p.call
  20 # never reached
end

def lambda_return
  l = lambda { return 10 }    # `return` exits only the lambda itself
  l.call
  20 # this line IS reached
end

proc_return   # => 10
lambda_return # => 20
```

### 2. How does `yield` work, and what is `block_given?` for?

**Short Answer**

`yield` runs the block that was passed to the method. `block_given?` tells you whether a block was actually passed, so you can avoid the `LocalJumpError` you'd get from calling `yield` with no block.

**Simple Explanation**

Every Ruby method can quietly accept a block, even if you never list it in the parameters. Inside the method, `yield` hands control to that block (optionally passing it values), then continues once the block finishes.

If nobody passed a block and you call `yield`, Ruby raises `LocalJumpError: no block given (yield)`.

`block_given?` lets you handle that case. A common pattern is to return an `Enumerator` instead (using `enum_for` / `to_enum`) when there's no block — that's exactly what Ruby's own `Array#each` does.

**Example**

```ruby
def my_each(array)
  return enum_for(:my_each, array) unless block_given?

  i = 0
  while i < array.length
    yield array[i]
    i += 1
  end
  array
end

my_each([1, 2, 3]) { |n| puts n * 2 }
# => 2
# => 4
# => 6

my_each([1, 2, 3])
# => #<Enumerator: ...>  -- no block, so block_given? was false and we returned an Enumerator
```

### 3. What's the difference between `attr_accessor`, `attr_reader`, and `attr_writer`?

**Short Answer**

They're shortcuts that write getter/setter methods for you:

- `attr_reader :name` → getter only
- `attr_writer :name` → setter only
- `attr_accessor :name` → both

**Simple Explanation**

Without them you'd hand-write this for every attribute:

```ruby
def name
  @name
end

def name=(value)
  @name = value
end
```

Pick the narrowest one that works. Using `attr_reader` on something that shouldn't change from outside says "read-only by design" without writing a comment. It's free, self-documenting encapsulation.

**Example**

```ruby
class Product
  attr_reader   :sku      # generates: def sku;  @sku;  end
  attr_writer   :price    # generates: def price=(v); @price = v; end
  attr_accessor :name     # generates both a reader and a writer for @name

  def initialize(sku, price, name)
    @sku, @price, @name = sku, price, name
  end
end

item = Product.new("SKU1", 10, "Widget")
item.name          # => "Widget"     (reader from attr_accessor)
item.price = 12     # works          (writer from attr_writer)
item.sku = "SKU2"   # NoMethodError: undefined method `sku=' -- attr_reader never defined a setter
```

### 4. What's the difference between `include`, `extend`, and `prepend`?

**Short Answer**

All three mix a module in, but they land in different places:

- `include` → adds instance methods *below* the class, so the class's own methods win on a name clash.
- `prepend` → adds methods *above* the class, so the module runs first and can call `super` to reach the class's version.
- `extend` → adds methods to a single object. Used inside a class body, it gives you class methods.

**Simple Explanation**

Think about method lookup order.

`include` is the everyday case. The module acts as a backup — if the class defines the same method, the class wins.

`prepend` flips that. The module intercepts the call first, and can call `super` to run the original. That's how you wrap or decorate existing behavior.

`extend` is different in kind. It doesn't touch instances at all — it adds methods to whatever single object you call it on. Since `self` inside a class body is the class itself, `extend SomeModule` there gives you class methods.

**Example**

```ruby
module Loud
  def speak
    super.upcase
  end
end

class Animal
  def speak = "..."
end

class Dog < Animal
  include Loud   # inserted BELOW Dog, ABOVE Animal
  def speak = "woof"
end

class Cat < Animal
  prepend Loud    # inserted ABOVE Cat -- Loud#speak runs first, can super into Cat#speak
  def speak = "meow"
end

Dog.ancestors   # => [Dog, Loud, Animal, Object, Kernel, BasicObject]
Cat.ancestors   # => [Loud, Cat, Animal, Object, Kernel, BasicObject]

Dog.new.speak   # => "woof"   -- Dog#speak wins; Loud is below Dog, never reached
Cat.new.speak   # => "MEOW"  -- Loud#speak runs first, calls super (Cat#speak → "meow"), upcases it

module MathHelpers
  def double(n) = n * 2
end

class Calculator
  extend MathHelpers   # adds `double` as a CLASS method, not an instance method
end

Calculator.double(5)      # => 10
Calculator.new.double(5)  # NoMethodError -- extend only affects the object it was called on
```

### 5. Walk through Ruby's method lookup / ancestor chain.

**Short Answer**

When you call a method, Ruby walks `object.class.ancestors` from left to right and runs the first match it finds. `Module#ancestors` shows you that exact search order.

**Simple Explanation**

Every class has an ordered list of "places to look": itself, any modules it mixed in, its superclass, and so on up to `BasicObject`. Method lookup is just a scan down that list that stops at the first hit.

One detail worth knowing cold: if you `include` two modules that define the same method, the **last one included wins**. Each new `include` gets inserted directly above the class, pushing the earlier ones further down the list.

Whenever a mixin "unexpectedly" wins, print `ancestors` — the answer is always in there.

**Example**

```ruby
module Walkable
  def move = "walks"
end

module Swimmable
  def move = "swims"
end

class Animal; end

class Duck < Animal
  include Walkable
  include Swimmable  # included LAST, so it sits closer to Duck than Walkable does
end

Duck.ancestors
# => [Duck, Swimmable, Walkable, Animal, Object, Kernel, BasicObject]

Duck.new.move
# => "swims"  -- Ruby scans ancestors left to right and stops at the first match
```

### 6. What are Ruby modules used for, besides namespacing?

**Short Answer**

Two main uses: namespacing (grouping classes/constants so names don't collide) and mixins (sharing behavior between unrelated classes without inheritance). Ruby's own `Comparable` and `Enumerable` are mixins.

**Simple Explanation**

A module can't be instantiated. It exists to group things.

**As a namespace:** `Api::V1::UsersController` and `Admin::UsersController` can both exist without clashing.

**As a mixin:** you write one method and get many for free.

- Define `<=>` and `include Comparable` → you get `<`, `>`, `==`, `between?`, `clamp`.
- Define `each` and `include Enumerable` → you get `map`, `select`, `reduce`, `sort`, and dozens more.

**Example**

```ruby
module Api
  module V1
    class UsersController; end   # full name Api::V1::UsersController avoids collisions
  end
end

class TemperatureReading
  include Comparable          # mixin: get <, >, ==, between?, clamp... for free

  attr_reader :degrees
  def initialize(degrees) = @degrees = degrees

  def <=>(other)
    degrees <=> other.degrees   # implement ONE method, Comparable derives the rest
  end
end

readings = [TemperatureReading.new(72), TemperatureReading.new(68)]
readings.min.degrees            # => 68
readings.first < readings.last  # => false
```

### 7. What are the four OOP pillars, and how does Ruby express them?

**Short Answer**

- **Encapsulation** — hide internal state behind a controlled interface.
- **Abstraction** — expose *what* an object does, not *how*.
- **Inheritance** — reuse behavior from a parent class.
- **Polymorphism** — the same method name behaves differently depending on the object.

Ruby supports all four, but prefers composition over deep inheritance trees.

**Simple Explanation**

These come up constantly because they map onto real design choices.

**Encapsulation** is just instance variables plus `attr_*` or custom methods controlling who can touch them.

**Abstraction** is a shared method name like `#area` hiding completely different implementations.

**Inheritance** is `class Circle < Shape`. It's easy to overuse. Once you need to mix and match behaviors across classes that don't have a clean "is-a" relationship, switch to composition — mix in a module, or hold a reference to a helper object.

**Polymorphism** is what lets `[circle, square].map(&:area)` work with no `case`/`when` on class names.

**Example**

```ruby
# Encapsulation: internal state (@balance) hidden behind a controlled public interface
class Account
  def initialize(balance) = @balance = balance
  def balance = @balance
  def withdraw(amount)
    raise ArgumentError, "insufficient funds" if amount > @balance
    @balance -= amount
  end
end

# Abstraction: callers depend on WHAT a shape can do (#area), not HOW each computes it
class Shape
  def area = raise NotImplementedError
end

class Circle < Shape
  def initialize(r) = @r = r
  def area = Math::PI * @r**2
end

class Square < Shape
  def initialize(s) = @s = s
  def area = @s**2
end

# Inheritance: Circle/Square reuse Shape's contract -- but prefer composition (mixing
# in modules, or holding a collaborator) once behavior needs mixing-and-matching
# rather than a strict "is-a" tree
class Rectangle < Shape
  def initialize(w, h) = (@w, @h = w, h)
  def area = @w * @h
end

# Polymorphism: same message (#area), different behavior per receiver, no branching
[Circle.new(2), Square.new(3), Rectangle.new(2, 5)].map(&:area)
# => [12.566370614359172, 9, 10]
```

### 8. What's actually falsy in Ruby?

**Short Answer**

Only `nil` and `false` are falsy. Everything else is truthy — including `0`, `""`, `[]`, and `{}`.

**Simple Explanation**

If you come from C, Python, or JavaScript this trips you up. In Ruby, `0` is truthy. An empty string is truthy. An empty array is truthy.

There's no "empty means false" shortcut. If you want that behavior, check for it explicitly with `array.empty?` or `string.empty?`.

**Example**

```ruby
puts "0 is truthy"    if 0    # prints -- unlike C, Python, or JS
puts "'' is truthy"   if ""   # prints
puts "[] is truthy"   if []   # prints

[nil, false, 0, "", [], {}].each { |v| puts "#{v.inspect}: #{!!v}" }
# nil: false
# false: false
# 0: true
# "": true
# []: true
# {}: true
```

### 9. What's the difference between `==`, `equal?`, and `eql?`?

**Short Answer**

Three different questions:

- `equal?` → "are these literally the same object in memory?"
- `==` → "do these represent the same value?" (classes override this)
- `eql?` → "same value **and** same type?" — this is what `Hash` uses for keys, along with `#hash`

**Simple Explanation**

`1 == 1.0` is `true` because `==` converts between numeric types. `1.eql?(1.0)` is `false` because `eql?` also cares about the type — Integer isn't Float.

The `eql?`/`hash` pair matters in practice: `Hash` lookups use `eql?`, not `==`. So if you want to use a custom class as a Hash key, you must override **both** `#eql?` and `#hash` consistently, or lookups will silently fail to find things.

**Example**

```ruby
a = "hello"
b = "hello"

a == b       # => true   -- value equality (String#== compares contents)
a.equal?(b)  # => false  -- two distinct String objects, different object_ids
a.eql?(b)    # => true   -- same value AND same type

1 == 1.0      # => true   -- == coerces across numeric types
1.eql?(1.0)   # => false  -- eql? also checks type: Integer vs Float
1.equal?(1)   # => true   -- small Integers are cached/interned in MRI

h = { 1 => "int" }
h[1.0]        # => nil -- Hash uses #eql?, and 1.eql?(1.0) is false, even though 1 == 1.0
```

### 10. What is duck typing, with a concrete example?

**Short Answer**

Duck typing means you call a method because the object *responds to it*, not because you checked its class. "If it quacks like a duck, treat it like a duck."

**Simple Explanation**

Instead of writing `if thing.is_a?(Duck)` before calling `#quack`, you just call `thing.quack` and trust that whatever was passed in implements it.

This is how idiomatic Ruby and Rails are written. It's why you can pass anything that responds to `#each` where an enumerable is expected, or anything that responds to `#call` (a Proc, a lambda, or a plain object with a `#call` method) where a callable is expected.

**Example**

```ruby
class Duck
  def quack = "Quack!"
end

class Person
  def quack = "I'm quacking!"
end

def make_it_quack(thing)
  thing.quack   # no type check -- if it responds to #quack, it's usable here
end

make_it_quack(Duck.new)    # => "Quack!"
make_it_quack(Person.new)  # => "I'm quacking!"
```

### 11. Symbols vs strings — mutability, memory/identity, and when to use which?

**Short Answer**

- **Symbols** are immutable and interned — Ruby keeps one copy per unique name and reuses it.
- **Strings** are mutable, and every string literal creates a brand-new object.

Use symbols for fixed labels (hash keys, statuses, method names). Use strings for actual data.

**Simple Explanation**

"Interned" means Ruby stores exactly one `:name` for the whole process and hands you a reference to it every time. That makes symbols cheap to compare (it's just an object identity check) and impossible to change — `Symbol` has no mutating methods at all.

Strings are the opposite. Each literal allocates a new object, and String has real in-place methods (`<<`, `gsub!`, `upcase!`) that change the object itself.

Rule of thumb: if the value is a fixed label your code branches on, use a symbol. If it came from a user, a file, or the network, use a string.

**Example**

```ruby
:name.object_id == :name.object_id     # => true  -- symbols are interned, one copy per process
"name".object_id == "name".object_id   # => false -- every string literal is a new object

:name.frozen?    # => true  -- symbols are always immutable, there's no Symbol#<< or #upcase!
"name".frozen?   # => false (unless the file has a `# frozen_string_literal: true` magic comment)

str = "name"
str << "!"        # mutates the SAME String object in place
str                # => "name!"

{ status: :active }   # :status and :active are identifiers here, not data being processed
```

### 12. What is a `Struct`, and when do you reach for one over a Hash or a full class?

**Short Answer**

`Struct` generates a small class for you with named accessors, easy construction, and value-based `==`.

Use it instead of a Hash when you want method-style access and a real class. Use it instead of a full class when you don't need validation or custom behavior.

**Simple Explanation**

`Struct.new(:x, :y)` returns a class that already has `x`, `y`, `x=`, `y=`, plus extras like `#to_a`, `#to_h`, `#==`, and `#each`.

Compared to a Hash: a typo raises `NoMethodError` instead of silently returning `nil`, and you get a real class name and `is_a?` check.

Compared to a hand-written class: you skip the boilerplate, but you lose a clean place to put validation. Once a Struct starts collecting business logic, promote it to a real class.

**Example**

```ruby
Point = Struct.new(:x, :y) do
  def distance_to(other)
    Math.hypot(x - other.x, y - other.y)
  end
end

p1 = Point.new(0, 0)
p2 = Point.new(3, 4)
p1.distance_to(p2)     # => 5.0
p1.to_a                 # => [0, 0]
p1 == Point.new(0, 0)   # => true -- value equality comes for free
```

### 13. What are `respond_to?` and `method_missing`, and when does `method_missing` become an anti-pattern?

**Short Answer**

- `respond_to?` asks an object "do you have this method?"
- `method_missing` catches calls to methods that don't exist.

`method_missing` becomes a problem when you forget to also define `respond_to_missing?` — then the object lies about what it can do — or when a few plain `def`s would have been simpler and easier to grep.

**Simple Explanation**

`method_missing(name, *args, &block)` is the hook Ruby calls right before it would raise `NoMethodError`. It's great for proxies and dynamic finders.

The catch: if you define `method_missing` without `respond_to_missing?`, then `object.respond_to?(:thing)` returns `false` even though `object.thing` works fine. That breaks anything relying on introspection — serializers, template helpers, `public_send` dispatch, plain duck typing.

Two rules: always define the pair together, and always call `super` for names you don't handle so real `NoMethodError`s still surface.

**Example**

```ruby
class LazyProxy
  def method_missing(name, *args, &block)
    if name.to_s.start_with?("find_by_")
      "pretending to query by #{name.to_s.sub('find_by_', '')}"
    else
      super   # fall back for names we don't recognize -- don't swallow everything
    end
  end
end

proxy = LazyProxy.new
proxy.find_by_email("a@b.com")     # => "pretending to query by email"
proxy.respond_to?(:find_by_email)  # => false! -- respond_to_missing? was never overridden
proxy.method(:find_by_email)       # NameError -- breaks reflection, `send`, duck typing checks
```

### 14. Class methods vs. instance methods — `self.foo` vs. `foo`, and what does `class << self` do?

**Short Answer**

- `def foo` → instance method, called on objects.
- `def self.foo` → class method, called on the class itself.
- `class << self ... end` → opens the class's singleton class so you can define several class methods (or class-level `attr_accessor`s) without repeating `self.` every time.

**Simple Explanation**

In Ruby a class is also an object (an instance of `Class`). So `def self.foo` is really just defining a method on that one object.

`class << self ... end` opens that same space explicitly. It's handy when you want normal-looking `def` syntax for several class methods at once, or when you want `attr_accessor` to create class-level getters and setters — those are backed by instance variables on the class object, not on individual instances.

**Example**

```ruby
class Report
  def self.generate   # class method, defined on Report's singleton class
    new.render
  end

  class << self
    attr_accessor :default_format   # class-level accessor

    def reset!
      @default_format = "pdf"
    end
  end

  def render = "rendering..."   # instance method, called on Report instances
end

Report.generate             # => "rendering..."
Report.default_format = "csv"
Report.default_format       # => "csv"
```

### 15. `public` vs. `private` vs. `protected` — what's the nuance with `protected`?

**Short Answer**

- **public** — anyone can call it.
- **private** — can only be called without an explicit receiver (implicitly on `self`).
- **protected** — can be called with an explicit receiver, but only from inside another instance method of the same class (or a subclass).

`protected` exists so one object can peek at another object of the same class.

**Simple Explanation**

Say you want to compare two `Money` objects:

```ruby
def >(other)
  cents > other.cents
end
```

`other.cents` uses an explicit receiver. If `cents` were private, that line would fail. `protected` allows it: still hidden from the outside world, but "family members" can call it on each other.

So the quick rule: private means "only me, on myself". Protected means "me, on any object of my own class".

**Example**

```ruby
class Money
  def initialize(cents) = @cents = cents

  def >(other)
    cents > other.cents   # explicit receiver `other.cents` -- only legal because cents is protected
  end

  protected

  def cents = @cents
end

Money.new(500) > Money.new(300)   # => true
Money.new(500).cents              # NoMethodError -- protected, no outside caller allowed
```

### 16. What does Ruby's Garbage Collector do, at a high level?

**Short Answer**

MRI (standard Ruby) uses a mark-and-sweep, generational, incremental garbage collector. It marks every object still reachable, frees everything unmarked, and — since most objects die young — checks young objects far more often than old ones.

**Simple Explanation**

**Mark:** starting from the roots (call stack, globals, class variables), Ruby walks every object it can reach and flags it as alive.

**Sweep:** it then walks the heap. Anything not flagged is garbage, and its memory slot is reclaimed.

**Generational (since Ruby 2.1):** most objects die quickly, so Ruby splits them into a young generation (checked on almost every run — a cheap "minor GC") and an old generation for survivors (checked only occasionally — an expensive "major GC").

You rarely tune this by hand, but knowing the vocabulary matters when you're reading `GC.stat` while chasing a memory problem.

**Example**

```ruby
GC.stat[:count]            # total GC runs so far
GC.stat[:minor_gc_count]   # cheap young-generation collections
GC.stat[:major_gc_count]   # expensive full collections
GC.start                   # force a full major collection
```

### 17. What does `freeze` do, and why would you freeze a constant?

**Short Answer**

`freeze` makes one object immutable — changing it afterwards raises `FrozenError`. It's **shallow**: freezing a container does not freeze what's inside it.

You freeze constants so an accidental change becomes a loud crash instead of silent data corruption.

**Simple Explanation**

Ruby only warns when you reassign a constant, and it never stops you from *mutating* the object a constant points at. So `PERMISSIONS << "admin"` happily changes a shared array.

Since constants are usually referenced from many places, one careless `<<` corrupts the value for everyone. Freezing turns that into an immediate exception at the exact line that caused it.

The shallow part catches people out: freezing `{ roles: ["admin"] }` does not freeze the inner array, so `NESTED[:roles] << "editor"` still works.

**Example**

```ruby
PERMISSIONS = %w[read write].freeze
PERMISSIONS << "admin"        # FrozenError: can't modify frozen Array

CONFIG = { retries: 3 }.freeze
CONFIG[:retries] = 5          # FrozenError

# freeze is SHALLOW -- it only locks the object itself, not what it contains
NESTED = { roles: ["admin"] }.freeze
NESTED[:roles] << "editor"    # works! the inner Array was never frozen
NESTED[:roles]                # => ["admin", "editor"]
```

### 18. `require` vs. `require_relative` vs. `load` — what's the difference?

**Short Answer**

- `require` — loads a file once, searching `$LOAD_PATH` (gems, stdlib). Calling it twice does nothing the second time.
- `require_relative` — same, but the path is relative to the *current file's* folder.
- `load` — runs the file again every single time, no caching.

**Simple Explanation**

Use `require` for gems and the standard library (`require "json"`). It's cached, so repeat calls are free.

Use `require_relative` for files inside your own project. It's anchored to the file doing the requiring, not to the directory you happened to run the program from — much safer.

`load` re-runs the file top to bottom on every call. It's occasionally handy for reload-style workflows in a console, but it's not what you want for normal dependency loading.

**Example**

```ruby
require "json"                    # searches $LOAD_PATH; loads once, cached in $LOADED_FEATURES
require_relative "lib/formatter"  # resolved relative to THIS file's directory, not the cwd

load "lib/formatter.rb"           # re-executes the file EVERY call, no caching, needs the .rb extension
```

### 19. Mutable vs. immutable objects in Ruby — show a surprising mutability bug.

**Short Answer**

Most objects are mutable: Array, Hash, String, and your own classes. Integers, Floats, `nil`, `true`, `false`, and Symbols are always immutable.

The classic bug is `Hash.new([])` — the default object is created **once** and shared by every missing key.

**Simple Explanation**

`Hash.new([])` looks like "give me a fresh empty array for each missing key", but it doesn't. Every missing key gets a reference to the *same* array.

Two problems follow:
1. `<<` mutates that one shared array, so values bleed across keys.
2. Reading a missing key with `hash[:a]` never actually inserts the key, so the hash stays empty.

The fix is the block form, `Hash.new { |hash, key| hash[key] = [] }`, because the block runs fresh for each missing key and actually stores it.

**Example**

```ruby
grouped = Hash.new([])
grouped[:a] << 1
grouped[:b] << 2
grouped   # => {}  !! -- both `<<` calls mutated the SAME shared default array,
          #             and neither :a nor :b was ever actually inserted as a key

# Fix: the block form runs once PER missing key
grouped = Hash.new { |hash, key| hash[key] = [] }
grouped[:a] << 1
grouped[:b] << 2
grouped   # => { a: [1], b: [2] }
```

### 20. How do `begin/rescue/ensure/raise` work, and how do you design a custom exception hierarchy?

**Short Answer**

`begin/rescue` catches errors, `ensure` always runs (error or not), and `raise` throws one. Ruby checks `rescue` clauses top to bottom and uses the first match, so list the most specific one first.

For custom errors, inherit from `StandardError`, create one base error for your app, and put specific errors under it.

**Simple Explanation**

Never `rescue Exception`. That also catches things like `SystemExit` and `NoMemoryError` that you almost never want to intercept. `StandardError` is the right root for application errors, and it's what a bare `rescue` catches by default.

A shared base class per domain (say `ApplicationError`) lets callers choose how specific they want to be. They can rescue one exact error when they need to react differently, or the base class when they just need "something in this area went wrong".

`ensure` is for cleanup — closing files, releasing locks — and runs even if you `raise` or `return`.

**Example**

```ruby
class ApplicationError < StandardError; end

class RecordNotFoundError < ApplicationError
  def initialize(id) = super("record #{id} not found")
end

class ValidationError < ApplicationError
  attr_reader :field
  def initialize(field, message)
    @field = field
    super(message)
  end
end

def fetch(id)
  raise RecordNotFoundError, id unless id == 1
  "record"
end

begin
  fetch(2)
rescue ValidationError => e         # most specific first
  puts "invalid #{e.field}"
rescue ApplicationError => e        # falls back to the shared domain base class
  puts "app error: #{e.message}"
rescue StandardError => e
  puts "unexpected: #{e.message}"
ensure
  puts "always runs, even on raise or return"   # cleanup: close files, release locks, etc.
end
# => "app error: record 2 not found"
# => "always runs, even on raise or return"
```

### 21. What do the splat (`*`) and double-splat (`**`) operators do?

**Short Answer**

- `*` collects extra positional arguments into an Array, or explodes an Array back into separate arguments.
- `**` does the same for keyword arguments and a Hash.

**Simple Explanation**

It's the same symbol used in two directions.

In a method **definition**, `*args` / `**opts` gather "everything else" into a collection.

At a **call site**, `*array` / `**hash` do the reverse — they spray the collection back out as individual arguments.

Splat also works in plain assignment for destructuring: `first, *rest = [1, 2, 3, 4]` gives you `first = 1` and `rest = [2, 3, 4]`.

**Example**

```ruby
def sum(*numbers) = numbers.sum
sum(1, 2, 3)   # => 6

def configure(**options) = options
configure(retries: 3, timeout: 10)   # => {retries: 3, timeout: 10}

nums = [1, 2, 3]
sum(*nums)      # splat EXPLODES the array back into positional args -- same as sum(1, 2, 3)

opts = { retries: 3 }
configure(**opts, timeout: 10)   # double-splat forwards a Hash as keyword args

first, *rest = [1, 2, 3, 4]   # destructuring: first = 1, rest = [2, 3, 4]
```

### 22. Keyword arguments — required vs. optional vs. `**kwargs`, and why prefer them over positional booleans?

**Short Answer**

- `name:` with no default → required, raises `ArgumentError` if missing.
- `name: default` → optional.
- `**kwargs` → catches any keyword arguments you didn't name.

Keyword arguments beat positional booleans because the call site explains itself and the order stops mattering.

**Simple Explanation**

Look at `create_user("Ada", true, false)`. What do `true` and `false` mean? You have to go read the method definition to find out.

Now look at `create_user(name: "Ada", admin: true, notify: false)`. It reads on its own, and you can reorder the arguments freely.

This is exactly why Rails APIs use keyword-style options almost everywhere.

**Example**

```ruby
def create_user(name:, admin: false, **extra)
  { name: name, admin: admin, extra: extra }
end

create_user(name: "Ada")                              # => {name: "Ada", admin: false, extra: {}}
create_user(name: "Ada", admin: true, role: "owner")  # => {name: "Ada", admin: true, extra: {role: "owner"}}
create_user(admin: true)                              # ArgumentError: missing keyword: :name

def create_user_bad(name, admin, notify)
  { name: name, admin: admin, notify: notify }
end
create_user_bad("Ada", true, false)   # what do `true` and `false` mean here? have to check the signature
```

### 23. What's the pitfall with `@foo ||= ...` memoization?

**Short Answer**

`@foo ||= expensive_call` re-runs the expensive call **every time** if the correct answer is `false` or `nil`.

That's because `a ||= b` expands to `a || (a = b)`, and `false || anything` is still falsy.

**Simple Explanation**

Memoizing with `||=` is a nice Ruby idiom, but it quietly breaks for any method whose real answer can be `false` or `nil`. The "cached" value never looks cached, so the work repeats on every call.

The fix is to check whether the variable was **set**, not whether it's truthy:

```ruby
return @admin if defined?(@admin)
@admin = compute_admin_status
```

`defined?(@admin)` is true once it's been assigned — even if it was assigned `false`.

**Example**

```ruby
class User
  def admin?
    @admin ||= compute_admin_status   # BUG when compute_admin_status legitimately returns false
  end

  def compute_admin_status
    puts "expensive check ran"
    false
  end
end

u = User.new
u.admin?  # prints "expensive check ran"
u.admin?  # prints "expensive check ran" AGAIN -- memoization never "stuck"

class FixedUser
  def admin?
    return @admin if defined?(@admin)   # true once @admin has been assigned, even to false
    @admin = compute_admin_status
  end

  def compute_admin_status
    puts "expensive check ran"
    false
  end
end
```

### 24. How does constant scoping work in Ruby?

**Short Answer**

Constants use `SCREAMING_SNAKE_CASE`, can be added to a class or module at any time by reopening it, and are resolved by **lexical scope** — the `module`/`class` blocks physically wrapping the code — before Ruby falls back to the ancestor chain.

**Simple Explanation**

When Ruby sees a bare constant, it first walks `Module.nesting` — the `module`/`class` blocks that literally surround that line in the file, from innermost outward. Only if nothing matches there does it search superclasses and included modules.

This causes a genuinely surprising gotcha: a method's constant lookup is decided by **where the method was written**, not by which subclass later calls it. So two methods on the same object can resolve the same constant name to different values if they were defined inside different modules.

**Example**

```ruby
module Shop
  TAX_RATE = 0.07

  class Order
    def total(subtotal)
      subtotal * (1 + TAX_RATE)   # lexical lookup finds Shop::TAX_RATE via the enclosing module
    end
  end
end

# Reopening a class/module to add a constant later is completely legal
class String
  MAX_DISPLAY_LENGTH = 100 unless defined?(MAX_DISPLAY_LENGTH)
end

class Base
  VALUE = "base"
  def show = VALUE
end

module Wrapper
  VALUE = "wrapper"
  class Derived < Base
    def show2 = VALUE   # lexically nested inside Wrapper -- resolves VALUE there FIRST
  end
end

Wrapper::Derived.new.show   # => "base"     (inherited from Base, resolved in Base's lexical scope)
Wrapper::Derived.new.show2  # => "wrapper"  (defined inside Wrapper, resolved there, ancestry never checked)
```

**— Enumerable & Collections —**

### 25. How do `map`, `select`/`filter`, `reject`, `each`, `find`/`detect`, `any?`, `all?`, and `none?` differ?

**Short Answer**

- `map` → new array of transformed values
- `each` → returns the original collection; use it for side effects
- `select` / `filter` → keep items where the block is true
- `reject` → keep items where the block is false
- `find` / `detect` → first match, then stops
- `any?` / `all?` / `none?` → single true/false answer

**Simple Explanation**

The mix-up people hit most is `map` vs `each`. They loop identically, but only `map`'s return value is useful. `each` always returns the original collection, no matter what the block returns — so using `each` when you meant to transform data quietly gives you the wrong result.

`select` and `reject` are mirror images of each other using the same condition.

`find` stops as soon as it hits a match, which matters on large collections.

**Example**

```ruby
numbers = [1, 2, 3, 4, 5]

numbers.map  { |n| n * 2 }    # => [2, 4, 6, 8, 10]  -- NEW array of transformed values
numbers.each { |n| n * 2 }    # => [1, 2, 3, 4, 5]   -- returns the ORIGINAL receiver, unchanged

numbers.select { |n| n.even? }  # => [2, 4]     -- keeps elements where the block is truthy
numbers.filter { |n| n.even? }  # => [2, 4]     -- `filter` is just an alias for `select`
numbers.reject { |n| n.even? }  # => [1, 3, 5]  -- keeps elements where the block is FALSY

numbers.find { |n| n > 3 }   # => 4     -- first match, stops iterating early (aliased `detect`)
numbers.any?  { |n| n > 4 }  # => true
numbers.all?  { |n| n > 0 }  # => true
numbers.none? { |n| n > 10 } # => true
```

### 26. How does `reduce`/`inject` work — both the symbol shorthand and the block form?

**Short Answer**

- `inject(:+)` — shorthand for "apply this operator between every pair of elements".
- `inject(start) { |acc, item| ... }` — the block form, where you build up any result you want.

`reduce` and `inject` are the same method.

**Simple Explanation**

The symbol form is a shortcut for the common "combine everything with one operator" case.

The block form is much more flexible. The block runs once per element with the running accumulator and the current item — and it **must return the accumulator** each time.

Forgetting that return value is the classic bug. If you're building up a Hash and the last line of the block isn't the hash itself, the accumulator gets replaced with whatever the block returned instead.

**Example**

```ruby
[1, 2, 3, 4].inject(:+)                          # => 10  -- symbol shorthand
[1, 2, 3, 4].inject(100) { |sum, n| sum + n }     # => 110 -- explicit start value + block

words = %w[apple banana avocado blueberry]
words.reduce({}) do |grouped, word|
  key = word[0]
  grouped[key] ||= []
  grouped[key] << word
  grouped   # MUST return the accumulator each iteration, or it gets clobbered
end
# => { "a" => ["apple", "avocado"], "b" => ["banana", "blueberry"] }
```

### 27. What does `Set` give you over an `Array`, and when do you reach for one?

**Short Answer**

A `Set` keeps only unique values and answers "do you contain X?" in roughly constant time, because it's backed by a Hash. `Array#include?` has to scan every element, which is O(n).

Reach for a Set when membership checks or removing duplicates matter more than order.

**Simple Explanation**

An Array can hold duplicates and has to walk the whole list to answer `include?`. That gets slow as it grows.

A Set rejects duplicates on insert, answers membership checks fast, and gives you real set operations: union (`|`), intersection (`&`), and difference (`-`). Doing that with arrays means hand-rolling combinations of `uniq` and `select`.

**Example**

```ruby
require "set"

seen = Set.new
seen.add(:a)
seen << :b
seen.include?(:a)   # => true, close to O(1) average -- backed by a Hash internally
seen.include?(:z)   # => false

a = Set[1, 2, 3]
b = Set[2, 3, 4]
a | b   # union        => #<Set: {1, 2, 3, 4}>
a & b   # intersection => #<Set: {2, 3}>
a - b   # difference   => #<Set: {1}>
```

**— Concurrency —**

### 28. Threads vs. Processes vs. Fibers vs. Ractors — what does each actually parallelize?

**Short Answer**

- **Threads** — share memory, but MRI's GVL lets only one thread run Ruby code at a time. Great for I/O, useless for CPU work.
- **Processes** — real parallelism (each has its own GVL), but more memory and no shared objects.
- **Fibers** — single-threaded and cooperative. You decide exactly when to switch.
- **Ractors** — real parallelism across cores, but no shared mutable state between them.

**Simple Explanation**

MRI (standard Ruby) has one Global VM Lock, so only one thread executes Ruby code at any instant. But the GVL **is released during blocking I/O** — network calls, file reads, DB queries, `sleep`. That's why threads are still the normal tool for running many HTTP or DB calls at once.

Forked processes skip the GVL entirely because each gets its own Ruby VM. You get true parallel CPU work, at the cost of more memory (copy-on-write helps) and no shared objects — they have to talk through pipes, sockets, or serialization.

Fibers don't run concurrently at all. They're lightweight execution contexts you resume manually. They power things like `Enumerator::Lazy` and non-blocking I/O schedulers.

Ractors (Ruby 3.0+) run truly in parallel with no shared GVL. The catch is that objects generally can't be shared between them — they're copied or explicitly moved — so most existing code needs rework to run inside one.

**Example**

```ruby
# Threads: help I/O-bound work; useless for CPU-bound work under MRI's GVL
threads = urls.map { |url| Thread.new { fetch(url) } }
threads.each(&:join)   # several HTTP calls overlap while waiting on the network

# Processes: true parallelism, even for CPU-bound work, at a higher memory cost
pid = fork { expensive_cpu_work }
Process.wait(pid)

# Fibers: cooperative, single-threaded, manually resumed/yielded
fiber = Fiber.new do
  puts "a"
  Fiber.yield
  puts "b"
end
fiber.resume   # prints "a"
fiber.resume   # prints "b"

# Ractors: true parallelism, but no shared mutable state between them
ractor = Ractor.new { Ractor.receive * 2 }
ractor.send(21)
ractor.take   # => 42, computed on a separate core, without the GVL
```

**— Ruby 3.x Features —**

### 29. How does pattern matching (`case/in`) work?

**Short Answer**

`case/in` matches the **shape** of a value — Hash keys, Array positions, nested structures — and pulls pieces out into local variables at the same time. No manual `[]` and `dig` needed.

**Simple Explanation**

`case/when` compares one value using `===`. `case/in` matches structure instead.

Writing `in { status: "ok", data: { user: } }` does two things at once: it checks the object has that shape, and it assigns `user` for you.

Use `else` for "nothing matched". Without it, an unmatched value raises `NoMatchingPatternError`.

One extra piece: putting `^` in front of an existing variable ("pinning") matches against that variable's current **value** instead of reassigning it.

**Example**

```ruby
response = { status: "ok", data: { user: { name: "Ada", roles: ["admin", "editor"] } } }

case response
in { status: "ok", data: { user: { name:, roles: [first_role, *] } } }
  puts "#{name}'s primary role is #{first_role}"
in { status: "error", message: }
  puts "failed: #{message}"
else
  puts "unrecognized shape"
end
# => "Ada's primary role is admin"

expected = "ok"
case response
in { status: ^expected }   # pin: matches against expected's VALUE, doesn't rebind it
  puts "matched exactly #{expected}"
end
```

### 30. `Data.define` (Ruby 3.2+) vs. `Struct` — when do you reach for the newer one?

**Short Answer**

- `Data.define` (Ruby 3.2+) → immutable value object. No setters. Use `#with` to get a changed copy.
- `Struct` → mutable by default, with setters.

Reach for `Data.define` whenever the thing should never change after it's built.

**Simple Explanation**

`Struct` gives you getters *and* setters plus extras like `to_a` and `[]`. That fits a loose, changeable bag of fields.

`Data.define` is deliberately narrower. It's Ruby's purpose-built API for value objects — things defined entirely by their attributes.

Instead of `point.x = 6`, you call `point.with(x: 6)`, which returns a **new** object and leaves the original alone. That plays much better with concurrency, caching, and reasoning about equality.

**Example**

```ruby
Point = Data.define(:x, :y) do
  def magnitude = Math.hypot(x, y)
end

p1 = Point.new(x: 3, y: 4)   # keyword init (Point.new(3, 4) positional also works)
p1.magnitude   # => 5.0
p1.x = 10      # NoMethodError -- Data objects are immutable, no setters at all

p2 = p1.with(x: 6)   # non-destructive update: returns a NEW Point, p1 untouched
p2   # => #<data Point x=6, y=4>
```

### 31. What is Ruby 3.1+ Hash shorthand ("punning")?

**Short Answer**

When a local variable and a hash key have the same name, `{ x:, y: }` is short for `{ x: x, y: y }`. The same shorthand works for keyword arguments at a call site.

**Simple Explanation**

It's a small readability win that removes repetition. It shows up a lot when you're building a Hash (or passing keyword arguments) straight out of local variables you just computed.

**Example**

```ruby
x = 10
y = 20
point = { x:, y: }   # equivalent to { x: x, y: y }
point   # => { x: 10, y: 20 }

def move(x:, y:) = "moved to (#{x}, #{y})"
move(x:, y:)   # same shorthand works for keyword arguments, since x and y are local vars in scope
```

### 32. What are endless method definitions, and when do they help vs. hurt readability?

**Short Answer**

`def square(x) = x * x` (Ruby 3.0+) defines a method as a single expression on one line. It reads well for tiny methods and predicates, and badly once the body needs a conditional, several statements, or a `rescue`.

**Simple Explanation**

Endless methods aren't a different kind of method — they compile to exactly the same thing as a normal `def ... end`. It's purely a style choice.

They're great for one-liners like `def admin? = role == "admin"`, where the `end` line was just noise.

They stop paying off as soon as you'd need a semicolon or a line continuation to squeeze real logic onto one line. At that point a normal multi-line `def` is easier to read and easier to diff.

**Example**

```ruby
def square(x) = x * x
def full_name = "#{first_name} #{last_name}"
def admin? = role == "admin"

# borderline -- still readable, but this is close to where it stops paying off
def discount(price) = price > 100 ? price * 0.9 : price
```

**— Metaprogramming —**

### 33. What do `send` and `public_send` do, and why is `public_send` the safer default?

**Short Answer**

- `send` calls a method by name and **ignores visibility** — it can call private methods.
- `public_send` does the same but respects visibility.

Use `public_send` whenever the method name comes from outside your code.

**Simple Explanation**

`send` is genuinely useful for testing private methods or for internal metaprogramming where you already trust the name.

But the moment the name comes from a URL parameter, a config file, or user input, `send` becomes a way to call methods that were deliberately hidden. `public_send` closes that hole by raising `NoMethodError` for anything that isn't public — exactly like a normal `.method_name` call would.

**Example**

```ruby
class Account
  def initialize(balance) = @balance = balance

  private

  def apply_interest(rate) = @balance *= (1 + rate)
end

acct = Account.new(1000)
acct.send(:apply_interest, 0.05)          # works -- send bypasses visibility entirely
acct.public_send(:apply_interest, 0.05)   # NoMethodError -- private method, respected
```

### 34. How does `define_method` work, with a real example?

**Short Answer**

`define_method` creates an instance method from a block at class level. Unlike a normal `def`, the block is a closure, so it can capture local variables from the surrounding scope.

**Simple Explanation**

This is the workhorse behind "generate a family of similar methods". Instead of writing six near-identical `def`s, you loop over a list of names and generate them.

A good chunk of how Rails' ActiveRecord attribute methods work comes down to this.

**Example**

```ruby
class Settings
  %i[timeout retries verbose].each do |attr|
    define_method(attr)        { instance_variable_get("@#{attr}") }
    define_method("#{attr}=")  { |value| instance_variable_set("@#{attr}", value) }
  end
end

s = Settings.new
s.timeout = 30
s.timeout   # => 30
```

### 35. How do you combine `method_missing` with `respond_to_missing?` for a dynamic proxy?

**Short Answer**

Write `method_missing` to handle (or forward) unknown calls, and **always** write `respond_to_missing?` next to it so `respond_to?`, `method()`, and duck-typing checks tell the truth about what the object supports.

**Simple Explanation**

The classic use case is a delegator that wraps another object and forwards calls to it — similar in spirit to stdlib's `SimpleDelegator`.

Without `respond_to_missing?`, any code that checks `respond_to?` before calling a method — and serializers, template helpers, and test doubles do this constantly — will wrongly believe your proxy doesn't support methods it handles perfectly well.

**Example**

```ruby
class Delegator
  def initialize(target) = @target = target

  def method_missing(name, *args, &block)
    if @target.respond_to?(name)
      @target.public_send(name, *args, &block)
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private) || super
  end
end

wrapped = Delegator.new([3, 1, 2])
wrapped.sort                 # => [1, 2, 3] -- forwarded to the wrapped Array
wrapped.respond_to?(:sort)   # => true      -- correct, because respond_to_missing? is defined
wrapped.method(:sort)        # works too, instead of raising NameError
```

### 36. What do `class_eval`/`module_eval` and `instance_eval` do, conceptually?

**Short Answer**

- `class_eval` (alias `module_eval`) — runs a block with `self` set to a class or module. Used to add instance methods to it.
- `instance_eval` — runs a block with `self` set to one single object. Used to reach its private state or build builder-style DSLs.

**Simple Explanation**

Both let you run code with a different `self`, just at different levels.

`class_eval` works at class level, so a `def` inside the block becomes a normal instance method available to every instance. This is how `ActiveSupport::Concern` and macros like `has_many` inject methods into a model.

`instance_eval` works on one object, so a `def` inside the block defines a method on **that object only**. That's the trick behind builder DSLs where you write `configure { set_timeout 5 }` instead of `configure.set_timeout(5)`.

**Example**

```ruby
String.class_eval do
  def shout = upcase + "!"   # adds an INSTANCE method to String; self here is the String class
end
"hi".shout   # => "HI!"

obj = Object.new
obj.instance_eval do
  @secret = 42              # self here is `obj` itself -- can reach into its private state
  def whisper = @secret     # defines a SINGLETON method, only on this one object
end
obj.whisper   # => 42
```

### 37. What is a Ruby DSL, and how does metaprogramming build one?

**Short Answer**

A DSL (domain-specific language) is just ordinary Ruby method calls and blocks arranged so they read like a small declarative language. RSpec's `describe`/`it` and the Rails routes file are the classic examples.

They're built with `instance_eval`/`class_eval` (to change what `self` means inside a block) plus `method_missing` or `define_method`.

**Simple Explanation**

There's no special "DSL mode" in Ruby. `describe`, `it`, and `resources` are completely normal method calls that take a block and run it in a context where bare names resolve without an explicit receiver.

That illusion of a mini-language is metaprogramming's biggest practical payoff: framework authors can give application code a vocabulary that reads almost like English while still being plain, callable Ruby underneath.

**Example**

```ruby
# RSpec -- describe/it take a block, evaluated against an example-group object
RSpec.describe User do
  it "is valid with a name" do
    expect(User.new(name: "Ada")).to be_valid
  end
end

# Rails routes -- `resources` is a method call, not a language keyword
Rails.application.routes.draw do
  resources :orders, only: %i[index show]
end
```

### 38. What's the real risk of heavy metaprogramming, and when is the cost actually worth it?

**Short Answer**

Generated methods are hard to `grep` for, editors can't "jump to definition" on them, and stack traces point into framework internals instead of your code.

It's worth it for high-leverage framework code that removes huge amounts of boilerplate (ActiveRecord associations, RSpec matchers). It's rarely worth it for one-off business logic in your own app.

**Simple Explanation**

`has_many :comments` generates `comments`, `comments=`, `comment_ids`, and more, all through `define_method`. None of them exist as a literal `def` you can search for.

That's a fair trade in ActiveRecord, because one macro call replaces dozens of methods for every model in every Rails app ever written.

Using the same trick to save writing five methods in one internal class usually isn't worth it. The person who has to find where a method is defined six months from now pays more than you saved.

**Example**

```ruby
class Post < ApplicationRecord
  has_many :comments
end

Post.new.comments
# - `grep -rn "def comments"` finds NOTHING in the app
# - "jump to definition" in most editors fails, or lands inside the has_many macro itself
# - the stack trace on a bug here shows activerecord/associations internals, not app code
#
# Worth it here: one macro call replaces dozens of hand-written methods for every model.
# Rarely worth it for one-off logic in your own classes, where a plain `def` costs nothing.
```

## Ruby on Rails

**— Fundamentals & Request Lifecycle —**

### 39. Walk through the Rails request lifecycle from the router to the response.

**Short Answer**

A request hits the web server (Puma), goes down through the Rack middleware stack, the router picks a controller action, the controller talks to models, a view or serializer builds the response, and the response travels back up through the same middleware to the client.

**Simple Explanation**

Step by step, for a normal HTML request:

1. **Web server** — Puma accepts the connection and hands Rails a Rack `env` hash.
2. **Middleware (going down)** — a chain of small objects that each wrap the next one. They serve static files, load the session cookie, parse cookies, turn `_method=patch` into a real PATCH, and so on.
3. **Routing** — `config/routes.rb` matches the HTTP verb and path to a `controller#action` and pulls out params like `:id`.
4. **Controller** — the action runs: builds strong parameters, calls models or services, and decides what to render.
5. **Model** — Active Record turns method calls into SQL, runs it through the connection pool, and returns Ruby objects.
6. **View** — an ERB template is rendered inside a layout, or a serializer builds JSON.
7. **Middleware (going up)** — the response (a `[status, headers, body]` triplet) bubbles back through the same middleware, which can add headers, log timing, or write the session cookie.
8. **Response** — Puma writes the bytes back to the client.

The key idea: middleware, the router, and the app all speak the same Rack interface — `call(env)` returns `[status, headers, body]`.

**Example**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :articles, only: [:index, :show]
end

# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  def show
    # Model: hits the DB through the connection pool
    @article = Article.find(params[:id])
    # View: renders app/views/articles/show.html.erb inside the layout
    render :show
  end
end

# See the actual middleware stack for your app:
# $ bin/rails middleware
# use Rack::Sendfile
# use ActionDispatch::Static
# use ActionDispatch::Executor
# use ActiveSupport::Cache::Strategy::LocalCache::Middleware
# use ActionDispatch::ServerTiming
# use Rack::Runtime
# use Rack::MethodOverride
# use ActionDispatch::RequestId
# use ActionDispatch::RemoteIp
# use Rails::Rack::Logger
# use ActionDispatch::ShowExceptions
# use ActionDispatch::Cookies
# use ActionDispatch::Session::CookieStore
# use ActionDispatch::Flash
# run MyApp::Application.routes
```

### 40. What is MVC, and why does Rails enforce it so rigidly?

**Short Answer**

MVC splits an app into three layers: **Model** (data and business rules), **View** (what the user sees), and **Controller** (handles the request and coordinates the other two).

Rails enforces it through folder conventions and "convention over configuration", so every Rails app looks the same to a new developer.

**Simple Explanation**

The Model owns data and domain logic — validations, associations, business rules. The View owns presentation and should have almost no logic beyond formatting and looping. The Controller is a thin coordinator: take the request, ask the model for data, pick what to render. It should not hold business logic.

Rails leans on this hard because it means anyone can open any Rails app and know that `UserMailer` lives in `app/mailers` and `User` validations live in `app/models/user.rb`.

When teams ignore the split you get "fat controllers" (business logic crammed into actions) or "fat views" (logic-heavy templates). The fix isn't to fight MVC — it's to extract service, query, and form objects, which this guide covers later.

**Example**

```ruby
# BAD: business logic and formatting leaking into the controller and view
class OrdersController < ApplicationController
  def create
    @order = Order.new(order_params)
    @order.total = @order.line_items.sum { |li| li.price * li.quantity } * 1.08
    if @order.total > 1000
      @order.status = "needs_review"
    else
      @order.status = "approved"
    end
    @order.save
    redirect_to @order
  end
end

# GOOD: model owns the rule, controller just orchestrates
class Order < ApplicationRecord
  TAX_RATE = 0.08
  REVIEW_THRESHOLD = 1000

  before_save :calculate_total_and_status

  private

  def calculate_total_and_status
    self.total = line_items.sum { |li| li.price * li.quantity } * (1 + TAX_RATE)
    self.status = total > REVIEW_THRESHOLD ? "needs_review" : "approved"
  end
end

class OrdersController < ApplicationController
  def create
    @order = Order.new(order_params)
    if @order.save
      redirect_to @order
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

### 41. What is Rack middleware, and what does a default Rails middleware stack actually do?

**Short Answer**

Rack is the simple interface every Ruby web app implements: an object with a `call(env)` method that returns `[status, headers, body]`.

Middleware are small Rack apps that wrap the next one, so you can inspect or change a request/response without touching controller code.

**Simple Explanation**

Each middleware is handed "the next app" in the chain. It usually does something before calling it, something after it returns, or both — like a set of nested function calls.

Run `bin/rails middleware` to see your app's exact stack. A few real examples:

- `ActionDispatch::Static` serves files straight out of `public/`, so `/robots.txt` never reaches a controller.
- `ActionDispatch::Session::CookieStore` decrypts the session cookie into `request.session` before your controller runs, and writes changes back on the way out.
- `Rack::Attack` (commonly added) checks the request for rate limiting before it goes any further.

Write your own middleware for things that apply to every request and don't belong in a controller — request-ID tagging, blanket auth checks, maintenance mode.

**Example**

```ruby
# app/middleware/maintenance_mode.rb — a custom middleware
class MaintenanceMode
  def initialize(app)
    @app = app
  end

  def call(env)
    if File.exist?(Rails.root.join("tmp/maintenance.txt"))
      [503, { "Content-Type" => "text/plain" }, ["Down for maintenance"]]
    else
      @app.call(env) # pass control down the stack
    end
  end
end

# config/application.rb
config.middleware.use MaintenanceMode

# $ bin/rails middleware   -> lists the full ordered stack, e.g.:
# use Rack::Sendfile
# use ActionDispatch::Static
# use ActionDispatch::Executor
# use ActionDispatch::Session::CookieStore
# use MaintenanceMode
# run MyApp::Application.routes
```

### 42. How do Rails environments (development/test/production) differ, and how is that configured?

**Short Answer**

Rails ships three environments — `development`, `test`, `production` — each with its own file in `config/environments/` and its own database. `RAILS_ENV` decides which one boots.

The differences are mostly about caching, eager loading, and how much error detail you show — not about what the app actually does.

**Simple Explanation**

**development** favours fast feedback: code reloads without restarting, you get full error pages in the browser, and eager loading is off so boot is quick.

**test** favours clean, fast, isolated runs: eager loading off, mail is captured instead of sent, and password hashing is usually made cheaper.

**production** favours correctness and speed. `eager_load = true` loads and checks every class at boot, which is why some naming bugs only appear in production. Users see a generic 500 page instead of a stack trace, assets are precompiled, and the cache store is something real like Redis.

You can add your own environment too — a `staging.rb` that copies production but points at different credentials. Rails just needs the matching file and `RAILS_ENV=staging`.

**Example**

```ruby
# config/environments/development.rb
Rails.application.configure do
  config.enable_reloading = true
  config.eager_load = false
  config.consider_all_requests_local = true
  config.action_mailer.raise_delivery_errors = false
  config.active_support.deprecation = :log
end

# config/environments/production.rb
Rails.application.configure do
  config.enable_reloading = false
  config.eager_load = true
  config.consider_all_requests_local = false
  config.action_mailer.raise_delivery_errors = true
  config.log_level = :info
  config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"] }
end

# Checking / switching environment:
# $ RAILS_ENV=production bin/rails console
# Rails.env.production? #=> true
```

### 43. Rails credentials vs plain `ENV` vars — when do you use each, and how does a secrets manager replace both?

**Short Answer**

- **Rails credentials** (`config/credentials.yml.enc`) — encrypted secrets committed to git, shared across the team. Good for values that rarely change.
- **ENV vars** — per-environment values like `DATABASE_URL` that ops can change without touching code.
- **A secrets manager** (Vault, AWS Secrets Manager) — for secrets that must rotate without a redeploy.

**Simple Explanation**

Credentials are one encrypted YAML file in the repo. Anyone with the master key can run `bin/rails credentials:edit`. That's convenient for something like a third-party API key that's the same everywhere.

The problem: rotating a credential means editing the file and shipping a deploy. There's no way to change it without new code.

Plain ENV vars fix that — an ops engineer can change them without touching the codebase — but they still need a restart, and they leak easily into logs and crash reports if you aren't careful.

A real secrets manager goes further. The app fetches the secret at runtime, secrets rotate centrally so every service picks up the new value with no redeploy, access is audited per secret, and rotation can be automated (for example, changing a DB password every 30 days). Neither credentials nor static ENV vars can do that on their own.

**Example**

```ruby
# Rails credentials — encrypted, versioned in git
# $ EDITOR="code --wait" bin/rails credentials:edit
# stripe:
#   secret_key: sk_live_xxx

Rails.application.credentials.stripe[:secret_key]
Rails.application.credentials.dig(:stripe, :secret_key)

# Plain ENV var — per-environment, set on the platform
DATABASE_URL = ENV.fetch("DATABASE_URL")

# config/database.yml
production:
  url: <%= ENV.fetch("DATABASE_URL") %>

# Secrets manager pattern — fetched at runtime, rotates without a deploy
class SecretsClient
  def self.stripe_secret_key
    Rails.cache.fetch("secrets/stripe_secret_key", expires_in: 5.minutes) do
      Aws::SecretsManager::Client.new.get_secret_value(
        secret_id: "prod/stripe/secret_key"
      ).secret_string
    end
  end
end
```

**— Routing & Controllers —**

### 44. What's the difference between resourceful routes and custom routes, and how do nested resources work?

**Short Answer**

- `resources :posts` generates the seven standard REST routes in one line.
- Custom routes (`get`, `post`, `match`) handle anything that doesn't fit CRUD.
- Nested resources put a parent-child relationship into the URL, like `/posts/1/comments`.

**Simple Explanation**

`resources :posts` expands into seven routes (index, show, new, create, edit, update, destroy) all pointing at `PostsController`. Using it signals "this is a standard CRUD resource" and keeps routes predictable.

When an action doesn't map to CRUD — "publish a post", "search" — add a custom route, ideally as a `member` or `collection` route on the resource so it still reads as belonging there, rather than a separate top-level route.

Nested resources give you URLs like `/posts/1/comments` and helpers like `post_comments_path(@post)`. Useful when the child doesn't make sense without the parent. But nesting more than one level deep (`/posts/1/comments/2/replies/3`) is a known smell — use `shallow: true` to flatten it, or just look the child up by its own ID.

**Example**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts do
    member do
      post :publish        # POST /posts/:id/publish
    end
    collection do
      get :search           # GET /posts/search
    end
    resources :comments, shallow: true
    # shallow: true keeps nested only for index/create (/posts/:post_id/comments)
    # and flattens show/edit/update/destroy to /comments/:id
  end

  # a fully custom route for something with no CRUD shape at all
  get "dashboard", to: "dashboards#show"
end

# PostsController
class PostsController < ApplicationController
  def publish
    @post = Post.find(params[:id])
    @post.update!(published_at: Time.current)
    redirect_to @post
  end
end
```

### 45. What's the real difference between `render` and `redirect_to`?

**Short Answer**

- `render` finishes the current request by building a response right now. No new request, and `params` and instance variables are still available.
- `redirect_to` sends a 3xx response telling the browser to make a **brand-new** request to a different URL, so everything from the current request is gone.

**Simple Explanation**

`render` is cheap — it just builds a view or JSON body inside the request you're already in. That's why you `render :new, status: :unprocessable_entity` after a failed create: you want the form redisplayed with the user's typed values and error messages still in memory. A redirect would lose all of that.

`redirect_to` sends a 302 (or 301/303/307) with a `Location` header, and the browser fires a fresh GET. Use it after a successful change so a page refresh doesn't resubmit the form — that's the Post/Redirect/Get pattern.

Common beginner bug: calling both in the same action, or forgetting to `return` after one, which raises `DoubleRenderError`.

**Example**

```ruby
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    if @post.save
      # New request, fresh GET, avoids resubmission on refresh
      redirect_to @post, notice: "Post created!"
    else
      # Same request/response cycle, @post (with errors) still in memory
      render :new, status: :unprocessable_entity
    end
  end

  def show
    @post = Post.find_by(id: params[:id])
    return redirect_to posts_path, alert: "Not found" unless @post
    # falls through to the implicit render :show
  end
end
```

### 46. What are strong parameters, and what vulnerability do they prevent?

**Short Answer**

Strong parameters make you list exactly which attributes can be set from user input:

```ruby
params.require(:user).permit(:name, :email)
```

They prevent **mass assignment** attacks, where someone adds an extra field like `admin=true` to a form submission to set something they shouldn't control.

**Simple Explanation**

Before strong parameters, `User.new(params[:user])` would set every attribute present in the params hash. If your `User` model had an `admin` column and the form only showed name and email, an attacker could still send `user[admin]=true` with curl and make themselves an admin.

Strong parameters close that by requiring an explicit allow-list per action. Anything not permitted is simply dropped — no error. (`require` is the exception: it raises `ActionController::ParameterMissing` if the top-level key is missing.)

Note this is a **controller** concern, not a model one. The model doesn't know which params were safe, so the controller is the real security boundary.

**Example**

```ruby
class UsersController < ApplicationController
  def update
    @user = User.find(params[:id])
    @user.update!(user_params)
    redirect_to @user
  end

  private

  def user_params
    # only :name and :email can ever be mass-assigned here,
    # even if the request body also includes user[admin]=true
    params.require(:user).permit(:name, :email)
  end
end

# Without strong params (vulnerable, pre-Rails-4 style):
# User.new(params[:user]) # attacker-controlled admin=true would be set directly
```

### 47. How does CSRF protection work in Rails, and what does `authenticity_token` actually do?

**Short Answer**

Rails puts a random per-session token (`authenticity_token`) in every form and AJAX request. Any non-GET request without a matching token is rejected.

This stops CSRF (Cross-Site Request Forgery), where a malicious site tricks a logged-in user's browser into submitting a request to your app.

**Simple Explanation**

CSRF works because browsers automatically attach cookies — including your session cookie — even to requests triggered by someone else's website. Without protection, a hidden auto-submitting form on `evil.com` pointing at `yourapp.com/transfer_money` would succeed.

Rails defeats this by requiring a second secret the attacker can't know: a token derived from the session, added to the page by `csrf_meta_tags` and included as a hidden field in every `form_with`. If the submitted token doesn't match the session's, Rails raises `ActionController::InvalidAuthenticityToken`.

API-only apps usually skip this and use token auth (JWT, API keys) instead, because CSRF specifically exploits **cookie-based** sessions. That's why `ActionController::API` doesn't include CSRF protection at all.

**Example**

```ruby
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end

# app/views/layouts/application.html.erb
# <head>
#   <%= csrf_meta_tags %>
#   <!-- renders:
#   <meta name="csrf-param" content="authenticity_token">
#   <meta name="csrf-token" content="RANDOM_TOKEN_TIED_TO_SESSION">
#   -->
# </head>

# form_with automatically includes a hidden authenticity_token field
# <%= form_with model: @post do |f| %>
#   <!-- <input type="hidden" name="authenticity_token" value="..."> -->
# <% end %>

# API-only controllers often disable this and use token auth instead
class Api::BaseController < ActionController::API
  # ActionController::API doesn't include CSRF protection at all —
  # it's meant for token-authenticated, non-cookie-session clients.
end
```

### 48. How would you version a Rails API, and what are the trade-offs?

**Short Answer**

Three common options:

- **URL path** — `/api/v1/posts`. Most common: easy to read in logs, easy to curl, and caches work because the URL itself differs.
- **Header** — `Accept: application/vnd.myapp.v1+json`. Cleaner URLs, closer to REST purism, but harder to test by hand.
- **Query param** — `?version=1`. Easiest to bolt on, but the weakest signal.

**Simple Explanation**

Path versioning is what most teams actually ship. It's visible everywhere, trivial to route with `namespace :v1`, and any HTTP cache keys off the URL so versions can't collide. The downside is that the same conceptual resource now has multiple URLs.

Header versioning keeps one URL per resource, which is closer to REST's original intent. But you can't just point a browser at it, curl needs an extra flag, and caches or CDNs won't tell versions apart unless you configure them to vary on that header.

Query-param versioning is easy to forget, and many caching layers ignore query params by default.

In practice: path versioning for public APIs with many external clients; header versioning when you control all the clients and want one clean URL space.

**Example**

```ruby
# Path-based versioning — namespace + module, most common in practice
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :posts, only: [:index, :show]
  end
  namespace :v2 do
    resources :posts, only: [:index, :show]
  end
end

module Api
  module V1
    class PostsController < ApplicationController
      def show
        render json: Post.find(params[:id]).as_json(only: [:id, :title])
      end
    end
  end
  module V2
    class PostsController < ApplicationController
      def show
        # v2 adds an "author" field without breaking v1 clients
        post = Post.find(params[:id])
        render json: post.as_json(only: [:id, :title]).merge(author: post.author.name)
      end
    end
  end
end

# Header-based versioning alternative:
# request.headers["Accept"] #=> "application/vnd.myapp.v2+json"
```

### 49. What does the `--api` flag do when generating a Rails app, and why strip those things out?

**Short Answer**

`rails new myapp --api` builds a leaner app for JSON-only APIs. It skips view rendering, the asset pipeline, and session/cookie/CSRF middleware, and generators stop creating views and helpers.

**Simple Explanation**

An API-only app inherits from `ActionController::API` instead of `ActionController::Base`. That base class includes only what a JSON API needs — strong params, error handling, basic auth, caching, MIME negotiation — and leaves out the HTML-serving parts like view helpers, flash messages, CSRF tokens, and cookie sessions.

The middleware stack is trimmed too, so there's no `ActionDispatch::Flash` and no need for `Rack::MethodOverride` when clients send real PATCH and DELETE verbs.

In practice this means faster boot, a smaller memory footprint, and a nudge toward a stateless, token-authenticated design. You can always add a piece back (like `ActionController::Cookies`) if you genuinely need it — the flag is a sensible default, not a lock-in.

**Example**

```ruby
# generated when you run: rails new my_api --api

# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
end

# config/application.rb
module MyApi
  class Application < Rails::Application
    config.api_only = true
  end
end

# A typical API-only controller — no views, just JSON
class Api::PostsController < ApplicationController
  def index
    render json: Post.all.select(:id, :title, :created_at)
  end
end

# $ bin/rails generate model Comment body:text post:references
# generates model + migration, but no view/helper/asset files —
# api_only apps configure generators to skip them by default
```

**— Active Record: Associations & Querying —**

### 50. Explain `has_many`, `belongs_to`, `has_one`, `has_and_belongs_to_many`, and `has_many :through` — when do you reach for `:through` over HABTM?

**Short Answer**

- `belongs_to` — this table holds the foreign key.
- `has_many` / `has_one` — the *other* table holds the foreign key.
- `has_and_belongs_to_many` (HABTM) — many-to-many through a plain join table with no model.
- `has_many :through` — many-to-many through a real join **model**.

Use `:through` as soon as the relationship itself needs its own columns, validations, or callbacks.

**Simple Explanation**

`belongs_to :author` means this table has an `author_id` column. `has_many :posts` on `Author` means the *posts* table has the `author_id`, and Rails just runs `Post.where(author_id: id)`. `has_one` is the same idea but expects at most one match.

HABTM is the quick many-to-many: a bare join table (`posts_tags`, no `id`, no model) that Rails manages for you. Fine for a genuinely dumb link like tagging.

`has_many :through` uses a full model for the join — say `Enrollment` between `Student` and `Course`. That model can have its own validations, callbacks, timestamps, and extra columns like `enrolled_at` or `grade`. HABTM structurally cannot do any of that.

In real apps, almost every many-to-many eventually wants one of those things, so `:through` is the safer default. Many teams just use it everywhere for consistency.

**Example**

```ruby
# belongs_to / has_many — one-to-many
class Author < ApplicationRecord
  has_many :posts
end
class Post < ApplicationRecord
  belongs_to :author # posts table has author_id
end

# has_one — one-to-one
class Supplier < ApplicationRecord
  has_one :account
end

# HABTM — dumb pivot, no extra data on the join
class Post < ApplicationRecord
  has_and_belongs_to_many :tags # needs a posts_tags join table, no model
end
class Tag < ApplicationRecord
  has_and_belongs_to_many :posts
end

# has_many :through — join has its own data/behavior
class Student < ApplicationRecord
  has_many :enrollments
  has_many :courses, through: :enrollments
end
class Course < ApplicationRecord
  has_many :enrollments
  has_many :students, through: :enrollments
end
class Enrollment < ApplicationRecord
  belongs_to :student
  belongs_to :course
  validates :grade, inclusion: { in: %w[A B C D F], allow_nil: true }
  before_create { self.enrolled_at = Time.current }
end
```

### 51. What's the N+1 query problem? Show a concrete before/after.

**Short Answer**

An N+1 happens when you load N records and then run one extra query per record to fetch an association — so 1 query becomes 1 + N.

Fix it by loading the association up front with `includes`.

**Simple Explanation**

Associations are lazy. `post.author` doesn't hit the database until you actually call it.

So if you fetch 50 posts and loop over them printing `post.author.name`, Rails runs 1 query for the posts and then 50 more — one per post. That's 51 queries where 2 would do.

This is invisible in development with a handful of seed rows, and shows up as a real production incident once the table has thousands of records.

`includes(:author)` tells Active Record "I'm going to need this", so it's fetched in a batch. The `bullet` gem detects N+1s automatically in development and test.

**Example**

```ruby
# BEFORE — N+1: 1 query for posts, then 1 query per post for its author
Post.limit(50).each do |post|
  puts "#{post.title} by #{post.author.name}"
end
# SELECT * FROM posts LIMIT 50
# SELECT * FROM authors WHERE id = 1
# SELECT * FROM authors WHERE id = 2
# ... 50 times

# AFTER — eager loaded: 2 queries total, regardless of how many posts
Post.includes(:author).limit(50).each do |post|
  puts "#{post.title} by #{post.author.name}"
end
# SELECT * FROM posts LIMIT 50
# SELECT * FROM authors WHERE id IN (1, 2, 3, ..., 50)
```

### 52. What's the real difference between `includes`, `preload`, `eager_load`, and `joins`?

**Short Answer**

- `preload` — always runs a **separate query** per association. You can't filter on that association.
- `eager_load` — always uses a **LEFT OUTER JOIN** in one query. You *can* filter or sort on it.
- `includes` — lets Rails pick between those two automatically.
- `joins` — an **INNER JOIN** used only for filtering. It does **not** load the association into memory.

**Simple Explanation**

`Post.preload(:comments)` runs two queries: posts, then `comments WHERE post_id IN (...)`. Since comments aren't in the same query, there's nothing to put a `WHERE` on.

`Post.eager_load(:comments)` runs one query with a LEFT OUTER JOIN, so you can filter and order by comment columns. The cost is a wide result set when a parent has many children.

`Post.includes(:comments)` behaves like `preload` by default (cheaper). But if you reference the association in a `where` or `order`, Rails automatically switches to the join version. That's useful but can surprise you — check the SQL log to see which one actually ran.

`Post.joins(:comments)` filters posts using an INNER JOIN, but does **not** populate `post.comments`. Calling `post.comments` afterwards still fires a fresh query, so you can still get an N+1. Use `joins` when you only care about filtering.

**Example**

```ruby
# preload — 2 queries, cannot filter on comments' columns
Post.preload(:comments)
# SELECT * FROM posts
# SELECT * FROM comments WHERE post_id IN (1,2,3,...)

# eager_load — 1 query, CAN filter/order on comments' columns
Post.eager_load(:comments).where(comments: { approved: true })
# SELECT posts.*, comments.* FROM posts
# LEFT OUTER JOIN comments ON comments.post_id = posts.id
# WHERE comments.approved = true

# includes — Rails picks preload here (no condition on comments)
Post.includes(:comments)
# same 2 queries as preload

# includes — but referencing comments in a condition flips it to eager_load's join
Post.includes(:comments).where(comments: { approved: true })
# SELECT posts.*, comments.* FROM posts
# LEFT OUTER JOIN comments ON comments.post_id = posts.id
# WHERE comments.approved = true

# joins — filters posts, does NOT load comments into memory
Post.joins(:comments).where(comments: { approved: true }).distinct
# SELECT DISTINCT posts.* FROM posts
# INNER JOIN comments ON comments.post_id = posts.id
# WHERE comments.approved = true
# post.comments here would still trigger a brand-new query
```

### 53. How does Active Record's lazy query evaluation work, and what's the real difference between `exists?`, `any?`, `count`, `size`, and `length`?

**Short Answer**

An `ActiveRecord::Relation` is just a query builder — no SQL runs until something forces it.

For counting and existence checks:

- `.exists?` → cheap `SELECT 1 ... LIMIT 1`
- `.any?` → like `exists?` if not loaded; checks memory if already loaded
- `.count` → always runs `SELECT COUNT(*)`, even if the records are already in memory
- `.size` → smart: `COUNT(*)` if not loaded, in-memory count if loaded
- `.length` → always loads all records first, then counts them in Ruby

**Simple Explanation**

`User.where(active: true)` runs no SQL. It builds a chainable query object. SQL only fires when you iterate, call `.to_a`, call `.load`, or ask something that needs the rows.

That laziness is why you can build a query across several conditionals and still pay for only one round trip.

The method you pick for counting matters a lot:

- Use `.exists?` for a plain "is there at least one?" check.
- Use `.size` when the collection might already be loaded — like `post.comments.size` after rendering the comments.
- Avoid `.count` on a relation you've already loaded and are about to iterate, because you'll pay for two round trips instead of one.
- Avoid `.length` on anything large, since it loads every row just to count them.

**Example**

```ruby
relation = User.where(active: true)         # no SQL yet — just builds the query

relation.exists?                             # SELECT 1 FROM users WHERE active = true LIMIT 1
relation.any?                                 # same as exists? here, since relation isn't loaded

relation.count                                # SELECT COUNT(*) FROM users WHERE active = true
relation.size                                 # same COUNT(*) query, since relation isn't loaded

users = relation.to_a                         # NOW it hits the DB, loads full rows into memory
users.size                                    # 0 queries — Array#size on the loaded rows
users.count                                   # 0 queries too (Enumerable#count on an Array)
relation.count                                # BUT calling .count again on the relation still
                                               # re-runs SELECT COUNT(*) — it doesn't know about `users`

post.comments.load                            # force-load the association
post.comments.size                            # in-memory count, no query
post.comments.count                           # still issues a fresh SELECT COUNT(*) query — avoid this
                                               # after you've already loaded the association
```

### 54. What are `find_each`, `find_in_batches`, and `in_batches` for, and what trade-off do you accept for the memory safety?

**Short Answer**

They walk through a large table in chunks instead of loading every row into memory at once.

- `find_each` — yields one record at a time (default batch size 1000)
- `find_in_batches` — yields an array per batch
- `in_batches` — yields a Relation per batch, so you can call `update_all` on it

The trade-off: they force ordering by primary key, so you lose custom `order` and `limit`, and rows changed mid-run can be missed.

**Simple Explanation**

`User.all.each { ... }` loads **every** row into memory before it starts. Fine for a few thousand rows, a production incident for ten million.

`find_each` fixes that by fetching in batches and holding only one batch at a time.

`find_in_batches` gives you the whole array per batch, which is handy for bulk processing.

`in_batches` gives you a Relation per chunk, so you can run `relation.update_all(...)` directly — one UPDATE per chunk instead of per row.

The cost: all three need to order by primary key internally to track where the last batch ended, so you can't pass your own `order`. And because batches are separate queries spread over time, a row deleted between batches can be silently skipped. That's fine for bulk maintenance jobs, not for a strict point-in-time report.

**Example**

```ruby
# BAD — loads all 10 million rows into memory before doing anything
User.all.each { |u| u.recalculate_score! }

# GOOD — streams in batches of 1000, low constant memory
User.find_each(batch_size: 1000) do |user|
  user.recalculate_score!
end

# find_in_batches — yields arrays, good for bulk operations per batch
User.find_in_batches(batch_size: 500) do |batch|
  UserScoreMailer.weekly_digest(batch.map(&:id)).deliver_later
end

# in_batches — yields a Relation, lets you bulk-update per chunk
User.where(active: false).in_batches(of: 1000) do |relation|
  relation.update_all(archived_at: Time.current)
  sleep(0.1) # throttle to avoid hammering a replica/primary under load
end
```

### 55. Polymorphic associations vs Single Table Inheritance (STI) — what are the long-term problems with each?

**Short Answer**

- **Polymorphic** (`belongs_to :commentable, polymorphic: true`) — one model can belong to several others via a `_type` + `_id` pair. The downside: the database can't enforce a foreign key on it.
- **STI** (a `type` column on one shared table) — subclasses share a table. The downside: the table fills up with columns only some subtypes use.

**Simple Explanation**

**Polymorphic:** a `Comment` with `commentable_type: "Post"` and `commentable_id: 5` can belong to a Post, a Photo, or anything else. It's flexible and avoids a separate comments table per type.

The catch is that no `FOREIGN KEY` can point at "whichever table the type column happens to name". So an orphaned or mistyped row is purely an application bug the database can't catch, and a comment pointing at a deleted or renamed model becomes silent data corruption. Indexing and joins across types get awkward too.

**STI:** a `Vehicle` table with a `type` column holding `"Car"`, `"Truck"`, `"Motorcycle"`. `Vehicle.all` returns all subtypes with the right classes, and they share associations and validations for free.

The long-term problem is column sprawl. As subtypes diverge you add `trailer_hitch_weight` for Truck and `sidecar` for Motorcycle, most rows have most columns `NULL`, and eventually the table stops representing one kind of thing.

Both patterns are fine early and both tend to get replaced once the divergence grows.

**Example**

```ruby
# Polymorphic — no real FK possible
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
  # columns: commentable_type (string), commentable_id (bigint) — no FK constraint
end
class Post < ApplicationRecord
  has_many :comments, as: :commentable
end
class Photo < ApplicationRecord
  has_many :comments, as: :commentable
end

# STI — one table, growing nullable-column problem
class Vehicle < ApplicationRecord
  # single `vehicles` table with a `type` string column
end
class Car < Vehicle
end
class Truck < Vehicle
  # trailer_hitch_weight only makes sense for Truck, but the column
  # lives on the shared `vehicles` table and is NULL for every Car row
end

Vehicle.create!(type: "Truck", trailer_hitch_weight: 5000)
Vehicle.all # returns a mix of Car and Truck instances, correctly typed
```

### 56. What do `counter_cache`, `touch`, and `dependent: :destroy`/`:delete_all`/`:nullify` actually cost you?

**Short Answer**

- `counter_cache` — stores a count in a column so you skip `COUNT(*)`. Cost: an extra write on every create and destroy, and it can drift if rows are inserted outside Active Record.
- `touch` — bumps the parent's `updated_at` when a child is saved. Cost: an extra UPDATE per child save, and write contention on a busy parent row.
- `dependent: :destroy` — loads each child and runs its callbacks. Safe but O(N) queries.
- `dependent: :delete_all` — one fast SQL DELETE, but **skips all callbacks and validations**.
- `dependent: :nullify` — one UPDATE that sets the foreign key to NULL. Children survive.

**Simple Explanation**

`counter_cache: true` with a `comments_count` column means `post.comments.size` reads a stored integer instead of counting rows every time. Cheap reads, slightly more expensive writes.

`touch: true` is what makes Russian-doll caching work — changing a comment bumps the post's `updated_at`, which busts the post's cached fragment. Just remember it's an extra write.

For `dependent:` on the parent side, pick based on whether the children's callbacks matter:

- `:destroy` is correct when children have their own `before_destroy` logic or their own `dependent: :destroy` children. It's slow for thousands of records.
- `:delete_all` is one statement and very fast, but any cleanup living in a child callback silently never runs.
- `:nullify` is right when a child can legitimately exist without that parent.

**Example**

```ruby
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
  # comments_count column must exist on posts for this to work:
end
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true, touch: true
end

post.comments.size # reads post.comments_count, no COUNT(*) query

# dependent: :destroy — N queries, runs every comment's callbacks/validations
post.destroy # SELECT comments; then DELETE comment 1; DELETE comment 2; ...

# dependent: :delete_all — 1 query, skips comment callbacks entirely
class Post < ApplicationRecord
  has_many :comments, dependent: :delete_all
end
post.destroy # DELETE FROM comments WHERE post_id = ?  (single statement)

# dependent: :nullify — children survive, FK just cleared
class Author < ApplicationRecord
  has_many :posts, dependent: :nullify
end
author.destroy # UPDATE posts SET author_id = NULL WHERE author_id = ?
```

**— Active Record: Validations, Callbacks & Transactions —**

### 57. What are Rails validations, and how do custom validators work?

**Short Answer**

Validations are rules on the model (`validates :email, presence: true`) that run before a record is saved. If one fails, the save is blocked and the problem is added to `record.errors`.

Custom validators let you either write a one-off check (`validate :method_name`) or a reusable validator class.

**Simple Explanation**

The built-ins cover the common cases: `presence`, `uniqueness`, `length`, `numericality`, `format`, `inclusion`.

They're plain Ruby checks that run in your app process, not database constraints. That makes them fast to write and gives friendly per-field error messages — but it also means they can be bypassed by raw SQL, another process, or a race condition (more on that in the next question).

For a rule that's specific to one model, `validate :custom_method` plus `errors.add` is enough.

For a rule you want to reuse across models (like "must be a valid US ZIP code"), write an `ActiveModel::EachValidator` subclass so it plugs into the normal `validates` syntax.

**Example**

```ruby
# simple inline custom validation
class Event < ApplicationRecord
  validate :end_date_after_start_date

  private

  def end_date_after_start_date
    return if end_date.blank? || start_date.blank?
    errors.add(:end_date, "must be after the start date") if end_date <= start_date
  end
end

# reusable custom validator class
# app/validators/zip_code_validator.rb
class ZipCodeValidator < ActiveModel::EachValidator
  ZIP_REGEX = /\A\d{5}(-\d{4})?\z/

  def validate_each(record, attribute, value)
    unless value.to_s.match?(ZIP_REGEX)
      record.errors.add(attribute, "is not a valid ZIP code")
    end
  end
end

class Address < ApplicationRecord
  validates :zip, zip_code: true
end
```

### 58. `validates :email, uniqueness: true` still lets two simultaneous requests insert duplicates — why, and what actually prevents it?

**Short Answer**

Because `uniqueness: true` runs a `SELECT` first and the `INSERT` after. Two requests can both run the SELECT, both see nothing, and both insert.

The only real fix is a **database unique index**, which makes the second insert fail no matter what.

**Simple Explanation**

The uniqueness validation is just "ask the database if a matching row exists" at the moment `valid?` runs. It is not atomic with the insert that follows.

If two requests for the same email arrive close together, both run `SELECT ... WHERE email = ?`, both get zero rows, both conclude it's safe, and both insert. Now you have two users with the same email even though the validation "passed" for both.

This isn't theoretical — it happens with double-submits, retried requests, and multiple app servers.

A unique index closes it for real, because uniqueness is enforced by the database itself, atomically, no matter how many processes are racing.

Keep the validation too — it gives a nice error message in the normal case. Just rescue `ActiveRecord::RecordNotUnique` as the actual backstop.

**Example**

```ruby
# Migration — the real fix
class AddUniqueIndexToUsersEmail < ActiveRecord::Migration[7.1]
  def change
    add_index :users, :email, unique: true
  end
end

class User < ApplicationRecord
  validates :email, uniqueness: true # fast, friendly — but NOT race-safe alone
end

# Handling the race: catch the DB-level failure as the real backstop
def create_user(email:)
  User.create!(email: email)
rescue ActiveRecord::RecordNotUnique
  # the unique index caught what the validation's SELECT-then-INSERT missed
  raise ActiveRecord::RecordInvalid, "Email already taken"
end
```

### 59. Walk through Active Record's callback execution order for `create` and `update`. What is `around_save` for?

**Short Answer**

For a create:

`before_validation` → validations → `after_validation` → `before_save` → `before_create` → **INSERT** → `after_create` → `after_save` → **COMMIT** → `after_commit`

An update is the same shape, with `before_update` / `after_update` instead of the create pair.

`around_save` wraps the whole write, so you can run code before and after it in one callback.

**Simple Explanation**

The `_save` callbacks fire on both create and update. The `_create` and `_update` ones only fire for that specific operation.

`after_commit` is different from the rest — it doesn't run as part of the save at all. It runs only after the surrounding database transaction actually commits. (The next question covers why that distinction matters so much.)

`around_save` (and `around_create` / `around_update`) let you wrap the write in a block, calling `yield` where the actual database write should happen. Useful for timing instrumentation or a lock that must be released no matter how the save turns out.

**Example**

```ruby
class Order < ApplicationRecord
  before_validation :normalize_email
  before_save       :calculate_total
  before_create     :generate_confirmation_number
  after_create      :log_creation
  after_save        :update_search_index
  after_commit      :send_confirmation_email, on: :create

  around_save :time_the_save

  private

  def normalize_email
    self.email = email&.downcase&.strip
  end

  def time_the_save
    started_at = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield # the actual INSERT/UPDATE happens here
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - started_at
    Rails.logger.info("Order save took #{duration}s")
  end

  def calculate_total
    self.total = line_items.sum(&:price)
  end

  def generate_confirmation_number
    self.confirmation_number = SecureRandom.hex(6)
  end

  def log_creation
    Rails.logger.info("Order #{id} created")
  end

  def update_search_index
    SearchIndexJob.perform_later(id)
  end

  def send_confirmation_email
    OrderMailer.confirmation(self).deliver_later
  end
end

# Execution order for Order.create!(...):
# before_validation -> validations -> after_validation ->
# before_save -> before_create -> [INSERT] -> after_create ->
# after_save -> [TRANSACTION COMMITS] -> after_commit
```

### 60. Why are `after_create`/`after_save` callbacks that call an external API risky, and why is `after_commit` correct instead?

**Short Answer**

`after_create` and `after_save` run **inside** the still-open transaction. If something later rolls that transaction back, the database row disappears — but the SMS you sent or the card you charged is already done, and you can't undo it.

`after_commit` only fires once the transaction has actually committed, so the data is guaranteed to exist before you cause any outside effect.

**Simple Explanation**

`save` and `create` wrap the whole callback chain in a transaction. If your `after_create` sends an SMS and then anything later raises, the transaction rolls back — the row is gone, but the SMS was already sent.

`after_commit` fixes this structurally. It's queued during the transaction and only actually runs after the COMMIT succeeds. By the time your external call runs, the data is definitely saved.

The trade-off: if the transaction never commits, the `after_commit` callback simply never runs. That's exactly what you want for side effects that should only happen for real, saved data.

One testing gotcha: RSpec's default transactional fixtures wrap each test in a transaction that's rolled back, never committed. Modern Rails does fire `after_commit` in tests anyway, but if your suite is configured unusually you may need `DatabaseCleaner` with a truncation strategy for those specs.

**Example**

```ruby
# RISKY — runs inside the open transaction
class Order < ApplicationRecord
  after_create :charge_card # if a later callback/validation rolls back the
                             # transaction, the card is STILL charged
  private
  def charge_card
    PaymentGateway.charge!(total_cents: total_cents, card: card_token)
  end
end

# CORRECT — only fires after the transaction actually commits
class Order < ApplicationRecord
  after_commit :charge_card, on: :create

  private
  def charge_card
    PaymentGateway.charge!(total_cents: total_cents, card: card_token)
  end
end

# Spec gotcha — after_commit callbacks need special handling
RSpec.describe Order do
  it "charges the card after commit" do
    allow(PaymentGateway).to receive(:charge!)
    # with transactional fixtures (the RSpec default), the transaction
    # is rolled back, not committed — after_commit will NOT fire here
    # unless you configure DatabaseCleaner truncation for this example
    # or use `test_after_commit`-aware helpers.
    Order.create!(total_cents: 500, card_token: "tok_visa")
    expect(PaymentGateway).to have_received(:charge!)
  end
end
```

### 61. `save`/`save!`, `update`/`update!`, `create`/`create!` — what's the real difference between the bang and non-bang versions?

**Short Answer**

- Non-bang (`save`, `update`) → returns `true`/`false` and puts problems in `.errors`. Nothing raises.
- Bang (`save!`, `update!`, `create!`) → raises `ActiveRecord::RecordInvalid` on failure.

Use bang when a failure is a bug you want to see loudly (background jobs, scripts). Use non-bang when a failure is a normal user outcome you need to handle (a form submission).

**Simple Explanation**

`user.save` returns `false` on failure and nothing raises. If you forget to check the return value, the failure is silently swallowed. That's exactly right in a controller where `if @user.save ... else render :new ... end` is a normal branch.

`user.save!` raises instead. That's what you want in a Sidekiq job or a rake task — you don't want a swallowed `false` quietly doing nothing three calls up the stack. You want the job to fail loudly, retry, and alert someone.

`create` is a trap worth remembering: `Model.create(invalid_attrs)` still **returns an object**, not `false`. So `if User.create(...)` is always truthy. You have to check `.persisted?` or `.errors.any?`. `create!` avoids the ambiguity by raising.

**Example**

```ruby
# Non-bang — expected failure path, handled explicitly
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user
    else
      render :new, status: :unprocessable_entity # @user.errors has the details
    end
  end
end

# Bang — failure is a bug/anomaly that should raise loudly
class BackfillUserScoresJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)
    user.update!(score: ScoreCalculator.call(user)) # raises if invalid — job retries/alerts
  end
end

# The create() trap
user = User.create(email: nil) # returns an OBJECT even though it's invalid
user.persisted? #=> false
user.errors.full_messages #=> ["Email can't be blank"]
# if you'd written `if User.create(...)` thinking it behaves like save, it's ALWAYS truthy
```

### 62. `update_column`/`update_columns`/`update_attribute` vs `update` — which skip validations and callbacks, and when is that dangerous?

**Short Answer**

| Method | Validations | Callbacks | Touches `updated_at` |
|---|---|---|---|
| `update` | ✅ runs | ✅ runs | ✅ yes |
| `update_attribute` | ❌ skipped | ✅ runs | ✅ yes |
| `update_column(s)` | ❌ skipped | ❌ skipped | ❌ no |

Reaching for the skipping versions without a clear reason is a common way to corrupt data or bypass business rules.

**Simple Explanation**

`update_column` and `update_columns` go straight to SQL. No `valid?`, no callbacks, no automatic `updated_at`. They exist for narrow cases: fixing one column on a record you know is valid, or a maintenance script over millions of rows where you've deliberately decided callbacks shouldn't run.

`update_attribute` is the odd one and arguably the most dangerous: it skips validations but **still runs callbacks**. So you can end up with an invalid record that still triggers `after_save` side effects as if everything were fine.

The real danger with all three is that downstream code assumes things like "if this record exists, it passed validation" or "if status changed, the notification fired". Those assumptions silently become false.

Default to `update` / `update!`. Use the others only with a comment explaining why you're skipping.

**Example**

```ruby
user = User.find(1)

# update — full validation + callback chain, the safe default
user.update(email: "new@example.com")

# update_attribute — SKIPS validations, but callbacks still fire (dangerous combo)
user.update_attribute(:email, "not-a-valid-email") # bypasses format validation!
# after_save callbacks (e.g. a search-index sync) still run on this invalid data

# update_column — skips BOTH validations and callbacks, single UPDATE, no updated_at bump
user.update_column(:last_seen_at, Time.current)

# update_columns — same as update_column but for multiple attributes at once
user.update_columns(last_seen_at: Time.current, sign_in_count: user.sign_in_count + 1)

# Legitimate use: a data-repair script that intentionally wants no side effects
User.where(status: nil).find_each do |u|
  u.update_column(:status, "unknown") # deliberate: don't re-trigger onboarding emails
end
```

### 63. How do you roll back a migration safely in production, and what's the difference between reversible and irreversible data migrations?

**Short Answer**

`bin/rails db:rollback` reverses the last migration by working out the opposite of each `change` statement. That works fine for pure schema changes.

A **data** migration usually can't be reversed, because undoing `update_all(status: "archived")` would require knowing what each row's status was before — and that information is gone.

**Simple Explanation**

Schema migrations written with `change` are reversible for free, because Rails knows the inverse of common operations: add a column → drop it, add an index → remove it, rename → rename back.

Data migrations are different. If a migration overwrites values, there's no mechanical way to get the old ones back.

For those, write explicit `up` and `down` methods and either accept that `down` is lossy, or raise `ActiveRecord::IrreversibleMigration` and plan to fix forward instead.

In production specifically: don't treat `db:rollback` as your incident plan for a bad data migration. Rolling back after live writes have happened can lose those writes. Writing a new corrective migration is safer and leaves an audit trail.

**Example**

```ruby
# Reversible — schema-only, `change` infers the down automatically
class AddStatusToOrders < ActiveRecord::Migration[7.1]
  def change
    add_column :orders, :status, :string, default: "pending"
    add_index :orders, :status
  end
end
# bin/rails db:rollback cleanly runs: remove_index :orders, :status; remove_column :orders, :status

# Irreversible data migration — needs explicit up/down, down is lossy at best
class BackfillOrderStatus < ActiveRecord::Migration[7.1]
  def up
    Order.where(status: nil).in_batches.update_all(status: "pending")
  end

  def down
    # We don't actually know which rows were nil before — this is a BEST EFFORT,
    # not a true reversal. In practice: don't rely on this, fix forward instead.
    raise ActiveRecord::IrreversibleMigration
  end
end
```

### 64. Explain nested Active Record transactions — why does an inner `transaction do` by default join the outer one, and what does `requires_new: true` change?

**Short Answer**

By default, a nested `transaction do` block does **not** create a separate transaction — it joins the outer one. Postgres and MySQL don't support truly nested transactions.

`transaction(requires_new: true)` creates a real SAVEPOINT, so the inner block can roll back on its own while the outer transaction continues.

**Simple Explanation**

This one trips up a lot of people. `ActiveRecord::Rollback` raised inside a plain nested `transaction do` looks like it should only undo the inner block. But without `requires_new: true`, there's no savepoint — Rails just treats it all as one transaction.

The genuinely dangerous version: an inner block that rescues its own exception and returns normally. Rails never learns anything went wrong, the transaction commits, and you've saved data you believed was rolled back.

`requires_new: true` fixes this by issuing a real `SAVEPOINT` / `ROLLBACK TO SAVEPOINT`. The inner block becomes an independent unit that can fail and roll back while the outer transaction's earlier work survives.

**Example**

```ruby
# WITHOUT requires_new — inner "rollback" actually undoes the WHOLE transaction
Order.transaction do
  order.update!(status: "paid")
  begin
    Order.transaction do # joins the SAME transaction, not a new one
      raise "inventory service down"
    end
  rescue => e
    Rails.logger.error(e.message)
    # looks like we "handled" it locally — but if this rescue happened
    # OUTSIDE the inner transaction block, order.update! above would still
    # get committed fine. The real danger is the reverse case below.
  end
end

# THE REAL TRAP — swallowing the exception INSIDE the inner block
# with no requires_new means Rails never even sees a rollback signal:
Order.transaction do
  order.update!(status: "paid")
  Order.transaction do
    begin
      InventoryService.reserve!(order) # raises
    rescue InventoryService::Error
      nil # swallowed silently — the transaction has no idea anything failed
    end
  end
  # order.status = "paid" COMMITS, even though inventory was never reserved!
end

# WITH requires_new — a real SAVEPOINT, inner failure truly isolated
Order.transaction do
  order.update!(status: "paid")
  begin
    Order.transaction(requires_new: true) do
      InventoryService.reserve!(order) # raises
      raise ActiveRecord::Rollback if inventory_insufficient?
    end
  rescue InventoryService::Error
    order.update!(status: "payment_failed") # outer transaction unaffected, can react
  end
end
```

### 65. What belongs in a database constraint (FK, `UNIQUE`, `CHECK`, `NOT NULL`) vs a Rails model validation — why do you want both?

**Short Answer**

- **Validations** — fast, friendly error messages for the normal case. But they only run when code goes through that model.
- **Database constraints** — enforced on every write, no matter what wrote the row.

Anything that must **never** be violated needs a constraint. Keep the validation alongside it purely for good UX.

**Simple Explanation**

A validation does nothing to stop: a raw SQL insert, another service writing to the same table, a console `update_column`, or a race condition between two requests.

A database constraint is enforced by the database engine itself, for every write, through every path, always.

The mental model:

- Validations are for **user experience** — catch the mistake early and show a nice message.
- Constraints are for **data integrity** — guarantee the rule actually holds.

You want both. A validation alone is optimistic and bypassable. A constraint alone gives you an ugly `ActiveRecord::StatementInvalid` instead of a friendly form error.

**Example**

```ruby
# Migration — the DB-level guarantee, unconditional
class AddConstraintsToOrders < ActiveRecord::Migration[7.1]
  def change
    add_column :orders, :customer_id, :bigint
    add_foreign_key :orders, :customers            # can't reference a nonexistent customer
    change_column_null :orders, :customer_id, false # can't be NULL, ever
    add_check_constraint :orders, "total_cents >= 0", name: "orders_total_cents_positive"
    add_index :orders, :confirmation_number, unique: true
  end
end

# Model — the friendly, fast, app-level layer on top of the same rules
class Order < ApplicationRecord
  belongs_to :customer
  validates :customer, presence: true
  validates :total_cents, numericality: { greater_than_or_equal_to: 0 }
  validates :confirmation_number, uniqueness: true

  # A raw SQL insert, a rake task, or a race condition will still be caught
  # by the FK/CHECK/UNIQUE constraints above even if this validation is skipped.
end
```

### 66. Optimistic locking vs pessimistic locking in Active Record — walk through "two people editing the same record."

**Short Answer**

- **Optimistic locking** — a `lock_version` column. Both people edit freely; whoever saves second gets `ActiveRecord::StaleObjectError` and has to reload. No blocking.
- **Pessimistic locking** — `SELECT ... FOR UPDATE` via `.lock`. The second person's query waits until the first transaction finishes.

**Simple Explanation**

Say two support agents open the same customer record.

**Optimistic:** Rails adds `WHERE lock_version = ?` to the UPDATE and bumps the column on success. Agent A loads it (version 3), Agent B loads it (version 3). A saves first — now version 4. B saves next, but their `UPDATE ... WHERE lock_version = 3` matches zero rows, so Rails raises `StaleObjectError` instead of silently overwriting A's change. Your app then decides: show a conflict screen, merge, or retry.

This is cheap (no locks held, nobody blocked) and right when conflicts are rare.

**Pessimistic:** `Customer.lock.find(id)` issues `SELECT ... FOR UPDATE`. Any other transaction trying to lock that row simply waits until the first one commits. Use it when conflicts are likely and correctness matters more than throughput — the classic case is decrementing inventory, where you want the second request to *wait* and see the updated number rather than read a stale one.

**Example**

```ruby
# Optimistic locking — needs a lock_version column (add_column :customers, :lock_version, :integer, default: 0)
class Customer < ApplicationRecord
end

agent_a_copy = Customer.find(1) # lock_version: 3
agent_b_copy = Customer.find(1) # lock_version: 3

agent_a_copy.update!(phone: "555-0100") # succeeds, lock_version becomes 4

begin
  agent_b_copy.update!(email: "new@example.com") # still thinks lock_version is 3
rescue ActiveRecord::StaleObjectError
  # someone else changed this record first — reload and let the user decide
  agent_b_copy.reload
  flash[:alert] = "This record was updated by someone else. Please review and retry."
end

# Pessimistic locking — SELECT ... FOR UPDATE, blocks concurrent access
Customer.transaction do
  customer = Customer.lock.find(1) # other transactions trying to lock this row now WAIT
  customer.update!(vip_status: "gold")
end # lock released on commit

# The classic inventory example — see the race-condition debugging question
# later in this guide for the full pattern.
```

### 67. Efficient bulk operations — `insert_all`/`upsert_all`/`update_all`/`delete_all` vs looping and calling `.save` — what do you lose, and when is that trade acceptable?

**Short Answer**

`insert_all`, `upsert_all`, `update_all`, and `delete_all` run **one SQL statement** for the whole batch, which is dramatically faster than looping and saving each record.

What you give up: validations, callbacks, `dependent: :destroy` cascades, and automatic `updated_at`.

**Simple Explanation**

`User.where(inactive: true).each(&:destroy)` builds a full object for every row and runs the whole callback chain per record. Correct when those callbacks matter, painfully slow at scale.

`User.where(inactive: true).delete_all` issues one DELETE. No callbacks, and no `dependent: :destroy` cascade — so if child rows need cleaning up, handle that yourself (a database `ON DELETE CASCADE`, or delete the children first).

`update_all` is the same idea for updates, with one surprise worth remembering: it does **not** bump `updated_at` automatically. You have to include it in the hash yourself.

`insert_all` / `upsert_all` (Rails 6+) take an array of attribute hashes and build one multi-row INSERT. `upsert_all` adds "update on conflict" behavior. Much faster than a create loop — but since validations don't run, you're responsible for making sure the data is already valid.

Worth it for large, machine-generated, already-validated batches. Not worth it when the per-record callbacks encode business logic you actually need.

**Example**

```ruby
# SLOW — one INSERT + full callback/validation chain per record
csv_rows.each do |row|
  Product.create!(sku: row[:sku], price_cents: row[:price_cents])
end

# FAST — one multi-row INSERT, no validations/callbacks
Product.insert_all(
  csv_rows.map { |row| { sku: row[:sku], price_cents: row[:price_cents], created_at: Time.current, updated_at: Time.current } }
)

# upsert_all — insert or update on conflict, one statement
Product.upsert_all(
  csv_rows.map { |row| { sku: row[:sku], price_cents: row[:price_cents], updated_at: Time.current } },
  unique_by: :index_products_on_sku
)

# update_all — one UPDATE, skips callbacks AND does not auto-touch updated_at
Order.where(status: "pending").where("created_at < ?", 30.days.ago)
     .update_all(status: "expired", updated_at: Time.current) # must set updated_at explicitly

# delete_all — one DELETE, skips dependent: :destroy cascades and callbacks
old_sessions = Session.where("expires_at < ?", Time.current)
old_sessions.delete_all # if Session had dependent children needing cleanup, handle that separately
```

**— Views, Concerns & Code Organization —**

### 68. What are partials and layouts in Rails views, and why use them?

**Short Answer**

- **Layout** — the outer shell around every page (nav, footer, `<head>`). Your view is dropped into it at `yield`.
- **Partial** — a reusable chunk of markup (`_form.html.erb`) you pull into multiple views with `render`.

Both exist to stop you repeating markup.

**Simple Explanation**

Most pages share the same header, nav, flash messages, and footer. A layout holds that once, and Rails wraps whatever the controller renders inside its `yield`.

Partials solve the sibling problem: the same markup used in several views — like a post card shown on both the index page and the "related posts" section of the show page.

Partials also render efficiently over a collection. `render partial: "post", collection: @posts` renders `_post.html.erb` once per post, and Rails can cache each one individually with `cached: true`. That's the foundation of Russian-doll caching, covered later.

**Example**

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
  <head><%= csrf_meta_tags %></head>
  <body>
    <%= render "shared/navbar" %>
    <%= yield %>
    <%= render "shared/footer" %>
  </body>
</html>

<%# app/views/posts/index.html.erb %>
<% @posts.each do |post| %>
  <%= render "post", post: post %>
<% end %>
<%# or equivalently, and more efficiently for large collections: %>
<%= render partial: "post", collection: @posts %>

<%# app/views/posts/_post.html.erb %>
<div class="post-card">
  <h2><%= post.title %></h2>
  <p><%= post.excerpt %></p>
</div>
```

### 69. What is `ActiveSupport::Concern`, and how is `included do ... end` different from a plain Ruby `module` + `include`?

**Short Answer**

`ActiveSupport::Concern` lets you put instance methods **and** class-level code (like `has_many`, `validates`, scopes) in one module.

The `included do ... end` block runs in the context of the class that includes it. With a plain Ruby module you'd have to write the `self.included(base)` hook by hand, and concerns that depend on each other can hit load-order errors.

**Simple Explanation**

A plain `include` only adds instance methods. To also run class-level code you need boilerplate like `def self.included(base); base.extend(ClassMethods); end` — easy to get wrong.

`ActiveSupport::Concern` handles three things for you:

1. `included do ... end` is deferred and evaluated in the host class, so `scope` and `validates` just work.
2. A nested `ClassMethods` module (or the `class_methods do` block) is extended automatically.
3. If concern A includes concern B, it makes sure B gets included into the host class first, instead of blowing up on load order.

**Example**

```ruby
# Plain Ruby module — need the manual included hook for class methods
module Archivable
  def self.included(base)
    base.extend(ClassMethods)
    base.scope :archived, -> { where.not(archived_at: nil) } # this line actually
    # can't even go here safely — `scope` needs to run in the class body context,
    # this is exactly the boilerplate ActiveSupport::Concern removes
  end

  def archive!
    update!(archived_at: Time.current)
  end

  module ClassMethods
    def archived_count
      archived.count
    end
  end
end

# ActiveSupport::Concern — class-level DSL just works inside included do...end
module Archivable
  extend ActiveSupport::Concern

  included do
    scope :archived, -> { where.not(archived_at: nil) }
    validates :archived_reason, presence: true, if: :archived?
  end

  def archive!
    update!(archived_at: Time.current)
  end

  def archived?
    archived_at.present?
  end

  class_methods do
    def archived_count
      archived.count
    end
  end
end

class Post < ApplicationRecord
  include Archivable
end
Post.archived_count
```

### 70. What is "concern soup," and how do you tell when a concern is the right tool vs a code smell?

**Short Answer**

"Concern soup" is dumping unrelated behavior into `app/models/concerns/` just to shrink a fat model file. The model still does too much — it's now spread across five files instead of one.

A concern is the right tool when it's a genuinely reusable, self-contained capability (like `Sluggable` or `Archivable`) used by more than one model. It's the wrong tool when it's really just half of `User` moved sideways.

**Simple Explanation**

Concerns are tempting because they make the model file shorter without you having to think hard about design. You find a natural-sounding boundary ("auth stuff", "notification stuff"), move it into a module, and the file shrinks.

But those modules still get included into `User` and still operate on `User`'s state. You haven't reduced coupling — you've hidden it. And now "what actually happens when a User is created?" means reading six files with callbacks scattered across all of them.

The tell: if a concern only makes sense in exactly one model, and its methods reach deep into that model's other attributes, it isn't reusable. It's an arbitrary slice of `User`.

Usually what's really hiding in there is either a **service object** (a distinct operation, like "authenticate a login") or a **value object** (a distinct concept). A concern earns its place when it's genuinely shared across unrelated models with a small public interface.

**Example**

```ruby
# SMELL — concern soup: three concerns, all still tightly coupled to User's
# internals, none of them reusable elsewhere, callbacks scattered across files
module UserAuthentication
  extend ActiveSupport::Concern
  included { before_save :hash_password }
  def hash_password; self.password_digest = BCrypt::Password.create(password); end
end

module UserNotifications
  extend ActiveSupport::Concern
  included { after_create :send_welcome_email }
  def send_welcome_email; UserMailer.welcome(self).deliver_later; end
end

class User < ApplicationRecord
  include UserAuthentication
  include UserNotifications
  # "what happens when a User is created?" now requires reading 3 files
end

# BETTER — a genuinely reusable concern (used by multiple unrelated models)...
module Sluggable
  extend ActiveSupport::Concern
  included do
    before_validation :generate_slug
    validates :slug, uniqueness: true
  end
  def generate_slug
    self.slug ||= title.to_s.parameterize
  end
end
class Post < ApplicationRecord
  include Sluggable
end
class Category < ApplicationRecord
  include Sluggable
end

# ...and the User-specific "operations" extracted as actual service objects instead:
class UserRegistration
  def self.call(attrs)
    user = User.new(attrs)
    user.password_digest = BCrypt::Password.create(attrs[:password])
    if user.save
      UserMailer.welcome(user).deliver_later
    end
    user
  end
end
```

### 71. What are service objects and form objects, and why use them over fat models? Give a real example of each.

**Short Answer**

- **Service object** — one plain Ruby class for one business operation, usually spanning several models. Called through a single entry point like `.call`.
- **Form object** — represents the shape of a form, with its own validations. Useful when one form maps to several models, or to no model at all.

Both exist to keep models focused on data and controllers thin.

**Simple Explanation**

Fat models happen when every piece of business logic gets bolted onto a model because "it has to live somewhere".

A model should represent one thing and the rules about itself — a `User` knows how to validate its own email. It starts to smell when it also knows how to send a welcome email, calculate a cart total across other models, and format itself for three different APIs.

A **service object** is a plain class named after an *action*, not a thing: `RegisterUser.call(params)`, `ChargeSubscription.call(user)`. No special Rails base class needed.

A **form object** solves a different problem. Not every form maps to one model. A signup might create both a `User` and an `Account`. A form object presents one flat set of attributes and validations to the view and handles the underlying models on submit, so the controller stays two lines.

**Example**

```ruby
# Service object — an operation spanning multiple models
class RegisterUser
  Result = Struct.new(:success?, :user, :errors)

  def self.call(params)
    new(params).call
  end

  def initialize(params)
    @params = params
  end

  def call
    user = User.new(@params.except(:plan))
    ActiveRecord::Base.transaction do
      user.save!
      Subscription.create!(user: user, plan: @params[:plan])
      user.update!(activated_at: Time.current)
    end
    UserMailer.welcome(user).deliver_later
    Result.new(true, user, nil)
  rescue ActiveRecord::RecordInvalid => e
    Result.new(false, nil, e.record.errors)
  end
end

# controller stays thin
class RegistrationsController < ApplicationController
  def create
    result = RegisterUser.call(registration_params)
    if result.success?
      redirect_to dashboard_path
    else
      @errors = result.errors
      render :new, status: :unprocessable_entity
    end
  end
end

# Form object — one form, multiple underlying models, its own validations
class SignupForm
  include ActiveModel::Model

  attr_accessor :name, :email, :company_name

  validates :name, :email, :company_name, presence: true
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }

  def save
    return false unless valid?

    ActiveRecord::Base.transaction do
      account = Account.create!(name: company_name)
      User.create!(name: name, email: email, account: account)
    end
    true
  end
end
```

### 72. When should you actually reach for a service object, and when should you NOT — how do you avoid over-engineering a simple CRUD feature?

**Short Answer**

Use a service object when an operation touches multiple models, calls an external service, needs its own transaction, has real branching logic, or is triggered from more than one place.

Skip it for plain single-model CRUD. Wrapping `Post.create(params)` in a `CreatePost` service that only calls `.save` adds a file and an indirection for zero benefit.

**Simple Explanation**

The over-engineering trap is real. Some codebases have a service object for every controller action, including ones that are genuinely just `Model.create(permitted_params)`. Now every trivial change means jumping through an extra layer for nothing.

The test I use: if the controller action were left inline, would it actually be hard to read or test? If it's two lines and one model, no.

Signs you **do** need one:

- It touches more than one model and needs a transaction around them.
- It calls an external service (payment gateway, third-party API).
- It has real conditional branching beyond validation.
- The same operation is triggered from a controller **and** a job **and** a rake task.

If none of those are true, a two-line controller action isn't fat — it's already thin.

**Example**

```ruby
# OVER-ENGINEERED — a service object with zero actual logic to encapsulate
class CreateComment
  def self.call(post:, params)
    post.comments.create(params)
  end
end
class CommentsController < ApplicationController
  def create
    comment = CreateComment.call(post: @post, params: comment_params)
    # this indirection buys nothing over the line below
  end
end

# RIGHT-SIZED — just do it inline, this is already thin
class CommentsController < ApplicationController
  def create
    @comment = @post.comments.create(comment_params)
    redirect_to @post
  end
end

# JUSTIFIED — multiple models, external call, transaction, reused from 2+ callers
class CancelSubscription
  def self.call(subscription)
    ActiveRecord::Base.transaction do
      subscription.update!(status: "canceled", canceled_at: Time.current)
      subscription.user.update!(plan: "free")
    end
    PaymentGateway.cancel!(subscription.external_id) # external side effect
    SubscriptionMailer.canceled(subscription).deliver_later
  end
end
# called from a controller action, a Sidekiq job (auto-cancel on payment failure),
# and an admin rake task — extracting it once avoided writing this logic 3 times
```

### 73. What's the Facade pattern in a Rails context, and when would you use one?

**Short Answer**

A facade is one class that gives callers a single simple method while it coordinates several other services behind the scenes. It doesn't hold business logic itself — it just sequences the work and hides the complexity.

**Simple Explanation**

The difference from a plain service object is mostly scope. A facade exists specifically to unify *several existing* services behind one entry point.

So a controller doesn't need to know that "checking out" really means "charge the payment, decrement inventory, create the order, send a confirmation, log an analytics event". It just calls `Checkout::Facade.new(cart).call`.

This earns its keep once you already have several focused services and you notice yourself calling all of them together, in the same order, from more than one place. The facade collapses that repeated sequence — and its error handling — into one spot.

**Example**

```ruby
module Checkout
  class Facade
    def initialize(cart, payment_token:)
      @cart = cart
      @payment_token = payment_token
    end

    def call
      ActiveRecord::Base.transaction do
        payment = PaymentService.charge(cart: @cart, token: @payment_token)
        InventoryService.decrement!(cart: @cart)
        order = OrderCreationService.create!(cart: @cart, payment: payment)
        NotificationService.order_confirmed(order)
        order
      end
    rescue PaymentService::DeclinedError => e
      InventoryService.release_hold!(cart: @cart)
      raise CheckoutFailed, e.message
    end
  end
end

class CheckoutsController < ApplicationController
  def create
    order = Checkout::Facade.new(current_cart, payment_token: params[:token]).call
    redirect_to order
  rescue Checkout::Facade::CheckoutFailed => e
    redirect_to cart_path, alert: e.message
  end
end
```

### 74. What are query objects, and why extract a complex query out of a scope or controller?

**Short Answer**

A query object wraps one complex, reused Active Record query in its own small class with a clear interface, like `OverdueInvoicesQuery.new(account).call`.

It keeps that logic out of an ever-growing pile of model scopes and out of controllers.

**Simple Explanation**

Simple scopes like `scope :active, -> { where(active: true) }` belong on the model. No problem there.

The trouble starts when a query grows real complexity — several joins, conditional filters, date-range logic, subqueries. A model file is a poor place for that to pile up, and a scope's rigid signature makes it hard to reuse.

A query object gives that logic its own named, testable class with clear inputs and an output you can keep chaining, since it usually just returns a Relation.

It's also the lightweight Rails version of the Repository pattern: instead of abstracting Active Record away entirely, you extract just the one gnarly query.

**Example**

```ruby
class OverdueInvoicesQuery
  def initialize(account, as_of: Time.current)
    @account = account
    @as_of = as_of
  end

  def call
    @account.invoices
            .where(status: "pending")
            .where("due_date < ?", @as_of)
            .where.not(id: @account.invoices.where(status: "disputed").select(:id))
            .order(due_date: :asc)
  end
end

# usage — reused from a controller, a mailer job, and an admin report
class InvoicesController < ApplicationController
  def overdue
    @invoices = OverdueInvoicesQuery.new(current_account).call
  end
end

class OverdueReminderJob < ApplicationJob
  def perform(account_id)
    account = Account.find(account_id)
    OverdueInvoicesQuery.new(account).call.find_each do |invoice|
      InvoiceMailer.overdue_reminder(invoice).deliver_later
    end
  end
end
```

### 75. Show the Factory pattern in a Rails context — choosing a payment gateway class from config without the caller knowing the concrete class.

**Short Answer**

A factory puts "which class do I build?" in one place, so callers ask for a capability (`PaymentGateway.for(account)`) instead of repeating `if provider == "stripe"` at every call site.

**Simple Explanation**

Without a factory, every place that charges a card ends up with its own copy of the `if/elsif` choosing Stripe vs Braintree vs a test double. Adding a third provider means hunting down all of them.

A factory owns that decision exactly once. Callers only depend on a shared interface — every gateway implements `#charge`, `#refund` — so adding a provider means one new class and one new line in the factory.

**Example**

```ruby
class PaymentGateway
  def self.for(account)
    case account.payment_provider
    when "stripe"    then StripeGateway.new(account)
    when "braintree" then BraintreeGateway.new(account)
    when "sandbox"   then SandboxGateway.new(account)
    else raise ArgumentError, "unknown provider: #{account.payment_provider}"
    end
  end
end

class StripeGateway
  def initialize(account) = @account = account
  def charge(amount_cents) = Stripe::Charge.create(amount: amount_cents, customer: @account.stripe_id)
end

class BraintreeGateway
  def initialize(account) = @account = account
  def charge(amount_cents) = Braintree::Transaction.sale(amount: amount_cents / 100.0, customer_id: @account.braintree_id)
end

# caller never knows or cares which concrete class it got back
gateway = PaymentGateway.for(current_account)
gateway.charge(1999)
```

### 76. Show the Observer pattern in Rails using `ActiveSupport::Notifications` — instrumenting and subscribing to events.

**Short Answer**

`ActiveSupport::Notifications` is Rails' built-in publish/subscribe system. One part of the code calls `instrument("event.name")` to announce something happened, and any number of subscribers elsewhere can react — without the publisher knowing who's listening.

**Simple Explanation**

This is the Observer pattern built into Rails. It's literally what powers Rails' own SQL and view-render logging (`sql.active_record`, `render_template.action_view`).

The value over a direct method call is decoupling. The code that completes an order doesn't need to know that analytics, a Slack notification, and a cache warm should all follow. Each of those subscribes independently, can be added or removed without touching the publisher, and several subscribers can react to the same event for completely different reasons.

**Example**

```ruby
# Publisher — announces the event, doesn't know who's listening
class OrderCompletionService
  def call(order)
    order.update!(status: "completed")
    ActiveSupport::Notifications.instrument("order.completed", order_id: order.id, total: order.total_cents)
    order
  end
end

# Subscribers — independent, added anywhere at boot (e.g. an initializer)
ActiveSupport::Notifications.subscribe("order.completed") do |name, start, finish, id, payload|
  AnalyticsClient.track("Order Completed", order_id: payload[:order_id], revenue: payload[:total])
end

ActiveSupport::Notifications.subscribe("order.completed") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  SlackNotifier.post("#sales", "New order completed: ##{event.payload[:order_id]}")
end

# Rails itself uses this for SQL instrumentation — you can subscribe to it too:
ActiveSupport::Notifications.subscribe("sql.active_record") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  Rails.logger.debug("SQL (#{event.duration.round(1)}ms) #{event.payload[:sql]}") if event.duration > 100
end
```

### 77. Show the Strategy pattern in Rails — swappable pricing/shipping calculators behind a common interface.

**Short Answer**

Strategy means several interchangeable classes implement the same method (say `#calculate`), and the caller is handed whichever one applies at runtime. Adding a new rule doesn't change the code that uses it.

**Simple Explanation**

The difference from Factory is subtle but real. Factory is about *which object to build*. Strategy is about *which algorithm to run*, and it's usually injected rather than looked up internally.

A shopping cart shouldn't need an `if/elsif` checking "is this flat-rate, weight-based, or free shipping?" every time it needs a cost. Each rule becomes its own class with the same interface, and the cart just calls `.calculate` on whatever it was given.

That makes each rule independently testable, and adding "international shipping" later is purely additive — no existing class changes.

**Example**

```ruby
class FlatRateShipping
  def calculate(order) = 5.99
end

class WeightBasedShipping
  def calculate(order) = order.total_weight_kg * 1.5
end

class FreeShippingOverThreshold
  THRESHOLD_CENTS = 5000
  def calculate(order) = order.subtotal_cents >= THRESHOLD_CENTS ? 0 : 5.99
end

class Order < ApplicationRecord
  def shipping_cost(strategy: FreeShippingOverThreshold.new)
    strategy.calculate(self)
  end
end

order.shipping_cost # uses the default strategy
order.shipping_cost(strategy: WeightBasedShipping.new) # swapped at the call site
```

### 78. Show the Decorator pattern in Rails — wrapping an object to add behavior without modifying its class.

**Short Answer**

A decorator wraps an object, adds extra methods, and forwards everything else to the original. In Ruby you get this with `SimpleDelegator` or the `draper` gem.

It's how you keep view-only formatting out of your models.

**Simple Explanation**

Models shouldn't accumulate methods like `full_name_with_title` or `formatted_price`. That's presentation logic, and it's only relevant to one view.

A decorator wraps an instance and adds exactly those methods, passing everything else straight through. So from the view's point of view a decorated `Post` still acts like a `Post` — `decorated_post.title` works normally — but it also gains `formatted_published_date`.

`SimpleDelegator` from the standard library is the lightweight way. The `draper` gem formalises it with a base class and helpers for bigger apps.

**Example**

```ruby
require "delegate"

class PostDecorator < SimpleDelegator
  def formatted_published_date
    published_at ? published_at.strftime("%B %-d, %Y") : "Draft"
  end

  def excerpt(length = 140)
    ActionView::Base.full_sanitizer.sanitize(body).truncate(length)
  end
end

# in the controller
@post = PostDecorator.new(Post.find(params[:id]))

# in the view — normal Post methods still work via delegation, plus the new ones
<h1><%= @post.title %></h1>
<p><%= @post.formatted_published_date %></p>
<p><%= @post.excerpt %></p>

# draper-gem equivalent, for larger apps with many decorated models
class PostDecorator < Draper::Decorator
  delegate_all

  def formatted_published_date
    object.published_at&.strftime("%B %-d, %Y") || "Draft"
  end
end
# usage: @post = Post.find(params[:id]).decorate
```

### 79. What's the Repository pattern, and what's Rails' practical version of it?

**Short Answer**

Repository means hiding data access behind an interface so the rest of the app doesn't talk to the database directly.

In Rails, Active Record models already do most of that job. The practical, incremental version for complex reads is the query object pattern.

**Simple Explanation**

In frameworks without a built-in ORM, Repository matters a lot: you'd hand-write a `UserRepository` with `#find`, `#save`, `#all` wrapping raw SQL, so the domain layer never touches the database.

Active Record already gives you most of that value. `User.find`, `User.where`, and `user.save` are already a clean interface over storage. Adding a *second* full repository layer on top is usually redundant, because you're realistically never going to swap Active Record out.

Where the idea still earns its keep is the query object: pull one complex read behind a small dedicated class instead of duplicating it or building a whole parallel layer.

A good senior answer is often "Active Record already is our repository layer — we only add an abstraction when the data access is genuinely complex or swappable."

**Example**

```ruby
# Active Record itself already functions as a repository for typical CRUD:
User.find(1)
User.where(active: true)
user.save

# Where a real repository-style abstraction earns its place: swappable backends,
# e.g. reading "recent activity" from Postgres today, Elasticsearch tomorrow —
# callers depend only on this interface, not on which store answers it.
class ActivityRepository
  def initialize(backend: ActiveRecordActivityBackend.new)
    @backend = backend
  end

  def recent_for(user, limit: 20)
    @backend.recent_for(user, limit: limit)
  end
end

class ActiveRecordActivityBackend
  def recent_for(user, limit:)
    user.activities.order(created_at: :desc).limit(limit)
  end
end
# swapping to Elasticsearch later means writing one new backend class —
# every caller of ActivityRepository#recent_for is unaffected.
```

### 80. How does Rails 7+ serve JavaScript — Importmap, esbuild, and the asset pipeline?

**Short Answer**

- **Importmap** (Rails 7 default) — no build step. The browser loads ES modules directly, mapped by an import map Rails generates.
- **jsbundling-rails + esbuild** (or Webpack/Rollup) — a real Node build step, for apps that need npm packages, JSX, or TypeScript.
- **Sprockets / Propshaft** — still handles fingerprinting and serving assets in either setup.

**Simple Explanation**

Before Rails 7, the default was Webpacker — a full Node pipeline for all JavaScript, which added a lot of complexity even for apps that only needed a sprinkle of JS.

Importmap skips all of that. Modern browsers can load ES modules natively, and an import map just tells the browser "when code says `import 'sortablejs'`, fetch it from this URL". No bundling, no `node_modules`, no build artifact to go stale. It's a great fit for Hotwire/Turbo/Stimulus apps.

The trade-off: no JSX, no TypeScript compilation, and some npm packages that assume a bundler don't work as plain ES modules.

For a JS-heavy frontend, `jsbundling-rails` wires up esbuild (fast, simple config) or Webpack as a real build step, still integrated with the asset pipeline for fingerprinting.

Either way, Sprockets (or Propshaft in newer Rails) is the layer that fingerprints and serves the compiled output.

**Example**

```ruby
# Gemfile — Importmap default (Rails 7 default for new apps)
gem "importmap-rails"

# config/importmap.rb
pin "application"
pin "sortablejs", to: "https://ga.jspm.io/npm:sortablejs@1.15.0/modular/sortable.esm.js"

# app/javascript/application.js
import "sortablejs"
import { Application } from "@hotwired/stimulus"

# app/views/layouts/application.html.erb
<%= javascript_importmap_tags %>

# --- OR, for apps that need a real bundler (React/TypeScript/npm-heavy) ---
# Gemfile
gem "jsbundling-rails"
# package.json
# "scripts": { "build": "esbuild app/javascript/*.* --bundle --outdir=app/assets/builds" }

# config/initializers or Procfile.dev often run:
# esbuild --watch alongside bin/rails server during development
```

**— Authentication & Authorization —**

### 81. Devise vs `has_secure_password` — when would you actually roll your own?

**Short Answer**

- **Devise** — a full auth system: registration, sessions, password reset, email confirmation, account lockout, OAuth via OmniAuth. Fast to add, but a lot of built-in behavior to learn.
- **`has_secure_password`** — built into Rails. Gives you bcrypt password hashing and an `authenticate` method. Everything else (sessions, reset flows) you write yourself.

**Simple Explanation**

Devise gets you a production-ready login system in an afternoon, plus a mature ecosystem. The cost is real: generated code you didn't write, a lot of implicit behavior that's tricky to debug, and far more features than many apps need.

`has_secure_password` is the minimal building block. Add a `password_digest` column, call `has_secure_password` in the model, and you get password confirmation validation plus `user.authenticate(password)`, backed by bcrypt. Sessions, "remember me", reset tokens, and login rate limiting are yours to build.

Roll your own when the auth model is unusual — multiple credential types, a custom SSO flow, auth tied to a non-standard user table — where fighting Devise's conventions costs more than writing the flows.

Use Devise when you want standard email/password plus social login shipped quickly and don't mind the dependency.

**Example**

```ruby
# has_secure_password — minimal, you build the rest
# Gemfile: gem "bcrypt"
# migration: add_column :users, :password_digest, :string

class User < ApplicationRecord
  has_secure_password
  validates :email, presence: true, uniqueness: true
end

class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email])
    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to root_path
    else
      flash.now[:alert] = "Invalid email or password"
      render :new, status: :unprocessable_entity
    end
  end
end

# Devise — full-featured, configured via modules
# Gemfile: gem "devise"
# $ bin/rails generate devise:install
# $ bin/rails generate devise User

class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :lockable, :trackable
end

# config/routes.rb
devise_for :users
# gives you /users/sign_in, /users/sign_up, /users/password/new, etc. for free
```

### 82. CanCanCan vs Pundit — side by side, and what is IDOR?

**Short Answer**

- **CanCanCan** — all rules in one `Ability` class, checked with `authorize!` / `can?`. Simple at first, but the file gets large.
- **Pundit** — one small policy class per model (`PostPolicy#update?`), checked with `authorize @post`. Scales better for per-record rules.

**IDOR** (Insecure Direct Object Reference) is the bug both exist to prevent: checking that someone is *logged in* but never checking they're allowed to touch *that specific record*.

**Simple Explanation**

Both answer the same question — "is this user allowed to do this to this record?" — with different organisation.

CanCanCan puts every rule for every model in one `Ability` class, usually keyed off `user.role`. Easy to scan in a small app; sprawling once you have a dozen models with different per-record rules.

Pundit gives each model its own small policy file with `update?`, `destroy?`, `index?` methods and a `Scope` class for filtering lists. That scales better and naturally handles ownership checks like "this user can edit this post because they wrote it".

**IDOR in practice:** `PostsController#update` checks `authenticate_user!` (are you logged in?) but forgets `authorize @post` (are you allowed to edit *this* post?). Any logged-in user can then `PATCH /posts/999` and edit someone else's post just by changing the ID.

Authentication answers "who are you". Authorization answers "are you allowed to do this to this thing". IDOR is what happens when you nail the first and skip the second.

**Example**

```ruby
# --- CanCanCan ---
# app/models/ability.rb
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new # guest
    if user.admin?
      can :manage, :all
    else
      can :read, Post, published: true
      can [:update, :destroy], Post, user_id: user.id # ownership check
    end
  end
end

class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    authorize! :update, @post # raises CanCan::AccessDenied if not allowed
    @post.update!(post_params)
    redirect_to @post
  end
end

# --- Pundit ---
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def update?
    user.admin? || record.user_id == user.id
  end

  class Scope < Scope
    def resolve
      user.admin? ? scope.all : scope.where(published: true)
    end
  end
end

class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    authorize @post # raises Pundit::NotAuthorizedError if update? returns false
    @post.update!(post_params)
    redirect_to @post
  end
end

# --- IDOR: the bug both gems exist to prevent ---
class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    # BUG: only checked that *someone* is logged in, never that THIS user
    # owns THIS post — any authenticated user can edit any post by ID.
    @post.update!(post_params) # <- missing authorize!/authorize call
    redirect_to @post
  end
end
```

### 83. How does Rails prevent SQL injection, and what does a vulnerable query actually look like?

**Short Answer**

Active Record sends your values to the database **separately** from the SQL text, so user input can never be treated as SQL. That's called parameterizing.

The danger comes back the moment you build a SQL string yourself with Ruby interpolation (`#{}`).

**Simple Explanation**

`User.where("email = ?", params[:email])` sends the string `"email = ?"` as the query and the value as a separate parameter. Even if someone submits `' OR '1'='1` as their email, the database treats it purely as data — it just matches nothing.

The vulnerability returns as soon as you write `"email = '#{params[:email]}'"`. Now that input becomes part of the actual SQL, and a crafted value can change what the query means — bypassing a `WHERE` clause entirely, or pulling out other tables with a `UNION`.

The same risk applies to `order`, `group`, `find_by_sql`, and any raw SQL fragment with interpolated input. For dynamic column names (like sorting), use an allow-list of permitted columns instead.

**Example**

```ruby
# VULNERABLE — raw string interpolation, user input becomes part of the SQL
def show
  # attacker submits email = "' OR '1'='1"
  # resulting SQL: SELECT * FROM users WHERE email = '' OR '1'='1'
  # matches EVERY row — total bypass of the intended filter
  @user = User.where("email = '#{params[:email]}'").first
end

# SAFE — parameterized, Active Record binds the value separately from the SQL
def show
  @user = User.where("email = ?", params[:email]).first
  # or, idiomatically:
  @user = User.find_by(email: params[:email])
end

# Also unsafe if interpolated — ORDER BY / raw SQL fragments are just as exposed
Post.order(params[:sort]) # DANGEROUS if params[:sort] isn't allow-listed —
                           # Rails will raise on some inputs but a crafted
                           # string can still inject SQL into an ORDER BY clause

# Safe pattern for dynamic sorting — allow-list the actual column
ALLOWED_SORTS = %w[created_at title].freeze
sort_column = ALLOWED_SORTS.include?(params[:sort]) ? params[:sort] : "created_at"
Post.order(sort_column)
```

**— Background Jobs & Async —**

### 84. When and why do you move work off the request cycle into a background job?

**Short Answer**

Move work to a background job when it's slow, unreliable (an external API), or simply not needed to answer the current request — emails, reports, webhooks, image processing.

That way the request returns fast instead of holding a web thread (and its database connection) open.

**Simple Explanation**

Every second a request spends waiting is a second a Puma thread is unavailable to anyone else. Under load, a few slow synchronous actions can starve the whole app even though the CPU looks idle.

Rule of thumb: if the user doesn't need the result *right now* to see their response, do it asynchronously. Enqueue a job and return immediately.

ActiveJob is Rails' adapter-agnostic job framework. Sidekiq is the most common production backend — Redis-based, multi-threaded, with mature retry and monitoring tools.

The trade-off you accept: the user doesn't get immediate confirmation that the work succeeded. So jobs should be safe to retry, and you need a way to see failures (the Sidekiq UI, alerts on exhausted retries) rather than the user just seeing an error page.

**Example**

```ruby
# Without a background job — the user waits on a slow email send/API call
class OrdersController < ApplicationController
  def create
    @order = Order.create!(order_params)
    OrderMailer.confirmation(@order).deliver_now # blocks the request, maybe seconds
    ThirdPartyCrm.sync_order(@order)              # blocks even more, could time out
    redirect_to @order
  end
end

# With a background job — request returns immediately
class OrdersController < ApplicationController
  def create
    @order = Order.create!(order_params)
    OrderMailer.confirmation(@order).deliver_later # enqueued, returns instantly
    CrmSyncJob.perform_later(@order.id)
    redirect_to @order
  end
end

class CrmSyncJob < ApplicationJob
  queue_as :default
  retry_on ThirdPartyCrm::TimeoutError, wait: :polynomially_longer, attempts: 5

  def perform(order_id)
    order = Order.find(order_id)
    ThirdPartyCrm.sync_order(order)
  end
end
```

### 85. Explain Sidekiq's concurrency model — processes, threads, queues, and retries.

**Short Answer**

Sidekiq runs one process with a pool of threads (often 10–25) pulling jobs off Redis-backed queues. Queues can be weighted by priority. Failed jobs retry automatically with growing delays, then land in a dead set.

Because jobs run later, in a different process, **always pass small values like an ID — never a full Active Record object**.

**Simple Explanation**

Each Sidekiq process boots your app once and starts N worker threads, each pulling the next job off Redis. With concurrency 20, twenty jobs run at once. I/O-bound work benefits a lot; CPU-bound work doesn't parallelize as well because of Ruby's GVL, so heavy CPU jobs often want more processes rather than more threads.

Queues let you prioritise — `critical`, `default`, `low` with weights — though under sustained load a low-priority queue can still starve if the higher ones stay busy.

Retries: a job that raises is re-enqueued with an increasing delay, up to a max (Sidekiq's default is 25). After that it goes to the Dead set for manual inspection instead of retrying forever.

**Why arguments must be small:** job arguments are serialized to JSON and stored in Redis until a worker picks them up. You can't meaningfully serialize an Active Record object, and even if you could you'd get a stale snapshot. Passing `order.id` and calling `Order.find(order_id)` inside `perform` guarantees fresh data — and forces you to handle the case where the record was deleted before the job ran.

**Example**

```ruby
# config/sidekiq.yml
:concurrency: 20
:queues:
  - [critical, 4]
  - [default, 2]
  - [low, 1]

# BAD — passing a full object; won't serialize meaningfully, and even if it
# did, the job would operate on stale data by the time it actually runs
class BadJob < ApplicationJob
  def perform(order) # don't do this
    order.charge!
  end
end

# GOOD — pass the ID, re-fetch fresh inside perform
class ChargeOrderJob < ApplicationJob
  queue_as :critical
  retry_on PaymentGateway::TimeoutError, wait: :polynomially_longer, attempts: 5

  def perform(order_id)
    order = Order.find(order_id) # fresh read — handles the record having changed
    order.charge!
  rescue ActiveRecord::RecordNotFound
    Rails.logger.warn("ChargeOrderJob: order #{order_id} no longer exists, skipping")
  end
end
```

### 86. What's the deploy-safety problem for background jobs, and how do you deploy a job class change safely?

**Short Answer**

Jobs already sitting in Redis were serialized with the **old** class name and argument shape. If you rename the class or change `perform`'s arguments and deploy before those drain, Sidekiq raises `NameError` or calls `perform` with arguments it no longer expects.

Three safe options: drain the queue first, leave a backward-compatible shim, or version the job.

**Simple Explanation**

A job enqueued at 2:00pm is stored in Redis as JSON like `{"class": "SyncOrderJob", "args": [123]}`. If you deploy at 2:05pm with that class renamed, the queued job still says `SyncOrderJob` — and that constant no longer exists.

The three approaches:

1. **Drain the queue** — stop enqueuing, let Sidekiq finish everything queued, confirm it's empty, then deploy the rename.
2. **Leave a shim** — keep the old class name as a thin subclass that delegates to the new one, for at least one deploy cycle.
3. **Version the job** — add `OrderSyncJobV2` alongside the old one, point new enqueues at V2, and only delete the old class once the queue and retry set are confirmed clear.

The general principle is the same as backward-compatible migrations: "deployed" never means "every in-flight thing is already running the new code".

**Example**

```ruby
# Renaming SyncOrderJob -> OrderSyncJob safely with a shim, so any job already
# enqueued as "SyncOrderJob" before the deploy still resolves and runs correctly.

class OrderSyncJob < ApplicationJob
  def perform(order_id)
    ThirdPartyCrm.sync_order(Order.find(order_id))
  end
end

# Temporary shim — keep for at least one full deploy cycle / until the
# old queue and retry set are confirmed empty in the Sidekiq UI
class SyncOrderJob < OrderSyncJob
end

# Changing perform's signature safely — accept the old shape too, for now
class OrderSyncJob < ApplicationJob
  def perform(order_id, options = {}) # options defaults, so old 1-arg jobs still work
    ThirdPartyCrm.sync_order(Order.find(order_id), force: options[:force])
  end
end
```

### 87. What's ActionCable for, and what's a real use case?

**Short Answer**

ActionCable is Rails' built-in WebSockets layer. It lets the server **push** data to connected browsers in real time instead of clients polling for changes. Live notifications and chat are the classic uses.

**Simple Explanation**

A normal request is always started by the client — the server can't send anything unless asked. ActionCable keeps a persistent WebSocket connection open so the server can push at any moment: a new chat message appears instantly, a dashboard count updates live.

Clients subscribe to named "channels", and the server broadcasts to a channel (often from a model callback or a background job). Everyone subscribed gets the message over their open socket.

In production, ActionCable needs a pub/sub backend so broadcasts reach clients connected to *other* app server processes. Redis is the traditional choice; Rails 8's Solid Cable is a database-backed alternative.

**Example**

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat_room_#{params[:room_id]}"
  end
end

# app/models/message.rb — broadcast whenever a new message is saved
class Message < ApplicationRecord
  belongs_to :room
  after_create_commit :broadcast_message

  private

  def broadcast_message
    ActionCable.server.broadcast(
      "chat_room_#{room_id}",
      { user: user.name, body: body, created_at: created_at }
    )
  end
end

# app/javascript/channels/chat_channel.js (client side)
import consumer from "channels/consumer"
consumer.subscriptions.create({ channel: "ChatChannel", room_id: 42 }, {
  received(data) {
    document.querySelector("#messages").insertAdjacentHTML("beforeend", `<p>${data.user}: ${data.body}</p>")
  }
})
```

### 88. Rails 8's Solid Queue, Solid Cache, Solid Cable, and Kamal — what problem do they solve?

**Short Answer**

- **Solid Queue / Solid Cache / Solid Cable** — database-backed replacements for Sidekiq's Redis queue, a Redis cache store, and Redis-backed ActionCable. They let a small or medium app run without a separate Redis.
- **Kamal** — a deploy tool that pushes Docker containers to your own servers over SSH, without Heroku or Kubernetes.

**Simple Explanation**

Running Redis means one more thing to provision, monitor, back up, and pay for. Necessary at real scale, but genuine overhead for a small app.

The Solid trio uses ordinary database tables instead. Solid Queue implements ActiveJob's backend with database polling. Solid Cache stores cached values in a table (designed to hold much more than you'd keep in Redis, at slightly higher read latency). Solid Cable backs ActionCable's pub/sub with the database. All three are Rails 8 defaults for new apps.

You'd still prefer Sidekiq + Redis when you need its mature ecosystem (retry UI, cron, per-queue metrics) or when your job volume is high enough that queue polling would add real load to your primary database.

Kamal solves a different problem — it's essentially "Capistrano for Docker". It builds your image, pushes it to a registry, SSHes into your servers, and does rolling restarts with health checks and Let's Encrypt support. Heroku-style ergonomics without Heroku's price or a Kubernetes cluster.

**Example**

```ruby
# config/application.rb — Rails 8 defaults, no Redis required
config.active_job.queue_adapter = :solid_queue
config.cache_store = :solid_cache_store
config.action_cable.adapter = :solid_cable

# Gemfile
gem "solid_queue"
gem "solid_cache"
gem "solid_cable"

# bin/rails db:prepare sets up the solid_queue/solid_cache tables via their engines

# config/deploy.yml — Kamal deploy config, deploys straight to your own servers
service: myapp
image: myregistry/myapp
servers:
  web:
    - 192.168.1.10
    - 192.168.1.11
registry:
  server: registry.example.com
  username: deploy
  password:
    - KAMAL_REGISTRY_PASSWORD
env:
  secret:
    - RAILS_MASTER_KEY

# $ kamal deploy   -> builds image, pushes, SSHes in, rolling-restarts containers
```

**— Caching —**

### 89. Explain fragment caching, Russian doll caching, and low-level caching in Rails.

**Short Answer**

- **Fragment caching** — cache a rendered chunk of a view, keyed by the object.
- **Russian doll caching** — nested fragment caches, so changing one child only busts the relevant keys.
- **Low-level caching** — `Rails.cache.fetch` for any expensive value, not just views.

**Simple Explanation**

Fragment caching wraps part of a template in `cache`, with a key built from the object's cache key — something like `"posts/1-20240101120000"`, which includes the model name, id, and `updated_at`. When the post changes, `updated_at` changes, so the key changes, and the old cached fragment is simply never looked up again.

Russian doll caching nests these: a post's fragment contains each comment's own fragment. If one comment changes, its key changes — but the outer post fragment's key would still look fresh on its own. That's exactly why `touch: true` on the comment's `belongs_to :post` matters: it bumps the post's `updated_at` too, so the outer fragment is rebuilt as well, while unrelated posts stay cached.

Low-level caching is the general tool underneath all of it: `Rails.cache.fetch(key, expires_in: ...) { expensive_work }` for anything costly that isn't a view — an API response, a computed total, a slow lookup.

**Example**

```erb
<%# Russian doll caching — app/views/posts/show.html.erb %>
<% cache @post do %>
  <h1><%= @post.title %></h1>
  <div id="comments">
    <% @post.comments.each do |comment| %>
      <% cache comment do %>
        <p><%= comment.author_name %>: <%= comment.body %></p>
      <% end %>
    <% end %>
  </div>
<% end %>
```

```ruby
# touch: true propagates a child change up to bust the parent's cache key too
class Comment < ApplicationRecord
  belongs_to :post, touch: true
end

# Low-level caching — arbitrary computed value, not view output
class DashboardStats
  def self.monthly_revenue
    Rails.cache.fetch("dashboard/monthly_revenue/#{Date.current.strftime('%Y-%m')}", expires_in: 1.hour) do
      Order.where(created_at: Date.current.beginning_of_month..).sum(:total_cents)
    end
  end
end
```

### 90. Why does Redis fit well as a Rails cache store, and how do you configure it?

**Short Answer**

Redis is an in-memory store with built-in expiry (TTL) that every app process can reach over the network. That's exactly what a cache needs: fast reads, automatic expiration, and one **shared** cache visible to all your processes.

**Simple Explanation**

The default `:memory_store` keeps cached values inside a single Ruby process. Fine in development, useless in production where you run multiple Puma workers across possibly several servers — a value cached by one process wouldn't be visible to another.

Redis solves this by being a separate shared service. Cache once, and every process benefits immediately.

Its built-in `EXPIRE` support means Rails' `expires_in:` maps onto a real Redis feature instead of being faked, and it's fast enough (sub-millisecond) that cache reads never become the bottleneck.

Configuring it is one line pointing `cache_store` at a Redis URL. Rails' built-in `redis_cache_store` handles serialization, namespacing, and connection pooling.

**Example**

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV.fetch("REDIS_URL"),
  namespace: "myapp_cache",
  expires_in: 1.day,        # default TTL if not specified per-call
  pool_size: 5,
  pool_timeout: 5,
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.error("Redis cache error in #{method}: #{exception.message}")
  }
}

Rails.cache.write("recent_signups_count", 42, expires_in: 10.minutes)
Rails.cache.read("recent_signups_count") #=> 42
Rails.cache.fetch("recent_signups_count", expires_in: 10.minutes) { User.recent.count }
```

### 91. Explain HTTP-level conditional caching in a Rails controller — `ETag`, `Last-Modified`, `fresh_when`/`stale?`, and `expires_in`.

**Short Answer**

- `fresh_when` / `stale?` — conditional GET. The browser still makes a request, but if nothing changed the server replies `304 Not Modified` with no body, skipping the render.
- `expires_in` / `Cache-Control` — tells the browser or CDN not to ask again at all for a set time.

**Simple Explanation**

`fresh_when(@post)` sets `ETag` and `Last-Modified` headers from the record and compares them with the `If-None-Match` / `If-Modified-Since` headers the browser sends. If they match, Rails short-circuits with a `304` and an empty body — saving bandwidth and the view render, though the request still reaches your server.

`stale?` is the same mechanism used as a guard: `if stale?(@post)` only does the expensive work when the client's copy is actually out of date.

`expires_in` is stronger — it tells the browser and any CDN not to send a request at all for N seconds. Much cheaper, but only safe for genuinely public content, since caches are shared and don't know who's asking.

So: conditional GET for anything per-user (cheap, still dynamic), `expires_in` for public content you want served without touching your app.

**Example**

```ruby
class PostsController < ApplicationController
  # Conditional GET — still hits the server, but skips the render if unchanged
  def show
    @post = Post.find(params[:id])
    fresh_when(@post) # sets ETag/Last-Modified, auto-304s if the client's copy matches
  end

  # stale? as an explicit guard around expensive work
  def show_expensive
    @post = Post.find(params[:id])
    if stale?(@post)
      @related = ExpensiveRelatedPostsQuery.new(@post).call
      render :show_expensive
    end
    # if not stale, Rails already sent a 304 and this render is skipped
  end

  # expires_in — skips the round trip entirely for public, cacheable content
  def public_stats
    expires_in 5.minutes, public: true
    render json: DashboardStats.public_summary
  end
end
```

**— File Uploads & Mailers —**

### 92. What is Active Storage, and how do you attach files to a model?

**Short Answer**

Active Storage is Rails' built-in file upload system. It attaches files to models (`has_one_attached`, `has_many_attached`), supports uploading straight from the browser to S3, and generates resized image variants on demand.

**Simple Explanation**

`has_one_attached :avatar` gives you `user.avatar.attach(...)`, `user.avatar.attached?`, and a URL helper without writing any upload code. Rails manages two tables behind the scenes (`active_storage_attachments` and `active_storage_blobs`) and stores the actual bytes wherever you configure — local disk in development, S3 or GCS in production.

**Direct uploads** (`direct_upload: true` on the file field) let the browser upload straight to S3 using a pre-signed URL your server generates. The file never passes through your Rails process — only the metadata does. That matters a lot for large files, because otherwise a slow upload ties up a web worker for its whole duration.

**Variants** (`user.avatar.variant(resize_to_limit: [200, 200])`) generate resized versions on demand using libvips or ImageMagick, and are stored after the first request so you don't reprocess on every page load.

**Example**

```ruby
# Gemfile — image_processing is required for variants
gem "image_processing", "~> 1.2"

class User < ApplicationRecord
  has_one_attached :avatar
  has_many_attached :documents
end

# controller
class UsersController < ApplicationController
  def update
    @user = current_user
    @user.avatar.attach(params[:user][:avatar]) if params[:user][:avatar]
    redirect_to @user
  end
end

# view — direct upload straight to S3, bypassing the app server for the bytes
<%= form_with model: @user do |f| %>
  <%= f.file_field :avatar, direct_upload: true %>
<% end %>

# variant — generated (and cached) on first request
<%= image_tag @user.avatar.variant(resize_to_limit: [200, 200]) %>

# config/storage.yml
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: us-east-1
  bucket: myapp-production
```

### 93. `deliver_now` vs `deliver_later` — why should it almost always be the latter?

**Short Answer**

- `deliver_now` — sends the email right there, inside the current request, blocking on the mail provider.
- `deliver_later` — queues a background job to send it, so the request returns immediately.

Use `deliver_later` for anything triggered from a controller.

**Simple Explanation**

Sending an email means a network call to an external service — anywhere from tens of milliseconds to several seconds, occasionally a timeout.

`deliver_now` makes the user wait for all of that. If the mail provider is slow or down, the request hangs or errors even though the thing the user actually cared about (placing an order) already worked.

`deliver_later` enqueues a small job with enough info to build and send the email later, and gets ActiveJob's retry behavior for free if the provider hiccups.

The rare case for `deliver_now`: you're already inside a background job or rake task, so blocking briefly doesn't affect any user-facing request.

**Example**

```ruby
class OrderMailer < ApplicationMailer
  def confirmation(order)
    @order = order
    mail(to: @order.customer_email, subject: "Order ##{@order.id} confirmed")
  end
end

class OrdersController < ApplicationController
  def create
    @order = Order.create!(order_params)
    # GOOD — enqueued, request returns immediately regardless of mail provider latency
    OrderMailer.confirmation(@order).deliver_later

    # AVOID in a request path — blocks on the mail provider's response time
    # OrderMailer.confirmation(@order).deliver_now

    redirect_to @order
  end
end

# Mailer previews for local dev — no real email sent, view in browser
# test/mailers/previews/order_mailer_preview.rb
class OrderMailerPreview < ActionMailer::Preview
  def confirmation
    OrderMailer.confirmation(Order.first)
  end
end
# visit http://localhost:3000/rails/mailers/order_mailer/confirmation
```

### 94. What is Action Text, and how does it relate to Active Storage?

**Short Answer**

Action Text gives you rich-text fields (`has_rich_text :body`) with a Trix editor in the browser. It stores the formatted HTML in its own table and routes any images you paste into the editor through Active Storage automatically.

**Simple Explanation**

Without it, "rich text with inline images" is real work: you'd need an editor, somewhere to store formatted HTML, and separate handling for images embedded in that content.

`has_rich_text :body` gives you a `body` attribute that behaves like a normal attribute (you can validate it, call `body.to_s`) but is actually stored in a separate polymorphic table. In forms, `f.rich_text_area :body` renders the Trix editor.

Any image dropped into the editor becomes a real Active Storage attachment, so it gets the same direct-upload and variant behavior as any other file — no extra code.

**Example**

```ruby
# Gemfile
gem "actiontext"
# $ bin/rails action_text:install (adds migration + Trix JS)

class Post < ApplicationRecord
  has_rich_text :body
end

# view — Trix-powered rich text editor, images dropped in are Active Storage-backed
<%= form_with model: @post do |f| %>
  <%= f.rich_text_area :body %>
<% end %>

# rendering
<%= @post.body %> # renders sanitized HTML, including any embedded <action-text-attachment> images

# in the model, body behaves like a normal attribute
@post.body.to_plain_text # strips HTML for e.g. a search index or preview text
```

**— Rails Internals & Upgrades —**

### 95. Rake vs Rack — what's the actual difference, and when does logic belong in a Rake task vs middleware?

**Short Answer**

They're unrelated things with similar-sounding names.

- **Rake** — a task runner (`rake db:migrate`). For one-off or scheduled work that isn't an HTTP request.
- **Rack** — the HTTP interface: `call(env)` returns `[status, headers, body]`.

**Simple Explanation**

Rake predates Rails and has nothing to do with HTTP. It's basically "make, in Ruby" — named tasks with dependencies. Rails uses it for things that run outside a request: migrations, seeds, nightly cleanup, one-off data fixes.

A custom rake task in `lib/tasks/` is the right home for anything you run manually or on a schedule, like purging expired sessions from cron.

Rack is specifically the contract for handling an HTTP request. Every request passes through a chain of Rack objects — middleware, then the router, then a controller.

Why maintenance logic doesn't belong in middleware: middleware runs on **every single request**, right in your latency path. Putting a cleanup task there means paying for it (or at least a check for it) forever. A rake task runs once, on your schedule, with zero impact on request latency.

**Example**

```ruby
# lib/tasks/cleanup.rake — Rake: a one-off/scheduled operational task
namespace :cleanup do
  desc "Purge sessions expired more than 30 days ago"
  task expired_sessions: :environment do
    count = Session.where("expires_at < ?", 30.days.ago).delete_all
    puts "Purged #{count} expired sessions"
  end
end
# $ bin/rails cleanup:expired_sessions
# run nightly via cron: 0 3 * * * cd /app && RAILS_ENV=production bin/rails cleanup:expired_sessions

# app/middleware/request_id_logger.rb — Rack: runs on every single HTTP request
class RequestIdLogger
  def initialize(app)
    @app = app
  end

  def call(env)
    status, headers, body = @app.call(env)
    Rails.logger.info("request_id=#{env['action_dispatch.request_id']} status=#{status}")
    [status, headers, body]
  end
end
```

### 96. Walk through a major Rails version upgrade (e.g. Rails 6 to 7) on a large production monolith.

**Short Answer**

Upgrade **one minor version at a time** (6.0 → 6.1 → 7.0), not straight to the target. Read the upgrade guide for each hop, run `bin/rails app:update`, and turn on the new framework defaults **one at a time** rather than all at once.

Your test suite — not manual QA — is the real safety net.

**Simple Explanation**

Jumping straight from 6.0 to 7.0 lands every breaking change on you at once, with no way to tell which change caused which failure. Going one minor version at a time keeps each hop small and attributable. Better still, 6.1 will already warn you about things that become hard errors in 7.0, giving you a preview of what to fix.

`bin/rails app:update` regenerates framework config files and drops a `new_framework_defaults_X_Y.rb` file with every new default **commented out**. You uncomment them one (or a few) at a time, run the suite, and commit. That turns "upgrade the framework" from one big cutover into a series of small, individually revertible changes.

Throughout, treat deprecation warnings as your map. `DEPRECATION WARNING: X will be removed in Rails 7.1` tells you exactly what to fix before the next hop breaks it.

Manual QA can't cover "did anything in this 200k-line app break". A good test suite can, repeatably. Teams with thin coverage usually find that's the real blocker, not the framework changes.

**Example**

```ruby
# Gemfile — one minor hop at a time
gem "rails", "~> 6.1.0" # first: land here, all specs green, no unresolved deprecation warnings
# then:
gem "rails", "~> 7.0.0" # only after 6.1 is fully clean

# $ bundle update rails
# $ bin/rails app:update
#   -> generates config/initializers/new_framework_defaults_7_0.rb with:
#
# Rails.application.config.action_controller.default_protect_from_forgery = true
#
# Uncomment ONE default (or a small related batch), then:
# $ bin/rails test && bin/rails test:system
# commit, move to the next default, repeat until the whole file is uncommented,
# then delete it (its settings become permanent in config/application.rb)

# Treat deprecation warnings as signal, not noise, e.g. in test output:
# DEPRECATION WARNING: Rendering views without explicit `formats` is deprecated
#   -> fix this now, before it becomes a hard error two versions from now
```

### 97. Explain Puma's process/thread architecture and why your DB connection pool has to be sized around it.

**Short Answer**

Puma runs several worker **processes**, each with a pool of **threads**. `preload_app!` loads the app once before forking so workers share memory (copy-on-write).

Each thread can hold its own database connection, so:

- `pool` size must be **at least** the thread count per worker
- total connections = `workers × pool`, and that must fit under the database's `max_connections`

**Simple Explanation**

With `WEB_CONCURRENCY=4` and `threads 5, 5`, you have 4 processes × 5 threads = up to 20 requests in flight on that machine.

Processes give real parallelism (each has its own GVL) at the cost of memory. `preload_app!` reduces that cost by loading the app before forking, so the OS can share unchanged memory pages.

Threads inside one worker share the GVL, but still help a lot for I/O — a thread waiting on a database query releases the GVL so another can run.

**The connection pool part is where people get burned.** The `pool:` setting in `database.yml` is per process, not global. If a worker has 5 threads and the pool is smaller, the 6th concurrent thread blocks waiting for a connection — an artificial bottleneck even though the database is fine.

Then, because each worker process has its own pool, your database needs to handle `workers × pool` connections, plus Sidekiq and anything else. Postgres defaults to 100. A common production incident is exactly this: too many workers times too generous a pool quietly exceeds `max_connections`, and everything starts erroring at once.

**Example**

```ruby
# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY", 4)          # process count
threads_count = ENV.fetch("RAILS_MAX_THREADS", 5)
threads threads_count, threads_count              # threads per process
preload_app!                                       # load once, fork after — COW memory savings

on_worker_boot do
  # each forked worker needs its OWN db connections re-established after fork
  ActiveRecord::Base.establish_connection
end

# config/database.yml — pool must be >= threads per worker
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>

# Total DB connections needed = WEB_CONCURRENCY x pool
# 4 workers x 5 pool = 20 connections from this app server alone —
# multiply across every app server + Sidekiq processes, and compare
# against your DB's max_connections (Postgres default: 100)
```

### 98. Explain Zeitwerk autoloading — `autoload_paths` vs `eager_load_paths`, naming convention, and why a `NameError` sometimes only shows up in production.

**Short Answer**

Zeitwerk maps file paths to constant names by convention: `app/models/user_profile.rb` must define `UserProfile`.

In **development** it loads files lazily, so a naming mistake goes unnoticed until something references that constant. In **production** `eager_load = true` loads everything at boot, so the same mistake crashes immediately at startup.

**Simple Explanation**

`autoload_paths` (mostly `app/*`) tells Zeitwerk which directories follow the naming convention. `eager_load_paths` is the subset that gets force-loaded at boot when eager loading is on — which it is in production by default, and off in development.

The naming rule is strict and mechanical: `app/models/admin/report.rb` must define `Admin::Report`.

If you typo the filename or the class name inside it, Zeitwerk can't resolve the constant. In development nothing loads that file until some code actually references the class — so if your local testing never hits that page, everything looks fine. In production, eager loading walks every file at boot and raises `NameError: expected file ... to define ...` right at deploy time, which is exactly the point: better a failed deploy than a live 500.

This is why running with eager loading on in CI is worth doing — it turns a surprise production crash into a caught build failure.

**Example**

```ruby
# config/application.rb
# config.autoload_paths += %W(#{config.root}/app/services) # add a custom autoload root
# config.eager_load_paths is a subset auto-populated from app/* by default

# app/models/user_profile.rb — MUST define UserProfile to match the file path
class UserProfile < ApplicationRecord
end

# BROKEN — file path says UserProfile, but the class defined doesn't match
# app/models/user_profile.rb
class UserProile < ApplicationRecord # typo — silently fine in dev until referenced
end
# development: no error until something calls UserProfile
# production (eager_load = true): boot immediately raises
#   NameError: expected file app/models/user_profile.rb to define UserProfile

# Catching this locally before it reaches production:
# $ RAILS_ENV=production bin/rails runner "puts 'eager load check passed'"
# or in CI:
# $ RAILS_ENV=test bin/rails zeitwerk:check
```

### 99. Give the Little's Law intuition for capacity planning a Rails app — how many Puma workers/threads, DB connections, and Sidekiq workers do you need?

**Short Answer**

Little's Law: **concurrency needed = arrival rate × time per request** (`L = λW`).

So if you get 100 requests/second and each takes 0.25s, you need about 25 requests in flight at once. That number sizes your Puma threads, your database pool, and (using its own numbers) your Sidekiq concurrency.

**Simple Explanation**

If your app receives 100 requests per second and each takes 200ms to finish, you need roughly `100 × 0.2 = 20` requests being processed at any moment just to keep up. With fewer slots, requests queue and latency climbs.

That "20" is your target total Puma concurrency (`workers × threads`), plus headroom for spikes. Use **p99** latency rather than average for this calculation — the slow requests are the ones actually causing queueing, and an average hides them.

The same maths cascades downstream. If most requests touch the database, the pool needs to support that same concurrency. If a chunk of requests enqueue jobs, size Sidekiq against **its own** arrival rate and job duration.

A web tier sized correctly is still bottlenecked if the pool or Sidekiq behind it wasn't sized with the same formula.

**Example**

```ruby
# Little's Law: L = λ * W
# L = concurrency needed, λ = arrival rate (req/s), W = avg time in system (s)

# Example: 100 req/s, p99 latency 250ms (use p99, not average, for headroom)
# L = 100 * 0.25 = 25 concurrent requests needed at peak

# Puma sizing to cover that concurrency, with headroom:
# WEB_CONCURRENCY (workers) x threads >= 25, e.g. 5 workers x 6 threads = 30

# config/puma.rb
workers 5
threads 6, 6

# DB pool must cover the same per-worker thread concurrency:
# config/database.yml
production:
  pool: 6 # >= threads per worker

# Total DB connections needed: 5 workers x 6 pool = 30, must fit under
# the database's max_connections (with room for Sidekiq/other consumers too)

# Sidekiq sized the same way against ITS OWN arrival rate and job duration:
# e.g. 20 jobs/s enqueued, avg job takes 0.5s -> need ~10 concurrent workers
# :concurrency: 15  # in config/sidekiq.yml, with margin above the bare minimum
```

**— Multi-tenancy & Production Database Changes —**

### 100. How do Rails apps handle multi-tenancy — row-level, schema-level, and database-per-tenant?

**Short Answer**

- **Row-level** — shared tables with a `tenant_id` column. Cheapest to run, but every query must remember to filter by tenant.
- **Schema-level** — one Postgres schema per tenant. Stronger isolation, but migrations run once per schema.
- **Database-per-tenant** — strongest isolation, most operational work.

**Simple Explanation**

**Row-level:** every table gets `tenant_id`, and every query needs `WHERE tenant_id = ?`. In practice you enforce that automatically with a default scope or a gem like `acts_as_tenant` rather than remembering it by hand. Cheapest to operate — one schema, one set of migrations, easy cross-tenant reporting. But the isolation is only as strong as your weakest query: one raw SQL statement or one forgotten scope is a real data leak.

**Schema-level:** each tenant gets its own Postgres schema with the same tables. Queries only see their own tenant's data because the connection is switched to that schema, so you don't need `tenant_id` filters everywhere. The cost shows up in operations: a migration has to run against every schema, so 100 tenants means running it 100 times. It gets painful past a few hundred tenants.

**Database-per-tenant:** a completely separate database, possibly on separate servers. Strongest isolation — relevant for enterprise customers with compliance or data-residency requirements — but now you're managing N databases for connections, migrations, backups, and monitoring.

Rule of thumb: row-level for many small tenants; schema or database-per-tenant for a smaller number of large customers where isolation justifies the cost.

**Example**

```ruby
# Row-level multi-tenancy — shared tables, tenant_id on every row
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
end

class Order < ApplicationRecord
  belongs_to :tenant
  default_scope { where(tenant_id: Current.tenant_id) } # every query auto-scoped
end

class ApplicationController < ActionController::Base
  around_action :set_current_tenant

  def set_current_tenant
    Current.tenant_id = current_user.tenant_id
    yield
  ensure
    Current.tenant_id = nil
  end
end

# Schema-level (via the apartment gem) — switches Postgres schema per request
class ApplicationController < ActionController::Base
  around_action :switch_tenant_schema

  def switch_tenant_schema
    Apartment::Tenant.switch(current_user.tenant.subdomain) { yield }
  end
end
# migrations run per schema: Apartment::Migrator.migrate("tenant_acme")

# Database-per-tenant — separate connection config resolved per request
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
end

class TenantResolver
  def self.connect!(tenant)
    ActiveRecord::Base.establish_connection(
      "postgres://user:pass@#{tenant.db_host}/tenant_#{tenant.id}"
    )
  end
end
```

### 101. How would you safely change a production database schema — e.g. add a `NOT NULL` column to a 50-million-row table?

**Short Answer**

Use **expand → backfill → contract**:

1. **Expand** — add the column as nullable, deploy code that writes to it.
2. **Backfill** — fill existing rows in small batches.
3. **Contract** — only then add the `NOT NULL` constraint and remove old code/columns.

Adding a `NOT NULL` column directly can lock a 50-million-row table while it rewrites, which is an outage.

**Simple Explanation**

On modern Postgres (11+), adding a plain nullable column with no default is a fast metadata-only change. But adding `NOT NULL` directly, or a non-constant default, can force a full table rewrite while holding a lock — minutes of the table being unwritable.

The three steps in practice:

1. **Expand:** migrate to add the column as nullable, no constraint. Deploy code that writes to it, while tolerating `nil` for rows not backfilled yet. Purely additive, so nothing breaks.
2. **Backfill:** populate existing rows in batches with `in_batches`, with a small `sleep` between them, rather than one giant UPDATE. That avoids a long-running transaction and keeps replication lag down.
3. **Contract:** once every row is filled and the deployed code no longer needs to handle `nil`, add the `NOT NULL` constraint in a separate deploy, and drop any old column later.

The other discipline that makes this safe: **every migration should be compatible with the previously deployed code for at least one deploy cycle.** Deploys aren't instant and rollbacks happen, so there's always a window where old code and new schema coexist.

**Example**

```ruby
# Step 1 — EXPAND: add nullable, no constraint yet (fast, no long lock)
class AddEmailVerifiedToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :email_verified, :boolean # nullable, no default rewrite
  end
end

# deploy app code that writes to the new column going forward,
# tolerant of it still being nil for not-yet-backfilled rows:
class User < ApplicationRecord
  before_save :set_email_verified_default
  def set_email_verified_default
    self.email_verified = false if email_verified.nil? && new_record?
  end
end

# Step 2 — BACKFILL in batches, throttled, no single giant transaction/lock
User.where(email_verified: nil).in_batches(of: 5_000) do |batch|
  batch.update_all(email_verified: false)
  sleep(0.1) # keep replication lag / lock contention low
end

# Step 3 — CONTRACT: only after backfill is confirmed complete and app code
# no longer needs to tolerate NULL, add the constraint (separate deploy)
class AddNotNullToUsersEmailVerified < ActiveRecord::Migration[7.1]
  def change
    change_column_null :users, :email_verified, false
    change_column_default :users, :email_verified, false
  end
end
```

**— Real-World Senior Debugging Questions —**

### 102. How would you debug a slow Rails endpoint in production?

**Short Answer**

Start with real production data, not a guess. Use an APM trace to see where the time actually went — database, external call, or Ruby/view rendering — then reach for the tool that matches that layer:

- N+1 queries → `bullet`
- Slow single query → `EXPLAIN ANALYZE`
- Per-request breakdown → `rack-mini-profiler`

**Simple Explanation**

My first move is an APM tool (Datadog, New Relic, Scout, Skylight). It breaks a slow request into a trace showing X ms in SQL, Y ms in view rendering, Z ms in an external HTTP call, and which specific queries or partials dominated. That turns "the endpoint is slow" into a ranked list instead of a guessing game.

**If it's SQL time:** look for the same query shape repeated dozens of times — that's the N+1 signature. `bullet` flags these automatically.

**For one slow query:** run `EXPLAIN ANALYZE` on it. A sequential scan where an index should be used, a bad join order, or a missing index on a filtered column are the usual culprits.

**If it's an external API:** move the call to a background job if it doesn't need to block the response, or add caching.

**If it's Ruby or view rendering:** `rack-mini-profiler` gives a per-request breakdown.

The discipline throughout: measure with production-scale data. A query that's instant against 200 local rows can be a full table scan against 50 million.

**Example**

```ruby
# Gemfile (dev/test group)
gem "bullet"
gem "rack-mini-profiler"

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
end

# Investigating a specific slow query flagged by APM:
# EXPLAIN ANALYZE
# SELECT * FROM orders WHERE customer_id = 4821 AND status = 'pending';
#
#  Seq Scan on orders  (cost=0.00..184532.00 rows=12 width=97)
#    (actual time=812.331..812.335 rows=3 loops=1)
#    Filter: ((customer_id = 4821) AND (status = 'pending'))
#    Rows Removed by Filter: 4999987
#
# -> sequential scan over ~5M rows for 3 matches: missing index

# fix:
class AddIndexToOrdersCustomerStatus < ActiveRecord::Migration[7.1]
  def change
    add_index :orders, [:customer_id, :status]
  end
end
```

### 103. How would you identify and fix an N+1 query in practice?

**Short Answer**

Find it with `bullet` in development or CI, or by spotting the same query repeated N times in production logs and APM traces.

Fix it by eager loading the association with `includes`, then **verify the query count actually dropped**.

**Simple Explanation**

In development, `bullet` hooks into Active Record and warns whenever an association is loaded lazily in a loop that could have been eager-loaded. It can log a warning, show a browser footer, or raise in test mode.

In production, the signature is the same query shape firing back to back with different IDs — `SELECT * FROM authors WHERE id = ?` fifty times in a row. That repetition is the tell.

The fix is almost always adding `.includes(:association)` where the collection is first queried — usually the controller action or a scope.

Then verify. Check the SQL log, confirm the bullet warning cleared, or add a query-count assertion in a request spec so the fix can't silently regress later.

**Example**

```ruby
# BEFORE — Bullet flags this: "N+1 Query detected: Post => [:author]"
class PostsController < ApplicationController
  def index
    @posts = Post.limit(50)
  end
end
# view: <%= post.author.name %> inside a @posts.each loop -> 1 + 50 queries

# AFTER — eager loaded, Bullet warning clears
class PostsController < ApplicationController
  def index
    @posts = Post.includes(:author).limit(50)
  end
end
# 2 queries total regardless of how many posts

# Verifying the fix holds with a request spec query-count assertion
RSpec.describe "GET /posts", type: :request do
  it "does not N+1 on authors" do
    create_list(:post, 10)
    expect { get "/posts" }.to make_database_queries(count: 2..3) # e.g. via query_count gem/matcher
  end
end
```

### 104. "Your Rails API's p95 suddenly goes from 300ms to 4s. CPU is normal. DB connections are at 100% of pool. Sidekiq latency is climbing." Walk through the investigation.

**Short Answer**

Normal CPU plus a maxed-out connection pool means threads are **waiting**, not computing. So something is holding connections open longer than usual: a query that got slow, a lock, a leaked connection, or an external API call made while holding a connection.

Sidekiq latency climbing at the same time strongly suggests one **shared** bottleneck — most likely the database — rather than two separate incidents.

**Simple Explanation**

If CPU were the problem, it'd be pegged. Normal CPU with an exhausted pool means threads are alive and busy waiting on I/O.

My first move is to look at the database directly: what queries are running right now, and are there locks? In Postgres, `pg_stat_activity` answers both.

If I see many connections in `idle in transaction`, that's the smoking gun — something opened a transaction and isn't committing promptly, holding the connection (and often locks) the whole time. A classic cause is exactly the callback problem from earlier: an `after_create` calling a slow external API while still inside the open transaction.

If queries look fast but the pool is still exhausted, the problem is upstream: a thread stuck on a slow external HTTP call **while holding a checked-out connection**, multiplied across enough requests to drain the pool. Everyone else queues behind that, which is why p99 spikes hard while p50 may still look fine.

Sidekiq backing up at the same time matters. If it shares the same database or Redis, that's evidence of one shared resource degrading, not two coincidences. That reframes the question from "what's wrong with the web app" to "what's wrong with the database right now".

**Investigation order**

1. Check `pg_stat_activity` for long-running or `idle in transaction` connections and lock waits, live.
2. Line up the incident start time with the most recent deploy — did a new query path or callback just ship?
3. Check whether Sidekiq and web share the same database or Redis. If so, treat it as one incident.
4. Look for external API calls made synchronously inside a request or transaction, and check that provider's status.
5. Fix: move the external call to `after_commit` plus a background job, add the missing index, or kill the offending long transaction to relieve pressure while the real fix ships.

**Example**

```sql
-- Live investigation: find connections stuck idle-in-transaction or blocked
SELECT pid, state, wait_event_type, query, now() - xact_start AS tx_duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY tx_duration DESC
LIMIT 20;

-- If several rows show state = 'idle in transaction' with a long tx_duration,
-- that's the smoking gun: something is holding a transaction open, likely
-- an after_create doing a slow external call before the transaction commits.
```

```ruby
# The likely structural culprit, matching the "callbacks in a transaction" issue:
class Order < ApplicationRecord
  after_create :notify_shipping_partner # <- synchronous external call, INSIDE
                                          #    the still-open transaction, holding
                                          #    the DB connection for its full duration
  def notify_shipping_partner
    ShippingPartnerApi.notify(self) # this partner just started responding slowly
  end
end

# Fix: move it out of the transaction entirely, and off the request cycle
class Order < ApplicationRecord
  after_commit :notify_shipping_partner, on: :create

  def notify_shipping_partner
    ShippingPartnerNotificationJob.perform_later(id)
  end
end
```

### 105. "Memory usage climbs steadily over hours and the process eventually gets OOM-killed." How do you distinguish a real leak from expected GC sawtooth?

**Short Answer**

Normal GC looks like a **sawtooth**: memory rises, GC reclaims it, repeat — within a stable range.

A real leak is when the **bottom** of that sawtooth keeps rising, because each GC reclaims less than the last. Something is holding references GC can never collect.

**Simple Explanation**

Ruby memory isn't flat, and it shouldn't be. Allocate, allocate, GC frees a chunk, allocate again. What matters is whether the *floor* stays level over hours.

If the floor creeps up regardless of traffic, something is being retained that should be garbage. Common Rails causes:

- `Rails.cache` misconfigured to `:memory_store` in production with no size limit (should be Redis, which lives outside the process and evicts on its own).
- A constant or class-level array that every request appends to and nothing ever clears.
- `ActiveSupport::Notifications.subscribe` called inside a controller action instead of once at boot, adding one more listener per request forever.
- Less commonly, a genuine leak in a native C extension.

To confirm and localise it: take an `ObjectSpace.count_objects` snapshot shortly after boot and another a few hours later, and see which class grew out of proportion to traffic. For more detail, take a heap dump with `ObjectSpace.dump_all` at both points and diff them with a tool like `heapy` or `derailed_benchmarks`.

**Example**

```ruby
# Cheap first pass — compare object counts at two points in time
before = ObjectSpace.count_objects
# ... let the app run under normal traffic for a few hours ...
after = ObjectSpace.count_objects
after.each do |klass, count|
  growth = count - (before[klass] || 0)
  puts "#{klass}: +#{growth}" if growth > 10_000
end

# Common real culprit #1 — in-process cache with no bound, should be Redis
# config/environments/production.rb
config.cache_store = :memory_store # BUG: unbounded within this process,
                                    # should be :redis_cache_store in production

# Common real culprit #2 — a global collection nothing ever trims
class RequestLog
  ENTRIES = [] # BUG: every request appends, nothing ever removes
  def self.record(req)
    ENTRIES << req # grows forever, retained by GC because ENTRIES is reachable
  end
end

# Common real culprit #3 — subscribing inside a request instead of once at boot
class ReportsController < ApplicationController
  def show
    # BUG: adds a NEW subscriber on every single request, none ever unsubscribed
    ActiveSupport::Notifications.subscribe("process_report") { |*args| track(args) }
  end
end
# fix: move the subscribe call to a config/initializers file, run once at boot
```

### 106. "Sidekiq queue depth is growing but p50 API latency looks completely normal." Why does this point at the worker tier, not the web tier?

**Short Answer**

Normal web latency means requests are being answered fine — the problem is downstream in job processing.

Three possible causes: one job class got slow and is backing up everything behind it, the enqueue rate spiked, or worker capacity dropped. Check **per-job-class latency** before just adding more workers.

**Simple Explanation**

Queue depth growing means jobs are being created faster than they're finished. That's arithmetic, not a web problem — `perform_later` is just a fast Redis write regardless of how backed up the queue is.

So the investigation moves entirely to Sidekiq.

**First:** check per-job-class duration in the Sidekiq UI or APM. Aggregate queue depth tells you *that* something's wrong, not *what*. If `ReportGenerationJob` went from 2s to 30s, that one class can back up everything behind it. Adding workers helps a bit, but the real fix is finding why it got slow — usually the same causes as a slow endpoint: a new N+1, a slow downstream API, a missing index.

**Second:** check the enqueue *rate* over time, not just current depth. A new feature or a bug re-enqueueing in a loop can flood the queue.

**Third:** check worker capacity. A deploy may have shipped lower concurrency, fewer pods may be running, or the jobs' downstream dependency got slow.

Adding workers only helps for the second and third causes. For the first, more workers running a badly-performing job just hold more database connections for longer and can make a shared bottleneck worse.

**Example**

```ruby
# Sidekiq UI / API: check per-job-class latency, not just aggregate queue depth
require "sidekiq/api"

Sidekiq::Queue.new("default").each do |job|
  puts "#{job.klass}: enqueued_at=#{job.enqueued_at}, latency=#{job.latency}s"
end

# Grouping to find which class is actually backing things up:
stats = Sidekiq::Queue.new("default").group_by(&:klass).transform_values(&:size)
# => { "ReportGenerationJob" => 4200, "WelcomeEmailJob" => 12 }
# -> ReportGenerationJob is clearly the one backing up the queue

# Check if worker capacity itself regressed (e.g. after a recent deploy)
Sidekiq::ProcessSet.new.each do |process|
  puts "#{process["hostname"]}: concurrency=#{process["concurrency"]}, busy=#{process["busy"]}"
end
# if concurrency dropped from 20 to 5 after the last deploy, that's the root cause
# before touching worker count at all
```

### 107. How would you handle a race condition in a Rails app — give a concrete example.

**Short Answer**

Classic example: two requests both buy the last item in stock. Both read `quantity: 1`, both decide it's available, both decrement — and you've oversold.

Two fixes:

1. **Pessimistic lock** — `SELECT ... FOR UPDATE` so the second request waits and re-reads.
2. **Atomic update** — one SQL statement that checks and decrements together, then check how many rows it actually changed.

**Simple Explanation**

The bug pattern is "read, decide, write" split across separate steps in Ruby:

```ruby
if product.quantity > 0
  product.update!(quantity: product.quantity - 1)
end
```

That isn't atomic. Two requests can both pass the `if` before either writes.

**Pessimistic locking** (`Product.lock.find(id)` inside a transaction) makes the second request's query block until the first commits, so it then reads the already-decremented value. Correct, but the second request waits — and under heavy contention on one popular product, that becomes a throughput bottleneck.

**Atomic update** pushes the whole check-and-write into one statement:

```ruby
Product.where(id: id).where("quantity > 0").update_all("quantity = quantity - 1")
```

It returns how many rows were updated. If that's `0`, someone else got the last one — no locking, no waiting.

I prefer the atomic update for a simple decrement because nobody blocks. I'd use pessimistic locking when the operation is more complex than one column of arithmetic — for example, when you need to read several related values and make a multi-step decision before writing.

**Example**

```ruby
# BUGGY — read-then-write race condition, two requests can both pass the check
def purchase(product_id)
  product = Product.find(product_id)
  if product.quantity > 0
    product.update!(quantity: product.quantity - 1) # both requests can get here
    Order.create!(product: product)
  else
    raise OutOfStockError
  end
end

# FIX 1 — pessimistic locking: second request blocks, then re-reads fresh data
def purchase(product_id)
  ActiveRecord::Base.transaction do
    product = Product.lock.find(product_id) # SELECT ... FOR UPDATE, blocks concurrent lockers
    raise OutOfStockError if product.quantity <= 0
    product.update!(quantity: product.quantity - 1)
    Order.create!(product: product)
  end
end

# FIX 2 — atomic DB-level update, no locking/blocking at all
def purchase(product_id)
  updated_rows = Product.where(id: product_id).where("quantity > 0")
                         .update_all("quantity = quantity - 1")
  raise OutOfStockError if updated_rows.zero? # someone else took the last one
  Order.create!(product_id: product_id)
end
```

### 108. How would you handle a long-running background job?

**Short Answer**

Break it into chunks. Each run processes a fixed-size batch, saves its position (a cursor) somewhere durable, and re-enqueues itself for the next chunk.

That makes it resumable after a crash, visible while it's running, and safely under any job timeout.

**Simple Explanation**

A single job looping over 10 million rows for an hour has three problems:

1. If the process restarts mid-run (deploy, crash, infra blip), all progress is lost.
2. Sidekiq or your infrastructure likely has a timeout that will kill it, again losing everything.
3. There's no visibility — it's a black box until it finishes or doesn't.

The fix is resumable chunks. Each invocation processes, say, 1,000 records, records where it stopped (last processed ID, stored in the database or Redis — not just in memory), and enqueues itself to continue from there.

Now a crash only loses one chunk's worth of work, progress is visible by reading the cursor, and no single execution risks a timeout.

For big one-off jobs, `in_batches` / `find_in_batches` combined with a self-re-enqueuing job is the standard Rails pattern.

**Example**

```ruby
class BulkRecalculateScoresJob < ApplicationJob
  BATCH_SIZE = 1_000

  def perform(cursor_id = 0)
    users = User.where("id > ?", cursor_id).order(:id).limit(BATCH_SIZE)
    return if users.empty? # done

    users.each { |user| user.update!(score: ScoreCalculator.call(user)) }

    last_id = users.last.id
    Rails.cache.write("bulk_recalc_scores:cursor", last_id) # durable progress tracking

    # re-enqueue for the next chunk instead of looping in one long-running job
    BulkRecalculateScoresJob.perform_later(last_id)
  end
end

# kick it off (or resume from a stored cursor after a crash/restart)
resume_from = Rails.cache.read("bulk_recalc_scores:cursor") || 0
BulkRecalculateScoresJob.perform_later(resume_from)

# progress is visible any time by just checking the cursor:
# Rails.cache.read("bulk_recalc_scores:cursor") #=> 482000  (out of ~10,000,000)
```

### 109. How do you structure business logic in a growing Rails app so it stays maintainable?

**Short Answer**

Keep controllers thin, keep models focused on their own data, and pull real operations out into **service objects**, **query objects**, and **form objects** as they earn it. Use concerns sparingly — only for genuinely reusable behavior.

**Simple Explanation**

The failure mode is well known: early on, putting everything in the model is convenient. Two years later `User` is 2,000 lines and nobody can say what creating a user actually does without reading half the codebase.

The discipline is a layered set of rules applied consistently:

- **Controllers** — receive params, call a model or service, pick what to render. Two or three lines when the operation really is that simple.
- **Models** — validations, associations, and behavior about *that record's own state*. Not every operation that touches it.
- **Service objects** — operations that span models, call external systems, or need a transaction. Extracted when they actually earn it, not preemptively.
- **Query objects** — complex, reused read logic, instead of sprawling scopes or copy-pasted queries.
- **Form objects** — when a form doesn't map to one model.
- **Concerns** — genuinely reusable behavior shared by multiple models (`Sluggable`, `Archivable`). If it only makes sense in one model and reaches into that model's state, it's concern soup, and there's probably a service object hiding inside it.

None of this gets applied from day one of a small app. These are extraction points you reach for when complexity actually shows up — which is why recognising over-engineering matters as much as recognising under-engineering.

**Example**

```ruby
# The shape of a maintainable growing app — each piece owns exactly one concern

# Controller — thin orchestration only
class SubscriptionsController < ApplicationController
  def cancel
    result = CancelSubscription.call(current_user.subscription)
    result.success? ? redirect_to(account_path, notice: "Canceled") :
                       redirect_to(account_path, alert: result.error)
  end
end

# Model — owns its own data/rules, not every operation involving it
class Subscription < ApplicationRecord
  belongs_to :user
  validates :status, inclusion: { in: %w[active canceled past_due] }
  scope :active, -> { where(status: "active") }
end

# Service object — owns the multi-step operation, reused from controller/job/rake
class CancelSubscription
  Result = Struct.new(:success?, :error)
  def self.call(subscription)
    ActiveRecord::Base.transaction do
      subscription.update!(status: "canceled", canceled_at: Time.current)
      subscription.user.update!(plan: "free")
    end
    PaymentGateway.cancel!(subscription.external_id)
    Result.new(true, nil)
  rescue PaymentGateway::Error => e
    Result.new(false, e.message)
  end
end

# Query object — owns complex, reused read logic
class ExpiringTrialSubscriptionsQuery
  def call
    Subscription.active.where(plan: "trial").where("trial_ends_at < ?", 3.days.from_now)
  end
end

# Concern — reserved for genuinely reusable, self-contained behavior only
module Cancelable
  extend ActiveSupport::Concern
  included { scope :canceled, -> { where(status: "canceled") } }
  def canceled?
    status == "canceled"
  end
end
```

### 110. How do you design a maintainable Rails application overall?

**Short Answer**

Respect MVC boundaries, keep each layer doing one kind of thing, push real data guarantees down to the database, treat the test suite as the actual safety net, and deliberately avoid both fat models and needless ceremony.

The goal: a new engineer can guess where any piece of logic lives without being told.

**Simple Explanation**

Pulling the whole section together, maintainability comes from consistently applying a small set of rules:

1. **MVC and conventions** as the base layer, so the layout is predictable.
2. **Thin controllers**, with logic extracted into services/queries/forms *when complexity warrants it*, never reflexively.
3. **The database as the real backstop** — a unique index alongside a uniqueness validation, `NOT NULL` and foreign keys alongside presence validations. Validations are bypassable; constraints aren't.
4. **Query discipline as a habit** — default to `includes` when you know you'll touch an association, use `find_each` on large tables — rather than discovering N+1s in production.
5. **Background jobs for anything slow or unreliable**, with small arguments, safe to retry, and `after_commit` (never `after_save`) for anything with an external side effect.
6. **Backward-compatible schema changes** (expand/contract), because there's always a window where old code and new schema run together.
7. **The test suite as the safety net** for everyday changes, framework upgrades, and risky migrations. Manual QA doesn't scale; a good suite runs on every deploy.
8. **Resisting both extremes** — a service object for a two-line CRUD action is as much of a problem as a 2,000-line model, just in the other direction.

**Example**

```ruby
# A maintainable slice of a real feature, showing the boundaries working together

# 1. DB constraint — the real backstop
class AddConstraintsToSubscriptions < ActiveRecord::Migration[7.1]
  def change
    add_foreign_key :subscriptions, :users
    add_check_constraint :subscriptions, "status IN ('active','canceled','past_due')",
                          name: "subscriptions_status_check"
  end
end

# 2. Model — validation mirrors the constraint, for fast/friendly UX
class Subscription < ApplicationRecord
  belongs_to :user
  validates :status, inclusion: { in: %w[active canceled past_due] }
end

# 3. Service object — the actual business operation, transaction-bounded
class DowngradeToFreePlan
  def self.call(subscription)
    ActiveRecord::Base.transaction do
      subscription.update!(status: "canceled")
      subscription.user.update!(plan: "free")
    end
  end
end

# 4. after_commit, not after_save — external side effect only after the DB guarantees it happened
class Subscription < ApplicationRecord
  after_commit :notify_user_of_cancellation, on: :update, if: :saved_change_to_status?

  def notify_user_of_cancellation
    SubscriptionMailer.canceled(self).deliver_later if status == "canceled"
  end
end

# 5. Thin controller — orchestration only
class SubscriptionsController < ApplicationController
  def downgrade
    DowngradeToFreePlan.call(current_user.subscription)
    redirect_to account_path, notice: "Your plan has been downgraded."
  end
end

# 6. A test suite covering the behavior, not just the happy path
RSpec.describe DowngradeToFreePlan do
  it "cancels the subscription and resets the user's plan atomically" do
    subscription = create(:subscription, status: "active", user: create(:user, plan: "pro"))
    DowngradeToFreePlan.call(subscription)
    expect(subscription.reload.status).to eq("canceled")
    expect(subscription.user.reload.plan).to eq("free")
  end
end
```

## PostgreSQL / SQL

**— SQL Fundamentals —**

### 111. How do SELECT, WHERE, and ORDER BY work together in a basic query?

**Short Answer**

- `SELECT` picks which columns you get back.
- `WHERE` filters rows.
- `ORDER BY` sorts the result.

The database runs them as `FROM` → `WHERE` → `SELECT` → `ORDER BY`, no matter what order you type them in.

**Simple Explanation**

The order you *write* is not the order the database *executes*.

It first decides which rows to look at (`FROM`), filters them (`WHERE`), then picks the columns you asked for (`SELECT`), then sorts (`ORDER BY`).

That explains two things people find odd:

- You **can** `ORDER BY` a column you didn't select — sorting happens after the rows are already chosen.
- You **can't** use a `SELECT` alias in `WHERE` — the alias doesn't exist yet at that stage.

`LIMIT` and `OFFSET` run last, after sorting. That's the normal way to paginate.

**Example**

```sql
SELECT id, email, status, created_at
FROM users
WHERE status = 'active'
  AND created_at >= NOW() - INTERVAL '30 days'
ORDER BY created_at DESC
LIMIT 20;
```

### 112. What's the difference between filtering with WHERE and filtering with HAVING?

**Short Answer**

- `WHERE` filters individual **rows**, before grouping.
- `HAVING` filters **groups**, after `GROUP BY` has calculated the aggregates.

You can't use `COUNT()` in `WHERE`, because grouping hasn't happened yet.

**Simple Explanation**

Think of it as a two-stage pipeline.

`WHERE` throws out rows you never wanted at all — cancelled orders, for example.

Then `GROUP BY` collapses what's left into groups and calculates aggregates per group.

Then `HAVING` throws out entire groups based on those aggregates — "only keep customers with more than 3 orders".

Quick rule: if the condition is about one raw row, use `WHERE`. If it depends on an aggregate, use `HAVING`.

**Example**

```sql
SELECT user_id, COUNT(*) AS order_count
FROM orders
WHERE status <> 'cancelled'          -- filters rows before grouping
GROUP BY user_id
HAVING COUNT(*) > 3                  -- filters groups after aggregation
ORDER BY order_count DESC;
```

### 113. What's the difference between INNER JOIN, LEFT JOIN, and RIGHT JOIN?

**Short Answer**

- `INNER JOIN` — only rows that match in **both** tables.
- `LEFT JOIN` — every row from the left table, plus matches from the right (`NULL` where there's no match).
- `RIGHT JOIN` — the mirror image. Rarely used, since you can just swap the table order and use `LEFT JOIN`.

**Simple Explanation**

Picture `users` on the left and `orders` on the right.

`INNER JOIN` keeps only users who actually placed an order. Anyone with zero orders disappears completely.

`LEFT JOIN` keeps **every** user. For users with no orders, all the order columns come back as `NULL`. That's the standard way to answer "which users have never ordered?" — `LEFT JOIN` then `WHERE orders.id IS NULL`.

`RIGHT JOIN` keeps every row from the right table instead. It's just a `LEFT JOIN` with the tables written the other way around, so most people never use it.

**Example**

```sql
-- INNER JOIN: only users who have placed at least one order
SELECT u.id, u.email, o.id AS order_id, o.total_cents
FROM users u
INNER JOIN orders o ON o.user_id = u.id;

-- LEFT JOIN: every user, even those with zero orders (order columns are NULL)
SELECT u.id, u.email, o.id AS order_id, o.total_cents
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;   -- isolates users who have never placed an order

-- RIGHT JOIN: mirror of LEFT JOIN; equivalent to swapping the FROM/JOIN tables
SELECT u.id, u.email, o.id AS order_id
FROM orders o
RIGHT JOIN users u ON o.user_id = u.id;
```

### 114. What is a subquery, and how is it different from a JOIN?

**Short Answer**

A subquery is a `SELECT` nested inside another query — in `WHERE`, in `FROM`, or in the select list. It computes a set of values or a temporary table that the outer query uses.

**Simple Explanation**

There are a few shapes:

- **Scalar subquery** — returns one value.
- **`IN` / `EXISTS` subquery** — returns a set of values used for filtering.
- **Subquery in `FROM`** — acts as a temporary table ("derived table") you can join against.

Performance-wise, the planner often rewrites a subquery into an equivalent join internally, so they're usually comparable. The real difference is readability — "rows where this ID appears in that other filtered set" reads more naturally as a subquery than as a join.

**Example**

```sql
-- subquery in WHERE: users who've spent over $1,000 lifetime
SELECT id, email
FROM users
WHERE id IN (
  SELECT user_id FROM orders GROUP BY user_id HAVING SUM(total_cents) > 100000
);

-- subquery in FROM (a "derived table"): users with more than 5 orders
SELECT sub.user_id, sub.order_count
FROM (
  SELECT user_id, COUNT(*) AS order_count
  FROM orders
  GROUP BY user_id
) sub
WHERE sub.order_count > 5;
```

**— Transactions & ACID —**

### 115. What are the ACID properties of a database transaction?

**Short Answer**

- **Atomicity** — all or nothing. If any part fails, the whole thing rolls back.
- **Consistency** — a transaction moves the database from one valid state to another, never breaking constraints.
- **Isolation** — concurrent transactions don't see each other's half-finished work.
- **Durability** — once it commits, it survives a crash or power loss.

**Simple Explanation**

The classic example is a money transfer: take ₹1,000 out of Account A, put ₹1,000 into Account B.

**Atomicity** means both happen or neither does. If the app crashes between them, the database rolls the whole thing back — you never end up with money vanishing.

**Consistency** means rules like "balance can't go negative" (a `CHECK` constraint) can't be broken along the way. If the debit would break that rule, the transaction is rejected.

**Isolation** means if two people try to buy the last item at the same time, they don't both see "1 in stock" and both succeed. Each transaction behaves as if it ran on its own.

**Durability** means once `COMMIT` returns, the write is safely on disk. Postgres guarantees this by writing to the WAL (write-ahead log) and flushing it before the commit returns.

**Example**

```sql
-- Atomicity: both updates happen, or neither does
BEGIN;
UPDATE accounts SET balance_cents = balance_cents - 5000 WHERE id = 1;  -- debit
UPDATE accounts SET balance_cents = balance_cents + 5000 WHERE id = 2;  -- credit
COMMIT;   -- if the app crashes before COMMIT, Postgres rolls the whole thing back

-- Consistency: a CHECK constraint enforces an invariant no transaction may violate
ALTER TABLE accounts ADD CONSTRAINT balance_non_negative CHECK (balance_cents >= 0);
-- the debit above is rejected (and the transaction rolled back) if it would
-- drive balance_cents negative

-- Isolation: two checkouts racing for the last unit of stock don't both win
BEGIN;
SELECT quantity FROM inventory WHERE product_id = 42 FOR UPDATE;  -- blocks the other transaction
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42 AND quantity > 0;
COMMIT;

-- Durability: once COMMIT returns, the write survives a crash
-- (Postgres fsyncs the WAL entry to disk before COMMIT is allowed to return)
```

### 116. What do Postgres's transaction isolation levels actually prevent?

**Short Answer**

Postgres never allows **dirty reads** at any level, thanks to MVCC. The levels differ in what else they prevent:

- **Read Committed** (default) — each statement sees the latest committed data, so re-running a query can give different results.
- **Repeatable Read** — your whole transaction sees one frozen snapshot, so re-running a query gives the same answer.
- **Serializable** — adds protection against subtler conflicts where two transactions each make a decision that's only safe in isolation.

**Simple Explanation**

Three problems, in increasing subtlety:

- **Dirty read** — seeing another transaction's *uncommitted* changes. Postgres never does this.
- **Non-repeatable read** — you run the same `SELECT` twice in one transaction and get a different **value**, because someone committed a change in between. Possible under Read Committed, prevented under Repeatable Read.
- **Phantom read** — you run the same filtered query twice and get **different rows**, because someone inserted or deleted matching rows. Postgres's Repeatable Read prevents this too (it's implemented as full snapshot isolation, which is stricter than the SQL standard requires).

Serializable goes furthest, catching write-skew — where two transactions each read the state, each make a decision that's fine alone, but together break a rule.

**Example**

```sql
-- Session A
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM orders WHERE status = 'pending';  -- returns 10

-- Session B (runs concurrently, then commits)
BEGIN;
INSERT INTO orders (user_id, status, total_cents) VALUES (7, 'pending', 2500);
COMMIT;

-- back in Session A, same transaction, later
SELECT COUNT(*) FROM orders WHERE status = 'pending';
-- still returns 10 under REPEATABLE READ (the snapshot is frozen at BEGIN)
-- would return 11 under READ COMMITTED (each statement sees the latest commit)
COMMIT;
```

**— Indexing —**

### 117. How does a B-tree index speed up a query, and when can an index actually hurt?

**Short Answer**

A B-tree index is a sorted structure the database can binary-search, so lookups go from scanning every row (O(n)) to a handful of comparisons (O(log n)).

The cost: every `INSERT`, `UPDATE`, and `DELETE` must also update every index on that table. A write-heavy table with six indexes is doing seven writes per insert.

**Simple Explanation**

Without an index, `WHERE user_id = 12345` forces Postgres to check every row — a sequential scan.

With a B-tree index on `user_id`, Postgres walks down a sorted tree and lands on the matching rows in a few steps.

But indexes aren't free. Each one adds work on every write and takes up disk space. So you index the columns you actually query on, and you think twice before adding more indexes to a very high-write table.

**Example**

```sql
-- before: full sequential scan of orders
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 12345;
-- Seq Scan on orders (cost=0.00..18334.00 rows=42 width=64) (actual time=45.211..45.213 rows=42 loops=1)

CREATE INDEX idx_orders_user_id ON orders (user_id);

-- after: Postgres can use an index scan instead
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 12345;
-- Index Scan using idx_orders_user_id on orders (cost=0.42..8.44 rows=42 width=64) (actual time=0.031..0.045 rows=42 loops=1)
```

### 118. Why does column order matter in a composite (multi-column) index?

**Short Answer**

A composite index is sorted left to right by its columns. It can be used for queries filtering on a **leading prefix** of those columns, but generally not for a query that only filters on a later column.

**Simple Explanation**

Think of an index on `(user_id, status)` like a phone book sorted by last name, then first name.

You can jump straight to "Smith", or "Smith, John". But you can't efficiently find "everyone named John" without scanning the whole book.

So an index on `(user_id, status)` helps:

- `WHERE user_id = 123`
- `WHERE user_id = 123 AND status = 'pending'`

But it generally won't help `WHERE status = 'pending'` alone, because `status` isn't a leading prefix. For that you'd need a separate index.

**Example**

```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

-- uses the index: filters on the leading column, or both columns
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 12345;
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 12345 AND status = 'pending';

-- cannot use this index efficiently: status alone isn't a leading prefix
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';
-- falls back to a Seq Scan, or would need a separate index on (status) alone
```

### 119. What's the conceptual difference between a clustered and a non-clustered index?

**Short Answer**

- **Clustered index** — the table's rows are physically stored in the index's order.
- **Non-clustered index** — a separate structure holding sorted keys that point back to the rows.

Postgres has no true clustered indexes. Every Postgres index is non-clustered.

**Simple Explanation**

In engines like MySQL's InnoDB, the primary key *is* the physical storage order, maintained automatically forever.

Postgres works differently. Rows live in a heap (roughly insertion order), and every index — including the primary key's — is a separate structure pointing back into that heap.

Postgres does have a `CLUSTER` command that physically reorders a table's rows once to match a chosen index, which can speed up range scans. But it's a **one-time** operation: rows inserted afterwards go back to unordered placement, so you'd have to re-run it periodically to keep the benefit.

**Example**

```sql
-- one-time physical reorder of the orders table to match this index's order
CLUSTER orders USING idx_orders_user_id;
-- note: rows inserted after this are NOT kept in this order automatically;
-- CLUSTER would need to be re-run periodically to maintain the benefit
```

### 120. What is a partial index, and when would you use one?

**Short Answer**

A partial index only covers the rows matching a `WHERE` clause in the index definition. That keeps it much smaller and cheaper to maintain.

**Simple Explanation**

Say 95% of your `orders` table is `status = 'completed'`, but your hot queries only ever look at `status = 'active'`.

Indexing every row wastes space and adds write overhead for rows nobody searches by that predicate.

A partial index only includes matching rows, so it stays small, is more likely to fit in memory, and doesn't get touched at all when you write a non-matching row.

**Example**

```sql
-- most hot-path queries only care about active orders, a small slice of the table
CREATE INDEX idx_orders_active ON orders (created_at) WHERE status = 'active';

EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'active' ORDER BY created_at DESC LIMIT 50;
```

### 121. What is a covering index, and how does it enable an index-only scan?

**Short Answer**

A covering index stores extra columns (via `INCLUDE`) so Postgres can answer a query entirely from the index, without reading the table itself. That's called an **index-only scan**.

**Simple Explanation**

Normally an index scan finds the matching row locations, then goes to the table to read any columns not in the index.

If you `INCLUDE` the columns your query actually selects, everything needed is right there in the index, and Postgres can skip that second step. That's a real speedup on hot read paths.

One caveat: it only fully avoids the table when the visibility map says all rows on that page are visible to everyone. `VACUUM` is what keeps that map up to date.

**Example**

```sql
CREATE INDEX idx_orders_user_covering
  ON orders (user_id)
  INCLUDE (status, total_cents);

-- Postgres can satisfy this entirely from the index (Index Only Scan)
EXPLAIN ANALYZE
SELECT status, total_cents FROM orders WHERE user_id = 12345;
```

### 122. What is an expression index, and when would you need one?

**Short Answer**

An expression index is built on the **result of a function**, not the raw column. You need one when your queries always apply the same transformation — like lowercasing an email for case-insensitive lookup.

**Simple Explanation**

A plain index on `email` can't help `WHERE LOWER(email) = ...`, because the index is sorted by the raw values, not the lowercased ones.

An expression index stores the computed values instead, so a query using that same expression can use it.

The key rule: the query must use **exactly** the same expression as the index definition, or Postgres can't match them up.

**Example**

```sql
CREATE UNIQUE INDEX idx_users_lower_email ON users (LOWER(email));

-- the query must use the same expression to hit the index
EXPLAIN ANALYZE SELECT * FROM users WHERE LOWER(email) = LOWER('Jane@Example.com');
```

### 123. When would you reach for a GIN or GiST index instead of a plain B-tree?

**Short Answer**

Use **GIN** or **GiST** when the value isn't a simple orderable scalar — full-text search, array containment, JSONB containment, geometric data. A B-tree's "less than / greater than" ordering doesn't apply there.

- **GIN** — faster to search, slower to write.
- **GiST** — more balanced, and supports nearest-neighbour searches.

**Simple Explanation**

A B-tree assumes "is this less than that?" makes sense for a column. That doesn't work for "does this array contain this element?" or "does this JSON document contain this key/value pair?"

**GIN** (Generalized Inverted Index) indexes the individual elements inside a value and points back to the rows containing them. That's perfect for containment (`@>`) and full-text search.

**GiST** (Generalized Search Tree) is more general-purpose and supports things like nearest-neighbour (`<->`) queries on geometric and range types, with cheaper writes than GIN but somewhat slower lookups.

**Example**

```sql
-- full-text search over product descriptions
ALTER TABLE products ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', name || ' ' || description)) STORED;
CREATE INDEX idx_products_search ON products USING GIN (search_vector);

-- array containment: does this order's tags array contain 'gift'?
CREATE INDEX idx_orders_tags ON orders USING GIN (tags);
SELECT * FROM orders WHERE tags @> ARRAY['gift'];

-- JSONB containment: orders whose metadata contains a given key/value
CREATE INDEX idx_orders_metadata ON orders USING GIN (metadata);
SELECT * FROM orders WHERE metadata @> '{"gift_wrapped": true}';
```

### 124. How do you confirm Postgres is actually using an index, rather than silently falling back to a sequential scan?

**Short Answer**

Run `EXPLAIN` or `EXPLAIN ANALYZE` and read the plan. Look for `Index Scan` / `Index Only Scan` versus `Seq Scan`, and compare the estimated row counts with the actual ones.

**Simple Explanation**

`EXPLAIN` shows the planner's intended plan without running the query. `EXPLAIN ANALYZE` actually runs it and reports real timings and row counts alongside the estimates. Adding `BUFFERS` shows how many pages came from cache versus disk.

If you see a `Seq Scan` on a large table where you expected an index, it's one of four things:

1. The index doesn't exist.
2. The query doesn't match the index's leading columns.
3. The planner decided a sequential scan was genuinely cheaper (common when the query matches a large share of the table).
4. The table statistics are stale.

A big gap between estimated and actual rows is a strong hint to run `ANALYZE` on the table to refresh those statistics.

**Example**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 12345 AND status = 'pending';

-- Index Scan using idx_orders_user_status on orders
--   (cost=0.42..8.44 rows=1 width=64) (actual time=0.031..0.033 rows=2 loops=1)
--   Index Cond: ((user_id = 12345) AND (status = 'pending'::text))
--   Buffers: shared hit=4
-- Planning Time: 0.112 ms
-- Execution Time: 0.055 ms
```

### 125. How do you add an index to a large production table without locking out writes?

**Short Answer**

Use `CREATE INDEX CONCURRENTLY`. A plain `CREATE INDEX` holds a lock that blocks writes for the whole build, which on a big hot table means an outage.

The trade-offs: it takes roughly twice as long, and if it's interrupted it leaves behind an invalid index you have to drop and retry.

**Simple Explanation**

A normal `CREATE INDEX` blocks `INSERT`, `UPDATE`, and `DELETE` until it finishes. On a table with millions of rows that can be minutes of downtime.

`CREATE INDEX CONCURRENTLY` builds the index in several passes so writes keep working the whole time.

If it fails partway (say the connection is killed), you're left with an `INVALID` index that isn't used but still costs write overhead. Find it with a query against `pg_index` and drop it before retrying.

In a Rails migration, `CREATE INDEX CONCURRENTLY` can't run inside a transaction, so you need `disable_ddl_transaction!` at the top of the migration class.

**Example**

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_status ON orders (user_id, status);

-- if it fails partway, it leaves an invalid index behind -- find and clean it up
SELECT indexrelid::regclass, indisvalid FROM pg_index WHERE NOT indisvalid;
DROP INDEX CONCURRENTLY IF EXISTS idx_orders_user_status;
```

In a Rails migration, `CREATE INDEX CONCURRENTLY` can't run inside a transaction block, so the migration needs `disable_ddl_transaction!` at the top of the migration class.

### 126. Walk through how you'd debug a slow query in production, end to end.

**Short Answer**

1. Find the actual slow query with `pg_stat_statements` or slow query logs — don't guess.
2. Run `EXPLAIN (ANALYZE, BUFFERS)` on it.
3. Look for sequential scans or wildly wrong row estimates.
4. Add or fix indexes (with `CONCURRENTLY` in production), or rewrite the query.
5. Re-check and keep monitoring.

**Simple Explanation**

**Find it:** `pg_stat_statements` aggregates real query timings, so you can sort by total or mean execution time and find the actual worst offenders.

**Explain it:** look for a `Seq Scan` on a big table, a huge gap between estimated and actual rows (stale stats — run `ANALYZE`), or a nested loop join blowing up because the inner side isn't indexed.

**Fix it:** check whether a suitable index already exists (`pg_stat_user_indexes` also shows unused indexes worth dropping) and add missing ones with `CREATE INDEX CONCURRENTLY`.

**Or rewrite it:** avoid `SELECT *`, don't wrap an indexed column in a function without a matching expression index, do aggregation in SQL rather than pulling rows into Ruby, and watch for N+1 patterns coming from the app.

**Then verify:** re-run `EXPLAIN ANALYZE`, deploy, and check `pg_stat_statements` afterwards to confirm the mean time actually dropped.

**Example**

```sql
-- find the worst offenders by total time spent (requires the pg_stat_statements extension)
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- then dig into the specific query
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total_cents, u.email
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.created_at >= '2026-09-01' AND o.status = 'refunded';
```

**— Concurrency & Locking —**

### 127. How does Postgres's MVCC model work, and why doesn't a reader block a writer?

**Short Answer**

MVCC (Multi-Version Concurrency Control) means Postgres keeps multiple versions of each row. Every transaction sees a consistent snapshot from when it started.

So readers never block writers, and writers never block readers. Only two writers touching the **same row** block each other.

**Simple Explanation**

Instead of locking rows for reading, Postgres keeps old versions around.

When you `UPDATE` a row, Postgres doesn't overwrite it. It writes a **new** version and marks the old one as no longer current — but doesn't delete it immediately.

Any transaction that started before your update keeps seeing the old version through its snapshot. Transactions starting afterwards see the new one.

That's a huge win over "lock the table for any access": a long-running report doesn't block checkout traffic, and checkout traffic doesn't block the report.

The trade-off is that dead row versions pile up and have to be cleaned. That's what `VACUUM` is for.

**Example**

```sql
-- Session A: takes a snapshot, sees the current committed balance
BEGIN;
SELECT balance_cents FROM accounts WHERE id = 1;  -- 10000, snapshot taken here

-- Session B: updates and commits a new row version concurrently
BEGIN;
UPDATE accounts SET balance_cents = 9000 WHERE id = 1;
COMMIT;

-- Session A: still sees its original snapshot value, unaffected by B's write
SELECT balance_cents FROM accounts WHERE id = 1;  -- still 10000 under REPEATABLE READ
COMMIT;
```

### 128. Why does Postgres need VACUUM, and what happens if a table isn't vacuumed enough?

**Short Answer**

Because of MVCC, `UPDATE` and `DELETE` don't actually remove old row versions — they just mark them dead. `VACUUM` reclaims that space and refreshes planner statistics.

Without enough vacuuming: tables and indexes bloat, scans get slower, and in the extreme you risk transaction ID wraparound.

**Simple Explanation**

Every dead row version still physically occupies space until something reclaims it. `VACUUM` scans the table, marks that space reusable, and updates the visibility map (which is what makes index-only scans possible).

`autovacuum` does this in the background automatically. But on a table with very heavy update churn — a jobs or sessions table — the default thresholds can fall behind. The table then uses far more disk than its live row count suggests, and every scan has to skip over dead rows.

`VACUUM FULL` actually returns space to the operating system by rewriting the whole table, but it takes an exclusive lock, so it usually needs a maintenance window. Plain `VACUUM` doesn't block reads or writes.

**Example**

```sql
-- see dead tuple bloat on a table
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- manually reclaim space and refresh planner statistics
VACUUM ANALYZE orders;

-- a table with heavy UPDATE churn may need more aggressive autovacuum than the defaults
SHOW autovacuum_vacuum_scale_factor;
```

### 129. How do you detect and debug a deadlock, and how do you prevent one?

**Short Answer**

A deadlock is two transactions each holding a lock the other one needs. Postgres detects the cycle (after `deadlock_timeout`, default 1 second) and kills one transaction with a `deadlock detected` error.

Prevent it by always locking rows in a **consistent order** everywhere in your codebase.

**Simple Explanation**

Transaction A locks row 1, then wants row 2. Transaction B locked row 2 and now wants row 1. Neither can move.

Postgres periodically checks for these wait cycles and aborts one transaction (the "victim") so the other can continue. Your app gets an error and normally retries.

The real fix is prevention. If every code path that touches multiple accounts always locks them in a fixed order — sorted by `id`, for example — the cycle can never form in the first place.

**Example**

```
-- Transaction A                               -- Transaction B
BEGIN;                                         BEGIN;
UPDATE accounts SET balance_cents = balance_cents - 100 WHERE id = 1;
                                                UPDATE accounts SET balance_cents = balance_cents - 100 WHERE id = 2;
UPDATE accounts SET balance_cents = balance_cents + 100 WHERE id = 2;  -- waits on B
                                                UPDATE accounts SET balance_cents = balance_cents + 100 WHERE id = 1;  -- waits on A

-- Postgres detects the cycle and aborts one transaction:
-- ERROR:  deadlock detected
-- DETAIL:  Process 1234 waits for ShareLock on transaction 5678; blocked by process 5678.
--          Process 5678 waits for ShareLock on transaction 1234; blocked by process 1234.
```

Fix: always touch accounts in a fixed order, e.g. `ORDER BY id`, before issuing the updates, in every code path in the app.

### 130. What does SELECT ... FOR UPDATE do, and when would you use it?

**Short Answer**

`SELECT ... FOR UPDATE` locks the rows you selected, so no other transaction can update or lock them until yours commits or rolls back.

Use it to serialize a read-then-write sequence, like decrementing inventory.

**Simple Explanation**

MVCC lets readers work without blocking writers, which is usually what you want. But sometimes you specifically need to stop two transactions from reading the same value and both acting on it.

The classic case is two checkouts both seeing "1 in stock".

`SELECT ... FOR UPDATE` takes a row lock at read time. A second transaction running the same statement waits until the first finishes, and then reads the already-updated value.

**Example**

```sql
BEGIN;
-- lock this row so a concurrent checkout can't also see quantity = 1 and both proceed
SELECT quantity FROM inventory WHERE product_id = 42 FOR UPDATE;
-- application checks quantity > 0 in Ruby, then:
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42;
COMMIT;
-- a concurrent transaction running the same SELECT ... FOR UPDATE blocks until
-- this one commits, then sees the already-decremented quantity
```

**— Query Techniques —**

### 131. CTE vs subquery vs temp table — when would a temp table actually win?

**Short Answer**

- **CTE** (`WITH x AS (...)`) — mainly for readability and recursion. Since Postgres 12 a non-recursive CTE can be inlined and optimized like a subquery.
- **Subquery** — inlines directly into the outer query's plan.
- **Temp table** — actually stores rows, so you can index it and reuse it across several separate queries.

A temp table wins when you need the same expensive intermediate result more than once.

**Simple Explanation**

If you only need the intermediate result once, a CTE or subquery is simplest, and modern Postgres treats them similarly.

But if you need to run **several different** queries against the same expensive result — and especially if you want an index on it — materializing it once into a temp table beats recomputing it. Referencing a CTE from separate statements doesn't cache anything; each statement re-runs the work.

**Example**

```sql
-- CTE: readable, fine for a single downstream query
WITH big_spenders AS (
  SELECT user_id, SUM(total_cents) AS lifetime_value
  FROM orders
  GROUP BY user_id
  HAVING SUM(total_cents) > 500000
)
SELECT u.email, bs.lifetime_value
FROM big_spenders bs
JOIN users u ON u.id = bs.user_id;

-- temp table: materialize once, index it, reuse across several later queries
CREATE TEMP TABLE big_spenders AS
SELECT user_id, SUM(total_cents) AS lifetime_value
FROM orders
GROUP BY user_id
HAVING SUM(total_cents) > 500000;

CREATE INDEX ON big_spenders (user_id);

SELECT COUNT(*) FROM big_spenders;
SELECT * FROM big_spenders bs JOIN users u ON u.id = bs.user_id WHERE bs.lifetime_value > 1000000;
-- the second query benefits from the index on the temp table; re-referencing a
-- CTE across separate statements would mean re-running the aggregation each time
```

### 132. How do window functions differ from a GROUP BY aggregate, and what are they useful for?

**Short Answer**

- `GROUP BY` collapses many rows into one row per group.
- A **window function** (`OVER (PARTITION BY ... ORDER BY ...)`) calculates a value per row while every row stays in the result.

Use window functions for ranking, running totals, and comparing a row to its neighbours.

**Simple Explanation**

`GROUP BY` answers "what's the total per user?" and gives you one row per user.

A window function answers "where does this order rank among that user's orders?" while still showing every individual order.

The common ones:

- `ROW_NUMBER()` — unique sequential numbers within each partition.
- `RANK()` / `DENSE_RANK()` — same, but handle ties. `RANK()` leaves gaps after a tie, `DENSE_RANK()` doesn't.
- `LAG()` / `LEAD()` — let a row see the previous or next row's value in its partition, without a self-join.

**Example**

```sql
-- top 3 orders per user by total_cents
SELECT * FROM (
  SELECT id, user_id, total_cents,
         ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total_cents DESC) AS rnk
  FROM orders
) ranked
WHERE rnk <= 3;

-- running total of daily revenue
SELECT order_date, daily_total,
       SUM(daily_total) OVER (ORDER BY order_date) AS running_total
FROM (
  SELECT created_at::date AS order_date, SUM(total_cents) AS daily_total
  FROM orders
  GROUP BY created_at::date
) daily;

-- compare each order to that customer's previous order
SELECT id, user_id, created_at, total_cents,
       LAG(total_cents) OVER (PARTITION BY user_id ORDER BY created_at) AS previous_order_total
FROM orders;
```

### 133. What is a materialized view, and when would you use one instead of just caching the result in Redis?

**Short Answer**

A materialized view stores a query's result physically, like a table. Reads are fast, but the data is stale until you `REFRESH` it.

Use one instead of Redis when other queries need to **filter, join, or index** the cached result with SQL. Use Redis when you just need a fast key-value lookup.

**Simple Explanation**

A regular view is just a saved query that re-runs every time. A materialized view actually runs the query once and stores the rows on disk, so reading it is as cheap as reading any table — until the underlying data changes.

`REFRESH MATERIALIZED VIEW CONCURRENTLY` updates it without locking out readers, but it requires a unique index on the view.

Reach for it when the cached thing is relational and more than one consumer needs to query it. Reach for Redis when it's a single precomputed value looked up by key.

**Example**

```sql
CREATE MATERIALIZED VIEW user_order_stats AS
SELECT user_id, COUNT(*) AS order_count, SUM(total_cents) AS lifetime_value
FROM orders
GROUP BY user_id;

CREATE UNIQUE INDEX ON user_order_stats (user_id);

-- fast reads, but only as fresh as the last refresh
SELECT * FROM user_order_stats WHERE lifetime_value > 100000;

-- CONCURRENTLY avoids locking out readers while refreshing (needs the unique index above)
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_stats;
```

### 134. What's the difference between DROP, DELETE, and TRUNCATE?

**Short Answer**

- `DELETE` — removes rows one at a time, can be filtered with `WHERE`, fires row-level triggers, and leaves dead rows for `VACUUM`.
- `TRUNCATE` — empties the whole table instantly by deallocating its pages. No `WHERE`, no row-level triggers. Far faster on big tables.
- `DROP` — removes the table itself, along with its data, indexes, and constraints.

**Simple Explanation**

`DELETE FROM orders WHERE ...` visits each matching row, fires any row-level triggers, and leaves dead row versions behind like any other MVCC write.

`TRUNCATE TABLE orders` skips all of that and deallocates the table's storage directly — dramatically faster on a huge table. But it can only remove **everything**, and it doesn't reset identity/serial counters unless you say `RESTART IDENTITY` (the default is `CONTINUE IDENTITY`).

`DROP TABLE` removes the table definition entirely.

A nice Postgres detail: unlike some databases, **all three are transactional**. You can `TRUNCATE` or `DROP` inside a `BEGIN` and `ROLLBACK` it cleanly.

**Example**

```sql
-- DELETE: DML, row-by-row, filterable, fires row-level triggers, leaves bloat for VACUUM
DELETE FROM orders WHERE status = 'cancelled' AND created_at < NOW() - INTERVAL '1 year';

-- TRUNCATE: deallocates all pages at once, skips row-level triggers, keeps the table
-- definition; identity counters only reset if you explicitly ask
TRUNCATE TABLE audit_logs RESTART IDENTITY;

-- DROP: removes the table (and its data, indexes, constraints) entirely
DROP TABLE audit_logs;

-- all three are transactional in Postgres -- this rolls back cleanly
BEGIN;
TRUNCATE TABLE audit_logs;
ROLLBACK;  -- audit_logs still has every row
```

### 135. What are efficient bulk write patterns for loading a lot of data?

**Short Answer**

Batch many rows into a **single multi-row `INSERT`** instead of one statement per row. For very large loads, use `COPY` — the fastest way to get data into Postgres.

**Simple Explanation**

One `INSERT` per row means one full network round trip per row, which is brutal for thousands of rows.

Putting many rows into one `INSERT ... VALUES (...), (...), (...)` cuts that overhead dramatically.

For genuinely large loads (hundreds of thousands of rows or more), `COPY` skips most of the per-row planning and execution overhead entirely.

In Rails, `Model.insert_all([...])` generates the multi-row INSERT — but remember it skips validations and callbacks.

**Example**

```sql
-- slow: one round trip per row
INSERT INTO line_items (order_id, product_id, quantity, price_cents) VALUES (1, 501, 2, 1999);
INSERT INTO line_items (order_id, product_id, quantity, price_cents) VALUES (1, 502, 1, 4999);

-- better: one multi-row INSERT
INSERT INTO line_items (order_id, product_id, quantity, price_cents)
VALUES
  (1, 501, 2, 1999),
  (1, 502, 1, 4999),
  (1, 503, 3, 799);

-- fastest for large loads: COPY streams rows in with minimal per-row overhead
COPY line_items (order_id, product_id, quantity, price_cents)
FROM '/tmp/line_items.csv'
WITH (FORMAT csv, HEADER true);
```

In Rails, `Model.insert_all([...])` generates the multi-row `INSERT` pattern above, skipping validations and callbacks for speed.

### 136. What does INSERT ... ON CONFLICT DO UPDATE (upsert) do?

**Short Answer**

An upsert inserts a row, or updates it in place if a row with the same unique key already exists — atomically, in one statement.

That avoids the race condition you get from "check if it exists, then insert or update".

**Simple Explanation**

The naive pattern — `SELECT` to check, then `INSERT` or `UPDATE` — has a gap. Two concurrent requests can both see "not found" and both try to insert, and one fails on the unique constraint.

`ON CONFLICT` handles the check and the action as one atomic operation inside the database.

Inside the `DO UPDATE` clause, `EXCLUDED` refers to the row values that *would have been* inserted, so you can combine old and new values.

**Example**

```sql
CREATE UNIQUE INDEX idx_inventory_product_id ON inventory (product_id);

INSERT INTO inventory (product_id, quantity)
VALUES (42, 10)
ON CONFLICT (product_id)
DO UPDATE SET quantity = inventory.quantity + EXCLUDED.quantity;
```

**— Schema Design, Scaling & Production Ops —**

### 137. What is database normalization, and when would you deliberately denormalize?

**Short Answer**

**Normalization** splits data into related tables so each fact is stored once. That prevents inconsistency.

**Denormalization** deliberately duplicates data to avoid expensive joins on reads. That's faster to read, but now you have to keep the copies in sync.

**Simple Explanation**

A fully normalized schema stores an order's total only as its line items — you derive it by summing them. That guarantees consistency (there's only one place to be wrong), but every read that needs the total has to join and aggregate.

Denormalizing — storing `total_cents` directly on `orders` — makes reads cheap. But now every code path that touches line items also has to update the order's total, which is exactly the kind of inconsistency normalization exists to prevent.

It's a legitimate trade-off for hot read paths. Just make it deliberately and narrowly, not by default.

**Example**

```sql
-- normalized: order total is always computed from line_items, single source of truth
SELECT o.id, SUM(li.quantity * li.price_cents) AS total_cents
FROM orders o
JOIN line_items li ON li.order_id = o.id
GROUP BY o.id;

-- denormalized: store total_cents directly on orders to avoid this join/aggregate
-- on every read -- but now every line_item change must also update orders.total_cents
ALTER TABLE orders ADD COLUMN total_cents integer;

UPDATE orders o
SET total_cents = (
  SELECT SUM(li.quantity * li.price_cents) FROM line_items li WHERE li.order_id = o.id
);
```

### 138. What's the difference between horizontal and vertical partitioning/sharding, and when does a single DB instance stop being enough?

**Short Answer**

- **Vertical partitioning** — split by columns or by feature area into separate tables or databases.
- **Horizontal partitioning / sharding** — split a table's **rows** across multiple partitions or servers using a key.

You need either once a single instance can't handle the write throughput or data volume, even after indexing, query tuning, and read replicas.

**Simple Explanation**

Vertical partitioning might mean moving rarely-used large columns into their own table, or splitting your app into services that each own their database.

Horizontal partitioning keeps one logical table but splits its rows — either within one Postgres instance (native declarative partitioning, still one server) or across separate instances (true sharding, which is a distributed systems problem).

With native partitioning, a query filtered on the partition key only scans the relevant partition. That's called partition pruning.

Worth being honest here: most apps never outgrow a well-indexed single instance with read replicas. Sharding is a big step, and you take it when writes or storage genuinely exceed one primary.

**Example**

```sql
-- horizontal partitioning of orders by month, within a single Postgres instance
CREATE TABLE orders (
  id bigint NOT NULL,
  user_id bigint NOT NULL,
  created_at timestamptz NOT NULL,
  total_cents integer
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_08 PARTITION OF orders
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
CREATE TABLE orders_2026_09 PARTITION OF orders
  FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');

-- a query filtered on created_at only scans the relevant partition ("partition pruning")
EXPLAIN ANALYZE SELECT * FROM orders WHERE created_at >= '2026-09-01';
```

True sharding takes this further, distributing those partitions across *separate* Postgres instances (often keyed by `tenant_id`/`account_id`) — an operational and application-routing problem, not just a schema feature.

### 139. How do you choose a shard key, and what happens if you choose badly?

**Short Answer**

Pick a shard key that (a) matches how you actually query and (b) spreads load evenly. Usually something like `user_id` or `account_id`.

Bad choices cause a **hot shard**:

- Low cardinality (like `status`) piles everything onto a few shards.
- Sequential keys (like an auto-increment `id`) send all new writes to the newest shard.

And changing the key later means migrating all your data.

**Simple Explanation**

Two failure modes to watch for.

**Low cardinality:** sharding by `status`, which might have four values, piles everything onto four shards no matter how many you provision.

**Sequential keys:** sharding by an ever-increasing `id` or `created_at` means *all* of today's writes land on the newest shard while older ones sit idle.

A good key is high-cardinality, evenly spread, and matches your dominant query. Most queries are "this user's data", so `user_id` usually means a query hits exactly one shard.

The trade-off: any query that **can't** include the shard key — "find all orders with this coupon across every customer" — has to hit every shard and merge results in the app.

And since the key decides where every row physically lives, changing it later isn't a config change. It's a full re-partitioning of your entire dataset.

**Example**

```sql
-- BAD shard key: status has only ~4 distinct values (pending/paid/shipped/refunded) --
-- almost every row lands on one of 4 shards regardless of how many shards you have
-- shard = hash('pending') % num_shards     -- massive hotspot

-- BAD shard key: sequential id / created_at means all "recent" writes hit one shard
-- shard = id % num_shards                  -- today's traffic all lands on one shard

-- BETTER: shard by user_id (or account_id/tenant_id) -- high cardinality, evenly
-- distributed, and matches the dominant query pattern ("this user's orders")
-- shard = hash(user_id) % num_shards

-- the cost: a query that can't include user_id, e.g. "find all orders with a given
-- coupon code across every customer," must fan out to every shard and merge results
```

### 140. What is connection pooling, and why does it matter at scale?

**Short Answer**

Each Postgres connection is a real OS process with real memory cost, and `max_connections` is a hard limit. A connection pool keeps a fixed set of connections open and shares them between workers instead of opening a new one per request.

**Simple Explanation**

Postgres uses one process per connection, so thousands of idle-but-open connections cost real memory even when doing nothing. Go past `max_connections` and new connections are simply refused.

In Rails, the `pool:` setting in `database.yml` needs to be at least as large as your web server's thread count, or requests queue waiting for a connection and eventually raise `ActiveRecord::ConnectionTimeoutError`.

At larger scale, with many app processes and containers, you put PgBouncer in front of Postgres in transaction-pooling mode, so hundreds of app-side connections share a much smaller number of real Postgres backends.

**Example**

```sql
-- see how many connections are actually open and what they're doing
SELECT pid, usename, state, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE datname = 'myapp_production'
ORDER BY duration DESC;

-- max_connections is a hard ceiling -- exceeding it means new connections are refused
SHOW max_connections;
```

### 141. Where does Postgres land on the CAP theorem, and why does it matter for your app's design?

**Short Answer**

A single Postgres instance is effectively CA — there's only one node, so there's no network partition to tolerate.

Once you add replicas, you're choosing:

- **Async replication** → stays available, but replicas can lag (AP-leaning).
- **Sync replication** → stays consistent, but the primary blocks if the replica is unreachable (CP-leaning).

**Simple Explanation**

CAP (Consistency, Availability, Partition tolerance) is really about distributed systems, so it doesn't apply cleanly to one Postgres server.

It becomes relevant when you add replicas.

With the **default async replication**, the primary commits and returns immediately without waiting for replicas. It stays available even if a replica is unreachable — but that replica can lag and serve stale reads.

With **sync replication**, the primary waits for a replica to confirm the write before `COMMIT` returns. Reads from that replica are guaranteed fresh, but the primary can't accept writes if the replica is down.

This drives real app design: tolerate slightly stale reads and use async replicas to scale reads cheaply; or, if you need read-your-writes correctness (a ledger, say), read from the primary or pay the sync latency cost.

**Example**

```sql
-- asynchronous replication (default): primary commits and returns immediately,
-- replicas catch up "eventually" -- available, but can lag (AP-leaning)

-- synchronous replication: primary waits for a named replica to confirm the WAL
-- write before COMMIT returns -- consistent, but primary blocks if that replica
-- is unreachable (CP-leaning)
ALTER SYSTEM SET synchronous_standby_names = 'replica_1';
ALTER SYSTEM SET synchronous_commit = 'on';
```

### 142. What do read replicas solve, and what do they not solve? What is read-after-write consistency?

**Short Answer**

Read replicas scale **reads** by taking `SELECT` traffic off the primary. They don't help with writes, and they lag behind the primary.

**Read-after-write consistency** is the problem where a user writes data, then immediately reads from a lagging replica and their own change appears to be missing.

**Simple Explanation**

Replicas stream the primary's WAL (write-ahead log) and apply it. That takes a nonzero amount of time — usually milliseconds, sometimes seconds under load.

So a very common pattern breaks: you create a record on the primary, redirect to the show page, and that read hits a replica that hasn't caught up yet. The row appears not to exist.

Common fixes:

- Route reads that immediately follow a write to the primary (Rails' `connected_to(role: :writing)`).
- Keep a short "stick to primary" window after any write for that user.
- Read the value from a cache you populated at write time instead of re-querying.

**Example**

```sql
-- on the primary
INSERT INTO orders (user_id, status, total_cents) VALUES (12345, 'pending', 4999);
-- app immediately redirects to an "order confirmation" page, which reads from a
-- replica that hasn't received this row's WAL yet:
SELECT * FROM orders WHERE user_id = 12345 ORDER BY created_at DESC LIMIT 1;
-- returns the user's PREVIOUS order, or nothing -- the new order appears to vanish

-- check current replication lag on a replica
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

### 143. What's the difference between a backup, a snapshot, and PITR? What are RPO and RTO?

**Short Answer**

- **Snapshot / backup** — restores you to one specific moment (last night, say).
- **PITR (Point-In-Time Recovery)** — a base backup plus continuously archived WAL, so you can restore to *any* moment.
- **RPO** — how much data you can afford to lose.
- **RTO** — how long you can afford to be down.

**Simple Explanation**

A nightly snapshot only gets you back to last night. Everything written since then is gone if you restore from it.

PITR combines a base backup with a continuous stream of WAL files, so you can replay changes forward to any timestamp — like one second before someone ran a bad `DELETE`.

**RPO (Recovery Point Objective)** is your acceptable data loss. Nightly-snapshots-only means an RPO of up to 24 hours. PITR with continuous WAL archiving brings it down to seconds.

**RTO (Recovery Time Objective)** is how long you can be down while restoring. That depends on data volume and how long the restore plus WAL replay actually takes.

Which is why you should **test restores regularly**. An untested backup is a hope, not a recovery plan.

**Example**

```sql
-- base backup, taken nightly, say
-- pg_basebackup -D /backups/base -Ft -z -P

-- WAL archiving must run continuously for PITR to work
-- postgresql.conf: archive_mode = on
-- postgresql.conf: archive_command = 'cp %p /wal_archive/%f'

-- disaster: at 14:32 someone runs this in production
DELETE FROM orders WHERE created_at < '2026-01-01';  -- forgot "AND status = 'test'"
-- wipes millions of real rows

-- recovery: restore the base backup, then replay WAL up to one second before
-- the bad statement, instead of losing everything since the last nightly snapshot
-- recovery_target_time = '2026-09-22 14:31:59'
```

**— PostgreSQL-Specific Data Types —**

### 144. How do you query JSON/JSONB columns, and how do you index them?

**Short Answer**

- `->` gets a value back as JSON(B)
- `->>` gets it back as **text** (usually what you want for comparisons)
- `@>` checks containment ("does this document contain that?")

Use `JSONB`, not `JSON` — it's stored parsed, it's faster to query, and it's the only one you can index.

**Simple Explanation**

`JSON` stores an exact text copy of what you inserted and has to re-parse it every time you read it.

`JSONB` stores a parsed binary form. Slightly slower to write, much faster to query, and indexable.

The operators:

- `metadata -> 'coupon'` → returns JSONB
- `metadata ->> 'coupon'` → returns text
- `metadata @> '{"gift_wrapped": true}'` → true if the left document contains the right one

A GIN index on the JSONB column is what makes those containment queries fast.

**Example**

```sql
ALTER TABLE orders ADD COLUMN metadata jsonb;

UPDATE orders SET metadata = '{"gift_wrapped": true, "coupon": "SAVE10", "notes": {"delivery": "leave at door"}}'
WHERE id = 1;

-- -> returns jsonb, ->> returns text
SELECT metadata -> 'coupon' AS coupon_jsonb,          -- "SAVE10" (jsonb)
       metadata ->> 'coupon' AS coupon_text,          -- SAVE10 (text)
       metadata -> 'notes' ->> 'delivery' AS delivery_note
FROM orders WHERE id = 1;

-- containment query: find orders with gift_wrapped = true
CREATE INDEX idx_orders_metadata ON orders USING GIN (metadata);
SELECT * FROM orders WHERE metadata @> '{"gift_wrapped": true}';
```

### 145. When is a Postgres array column a reasonable fit, and when do you actually want a join table?

**Short Answer**

An array column is fine for a small list of simple values that belongs entirely to one row and isn't queried relationally — free-form tags, for example.

Use a **join table** as soon as you need foreign key integrity, per-relationship data (like quantity or price), or efficient joins.

**Simple Explanation**

An array is convenient when the values are simple, don't need their own identity, and you don't need to enforce that they reference another table.

The moment you need foreign key integrity, extra columns about the relationship, or joins and aggregates across the related values, an array is the wrong tool. At that point you've reinvented a join table without the indexing or the constraints.

A quick test: would you ever want to say "this order line has quantity 3 and price ₹499"? If yes, you need a join table, not an array of IDs.

**Example**

```sql
-- reasonable: a handful of free-form tags that only ever belong to this order
ALTER TABLE orders ADD COLUMN tags text[];
UPDATE orders SET tags = ARRAY['gift', 'expedited'] WHERE id = 1;
SELECT * FROM orders WHERE tags @> ARRAY['gift'];

-- NOT reasonable: modeling order <-> product as an array of product_ids
-- ALTER TABLE orders ADD COLUMN product_ids bigint[];  -- avoid this
-- you lose foreign key integrity, per-line quantity/price, and easy joins --
-- exactly what a join table is for:
CREATE TABLE line_items (
  id bigserial PRIMARY KEY,
  order_id bigint NOT NULL REFERENCES orders (id),
  product_id bigint NOT NULL REFERENCES products (id),
  quantity integer NOT NULL,
  price_cents integer NOT NULL
);
```

### 146. UUID primary keys vs serial/bigint — what are the trade-offs?

**Short Answer**

- **bigint / serial** — smaller (8 bytes), sequential, great for index performance. But the values are guessable and leak how many records you have.
- **UUID** — unguessable, can be generated on the client, safe to merge across databases. But random values scatter index inserts, which hurts write performance.

**Simple Explanation**

An ever-increasing `bigint` always inserts at the "end" of its index, which is cheap and cache-friendly. The downside: `/orders/1042` tells anyone roughly how many orders you've processed, and IDs are trivially guessable.

A random UUID (v4) is unguessable and can be created offline — a mobile client can generate one before it ever syncs — and two independently-seeded databases can be merged without collisions. The downside is that random values scatter inserts across the whole index, causing more page splits and cache misses under heavy write load.

A popular middle ground is **UUIDv7** — time-ordered UUIDs that are still unguessable but roughly sequential, so you get back most of the insert performance.

Note: `gen_random_uuid()` is built into Postgres from version 13. On older versions it needed the `pgcrypto` extension.

**Example**

```sql
-- bigint: compact, sequential, great index locality, but leaks growth rate/is guessable
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  user_id bigint NOT NULL
);

-- UUID: unguessable, generatable offline, safely mergeable across databases,
-- but scatters inserts across the whole index
CREATE TABLE orders (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id bigint NOT NULL
);
-- gen_random_uuid() has been built into Postgres since version 13 (no extension
-- needed); on older versions it required the pgcrypto extension
```

### 147. What are the basics of full-text search in Postgres, and when would you use it instead of a dedicated search engine like Elasticsearch?

**Short Answer**

- `to_tsvector` turns text into normalized searchable words.
- `to_tsquery` / `plainto_tsquery` turns a search string into a query.
- `@@` matches them, and `ts_rank` scores the result.

Use Postgres full-text search when search is a secondary feature. Move to Elasticsearch when you need typo tolerance, faceting, or serious relevance tuning at scale.

**Simple Explanation**

A `tsvector` is your text reduced to normalized word stems ("running" becomes "run") with common stop words removed. A `tsquery` is a search expressed in that same normalized form.

`@@` checks whether the query matches the document, and `ts_rank` scores how well.

That's plenty for "search my products" or "search my articles" in an app already running on Postgres — no new infrastructure, and the index is transactionally consistent with your data.

You'd move to a dedicated engine like Elasticsearch once you need typo tolerance, heavy relevance tuning, faceted filtering, or when search volume starts competing with your transactional workload for resources.

**Example**

```sql
ALTER TABLE products ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', name || ' ' || coalesce(description, ''))) STORED;

CREATE INDEX idx_products_search ON products USING GIN (search_vector);

SELECT id, name, ts_rank(search_vector, query) AS rank
FROM products, plainto_tsquery('english', 'wireless bluetooth headphones') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

## Redis

**— Fundamentals —**

### 148. What is Redis, and why is it used alongside a primary database instead of replacing it?

**Short Answer**

Redis is an in-memory key-value store used for caching, sessions, job queues, rate limiting, and pub/sub.

It doesn't replace Postgres because it trades durability and rich querying for raw speed.

**Simple Explanation**

Redis keeps its data in RAM, which makes it extremely fast for simple lookups. But that means your dataset is limited by available memory, and without careful persistence settings you're more exposed to data loss on a crash than with a disk-backed, WAL-logged database.

It also has no joins, no complex querying, and no schema.

So the usual pattern is: **Postgres stays the source of truth**, and Redis sits beside it to take load off expensive reads, hold throwaway data (sessions, cache, rate-limit counters), or move background jobs around.

**Example**

```ruby
# Rails.cache is commonly backed by Redis in production
Rails.application.config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"] }

# cache-aside read: check Redis first, fall back to Postgres, populate Redis
def expensive_dashboard_stats(user)
  Rails.cache.fetch("dashboard_stats/#{user.id}", expires_in: 5.minutes) do
    user.orders.sum(:total_cents)  # only hits Postgres on a cache miss
  end
end
```

### 149. What are Redis's core data structures, and what's a real use case for each?

**Short Answer**

- **String** — a single value: a cached fragment, a token, or a counter you `INCR`.
- **Hash** — an object with named fields, so you can read one field without decoding everything.
- **List** — an ordered sequence. Good as a simple queue.
- **Set** — unique values, unordered. Good for "has this happened?" and counting unique things.
- **Sorted set** — unique values each with a score, kept in order. Perfect for leaderboards.

**Simple Explanation**

The point is to pick the structure that matches how you'll read the data, rather than dumping JSON into a string every time.

If you only ever read the whole thing, a string is fine. If you regularly read or update one field, a hash saves you encoding and decoding the whole object.

A list gives you push/pop from either end, which is exactly what a job queue needs.

A set automatically rejects duplicates, so counting unique visitors is just "add everyone, then ask for the size".

A sorted set keeps everything ranked by a score, so "top 10" is a single command.

**Example**

```ruby
redis = Redis.new

# STRING -- a cached value, or a counter
redis.set("user:123:session_token", "abc123", ex: 3600)
redis.incr("orders:today:count")

# HASH -- an object's fields, without round-tripping full JSON encode/decode
redis.hset("user:123", "email", "jane@example.com", "plan", "pro")
redis.hget("user:123", "plan")

# LIST -- a simple FIFO/LIFO queue, or a bounded activity feed
redis.lpush("notifications:123", "Your order shipped")
redis.rpop("notifications:123")

# SET -- unique membership tracking, e.g. unique visitors to a page today
redis.sadd("page:home:visitors:2026-09-22", "user:123")
redis.scard("page:home:visitors:2026-09-22")  # unique visitor count

# SORTED SET -- a leaderboard ranked by score
redis.zadd("leaderboard", 4820, "user:123")
redis.zrevrange("leaderboard", 0, 9, with_scores: true)  # top 10
```

### 150. How does Sidekiq use Redis as a job broker?

**Short Answer**

Sidekiq pushes serialized jobs onto Redis **lists** (one per queue) and uses **sorted sets** for scheduled and retrying jobs. Worker processes pop jobs off those lists.

Redis being fast at list and sorted-set operations is what makes Sidekiq's throughput possible.

**Simple Explanation**

`SomeJob.perform_async(args)` turns the job class and arguments into JSON and pushes it onto a Redis list named after the queue.

Idle Sidekiq processes use a blocking pop (`BRPOP`), so a job becomes available to a worker almost instantly.

Scheduled jobs and retries live in sorted sets scored by the timestamp they're due. A scheduler thread periodically moves due jobs back onto the active queue.

One honest caveat: Redis persistence is best-effort compared to Postgres. In rare failure cases a queued job can be lost if Redis crashes before flushing to disk. That's why jobs should be designed to be safely retried rather than assumed to run exactly once.

**Example**

```ruby
class ChargeOrderJob
  include Sidekiq::Job
  sidekiq_options queue: :billing, retry: 5

  def perform(order_id)
    order = Order.find(order_id)
    PaymentProcessor.charge!(order)
  end
end

ChargeOrderJob.perform_async(order.id)
# under the hood: LPUSH queue:billing '{"class":"ChargeOrderJob","args":[42],...}'
# a Sidekiq process runs: BRPOP queue:billing 2
# scheduled/retry jobs live in sorted sets, scored by the time they're due to run
```

**— Expiry & Caching Patterns —**

### 151. How do you use TTL/EXPIRE, and what different problems does it solve?

**Short Answer**

`EXPIRE` (or `SET ... EX`) attaches a time-to-live to a key, so Redis deletes it automatically after N seconds.

That one mechanism powers three things: caches (stale data removes itself), sessions (idle sessions time out), and rate limiters (a counter resets itself).

**Simple Explanation**

Because every key can carry its own expiry, you don't need a separate cleanup process.

- **Cache:** the TTL bounds how stale data can get without any manual invalidation.
- **Session:** refreshing the TTL on every request gives you a sliding "log out after 30 minutes of inactivity" behavior.
- **Rate limiter:** a counter key that expires when the window closes resets itself with no extra bookkeeping.

`TTL key` tells you how many seconds are left, or `-1` if no expiry is set.

**Example**

```ruby
redis = Redis.new

# cache: expires after 10 minutes
redis.set("product:42:price_cents", "4999", ex: 600)
redis.ttl("product:42:price_cents")  # seconds remaining, or -1 if no expiry is set

# session store: sliding expiration, refreshed on every request
redis.set("session:#{session_id}", session_data.to_json, ex: 30.minutes.to_i)

# rate limiter: a per-minute counter that resets itself
key = "rate_limit:user:123:#{Time.now.strftime('%Y%m%d%H%M')}"
redis.multi do |m|
  m.incr(key)
  m.expire(key, 60)
end
```

### 152. Cache-aside vs write-through vs write-behind — which does a typical Rails app use?

**Short Answer**

- **Cache-aside** — check the cache, fall back to the DB on a miss, then store the result. This is what `Rails.cache.fetch` does.
- **Write-through** — write to the cache and the DB together on every write.
- **Write-behind** — write to the cache immediately and to the DB asynchronously.

Rails apps almost always use cache-aside.

**Simple Explanation**

**Cache-aside** is lazy: nothing is cached until someone reads it. A miss just means going to the database and storing the result for next time. Simple, self-healing, and the default in Rails.

**Write-through** keeps the cache always warm, but every write path in your app has to remember to update it — easy to miss one.

**Write-behind** is faster for writes but riskier: if the process crashes before the deferred database write happens, you lose data. It's rare outside specialized systems.

**Example**

```ruby
# cache-aside via Rails.cache.fetch -- the idiomatic Rails pattern
def order_summary(order)
  Rails.cache.fetch(order.cache_key_with_version) do
    OrderSummarySerializer.new(order).as_json  # only runs on a cache miss
  end
end

# write-through would look like this: every write updates cache and DB together
def update_price!(product, new_price_cents)
  product.update!(price_cents: new_price_cents)
  Rails.cache.write("product:#{product.id}:price_cents", new_price_cents)
end
```

### 153. What is a cache stampede (thundering herd), and how do you mitigate it?

**Short Answer**

A cache stampede is when a popular cache key expires and hundreds of requests all miss at the same instant, so they all hammer the database rebuilding the same value.

Fixes: let one request rebuild while others serve slightly stale data, add random jitter to TTLs, or use a lock so only one rebuild runs.

**Simple Explanation**

If a hot key has a flat 5-minute TTL, then every 5 minutes there's a moment where it's gone and every concurrent request becomes a miss. They all recompute the identical value at once, which can take the database down.

Three mitigations:

1. **Grace window** — Rails' `race_condition_ttl` gives the first request that notices the expiry extra time to rebuild while everyone else keeps serving the old value.
2. **Jittered TTL** — add a random few seconds so thousands of keys written at the same moment (say, right after a deploy) don't all expire together.
3. **Explicit lock** — for a rebuild too expensive to risk running twice, have one process take a lock and rebuild while others read the stale value.

**Example**

```ruby
def homepage_stats
  Rails.cache.fetch("homepage_stats", expires_in: 5.minutes, race_condition_ttl: 10.seconds) do
    # race_condition_ttl: the first thread that sees an expired value gets an
    # extended window to rebuild it, while other threads keep serving the
    # slightly-stale old value instead of all recomputing at once
    ExpensiveStatsCalculator.call
  end
end

# jittered TTL: don't let thousands of keys set at the same deploy all expire together
Rails.cache.write("homepage_stats", data, expires_in: 5.minutes + rand(60).seconds)

# explicit lock, for a rebuild too expensive to risk running twice concurrently
def rebuild_with_lock(key)
  lock_key = "lock:#{key}"
  if Rails.cache.write(lock_key, true, unless_exist: true, expires_in: 30.seconds)
    begin
      value = yield
      Rails.cache.write(key, value, expires_in: 5.minutes)
      value
    ensure
      Rails.cache.delete(lock_key)
    end
  else
    Rails.cache.read(key)  # someone else is already rebuilding -- serve stale
  end
end
```

### 154. What's the cache invalidation strategy in a typical Rails app using Redis?

**Short Answer**

Mostly **key-based expiration**, not manual deletes. Rails puts the record's version into the cache key itself (`cache_key_with_version`), so when the record changes the key changes, and the old entry is simply never looked up again.

**Simple Explanation**

Instead of "update the record, then remember to delete its cache entry", Rails bakes the record's `updated_at` into the key — something like `"orders/42-20260922103015123456"`.

When the order is updated, the key changes, so the next `fetch` misses and rebuilds. Nobody has to remember a delete call, and there's no window where a stale key lingers because someone forgot.

Russian-doll nesting means a parent's key should also change when a child changes — that's what `touch: true` on the child's `belongs_to` is for.

Manual `Rails.cache.delete` is the exception, used when the dependency can't be expressed in a key — like an aggregate that spans many unrelated records.

**Example**

```ruby
# cache_key_with_version bakes in the record's updated_at (or explicit lock_version)
order.cache_key_with_version  # => "orders/42-20260922103015123456"

Rails.cache.fetch(["order_summary", order.cache_key_with_version]) do
  OrderSummarySerializer.new(order).as_json
end
# when order is updated, cache_key_with_version changes, so this fetch call
# automatically misses and rebuilds -- nobody has to remember to delete the old key

# Russian-doll nesting: a parent's cache key should depend on its children too
class LineItem < ApplicationRecord
  belongs_to :order, touch: true  # updating a line_item invalidates its parent order's cache
end

# manual invalidation is the exception -- used when the dependency can't be
# expressed as part of the key (e.g. an aggregate across many unrelated records)
Rails.cache.delete("dashboard_stats/#{user.id}")
```

### 155. How would you implement rate limiting with Redis?

**Short Answer**

Use `INCR` on a key that includes the user and the time window, and `EXPIRE` so it resets itself. That's a **fixed-window** limiter — cheap and simple.

A **sliding window** (a sorted set of timestamps) is more accurate but costs more work per request.

**Simple Explanation**

**Fixed window:** the key includes the current minute, `INCR` bumps the count, and the TTL makes the key disappear when the window passes. Very cheap.

Its weakness is the boundary: a client could send 100 requests at 0:59 and another 100 at 1:00 — 200 requests in two seconds, without ever technically exceeding "100 per window".

**Sliding window:** store each request's timestamp in a sorted set, remove anything older than the window on each check, and count what's left. Much more accurate, at the cost of more Redis work per request.

**Example**

```ruby
class RateLimiter
  def initialize(redis, limit:, window_seconds:)
    @redis = redis
    @limit = limit
    @window_seconds = window_seconds
  end

  # fixed-window counter
  def allow?(identifier)
    key = "rate_limit:#{identifier}:#{Time.now.to_i / @window_seconds}"
    count = @redis.incr(key)
    @redis.expire(key, @window_seconds) if count == 1  # only set TTL on the first hit
    count <= @limit
  end
end

limiter = RateLimiter.new(Redis.new, limit: 100, window_seconds: 60)
raise TooManyRequests unless limiter.allow?("user:123")

# sliding-window variant using a sorted set: keep only timestamps within the window
def allow_sliding?(identifier)
  key = "rate_limit_sliding:#{identifier}"
  now = Time.now.to_f
  @redis.multi do |m|
    m.zadd(key, now, now)
    m.zremrangebyscore(key, 0, now - @window_seconds)
    m.expire(key, @window_seconds)
  end
  @redis.zcard(key) <= @limit
end
```

**— Coordination & Messaging —**

### 156. How does Redis Pub/Sub differ from a durable queue?

**Short Answer**

Pub/Sub broadcasts a message to whoever is connected **right now**, then forgets it. There's no storage and no redelivery.

A durable queue keeps the message until a worker actually consumes it.

**Simple Explanation**

`PUBLISH` fires a message at all currently-subscribed clients and Redis discards it immediately. It's never stored.

So if your subscriber was down, restarting, or briefly disconnected, that message is gone forever — there's nothing to catch up on.

Compare that with a Sidekiq queue, where a job sits in the list until a worker pops it. A temporarily offline worker just processes it late instead of losing it.

Pub/Sub is right for ephemeral, best-effort broadcasts — like relaying a WebSocket message between app processes, which is exactly what ActionCable's Redis adapter does. It's wrong for anything that must eventually be processed.

**Example**

```ruby
# publisher
redis.publish("order_updates", { order_id: 42, status: "shipped" }.to_json)

# subscriber -- must already be connected and listening to receive this
redis.subscribe("order_updates") do |on|
  on.message do |channel, message|
    OrderChannel.broadcast(JSON.parse(message))
  end
end
# if this subscriber process was down/restarting when publish happened, that
# message is gone forever -- there's no queue to catch up from
```

This is essentially what ActionCable's Redis adapter uses to broadcast messages between separate app server processes.

### 157. How do you use Redis as a distributed lock, and what's the known risk?

**Short Answer**

`SET key value NX PX ttl` sets a key **only if it doesn't exist**, with an expiry. That gives you a lock so two workers don't process the same thing at once.

The known risk: the lock can expire while the first worker is still running, letting a second worker acquire it and both run at the same time.

**Simple Explanation**

`NX` means "only if not exists", and it makes acquiring the lock atomic — there's no gap where two processes both see "no lock" and both set it.

`PX` sets a millisecond TTL so a crashed holder doesn't lock the resource forever.

The danger: if the work takes longer than the TTL — a slow query, a GC pause, a stalled network call — the lock silently expires while the first worker is still going. A second worker can then acquire it, and now both believe they hold it exclusively.

Using a random token per acquisition and only deleting the lock if you still own it (via an atomic Lua check-and-delete) stops you deleting *someone else's* lock. But it doesn't fix the expiry race — that's a fundamental limit of TTL-based locks. Mitigate it by keeping the critical section short, or by extending the TTL with a watchdog.

**Example**

```ruby
def with_lock(key, ttl_ms: 10_000)
  token = SecureRandom.hex(16)
  acquired = redis.set(key, token, nx: true, px: ttl_ms)
  return false unless acquired

  begin
    yield
  ensure
    # only delete if we still own it -- do this atomically via a Lua script
    redis.eval(
      "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end",
      keys: [key], argv: [token]
    )
  end
end

with_lock("lock:order:#{order.id}:fulfillment") do
  FulfillOrderService.call(order)
end
# risk: if FulfillOrderService runs longer than ttl_ms, the lock expires mid-run
# and a second worker can acquire it and run fulfillment concurrently --
# mitigate with a watchdog that extends the TTL, or keep the critical section short
```

### 158. How do you get atomicity across multiple Redis commands — MULTI/EXEC vs Lua scripting?

**Short Answer**

- `MULTI`/`EXEC` — runs a batch of commands together with nothing interleaved, but **can't make decisions** based on an intermediate result.
- **Lua script** (`EVAL`) — runs atomically on the server and **can** read a value, decide, and write, all in one step.

That's why "delete this lock only if I still own it" needs Lua.

**Simple Explanation**

`MULTI`/`EXEC` is Redis's basic transaction. Commands queued between them all run back to back with nothing else in between.

Its limitation is that it's blind — you queue commands without being able to look at an earlier result and branch on it.

A Lua script runs as one indivisible unit on the server, so it *can* read, decide, and act. That's exactly what the safe lock release from the previous question needs.

**Example**

```ruby
# MULTI/EXEC: atomic, but can't branch on GET's result mid-transaction
redis.multi do |m|
  m.incr("orders:today:count")
  m.expire("orders:today:count", 86_400)
end

# Lua script: atomic AND conditional -- read, decide, and write in one
# indivisible server-side step
script = <<~LUA
  local current = tonumber(redis.call('get', KEYS[1]) or '0')
  if current < tonumber(ARGV[1]) then
    return redis.call('incr', KEYS[1])
  else
    return -1
  end
LUA
redis.eval(script, keys: ["inventory:42:reserved"], argv: [10])
```

**— Persistence & Operations —**

### 159. RDB vs AOF persistence — what data can you actually afford to lose?

**Short Answer**

- **RDB** — periodic snapshots of the whole dataset. Fast restarts, but you lose everything written since the last snapshot.
- **AOF** — logs every write, and can fsync as often as once per second. Much smaller loss window, but bigger files and slower restarts.

**Simple Explanation**

RDB is cheap and gives you one compact file, but it only captures state as of the last snapshot. A crash between snapshots loses everything since.

AOF logs each write as it happens. With `appendfsync everysec`, you lose at most about one second of writes on a hard crash, at the cost of a larger file that takes longer to replay on startup.

Many production setups enable both — RDB for fast full backups, AOF for a tighter loss window.

Rule of thumb: for a pure cache you can rebuild from Postgres, durability barely matters, so RDB (or nothing) is fine. For a Sidekiq queue you can't cheaply regenerate, AOF with `appendfsync everysec` is the safer default.

**Example**

```
# redis.conf -- RDB: snapshot if >= 100 keys changed in 60s, or >= 10000 in 1s
save 60 100
save 3600 1

# redis.conf -- AOF: log every write, fsync once per second (a good durability/perf balance)
appendonly yes
appendfsync everysec
```

For a pure, rebuildable-from-Postgres cache, durability barely matters — RDB, or even persistence off, is fine. For Sidekiq's job queue or anything you can't cheaply regenerate, AOF with `appendfsync everysec` is the safer default: accept up to roughly a second of lost writes on a crash, instead of minutes with RDB-only.

### 160. What happens when Redis runs out of memory, and how do eviction policies control it?

**Short Answer**

When Redis hits `maxmemory`, the `maxmemory-policy` decides what happens:

- `noeviction` — reject new writes.
- `allkeys-lru` — evict the least recently used key, whatever it is.
- `volatile-lru` / `volatile-ttl` — only evict keys that have a TTL set.

Picking the wrong one turns memory pressure into an outage.

**Simple Explanation**

`noeviction` starts rejecting writes once the limit is hit. Right when losing data (like unfinished job queues) would be worse than an error — but dangerous if you didn't plan for that failure.

`allkeys-lru` evicts anything, TTL or not. Fine for a pure cache where everything is disposable and rebuildable.

`volatile-*` only evicts keys with an explicit expiry. That matters a lot if one Redis instance holds both cache keys **and** Sidekiq queues: you never want a job list evicted to make room for a cache entry. Since queue keys have no TTL, `volatile-lru` leaves them alone.

Best practice if you can: run separate Redis instances (or at least separate databases) for cache and for jobs.

**Example**

```
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

```ruby
# a pure cache: fine to evict anything under memory pressure -- allkeys-lru,
# since every key is disposable and rebuildable from Postgres

# a mixed instance holding both cache keys AND Sidekiq queues/session data:
# you do NOT want Sidekiq's job lists evicted to make room for a cache key --
# either run separate Redis instances/DBs for cache vs jobs, or use volatile-lru
# (only evicts keys with an explicit TTL) so the un-expiring job-queue keys
# are never touched
```

### 161. How does Redis achieve high availability, and what does that involve operationally?

**Short Answer**

- **Redis Sentinel** — watches a primary/replica setup and automatically promotes a replica if the primary dies.
- **Redis Cluster** — shards data across multiple primaries, each with its own replicas, for scaling plus failover.

Either way, you trade some consistency during failover for availability.

**Simple Explanation**

Sentinel processes monitor the primary and each other. If they agree it's down, they promote a replica and point clients at the new one — no human needed.

Redis Cluster goes further by splitting the keyspace across multiple primaries, so you get horizontal scaling and per-shard failover.

The honest trade-off during failover: replication is asynchronous by default, so a write acknowledged by the old primary right before it died can be lost if it hadn't replicated yet. You're choosing "keep serving" over "never lose an acknowledged write".

For most Rails apps a managed Redis (ElastiCache, Redis Cloud) handles this for you. Understanding the mechanics still matters for reasoning about what happens to in-flight cache writes or locks during a failover.

**Example**

```ruby
# app-side: connect to Sentinel, which hands back the current primary's address,
# so failover doesn't require an app config change/deploy
Redis.new(
  sentinels: [{ host: "sentinel1", port: 26379 }, { host: "sentinel2", port: 26379 }],
  name: "mymaster"
)
```

For most Rails apps, a managed Redis (ElastiCache, Redis Cloud, etc.) handles this transparently — understanding the mechanics matters for reasoning about what happens to in-flight cache writes or locks during a failover event.

### 162. When would you NOT put something in Redis?

**Short Answer**

Don't use Redis for data that must survive a crash with zero loss, or for data too big to hold in memory affordably.

Anything representing money already moved, a legal record, or any fact you can't afford to lose belongs in Postgres.

**Simple Explanation**

Redis is fast because it keeps its working set in RAM, which makes it expensive to scale to very large datasets compared to disk-backed Postgres. And even with AOF, its durability guarantee is weaker than a WAL-backed relational database's `COMMIT`.

So: Postgres is the source of truth. Redis is the right place for a **cached, derived, rebuildable** view of that data — not the record of truth itself.

A ledger entry goes in Postgres inside a transaction. The cached balance you show on a dashboard can go in Redis.

**Example**

```ruby
# wrong: Redis as the source of truth for money already moved
redis.hincrby("ledger:account:1", "balance_cents", -5000)  # no ACID guarantee,
                                                              # no durable audit trail

# right: Postgres is the source of truth; Redis only caches a derived read
Account.transaction do
  account.decrement!(:balance_cents, 5000)
  LedgerEntry.create!(account: account, amount_cents: -5000)
end
Rails.cache.write("account:#{account.id}:balance_cents", account.balance_cents, expires_in: 1.minute)
```

### 163. When would you drop down to the raw Redis client instead of using Rails.cache?

**Short Answer**

`Rails.cache` only gives you cache-shaped operations — `fetch`, `read`, `write`, `delete`.

Use the raw Redis client when you need Redis-specific features: sorted sets, lists, pub/sub, atomic counters, Lua scripts, or distributed locks.

**Simple Explanation**

`Rails.cache` is deliberately limited so it can be swapped between Redis, Memcached, or an in-memory store without changing your code. But that abstraction only models "store a blob under a key, maybe with a TTL".

The moment you want a leaderboard (sorted set), a queue (list), an `INCR`-based rate limiter, `MULTI`/`EXEC` or Lua for atomicity, or a `SET ... NX` lock, you go to the Redis client directly — `Rails.cache` has no vocabulary for any of that.

**Example**

```ruby
# Rails.cache: fine for simple key -> value caching, storage-agnostic
Rails.cache.fetch("product:42:price_cents", expires_in: 10.minutes) { product.price_cents }

# raw Redis client: needed for anything Rails.cache doesn't model, e.g. a
# sorted-set leaderboard or an atomic counter used for rate limiting
redis = Redis.new(url: ENV["REDIS_URL"])
redis.zadd("leaderboard:2026-09", score, "user:#{user.id}")
redis.zrevrank("leaderboard:2026-09", "user:#{user.id}")  # this user's current rank
```

## JavaScript

**— Core Language Fundamentals —**

### 164. What is a closure, and how would you use one to build a private counter?

**Short Answer**

A closure is a function that still has access to the variables from where it was defined, even after that outer function has finished running.

That's how you get private state in JavaScript.

**Simple Explanation**

Normally a function's local variables are cleaned up once it returns. But if a function defined inside it still references those variables, they stay alive as long as that inner function is reachable.

The inner function keeps a **live reference**, not a copy.

So if you return an object of small functions that all touch the same `count` variable, `count` becomes private state: nothing outside can read or change it except through the functions you exposed. Each call to the outer function creates a fresh, independent set.

**Example**

```js
function makeCounter() {
  let count = 0; // private state — not accessible from outside makeCounter
  return {
    increment() { return ++count; },
    decrement() { return --count; },
    value() { return count; },
  };
}

const counterA = makeCounter();
const counterB = makeCounter();

counterA.increment();
counterA.increment();
counterB.increment();

console.log(counterA.value()); // 2
console.log(counterB.value()); // 1 — separate closure, separate `count`
```

### 165. How do `var`, `let`, and `const` differ in scoping and hoisting?

**Short Answer**

- `var` — function-scoped, and hoisted with a starting value of `undefined`.
- `let` / `const` — block-scoped (`{ }`), and unusable before their declaration line (the "temporal dead zone").
- `const` also stops you reassigning the variable.

**Simple Explanation**

"Hoisting" means the engine registers declarations at the top of their scope before running any code.

With `var`, that scope is the whole function, and reading it before the declaration line gives `undefined` instead of an error.

With `let` and `const`, the scope is the nearest `{ }` block, and reading it before the declaration throws a `ReferenceError` — which is usually what you want, because it catches a real mistake.

The classic interview trap is a loop with async callbacks. With `var`, all callbacks share **one** variable, so by the time they run it's already the final value. With `let`, each iteration gets its own binding, so each callback sees the value from its own loop pass.

Note `const` prevents **reassignment**, not mutation — you can still push to a `const` array.

**Example**

```js
// var: all three callbacks share ONE `i`, which is 3 by the time any of them run
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log('var i:', i), 0);
}
// logs: var i: 3 / var i: 3 / var i: 3

// let: each iteration gets its own binding of `j`
for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log('let j:', j), 0);
}
// logs: let j: 0 / let j: 1 / let j: 2
```

### 166. What's the difference between `==` and `===`?

**Short Answer**

- `===` compares value **and** type, with no conversion.
- `==` converts the operands to a common type first, which produces surprising results.

Use `===` everywhere, except the deliberate `== null` check.

**Simple Explanation**

Type coercion means JavaScript automatically converts one or both values before comparing. `==` does this: `1 == '1'` is `true` because the string gets converted to a number.

`===` skips all of that. Different types means immediately `false`.

Some surprises `==` produces: `0 == false` is true, `'' == false` is true, `null == undefined` is true.

The one place `==` is still idiomatic is `value == null`, which is true for **both** `null` and `undefined` — often exactly the check you want.

Also worth remembering: `NaN` is never equal to anything, including itself, with either operator.

**Example**

```js
console.log(1 == '1');          // true  — string coerced to number
console.log(1 === '1');         // false — different types, no coercion
console.log(0 == false);        // true
console.log('' == false);       // true
console.log(null == undefined); // true  — special-cased by the spec
console.log(null === undefined);// false
console.log(NaN == NaN);        // false — NaN is never equal to anything

function isMissing(value) {
  return value == null; // true for null OR undefined — the one legitimate == idiom
}
```

### 167. What's the difference between `null`, `undefined`, and `NaN`?

**Short Answer**

- `undefined` — a variable or property was never given a value. JavaScript sets this automatically.
- `null` — an intentional "empty", set by a developer.
- `NaN` — "Not a Number", the result of a failed numeric operation. It's the only value not equal to itself.

**Simple Explanation**

JavaScript uses `undefined` as its default "nothing here yet". You get it from an uninitialized variable, a missing object property, or a function with no `return`.

`null` is never assigned automatically — someone wrote it to mean "deliberately empty".

`NaN` shows up when a numeric operation can't produce a real number, like `Number('abc')`.

Two quirks worth knowing: `typeof null` returns `"object"` (a long-standing JavaScript bug kept for compatibility), and because `NaN === NaN` is `false`, you must use `Number.isNaN()` to test for it.

**Example**

```js
let a;
console.log(a); // undefined — declared, never assigned

let b = null;
console.log(b); // null — explicitly "no value", by intent

console.log(typeof undefined); // "undefined"
console.log(typeof null);      // "object" — a long-standing JS quirk, kept for compatibility

console.log(Number('abc'));     // NaN
console.log(NaN === NaN);       // false
console.log(Number.isNaN(NaN)); // true — the correct way to check
```

### 168. How does JavaScript's prototypal inheritance differ from classical inheritance?

**Short Answer**

In classical inheritance (Ruby, Java), a class is a blueprint that objects are stamped out from.

In JavaScript, objects **delegate** to other objects through a prototype chain. `class` syntax in modern JavaScript is just nicer syntax over that same mechanism.

**Simple Explanation**

Every JavaScript object has an internal link to another object — its prototype. When you access a property the object doesn't have, the engine walks up that chain looking for it.

So `Object.create(animal)` makes an object whose prototype is `animal`. Calling `dog.speak()` finds `speak` by walking up to `animal`.

`class Dog extends Animal` sets up exactly the same chain underneath — `Dog.prototype`'s prototype is `Animal.prototype`. It's the same lookup, just written more familiarly.

**Example**

```js
const animal = {
  speak() { return `${this.name} makes a sound.`; },
};

const dog = Object.create(animal); // dog's prototype is `animal`
dog.name = 'Rex';
dog.speak(); // "Rex makes a sound." — found by walking the prototype chain

// `class` is sugar over the same mechanism:
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound.`; }
}
class Dog extends Animal {} // Dog.prototype's prototype is Animal.prototype

new Dog('Rex').speak(); // same lookup mechanism under the hood
```

### 169. What does `this` refer to, and how does it change across different call styles?

**Short Answer**

`this` depends on **how the function is called**, not where it was written:

- `obj.method()` → `this` is `obj`
- standalone call → `this` is `undefined` in strict mode
- called with `new` → `this` is the new object

Arrow functions are the exception: they ignore the call style and take `this` from the surrounding scope where they were written.

**Simple Explanation**

The surprising case is detaching a method: `const fn = obj.method; fn();`. Now there's no `obj` on the left of the dot, so the connection is lost and `this` is no longer `obj`.

Arrow functions never get their own `this`. They permanently capture it from wherever they were defined.

That's exactly why arrow functions are so useful for callbacks inside class methods — a `setInterval(() => this.seconds++, 1000)` keeps the instance's `this` without needing `.bind(this)`.

**Example**

```js
const obj = {
  name: 'Widget',
  regular: function () { console.log(this.name); },
  arrow: () => { console.log(this.name); }, // `this` from the surrounding scope, not `obj`
};

obj.regular();           // "Widget" — called as obj.regular(), this === obj
const detached = obj.regular;
detached();               // undefined (or throws in strict mode) — this is no longer obj

obj.arrow();              // undefined — arrow ignored obj entirely

class Timer {
  constructor() { this.seconds = 0; }
  start() {
    setInterval(() => {
      this.seconds++; // arrow function: `this` is the Timer instance, captured lexically
    }, 1000);
  }
}
```

### 170. What are the key ES6+ features you use day to day?

**Short Answer**

The ones you use daily: arrow functions, destructuring, spread/rest (`...`), and template literals — plus `let`/`const`, default parameters, `class`, modules, and Promises.

**Simple Explanation**

- **Destructuring** pulls values out of objects and arrays into named variables in one line, including nested values and defaults.
- **Spread (`...`)** expands an array or object into individual items — handy for merging objects or combining arrays.
- **Rest (`...`)** does the reverse, collecting leftover arguments into an array.
- **Template literals** (backticks) let you embed `${expressions}` and write multi-line strings without concatenation.
- **Arrow functions** give shorter syntax and lexical `this` (see the previous question).

None of these changed how JavaScript fundamentally works — they just made it far more pleasant to write.

**Example**

```js
// Arrow function + default parameter + template literal
const greet = (name = 'friend') => `Hello, ${name}!`;

// Destructuring: nested + default value, plus array destructuring + rest
const { id, profile: { email } = {} } = user;
const [first, second, ...rest] = [1, 2, 3, 4];

// Spread: shallow-merge objects, concatenate arrays
const merged = { ...defaults, ...overrides };
const combined = [...arrA, ...arrB];

// Rest parameters: collect all remaining arguments into an array
function sum(...nums) {
  return nums.reduce((total, n) => total + n, 0);
}

console.log(greet());      // "Hello, friend!"
console.log(sum(1, 2, 3)); // 6
```

**— Asynchronous JavaScript —**

### 171. What's the JavaScript event loop, and what's the difference between microtasks and macrotasks?

**Short Answer**

JavaScript runs on one thread with one call stack. After each **macrotask** (`setTimeout`, I/O, UI events), the event loop drains the **entire microtask queue** (Promise callbacks) before picking up the next macrotask.

That's why a resolved Promise's `.then` always runs before a `setTimeout(fn, 0)`.

**Simple Explanation**

Picture two queues waiting behind the currently running code:

- **Microtasks** — Promise `.then`/`.catch`/`.finally`, `queueMicrotask`.
- **Macrotasks** — `setTimeout`, `setInterval`, DOM events, network callbacks.

Once the current synchronous code finishes, the event loop empties the **whole** microtask queue — including any new microtasks added along the way — before it takes even one macrotask.

So for code that logs synchronously, then schedules a `setTimeout`, then schedules a Promise callback, the order is: both synchronous logs, then the Promise callback, then the `setTimeout` callback.

**Example**

```js
console.log('1: sync start');

setTimeout(() => console.log('4: macrotask (setTimeout)'), 0);

Promise.resolve().then(() => console.log('3: microtask (promise)'));

console.log('2: sync end');

// Output order:
// 1: sync start
// 2: sync end
// 3: microtask (promise)
// 4: macrotask (setTimeout)
```

### 172. What are the states of a Promise, and how do `Promise.all`, `Promise.allSettled`, and `Promise.race` differ?

**Short Answer**

A Promise is **pending**, then either **fulfilled** or **rejected** — one transition, never both.

- `Promise.all` — fails as soon as any one rejects.
- `Promise.allSettled` — waits for all of them and reports each outcome.
- `Promise.race` — settles as soon as the first one settles, success or failure.

**Simple Explanation**

**pending** means the work hasn't finished. **fulfilled** means it succeeded and `.then` runs with the value. **rejected** means it failed and `.catch` runs with the error.

Use `Promise.all` when you need everything to succeed — loading several required resources. One failure aborts the batch.

Use `Promise.allSettled` when partial failure is acceptable and you want to know what happened to each one. You get back an array of objects with a `status` of `'fulfilled'` or `'rejected'`.

Use `Promise.race` when whichever finishes first wins. The classic pairing is racing a real request against a timeout promise.

**Example**

```js
const fetchUser = id => fetch(`/api/users/${id}`).then(r => r.json());

Promise.all([fetchUser(1), fetchUser(2)])
  .then(([u1, u2]) => console.log(u1, u2))
  .catch(err => console.error('one failed, all rejected:', err));

Promise.allSettled([fetchUser(1), fetchUser(999)])
  .then(results => {
    // [{ status: 'fulfilled', value: ... }, { status: 'rejected', reason: ... }]
    results.forEach(r => (r.status === 'fulfilled' ? console.log(r.value) : console.warn(r.reason)));
  });

Promise.race([
  fetchUser(1),
  new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), 3000)),
]).catch(err => console.error(err.message)); // whichever settles first wins
```

### 173. How does `async`/`await` compare to raw Promises, and how do you handle errors with it?

**Short Answer**

`async`/`await` is nicer syntax over Promises. It lets asynchronous code read top to bottom like synchronous code, and you handle errors with a normal `try`/`catch` instead of chaining `.catch()`.

**Simple Explanation**

An `async` function always returns a Promise. `await` pauses that function until the awaited Promise settles — without blocking the rest of the program.

That flattens nested `.then()` chains into linear code.

And because an awaited rejection **throws** at the `await` line, you can wrap it in a normal `try`/`catch`, which is a much more familiar way to handle failures.

**Example**

```js
// Raw promise chain
function loadProfile(id) {
  return fetchUser(id)
    .then(user => fetchPosts(user.id))
    .then(posts => posts.length)
    .catch(err => { console.error(err); return 0; });
}

// async/await equivalent — same behavior, linear control flow
async function loadProfileAsync(id) {
  try {
    const user = await fetchUser(id);
    const posts = await fetchPosts(user.id);
    return posts.length;
  } catch (err) {
    console.error(err);
    return 0;
  }
}
```

### 174. What's the difference between debounce and throttle, and when would you use each?

**Short Answer**

- **Debounce** — wait until the events stop for N milliseconds, then run once. Good for search-as-you-type.
- **Throttle** — run at most once every N milliseconds while events keep firing. Good for scroll and resize handlers.

**Simple Explanation**

Both limit how often a handler runs, but they solve different problems.

Debounce is "wait until the user stops". A search box doesn't fire a request on every keystroke — only after typing pauses.

Throttle is "run regularly no matter what". A scroll handler that repositions a sticky header still needs to run during a long scroll, not just once at the very end.

Quick way to remember: debounce fires **after** the burst; throttle fires **during** it, at a steady rate.

**Example**

```js
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

function throttle(fn, interval) {
  let lastCall = 0;
  return (...args) => {
    const now = Date.now();
    if (now - lastCall >= interval) {
      lastCall = now;
      fn(...args);
    }
  };
}

searchInput.addEventListener('input', debounce(e => {
  fetchSearchResults(e.target.value); // fires 300ms after typing stops
}, 300));

window.addEventListener('scroll', throttle(() => {
  updateStickyHeader(); // fires at most once every 200ms while scrolling
}, 200));
```

**— Functional Patterns & Data Handling —**

### 175. What are higher-order functions, and how do `map`, `filter`, and `reduce` use that pattern?

**Short Answer**

A higher-order function takes a function as an argument, returns one, or both.

- `map` — transform each item, same length out.
- `filter` — keep matching items, shorter or equal length.
- `reduce` — fold everything into one value (a number, an object, anything).

**Simple Explanation**

Instead of a manual `for` loop with a mutable accumulator, these let you say *what* transformation you want, and they chain together cleanly.

`map` always returns an array of the same length. `filter` returns the same length or fewer. `reduce` can return absolutely anything, depending on what you build up.

A function that *returns* a function is also higher-order — that's how you make configurable helpers, like a `withTax(rate)` that returns a function calculating totals at that rate.

**Example**

```js
const orders = [
  { id: 1, total: 42.5, status: 'paid' },
  { id: 2, total: 15.0, status: 'refunded' },
  { id: 3, total: 99.99, status: 'paid' },
];

const paidTotals = orders
  .filter(o => o.status === 'paid') // keep only paid orders
  .map(o => o.total);                // [42.5, 99.99]

const grandTotal = paidTotals.reduce((sum, total) => sum + total, 0); // 142.49

// A higher-order function that returns a function:
const withTax = rate => total => total * (1 + rate);
const addSalesTax = withTax(0.0825);
console.log(addSalesTax(100)); // 108.25
```

### 176. What's the difference between a shallow copy and a deep copy in JavaScript?

**Short Answer**

- **Shallow copy** — only the top level is duplicated. Nested objects are still **shared** with the original.
- **Deep copy** — everything is cloned recursively, so nothing is shared.

`structuredClone()` is the modern built-in way to deep copy.

**Simple Explanation**

Spreading an object (`{ ...obj }`) or using `Object.assign` copies one level. If a property's value is itself an object, both the copy and the original point at the **same** nested object — so changing it through one is visible through the other. That's a very common source of bugs.

`structuredClone()` clones the whole structure, including `Date`, `Map`, `Set`, and circular references.

The older `JSON.parse(JSON.stringify(obj))` trick works for plain data but silently drops functions and `undefined` values, turns `Date` objects into strings, and throws on circular references.

**Example**

```js
const original = { name: 'Order', meta: { tags: ['urgent'] } };

const shallow = { ...original };
shallow.meta.tags.push('rush');
console.log(original.meta.tags); // ['urgent', 'rush'] — nested object was shared!

const deep = structuredClone(original); // real deep clone — Dates, Maps, circular refs all handled
deep.meta.tags.push('changed');
console.log(original.meta.tags); // unaffected

// The old workaround and its limits:
const viaJson = JSON.parse(JSON.stringify({ fn: () => {}, when: new Date(), bad: undefined }));
console.log(viaJson); // { when: "2026-...Z" } — function and undefined silently dropped
```

**— Memory, Modules & Scope —**

### 177. How can a closure cause a memory leak?

**Short Answer**

A closure keeps everything in its surrounding scope alive as long as the closure itself is reachable.

So attaching a closure to something long-lived — like an event listener you never remove — keeps any large object it references in memory forever.

**Simple Explanation**

Garbage collection frees memory once nothing can reach it. A closure counts as a reference.

So imagine a button that stays on the page forever with a listener attached. If that listener's closure captured a big array or a whole API response, that data is pinned in memory for the life of the page — even though the code has no further use for it.

Two fixes:

1. Capture only the small value you actually need, not the whole dataset.
2. Call `removeEventListener` when the element or component is torn down.

**Example**

```js
function attachHandler() {
  const largeDataset = new Array(1_000_000).fill('leaked'); // a big chunk of memory

  document.getElementById('save-btn').addEventListener('click', () => {
    console.log(largeDataset.length); // closure keeps largeDataset alive indefinitely
  });
}
attachHandler();
// largeDataset can never be garbage collected — the listener's closure still
// references it, and the listener is never removed while the button stays on the page.

// Fix: capture only the small value you actually need, and clean up when done
function attachHandlerFixed() {
  const count = computeExpensiveCount(); // derive just what's needed, don't keep the whole dataset
  const btn = document.getElementById('save-btn');
  const handler = () => console.log(count);
  btn.addEventListener('click', handler);
  // later, when the component/page tears down:
  // btn.removeEventListener('click', handler);
}
```

### 178. What's the practical difference between CommonJS and ES Modules?

**Short Answer**

- **CommonJS** (`require` / `module.exports`) — resolved at runtime, so `require()` can be conditional or use a computed path.
- **ES Modules** (`import` / `export`) — statically analyzable at parse time, which is what makes tree-shaking and top-level `await` possible.

**Simple Explanation**

With CommonJS, `require()` is just a function call. It can appear inside an `if`, with a variable path, because it's resolved while the code runs.

ES Modules require `import`/`export` at the top level with a literal path. That sounds restrictive, but it means a bundler can build the whole dependency graph **without running any code**.

From that, it can prove "this export is never imported anywhere" and delete it from the shipped bundle. That's tree-shaking, and it's why ESM produces smaller bundles.

**Example**

```js
// CommonJS (Node's original module system)
const { formatCurrency } = require('./money'); // resolved synchronously, at any point in the code
module.exports = { chargeCustomer };

// ES Modules (modern standard — browsers and modern Node)
import { formatCurrency } from './money.js'; // must be top-level, statically analyzable
export function chargeCustomer(amount) { /* ... */ }

// Because ESM imports/exports are static, a bundler can prove "formatOnlyCents
// is never imported anywhere" and delete it from the shipped bundle — that's
// tree-shaking. CommonJS's dynamic require(someVariable) calls can't be analyzed that way.
```

**— DOM & Browser APIs —**

### 179. What do optional chaining (`?.`) and nullish coalescing (`??`) do, and how are they different from plain null checking?

**Short Answer**

- `?.` (optional chaining) — returns `undefined` instead of throwing when you access something on `null` or `undefined`.
- `??` (nullish coalescing) — provides a fallback **only** when the left side is `null` or `undefined`.

The difference from `||` matters: `||` also overrides valid falsy values like `0` and `''`.

**Simple Explanation**

Before optional chaining, safely reading a nested property meant a chain of `&&` checks. `?.` collapses that into one expression, and it works for method calls too (`obj.maybeMethod?.()`).

`??` solves a related but different problem. `quantity || 10` gives you `10` when quantity is `0` — which is wrong, because `0` is a perfectly valid quantity. `quantity ?? 10` gives you `0`, because only `null` and `undefined` trigger the fallback.

**Example**

```js
const user = { profile: null };

// Without optional chaining — throws:
// console.log(user.profile.email); // TypeError: Cannot read properties of null

console.log(user.profile?.email);      // undefined, no throw
console.log(user.getPreferences?.());  // undefined — safe even for a missing method

const quantity = 0;
console.log(quantity || 10); // 10 — WRONG, 0 is falsy so || overrides a valid value
console.log(quantity ?? 10); // 0  — correct, only null/undefined trigger the fallback
```

### 180. What is event delegation, and why is it useful?

**Short Answer**

Instead of adding a listener to every child element, add **one** listener to a shared parent and check `event.target` to see what was actually clicked.

This automatically covers elements added later, and uses far less memory.

**Simple Explanation**

DOM events bubble up from the element they happened on through all its ancestors. So a click anywhere inside a table also reaches the table itself.

One listener on the table can catch it and use `event.target.closest('.delete-btn')` to work out exactly which button was clicked.

The big win is dynamic content. With per-row listeners, rows added after page load get no listener and silently don't work. A delegated listener on the parent just keeps working.

**Example**

```js
// Bad: one listener per row — rows added later get no listener at all
document.querySelectorAll('.order-row .delete-btn').forEach(btn => {
  btn.addEventListener('click', handleDelete);
});

// Good: one listener on the parent, works for rows added after page load
document.getElementById('orders-table').addEventListener('click', (event) => {
  const btn = event.target.closest('.delete-btn');
  if (!btn) return; // click wasn't on (or inside) a delete button
  const row = btn.closest('.order-row');
  deleteOrder(row.dataset.orderId);
});
```

### 181. How do you select elements and wire up basic behavior with the DOM API?

**Short Answer**

- `querySelector` / `querySelectorAll` — select elements using CSS selectors.
- `.textContent` / `.innerHTML` / `.value` — read and write content.
- `addEventListener` — wire up behavior.

**Simple Explanation**

`querySelector` returns the first match or `null`. `querySelectorAll` returns a static list of all matches that you can `forEach` over.

Prefer `.textContent` over `.innerHTML` when inserting plain text. `.innerHTML` parses its input as HTML, which is an XSS risk if the content ever comes from a user.

`dataset` gives you easy access to `data-*` attributes — `element.dataset.tagId` reads `data-tag-id`.

For forms, call `event.preventDefault()` in a submit handler to stop the default full-page reload.

**Example**

```js
const banner = document.querySelector('.flash-banner');
banner.textContent = 'Saved successfully'; // safe: sets text, no HTML parsing
banner.classList.add('flash-banner--success');

document.querySelectorAll('.tag').forEach(tag => {
  console.log(tag.dataset.tagId); // reads a data-tag-id="..." attribute
});

const form = document.querySelector('#new-comment-form');
form.addEventListener('submit', (event) => {
  event.preventDefault(); // stop the default full-page form submit
  const body = form.querySelector('textarea[name="body"]').value;
  submitComment(body);
});
```

### 182. How does `fetch` work end to end, including error handling?

**Short Answer**

`fetch()` returns a Promise that resolves as soon as the response headers arrive. It does **not** reject on HTTP errors like 404 or 500 — only on network failure.

So you must check `response.ok` yourself before trusting the body.

**Simple Explanation**

This trips up a lot of people coming from other HTTP clients. A `500` response is still a "successful" fetch as far as the Promise is concerned, because the browser did get a response.

Only things like no network connection, DNS failure, or a CORS block cause the Promise itself to reject.

So correct `fetch` usage always has an explicit `if (!response.ok)` check before parsing the body as the success case. The `catch` block then handles both network failures and any error you threw yourself.

**Example**

```js
async function loadOrder(id) {
  try {
    const response = await fetch(`/api/orders/${id}`, {
      headers: { Accept: 'application/json' },
    });

    if (!response.ok) {
      // fetch does NOT throw for 404/422/500 — you have to check manually
      const body = await response.json().catch(() => null);
      throw new Error(body?.error?.message || `Request failed: ${response.status}`);
    }

    return await response.json();
  } catch (err) {
    // network failure (offline, DNS, CORS) lands here too
    console.error('Failed to load order:', err.message);
    throw err;
  }
}
```

### 183. How would you call a Rails JSON API from a Stimulus controller, including the CSRF token?

**Short Answer**

Rails rejects non-GET requests without a valid CSRF token. So your JavaScript reads the token Rails puts in a `<meta name="csrf-token">` tag and sends it back as the `X-CSRF-Token` header.

**Simple Explanation**

`<%= csrf_meta_tags %>` in the layout renders a meta tag containing a per-session token.

Rails checks that token on every state-changing request (POST, PATCH, PUT, DELETE) to prove the request came from your own page — not from a malicious site tricking a logged-in user's browser into submitting something.

Your JavaScript reads that meta tag's `content` and sends it as a header. Skip it, and Rails responds with `422 Unprocessable Entity` and an `ActionController::InvalidAuthenticityToken` error.

**Example**

```erb
<!-- app/views/layouts/application.html.erb -->
<%= csrf_meta_tags %> <!-- renders <meta name="csrf-token" content="..."> -->
```

```js
// app/javascript/controllers/order_controller.js
import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
  static values = { id: Number };

  async cancel() {
    const csrfToken = document.querySelector('meta[name="csrf-token"]')?.content;

    const response = await fetch(`/orders/${this.idValue}/cancel`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': csrfToken, // required or Rails responds 422 InvalidAuthenticityToken
        Accept: 'application/json',
      },
      body: JSON.stringify({ reason: 'customer_request' }),
    });

    if (!response.ok) {
      const { error } = await response.json();
      alert(error.message);
      return;
    }

    this.element.remove();
  }
}
```

### 184. How do `localStorage` and `sessionStorage` work, and what are their limitations?

**Short Answer**

- `localStorage` — stays until you explicitly clear it, shared across tabs for that site.
- `sessionStorage` — scoped to one tab, cleared when that tab closes.

Both store **strings only**, and both are synchronous, so large values can briefly freeze the UI.

**Simple Explanation**

Neither accepts objects directly. Assigning a non-string silently turns it into `"[object Object]"`, so you `JSON.stringify` on the way in and `JSON.parse` on the way out.

Because they're synchronous, a very large value blocks the main thread while it's read or written. That's why they're unsuitable for large datasets — use IndexedDB for that.

Also worth remembering: anything in either one is readable by any JavaScript on the page, so they're not a safe place for sensitive tokens.

**Example**

```js
// Writing (must be a string — objects get coerced silently otherwise)
localStorage.setItem('draftComment', JSON.stringify({ body: 'Great point!', postId: 42 }));

// Reading
const raw = localStorage.getItem('draftComment');
const draft = raw ? JSON.parse(raw) : null;

localStorage.removeItem('draftComment');

// sessionStorage: identical API, but scoped to this tab only — gone once the tab closes
sessionStorage.setItem('wizardStep', '3');
```

## APIs

**— HTTP Fundamentals —**

### 185. What do "safe," "idempotent," and "cacheable" mean for HTTP verbs?

**Short Answer**

- **Safe** — the request doesn't change anything on the server.
- **Idempotent** — sending it many times has the same effect as sending it once.
- **Cacheable** — the response can be stored and reused.

GET/HEAD/OPTIONS are safe and idempotent. PUT/DELETE are idempotent but not safe. POST is neither.

**Simple Explanation**

| Verb | Safe | Idempotent | Cacheable |
|---|---|---|---|
| GET | Yes | Yes | Yes |
| HEAD | Yes | Yes | Yes |
| OPTIONS | Yes | Yes | No |
| POST | No | No | Only with explicit cache headers |
| PUT | No | Yes | No |
| PATCH | No | Not guaranteed | No |
| DELETE | No | Yes | No |

**Idempotent** matters most for retries. If a network blip means the client doesn't know whether a `PUT` succeeded, it's always safe to just send it again — the end state is the same either way.

A `POST` doesn't have that guarantee. Sending it twice can create two records. That's exactly why idempotency keys exist (covered later).

**Example**

```ruby
class OrdersController < ApplicationController
  # GET is safe + idempotent — browsers/proxies can cache and safely prefetch it
  def show
    @order = Order.find(params[:id])
  end

  # PUT is idempotent: sending the same payload twice leaves the same end state
  def update
    order = Order.find(params[:id])
    order.update!(order_params)
    render json: order
  end

  # DELETE is idempotent: deleting an already-deleted resource is a no-op, not a new deletion
  def destroy
    Order.find(params[:id]).destroy!
    head :no_content
  end

  # POST is NOT idempotent — calling it twice creates two orders unless guarded by an idempotency key
  def create
    order = Order.create!(order_params)
    render json: order, status: :created
  end
end
```

### 186. What do the common HTTP status codes mean, and when would a Rails app actually return each?

**Short Answer**

- **2xx** — success
- **3xx** — redirect
- **4xx** — the client's request was the problem
- **5xx** — the server broke

A good API picks the specific code that tells the client what to do next.

**Simple Explanation**

The ones you'll actually use:

- **200 OK** — successful GET or update with a body.
- **201 Created** — successful POST that created something. Include a `Location` header.
- **204 No Content** — successful DELETE, or nothing to return.
- **301 / 302** — permanent vs temporary redirect.
- **304 Not Modified** — the client's cached copy is still valid.
- **400 Bad Request** — the request itself is malformed (unparseable JSON, missing required param).
- **401 Unauthorized** — not authenticated.
- **403 Forbidden** — authenticated, but not allowed.
- **404 Not Found** — doesn't exist (or is hidden on purpose).
- **409 Conflict** — a version mismatch or a unique constraint violation.
- **422 Unprocessable Entity** — well-formed but failed validation. Rails' classic response to a failed save.
- **429 Too Many Requests** — rate limited. Include `Retry-After`.
- **500 Internal Server Error** — an unhandled exception in your app.
- **502 Bad Gateway** — an upstream service returned something invalid.
- **503 Service Unavailable** — deliberately down or overloaded.
- **504 Gateway Timeout** — an upstream took too long.

The key distinction: 400 means "your request is the wrong shape", 422 means "the shape is fine but the data failed our rules".

**Example**

```ruby
class Api::V1::OrdersController < Api::V1::BaseController
  def create
    order = current_user.orders.build(order_params)

    if order.save
      render json: order, status: :created                       # 201
    else
      render json: { errors: order.errors }, status: :unprocessable_entity # 422
    end
  end

  def show
    order = current_user.orders.find(params[:id]) # raises RecordNotFound -> 404 below
    fresh_when order                                # sets ETag; returns 304 if client's copy matches
    render json: order
  end

  rescue_from ActiveRecord::RecordNotFound do
    render json: { error: 'not_found' }, status: :not_found      # 404
  end

  rescue_from ActiveRecord::StaleObjectError do
    render json: { error: 'conflict' }, status: :conflict         # 409, optimistic locking clash
  end
end
```

### 187. What's the difference between 401 and 403?

**Short Answer**

- **401 Unauthorized** — "I don't know who you are." Missing or invalid credentials.
- **403 Forbidden** — "I know exactly who you are, and you still can't do this."

The names are misleading, which is why this comes up in interviews.

**Simple Explanation**

Think of 401 as "please log in" and 403 as "you're logged in, but no".

A request with no `Authorization` header, or with an expired token, gets **401**.

A request from a perfectly valid, authenticated non-admin user hitting an admin-only endpoint gets **403** — the server identified them fine and made a deliberate decision to deny.

**Example**

```ruby
class ApplicationController < ActionController::API
  before_action :authenticate_user!

  def authenticate_user!
    token = request.headers['Authorization']&.split('Bearer ')&.last
    @current_user = AuthToken.find_user(token)
    render json: { error: 'unauthenticated' }, status: :unauthorized unless @current_user # 401
  end
end

class Admin::ReportsController < ApplicationController
  before_action :require_admin!

  def require_admin!
    # We KNOW who they are — authenticate_user! already passed — they're just not allowed here
    render json: { error: 'forbidden' }, status: :forbidden unless @current_user.admin? # 403
  end
end
```

**— REST API Design —**

### 188. What does RESTful resource design actually mean in practice?

**Short Answer**

Design your API around **nouns** (resources) acted on by standard HTTP verbs — not verbs stuffed into the URL like `/cancelOrder?id=5`.

**Simple Explanation**

A resource is a "thing" your API exposes — an order, a user, a line item.

Instead of inventing a new endpoint per action (`/getOrder`, `/updateOrder`, `/cancelOrder`), REST reuses the same URL (`/orders/:id`) with different verbs:

- `GET /orders/:id` → read
- `PATCH /orders/:id` → update
- `DELETE /orders/:id` → delete

For genuinely action-like operations that don't fit CRUD, model them as a sub-resource: `POST /orders/:id/cancel`. That still reads as something belonging to the order, rather than a random verb in a query string.

**Example**

```ruby
# config/routes.rb
resources :orders, only: [:index, :show, :create, :update, :destroy] do
  member do
    post :cancel   # POST /orders/:id/cancel — an action modeled as a sub-resource, still noun-ish
  end
  resources :line_items, only: [:index, :create, :destroy]
end

# Generates:
# GET    /orders               -> index
# POST   /orders               -> create
# GET    /orders/:id           -> show
# PATCH  /orders/:id           -> update
# DELETE /orders/:id           -> destroy
# POST   /orders/:id/cancel    -> OrdersController#cancel
# GET    /orders/:id/line_items -> LineItemsController#index
```

### 189. What is idempotency, and why does a retried POST need an idempotency key?

**Short Answer**

**Idempotent** means doing the same operation several times gives the same final result.

GET, PUT, and DELETE are idempotent by design. POST isn't. So for something like a payment charge, the client sends a unique **idempotency key** with the request, and the server uses it to recognise a repeat and return the original result instead of charging twice.

**Simple Explanation**

The problem: a client sends `POST /charges`, the network times out, and the client doesn't know whether the charge went through. If it retries and the first one actually succeeded, you've double-charged.

The fix: the client generates a unique key (a UUID) once per logical operation and sends it with the request.

The server records which keys it has already processed and what the result was. If the same key shows up again, it replays the stored response instead of doing the work again.

This is standard for payment APIs — Stripe popularised the `Idempotency-Key` header — and for anything else where doing it twice is dangerous.

**Example**

```ruby
class ChargesController < ApplicationController
  def create
    key = request.headers['Idempotency-Key']
    return render json: { error: 'idempotency_key_required' }, status: :bad_request unless key

    existing = IdempotencyRecord.find_by(key: key)
    if existing
      return render json: existing.response_body, status: existing.response_status # replay, don't re-charge
    end

    charge = PaymentGateway.charge(amount: params[:amount], customer: current_user)

    IdempotencyRecord.create!(key: key, response_status: 201, response_body: charge.to_json)
    render json: charge, status: :created
  end
end
```

### 190. What are the trade-offs between the common API versioning strategies?

**Short Answer**

- **URL path** (`/v1/orders`) — most explicit, easiest to debug, cache-friendly.
- **Header** (`Accept: application/vnd.myapp.v2+json`) — clean URLs, but invisible when debugging and easy to forget.
- **Query param** (`?version=1`) — easiest to add, but pollutes caching and logging.

**Simple Explanation**

URL versioning wins on practicality. You can see the version in every log line and every curl command, and CDNs naturally treat `/v1/` and `/v2/` as different cache entries.

Header versioning is what REST purists prefer, since the URL should identify the resource, not its format. But you can't just point a browser at it, and a client that forgets the header silently gets whatever default the server picks.

Query-param versioning is easy to bolt on but is the weakest signal — easy to omit, and many caches ignore query params by default.

Most Rails APIs use URL path versioning because the operational simplicity outweighs the purity argument.

**Example**

```ruby
# URL-path versioning (most common in Rails APIs)
namespace :api do
  namespace :v1 do
    resources :orders, only: [:index, :show]
  end
  namespace :v2 do
    resources :orders, only: [:index, :show] # v2 can change the response shape freely
  end
end

# Header-based versioning via the Accept header
class ApplicationController < ActionController::API
  def api_version
    request.headers['Accept'][/vnd\.myapp\.v(\d+)/, 1]&.to_i || 1
  end
end
```

### 191. What is a webhook, and how do you make webhook delivery reliable?

**Short Answer**

A webhook is a callback: instead of you polling a third party, they POST an event to a URL you registered whenever something happens.

To make it reliable: the sender retries with backoff, and the receiver **verifies a signature**, **rejects replays**, and **handles duplicates safely**.

**Simple Explanation**

Three things the receiver must do:

**1. Verify the signature.** The sender computes an HMAC (a hash of the payload using a shared secret) and sends it as a header. You recompute it and compare. That proves the payload really came from them and wasn't tampered with.

**2. Reject replays.** Signatures also include a timestamp, checked against a tolerance window, so someone who intercepts a valid webhook can't just resend it later to trigger the same effect again.

**3. Be idempotent.** Senders retry on any non-2xx response or timeout, so you *will* see the same event more than once. Store the event ID and skip anything you've already processed.

Also: respond fast. Do the real work in a background job rather than making the sender wait.

**Example**

```ruby
class WebhooksController < ActionController::API
  skip_before_action :verify_authenticity_token

  def stripe
    payload = request.body.read
    signature = request.headers['Stripe-Signature']

    event = begin
      # construct_event verifies the HMAC signature AND checks the timestamp
      # against a tolerance window to reject replayed/stale requests
      Stripe::Webhook.construct_event(payload, signature, Rails.application.credentials.stripe_webhook_secret)
    rescue Stripe::SignatureVerificationError
      return head :bad_request
    end

    # Idempotent handling — webhooks can arrive more than once for the same event
    return head :ok if WebhookEvent.exists?(external_id: event.id)

    WebhookEvent.create!(external_id: event.id, event_type: event.type)
    OrderPaymentJob.perform_later(event.data.object.id) # hand off, respond fast

    head :ok
  end
end
```

### 192. How would you design API rate limiting — token bucket, fixed window, or sliding window?

**Short Answer**

- **Fixed window** — count requests per clock-aligned interval. Simple, but allows up to double the limit right at a window boundary.
- **Sliding window** — smooths that boundary problem out, at the cost of more bookkeeping.
- **Token bucket** — allows short bursts up to a bucket size while refilling at a steady rate.

**Simple Explanation**

**Fixed window** is easiest: "100 requests per IP per minute". Its flaw is the boundary — a client can send 100 at 0:59 and another 100 at 1:00, so 200 requests in two seconds without ever breaking the stated rule.

**Sliding window** fixes that by weighting the previous window into the current one, giving a much closer approximation of "100 per any rolling 60 seconds".

**Token bucket** models it as a bucket holding up to N tokens, refilled at a steady rate. Each request costs a token. This naturally allows short bursts — useful for bursty-but-legitimate traffic — while still enforcing a long-run average.

Whichever you pick, return `429` with a `Retry-After` header so well-behaved clients back off properly.

**Example**

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Fixed window: at most 100 requests per IP per 1-minute clock-aligned window.
  # Gotcha: a client could send 100 requests at 0:59 and another 100 at 1:00 — 200 in 2 seconds.
  throttle('api/ip', limit: 100, period: 60) do |req|
    req.ip if req.path.start_with?('/api/')
  end
end

# Token bucket (conceptually, e.g. via redis-cell or a Redis Lua script):
# Bucket holds up to `burst` tokens, refills at `rate` tokens/sec.
# Each request costs 1 token; if none are available, reject with 429 + Retry-After.
```

### 193. How do you design consistent error responses across an API?

**Short Answer**

Use **one predictable JSON shape** for every error: an HTTP status code, a machine-readable error code, a human-readable message, and optional field-level details.

That way clients branch on `error.code` instead of parsing message strings.

**Simple Explanation**

Inconsistent error shapes — sometimes a string, sometimes an array, sometimes nested differently — force every client to special-case each endpoint.

A single `rescue_from` chain in `ApplicationController` that maps each exception type to the same envelope keeps it uniform across the whole API, so frontend and mobile clients have one parsing path.

A good envelope looks like:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "One or more fields are invalid.",
    "fields": { "email": ["can't be blank"] }
  }
}
```

The `code` is what clients should branch on. The `message` is for humans. Never put a stack trace in there for external clients.

**Example**

```ruby
class ApplicationController < ActionController::API
  rescue_from ActiveRecord::RecordInvalid, with: :render_validation_error
  rescue_from ActiveRecord::RecordNotFound, with: :render_not_found

  private

  def render_validation_error(exception)
    render json: {
      error: {
        code: 'validation_failed',
        message: 'One or more fields are invalid.',
        fields: exception.record.errors.to_hash(full_messages: true),
      },
    }, status: :unprocessable_entity
  end

  def render_not_found(exception)
    render json: { error: { code: 'not_found', message: exception.message } }, status: :not_found
  end
end

# Example response body:
# {
#   "error": {
#     "code": "validation_failed",
#     "message": "One or more fields are invalid.",
#     "fields": { "email": ["can't be blank"], "total": ["must be greater than 0"] }
#   }
# }
```

### 194. When does a Rails app need async messaging (a queue) instead of just answering inline?

**Short Answer**

Answer inline when the client needs the result immediately and the work is fast.

Use a queue when the work is slow, calls something unreliable, or isn't needed to answer the request — then return `202 Accepted` and let a background job do it.

**Simple Explanation**

Every request held open ties up a web worker for that whole time.

A synchronous controller action is fine for a fast database read. But generating a big report, sending a batch of emails, or calling a flaky third-party API inline means the client's connection sits open for however long that takes — and it can time out.

Worse, one slow dependency can back up your entire request-handling capacity.

Moving it to a background job frees the request immediately and isolates that dependency's failures from your request cycle. The client gets a `202` meaning "accepted, working on it", and polls or gets notified later.

**Example**

```ruby
class ReportsController < ApplicationController
  # Fast, needed immediately -> answer inline
  def show
    render json: OrderSummary.for(params[:order_id])
  end

  # Slow (generates a large PDF, emails it) -> don't make the client wait on an open connection
  def create
    ReportGenerationJob.perform_later(current_user.id, report_params)
    render json: { status: 'queued' }, status: :accepted # 202
  end
end

class ReportGenerationJob < ApplicationJob
  queue_as :reports

  def perform(user_id, params)
    pdf = ReportBuilder.new(params).generate
    ReportMailer.with(user_id: user_id, pdf: pdf).ready_email.deliver_later
  end
end
```

### 195. What is OpenAPI/Swagger, and why document an API contract before or while building it?

**Short Answer**

OpenAPI is a machine-readable file (YAML or JSON) describing every endpoint, request/response shape, and auth requirement in your API. "Swagger" is the older name for the same thing.

Writing it alongside the code gives you generated docs, client SDKs, and contract tests — instead of documentation that quietly drifts from reality.

**Simple Explanation**

The problem with hand-written API docs is that nobody updates them, so they slowly become wrong and then actively harmful.

In Rails, tools like `rswag` generate the OpenAPI file directly from your request specs. That means the contract and the tests that enforce it are the same artifact — if the API changes without updating the spec, the test fails.

That keeps docs honest by construction rather than by discipline. And once you have the spec, you get interactive docs (Swagger UI) and generated client libraries for free.

**Example**

```yaml
# swagger/v1/swagger.yaml (generated by rswag from request specs, or hand-written)
paths:
  /orders/{id}:
    get:
      summary: Retrieve an order
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        '200':
          description: order found
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: { type: integer }
                  status: { type: string, enum: [pending, paid, cancelled] }
        '404':
          description: order not found
```

```ruby
# spec/requests/orders_spec.rb — rswag generates the yaml above from this
RSpec.describe 'Orders API' do
  path '/orders/{id}' do
    get 'Retrieve an order' do
      produces 'application/json'
      parameter name: :id, in: :path, type: :integer
      response '200', 'order found' do
        let(:id) { create(:order).id }
        run_test!
      end
    end
  end
end
```

### 196. How do you evolve an API's response shape without breaking existing clients?

**Short Answer**

Only make **additive** changes. Add new optional fields; never change what an existing field means or its type.

When a change is genuinely breaking, ship a new version instead of changing the old one.

**Simple Explanation**

Existing clients only read the fields they know about, so adding a brand-new field is invisible to them and harmless.

The danger is changing an **existing** field. A client parsing `status` as a string breaks the moment you turn it into a nested object — even though from your side it feels like "just adding more detail".

So:

- Safe: add `tracking_url` to the response.
- Unsafe: change `"status": "paid"` into `"status": { "code": "paid", ... }`.

That second one needs `/v2/`, not an in-place change.

**Example**

```ruby
# Safe (additive): existing clients that don't know about `tracking_url` just ignore it
def serialize_order(order)
  {
    id: order.id,
    status: order.status,
    total_cents: order.total_cents,
    tracking_url: order.tracking_url, # NEW field — additive, nothing breaks
  }
end

# Unsafe: repurposing `status` from a string to a nested object breaks every client
# parsing it as a string. This needs a new version (/v2/orders) instead:
# v1: { "status": "paid" }
# v2: { "status": { "code": "paid", "updated_at": "2026-09-20T10:00:00Z" } }
```

### 197. What's the real trade-off between REST and GraphQL?

**Short Answer**

- **REST** — simple, cacheable, easy to rate-limit, but can mean multiple round trips or fetching more data than you need.
- **GraphQL** — one round trip, client asks for exactly the fields it wants, but the server is more complex, HTTP caching mostly doesn't work, and rate limiting is harder.

**Simple Explanation**

With REST, `GET /orders/5` returns a fixed shape. A CDN or browser can cache it by URL, and rate limiting is a simple "requests per endpoint per client" count.

With GraphQL, nearly every request is a POST to one `/graphql` endpoint with a different query body. So standard HTTP caching — which keys on URL and method — mostly doesn't apply.

And two queries that look similar can cost wildly different amounts depending on how deeply nested they are, which makes naive per-request rate limiting insufficient. You need query cost analysis and depth limits instead.

The real question is whether your clients' need for flexibility (mobile apps avoiding over-fetching, many frontend teams) outweighs the operational simplicity you give up.

**Example**

```ruby
# REST: fixed, predictable response shape per endpoint — trivially cacheable by URL
class OrdersController < ApplicationController
  def show
    render json: OrderSerializer.new(Order.find(params[:id])) # always the same shape
  end
end
```

**— Auth for APIs —**

### 198. What's the difference between JWT-based and session-based authentication?

**Short Answer**

- **Session-based** — a session ID in a cookie that maps to server-side state. You can revoke it instantly by deleting that record.
- **JWT** — a self-contained signed token. No database lookup needed to verify it, but you **can't easily revoke it** before it expires.

**Simple Explanation**

With sessions, the cookie is just an opaque ID. The server looks it up to find out who you are. That indirection is exactly what makes logout instant — delete the record and the cookie is worthless.

With a JWT, all the information (user ID, expiry, roles) is inside the signed token. Any service with the signing key can verify it without touching a database — great for high-throughput and distributed systems.

But that same statelessness means "logging out" doesn't actually invalidate a JWT that's already out there. To revoke one early you need a denylist (store its `jti` until it naturally expires) — which reintroduces the database lookup you were avoiding.

In practice, many teams use short-lived JWTs (15 minutes) plus a refresh token, so the damage window from a stolen token is small.

**Example**

```ruby
# Session-based (Rails default with cookies): instantly revocable
class SessionsController < ApplicationController
  def destroy
    session.delete(:user_id) # gone immediately, server-side
    head :no_content
  end
end

# JWT-based: stateless, but revocation needs extra machinery
class JsonWebToken
  SECRET = Rails.application.credentials.jwt_secret

  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET, 'HS256')
  end

  def self.decode(token)
    JWT.decode(token, SECRET, true, algorithm: 'HS256').first
  rescue JWT::ExpiredSignature, JWT::DecodeError
    nil
  end
end
# To revoke a JWT before it naturally expires, you need a denylist (e.g. a Redis set
# of revoked `jti` claims) checked on every request — reintroducing a server-side lookup.
```

### 199. How does the OAuth 2.0 authorization code flow work end to end?

**Short Answer**

1. Your app redirects the user to the provider to log in and approve.
2. The provider redirects back with a short-lived **authorization code**.
3. Your **server** (not the browser) exchanges that code plus your client secret for an access token.

The token never passes through the browser.

**Simple Explanation**

The key security property is that the sensitive exchange happens server-to-server, over a back channel. The access token never appears in a URL, browser history, or JavaScript.

The `state` parameter matters too. Your server generates a random value before the redirect and checks it when the provider redirects back. A malicious site can't forge the callback because it can't guess your `state` — that's CSRF protection for the login flow itself.

The authorization code is short-lived and single-use, so even if it leaked from the redirect URL, it's useless without the client secret.

**Example**

```ruby
class OauthController < ApplicationController
  # Step 1: redirect the user to the provider
  def authorize
    redirect_to "https://provider.example.com/oauth/authorize?" + {
      client_id: ENV['OAUTH_CLIENT_ID'],
      redirect_uri: oauth_callback_url,
      response_type: 'code',
      scope: 'profile email',
      state: session[:oauth_state] = SecureRandom.hex(16), # CSRF protection for the flow itself
    }.to_query
  end

  # Step 2: provider redirects back here with ?code=...&state=...
  def callback
    raise 'state mismatch' unless params[:state] == session.delete(:oauth_state)

    # Step 3: server-to-server exchange — code AND client secret never touch the browser
    response = Faraday.post('https://provider.example.com/oauth/token', {
      grant_type: 'authorization_code',
      code: params[:code],
      redirect_uri: oauth_callback_url,
      client_id: ENV['OAUTH_CLIENT_ID'],
      client_secret: Rails.application.credentials.oauth_client_secret,
    })

    access_token = JSON.parse(response.body)['access_token']
    sign_in_user_with(access_token)
  end
end
```

### 200. What's the difference between an access token and a refresh token?

**Short Answer**

- **Access token** — short-lived (minutes to hours), sent on every API request.
- **Refresh token** — long-lived, stored more carefully, used only to get a new access token without making the user log in again.

**Simple Explanation**

Splitting them limits the damage from a leak. Since the access token expires quickly, a stolen one is only useful for a short window.

The refresh token is used far less often — only when the access token expires — and typically only sent to one dedicated token endpoint, so it's exposed much less.

It's also usually **rotated**: each time you use it, you get a new refresh token and the old one is invalidated. So a stolen-but-unused refresh token becomes useless as soon as the real client refreshes.

**Example**

```ruby
class TokensController < ApplicationController
  def refresh
    record = RefreshToken.find_by(token: params[:refresh_token])

    return render json: { error: 'invalid_refresh_token' }, status: :unauthorized if record.nil? || record.expired?

    new_access_token = JsonWebToken.encode({ user_id: record.user_id }, 15.minutes.from_now) # short-lived
    render json: { access_token: new_access_token, expires_in: 900 }
    # the refresh_token itself is typically rotated here too, invalidating the old one
  end
end
```

### 201. What is PKCE, and why does a public client need it?

**Short Answer**

PKCE (Proof Key for Code Exchange) lets an app that **can't keep a secret** — a browser SPA or a mobile app — prove it's the same app that started the login flow.

It replaced the old implicit grant, which put the access token directly in the URL.

**Simple Explanation**

A backend server can hold a client secret safely, because it never ships to users. A browser app or a mobile binary can't — anything embedded there is effectively public.

PKCE closes that gap:

1. The client generates a random `code_verifier`.
2. It sends the **hash** of it (`code_challenge`) with the initial authorization request.
3. When exchanging the code for a token, it sends the **original** `code_verifier`.

The server checks that hashing the verifier produces the challenge it saw earlier. So even if an attacker intercepts the authorization code, they can't complete the exchange without the verifier, which never left the real client.

The old implicit grant skipped the code exchange entirely and handed the token back in the redirect URL — visible in browser history and server logs — which is why it's deprecated.

**Example**

```js
// Public client (browser SPA) — no client secret, uses PKCE instead
function base64UrlEncode(buffer) {
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

async function startOAuthFlow() {
  const verifier = base64UrlEncode(crypto.getRandomValues(new Uint8Array(32)));
  sessionStorage.setItem('pkce_verifier', verifier); // needed again at step 2

  const challengeBuffer = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier));
  const challenge = base64UrlEncode(challengeBuffer);

  window.location = 'https://provider.example.com/oauth/authorize?' + new URLSearchParams({
    client_id: 'my-spa-client',
    redirect_uri: window.location.origin + '/callback',
    response_type: 'code',
    code_challenge: challenge,
    code_challenge_method: 'S256',
  });
}

// Step 2, at /callback: exchange code + verifier — no client secret needed or possible
async function exchangeCode(code) {
  const verifier = sessionStorage.getItem('pkce_verifier');
  await fetch('https://provider.example.com/oauth/token', {
    method: 'POST',
    body: new URLSearchParams({ grant_type: 'authorization_code', code, code_verifier: verifier }),
  });
}
```

**— Pagination, Uploads, Filtering & Reliability —**

### 202. What's the trade-off between offset-based and cursor-based pagination?

**Short Answer**

- **Offset** (`LIMIT 20 OFFSET 100`) — simple, lets you jump to any page, but gets slower the deeper you go and can skip or duplicate rows when data changes.
- **Cursor** (`WHERE id > last_seen_id LIMIT 20`) — stays fast at any depth and is stable, but you can't jump to an arbitrary page number.

**Simple Explanation**

`OFFSET 100000` still makes the database scan and throw away 100,000 rows before returning your 20. That cost grows the deeper you page.

It's also unstable. If a row is inserted before your current position between two page requests, every later page shifts by one — so you silently skip or repeat a row.

Cursor pagination avoids both. "Give me the next 20 rows after this one" is a fast indexed range scan, the same cost on page 1 or page 5000, and it's immune to inserts elsewhere because the cursor is a row identity, not a position.

The cost: it's inherently sequential. You can go next and previous, but not straight to page 47.

Use offset for small admin tables where page numbers matter. Use cursors for public feeds and infinite scroll.

**Example**

```ruby
# Offset pagination — simple, but OFFSET 100_000 still scans/skips 100,000 rows
def index
  @orders = Order.order(:id).limit(20).offset(params[:page].to_i * 20)
end

# Cursor pagination — uses an indexed column, roughly constant-time regardless of depth
def index
  cursor = params[:after] # last-seen id from the previous page
  @orders = Order.order(:id).where(cursor ? ['id > ?', cursor] : nil).limit(20)
  render json: { data: @orders, next_cursor: @orders.last&.id }
end
```

### 203. Should pagination info live in headers or in the response body?

**Short Answer**

- **Headers** (`Link` with `rel="next"`) — keeps the JSON body a clean array of the resource.
- **Body** (`{ data: [...], next_cursor: ... }`) — easier to read from JavaScript, but every client has to unwrap an envelope.

Both are fine. Pick one and be consistent.

**Simple Explanation**

With headers, `GET /orders` returns literally just `[{...}, {...}]`. That matters if you're feeding it into something that expects an array. It also follows a long-standing HTTP convention — GitHub's API does exactly this.

With body metadata, consuming it from JavaScript is slightly simpler because `response.json()` gives you everything at once, with no header parsing.

Body-based is more common in APIs built primarily for JavaScript clients; header-based is more common in general-purpose public APIs.

**Example**

```ruby
class OrdersController < ApplicationController
  def index
    orders = Order.page(params[:page]).per(20)
    response.set_header('Link', link_header(orders))
    render json: orders # body is JUST the array, nothing else
  end

  private

  def link_header(orders)
    [
      %(<#{request.base_url}#{request.path}?page=#{orders.next_page}>; rel="next"),
      %(<#{request.base_url}#{request.path}?page=#{orders.prev_page}>; rel="prev"),
      %(<#{request.base_url}#{request.path}?page=#{orders.total_pages}>; rel="last"),
    ].compact.join(', ')
  end
end
```

### 204. What's the cost of exposing `X-Total-Count` on a large paginated resource?

**Short Answer**

An exact total means running `COUNT(*)` on every request. On a large table that count can be slower than the query returning the actual page.

**Simple Explanation**

On a small table this is instant. On tens of millions of rows with complex `WHERE` clauses, the count can take seconds and add real load under traffic.

Ways around it:

- Only compute it on the first page request.
- Cache it and refresh periodically.
- Use an approximate count from Postgres statistics.
- Drop the exact total entirely and return a cheap `has_more: true/false` instead.

That last one is often the best answer. You can get it by fetching `LIMIT 21` and checking whether a 21st row came back — no counting at all.

**Example**

```ruby
def index
  orders = current_user.orders.order(id: :desc)
  page = orders.page(params[:page]).per(20)

  # COUNT(*) on a multi-million row table can itself take seconds and contend for locks
  response.set_header('X-Total-Count', page.total_count.to_s) if params[:page].to_i <= 1

  render json: page
end

# Cheaper alternative: skip the exact count entirely — just report whether there's
# a next page (`has_more: true`), which only needs LIMIT 21 and checking for a 21st row.
```

### 205. Why shouldn't you proxy large file uploads through your app server?

**Short Answer**

Streaming a large file through your app ties up a web worker and memory for the whole upload. A **presigned URL** lets the browser upload straight to S3, and your server only issues a short-lived signed URL.

**Simple Explanation**

A presigned URL is a storage URL with a signature embedded, granting time-limited permission to upload to one specific key — without the uploader needing any AWS credentials.

So your server's job shrinks to "check this user is allowed to upload, then hand back a signed URL". That's a tiny, fast request.

Compare that to proxying: a 500MB upload on a slow connection holds one of your limited worker processes open for minutes, doing nothing but copying bytes.

You can also enforce limits at the storage layer — the presigned POST can specify a maximum content length, so an oversized file is rejected by S3 itself.

**Example**

```ruby
class UploadsController < ApplicationController
  def presign
    key = "uploads/#{current_user.id}/#{SecureRandom.uuid}/#{params[:filename]}"

    presigned_post = S3_BUCKET.presigned_post(
      key: key,
      success_action_status: '201',
      content_length_range: 1..25.megabytes, # enforce a size limit at the storage layer too
    )

    render json: { url: presigned_post.url, fields: presigned_post.fields, key: key }
  end
end
```

```js
// Browser uploads directly to S3 — bytes never pass through the Rails app server
async function uploadDirect(file) {
  const { url, fields, key } = await fetch('/uploads/presign?filename=' + file.name).then(r => r.json());

  const formData = new FormData();
  Object.entries(fields).forEach(([k, v]) => formData.append(k, v));
  formData.append('file', file);

  await fetch(url, { method: 'POST', body: formData });             // straight to S3
  await fetch('/uploads/complete', { method: 'POST', body: JSON.stringify({ key }) }); // tell Rails it's done
}
```

### 206. How does multipart/chunked upload work for very large files?

**Short Answer**

Split the file into parts, upload each part as its own request (so they can run in parallel and be retried individually), then make one final "complete" call that tells storage to stitch them together in order.

**Simple Explanation**

For a multi-gigabyte file, a single HTTP request is fragile — any network blip forces a full restart.

Multipart upload breaks the file into chunks (say 10MB each) and uploads each as a separate request. If chunk 7 fails, you only retry chunk 7, not the whole file. And chunks can upload in parallel, which is much faster.

The flow is three steps:

1. **Initiate** — storage returns an upload ID.
2. **Upload parts** — each part returns an ETag.
3. **Complete** — send the ordered list of part numbers and ETags, and storage assembles the final object.

**Example**

```ruby
class MultipartUploadsController < ApplicationController
  def create
    upload = S3_BUCKET.object(params[:key]).initiate_multipart_upload
    render json: { upload_id: upload.id, key: upload.object_key }
  end

  def presign_part
    upload = Aws::S3::MultipartUpload.new(bucket_name: S3_BUCKET.name, object_key: params[:key], id: params[:upload_id])
    url = upload.part(params[:part_number]).presigned_url(:upload_part)
    render json: { url: url }
  end

  def complete
    upload = Aws::S3::MultipartUpload.new(bucket_name: S3_BUCKET.name, object_key: params[:key], id: params[:upload_id])
    # `parts` is [{ part_number: 1, etag: "..." }, ...], collected from each part's upload response
    upload.complete(multipart_upload: { parts: params[:parts] })
    render json: { status: 'complete' }
  end
end
```

### 207. How do you validate a file upload before trusting it?

**Short Answer**

Never trust the client's `Content-Type` header or the file extension — both are trivially faked.

Instead: check the file's actual **magic bytes** (the fixed signature real formats start with), enforce a hard size limit, and scan for malware.

**Simple Explanation**

A malicious upload can claim `Content-Type: image/png`, be named `photo.png`, and actually contain an executable or a script.

Real formats start with a fixed byte signature — JPEGs begin with `FF D8 FF`, PNGs with `89 50 4E 47`, PDFs with `%PDF`. Reading the first few bytes and matching against known signatures gives you an answer the client can't fake by renaming a file.

Then:

- Enforce the size limit **during** the read, not after buffering the whole thing into memory.
- Run a malware scan before storing or processing it.
- Never serve user uploads from your own domain without care, and never execute them.

**Example**

```ruby
class UploadValidator
  ALLOWED_SIGNATURES = {
    "\xFF\xD8\xFF".b => 'image/jpeg',
    "\x89PNG".b      => 'image/png',
    "%PDF".b         => 'application/pdf',
  }.freeze

  MAX_SIZE = 10.megabytes

  def self.validate!(file)
    raise 'file_too_large' if file.size > MAX_SIZE

    header = file.read(8)
    file.rewind
    detected = ALLOWED_SIGNATURES.find { |sig, _| header.start_with?(sig) }
    raise 'unrecognized_or_spoofed_file_type' unless detected # ignores client Content-Type entirely

    scan_result = ClamScan.check(file) # or a virus-scanning service/sidecar
    raise 'malware_detected' unless scan_result.clean?

    detected.last # the real, sniffed content type
  end
end
```

### 208. How do you design safe dynamic filtering and sorting on an API?

**Short Answer**

**Whitelist** the exact columns clients may filter or sort by. Never put a client-supplied column name directly into SQL.

Also make sure every sortable column actually has an index — otherwise a client can force a full table sort.

**Simple Explanation**

`Order.order(params[:sort])` looks harmless but isn't. A client can pass any string, including a SQL injection payload, or a legitimate-looking but unindexed column that makes the database sort the entire table on every request.

The fix is a fixed array of allowed values, checked with `include?`, falling back to a safe default:

```ruby
SORTABLE = %w[created_at total_cents].freeze
sort = SORTABLE.include?(params[:sort]) ? params[:sort] : "created_at"
```

Same for filters — loop over an explicit list of filterable fields rather than passing the whole params hash to `where`.

**Example**

```ruby
class OrdersController < ApplicationController
  FILTERABLE = %w[status customer_id].freeze
  SORTABLE = %w[created_at total_cents].freeze # every one of these MUST have a DB index

  def index
    scope = Order.all

    FILTERABLE.each do |field|
      scope = scope.where(field => params[field]) if params[field].present?
    end

    sort_field = params[:sort]&.delete_prefix('-')
    if SORTABLE.include?(sort_field)
      direction = params[:sort]&.start_with?('-') ? :desc : :asc
      scope = scope.order(sort_field => direction)
    else
      scope = scope.order(created_at: :desc) # safe default — never trust an unrecognized column
    end

    # NEVER do this: scope.order("#{params[:sort]} #{params[:direction]}") — SQL injection risk
    render json: scope.page(params[:page])
  end
end
```

### 209. How does request tracing with correlation IDs work across services and background jobs?

**Short Answer**

Generate a unique request ID at the edge, include it in every log line, pass it into background jobs, and send it as a header on outgoing calls.

Then one user-reported issue can be traced end to end with a single search.

**Simple Explanation**

Without a shared ID, debugging "the report that failed at 2:14pm" means manually lining up timestamps across web logs, Sidekiq logs, and third-party logs — unreliable under concurrent traffic.

A correlation ID fixes that. Rails already generates one (`request.request_id`, exposed as the `X-Request-Id` header) via `ActionDispatch::RequestId`.

The work is in carrying it forward:

- Tag your logger with it for the whole request.
- Pass it as a job argument so background work logs under the same ID.
- Send it as a header on outgoing HTTP calls so downstream services can log it too.

Now one `grep` across every log source reconstructs the whole story.

**Example**

```ruby
# config/application.rb
config.middleware.use ActionDispatch::RequestId # sets X-Request-Id, available as request.request_id

class ApplicationController < ActionController::API
  around_action :tag_logs_with_request_id

  def tag_logs_with_request_id
    Current.request_id = request.request_id
    Rails.logger.tagged(Current.request_id) { yield }
  end
end

class OrderProcessingJob < ApplicationJob
  def perform(order_id, correlation_id:)
    Current.request_id = correlation_id # carried into the async job
    Rails.logger.tagged(correlation_id) do
      Rails.logger.info("Processing order #{order_id}")
      ExternalApi.charge(order_id, headers: { 'X-Correlation-Id' => correlation_id }) # carried downstream too
    end
  end
end

# Enqueue with the id so the job can pick it up:
OrderProcessingJob.perform_later(order.id, correlation_id: Current.request_id)
```

### 210. Which failures in a request chain are safe to retry, and who should retry them?

**Short Answer**

Retry only failures that are genuinely temporary: timeouts, `502`, `503`, `504`, and `429` (respecting `Retry-After`). Use exponential backoff.

Never retry `400` or `422` — the request itself is wrong, so sending it again just fails identically.

And never blindly retry a non-idempotent POST without an idempotency key.

**Simple Explanation**

The right behavior depends on **why** it failed.

A timeout or a `503` suggests a temporary problem, so retrying with growing delays is reasonable. A `429` is the server explicitly telling you when to try again.

A `400` or `422` means the request is malformed or failed validation. The exact same bytes will fail the exact same way forever, so retrying is pure waste and can hide a real bug.

The most dangerous case is retrying something like `POST /charges` after an ambiguous failure — a timeout where you genuinely don't know whether it went through. Without an idempotency key, that retry can create a duplicate charge.

**Example**

```ruby
class ExternalApiClient
  RETRYABLE_STATUSES = [429, 502, 503, 504].freeze

  def call_with_retry(request, max_attempts: 3)
    attempt = 0
    begin
      attempt += 1
      response = request.call

      if RETRYABLE_STATUSES.include?(response.status) && attempt < max_attempts
        sleep(retry_delay(response, attempt))
        raise RetryableError
      end

      response
    rescue Net::OpenTimeout, Net::ReadTimeout, RetryableError
      retry if attempt < max_attempts
      raise
    end
    # 400/422 responses are NEVER retried here — the request itself is malformed
    # or fails validation, and resending identical bad input just fails identically.
  end

  def retry_delay(response, attempt)
    response.headers['Retry-After']&.to_i || (2**attempt) # exponential backoff fallback
  end
end
```

## System Design

**— Design Prompts —**

### 211. Design a URL shortener.

**How to think about it**

**1. Pin down the requirements.** Two operations: POST a long URL and get a short code back, then GET the short code and redirect. Ask about the read/write ratio early — this is almost always read-heavy (100:1 or more, since links are clicked far more often than created), and that ratio drives where you spend effort. Park custom aliases, expiry, and per-user link management as extensions.

**2. Data model.** One `links` table: `id`, `short_code` (unique, indexed), `original_url`, `user_id`, optional `click_count`, `created_at`, `expires_at`.

For generating the code you have two options:

- Hash the URL and base62-encode part of the hash, retrying if you hit a collision.
- Base62-encode the row's own auto-increment primary key.

I'd lead with the second. It's collision-free by construction, because the database sequence already guarantees uniqueness — so there's no retry loop to write at all.

**3. API shape.** `POST /links { url }` returns the short code. `GET /:short_code` redirects.

Worth calling out the **301 vs 302** decision explicitly. A 301 (permanent) lets browsers cache the redirect and skip your server entirely on repeat visits — cheaper for you, but you lose click analytics and can never repoint that code. A 302 costs a request every time but keeps every click visible. I'd pick 302 when analytics matter.

**4. End-to-end flow.** Write path: validate the URL, insert, return the code. Read path: look up the code, redirect on a hit, 404 on a miss.

For click analytics, **don't** write an analytics row inside the redirect request — that adds a database write to the hottest path in the system. Enqueue a lightweight background job with the code, timestamp, referrer, and user agent, and batch-insert it off the critical path.

**5. Where it breaks at scale.** The redirect is a pure key lookup, so caching is the biggest lever — short codes never change meaning once minted, so they're perfectly cacheable in Redis or at a CDN. Next, the `links` table grows forever, so partition or archive cold codes. Finally, if you go multi-region, a single auto-increment sequence stops working as the source of uniqueness — that's when you move to a pre-generated key pool or a Snowflake-style ID scheme.

**Trade-off to say out loud:** "I'd base62-encode the primary key rather than hash, because it's collision-free with no retry logic — but that makes codes sequential and therefore guessable. If people shouldn't be able to enumerate other users' links, I'd add randomness, which brings collision handling back."

**Example**

```
links
  id            bigint PK
  short_code    string  unique index
  original_url  text
  user_id       bigint  nullable, index
  click_count   integer default 0
  created_at    datetime
  expires_at    datetime nullable

POST /links { url }  ->  { short_code: "8fK3z1", short_url: "https://tt.p/8fK3z1" }
GET  /8fK3z1          ->  302 Location: <original_url>

id=125_000_001 --base62--> "8fK3z1"   # collision-free, no lookup needed to mint a code

Redirect read path:
  Redis GET "code:8fK3z1"
    hit  -> redirect immediately
    miss -> SELECT ... WHERE short_code = '8fK3z1' -> write-through to Redis -> redirect
  ClickTrackingJob.perform_later(code, referrer, user_agent)   # async, off the critical path
```

---

### 212. Design a REST API for a content/blogging platform.

**How to think about it**

**1. Pin down the requirements.** A multi-author blog: posts, comments, authors, tags, maybe media. A public read API (anyone can read published content) and an authenticated write API (only the author or an editor can change things). Reads vastly outnumber writes.

**2. Data model.** `users` (authors), `posts` (belongs to a user, status draft/published, has many comments, tags via a join table rather than a comma-separated column so tag filtering can be indexed), `comments` (belongs to a post and a user, optionally threaded via `parent_id`), and `tags`.

**3. API shape.** Namespace under `/api/v1` for versioning.

Nest routes **one level** where ownership is obvious (`/posts/:id/comments`), but stop there. Past one level it gets unwieldy, so switch to a top-level resource with a filter (`/comments?post_id=42`).

Auth: a bearer token in the `Authorization` header. Public GETs skip auth; anything that changes data requires it.

Pagination: cursor-based for the public feed (stable and cheap at any depth), offset-based for an admin table where jumping to a page number matters more.

**4. End-to-end flow.** Client POSTs with a bearer token → authenticate the token → authorize with a policy object (does this user own this post, or have the editor role) → validate and save → serialize through a dedicated serializer so the JSON shape is explicit and decoupled from raw database columns → return `201` with a `Location` header.

**5. Where it breaks at scale.** N+1 queries on list endpoints (rendering author name and tags per row) — fix with `includes` and catch regressions with `bullet`. Public GET traffic — add `ETag`/`Cache-Control` and a CDN, since published posts are nearly static. Search across post bodies outgrows `LIKE` quickly — move to Postgres full-text search, then a dedicated engine. And comment creation is the most abuse-prone endpoint, so rate-limit it more tightly than reads.

**Trade-off to say out loud:** "I'd version with a URL prefix rather than an Accept header, because it's discoverable and testable with curl — but it costs duplicated controllers once v2 genuinely diverges from v1."

**Example**

```
GET    /api/v1/posts?cursor=...&per_page=20   # public, paginated
GET    /api/v1/posts/:id
POST   /api/v1/posts                          # auth required
PATCH  /api/v1/posts/:id                      # auth + Pundit: author or editor
POST   /api/v1/posts/:id/comments             # auth required, rate-limited
GET    /api/v1/comments?post_id=42            # flat, not triple-nested

{
  "id": 42,
  "title": "Scaling Rails Reads",
  "author": { "id": 7, "name": "Akhilesh" },
  "tags": ["rails", "postgres"],
  "published_at": "2026-09-10T12:00:00Z"
}
```

---

### 213. Design a real-time notification system.

**How to think about it**

**1. Pin down the requirements.** An event ("someone commented on your post") needs to reach a user across several channels — in-app inbox, push notification, email — each with different delivery guarantees. In scope: fan-out to those three, read/unread state, and basic deduplication. Out of scope: SMS gateways and ranked feeds.

**2. Data model.** A `notifications` table is the durable source of truth for the in-app inbox: `recipient_id`, `actor_id`, `notifiable_type`/`notifiable_id` (polymorphic, e.g. a Comment), `action`, `read_at`, `created_at`.

Push and email delivery are inherently throwaway — you don't need a permanent row proving you pushed something, just a job that tries. Add a `notification_deliveries` table only if you need per-channel status for debugging.

**3. API shape.** Triggering is internal — some other part of the app calls `NotificationService.notify(...)`, not a public endpoint.

Client-facing: `GET /notifications` (cursor-paginated), `PATCH /notifications/:id/read`, `PATCH /notifications/read_all`, plus a WebSocket subscription for live updates.

**4. End-to-end flow.** A comment is created → a service object (not a tangle of model callbacks) writes one notification row → enqueues **one job per channel**:

- in-app broadcast over ActionCable so the bell icon updates live
- push notification via FCM/APNs
- email, either immediately or batched into an hourly digest

Separate jobs per channel, not one combined job, so a bad push token or a down email provider can't delay the in-app notification.

**5. Read/unread and dedup.** `read_at` is a nullable timestamp; the unread count is `where(read_at: nil).count`, cached in Redis per user if the bell polls often. For dedup, decide whether repeat actions should collapse ("Alice and 2 others commented") — enforce it with a uniqueness key, or coalesce events within a short window.

**6. Where it breaks at scale.** The notifications table grows fast since every action fans out to N rows — partition by recipient or time and archive old read rows. In-app delivery scales on concurrent WebSocket connections (see the chat question). And push/email jobs need their own low-priority queue so a viral post can't starve latency-sensitive jobs.

**Trade-off to say out loud:** "I'd fan out with separate background jobs rather than one synchronous multi-channel send, because a slow push provider shouldn't delay the in-app notification — but that means eventual consistency, so each handler has to be safe to retry."

**Example**

```
notifications
  id            bigint PK
  recipient_id  bigint index
  actor_id      bigint
  notifiable_type/id  polymorphic
  action        string
  read_at       datetime nullable
  created_at    datetime

Comment created
  -> NotificationService.notify(...)
       -> INSERT notifications row
       -> InAppBroadcastJob.perform_later     # ActionCable push
       -> PushNotificationJob.perform_later   # FCM/APNs, retries independently
       -> EmailDigestJob.perform_later        # batched hourly, separate failure domain
```

---

### 214. Design a real-time chat system.

**How to think about it**

**1. Pin down the requirements.** One-to-one and group conversations, saved history, instant delivery to people online now, and offline users catching up on next login. The number that matters here is **concurrent open WebSocket connections**, not total registered users.

**2. Data model.** `conversations`, `conversation_participants` (tracking `last_read_message_id` per user for read receipts), and `messages` (conversation, sender, body, created_at).

Use the database's own auto-increment ID as the ordering source of truth, **not** client timestamps. Device clocks drift, and two messages sent in the same millisecond need a deterministic tiebreaker.

**3. API shape.** REST for history: `GET /conversations/:id/messages?before=cursor` for scrolling back. WebSocket for live traffic: subscribe to a channel for a conversation, send messages through it, receive broadcasts.

**4. End-to-end flow.** Client sends over its open socket → the server saves the message → broadcasts it to the conversation's stream → every subscribed client appends it instantly.

For people who aren't connected, you don't need a special offline delivery path. They'll pull it from the REST history endpoint next time they open the app (optionally also getting a push notification).

**5. Presence and typing indicators.** Don't store these — they're throwaway. Broadcast typing indicators on a separate lightweight channel with no database write, expiring client-side after a couple of seconds. Keep presence (who's online) in Redis as a set updated on connect and disconnect. Nobody needs to query who was online last Tuesday.

**6. Scaling WebSockets across servers.** This is the genuinely hard part. A message sent by someone connected to server A must reach someone connected to server B — two separate processes with two separate sets of open sockets in memory.

ActionCable solves this with a pub/sub adapter (Redis, or Postgres `LISTEN`/`NOTIFY` at smaller scale). Server A publishes the broadcast to Redis, every app server subscribes, and whichever server holds that participant's socket relays it down. This decouples "which server produced the message" from "which server holds the recipient's connection" — which also means you don't need sticky load balancing.

**7. Where it breaks at scale.** A single Redis pub/sub instance becomes a fan-out bottleneck, so shard channels across instances. Each app server has a hard ceiling on concurrent sockets (memory, file descriptors), so you scale horizontally. And very large group chats turn one broadcast into a storm across every server, so throttle or batch broadcasts for oversized rooms.

**Trade-off to say out loud:** "I'd order messages by the database's own sequence rather than client timestamps, because clocks aren't reliable across devices — but that costs a round trip, so the UI has to render optimistically and reconcile against the server's order."

**Example**

```
conversations, conversation_participants(last_read_message_id), messages(id, conversation_id, sender_id, body, created_at)

Client A                     Redis pub/sub                 Client B (on app server 2)
   |--WS speak-->[server 1]        |                              |
                  INSERT message   |                              |
                  publish(conv:9) ->|--relay-->[server 2]--WS-->  received instantly
```

---

### 215. Design a booking/e-commerce checkout system.

**How to think about it**

**1. Pin down the requirements.** Cart, then checkout against **finite** inventory — concert tickets, the last unit of a product. The interesting part here is correctness under concurrency, not the UI. Traffic is often bursty (flash sales) with heavy contention on the same rows.

**2. Data model.** `products` (with stock), `carts`/`cart_items`, `orders` (pending/paid/failed/cancelled), `order_items`, and `payments` (status, provider transaction ID, idempotency key). Optionally a `reservations` table if you want to hold stock for a few minutes during checkout.

**3. API shape.** `POST /cart/items`, `POST /checkout` (kicks off reserve → charge → confirm), and `GET /orders/:id` to check final status.

**4. End-to-end flow.** Open a **short** transaction to lock and decrement inventory and create a pending order → **release that lock** → call the payment provider → mark the order paid on success, or release the stock on failure.

The crucial point: the payment call must **not** happen inside the transaction that holds the inventory lock. Holding a row lock for the duration of an external HTTP call is how one slow payment gateway serializes every other buyer behind it.

**5. Where race conditions bite.** Two customers buying the last unit.

- **Optimistic locking** (a `lock_version` column): both read stock of 1, both try to decrement, the second save raises `StaleObjectError` and must retry or fail cleanly. Good default because it never holds a lock.
- **Pessimistic locking** (`SELECT ... FOR UPDATE`): the second transaction blocks until the first commits, then sees the updated stock. Better for a known hot single-SKU flash sale, since optimistic retries would thrash — at the cost of serializing everyone against that row.

**6. Idempotent payments.** The real danger is a retried checkout POST — a double-click, a flaky network, a mobile client's auto-retry — charging the card twice. Attach an idempotency key to the checkout attempt, store it against the order **before** calling the provider, and return the original result if the same key arrives again. Most providers (Stripe) also accept your key, so even a retried call to them is deduplicated on their side.

**7. Where it breaks at scale.** A wildly popular item's inventory row becomes a hot lock serializing every buyer. Options: drain bulk decrements from a Redis counter with a single worker, or accept eventual consistency with rare-oversell reconciliation. And payment calls are the slowest, most failure-prone step, so drive retries through a background job rather than making the browser hang.

**Trade-off to say out loud:** "I'd hold a short pessimistic lock just for the decrement and release it before calling the gateway — that keeps the hot row available, but it opens a window where stock is reserved and payment hasn't completed, so I need a reservation timeout that returns stock to the pool."

**Example**

```
Checkout:
  BEGIN
    SELECT stock FROM products WHERE id = ? FOR UPDATE   -- pessimistic, hot SKU
    stock > 0 ? decrement, create order(pending) : ROLLBACK, return 409 "sold out"
  COMMIT                                                   -- lock released here

  charge = PaymentGateway.charge(order, idempotency_key: order.idempotency_key)
  charge.success? ? order.update(status: :paid) : release_reservation(order)
```

---

### 216. Design a scalable Rails application from scratch for a new product.

**How to think about it**

**1. Pin down the requirements.** "New product" means unknown traffic and, more importantly, unknown product-market fit. The dominant early constraint is **iteration speed**, not scale. Say that out loud, then justify starting with a monolith rather than defending it apologetically.

**2. Architecture on day one.** A single Rails app, a single Postgres primary, Sidekiq plus Redis for background jobs from the start (cheap to add, and it stops slow work creeping into the request cycle later), standard MVC, and a few app processes behind a load balancer.

Don't pre-shard the database. Don't pre-split into services. Don't reach for Kafka. All of that costs iteration speed before you even know what the product is.

**3. Code shape.** Not really the point of this question, but worth one line: keep controllers thin and push logic into plain Ruby service objects from day one. If a piece genuinely needs extracting into a service later, the logic is already decoupled.

**4. End-to-end flow.** Requests hit a load-balanced pool of app servers. Most reads and writes go to the primary Postgres. Anything slow or unreliable — email, PDFs, third-party API calls — goes through Sidekiq.

**5. Where it breaks as traffic grows, roughly in order.**

1. Database read load climbs → add a read replica and route staleness-tolerant reads (dashboards, reports) to it.
2. Expensive repeated computation → add caching (fragment caching and `Rails.cache`, backed by Redis).
3. Job volume grows and job types interfere → split Sidekiq queues by priority so bulk exports can't starve password-reset emails.
4. One subsystem genuinely outgrows the monolith → *only now* consider extracting a service.

**6. When you'd actually split out a service.** Not "because microservices are best practice", but for one of a few concrete reasons: a subsystem needs a fundamentally different scaling profile (a search indexing pipeline), a subsystem needs strong isolation for compliance, or a large enough org needs independent deploy cadences and the shared pipeline is now blocking **people**, not servers.

Until one of those is genuinely painful, the network hop, the distributed transaction problem, and the extra operational surface cost more than they save.

**Trade-off to say out loud:** "I'd start with a monolith and add replicas, caching, and queues incrementally rather than pre-building for microservices — because premature extraction draws boundaries before you know where they should be. The cost is that some future extraction is more work than if I'd guessed right up front, and I'd rather pay that later with real usage data."

**Example**

```
Stage 1 (launch):      LB -> [app x2] -> Postgres primary
                                       -> Sidekiq/Redis (email, PDFs)

Stage 2 (growth):       LB -> [app x N] -> Postgres primary (writes)
                                        -> Postgres replica (reads: dashboards, reports)
                                        -> Redis cache (fragments, hot queries)
                                        -> Sidekiq (priority-split queues)

Stage 3 (real scale):   ... + one extracted service for the subsystem that
                              genuinely outgrew shared scaling/deploys
                              (e.g. search indexing), everything else stays put
```

---

### 217. Design a background-job processing system.

**How to think about it**

**1. Frame it as two questions.** The general architecture of a job system, and — as a running example — "send a welcome email on signup" designed three different ways, to show when each level of complexity actually earns its keep.

**2. General architecture.** Producer (the app enqueuing work) → broker (Redis for Sidekiq, or SQS/RabbitMQ) → workers (processes pulling jobs and running them). Plus: a retry policy with backoff, a dead-letter queue for jobs that exhaust retries, and monitoring (queue depth, job latency, failure rate).

**3. Level 1 — synchronous.** After `user.save`, call `UserMailer.welcome(user).deliver_now` directly. Simplest possible thing, fine for a prototype.

It breaks immediately in production: the signup request now blocks on an SMTP round trip, so a slow or down mail provider makes account creation slow or fails it outright. An unrelated, non-critical side effect is now coupled to the availability of your core feature.

**4. Level 2 — background job.** Same code, but `deliver_later`. The signup request returns the moment the user row is saved. If the provider hiccups, Sidekiq's retry and backoff handle it without the user noticing.

This is the right level of complexity for the large majority of "do a thing after this other thing happens" cases in a single app. I'd default here without a specific reason to go further.

**5. Level 3 — event-driven pub/sub.** Instead of the controller enqueuing a specific job, it publishes a `UserRegistered` event that any number of independent subscribers react to — one sends the welcome email, another syncs to a CRM, another runs fraud checks, another feeds analytics. The signup code doesn't know or care who's listening.

This earns its keep when several independent systems or teams genuinely need to react to the same event without the signup controller turning into a dumping ground of "and also call this".

For one team sending one welcome email, it's overkill — you'd be standing up a broker, event schemas, and subscriber infrastructure to solve a problem `deliver_later` already solved in one line.

**6. Where a job system breaks at scale.** A single queue becomes a bottleneck — partition by priority and workload. Idempotency becomes mandatory the moment retries exist, since at-least-once delivery means a job can run twice. And a job that always fails ("poison message") needs a max retry count plus a dead-letter queue so it doesn't clog the pipe forever.

**Trade-off to say out loud:** "I'd start every 'react to X' feature as a background job, not pub/sub, because it's the simplest thing that decouples the side effect from the request — and only move to a real event bus once there are genuinely multiple independent consumers."

**Example**

```ruby
# Level 1: synchronous — signup blocks on SMTP
def create
  @user = User.create!(user_params)
  UserMailer.welcome(@user).deliver_now   # request hangs if mail provider is slow
end

# Level 2: background job — signup returns immediately
def create
  @user = User.create!(user_params)
  UserMailer.welcome(@user).deliver_later # Sidekiq handles retry/backoff
end

# Level 3: event-driven — signup doesn't know who's listening
def create
  @user = User.create!(user_params)
  EventBus.publish(UserRegistered.new(user_id: @user.id))
  # subscribers, wired independently: WelcomeEmailSubscriber, CrmSyncSubscriber, ...
end
```

---

### 218. Design a file upload system.

**How to think about it**

**1. Pin down the requirements.** Uploads range from small (avatars) to large (video). There are really two questions inside this: how bytes get from the user to storage, and what happens after they land.

**2. Data model.** An `uploads` table (or Rails' Active Storage blobs): storage key, content type, byte size, status (pending/processing/processed/failed), owner, checksum.

**3. API shape — direct-to-S3, which I'd lead with.**

- `POST /uploads/presign` → server returns a presigned URL and a blob ID
- Client uploads **directly to S3** with that URL
- `POST /uploads/:id/complete` → tells the app the upload finished, which kicks off processing

**4. Why not proxy through Rails.** Proxying means every upload's bytes flow through an app server's memory and network for the whole transfer. That's an easy way to exhaust your worker pool during any burst of uploads.

Presigned direct uploads let the client talk straight to S3, which is built for exactly that, while your app's involvement shrinks to two cheap metadata calls.

**5. End-to-end flow.** Request presigned URL → client uploads to S3 → client notifies the app → app enqueues a background job for post-processing (image resizing, virus scanning, video transcoding, metadata extraction) → job updates the record's status (or marks it failed and deletes the S3 object if the virus scan trips) → serve the file back through a CDN with a signed, expiring URL rather than proxying downloads either.

**6. Where it breaks at scale.** Large uploads time out on flaky connections — support S3 multipart upload above a size threshold so transfers are resumable. Long-running processing (video transcoding) deserves its own worker pool so a transcode doesn't starve quick thumbnail jobs. And virus scanning itself becomes a bottleneck at volume, so treat it as its own stage with concurrency tied to the scanner's throughput.

**Trade-off to say out loud:** "I'd do presigned direct-to-S3 rather than proxying, because it keeps large uploads off my request-handling capacity — but it costs complexity: a two-step client flow instead of one POST, and I have to garbage-collect orphaned pending records where someone got a URL and never used it."

**Example**

```
POST /uploads/presign { filename: "video.mp4", content_type: "video/mp4" }
  -> { upload_url: "https://s3.../bucket", fields: {...}, blob_id: 88 }

Client --PUT bytes directly--> S3          # Rails app not in this path at all

POST /uploads/88/complete
  -> mark blob "uploaded"
  -> ProcessUploadJob.perform_later(88)
       -> virus scan -> resize/transcode -> mark "processed" (or "failed" + delete from S3)
```

---

**— Scalability & Reliability Concepts —**

### 219. When do you scale vertically vs horizontally?

**Short Answer**

- **Vertical** — give one server more CPU, RAM, or disk.
- **Horizontal** — add more servers and spread the load.

Start vertical when it's a quick single-box fix. Go horizontal when one machine hits its ceiling, or when you need redundancy against a machine dying.

**Simple Explanation**

**Vertical scaling** is a one-line ops change — upgrade the instance type, no application changes. It's the usual first move for a database primary that's CPU- or memory-bound.

Two limits: there's a biggest instance money can buy, and a bigger box is still a single point of failure.

**Horizontal scaling** fits workloads you can split across independent units — classically a Rails app server tier behind a load balancer. You need it once vertical scaling maxes out, or as soon as you need high availability, since one bigger box never gives you that.

**Example**

```
DB primary pegged at 90% CPU under query load
  -> vertical: upgrade db.r5.xlarge -> db.r5.4xlarge   (one config change)

App tier queueing requests under traffic growth
  -> horizontal: add app server instances 3 -> 8 behind the load balancer
     (also buys redundancy: losing one of eight barely dents capacity)
```

---

### 220. What do a load balancer and autoscaling actually do?

**Short Answer**

- **Load balancer** — spreads incoming traffic across multiple servers, and keeps traffic flowing if one dies.
- **Autoscaling** — automatically adds or removes those servers based on load.

**Simple Explanation**

**L4 vs L7.** An L4 (transport layer) balancer routes by IP and port without understanding HTTP — fast and simple. An L7 (application layer) balancer reads HTTP headers, paths, and cookies, so it can send `/api` to one pool and `/admin` to another, and terminate TLS. Rails apps usually sit behind L7.

**Routing algorithms:**

- **Round-robin** — cycle through servers evenly. Simplest, fine when requests cost roughly the same.
- **Least-connections** — send new traffic to whoever has the fewest active connections. Better when request durations vary a lot.
- **Consistent hashing** — always route the same key to the same server. Used for cache-friendly routing or WebSocket stickiness.

**Autoscaling** watches a metric (CPU, request queue depth, p99 latency). Crossing a threshold triggers new instances; dropping below removes them.

The **cooldown** period after each action matters: without it, you'd launch instances, re-check before they're even serving traffic, and launch far more than you need.

**Example**

```
ALB (L7) --round-robin--> [app-1, app-2, app-3]
   path /api/*  -> api-target-group
   path /admin/* -> admin-target-group

Autoscaling policy: CPU > 70% for 3 min -> +2 instances, then 5 min cooldown
                     CPU < 30% for 10 min -> -1 instance
```

---

### 221. Walk through the failure chain when a server gets overloaded, and how you prevent it.

**Short Answer**

Overload → requests queue → latency climbs → clients time out → clients retry → **more** load on an already-struggling server → it cascades outward.

Prevent it with backpressure, circuit breakers, bulkheads, and graceful degradation.

**Simple Explanation**

The chain, step by step:

1. Traffic spikes (or a downstream dependency gets slow, or a GC pause hits).
2. Requests arrive faster than they can be processed, so they pile up in a queue.
3. Average latency climbs as the queue grows.
4. Client timeouts start firing on requests that are still being processed.
5. Those clients retry, adding load to a server that's already behind.
6. The server keeps working on requests whose callers already gave up — wasting capacity on work nobody will use.
7. If this server is a dependency for something else, the same pattern repeats one layer up.

The dangerous part is step 5 and 6: the retries make the problem worse, and the wasted work means you're burning capacity for nothing.

Prevention: **backpressure** (reject before you're fully saturated), **circuit breakers** (stop calling something that's clearly broken), **bulkheads** (contain the damage), and **graceful degradation** (serve less rather than failing entirely).

**Example**

```
t0    normal load, p99 latency 120ms
t0+1m traffic spike / DB slow query -> requests start queueing
t0+2m p99 latency 4s, still climbing
t0+2m30s client timeouts (set at 3s) start firing -> clients retry
t0+3m retry volume adds to already-overloaded server -> queue grows faster
t0+4m server effectively unavailable to everyone, including healthy requests
```

---

### 222. What are circuit breakers, bulkheads, and fallback responses, and how do they work together?

**Short Answer**

- **Circuit breaker** — stop calling a dependency that's clearly failing, so you don't pile more requests onto something broken.
- **Bulkhead** — give each dependency its own resource pool, so one slow one can't starve the others.
- **Fallback** — what you return instead when the breaker is open or a call fails.

**Simple Explanation**

**Circuit breaker** has three states:

- **Closed** — calls go through normally.
- **Open** — after too many failures, calls fail instantly without even attempting the network request. That protects your capacity *and* gives the struggling dependency room to recover.
- **Half-open** — after a cooldown, a few probe requests go through. If they succeed, it closes again.

**Bulkhead** is named after ship compartments that stop one breach sinking the whole vessel. In practice it's a separate connection or thread pool per dependency — so a slow payment gateway saturating its own pool can't also block calls to an unrelated shipping API.

**Fallback** is a cached last-known-good value, a sensible default, or a visibly reduced response — render the product page without the "12 people viewing this" widget rather than returning a 500.

**Example**

```ruby
breaker = Circuitbox.circuit(:payments_api, exceptions: [Faraday::Error]) do
  threshold: 0.5, timeout_seconds: 30, sleep_window: 60
end

result = breaker.run(exception: false) { PaymentsClient.charge(order) }
result || Order.mark_pending_manual_review!(order)   # fallback, not a 500

# bulkhead: separate connection pools so payments-api never starves shipping-api
Faraday.new(url: PAYMENTS_URL) { |f| f.adapter :net_http_persistent, pool_size: 5 }
Faraday.new(url: SHIPPING_URL) { |f| f.adapter :net_http_persistent, pool_size: 5 }
```

---

### 223. What are idempotency keys, and why do retried requests need them?

**Short Answer**

An idempotency key is a unique token the client sends with a request, so if the same request is retried — network blip, double-click, client auto-retry — the server recognises it and returns the original result instead of doing the work twice.

**Simple Explanation**

HTTP says POST isn't idempotent, but real clients retry POSTs anyway. Mobile clients auto-retry on timeout. Users double-click submit buttons.

Without a guard, a retried checkout can charge a card twice or create two orders.

The fix: the client generates a unique key (a UUID) before the first attempt. The server checks a lookup table before doing anything:

- **Key seen before?** Return the stored response. Do no work.
- **New key?** Process it, then store the key with its result.

This matters anywhere a request has a real-world side effect that must not happen twice.

**Example**

```ruby
def checkout
  key = request.headers["Idempotency-Key"]
  if (existing = IdempotencyKey.find_by(key: key))
    return render json: existing.response_body, status: existing.status
  end

  order = process_checkout!(params)
  IdempotencyKey.create!(key: key, response_body: order.to_json, status: 201)
  render json: order, status: 201
end
```

---

### 224. What do logs, metrics, and distributed tracing each tell you that the others can't?

**Short Answer**

- **Logs** — exactly what happened for one specific request.
- **Metrics** — how the system is trending overall.
- **Traces** — where one request spent its time across multiple services.

Each answers a question the other two structurally can't.

**Simple Explanation**

**Logs** are detailed records of individual events. Great for "what exactly happened at 10:03:12 on this request", including error messages and stack traces. Bad at showing trends, and expensive to search at scale.

**Metrics** are numbers over time — request rate, error rate, p99 latency. Cheap to store, graph, and alert on. But a metric can tell you latency went up; it can't tell you *why* one specific request was slow.

**Distributed tracing** follows one request across every hop — frontend, API, database, downstream service — recording timing at each step as **spans**, tied together by a shared **trace ID** passed along with the request. That's the only one that answers "which specific hop ate the time".

You need all three. Metrics tell you something's wrong, traces tell you where, logs tell you what exactly.

**Example**

```
Same incident, three lenses:
  log:    [ERROR] OrderController#create: PG::QueryCanceled after 5.2s (request_id=abc123)
  metric: p99 latency for POST /orders jumped from 200ms to 4.8s at 14:02 UTC
  trace:  trace_id=abc123
            span: api gateway        [0ms   - 20ms ]
            span: orders-service     [20ms  - 5200ms]  <- the slow hop
            span:   -> inventory-db  [30ms  - 5190ms]  <- root cause: DB query
```

---

### 225. Health check vs. readiness probe vs. liveness probe — what's the difference?

**Short Answer**

- **Liveness probe** — "is this process alive enough to keep running, or should it be restarted?" Failure → kill and restart.
- **Readiness probe** — "can this instance serve traffic right now?" Failure → stop sending it traffic, but leave it running.

**Simple Explanation**

The distinction matters because the response is different.

If **liveness** fails, the orchestrator restarts the process. That's right for a genuinely stuck or deadlocked process where a restart actually fixes something.

If **readiness** fails, the orchestrator stops routing traffic there but leaves it alone. That's right for a temporary condition it can recover from — still warming up on boot, or a momentarily exhausted database pool.

Here's the key insight: restarting your process won't fix an unreachable database. But you also don't want traffic routed there while it can't serve. That's exactly the readiness/liveness split.

In practice, liveness checks are shallow ("does the process respond at all") and readiness checks are deeper ("can I actually reach my database and Redis").

**Example**

```ruby
# config/routes.rb
get "/up",    to: "health#liveness"    # just: process responds -> 200
get "/ready", to: "health#readiness"   # checks: ActiveRecord.connected?, Redis.ping
```

```yaml
livenessProbe:  { httpGet: { path: /up },    periodSeconds: 10 }
readinessProbe: { httpGet: { path: /ready }, periodSeconds: 5  }
```

---

### 226. What's a timeout budget, and how does exponential backoff with jitter prevent retry storms?

**Short Answer**

A **timeout budget** is how you divide the total time a user will tolerate across every hop in the chain, so a slow downstream can't blow the whole thing.

**Exponential backoff with jitter** spreads retries out over time, so a brief failure doesn't turn into everyone hammering the recovering service in lockstep.

**Simple Explanation**

**Timeout budget:** if a user tolerates 2 seconds for A → B → C, then C's timeout must be well under 2 seconds — leaving room for B's own work, network overhead, and possibly one retry. Setting C's timeout to 2 seconds too means a slow C burns the entire budget and A has no time left to handle the failure gracefully.

**Retry storm:** many clients' timeouts fire around the same moment (common right after a deploy). They all retry after the same fixed interval, so load spikes again exactly when the service was trying to recover — and it may never get room to recover.

**Exponential backoff** waits progressively longer between retries (200ms, 400ms, 800ms). **Jitter** adds randomness to each wait, so the retries spread into a gradual ramp instead of synchronised waves.

Both matter. Backoff alone still leaves everyone retrying at the same moments.

**Example**

```
Timeout budget:  user tolerance 2000ms
  A allocates:    1800ms for calling B
  B allocates:      800ms for calling C, keeps 1000ms for its own work + 1 retry of C
  C timeout:        350ms  (leaves room inside B's 800ms for one retry)

Backoff + jitter sequence for a failed call:
  attempt 1: fails
  attempt 2: wait 200ms  + random(0-100ms)
  attempt 3: wait 400ms  + random(0-200ms)
  attempt 4: wait 800ms  + random(0-400ms)   -> give up, surface error
```

---

### 227. What are load shedding and graceful degradation, and how do they differ?

**Short Answer**

- **Load shedding** — deliberately rejecting new requests (`503` with `Retry-After`) once you're past capacity, instead of queueing them forever.
- **Graceful degradation** — deciding in advance which features can fail without taking down the critical path.

**Simple Explanation**

An unbounded queue **feels** safer than rejecting requests, but it isn't. Everyone waits longer and longer, most time out anyway, and by then they've tied up resources the whole time. Recovery is also harder, because there's now a backlog to work through even after the root cause is fixed.

**Load shedding** says: past a defined threshold, reject immediately with `503` and `Retry-After: N`. The capacity you do have keeps serving what it can, and well-behaved clients back off.

**Graceful degradation** is the decision you make *before* an incident: if the recommendations service is down, the product page still renders without the "you might also like" widget rather than returning a 500.

The critical rule that comes out of this: make sure genuinely critical paths (checkout, login) never depend on non-critical ones, so a minor outage can't take down a major flow.

**Example**

```
Product page render:
  core: title, price, "add to cart" button      -> must succeed, no fallback
  widget: "you might also like" (recs service)   -> if down, render page without it

Overload response:
  requests_in_flight > capacity_threshold
    -> new requests get 503 + Retry-After: 5   (fast, cheap rejection)
    -> NOT: queue everything and let it time out slowly 30s later
```

---

**— Architecture Concepts —**

### 228. Monolith vs. microservices — what does splitting actually solve, and what does it cost? What is the strangler fig pattern?

**Short Answer**

Microservices split an app into independently deployable services talking over the network. You gain independent scaling and deployment; you pay in network latency, distributed transactions, and operational complexity.

The **strangler fig pattern** is how you migrate gradually: route a growing slice of traffic to new services while the monolith handles the rest.

**Simple Explanation**

A monolith is one codebase, one deploy, usually one database. Simple to reason about and fast to build — but everything scales and deploys together, even when only one piece needs it.

Microservices solve two kinds of scaling:

- **Organisational** — many teams shipping independently without blocking each other.
- **Technical** — one genuinely hot subsystem scales without dragging the rest along.

The cost is real. A function call becomes a network call, so latency and partial failure are now possible where they weren't. A single database transaction across tables becomes a multi-service saga. And you take on service discovery, more pipelines, and debugging across processes.

The **strangler fig pattern** (named after a vine that grows around a tree and gradually replaces it) is the safe migration path: put a gateway in front of the monolith, stand up a new service for one bounded piece, route just that slice of traffic to it, and repeat feature by feature — instead of a risky big-bang rewrite.

**Example**

```
Before:                          Mid-migration (strangler fig):
  clients -> monolith               clients -> gateway --/search--> new search-service
                                                        --/*-------> monolith (everything else)

Over time, more routes get peeled off the monolith one bounded context at a time.
```

---

### 229. What is the Saga pattern, and what's a compensating transaction?

**Short Answer**

A **saga** is a sequence of local transactions across multiple services, used because you can't wrap a multi-service operation in one database transaction.

If a later step fails, you run **compensating transactions** that undo the earlier ones — you can't just roll back.

**Simple Explanation**

Take an order flow across Order, Inventory, and Payment services, each with its own database. There's no cross-service ACID transaction available.

So a saga runs it as a chain of local commits: order created (pending) → inventory reserved → payment charged → order confirmed. Each step commits in its own service.

If payment fails, there's nothing spanning all three to roll back. Instead you run compensating actions: release the inventory reservation, then cancel the order. Each undo is its own explicit local transaction.

Two styles:

- **Choreography** — each service listens for the previous step's event and reacts. No central coordinator, but hard to see the whole flow in one place.
- **Orchestration** — a central coordinator calls each step and triggers compensations on failure. Easier to reason about and monitor, at the cost of a new component to build.

**Example**

```
Happy path:   OrderCreated -> InventoryReserved -> PaymentCharged -> OrderConfirmed

Failure at PaymentCharged:
  compensate: ReleaseInventoryReservation
  compensate: CancelOrder
  (no DB rollback spans services — each undo is its own explicit local transaction)
```

---

### 230. What role does an API Gateway play in a microservices architecture?

**Short Answer**

An API gateway is a single entry point in front of your services. It handles the cross-cutting concerns once — routing, authentication, rate limiting, and sometimes combining several service calls into one response.

**Simple Explanation**

- **Routing** — path or host rules send `/users` to the user service and `/orders` to the order service, so clients don't need to know your internal layout.
- **Auth** — validate the token once at the edge instead of every service reimplementing it.
- **Rate limiting** — enforce per-client limits centrally instead of duplicating limiter logic everywhere.
- **Aggregation** — a mobile screen needing data from three services can hit one gateway endpoint that fans out internally and composes the response, saving the client three round trips. When it's purpose-built for one client type, that's called a "backend for frontend".

**Example**

```
gateway routes:
  /users/*   -> user-service
  /orders/*  -> order-service
  /products/* -> catalog-service

GET /gateway/order-summary/:id   # aggregation endpoint
  -> internally calls order-service + user-service + catalog-service
  -> composes one JSON response for the mobile client
```

---

### 231. What is eventual consistency across services, and when is it acceptable?

**Short Answer**

Eventual consistency means that after a write, other parts of the system — a replica, a cache, another service's copy — may briefly show old data before catching up.

It's fine when a short delay causes no harm. It's not fine where it does, like an inventory check before a purchase.

**Simple Explanation**

Example: a user changes their display name. An Order service that keeps a denormalized copy of "customer name" gets the update a few hundred milliseconds later via an event. For that window, an order page might show the old name.

That's harmless. Nobody is hurt by a stale display name for two seconds.

Now compare that with an inventory count used to decide "can this purchase go through". Staleness there causes overselling, real money lost, and an angry customer.

So the design move is: accept eventual consistency for display and denormalized data everywhere it's cheap, and add a **strong, synchronous check at the exact point** where staleness causes real harm — the inventory reservation itself, not every place the count is displayed.

**Example**

```
t0        User service:  name "Alex" -> "Alexandra"  (committed)
t0+50ms   UserRenamed event published
t0+300ms  Order service consumes event, updates its denormalized copy
t0-t0+300ms  order detail page may still show "Alex"  <- acceptable staleness window
```

---

### 232. What is the shared-database anti-pattern in microservices?

**Short Answer**

Multiple "independent" services reading and writing the **same database** defeats the whole point of splitting them up. They're still tightly coupled through a shared schema, and one service's migration can break another.

**Simple Explanation**

The value of microservices is independent deployability.

If service B directly queries service A's tables, then A can't change its schema without coordinating a deploy with B. A migration in A can silently break B at runtime. And nobody can tell from the code which service actually owns a table.

That's monolith coupling without any of the monolith's simplicity.

The fix: each service owns its own database (or at minimum its own schema), and exposes data to others **only** through its API or through published events — never direct database access across a service boundary.

**Example**

```
Anti-pattern:                         Correct:
  user-service   \                      user-service  -- owns --> users_db
                   > same Postgres        exposes: GET /users/:id, UserUpdated event
  order-service  /                      order-service -- owns --> orders_db
                                          consumes UserUpdated to keep its own denormalized copy
```

---

### 233. Where do you put caching across a full application — browser, CDN, server-side, DB?

**Short Answer**

Cache at several layers, and pick the layer based on how **shared** and how **fresh** the data needs to be:

- **Browser** — static assets
- **CDN** — content that's the same for many users
- **Application cache** (Redis/Memcached) — expensive shared computations
- **Database** (materialized views) — expensive aggregates

The closer to the user, the faster and cheaper — but also the staler and less personalized.

**Simple Explanation**

**Browser cache** handles JS, CSS, and images via `Cache-Control`. Fastest possible, but per-browser and only right for things that essentially never change.

**CDN** caches at edge locations near users. Right for content that's identical for many people — public pages, images, non-personalized API responses. On a cache hit, your origin does nothing at all.

**Application cache** (`Rails.cache` backed by Redis) handles fragment caching and memoized expensive results — data that's costly to produce and shared across requests, but too dynamic or personalized for a CDN.

**Database-level caching** — a materialized view for an expensive aggregate that doesn't need to be real-time, like a daily rollup computed once and read many times.

**Example**

```
Request for a public blog post:
  browser cache    -> static CSS/JS, not the post body
  CDN               -> full HTML response cached at the edge, huge hit rate
  Rails.cache       -> fragment-cached "related posts" partial (shared, not fully public-cacheable)
  materialized view -> "most popular this week" ranking, refreshed hourly, not per-request
```

---

### 234. How do you design authentication and authorization for a multi-role SaaS app (RBAC)?

**Short Answer**

- **Authentication** — who are you (login, tokens, sessions).
- **Authorization** — what are you allowed to do.

**RBAC** implements authorization by assigning users to roles (admin, editor, viewer) that map to permissions, checked consistently at every action.

**Simple Explanation**

The multi-tenant wrinkle: roles are usually scoped **per organization**, not globally. Someone can be an admin in Company A's workspace and a viewer in Company B's.

That means you need a join table — `memberships` with `user_id`, `organization_id`, and `role` — not a single `role` column on the user.

Enforcement should go through one policy layer (Pundit policies or CanCanCan abilities) called from controllers. So "can this user edit this post?" is one testable method rather than `if current_user.admin? || ...` scattered across views and controllers.

Finer-grained rules ("editors can only edit posts they wrote, unless they're a senior editor") still fit naturally as extra conditions inside the same policy object.

**Example**

```
memberships
  user_id          bigint
  organization_id  bigint
  role             string  # "admin" | "editor" | "viewer"

class PostPolicy < ApplicationPolicy
  def update?
    membership.admin? || (membership.editor? && record.author == user)
  end
end
```

---

### 235. Where should rate limiting be implemented — gateway, middleware, or DB layer?

**Short Answer**

Put cheap, coarse limits as early as possible (at the gateway or edge), and finer, business-aware limits closer to the app (Rack::Attack middleware).

Database-level limits (connection caps, statement timeouts) are a safety net, not product rate limiting.

**Simple Explanation**

**Gateway / edge** — block by IP or API key before it costs any app capacity at all. Right for abuse and simple global quotas.

**Middleware** (Rack::Attack in Rails) — can key off things the edge doesn't know, like a logged-in user ID or a specific endpoint's sensitivity. "5 login attempts per 15 minutes per account" is a business rule, not generic traffic shaping, so it belongs here.

**Database layer** — not really rate limiting. Connection pool limits and statement timeouts protect the database itself if something upstream fails to limit properly.

In practice you layer them: cheap global limits at the edge, precise business limits in middleware.

**Example**

```ruby
# config/initializers/rack_attack.rb
Rack::Attack.throttle("logins/account", limit: 5, period: 15.minutes) do |req|
  req.params["email"] if req.path == "/login" && req.post?
end

Rack::Attack.throttle("api/ip", limit: 300, period: 5.minutes) do |req|
  req.ip if req.path.start_with?("/api/")
end
```

---

### 236. How do you make background jobs safe under at-least-once delivery — idempotency, retries, and dead-letter queues?

**Short Answer**

Most queues guarantee **at-least-once** delivery, not exactly-once. A worker can finish the work and crash before acknowledging, so the job runs again.

So: make handlers safe to run twice, retry with backoff up to a max, and send exhausted jobs to a **dead-letter queue** instead of retrying forever.

**Simple Explanation**

The classic example: a `ChargeCardJob` charges the card successfully, then the worker crashes before marking the job done. The job is redelivered and, without a guard, charges again.

Guard it the same way as an API idempotency key — before charging, check whether this order was already charged and no-op if so.

**Retry strategy:** exponential backoff between attempts so a transient blip doesn't hammer a struggling dependency, plus a max retry count.

**Dead-letter queue:** once retries are exhausted, the job moves there rather than disappearing. Someone can then look at *why* it kept failing — bad data, a permanently broken integration — and decide to fix and requeue, or discard it.

**Example**

```ruby
class ChargeCardJob
  include Sidekiq::Job
  sidekiq_options retry: 5, queue: "payments"

  def perform(order_id)
    order = Order.find(order_id)
    return if order.charged?              # idempotency guard against redelivery

    charge = PaymentGateway.charge(order, idempotency_key: order.idempotency_key)
    order.update!(status: :charged) if charge.success?
  end
end
# after 5 exhausted retries -> Sidekiq::DeadSet, not silently dropped
```

---

### 237. Pub/sub vs. point-to-point queues — what's the difference?

**Short Answer**

- **Point-to-point queue** — one message goes to exactly **one** consumer. That's a background job.
- **Pub/sub** — one event is broadcast to **every** interested subscriber, each processing its own copy.

**Simple Explanation**

**Point-to-point** (a Sidekiq queue, or a standard SQS queue): "resize this image". Exactly one worker should do it, and it doesn't matter which.

**Pub/sub** (SNS, Kafka topics, EventBridge): "a user signed up". A welcome-email consumer, a CRM-sync consumer, and an analytics consumer each need their own copy — and adding a fourth consumer later shouldn't require the publisher to change at all.

Quick test: if two consumers both need to react, you need pub/sub. If the work should happen exactly once, you need a queue.

**Example**

```
Point-to-point:            Pub/sub:
  queue: resize_image         topic: user.registered
    -> worker A picks it up     -> subscriber: welcome-email-service (own queue)
       (only one worker does it) -> subscriber: crm-sync-service (own queue)
                                 -> subscriber: analytics-service (own queue)
                              (each gets its own independent copy of the event)
```

---

### 238. What is the outbox pattern, and what problem does it solve?

**Short Answer**

The outbox pattern writes the event to an **outbox table in the same transaction** as your business data, then a separate process reads that table and publishes to the message broker.

It fixes the gap where "save to DB, then publish" can lose the event on a crash, or publish an event for a write that then rolls back.

**Simple Explanation**

The problem: if you save a record and then separately call `publish_event`, those two aren't atomic. A crash in between means the data saved but nobody heard about it. Or the publish succeeds and then the database write fails, so you've announced something that never happened.

The fix: in the **same transaction** as the business write, also insert a row into an `outbox_events` table. Both commit together or neither does — there's no window where one happened without the other.

Then a separate relay process (a poller, or a change-data-capture tool reading the WAL) reads unpublished outbox rows, publishes them, and marks them published.

Because the relay can safely retry, the only remaining requirement is that consumers are idempotent — an occasional duplicate delivery is fine.

**Example**

```ruby
ActiveRecord::Base.transaction do
  order = Order.create!(order_params)
  OutboxEvent.create!(event_type: "OrderCreated", payload: order.to_json)
end
# both commit together, or neither does — no lost-event window

# separate relay process, polling:
OutboxEvent.where(published_at: nil).find_each do |event|
  Broker.publish(event.event_type, event.payload)
  event.update!(published_at: Time.current)
end
```

---

### 239. What is event sourcing, and what's the real trade-off?

**Short Answer**

Instead of storing only the current state and overwriting it, event sourcing stores the **sequence of events** that led to that state. Current state is calculated by replaying them.

You get a complete audit trail and point-in-time reconstruction. You pay in replay complexity, harder schema evolution, and a real learning curve.

**Simple Explanation**

The normal model: an `orders` row with a `status` column that gets `UPDATE`d. Once updated, the old value is gone.

The event-sourced model: you append immutable events — `OrderCreated`, `ItemAdded`, `PaymentReceived`, `OrderShipped`. "Current state" is what you get by replaying all of them in order. Replaying only the events up to a given point tells you what it looked like then.

**What you gain:** a complete audit trail for free, and the ability to answer "what did this look like last Tuesday?" naturally. That fits domains that already think in terms of history — accounting ledgers, order lifecycles.

**What it costs:** every read needs either a full replay or a maintained projection kept in sync. Schema evolution is genuinely hard, because old events were written in an old shape and still have to replay correctly. And it's a much bigger mental shift than a CRUD table.

Reach for it when the audit trail is an actual product requirement — not because it sounds elegant.

**Example**

```
Event log for order #42:
  OrderCreated   { items: [] }
  ItemAdded      { sku: "A1", qty: 2 }
  PaymentReceived{ amount: 40.00 }
  OrderShipped   { carrier: "UPS" }

Folded current state (replay all events in order):
  { status: "shipped", items: [{sku: "A1", qty: 2}], paid: true, carrier: "UPS" }

State "as of" ItemAdded only (replay up to that event):
  { status: "created", items: [{sku: "A1", qty: 2}], paid: false }
```

---

### 240. Why does offset pagination degrade at scale, and why does cursor-based pagination stay stable?

**Short Answer**

`LIMIT 20 OFFSET 100000` still makes the database scan and throw away 100,000 rows. That cost grows with depth.

Cursor pagination (`WHERE id > last_seen_id LIMIT 20`) is a plain indexed range scan, so it costs the same on page 1 or page 5000 — and it doesn't shift when rows are inserted elsewhere.

**Simple Explanation**

Two problems with offset.

**Performance:** page 1 stays fast forever, but page 5000 gets steadily worse as the table grows. And those deep pages are exactly what gets hit during a viral moment or a large export.

**Consistency:** if a row is inserted or deleted while someone is paging, later pages can skip or duplicate rows. That's because "offset 100" is a *position*, and positions shift under concurrent writes.

Cursor pagination sidesteps both. Instead of "skip N rows", you say "give me rows after this specific row". That's anchored on a row's identity, not a position, so it's stable — and it's a direct indexed lookup, so it's fast at any depth.

The real cost: you lose "jump to page 5000". Cursors only support sequential paging, which is a genuine trade-off for an admin table that wants page numbers.

**Example**

```sql
-- Offset: cost grows with depth, inconsistent under concurrent writes
SELECT * FROM posts ORDER BY id LIMIT 20 OFFSET 100000;  -- scans+discards 100k rows

-- Cursor: constant cost regardless of depth
SELECT * FROM posts WHERE id > 4213000 ORDER BY id LIMIT 20;  -- indexed range scan
```

Use offset for small/bounded admin tables that need page-number jumping; use cursors for any public, high-volume, or infinite-scroll feed.

---

### 241. How do zero-downtime deploys, graceful shutdown, and migration safety fit together?

**Short Answer**

Zero-downtime deploys need three things working together:

1. **Rolling or blue-green deploys** — new instances take traffic before old ones are removed.
2. **Graceful shutdown** — a terminating instance finishes its in-flight requests instead of dropping them.
3. **Backward-compatible migrations** — because old and new code run against the same database during the rollout.

**Simple Explanation**

**Rolling deploy:** new instances boot, pass readiness checks, and start receiving traffic. Old instances stop getting *new* traffic but get a grace period to finish what they're already handling — that grace period is graceful shutdown. Skip it and every deploy drops whatever was mid-flight.

**Blue-green** is the more dramatic version: run a full second environment, verify it's healthy, cut traffic over, and keep the old one around briefly for instant rollback.

**The part people forget:** during any rolling deploy, old and new code are running simultaneously against the **same** database. So a migration that isn't backward compatible — renaming or dropping a column the old code still reads — breaks the old instances mid-deploy.

The fix is the expand/contract pattern:

1. Add the new column; deploy code that writes to both.
2. Backfill existing rows.
3. Deploy code that reads only the new column.
4. In a **later** deploy, drop the old column.

Every individual step is safe for both the old and new code to run against.

**Example**

```
Rolling deploy timeline:
  t0   new instances boot, fail readiness (still warming up)
  t1   new instances pass readiness -> LB starts routing to them
  t2   old instances deregistered from LB (no new traffic)
  t2-t2+30s  old instances finish in-flight requests (graceful shutdown), then exit

Expand/contract migration for renaming `full_name` -> `name`:
  deploy 1: add `name` column, app writes to both `full_name` and `name`
  backfill: copy `full_name` -> `name` for existing rows
  deploy 2: app reads/writes only `name` (old code no longer running)
  deploy 3 (later): drop `full_name` column
```

## Testing / RSpec

**— Testing Philosophy & Strategy —**

### 242. What's the testing pyramid, and how would you apply the ratio to a Rails app?

**Short Answer**

Lots of fast unit specs at the bottom, fewer request specs in the middle, and very few slow browser specs at the top. Roughly 70 / 20 / 10.

**Simple Explanation**

The pyramid is a shape, not a law. The point is that speed and isolation should dominate your suite.

- **Unit specs** (models, service objects, plain Ruby classes) don't touch a browser and often don't even hit the database. You can run thousands in seconds, so this is where your edge cases and branching logic belong.
- **Request specs** boot the Rails stack and hit a real HTTP endpoint. Slower, but they catch routing, serialization, and controller bugs that unit specs can't see.
- **System specs** (Capybara driving a browser) are the slowest and most fragile — timing issues, JS rendering, flaky selectors — but they're the only layer proving a user can actually click through a flow.

If you invert the pyramid, CI balloons to 30+ minutes and becomes unreliable, because one flaky browser wait dominates your feedback loop.

The goal: push each check down to the cheapest layer that can catch that bug.

**Example**

```ruby
# spec/models/invoice_spec.rb  (base of the pyramid — milliseconds, no HTTP, no browser)
RSpec.describe Invoice, type: :model do
  it "is invalid without a total" do
    invoice = build(:invoice, total: nil)
    expect(invoice).not_to be_valid
  end
end

# spec/requests/invoices_spec.rb  (middle layer — real HTTP round trip)
RSpec.describe "GET /invoices/:id", type: :request do
  it "returns the invoice as JSON" do
    invoice = create(:invoice, total: 42.00)
    get "/invoices/#{invoice.id}"
    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body)["total"]).to eq("42.0")
  end
end

# spec/system/checkout_spec.rb  (top of the pyramid — slow, full browser)
RSpec.describe "Checkout flow", type: :system, js: true do
  it "lets a customer complete a purchase" do
    visit new_checkout_path
    fill_in "Card number", with: "4242424242424242"
    click_button "Pay now"
    expect(page).to have_content("Payment successful")
  end
end
```

---

### 243. What's the difference between mocking, stubbing, and faking — and when would you use each?

**Short Answer**

- **Stub** — replaces a method with a canned return value. You don't care if it's called.
- **Mock** — a stub **plus** an assertion that it was called.
- **Fake** — a simplified working implementation of a whole dependency.

In RSpec: `allow(...).to receive(...)` is a stub; `expect(...).to receive(...)` is a mock.

**Simple Explanation**

**Stub:** "when this is called, return this". Useful for controlling an input without invoking the real dependency.

**Mock:** the fact that the call happened **is** the thing you're testing. Use it when the behavior you care about is "we notified the user" rather than a returned value.

**Fake:** a small working substitute — an in-memory payment gateway that records charges instead of calling Stripe. Useful when the collaborator's behavior matters across several calls, so hand-stubbing every method would be more work and more brittle than writing one small class.

Quick rule: stub to control an input, mock to verify an interaction, fake when the dependency has real behavior you need to simulate.

**Example**

```ruby
# Stub: I don't care if it's called, I just need a controlled return value
RSpec.describe OrderSummary do
  it "formats the tax-inclusive total" do
    allow(TaxRateService).to receive(:rate_for).and_return(0.08)
    summary = OrderSummary.new(subtotal: 100)
    expect(summary.total_with_tax).to eq(108.0)
  end
end

# Mock: the call itself is the thing under test
RSpec.describe OrderCompleter do
  it "notifies the customer when the order completes" do
    order = create(:order)
    expect(NotificationMailer).to receive(:order_completed).with(order).and_call_original
    OrderCompleter.new(order).call
  end
end

# Fake: a working substitute for a whole external dependency
class FakePaymentGateway
  attr_reader :charges
  def initialize = @charges = []
  def charge(amount:, card:) = @charges << { amount: amount, card: card } && true
end

RSpec.describe Checkout do
  it "charges the configured gateway" do
    gateway = FakePaymentGateway.new
    Checkout.new(gateway: gateway).charge(amount: 50, card: "tok_visa")
    expect(gateway.charges).to eq([{ amount: 50, card: "tok_visa" }])
  end
end
```

---

### 244. What should and shouldn't be mocked in a test suite?

**Short Answer**

**Mock** external things you don't control — payment gateways, third-party HTTP calls, email delivery.

**Don't mock** your own application classes. That ties the test to implementation details instead of behavior, and it's how a suite goes green while production is broken.

**Simple Explanation**

The question to ask is: do I own this code, and can I run it safely and quickly in a test?

Third-party APIs are slow, cost money, rate-limit you, or aren't reachable in CI. So you stub them at the boundary with WebMock or VCR.

Your own domain objects should mostly run for real. If you mock `OrderCalculator` inside a `Checkout` test, you've proven "Checkout calls OrderCalculator" but not "checkout produces the right total". If someone breaks the calculator's maths, your mocked test still passes.

Over-mocking your own code is the single most common way a test suite becomes useless.

Rule of thumb: mock at the **edges** of your system (I/O, third parties, time, randomness) and let internal collaborators talk to each other for real.

**Example**

```ruby
# Good: mock the boundary — a third-party HTTP call we don't control
RSpec.describe WeatherClient do
  it "returns the parsed temperature" do
    stub_request(:get, "https://api.weather.example/current")
      .to_return(status: 200, body: { temp_f: 72 }.to_json)

    expect(WeatherClient.new.current_temp("Austin")).to eq(72)
  end
end

# Bad: mocking your own collaborator hides real breakage
RSpec.describe Checkout do
  it "charges the correct total (BAD — over-mocked)" do
    allow_any_instance_of(OrderCalculator).to receive(:total).and_return(100)
    # This "passes" even if OrderCalculator's real tax math is broken.
  end
end

# Better: let real collaborators run, only stub the true external edge
RSpec.describe Checkout do
  it "charges the correct total" do
    order = create(:order, :with_two_line_items) # real OrderCalculator runs
    allow(PaymentGateway).to receive(:charge).and_return(success: true)

    Checkout.new(order).complete!

    expect(PaymentGateway).to have_received(:charge).with(amount: order.total, card: order.card_token)
  end
end
```

---

### 245. What are some common RSpec mistakes you've seen (or made) in a real codebase?

**Short Answer**

- Testing implementation instead of behavior
- Factories creating huge object graphs and silently slowing the suite
- Leaking global state between examples
- Over-stubbing until the test can't catch a real break
- Asserting on a mock when you could assert on the real outcome

**Simple Explanation**

**Testing implementation:** asserting a private method was called instead of checking the actual result. It breaks on every refactor even when behavior is unchanged.

**Factory bloat:** a factory whose associations each spin up their own factories, multiplied across hundreds of examples. The suite gets slower every sprint and nobody notices until CI takes 40 minutes.

**Leaking state:** a stubbed `Time.now` that's never reset, a class-level cache, a value set in one example bleeding into the next. The symptom is tests that pass alone but fail in a full run — and the failure depends on run order.

**Over-stubbing:** stubbing so many layers that the test just re-asserts your own setup. It goes green forever, including the day you ship a bug.

**Asserting on a mock:** checking `have_received(:save)` when you could check that the record actually exists. The mock proves a message was sent; it doesn't prove the right thing happened.

**Example**

```ruby
# BAD: over-stubbed — this test can never fail even if the discount logic is wrong
RSpec.describe Cart do
  it "applies the discount (BAD)" do
    allow_any_instance_of(Cart).to receive(:apply_discount).and_return(true)
    expect(Cart.new.apply_discount).to eq(true)
  end
end

# GOOD: asserts the real, observable outcome
RSpec.describe Cart do
  it "applies a 10% discount to the subtotal" do
    cart = create(:cart, subtotal: 100)
    create(:promotion, :ten_percent_off, cart: cart)

    cart.apply_discount

    expect(cart.total).to eq(90)
  end
end

# BAD: leaking global state via an unreset stub
RSpec.describe "when a promotion expires" do
  it "hides the banner" do
    allow(Time).to receive(:now).and_return(Time.new(2099, 1, 1)) # never reset!
    expect(PromotionBanner.new.visible?).to eq(false)
  end
end

# GOOD: scoped time travel that cleans up automatically
RSpec.describe "when a promotion expires" do
  it "hides the banner" do
    travel_to(Time.zone.local(2099, 1, 1)) do
      expect(PromotionBanner.new.visible?).to eq(false)
    end
  end
end
```

---

### 246. How would you debug a flaky or slow test suite in CI?

**Short Answer**

- **Flaky:** run the spec alone, then with random ordering and `--bisect` to find the hidden state dependency.
- **Slow:** run `--profile` to find the worst offenders, and attack the biggest wins first — usually factory bloat or unnecessary system specs.

**Simple Explanation**

**For flakiness**, my first move is to reproduce reliably. Run the failing spec on its own. If it passes alone but fails in the full run, it's a state leak.

`rspec --bisect` automatically narrows it down to the minimal combination of examples that triggers the failure — that's usually the fastest path to the culprit.

Common root causes: `Time.now` not reset, a `let!` record affecting a `.first`/`.last` query elsewhere, a real network call timing out intermittently, or a Capybara wait racing an async JS update.

**For slowness**, `rspec --profile 10` lists the ten slowest examples. Look for **patterns**, not one-off slow tests — usually it's system specs that didn't need to be system specs, or factories eagerly building large object graphs (`create` where `build_stubbed` would do).

Also check whether the suite runs in parallel and whether the database reset strategy per worker is efficient.

**Example**

```ruby
# Finding a flaky/order-dependent test
# $ bundle exec rspec --seed 12345 --order random
# $ bundle exec rspec --bisect --seed 12345    # narrows down to the minimal failing combination

# Profiling the slowest examples
# $ bundle exec rspec --profile 10

# A classic order-dependency bug this workflow catches:
RSpec.describe Report, type: :model do
  let!(:old_report) { create(:report, created_at: 1.year.ago) }

  it "returns the most recent report" do
    # BUG: relies on `.last` without an explicit order — passes/fails
    # depending on which other examples created Report rows first.
    expect(Report.last).to eq(old_report)
  end
end

# Fix: make the query (and therefore the test) deterministic
RSpec.describe Report, type: :model do
  it "returns the most recent report" do
    old_report = create(:report, created_at: 1.year.ago)
    recent_report = create(:report, created_at: 1.day.ago)

    expect(Report.most_recent).to eq(recent_report)
    expect(Report.most_recent).not_to eq(old_report)
  end
end
```

---

### 247. Is test coverage percentage a good metric to optimize for?

**Short Answer**

It's a useful alarm for completely untested code, but a bad target to chase.

100% line coverage only means every line **ran** at least once — not that the behavior was actually checked.

**Simple Explanation**

Coverage tools count executed lines, not assertions. You can hit 100% with tests that call every method and assert nothing meaningful.

So I treat coverage as a gap-finder — "here's a whole class with zero coverage, that's a real hole" — rather than a number to optimise.

Chasing a coverage percentage as a target produces exactly the anti-patterns above: shallow assertions written to satisfy the tool rather than catch regressions.

The question I actually care about: **if I introduce a deliberate bug, does the suite catch it?** Coverage percentage can't tell you that.

**Example**

```ruby
# This achieves 100% line coverage of Discount#apply...
class Discount
  def apply(price)
    price * (1 - rate)
  end
end

# ...but this test is worthless: it exercises every line and asserts nothing useful.
RSpec.describe Discount do
  it "applies (BAD — coverage-chasing, not behavior-verifying)" do
    expect { Discount.new(rate: 0.1).apply(100) }.not_to raise_error
  end
end

# A real behavioral test — still 100% coverage, but now it can actually fail
RSpec.describe Discount do
  it "reduces the price by the configured rate" do
    discount = Discount.new(rate: 0.1)
    expect(discount.apply(100)).to eq(90)
  end
end
```

---

**— Types of Specs —**

### 248. What are the different types of specs in a Rails/RSpec suite, and when do you reach for each?

**Short Answer**

- **Model specs** — validations, scopes, and business logic in isolation. Fast, no HTTP.
- **Request specs** — hit a real endpoint, check status and response body.
- **System specs** — drive a real browser through a full user flow.
- **Service object specs** — call a plain Ruby class directly and check its result and side effects.

**Simple Explanation**

**Model specs** exercise an Active Record model directly — validations, scopes, associations, and logic that lives on the model.

**Request specs** are the modern replacement for controller specs. They send a real HTTP request through the router and middleware and check the response. This is the right level for "does this endpoint behave correctly", including auth and serialization, without a browser.

**System specs** drive a headless browser via Capybara. They're the only level proving JavaScript and multi-page flows actually work. They're slow and comparatively brittle, so save them for critical happy paths like signup and checkout.

**Service object specs** call a plain class's `.call` directly and check the return value and side effects (records created, jobs enqueued, emails sent). As fast as model specs, but testing orchestration logic that doesn't belong on a model.

Rule of thumb: push logic as low as it'll go, use request specs to confirm the HTTP contract, and save system specs for the few flows where the browser itself is the risk.

**Example**

```ruby
# Model spec — validations/associations in isolation
RSpec.describe User, type: :model do
  it { is_expected.to validate_presence_of(:email) }
  it { is_expected.to have_many(:orders) }
end

# Request spec — the HTTP contract
RSpec.describe "POST /users", type: :request do
  it "creates a user and returns 201" do
    post "/users", params: { user: { email: "a@example.com", password: "secret123" } }
    expect(response).to have_http_status(:created)
  end
end

# System spec — a real browser flow
RSpec.describe "User signup", type: :system, js: true do
  it "lets a visitor sign up" do
    visit new_user_registration_path
    fill_in "Email", with: "a@example.com"
    fill_in "Password", with: "secret123"
    click_button "Sign up"
    expect(page).to have_content("Welcome!")
  end
end

# Service-object spec — orchestration logic, no HTTP/browser
RSpec.describe UserRegistrar do
  it "creates a user and sends a welcome email" do
    result = UserRegistrar.new(email: "a@example.com", password: "secret123").call

    expect(result).to be_success
    expect(ActionMailer::Base.deliveries.map(&:to).flatten).to include("a@example.com")
  end
end
```

---

### 249. Walk me through writing a request spec for an API endpoint, end to end.

**Short Answer**

1. Set up data with FactoryBot.
2. Send the request with the right verb, params, and headers.
3. Assert on **both** the status code **and** the shape of the JSON — not just "200 OK".

**Simple Explanation**

A good request spec checks three things:

- **Status code** — did the right thing happen? 200, 201, 422, 404?
- **Response shape** — does the JSON have the keys and types the client expects?
- **Side effects that matter at this level** — a `Location` header, a persisted record.

Avoid asserting on the entire JSON blob with one giant hardcoded hash. That breaks whenever an unrelated field is added. Assert on the specific keys the endpoint actually promises.

For authenticated endpoints, set up the auth context properly (a signed-in user, a bearer token) rather than stubbing authentication away — request specs are exactly the layer that should catch an authorization bug.

And test the unhappy paths: unauthorized, forbidden, and not-found. A spec that only covers the happy path misses most of the risk.

**Example**

```ruby
# spec/requests/api/v1/invoices_spec.rb
RSpec.describe "Api::V1::Invoices", type: :request do
  let(:user) { create(:user) }
  let(:headers) { { "Authorization" => "Bearer #{user.api_token}" } }

  describe "GET /api/v1/invoices/:id" do
    context "when the invoice belongs to the current user" do
      it "returns the invoice with a 200" do
        invoice = create(:invoice, user: user, total: 99.50)

        get "/api/v1/invoices/#{invoice.id}", headers: headers

        expect(response).to have_http_status(:ok)
        body = JSON.parse(response.body)
        expect(body).to include(
          "id" => invoice.id,
          "total" => "99.5",
          "status" => "pending"
        )
      end
    end

    context "when the invoice belongs to a different user" do
      it "returns a 404 rather than leaking the record's existence" do
        other_users_invoice = create(:invoice, user: create(:user))

        get "/api/v1/invoices/#{other_users_invoice.id}", headers: headers

        expect(response).to have_http_status(:not_found)
      end
    end

    context "without a valid auth token" do
      it "returns a 401" do
        get "/api/v1/invoices/1"
        expect(response).to have_http_status(:unauthorized)
      end
    end
  end
end
```

---

### 250. How do you test that a background job runs correctly — and what's the difference between asserting it was enqueued vs actually performing it?

**Short Answer**

- `have_enqueued_job` — when the thing you're testing is just supposed to **schedule** the job.
- `perform_enqueued_jobs` (or calling `perform_now`) — when you need to prove the job's actual side effects happen.

**Simple Explanation**

These test two different responsibilities, and mixing them up either slows your suite down or leaves a gap.

If I'm testing a controller that should **trigger** a job, I only care that it was scheduled with the right arguments. Actually running the job — with its own database writes and external calls — is redundant here and belongs in the job's own spec.

Conversely, when I'm testing the **job itself**, I want it to really run, so I can assert its real side effects: a record updated, an email sent.

The trap is doing neither properly — mocking the job's internals inside a controller spec tests nothing useful, and never testing `perform` leaves the job's actual logic unverified.

**Example**

```ruby
# Testing that the *caller* schedules the job correctly (don't run it here)
RSpec.describe OrdersController, type: :request do
  it "enqueues a receipt email job when an order is placed" do
    user = create(:user)
    sign_in user

    expect {
      post "/orders", params: { order: attributes_for(:order) }
    }.to have_enqueued_job(ReceiptMailerJob).with(user.id, kind_of(Integer))
  end
end

# Testing the job's own behavior (run it for real)
RSpec.describe ReceiptMailerJob, type: :job do
  include ActiveJob::TestHelper

  it "sends a receipt email when performed" do
    order = create(:order)

    perform_enqueued_jobs do
      ReceiptMailerJob.perform_later(order.user_id, order.id)
    end

    expect(ActionMailer::Base.deliveries.last.to).to eq([order.user.email])
  end

  # Equivalent, more direct style — skips the queue entirely
  it "marks the order as receipted" do
    order = create(:order, receipted: false)

    ReceiptMailerJob.new.perform(order.user_id, order.id)

    expect(order.reload.receipted).to eq(true)
  end
end
```

---

### 251. How do you test an integration with an external HTTP API without hitting the real network?

**Short Answer**

Stub the HTTP layer with **WebMock**, or record and replay real responses once with **VCR**.

Then add `WebMock.disable_net_connect!` so no spec can silently make a real network call.

**Simple Explanation**

Hitting the real network in a test suite is slow, flaky (the third party might be down or rate-limit you), and sometimes expensive (a real payment charge).

**WebMock** lets you stub specific requests and return canned responses — including error statuses, so you can test your error handling too.

**VCR** goes further: it records a real interaction once into a "cassette" (a YAML file) and replays it afterwards. Good for a complex API response you don't want to hand-write.

I use WebMock for simple stubs where I control the exact payload, and VCR when the real response is complex and I want it captured faithfully.

Either way, `disable_net_connect!` in `rails_helper.rb` is what actually **enforces** that no real call sneaks through.

**Example**

```ruby
# spec/rails_helper.rb
require "webmock/rspec"
WebMock.disable_net_connect!(allow_localhost: true)

# spec/services/geocoder_client_spec.rb
RSpec.describe GeocoderClient do
  it "returns coordinates for a valid address" do
    stub_request(:get, "https://geocode.example/v1/lookup")
      .with(query: { address: "1600 Amphitheatre Pkwy" })
      .to_return(
        status: 200,
        headers: { "Content-Type" => "application/json" },
        body: { lat: 37.4224, lng: -122.0842 }.to_json
      )

    result = GeocoderClient.new.lookup("1600 Amphitheatre Pkwy")

    expect(result).to eq(lat: 37.4224, lng: -122.0842)
  end

  it "raises a domain error when the API returns a 500" do
    stub_request(:get, "https://geocode.example/v1/lookup")
      .with(query: { address: "bad address" })
      .to_return(status: 500)

    expect {
      GeocoderClient.new.lookup("bad address")
    }.to raise_error(GeocoderClient::UpstreamError)
  end
end
```

---

**— RSpec Structure —**

### 252. What's the difference between `describe`, `context`, and `it`?

**Short Answer**

They're the same method — `context` is literally an alias for `describe`. The difference is convention:

- `describe` — the **thing** being tested (a class or a method).
- `context` — a **state or condition**, usually starting with "when" or "with".
- `it` — one expected behavior.

**Simple Explanation**

Following the convention matters because `rspec --format documentation` then reads like a specification.

Use `describe` for nouns: `describe User` or `describe "#total"`.

Use `context` for conditions: `context "when the user is not logged in"`.

Nesting them lets you group setup that only applies to one state, so you're not repeating it in every example.

**Example**

```ruby
RSpec.describe Order, type: :model do
  describe "#total" do
    context "when there are no line items" do
      it "returns zero" do
        order = create(:order, line_items: [])
        expect(order.total).to eq(0)
      end
    end

    context "when a discount code is applied" do
      it "subtracts the discount from the subtotal" do
        order = create(:order, :with_line_items, subtotal: 100)
        order.apply_discount_code("SAVE10")
        expect(order.total).to eq(90)
      end
    end
  end
end
# Documentation-format output reads as:
#   Order
#     #total
#       when there are no line items
#         returns zero
#       when a discount code is applied
#         subtracts the discount from the subtotal
```

---

### 253. What does `subject` give you, and when is it worth naming explicitly?

**Short Answer**

`subject` is the object being tested, which matchers like `is_expected.to` use by default.

Name it (`subject(:invoice) { ... }`) as soon as more than one example refers to it — a bare `subject` reads badly beyond one-liners.

**Simple Explanation**

If you don't define one, RSpec generates an unnamed subject from the outermost `describe` class. That's what powers the terse `it { is_expected.to be_valid }` syntax, which is great for compact validation checks.

But once examples need to **do** something with it — call methods, pass it around, reference it several times — an anonymous `subject` becomes hard to read. A reader has to scroll up to work out what it is.

Naming it gives you a self-documenting variable you can use like any `let`, while still supporting the `is_expected` shorthand.

**Example**

```ruby
RSpec.describe Invoice, type: :model do
  # Anonymous subject — great for compact one-liners
  subject { build(:invoice) }

  it { is_expected.to be_valid }
  it { is_expected.to validate_numericality_of(:total).is_greater_than(0) }
end

RSpec.describe InvoicePresenter do
  # Named subject — reads clearly across multiple examples
  subject(:presenter) { described_class.new(invoice) }
  let(:invoice) { create(:invoice, total: 1234.5, currency: "USD") }

  it "formats the total with the currency symbol" do
    expect(presenter.formatted_total).to eq("$1,234.50")
  end

  it "exposes the invoice's due date" do
    expect(presenter.due_date).to eq(invoice.due_date)
  end
end
```

---

### 254. What's the difference between `let`, `let!`, and a plain instance variable in a `before` block?

**Short Answer**

- `let` — **lazy**. The block only runs the first time an example references it, then the value is cached for that example.
- `let!` — runs the same block **before every example**, whether it's referenced or not.
- `@ivar` in a `before` block — also runs every time, but with no memoization and no typo protection.

**Simple Explanation**

`let(:user) { create(:user) }` doesn't create a user unless an example actually calls `user`. That keeps unrelated examples fast.

The flip side: a bug in a `let` block does nothing until something references it, so a broken factory can stay hidden.

`let!` forces the block to run before each example. Use it when a record needs to **exist in the database** for something else to find it — like an index action that should return "all users" — rather than being used by name.

A plain `@user = create(:user)` in a `before` block behaves like `let!` in timing, but it has no safety net: typo `@usre` and Ruby silently gives you `nil` instead of raising `NameError` the way a mistyped `let` helper would.

**Example**

```ruby
RSpec.describe "User listing" do
  # let: lazy — only created if an example references `admin`
  let(:admin) { create(:user, :admin) }

  # let!: eager — always exists in the DB, even though no example below references `everyone_user` by name
  let!(:everyone_user) { create(:user, role: "member") }

  it "includes members returned by the query, without needing to name them" do
    expect(User.members.count).to eq(1) # relies on everyone_user existing
  end

  it "creates the admin only when referenced" do
    expect(admin.role).to eq("admin") # `admin` is created here, on first reference
  end
end

# Plain instance variable — eager, no memoization guard, typo-prone
RSpec.describe "User listing (ivar style)" do
  before do
    @admin = create(:user, :admin)
  end

  it "typo silently returns nil instead of raising" do
    expect(@admni).to be_nil # no NameError — just a silently wrong assertion
  end
end
```

---

### 255. What are shared examples, and when would you reach for them?

**Short Answer**

`shared_examples` lets you write a set of assertions once and run them against every class that shares a behavior — like every model that includes an `Archivable` concern.

You pull them in with `it_behaves_like` or `include_examples`.

**Simple Explanation**

When several classes share a concern, you want to prove the behavior works everywhere it's used, without copy-pasting the same five assertions into each model's spec.

`shared_examples` defines those assertions once, usually relying on `subject` being set by whoever includes them.

- `it_behaves_like` pulls them into a **nested** context, so they can't collide with the including spec's own setup.
- `include_examples` merges them directly into the current context.

Prefer `it_behaves_like` unless you specifically need the merge.

**Example**

```ruby
# spec/support/shared_examples/archivable.rb
RSpec.shared_examples "archivable" do
  it "is not archived by default" do
    expect(subject).not_to be_archived
  end

  it "sets archived_at when archived" do
    subject.archive!
    expect(subject.archived_at).to be_present
  end

  it "excludes archived records from the default scope" do
    subject.archive!
    expect(described_class.all).not_to include(subject)
  end
end

# spec/models/project_spec.rb
RSpec.describe Project, type: :model do
  subject { create(:project) }
  it_behaves_like "archivable"
end

# spec/models/document_spec.rb
RSpec.describe Document, type: :model do
  subject { create(:document) }
  it_behaves_like "archivable"
end
```

---

### 256. What's the difference between a stub, a mock, and a spy in RSpec syntax?

**Short Answer**

- **Stub:** `allow(obj).to receive(:msg)` — no verification.
- **Mock:** `expect(obj).to receive(:msg)` — set up **before** the action, and the example fails if it never happens.
- **Spy:** `allow` first, run the code, then verify afterwards with `expect(obj).to have_received(:msg)`.

**Simple Explanation**

The real difference is **when** you declare the expectation relative to when the code runs.

With the **mock** style, you state the expectation up front. RSpec fails the example at the end if the message was never sent.

With the **spy** style, you stub first (so the code can run), invoke it, and *then* assert. That reads more naturally as arrange → act → assert, which is easier to follow in longer examples.

Both use the same machinery underneath, so it's a style choice. Many style guides prefer spies for readability.

**Example**

```ruby
RSpec.describe SubscriptionCanceler do
  # Mock style: expectation declared up front, before acting
  it "notifies billing when canceled (mock style)" do
    subscription = create(:subscription)
    expect(BillingService).to receive(:cancel).with(subscription.id)

    SubscriptionCanceler.new(subscription).call
  end

  # Spy style: stub first, act, then verify after the fact
  it "notifies billing when canceled (spy style)" do
    subscription = create(:subscription)
    allow(BillingService).to receive(:cancel)

    SubscriptionCanceler.new(subscription).call

    expect(BillingService).to have_received(:cancel).with(subscription.id)
  end

  # Plain stub: no verification at all, just controls a return value
  it "treats a failed billing cancellation as non-fatal" do
    allow(BillingService).to receive(:cancel).and_return(false)
    expect { SubscriptionCanceler.new(create(:subscription)).call }.not_to raise_error
  end
end
```

---

### 257. What's the difference between a plain `double` and a verifying double like `instance_double`, and why does it matter?

**Short Answer**

- `double("thing")` — accepts **any** method you stub, even ones that don't exist on the real class.
- `instance_double(RealClass)` — checks that the stubbed methods actually exist on `RealClass`, with a compatible signature.

Verifying doubles stop your mocks from drifting away from the real API.

**Simple Explanation**

The biggest risk of mocking is **mock drift**. You write `double(:gateway, charge: true)`, someone later renames the real method to `process_charge`, and your test keeps passing for a method that no longer exists — while production breaks.

`instance_double` closes that gap. It loads the real class and raises immediately if you stub a method it doesn't have, or call it with the wrong number of arguments.

You keep most of the speed benefit of mocking without losing the safety net. So as a default, prefer `instance_double` / `class_double` over a bare `double` whenever the real class is loaded in your test environment.

**Example**

```ruby
RSpec.describe RefundProcessor do
  it "calls charge on the gateway (verifying double catches drift)" do
    gateway = instance_double(PaymentGateway, refund: true)
    # If PaymentGateway doesn't define #refund, or refund's real arity
    # doesn't match, this raises immediately instead of silently passing.

    RefundProcessor.new(gateway: gateway).process(amount: 20)

    expect(gateway).to have_received(:refund).with(amount: 20)
  end

  it "a plain double would NOT catch a renamed method (illustration, not recommended)" do
    gateway = double("gateway", refund: true) # accepts ANY stub, real or not
    expect(gateway.refund(amount: 20)).to eq(true) # passes even if #refund never existed
  end
end
```

---

### 258. What are custom RSpec matchers, and when would you write one?

**Short Answer**

Write a custom matcher when the same non-trivial assertion — and its failure message — is repeated across many specs.

You get a readable one-liner plus a **useful failure message** instead of a generic diff.

**Simple Explanation**

Built-in matchers cover most cases. But domain-specific checks — "this JSON matches our error envelope", "this money value is within a cent", "this user has exactly these permissions" — often get re-implemented ad hoc, with inconsistent and unhelpful failure output.

A custom matcher centralises that logic once, gives it a name that reads naturally (`expect(response).to have_error_code(:invalid_input)`), and — importantly — lets you define a `failure_message` that says exactly what was wrong.

I reach for one after I've copy-pasted the same multi-line assertion a third time.

**Example**

```ruby
# spec/support/matchers/have_error_code.rb
RSpec::Matchers.define :have_error_code do |expected_code|
  match do |response|
    @body = JSON.parse(response.body)
    @body["error"]["code"] == expected_code.to_s
  end

  failure_message do |response|
    "expected response to have error code #{expected_code.inspect}, " \
      "but got #{@body&.dig("error", "code").inspect} (status #{response.status})"
  end
end

# spec/requests/api/v1/orders_spec.rb
RSpec.describe "POST /api/v1/orders", type: :request do
  it "returns a structured error for an invalid payload" do
    post "/api/v1/orders", params: { order: { quantity: -1 } }

    expect(response).to have_error_code(:invalid_input)
  end
end
```

---

### 259. What's the difference between `before(:each)` and `before(:all)`/`before(:suite)`, and what are the risks of `before(:all)`?

**Short Answer**

- `before(:each)` — runs fresh before every example, and **is rolled back** with the example's transaction. This is what you want almost always.
- `before(:all)` — runs once per describe block, **outside** the transaction, so anything it creates or changes **leaks between examples**.
- `before(:suite)` — runs once for the whole test run.

**Simple Explanation**

`before(:each)` is the safe default. Each example is wrapped in its own database transaction, so anything created is rolled back automatically with essentially no teardown cost.

`before(:all)` is tempting for expensive setup, but it runs **outside** that per-example transaction. So records it creates aren't rolled back, and if one example mutates that shared state, later examples see the mutation. That produces order-dependent, confusing failures.

`before(:suite)` runs once for the entire run — right for genuinely global, immutable setup like an initial database clean.

My default: avoid `before(:all)` for anything touching the database. The small speed win is rarely worth the flakiness.

**Example**

```ruby
# Risky: before(:all) state leaks and mutates across examples
RSpec.describe Wallet, type: :model do
  before(:all) do
    @wallet = create(:wallet, balance: 100)
  end

  it "debits the wallet" do
    @wallet.debit(30)
    expect(@wallet.balance).to eq(70)
  end

  it "has the original balance (FAILS — previous example's debit leaked)" do
    expect(@wallet.balance).to eq(100) # fails: still 70, not rolled back
  end
end

# Safe: before(:each) gives every example a clean, isolated wallet
RSpec.describe Wallet, type: :model do
  before(:each) do
    @wallet = create(:wallet, balance: 100)
  end

  it "debits the wallet" do
    @wallet.debit(30)
    expect(@wallet.balance).to eq(70)
  end

  it "has the original balance" do
    expect(@wallet.balance).to eq(100) # passes: fresh wallet, transaction rolled back after each
  end
end
```

---

**— FactoryBot —**

### 260. Why do most modern Rails teams prefer FactoryBot over fixtures?

**Short Answer**

Factories are readable Ruby, composable, and create exactly the data one example needs. A shared fixture file couples every spec to one big dataset.

**Simple Explanation**

Fixtures load a fixed dataset once and share it across the whole suite. That's fast, but every spec is implicitly tied to it — change a fixture to fix one test and you can silently break an unrelated one. Fixtures also skip model callbacks and validations by default, so they can drift out of sync with real invariants.

FactoryBot generates data at the point of use, in plain Ruby, with the exact attributes that example cares about visible **right there in the spec**:

```ruby
create(:user, :admin, email: "specific@example.com")
```

That makes each spec self-contained and easy to read on its own. The cost is speed — real inserts and real validations are slower than pre-loaded fixtures.

**Example**

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "secret123" }
    role { "member" }
  end
end

# The data an example needs is visible right in the spec — no jumping to a YAML file
RSpec.describe "Admin dashboard access" do
  it "denies access to non-admins" do
    member = create(:user, role: "member")
    sign_in member
    get admin_dashboard_path
    expect(response).to have_http_status(:forbidden)
  end
end
```

---

### 261. What's the difference between `build`, `create`, and `build_stubbed`?

**Short Answer**

- `build` — creates the object in memory. No database write.
- `create` — saves it to the database, running validations and callbacks.
- `build_stubbed` — fakes a saved object (it has an ID and `persisted?` is true) but **never touches the database**. Fastest.

**Simple Explanation**

`create` is slowest but most realistic. Use it when the code under test does its own database lookup, or when callbacks matter.

`build` skips the insert but still runs validations if you call `.valid?`. Perfect for testing validation logic itself.

`build_stubbed` is the performance tool. It assigns a fake ID and reports itself as persisted, so code that reads attributes or checks `persisted?` works — but no SQL runs at all.

Use it for fast unit specs of things like presenters and serializers, where you need something that *looks* like a real record but never needs to be found by a query.

**Example**

```ruby
RSpec.describe InvoicePresenter do
  # build_stubbed: fastest — no DB hit, but has a realistic id and persisted? == true
  it "formats a stubbed invoice without touching the database" do
    invoice = build_stubbed(:invoice, total: 100)
    expect(invoice.persisted?).to eq(true)
    expect(InvoicePresenter.new(invoice).formatted_total).to eq("$100.00")
  end
end

RSpec.describe Invoice, type: :model do
  # build: in-memory, validations run, but never saved
  it "is invalid with a negative total" do
    invoice = build(:invoice, total: -1)
    expect(invoice).not_to be_valid
  end

  # create: fully persisted — needed when the code under test queries the DB
  it "is findable by Invoice.find after creation" do
    invoice = create(:invoice)
    expect(Invoice.find(invoice.id)).to eq(invoice)
  end
end
```

---

### 262. How do traits work in FactoryBot, and how would you compose more than one on a single factory call?

**Short Answer**

A `trait` is a named variation of a factory that you opt into: `create(:user, :admin)`.

You can stack several: `create(:user, :admin, :suspended)`. They're applied in order, so later traits can override earlier ones.

**Simple Explanation**

Without traits, you end up defining a separate factory for every combination — `:admin_user`, `:suspended_user`, `:suspended_admin_user` — which multiplies fast.

Traits let you define small composable pieces once and mix them per example, right at the call site. That's far more readable than a pile of factory names, and it keeps the base factory minimal.

Traits can also set up associations or run `after(:create)` hooks, not just set attributes.

**Example**

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "secret123" }
    role { "member" }
    suspended_at { nil }

    trait :admin do
      role { "admin" }
    end

    trait :suspended do
      suspended_at { 1.day.ago }
    end

    trait :with_orders do
      after(:create) { |user| create_list(:order, 2, user: user) }
    end
  end
end

RSpec.describe "Admin panel access" do
  it "blocks a suspended admin even though they'd normally have access" do
    # composing two traits on one call
    admin = create(:user, :admin, :suspended)

    sign_in admin
    get admin_dashboard_path

    expect(response).to have_http_status(:forbidden)
  end

  it "shows the user's order history" do
    user = create(:user, :with_orders)
    expect(user.orders.count).to eq(2)
  end
end
```

---

### 263. How do you avoid factory creation becoming an N+1 problem that silently slows the whole suite down?

**Short Answer**

Keep base factories minimal — only associate what's genuinely **required** for the record to be valid. Make everything else opt-in via traits, and periodically run `--profile` to catch factories that have quietly grown expensive.

**Simple Explanation**

The problem creeps in gradually. A factory's association is defined as `association :account`, which always creates a real Account. That Account factory later grows its own associations, which grow theirs. Now creating one `:user` quietly cascades into six extra inserts nobody asked for.

Multiply that by thousands of examples and the suite crawls.

Four habits that prevent it:

1. Only associate what the object **requires** to be valid, not everything it could have.
2. Let examples create extra related records explicitly when they need them.
3. Use `build_stubbed` when the spec doesn't need anything persisted.
4. Run `rspec --profile` occasionally to catch a factory that's grown heavy.

**Example**

```ruby
# BAD: every :order pulls in a fully persisted graph nobody asked for
FactoryBot.define do
  factory :order do
    association :user           # creates a full :user...
    association :warehouse      # ...and a full :warehouse...
    association :shipping_plan  # ...and a full :shipping_plan, every single time
  end
end

# BETTER: only the truly required association is eager; extras are opt-in
FactoryBot.define do
  factory :order do
    user # required for a valid order — but reuse an existing one where possible
    warehouse { nil }        # optional association: nil unless a spec needs it
    shipping_plan { nil }

    trait :with_shipping_plan do
      association :shipping_plan
    end
  end
end

# The spec asks explicitly for what it needs, instead of paying for it every time
RSpec.describe ShippingCalculator do
  it "uses the order's shipping plan when present" do
    order = create(:order, :with_shipping_plan)
    expect(ShippingCalculator.new(order).cost).to be_positive
  end

  it "falls back to standard shipping without a plan" do
    order = create(:order) # no warehouse/shipping_plan created — fast
    expect(ShippingCalculator.new(order).cost).to eq(ShippingCalculator::STANDARD_RATE)
  end
end
```

---

**— Test Isolation & Time —**

### 264. How does RSpec achieve database isolation between examples, and when do you need `DatabaseCleaner` instead of transactional fixtures?

**Short Answer**

By default RSpec wraps each example in a **database transaction** and rolls it back afterwards. Fast and clean.

That breaks for JavaScript system specs, because the browser talks to your app over a real HTTP server using a **different database connection** — which can't see your uncommitted transaction. Those need `DatabaseCleaner` with a truncation strategy.

**Simple Explanation**

Transactional fixtures work because the test code and the application code share one connection and one open transaction. Anything created disappears on rollback with essentially zero teardown cost.

With a `js: true` system spec, the browser driver hits your app through an actual server running in a separate thread, with its **own** connection. It can't see data sitting inside your uncommitted transaction, so pages render as if the records don't exist.

`DatabaseCleaner` with `:truncation` fixes this by actually committing the data and then deleting all rows between examples. It's slower — it physically clears tables instead of rolling back — which is exactly why you only switch strategies for the specs that need it.

**Example**

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.use_transactional_fixtures = true # fast default for non-JS specs

  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
  end

  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
  end

  config.before(:each, js: true) do
    DatabaseCleaner.strategy = :truncation # needed: browser thread has its own connection
  end

  config.before(:each) { DatabaseCleaner.start }
  config.after(:each) { DatabaseCleaner.clean }
end

# spec/system/checkout_spec.rb
RSpec.describe "Checkout", type: :system, js: true do
  it "shows items added via a real browser session" do
    product = create(:product, name: "Widget") # must be truncation-strategy visible to the browser thread
    visit root_path
    click_link "Widget"
    expect(page).to have_content("Widget")
  end
end
```

---

### 265. Why might an `after_commit` callback not fire in a spec, and how do you fix it?

**Short Answer**

Because transactional fixtures roll the transaction back instead of committing it, and `after_commit` only fires on a real commit.

Modern Rails (5+) handles the common case for you. Where it still bites, turn transactional tests off for that specific spec and clean up with truncation.

**Simple Explanation**

`after_commit` exists precisely so code only runs once data is safely committed. But in a test, nothing is ever really committed — it's created, used, then rolled back.

Since Rails 5, `ActiveRecord::TestFixtures` fires `after_commit` callbacks at the point the transaction *would* have committed, so most specs just work.

Where it still causes trouble: a spec wrapped in a manual nested transaction, a gem that doesn't hook into that mechanism, or a case needing a genuinely durable commit visible to another connection or process.

For those, set `self.use_transactional_tests = false` for that example and clean up with `DatabaseCleaner` truncation, so a real commit actually happens.

**Example**

```ruby
class Order < ApplicationRecord
  after_commit :enqueue_confirmation_job, on: :create
  def enqueue_confirmation_job = OrderConfirmationJob.perform_later(id)
end

# Works in modern Rails (5+) even under transactional fixtures —
# ActiveRecord::TestFixtures fires after_commit at the right point automatically.
RSpec.describe Order, type: :model do
  it "enqueues a confirmation job after commit" do
    expect {
      create(:order)
    }.to have_enqueued_job(OrderConfirmationJob)
  end
end

# For a case that truly needs a real, durable commit (e.g. visible to another
# process/connection), disable transactional fixtures for that example:
RSpec.describe Order, type: :model do
  it "is visible to a separate DB connection after commit", :truncate_db do
    order = create(:order)
    expect(ActiveRecord::Base.connection_pool.with_connection { |c|
      c.select_value("SELECT COUNT(*) FROM orders WHERE id = #{order.id}")
    }).to eq(1)
  end
end

# spec/rails_helper.rb
RSpec.configure do |config|
  config.before(:each, :truncate_db) { self.use_transactional_tests = false }
  config.after(:each, :truncate_db) { DatabaseCleaner.clean_with(:truncation) }
end
```

---

### 266. How do you test time-dependent code, and why does hardcoding `Time.now` calls make it harder than it should be?

**Short Answer**

Use `travel_to`, `travel`, or `freeze_time` from `ActiveSupport::Testing::TimeHelpers` (built into Rails — no extra gem needed). They move the whole process's idea of "now" for a block, then restore it automatically.

The reason it's harder than it should be: application code that calls `Time.now` directly instead of `Time.current`.

**Simple Explanation**

Time-dependent logic — subscription expiry, business-hours checks, date-range reports — is a classic source of flaky tests, because correctness depends on **when** the test happens to run.

`travel_to(some_time) { ... }` freezes `Time.current`, `Time.now`, `Date.today`, and `DateTime.now` for the block and restores the real clock afterwards. No manual cleanup, no leaking into the next example.

`freeze_time` is the shorthand when you just need a stable "now" rather than a specific date.

The catch: this only works cleanly if your code consistently asks the framework for the time. Code that computes a value once at class-load time can't be affected by any test helper.

That's why "always use `Time.current`, never bare `Time.now`" is worth treating as a lint-level rule — it keeps time-dependent code testable.

**Example**

```ruby
class Subscription < ApplicationRecord
  def expired?
    expires_at < Time.current # framework-aware; travel_to affects this
  end
end

RSpec.describe Subscription, type: :model do
  it "is expired once the expiration date has passed" do
    subscription = create(:subscription, expires_at: Time.zone.local(2026, 1, 1))

    travel_to(Time.zone.local(2026, 1, 2)) do
      expect(subscription.expired?).to eq(true)
    end

    # clock automatically restored here — no leakage into later examples
  end

  it "is not expired before the expiration date" do
    subscription = create(:subscription, expires_at: Time.zone.local(2026, 1, 1))

    travel_to(Time.zone.local(2025, 12, 31)) do
      expect(subscription.expired?).to eq(false)
    end
  end
end

# freeze_time is shorthand for travel_to(Time.current) — useful when you just
# need a stable "now" rather than moving to a specific point:
RSpec.describe AuditLog do
  it "records the creation timestamp" do
    freeze_time do
      log = create(:audit_log)
      expect(log.created_at).to eq(Time.current)
    end
  end
end
```

---

**— Process —**

### 267. How do you approach structuring a PR from a testing-discipline standpoint?

**Short Answer**

Tests ship in the **same PR** as the behavior they cover — never "tests coming later". And the PR should be small enough that a reviewer can see the new tests actually exercise the new code.

**Simple Explanation**

A PR that adds a feature without tests puts the reviewer in a bad spot: block it and create friction, or trust that tests will show up later (they usually don't).

I write the test alongside the implementation — often test-first for anything with real branching — so the PR tells a complete story: here's the behavior, here's the proof, here's the edge case I thought about.

I keep commits scoped so implementation and its tests land together, rather than "all code" then "all tests" as separate commits, which makes `git bisect` and review both less useful.

And I say explicitly in the description what **isn't** covered, rather than leaving it ambiguous.

**Example**

```ruby
# A PR adding "orders can be cancelled within 24 hours" ships with its edge cases
# covered in the same diff, not as a follow-up:

RSpec.describe Order, type: :model do
  describe "#cancellable?" do
    it "is true within 24 hours of placement" do
      order = create(:order, placed_at: 23.hours.ago)
      expect(order.cancellable?).to eq(true)
    end

    it "is false after 24 hours" do
      order = create(:order, placed_at: 25.hours.ago)
      expect(order.cancellable?).to eq(false)
    end

    it "is false once the order has already shipped, regardless of timing" do
      order = create(:order, placed_at: 1.hour.ago, status: "shipped")
      expect(order.cancellable?).to eq(false)
    end
  end
end
```

---

### 268. What do you look for first in a code review?

**Short Answer**

**Correctness and real test coverage first.** Style comes last — a linter should own that.

A beautifully formatted PR with a subtle bug is a worse outcome than an ugly one that's correct and proven.

**Simple Explanation**

My review order:

1. **Does it actually do what the description claims**, and what edge cases does it miss? Nil handling, empty collections, authorization boundaries, concurrent access.
2. **Are the tests real?** Do they assert on behavior, and would they fail if the logic were subtly wrong?
3. **Does it fit the existing architecture**, or does it introduce a new pattern that should be discussed first?

Only then do I comment on naming or style, and I mark those clearly as non-blocking ("nit:") so they don't hold up a correct change.

A gut-check I apply to the tests: **if I mentally revert the implementation to the broken behavior, would at least one test in this diff go red?** If not, the test isn't testing anything.

**Example**

```ruby
# A reviewer applying "would this test catch a real regression?" to a submitted spec:

# BAD — passes regardless of whether the discount logic is right or wrong
it "applies a discount" do
  order = create(:order)
  expect { order.apply_discount("SAVE10") }.not_to raise_error
end

# GOOD — would actually fail if the discount math regressed
it "applies a discount" do
  order = create(:order, subtotal: 100)
  order.apply_discount("SAVE10")
  expect(order.total).to eq(90)
end
```

---

### 269. How do you handle merge conflicts in generated files — migrations, `schema.rb`, `Gemfile.lock`?

**Short Answer**

**Regenerate, don't hand-merge.**

- **Migrations** — rename one file's timestamp so they apply in a sane order.
- **`schema.rb`** — take either side, then run `db:migrate` to regenerate it.
- **`Gemfile.lock`** — fix the conflict in `Gemfile`, then run `bundle install`.

**Simple Explanation**

All three are **generated** files. Hand-editing conflict markers risks producing something syntactically valid but semantically wrong — a `schema.rb` that doesn't match what running the migrations would actually produce.

**Migrations:** don't merge migration *content*. Migrations are an ordered, append-only log, so just bump one file's timestamp so both apply in order.

**`schema.rb`:** accept either version as a starting point, then run `bin/rails db:migrate` to regenerate it truthfully from the migration files.

**`Gemfile.lock`:** resolve the real conflict in `Gemfile` (the human-authored source), then delete the lockfile's conflict markers and run `bundle install`. Never hand-edit version numbers in the lockfile.

**Example**

```bash
# Migration timestamp collision after a rebase:
# both branches added a migration at 20260115120000 — rename one forward
mv db/migrate/20260115120000_add_status_to_orders.rb \
   db/migrate/20260115120500_add_status_to_orders.rb
# then update the class's version-derived name only if needed, and re-run:
bin/rails db:migrate

# schema.rb conflict — don't hand-merge column diffs, regenerate it:
git checkout --ours db/schema.rb   # or --theirs, doesn't matter which as a starting point
bin/rails db:migrate                # regenerates schema.rb truthfully from the migrations

# Gemfile.lock conflict — resolve Gemfile by hand, regenerate the lock:
# 1. fix conflict markers in Gemfile
# 2. remove the lockfile's conflict markers (or just delete Gemfile.lock)
bundle install                      # regenerates a consistent Gemfile.lock
```

---

### 270. What should a CI pipeline check before allowing a merge, and how does that relate to keeping the test suite meaningful?

**Short Answer**

At minimum: **linting**, the **full test suite**, and a **security/dependency scan** — all required, and fast enough that people don't start ignoring red builds.

**Simple Explanation**

Each gate protects something specific:

- **Linting** (RuboCop) catches style and consistency issues cheaply, before a human reviewer has to.
- **The test suite** is the actual correctness gate — and it's only as good as the specs feeding it, which is why suite health matters as a real engineering concern.
- **Security scanning** (Brakeman for code vulnerabilities, `bundler-audit` or Dependabot for gems with known CVEs) catches things code review typically misses.

The connection to keeping the suite meaningful is **trust**. A gate people trust — green means safe, red means something's actually wrong — only stays trustworthy if flaky tests get fixed rather than silenced with `skip`, and slow specs get addressed rather than people merging on a stale CI run out of impatience.

A pipeline that's routinely red for unrelated reasons trains everyone to ignore it, which defeats the point entirely.

**Example**

```yaml
# .github/workflows/ci.yml (abbreviated)
jobs:
  lint:
    steps:
      - run: bundle exec rubocop

  test:
    steps:
      - run: bundle exec rspec --profile 10 # surfaces slow specs on every run, not just when someone remembers to check

  security:
    steps:
      - run: bundle exec brakeman --no-pager
      - run: bundle exec bundler-audit check --update
```

```ruby
# Keeping the suite meaningful: never silently skip a flaky spec without a tracked reason
RSpec.describe PaymentsReconciler do
  # BAD: silently disables coverage with no accountability
  # xit "reconciles mismatched charges" do ... end

  # BETTER: skipped with a reason and a tracked follow-up, so it can't be forgotten
  it "reconciles mismatched charges", skip: "flaky on shared runner, see JIRA-4821" do
    # ...
  end
end
```

---

### 271. How do you debug a production incident you can't reproduce locally?

**Short Answer**

Start with **observability, not guessing**: logs, APM traces, and error-tracker context to narrow down where and when it happens.

Then try to reproduce with **production-like data**, since production bugs are very often data-shape bugs.

Only as a last resort, add targeted logging and ship it to gather real signal — don't ship a speculative fix.

**Simple Explanation**

The instinct to start changing code locally is usually wrong when you can't reproduce the bug — you're guessing blind.

**First, gather evidence:** application logs around the reported time, APM traces for the failing request showing exactly which line or query it died on, and error-tracker context (params, user, request ID). That usually narrows "something's broken" to a specific method.

**Then reproduce with realistic data.** Production bugs are very often about data you don't have locally — a `nil` that "can't happen" but does, a record in a state your happy path never creates, unusual user input. A sanitized production snapshot or a staging environment with representative data is often the difference between reproducing it and not.

**If you still can't reproduce it**, add a narrow, low-risk log line or metric at the suspected spot — not a speculative fix — ship it behind a flag or to a canary, and let production traffic tell you what's happening.

Fixing blind is how you ship a second bug on top of the first. And once you find the real cause, it gets a regression test.

**Example**

```ruby
# Last-resort instrumentation: narrow, low-risk, removable — not a speculative fix
class RefundProcessor
  def process(order)
    if order.total.nil?
      # Seen in prod error tracker but not reproducible locally — add
      # targeted context before assuming a fix, ship behind a flag.
      Rails.logger.warn(
        "RefundProcessor: order #{order.id} has nil total; " \
        "status=#{order.status} source=#{order.source.inspect}"
      )
      Honeybadger.notify("RefundProcessor: nil total encountered", context: { order_id: order.id })
    end

    # ... existing logic, unchanged, so no speculative behavior change ships yet
  end
end

# Once the log/trace data reveals the real cause, the fix gets a real regression test:
RSpec.describe RefundProcessor do
  it "handles an order imported from the legacy system with a nil total" do
    legacy_order = create(:order, total: nil, source: "legacy_import")
    expect { RefundProcessor.new.process(legacy_order) }.not_to raise_error
  end
end
```

## Docker

**— Images & Containers —**

### 272. What's the difference between a Docker image and a container?

**Short Answer**

- **Image** — the blueprint. Read-only, layered, built from a Dockerfile.
- **Container** — a running instance of that image, with its own writable layer, process, and network.

**Simple Explanation**

Think of the image as a class and the container as an object created from it.

The image bundles your code, the Ruby runtime, gems, and OS packages into read-only layers. It lives in a registry (Docker Hub, ECR).

You can start many containers from the same image. Each gets its own thin writable layer on top, so one container writing to `/tmp` doesn't affect another, plus its own process tree and network interface.

So "rebuild the image" means regenerate the blueprint. "Restart the container" means stop and start an instance without touching the blueprint.

**Example**

```bash
# Build the image once — produces an immutable, tagged artifact
docker build -t myapp/web:1.4.0 .

# Start two independent containers from the *same* image
docker run -d --name web1 -p 3000:3000 myapp/web:1.4.0
docker run -d --name web2 -p 3001:3000 myapp/web:1.4.0

# Each container has its own writable layer and process —
# killing web1 does not affect web2 or the underlying image
docker rm -f web1
docker images        # the image myapp/web:1.4.0 is still there
```

### 273. Walk through a Dockerfile for a Rails app, line by line.

**Short Answer**

- `FROM` — the base image to start from.
- `WORKDIR` — sets the working directory for everything that follows.
- `COPY` — brings files into the image.
- `RUN` — executes a command **at build time** and bakes the result into a layer.
- `EXPOSE` — documents the port (it doesn't actually publish it).
- `ENTRYPOINT` / `CMD` — what runs when the container starts.

**Simple Explanation**

Each instruction adds a layer.

`FROM ruby:3.3-slim` starts from a minimal Debian image with Ruby already installed, rather than building Ruby yourself.

`WORKDIR /app` is like `cd /app` for every following instruction, and creates the directory if needed.

`COPY` moves files from your build context (the folder you ran `docker build` in) into the image.

`RUN` is where `bundle install` and `rails assets:precompile` happen — during the build, not at startup.

`EXPOSE 3000` is documentation only. Actually publishing the port happens with `-p` on `docker run` or `ports:` in Compose.

`ENTRYPOINT` is the fixed program that always runs; `CMD` provides default arguments that `docker run` can override.

**Example**

```dockerfile
# syntax=docker/dockerfile:1
FROM ruby:3.3.4-slim AS base

# Runtime OS packages: libpq5 for the `pg` gem, curl for HEALTHCHECK,
# nodejs only if you still shell out to it for asset compilation
RUN apt-get update -qq && apt-get install -y --no-install-recommends \
      libpq5 \
      curl \
      nodejs \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Gemfiles copied first so `bundle install` is cached across builds (Q6)
COPY Gemfile Gemfile.lock ./

RUN apt-get update -qq && apt-get install -y --no-install-recommends \
      build-essential libpq-dev git \
    && bundle config set --local deployment 'true' \
    && bundle config set --local without 'development test' \
    && bundle install --jobs 4 --retry 3 \
    && apt-get remove -y build-essential libpq-dev git \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/*

# Now bring in the rest of the app
COPY . .

RUN bundle exec bootsnap precompile app/ lib/ \
    && SECRET_KEY_BASE=dummy RAILS_ENV=production bundle exec rails assets:precompile

EXPOSE 3000

ENTRYPOINT ["bin/docker-entrypoint"]
CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

### 274. What does a Rails `docker-entrypoint` script do, and why not just put everything in `CMD`?

**Short Answer**

The entrypoint script does the one-time setup every container needs — clearing a stale `server.pid`, waiting for the database, running migrations — then hands off to whatever command was actually requested with `exec "$@"`.

Putting it in `CMD` would mean duplicating it for every way you start the image.

**Simple Explanation**

The same image often runs three different ways: as the web server, as a `rails console`, and as a Sidekiq worker. All three still need the stale PID file cleared and the database ready.

An entrypoint script runs first, does that shared work, and finishes with `exec "$@"` — which **replaces** the shell process with whatever `CMD` (or your `docker run` override) asked for.

That `exec` matters: it means signals like `SIGTERM` reach your Rails process directly, instead of being swallowed by a wrapper shell. Without it, graceful shutdown breaks.

**Example**

```bash
#!/bin/bash
# bin/docker-entrypoint
set -e

# Rails writes a PID file on boot; a leftover one from a previous
# (uncleanly stopped) container makes `rails server` refuse to start
rm -f /app/tmp/pids/server.pid

# Don't just assume the DB container is "up" — wait until Postgres is
# actually accepting connections (see Q14 on depends_on's limits)
until pg_isready -h "${DATABASE_HOST:-postgres}" -U "${DATABASE_USER:-postgres}" -q; do
  echo "Postgres is unavailable - sleeping"
  sleep 1
done

bundle exec rails db:prepare

# Hand off to whatever CMD (or docker run override) was requested,
# replacing this shell so PID 1 is the real Rails/Sidekiq process
exec "$@"
```

### 275. Why should a Rails container run as a non-root user?

**Short Answer**

If your app is compromised, running as root inside the container gives an attacker far more power — including on the host, since containers share the host kernel.

A dedicated unprivileged user limits the damage.

**Simple Explanation**

By default a container process runs as `root` unless the Dockerfile says otherwise.

Container isolation is weaker than a virtual machine boundary — it's namespaces and cgroups on a **shared kernel**, not separate hardware. So if an attacker finds a container escape, or can write to a mounted host path, root inside the container often means meaningful privileges outside it.

Creating an app-specific user and switching with `USER` before the app runs means a compromised Rails process can only touch files you explicitly gave it access to.

**Example**

```dockerfile
# Create an unprivileged user/group for the app
RUN groupadd --system rails && useradd --system --gid rails --create-home rails

WORKDIR /app
COPY --chown=rails:rails . .

USER rails

CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

### 276. What does `HEALTHCHECK` do in a Dockerfile, and how does it fit a Rails app?

**Short Answer**

`HEALTHCHECK` tells Docker how to check whether the app **inside** the container is actually working — not just whether the process is running.

Rails 7.1+ ships a `/up` endpoint built for exactly this.

**Simple Explanation**

A container can be "up" — its main process alive — while the app inside is deadlocked, still booting, or unable to reach the database.

`HEALTHCHECK` runs a command on an interval. If it fails enough times in a row, Docker marks the container `unhealthy`. Compose (`condition: service_healthy`) and orchestrators use that status to hold back traffic or restart it.

Rails' built-in `/up` route returns `200` if the app booted without raising, which is exactly what a lightweight liveness check needs.

The `--start-period` option matters too: it gives the app time to boot before failures start counting against it.

**Example**

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3000/up || exit 1
```

**— Build Performance & Image Size —**

### 277. Why copy `Gemfile`/`Gemfile.lock` and run `bundle install` before copying the rest of the app?

**Short Answer**

Docker caches each layer and only rebuilds it when its inputs change.

By copying **only** `Gemfile` and `Gemfile.lock` before `bundle install`, that expensive step is only re-run when your dependencies actually change — not on every code edit.

**Simple Explanation**

Docker builds layer by layer, and before running each instruction it checks whether the inputs are identical to a previous build. If so, it reuses the cached layer.

`bundle install` can take minutes when it has to compile native extensions like `pg` or `nokogiri`.

If you `COPY . .` (the whole app) **before** running `bundle install`, then **any** code change — even a one-line controller edit — invalidates that layer and forces a full gem reinstall every single build.

Copying just the Gemfiles first means the `bundle install` layer only breaks when a gem actually changes. Day-to-day rebuilds then skip straight to the fast `COPY . .` step.

**Example**

```dockerfile
# Slow: any file change anywhere in the app busts the bundle install cache
# COPY . .
# RUN bundle install

# Fast: bundle install is only re-run when Gemfile/Gemfile.lock change
COPY Gemfile Gemfile.lock ./
RUN bundle install --jobs 4 --retry 3

# Now app code changes only invalidate layers from here down
COPY . .
```

### 278. What is a multi-stage Docker build, and why use one for a Rails app?

**Short Answer**

A multi-stage build compiles gems in one throwaway "builder" stage, then copies **only the finished artifacts** into a slim runtime image that never contains the compiler toolchain.

**Simple Explanation**

Gems like `pg` and `nokogiri` need a compiler and dev headers to build. But your production container doesn't need those **after** the gems are compiled.

Shipping them is pure waste: bigger images, slower pulls and deploys, and a larger attack surface (a compiler is a handy tool for an attacker who gets code execution).

A multi-stage Dockerfile has several `FROM` lines, each starting a new stage. A later stage uses `COPY --from=builder` to take just what it needs — the compiled gems and precompiled assets — without any of the tools used to produce them.

**Example**

```dockerfile
# syntax=docker/dockerfile:1

# ---- Stage 1: build gems and precompiled assets ----
FROM ruby:3.3.4-slim AS builder

RUN apt-get update -qq && apt-get install -y --no-install-recommends \
      build-essential libpq-dev git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment 'true' \
    && bundle config set --local without 'development test' \
    && bundle install --jobs 4 --retry 3

COPY . .
RUN SECRET_KEY_BASE=dummy RAILS_ENV=production bundle exec rails assets:precompile

# ---- Stage 2: slim runtime image, no compiler toolchain ----
FROM ruby:3.3.4-slim AS runtime

RUN apt-get update -qq && apt-get install -y --no-install-recommends \
      libpq5 curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /usr/local/bundle /usr/local/bundle
COPY --from=builder /app /app

EXPOSE 3000
ENTRYPOINT ["bin/docker-entrypoint"]
CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

### 279. How do you reduce the size of a Rails Docker image?

**Short Answer**

1. Start from a slim base image.
2. Add a `.dockerignore` so junk never enters the build context.
3. Combine related `RUN` commands into **one layer**.
4. Remove build-only packages **in the same `RUN`** that installed them.

**Simple Explanation**

Bigger images take longer to build, push, pull, and start.

`ruby:3.3-slim` (Debian, minus docs and manuals) is a good default. `alpine` is smaller but uses `musl` instead of `glibc`, which occasionally breaks native gem compilation — many Rails teams decide the size saving isn't worth the debugging.

A `.dockerignore` keeps `.git`, `log/`, `tmp/`, `node_modules/`, and local `.env` files out of both the build context (faster builds) and the image (smaller, and it stops secrets being baked in accidentally).

The **single `RUN`** point is the one people miss. Each `RUN` creates a layer. If you install build tools in one `RUN` and remove them in a later one, the earlier layer **still contains them** in the image history — so the image is just as big, even though the files look deleted. Removing them in the same `RUN` is what actually saves space.

**Example**

```
# .dockerignore
.git
.gitignore
log/*
tmp/*
node_modules/
spec/
test/
.env
.env.*
README.md
coverage/
storage/*
```

```dockerfile
# Bad: two layers — the build-essential weight persists even after removal
# RUN apt-get install -y build-essential libpq-dev
# RUN bundle install
# RUN apt-get remove -y build-essential libpq-dev

# Good: one layer — build tools never persist in the image history
RUN apt-get update -qq && apt-get install -y --no-install-recommends \
      build-essential libpq-dev \
    && bundle install --jobs 4 --retry 3 \
    && apt-get remove -y build-essential libpq-dev \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/*
```

### 280. How do you tag and push a Rails image to a registry like Amazon ECR?

**Short Answer**

Build the image, tag it with the registry URI (including a commit SHA), authenticate the Docker CLI, and push.

**Simple Explanation**

A registry is a versioned store for images, addressed as `<registry-host>/<repository>:<tag>`.

Tagging with `latest` is fine locally, but for deploys you want an **immutable, traceable** tag. The git SHA is the usual choice because it ties an image directly to the exact code that produced it.

That makes rollback simple: redeploy a known-good tag, rather than guessing which `latest` was good.

**Example**

```bash
# Authenticate Docker against your ECR registry
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build and tag with the git SHA for traceability
GIT_SHA=$(git rev-parse --short HEAD)
docker build -t myapp/web:$GIT_SHA .
docker tag myapp/web:$GIT_SHA \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp-web:$GIT_SHA

# Push
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp-web:$GIT_SHA
```

**— CMD, Volumes & Networking —**

### 281. What's the difference between `CMD` and `ENTRYPOINT`, and when do you use both together?

**Short Answer**

- `CMD` alone — a default command that `docker run` completely replaces.
- `ENTRYPOINT` — always runs, and anything you pass on `docker run` becomes its **arguments** rather than replacing it.

Use both together: `ENTRYPOINT` for shared setup, `CMD` for the default thing to run.

**Simple Explanation**

If the Dockerfile only sets `CMD`, then `docker run myimage rails console` replaces it entirely.

If it sets `ENTRYPOINT`, that program always runs, and arguments after the image name are passed to it.

The common Rails pattern is `ENTRYPOINT ["bin/docker-entrypoint"]` doing shared boot work and finishing with `exec "$@"`, paired with `CMD ["bin/rails", "server", "-b", "0.0.0.0"]` as the default.

So you can run the same image as a web server, a console, or a Sidekiq worker — and the entrypoint setup runs every time, without duplicating it.

**Example**

```dockerfile
ENTRYPOINT ["bin/docker-entrypoint"]
CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

```bash
# Uses the default CMD -> runs the entrypoint, then `rails server -b 0.0.0.0`
docker run myapp/web:1.4.0

# Overrides CMD only -> entrypoint still runs its setup, then execs this instead
docker run myapp/web:1.4.0 bundle exec sidekiq

docker run -it myapp/web:1.4.0 rails console
```

### 282. Docker volumes vs bind mounts — when do you use each for a Rails app?

**Short Answer**

- **Named volume** — Docker-managed storage. Right for durable state like a Postgres data directory.
- **Bind mount** — maps a folder on your host straight into the container. Right for live-reloading your source code in development.

**Simple Explanation**

A **named volume** lives in Docker's own storage area and survives `docker-compose down` (unless you pass `-v`). It's decoupled from any one container's lifecycle, which is exactly what a database's files need.

A **bind mount** points at a real folder on your machine, so edits in your editor are instantly visible inside the container. That's how `rails server` picks up code changes without a rebuild.

You wouldn't bind-mount a database's data directory (you don't want that tied to a host path), and you wouldn't use a named volume for source code you're actively editing (you'd lose live reload).

One common trick: mount your source with a bind mount, **and** mount a named volume over the gem directory, so the host mount doesn't shadow the gems installed inside the image.

**Example**

```yaml
services:
  web:
    build: .
    volumes:
      # Bind mount: host source code, live-editable, mounted straight in
      - .:/app
      # Named volume: keep installed gems inside the image/container,
      # not shadowed by the host bind mount above
      - bundle_data:/usr/local/bundle
    ports:
      - "3000:3000"

  postgres:
    image: postgres:16
    volumes:
      # Named volume: Postgres owns this, survives container recreation
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
  bundle_data:
```

### 283. How do a Rails container and a Postgres container find each other over the network?

**Short Answer**

Docker Compose puts all services on a shared network and gives each one a DNS name matching its **service name**.

So Rails connects to Postgres using the hostname `postgres`, not an IP.

**Simple Explanation**

When you run `docker-compose up`, Compose creates a private network for the project, attaches every service to it, and runs an embedded DNS server that resolves each service name to its current container IP.

That means `database.yml` can just say `host: postgres`. Even if the container restarts and gets a new IP, the name still resolves.

Containers on the same network can reach each other on any port the target listens on internally. `ports:` is only needed to reach a container from **outside** Docker — like your laptop's browser.

**Example**

```yaml
# config/database.yml
production:
  adapter: postgresql
  host: postgres          # resolves via Compose's internal DNS
  port: 5432
  username: <%= ENV["DATABASE_USER"] %>
  password: <%= ENV["DATABASE_PASSWORD"] %>
  database: myapp_production
```

```bash
# Confirm both containers share a network and can resolve each other
docker network inspect myapp_default --format '{{range .Containers}}{{.Name}} {{end}}'

docker-compose exec web bash -c "getent hosts postgres"
# => 172.19.0.3      postgres
```

**— Docker Compose —**

### 284. Write a `docker-compose.yml` for a Rails app with Postgres, Redis, and Sidekiq.

**Short Answer**

Define one service per process: `web`, `sidekiq`, `postgres`, `redis`.

`web` and `sidekiq` build from the **same image** with different `command:` overrides, since they run the same codebase. Keep the database's data in a named volume.

**Simple Explanation**

Each entry under `services:` is a separate container.

`web` runs the Puma server and publishes port 3000 so you can reach it from your browser. `sidekiq` runs the job processor and needs **no** published ports, since nothing connects to it directly — but it needs the same gems and the same access to Postgres and Redis, which is why it uses the same build.

Share configuration through `environment:` and `env_file:`, wire up startup order with `depends_on`, and put Postgres's data in a named volume so it survives container recreation.

**Example**

```yaml
services:
  web:
    build: .
    command: bin/rails server -b 0.0.0.0
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - bundle_data:/usr/local/bundle
    environment:
      RAILS_ENV: development
      DATABASE_HOST: postgres
      REDIS_URL: redis://redis:6379/0
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  sidekiq:
    build: .
    command: bundle exec sidekiq
    volumes:
      - .:/app
      - bundle_data:/usr/local/bundle
    environment:
      RAILS_ENV: development
      DATABASE_HOST: postgres
      REDIS_URL: redis://redis:6379/0
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: myapp
      POSTGRES_DB: myapp_development
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  pgdata:
  redis_data:
  bundle_data:
```

### 285. What does `depends_on` guarantee, and what does it not?

**Short Answer**

`depends_on` only guarantees the dependency's container was **started** — not that Postgres is ready to accept connections.

For that you need a `healthcheck` plus `condition: service_healthy`, or a retry loop in your entrypoint.

**Simple Explanation**

Compose starts containers in dependency order, but "container started" and "database ready" are different moments. Postgres takes a couple of seconds after startup to finish initialising.

So Rails can boot, try to connect, and crash with "connection refused" in that gap. Classic flaky-startup bug: works most of the time, then randomly fails on a slower CI runner.

Two fixes, and I'd use both:

1. A `healthcheck` on the Postgres service (`pg_isready`) combined with `depends_on: condition: service_healthy`. Compose then won't start `web` until the check passes.
2. A retry loop in the entrypoint script as a backstop — useful because plain `docker run`, outside Compose, doesn't honour health conditions at all.

**Example**

```yaml
services:
  web:
    depends_on:
      postgres:
        condition: service_healthy   # waits for the healthcheck, not just "started"

  postgres:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 5s
      timeout: 3s
      retries: 5
```

```bash
# Belt-and-suspenders: the entrypoint script also retries, so the app
# is safe even outside Compose (e.g. `docker run` in CI, or ECS)
until pg_isready -h "$DATABASE_HOST" -q; do sleep 1; done
```

### 286. When do you need `docker-compose up --build` instead of plain `docker-compose up`?

**Short Answer**

`--build` forces Compose to rebuild the image before starting. Without it, Compose reuses whatever image already exists — even if the Dockerfile or Gemfile changed.

**Simple Explanation**

`docker-compose up` only builds automatically the **first** time, when there's no image to start from. After that it happily starts stale images forever.

So if you pull a branch that added a gem or changed the Dockerfile, plain `up` boots the old image and your new gem is missing — because those changes only take effect at build time, not through the bind-mounted source code.

In practice many teams just always use `--build` in development to avoid that whole class of "works on my machine" confusion. Layer caching usually makes it fast anyway.

**Example**

```bash
# Reuses whatever image is already built — fast, but can run stale code
docker-compose up

# Forces a rebuild first — needed after Dockerfile/Gemfile changes
docker-compose up --build

# Equivalent two-step version
docker-compose build
docker-compose up
```

### 287. How do you pass secrets into Compose services safely?

**Short Answer**

Put secrets in a gitignored `.env` file and reference them with `env_file:` or `environment:`.

**Never** bake them into the image with `ENV` or a `COPY`'d credentials file — anyone who can pull the image can extract them.

**Simple Explanation**

Compose automatically reads a `.env` file for variable substitution in `docker-compose.yml`, and `env_file:` injects a file's contents as environment variables **at container start**.

The important distinction is *when*. A secret set with `ENV` in the Dockerfile becomes part of an image layer. Image layers are cached, pushed to registries, and inspectable with `docker history` — so that secret is effectively public to anyone with image access, and rotating it means rebuilding and redeploying.

Runtime environment variables are supplied fresh each time a container starts and never become part of the image.

**Example**

```
# .env  (gitignored — never commit this file)
DATABASE_PASSWORD=supersecret
RAILS_MASTER_KEY=abc123...
```

```yaml
services:
  web:
    build: .
    env_file:
      - .env          # injected at container start, not baked into the image
    environment:
      DATABASE_URL: postgres://myapp:${DATABASE_PASSWORD}@postgres:5432/myapp_development
```

```dockerfile
# Never do this — the value becomes a permanent, extractable image layer
# ENV RAILS_MASTER_KEY=abc123...
```

### 288. Should you run Postgres/Redis in a container for production, or just for local dev?

**Short Answer**

Containerized Postgres and Redis are great for **local development and CI** — disposable and reproducible.

For **production**, most teams use a managed service (RDS, ElastiCache), because backups, patching, and failover are hard to get right and expensive to get wrong.

**Simple Explanation**

In development you *want* the database to be easy to destroy and recreate. Nobody cares about losing local seed data.

In production the calculus flips. You need point-in-time backups, automated security patching, replication for high availability, and monitoring. A managed service provides all of that out of the box.

Running your own containerized Postgres in production isn't impossible, but it means your team owns backup verification, failover orchestration, and storage durability — work that usually isn't the best use of a Rails team's time.

It's very common to run **stateless** pieces (the Rails app, Sidekiq) in containers via ECS or Kubernetes, while the stateful data stores stay managed services outside the container platform.

**Example**

```yaml
# docker-compose.yml — fine for local dev: disposable, reproducible
services:
  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data   # gone if you run `down -v`

volumes:
  pgdata:
```

```bash
# config/database.yml (production) — points at a managed RDS instance,
# not a container, for backups/failover/patching handled by AWS
# host: myapp-prod.abc123xyz.us-east-1.rds.amazonaws.com
```

### 289. A container won't start (or keeps restarting) — how do you debug it?

**Short Answer**

1. `docker ps -a` — see the exit code.
2. `docker logs <container>` — read the actual error.
3. `docker exec -it <container> sh` — poke around inside, if it stays up long enough.

**Simple Explanation**

`docker logs` shows stdout and stderr from the main process. For Rails, that's usually where the boot error is — a missing `RAILS_MASTER_KEY`, a pending migration, a syntax error.

`docker ps -a` (the `-a` includes stopped containers) shows the exit code in the STATUS column:

- `Exited (1)` — a generic app error.
- `Exited (137)` — killed, often out of memory.
- `Exited (0)` — exited cleanly, which is suspicious for a server that should run forever.

If the container dies too fast to `exec` into, run it with the entrypoint overridden to a shell (`--entrypoint sh`) so you can step through the boot sequence by hand.

Note that slim images often don't have `bash`, so use `sh`.

**Example**

```bash
# See exit codes for all containers, including stopped ones
docker ps -a

# Tail logs for the crash reason
docker logs -f myapp_web_1

# Get a shell inside a running container (slim images often lack bash)
docker exec -it myapp_web_1 sh

# If it crashes before you can exec in, override the entrypoint
# to get a shell and run the boot steps manually
docker run -it --entrypoint sh myapp/web:1.4.0
```

**— Beyond Compose —**

### 290. When do you outgrow Docker Compose and need an orchestrator like Kubernetes?

**Short Answer**

Compose manages containers on **one host**, with no auto-healing, rolling deploys, or scheduling across machines.

You need an orchestrator (Kubernetes, ECS) once you need multiple hosts, zero-downtime rolling deploys, automatic restarts when a node dies, or real autoscaling.

**Simple Explanation**

Compose is great for defining a set of related containers, but it assumes they all run on one Docker daemon on one machine. There's no concept of "if this host dies, run these somewhere else".

Kubernetes (or AWS ECS) is built for a fleet. It schedules containers across nodes, restarts or reschedules them automatically on failure, supports rolling and canary deploys gated on health checks, and can autoscale based on load.

The trade-off is genuine added complexity — manifests, cluster management, more moving parts. So it's a real trade-off, not a straight upgrade. A single-host app with modest traffic often doesn't need it. A service that must survive machine failures and scale elastically usually does.

**Example**

```bash
# Compose: single host, no cross-machine scheduling, no auto-healing
docker-compose up -d

# Kubernetes: describes desired state; the cluster scheduler figures out
# which of many nodes to run pods on, and restarts/reschedules on failure
kubectl apply -f deployment.yaml
kubectl get pods -o wide   # spread across multiple nodes/AZs
```

## AWS / Cloud

**— Core Building Blocks —**

### 291. What are the core AWS building blocks a Rails app typically uses, and how do they fit together?

**Short Answer**

- **Compute** runs the app (EC2, ECS/Fargate, or App Runner).
- **S3** stores uploads and static assets.
- **RDS** runs managed Postgres.
- **CloudFront** is the CDN that caches and serves content close to users.

**Simple Explanation**

Follow a request through:

A browser hits **CloudFront** first. Static, fingerprinted assets (compiled JS/CSS, images) are served from its cache or from S3 behind it.

Dynamic requests fall through to a **load balancer** in front of your compute layer — plain EC2 instances you manage, ECS/Fargate running your Docker image without you managing servers, or App Runner, which turns a container image directly into a running autoscaled HTTPS service with minimal config.

That compute layer talks to **RDS** for the database and **ElastiCache** for Redis.

None of this is Rails-specific — it's the same shape as any containerized web app. Knowing which piece does what is what lets you reason about an incident or a cost review.

**Example**

```
Browser
   │
   ▼
CloudFront (CDN, caches static assets + can cache API responses)
   │
   ├── S3 (compiled assets, user uploads) — served directly for static paths
   │
   ▼
Application Load Balancer (health-checks /up, routes to healthy targets)
   │
   ▼
ECS/Fargate tasks running the Rails container (see Q22)
   │
   ├── RDS Postgres (managed primary DB)
   └── ElastiCache Redis (cache store + Sidekiq broker)
```

### 292. What's a health check endpoint for, and why does a load balancer/orchestrator need one?

**Short Answer**

A health check endpoint is a lightweight route the infrastructure hits repeatedly to decide whether an instance should receive traffic.

Without one, a load balancer can only tell whether the TCP port is open — not whether the app behind it actually works.

**Simple Explanation**

A process can be running with its port open while the app is still booting, deadlocked, or unable to reach its database. A raw TCP check can't see any of that.

An HTTP health check hits a real route and looks for a `200`, which means the framework is alive and (depending what the route checks) able to reach its dependencies.

If a target fails, the load balancer stops routing to it. Combined with an auto-scaling group, the unhealthy instance can be terminated and replaced automatically.

Rails 7.1+ generates a `/up` route for exactly this.

**Example**

```ruby
# config/routes.rb (Rails 7.1+ generates this by default)
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check
end
```

```yaml
# ALB target group health check settings (CloudFormation-style)
HealthCheckPath: /up
HealthCheckIntervalSeconds: 15
HealthyThresholdCount: 2
UnhealthyThresholdCount: 3
Matcher:
  HttpCode: "200"
```

### 293. ECS with Fargate vs. the EC2 launch type — what's the difference, and which fits a Rails app?

**Short Answer**

Both run the same ECS task definitions. The difference is who manages the servers:

- **Fargate** — serverless. AWS provisions the compute per task. Simpler, costs more per task.
- **EC2 launch type** — you run and manage a fleet of EC2 instances yourself. More work, cheaper at scale.

**Simple Explanation**

A **task definition** is ECS's version of a `docker-compose` service entry: image, CPU and memory, environment variables, port mappings.

With **Fargate**, you never see a server. You ask for "0.5 vCPU, 1GB RAM" and AWS finds the capacity. Simpler to operate and scales cleanly.

With the **EC2 launch type**, you run EC2 instances and ECS schedules tasks onto them. That means patching AMIs and managing an auto-scaling group underneath your containers — but it's cheaper at scale, lets you use instance types Fargate doesn't support, and lets you pack many small tasks onto fewer instances.

Most Rails teams start on Fargate for the simplicity and only move to EC2-backed ECS once cost or a specific hardware need justifies the extra work.

**Example**

```yaml
# Simplified ECS task definition (CloudFormation-style YAML)
Family: myapp-web
RequiresCompatibilities: [FARGATE]
Cpu: "512"
Memory: "1024"
NetworkMode: awsvpc
ContainerDefinitions:
  - Name: web
    Image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp-web:abc1234
    PortMappings:
      - ContainerPort: 3000
    Environment:
      - Name: RAILS_ENV
        Value: production
    Secrets:
      - Name: DATABASE_URL
        ValueFrom: arn:aws:secretsmanager:us-east-1:123456789012:secret:myapp/database_url
```

### 294. When does AWS Lambda actually fit a Rails-adjacent workload, and when doesn't it?

**Short Answer**

Lambda fits **short, event-triggered** work — like generating a thumbnail when a file lands in S3.

It doesn't fit the main Rails app, which is a long-running stateful process that doesn't suit cold starts, execution time limits, or per-invocation isolation.

**Simple Explanation**

Lambda runs your code in response to an event and then shuts the environment down. There's no persistent process holding a warm database connection pool the way Puma or Sidekiq does, and a "cold start" adds latency when a new environment has to spin up.

That's a poor fit for a Rails app serving continuous web traffic.

It's a great fit for something self-contained and bursty: a user uploads a photo directly to S3, which fires an `ObjectCreated` event, which triggers a small function that generates and stores a thumbnail. No Rails boot, no framework overhead, no worker tied up.

Some teams do run Rails on Lambda via adapters, but it's a niche choice, not the default.

**Example**

```ruby
# lambda/thumbnail_generator.rb — triggered by an S3 ObjectCreated event
require "aws-sdk-s3"
require "image_processing/vips"

def handler(event:, context:)
  record = event["Records"].first
  bucket = record["s3"]["bucket"]["name"]
  key    = record["s3"]["object"]["key"]

  s3 = Aws::S3::Client.new
  original = s3.get_object(bucket: bucket, key: key).body

  thumbnail = ImageProcessing::Vips
    .source(original)
    .resize_to_limit(200, 200)
    .call

  s3.put_object(
    bucket: bucket,
    key: key.sub("uploads/", "thumbnails/"),
    body: File.read(thumbnail.path)
  )
end
```

**— Storage, Caching & Messaging —**

### 295. What are S3 storage classes, and how do presigned URLs fit in?

**Short Answer**

**Storage classes** trade retrieval speed and cost against storage cost:

- **Standard** — frequently accessed files.
- **Standard-IA** — rarely read, but needed instantly when they are. Cheaper storage, plus a retrieval fee.
- **Glacier** — archival. Very cheap storage, retrieval takes minutes to hours.

**Presigned URLs** let a browser upload or download directly to/from S3, so the file never passes through your Rails servers.

**Simple Explanation**

Standard is the default for active data — recent uploads, live assets.

Standard-IA suits things like completed-order invoices: rarely opened, but must be available instantly on the rare occasion someone asks.

Glacier suits compliance or audit data you're required to keep for years but essentially never read.

You automate the transitions with a **lifecycle rule** — move objects to Standard-IA after 30 days, Glacier after a year — rather than doing it by hand.

A **presigned URL** is a time-limited signed URL your app generates, granting temporary permission to `PUT` or `GET` one specific object. That lets large uploads skip your app servers entirely.

**Example**

```ruby
# Generate a presigned upload URL for direct-to-S3 uploads
s3 = Aws::S3::Resource.new(region: "us-east-1")
presigned_url = s3.bucket("myapp-uploads")
                   .object("uploads/#{SecureRandom.uuid}/avatar.jpg")
                   .presigned_url(:put, expires_in: 300)
```

```yaml
# S3 lifecycle rule: auto-transition older objects to cheaper storage classes
Rules:
  - Id: archive-old-uploads
    Status: Enabled
    Transitions:
      - Days: 30
        StorageClass: STANDARD_IA
      - Days: 365
        StorageClass: GLACIER
```

### 296. What is ElastiCache, and where does it fit for a Rails app?

**Short Answer**

ElastiCache is AWS-managed Redis (or Memcached). AWS handles patching, backups, replication, and failover, so you just point Rails at an endpoint.

**Simple Explanation**

The same Redis your `docker-compose.yml` runs in a container for development needs real care in production: persistence configuration, memory eviction policy, failover if the node dies, security patching.

ElastiCache is that Redis, run by AWS. You pick a node type and optionally a Multi-AZ replication group, and AWS handles the operational burden behind a stable endpoint.

A Rails app typically points two things at it: `config.cache_store` for `Rails.cache`, and Sidekiq's broker connection.

Sometimes both share one cluster in a smaller app. But once either workload grows, split them — a cache is fine to evict under memory pressure, a Sidekiq queue absolutely isn't.

**Example**

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV.fetch("ELASTICACHE_REDIS_URL"),  # e.g. myapp-cache.abc123.ng.0001.use1.cache.amazonaws.com:6379
  expires_in: 1.day
}
```

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch("ELASTICACHE_SIDEKIQ_URL") }
end
```

### 297. What are SQS and SNS, and when would a Rails app reach for them instead of (or alongside) Sidekiq+Redis?

**Short Answer**

- **SQS** — a managed queue. One message, one consumer.
- **SNS** — managed pub/sub. One message fanned out to many subscribers.

Reach for them when a job needs to cross service boundaries, be consumed by something that isn't Rails, or survive independently of your own Redis.

**Simple Explanation**

Sidekiq plus Redis is the default for in-app jobs, because the producer and consumer are the same codebase.

**SQS** shines when you want a durable queue decoupled from any particular Redis instance's uptime, or when the consumer isn't Rails at all — a Lambda function, say.

**SNS** adds fan-out. Publish one "order.placed" event to a topic, and multiple independent subscribers each get their own copy, without the publisher knowing who's listening.

A common pattern is **SNS fanning out to multiple SQS queues** — one per consumer — so each consumer processes at its own pace with its own retry and dead-letter behavior.

Rails apps can consume SQS directly with gems like `shoryuken`, running alongside — not instead of — Sidekiq for internal jobs.

**Example**

```ruby
# Publishing an event that fans out via SNS to multiple SQS-backed consumers
sns = Aws::SNS::Client.new
sns.publish(
  topic_arn: "arn:aws:sns:us-east-1:123456789012:order-placed",
  message: { order_id: order.id, total_cents: order.total_cents }.to_json
)
```

```ruby
# app/workers/order_notification_worker.rb — a Shoryuken consumer for one
# of the SQS queues subscribed to that SNS topic
class OrderNotificationWorker
  include Shoryuken::Worker
  shoryuken_options queue: "order-placed-email-queue", auto_delete: true

  def perform(sqs_msg, body)
    payload = JSON.parse(body)
    OrderMailer.confirmation(payload["order_id"]).deliver_now
  end
end
```

**— Observability & DNS —**

### 298. What would you alarm on for a Rails app in CloudWatch?

**Short Answer**

At minimum four things, each catching a different failure:

1. **HTTP error rate** (5xx from the load balancer)
2. **p99 latency**
3. **Background job queue depth**
4. **CPU and memory** on your tasks

**Simple Explanation**

CloudWatch is AWS's metrics, logs, and alarms service. Resources like load balancers, ECS tasks, and RDS publish metrics automatically, and you can push custom application metrics too.

- **Error rate** catches the app actively failing requests — the most direct "users are having a bad time" signal.
- **p99 latency** (the worst 1% of requests, which an average completely hides) catches degradation before it becomes outright errors. A slow query or a saturated connection pool shows up here first.
- **Queue depth** catches jobs backing up faster than workers drain them. This is silent until it isn't — the web tier can look perfectly healthy while background work falls further behind.
- **CPU and memory** catch resource exhaustion before a task gets OOM-killed or scaling can't keep up.

**Example**

```yaml
# CloudFormation-style CloudWatch alarm on ALB 5xx rate
AlarmName: myapp-prod-5xx-error-rate
MetricName: HTTPCode_Target_5XX_Count
Namespace: AWS/ApplicationELB
Statistic: Sum
Period: 60
EvaluationPeriods: 3
Threshold: 10
ComparisonOperator: GreaterThanThreshold
AlarmActions:
  - arn:aws:sns:us-east-1:123456789012:pagerduty-critical
```

### 299. How does Route 53 factor into a DNS-based migration or cutover?

**Short Answer**

Lower the record's TTL **well before** the cutover so caches expire quickly, then use weighted routing to shift traffic gradually, or failover routing to switch automatically on a health check failure.

But be honest: DNS is slow and best-effort, not an instant switch.

**Simple Explanation**

A DNS record's TTL tells resolvers how long they may cache the answer. If your record has a 24-hour TTL and you change it, some clients keep using the old answer for up to a day.

So before a planned cutover you lower the TTL to something like 60 seconds **days in advance**, wait for the old long TTL to expire out of caches everywhere, and only then flip.

**Weighted routing** lets you assign weights to multiple records for the same name (90% old stack, 10% new) and dial the split up over time — a DNS-level canary.

**Failover routing** pairs a primary and secondary with health checks, switching automatically when the primary fails.

The honest caveat: propagation is never instant. Some resolvers ignore TTLs, and any connection a client already has open doesn't care about a DNS change until it reconnects. Treat DNS failover as a minutes-scale mechanism, not a sub-second one.

**Example**

```bash
# Days before cutover: lower TTL so it has time to actually propagate
aws route53 change-resource-record-sets --hosted-zone-id Z123 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "203.0.113.10"}]
      }
    }]
  }'
```

```json
{
  "Name": "app.example.com",
  "Type": "A",
  "SetIdentifier": "new-stack",
  "Weight": 10,
  "TTL": 60,
  "ResourceRecords": [{"Value": "203.0.113.20"}]
}
```

### 300. What's the cache invalidation strategy for CloudFront in front of a Rails app?

**Short Answer**

For **fingerprinted assets** (Rails' digested filenames), you never need to invalidate — a new deploy produces a new URL, so stale cache is impossible.

You only issue an explicit CloudFront invalidation for content that must live at a **fixed** URL.

**Simple Explanation**

Rails' asset pipeline appends a content hash to compiled filenames (`application-4f2a9c1b.js`). If the contents change, the filename changes.

That means you can set a very long `Cache-Control: max-age=31536000, immutable` safely. The **old** URL never changes meaning, and a new deploy simply references a **new** URL that CloudFront has never seen and will fetch fresh.

That sidesteps invalidation entirely for most assets.

What's left is content that must stay at a fixed URL — a `sitemap.xml`, a CMS-managed landing page, a PDF replaced in place. There, nothing about the URL signals "this is new", so you explicitly invalidate that path.

**Example**

```erb
<%# fingerprinted asset URL changes automatically on every content change %>
<%= javascript_include_tag "application" %>
<%# => /assets/application-4f2a9c1b3e8d.js — old deploys' URLs stay valid forever %>
```

```bash
# Explicit invalidation only needed for fixed-URL, mutable content
aws cloudfront create-invalidation \
  --distribution-id E1A2B3C4D5E6F7 \
  --paths "/sitemap.xml" "/pages/pricing"
```

**— Security & Identity —**

### 301. How does AWS Secrets Manager fit alongside Rails encrypted credentials?

**Short Answer**

Secrets Manager stores secrets **outside** your app and hands them out at runtime, so you can **rotate a secret without a redeploy**.

Rails encrypted credentials are great for values that rarely change and can safely live (encrypted) in git.

**Simple Explanation**

Rails credentials (`rails credentials:edit`, decrypted with `RAILS_MASTER_KEY`) work well for config that changes rarely — a third-party API key you set up once.

But rotating one means editing the file, committing, and redeploying. And the same value is shared by every environment using that file.

Secrets Manager goes further. An ECS task definition can inject a secret's **current** value as an environment variable at container start, so rotating it needs no image change at all. It can even automate rotating an RDS password and updating the secret itself.

So: credentials for stable, versioned config; Secrets Manager for anything that needs rotation, per-environment separation, or an audit trail.

**Example**

```yaml
# ECS task definition — injects the *current* secret value as an env var
# at container start; rotating the secret needs no image rebuild
Secrets:
  - Name: DATABASE_URL
    ValueFrom: arn:aws:secretsmanager:us-east-1:123456789012:secret:myapp/database_url
```

```ruby
# Alternative: fetch directly at boot, e.g. for a value ECS secrets
# injection doesn't cover
require "aws-sdk-secretsmanager"

client = Aws::SecretsManager::Client.new(region: "us-east-1")
secret = client.get_secret_value(secret_id: "myapp/stripe_api_key")
Rails.application.credentials.stripe_api_key ||= JSON.parse(secret.secret_string)["key"]
```

### 302. What's the difference between an IAM user and an IAM role, and what does "least privilege" mean in practice?

**Short Answer**

- **IAM user** — long-lived credentials tied to a person or a legacy integration. They don't expire on their own.
- **IAM role** — temporary, auto-rotating credentials that a service assumes. Your app code never holds a static AWS key at all.

**Least privilege** means scoping a policy to exactly the resources needed — not `Resource: "*"`.

**Simple Explanation**

An IAM user's access key works until someone manually revokes it. If it leaks, that's a long window.

A role has no credentials of its own. Something is granted permission to **assume** it, and AWS hands out short-lived temporary credentials for that session.

When you attach a role to an ECS task or EC2 instance, the AWS SDK picks those credentials up automatically. There's no `AWS_ACCESS_KEY_ID` sitting in an env file to leak.

**Least privilege** means the policy names the specific bucket or table, not a wildcard. So if the credentials ever do leak — say via an SSRF bug reaching the instance metadata endpoint — the damage is capped at exactly what that role could do.

**Example**

```yaml
# Bad: broad access — any leaked credential from this role can touch
# every bucket in the account
Effect: Allow
Action: "s3:*"
Resource: "*"
```

```yaml
# Good: least privilege — scoped to exactly the bucket/prefix this
# app's role actually needs
Effect: Allow
Action:
  - s3:GetObject
  - s3:PutObject
Resource: "arn:aws:s3:::myapp-uploads/*"
```

### 303. What's the difference between a public and a private subnet in a VPC?

**Short Answer**

- **Public subnet** — has a route to an Internet Gateway, so things in it can be reached from the internet. That's where your load balancer goes.
- **Private subnet** — only routes outbound through a NAT Gateway, so there's **no direct inbound path** from the internet. That's where app servers and databases go.

**Simple Explanation**

A VPC is your own isolated slice of AWS networking, carved into subnets. What makes a subnet "public" or "private" is purely its **route table**, not a label.

A public subnet's route table sends internet-bound traffic (`0.0.0.0/0`) to an Internet Gateway, which allows traffic both in and out.

A private subnet sends that traffic to a NAT Gateway (which itself sits in a public subnet). That lets things inside make **outbound** calls — pulling a gem, calling a third-party API — without ever being reachable **from** the internet.

So the typical layout: load balancer in public subnets, and app servers, database, and cache all in private subnets behind it.

**Example**

```yaml
# Public subnet: routes 0.0.0.0/0 to an Internet Gateway
PublicSubnetRouteTable:
  Routes:
    - DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

# Private subnet: routes 0.0.0.0/0 to a NAT Gateway (outbound only)
PrivateSubnetRouteTable:
  Routes:
    - DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway
```

### 304. Security Groups vs. NACLs — what's the difference, and when would you actually need a NACL?

**Short Answer**

- **Security Group** — **stateful**, attached to instances, and can only **allow**. Return traffic is handled automatically.
- **NACL** — **stateless**, attached to a whole subnet, and can both allow **and deny**.

Most setups use security groups only. NACLs are for subnet-wide blocks, like blacklisting a bad IP range.

**Simple Explanation**

**Stateful** means a security group automatically allows the response to a connection you allowed. You don't write a matching rule for the return traffic.

**Stateless** means a NACL checks every packet independently in both directions, so you must explicitly allow the inbound request **and** the outbound response (including high ephemeral ports). That's easy to get wrong, which is part of why NACLs are used sparingly.

The one thing a security group **can't** do is deny. Since it only holds allow rules (everything else is implicitly denied), there's no way to say "block this one IP but allow everything else".

That's exactly what a NACL's explicit deny rule is for, applied at the subnet boundary so the traffic is dropped before it reaches any instance.

**Example**

```bash
# Security group: stateful, allow-only, attached to the ALB
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123abcd \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# NACL: stateless, subnet-wide, can explicitly deny a bad actor's CIDR
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0456efgh \
  --rule-number 100 --protocol -1 \
  --cidr-block 198.51.100.0/24 \
  --egress false --rule-action deny
```

**— High Availability & Scaling —**

### 305. How does auto scaling work for a Rails app, and how does it relate to Availability Zones?

**Short Answer**

An auto-scaling group (ASG) keeps a target number of instances running, spread across multiple Availability Zones, and adds or removes them based on a metric like CPU or request count.

Combined with load balancer health checks, unhealthy instances are pulled from rotation and replaced automatically.

**Simple Explanation**

You give the ASG a range (min, max, desired) and a set of subnets spanning multiple AZs. It keeps the actual instance count matching demand — scaling out under load, scaling in when quiet.

Spreading across AZs means it handles **failure**, not just load. If one AZ has a problem, instances in the other AZs are already serving traffic, and the ASG launches replacements in healthy AZs to get back to the desired count.

A **target-tracking** policy is the simplest form: "keep average CPU at 60%" and AWS works out the rest.

None of this needs a human during a routine instance failure.

**Example**

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name myapp-web-asg \
  --min-size 2 --max-size 10 --desired-capacity 3 \
  --vpc-zone-identifier "subnet-az1,subnet-az2,subnet-az3" \
  --target-group-arns arn:aws:elasticloadbalancing:...:targetgroup/myapp-web \
  --health-check-type ELB --health-check-grace-period 60

aws autoscaling put-scaling-policy \
  --auto-scaling-group-name myapp-web-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
    "TargetValue": 60.0
  }'
```

### 306. What is an Availability Zone, and why does deploying across multiple AZs matter?

**Short Answer**

An Availability Zone is one or more physically separate data centres within a region, with independent power, cooling, and networking.

Deploying across multiple AZs protects you from a single data-centre failure that a single-AZ setup simply cannot survive.

**Simple Explanation**

A region (like `us-east-1`) contains several AZs, connected by fast links but physically isolated from each other. A power failure or network issue in one AZ is designed not to affect the others.

If your entire app — web servers, database, everything — runs in one AZ, that AZ's failure is a total outage, no matter how much redundancy you built inside it.

Running across (typically) three AZs, with load balancing and auto scaling spreading capacity, plus a database that can fail over between AZs, means a data-centre-level event reduces your capacity instead of taking you down.

**Example**

```
Region: us-east-1
 ├── AZ: us-east-1a  → web instance(s), RDS primary
 ├── AZ: us-east-1b  → web instance(s), RDS standby
 └── AZ: us-east-1c  → web instance(s)

ALB spans all three AZs' subnets — a full failure of us-east-1a still
leaves 1b and 1c serving traffic, and RDS can fail over to its standby.
```

### 307. How does Multi-AZ RDS work, and what's the honest failover expectation?

**Short Answer**

Multi-AZ RDS keeps a **synchronously replicated standby** in another AZ and promotes it automatically if the primary fails — typically completing in roughly 60–120 seconds.

It does **not** protect you from a bad migration or a destructive query, because those replicate to the standby too.

**Simple Explanation**

"Synchronous" means every write is confirmed on the standby before the commit is acknowledged. So unlike a read replica, the standby is never behind.

If the primary or its whole AZ fails, RDS promotes the standby and updates the database endpoint's DNS to point at it. Your app doesn't change any connection string — it just sees a brief connection interruption.

The honest number to give a stakeholder is "typically under two minutes", not "instant".

The critical limitation: Multi-AZ is a **replica of your data**, not a time machine. If a migration drops a column, or code runs a `DELETE` without a `WHERE`, that change is replicated to the standby immediately. Failing over doesn't undo it.

That's what backups and point-in-time recovery are for.

**Example**

```yaml
# RDS instance with Multi-AZ enabled (CloudFormation-style)
Engine: postgres
MultiAZ: true
DBInstanceClass: db.r6g.large
BackupRetentionPeriod: 7   # separate mechanism — protects against bad writes/migrations
```

### 308. How does an RDS read replica differ from a Multi-AZ standby?

**Short Answer**

- **Multi-AZ standby** — synchronous, **not readable**, exists only for automatic failover.
- **Read replica** — asynchronous (so it can lag), **readable**, exists to scale reads. Promotion is manual.

**Simple Explanation**

The standby must stay perfectly in sync for instant safe failover, so RDS doesn't let you query it. It's pure insurance, sitting idle until needed.

A read replica takes the opposite trade-off. Replication is asynchronous, so it can fall a few seconds behind — but that looser guarantee is exactly what makes it usable for real traffic, offloading expensive reporting queries or read-heavy endpoints from the primary.

Rails supports routing specific queries to a replica with `connected_to(role: :reading)`.

Many production setups run **both**: Multi-AZ for failover protection, plus read replicas for scaling. They solve different problems and don't replace each other.

**Example**

```ruby
# config/database.yml
production:
  primary:
    <<: *default
    database: myapp_production
  primary_replica:
    <<: *default
    database: myapp_production
    replica: true
    host: myapp-prod-replica.abc123xyz.us-east-1.rds.amazonaws.com
```

```ruby
# Route a reporting query to the replica explicitly
ActiveRecord::Base.connected_to(role: :reading) do
  MonthlyReport.expensive_aggregation
end
```

**— Load Balancing & Orchestration —**

### 309. ALB vs. NLB vs. a self-hosted Nginx/HAProxy reverse proxy — which fits a typical Rails REST API?

**Short Answer**

- **ALB** — Layer 7, understands HTTP. Path and host routing, WebSocket support. The default for a Rails API.
- **NLB** — Layer 4, raw TCP. Ultra-low latency and a static IP, but no HTTP awareness.
- **Nginx/HAProxy** — when you need a reverse proxy inside your own stack, or you're not on AWS.

**Simple Explanation**

**Layer 7** means the ALB reads HTTP. It can route `/api/*` to one target group and `/admin/*` to another, inspect headers, and terminate TLS. It also supports WebSocket upgrades, which ActionCable needs. That maps naturally onto how a Rails app is organised.

**NLB** works at the raw TCP level with almost no understanding of HTTP. You give up routing intelligence for extremely low, consistent latency and a **static IP** — useful when a partner needs to allowlist a fixed address.

**Self-hosted Nginx or HAProxy** is still common inside the compute layer — buffering slow client connections before they reach Puma, serving a maintenance page, handling rewrites.

Beyond round-robin, **least-connections** routing helps when request durations vary a lot, and **IP hash / consistent hashing** helps with session stickiness or cache locality.

**Example**

```yaml
# ALB listener rule: path-based routing to two different target groups
Rules:
  - Conditions:
      - Field: path-pattern
        Values: ["/api/*"]
    Actions:
      - Type: forward
        TargetGroupArn: arn:aws:elasticloadbalancing:...:targetgroup/api-service
  - Conditions:
      - Field: path-pattern
        Values: ["/cable"]
    Actions:
      - Type: forward
        TargetGroupArn: arn:aws:elasticloadbalancing:...:targetgroup/actioncable-service
```

### 310. What Kubernetes objects do you touch deploying a Rails app, and what do `requests`/`limits` control?

**Short Answer**

- **Deployment** — manages your pod replicas and rolling updates.
- **Service** — a stable internal address in front of those pods.
- **ConfigMap / Secret** — inject configuration and credentials.
- **Ingress** — routes external HTTP traffic in.

For resources: `requests` is what's **guaranteed** for scheduling; `limits` is the **hard ceiling** that gets a container OOM-killed if exceeded.

**Simple Explanation**

A **Deployment** describes the desired state — image, replica count, rolling update strategy — and Kubernetes continuously works to match reality to it.

A **Service** gives that changing set of pods one stable DNS name and load-balances across whichever are healthy, so nothing else needs to track individual pod IPs.

A **ConfigMap** holds non-secret config; a **Secret** holds sensitive values. Important caveat: Kubernetes Secrets are only **base64-encoded** by default, which is encoding, not encryption. Many teams pair them with an external secrets operator that pulls the real values from Secrets Manager.

**Ingress** defines HTTP routing rules for traffic entering the cluster, usually backed by an ALB.

**requests vs limits:** the scheduler uses `requests` to decide which node has room. `limits` is enforced at runtime — exceed the memory limit and the kernel kills the container; exceed the CPU limit and you're just throttled, not killed.

**Example**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-web
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 1, maxSurge: 1 }
  template:
    spec:
      containers:
        - name: web
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp-web:abc1234
          envFrom:
            - configMapRef: { name: myapp-config }
            - secretRef: { name: myapp-secrets }
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits: { cpu: "1", memory: "1Gi" }
          readinessProbe:
            httpGet: { path: /up, port: 3000 }
            initialDelaySeconds: 5
---
apiVersion: v1
kind: Service
metadata: { name: myapp-web }
spec:
  selector: { app: myapp-web }
  ports: [{ port: 80, targetPort: 3000 }]
```

### 311. A Kubernetes pod isn't serving traffic — how do you triage `Pending` vs. `CrashLoopBackOff` vs. a failing readiness probe, and what does `OOMKilled` tell you?

**Short Answer**

- **Pending** — the scheduler can't place the pod anywhere (not enough resources, or an unbound volume).
- **CrashLoopBackOff** — the container starts and keeps exiting. That's an app boot crash.
- **Readiness probe failing** — the container runs fine but isn't ready, so it's removed from the Service but not restarted.
- **OOMKilled** — it exceeded its memory limit and the kernel killed it (exit code 137).

**Simple Explanation**

`kubectl describe pod` is the first move for all of these — the Events section explains why.

**Pending** means it never got scheduled. `describe` will say something like "Insufficient memory" or name an unbound volume claim.

**CrashLoopBackOff** means the process itself keeps exiting. Use `kubectl logs --previous` to see the **crashed** container's logs, since the current one may be a fresh restart with nothing in it yet. For Rails that's usually a missing master key, a pending migration, or a bad entrypoint.

**A failing readiness probe** is quieter. The process is running and not crashing, but the probe isn't returning 200 — often because it can't reach the database. Kubernetes correctly leaves it running but takes it out of load balancing, which is exactly right: don't send users to a pod that can't serve them.

**OOMKilled** shows in `describe` under Last State. For Rails this is usually too many Puma workers for the memory limit, a genuine leak, or a limit set too tight for memory-heavy work like image processing.

**Example**

```bash
kubectl describe pod myapp-web-7d9f8c6b5-x2k9p   # Events section explains Pending/CrashLoopBackOff/OOMKilled

kubectl logs myapp-web-7d9f8c6b5-x2k9p --previous  # crash reason from the last exited container

kubectl get pod myapp-web-7d9f8c6b5-x2k9p -o jsonpath='{.status.containerStatuses[0].lastState}'
# {"terminated":{"exitCode":137,"reason":"OOMKilled", ...}}
```

### 312. What are the infra-level trade-offs between blue-green, canary, and rolling deployment strategies?

**Short Answer**

- **Blue-green** — run a full second environment and switch all traffic at once. Instant cutover and instant rollback, but double the infrastructure during the switch.
- **Canary** — send a small percentage to the new version first and ramp up while watching metrics. Smallest blast radius, but slower and needs good monitoring.
- **Rolling** — replace instances a few at a time. No extra infrastructure, but old and new run together for the whole rollout.

**Simple Explanation**

**Blue-green:** "green" (new) is fully deployed and warmed up alongside "blue" (current). At cutover you flip the load balancer's target group from blue to green. Rollback is flipping it back — both nearly instant. The cost is running two environments at once.

**Canary:** route maybe 5% of traffic to the new version. If error rates and latency look fine, ramp up over minutes or hours. It catches a bad release with limited damage, but it's slower and only worth the complexity if you have automated metric-watching.

**Rolling:** the default in Kubernetes and ECS. Replace pods a few at a time, no duplicate infrastructure. But for the whole rollout, both versions are live and taking traffic.

One rule applies to **all three**: your database migrations and API contracts must tolerate old and new code running simultaneously. That's not optional in any of these strategies.

**Example**

```bash
# Blue-green cutover at the infra level: swap the ALB listener's
# default action from the blue target group to the green one
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/myapp/abc/def \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/myapp-green

# Instant rollback: point it right back at blue
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/myapp/abc/def \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/myapp-blue
```

## DevOps / CI-CD

**— Git, PRs & CI Pipeline —**

### 313. Walk through the stages of a CI/CD pipeline for a Rails app. What does each stage actually check?

**Short Answer**

**lint → test → build → deploy**, in that order, so the cheap checks fail fast and each stage gates the next.

**Simple Explanation**

- **Lint** — static analysis: RuboCop for style, Brakeman for security issues, `bundle audit` for gems with known vulnerabilities. This takes seconds, so it runs first and blocks the pipeline cheaply.
- **Test** — the full suite against a real database service container. This is the expensive stage, so you don't want to reach it if lint already failed.
- **Build** — package the tested code into one immutable artifact, almost always a Docker image tagged with the git SHA. This matters because you want the exact bytes you tested to be the exact bytes you deploy.
- **Deploy** — push that specific artifact to an environment. Staging automatically on merge; production often behind a manual approval.

The ordering isn't arbitrary — each stage is more expensive than the last, so failing early saves time and money.

**Example**

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true
      - run: bundle exec rubocop
      - run: bundle exec brakeman -q --no-pager

  test:
    needs: lint
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: postgres }
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-retries 5
    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/app_test
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with: { ruby-version: '3.3', bundler-cache: true }
      - run: bin/rails db:schema:load
      - run: bundle exec rspec

  build:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: registry.example.com/app:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: ./bin/deploy registry.example.com/app:${{ github.sha }}
```

---

### 314. What does a good pull request look like, and what do you check first as a reviewer?

**Short Answer**

Small, scoped to one logical change, with a description explaining **why** (not just what), a linked ticket, and tests for the new behavior.

As a reviewer, check **correctness and tests first**. Style last — a linter should already own that.

**Simple Explanation**

My review is three passes:

1. **Does it work?** Does the happy path do what the description claims, and are edge cases handled — nil, empty collections, concurrent writes? Is there a test that pins the fix down so it can't silently regress?
2. **Is it in the right place?** Does this belong in this layer (model vs service vs controller)? Does it introduce an N+1? Is the migration reversible?
3. **Is it readable?** Naming, clarity, consistency with the rest of the codebase.

Style nits come last, and ideally never come from a human at all — RuboCop should catch them in CI before a reviewer opens the diff.

Arguing about style in a PR that has a correctness bug means the review is happening in the wrong order.

**Example**

```markdown
## What
Fixes double-charging when a user double-clicks "Submit" on checkout.

## Why
Support ticket #4821 — race condition let two Payment records get created
for one order because the submit button wasn't disabled after first click
and there was no idempotency guard server-side.

## Changes
- Add `idempotency_key` uniqueness constraint on `payments` (migration, reversible)
- Reject duplicate submits in `PaymentsController#create` with a 409
- Add request spec covering the double-submit race

## Test plan
- [x] `bundle exec rspec spec/requests/payments_spec.rb`
- [x] Verified manually in staging by double-clicking submit
```

---

### 315. What is environment parity, and why does config drift between dev/staging/prod cause "works on my machine" bugs?

**Short Answer**

Environment parity means dev, staging, and production run the same OS, language and gem versions, and the same service topology.

When they drift — different Postgres version, different Ruby patch, staging missing a Redis that production has — bugs only appear in the environment where the difference lives. That's "works on my machine".

**Simple Explanation**

The cause is almost always some invisible difference. A gem at a different version. A different Postgres collation. A background queue that exists in production but is stubbed in dev. An env var set in your local `.env` that was never added to staging.

The fix isn't heroic debugging — it's removing those differences:

- Pin versions with `.ruby-version` and `Gemfile.lock`.
- Run the same Docker image (or at least the same base image) everywhere.
- Provision infrastructure from the same Terraform modules, so staging and production are structurally identical.
- Never let a config key exist in one environment without existing (even with a fake value) in every other — and ideally check that automatically in CI.

**Example**

```bash
# A drift-detection check worth running in CI: diff the *keys* (not values)
# of each environment's config against a canonical list, so a var added
# to prod but never wired into staging fails the build instead of prod.
comm -3 \
  <(aws ssm get-parameters-by-path --path /app/staging/ --query 'Parameters[].Name' | sort) \
  <(aws ssm get-parameters-by-path --path /app/production/ --query 'Parameters[].Name' | sort)
# non-empty output = staging and production have different config keys
```

---

### 316. What is a reverse proxy like Nginx doing in front of a Rails app?

**Short Answer**

Nginx does the things an app server is bad at: terminating TLS, serving static files, buffering slow client connections, and load-balancing across multiple Puma processes.

That leaves Rails handling only application logic.

**Simple Explanation**

Puma is optimised for running Ruby code, not for holding thousands of slow connections open or streaming large uploads byte by byte.

Nginx sits in front and:

- **Terminates SSL/TLS**, so certificate handling isn't Rails' problem.
- **Serves static assets** directly from `public/` without touching Ruby.
- **Buffers** a slow client's request and response, so a Puma worker isn't tied up waiting on a bad network connection. This is the one people forget, and it matters a lot.
- **Load-balances** across app processes with health checks.

It's also a natural place for rate limiting, gzip compression, and request logging — before traffic ever reaches the app.

**Example**

```nginx
upstream rails_app {
  server unix:/var/run/puma/app.sock fail_timeout=0;
}

server {
  listen 443 ssl;
  server_name app.example.com;

  ssl_certificate     /etc/ssl/certs/app.crt;
  ssl_certificate_key /etc/ssl/private/app.key;

  location /assets/ {
    root /var/www/app/public;
    expires max;
    add_header Cache-Control "public";
  }

  location / {
    proxy_pass http://rails_app;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 30s;
  }
}
```

---

### 317. Why is Infrastructure as Code (Terraform, CloudFormation) preferred over making changes by hand in the cloud console?

**Short Answer**

Infrastructure as Code makes changes **reviewable, repeatable, and versioned** like application code.

A manual console change is invisible, unauditable, and impossible to reliably reproduce in another environment.

**Simple Explanation**

If someone clicks around the AWS console to add a security group rule, that change exists only in AWS's current state. There's no diff, no PR, no record of who did it or why, and no guarantee staging has the same rule.

With Terraform, the same change is a diff in a pull request. Someone reviews it, CI runs `terraform plan` to show exactly what will change **before** it's applied, and the same module can be applied to staging and production so they're genuinely identical.

That's also how you get real environment parity — not "we tried to remember to do the same thing twice".

And disaster recovery becomes "run `terraform apply` against a new account" instead of "hope someone remembers all the manual steps".

**Example**

```hcl
# main.tf
resource "aws_db_instance" "app" {
  identifier        = "app-${var.environment}"
  engine            = "postgres"
  engine_version    = "16.3"
  instance_class    = var.environment == "production" ? "db.r6g.xlarge" : "db.t4g.medium"
  allocated_storage = var.environment == "production" ? 500 : 50
  multi_az          = var.environment == "production"

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

```bash
# CI runs plan on PR, apply only after merge + approval
terraform plan -var="environment=staging" -out=tfplan
terraform apply tfplan
```

---

### 318. How would you roll back a bad deploy quickly — what has to be true beforehand, and when do you roll back vs. fix forward?

**Short Answer**

Fast rollback needs two things in place **beforehand**:

1. **Immutable, versioned artifacts** — so "the previous version" is one known image tag.
2. **Backward-compatible migrations** — so the old code still works against the new schema.

Prefer rollback under time pressure, **unless** the bad deploy already ran a destructive migration. Then fix forward.

**Simple Explanation**

Rollback only works cleanly if every deploy produced an artifact you can point back to — a Docker image tagged with the git SHA, not "whatever's checked out on the server".

The dangerous case is migrations. If the bad deploy dropped a column the previous version still reads, rolling back the **app** now crashes against a schema that no longer matches. You've made things worse.

That's why migrations should be backward-compatible for at least one deploy (the expand/contract pattern).

Given that discipline, I prefer rollback over fixing forward during an incident. It's faster and lower-risk than writing and reviewing new code while the site is down.

I fix forward when the bug is only in new code with no schema change, the fix is trivial and well understood, or rolling back isn't safe because of an already-applied destructive migration.

**Example**

```bash
# Rollback is just redeploying the last known-good, already-built artifact —
# no rebuild, no new code review, minutes not tens of minutes.
PREVIOUS_SHA=$(git rev-parse HEAD~1)
./bin/deploy registry.example.com/app:${PREVIOUS_SHA}

# Reversible migration written with expand/contract in mind:
class AddNewEmailColumn < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :contact_email, :string   # additive, old code ignores it
    # a LATER migration, deployed after all readers use contact_email,
    # removes :email — never both in the same deploy
  end
end
```

---

**— Deployment Strategies —**

### 319. Compare blue-green, canary, and rolling deployments. What's the risk trade-off of each?

**Short Answer**

- **Blue-green** — flip 100% of traffic to a second, fully-deployed environment. Instant rollback, but a bug hits everyone at once and you pay for double infrastructure.
- **Canary** — send a small percentage to the new version first. Smallest blast radius, but slower and needs weighted routing plus good metrics.
- **Rolling** — replace instances a few at a time. No extra infrastructure, but old and new versions run side by side throughout.

**Simple Explanation**

**Blue-green:** deploy to the idle environment, smoke-test it, then flip the router. Rollback is just flipping back. The downside is all-or-nothing exposure — if a bug only appears under real production load, every user hits it simultaneously.

**Canary:** route maybe 5% of real traffic to the new version, watch error rates and latency, then ramp up. You catch problems while only a few users are affected. It needs infrastructure that supports weighted routing, and metrics good enough to spot a regression from a 5% sample.

**Rolling:** take one instance out of the load balancer, deploy, put it back, repeat. Cheapest, but for the whole rollout both versions are serving traffic — so the deploy must be backward-compatible.

**Example**

```yaml
# Kubernetes rolling update config — old and new pods coexist during rollout
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # at most 1 pod down at a time
      maxSurge: 1         # at most 1 extra pod above desired count
```

```bash
# Canary via weighted traffic split (e.g. using a service mesh or ALB target groups)
aws elbv2 modify-listener --listener-arn $LISTENER_ARN --default-actions '[
  {"Type":"forward","ForwardConfig":{"TargetGroups":[
    {"TargetGroupArn":"'$STABLE_TG'","Weight":95},
    {"TargetGroupArn":"'$CANARY_TG'","Weight":5}
  ]}}
]'
```

---

### 320. How do feature flags change the deploy/release process, and why do stale flags become tech debt?

**Short Answer**

Feature flags separate **deploying** code (getting it onto servers) from **releasing** it (making it visible to users).

So you merge and deploy continuously, then control exposure separately — percentage rollout, specific users, or an instant kill switch.

The cost: every flag left behind after a feature ships is a permanent branch in your code.

**Simple Explanation**

Without flags, merging and shipping to 100% of users happen together, which makes every deploy high-stakes.

With a flag, you merge and deploy "dark" (code is live but gated off), then release gradually: internal staff, then 1% of users, then ramp to 100% while watching error rates.

Good flag systems do more than a boolean:

- **Percentage rollout** — 10% of users.
- **Attribute targeting** — all users on the enterprise plan, or one specific beta customer.
- **Kill switch** — instantly go back to 0% without a deploy.

The debt side is real. Every `if flag_enabled?` is code that needs testing in both states. Once a feature is at 100% and stable, delete the flag **and** the old branch. A codebase with dozens of "temporary" flags from finished features is one where nobody's sure which path is actually live.

**Example**

```ruby
# Using Flipper (a common Rails feature-flag gem)
class CheckoutController < ApplicationController
  def create
    if Flipper.enabled?(:new_checkout_flow, current_user)
      NewCheckoutService.call(current_user, cart)
    else
      LegacyCheckoutService.call(current_user, cart)
    end
  end
end
```

```ruby
# Rollout progression, all without a new deploy:
Flipper.enable_actor(:new_checkout_flow, current_org)      # target one tenant
Flipper.enable_percentage_of_actors(:new_checkout_flow, 10) # ramp to 10% of users
Flipper.disable(:new_checkout_flow)                         # kill switch if it misbehaves

# Cleanup once at 100% and stable for a release or two:
# 1. Remove the `if` branch and delete LegacyCheckoutService
# 2. Flipper.remove(:new_checkout_flow)
```

---

**— Multi-Environment Workflows —**

### 321. Describe a typical local → dev → staging → production pipeline. What should staging mirror about production to actually be useful?

**Short Answer**

**local → dev → staging → production**, each catching different problems.

Staging is only useful if it mirrors production's **infrastructure**, **integrations**, and **data shape** — not just the same code.

**Simple Explanation**

- **Local** — fast individual iteration, often with stubs.
- **Dev** — a shared environment where merged branches land, catching integration issues early.
- **Staging** — the last stop before production. This is where "if it works here, it'll work there" has to be true.
- **Production** — real users.

Staging earns its keep only if it's structurally the same as production: the same infrastructure shape (not one tiny box standing in for an autoscaled fleet), the same third-party integrations in sandbox mode (not mocked out), and data that resembles production in **shape** — similar table sizes, similar edge cases (accounts with thousands of records, unicode names, null-heavy columns).

That usually means a regularly refreshed, PII-scrubbed copy of production data. A staging environment with 12 rows per table will never catch the N+1 or missing index that only appears at real scale.

**Example**

```yaml
# A staging DB refresh job: nightly copy of prod, scrubbed of PII, so staging
# actually has production-shaped data instead of stale hand-made fixtures.
name: Refresh Staging DB
on:
  schedule:
    - cron: '0 3 * * *'
jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - run: pg_dump $PRODUCTION_READONLY_URL | pg_restore -d staging_db
      - run: bundle exec rails runner scripts/scrub_pii.rb RAILS_ENV=staging
```

---

### 322. How do you manage environment-specific configuration (DATABASE_URL, API keys, feature flags) without hardcoding or duplicating code?

**Short Answer**

Read configuration from **environment variables** (or an encrypted secrets store) at runtime. The same code runs everywhere — only the injected values differ.

Application code should never branch on `Rails.env` to pick a URL or a key.

**Simple Explanation**

Rails' defaults already point this way: `config/database.yml` reads `ENV["DATABASE_URL"]`, and Rails credentials give you an encrypted per-environment secrets file that's safe to commit.

The rule of thumb: the **behavior** should be identical across environments; only the **target** differs. An `if Rails.env.production?` that swaps an API endpoint is a smell — that belongs in config.

Secrets specifically belong in a secrets store scoped per environment and injected at deploy time. Never committed in plaintext, and never hand-copied into each environment's dashboard where they can silently drift apart.

**Example**

```yaml
# config/database.yml — same for every environment, value comes from ENV
production:
  <<: *default
  url: <%= ENV.fetch("DATABASE_URL") %>

staging:
  <<: *default
  url: <%= ENV.fetch("DATABASE_URL") %>
```

```bash
# rails credentials:edit --environment staging
# creates config/credentials/staging.key (kept out of git) and
# config/credentials/staging.yml.enc (safe to commit, decrypted at boot)
stripe:
  secret_key: sk_test_...
```

```ruby
# usage is identical across environments — no environment branching in app code
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)
```

---

### 323. How do database migrations flow through environments, and what do you do if staging's schema has drifted from production's?

**Short Answer**

The same migration files run in the same order in every environment, as part of the deploy pipeline.

If staging has drifted, `rails db:migrate:status` shows exactly which migrations are missing — reconcile by **running them**, not by hand-editing the schema.

**Simple Explanation**

Migrations are version-controlled and checked in with the code that needs them. The pipeline applies them in order: dev, then staging, then production.

`schema.rb` (or `structure.sql`) is the source of truth for what the schema should look like.

Drift usually means someone ran a migration by hand against one environment, or rolled one back in one place but not another.

`db:migrate:status` prints an up/down list per migration. Diff that output between staging and production to find the divergence point, then run (or roll back) the missing one.

Never fix drift by manually altering tables to "match". That leaves no record of what happened and the next migration may fail unpredictably.

**Example**

```bash
# On staging
$ bin/rails db:migrate:status
   up     20240101120000  Create users
   up     20240115093000  Add index to orders
  down    20240201110000  Add contact_email to users   # <-- drifted!

# Reconcile by actually running it, not hand-editing the table
$ RAILS_ENV=staging bin/rails db:migrate

# Confirm both environments now agree
$ diff <(RAILS_ENV=staging bin/rails db:migrate:status) \
       <(RAILS_ENV=production bin/rails db:migrate:status)
```

---

### 324. What problem do ephemeral, per-PR environments solve that a single shared staging environment doesn't?

**Short Answer**

Per-PR environments give every pull request its own isolated deployment, so reviewing one change is never blocked or polluted by someone else's half-finished work on the single shared staging box.

**Simple Explanation**

A single shared staging environment is a queue. If two people deploy their branches around the same time, whoever tests second sees a mix of both changes — or their feature is broken by someone else's unfinished work sitting there.

Ephemeral environments are created automatically when a PR opens (a Kubernetes namespace per branch, a Heroku Review App, an ECS task set) and destroyed when it closes. Each gets its own URL and its own database.

That removes the "staging is busy, wait your turn" bottleneck. A reviewer clicks a link in the PR and sees **exactly** that change running, nothing else mixed in.

**Example**

```yaml
# .github/workflows/review-app.yml
on:
  pull_request:
    types: [opened, synchronize, closed]

jobs:
  deploy-review-app:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - run: |
          kubectl create namespace pr-${{ github.event.number }} --dry-run=client -o yaml | kubectl apply -f -
          helm upgrade --install pr-${{ github.event.number }} ./chart \
            --namespace pr-${{ github.event.number }} \
            --set image.tag=${{ github.sha }}

  teardown-review-app:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - run: kubectl delete namespace pr-${{ github.event.number }}
```

---

### 325. Why promote one build artifact through environments instead of rebuilding at each stage?

**Short Answer**

Build **once**, then promote that identical artifact through environments.

Rebuilding at each stage means staging and production aren't provably the same bits — a dependency could resolve differently, or a base image could get patched between the two builds.

**Simple Explanation**

Even with the same Dockerfile and the same git commit, two separate builds can produce different results. An unpinned gem resolves differently. A base image tag like `ruby:3.3` picks up a security patch. The build environment itself differs.

Then "it passed in staging" is no longer a real guarantee about production, because they're not the same artifact.

The fix: build exactly once per commit, tag it immutably (the git SHA), push it to a registry, and have every environment's deploy pull **that same tag**.

So staging runs `app:a1b2c3d`, and promoting to production means pointing production at that same `app:a1b2c3d` — not rebuilding from source.

**Example**

```yaml
# Build once...
- uses: docker/build-push-action@v6
  with:
    push: true
    tags: registry.example.com/app:${{ github.sha }}

# ...deploy that exact tag to staging
- run: ./bin/deploy registry.example.com/app:${{ github.sha }} --env staging

# ...and later, promote the SAME tag to production (no rebuild)
- run: ./bin/deploy registry.example.com/app:${{ github.sha }} --env production
```

---

### 326. A config value like a payment gateway's sandbox vs. live API key is easy to get wrong between environments. What safeguards would you put in place?

**Short Answer**

Three layers:

1. **Namespace secrets per environment** so there's nowhere obvious to copy-paste the wrong one.
2. **Assert at boot** — most providers' keys have a recognisable prefix (`sk_live_` vs `sk_test_`), so refuse to start if it doesn't match the environment.
3. **Make it an automated gate**, not a checklist item.

**Simple Explanation**

The failure mode is almost always a human pasting a value into the wrong environment's config panel.

So make copying awkward on purpose: separate Vault paths or Secrets Manager entries per environment, separate `credentials/staging.yml.enc` and `credentials/production.yml.enc` — not one flat list.

Then add a startup check. If `Rails.env.production?` and the Stripe key doesn't start with `sk_live_`, refuse to boot. And the reverse — refuse to boot staging with a **live** key, which is the more dangerous direction.

Making it a failing boot (or a failing build) matters. Checklists get skipped under deadline pressure; a crash doesn't.

**Example**

```ruby
# config/initializers/stripe.rb
key = Rails.application.credentials.dig(:stripe, :secret_key)

if Rails.env.production? && !key.start_with?("sk_live_")
  raise "Refusing to boot production with a non-live Stripe key"
elsif !Rails.env.production? && key.start_with?("sk_live_")
  raise "Refusing to boot #{Rails.env} with a LIVE Stripe key"
end

Stripe.api_key = key
```

---

**— Monitoring & Observability —**

### 327. Why does Prometheus use a pull model — scraping `/metrics` — instead of apps pushing metrics to it?

**Short Answer**

Prometheus **polls** each target's `/metrics` endpoint on a schedule rather than apps pushing to it.

That makes Prometheus the single place that knows what *should* be up — so a target disappearing is instantly detectable — and keeps the pipeline simple, with no queue or push agent needed.

**Simple Explanation**

With pull, your app just exposes a plain-text `/metrics` endpoint describing its current counters and gauges. Prometheus decides when to scrape it and stores the result as a time series.

That has a nice side effect: if a target stops responding, Prometheus knows immediately — the built-in `up` metric goes to `0`. With a push model, a dead app just... stops sending data, and you can't easily tell "app is down" from "network hiccup".

Pull also means Prometheus owns scrape scheduling and retries, rather than every app reimplementing it. And you can scrape things you can't instrument directly by using an exporter.

The exception is short-lived batch jobs that don't live long enough to be scraped. Those push to the **Pushgateway** as a deliberate special case.

**Example**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'rails_app'
    scrape_interval: 15s
    static_configs:
      - targets: ['app-1.internal:9394', 'app-2.internal:9394']
```

---

### 328. What are the four core Prometheus metric types, and when would you use each?

**Short Answer**

- **Counter** — only goes up (total requests). Always read with `rate()`.
- **Gauge** — goes up and down (queue depth, memory usage).
- **Histogram** — buckets observations so percentiles are calculated **server-side**, and can be aggregated across instances.
- **Summary** — similar, but calculates percentiles in the app, so they **can't** be meaningfully averaged across instances.

**Simple Explanation**

A **counter** is cumulative and only resets on restart. You never read it raw — you apply `rate()` to get "requests per second".

A **gauge** is a snapshot that can move either way.

A **histogram** records observations (like request duration) into configurable buckets, exposing `_bucket`, `_sum`, and `_count`. Prometheus then computes percentiles from those buckets at query time. Because the raw buckets are additive, you can compute a p99 across all your instances combined.

A **summary** calculates quantiles inside the app before exposing them. That's cheaper to query, but you can't average quantiles across instances — which is exactly why histograms are preferred for anything cluster-wide.

**Example**

```ruby
# Using the prometheus-client gem in a Rails app
require 'prometheus/client'
prometheus = Prometheus::Client.registry

REQUEST_COUNT = prometheus.counter(:http_requests_total,
  docstring: 'Total HTTP requests', labels: [:method, :status])

QUEUE_DEPTH = prometheus.gauge(:sidekiq_queue_depth,
  docstring: 'Current Sidekiq queue depth', labels: [:queue])

REQUEST_DURATION = prometheus.histogram(:http_request_duration_seconds,
  docstring: 'Request latency', labels: [:controller, :action],
  buckets: [0.05, 0.1, 0.25, 0.5, 1, 2.5, 5])

# in a controller around_action:
REQUEST_COUNT.increment(labels: { method: 'GET', status: 200 })
REQUEST_DURATION.observe(duration, labels: { controller: 'orders', action: 'show' })
```

---

### 329. How would you write a PromQL query for "5xx error rate over the last 5 minutes" and "p99 latency"?

**Short Answer**

- **Error rate** — divide the rate of 5xx responses by the rate of all responses.
- **p99 latency** — `histogram_quantile(0.99, ...)` applied to a rate of histogram buckets.

**Simple Explanation**

`rate()` turns a constantly-increasing counter into a per-second average over a window. That's what makes counters usable for alerting — you want "how fast is this growing right now", not "total since boot".

For **error rate**, dividing 5xx by total gives you a percentage that's independent of traffic volume. That matters: "50 errors" means something very different at 100 requests/second than at 100,000.

For **latency percentiles**, `histogram_quantile()` works on the `_bucket` series, summed by their `le` ("less than or equal") label, to interpolate the value below which 99% of requests fell.

Remember to `sum by (le, ...)` before applying the quantile, or the maths won't aggregate correctly across instances.

**Example**

```promql
# 5xx error rate (%) over the last 5 minutes
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# p99 request latency over the last 5 minutes, per controller/action
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, controller, action)
)
```

---

### 330. What is a Prometheus exporter, and when do you need one?

**Short Answer**

An exporter is a small process that sits next to something that doesn't speak Prometheus — a database, an OS, a queue — and translates its internal stats into a scrapeable `/metrics` endpoint.

**Simple Explanation**

Your Rails app can expose `/metrics` itself because you control the code.

Postgres, Redis, and the Linux host can't — they weren't written with Prometheus in mind.

An exporter runs alongside them, queries their native stats interface (`pg_stat_activity` for Postgres, `INFO` for Redis, `/proc` for the OS), and re-exposes that data in Prometheus format.

The common ones: `node_exporter` (CPU, memory, disk, network), `postgres_exporter` (connections, replication lag, slow queries), and `redis_exporter` (memory, hit rate, connected clients).

**Example**

```yaml
# docker-compose.yml
services:
  postgres_exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://readonly_user:pass@db:5432/app?sslmode=disable"
    ports: ['9187:9187']

  node_exporter:
    image: prom/node-exporter
    ports: ['9100:9100']
```

```yaml
# prometheus.yml addition
scrape_configs:
  - job_name: 'postgres'
    static_configs: [{ targets: ['postgres_exporter:9187'] }]
  - job_name: 'node'
    static_configs: [{ targets: ['node_exporter:9100'] }]
```

---

### 331. How does Grafana relate to Prometheus?

**Short Answer**

Prometheus **stores and queries** metrics. Grafana **visualizes** them.

Grafana stores nothing itself — it queries Prometheus (and other sources) and draws the result as dashboards.

**Simple Explanation**

They're deliberately separate. Prometheus's job is scraping, storing, and answering PromQL queries. Grafana's job is turning those results into graphs a human can read at a glance.

A single Grafana dashboard can mix panels from several data sources — a PromQL panel for request rate next to a Loki panel showing recent error logs next to a CloudWatch panel for an RDS metric.

That's genuinely useful, because "what does the system look like right now" almost always spans more than one backend.

**Example**

```yaml
# Grafana datasource provisioning — same dashboard, two backends
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
  - name: Loki
    type: loki
    url: http://loki:3100
```

---

### 332. What would you actually put on a monitoring dashboard for a Rails app?

**Short Answer**

Six things: **request rate**, **error rate**, **latency percentiles** (p50/p95/p99), **CPU/memory**, **background job queue depth**, and **database connection pool usage**.

That's enough to answer "is anything on fire?" and "where's the bottleneck?" without digging through logs.

**Simple Explanation**

- **Request rate** — traffic volume, so you can read a latency spike in context. Is it a real problem or just a traffic surge?
- **Error rate** — the most direct "users are having a bad time" signal.
- **Latency p50/p95/p99** — p50 is the typical experience, p99 is your worst-off users. An average hides both.
- **CPU / memory** — catches resource exhaustion before it becomes an outage.
- **Queue depth** — a growing queue means jobs are being created faster than processed. That's an early warning long before a user notices their email never arrived.
- **Connection pool usage** — if the pool is maxed, requests queue waiting for a connection, which shows up as latency **everywhere**, not just in database-heavy endpoints. Classic hidden bottleneck.

**Example**

```promql
# Panel queries for a "Rails App Health" Grafana dashboard
sum(rate(http_requests_total[5m])) by (status)                     # request rate by status
sum(rate(http_requests_total{status=~"5.."}[5m]))                  # error rate
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))  # p95
sidekiq_queue_depth                                                 # job backlog
active_record_connection_pool_size - active_record_connection_pool_available  # pool usage
```

---

### 333. How does Alertmanager decide when a rule fires and who gets paged?

**Short Answer**

Prometheus evaluates the alert rule and, once the condition holds for the configured `for` duration, sends it to Alertmanager.

Alertmanager then **groups** related alerts, **deduplicates** repeats, **routes** them to the right receiver, and can **silence** or **inhibit** redundant ones.

**Simple Explanation**

The rule itself lives in Prometheus and includes a `for` duration, so a one-second blip doesn't page anyone.

Alertmanager does the parts that make alerting survivable at scale:

- **Grouping** — bundle 30 "pod down" alerts from one node outage into a single notification instead of 30 pages.
- **Deduplication** — don't re-page every evaluation cycle for an alert that's still ongoing.
- **Routing** — send database alerts to the DB team's Slack and payment alerts to PagerDuty, based on labels.
- **Inhibition** — suppress "API is slow" when "API is down" is already firing. The first is just noise once you know the second.

**Example**

```yaml
# alert.rules.yml (evaluated by Prometheus)
groups:
  - name: rails_app
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels: { severity: critical, team: platform }
        annotations:
          summary: "5xx error rate above 5% for 5 minutes"

# alertmanager.yml (routes the fired alert)
route:
  receiver: default-slack
  group_by: ['alertname', 'team']
  routes:
    - match: { severity: critical }
      receiver: pagerduty
      group_wait: 30s
      repeat_interval: 1h
```

---

### 334. What causes alert fatigue, and how would you redesign alerting to avoid it?

**Short Answer**

Alert fatigue comes from paging on every possible **cause** instead of on user-visible **symptoms**. People get woken for things that don't matter, start ignoring pages, and then miss the real one.

The fix: alert on symptoms tied to your SLOs, and let dashboards handle root-cause diagnosis **after** the page.

**Simple Explanation**

The anti-pattern is alerting on every low-level signal — one CPU spike on one of fifty pods, one slow query, a transient 502. Individually these are mostly noise; pods self-heal, load balancers retry, autoscaling kicks in.

If every one of those pages someone, the signal-to-noise ratio collapses. Within weeks people mute the channel or reflexively acknowledge without reading — which means the real incident buried in that noise gets missed too.

The redesign: page only on symptoms that mean users are actually affected right now — error rate over threshold, p99 latency over threshold, service unreachable — with a `for:` duration long enough to ignore blips.

Causes (one pod's CPU, one slow query) belong on dashboards you check **after** being paged, or as low-priority tickets. Not as 3am pages.

**Example**

```yaml
# Bad: pages on every possible cause, low signal
- alert: OneInstanceCPUHigh
  expr: instance:cpu_usage > 80
  for: 1m
  labels: { severity: page }

# Better: pages only on user-visible symptom, tied to the SLO that matters
- alert: ErrorBudgetBurnFast
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m])) > 0.05
  for: 5m
  labels: { severity: page }
  annotations:
    runbook: "https://runbooks.internal/high-error-rate"
```

---

### 335. What's the difference between an SLI, SLO, and SLA, and how do error budgets connect them to release decisions?

**Short Answer**

- **SLI** — the measured number ("% of requests under 300ms").
- **SLO** — your internal target for it ("99.5% over 30 days").
- **SLA** — the customer-facing promise, with consequences.

The gap between 100% and your SLO is your **error budget**, and how much is left tells you whether it's safe to ship something risky this week.

**Simple Explanation**

An SLI is a raw number you can graph. An SLO is a target you hold yourself to, deliberately stricter than the SLA so you have margin before you'd actually breach a contract.

The **error budget** is just `1 - SLO`. If your SLO is 99.9% over 30 days, your budget is 0.1% of that month's requests allowed to fail.

That's what makes it useful:

- **Burned most of the budget already** (a bad deploy, an outage)? Freeze risky releases and focus on stability.
- **Well within budget**? That's explicit permission to ship the risky migration this week, because you have room to absorb a mistake.

It turns "can we ship this?" from a gut-feeling argument into a number everyone can look at.

**Example**

```promql
# Error budget remaining, 30-day rolling window, SLO = 99.9% success rate
1 - (
  sum(increase(http_requests_total{status=~"5.."}[30d]))
  / sum(increase(http_requests_total[30d]))
) / 0.001
# result > 0 means budget remaining; result <= 0 means budget already exhausted
```

---

### 336. What's the difference between white-box and black-box monitoring, and why do you need both?

**Short Answer**

- **White-box** — internal metrics your app exposes (queue depth, pool usage). Tells you **why** something is wrong.
- **Black-box** — checking your service from outside, like a real user. Tells you **whether** it's actually broken.

You need both, because every internal metric can look green while DNS, the CDN, or TLS is broken.

**Simple Explanation**

White-box requires instrumenting the thing itself — everything Prometheus scraping `/metrics` gives you. It's great for diagnosis: connection pool exhausted, queue backed up, GC pauses climbing.

But it can't see failures in front of your app that never reach the instrumented code — a bad DNS record, an expired TLS certificate, a CDN returning cached errors, a load balancer routing nowhere.

Black-box monitoring (a synthetic check from outside your network) doesn't care about internals. It just asks "can an outside user load this URL and get the right response?" — which is much closer to what your customers experience.

Relying on only one leaves a blind spot: white-box alone misses "everything internal is green but the site is unreachable"; black-box alone tells you something's wrong but nothing about why.

**Example**

```yaml
# Black-box probe via the Prometheus blackbox_exporter — an external-style
# HTTP check, run from outside the app's own process
- job_name: 'blackbox-http'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://app.example.com/health
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox_exporter:9115
```

---

**— Incident Response —**

### 337. What makes a fix a "hotfix" rather than a normal bugfix, and what's your hotfix process end to end?

**Short Answer**

It's a hotfix because of **urgency and customer impact**, not because the code change is small or large.

The process trades the normal review cadence for a faster one — but it's still reviewed: triage → branch from the deployed release → minimal fix → expedited review → deploy → verify → backport.

**Simple Explanation**

A one-line fix for an outage affecting everyone is a hotfix. A large refactor for a cosmetic issue is not.

The seven steps:

1. **Triage** — confirm it's really happening in production and gauge the blast radius.
2. **Branch** off the currently deployed release tag or main — **not** off someone's half-finished feature branch — so the fix contains nothing else in flight.
3. **Fix** — the smallest change that resolves the symptom. Resist cleaning up nearby code while you're in there.
4. **Expedited review** — still reviewed, just by whoever's available fastest, focused on "does this fix it and does it break anything else".
5. **Deploy** straight to production, skipping the normal staging soak if the situation warrants it.
6. **Verify** against real traffic and metrics that the symptom is gone.
7. **Backport** — if `main` has moved on since the tag you branched from, cherry-pick the fix there too, so the next release doesn't reintroduce the bug.

**Example**

```bash
# 1. Branch from the exact deployed tag, not main
git checkout -b hotfix/nil-address-crash v2.14.3

# 2. Minimal fix + a regression test
git commit -am "Guard against nil shipping_address in OrderSerializer"

# 3. Push, fast-track PR review, merge
git push origin hotfix/nil-address-crash

# 4. Tag and deploy directly
git tag v2.14.4
./bin/deploy registry.example.com/app:v2.14.4 --env production

# 5. Backport into main if it has moved on since v2.14.3
git checkout main
git cherry-pick <hotfix-commit-sha>
```

---

### 338. Rolling back vs. rolling forward for a bad deploy in a live incident — how do you decide, and what's the schema-mismatch risk?

**Short Answer**

Prefer **rolling back** whenever it's clean — you already know the previous version worked, and there's no new code to write and review mid-incident.

**Roll forward** when the bad deploy already ran a destructive migration, because reverting the app against a schema the old code doesn't understand can make things worse.

**Simple Explanation**

Under time pressure, rollback is almost always faster and safer. Writing a fix while the incident channel is loud and people are anxious is exactly when you're most likely to introduce a second bug.

The case that flips it is migrations.

If you roll back only the **application** code but the migration already removed a column, the old code now queries a column that doesn't exist — and crashes just as hard as the original bug. Now you have two problems and a schema you can't easily un-migrate without risking data loss.

In that situation, rolling forward with a small targeted fix (or a compensating migration) is safer than unwinding both code and schema at once.

This is exactly why backward-compatible migrations matter: they're what keeps "just roll back" available as a safe option.

**Example**

```bash
# Safe: pure app-code rollback, no schema involved
./bin/deploy registry.example.com/app:${PREVIOUS_SHA}

# Unsafe if this already ran: rolling back app code now crashes on the
# missing column, because the old code still expects `email` to exist
class RemoveEmailColumn < ActiveRecord::Migration[7.1]
  def change
    remove_column :users, :email, :string   # destructive, hard to roll back cleanly
  end
end
# Correct pattern instead: deploy that stops reading :email first,
# THEN a later, separate migration drops the column.
```

---

### 339. What is an RCA (root cause analysis / postmortem), and what goes into a good one?

**Short Answer**

An RCA (root cause analysis, or postmortem) is a written record after an incident covering the **timeline**, **impact**, **root cause**, and **owned follow-up actions**.

You write one even after the fix ships, because the fix addresses **this** instance — the RCA is what prevents the **next** one.

**Simple Explanation**

Shipping the fix stops the bleeding. It doesn't answer "why did our process or architecture allow this?"

A good RCA has five parts:

- **Timeline** — when it started, when it was detected, key actions, when it was resolved. Real timestamps, not "later that day".
- **Impact** — how many users or requests affected, for how long.
- **Root cause** — not just the immediate trigger (see the next question).
- **What went well** — what limited the damage or sped up detection. Worth reinforcing.
- **Action items with named owners and due dates** — not "we should add more tests", but a ticket assigned to a person.

The action items are the part that actually prevents recurrence. An RCA that's just a narrative with no owned follow-ups gets read once and forgotten.

**Example**

```markdown
# RCA: Checkout outage, 2026-09-14

## Timeline (UTC)
- 14:02 — Deploy of v2.14.3 completes
- 14:05 — Error rate alert fires (5xx > 5% for 5m)
- 14:09 — On-call acknowledges, begins triage
- 14:17 — Root cause identified: nil shipping_address crashes OrderSerializer
- 14:22 — Rollback to v2.14.2 deployed
- 14:26 — Error rate back to baseline, incident resolved

## Impact
~8% of checkout requests failed for 20 minutes (~340 orders affected).

## Root cause
See "5 Whys" — missing test coverage for orders with no shipping address,
combined with no review coverage on that serializer path.

## What went well
Alert fired within 3 minutes of the bad deploy; rollback took under 5 minutes
because the previous artifact was already built and tagged.

## Action items
- [ ] Add regression test for nil shipping_address (@akhilesh, due 9/16)
- [ ] Require a second reviewer on OrderSerializer changes (@leads, due 9/20)
- [ ] Add contract test asserting serializer output shape (@akhilesh, due 9/23)
```

---

### 340. What's the "5 Whys" technique, and how do you distinguish the proximate cause from the root cause?

**Short Answer**

"5 Whys" means repeatedly asking "why did that happen?" until you reach a **systemic** cause instead of a symptom.

- **Proximate cause** — the last thing that broke (a missing nil check).
- **Root cause** — the process gap that let it reach production (no review coverage on that code path).

**Simple Explanation**

It's tempting to stop at the first explanation because it's concrete and satisfying — "a nil check was missing" — and just add the nil check.

But that only fixes this one instance. The same class of bug will happen again elsewhere unless you find out **why** a missing nil check reached production undetected.

Each "why" should point at a decision, a process, or a missing safeguard — not just restate the previous symptom in different words.

And "five" isn't a rule. Sometimes it's three, sometimes seven. You stop when you hit something you can actually change.

**Example**

```text
1. Why did checkout return 500s?
   → OrderSerializer raised NoMethodError on a nil shipping_address.
2. Why was shipping_address nil?
   → A subset of legacy orders were created before shipping_address was
     required, and the serializer assumed it always present.
3. Why didn't a test catch this?
   → No test exercised serialization of an order without a shipping_address.
4. Why wasn't that gap caught in review?
   → The PR that added shipping_address.city to the serializer was reviewed
     by someone unfamiliar with legacy order data and merged without
     asking "what about older records."
5. Why does that keep happening?
   → There's no review-assignment rule routing changes to serializers used
     by high-traffic paths to someone who owns that domain.

Proximate cause: missing nil check on shipping_address.
Root cause: no review-coverage policy for high-traffic serializer changes,
and no test fixture representing legacy nil-shipping-address orders.
```

---

### 341. What's a blameless postmortem, and why does assigning blame to a person make the next incident more likely, not less?

**Short Answer**

A blameless postmortem treats the incident as a **systems and process** failure rather than one person's mistake.

Blaming a person doesn't make people less error-prone — it makes them hide mistakes, so the next near-miss goes unreported until it's a real outage.

**Simple Explanation**

If "who broke prod?" becomes the headline, the rational response from everyone watching is to be more careful about what they **admit to** — not to become more careful in general.

So the next person who notices something risky, or who almost causes an incident, quietly fixes it and says nothing. You lose the early warning signal entirely.

Blameless doesn't mean no accountability. It means the accountability is about **completing the action items** — fixing the process gap — not punishing whoever happened to push the button that a broken process allowed.

Concretely: instead of "Akhilesh deployed without running the full suite", you write "the pipeline allowed a merge to production without the full suite passing, because the required-checks list didn't include the new CI job". Then fix the config so it's impossible regardless of who deploys.

**Example**

```text
Blaming (bad): "Akhilesh deployed without running the full test suite,
that's why this happened."
→ Next engineer under deadline pressure quietly skips a step too,
  but doesn't mention it, because admitting it invites the same blame.

Blameless (good): "The deploy pipeline allowed a merge to production
without the full suite passing because the required-checks list on the
branch protection rule didn't include the new spec file's CI job."
→ Action item: fix the branch protection config so this is impossible
  regardless of who's deploying.
```

---

### 342. What's the difference between severity and priority, and how would you triage a new bug report?

**Short Answer**

- **Severity** — how bad the impact objectively is (data loss, users affected, is there a workaround).
- **Priority** — what you actually work on first.

Severity strongly influences priority, but business context can override it.

**Simple Explanation**

Severity is a technical assessment. Priority is a scheduling decision.

They usually track together — a SEV1 is almost always P0 — but not always. A high-severity bug in a feature being deprecated next sprint, with a documented workaround, might sit below a medium-severity bug in the signup flow that every new customer hits.

**Triage in practice:**

1. **Reproduce it.** An unreproducible report needs more information before anything else.
2. **Assess blast radius.** How many users, is data at risk, is there a workaround?
3. **Assign severity** from that objective assessment.
4. **Set priority** by weighing severity against what's already committed and who's available.

A useful severity scale: SEV1 = outage or data loss with no workaround, page immediately. SEV2 = major feature broken for many users, fix same day. SEV3 = minor breakage with a clear workaround, fix this sprint. SEV4 = cosmetic or tiny edge case, backlog.

**Example**

```text
Severity scale (used to assign, independent of what else is in flight):
SEV1 — Full outage or data loss/corruption, no workaround. Page immediately.
SEV2 — Major feature broken or significantly degraded for many users;
        workaround exists but is painful. Fix same day.
SEV3 — Minor feature broken or degraded for a subset of users;
        clear workaround exists. Fix within the sprint.
SEV4 — Cosmetic issue, edge case, or affects a tiny fraction of users.
        Backlog, fix when convenient.

Triage example:
Report: "Export to CSV produces garbled unicode for names with accents."
1. Reproduce: confirmed on staging with a French-named test account.
2. Blast radius: affects CSV export only, ~3% of users have non-ASCII
   names, no data loss (source data is correct, only export is wrong),
   workaround is exporting to a different format.
3. Severity: SEV3 — real bug, real users affected, but a workaround exists
   and no data integrity risk.
4. Priority: scheduled for next sprint, behind the SEV2 already in progress,
   not paged.
```

## Security

**— Core Web App Security —**

### 343. How does SQL injection happen in a Rails app, and how does ActiveRecord prevent it?

**Short Answer**

SQL injection happens when user input is glued directly into a SQL string, so the database can't tell data from code.

Active Record prevents it by sending values as **bound parameters**, separate from the SQL text.

**Simple Explanation**

The attack is crafted input (like `' OR '1'='1`) that changes what a query means — turning "find one row" into "return every row", or worse.

Active Record is safe by default whenever you use hash conditions, `?` placeholders, or named placeholders. The value goes to the database separately from the SQL, so the driver treats it purely as data.

The danger zone is anywhere you see Ruby string interpolation (`#{}`) in something that looks like SQL:

- `where("name = '#{params[:name]}'")`
- `order(params[:sort])`
- `find_by_sql` with interpolated input
- `connection.execute` with interpolated input

In code review, `#{}` inside a SQL-ish string is an immediate flag. For dynamic column names (like sorting), use an explicit allow-list.

**Example**

```ruby
# VULNERABLE — string interpolation builds the SQL, user input becomes part of the query
def search
  @users = User.where("name = '#{params[:name]}'")
  # params[:name] = "' OR '1'='1" returns every row in the table
end

# Also vulnerable — a dynamic ORDER BY isn't covered by hash conditions
@users = User.order(params[:sort]) # params[:sort] = "1; DROP TABLE users;--"

# FIXED — bound parameters; the driver sends the value separately from the SQL text
@users = User.where("name = ?", params[:name])
@users = User.where(name: params[:name])              # hash conditions get the same protection
@users = User.where("name = :name", name: params[:name])

# FIXED — allowlist when you genuinely need a dynamic column/direction
ALLOWED_SORT_COLUMNS = %w[name created_at email].freeze
sort_column = ALLOWED_SORT_COLUMNS.include?(params[:sort]) ? params[:sort] : "name"
@users = User.order(sort_column)
```

---

### 344. What is XSS, and how does Rails prevent it by default?

**Short Answer**

XSS (cross-site scripting) is when an attacker gets malicious JavaScript to run in another user's browser.

Rails escapes everything you interpolate into an ERB template by default. So the main way to reintroduce the hole is calling `raw` or `.html_safe` on user content.

**Simple Explanation**

Three flavours:

- **Stored XSS** — the script is saved to the database (in a comment, say) and served to every future viewer.
- **Reflected XSS** — the script comes from the current request, like a search query echoed back into the page. Only affects whoever clicks the crafted link.
- **DOM-based XSS** — the bug is entirely in client-side JavaScript manipulating the page from untrusted data. The server never even sees the payload.

Rails' default protection is automatic HTML escaping: `<%= %>` converts `<`, `>`, `&`, and quotes into entities, so an injected `<script>` tag renders as harmless text.

The footgun is explicitly telling Rails to trust something. The moment user content flows through `raw(...)` or `.html_safe`, you've turned the protection off for that value.

If you genuinely need to render user-authored HTML, use `sanitize` with an allow-list of tags and attributes — don't trust the whole string.

**Example**

```erb
<%# SAFE by default — ERB auto-escapes interpolated values %>
<p><%= comment.body %></p>
<%# comment.body == "<script>fetch('/steal?c='+document.cookie)</script>" %>
<%# renders as literal text on the page — the browser never executes it %>

<%# VULNERABLE — html_safe / raw tell Rails "don't escape this" %>
<p><%= raw comment.body %></p>
<p><%= comment.body.html_safe %></p>
<%# now the <script> tag actually executes — stored XSS if comment.body came from a user %>
```

```ruby
# If you must render some user-authored HTML (e.g. a rich-text field), sanitize it —
# allowlist which tags/attributes survive, rather than trusting the whole string.
class Comment < ApplicationRecord
  def safe_body
    ActionController::Base.helpers.sanitize(body, tags: %w[b i em strong a], attributes: %w[href])
  end
end
```

---

### 345. What is CSRF, and how does Rails prevent it?

**Short Answer**

CSRF tricks a logged-in user's browser into submitting a request to your app, because browsers attach cookies automatically.

Rails prevents it with a per-session `authenticity_token` that a forged cross-site request can't know, plus `SameSite` cookies as a second layer.

**Simple Explanation**

A malicious site can auto-submit a form to `your-app.com`, and the victim's browser happily attaches their session cookie. The browser has no idea the request shouldn't be trusted just because it came from somewhere else.

Rails embeds a random `authenticity_token` in every form it renders and requires it on every state-changing request. The attacker's page can't read or guess that token, because it's tied to the victim's session on your domain. So the forged request is rejected with `ActionController::InvalidAuthenticityToken`.

`SameSite=Lax` (Rails' cookie default) adds defense in depth at the browser level: the session cookie isn't sent on most cross-site requests in the first place, so even a naive forged request arrives with no session at all.

**Example**

```ruby
# ApplicationController — enabled by default in new Rails apps
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end
```

```erb
<%= form_with model: @post do |f| %>
  <%# Rails injects a hidden authenticity_token field automatically %>
<% end %>
```

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: "_app_session",
  same_site: :lax,
  secure: Rails.env.production?
```

---

### 346. What's the difference between authentication and authorization?

**Short Answer**

- **Authentication** — who are you? (verifying identity)
- **Authorization** — what are you allowed to do? (verifying permission)

An app can get authentication perfectly right and still leak data by skipping authorization.

**Simple Explanation**

These get confused constantly.

An authentication failure looks like "I let someone in who shouldn't be" or "I trusted an identity claim I never verified".

An authorization failure looks like "I correctly know who you are, but I forgot to check whether **you** are allowed to touch **this** record".

That second one is the classic **IDOR** bug: a controller checks `current_user` exists but loads `Order.find(params[:id])` without checking it belongs to them. Any logged-in user can then read anyone's order by changing the ID.

The fix is usually as simple as scoping the query: `current_user.orders.find(params[:id])`, which raises a 404 if it isn't theirs.

**Example**

```ruby
# Authentication gone wrong — no identity check at all
def show
  @user = User.find(params[:id])   # anyone, logged in or not, can view any profile
end

# Authorization gone wrong — authenticated, but not authorized (IDOR)
before_action :authenticate_user!
def show
  @order = Order.find(params[:id])   # any logged-in user can view ANY user's order by guessing the id
end

# Fixed — authorization scopes the record to the authenticated identity
def show
  @order = current_user.orders.find(params[:id])   # raises 404 if it isn't theirs
end
```

---

### 347. JWT vs session-based auth — what are the trade-offs, including logout?

**Short Answer**

Store a **salted, deliberately slow, one-way hash** — bcrypt (Rails' `has_secure_password`) or argon2.

Never plaintext, never a fast hash like SHA-256, and never something you designed yourself.

**Simple Explanation**

Passwords need an algorithm designed to be **slow** and resistant to GPU acceleration. A leaked database of fast hashes like MD5 or SHA-256 can be brute-forced at billions of guesses per second on ordinary hardware.

`has_secure_password` generates a random **salt** per user and folds it into the bcrypt hash, so you never manage salts by hand. It also gives you an `authenticate` method that compares safely.

Rolling your own is a trap even for strong engineers. The subtle bugs — a non-constant-time comparison enabling timing attacks, a missing per-user salt enabling rainbow tables, a fast hash enabling brute force — are exactly what battle-tested libraries have already eliminated.

**Example**

```ruby
# Session-based: server holds state, revocation is instant
def destroy
  reset_session   # session id invalidated server-side immediately — the old cookie is now useless
end

# JWT-based: stateless, must maintain a denylist to revoke before natural expiry
class JsonWebToken
  SECRET = Rails.application.credentials.secret_key_base

  def self.encode(payload, exp = 15.minutes.from_now)
    payload[:exp] = exp.to_i
    payload[:jti] = SecureRandom.uuid
    JWT.encode(payload, SECRET, "HS256")
  end

  def self.decode(token)
    body = JWT.decode(token, SECRET, true, algorithm: "HS256")[0]
    raise JWT::DecodeError, "revoked" if Rails.cache.read("revoked_jti:#{body['jti']}")
    HashWithIndifferentAccess.new(body)
  end
end
# "Logging out" early means writing the token's jti to a denylist (TTL = remaining time to exp) —
# without that, a stolen JWT stays valid until it naturally expires.
```

---

### 348. How do you securely store passwords, and why not roll your own hashing?

**Short Answer**

HTTPS (TLS underneath) gives you three things: **confidentiality** (encrypted), **integrity** (tamper-evident), and **authenticity** (you're really talking to the right server).

Without it, passwords, session cookies, and tokens travel as plain text that anyone on the network path can read or change.

**Simple Explanation**

Without TLS, an attacker anywhere between the browser and your server — public Wi-Fi, a compromised router, an ISP — can passively read every request and response.

Worse, they can actively **rewrite** them: inject malicious JavaScript into an HTML response, swap a download link, or simply capture the session cookie and replay it to impersonate the user.

TLS doesn't just hide the payload. It also makes tampering detectable (any modification breaks the integrity check) and proves you're talking to the real certificate holder rather than an impostor.

In Rails, `config.force_ssl = true` is the one-line way to stop accepting plain HTTP in production.

**Example**

```ruby
# Gemfile: gem 'bcrypt', '~> 3.1.7'
class User < ApplicationRecord
  has_secure_password   # expects a password_digest column; bcrypt-hashes + per-user-salts automatically
end

user = User.new(password: "correct horse battery staple")
user.authenticate("wrong guess")                  # => false
user.authenticate("correct horse battery staple")  # => user

# NEVER do this — fast, unsalted, trivially reversible at scale
class User < ApplicationRecord
  before_save { self.password_digest = Digest::SHA256.hexdigest(password) }
end
```

---

### 349. What does HTTPS/TLS actually solve, and what breaks if you skip it?

**Short Answer**

CORS controls which **browser origins** are allowed to read a response from your API.

Allowing any origin **together with credentials** means any website's JavaScript can make authenticated, cookie-carrying requests to your API and read the response — turning every visiting user into an exploit.

**Simple Explanation**

Browsers actually forbid combining `Access-Control-Allow-Origin: *` with `credentials: true` for exactly this reason. But the equivalent mistake — **reflecting** whatever `Origin` header the request sent back as the allowed origin — has the same effect and passes the browser's check.

Once that's in place, a page on `evil.com` can fetch `api.yourapp.com/account` with `credentials: 'include'`. The browser attaches the victim's session cookie, your API answers because the origin check "passed", and `evil.com` reads the private data.

The fix is a hard **allow-list** of exactly the origins your frontends run on. Never reflect the request's origin back.

**Example**

```ruby
# config/environments/production.rb
config.force_ssl = true
# redirects all http:// requests to https://, marks cookies Secure, and adds
# the Strict-Transport-Security header (see the HSTS question below)
```

---

### 350. What's the risk of a misconfigured CORS policy, e.g. `Access-Control-Allow-Origin: *` with cookie-based auth?

**Short Answer**

Rate-limit login attempts by **IP** and by **account** (with something like `rack-attack`), and add account lockout after repeated failures.

Otherwise the endpoint is just a free password-guessing machine.

**Simple Explanation**

Without throttling, credential-stuffing bots will happily try thousands of leaked username/password pairs per minute.

`rack-attack` runs as Rack middleware and can throttle:

- **By IP** — stops one source hammering you.
- **By submitted email** — stops a distributed attack spread across many IPs against one account.

You want both, because either alone has a gap.

Application-level lockout (like Devise's `:lockable`) is complementary, not a replacement. Throttling slows the attacker at the network layer; lockout protects a specific account even from a many-IP attack.

One caution: return the same generic "invalid email or password" message either way, so you don't leak which accounts exist.

**Example**

```ruby
# Gemfile: gem 'rack-cors'
# config/initializers/cors.rb

# VULNERABLE — wide-open origin combined with credentials
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "*"
    resource "/api/*", headers: :any, methods: %i[get post], credentials: true
  end
end

# FIXED — explicit allowlist of known frontends
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "app.example.com", "admin.example.com"
    resource "/api/*", headers: :any, methods: %i[get post patch delete], credentials: true
  end
end
```

---

### 351. How do you protect a login endpoint from brute-force attacks?

**Short Answer**

Give every identity — a database role, an API key, an IAM role — **only** the permissions it needs, so a single leaked credential does the least possible damage.

**Simple Explanation**

If your Rails app connects to Postgres as the superuser, then a SQL injection bug or a leaked `database.yml` doesn't just leak data — it can drop tables, alter schemas, and read other applications' data on the same cluster.

A narrowly-scoped application role that can only `SELECT`, `INSERT`, `UPDATE`, and `DELETE` on its own tables limits the damage to data within those tables. Migrations then run under a separate, more privileged role, only from CI.

The same logic applies everywhere:

- Use a third-party provider's **restricted** API keys scoped to the operations you actually call, not the full-access secret key.
- Give a background job that reads one S3 bucket an IAM role that can read **that bucket**, not every bucket in the account.

**Example**

```ruby
# Gemfile: gem 'rack-attack'
# config/initializers/rack_attack.rb
class Rack::Attack
  throttle("logins/ip", limit: 5, period: 20.seconds) do |req|
    req.ip if req.path == "/login" && req.post?
  end

  throttle("logins/email", limit: 5, period: 1.minute) do |req|
    if req.path == "/login" && req.post?
      req.params.dig("session", "email")&.downcase
    end
  end
end
```

---

### 352. What is the principle of least privilege, applied to DB users, API keys, and IAM roles?

**Short Answer**

Run `bundler-audit` (checks your `Gemfile.lock` against a database of known vulnerabilities) and Dependabot (opens PRs for outdated gems) in CI.

And understand that `Gemfile.lock` only pins **versions** — it doesn't vet whether a version is safe.

**Simple Explanation**

Two related but distinct risks:

1. **A gem you use has a known vulnerability** disclosed after you pinned it. `bundler-audit` catches this by comparing your lockfile against the public advisory database.
2. **A gem itself is malicious** — a maintainer's account gets compromised and a backdoored version is published, or an attacker publishes a package with a name close to a popular one (typosquatting) hoping for a typo in your Gemfile.

`Gemfile.lock` protects **reproducibility** — everyone installs identical versions — but says nothing about trust. If you pinned a compromised version, `bundle install` faithfully reinstalls it forever.

An **SBOM** (Software Bill of Materials — a structured list of every dependency and version) doesn't prevent attacks, but it's what lets you instantly answer "are we affected?" when a new CVE drops.

**Example**

```sql
-- app's runtime DB role: no DDL, no access to other schemas, no superuser
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
REVOKE ALL ON SCHEMA public FROM app_readwrite;
GRANT USAGE ON SCHEMA public TO app_readwrite;
-- migrations run under a separate, more privileged role, only from CI — never the runtime app role
```

```yaml
# config/database.yml
production:
  username: app_readwrite   # not the cluster owner/superuser
  password: <%= ENV.fetch("DATABASE_PASSWORD") %>
```

---

### 353. How do you manage dependency and supply-chain security for your gems?

**Short Answer**

Never hardcode secrets. Use environment variables or Rails encrypted credentials at minimum.

A dedicated secrets manager adds three things env vars can't: **automatic rotation**, **fine-grained access control**, and an **audit trail**.

**Simple Explanation**

A hardcoded secret ships with every clone of the repo, appears in every diff, and stays in git history forever even after you delete the line.

Environment variables fix "not in source control" but are still static, plaintext in the process, and visible to anything that can read the environment — including a crash reporter that dumps `ENV`.

Rails encrypted credentials are a step up: encrypted at rest in the repo, decrypted at boot with a master key kept out of git.

A real secrets manager goes further. It can issue short-lived credentials that rotate automatically (a database password that expires in an hour), enforce per-service access policies, and log every read — so you know exactly what accessed a secret and when.

**Example**

```bash
# CI step — fails the build if any gem in Gemfile.lock has a known advisory
bundle audit check --update
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "bundler"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

### 354. How should you manage application secrets, and what does a secrets manager add over `ENV`?

**Short Answer**

Mass assignment is when an attacker sets attributes you never intended — like adding `admin=true` to a form submission — because the controller passed the whole params hash to the model.

Strong parameters fix it by requiring an explicit allow-list of permitted attributes.

**Simple Explanation**

Without strong parameters, `@user.update(params[:user])` sets **every** attribute present in the params.

So if `User` has an `admin` column and the form only shows name and email, an attacker can add `user[admin]=true` to the raw request body and promote themselves. Nothing in the form stops them, because they're not using your form.

`permit(:name, :email)` is the allow-list. Only those attributes are extracted; anything else — `admin`, `role`, `account_balance` — is silently dropped before it ever reaches the model.

Note this is a controller-layer fix. The model doesn't know which params were safe, so the controller is the actual security boundary.

**Example**

```ruby
# BAD — hardcoded, committed, and now in every clone's git history
STRIPE_SECRET = "sk_live_51H..."

# BETTER — environment variable, not committed
Stripe.api_key = ENV.fetch("STRIPE_SECRET_KEY")

# BETTER STILL — Rails encrypted credentials (edited via `rails credentials:edit`)
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)
```

---

### 355. What is mass assignment, and how do Rails strong params prevent it?

**Short Answer**

- **Validation** — "is this data well-formed and valid?" Belongs at the real trust boundary: the model or API layer, server-side.
- **Sanitization** — "make this value safe for **this specific context**". Belongs right before that use: escape before HTML rendering, parameterize before SQL.

**Simple Explanation**

Client-side validation is **UX only**. HTML5 `required` and JavaScript checks give instant feedback to a well-behaved browser user, but they're trivially bypassed with curl. They can never be the security control.

The real gate is server-side, because that's the boundary an attacker can't route around.

Sanitization is a separate concern. A comment body can be perfectly **valid** (non-blank, under the length limit) while still containing a `<script>` tag that's dangerous **specifically when rendered as HTML**.

That's why sanitization happens at the point of use, not at input time. The same raw value may need to flow safely through several contexts — HTML, a CSV export, a log line — and mangling it on the way in destroys data you needed intact.

**Example**

```ruby
# VULNERABLE — assigns whatever the client sent, including attributes the form never intended to expose
def update
  @user.update(params[:user])   # params[:user][:admin] = "true" → attacker becomes admin
end

# FIXED — strong params allowlist exactly which attributes can be mass-assigned
def update
  @user.update(user_params)
end

private

def user_params
  params.require(:user).permit(:name, :email)   # :admin isn't in the list — Rails drops it, not the DB
end
```

---

### 356. Where does input validation belong vs sanitization?

**Short Answer**

Policy objects put "can this user do this to this record?" in **one place per model**, and make the check a required step (`authorize @post`) instead of something a developer has to remember.

That closes the exact gap that causes IDOR.

**Simple Explanation**

IDOR is common precisely because it's an **omission**, not a visible bug. `Order.find(params[:id])` looks completely normal, and `before_action :authenticate_user!` makes the endpoint *feel* secure — but nothing confirms the record belongs to the requester.

Pundit forces every action to call `authorize record`, which raises unless the matching policy method returns true. So the check becomes structural rather than something you hope someone remembered.

Concentrating the rules in one policy class per model also makes it auditable: a security review reads `OrderPolicy` once instead of hunting through every controller for a missing check.

**Example**

```ruby
class User < ApplicationRecord
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :age, numericality: { greater_than_or_equal_to: 0 }
end
# Client-side `required`/`pattern` attributes are UX sugar only — the model validation above
# (or an equivalent controller check on an API) is the real gate; a raw POST bypasses HTML entirely.

# Sanitization happens at the boundary where the value becomes dangerous, not at input time:
sanitize(comment.body)                          # right before rendering as HTML
ActiveRecord::Base.sanitize_sql_like(query)      # right before using inside a LIKE clause
```

---

### 357. How do policy objects (Pundit/CanCanCan) prevent IDOR vulnerabilities?

**Short Answer**

SSRF (Server-Side Request Forgery) is when an attacker gives you a URL that **your server** fetches, and points it at internal infrastructure instead of the public internet.

Mitigate it by allow-listing schemes and hosts, **resolving the hostname and blocking private IP ranges**, and never auto-following redirects.

**Simple Explanation**

Any feature where your backend fetches a user-supplied URL — a webhook target, "import image from URL" — is a potential SSRF vector. Your server sits inside your private network and can reach things the internet can't:

- Cloud metadata endpoints (`169.254.169.254`), which on AWS can hand back IAM credentials.
- An internal admin panel with no auth because "it was never internet-facing".
- Redis or another service listening on localhost.

Naive fixes don't work. Just checking the hostname string is bypassable via DNS rebinding or a redirect.

So the mitigation has to: resolve the hostname to an IP, check **that IP** against blocked ranges (loopback, private ranges, link-local), and refuse to follow redirects — because an allow-listed URL could 302 to an internal one.

**Example**

```ruby
# VULNERABLE — authenticated but not authorized (classic IDOR)
before_action :authenticate_user!
def show
  @order = Order.find(params[:id])   # returns ANY user's order — nothing scopes it to current_user
end
```

```ruby
# Gemfile: gem 'pundit'
class OrderPolicy < ApplicationPolicy
  def show?
    user.admin? || record.user_id == user.id
  end
end

class OrdersController < ApplicationController
  def show
    @order = Order.find(params[:id])
    authorize @order   # raises Pundit::NotAuthorizedError if show? returns false
  end
end
```

---

### 358. What is SSRF, and how do you mitigate it?

**Short Answer**

- **In transit** (TLS) — protects data while it moves across a network.
- **At rest** — protects stored data from someone who gets the disk, a backup, or a database snapshot.

They defend against different attackers, so you need both.

**Simple Explanation**

TLS is useless against someone who steals your database backup file. That data was never "in transit" at that point — it's sitting on disk, and if the disk isn't encrypted they can read it directly.

Conversely, at-rest encryption doesn't help if the connection carrying live queries is plaintext. A network attacker never touches the disk.

So production needs both: TLS everywhere data moves (client to server, server to database, server to third-party APIs) **and** encryption at rest for the database and its backups.

**Example**

```ruby
# VULNERABLE — server fetches whatever URL the user supplies, no restrictions
class WebhookController < ApplicationController
  def create
    HTTParty.get(params[:image_url])
    # attacker passes http://169.254.169.254/latest/meta-data/iam/security-credentials/
    # or http://localhost:6379/ to probe an internal Redis instance
  end
end
```

```ruby
# FIXED — allowlist scheme, resolve the host, reject private/link-local ranges, no redirect follow
require "resolv"

ALLOWED_SCHEMES = %w[http https].freeze
BLOCKED_RANGES = [
  IPAddr.new("127.0.0.0/8"), IPAddr.new("10.0.0.0/8"), IPAddr.new("172.16.0.0/12"),
  IPAddr.new("192.168.0.0/16"), IPAddr.new("169.254.0.0/16"), # cloud metadata lives here
  IPAddr.new("::1/128")
].freeze

def safe_fetch(url_string)
  uri = URI.parse(url_string)
  raise "scheme not allowed" unless ALLOWED_SCHEMES.include?(uri.scheme)

  ip = IPAddr.new(Resolv.getaddress(uri.host))
  raise "blocked host" if BLOCKED_RANGES.any? { |range| range.include?(ip) }

  HTTParty.get(url_string, follow_redirects: false)   # a followed redirect could repoint past this check
end
```

**— Encryption & Transport Security —**

### 359. What's the difference between encryption in transit and encryption at rest?

**Short Answer**

The client and server use **asymmetric** cryptography briefly to verify the server's certificate and agree on a shared key, then switch to fast **symmetric** encryption for all the actual data.

Asymmetric is too slow to use for the whole connection, so it's only used to bootstrap trust.

**Simple Explanation**

**Symmetric** encryption uses the **same** key to encrypt and decrypt. Fast, but both sides need the key already — which is the hard part.

**Asymmetric** uses a public/private key pair. Slower, but it lets two strangers establish trust without a pre-shared secret.

The handshake, roughly:

1. Client says hello with its supported TLS versions and cipher suites.
2. Server replies with its chosen cipher and its **certificate** (containing its public key, signed by a Certificate Authority).
3. Client validates that certificate chain up to a CA its OS or browser already trusts.
4. Both sides run a key exchange (like ECDHE) so they derive the **same** symmetric key without ever sending that key over the wire.
5. Everything from then on uses fast symmetric encryption (like AES-256-GCM).

**Example**

```ruby
# In transit
config.force_ssl = true   # forces HTTPS for every request

# At rest — see the "encrypting sensitive columns" question below for the application-level version;
# the infrastructure-level version is usually a managed-database setting, e.g.:
# RDS: enable "Encryption at rest" when provisioning the instance (transparent, disk-level AES-256)
```

---

### 360. Walk through what actually happens in a TLS handshake.

**Short Answer**

A self-signed certificate is signed by its own key instead of a Certificate Authority the browser already trusts.

The connection is still **encrypted** — the warning is about **authenticity**: there's nobody independent vouching that you're talking to the right server.

**Simple Explanation**

Certificate validation works by checking the signature chain up to a trusted root already installed in the OS or browser.

A self-signed certificate's chain ends at itself. Nothing external vouches for it, so the browser can't tell "this really is example.com's key" from "this is an attacker's key claiming to be example.com".

Encryption still works fine — the handshake doesn't need CA trust to agree a symmetric key. That's why the warning is specifically about identity, not privacy.

Self-signed certificates are normal for local development and internal tooling behind a private trust store. They're a red flag on anything public-facing. For local development, tools like `mkcert` install a locally-trusted CA so you don't get the warning.

**Example**

```text
1. Client → Server: ClientHello (supported TLS version, cipher suites, a random value)
2. Server → Client: ServerHello (chosen cipher), Certificate (server's public key, signed by a CA),
   ServerHelloDone
3. Client validates the certificate chain up to a Certificate Authority already trusted by its OS/browser
4. Client and server perform a key exchange (e.g. ECDHE) — asymmetric crypto lets both sides derive the
   same symmetric session key without ever transmitting that key itself over the wire
5. Both sides switch to symmetric encryption (e.g. AES-256-GCM) using that session key for all
   subsequent application data — this is what actually carries your HTTP request/response bodies
```

---

### 361. What is a self-signed certificate, and why does a browser warn about it?

**Short Answer**

HSTS tells the browser "always use HTTPS for this site, never even try plain HTTP".

It prevents **SSL stripping**, where an attacker intercepts a user's very first plaintext request and quietly keeps them on HTTP.

**Simple Explanation**

If a user types `example.com` with no scheme, the browser's first request is plain HTTP, and it only redirects to HTTPS **after** the server responds.

An attacker on that network (public Wi-Fi is the classic case) can intercept that first request and proxy everything over plain HTTP, stripping the upgrade while the user sees a normal-looking but unencrypted page.

HSTS fixes this for **returning** visitors. Once a browser has seen the header, it refuses to make plain-HTTP requests to that host for `max-age` seconds, upgrading internally before any request leaves the machine.

The `preload` directive closes the gap even for a first-ever visit, by getting the domain baked into a list shipped with the browser itself.

In Rails, `config.force_ssl = true` sends this header for you.

**Example**

```bash
# Generating a self-signed cert for local HTTPS development
openssl req -x509 -newkey rsa:4096 -keyout dev.key -out dev.crt -days 365 -nodes \
  -subj "/CN=localhost"
# Browsers will warn on this because "localhost" here isn't vouched for by any CA in their trust store —
# for real local dev, tools like `mkcert` install a locally-trusted CA instead to avoid the warning.
```

---

### 362. What does HSTS do, and what attack does it prevent?

**Short Answer**

Use Rails' built-in `encrypts` (Active Record Encryption) on specific sensitive columns.

Choose **deterministic** mode only when you need to query that column by equality — it's less secure than the default.

And never hardcode the keys: use encrypted credentials at minimum, ideally a KMS.

**Simple Explanation**

Column encryption protects a specific field even if the database itself is compromised — a leaked backup, an over-permissioned analytics tool, a DBA with broad read access. None of them see plaintext without the key.

That's a **different** threat model from full-disk encryption, which protects against physical media theft but does nothing once someone has a live authenticated connection.

The trade-off is queryability:

- **Non-deterministic** (the default, more secure) — the same plaintext produces different ciphertext each time, so you can't use it in a `WHERE` clause or index it.
- **Deterministic** — same plaintext, same ciphertext. Equality queries and unique indexes work, at some loss of security.

Keys belong outside source code. A KMS adds rotation without re-encrypting every row, access-controlled decryption, and an audit log of every decrypt call.

**Example**

```ruby
# config/environments/production.rb
config.force_ssl = true
# sends: Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

---

### 363. How do you encrypt sensitive columns at rest in Rails, and how do you manage the keys?

**Short Answer**

- **Hashing** — one-way. There's no key that turns it back.
- **Encryption** — reversible with the right key.

You hash passwords because you only ever need to **check a match**, never recover the original.

**Simple Explanation**

If you encrypted passwords instead of hashing them, every password becomes recoverable the instant the key leaks. One compromised key hands over every user's password in plaintext, immediately.

Hashing has no equivalent failure. Even a full database leak only exposes hashes, which — with bcrypt's deliberate slowness and per-user salt — are expensive to crack even offline.

The mental model:

- **Encryption** is for data you need to **read back** later (a stored SSN you display to the user).
- **Hashing** is for data you only need to **compare against** later (a password you check at next login).

**Example**

```ruby
class User < ApplicationRecord
  encrypts :ssn, deterministic: true     # queryable via equality, but same plaintext -> same ciphertext
  encrypts :bank_account_number          # non-deterministic (default) — stronger, but not queryable
end
```

```yaml
# config/credentials.yml.enc (edited via `rails credentials:edit`) — never in plain source
active_record_encryption:
  primary_key: <generated>
  deterministic_key: <generated>
  key_derivation_salt: <generated>
```

---

### 364. What's the difference between hashing and encryption?

**Short Answer**

mTLS (mutual TLS) means **both** sides present and verify a certificate during the handshake — not just the server.

It's used for service-to-service authentication inside a private network, or between tightly-coupled partners.

**Simple Explanation**

Ordinary TLS only authenticates the **server** to the client. The client proves who it is separately, afterwards, with something like an API key.

mTLS moves client authentication into the handshake itself. The server validates the client's certificate against a trusted CA before the connection is even established.

That's common in internal microservice architectures and service meshes, where a leaked API key would be a bigger risk than a leaked certificate, and where you want the transport layer itself to enforce identity.

The trade-off is certificate management — issuing, distributing, and rotating certificates for every service is real operational work, which is why service meshes usually automate it.

**Example**

```ruby
# WRONG — reversible; a single key leak exposes every user's password in plaintext
encrypted = Encryptor.encrypt(password, key: ENV["APP_KEY"])

# RIGHT — one-way, salted per-user, deliberately slow (bcrypt's cost factor)
BCrypt::Password.create(password)   # what has_secure_password does under the hood
```

---

### 365. What is mTLS, and when would you use it?

**Short Answer**

PII (Personally Identifiable Information) is any data that can identify a specific person — name, email, phone, address, government ID, sometimes IP address.

Typical handling: encrypt at rest, restrict access by role, support deletion on request, and **never log it in plaintext**.

**Simple Explanation**

The practical checklist:

1. Does this field identify a real person?
2. Is it encrypted at rest?
3. Is access scoped by role, rather than available to every engineer with database read access?
4. Can it actually be **deleted** when requested — not just soft-deleted while remaining in every backup and log line forever?
5. Is it accidentally leaking into logs, error trackers, or analytics events?

That last one bites teams constantly. Rails' `filter_parameters` handles request params, turning them into `[FILTERED]` in logs.

But it does **not** cover explicit `logger.info` calls elsewhere in your code, or data you send to a third-party crash reporter. Those need the same discipline applied by hand.

**Example**

```ruby
# Faraday client configured to present a client certificate for internal service-to-service calls
conn = Faraday.new(url: "https://internal-payments.svc") do |f|
  f.ssl.client_cert = OpenSSL::X509::Certificate.new(File.read("client.crt"))
  f.ssl.client_key  = OpenSSL::PKey::RSA.new(File.read("client.key"))
  f.ssl.ca_file     = "/etc/ssl/certs/internal-ca.pem"
end
```

---

### 366. What is PII, and what handling does it typically require?

**Short Answer**

- `HttpOnly` — JavaScript can't read the cookie, so an XSS bug can't steal the session token directly.
- `Secure` — only sent over HTTPS.
- `SameSite` — limits when it's attached to cross-site requests (CSRF protection).

You rotate the session ID on login to defeat **session fixation**.

**Simple Explanation**

Each flag closes a specific hole.

Without `HttpOnly`, any successful XSS can read `document.cookie` and send the session token to an attacker. With it, the cookie exists but JavaScript literally cannot see it.

Without `Secure`, the cookie could be sent over plain HTTP where it's trivially sniffable.

`SameSite` (Lax or Strict) stops the cookie being attached to most cross-site requests, which is one of the layers defending against CSRF.

**Session fixation** is a separate issue: an attacker sets or discovers a session ID **before** the victim logs in, then once the victim authenticates under that ID, the attacker's copy becomes a valid authenticated session.

Calling `reset_session` at login issues a brand-new session ID, so any pre-login ID an attacker planted becomes worthless.

**Example**

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [:password, :ssn, :credit_card, :email]
# log line becomes: Parameters: {"email"=>"[FILTERED]", "password"=>"[FILTERED]"}
# Note: this filters *params* logging specifically — it does not protect explicit `logger.info`
# calls elsewhere in the codebase, which need the same discipline applied by hand.
```

---

### 367. What do the cookie security flags protect against, and why rotate the session ID on login?

**Short Answer**

- **Content-Security-Policy (CSP)** — restricts which scripts and resources may load. A safety net for when escaping fails.
- **X-Content-Type-Options: nosniff** — stops the browser guessing a file's type into something executable.
- **X-Frame-Options / frame-ancestors** — prevents clickjacking by controlling whether your page can be iframed.
- **Referrer-Policy** — limits how much of your URL leaks to third parties.

**Simple Explanation**

**CSP** works by allow-listing where scripts and styles may come from. If an XSS bug slips past your escaping, a good CSP makes the browser **refuse to execute** the injected script unless it carries a valid **nonce** (a random per-request token).

That's why CSP is called defense in depth — it's a backup for when escaping fails, never a replacement for escaping properly. And avoid `'unsafe-inline'`, which allows any inline script and defeats most of the point.

**Clickjacking** is a different attack: a malicious site iframes your page invisibly and tricks the user into clicking something ("click to win a prize") positioned over your real "Transfer funds" button. `X-Frame-Options: DENY` stops your page being framed at all.

**nosniff** closes a narrower hole where a browser ignores the declared `Content-Type`, guesses from the contents, and ends up executing something served as an upload.

**Referrer-Policy** matters because the full URL — including query params that might contain tokens or IDs — is sent as the `Referer` header when a user clicks off your page.

**Example**

```ruby
Rails.application.config.session_store :cookie_store,
  key: "_app_session",
  httponly: true,                         # JS can't read it — blocks XSS-based token theft
  secure: Rails.env.production?,          # only sent over HTTPS
  same_site: :lax                          # not attached on most cross-site requests — CSRF mitigation

def create
  user = User.find_by(email: params[:email])
  if user&.authenticate(params[:password])
    reset_session               # issues a brand-new session id — defeats session fixation
    session[:user_id] = user.id
  end
end
```

---

### 368. What security headers should a Rails app set, and what does each defend against?

**Short Answer**

- **HttpOnly cookie** — immune to XSS token theft, but needs CSRF protection since the browser attaches it automatically.
- **localStorage** — no CSRF exposure, but **any** XSS can read and steal the token.

There's no risk-free option. You're choosing which attack you defend harder against.

**Simple Explanation**

With an `HttpOnly` cookie, even a page with an XSS bug can't read the token. But because the browser attaches it to every matching request automatically, you must defend against CSRF with an authenticity token and `SameSite`.

With `localStorage`, nothing is attached automatically, so a forged cross-site request gets nothing for free — CSRF isn't a concern. But there's no `HttpOnly` equivalent: any XSS on the page, however small, can read the token and send it away.

Either way, correct output escaping and a solid CSP remain load-bearing. The storage choice only decides which **additional** control does the heavy lifting.

For non-token data, pick by shape and lifetime:

- **Cookies** — small (~4KB), sent automatically, can be `HttpOnly`. Right for session tokens.
- **localStorage** — several MB, persists across tabs and restarts. Right for non-sensitive UI preferences.
- **sessionStorage** — same API, but one tab only, cleared when it closes. Right for short-lived per-tab state.
- **IndexedDB** — a real in-browser database with a large capacity and an async API. Right for offline datasets.

**Example**

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self
  policy.script_src  :self, -> (request) { "'nonce-#{request.session[:csp_nonce]}'" }
  policy.style_src   :self
  policy.img_src     :self, :data, "https://cdn.example.com"
  policy.object_src  :none
  policy.frame_ancestors :none
end
Rails.application.config.content_security_policy_nonce_generator = ->(request) { SecureRandom.base64(16) }
```

```erb
<%= javascript_tag nonce: true do %>
  console.log("inline script allowed only because it carries the per-request nonce");
<% end %>
```

```ruby
# The rest of the standard set
config.action_dispatch.default_headers = {
  "X-Content-Type-Options" => "nosniff",
  "X-Frame-Options" => "DENY",
  "Referrer-Policy" => "strict-origin-when-cross-origin"
}
```

---

### 369. Where should you store an auth token on the frontend, and how do the browser storage options compare?

**Short Answer**

- **Devise** — a full authentication framework for your own username/password login.
- **OmniAuth** — a standard adapter for logging in **through** a third party (Google, GitHub).

They're not competitors. Devise uses OmniAuth internally for social login.

**Simple Explanation**

If a user authenticates against **your** database with a password, that's Devise's job.

If they authenticate through Google or GitHub — where your app never sees a password, just a signed assertion that the provider verified this identity — that's OmniAuth's job.

OmniAuth's value is normalisation. Dozens of providers have slightly different OAuth flows, and OmniAuth puts them all behind one consistent callback (`request.env["omniauth.auth"]`), so your code has no provider-specific logic.

Most apps supporting both end up using Devise's `:database_authenticatable` for passwords and its `:omniauthable` module wired to OmniAuth strategies, sharing one `User` model.

Note OmniAuth 2 requires the request phase to be a POST with CSRF protection, which is why `omniauth-rails_csrf_protection` is usually in the Gemfile.

**Example**

```ruby
# HttpOnly cookie approach — Rails sets this automatically via the session store;
# the frontend never touches the token directly, the browser handles attachment.
Rails.application.config.session_store :cookie_store, httponly: true, secure: true, same_site: :lax
```

**— Authentication Libraries & Identity —**

### 370. Devise vs OmniAuth — what's the difference, and when do you need OmniAuth?

**Short Answer**

- **SAML** — older, XML-based, dominant in **enterprise** single sign-on (one corporate identity provider like Okta logging you into many internal apps).
- **OAuth** (with **OIDC** on top for identity) — newer, JSON/REST-based, behind modern "log in with Google" and API authorization.

**Simple Explanation**

Both let a trusted third party vouch for a user's identity, but they come from different eras.

**SAML** exchanges signed XML documents (assertions) between an Identity Provider and your app. It's deeply embedded in enterprise tooling — think an employee reaching dozens of internal SaaS tools through one corporate session.

**OAuth 2.0** was originally designed for **authorization** — letting an app access your data on another service without giving it your password. **OIDC** (OpenID Connect) is the identity layer built on top that turns it into a proper login protocol, adding an ID token carrying identity claims.

In Rails: SAML integration usually means `ruby-saml` or `omniauth-saml` configured against each enterprise customer's IdP metadata. OAuth/OIDC means the various `omniauth-*` provider gems.

**Example**

```ruby
# Gemfile
gem "devise"
gem "omniauth-google-oauth2"
gem "omniauth-rails_csrf_protection"   # OmniAuth 2 requires POST + CSRF protection on the request phase

# config/initializers/devise.rb
config.omniauth :google_oauth2, ENV["GOOGLE_CLIENT_ID"], ENV["GOOGLE_CLIENT_SECRET"]

class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :omniauthable, omniauth_providers: [:google_oauth2]
end

class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def google_oauth2
    user = User.from_omniauth(request.env["omniauth.auth"])
    sign_in_and_redirect user
  end
end
```

---

### 371. How does SSO/SAML differ from OAuth?

**Short Answer**

MFA means requiring a second proof of identity beyond a password.

TOTP (an authenticator app) is stronger than SMS because SMS is vulnerable to **SIM swapping** — an attacker social-engineers the carrier into moving the victim's number to a new SIM and receives the codes directly.

**Simple Explanation**

**TOTP** works by having both the server and your authenticator app independently compute a code from a shared secret plus the current time. Nothing travels over the network to generate it, so there's no interception point beyond the initial QR code setup.

**SMS 2FA** depends on the security of the phone network and your carrier's identity checks for SIM changes — both weaker links. A SIM swap redirects the victim's texts to the attacker's phone, no malware or phishing needed. SS7 network-level interception is another known, if rarer, attack.

TOTP isn't perfectly phishing-proof either — a fake login page can relay the code in real time — but it removes the telecom attack surface entirely. Hardware keys (WebAuthn/FIDO2) go further and are phishing-resistant.

**Example**

```ruby
# Gemfile: gem 'omniauth-saml'
# config/initializers/devise.rb — SAML config points at the enterprise customer's IdP
config.omniauth :saml,
  idp_sso_target_url: "https://customer-idp.example.com/sso",
  idp_cert_fingerprint: ENV["SAML_IDP_FINGERPRINT"],
  issuer: "https://yourapp.com",
  name_identifier_format: "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
```

---

### 372. What's MFA, and why is SMS-based 2FA considered weaker than TOTP?

**Short Answer**

Beyond CSRF and cookie flags: a **Content-Security-Policy** to limit what can execute, **`X-Content-Type-Options: nosniff`**, **`X-Frame-Options`** against clickjacking, and a **`Referrer-Policy`** to limit URL leakage.

**Simple Explanation**

These are the same four headers covered above, so the practical point here is how you actually ship them in Rails.

CSP is configured in `config/initializers/content_security_policy.rb`, where you declare allowed sources per resource type (`script_src`, `style_src`, `img_src`). Use a **nonce generator** so your own inline scripts can be allowed individually, rather than opening up `'unsafe-inline'` for everything.

`frame_ancestors :none` inside the CSP is the modern equivalent of `X-Frame-Options: DENY`; setting both is fine for older browser support.

The other three are plain response headers set through `config.action_dispatch.default_headers`.

Worth knowing: CSP can run in **report-only** mode first, so you can see what it *would* block before enforcing it. That's how you roll one out on an existing app without breaking pages.

**Example**

```ruby
# Gemfile: gem 'devise-two-factor'
class User < ApplicationRecord
  devise :two_factor_authenticatable, otp_secret_encryption_key: ENV["OTP_SECRET_KEY"]
end

# Enrollment
user.otp_secret = User.generate_otp_secret
user.otp_required_for_login = true
user.save!
# user.otp_secret is rendered as a QR code the user scans into their authenticator app

# Verifying a code from the app at login
if user.validate_and_consume_otp!(params[:otp_attempt])
  sign_in(user)
else
  render_error("Invalid authentication code")
end
```

## Git

**— Branching Strategy —**

### 373. What is Git Flow, and what kind of team does it fit?

**Short Answer**

Git Flow uses two permanent branches — `main` and `develop` — plus temporary `feature/*`, `release/*`, and `hotfix/*` branches.

It fits teams with **scheduled releases** or multiple supported versions in production at once.

**Simple Explanation**

`develop` is where finished features accumulate. `main` only ever holds what's actually been released, with each commit tagged.

Feature branches fork off `develop` and merge back into it. When you're ready to ship, you cut a `release/*` branch — bug fixes only, no new features — then merge it into **both** `main` and `develop` and tag it.

`hotfix/*` branches fork off `main` directly, so you can patch production without dragging in half-finished work from `develop`.

That process overhead pays off when releases are infrequent and discrete: versioned enterprise software, mobile apps waiting on app-store review, firmware.

It's overkill for a SaaS app deploying ten times a day — the `develop` and `release` branches just add delay between writing code and users seeing it.

**Example**

```bash
# Git Flow shape, illustrated manually (no plugin needed)
git checkout develop
git checkout -b feature/user-notifications
# ... work, commit, PR back into develop ...

git checkout develop
git checkout -b release/2.4.0
# stabilize: bug fixes only on this branch
git checkout main
git merge --no-ff release/2.4.0
git tag v2.4.0
git checkout develop
git merge --no-ff release/2.4.0
```

### 374. What is trunk-based development, and why does it fit continuous deployment better than Git Flow?

**Short Answer**

Everyone merges **short-lived** branches into `main` constantly — often several times a day — and hides unfinished work behind feature flags instead of behind a long-lived branch.

That fits continuous deployment because `main` is always releasable, so there's no separate stabilization step.

**Simple Explanation**

The core idea: integration is the risky part of merging, and the way to make it safe is to do it in **small frequent doses** rather than one big dose at the end.

A branch that lives a day or two can't drift far from `main`, so conflicts stay small and CI catches them immediately rather than weeks later in a giant merge.

Because a half-built feature might be sitting on `main` at any moment, teams use **feature flags** — a runtime toggle that hides the code path from users until it's ready.

That's the key insight: it decouples "merged" from "released", which is exactly what continuous deployment needs. Every commit on `main` should be deployable, whether or not every feature on it is switched on.

**Example**

```bash
# Trunk-based: branch lives hours, not weeks
git checkout main
git pull
git checkout -b add-export-button
# small change, gated behind a flag
git commit -am "Add CSV export button behind :csv_export flag"
git push -u origin add-export-button
# open PR, get it merged same day, delete the branch
git checkout main && git pull && git branch -d add-export-button
```

### 375. What is GitHub Flow, and where does it sit between Git Flow and trunk-based development?

**Short Answer**

`main` plus short-lived feature branches, pull requests, and deploy-on-merge. No `develop`, no `release` branches.

It's simpler than Git Flow, and slightly less strict than pure trunk-based development because integration happens through a reviewed PR.

**Simple Explanation**

Every change starts as a branch off `main`, opens as a pull request for review and CI, and merges straight back into `main` — which is what gets deployed.

Compared to Git Flow, there's no intermediate branch collecting unreleased work, so you avoid all that ceremony.

Compared to strict trunk-based development, branches are allowed to live a little longer (a few days, covering the review cycle) and integration is explicit through the PR rather than continuous pushes to `main`.

In practice, most product teams doing continuous deployment are really doing GitHub Flow with feature flags layered on for anything too big to land in one PR.

**Example**

```bash
git checkout main && git pull
git checkout -b fix-checkout-timeout
# commit work, push, open PR against main
git push -u origin fix-checkout-timeout
gh pr create --base main --fill
# merge triggers CI/CD to deploy main automatically
```

### 376. What is a release branch for, and when do you cut one instead of deploying straight from `main`?

**Short Answer**

A release branch **freezes the scope** for one release so it can be stabilized and QA'd, while feature work keeps landing on `main` unaffected.

Cut one when releases are discrete events (mobile apps, versioned packages, on-prem software). Skip it if you deploy from `main` many times a day.

**Simple Explanation**

Once you branch `release/2.4.0` off `main`, that branch only accepts bug fixes for the release. New features keep merging into `main` normally.

That gives you a stable target to run QA against, submit to an app store, or hand to a customer — without development having to pause.

If a critical bug is found during stabilization, you fix it on the release branch and **merge that fix back into `main`** too, so it isn't lost in the next release.

If you're deploying continuously, there's no freeze window to protect — every commit is already meant to be production-ready — so a release branch is just overhead.

**Example**

```bash
git checkout main
git checkout -b release/2.4.0
# only cherry-picked bug fixes land here from now on
git cherry-pick <sha-of-bugfix-on-main>
git tag v2.4.0
git push origin release/2.4.0 v2.4.0
```

### 377. How does a hotfix strategy work when `main` has already moved past your last release tag?

**Short Answer**

Branch the hotfix **off the release tag**, not off `main`, so you ship only the fix without any unreleased work.

Then cherry-pick (or merge) that fix forward into `main` so the bug doesn't come back in the next release.

**Simple Explanation**

Say production is running `v1.2.0` but `main` has three sprints of unreleased work on it.

Branching the hotfix from `main` would ship all of that unfinished work along with your fix — not acceptable for an urgent patch.

So you branch from the `v1.2.0` tag, make the minimal fix, tag it `v1.2.1`, and deploy that.

Now the fix exists in a place that can drift from `main`. To prevent the same bug reappearing in the next release, you cherry-pick that commit forward into `main` — and into any active release branch too.

**Example**

```bash
git checkout -b hotfix/1.2.1 v1.2.0
# fix the bug, minimal diff
git commit -am "Fix null pointer in invoice PDF generation"
git tag v1.2.1
git push origin hotfix/1.2.1 v1.2.1

# now bring the fix forward so main doesn't regress
git checkout main
git cherry-pick <sha-of-fix>
git push origin main
```

### 378. Why do branch naming and PR conventions matter beyond just tidiness?

**Short Answer**

CI pipelines, changelog generators, and ticket-linking bots often **pattern-match on branch and commit names**.

An inconsistent name can silently skip a pipeline stage or break traceability — so it's not just tidiness.

**Simple Explanation**

A CI config might only trigger deploy previews for branches matching `feature/*`. A bot might auto-link a PR to a ticket because the branch name starts with `TICKET-123-`. Release tooling might build a changelog by scanning commit prefixes like `fix:` or `feat:` (Conventional Commits).

If someone names a branch `my-fix-thing` instead of `fix/TICKET-123-thing`, none of that fires. No linked ticket, no changelog entry, maybe no CI at all.

At senior level this matters because you're often the one debugging **why** a pipeline didn't run — and "someone didn't follow the naming convention" is a very common answer.

**Example**

```bash
# Convention: <type>/<ticket-id>-<short-desc>
git checkout -b fix/TTP-4821-checkout-timeout-retry
git commit -m "fix(checkout): retry payment gateway timeout up to 3x"
# CI config keys off the branch prefix and commit type:
#   on: push: branches: ['feature/**', 'fix/**']
```

### 379. What's the trade-off of long-lived feature branches versus small, frequent PRs?

**Short Answer**

- **Long-lived branches** defer all the integration pain to one big risky merge at the end.
- **Small frequent PRs** integrate constantly, so conflicts stay tiny — but you need feature flags to ship incomplete work safely.

**Simple Explanation**

The longer a branch lives without merging `main` in, the more the two diverge. Other people touch the same files, assumptions go stale, and by the time you merge you're resolving a sprawling conflict — or worse, getting a conflict-free merge that's still semantically wrong.

Small PRs flip that. Each one is reviewable in isolation, CI validates it against a nearly-current `main`, and if two people touch the same area the conflict is a few lines.

The cost is that a big feature can't land as one clean PR. It has to be sliced into safe incremental pieces, often behind a flag, which takes more upfront thought about how to decompose the work.

**Example**

```bash
# Long-lived branch: divergence grows every day it isn't merged
git log --oneline main..feature/big-redesign | wc -l
git log --oneline feature/big-redesign..main | wc -l   # how far behind it is

# Small PR habit: rebase onto latest main before opening/continuing work
git fetch origin
git rebase origin/main
```

**— Git Internals & Daily Commands —**

### 380. `git merge` vs `git rebase` — what's the actual difference?

**Short Answer**

- **Merge** — creates a new commit joining two histories. Both branches' original commits stay untouched.
- **Rebase** — replays your commits **one by one onto a new base**, giving each a new SHA and producing a linear history.

**Simple Explanation**

With `git merge`, if the branches have diverged, Git creates a merge commit with two parents. The history shows what actually happened, forks and all.

With `git rebase main`, Git takes your commits off your branch, moves your branch to the tip of `main`, and re-applies your commits on top one at a time. Each one is technically a **brand-new commit** with a new SHA, even though the content looks the same.

The payoff is a clean linear history that's easier to read and easier to `bisect` through.

The cost is that you've rewritten history — which is only safe if nobody else has built on the old commits. After rebasing a pushed branch you need `git push --force-with-lease`.

**Example**

```bash
git checkout feature/x
git fetch origin
git rebase origin/main        # replay feature/x commits on top of latest main
# resolve any conflicts, then:
git rebase --continue
git push --force-with-lease   # required: history was rewritten
```

### 381. When is it unsafe to rebase?

**Short Answer**

Never rebase commits that have already been **pushed and pulled by someone else**.

Rewriting history other people have built on gives them duplicate commits and broken histories.

**Simple Explanation**

Rebasing changes commit SHAs.

If you rebase a branch only you have, that's fine — nobody else has a copy of the old commits.

But if you've pushed a branch and a teammate has pulled it, branched from it, or is tracking it, and then you rebase and force-push, their local history now refers to commits that no longer exist upstream. Their next pull either fails or silently duplicates every commit as a "new" one with identical content.

The rule of thumb: **rebase freely on your own unshared work; once a branch is shared — especially `main` — only ever add commits to it.** Merge or revert, don't rebase or force-push.

**Example**

```bash
# Safe: your own branch, nobody else has pulled it
git rebase -i origin/main

# Unsafe: main is shared — never do this
# git checkout main
# git rebase feature/x     # DON'T — rewrites history everyone relies on
# git push --force origin main
```

### 382. What is `git cherry-pick`, and when would you actually use it?

**Short Answer**

`git cherry-pick` applies the changes from **one specific commit** onto your current branch, without merging the whole branch it came from.

The classic use is a hotfix that needs to land on both `main` and a release branch.

**Simple Explanation**

Say you fix a critical bug on `release/2.4.0`. `main` has since moved on with unrelated features, so you can't merge the whole release branch into `main` without pulling in things that shouldn't be there.

Instead you cherry-pick just that one commit SHA onto `main`, replaying its diff as a new commit.

It's a scalpel where `merge` is a transfusion. Useful for hotfixes, backporting a fix to an older supported version, or pulling one useful commit off an abandoned branch.

If the surrounding code has changed on the target branch, you may need to resolve conflicts and then `git cherry-pick --continue`.

**Example**

```bash
git log release/2.4.0 --oneline -5      # find the fix's SHA
git checkout main
git cherry-pick a1b2c3d
# if it touches lines that changed differently on main:
git status                              # resolve conflicts
git cherry-pick --continue
```

### 383. What is `git bisect`, and how do you use it to find a regression?

**Short Answer**

`git bisect` binary-searches your history between a known-good and known-bad commit, checking out candidates for you to test until it finds the exact commit that introduced the bug.

**Simple Explanation**

You tell it a commit where the bug definitely wasn't there ("good") and one where it definitely is ("bad").

Git checks out the commit halfway between. You test it and say `good` or `bad`. Git halves the remaining range again.

For 1,000 commits that finds the culprit in about 10 steps instead of 1,000.

And if you have a script that can decide pass/fail on its own (a test, a curl check — anything returning exit code 0 or 1), `git bisect run` automates the whole loop with no manual testing at all.

Always finish with `git bisect reset` to return to where you started.

**Example**

```bash
git bisect start
git bisect bad HEAD                # current commit has the bug
git bisect good v1.4.0              # this tag was fine

# manual: test each checkout, then mark it
git bisect good   # or: git bisect bad

# or fully automated with a script that exits 0/1:
git bisect run bin/rails test test/models/invoice_test.rb
git bisect reset                    # return to original HEAD when done
```

### 384. What is `git reflog`, and how does it save you after a bad `reset --hard`?

**Short Answer**

`git reflog` is a **local** log of every position `HEAD` and your branches have pointed to.

So even after a destructive `reset --hard`, you can find the "lost" commit's SHA in the reflog and point your branch back at it.

**Simple Explanation**

Commits you "lose" with `reset --hard`, a botched rebase, or a deleted branch aren't actually deleted right away. Git just stops pointing anything at them. They stay around until garbage collection eventually removes them, typically after weeks.

`git reflog` shows the history of ref movements on your machine, so you can see "HEAD@{1} was the state before that bad reset" and recover it with `git reset --hard HEAD@{1}`, or save it as a new branch.

Two things to remember: it's **local only** — not pushed or shared — and it's the single most useful safety net in Git. Worth knowing before you need it in a panic.

**Example**

```bash
git reset --hard HEAD~3     # oops, meant to only undo 1 commit
git reflog                  # shows HEAD@{0} reset..., HEAD@{1} commit: ..., etc.
git reset --hard HEAD@{1}   # point back at the state before the bad reset
# or, if you just need the commit, not the branch position:
git branch recovered-work HEAD@{1}
```

### 385. `git reset` (soft/mixed/hard) vs `git revert` — what's the difference, and when is each safe?

**Short Answer**

- **`reset`** — moves your branch pointer backward. It **rewrites history**, so it's dangerous on shared branches.
  - `--soft` — keeps changes staged
  - `--mixed` (default) — keeps changes unstaged
  - `--hard` — discards changes entirely
- **`revert`** — creates a **new commit** that undoes an earlier one. History stays intact, so it's safe on shared branches.

**Simple Explanation**

The three `reset` modes differ only in how much they touch. `--soft` moves the pointer. `--mixed` also unstages. `--hard` also wipes your working directory — that's the one that can destroy uncommitted work.

All three change what the branch points to, so using them on a branch others have pulled causes the same problems as rebasing.

`git revert` removes nothing. It computes the inverse of a commit's changes and applies that as a new commit on top. History stays truthful — you can see both the mistake and its correction.

That's exactly why `revert` is the right tool for undoing something on `main` or any shared branch.

**Example**

```bash
# reset: rewrites history — fine on a local/unshared branch
git reset --soft HEAD~1    # undo last commit, keep changes staged
git reset --mixed HEAD~1   # undo last commit, keep changes unstaged
git reset --hard HEAD~1    # undo last commit, discard changes entirely

# revert: safe on shared branches — adds a new commit
git revert a1b2c3d         # undoes that commit's changes without rewriting history
```

### 386. When would you reach for `git stash` instead of just making a quick WIP commit?

**Short Answer**

Use `stash` when you need a clean working tree **right now** — an urgent bug, a branch switch — without leaving a throwaway commit in history.

Use a WIP commit when the work is worth saving more durably, so you can push it as a backup or move machines.

**Simple Explanation**

`git stash` shelves your uncommitted changes (staged and unstaged) and gives you back a clean working directory, without touching branch history.

That's ideal for "I need to context-switch immediately and come back to this". A P1 comes in mid-refactor, so you stash, fix the bug, then `git stash pop` to pick up exactly where you left off.

A WIP commit is better when you want the work pushed somewhere safe, or you're about to try something risky and want a named point to return to.

One practical caution: stash entries are easy to forget about and aren't pushed anywhere. Anything you genuinely care about is safer as a real commit. And use `git stash push -m "message"` so you can tell entries apart later.

**Example**

```bash
# mid-refactor, urgent bug comes in
git stash push -m "WIP: extracting InvoiceCalculator"
git checkout main
git checkout -b hotfix/nil-currency
# ...fix, commit, push...
git checkout feature/invoice-refactor
git stash pop
```

### 387. What are the three trees in Git, and how do `add`/`commit`/`checkout` move data between them?

**Short Answer**

- **Working directory** — your actual files.
- **Index / staging area** — what's queued for the next commit.
- **HEAD** — the commit history.

`git add` moves working directory → index. `git commit` moves index → HEAD. `git restore` / `checkout` move data back the other way.

**Simple Explanation**

This model explains nearly every confusing Git moment.

Edit a file and the change exists only in your working directory — `git status` calls it "not staged".

Run `git add file.rb` and **that exact version** is copied into the index. It's now "staged", and that's what will go into the next commit, regardless of further edits you make afterwards. (That last part surprises people: edit again after staging, and you have two different versions in play.)

Run `git commit` and the index's contents become a permanent snapshot, and `HEAD` moves to point at it.

Going backwards: `git restore --staged file.rb` unstages without touching your edits. `git restore file.rb` overwrites your working copy from the index or HEAD, **discarding** local edits.

**Example**

```bash
echo "# note" >> app/models/user.rb   # working directory changed
git status                            # shows it as unstaged

git add app/models/user.rb            # working dir -> index (staged)
git status                            # shows it as staged

git commit -m "Add note to User model"  # index -> HEAD (new commit)

git restore --staged some_file.rb     # index -> unstage (keep edits)
git restore some_file.rb              # HEAD/index -> working dir (discard edits)
```

### 388. Fast-forward merge vs a merge commit — when does Git choose each, and how does `--no-ff` change that?

**Short Answer**

Git **fast-forwards** — just moves the branch pointer, no new commit — when the target branch hasn't moved since you branched off.

If both have moved, Git must create a **merge commit** with two parents.

`--no-ff` forces a merge commit even when a fast-forward was possible.

**Simple Explanation**

If `main` hasn't changed since you branched, there's nothing to actually combine — Git can just slide `main` forward to your branch's latest commit.

If `main` **has** moved (someone else merged something), Git creates a merge commit representing that fork and rejoin.

Some teams pass `--no-ff` deliberately, so every feature always produces a merge commit. That keeps a visible marker in `git log --graph` of where a feature was integrated, and it means you can revert an entire feature with one `git revert -m 1 <merge-sha>`.

**Example**

```bash
# fast-forward (default, when possible): main just moves the pointer
git checkout main
git merge feature/small-fix

# force a merge commit even if a fast-forward was possible
git checkout main
git merge --no-ff feature/small-fix -m "Merge feature/small-fix"
```

### 389. Why would a team squash a feature branch's commits into one before merging to `main`?

**Short Answer**

Squashing collapses every commit on a branch — including "fix typo", "address review comments", "oops" — into **one commit** on `main`.

You get a readable mainline history, at the cost of losing the branch's step-by-step detail.

**Simple Explanation**

During development, a branch's commit history is a personal work log, not permanent documentation.

Squashing applies the net effect of the whole branch as one clean commit, usually titled after the PR.

That makes `git log` on `main` read like a list of features and fixes. It also makes `git bisect` and `git blame` far more useful: one commit per logical change with a clear message, instead of `blame` pointing at "fix lint" as the origin of a bug.

The trade-off: you can't see how the feature evolved commit by commit on `main`. If you need that, you go look at the closed PR.

Most teams just use the platform's "Squash and merge" button, which does exactly this.

**Example**

```bash
# manually
git checkout main
git merge --squash feature/x
git commit -m "Add invoice export (PR #482)"

# most teams just use the platform's button, e.g. GitHub's
# "Squash and merge" option on the PR, which does the same thing
```

### 390. What is `git worktree`, and how does it differ from `git clone` or just stashing/checking out?

**Short Answer**

`git worktree` checks out a second branch into its **own directory** while sharing the same repository — same objects, same remotes, same config.

That's different from `git clone` (which duplicates everything) and from stashing (which forces you to have only one branch checked out).

**Simple Explanation**

Normally a repository has one working directory, so switching branches changes your files under you — anything uncommitted has to be stashed or committed first.

`git worktree add ../review-pr-482 origin/some-branch` creates an **additional** working directory linked to the same history. Now you have `main` checked out in one folder and a PR branch in another, with no duplicated repository and nothing stashed.

The real scenario: you're mid-feature with a messy working directory and someone needs a PR reviewed right now. Instead of stashing (risky) or re-cloning (wasteful, and you'd have to reconfigure it), you add a worktree, review it in its own folder, and `git worktree remove` it when done.

Your original working directory never moved.

**Example**

```bash
# from your main repo, still mid-feature with uncommitted changes
git worktree add ../review-pr-482 origin/feature/csv-export
cd ../review-pr-482
# review, run tests, etc. — your original working dir is untouched

cd -
git worktree remove ../review-pr-482   # clean up when done
```

### 391. What does your "safe Git practices" checklist look like?

**Short Answer**

- Never force-push a shared branch without `--force-with-lease`.
- Never rebase commits that are already shared.
- Always read **both sides** of a conflict before resolving it.
- Never blindly `git add -A` — review what you're staging.

**Simple Explanation**

These are the habits that separate "confident with Git" from "dangerous with Git" on a team.

`--force-with-lease` refuses to push if the remote has commits you haven't seen yet. A plain `--force` will happily destroy a teammate's work you didn't know existed.

Rebasing or resetting history that others have pulled creates duplicate commits and broken histories downstream. Treat anything pushed to a shared branch as immutable.

During a merge conflict, resolving by just picking "ours" or "theirs" without reading both sides can silently discard a real fix someone else made. Understand what each side was trying to do first.

And `git add -A` stages everything indiscriminately — that's how `.env` files, credentials, and build artifacts end up committed. Reviewing `git status` and `git diff --staged` before committing, plus a solid `.gitignore`, avoids that whole class of mistake.

**Example**

```bash
# safe force-push: aborts if remote has commits you don't have locally
git push --force-with-lease origin feature/x

# check exactly what you're about to stage before committing
git status
git diff                     # review unstaged changes
git add app/models/user.rb   # stage specific files, not blindly -A
git diff --staged            # final check before commit
```

## Behavioral / Senior Engineer Questions

**— Getting to Know You —**

### 392. Tell me about yourself.

**Framework**

**Present → Past → Why this role.** Start with where you are now, briefly trace how you got there, and finish with why this role is the logical next step.

Aim for 60–90 seconds. This is a trailer, not the whole film.

**What a strong answer covers**

- A clear statement of what you own right now — the kind of system, the team, the scope — not just a job title.
- A trajectory, not a full CV. Pick the two or three moves that show a clear line toward seniority: growing scope, more ownership, deeper technical work.
- Enough specifics to sound like a real career rather than a template — a domain, a stack, the kind of problem you gravitate toward.
- A closing line connecting your path to **this** role specifically. That shows you've thought about why here and why now, instead of reciting the same answer everywhere.
- Restraint. A good answer leaves the interviewer wanting to ask a follow-up, not feeling like they've just heard your autobiography.

### 393. Explain your current project — what it does, your role, and the interesting technical problem in it.

**Framework**

**Context → Your role → The interesting technical problem → Outcome.** Treat it as a short technical story the interviewer can dig into.

**What a strong answer covers**

- One sentence of business context — what the product does and who uses it — before any implementation detail.
- What **you** actually owned, said in "I" terms where that's honest. A vague "we" hides your individual contribution.
- One genuinely interesting technical problem — a scaling issue, a tricky data model, a concurrency bug, a legacy migration — rather than a list of features. This is the part they'll probe hardest.
- The trade-offs you considered and why your approach won. That shows judgement, not just execution.
- A concrete outcome where you have one: latency improved, incidents dropped, a migration completed with zero downtime. Numbers if you have them.
- Only describe things you can defend under "why not X instead?" Don't mention a technology you can't discuss in depth.

### 394. Explain a difficult production issue you solved.

**Framework**

**STAR** (Situation, Task, Action, Result), weighted toward **Action** (how you diagnosed it) and **Result** (the fix plus what changed afterwards to stop it recurring).

**What a strong answer covers**

- Severity and blast radius up front — who was affected and how urgently — so the stakes are clear.
- A calm, methodical diagnostic process: logs, metrics, error tracking, recent deploys. Not "I just knew" or random guessing.
- The actual **root cause**, not just the symptom you patched. That's the difference between real understanding and surface-level firefighting.
- Communication during the incident: who you kept informed, how status updates worked, whether you escalated.
- Follow-through **after** the fix: added monitoring, a regression test, a runbook entry, or a process change. That's what shows ownership rather than just stopping the pain.

### 395. Tell me about a performance problem you solved.

**Framework**

**STAR**, with explicit numbers **before and after**. A performance story without measurements isn't convincing.

**What a strong answer covers**

- How you knew it was a real problem — profiling data, APM metrics, user-reported latency — rather than a hunch.
- How you found the actual bottleneck: N+1 queries, a missing index, an inefficient algorithm, memory pressure, a synchronous call that should have been async. Emphasise that you measured rather than guessed.
- The specific fix, and why it addressed the cause rather than hiding the symptom.
- Quantified before and after: response time, query count, memory, throughput. This is the detail that separates a real story from a vague one.
- The trade-offs your fix introduced — a cache adds invalidation complexity, denormalization adds write complexity — and how you made sure nothing else regressed.

### 396. Tell me about a difficult bug you tracked down.

**Framework**

**STAR**, with the Action focused on your **investigative method**. This is really a story about how you think, not about what the bug turned out to be.

**What a strong answer covers**

- Why it was hard: intermittent, only reproducible under specific timing or load or data, or spanning multiple systems.
- A systematic narrowing process — bisecting recent changes, adding targeted logging, forming and testing one hypothesis at a time — rather than a lucky guess.
- The actual root cause and why it was non-obvious: a race condition, a timezone or locale bug, an encoding mismatch, a stale cache, undocumented third-party behavior.
- What you put in place so the same class of bug can't come back quietly — a regression test, an assertion, better logging at that boundary.
- Some honesty about the process. Good stories often include a wrong hypothesis you ruled out. A story that's too clean sounds rehearsed rather than real.

**— Working With Others —**

### 397. How do you review code? What do you look for first?

**Framework**

Describe your actual process and priorities, in the order you apply them. This isn't a STAR question.

**What a strong answer covers**

- **Correctness and safety first.** Does it do what it claims, are edge cases handled, is there a security or data-integrity risk? Before anything about style.
- **Blast radius.** Are there tests? Is the migration reversible? Does this touch something that fans out widely?
- Reading the PR description and linked ticket **before** judging the diff, so your feedback is grounded in what the author was trying to do.
- Readability and maintainability for whoever touches this next — weighed **after** correctness, not before.
- Clearly separating blocking issues from suggestions (marking optional ones as "nit:") so the author knows what actually needs to change.
- Treating **turnaround time** as part of the job. A technically excellent review that sits unread for three days still blocks the team.

### 398. How do you handle disagreements during code review?

**Framework**

A process answer. You can illustrate it with a short real example, but the core is your approach.

**What a strong answer covers**

- Leading with **technical reasoning**, not seniority. The argument should stand on its own regardless of who's making it.
- Citing a concrete trade-off — performance, readability, long-term maintenance cost, consistency with existing patterns — rather than personal preference.
- Separating objective issues (a real bug, a security gap) from subjective style, and deferring readily on style — ideally pointing at a linter or style guide instead of re-arguing taste.
- Knowing when to move a thread out of async comments and into a five-minute call. Some disagreements resolve instantly that way.
- Knowing when to get a third opinion versus when to defer to the author on their own code. You don't need to win every review.
- Following up if the disagreement revealed something systemic — adding a lint rule or documenting a convention — so the same debate doesn't repeat forever.

### 399. How do you prioritize technical debt against feature work?

**Framework**

A process answer, ideally with one real example of how you've made or pitched this trade-off.

**What a strong answer covers**

- Distinguishing debt that's **actively costing velocity or causing incidents** from debt that's merely untidy. Not all debt deserves the same urgency.
- Quantifying the cost where you can — time lost per sprint, incident frequency, onboarding friction — so the conversation is evidence-based rather than a matter of taste.
- Negotiating realistic mechanisms: a fixed share of capacity each sprint, or opportunistic cleanup alongside feature work. Not "stop all features for a rewrite".
- Translating debt into terms a non-engineering stakeholder can weigh: risk, delivery speed, reliability.
- Avoiding both extremes — ignoring debt until it causes a crisis, and over-investing in cleanup nobody asked for while shipping nothing.

### 400. How do you mentor junior developers?

**Framework**

A principles answer, ideally anchored with one real mentoring relationship or moment.

**What a strong answer covers**

- Adjusting to the person's actual level rather than using one approach for everyone.
- Asking guiding questions rather than just handing over the answer, so they build independent judgement instead of dependency on you.
- Giving feedback that's specific and timely — in code review and in regular one-to-ones — not saved up for a formal review cycle.
- Modelling good practice visibly: pairing, thinking out loud while debugging, explaining the **why** behind a design choice rather than just the what.
- Creating enough safety to ask questions and make mistakes, since fear of looking incompetent is what actually slows learning down.
- Balancing unblocking someone quickly against letting them struggle productively. Knowing the difference between useful friction and wasted time.
- Measuring success by their growing independence over time, not by how many of their problems you personally solved.

**— Operating Under Pressure and Ambiguity —**

### 401. How do you handle production incidents — as the actual responder, not just your process on paper?

**Framework**

A real-time operational sequence: **assess → stabilize → communicate → diagnose → resolve → follow up.**

**What a strong answer covers**

- **Stabilize before you fully root-cause.** Mitigate or roll back first when users are actively affected; understand the full why once the bleeding has stopped.
- A clear communication cadence: regular status updates, and clarity on who's coordinating versus who's heads-down fixing, so effort isn't duplicated.
- Judgement about when to escalate or pull others in versus keep digging alone. Knowing your own limits under pressure is a senior trait, not a weakness.
- Leaning on data — dashboards, logs, traces — instead of guessing, even when the pressure to "just try something" is high.
- Avoiding tunnel vision on the first hypothesis. Staying willing to drop a theory that isn't panning out rather than chasing it because you've already invested in it.
- Following through afterwards: writing the postmortem (blameless, focused on systems rather than people) and actually completing the action items instead of letting them rot in a backlog.

### 402. How do you approach an unfamiliar codebase?

**Framework**

Walk through your actual ramp-up method, roughly in the order you'd do it.

**What a strong answer covers**

- Starting with the **why** — the business domain, the README, and a conversation with someone who knows it — before opening files. Code read without context is just noise.
- Following one concrete path end to end (a real request, job, or user flow) rather than reading files alphabetically.
- Using the **test suite** as the source of truth for intended behavior, especially where documentation is thin.
- Making a small, low-risk change first — a bug fix, a minor improvement — to learn the conventions and the deploy and review process before attempting anything large.
- Asking questions rather than guessing silently for hours, while still doing a reasonable amount of self-directed digging first.
- Treating your own "obvious" questions as useful signal. Something confusing to a newcomer is often a genuine gap in the team's shared knowledge, worth documenting once you've worked it out.

### 403. How do you make architectural decisions, especially ones that are hard to reverse later?

**Framework**

A process answer. Referencing a concrete framework — reversible versus irreversible decisions — shows structured thinking even without a specific story.

**What a strong answer covers**

- Explicitly separating **reversible** ("two-way door") decisions from **hard-to-reverse** ("one-way door") ones, and spending proportionally more time and review on the second kind rather than treating every decision the same.
- Getting input from the people who'll actually live with the consequences, instead of deciding alone and announcing it.
- Documenting the **reasoning**, not just the conclusion — an architecture decision record capturing the options, the trade-offs, and why one won. So future engineers (including future you) understand why, not just what.
- Weighing non-technical constraints alongside technical elegance: the team's current skills, operational burden, cost, and how easy it'll be to hire for and operate.
- Building in a revisit trigger or an escape hatch where possible, so a hard decision isn't treated as permanently unquestionable.
- Comfort making a call under genuine uncertainty, rather than stalling in search of consensus or a risk-free option that doesn't exist.

### 404. How do you balance speed and code quality under deadline pressure?

**Framework**

A principles answer, grounded with a brief real example of a trade-off you've navigated.

**What a strong answer covers**

- Treating the trade-off as a **conscious, communicated** decision. Stakeholders should know what's being deferred, not discover it later.
- Distinguishing corners that are safe to cut (minor duplication, imperfect naming, deferred polish) from ones that aren't (missing tests around money or data integrity, skipped security review, no rollback plan).
- Using techniques that reduce the risk of moving fast — feature flags, incremental rollout, monitoring around the risky change — rather than just hoping.
- Converting the shortcut into **tracked, visible** debt (a ticket, a follow-up task) instead of letting it quietly rot.
- Protecting a non-negotiable minimum bar: tests pass, code is reviewed, the change is reversible. Under pressure the lever that moves is scope or polish — not the safety net.

### 405. Tell me about a time you were wrong, or changed your mind based on new information.

**Framework**

**STAR** — your original position, the evidence that changed it, and what you did with that update.

**What a strong answer covers**

- Genuine honesty. This should read as a real moment of being wrong, not a humble-brag disguised as a flaw.
- A clearly stated original position and the reasoning behind it, so the "before" is specific rather than vague.
- The specific evidence that changed your mind — data, a colleague's argument, a production result you didn't expect.
- How quickly and gracefully you updated. Did you get defensive first, or move once the evidence was clear?
- What actually changed as a result: a decision reversed, a process adopted, a habit formed. That shows the update affected behavior, not just opinion.
- Overall it should signal a growth mindset — that your ego isn't attached to having been right the first time.

### 406. Tell me about a time you disagreed with a decision but had to commit to it anyway.

**Framework**

**STAR** — the classic "disagree and commit" story: you raised the objection, you lost the argument, and you executed anyway in good faith.

**What a strong answer covers**

- The disagreement was raised clearly, with reasoning, **at the right time** — before the decision was final, not relitigated afterwards.
- You raised it in the right place: with the actual decision-maker or in the actual forum, not just venting to peers.
- Once the decision was made, you genuinely accepted it — no silent compliance, no quiet resistance.
- Your execution afterwards was full effort, not a half-hearted attempt engineered to prove your objection right.
- Your reflection afterwards was evidence-based — "here's what actually happened, here's what I'd flag differently next time" — rather than holding a grudge or an "I told you so".
- Overall it should signal maturity: strong technical opinions held firmly but loosely, in service of the team moving forward together.

---

## Prep Checklist

**Ruby**
- [ ] Can explain blocks vs Procs vs lambdas, method lookup/ancestors, and duck typing without notes
- [ ] Can write a `define_method`/`method_missing` example and explain when metaprogramming is worth its cost
- [ ] Can name 3 Ruby 3.x features (pattern matching, `Data.define`, endless methods) and explain Ractors vs Threads vs Fibers and the GVL

**Ruby on Rails**
- [ ] Can walk through the ActiveRecord callback execution order, including the `after_save` vs `after_commit` trap
- [ ] Can explain `includes`/`preload`/`eager_load`/`joins` and diagnose an N+1 query from a log line
- [ ] Can explain when to reach for a service object, form object, query object, or concern — and when NOT to
- [ ] Can walk through a zero-downtime production schema change (expand/contract) end to end
- [ ] Can explain Puma workers vs threads and the DB-pool-sizing relationship, and why Zeitwerk bugs sometimes only surface in production
- [ ] Can narrate at least two of the "p95 spike," "memory climbing," or "Sidekiq backlog" production scenarios out loud

**PostgreSQL / SQL**
- [ ] Can write a JOIN, a window function, and a CTE from memory
- [ ] Can explain MVCC, deadlocks, and `SELECT ... FOR UPDATE`
- [ ] Can explain composite/partial/covering indexes and confirm index usage with `EXPLAIN ANALYZE`
- [ ] Can explain PITR, RPO, and RTO with a real number for each

**Redis**
- [ ] Can name Redis's core data structures with a real use case for each
- [ ] Can explain cache stampede and at least one mitigation
- [ ] Can explain a `SET ... NX PX`-style distributed lock and its known risk

**JavaScript**
- [ ] Can explain closures, the event loop, and `this` across function styles without notes
- [ ] Can write a `fetch` call against a Rails JSON endpoint, including a CSRF header

**APIs**
- [ ] Can explain idempotency, and which HTTP methods are supposed to be idempotent
- [ ] Can walk through the OAuth authorization code flow end to end
- [ ] Can design a paginated, filterable, sortable endpoint safely (column whitelisting)

**System Design**
- [ ] Can whiteboard one "design a ___" prompt in 15 minutes, narrating requirements → data model → API → scaling
- [ ] Can trace a service failure end to end — timeout → retry with backoff+jitter → circuit breaker → fallback
- [ ] Can design the same feature three ways — sync, background job, event-driven — and argue which fits when

**Testing / RSpec**
- [ ] Can explain `let` vs `let!` vs an instance variable in `before`, and why `let`'s laziness can hide a bug
- [ ] Can explain `build` vs `create` vs `build_stubbed` and when each is the right call
- [ ] Can explain stub vs mock vs spy, and why over-mocking your own code makes tests brittle

**Docker**
- [ ] Can write a real Rails Dockerfile and `docker-compose.yml` from memory, including a multi-stage build
- [ ] Can explain the `depends_on` "started vs ready" gap and why you still need a healthcheck

**AWS / Cloud**
- [ ] Can explain Multi-AZ for compute vs for a managed database, with an honest RTO
- [ ] Can explain IAM roles vs users and least privilege, and VPC public vs private subnets
- [ ] Can explain ALB vs NLB and when you'd reach for each

**DevOps / CI-CD**
- [ ] Can describe your CI/CD pipeline end to end and what gates a merge/deploy
- [ ] Can explain blue-green vs canary vs rolling deploys and a feature-flag-based rollback
- [ ] Can write a 5-Whys RCA for a real incident and explain severity vs priority with your own example
- [ ] Can explain Prometheus's pull model, PromQL basics, and SLI/SLO/error budget without notes

**Security**
- [ ] Can explain five security vulnerabilities and their fix without notes (SQLi, XSS, CSRF, IDOR, SSRF)
- [ ] Can explain the TLS handshake in real detail, and why you hash rather than encrypt passwords
- [ ] Can explain cookie security flags, session fixation, and where you'd store an auth token and why

**Git**
- [ ] Can explain rebase vs merge and when rebase is unsafe
- [ ] Can use `git bisect` and `git reflog` from memory
- [ ] Can explain `git reset` (soft/mixed/hard) vs `git revert` and when each is safe

**Behavioral**
- [ ] Have 3 real stories mapped to STAR, including a production issue and a disagreement you resolved
- [ ] Have a clear, honest answer for "explain your current project" ready without sounding rehearsed
- [ ] Can talk through your architectural-decision-making process with one real example

