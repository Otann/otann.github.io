---
layout: post
comments: false
title:  "Web Fandango, Episode 01. Clojure Kickoff"
date:   2016-03-01 00:00:00 +0300
category: web-fandango
---

I am starting a series of posts to share with you my passion about Clojure programming.
I will show you how with minimal effort you can build amazing web applications with it and how you can benefit from choosing it for your next small experimental project.

By the way I hope to touch topics like deployment, metrics or monitoring so in case your experiment grows into serious application, you would know.

I suspect you have the experience of building some web applications using Node.js, Python or Ruby so the concepts here will be easier to grasp. However, you might be able to get through without such an experience and still have fun ;)

To help you understand the content of the code snippets, I'll introduce you to the very few basic concept of the language.

Obviously there is no lack of good starting points on Clojure if you want to know more:

 - Kyle Kingsbury's [From The Ground Up](https://aphyr.com/posts/301-clojure-from-the-ground-up-welcome)
 - Clojure [For The Brave And True](http://www.braveclojure.com/)
 - My personal favorite &mdash; [Joy of Clojure](http://www.joyofclojure.com/foreword/index.html) Book


## Build system

Each of the mentioned languages has at least one it's own package management and task running system, like Pip, Rake, NPM, Gulp, Grunt. Clojure is no exception and it's build tool is called **Leiningen**.

You can install it on a mac using Homebrew:

    $ brew install leiningen

For other platforms visit [official website](http://leiningen.org/) and follow instructions there.

To try concepts that I'm about to explain, you need to run REPL. This is similar to how you do it with `ptyhon`, `node` or `irb` commands. Here is what it looks like:

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

## Functions, symbols, collection and namespaces

That's all that we are going to need to start the Web Fandango series. Other concepts I will explain by the way, but these four are essential.

Let's start with data, here is some primitives. Like any of the mentioned above languages we have booleans, integers, real numbers and strings:

```clojure
user=> ;; this is a comment

user=> true ;; this is a boolean
true
user=> false ;; and it's opposite
false
user=> 0
0
user=> 1.5
1.5
user=> "Quick brown fox"
"Quick brown fox"
```

As a bonus, for some reason I personally do not understand fully, we also have fractions as a separate primitive.

```clojure
user=> 2/3
2/3
```

What can you do with a primitives? Same things that you used to do in other languages — perform operations on them, but with a small twist: you put operation first and the all of it's argument. To group things together you use parenthesis:

```clojure
user=> (+ 2/3 4/5)
22/15
user=> (println "2 + 2 =" (+ 2 2))
2 + 2 = 4
nil
```

As you can see, there is no special delimiters for function arguments (like comma) or statements (like semicolons). This form of writing code may look weird to you at first, as it definitely did to me, but after a while I got used to it and I can say that it's quite the same. After you learn just one extra shortcuts for cutting expressions, moving code around becomes a pleasure instead of pain.

That also becomes handy when your familiar binary operators start to accept more that two parameters:

```clojure
user=> (+ 1 2 3 4 5) ; returns sum of a sequence
15
user=> (< 0 1 2 3 4) ; returns true for monotonically increasing sequence
true
```

### Symbols

Symbols' closest analogy is constants identifiers in other languages.

I do not call them variables, because there is not much variation of their value, usually. You might think that it's a tough world without variation, but in our daily work with web applications we do not operate with processor registers or memory, we just passing data from on source to another, modifying it a little on the way, so we'll be fine.


Let's define some data to modify later, a number for example: <sup id="a1">[1](#f1)</sup>

```clojure
(def seventy-one 71)
(println "Seventy and one equals: " seventy-one)
```

Noticed a dash in a name? This is clojure naming convention. For me it turned out to be more readable that underscores and camelCaseNotation. It keeps visual baseline and have some space between words, which helps eyes to grasp information quickly.

### Functions

There is nothing special syntax in Clojure to perform actions with objects, like operators. Functions come first in Clojure. It offers you a little bit different view on every problem you solve with a code, instead of start thinking about "what is that thing and what it can do" you go with "what should I do with this data".

That said, in example above `+` and `println` are functions and they get called, when Clojure evaluates expression with parenthesis (spoiler: it's just a list).

Say you want to define a function that will return square for a number. You define function using following notation:

```clojure
(defn square [x] (* x x))
```

Or you can expand it to multiple lines and if you are in a mood — add a comment:

```clojure
(defn square
  "Returns square of a number"
  [x]
  (* x x))
```

Most of the functional programming goodies you might have used, you can expect to be here:
