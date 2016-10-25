---
layout: post
title:  "Ruby Metaprogramming: Classes, Methods and Singleton Classes"
author: "Máté Solymosi"
date:   2016-10-25 12:00:00
categories: technology
---

At Sittercity we primarily use Go and Ruby to develop our applications, including our new product, [Chime](http://hellochime.com). In this very technical article (don’t say I didn’t warn you!) I talk about an advanced Ruby topic, _metaprogramming_. If you already have a solid knowledge of Ruby but want to better understand how the language works and learn a few metaprogramming tricks, then this article is for you.

When they hear the word “metaprogramming”, many Ruby developers think that it is some mysterious, hard to understand and therefore dangerous thing that is irrelevant to their daily work, while not realizing that they may already be using some of its concepts in their code. Part of the confusion is that in Ruby there is no clear boundary between metaprogramming and “doing normal things that normal people do,” given how dynamic the language is. But there is really no reason to be “afraid” of metaprogramming in general: many of its concepts are straightforward, and even those that are harder to wrap your head around have a well thought out system behind them.

<!-- More -->

That said, I also think it’s important to highlight that…

### You might hurt yourself doing this

Not being afraid of metaprogramming does not mean that it should be used in every possible scenario, because it tends to come with the cost of decreased code readability. This cost always has to be offset with a greater benefit, such as a significant decrease in the amount of code needed to implement certain features, increased code reliability by making certain programming mistakes harder to make, a large reduction in code duplication or the increased readability of large amounts of other code.

The need for significant benefits explains why metaprogramming is often used in frameworks and base classes: in order to justify the hard to understand “Ruby magic” in your code, it needs to be used in lots of places so that they all enjoy the benefits. Rails, in particular, is full of it, which makes the source code of the framework challenging to read, but that is also the main reason why Rails _application code_ tends to be very descriptive and a joy to work with.

So, as a general principle, feel free to use Ruby metaprogramming when developing the supporting structure of your app, but be very careful if you find yourself contorting your application code itself.

### What are classes in Ruby?

In Ruby, like everything else, classes are dynamic. In particular, their contents can be changed anytime from anywhere in your code. This is a very well-known property of the language, as you can’t go through an introductory Ruby tutorial without learning that you can “monkey-patch” existing classes by reopening them and then adding or overwriting methods. You can even change core Ruby classes such as `String` or `Integer` if you want &ndash; a feature heavily used by Rails and its ActiveSupport library, which has tons of these so-called “core extensions.” In fact, as a Rails developer I am often unsure whether a method I am calling on a string or a number comes from Rails or from core Ruby. And it’s for a good reason: core extensions of popular frameworks often end up being added to Ruby itself.

A less well-known feature of Ruby is that classes… are actually just plain objects. They are instances of the class `Class`. When I first learned about this, I was in a “mind = blown” state for a while, because after having worked with PHP, Visual Basic and C#, the fact that a class is not a language construct but a piece of data in your running application is not particularly easy to come to terms with.

What does this bring us? Well, for starters, we can create anonymous classes, and even store classes in variables. In the end, the `class` keyword is just syntactic sugar (well, sort of) for this:

{% highlight ruby %}
my_klass = Class.new do
  def foo
    "bar"
  end
end
# => #<Class:0x27aa880>

my_instance = my_klass.new
# => #<#<Class:0x27aa880>:0x2ed0478>

my_instance.foo
# => "bar"
{% endhighlight %}

If we want inheritance, we can pass in the parent class as the first parameter of `Class.new`:

{% highlight ruby %}
my_subclass = Class.new(my_klass)
# => #<Class:0x82fa9d0>

my_subclass.new.foo
# => "bar"
{% endhighlight %}

Can we turn anonymous classes into named ones? Well indeed we can! All we need to do is assign it to a constant. Under the hood, every named class is just a `Class` instance assigned to a constant somewhere in the constant tree:

{% highlight ruby %}
SomeClass = my_klass
# => SomeClass

SomeClass.new.foo
# => "bar"
{% endhighlight %}

Now you may be thinking: if every class is an instance of `Class`, then `Class` must be an instance of _itself_, right? Yes, that is correct; we have some nice recursion on our hands here:

{% highlight ruby %}
SomeClass.class.class.class.class.class
# => Class
{% endhighlight %}

What about class definitions? After all, `Class.new` is just a regular method that takes a block, so in theory we could put whatever code we want in there, not just method definitions or calls to `attr_accessor` and the like. It turns out that the body of “regular” class definitions using the `class` keyword is treated as plain Ruby code as well, similarly to the body of a method:

{% highlight ruby %}
class MyClass
  # anything we put here is executed when the class is created
  puts "Hello Ruby!"

  # all this is executed in the context of the class, so `self` is our class
  puts self
end
# Hello Ruby!    # from the first `puts`
# MyClass        # from `puts self`
# => nil
{% endhighlight %}

This also means that in certain cases it matters how the code in a class definition block is ordered. For example, unlike in many other programming languages, class methods are only available from the point in code where they are defined:

{% highlight ruby %}
class Abc
  puts self.respond_to?(:foo)

  def self.foo
    "bar"
  end

  puts self.respond_to?(:foo)
end
# false
# true
# => nil
{% endhighlight %}

Let’s recap the main points so far. Classes in Ruby are objects, just like everything else. They are instances of the class `Class`. Anonymous classes can be created by calling `Class.new`, which can later be named by assigning them to a constant. The bodies of class definitions are just plain Ruby code where we can write whatever we want, and it is executed when the class is created.

### Looking into classes

Ruby has a rich set of tools for class introspection (or _reflection_, as it is called in certain languages), so once a class is created, there are many ways to change it or to see what’s inside. Continuing the previous examples, here are a few methods that I’ve used multiple times over the past year:

{% highlight ruby %}
SomeClass.name
# => "SomeClass"

my_subclass.superclass
# => SomeClass     # remember, `SomeClass` is `my_klass` with a name

my_subclass.ancestors     # this class, superclass, its parent, etc.
# => [#<Class:0x82fa9d0>, SomeClass, Object, Kernel, BasicObject]

class SomeClass
  define_method :abc do |x, y|
    x + y
  end
end
# => :abc

SomeClass.new.abc(3, 4)
# => 7
{% endhighlight %}

Hold on for a minute! Why would someone ever use `define_method` to add a new instance method to a class, instead of the standard `def` keyword? The answer is simple: `define_method` takes a symbol (or a string) as the method name, which lets you programmatically set the name of the method, or even define multiple methods in bulk. This is handy when you need to create a bunch of getter and setter methods all at once:

{% highlight ruby %}
class User
  [:id, :first_name, :last_name, :email].each do |field|
    define_method field do
      instance_variable_get("@#{field}")
    end

    define_method "#{field}=" do |new_value|
      instance_variable_set("@#{field}", new_value)
    end
  end
end
# => [:id, :first_name, :last_name, :email]

dave = User.new
# => #<User:0x2d05f58>

dave.first_name = "Dave"
# => "Dave"

dave.first_name
# => "Dave"
{% endhighlight %}

In fact, this is so useful that Ruby has the same thing built in by default: `attr_accessor` works very similarly to our code above.

{% highlight ruby %}
class User
  attr_accessor :id, :first_name, :last_name, :email
end
{% endhighlight %}

When an instance method is added to a class, it gets saved into the class object. We can look up the instance methods supported by a class by calling `instance_methods` on it. By default, this will list all instance methods either defined in that class or inherited from an ancestor, but we can exclude inherited methods by passing in `false` as the first parameter:

{% highlight ruby %}
SomeClass.instance_methods
# => [:foo, :abc, :nil?, :===, :=~, :!~, :eql?, :hash, :<=>, :class, :singleton_class, :clone, :dup, :taint, :tainted?, :untaint, ...

SomeClass.instance_methods(false)
# => [:foo, :abc]
{% endhighlight %}

Based on what I’ve explained so far, the rules of how Ruby finds the right instance method to dispatch when a method is called on an object seem pretty straightforward:

1.  The object knows which class it was instantiated from. Go get that class object.
2.  Look at the own instance methods of the class. If the method is found there, we’re done.
3.  Otherwise, repeat this for every class (or module) in `our_class.ancestors`.
4.  If the method is not found in any of them, raise a `NoMethodError`.

This list, however, is incomplete. There are two important steps missing: one related to a special method called `method_missing`, which is a story for another time, and another related to the object’s _singleton class_, which we discuss next.

### Singleton classes: hidden in plain sight

Here is a controversial thought: in Ruby, there is no language-level distinction between class methods and instance methods. Both are just plain instance methods, just stored in different places. I hope that by the end of the article I will be able to convince you that this statement is true. But for now, let’s take a step back and talk first about singleton methods.

When it comes to learning Ruby, most developers first hear the word “singleton” in the context of _singleton methods_. This feature of the language lets you define methods on individual objects:

{% highlight ruby %}
str1 = "foo"
# => "foo"

str2 = "bar"
# => "bar"

def str1.some_method
  "I have just been called"
end
# => :some_method

str1.some_method
# => "I have just been called"

str2.some_method
# => NoMethodError: undefined method `some_method` for "bar":String
{% endhighlight %}

As you can see, using the `def object.method_name` syntax we were able to add a new method to just `str1`, but not `str2` or any other string created in the past or the future. But where is this new method stored? It is definitely not an instance method of the `String` class, because then every string would get it automatically. We can also prove this:

{% highlight ruby %}
String.instance_methods.include?(:some_method)
# => false
{% endhighlight %}

The answer is that it was added to the _singleton class_ of `str1`. In Ruby, every single object has its own hidden singleton class, inserted between the object and the regular class it was created from. The singleton class is the very first thing that is searched whenever a method is invoked on the object, so it can be used to override an existing method just for that specific instance. Continuing our example, the best way to illustrate how the singleton class works is to imagine it as a custom subclass of `String`, the only instance of which is `str1`:

{% highlight ruby %}
# This is an illustration
class HiddenSingletonClassOfStr1 < String
  def some_method
    "I have just been called"
  end
end
# => nil

HiddenSingletonClassOfStr1.new("foo")
# => "foo"
{% endhighlight %}

Singleton classes work the same way regular classes do. The only difference between them and regular classes is that Ruby is going to great lengths to keep them hidden from you. Calling `object.class` will skip over the singleton class in the inheritance hierarchy and give you the regular class the object was instantiated from; `object.ancestors` will exclude the singleton class from its list (even though the singleton class _is_ searched when dispatching method calls), and so on. But if you want to mess with it, there is an easy way to get to the singleton class:

{% highlight ruby %}
str1.singleton_class
# => #<Class:#<String:0x2ccddf8>>

str1.singleton_class.instance_methods(false)
# => [:some_method]
{% endhighlight %}

Hey, look &ndash; there is our custom instance method we defined for `str1`. Now let’s see how else we can define singleton methods:

{% highlight ruby %}
def str1.method_2
  "the original way"
end
# => :method_2

str1.singleton_class.send(:define_method, :method_3) do
  "calling define_method on the singleton class"
end
# => :method_3

class << str1
  def method_4
    "opening up the singleton class and adding the method"
  end
end
# => :method_4

str1.singleton_class.instance_methods(false)
# => [:some_method, :method_2, :method_3, :method_4]
{% endhighlight %}

If you look at the first and the last approach, you may notice some similarity between them and the two most frequently used ways we create class methods in class definitions. In fact, the code looks exactly the same, with the exception that class method definitions use `self` instead of `str1`. This similarity is not accidental, and it brings us to the question as to where class methods are stored in Ruby. Consider the following example:

{% highlight ruby %}
class Wat
  attr_accessor :foo

  def self.bar
    "bar"
  end

  class << self
    def baz
      "baz"
    end
  end
end
{% endhighlight %}

Can you guess where the `attr_accessor` common class method and our own `bar` and `baz` class method are stored? Well, `attr_accessor` should be easy. We already know that every class is an instance of `Class`, and `self` in a class definition block points to the class being defined. Since `attr_accessor` is present on every class object, it must be an instance method of `Class` (turns out it’s a private one):

{% highlight ruby %}
Class.private_instance_methods.include?(:attr_accessor)
# => true
{% endhighlight %}

Our `bar` and `baz` class methods, however, are not available in every class, so they cannot be stored in the `Class` class. But guess what: since classes are just objects (of `Class`), they also have their own singleton classes, just like any other object. So if `bar` and `baz` are stored in the singleton class of `Wat`, they should only be available on `Wat` but not on other classes. This is consistent with the behavior of class methods. Let’s verify that it is true:

{% highlight ruby %}
Wat.singleton_class.instance_methods(false)
# => [:bar, :baz]
{% endhighlight %}

What we’ve just demonstrated is that in the end, class methods are regular instance methods. They only act as class methods because they are stored in the singleton class of their associated class object. This notion is critical to understanding what’s going on in much of the Ruby metaprogramming code out there, so if you are still a bit confused, take your time to experiment with it before we move on to our…

### Example: transparent lazy loading

Let’s say we have a model class called User for representing users in a typical MVC application. In addition, let’s say that in our controller we use an instance of this class called `current_user`, to store all the attributes of the currently signed in user. When a request comes in, we instantiate our current user object and load all of its attributes from our backend system:

{% highlight ruby %}
class User
  attr_accessor :id, :first_name, :last_name, :profile_pictures, :credit_cards

  def default_profile_picture
    profile_pictures.find(&:default?)
  end

  def default_credit_card
    credit_cards.find(&:default?)
  end
end

class Controller
  attr_reader :current_user

  # ...

  def load_current_user
    user_id = get_user_id_from_session
    @current_user = Database.get_user(user_id)
    @current_user.profile_pictures = SlowExternalApi.fetch_pictures(user_id)
    @current_user.credit_cards = SlowExternalApi.fetch_credit_cards(user_id)
    @current_user
  end
end
{% endhighlight %}

Soon, people start complaining that every page load is very slow. The problem is traced back to the fact that we are fetching the profile pictures and the credit cards in every single request, whether or not we actually need them to fulfill it. So we come up with the following options:

1.  Remove the lines that fetch the pictures and the cards from `load_current_user`, and add them only to those pages where we actually need these attributes. This is a simple solution, but if we need the slow-loading attributes in many places, we end up sprinkling our code base with lots of external API calls. Plus, we would have to move the `default_*` methods outside of `User` as well. It would be much more elegant to be able to call `current_user.default_profile_picture` from our controllers and not have to worry about how the data is retrieved.
2.  Load the attributes once, and cache them. This introduces the problem of cache invalidation, which is, as they say, one of the only [two hard things](http://martinfowler.com/bliki/TwoHardThings.html) in computer science. This is not cool, so let’s move on.
3.  Lazy load the affected attributes so that the external API is only invoked when a profile picture or credit card related method is called on `current_user`. But putting lazy loading code into a _model class_ &ndash; especially code that talks to an external API? That is certainly not a good idea. Also, we want the lazy loading logic to only apply to `current_user`, but not to other instances of `User`.

Fortunately, singleton classes come to the rescue, and enable us to add custom behavior to just the `current_user` instance, without affecting either the model class `User` or any of its other instances. What’s more, this way the lazy loading code can reside in the controller class:

{% highlight ruby %}
class User
  # nothing needs to change here 
end

class Controller
  attr_reader :current_user

  # ...

  def load_current_user
    user_id = get_user_id_from_session
    @current_user = Database.get_user(user_id)

    def @current_user.profile_pictures     # define singleton method
      puts "Fetching profile pictures..."
      super || @profile_pictures = SlowExternalApi.fetch_pictures(self.id)
    end

    def @current_user.credit_cards
      puts "Fetching credit cards..."
      super || @credit_cards = SlowExternalApi.fetch_credit_cards(self.id)
    end

    @current_user
  end
end
{% endhighlight %}

There are a few things that should be noted here. First, since the singleton methods we’ve defined take precedence over the existing methods in `User` (which in this case were auto-created with `attr_accessor`), the lazy loading will also happen when `default_profile_picture` calls `profile_pictures` on the current user. So we only need to install lazy loaders for the actual attribute methods, but not for other methods in `User` that might use them.

Second, since the singleton class is at the end of the inheritance hierarchy and it’s a subclass of `User`, we can call `super` in our singleton method if we want to fall back to the default, non-lazy-loaded implementation in `User`.

The resulting `current_user` object is just as easy to use as before, and the callers do not even need to be aware that the loading logic has changed; everything is neatly contained within this single User instance:

{% highlight ruby %}
c = Controller.new
# => #<Controller:0x2aaf020>

user = c.load_current_user
# => #<User:0x2cd4b90 @id=123>

user.profile_pictures
# Fetching profile pictures...
# => [#<ProfilePicture:0x28e60bd>, #<ProfilePicture:0x2e0722a>]

user.default_profile_picture     # already fetched, no need to fetch again
# => #<ProfilePicture:0x2e0722a>

user.default_credit_card
# Fetching credit cards...
# => #<CreditCard:0x2a40b36>

user
# => #<User:0x2cd4b90 @id=123, @profile_pictures=[#<ProfilePicture:0x28e60bd>, #<ProfilePicture:0x2e0722a>], @credit_cards=[#<CreditCard:0x2a40b36>]>
{% endhighlight %}

I hope this article has been useful in shedding some light on a few lesser known properties and features of Ruby. In any case, I have only scratched the surface here; metaprogramming in Ruby is such a broad topic that entire books have been written about it (such as [this one](https://www.amazon.com/Metaprogramming-Ruby-Program-Like-Facets/dp/1941222129)), and these are just a few of the many different metaprogramming techniques available to Ruby programmers.
