---
layout : post
category : programming
tagline : "Objects vs. Closures"
tags : [javascript, ruby, procs, lambdas, closures]
---
{% include JB/setup %}

### Ancient(-ish) wisdom on closures and objects

*The venerable master Qc Na was walking with his student, Anton. Hoping to prompt the master into a discussion, Anton said “Master, I have heard that objects are a very good thing – is this true?” Qc Na looked pityingly at his student and replied, “Foolish pupil – objects are merely a poor man’s closures.”*

*Chastised, Anton took his leave from his master and returned to his cell, intent on studying closures. He carefully read the entire “Lambda: The Ultimate…” series of papers and its cousins, and implemented a small Scheme interpreter with a closure-based object system. He learned much, and looked forward to informing his master of his progress.*

*On his next walk with Qc Na, Anton attempted to impress his master by saying “Master, I have diligently studied the matter, and now understand that objects are truly a poor man’s closures.” Qc Na responded by hitting Anton with his stick, saying “When will you learn? Closures are a poor man’s object.” At that moment, Anton became enlightened.*

--- Anton van Straaten


### Closures and Objects as State

To get too literal with these Zen sort of tales and [koans](http://en.wikipedia.org/wiki/Koan) is to kind of miss the point.
They are written the way they are because the meaning they attempt to convey doesn't lend itself to a more straightforward discussion.
But one bit of knowledge that's wrapped up in the above koan is that both closures and objects can be used for similar purposes, and for today, focus on the fact that they can both be used to keep track of state.
In terms of their state tracking abilities, it's incredibly difficult to say which is better, because it depends on what you are trying to do.
To assert that one is strictly better than the other is to invite an ancient Zen master to smack you with a stick.

Idiomatic ruby code almost always tracks state using its object system.  Idiomatic Javascript is a little more flexible.  Sometimes you will choose to track state in objects, sometimes closures, and sometimes a combination the two.

For example, the following simple Javascript counter tracks state using a closure:

    {% highlight javascript %}
    counter = (function() {
      var public = {}
        , count  = 0;

      public.reset = function() {
        count = 0;
      };

      public.getCount = function() {
        return count;
      };

      public.inc = function() {
        count += 1;
      };

      return public;
    })();
    {% endhighlight %}

If you haven't seen the above pattern before, it's pretty neat.  If you have, you might want to skip to the next section.

In jargon-y terms, the code uses of a self-executing anonymous function to return an object holding functions that all have access to that anonymous function's closure.  But jargon is only useful if you already know it, so let's clear some of that up.  If you ignore the stuff inside of the function's brackets the code basically looks like this:

    {% highlight javascript %}
    counter = (function() { ... })();
    {% endhighlight %}

So, inside the first set of parentheses, a function is defined. The second set of parenthesis just call that function, much like you call any function, e.g.

    {% highlight javascript %}
    myFun();
    {% endhighlight %}

The function inside the first set of parentheses is called an anonymous function because it never actually has a name, and it's never assigned to a variable.  Being nameless, once we're done executing the code, we'll have no way of referencing that function again in order to call it.  You might think that counter would end up equalling the anonymous function itself, but it doesn't, because we immediately execute the function.  Instead counter ends up referencing whatever that function returns:

    {% highlight javascript %}
    counter = (function() {
      var public = {}

      ...

      return public;
    })();
    {% endhighlight %}

And public itself is just an object, which has had several functions assigned to its keys, such as:

    {% highlight javascript %}
      public.inc = function() {
        count += 1;
      };
    {% endhighlight %}

Note that the function doesn't say "var count" anywhere in it.  Because count hasn't been defined within the function, whenever you reference it, Javascript will look for it in progressively higher scopes. In this case, it finds it almost immediately, in the anonymous function's scope:

    {% highlight javascript %}
    counter = (function() {
      var public = {}
        , count  = 0;
      ...
    {% endhighlight %}

So whenever you call that inc function, it will look up count, find it in the anonymous function's scope, and then add one to it.  Whenever you call getCount, it will return the count that exists in that anonymous function's scope.  Whenever you call reset, it will reset that anonymous function's count to 0.

Immediately after defining counter, you can run the following code, and that setup ends up working remarkably similarly to a more object oriented implementation.

    {% highlight javascript %}
    console.log("The count is: "  + counter.getCount());
    //outputs "The count is: 0"

    counter.inc();
    counter.inc();

    console.log("The count went up to: "  + counter.getCount());
    //outputs "The count went up to: 2"
    {% endhighlight %}


### Closures and Privacy

One of the primary advantages of defining things in this closure oriented fashion is that once the closure has been defined, that's it.  Unless you deliberately expose a way of accessing those variables, none exists.  You might try and alter counter, but it doesn't do you any good.

    {% highlight javascript %}
    counter.count = 3.14159;

    console.log("The count is still: "  + counter.getCount());
    //outputs "The count is still: 2"
    {% endhighlight %}

The variable counter, after all, isn't actually the closure. It's just an object that contains some functions that have access to the closure.  In Javascript it's relatively easy to create true private variables.


### Ruby Objects and Privacy, or the lack thereof

Consider an idiomatic Ruby implementation of that same Javascript code:

    {% highlight ruby %}
    class Counter
      attr_reader :count

      def initialize
        reset!
      end

      def inc
        @count += 1
      end

      def reset!
        @count = 0
      end
    end
    {% endhighlight %}

It works just like you'd expect:

    {% highlight ruby %}
    c = Counter.new

    puts "The count is: #{ c.count }"
    # outputs "The count is: 0

    c.inc
    c.inc

    puts "The count went up to: #{ c.count }"
    # outputs "The count went up to: 2

    begin
      c.count = 3.14159
    rescue NoMethodError => e
      puts "The counter doesn't have that method"
    end
    # outputs "The counter doesn't have that method"
    # because we only defined an attr_reader on @count, not an attr_writer

    puts "The count is still: #{ c.count }"
    # outputs: "The count is still: 0"
    {% endhighlight %}

This seems like what we want.  Ruby looks like it's protecting your objects' internal state from other code.  But Ruby is also an incredibly dynamic language which has things like the instance_variable_set method, which means your objects don't really have any protection at all.

    {% highlight ruby %}
    c.instance_variable_set(:@count, 3.14159)

    puts "The count is now: #{ c.count }"
    # outputs "The count is now: 3.14159"
    {% endhighlight %}

Between instance_variable_set and reopening classes, it's basically impossible to hide your objects' instance variables in Ruby.


### Ruby as Javascript

Javascript is able to hide its internal state because of closures and anonymous functions.  While Ruby's objects have no real privacy, Ruby does have closures and anonymous functions.  It's actually quite possible to do a (nearly) direct translation of the javascript code in Ruby, bypassing Ruby's class / object system, and instead relying on closures.

    {% highlight ruby %}
    counter = (-> do
      pub = {}
      count = 0

      pub[:reset]     = -> { count = 0 }
      pub[:inc]       = -> { count += 1 }
      pub[:get_count] = -> { count }

      pub
    end).call
    {% endhighlight %}


If you haven't seen Ruby's "stabby" lambda syntax before, all those "->" are just syntactic sugar for writing "lambda".  We do have to name our hash "pub" instead of "public", because public is a reserved keyword in Ruby. But aside from that one change, the code is basically the same, just with different syntax.  We follow the same process of creating an anonymous function, calling it, and storing the result in "counter"

    {% highlight ruby %}
    counter = (-> do
      ...
    end).call
    {% endhighlight %}

And from that anonymous function we return a hash that contains a bunch of functions defined for accessing the anonymous function's closure.

It works just like you'd expect:

    {% highlight ruby %}
    puts "The count is: #{ counter[:get_count].call }"
    # outputs "The count is 0"

    counter[:inc].call
    counter[:inc].call

    puts "The count went up to: #{ counter[:get_count].call }"
    # outputs "The count went up to: 2"
    {% endhighlight %}

And you are protected from any instance_variable_set shenanigans.

    {% highlight ruby %}
    # first we'll try setting "count"
    begin
      counter.instance_variable_set(:count, 3.14159)
    rescue NameError => e
      puts "Instance variables have to start with the '@' symbol"
    end
    # outputs "Instance variables have to start with the '@' symbol"

    # Then we'll try setting "@count", which makes even less sense
    counter.instance_variable_set(:@count, 3.14159)

    puts "The count is still: #{ counter[:get_count].call }"
    # outputs "The count is still: 2"

    # We can only access count through the methods we defined
    counter[:reset].call

    puts "The count is now: #{ counter[:get_count].call }"
    # outputs "The count is now: 0"
    {% endhighlight %}

So it appears that we've solved the privacy issue by bypassing Ruby's object system and making use of closures.  In almost any language with both objects and closures, this would be an excellent example of how they can both be used for similar purposes, and how each one has its place...


### But no, wait, this is Ruby

So even though we defined all of those functions in pub as lambdas, lambdas are actually just special cases of the Proc class.

    {% highlight ruby %}
    puts "It is a lambda" if counts[:inc].lambda?
    # outputs "It is a lambda"

    puts "But it is also Proc" if counter[:inc].class == Proc
    # outputs "But it is also Proc"

    puts "And it therefore has a binding!" if counter[:inc].binding
    # outputs "And it therefore has a binding!"
    {% endhighlight %}

If you've never had to deal with bindings before, they can be thought of as the environment contained by a closure.  And because we have direct access to the environment that was supposed to be hidden from us, we also have access to the variables contained within.

    {% highlight ruby %}
    binding_count = eval("count", counter[:inc].binding)
    puts "Accessing the count through the binding: #{ binding_count }"
    # outputs  "Accessing the count through the binding: 0"
    {% endhighlight %}

We can not only read variables within the binding, we can also change them.

    {% highlight ruby %}
    eval("count = 3.14159", counter[:inc].binding)
    puts "The count is now: #{ counter[:get_count].call }"
    # outputs "The count is now: 3.14159"
    {% endhighlight %}

So while closures totally do have their uses in Ruby, privacy is not one.

### code

If you are interested in playing around with the example code in this blog post, you can find it [here](https://github.com/rcjara/blog-code/tree/master/ruby-as-javascript).
