---
title: "Golang for Rubyists Part 1: Typing"
date: 2014-07-11
author: Alex Fong
github: actfong
Instagram: fromatob_engineering
---

Here at FromAtoB, we are making the transition from a Ruby on Rails monolith application to a service-oriented architecture where Go is becoming a more prominent language.

As we are making this transition, it also requires us to make the transition from Ruby to Go. In this serie of blog-posts, we would like to highlight some of the main differences between Ruby and Go. We hope that this content will be useful for any Ruby developer who considers learning Go.


# Typing in Ruby #

> "If it walks like a duck and it quacks like a duck, then it must be a duck"

Ruby is known to be a [“duck typed”](https://en.wikipedia.org/wiki/Duck_typing) language. The idea behind it, is that the type of an object does not matter, what matters is whether it can be used for a particular purpose.

As a result, whenever we assign variables, no type needs to be declared.
The only thing that is required from an object, is that it is able to respond to a specific message when it is needed.

Take the following simple application as an example.
Here, a PR Officer of a company can publish content to various social platforms. The published content contains some text and is appended with an URL, where viewers can navigate to for the original press release.

<script src="https://gist.github.com/actfong/920a543fd2daddee642d3dc6f0773c96.js"></script>

The main take-away from this snippet is that `PROfficer#publish` expects you to pass an object that responds to `#announce` as the second argument.

So if our PR Officer Marvin the Martian needs to beam this content to his friends in outer space, he could just plug in any object that responds to `#announce` and takes the content as an argument.
Whether this object is inherited from `PRChannel` or not,  and whether it does anything with the API-keys or the original URL of the press release..... the Ruby interpreter really does not care.

<script src="https://gist.github.com/actfong/e1cb91c9b48a7b316fc0bbe510f6460d.js"></script>

----

# Typing in Go #

In Go, typing is mainly done through Interfaces.
When you declare an interface, you define a set of methods that the implementing structs must be able to respond to.

In the following example, the PRChannel interface declares that any struct implementing `announce(content String)` is in effect a PRChannel.

The struct on its turn, doesn't need to explicitly declare that it implements an interface (unlike a language like Java). This is known as *structural typing*.

<script src="https://gist.github.com/actfong/bf72556d4dce5138c5ef642c80755628.js"></script>

A struct can also implement multiple interfaces, as long as it implements all methods declared in those interface declarations. In the following example, our Facebook struct implements both the PRChannel and Messenger interfaces.

<script src="https://gist.github.com/actfong/bb14c19946393808888bb0849dfedec0.js"></script>

Now let's revisit our PR officer Mavin the Martian's case. Based on our example in Ruby, we want a `PROfficer` to be able to publish content through a `PRChannel`.

The Facebook channel is already in place. If we want another channel to transmit messages to his friends in outer space, all that is needed is another struct (in our case, the `OuterSpaceTransmitter`) with its own implementation of `announce(content String)`

<script src="https://gist.github.com/actfong/31fbd33de44f00cc7a2c8b0ddf801978.js"></script>

---

# Wrapping Up #

Ruby is a duck typed language. The type of our objects is secondary to their behaviours. All that our Ruby Intepreter cares about is whether an object can respond to a specific message (i.e. whether it passes the Duck Test).

In Go, you use interface for this purpose. As an example, when you use an interface type for a function parameter, whenever you call the function, the compiler will check that the argument's type implements the interface.

Typing in Go is mainly done through interfaces. Structs can implement an interface by implementing all methods listed in the declaration of the corresponding interface.
