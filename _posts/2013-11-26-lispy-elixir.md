---
layout: post
title: Lispy Elixir
date: 2013-11-26
---

There has been some recent interest in the [Elixir language](https://github.com/elixir-lang/elixir) and with good reason, it's an exciting language! Elixir not only brings the power of the Erlang VM, but also offers a macro system which ought to feel very familiar to Lispers. The Macro system introduces flexibility that I personally find to be easier to work with than Erlang's [parse transform](http://www.erlang-factory.com/upload/presentations/521/yrashk_parse_transformations_sf12.pdf). In this blog post I'll be assuming that you're somewhat familiar with Lisp style macros (there are, of course, many [resources](http://blog.8thlight.com/colin-jones/2012/05/22/quoting-without-confusion.html)). I wanted to take the time to show you the macro system while also highlighting some of its Lisp heritage.

### Homoiconicity ###
Homoiconicity is one of the hallmarks of Lisp, it's the reason why everything is wrapped in parantheses. To be homoiconic means that there is one representation for both data and executable code, that is to say that executable statements are contained within first class data structures. This has a number of benefits, one of which is permitting a Lisp-style macro system.

John McCarthy's original Lisp paper<sup><a href="footnote_1">[1]</a></sup> described a <i>LIS</i>t <i>P</i>rocessing language, containing all code and data within S-Expressions. These S-Expressions were lists that, when executed, naturally formed nested call structures. The same is true for data, the lists could be composed to form nested data structures just as they could be composed of nested execution structures.

Elixir, on the surface, is not homoiconic. However, the syntax on the surface is just a facade for the homiconic structure underneath. To peer under the covers we're going to need explore one more of Lisp's ideas.

### Quoting ###
Quoting is a Lispism which refers to the ability to not automatically execute an S-Expression containing a function call, but to instead allow that list to remain unexecuted and treated as data. This can be useful for rearranging pieces of code in macros which we'll get to later.

In Clojure, for instance, a quoted hello world expression might look like this

{% highlight clojure %}
  (quote (println "Hello world!"))
{% endhighlight %}

which can later be executed

{% highlight clojure %}
  (let [hello-world-program (quote (println "Hello world!"))]
    (eval hello-world-program)) ; => "Hello world!"
{% endhighlight %}

In Elixir, we have quote available which will return an unexecuted form. We can use this to explore the data structure underneath. Let's see an example of what a hello world program really looks like in Elixir.

{% highlight elixir %}
  quote do
    IO.puts("Hello world!")
  end # => { { {:., [], [{:__aliases__, [alias: false], [:IO]}, :puts]}, [], ["Hello world!"]} }
{% endhighlight %}

which can also later be executed

{% highlight elixir %}
  hello_world_program = quote do
    IO.puts("Hello world!")
  end
  Code.eval_quoted(hello_world_program) # => "Hello world!"
{% endhighlight %}

We will notice that Elixir's homoiconic representation is a bit different than Clojure's, but they're the same at heart. In Elixir, all expressions are represented as three element tuples. Let's look at a simple, straightforward example where we are simply adding two numbers.

{% highlight elixir %}
  quote do
    1 + 1
  end # => {:+, [], [1, 1]}
{% endhighlight %}

The first element in the tuple is the function to be executed, in this case we're executing `+`. The second element is metadata, which we'll gloss over for now. The third element is the function arguments. Knowing this we can nest these structures to form arbitrarily complex addition operations.

{% highlight elixir %}
  Code.eval_quoted({:+, [], [{:+, [], [1, 2]}, {:+, [], [3, 4]}]}) # => 10
{% endhighlight %}

### Abstract Syntax Trees ###

What we've been looking at thus far is the homoiconic representation of Abstract Syntax Trees<sup><a href="footnote_2">[2]</a></sup> in both Lisp languages and Elixir. Abstract syntax trees are simply a way to represent call structures in a tree like fashion. The nested code data structures described thus far are already in syntax tree form! Let's look at the addition example again.

{% highlight elixir %}
  { :+, [], # Root
      [{:+, [], # Child of Root
        [1,  # Leaf Node
        2]}, # Leaf Node
      {:+, [], # Child of Root
        [3, # Leaf Node
        4]}] # Leaf Node
    }
{% endhighlight %}

The root of execution according to this data structure is a function call to `+`. In order to complete this function call we need two sub-calls, namely two more calls to `+`. Then, at the leaves, we have the values which do not require execution because they are simply data. The leaf data is returned to the sub-calls, which can then complete. These sub-calls return to the root which can then complete and return the answer 10. We can see the relationship of the nested calls to the parent call in order to execute the addition of the sum of 1 and 2 with the sum of 3 and 4. Clojure represents the abstract syntax tree for this program in much the same way.

{% highlight clojure %}
  (+ # Root
    (+ # Child of Root
      1  # Leaf Node
      2) # Leaf Node
    (+ # Child of Root
      3  # Leaf Node
      4) # Leaf Node
  )
{% endhighlight %}

### Macros ###

Macros are a form of metaprogramming that appear in Lisp dialects as well as Elixir. A macro takes quoted forms (abstract syntax trees) and is free to edit the tree as it chooses in order to do any number of interesting things.

In order to understand macros, we need knowledge of one more Lispism, which is the unquote mechanism. All unquote does is execute what would be a quoted representation to produce a value. For example:

{% highlight elixir %}
  defmodule Example do
    def foo, do: 1

    def quoted_foo do
      quote do
        foo + foo
      end
    end

    def unquoted_foo do
      quote do
        unquote(foo) + unquote(foo)
      end
    end
  end

  Example.quoted_foo   # => {:+, [], [{:foo, [], Example}, {:foo, [], Example}]}
  Example.unquoted_foo # => {:+, [], [1, 1]}
{% endhighlight %}

To show what this can look like let's build a very simple [rspec](http://rspec.info/) style testing framework. We'll define a `describe` macro which will be semantically important to our specs, but will only serve to execute everything in the block underneath it. We'll define an `it` macro which will make a function on the module to later be invoked. Lastly, we'll define a `should_eq` macro which will raise an exception if the arguments it received are not equal. It's worth noting here that the reason we can produce functions on a module (or more broadly the reason macros can edit abstract syntax trees) is because macro expansion is a compile time step.

{% highlight elixir %}
  defmodule Specs do
    defmacro describe(_, specs) do
      quote do
        unquote(specs)
      end
    end

    defmacro it(name, spec) do
      quote do
        def unquote(binary_to_atom("#{name} spec"))() do # Note: this is generating a function on the module
          unquote(spec)
        end
      end
    end

    defmacro should_eq(value1, value2) do
      quote do
        if unquote(value1) != unquote(value2), do: raise "#{unquote(value1)} did not equal #{unquote(value2)}!"
      end
    end
  end
{% endhighlight %}

We can then use our Specs module's macros like so

{% highlight elixir %}
  defmodule SpecExample do
    import Specs

    describe "using macros" do
      it "passes as expected" do
        should_eq 1, 1
      end

      it "fails as expected" do
        should_eq 1, 2
      end
    end
  end

  apply(SpecExample, :"passes as expected spec", []) # => nil
  apply(SpecExample, :"fails as expected spec", [])  # => (RuntimeError) 1 did not equal 2!
{% endhighlight %}

### Macros (Almost) All The Way Down ###

Once you understand the power of macros you might begin to see why language implementers would use macros to implement the syntax we've all become accustomed to. For instance, `defn` in clojure is simply a macro! Similarly, `def` in Elixir is a macro. Upon further inspection we can see what `def` produces in terms of an abstract syntax tree.

{% highlight elixir %}
  defmodule Example do
    IO.inspect(quote do
      def foo do
        "bar"
      end
    end)
  end # => {:def, [context: Example], [{:foo, [], Example}, [do: "bar"]]}}
{% endhighlight %}

Lispers and Elixir implementers alike use macros to expand the ease of use of their languages. I'll leave spotting the other useful macro in the above example as an exercise to the reader.

### What Now? ###

With great power comes great responsibility. Macros require a high cognitive overhead and should not be introduced unless the benefits make that effort worthwhile. Of course, I invite all of you to explore the possibilities macros can provide.

### Standing On The Shoulders Of Giants ###
By now it should be obvious that lots of inspiration for Elixir comes from Lisp. Lisp's macro system offers huge benefits for those who can tame it. Needless to say, Elixir's future looks bright and exciting in part because it's standing on the shoulders of giants.

<a name="footnote_1"></a>
1.[Recursive Functions Of Symbolic Expressions And Their Computation By Machine](ftp://publications.ai.mit.edu/ai-publications/pdf/AIM-008.pdf) by John McCarthy defines a homoiconic language.

<a name="footnote_2"></a>
2.[Abstract Syntax Trees](http://en.wikipedia.org/wiki/Abstract_syntax_tree) are not homoiconic in some languages, but they appear as an intermediary data structure in compilation or interpretation of a program.
