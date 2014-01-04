---
project:     oo
tagline:     object system with virtual properties and method overriding hooks
category:    Language
---

v1.0 | Lua 5.1, Lua 5.2, LuaJIT 2

## `local oo = require'oo'`

## Features

 * single, dynamic inheritance by default:
   * `oo.class([superclass]) -> class`
   * `class() -> instance`
   * `instance.super -> class`
   * `class.super -> superclass`
 * static, multiple inheritance by request:
   * `self:inherit(other[,override])` - statically inherit properties of `other`, optionally overriding existing properties.
   * `self:detatch()` - detach from the parent class, in other words statically inherit `self.super`.
 * virtual properties with getter and setter:
   * reading `self.foo` calls `self:get_foo()` to get the value.
   * assignment to `self.foo` calls `self:set_foo(value)`.
   * missing the setter, the property is read-only and assignment fails.
 * stored properties (no getter):
   * assignment to `self.foo` calls `self:set_foo(value)` and sets `self.state.foo`.
   * reading `self.foo` reads back `self.state.foo`.
 * before/after method hooks:
   * `self:before_m()` installs a before-hook for `self:m()`.
   * `self:after_m()` installs an after-hook for `self:m()`.
 * introspection:
   * `self:allpairs() -> iterator() -> name, value, source` - iterate all properties, including inherited _and overriden_ ones.
   * `self:properties()` -> get a table of all current properties and values, including inherited ones.
   * `self:inspect()` - inspect the class/instance structure and contents in detail.

## In detail

**Classes are created** with `oo.class([super])`, where `super` is usually another class, but can also be an instance, which is useful for creating polymorphic "views" on existing instances.

~~~{.lua}
local cls = oo.class()
cls.classname = 'cls'
~~~

**Instances are created** with `myclass:create(...)` or simply `myclass()`, which in turn calls `myclass:init(...)` which is the object constructor. While `myclass` is normally a class, it can also be an instance, which effectively enables prototype-based inheritance.

~~~{.lua}
local obj = cls()
~~~

**The superclass** of a class or the class of an instance is accessible as `self.super`.

~~~{.lua}
assert(obj.super == cls)
assert(cls.super == oo.object)
~~~

**Inheritance is dynamic**: properties are looked up at runtime in `self.super` and changing the superclass
reflects on all subclasses and instances. This can be slow, but it saves space.

~~~{.lua}
cls.the_answer = 42
assert(obj.the_answer == 42)
~~~

You can detach the class/instance from its parent class by calling `self:detach()`. This copies all inherited fields
to the class/instance and removes `self.super`.

~~~{.lua}
cls:detach()
obj:detach()
assert(obj.super == nil)
assert(cls.super == nil)
assert(cls.the_answer == 42)
assert(obj.the_answer == 42)
~~~

**Static inheritance** can be achieved by calling `self:inherit(other[,override])` which copies over the properties of
another class or instance, effectively *monkey-patching* `self`, optionally overriding properties with the same name.
The fields `self.classname` and `self.super` are always preserved though, even with the `override` flag.

~~~{.lua}
local other_cls = oo.class()
other_cls.the_answer = 13

obj:inherit(other_cls)
assert(obj.the_answer == 13) --obj continues to dynamically inherit cls.the_answer
                             --but statically inherited other_cls.the_answer

obj.the_answer = nil
assert(obj.the_answer == 42) --reverted to class default

cls:inherit(other_cls)
assert(cls.the_answer == 42) --no override

cls:inherit(other_cls, true)
assert(cls.the_answer == 13) --override
~~~

In fact, `self:detach()` is written as `self:inherit(self.super)` with the minor detail of setting
`self.classname = self.classname` and removing `self.super`.

To further customize how the values are copied over for static inheritance, override `self:properties()`.

**Virtual properties** are created by defining a getter and a setter. Once you have defined `self:get_foo()`
and `self:set_foo(value)` you can read and write to `self.foo` and the getter and setter will be called to fulfill
the indexing. The setter is optional: without it, the property is read-only and assigning it fails with an error.

~~~{.lua}
function cls:get_answer_to_life() return deep_thought:get_answer() end
function cls:set_answer_to_life(v) deep_thought:set_answer(v) end
obj = cls()
obj.answer_to_life = 42
assert(obj.answer_to_life == 42) --assuming deep_thought can store a number
~~~

**Stored properties** are virtual properties with a setter but no getter. The values of those properties are stored
in the table `self.state` upon assignment of the property and read back upon indexing the property.
If the setter breaks, the value is not stored.

~~~{.lua}
function cls:set_answer_to_life(v) deep_thought:set_answer(v) end
obj = cls()
obj.answer_to_life = 42
assert(obj.answer_to_life == 42) --we return the stored the number
assert(obj.state.answer_to_life == 42) --which we stored here
~~~

Virtual and inherited properties are all read by calling `self:getproperty(name)`. Virtual and real properties
are written to with `self:setproperty(name, value)`. You can override these methods for *finer control* over the
behavior of virtual and inherited properties.

Virtual properties can be *generated in bulk* given a getter and a setter and a list of names by calling
`self:gen_properties(names, getter, setter)`. The setter and getter must be methods of form:

  * `self:getter(k) -> v`
  * `self:setter(k, v)`

**Before/after hooks** are sugar for overriding methods. Overriding a method can be done by simply redefining
it and calling `class.super.<method>(self,...)` inside the new implementation. Most of the time this call
is done either on the first or on the last line of the new function. With before and after hooks you can
have that done automatically.

By defining `self:before_<method>(...)` a new implementation for `self.<method>` is created which calls the
before hook (which receives all method's arguments) and then calls the existing (inherited) implementation
with whatever the hook returns as arguments.

By defining `self:after_<method>(...)` a new implementation for `self.<method>` is created which calls the
existing (inherited) implementation, after which it calls the hook with whatever the method returns as arguments,
and returns whatever the hook returns.

~~~{.lua}
function cls:before_init(foo, bar)
  foo = foo or default_foo
  bar = bar or default_bar
  return foo, bar
end

function cls:after_init()
  --allocate resources
end

function cls:before_destroy()
  --destroy resources
end
~~~
