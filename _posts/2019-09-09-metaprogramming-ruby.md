---
layout: post
title: "[Book] Metaprogramming in Ruby 2"
date: 2019-09-09 12:00:00 +0900
author: 이현민 lhm1442@gmail.com
tags:
  - Ruby
  - Metaprogramming
summary: "
Paolo Perrotta의 [Metaprogramming Ruby 2](http://shop.oreilly.com/product/9781941222126.do) 를 읽으면서 메모한 내용입니다. 퀴즈 솔루션은 https://github.com/qpzm/metaprogramming_ruby 에 있습니다.
"
---

# [Book] Metaprogramming in Ruby 2

## Ch2. The Object Model
### instance variable은 할당할 때 생긴다
> Instance variables just spring into existence when you assign them a value, so you can have objects of the same class that carry different instance variables.

### instance methods
An object’s methods live in the object’s class.

### class vs module
> a class is a module with three additional instance methods (new, allocate, and superclass)

### A class is just an object
![A class is just an object](https://user-images.githubusercontent.com/18223805/62420415-c9fe9880-b6cc-11e9-9d17-8f5fdea70df1.png)

### Wrap-up
> What’s a class? It’s an object (an instance of Class), plus a list of instance methods and a link to a superclass.

### Question
모든 클래스의 클래스가 Class면 `Class`라는 클래스는 재귀적으로 정의되나?
```ruby
class Class
end
```

### Ruby class에 대한 보충 설명
`intialize` is not a constructor!
> When Name.**new** is called to create a new object, the **new** class method in **Class** is run by default, which in turn invokes **allocate** to allocate memory for the object, before finally calling the new object’s **initialize** method.

### What happens when you call a method?
> When you call a method, Ruby does two things:
> 1. It finds the method. This is a process called *method lookup*.
> 2. It executes the method. To do that, Ruby needs something called self.

### Method lookup
- receiver
- ancestors chain
```ruby
MySubclass.ancestors # => [MySubclass, MyClass, Object, Kernel, BasicObject]
```

#### Modules and lookup
include는 ancestors chain에서 해당 클래스 위에, prepend는 밑에 붙임. 따라서 prepend는 현재 클래스보다 더 우선. 아래 코드의 ancestors chain을 그려보자.
```ruby
class C1; include M1; end
class D1 < C1; end
class C2; prepend M2; end
class D2 < C2; end

class Object < BasicObject
  include Kernel
end
```

![Method lookup chain](https://user-images.githubusercontent.com/18223805/62420541-b94f2200-b6ce-11e9-8b77-c8a965825ae3.png){:.center}

#### Multiple inclusion
> if that module is already in the chain, Ruby silently ignores the second inclusion.

M2가 M1을 include하지만 이미 M3에서 `prepend M1`을 선언했기 때문에 무시된다.
```ruby
module M1; end
module M2 include M1
end
module M3 prepend M1 include M2
end
M3.ancestors # => [M1, M3, M2]
```

### The Kernel
```ruby
Kernel.private_instance_methods.grep(/^pr/) # => [:printf, :print, :proc]
```

> Every line of Ruby is always executed inside an object, so you can call the instance methods in Kernel from anywhere. This gives you the illusion that print is a language keyword, when it’s actually a method.

### What private really means
> Private methods are governed by a single simple rule: you cannot call a private method with an explicit receiver.

### Methods in example code
#### \w
> 밑줄 문자를 포함한 영숫자 문자에 대응됩니다. [A-Za-z0-9_] 와 동일합니다.

#### Enumerable#grep
```ruby
[].methods.grep /^re/ # => [:reverse_each, :reverse, ..., :replace, ...]
```

#### Array#replace
```ruby
a = [ “a”, “b”, “c”, “d”, “e” ]
a.replace([ “x”, “y”, “z” ])   #=> ["x", "y", "z"]
a                              #=> ["x", "y", "z"]
```


## Ch3. Methods
```ruby
data_source.methods.grep(/^get_(.*)_info$/) { Computer.define_component $1 }
```

`method_missing` and `block_given?`
```ruby
class Lawyer   def method_missing(method, *args)      puts "You called: #{method}(#{args.join(', ')})"      puts "(You also passed it a block)" if block_given? end  end
```

`String#capitalize`  : `String#upcase` 와 구별
> Returns a copy of *str* with the first character converted to uppercase and the remainder to lowercase.

## Ch4. Blocks
`Kernel#block_given?` 으로 메소드에 블록이 주어졌는 지 알 수 있다.

### Quiz using을 풀 때 유용한 정보
#### Kernel module
> The Kernel module is included by class  [Object](file:///Users/hm/Library/Application%20Support/Dash/Versioned%20DocSets/Ruby%202%20-%20DHDocsetDownloader/2-5-3/Ruby.docset/Contents/Resources/Documents/Object.html) , so its methods are available in every Ruby object.
 [Module: Kernel (Ruby 2.5.3)](http://ruby-doc.org/core-2.5.3/Kernel.html)

#### The main object
> When a new program starts, Ruby automatically creates the main object which is an instance of the Object class. main is the top-level context of any program. This is probably a nod to the C language (as the main() function is the entry point of any C program and that Matz loves C)
```ruby
irb> self
 => main
irb> self.class
 => Object
```
https://medium.com/rubycademy/ruby-object-model-part-1-4d06fa486bec

#### Ruby exception handling
```ruby
begin
  # something which might raise an exception
rescue SomeExceptionClass => some_variable
  # code that deals with some exception
rescue SomeOtherException => some_other_variable
  # code that deals with some other exception
else
  # code that runs only if *no* exception was raised
ensure
  # ensure that this code always runs, no matter what
  # does not change the final value of the block
end
```

### Closure
When you create the block, you capture the local bindings, such as x.
```ruby
def my_method
x = "Goodbye" yield ("cruel")
end
x = "Hello"
my_method {|y| "#{x},#{y} world"} # => "Hello, cruel world"
```

block 안에서 만든 변수는 블록이 끝나면 사라짐. 이 두 가지 특징으로 인해 블록은 클로져다. For the rest of us, this means a block captures the local bindings and carries them along with it.

### & operator turns a proc to a block
```ruby
def my_method(greeting)
  "#{greeting}, #{yield}!"
end
my_proc = proc {"Bill"} my_method("Hello", &my_proc)
```

### Scope
`Kernel#local_variables` 로 지역 변수를 확인할 수 있다.

ruby는 scope gate를 지날 때 마다 새 스코프로 교체된다.

#### Scope Gates
There are exactly three places where a program leaves the previous scope behind and opens a new one:
* Class definitions
* Module definitions
* Methods
클래스나 모듈 스코프에 있는 코드는 바로 실행되는데 메소드 스코프에 있는 코드는 호출이 될 때야 비로서 실행된다.

Top level main object의 instance variable을 전역변수처럼 사용하는 예. `@var`은 main이 self인 스코프에서 접근 가능하다.
```ruby
@var = "The top-level @var"
def my_method
  @var
end
```

#### Passing scope gates
> Ruby coders refer to it simply as “flattening the scope,” meaning that the two scopes share variables as if the scopes were squeezed together.

```ruby
my_var = "Success"
MyClass = Class.new do
  "#{my_var} in the class definition"
  define_method :my_method do
    "#{my_var} in the method"
  end
end
```

#### Sharing the scope without global var
```ruby
def define_methods
  shared = 0
  Kernel.send :define_method, :counter do
    shared
  end
  Kernel.send :define_method, :inc do
    |x| shared += x
  end
end

define_methods
counter # => 0 inc(4) counter # => 4
```

### instance_eval()
```ruby
class MyClass
  def initialize
    @v = 1
  end
end

obj = MyClass.new
obj.instance_eval do   self # => #<MyClass:0x3340dc @v=1> @v # => 1
end
```

> The block is evaluated with the receiver as self, so it can access the receiver’s private methods and instance variables, such as @v. Even if instance_eval changes self, the block that you pass to instance_eval can still see the bindings from the place where it’s defined, like any other block:
```ruby
v=2 obj.instance_eval { @v = v }
obj.instance_eval { @v } # => 2
```

자세한 내용은 `BasicObject#instace_eval` 문서를 참조하자.

### `instance_exec` 은 parameter를 넘겨줄 수 있음.
`instance_eval` 로 해결할 수 없는 경우
```ruby
class C
  def initialize
    @x = 1
  end
end

class D
  def twisted_method
    @y = 2
    C.new.instance_eval { "@x: #{@x}, @y: #{@y}" }
  end
end
D.new.twisted_method # => "@x: 1, @y: "
```
`instance_eval` 은 self를 바꾸기 때문에 block이라 하더라도 이전 scope의 instance variable에는 접근할 수 없다. 이 때 `instance_exec`을 사용하면 parameter를 넘겨주는 방법으로 해결할 수 있다.
```ruby
class D
  def twisted_method
    @y = 2
    C.new.instance_exec(@y) {|y| "@x: #{@x}, @y: #{y}" }
  end
end
D.new.twisted_method # => "@x: 1, @y: 2"
```

### Proc vs Lambda의 두 가지 차이
#### arity
lambda는 arity가 안 맞으면 에러. proc은 인자가 모자르면 nil을 넣고 남으면 무시함.

#### return
lambda에서 return 하면 method return과 동일하게 그 값을 반환.
Proc은 call된 스코프에서 return.

### Method objects
> *Methods*: Bound to an object, they are evaluated in that object’s scope. They can also be unbound from their scope and rebound to another object or class.

Method는 callable object다. 아래와 같이 메소드를 변수에 저장하고 call할 수 있다.
```ruby
object = MyClass.new(1) m = object.method :my_method m.call # => 1
```

#### Method -> Proc, Block -> Method
> you can convert a Method to a Proc by calling `Method#to_proc`, and you can convert a block to a method with `define_method`.

#### Unbound Methods
```ruby
module MyModule
  def my_method
    42
  end
end

unbound = MyModule.instance_method(:my_method)
unbound.class # => UnboundMethod
```

[UnboundMethod#bind](http://ruby-doc.org/core-2.5.3/UnboundMethod.html#￼method-i-bind) 의 예시를 보자.
> Bind *umeth* to *obj*. If Klass was the class from which *umeth* was obtained, obj.kind_of?(Klass) must be true.
```ruby
class A
  def test
    puts “In test, class = #{self.class}"
  end
end
class B < A
end
class C < B
end

um = B.instance_method(:test)
bm = um.bind(C.new)
bm.call # => In test, class = C
bm = um.bind(B.new)
bm.call # => In test, class = B
bm = um.bind(A.new)
bm.call # => prog.rb:16:in `bind’: bind argument must be an instance of B (TypeError) from prog.rb:16
```


## Ch5. Class Definitions
### The truth About Class Methods
> `Klass.a_class_method` calls a method on an object (that also happens to be a class) referenced by a constant.

> if you compare the definition of a Single- ton Method and the definition of a class method, you’ll see that they’re the same:
```ruby
def obj.a_singleton_method;  end
def MyClass.another_class_method; end
```

### Class Macros
```ruby
class Book
  def self.deprecate(old_method, new_method)
    define_method(old_method) do |*args, &block|
      warn "Warning: #{old_method}() is deprecated. Use #{new_method}()."
      send(new_method, *args, &block)
    end
  end
  deprecate :GetTitle, :title
end
```


### Singleton Classes
Where does a singleton class belong?
> an object can have its own special, hidden class. That’s called the *singleton class*of the object. (You can also hear it called the *metaclass*or the *eigenclass*. However, “singleton class” is the official name.)

```ruby
obj = Object.new
singleton_class = class << obj
  self
end
singleton_class.class # => Class

# Easier way
"abc".singleton_class # => #<Class:#<String:0x331df0>>
```

> singleton classes have only a single instance (that’s where their name comes from), and they can’t be inherited. More important, *a singleton class is where an object’s Singleton Methods live*:

`instance_methods` 임에 주의!
```ruby
def obj.my_singleton_method; end
singleton_class.instance_methods.grep(/my_/) # => [:my_singleton_method]
```

### Method Lookup Revisited
아래 코드의 method lookup 과정을 다시 한 번 그려보자.
```ruby
class C
  def self.a_class_method; end
  def a_method; end
end
class D < C
end
obj = D.new
def obj.a_singleton_method; end
```
![method_lookup](https://user-images.githubusercontent.com/18223805/62420737-58294d80-b6d2-11e9-859f-f799d9d4d13c.png)

> The superclass of the singleton class of an object is the object’s class. The superclass of the singleton class of a class is the singleton class of the class’s superclass.
한국어로 정리하면 아래와 같다.
- 오브젝트의 싱글톤 클래스의 수퍼클래스는 오브젝트의 클래스
- 클래스의 싱글톤 클래스의 수퍼 클래스는 수퍼클래스의 싱글톤 클래스

> When you call a method, Ruby goes “right” in the receiver’s real class and then “up” the ancestors chain.
오른쪽의 singleton class를 먼저 보고 그 다음 위의 수퍼 클래스로 올라간다.

irb에서 출력해보면 singleton class 앞에는 `#` 가 붙는다.
```ruby
irb(main):001:0> "abc".class
String < Object
irb(main):002:0> "abc".singleton_class
#<Class:#<String:0x00007fc6e9130918>> < String
```

### instance_eval의 진짜 의미
> you learned that instance_eval changes self, and class_eval changes both self and the current class. However, instance_eval also changes the current class; it changes it to the *singleton class*of the receiver.

### Add a class method through including a module
```ruby
module MyModule
  def my_method; 'hello'; end
end

class MyClass
  class << self
    include MyModule
  end
end

puts MyClass.my_method
```

### Object#extend is a shortcut!
`Object#extend` 는 싱글톤 클래스에 모듈을 include한다.




## Ch6. Code that writes code
### Kernel#eval
문자열로 주어진 코드를 실행시킨다.

### Binding objects
> A Binding is a whole scope packaged as an object. The idea is that you can create a Binding to capture the local scope and carry it around.

책의 예시를 보자.
```ruby
class MyClass
  def my_method
    @x = 1
    binding
  end
end
b = MyClass.new.my_method # binding을 얻음
eval "@x", b # => 1
```

`Kernel#binding` 문서의 예시를 보자.
```ruby
def get_binding(param)
  binding
end
b = get_binding("hello")
eval("param", b)   #=> "hello"
```

#### irb
> irb is just a simple program that parses the standard input or a file and passes each line to eval.

#### Object#taint
> Objects that are marked as tainted will be restricted from various built-in methods. This is to prevent insecure data, such as command-line arguments or strings read from `Kernel#gets`

### Hooks
상속, include, prepend, extend될 때 특정 코드를 실행시킨다.
```ruby
class String
  def self.inherited(subclass)
    puts "#{self} was inherited by #{subclass} " end
  end
class MyString < String; end
# String was inherited by MyString
```
