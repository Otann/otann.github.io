---
layout:   page
comments: true
sharable: true

title:  Chroma Bot
image:  /media/projects/chroma-bot/header.jpg
description: >
  Here I tell how to implement a dead simple bot for Telegram that detects colors in messages in hex form, like #dc322f, and sends back a picture filled with that color.
---

This project is a dead simple bot for Telegram that detects colors in messages in hex form (like `#dc322f`) and sends back a picture filled with that color.

I developed it to see how Telegram's Bot API work and to play with it.

The project turned out to be very simple, so I decided to use it for an opening for Web Fandango series. That simplicity allowed me to cover basic things, like projects, namespaces and routing for web services. This project is also an excellent illustration of how to use core.async.

You can check out sources on the GitHub [repository](https://github.com/Otann/chroma-bot).

Here is the list of posts from the series about the bot:

* ### [Episode 02. Chroma Bot Landing Page](/web-fandango/chroma-bot-1/)
  * How to start a Clojure application from a template
  * How to set the project's external dependencies
  * What are namespaces and how to use them
  * How to run an application locally
  * How to generate and serve an HTML page using Clojure data structures
  * How to deploy an app to Heroku

* ### [Episode 03. Chroma Bot Telegram API](web-fandango/chroma-bot-telegram-api/)
  * How to make an HTTP request and deal with a response
  * How to wrap the Telegram API in Clojure functions
  * What are two basic concurrency models
  * How to implement a long-polling using `core.async`
  * How to load and store secret tokens and sensitive keys
