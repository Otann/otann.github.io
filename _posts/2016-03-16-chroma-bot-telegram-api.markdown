---
layout:   post
comments: true
sharable: true
category: Web-Fandango

date:   2016-03-13 00:00:00 +0300
title:  Episode 03. Telegram Bot API
image:  /media/posts/2016-03-15-chroma-bot-telegram-api/header.jpg
description: >
  In this episode we will connect Clojure application to Telegram
  via long-polling with help of core.async
---

Hello, dear fellows!

Welcome to third episode of Web Fandango series, where we build simple and awesome projects in Clojure. In the last episode, we made a simple web service with a landing page for Chroma Bot. You can check out where we stopped on the previous episode at GitHub repo  [otann/chroma-bot:episode-02](https://github.com/Otann/chroma-bot/tree/episode-02)

Today we follow previously sketched plan and after the landing and deploy works, let's connect the bot to Telegram. As you can see in their [Bot API Documentation](https://core.telegram.org/bots/api), we should make HTTP calls on provided endpoint to receive new messages from users. To do that let's implement some wrappers for Telegram API.

## Wrap Telegram API

Add `[clj-http "2.0.0"]` and `[cheshire "5.5.0"]` to your project dependencies in `project.clj` file.

* [`clj-http`](https://github.com/dakrone/clj-http) is a library for making external HTTP calls
* [`cheshire`](https://github.com/dakrone/cheshire) parses and constructs JSON data from Clojure structures

Then create `telegram` folder in your source `src` directory, which will be a home namespace for all our Telegram related code.

> If we keep it clean and independent from other pieces of the code, we might even turn it into a separate library later! That, in my opinion, is a good way of thinking to keep your system truly modular.

Start by adding `api.clj` file there with the following content. Let's do a namespace declaration and some constants first:

```clojure
(ns telegram.api
  (:require [taoensso.timbre :as log]
            [clj-http.client :as http]))

(def base-url "https://api.telegram.org/bot")

(def token (atom nil))            
```

What is an atom? It's a box for variadic things, which you can use for state management. They are incredibly helpful when dealing with concurrent operations because they guarantee that no two threads will access their value at the same time. And they are very straightforward. There are only three things you can do with atoms:

* `deref`erence them to get their value
* `swap!` to apply a function to a value and receive a result
* `reset!` to completely replace old value with your new one

Here is some example from the REPL:

```clojure
user=> (def n (atom 0))
#'user/n
user=> (deref n)
0
user=> @n
0
user=> (reset! n 100)
100
user=> (swap! n inc)
101
user=> @n
101
```

We will fill value for `token` atom later, in the function responsible for initialization of whole Telegram API. Now let's imagine we have it and write a function to get updates from Telegram, using [provided documentation](https://core.telegram.org/bots/api#getupdates):

```clojure
(defn get-updates
  "Receive updates from Bot via long-polling endpoint"
  [{:keys [limit offset timeout]}]
  (let [url (str base-url @token "/getUpdates")
        query {:timeout (or timeout 1)
               :offset  (or offset 0)
               :limit   (or limit 100)}
        resp (http/get url {:as :json :query-params query})]
    (-> resp :body :result)))
```

This function receives one argument and expects it to be a map with following keys: `[limit offset timeout]`. This thing is called destructuring, which I explained briefly [a couple of episodes ago](web-fandango/clojure-kickoff/). So this definition presumes that it one will call this function like this:

```clojure
(get updates {:limit 10
              :offset 0
              :timeout 100})
```

If some of the keys will be omitted, then corresponding symbol inside method definition will have `nil` value.

Then there is `let` expression that declares some intermediary stuff:

* `url (str base-url @token "/getUpdates")` declares a new symbol `url` that is string concatenation of `base-url`, Telegram token and method that we should use to get updates
* `query` defines map with same values as method parameters but adds defaults if parameters are missing
* `resp` is a result of calling (http/get ...) function that will make an HTTP request and return a map with the response from remote server. We also ask to interpret the response as JSON and, therefore, coerce it to standard Clojure map.

What does the `->` arrow means? It is a handy way of writing multiple functions that call each other. Instead of this clumsy way for getting something deep from a map:

```
(first (:messages (:result (:body resp))))
```

We can write this, which is more easy on the eye:

```
(-> resp :body :result :messages first)
```

After receiving results, we would want to make a reply. Let's implement a function for sending messages using a method `sendMessage` from [documentation](https://core.telegram.org/bots/api#sendmessage):

```clojure
(defn send-message
  "Sends message to chat"
  [chat-id options text]
   (log/debug "Sending message" text "to" chat-id)
   (let [url   (str base-url @token "/sendMessage")
         query (into {:chat_id chat-id :text text} options)
         resp  (http/get url {:as :json :query-params query})]
     (log/debug "Got response from server" (:body resp))))
```

That function is identical to one above, the same thing going on here, we prepare arguments for a method using intermediary symbols, then make a call and log it to the console.

Let's make this function a little bit more convenient. Say, most of the time we don't have any special options to pass, so we want to call it with two arguments instead of four. That is no problem in Clojure to define multi-[arity](https://en.wikipedia.org/wiki/Arity) function:

```clojure
(defn send-message
  "Sends message to user"

  ([chat-id text] (send-message chat-id {} text))

  ([chat-id options text]
   (log/debug "Sending message" text "to" chat-id)
   (let [url (str base-url @token "/sendMessage")
         query (into {:chat_id chat-id :text text} options)
         resp (http/get url {:as :json :query-params query})]
     (log/debug "Got response from server" (:body resp)))))
```

It's like two definitions by the price of one! Function with the arity of two calls three arity version with the default argument. In that way you can call `send-message` in two ways:

```clojure
(api/send-message chat-id "Hello there!")
; or like this:
(api/send-message chat-id
                  {:parse_mode "Markdown"}
                  "Well, *hello there!*")
```

## Do infinite loop

Now having a way to get updates in our code, we should implement infinite loop where we will do it.

Create a file `polling.clj` in `telegram` namespace, where we are going to implement this. Start with `ns` definition:

```clojure
(ns telegram.polling
  "Declares long-polling routines to communicate with Telegram Bot API"
  (:require [clojure.core.async :refer [>!! <! go chan close! thread]]
            [telegram.api :as api]            
            [taoensso.timbre :as log]))
```

Now, let's talk about concurrency a little bit because there are no infinite loops without touching that topic.

I am not in the mood for taming threads manually today and would prefer a good abstraction for that matter. There are two major classes: [actors](https://en.wikipedia.org/wiki/Actor_model) and [communicating sequential processes](https://en.wikipedia.org/wiki/Communicating_sequential_processes). They both utilise an idea of passing messages between independent entities:

* **Actors** are more OOP-like. You focus on actors, define their purpose of life (like class) and how they react to messages (like methods). And they also have a state, like instances.
  * **pro:** You can easily put different actors on different network nodes and scale in quantities since they process messages independently of each other
  * **con:** You have to keep an eye on message queues not to overflow, because processes of sending and receiving messages are not tied together.
  * **con:** Actors exchange messages with each other directly, so they are logically tightly coupled to each other because they have to keep in mind each other's roles.

* **SCP** is functional-like approach. You focus on message queues. You define channel's role like you define a function, it should receive things of one sort and return the other.
  * **pro:** Emitters and receivers of messages knows only about message queue purpose, so they are loosely coupled together.
  * **con:** Often processes should wait for rendezvous to happen to continue their work. Putting or pulling messages will block execution until other side processes or provides a message. In that way, you don't have to worry about overflows of messages.
  * **con:**  That rendezvous could give you troubles on scaling your system physically.

Since Clojure is a deeply functional language, it is no surprise that we would use SCP approach today. The library for that is `core.async`. Its main concept is *channel*. Imagine it as a virtual place where data flows from one end to another, like a conveyor belt or a queue. You put data to one side and pull from the other:

<img src="/media/posts/2016-03-15-chroma-bot-telegram-api/channel.png" width="300px"/>

This is a very convenient concept to reason about, because you can logically separate consuming and producing data in your code.

There are two flavors of threads when working with channels: using real and parked ones.

 * Real threads are wrapped around Java primitives and ways of blocking
 * Parked ones are modeled after Go threading model.
   If you choose to work with parked threads, then you should wrap communications with channels into `go` expression.

I advise using parked threads unless there is real need for real ones.

### Consume

If you imagine yourself as a consumer, your job is to ask channel if it has any data that you could process and wait until it does.

<img src="/media/posts/2016-03-15-chroma-bot-telegram-api/consumers.png" width="300px"/>

So imagine we have a channel called `updates`, where updates from Telegram would appear and a function `handle` that processes them. Here is how we can do the infinite loop trick for consumer side:

```clojure
(go (loop []
      (handle (<! updates))
      (if running (recur))))
```

* `(<! updates)` pulls a new message from the channel. If there are no messages, thread that is performing this operation within `go` expression will be parked, and dedicated thread pool will carry out some other *go-routine*.
* `(if running (recur))` checks some previously declared state that marks if the process should continue and if so, calls `recur`, which means that expression within `loop` will be executed again.

That will make an infinite loop for handling messages.

### Produce

Now, how would we populate this channel with data? As consumers, producers also care just for their end of the conveyor belt:

<img src="/media/posts/2016-03-15-chroma-bot-telegram-api/producers.png" width="300px"/>

For producing messages, we would need the same kind of a `loop` expression, that will verify if it should be `running` and if so, call `recur`. But here is the difference with consuming — HTTP call that checks updates will hang till there is new data on the Telegram server. It will be a very unpleasant thing to do, if we block consistently part of the shared thread pool, which others rely on. So here we have a genuine need for using real dedicated Java thread. Therefore we use `>!!` operation for pushing data into the channel.

```clojure
(thread (loop [offset 0]
          (let [updates-data (api/get-updates {:offset offset})
                new-offset   (if (empty? updates-data)
                               offset
                               (-> updates-data last :update_id inc))]
            (doseq [update updates-data] (>!! updates update))
            (if running (recur new-offset)))))
```

* Here instead of `go` you see `thread`. That starts a new thread that will execute expression passed as an argument untill it finishes. After that thread will be disposed.
* `loop` here declares a binding: `offset` symbol with initial value 0. When later this loop will be called again with `recur`, offset symbol will be equal to parameter that was passed to `recur`
* We use this technique to increase offsets of the query gradually, as [Telegram documentation](https://core.telegram.org/bots/api#getupdates) suggest. Initial offset equals zero, but for subsequent calls, we choose new offset as an incremented id of the last received message.
* `doseq` means *do for sequence* and in same way as `map` does, but with a neat binding that improves readability

Now you can see that there are a couple of things (`running` and `updates`) that represent the state and will be used by multiple threads. They are perfect spots for using atoms.

Let's declare them and combine both pieces of code into one function `start!`. Here how the whole `polling.clj` will look like then:

```clojure
(ns telegram.polling
  "Declares long-polling routines to communicate with Telegram Bot API"
  (:require [taoensso.timbre :as log]
            [clojure.core.async :refer [>!! <! go chan close! thread]]

            [telegram.api :as api]
            [telegram.handlers :refer [handle]]))

;; this holds updates from Telegram
(def updates (atom nil))

;; this controls if loops are rolling
(def running (atom false))

(defn start!
  "Starts long-polling process"
  []
  (log/debug "Trying to start polling threads")
  (reset! updates (chan))
  (reset! running true)

  ;; Start infinite loop inside go-routine
  ;; that will pull messages from channel
  (go (loop []
        (handle (<! @updates))
        (if @running (recur))))

  ;; Start thread with polling process
  ;; that will populate channel
  (thread (loop [offset 0]
            (let [updates-data (api/get-updates {:offset offset})
                  new-offset (if (empty? updates-data)
                               offset
                               (-> updates-data last :update_id inc))]
              (doseq [update updates-data] (>!! @updates update))
              (if @running (recur new-offset)))))

  (log/info "Started long-polling for Telegram updates"))

(defn stop!
  "Stops everything"
  []
  (reset! running false)
  (close! @updates))
```

If you have any questions about the code, please don't hesitate and ask in a comment section below the post.

And if you are interested more in the details of `core.async` and reasoning behind its design decisions, check out [this talk by Rich Hickey](http://www.infoq.com/presentations/clojure-core-async), author of Clojure and creator of that library. As usual, I've covered only basics for the sake of fun and speed.

Here I referenced unmentioned before `telegram.handlers` namespace. That is because I think handling messages is a matter of concern of a whole different module, even if id does not exists yet. It's not a big deal us to create one, so let's do this right now.

## Handle messages

If we want to externalize behavior of our bot from a Telegram communication, we should design a place where we would store chat message handling and let Telegram module use them. Here it is:

```clojure
(ns telegram.handlers
  (:require [taoensso.timbre :as log]))

(def ^:private handlers (atom []))

(defn add-handler! [handler]
  (swap! handlers #(conj % handler)))

(defn reset-handlers!
  ([] (reset-handlers! []))
  ([value] (reset! handlers value)))

(defn handle [update]
  (if (empty? @handlers)
    (log/warn "There were no handlers to process update from Telegram")
    (doseq [handler @handlers]
      (try
        (handler update)
        (catch Exception e
          (log/error e "Got error in on of the handlers:"))))))
```

* `^:private` is a way of making a symbol visible inside its namespace only. Since handlers is a state, we want to be careful and keep to ourselves.
* `handlers` is an atom that holds list and `add-handler!` and `reset-handlers!` are public functions that add and remove handlers from that list
* what's up with the exclamation marks, do they add specific behavior? No! But it's a beautiful part of a convention to indicate that this method operates on the state. The Similar thing is to use question marks at the end of the name for boolean predicates, like `empty?`
* `handle` is function that we planned on using before in the polling loop
* `try` checks that whatever happens inside the handler provided from the outside will not ruin running thread with an exception.

## Wrap it up

The last thing to do is to write initialization that will be a handy way to set up a value for the token, add some handlers and start long-polling loops.

Add file `core.clj` to your `telegram` namespace:


```clojure
(ns telegram.core
  (:require [telegram.api :as api]
            [telegram.handlers :as h]
            [telegram.polling :as polling]))


(defn init!
  "Initializes Telegram cliend and starts all necessary routines"
  [{:keys [token handlers polling]}]
  (if token
    (reset! api/token token)
    (throw (Exception. "Can't intialize Telegram without a token")))

  (if (seq handlers) (h/reset-handlers! handlers))
  (if polling (polling/start!)))
```  

Here we have single `init` function that ties everything we've done together:

* it resets Telegram token
* it sets handlers if the caller provided them
* it starts long-polling process if `:polling` key pointed to some true-like value

## Try it out

Now it's time to start our bot and give it a ride.

If you are not a Telegram user yet, got to [Telegram.org](https://telegram.org/) and sign up using your favorite platform. Then start a conversation with the [Botfather](https://telegram.me/BotFather).

Type there `/newbot` to start bot creation process:

<!-- this is my retina screenshot, so we force width here -->
<img src="/media/posts/2016-03-15-chroma-bot-telegram-api/botfather-1.png" width="538px"/>

Work through the all necessary steps to create your bot until you get something like this:

<!-- this is my retina screenshot, so we force width here -->
<img src="/media/posts/2016-03-15-chroma-bot-telegram-api/botfather-2.png" width="512px"/>

Note the token Botfather gave to you, because we will need it in a moment. Add `:telegram-token` to your configuration in `main.clj`, so the whole thing will look like this:

```clojure
(cfg/define {:port {:description "HTTP port"
                    :type :number
                    :default 8080}
             :telegram-token {:description "Token to connect to Telegram API"
                              :type :string
                              :required true
                              :secret true}})
```

> I strongly encourage you not to store any default values for sensitive tokens or keys from the start of your development. It is an important matter and do not put it away, deal with it immediately as you introduce this type of data to your app.

That is why there is no default value for the token and why we marked it as secret. Omniconf prints configuration when it validates it, so when it sees that mark, it will print `<SECRET>` instead of actual value.

But in this case, we won't be able to run our app as usual with `lein ring server` because there would be no token and no polling started. That is unacceptable! Let's fix it.

Replace `:ring` section in your `project.clj` file with following:

```clojure
:ring {:handler chroma-bot.handler/app
       :init chroma-bot.main/ring-init}
```

That will tell ring-plugin to call `ring-init` function from `chroma-bot.main` namespace when you run your server! Let's add it.

Here is updated content of `main.clj`:

```clojure
(ns chroma-bot.main
  "Responsible for starting application from command line"
  (:gen-class)
  (:require [taoensso.timbre :as log]
            [clojure.java.io :as io]
            [omniconf.core :as cfg]
            [ring.adapter.jetty :refer [run-jetty]]

            [telegram.core :as telegram]
            [telegram.api :as api]
            [chroma-bot.handler :refer [app]]))

(cfg/define {:port {:description "HTTP port"
                    :type :number
                    :default 8080}
             :telegram-token {:description "Token to connect to Telegram API"
                              :type :string
                              :required true
                              :secret true}})

(defn handler
  "Handles update object that the bot received from a Telegram API"
  [update]
  (when-let [message (:message update)]
    (api/send-message (-> update :message :chat :id)
                      (str "Hi there! 😊"))))

(defn init []
  (cfg/verify :quit-on-error true)
  (telegram/init! {:token (cfg/get :telegram-token)
                   :handlers [handler]
                   :polling true}))

(defn ring-init []
  (let [local-config "dev-config.edn"]
    (if (.exists (io/as-file local-config))
      (cfg/populate-from-file local-config)
      (log/warn "Can't find local dev configuration file" local-config))
    (init)))

(defn -main [& args]

  (cfg/populate-from-env)
  (cfg/verify :quit-on-error true)
  (log/info "Starting server")
  (run-jetty app {:port (cfg/get :port) :join? false}))  
```

* YES! Clojure can work with emojis! 👍
* `handler` is dumb function that will send the same reply for every incoming message
* now `-main` and `ring-init` will both call `init` to will start Telegram API, including long polling
* `ring-init` also checks if there is `dev-config.edn` file in your project root folder and if there is one, reads configuration from it

One last little thing: add mentioned above `dev-config.edn` line to your `.gitignore` file and never, **never ever**, commit your tokens to the repository.

Only after you done that, create `dev-config.edn` with following content:

```clojure
{:port 3000
 :telegram-token "PLACE YOUR ACTUAL TOKEN HERE, SERIOUSLY, DON'T FORGET TO GITIGNORE THIS FILE"}
```

Yes, this is Clojure data, *A superset of [edn](https://github.com/edn-format/edn) format is used by Clojure to represent programs* (as docs say), so it looks unsurprisingly like Clojure data structures.

Now finally you can start your bot with

    $ lein ring server

By the way, if you don't want `ring-plugin` to open a browser for you, use following command instead:

    $ lein ring server-headless

Then start conversation with your bot in telegram:

<!-- this is my retina screenshot, so we force width here -->
<img src="/media/posts/2016-03-15-chroma-bot-telegram-api/chroma-bot-alive.png" width="463px"/>

## It's alive!

Congratulations! You now can have your personal bot in Telegram. That is fantastic!

And because you already know some Clojure, you can teach your bot to do all kinds of fun stuff on your own, but here we are going to add some colors to it.

Unfortunately, that would be it for this episode. I hope you had a great time following through and excited about learning a couple of new things today.

If you enjoyed it, press like button or share this post with your friends. I hope to see you again in the next episode, bye!
