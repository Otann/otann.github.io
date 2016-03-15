---
layout:   post
comments: true
sharable: true
category: Web-Fandango

date:   2016-03-05 00:00:00 +0300
title:  Episode 01. Clojure Kickoff
image:  /media/posts/2016-03-05-clojure-kickoff/header.jpg
description: >
  Today I am going to cover basic things about Clojure, so you can understand bits and pieces of code, that I will share with you to illustrate progress on my projects.
---

Cheers!

Today I am going to cover basic things about Clojure, so you can understand bits and pieces of code, that I will share with you to illustrate progress on my projects.

I hope you have some experience of building web applications using Node.js, Python or Ruby, so the concepts here will be easier to grasp. However, you might be able to get through without such an experience and still have fun ;)

Obviously, there is no lack of good starting points on Clojure if you want to know more:

 - Kyle Kingsbury's [From The Ground Up](https://aphyr.com/posts/301-clojure-from-the-ground-up-welcome)
 - Clojure [For The Brave And True](http://www.braveclojure.com/)
 - My personal favorite &mdash; [Joy of Clojure](https://www.manning.com/books/the-joy-of-clojure-second-edition) Book


## Build system

Each of the current stacks today has, at least, one it's own dependency management and task running system, like Pip, Rake, NPM, Gulp, Grunt. Clojure is no exception, and our build tool is called **Leiningen**. We will use it to run REPL today to get the taste of Clojure.

You can install Leiningen on a Mac using Homebrew:

    $ brew install leiningen

For other platforms visit [official website](http://leiningen.org/) and follow instructions there.

To try concepts that I'm about to explain, you need to launch REPL. This is similar to how you do it with `ipython`, `node` or `irb` commands. Here is what it looks like:

```
$ lein repl
nREPL server started on port 62066 on host 127.0.0.1 - nrepl://127.0.0.1:62066
REPL-y 0.3.7, nREPL 0.2.10
Clojure 1.7.0
Java HotSpot(TM) 64-Bit Server VM 1.8.0_45-b14
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

user=>
```

REPL stands for read-eval-print-loop, where you can type in some Clojure code, which will be evaluated and results will be printed out to you.

## Symbols, functions, collections

That's all that we are going to need to start the Web Fandango series. Other concepts I will explain by the way, but these three are essential.

Let's start with data and primitives. Like any of the mentioned above languages, there are booleans, integers, real numbers and strings:

```clojure
user=> ; this is a comment, to let you distinguish description from code

user=> nil ; this stands for eternal void and nothingness
nil
user=> true ; this is a positive boolean
true
user=> false ; and it's negative opposite
false
user=> 0 ; integer numbers are presented usually
0
user=> 1.5 ; and so are real numbers
1.5
user=> "Quick brown fox" ; and strings
"Quick brown fox"
```

As a bonus, we also have rationals as a separate primitive. I do not understand fully why just letting you know :)

```clojure
user=> 100/6
50/3
```

There is another uncommon primitive &mdash; keywords. They are much like their doppelgangers from Ruby. It is a way of representing a string literal and efficiently storing them in memory. They look like this:

```clojure
user=> :keyword
:keyword
user=> :another-keyword
:another-keyword
```

Their main strength and power comes with collections, which I will show you in a moment.

### Symbols

Closest analogy is identifiers in other languages.

Let's give a piece of data a name to to play with it:

```clojure
user=> (def seventy-one 71) ; this binds value 71 to symbol seventy-one
#'user/seventy-one
user=> seventy-one ; so we can reference this value later
71
```

> Note that naming convention in Clojure uses dashes instead of underscores or camelCase. I find it more eye-friendly and a little bit close-to-real-english.

If we decide to modify it's value, initial value will stay intact, like so (`inc` is an increment function):

```clojure
user=> (inc seventy-one)
72
user=> seventy-one
71
```

It is very comforting that once you construct some data and have a reference to it, the data stays forever as it is. No other thread or a library will ever change it. And if you want to modify its value, you will get a new *thing*. Your old *thing* will be left untouched. Don't worry about memory consumption, Clojure has something called *persistent data structures* that will effectively store immutable collections in memory.

Variables are rarely used in Clojure and people are very comfortable without them. Usually, they are used as a constant reference to some variadic value.

You might think that it's a terrible life without reassignment values to *identifiers*. But think about daily routines with web applications -- we do not need those things for solving problems. We do not operate with processor registers or direct memory access, we just passing data from on source to another, modifying it a little on the way. Reassignments are just how we used to solve them, and it doesn't have to be that way.

So `def` defines root binding, that other pieces of your program can use. What if we want a closer scope? We can use "temporary" bindings that will be valid only within a scope that we defined:

```clojure
user=> (let [a "Hello"
  #_=>       b " world"]
  #_=>   (println a b))
Hello  world
nil
user=> a

CompilerException java.lang.RuntimeException: Unable to resolve symbol: a in this context
```

As you can see here, `let` *lets* us to use symbols like temporary aliases and they are not accessible outside let expression. Binding is probably the most complicated concept I'm going to introduce you to, but this is also the most important one. That notation is very widely used in function definitions when you define one thing after another, where next one is a combination of some previously defined symbols.

### Functions

Primitives are not much use without a way to perform operations on them. Remember how you would call a function on a value in those popular languages?

```python
# this is Python
>>> print(71)
71
>>>
```

```ruby
# this is Ruby
irb(main):001:0> puts 71
71
=> nil
irb(main):002:0>
```

```javascript
// this is javascript
> console.log(71)
71
undefined
```

It's almost the same in Clojure. There is a small twist that scares a lot of people away, but I ask you to be brave. Behold:

```clojure
user=> (println seventy-one)
71
nil
```

Yep. The function name is inside the parenthesis. And that is the main reason everyone is grumbling about Clojure's syntax — parenthesis!

> You might even find some pros in that. But I think that parenthesis topic is hugely overrated and does not worth a real discussion. In my opinion, the comfort of using a language comes from things like the standard library, community frameworks and integrity if ideas in that community. Not from a notion of where to place parenthesis.

So here a few examples of basic functions in Clojure:

```clojure
user=> (+ 2/3 4/5)
22/15
user=> (println "2 + 2 =" (+ 2 2))
2 + 2 = 4
nil
user=> (inc seventy-one)
72
user=> (+ seventy-one 10)
81
```

Note, that there are no special delimiters, like a comma for arguments or semicolon for statements.

Again, this prefix notation may look weird at first glance for infix operations like plus or comparison. But, what if I say that this operation in Clojure can take more that two arguments?

```clojure
user=> (+ 1 2 3 4 5) ; returns sum of a sequence
15
user=> (< 0 1 2 3 4) ; returns true for monotonically increasing sequence
true
```

There is no special syntax in Clojure to perform actions on data, like operators or methods, only functions. To call a function, you place it's name as the first argument in parenthesis.

Say you want to define your function that will return square of a number. You do it like so:

```clojure
user=> (defn square [x] (* x x))
#'user/square
user=> (square 7)
49
```

It's considered to be a good practice to expand function definition to multiple lines and add a comment when you are writing project code:

```clojure
(defn square
  "Returns square of a number"
  [x]
  (* x x))
```

## Collections

Clojure have all primary collections that we are used to: vectors (arrays), maps (hashes, dictionaries) and sets:

```clojure
user=> [1 2 3 4] ; things in square braces are turned in a vector
[1 2 3 4]
user=> {:first "Manny" :second "Calavera"} ; in curly braces it becomes a map
{:first "Manny", :second "Calavera"}
user=> #{"Quick" "brown" "fox"} ; with a hash in front is is a set
#{"fox" "Quick" "brown"}
```

Like other Lisp languages (list-processing) operations with collections are amazingly convenient and remarkably easy to read:

```clojure
user=> (first [1 2 3])
1
user=> (nth [4 5 6 7] 2)
6
user=> (last [8 9 10])
10
```

You can use functional goodies when working with lists:

```clojure
user=> (map inc [1 2 3 4])
(2 3 4 5)
user=> (reduce + [1 2 3 4])
10
```

But unlike other Lisps, Clojure is especially powerful with maps. Here is how you would imagine getting value from a map:

```clojure
user=> (def character {:first "Manny" :last "Calavera"})
#'user/manny
user=> (get character :last)
"Calavera"
```

But in Clojure maps act like `get` functions on themselves. And so works keywords, when applied as a function to a map:

```clojure
user=> (character :last)
"Calavera"
user=> (:last character)
"Calavera"
```

So are sets, they return you an element, if it is present and `nil` otherwise:

```clojure
user=> (def guys #{"The Good" "The Bad" "The Ugly"})
#'user/guys
user=> (guys "The Good")
"The Good"
user=> (guys "Unexisting")
nil
```

Some brief example of how to put more stuff into your collections:

```clojure
user=> (conj [1 2] 3 4) ; reads as conjoin
[1 2 3 4]
user=> (conj #{:one :three :four} :two)
#{:one :three :four :two}
user=> (assoc character :status :trustworthy)
{:first "Manny", :last "Calavera", :status :trustworthy}

; you can also combine collections together:
user=> (into [1 2] [3 4 5])
[1 2 3 4 5]
user=> (into #{:one :three :four} [:two])
#{:one :three :four :two}
user=> (into character {:status :trustworthy :full "Manuel Calavera"})
{:first "Manny", :last "Calavera", :status :trustworthy, :full "Manuel Calavera"}
```

## One more thing

As a bonus, I will try to explain you the relatively complex concept in Clojure that is often used to simplify things, but can be tricky if overused. I'm talking about destructuring.

When you have a binding, as you saw in `let` form or arguments in a function definition, you can apply some operations to collections right in the definition to extract values.

For instance here is how you might implement a function that greets a person:

```clojure
user=> (defn greeting [person] (str "Hello " (:first person)))
#'user/greeting
user=> (greeting character)
"Hello Manny"
```

You can optimize it a little with moving extraction of a name to parameters list:

```clojure
user=> (defn advanced-greeting [{name :first}] (str "Hello, dear " name))
#'user/advanced-greeting
user=> (advanced-greeting character)
"Hello, dear Manny"
```

That gives you a little more understanding of what function expects as an argument, but don't rely on this too much. My advice is to use more reasonable function name instead and extract values inside a function in most of the cases.

## That's all

**That is all you need to start read and understand Clojure code!**

I bet it wasn't that hard and sure you are doing great after you learned a couple of things.

There are plenty of stuff you can do with functions and collections, but as I promised, I'm covering only the basics you need to understand the code. Though I encourage you to dive deeper into Clojure and read [free chapter](https://manning-content.s3.amazonaws.com/download/2/ecbf832-5092-44f9-a7b8-8a7336bb40d1/Joy2_Ch02.pdf), from an excellent book called [Joy of Clojure](https://www.manning.com/books/the-joy-of-clojure-second-edition). It not only shows you how to build things with that language but why the language is designed as it is and what are the reasons behind those decisions.

*Now that is absolutely all for today. Thank you for reading and I hope you enjoyed this post.*
