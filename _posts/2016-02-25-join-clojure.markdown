---
layout: post
comments: false
title:  "Let's build something fun"
date:   2016-02-26 00:00:00 +0300
---

I want to show you something fun. I want you to show how simple Clojure is, and how fun creating something with it could be. I'm going to show you how to build something cool with it. And *please* don't be afraid of parenthesis, we are all adults here.

Let's build a chat bot, because everybody is into chat bots now. My girlfriend works for [Strelka Institute](http://strelka.com/en), which often organize public events about modern topic, so we are going to help people figure out is there anything happening there at certain moment. We'll call it *WAT bot for Strelka*.

In our way I will compare **Clojure** with **JavaScript**, because so many people are into node.js now or at least have tried and it should be easy to understand an analogy. Also in my opinion there are really no significant benefits of using node.js over Clojure in terms of convenience, tooling, speed or infrastructure ;)

Here is the big plan:

1. We will make a simple website first and ship it to Heroku
2. Next we will implement endpoint for Telegram and make parrot-bot
3. Then through the power of REPL we will improve our working environment
4. Finally we will teach our bot to understand plain english with help of [wit.ai](http://wit.ai)

In this post I'll cover first part in three pieces:

1. Creating a project
2. Making fancy Layouts
3. Deploying

## 0. Clojure very basics

To gain some understanding of what's going on here I have to explain just a little bit of Clojure syntax to you. Don't worry, I wont bother you you with *homoiconicity* and pull you through the ideas how awesome it is when your code is also a data. That is other flavor of fun than I'm serving you today. I will touch only few concepts as we go through. We start with function calls, symbols and keywords.

Weirdest is function calls, so we'll touch it first.

```clojure
(println "I am awesome")
```

This, as you probably guessed, will print a string to console. That is not that much different from node.js, right? Look, it even have same amount of parenthesis!

```javascript
// this is javascript
console.log("Also awesome")
```

What about more arguments to a function? Just stuff them into those parenthesis, exactly same as you used to do:

```clojure
(println "Spectacular " 3 " arguments")
```

Moving on, symbols a.k.a variables is something you define for later use.

I do not call them variables, because there wont be any much variation of their value, as they are immutable. So there is no point of calling them variables. You might ask how can you live without reassigning values? It happens that you merely need that in your daily routines, most of our programs nowadays are just passing data from on source to another, modifying it on the way. We do not operate with processor registers or memory for web-services, so we will be fine.

Let's define a number: <sup id="a1">[1](#f1)</sup>

```clojure
(def seventy-one 71)
(println "Seventy and one equals: " seventy-one)
```

Noticed a dash in a name? Yeah, that turned out to be more readable that underscores and camelCaseStuff. It keeps visual baseline and have some space for the eye, nice!

What about keywords? They are just constant strings and used mostly as keys in maps. Here is how you define map in JavaScript:

```javascript
// this is javascript
let user = {first_name: "Anton",
            anagram:    "Otann"};
```

And almost same thing in Clojure:

```clojure
(def user {:first-name "Anton"
           :anagram    "Otann"})
```

You can also notice, if you haven't before, the lack of commas. Yep, you don't need them, and no arguing over semicolons also. Both of them — gone.

One more thing that we'll need — arrays. Nothing complex here, they are much same as maps:

```javascript
// this is javascript
let digits = [1, 2, 3, 4, 5, 6, 7, 8, 9, 0]
```

Less commas, couple of parenthesis:

```clojure
(def digits [1 2 3 4 5 6 7 8 9 0])
```

That's everything we'll need for today from syntax.

## 1. Let's start building

Grab yourself **Leiningen**, which we will use for building and packaging our application. On Mac simply do it with a `brew` command and for other platforms check [their website](http://leiningen.org/).

    $ brew install leiningen

Right after that you'll be all set to create your new Clojure project:

    $ lein new compojure strelka-wat-bot

**Compojure** is a name of template that we are using and **strelka-wat-bot** is a name of our application. It should've created following file structure for you:

```
strelka-wat-bot
├── README.md
├── project.clj
├── resources
│   └── public
├── src
│   └── strelka_wat_bot
│       └── handler.clj
└── test
    └── strelka_wat_bot
        └── handler_test.clj
```        

Let's examine it. **README.md** is of no particular interest for us, is just what it is, prefilled description of our application that GitHub will render nicely for us, when we push code there. **resources/public** folder is empty inside and will store all public resources for our websrvice. **test** store unit-tests, which we will touh briefly in the end.

We will examine **src** later in this post and **project.clj** now.

### project.clj

```clojure
(defproject strelka-wat-bot "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :min-lein-version "2.0.0"

  :dependencies [[org.clojure/clojure "1.7.0"]
                 [ring/ring-defaults "0.1.5"]
                 [compojure "1.4.0"]]

  :plugins [[lein-ring "0.9.7"]]

  :ring {:handler strelka-wat-bot.handler/app}

  :profiles {:dev {:dependencies [[javax.servlet/servlet-api "2.5"]
                                  [ring/ring-mock "0.3.0"]]}})
```

So what about our project definition? Well, `defproject` tells Leiningen to define project with symbol named `strelka-wat-bot` and parameters passed in a same way you would pass it to the map or declare in your `package.json`. Important things to notice here is `:dependencies` and `:plugins` and I hope their meaning is intuitive for you, they declare external libraries that your project and Leiningen will use.

**Ring** is some king of standard of doing websrvices in Clojure land and that is what we will need to run our service locally and on Heroku.

**Compojure** is library that will save us a lot of time defining routes for Ring. I'll show you in a minute.

In Clojure land sources are by default stored in `src` subfolder and tests, well... in `test`! Then they are separated by namespaces (like modules) and there is subfolder for each namespace segment.

### src/strelka_wat_bot/handler.clj

```clojure
(ns strelka-wat-bot.handler
  (:require [compojure.core :refer :all]
            [compojure.route :as route]
            [ring.middleware.defaults :refer [wrap-defaults site-defaults]]))

(defroutes app-routes
  (GET "/" [] "Hello World")
  (route/not-found "Not Found"))

(def app
  (wrap-defaults app-routes site-defaults))
```

This is dumbest http handler you can possibly imagine, but first things first.

`(ns ...)` is namespace (again, just like modules in node) definition. And then follows list of dependencies for our namespace under `:requires` keyword. It reads like this:

*From **compojure.core** namespace we import everything there is, **compojure.core** we will reference as **route** and from **ring.middleware.defaults** we want only **wrap-defaults** and **site-defaults** symbols.*

Same thing would look like this in javascript:

```javascript
//this is almost javascript
import { * } from 'compojure/core' // not possible in js, so using imagination here
import * as route from 'compojure/route'
import {wrap-defaults, site-defaults} from 'ring/middleware/defaults'
```

And no relative imports here, you decalre namespace, you reference it by absolute name. No more misunderstanding, how deep you are nested and how many `/../../../` you need to reach *that thing* from up top.

Next we define simple route, for root request we will return `"Hello world"` string and for everything else `"Not Found"`.

Let's try it out! From your console run following

    $ lein ring server

Remember I mentioned Ring plugin? Here we are using it. You should've had new tab opened in your browser with just "Hello world" there. Good! And id you are not impressed, I remind you that until now you just typed two commands in condole and you have a running server already!

## 2. Fancy layout

But let's improve that page a little and develop a little bit more elaborate landing page for our application. We'll use awesome **Hiccup** library for that.

Add it's dependency `[hiccup "1.0.5"]` to your `project.clj` so it will look like this:

```clojure
:dependencies [[org.clojure/clojure "1.7.0"]
               [ring/ring-defaults "0.1.5"]
               [compojure "1.4.0"]
               [hiccup "1.0.5"]]
```

Then add file `src/strelka_wat_bot/static.clj` with following content:

```clojure
(ns strelka-wat-bot.static
  "Layouts for server-side rendered pages"
  (:require [hiccup.core :refer [html]]
            [hiccup.page :refer [include-css]]))

(def home
  (html
    [:html
     [:head
      [:meta {:charset "utf-8"}]
      [:meta {:name "viewport" :content "width=device-width, initial-scale=1"}]
      [:title "Strelka Wat Bot"]

      (include-css "//oss.maxcdn.com/semantic-ui/2.1.7/semantic.min.css")]

     [:body
      [:div.ui.container
       [:div.ui.vertical.masthead.center.aligned.segment
        {:style "padding-top: 100px"}
        [:div.ui.test.container
         [:img {:src "/img/1024.png" :style "height: 256px; width: 256px;"}]
         [:h1.header "Welcome to Strelka Wat Bot"]
         [:h3 "Telegram bot that knows what's happening in Strelka Institute"]
         [:a.ui.huge.button
          {:href "https://telegram.me/StrelkaWatBot"}
          "Start Conversation"
          [:i.icon.right.arrow]]]]]]]))
```

Whoa! We just declared whole html page! How did that happened?

To understand it, start with imports in `:require` section. We require `html` and `include-css` symbols. Then we define symbol with name `home`. That symbol will equal result of calling `html` function with really nested structure as an argument.

As might guessed, function called html will produce, right, HTML! Hiccups has a compact way of declaring markup without XML-style redundancy and it's kinda intuitive to understand. First element in array is tag's name, second is attributes if it is a map and everything else is tag's body. That's it!

And you can even call methods inside like we do with css inclusion, and here is your template composition technique.

Now let's update our `handler.clj` to look like this:

```clojure
(ns strelka-wat-bot.handler
  (:require [compojure.core :refer :all]
            [compojure.route :as route]
            [ring.middleware.defaults :refer [wrap-defaults site-defaults]]

            [strelka-wat-bot.static :as static]))

(defroutes app-routes
  (GET "/" [] static/home)
  (route/not-found "Not Found"))

(def app
  (wrap-defaults app-routes site-defaults))
```

We added one more import and replaced `"Hello world"` with `static/home` symbol. Refresh your browser and you should see some fancy layout. Now you can be really proud of yourself! You can play with hiccups and check results without restarting `lein` by just saving file and refreshing the page. <sup id="a2">[2](#f2)</sup>

## 3. Ship it

So far so good, but let's deploy it! Go to [Heroku.com](https://signup.heroku.com/) and create an account if you don't have one yet. This is amazing service that will allow you to see your code online in no time and after just a git commit.

Once installed, authenticate with your credentials:

```shell
$ heroku login
```

For Heroku we'll need something runnable from command line on a custom port, provided through `PORT` environment variable. That does not sound very complex, let's add one more file: `src/strelka_wat_bot/main.clj`:

```clojure
(ns strelka-wat-bot.main
  (:gen-class)
  (:require [ring.adapter.jetty :refer [run-jetty]]
            [strelka-wat-bot.handler :refer [app]]))

(def port (Integer/parseInt (System/getenv "PORT")))

(defn -main [& args]
  (run-jetty app {:port port :join? false}))
```

Only new things here is `(:gen-class)` which tells compiler to generate old-style java class for that namespace and `defn`, which is shorthand for *define function*.

We don't import `Integer` and `System`, because they are part of globally acessible symbols, like `ns`, `def` or `defn` (and many-many others).

```javascript
// this is poor ES6 impersonation of code above
import { run_jetty } from 'ring/adapter/jetty'
import { app } from 'strelka-wat-bot/handler'

const port = parseInt(System.getenv("PORT"))

function _main(...args) {
  run_jetty(app, {port: port, join: false})
}
```

We will also need to mention our new work in `project.clj`, add following anywhere there:

```clojure
:profiles {:uberjar {:main strelka-wat-bot.main
                     :uberjar-name "strelka-wat-bot.jar"
                     :aot :all}}
```

That says that when Heroku will pack our application to jar-file, our newly created namespace will be used as an entrypoint and ahead-of-time compilation will be applied to all the code, which will speed up things for the first start.

Finally, add `Procfile` for Heroku with following content:

```
web: java -cp target/strelka-wat-bot.jar clojure.main -m strelka-wat-bot.main
```

Now we are ready to deploy it.
In order to push code to heroku, create git repository and, everything there and push:

```shell
$ git init
$ git add .
$ git commit -m "Initial commit"
$ heroku create
$ git push heroku master
```

After you see really long log of how your application was built remotely, you now will be able to check it out online!

```shell
$ heroku open
```

Should open a new website in your browser. Now not just you, but your friends can be really proud of you, when you tell them that you launched a website in less that an hour.

<hr>

<b id="f1">1</b> By the way, there is [whole library](https://github.com/gfredericks/seventy-one) just for that number, if you even need an extra one (or seventy-one, ha!) in your project. [↩](#a1)

<b id="f2">2</b> In the next posts I will show you how to do same thing without even reloading your page! [↩](#a2)
