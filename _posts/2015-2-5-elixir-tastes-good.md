---
layout: post
title: elixir tastes good
date: 2015-2-5
comments: true
categories:
  - programming
tags:
  - elixir
---

In the last six months I have become enamored with a new programming language. It started with curiosity and has grown into a bit of obsession. If I'm talking about programming with someone long enough, I will at some point start gushing about [elixir][0].

## What is elixir?

[0]: http://elixir-lang.org/

{% include wrapimage.html url="/assets/img/posts/2015-2-5-elixir-logo.png" %}

> Elixir is a dynamic, functional language designed for building scalable and maintainable applications.

> Elixir leverages the Erlang VM, known for running low-latency, distributed and fault-tolerant systems, while also being successfully used in web development and the embedded software domain.

## My introduction to elixir

The first time I heard about elixir was listening to [Bryan Hunter on the DotNetRocks podcast][0] while I drove to work, which really peaked my interest in elixir. At the time I was exploring Node.js on the side. I was looking for other things to stretch my programming skills some more.

A little while later I picked up [Elixir in Action][1] early access e-book on sale, and start diving into learning this new langauge. Elixir in Action is good, but I ended up getting [Programming Elixir][2] by Dave Thomas too. I found Programming Elixir to be a better intro book, then I circled back to Elixir in Action.

At this point I started spending my nights reading chapters and working through coding challenges to get comfortable. Writing code in elixir was a radical shift from my day job writing C#. The shift was worth all it for the enjoyment I got from programming in elixir.

## Functional Programming

I first started programming in high school, we started with QBASIC in 10th grade, PASCAL in 11th and I learned C++ in 12th grade. From high school to until I found elixir 12 years later all my experience was with object-oriented programming. Imperative languages were all I ever knew. Jumping into elixir as a purely functional language was a bit jarring.

The challenge of learning functional programming was very much worth the effort in my opinion. It has shown me different ways to think about problems, and even improved my OO code.

## Bitstring pattern matching

Once I got my feet wet with functional programming, and was exploring the features in elixir more I came across something that was truly exceptional, pattern matching bitstrings. Pattern matching as a whole is such a powerful feature. The ability to do that down to individual bits really blew me away.

This was the **Eureka** moment for me with elixir. I have spent the last several years of my professional career integrating dozens of hardware interfaces and network APIs. A major pain point I had dealt with was the bit bashing of taking a spec then translating that to objects to encode and decode data. This will be the base for communicating to some system. That integration point get tedious and can be prown to errors or at least constant maintence when firmwares get upgraded.

Pattern matching on the payloads as a whole would take a big chunk out of the development time for integrating a new system. But more than that the end result would be very easy to read and maintain code.

For example, I recently wrote an interface for a magnetic compass breakout board, the [HMC5883L][3]. Interfacing with this device consists of reading a 13 byte register over an I2C bus (I'll write about that further in a future post). The bytes break down like this:


Byte| Name | Access
---|---|---|
0|Configuration A&nbsp;| Read / Write
1|Configuration B&nbsp;| Read / Write
2|Mode|Read / Write
...|...|...
12|...|...

<br />
Decoding the first three bytes with elixir pattern matching bit syntax looks like this:

```elixir
#mock data, create a 13 byte binary to sub for data from compass
data = <<0,1,2,3,4,5,6,7,8,9,10,11,12>>

#pattern match the first three bytes and disregard the rest
<<cfga, cfgb, mode, _rest::binary>> = data

#decode Configuration A byte

#first bit is spare, followed by 2 bits for the averaging,
#3 bits for the data rate and the last 2 bits are the bias
<<_spare::size(1), bs_avg::size(2), bs_data_rate::size(3), bs_bias::size(2)>> = cfga

#Configuration B is similar, but with only one field

# 3 bits for the gain, then throw away the last 5 bits with _
<<bs_gain:: size(3), _ :: size(5)>> = cfgb

#Last is the mode

# 1 bit for high speed i2c is on or off, 5 spare bits then 2 bits for the mode setting
<<high_speed_i2c:: size(1), _:: size(5), bs_mode:: size(2)>> = mode

#after this I would process the variables to set the configuration state of the compass.
```

To do this same thing in C# would have taken many shifts and masks. Lots of <<, data-preserve-html-node="true" data-preserve-html-node="true" >> and & in my code. It can be easier in C and C++, but the pattern matching in elixir is just so much nicer to me. And to have that in a high level language is incredibly powerful.

As someone who likes to deal with hardware, this was a huge selling point for using elixir.

[Benjamin Tan has a great post going over elixir's bit syntax.][4]

## Concurrency & Fault Tolerance

While pattern matching is great, it is common to functional languages. The concurrency and fault tolerance models you get in elixir (from erlang) are not common. Unfortunately I would do a terrible job trying to explain these with any depth, so instead I suggest you read [The Hitchhiker's Guide to Concurrency][5] or watch [Understanding the Erlang Scheduler][6].

The actor model has gained traction in other languages, but the light weight processes, links, message passing and scheduling you get on the erlang VM are very unique. Being able to write concurrent code that can run locally or distributed easily is great. Writing that code and never worrying about threads is amazing.

## In closing

I really like elixir. I still have a lot to learn with it and functional programming in general, but I've enjoyed all the projects I've worked on in elixir. I'm very excited to see how much the community grows over the next few years and I would not be surprised to see elixir become a top 10 language. If elixir is not on your radar yet, I think it should be.

## Resources

### Books

* [Programming Elixir][2] by [Dave Thomas][7]
* [Elixir in Action][1] by [Saša Jurić][8]


### Videos

* [José Valim ElixirConf '14 Keynote][9]
* [Dave Thomas ElixirConf '14 Keynote][10]
* [Elixir Sips][11] - [Josh Adams][12]


### Podcasts

* [Mostly Erlang][13] - [Episode 19 Elixir w/ José Valim][14]
* [Functional Geekery][15] - [Episode 17 José Valim][16]
* [DotNetRocks: Programming in Elixir w/ Bryan Hunter][17]


### IRC

#elixir-lang on irc.freenode.net

[0]: http://www.dotnetrocks.com/default.aspx?showNum=995
[1]: http://www.manning.com/juric/
[2]: https://pragprog.com/book/elixir/programming-elixir
[3]: http://www.adafruit.com/products/1746
[4]: http://benjamintan.io/blog/2014/06/10/elixir-bit-syntax-and-id3/
[5]: http://learnyousomeerlang.com/the-hitchhikers-guide-to-concurrency
[6]: https://www.erlang-solutions.com/resources/webinars/understanding-erlang-scheduler
[7]: https://twitter.com/pragdave
[8]: https://twitter.com/sasajuric
[9]: https://www.youtube.com/watch?v=aZXc11eOEpI
[10]: https://www.youtube.com/watch?v=5hDVftaPQwY
[11]: http://elixirsips.com/
[12]: https://twitter.com/knewter
[13]: http://mostlyerlang.com/
[14]: http://mostlyerlang.com/2013/10/07/019-elixir-with-jose-valim/
[15]: http://www.functionalgeekery.com/
[16]: http://www.functionalgeekery.com/episode-17-jose-valim/
[17]: http://www.dotnetrocks.com/default.aspx?showNum=1080
